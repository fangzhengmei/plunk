# Plunk 邮件投递退信状态机笔记

## 一、EmailStatus 枚举（状态全集）

定义于 [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/packages/db/prisma/schema.prisma#L777-L788)：

```
PENDING     // 入队等待发送
SENDING     // Worker 正在处理
SENT        // SES 接受了邮件（拿到 messageId）
DELIVERED   // SES 确认投递到收件方 MTA
RECEIVED    // 入站邮件（SES Receiving 收到的）
OPENED      // 收件人打开
CLICKED     // 收件人点击链接
BOUNCED     // 退信（硬退/软退统一用此状态，但处理逻辑不同）
COMPLAINED  // 收件人举报垃圾邮件
FAILED      // 发送失败（应用层/SES 拒绝）
```

---

## 二、状态来源与完整流转路径

### 2.1 正常投递主路径

```
PENDING → SENDING → SENT → DELIVERED → OPENED → CLICKED
```

触发链路：

1. **创建邮件** — [EmailService](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L89-L105) 以 `status: PENDING` 写入 DB，调用 `QueueService.queueEmail()` 入 BullMQ 队列
2. **Worker 消费** — [email-processor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L82-L280) 从队列取 job：
   - 校验 `status === PENDING` 才继续
   - 检查项目是否 disabled → 是则直接标 FAILED
   - 更新 `SENDING`，调 SES `sendRawEmail`
   - 成功 → 更新 `SENT` + `sentAt` + `messageId`，追踪 `email.sent` 事件
   - 失败 → 更新 `FAILED` + `error`，**throw 让 BullMQ 触发重试**
3. **SNS Webhook 回写** — SES 通过 SNS 推送投递/打开/点击/退信/投诉事件到 [Webhooks.receiveSNSWebhook](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L38-L477)，由 `messageId` 查回 Email 记录并更新状态

### 2.2 硬退（Permanent Bounce）

**触发**：SES SNS 推送 `eventType: "Bounce"` 且 `bounceType: "Permanent"`

**代码位置**：[Webhooks.ts L374-L396](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L374-L396)

**处理动作**：

| 动作 | 代码 |
|------|------|
| Email 状态 → BOUNCED | `updateData.status = EmailStatus.BOUNCED` |
| 记录 bouncedAt | `updateData.bouncedAt = now` |
| **退订 Contact** | `prisma.contact.update({ data: { subscribed: false } })` |
| 追踪事件 `email.bounce` | `EventService.trackEvent('email.bounce', ...)` |
| Ntfy 通知 | `NtfyService.notifyEmailBounce(...)` |
| **安全检查** | `SecurityService.checkAndEnforceSecurityLimits(projectId)` |

**典型原因**：邮箱不存在、域名无效、收件方永久拒绝。

### 2.3 软退（Transient Bounce）

**触发**：SES SNS 推送 `eventType: "Bounce"` 且 `bounceType: "Transient"`

**代码位置**：[Webhooks.ts L397-L408](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L397-L408)

**处理动作**：

| 动作 | 代码 |
|------|------|
| Email 状态 → **不变** | 不更新 `updateData.status` |
| Contact 订阅 → **不变** | 不退订 |
| 追踪事件 `email.bounce` | `EventService.trackEvent('email.bounce', { bounceType, transientBounce: true })` |
| **不触发安全检查** | 软退不计入退信率 |

**典型原因**：邮箱满、收件方 MTA 暂时不可达、灰名单（greylisting）。

> ⚠️ 关键差异：软退**不修改 Email 状态**，也不退订 Contact。它仅以事件形式记录，供日后查询，但不会影响退信率计算和安全阈值。

### 2.4 未知退信类型

**触发**：SES SNS 推送 `eventType: "Bounce"` 但 `bounceType` 既非 Permanent 也非 Transient

**代码位置**：[Webhooks.ts L409-L427](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L409-L427)

**策略**：**当作硬退处理**（安全优先）——同样设 BOUNCED、退订 Contact、触发安全检查。

### 2.5 投诉（Complaint）

**触发**：SES SNS 推送 `eventType: "Complaint"`

**代码位置**：[Webhooks.ts L431-L447](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L431-L447)

**处理动作**：

| 动作 | 代码 |
|------|------|
| Email 状态 → COMPLAINED | `updateData.status = EmailStatus.COMPLAINED` |
| 记录 complainedAt | `updateData.complainedAt = now` |
| **退订 Contact** | `prisma.contact.update({ data: { subscribed: false } })` |
| 追踪事件 `email.complaint` | `EventService.trackEvent('email.complaint', ...)` |
| Ntfy 通知 | `NtfyService.notifyEmailComplaint(...)` |
| **安全检查** | `SecurityService.checkAndEnforceSecurityLimits(projectId)` |

---

## 三、延迟与重试机制

### 3.1 BullMQ 队列重试

定义于 [QueueService.ts L47-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/QueueService.ts#L47-L58)：

```ts
emailQueue = new Queue('email', {
  defaultJobOptions: {
    attempts: 3,            // 最多尝试 3 次
    backoff: {
      type: 'exponential',
      delay: 2000,          // 首次 2s，后续指数增长
    },
  },
});
```

**重试时序**：

| 重试轮次 | 等待时间 | 说明 |
|----------|---------|------|
| 第 1 次 | 立即 | Worker 取 job 执行 |
| 第 2 次 | ~2s | 失败后指数退避 |
| 第 3 次 | ~4s+ | 再次失败后退避 |

**重试触发条件**：[email-processor.ts L266-L278](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L266-L278) 中 catch 块先标 `FAILED`，再 `throw error` 让 BullMQ 重新入队。

> ⚠️ 注意：每次重试时 Worker 先读 DB，检查 `status !== PENDING` 则跳过（L99）。但失败后标的是 `FAILED`，不是 `PENDING`——**所以重试实际上不会再次执行发送**，因为 `status` 已经不是 `PENDING`。这是一个值得关注的点：当前代码中，邮件一旦被标为 FAILED，即使 BullMQ 重新投递 job，Worker 也会直接 return。

### 3.2 发送速率限制

[QueueService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/QueueService.ts#L177-L188) 定义了优先级：事务性邮件 priority=1 > 工作流=5 > 营销=10。

[email-processor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L62-L71) Worker 并发度根据 SES 配额自动推导，限速器 `max: rateLimit, duration: 1000` 确保每秒不超过 SES 配额。

### 3.3 SES 层面的延迟

AWS SES 本身会在收件方 MTA 返回 4xx（临时性拒绝）时自动重试投递，通常持续数小时。SES 不向 SNS 推送 "Delay" 事件类型——Plunk 当前的事件类型枚举为 `'Bounce' | 'Delivery' | 'Open' | 'Complaint' | 'Click'`，不包含延迟。

**结论**：Plunk 体系内没有独立的"延迟"状态。SES 层面的延迟/软退通过 `bounceType: "Transient"` 反馈，Plunk 只是记录事件，不改变状态。

---

## 四、清单清洗策略

### 4.1 自动退订

以下两种情况会自动将 Contact 的 `subscribed` 设为 `false`：

| 场景 | 触发位置 | 后续效果 |
|------|---------|---------|
| 硬退 | [Webhooks.ts L385-L388](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L385-L388) | 后续营销邮件不再发送给该 Contact |
| 投诉 | [Webhooks.ts L436-L439](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L436-L439) | 同上 |

软退**不触发退订**。

### 4.2 发送前过滤

在邮件实际发送前，有多层订阅检查：

1. **EmailService.sendWorkflowEmail** — [L203-L234](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L203-L234)：营销邮件对已退订 Contact 直接创建 `FAILED` 记录（error='Contact is unsubscribed'），不入队
2. **EmailService.sendEmail** — [L311-L326](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L311-L326)：再次检查 `contact.subscribed`，营销邮件跳过
3. **EmailService.sendTransactionalEmail** — [L55-L75](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L55-L75)：营销模板发给退订 Contact 会抛 400 错误

**事务性邮件（TRANSACTIONAL sourceType）不受订阅状态限制**，始终发送。

### 4.3 安全阈值与项目停用

[SecurityService.ts L30-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/SecurityService.ts#L30-L69) 定义了双层阈值体系：

**老项目（>30天）— 仅看比率**：

| 指标 | 警告阈值 | 严重阈值 | 最低数量门槛 |
|------|---------|---------|-------------|
| 7 天退信率 | 5% | 10% | 5/10 次 |
| 全局退信率 | 4% | 8% | 5/10 次 |
| 7 天投诉率 | 0.075% | 0.15% | 3/5 次 |
| 全局投诉率 | 0.03% | 0.12% | 3/5 次 |

> 仅硬退计入退信率。软退和 transiventBounce 不参与计算。

**新项目（≤30天）— 额外看绝对数量天花板**：

| 指标 | 警告天花板 | 严重天花板 |
|------|-----------|-----------|
| 24h 退信数 | 10 | 25 |
| 7d 退信数 | 25 | 50 |
| 24h 投诉数 | 3 | 7 |
| 7d 投诉数 | 10 | 20 |

**触发时机**：仅硬退和投诉事件后调用 [checkAndEnforceSecurityLimits](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/SecurityService.ts#L272-L332)（[Webhooks.ts L465-L468](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L465-L468)）。

**严重后果**：`shouldDisable === true` 且 `AUTO_PROJECT_DISABLE` 开启时，项目被设 `disabled: true, disabledReason: 'EMAIL_REPUTATION'`，同时：
- 取消该项目所有队列中的 job
- 将所有 PENDING 邮件标为 FAILED
- 终结所有 SENDING 状态的 Campaign

---

## 五、完整状态流转图

```
                              ┌──────────────────────┐
                              │     EmailService      │
                              │  创建邮件 PENDING      │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │  BullMQ emailQueue    │
                              │  3次重试, 指数退避      │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │  email-processor      │
                              │  项目disabled?        │───YES──→ FAILED
                              │  status≠PENDING?      │───YES──→ 跳过(return)
                              └──────────┬───────────┘
                                         │ NO
                                         ▼
                              ┌──────────────────────┐
                              │  SENDING             │
                              │  SES sendRawEmail()  │
                              └──────────┬───────────┘
                                   ┌─────┴─────┐
                                   │           │
                                成功          失败
                                   │           │
                                   ▼           ▼
                              ┌─────────┐  ┌────────────────┐
                              │  SENT   │  │  FAILED        │
                              │ +msgId  │  │  +error        │
                              └────┬────┘  │  throw→BullMQ  │
                                   │       │  重试(但status  │
                                   │       │  已非PENDING,   │
                                   │       │  实际不会重发)   │
                                   │       └────────────────┘
                                   │
                        ┌──────────┼──────────────────────┐
                        │          │                      │
                   SES SNS     SES SNS               SES SNS
                   Delivery    Bounce               Complaint
                        │          │                      │
                        ▼     ┌────┴────┐                 ▼
                   ┌─────────┐│         │          ┌───────────┐
                   │DELIVERED││         │          │ COMPLAINED│
                   └────┬────┘│         │          │ 退订Contact│
                        │     │         │          │ 安全检查   │
                   SES SNS   │         │          └───────────┘
                   Open      │         │
                        │    ▼         ▼
                   ┌─────────┐  ┌──────────┐  ┌───────────────────┐
                   │ OPENED  │  │ BOUNCED  │  │ (不更新状态)        │
                   └────┬────┘  │ 退订Contact│ │ 仅记录事件          │
                        │       │ 安全检查   │ │ bounceType=Transient│
                   SES SNS     └──────────┘  │ 不退订Contact       │
                   Click                    │ 不触发安全检查        │
                        │                   └───────────────────────┘
                   ┌──────────┐
                   │ CLICKED  │              未知bounceType
                   └──────────┘              → 当作硬退处理
```

---

## 六、关键发现与注意点

1. **软退不留痕**：软退（Transient Bounce）不改变 Email.status，不退订 Contact，不计入退信率。仅以 `email.bounce` 事件（含 `transientBounce: true`）留存记录。如果需要"软退 N 次后退订"策略，当前代码不支持。

2. **重试实质失效**：BullMQ 配置了 3 次重试，但 email-processor 在 catch 块中先将 Email 标为 FAILED 再 throw——下次 Worker 取到 job 后发现 `status !== PENDING` 直接 return。**等于说重试机制形同虚设**，第一次失败后不会再实际重发。

3. **两条退信处理路径**：
   - **主路径**：SNS Webhook → [Webhooks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts) — 区分硬退/软退，精细处理
   - **遗留路径**：[EmailService.handleWebhookEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L462-L584) — 不区分软硬退，一律退订。此方法目前未被实际调用（Webhook 控制器直接操作 DB），但代码仍存在

4. **安全检查仅限硬退+投诉**：[Webhooks.ts L465-L468](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L465-L468) 的 `isPermanentBounce` 条件确保软退不触发安全阈值检查，不会误杀项目。

5. **Campaign 终结条件**：[CampaignService.finalizeIfDone](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/CampaignService.ts#L617-L663) 在每封邮件 SENT/FAILED 后检查是否所有邮件都已到达终态（`sentAt != null || status === FAILED`），满足则将 Campaign 从 SENDING 改为 SENT。退信（BOUNCED）不阻碍 Campaign 终结——因为 BOUNCED 的邮件必然已经 SENT 过。

6. **SES 没有 Delay 事件**：AWS SES 的事件通知类型为 Send/Delivery/Bounce/Complaint/Open/Click/Rendering Failure，没有"延迟"类型。SES 在 MTA 返回 4xx 时自行重试，最终要么 Delivery 要么 Bounce（此时 bounceType=Transient）。
