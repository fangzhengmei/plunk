# 邮件投递队列可靠性边界 · 代码理解

本文深入分析 Plunk 邮件投递队列的可靠性边界，重点聚焦四个核心问题：Worker PENDING 短路导致 BullMQ 重试失效、SENDING 状态永久丢失、SES/SNS Webhook 签名校验与幂等去重、Worker 并发拉取缺少乐观锁的重复发送风险。

---

## 一、PENDING 状态短路：BullMQ 重试如何被架空

### 1.1 问题代码的精确位置

[email-processor.ts#L85-L101](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L85-L101)：

```typescript
const email = await prisma.email.findUnique({
  where: {id: emailId},
  include: { contact: true, project: true, template: {select: {type: true}}, campaign: {select: {type: true}} },
});

if (!email) {
  throw new Error(`Email ${emailId} not found`);
}

if (email.status !== EmailStatus.PENDING) {   // ← 短路点
  return;                                      // ← 静默返回"成功"
}
```

### 1.2 失效路径的逐步推演

BullMQ 的重试机制依赖于一个前提：**job 函数抛出异常时，BullMQ 将 job 标记为 failed 并根据 backoff 策略调度重试**。但当前的代码在 catch 块中先将 Email 标记为 FAILED，然后才 throw——这导致了一个逻辑死结：

```
时序推演（以 SES 网络超时为例）：

T0  Worker 拉取 job {emailId: "abc"}
T1  查询 email → status = PENDING → 通过检查
T2  更新 status = SENDING
T3  EmailService.format() → 成功
T4  EmailService.compile() → 成功
T5  sendRawEmail() → 网络超时，抛出 Error
T6  catch 块：
      await prisma.email.update({status: FAILED, error: "Network timeout"})
T7  throw error → BullMQ 捕获，标记 job failed
T8  BullMQ 调度重试（2s 后）

─── 2 秒后 ───

T9  Worker 再次拉取同一 job {emailId: "abc"}
T10 查询 email → status = FAILED（不是 PENDING）
T11 if (email.status !== PENDING) → true
T12 return ← 静默成功，job 标记 completed

结果：邮件永远停留在 FAILED 状态，BullMQ 认为重试成功
```

**关键矛盾**：catch 块中的 `status = FAILED` 写操作与 `throw error` 的组合，使得重试时 PENDING 检查必定短路。Worker 的设计意图是「防止重复发送已完成的邮件」，但副作用是「防止了任何有意义的重试」。

### 1.3 为什么不用 PENDING 而用 SENDING 做短路检查

一个自然的修复思路是：将短路条件从 `status !== PENDING` 改为 `status === SENT || status === DELIVERED || ...`（即只短路终态），但这会引入另一个问题——**SENDING 状态下的重复发送风险**（详见第四章）。

另一种思路是：catch 块中不标记 FAILED，而是回退为 PENDING，让重试时能再次进入发送流程。但这也存在风险——如果 SES 实际上已经成功接受但网络超时（发送成功但响应丢失），回退 PENDING 会导致重复发送。

### 1.4 当前设计的隐含假设

当前代码隐含了一个假设：**所有发送失败都是不可恢复的**。这个假设在以下场景中成立：
- 域名未验证（每次都会失败）
- 联系人已退订（每次都会被拦截）
- 项目已禁用（每次都会被拦截）
- 钓鱼内容（每次都会被拦截）

但在以下场景中**不成立**：
- SES 临时网络抖动（下次可能成功）
- SES 速率限制（Throttling，过一会可能成功）
- Prisma 连接池耗尽（临时错误）

**结论**：当前重试配置（3 次、指数退避 2s/4s/8s）实际上只在 PENDING→SENDING 转换**之前**的异常中有效——即 `findUnique` 抛异常或项目禁用检查失败的场景。一旦进入 SENDING 后的异常路径，重试即失效。

### 1.5 可能的修复方向

**方案 A：条件性回退 PENDING**
```typescript
catch (error) {
  const isRetryable = error instanceof Error && (
    error.message.includes('throttl') ||
    error.message.includes('timeout') ||
    error.message.includes('ECONNRESET') ||
    error.message.includes('rate limit')
  );
  
  if (isRetryable && retryCount < MAX_RETRIES) {
    // 回退到 PENDING，让重试时能重新进入发送流程
    await prisma.email.update({
      where: {id: emailId},
      data: { status: EmailStatus.PENDING, error: error.message },
    });
  } else {
    // 不可重试的失败
    await prisma.email.update({
      where: {id: emailId},
      data: { status: EmailStatus.FAILED, error: error.message },
    });
  }
  throw error;
}
```

**方案 B：使用 Prisma 乐观锁替代 PENDING 检查**
```typescript
// 用原子 CAS 操作替代 read-then-check
const updated = await prisma.email.updateMany({
  where: { id: emailId, status: EmailStatus.PENDING },  // 条件更新
  data: { status: EmailStatus.SENDING },
});
if (updated.count === 0) {
  return; // 另一个 Worker 已经在处理
}
```
这同时解决了 PENDING 短路和并发重复发送两个问题（详见第四章）。

---

## 二、SENDING 状态永久丢失的异常场景

### 2.1 SENDING 状态的含义

[email-processor.ts#L124-L127](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L124-L127)：

```typescript
await prisma.email.update({
  where: {id: emailId},
  data: {status: EmailStatus.SENDING},
});
```

SENDING 是一个**瞬时状态**，理论上应该很快过渡到 SENT 或 FAILED。但以下异常可能导致邮件永久停留在此状态：

### 2.2 进程崩溃场景

```
T0  Worker 拉取 job
T1  status → SENDING
T2  format() → 成功
T3  compile() → 成功
T4  sendRawEmail() → SES 返回 MessageId
T5  ──── Worker 进程在此刻崩溃（OOM kill / SIGKILL / 硬件故障）────
T6  以下操作永远不会执行：
      - status → SENT
      - MeterService.recordEmailSent()
      - EventService.trackEvent('email.sent')
      - CampaignService.finalizeIfDone()
```

**后果**：
- Email 记录永远停留在 SENDING
- SES 实际上已经投递了邮件，用户收到了但系统不知道
- Campaign 可能永远停留在 SENDING 状态（因为 finalizeIfDone 永远不会被调用）
- 计费遗漏（MeterService 未扣费）

### 2.3 异常传播中断场景

```typescript
// email-processor.ts L122-L279
try {
  await prisma.email.update({status: SENDING});    // ← 已提交
  
  // ... 中间 30+ 行代码 ...
  
  const result = await sendRawEmail({...});          // ← SES 成功
  
  await prisma.email.update({status: SENT});        // ← 如果这里抛异常呢？
  
  // 以下代码都不会执行：
  await MeterService.recordEmailSent();              // 计费遗漏
  await EventService.trackEvent('email.sent');       // 事件丢失
  await CampaignService.finalizeIfDone();            // campaign 卡住
} catch (error) {
  await prisma.email.update({status: FAILED});       // ← SES 已成功但被标记 FAILED！
  throw error;
}
```

**最危险的场景**：`sendRawEmail()` 成功，但后续的 Prisma 写入（status→SENT）因为数据库连接池耗尽而失败。此时 catch 块将 Email 标记为 FAILED，但**邮件实际已发送**——导致用户收到邮件但系统显示发送失败。

### 2.4 无 SENDING 状态恢复机制

搜索整个代码库，**没有任何定时任务或补偿机制**来处理停留在 SENDING 状态的 Email 记录。当前的恢复依赖完全是手动的：

- Campaign 有 `finalizeIfDone()` 检查，但只在每次邮件发送后被调用——如果最后几封邮件都卡在 SENDING，finalizeIfDone 不会被触发
- 没有「扫描 SENDING 超过 N 分钟的记录并回退为 PENDING」的 cron job
- 没有 admin API 端点来手动重发 SENDING 状态的邮件

### 2.5 SENDING 丢失的影响矩阵

| 影响维度 | 后果 | 有自动恢复？ |
|---------|------|------------|
| 邮件实际投递 | 已发但系统不知道 | ❌ |
| Campaign 状态 | 可能永久 SENDING | ❌（finalizeIfDone 只在后续发送时触发） |
| 计费 | 遗漏（少扣费） | ❌ |
| 事件追踪 | email.sent 事件丢失 | ❌ |
| Workflow 触发 | 依赖 email.sent 的 workflow 不会触发 | ❌ |
| 数据统计 | 发送数被低估 | ❌ |

---

## 三、SES/SNS Webhook：注册、签名校验、幂等去重

### 3.1 SNS Topic 注册与 ConfigurationSet 关联

SES 事件通知的注册发生在 AWS 基础设施层（不在应用代码中），但应用通过以下机制与之关联：

[constants.ts#L100-L106](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/app/constants.ts#L100-L106)：

```typescript
export const SES_CONFIGURATION_SET = validateEnv('SES_CONFIGURATION_SET', 'plunk-configuration-set');
export const SES_CONFIGURATION_SET_NO_TRACKING = validateEnv('SES_CONFIGURATION_SET_NO_TRACKING', '');
export const TRACKING_TOGGLE_ENABLED = process.env.SES_CONFIGURATION_SET_NO_TRACKING !== undefined;
```

[SESService.ts#L211-L224](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/SESService.ts#L211-L224)：

```typescript
const configurationSetName =
  TRACKING_TOGGLE_ENABLED && !tracking
    ? SES_CONFIGURATION_SET_NO_TRACKING
    : SES_CONFIGURATION_SET;

const response = await ses.sendRawEmail({
  ConfigurationSetName: configurationSetName,   // ← 关联 ConfigurationSet
  ...
});
```

**完整注册链路**（AWS 侧，非代码）：

```
SES ConfigurationSet (plunk-configuration-set)
  └─ Event Publishing 规则
       ├─ 发送事件 (Send, Delivery, Bounce, Complaint, Open, Click)
       └─ 目标: SNS Topic (plunk-events)
            └─ SNS Subscription (HTTPS)
                 └─ Endpoint: https://api.useplunk.com/webhooks/sns
```

每封邮件发送时指定 `ConfigurationSetName`，SES 自动将事件发布到关联的 SNS Topic，SNS 通过 HTTPS POST 推送到应用。

### 3.2 SNS 签名校验的完整流程

[SecurityService.verifySnsSignature()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/SecurityService.ts#L177-L211) 实现了 AWS SNS 消息签名验证：

```
① 提取 SigningCertURL 和 Signature 字段
② 校验 SigningCertURL 的 hostname 匹配 /^sns\.[a-z0-9-]+\.amazonaws\.(com|cn)$/
③ 从 SigningCertURL 获取 X.509 证书（带内存缓存）
④ 构建签名字符串（buildSnsStringToSign）
⑤ 根据 SignatureVersion 选择算法（v1=RSA-SHA1, v2=RSA-SHA256）
⑥ crypto.createVerify() 验证签名
```

**签名字符串的构建规则**（[SecurityService.ts#L158-L168](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/SecurityService.ts#L158-L168)）：

```typescript
function buildSnsStringToSign(message: Record<string, string>): string {
  const fields =
    message['Type'] === 'Notification'
      ? ['Message', 'MessageId', 'Subject', 'Timestamp', 'TopicArn', 'Type']
      : ['Message', 'MessageId', 'SubscribeURL', 'Timestamp', 'Token', 'TopicArn', 'Type'];

  return fields
    .filter(key => message[key] !== undefined)
    .map(key => `${key}\n${message[key]}\n`)
    .join('');
}
```

注意：签名字符串**只包含特定字段的值**，不包括整个 JSON body。这是 AWS SNS 的标准签名算法——只签名关键字段，`Signature` 字段本身不参与签名计算。

**证书缓存**：[SecurityService.ts#L144-L156](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/SecurityService.ts#L144-L156) 使用模块级 `Map<string, string>` 缓存证书 PEM，避免每次 webhook 请求都向 AWS 请求证书。但缓存没有 TTL——一旦加载就永不过期，除非进程重启。在证书轮换场景下可能导致签名验证失败。

### 3.3 SubscriptionConfirmation 的 SSRF 防护

[Webhooks.ts#L48-L98](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L48-L98)：

```typescript
if (req.body.Type === 'SubscriptionConfirmation') {
  const subscribeURL: unknown = req.body.SubscribeURL;
  
  // 校验 URL 格式
  let parsedURL: URL;
  try { parsedURL = new URL(subscribeURL); } catch { return 400; }
  
  // 白名单主机名
  const SNS_HOST_RE = /^sns\.[a-z0-9-]+\.amazonaws\.(com|eu)$/;
  if (parsedURL.protocol !== 'https:' || !SNS_HOST_RE.test(parsedURL.hostname)) {
    return 400;
  }
  
  // 自动确认订阅
  const confirmResponse = await fetch(subscribeURL);
  ...
}
```

这里有三层 SSRF 防护：
1. URL 解析失败拒绝
2. 协议必须 HTTPS
3. 主机名白名单（`sns.*.amazonaws.com/cn`）

**但注意**：`SNS_HOST_RE` 与签名校验中的 `SNS_CERT_HOST_RE` 使用了**不同的 TLD 白名单**——前者允许 `.eu`，后者允许 `.cn`。这可能导致一个边界情况：来自 `amazonaws.eu` 的 SubscribeURL 被接受，但来自同一区域的 SigningCertURL 可能被签名校验拒绝（如果它使用 `amazonaws.cn` 域名）。实际中 AWS 不会跨区域混用域，但代码层面存在不一致。

### 3.4 Webhook 事件处理的幂等性分析

[Webhooks.ts#L288-L471](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L288-L471) 处理出站邮件事件。幂等性分析：

**有幂等保护的操作**：

| 操作 | 幂等机制 | 代码位置 |
|------|---------|---------|
| Open 事件的 openedAt | `if (!email.openedAt)` 只首次设置 | [Webhooks.ts#L342-L344](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L342-L344) |
| Click 事件的 clickedAt | `if (!email.clickedAt)` 只首次设置 | [Webhooks.ts#L359-L361](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L359-L361) |
| Open 计数 opens | `(email.opens \|\| 0) + 1`——每次都递增 | [Webhooks.ts#L345](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L345) |
| Click 计数 clicks | `(email.clicks \|\| 0) + 1`——每次都递增 | [Webhooks.ts#L364](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L364) |

**没有幂等保护的操作**：

| 操作 | 重复执行后果 | 代码位置 |
|------|-----------|---------|
| status 更新 | 同一事件重复推送到 SNS → 重复更新同一状态（幂等，无害） | [Webhooks.ts#L455-L458](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L455-L458) |
| contact.subscribed = false | Bounce/Complaint 重复设置 false（幂等） | [Webhooks.ts#L385-L388](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L385-L388) |
| EventService.trackEvent() | **每次都创建新的 Event 记录**——SNS 重试会导致重复事件 | [Webhooks.ts#L461](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L461) |
| Campaign 统计更新 | Open/Click 重复递增（在 handleWebhookEvent 分支，当前 webhook 路径未使用此分支） | N/A |
| SecurityService 检查 | 重复触发安全检查（有 Redis 缓存，5 分钟 TTL，影响有限） | [Webhooks.ts#L466-L468](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L466-L468) |

**最大的幂等问题**：`EventService.trackEvent()` 每次调用都创建新记录。SNS 的 at-least-once 投递语义意味着同一事件可能被推送多次，导致：

1. 同一 Bounce 事件创建多条 `email.bounce` 事件记录
2. 如果有 workflow 监听 `email.bounce`，会触发多次
3. 统计数据被重复计数

### 3.5 SNS 重试与 200 响应的交互

[Webhooks.ts#L472-L476](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L472-L476)：

```typescript
catch (error) {
  signale.error('[WEBHOOK] Error processing SNS webhook:', error);
  // Always return 200 to prevent SNS from retrying
  return res.status(200).json({success: true});
}
```

**关键设计决策**：即使处理失败也返回 200，阻止 SNS 重试。这是一个有意识的选择——宁可丢失单次事件，也不愿处理 SNS 重试带来的幂等问题。但这意味着 webhook 处理中的任何异常（数据库连接失败、Prisma 查询超时等）都会**静默丢失事件**。

### 3.6 Express 中间件对 SNS body 的处理

[app.ts#L54-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/app.ts#L54-L61)：

```typescript
// Specify that we need raw json for the webhook
this.app.use('/webhooks/incoming/stripe', raw({type: 'application/json'}));

// Set the content-type to JSON for any request coming from AWS SNS
this.app.use(function (req, res, next) {
  if (req.get('x-amz-sns-message-type')) {
    req.headers['content-type'] = 'application/json';
  }
  next();
});
```

SNS 发送的 Content-Type 是 `text/plain`，但 body 是 JSON 格式。中间件检测到 `x-amz-sns-message-type` header 后将 Content-Type 强制设为 `application/json`，使 Express 的 `json()` 中间件能正确解析 body。这是 SNS webhook 集成的标准做法。

---

## 四、Worker 并发拉取的重复发送风险

### 4.1 BullMQ Worker 并发模型

[email-processor.ts#L80-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L80-L88)：

```typescript
const worker = new Worker<SendEmailJobData>(
  emailQueue.name,
  async (job: Job<SendEmailJobData>) => { ... },
  {
    connection: emailQueue.opts.connection,
    concurrency,           // 默认 >= 5，根据 SES 配额可高达 MAX_CONCURRENCY
    limiter: {
      max: rateLimit,      // 每秒最多发送 rateLimit 封
      duration: 1000,
    },
  },
);
```

BullMQ 的 `concurrency` 参数允许**同一个 Worker 进程**同时处理多个 job。当 `concurrency > 1` 时，Worker 会并发拉取多个 job 并行执行。

### 4.2 单 Worker 进程内的并发竞态

由于 `concurrency >= 5`，以下场景在理论上是可能的：

```
Worker 进程内有两个并发执行的 job：

Job A (emailId: "aaa")                    Job B (emailId: "bbb")
  │                                          │
  ├─ findUnique("aaa") → PENDING            │
  │                                          ├─ findUnique("bbb") → PENDING
  ├─ update status = SENDING                │
  │                                          ├─ update status = SENDING
  ├─ format()                               │
  │                                          ├─ format()
  ├─ compile()                              │
  │                                          ├─ compile()
  ├─ sendRawEmail() → 成功                   │
  │                                          ├─ sendRawEmail() → 成功
  ├─ update status = SENT                   │
  │                                          ├─ update status = SENT
```

这个场景中两个 job 操作不同的 Email 记录，**没有竞态条件**。BullMQ 的 job 粒度是 emailId，每个 job 对应唯一的 Email 记录。

### 4.3 真正的重复发送风险：BullMQ job 去重

[QueueService.ts#L202-L216](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L202-L216)：

```typescript
public static async queueEmail(emailId: string, sourceType: EmailSourceType, delay?: number) {
  return emailQueue.add('send-email', {emailId}, {
    delay,
    jobId: `email-${emailId}`,   // ← 基于 emailId 的确定性 jobId
    priority: emailPriorityFor(sourceType),
  });
}
```

BullMQ 使用 `jobId` 做去重——如果 `email-${emailId}` 这个 jobId 已存在于队列中，`add()` 会**静默忽略**新 job（不报错，不覆盖）。这是第一层防重保护。

### 4.4 缺少乐观锁的风险场景

虽然有 jobId 去重，但以下场景仍然可能导致重复发送：

**场景：Worker 进程在 SES 调用后、数据库更新前崩溃**

```
T0  Worker 拉取 job email-abc
T1  findUnique → PENDING → 通过
T2  update status = SENDING
T3  sendRawEmail() → 成功！SES 已接受
T4  ──── 进程崩溃（SIGKILL / OOM）────
T5  update status = SENT → 未执行
T6  EventService.trackEvent → 未执行

─── 进程重启后 ───

T7  BullMQ 检测到 job 未完成，重新激活
T8  新 Worker 拉取 job email-abc
T9  findUnique → status = SENDING（不是 PENDING！）
T10 if (email.status !== PENDING) → return ← 短路
```

在这种情况下，邮件实际上**不会重复发送**——因为 SENDING 检查阻止了重新进入发送流程。但邮件会**永远停留在 SENDING 状态**（见第二章）。

**场景：数据库事务回滚但 SES 已发送**

```
T0  Worker 拉取 job email-abc
T1  findUnique → PENDING → 通过
T2  update status = SENDING → 成功
T3  sendRawEmail() → 成功，SES 返回 MessageId
T4  update status = SENT → Prisma 连接池耗尽，抛异常
T5  catch 块：update status = FAILED → 也可能失败（同一连接池问题）
T6  throw error → BullMQ 标记 job failed
T7  BullMQ 重试 → findUnique → FAILED → 短路 → return

结果：邮件已发送但被标记为 FAILED（或 SENDING）
      如果 catch 块的 update 也失败了，邮件停留在 SENDING
```

### 4.5 真正的并发重复发送：多个 Worker 进程

如果部署了**多个 Worker 进程**（水平扩展），BullMQ 的 `jobId` 去重仍然有效——因为 jobId 是全局的。但有一个边界场景：

```
T0  Worker-1 处理 email-abc：status = SENDING → SES 调用中...
T1  同一 email 的 campaign 触发了重新入队（代码中没有，但假设有）
T2  新 job email-abc 入队 → BullMQ 发现 jobId 已存在，拒绝
T3  不会重复发送 ✓
```

**当前代码不会产生多 Worker 重复发送**，因为：
1. `queueEmail()` 使用 `jobId: email-${emailId}` 做确定性去重
2. 没有「重新入队」的代码路径
3. `sendWorkflowEmail()` 等方法在创建 Email 记录后立即入队，不会重复调用

### 4.6 缺少乐观锁的实际风险定位

虽然没有多 Worker 重复发送的风险，但**PENDING→SENDING 的转换缺少原子性保障**：

```typescript
// 当前代码：非原子的 read-then-update
const email = await prisma.email.findUnique({where: {id: emailId}});  // READ
if (email.status !== EmailStatus.PENDING) return;                      // CHECK
await prisma.email.update({where: {id: emailId}, data: {status: SENDING}});  // WRITE
```

在 READ 和 WRITE 之间，另一个进程/线程可能已经更新了 status。虽然当前架构中不存在这种并发（每个 emailId 只有一个 job），但**代码层面缺少防御性约束**——如果未来引入了重发机制或 admin API 手动重试，就会触发竞态。

**乐观锁修复**（方案 B，第一章提到的）：

```typescript
// 原子 CAS：只有 status 仍为 PENDING 时才更新为 SENDING
const result = await prisma.email.updateMany({
  where: { id: emailId, status: EmailStatus.PENDING },
  data: { status: EmailStatus.SENDING },
});

if (result.count === 0) {
  // 另一个 Worker 已经在处理，或已被标记为非 PENDING
  return;
}
```

`updateMany` 的 WHERE 条件同时承担了「检查」和「更新」两个职责，在数据库层面保证了原子性。即使多个 Worker 同时尝试处理同一个 emailId，只有一个能成功更新。

### 4.7 SES sendRawEmail 的自然幂等性

即使发生重复发送，AWS SES 本身**不做消息去重**——相同的邮件内容调用两次 `sendRawEmail` 会产生两封不同的邮件（不同的 MessageId）。SES 的幂等性完全依赖调用方控制。

但 SESService 中的一个设计**间接提供了部分保护**：

[SESService.ts#L97-L103](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/SESService.ts#L97-L103)：

```typescript
const regex = /unsubscribe\/([a-f\d-]+)"/;
const containsUnsubscribeLink = regex.exec(content.html);

if (containsUnsubscribeLink?.[1]) {
  unsubscribeHeader = `List-Unsubscribe: <${DASHBOARD_URI}/unsubscribe/${unsubscribeId}>`;
}
```

List-Unsubscribe header 的存在让邮件客户端（Gmail 等）显示「退订」按钮。重复发送的邮件会有相同的 List-Unsubscribe URL，但邮件客户端不会去重——用户会收到两封邮件。

---

## 五、SENDING 状态恢复的缺失与补偿机制

### 5.1 当前没有补偿机制

搜索全部代码，**没有**以下任何一种机制：

1. ❌ 定时扫描 SENDING 超过 N 分钟的 Email 记录
2. ❌ 将超时 SENDING 记录回退为 PENDING 或 FAILED 的 cron job
3. ❌ Admin API 端点手动修复 SENDING 记录
4. ❌ Campaign 的超时检查（campaign 只在 `finalizeIfDone()` 时检查是否所有邮件都已终态）

### 5.2 Campaign 层面的间接保护

[CampaignService.finalizeIfDone()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/CampaignService.ts) 会在每封邮件发送后检查 campaign 是否所有邮件都已到达终态。但 SENDING 不是终态——如果 campaign 的最后一批邮件都卡在 SENDING，`finalizeIfDone()` 会认为还有未完成的邮件，campaign 永远停留在 SENDING 状态。

唯一的间接恢复路径是 [QueueService.cancelAllProjectJobs()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L609-L612)：

```typescript
const failed = await prisma.email.updateMany({
  where: {projectId, status: EmailStatus.PENDING},
  data: {status: EmailStatus.FAILED, error: 'Project is disabled'},
});
```

但这只处理 PENDING 状态——**SENDING 状态的记录不会被清理**。

### 5.3 建议的补偿机制

**方案：定时扫描 SENDING 超时记录**

```typescript
// 每分钟执行的 cron job
async function recoverStaleSendingEmails() {
  const timeout = new Date(Date.now() - 10 * 60 * 1000); // 10 分钟
  const stale = await prisma.email.findMany({
    where: {
      status: EmailStatus.SENDING,
      updatedAt: { lt: timeout },
    },
  });
  
  for (const email of stale) {
    // 无法确定 SES 是否已发送，保守回退为 FAILED
    await prisma.email.update({
      where: {id: email.id},
      data: {
        status: EmailStatus.FAILED,
        error: 'Sending timeout - worker may have crashed',
      },
    });
    
    if (email.campaignId) {
      await CampaignService.finalizeIfDone(email.campaignId);
    }
  }
}
```

**为什么回退为 FAILED 而不是 PENDING**：因为 SENDING 意味着 SES 可能已经接受了邮件，回退为 PENDING 会导致重复发送。FAILED 更安全——宁可丢失一封邮件也不重复发送。

---

## 六、可靠性边界全景总结

### 6.1 已有的保护机制

| 保护层 | 机制 | 覆盖的风险 |
|-------|------|-----------|
| Job 去重 | `jobId: email-${emailId}` | 防止同一邮件重复入队 |
| PENDING 短路 | `status !== PENDING → return` | 防止已发送/已失败的邮件被重复处理 |
| 域名校验 | `DomainService.verifyEmailDomain()` | 防止未验证域名发送 |
| 退订拦截 | `!contact.subscribed → FAILED` | 合规保护 |
| 钓鱼检测 | `SecurityService.checkPhishingContent()` | 内容安全保护 |
| 计费限额 | `BillingLimitService.checkLimit()` | 防止超量发送 |
| SNS 签名校验 | `verifySnsSignature()` | 防止伪造 webhook |
| SubscribeURL SSRF 防护 | 白名单主机名校验 | 防止 SSRF 攻击 |
| Webhook 始终返回 200 | 阻止 SNS 重试 | 避免幂等问题（但引入事件丢失风险） |

### 6.2 缺失的保护机制

| 缺失 | 风险 | 严重程度 |
|------|------|---------|
| PENDING→SENDING 原子转换 | 理论上的并发竞态（当前无实际触发路径） | 中（未来风险） |
| BullMQ 重试有效性 | SES 临时故障不可自动恢复 | 高 |
| SENDING 超时恢复 | Worker 崩溃后邮件永久卡在 SENDING | 高 |
| Webhook 事件去重 | SNS 重试导致重复事件记录 | 中 |
| SES 发送后确认 | 发送成功但数据库更新失败导致状态不一致 | 高 |
| 证书缓存 TTL | SNS 签名证书轮换后验证失败 | 低 |

### 6.3 可靠性改进路线图

1. **P0（立即）**：将 PENDING→SENDING 的 `findUnique + update` 改为 `updateMany` 原子 CAS，消除竞态窗口
2. **P0（立即）**：添加 SENDING 超时恢复 cron job（10 分钟阈值，回退 FAILED）
3. **P1（短期）**：实现条件性重试——区分可重试错误（网络超时、SES Throttling）与不可重试错误（域名未验证），对可重试错误回退 PENDING
4. **P1（短期）**：Webhook 事件幂等——基于 `{messageId, eventType}` 做去重，防止 SNS 重试创建重复 Event 记录
5. **P2（中期）**：实现至少一次发送保障——在 `sendRawEmail()` 成功后、`update status=SENT` 前增加重试逻辑，确保数据库状态最终一致
6. **P2（中期）**：SNS 签名证书缓存增加 TTL（如 24 小时），避免证书轮换后验证失败
