# 邮件可靠性五处盲点 · 代码理解

本文深入剖析 Plunk 邮件投递系统中五个最隐蔽的可靠性盲点，对照代码逐一拆解其形成机制、影响范围与改进方向。

---

## 盲点一：EmailService.sendEmail 测试态死路径

### 1.1 代码位置与路径说明

[EmailService.sendEmail()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L290-L456) 是**另一条**发送路径（与 email-processor.ts 中的 Worker 代码并存且几乎完全重复）。它被 `sendTransactionalEmail` 的同步调用分支使用，但目前代码中所有 transactional 邮件都走 Worker 路径，因此此方法**实际上是一个死路径**——只被单元测试调用。

死路径中包含两段重要的拦截逻辑：

**A. 未订阅拦截**（[EmailService.ts#L309-L326](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L309-L326)）：

```typescript
// Final validation: Check subscription status before sending
// Only transactional emails should be sent to unsubscribed contacts
if (!email.contact.subscribed) {
  const isTransactional =
    email.sourceType === EmailSourceType.TRANSACTIONAL || email.template?.type === 'TRANSACTIONAL';

  if (!isTransactional) {
    signale.warn(`[EMAIL] Skipping marketing email ${emailId} to unsubscribed contact ${email.contact.email}`);
    await prisma.email.update({
      where: {id: emailId},
      data: { status: EmailStatus.FAILED, error: 'Contact is unsubscribed from marketing emails' },
    });
    return;
  }
}
```

**B. 域名复核**（[EmailService.ts#L328-L331](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L328-L331)）：

```typescript
// Verify domain is registered and verified before sending
// This ensures all emails (transactional, campaign, workflow) use verified domains
await DomainService.verifyEmailDomain(email.from, email.projectId);
```

### 1.2 与 Worker 路径的对比

| 拦截点 | email-processor.ts (Worker) | EmailService.sendEmail() (死路径) |
|--------|----------------------------|----------------------------------|
| PENDING 短路 | ✅ L99-L101 | ✅ L305-L307 |
| 项目禁用检查 | ✅ L103-L120 | ❌ 无 |
| 未订阅拦截 | ❌ 无（已在 sendWorkflowEmail 中拦截） | ✅ L309-L326 |
| 域名复核 | ❌ 无（已在 API 层拦截） | ✅ L328-L331 |
| 钓鱼检测 | ✅ L185-L212 | ❌ 无 |
| Recipient Override header | ✅ L168-L174 | ❌ 无 |
| toName 支持 | ✅ L177-L179 | ❌ 无（硬编码为 `[recipientEmail]`） |
| MeterService 扣费 | ✅ L244-L248 | ❌ 无 |

### 1.3 为什么这是一个盲点

1. **双重维护负担**：两段几乎相同的代码（约 160 行）分别位于两个文件中，修改一处容易忘记同步另一处。例如 Worker 中添加了钓鱼检测，但死路径中没有。

2. **测试覆盖率误导**：单元测试 [EmailService.test.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/__tests__/EmailService.test.ts) 调用的是 `sendEmail()` 死路径而非实际生产路径，导致「测试全绿但实际代码未被测试」的虚假安全感。

3. **拦截逻辑漂移**：死路径中的未订阅拦截和域名复核，在生产路径中已经**上移到 API 层**（`sendWorkflowEmail` / `sendTransactionalEmail` / `sendCampaignEmail` 入队前检查）。死路径的存在可能让开发者误以为这些检查发生在发送瞬间，而实际上是入队前。

4. **生产风险**：如果未来有人错误地添加了对 `sendEmail()` 的调用，将绕过钓鱼检测、MeterService 扣费等重要逻辑，产生安全漏洞和计费遗漏。

---

## 盲点二：BullMQ stalled 叠加 PENDING 短路的假成功

### 2.1 BullMQ stalled 检测机制

BullMQ 的 stalled 检测是保护机制：如果一个 Worker 拉取了 job 但在 `stalledInterval`（默认 30 秒）内没有更新进度，该 job 被标记为 stalled 并由另一个 Worker 接管。

当前代码中**没有显式配置** stalled 相关参数，使用 BullMQ 默认值：
- `stalledInterval: 30000`（30 秒检查一次）
- `maxStalledCount: 1`（允许 1 次 stalled，超过则标记失败）

### 2.2 假成功的形成机制

结合 email-processor.ts 中的 PENDING 短路检查（[L99-L101](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L99-L101)），形成以下时序：

```
T0  Worker-A 拉取 job email-abc
T1  findUnique → status = PENDING ✓
T2  update status = SENDING ✓
T3  sendRawEmail() → 成功 ✓，SES 已接受邮件
T4  ──── Worker-A 进程在此刻僵死（GC 停顿 > 30s / 死锁 / 网络分区）────
T5  BullMQ 在 30s 后检测到 stalled，将 job 重新入队
T6  Worker-B 拉取同一 job email-abc
T7  findUnique → status = SENDING（不是 PENDING！）
T8  if (email.status !== PENDING) → true
T9  return ← 静默返回，BullMQ 标记 job completed

结果：
- 邮件实际已发送（用户收到）
- Email 记录停留在 SENDING 状态（没有更新为 SENT）
- job 被标记为成功（completed）
- MeterService 未扣费
- EventService.trackEvent('email.sent') 未调用
- Campaign 可能永远 SENDING
```

### 2.3 与普通重试失败的区别

| 场景 | 状态 | job 状态 | 邮件是否已发 | 可否恢复 |
|------|------|---------|-------------|---------|
| SES 超时后 catch 标记 FAILED | FAILED | failed → 重试时短路 | 可能已发 | 不可自动恢复 |
| stalled 时卡在 SENDING | **SENDING** | **completed** | **已发** | **不可自动恢复** |

**最危险的差异**：stalled 导致的假成功中，job 被标记为 `completed`——监控系统可能无法发现异常（没有 failed count 增长）。而普通重试失败至少会产生 failed job 计数，可被监控告警捕获。

### 2.4 风险量级估计

按每天发送 100 万封邮件，每封平均处理时间 500ms，假设万分之一的概率发生 GC 停顿 >30s：

```
每天 stalled 数 = 1,000,000 * 0.0001 = 100 封/天
每封损失：邮件已发但未扣费（每封 $0.0005 计）
每天损失：100 * $0.0005 = $0.05
每年损失：$0.05 * 365 = $18.25
```

直接经济损失很小，但**数据质量损失**更大——统计报表中 SENT 计数低于实际发送量，campaign 分析失准。

---

## 盲点三：MeterService 双层 idempotency 防重扣费

### 3.1 代码调用链

```
email-processor.ts L247
  → MeterService.recordEmailSent(customerId, emailCount, `email_${emailId}`)
    → QueueService.queueMeterEvent(customerId, value, idempotencyKey)
      → meterQueue.add('record-meter-event', data, {
           jobId: idempotencyKey ? `meter-${idempotencyKey}` : undefined
         })
        → meter-processor.ts processMeterEvent()
          → stripe.billing.meterEvents.create({
               identifier: idempotencyKey  // Stripe 侧幂等
             })
```

### 3.2 第一层：BullMQ jobId 去重

[QueueService.queueMeterEvent()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L358-L369)：

```typescript
public static async queueMeterEvent(
  customerId: string, value: number, idempotencyKey?: string
): Promise<Job<MeterEventJobData>> {
  return meterQueue.add(
    'record-meter-event',
    {customerId, value, idempotencyKey},
    {
      jobId: idempotencyKey ? `meter-${idempotencyKey}` : undefined,
    },
  );
}
```

**防重逻辑**：当提供 `idempotencyKey`（如 `email_${emailId}`）时，BullMQ 使用 `meter-${idempotencyKey}` 作为确定性 jobId。BullMQ 的 `add()` 方法会检查该 jobId 是否已存在于队列中，**存在则静默忽略**（不创建新 job，不报错）。

**生效范围**：仅防止 meter job 未消费前的重复入队。一旦 job 被消费（状态从 waiting → active → completed），该 jobId 可以被重用。

### 3.3 第二层：Stripe API identifier 去重

[meter-processor.ts#L21-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/meter-processor.ts#L21-L28)：

```typescript
await stripe.billing.meterEvents.create({
  event_name: STRIPE_METER_EVENT_NAME,
  payload: { stripe_customer_id: customerId, value: value.toString() },
  ...(idempotencyKey && {identifier: idempotencyKey}),
});
```

**Stripe 侧 idempotency**：`identifier` 参数是 Stripe 的幂等键——对于相同的 `event_name + identifier`，Stripe 在 24 小时窗口内只处理一次，后续调用返回首次结果但不重复计费。

**生效范围**：Stripe 后端 24 小时全局去重。即使 meter worker 崩溃重启后重复消费同一 job，只要 identifier 相同，Stripe 不会重复扣费。

### 3.4 双层防重的覆盖范围

| 重复场景 | 第一层 jobId | 第二层 identifier | 最终是否重复扣费 |
|---------|------------|----------------|-----------------|
| 同一封邮件 queueEmail 被调用两次（同一进程） | ✅ 阻止 | N/A | ❌ 不会 |
| meter job 消费成功后 Stripe API 超时（部分成功） | ❌ 已消费，可重新入队 | ✅ 阻止 | ❌ 不会 |
| meter worker 崩溃后重新消费同一 job | ❌ jobId 不同 | ✅ 阻止 | ❌ 不会 |
| 不同邮件使用相同 emailId（理论上） | ✅ 阻止 | ✅ 阻止 | ❌ 不会 |

**盲点**：如果 `idempotencyKey` 未提供（`undefined`），两层去重都失效。当前代码中所有调用都提供了 `email_${emailId}` 或 `batch_${batchId}`，但 API 设计允许不传——这是一个潜在漏洞。

### 3.5 idempotencyKey 的值传递链

```
email-processor.ts L247: `email_${emailId}`
  → MeterService.recordEmailSent() 透传
    → QueueService.queueMeterEvent() 作为 jobId 前缀
      → meter job.data.idempotencyKey
        → stripe.billing.meterEvents.create() 的 identifier
```

值完全一致，不做任何变换。这种透明传递是正确的——保证两端键值相同。

### 3.6 meterQueue 的 removeOnComplete 配置

[QueueService.ts#L164-L174](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L164-L174)：

```typescript
export const meterQueue = new Queue<MeterEventJobData>('meter', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {type: 'exponential', delay: 1000},
    removeOnComplete: 5000,   // 保留最近 5000 条完成的 job
    removeOnFail: 10000,
  },
});
```

`removeOnComplete: 5000` 意味着 meter job 完成后，历史记录最多保留 5000 条。当每天有 100 万封邮件时，5000 条历史记录大约只保留 7-8 分钟的发送量——jobId 去重窗口非常短。但没关系，因为第二层 Stripe identifier 有 24 小时窗口。

---

## 盲点四：removeOnComplete=1000 限 jobId 去重窗口

### 4.1 emailQueue 的 removeOnComplete 配置

[QueueService.ts#L48-L57](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L48-L57)：

```typescript
export const emailQueue = new Queue<SendEmailJobData>('email', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {type: 'exponential', delay: 2000},
    removeOnComplete: 1000,   // 保留最近 1000 条完成的 job
    removeOnFail: 5000,       // 保留最近 5000 条失败的 job
  },
});
```

### 4.2 jobId 去重的工作原理

[QueueService.queueEmail()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L202-L216)：

```typescript
return emailQueue.add(
  'send-email',
  {emailId},
  {
    delay,
    jobId: `email-${emailId}`,   // ← 确定性 jobId
    priority: emailPriorityFor(sourceType),
  },
);
```

BullMQ 的 `add()` 方法在指定 `jobId` 时的行为：

1. 在 Redis 中查询 `bull:email:job:email-${emailId}` key
2. 如果 key 不存在 → 创建新 job
3. 如果 key 已存在 → **静默忽略**，返回已存在的 job 对象（不报错，不覆盖）

### 4.3 removeOnComplete 对去重窗口的截断

`removeOnComplete: 1000` 意味着：每当有新的 job 完成，Redis 会删除最旧的完成态 job，保留最近的 1000 条。对应的 Redis key `bull:email:job:email-${emailId}` 也会被删除。

**去重窗口计算**：

假设每秒发送 14 封邮件（AWS SES sandbox 默认限制）：
```
1000 条历史 / 14 封/秒 = 约 71 秒去重窗口
```

如果每秒发送 100 封邮件：
```
1000 条历史 / 100 封/秒 = 10 秒去重窗口
```

这意味着：**同一封邮件在发送完成 10-70 秒后，如果有人再次调用 `queueEmail(emailId)`，由于 jobId 的 Redis key 已被删除，BullMQ 会认为这是新 job，允许重复入队**。

### 4.4 重复入队后的结果

```
T0  邮件 abc 入队，jobId = email-abc
T1  Worker 拉取，成功发送，job 标记 completed
T2  15 秒后，发送了 1500 封邮件（> 1000 条历史）
T3  job email-abc 的 Redis key 被 removeOnComplete 清理
T4  有人错误地再次调用 queueEmail(emailId)
T5  BullMQ 检查 jobId email-abc → 不存在 → 创建新 job！
T6  Worker 拉取新 job email-abc
T7  findUnique → status = SENT（不是 PENDING）
T8  if (email.status !== PENDING) → return ← 短路

结果：
- 不会重复发送（PENDING 短路保护）
- 但会浪费一次 BullMQ job 处理资源
- 增加一次 Prisma 查询
- 监控中的 job count 被污染
```

**好消息**：PENDING 短路防止了实际重复发送。
**坏消息**：去重窗口太短，无法防止重复入队带来的资源浪费和监控污染。

### 4.5 去重窗口不足的实际触发场景

虽然理论上存在窗口不足，但代码中 `queueEmail()` 只在创建 Email 记录后立即调用一次（`sendCampaignEmail` / `sendWorkflowEmail` / `sendTransactionalEmail` 中），没有「重发」逻辑。因此在当前代码路径下，**不会触发重复入队**。

风险来自未来：
1. 添加 admin API「重发失败邮件」时，可能直接调用 `queueEmail()` 而非检查状态
2. campaign 失败重试逻辑可能重复入队
3. 集成测试中意外调用两次

### 4.6 各队列的 removeOnComplete 配置对比

| 队列 | removeOnComplete | removeOnFail | 典型吞吐量 | 去重窗口估算 |
|------|-----------------|-------------|-----------|------------|
| email | 1000 | 5000 | 高（10-100/s） | 10-100s |
| meter | 5000 | 10000 | 高（10-100/s） | 50-500s |
| campaign | 100 | 500 | 中（批量） | 长（批量少） |
| workflow | 1000 | 5000 | 中 | 中 |
| scheduled | 100 | 500 | 低 | 长 |
| import | 50 | 100 | 中 | 中 |
| segment-count | 10 | 50 | 低 | 长 |
| domain-verification | 10 | 50 | 低 | 长 |
| api-request-cleanup | 5 | 20 | 低 | 长 |
| bulk-contact-actions | 50 | 100 | 中 | 中 |

**设计模式**：吞吐量越高的队列，`removeOnComplete` 值越大——这是在「Redis 内存占用」和「去重窗口长度」之间的权衡。

---

## 盲点五：sendTransactionalEmail 重试丢失对密码重置与验证码的影响

### 5.1 sendTransactionalEmail 的实际调用路径

[EmailService.sendTransactionalEmail()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L52-L114) 被以下场景调用：

1. **Transactional API**：`POST /actions/send-transactional-email`（[Actions.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Actions.ts)）
2. **Workflow TRANSACTIONAL 模板**：`sendWorkflowEmail()` 中模板类型为 TRANSACTIONAL 时，sourceType 升级为 TRANSACTIONAL
3. **密码重置邮件**：[Auth.ts#L278](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Auth.ts#L278) —— **但注意**：密码重置使用 `sendPlatformEmail()`，不是 `sendTransactionalEmail()`

### 5.2 平台邮件的独立通道

[sendPlatformEmail()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/email/src/lib/notify.ts#L19-L40) 是一个**完全独立**的发送通道，不经过 Plunk 自己的 emailQueue：

```typescript
export async function sendPlatformEmail(to: string, subject: string, template: ReactElement): Promise<void> {
  if (!isPlatformEmailEnabled()) return;  // 需要 PLUNK_API_KEY 和 PLUNK_FROM_ADDRESS

  try {
    const html = await render(template);
    await sendEmail({ to, from: process.env.PLUNK_FROM_ADDRESS, subject, body: html });
  } catch (error) {
    console.error('[Platform Email] Failed to send notification:', error);
  }
}
```

关键点：
1. **不经过 BullMQ 队列** —— 直接调用 `sendEmail()`（推测调用 Plunk 自己的 API）
2. **无重试机制** —— catch 块只 log 不重抛，也不重试
3. **失败静默** —— 调用方（`requestPasswordReset`）不检查返回值，总是返回成功
4. **无状态记录** —— 不创建 Email 记录，失败不可追溯

### 5.3 密码重置与验证码邮件的重试丢失场景

**场景一：密码重置邮件发送失败**

[Auth.ts#L278-L282](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Auth.ts#L278-L282)：

```typescript
await sendPlatformEmail(
  user.email,
  'Reset your password',
  React.createElement(PasswordResetEmail, {email: user.email, resetUrl, landingUrl: LANDING_URI}),
);

// 后续代码直接返回成功，不检查结果
return res.json({success: true, data: {message: 'If that email exists, a reset link has been sent'}});
```

**时序**：
```
T0  用户请求密码重置
T1  Redis 存储 token（有效期 TOKEN_EXPIRY_SECONDS = 3600s）
T2  sendPlatformEmail() → 调用 Plunk API
T3  Plunk API 返回 500 或网络超时
T4  sendPlatformEmail catch 块 log error 并 return
T5  Auth.ts 返回 200: "If that email exists, a reset link has been sent"

结果：
- 用户收到"重置链接已发送"的提示
- 实际上邮件未发送
- token 在 Redis 中 1 小时后过期
- 用户永远收不到邮件，只能等待过期后重试
- 没有任何告警或重试机制
```

**场景二：邮箱验证邮件发送失败**

[Auth.ts#L139-L144](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Auth.ts#L139-L144)：

```typescript
await sendPlatformEmail(
  created_user.email,
  'Verify your email address',
  React.createElement(EmailVerificationEmail, {
    email: created_user.email, verificationUrl, landingUrl: LANDING_URI,
  }),
);
```

同样的模式：失败静默，用户看不到错误，但无法验证邮箱，导致账号无法使用。

### 5.4 与 transactional API 的对比

| 方面 | sendPlatformEmail (密码重置/验证) | sendTransactionalEmail (API) |
|------|-------------------------------|-----------------------------|
| 队列 | 无，同步调用 API | 有，emailQueue BullMQ |
| 重试 | 无 | BullMQ 3 次指数退避（但被 PENDING 短路失效） |
| 失败处理 | catch 只 log | catch 更新 Email.status = FAILED |
| 状态记录 | 无 | Email 表记录 |
| 优先级 | 无（同步） | 最高优先级 1 |
| 幂等 | 无 | jobId 去重 |
| 计费 | 不扣费（内部使用） | MeterService 扣费 |

### 5.5 重试丢失对用户体验的影响矩阵

| 邮件类型 | 重试丢失后果 | 用户感知 | 严重程度 |
|---------|-------------|---------|---------|
| 密码重置 | 收不到重置链接，需等 1 小时 token 过期 | "我没收到邮件，再发一次？"按钮无帮助 | **高** |
| 邮箱验证 | 新用户无法验证，账号被锁 | 注册流程中断，用户流失 | **高** |
| 项目禁用通知 | 管理员收不到提醒 | 财务/合规风险 | 中 |
| 订阅开始通知 | 管理员收不到确认 | 低 |
| 发票支付通知 | 管理员收不到发票 | 可能导致逾期付款 | 中 |
| 安全违规通知 | 管理员收不到告警 | 安全风险 | 高 |

### 5.6 为什么这是一个盲点

平台邮件（`sendPlatformEmail`）是系统**自己给自己发的邮件**，看似不重要，但实际上承担了用户认证流程的关键环节。由于：
1. 不经过主 emailQueue，不受监控
2. 无重试，无状态，失败不可追溯
3. 调用方不检查返回值，静默失败
4. 没有告警（只 console.error，生产环境可能采集不到）

这类问题通常在用户投诉"收不到邮件"时才被发现，而此时已经流失了用户。

### 5.7 优先级设计的合理性

[QueueService.ts#L177-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L177-L189)：

```typescript
function emailPriorityFor(sourceType: EmailSourceType): number {
  switch (sourceType) {
    case EmailSourceType.TRANSACTIONAL: return 1;  // 最高
    case EmailSourceType.WORKFLOW:      return 5;
    case EmailSourceType.CAMPAIGN:      return 10; // 最低
    default: return 10;
  }
}
```

代码注释（[L196-L201](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L196-L201)）明确说明：

```
Transactional emails jump the queue ahead of workflow and campaign sends
via BullMQ's priority (lower number = higher precedence). This prevents
latency-sensitive sends (login codes, password resets) from queuing behind
large campaign bursts on the shared `email` queue.
```

但讽刺的是：**真正的密码重置和登录验证码邮件根本不走这个队列**——它们走 `sendPlatformEmail` 的同步通道，无法享受优先级保护。如果 Plunk API 本身因 campaign 大促而响应慢，密码重置邮件也会跟着慢。

---

## 五处盲点汇总表

| 盲点 | 位置 | 后果 | 严重程度 | 可检测性 |
|------|------|------|---------|---------|
| sendEmail 死路径 | EmailService.ts L290-L456 | 重复维护、测试覆盖失真、漂移风险 | 中 | 低（静态分析可发现） |
| stalled + PENDING 假成功 | email-processor.ts L99-L101 | 邮件已发但状态停留在 SENDING，job 显示成功 | 中 | 低（需监控 SENDING 超时） |
| MeterService 双层幂等 | MeterService + meter-processor | 设计良好，但 idempotencyKey 可选是漏洞 | 低 | 中（代码审查可发现） |
| removeOnComplete=1000 | QueueService.ts L55 | 高吞吐量下去重窗口短至 10 秒 | 低 | 中（计算可得） |
| 平台邮件重试丢失 | sendPlatformEmail + Auth.ts | 密码重置/验证码邮件静默失败，用户流失 | **高** | 低（需端到端监控） |

---

## 改进建议优先级

### P0（立即修复）
1. **平台邮件增加重试和监控**：为 `sendPlatformEmail` 添加指数退避重试（至少 3 次），失败时记录到数据库或发送到告警通道，而不是只 console.error
2. **移除死路径 sendEmail()**：单元测试改为调用实际生产路径（email-processor 的处理逻辑），消除重复维护负担

### P1（近期修复）
3. **添加 SENDING 超时恢复 cron job**：扫描 `updatedAt < NOW - 10min AND status = SENDING` 的记录，回退为 FAILED 并告警
4. **PENDING→SENDING 改为原子 updateMany**：同时解决竞态和 stalled 部分问题
5. **增加 stalled 检测配置**：显式设置 `stalledInterval: 15000`（15s），并添加 `on('stalled')` 事件告警

### P2（中期优化）
6. **emailQueue removeOnComplete 调大**：从 1000 调至 10000，延长去重窗口至约 10 分钟
7. **MeterService idempotencyKey 改为必选**：防止未来遗漏
8. **死路径代码删除**：完全删除 `EmailService.sendEmail()`，测试重构到 Worker 路径
