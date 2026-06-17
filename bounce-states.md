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

## 四、三条发送路径与队列关系

Plunk 有三条独立的邮件发送入口，**全部汇聚到同一个 BullMQ `emailQueue`**，由同一个 [email-processor](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L73-L304) Worker 消费。区别在于入口层的订阅检查和受众筛选逻辑不同。

### 4.1 事务性邮件（API /v1/send）

**入口**：[Actions.send](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Actions.ts#L172-L348) → `EmailService.sendTransactionalEmail`

**受众**：调用方在请求体中指定收件人，不经过任何受众筛选。

**订阅检查（仅限入口层，Worker 不兜底）**：
- 使用营销模板时（[L55-L74](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L55-L74)）：`template.type === 'MARKETING'` 且 `!contact.subscribed` → 抛 400 错误
- 事务性模板或无模板 → **完全不检查订阅**，退订 Contact 也会正常发送（事务性邮件设计上不受退订限制）
- **没有 Worker 层兜底**：email-processor 不检查 `contact.subscribed`，入口层是事务性邮件的唯一订阅拦截点

**事务性 API 的退订相关行为**：
- 新收件人通过 `ContactService.upsert(..., defaultSubscribed=false)` 创建 → 默认 `subscribed=false`（[Actions.ts L274](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Actions.ts#L274)）
- 已有收件人的订阅状态被保留（除非请求体显式传 `subscribed`）
- 退订的收件人仍能收到事务性邮件（非营销模板时）——这是**故意的设计**，用于验证码、订单通知、密码重置等场景

**sourceType**：`EmailSourceType.TRANSACTIONAL`

### 4.2 营销活动（Campaign）

**入口**：[CampaignService.send](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/CampaignService.ts#L381-L454) → `startSending` → `processBatch`

**完整发送链路**：

```
CampaignService.send()
  ├─ scheduledFor? → QueueService.scheduleCampaign() → scheduledQueue → scheduled-processor → startSending()
  └─ 立即发送 → startSending()
                    └─ QueueService.queueCampaignBatch() → campaignQueue → campaign-processor
                         └─ CampaignService.processBatch()  ← 逐批处理
                              ├─ getRecipientsCursor()      ← 查 DB 筛选受众
                              │    └─ buildRecipientWhereAsync() ← 构造 WHERE 子句
                              ├─ 对每个 Contact: EmailService.sendCampaignEmail()
                              │    └─ 创建 Email 记录(PENDING) → QueueService.queueEmail() → emailQueue
                              └─ 有更多批次? → queueCampaignBatch(下一批)
                                  无更多批次? → finalizeIfDone()
```

**受众筛选（活动路径的核心拦截机制）**（[buildRecipientWhereAsync](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/CampaignService.ts#L866-L906)）：

```ts
const baseWhere = {
  projectId,
  // 营销类活动只选 subscribed: true 的 Contact
  ...(campaign.type !== TemplateType.TRANSACTIONAL && { subscribed: true }),
};
```

三种受众模式在此基础上叠加：

| audienceType | 筛选方式 |
|---|---|
| `ALL` | 仅 `baseWhere`，即项目内所有已订阅 Contact |
| `SEGMENT` | `baseWhere` + 静态分段的 `segmentMemberships`（exitedAt=null）/ 动态分段的 `SegmentService.buildConditionClause` |
| `FILTERED` | `baseWhere` + `SegmentService.buildConditionClause(condition)` |

**关键**：**受众查询在 processBatch 时实时执行**，不是在活动创建时快照。这意味着：
- 活动发送期间如果某个 Contact 因硬退被退订，**后续批次查询时自动排除该 Contact**
- 动态 Segment 的条件在每批查询时重新求值，联系人离开 Segment 后不会收到后续批次的邮件

**退订后的拦截（活动路径）**：

| 拦截层 | 位置 | 行为 |
|---|---|---|
| 受众查询 | [buildRecipientWhereAsync](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/CampaignService.ts#L870-L874) `subscribed: true` | 退订 Contact 不出现在受众列表中，**根本不会为其创建 Email 记录** |
| Email 创建 | `sendCampaignEmail` 无额外订阅检查 | — （已被受众查询过滤掉） |
| Worker 层 | [email-processor](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L73-L304) | 不检查 `contact.subscribed`（见 4.4） |

**事务性活动（campaign.type=TRANSACTIONAL）**：不附加 `subscribed: true` 条件，对所有 Contact 发送，不管订阅状态如何。

**sourceType**：营销活动默认 `CAMPAIGN`；若 `isTransactional=true` 或模板类型为 `TRANSACTIONAL` 则为 `TRANSACTIONAL`

### 4.3 工作流邮件（Workflow）

**入口**：[WorkflowExecutionService.executeSendEmail](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/WorkflowExecutionService.ts#L526-L593) → `EmailService.sendWorkflowEmail`

**受众**：工作流执行绑定的 Contact（由触发事件决定），不走受众筛选查询。

**订阅检查（入口层拦截，Worker 不兜底）**（[L200-L234](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L200-L234)）：
- 营销工作流（sourceType ≠ TRANSACTIONAL）且无自定义收件人 → 查 `contact.subscribed`
- `!contact.subscribed` → **静默跳过**：创建 `FAILED` 记录（error='Contact is unsubscribed from marketing emails'），**不入队**，工作流继续执行后续步骤
- 事务性工作流（sourceType=TRANSACTIONAL，由模板类型决定） → 不检查订阅
- 自定义收件人（`recipientEmail`） → 不检查订阅

**sourceType**：默认 `WORKFLOW`；模板类型为 `TRANSACTIONAL` 时为 `TRANSACTIONAL`

### 4.4 Worker 层的实际行为——无订阅兜底

所有三条路径最终都汇入 [email-processor](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L73-L304)。Worker 在实际调 SES 前做的检查是：

| 检查项 | 代码位置 | 说明 |
|---|---|---|
| `status !== PENDING` | [L99-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L99-L101) | 幂等保护，跳过已处理邮件 |
| `project.disabled` | [L103-L119](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L103-L119) | 项目禁用则标 FAILED |
| 钓鱼内容检测 | [L185-L212](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L185-L212) | 危险内容则禁项目 + 标 FAILED |
| **contact.subscribed** | **不存在** | **Worker 完全不检查订阅状态** |

> ⚠️ 关于 `EmailService.sendEmail` 方法：
> - 此方法在 [L290-L456](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L290-L456) 有完整的发送逻辑（含 L311-L326 的订阅检查）
> - 但 grep 全局搜索确认：**生产代码中没有任何地方调用它**，仅在 `__tests__/EmailService.test.ts` 中使用
> - 生产发送路径是 Worker 内联的发送逻辑 + 入口 Service 层的订阅检查
> - 结论：`EmailService.sendEmail` 中的订阅检查是**死代码**，对实际行为无影响

**三层防护的实际边界**：

| 路径 | 订阅检查发生在哪一层 | 是否可能有漏网之鱼 |
|---|---|---|
| 事务性（营销模板） | `sendTransactionalEmail` L55-74 | 不会——抛 400 阻断 |
| 事务性（事务性/无模板） | **完全不检查**（故意设计） | N/A——本就不应该拦 |
| 活动（营销） | `buildRecipientWhereAsync` subscribed=true | 不会——根本不创建 Email |
| 活动（事务性） | **完全不检查**（故意设计） | N/A——本就不应该拦 |
| 工作流（营销） | `sendWorkflowEmail` L200-234 | 不会——创建 FAILED 不入队 |
| 工作流（事务性） | **完全不检查**（故意设计） | N/A——本就不应该拦 |
| 所有路径的 Worker | **不检查** subscribed | 仅当绕过所有 Service 层直接写 DB 时才可能 |

### 4.5 三条路径的订阅拦截汇总

| 路径 | 拦截时机 | 拦截方式 | 退订 Contact 的结果 |
|---|---|---|---|
| 事务性 (API) + 营销模板 | 入口 Service | `sendTransactionalEmail` L55-74 | 抛 400 错误，不创建记录 |
| 事务性 (API) + 事务性/无模板 | 不拦截 | — | 正常发送（设计上允许） |
| 活动（营销类） | 受众查询 | `buildRecipientWhereAsync` subscribed=true | 不创建 Email 记录 |
| 活动（事务性类） | 不拦截 | — | 正常发送（设计上允许） |
| 工作流 + 营销模板 | 入口 Service | `sendWorkflowEmail` L200-234 | 创建 FAILED 记录，不入队 |
| 工作流 + 事务性模板 | 不拦截 | — | 正常发送（设计上允许） |
| Worker 层（所有路径） | — | **不检查** `subscribed` | — |

---

## 五、订阅状态（subscribed）写入入口全集

Contact.subscribed 字段是清单清洗机制的基石。以下是所有能修改该字段的入口，按调用路径分类：

### 5.1 SNS Webhook（系统自动退订）

这是硬退/投诉触发的自动退订入口，由 SES SNS 回调驱动。

| 触发事件 | 代码位置 | 写入动作 |
|---|---|---|
| 硬退 Permanent Bounce | [Webhooks.ts L385-L388](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L385-L388) | `subscribed = false` |
| 未知类型 Bounce | [Webhooks.ts L421-L424](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L421-L424) | `subscribed = false`（当作硬退处理） |
| 投诉 Complaint | [Webhooks.ts L436-L439](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L436-L439) | `subscribed = false` |
| 软退 Transient Bounce | — | **不修改**（仅记录事件） |

**特点**：所有写操作都是单向的——只会设 `false`，永远不会在此处重新设 `true`。

### 5.2 单个联系人的增删改（ContactService 基础方法）

这些是 REST API 单条操作入口，最终都走 [ContactService](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/ContactService.ts)。

| 方法 | 入口端点 | 订阅写入规则 | 状态变更事件 |
|---|---|---|---|
| [create](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/ContactService.ts#L124-L144) | `POST /contacts` | `subscribed ?? true`，不传默认 **订阅** | 不追踪（新建无变更） |
| [update](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/ContactService.ts#L191-L245) | `PATCH /contacts/:id` | 请求体中传 `subscribed` 才更新，不传保留原值 | 变化时追踪 `contact.subscribed` / `contact.unsubscribed` |
| [upsert](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/ContactService.ts#L273-L339) | `POST /contacts`、`POST /v1/send`、`POST /v1/track` | 新建时 `subscribed ?? defaultSubscribed`（默认 true）；更新时仅在显式传 `subscribed` 时才改 | 变化时追踪 |
| [subscribe](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/ContactService.ts#L427-L437) | `POST /contacts/public/:id/subscribe` | 强制 `true`（公开页面，用户点击订阅链接） | 追踪 `contact.subscribed` |
| [unsubscribe](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/ContactService.ts#L442-L452) | `POST /contacts/public/:id/unsubscribe`、邮件退订链接 | 强制 `false`（公开页面，用户点击退订链接） | 追踪 `contact.unsubscribed` |

**注意 `upsert` 的 `defaultSubscribed` 参数默认值差异**：

| 调用方 | `defaultSubscribed` 值 | 含义 |
|---|---|---|
| `POST /contacts`（Contacts 控制器） | 默认 `true` | 通过 Dashboard 手动添加默认订阅 |
| `POST /v1/send`（事务性 API，[Actions.ts L274](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Actions.ts#L274)） | 显式传 `false` | 新收件人默认**不订阅**（符合反垃圾邮件要求） |
| `POST /v1/track`（事件追踪，[Actions.track L77-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Actions.ts#L77-L82)） | 默认 `true` | 追踪事件创建的新联系人默认订阅 |

### 5.3 批量订阅/退订（bulk 操作队列）

入口端点：`POST /contacts/bulk-subscribe`、`POST /contacts/bulk-unsubscribe`，由 [Contacts.ts L435-L451](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Contacts.ts#L435-L451) 接收，通过 `queueBulkContactAction` 进入 BullMQ 的 `bulkContactQueue`，由 [bulk-contact-processor](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/bulk-contact-processor.ts#L62-L190) Worker 处理。

**两种匹配模式**（BATCH_SIZE=100）：

| selector.mode | 匹配方式 | 实现 |
|---|---|---|
| `ids` | 直接传 `contactIds` 数组 | 按 id 分批调用 Service 方法 |
| `query` | 基于 `filter.search / filter.subscribed / excludeIds` 构建 WHERE 子句 | 先 count 再用游标 `id < lastId` 倒序遍历（对 delete 安全，避免删除后游标跳跃） |

实际修改委托给 [ContactService.bulkSubscribe](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/ContactService.ts#L726-L758) / [bulkUnsubscribe](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/ContactService.ts#L765-L797)：
- 先查询每个 Contact 的当前状态，只对**确实需要翻转**的 id 调用 `updateMany`
- 返回 `{updated, unchanged}`，已在目标状态的联系人被当作"无操作"不计入失败
- 成功翻转的联系人逐个追踪 `contact.subscribed` / `contact.unsubscribed` 事件（异步串行，避免 DB 死锁）

### 5.4 CSV 导入任务

入口端点：`POST /contacts/import`，[Contacts.ts L316-L344](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Contacts.ts#L316-L344) 接收 CSV 文件后转 base64 入 `importQueue`，由 [import-processor](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/import-processor.ts#L27-L214) Worker 处理。

**CSV 中订阅状态的处理规则**（[import-processor L114-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/import-processor.ts#L114-L122)）：

```
CSV 的 subscribed 列（大小写不敏感）
  ├─ 空 / 不存在      → subscribed = undefined
  │     ├─ 新建 Contact → 默认 subscribed=true（由 ContactService.upsert defaultSubscribed）
  │     └─ 已有 Contact → 保持原有值不变
  ├─ "true" / "1" / "yes" → subscribed = true
  └─ 其他值             → subscribed = false
```

**注意**：
- 数字 `0` 和 `1` 不被识别（代码里只判断了字符串 "1"），数字形式会被当作 `false`
- 已有 Contact 如果 CSV 没传 subscribed 列，**不会**被误改订阅状态
- 每行走 `ContactService.upsert`，更新时会触发订阅状态变更事件

### 5.5 工作流 UPDATE_CONTACT 步骤

工作流中的 `UPDATE_CONTACT` 类型步骤可在运行时修改联系人订阅状态。

代码位置：[WorkflowExecutionService.executeUpdateContact](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1033-L1085)

```
config.subscriptionAction
  ├─ 'subscribe'   → subscribed = true（仅当原值为 false 时更新）
  ├─ 'unsubscribe' → subscribed = false（仅当原值为 true 时更新）
  └─ undefined     → 不修改订阅状态
```

同时还可通过 `config.updates` 更新 data 自定义字段。**订阅变更只在实际翻转时才写 DB 并追踪事件**（`subscriptionChanging` 判断），不做无意义的同值更新。

这是工作流中唯一一个能主动让联系人重新订阅的入口（硬退/投诉只能退订）。典型场景：联系人在工作流中点击"确认订阅"按钮 → 触发 UPDATE_CONTACT subscribe → 后续营销类步骤可正常发送。

### 5.6 事件追踪 API（/v1/track）

入口：[Actions.track](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Actions.ts#L52-L92)（`POST /v1/track`）

订阅状态来源是请求体的 `subscribed` 字段，通过 `ContactService.upsert(auth.projectId, email, data, subscribed)` 写入。

- **显式传 `subscribed: true/false`** → 按值写入，有变化则追踪事件
- **不传** → 新建时默认订阅（`defaultSubscribed=true`），已有联系人保持不变

这是最灵活的外部入口，可被集成系统用来同步外部系统的订阅偏好。

### 5.7 写入入口汇总

```
                          subscribed 字段写入点
  ┌──────────────────────────────────────────────────────────┐
  │ 系统自动（单向退订）                                       │
  │   ├─ 硬退 Permanent Bounce → Webhooks → subscribed=false │
  │   ├─ 未知类型 Bounce       → Webhooks → subscribed=false │
  │   └─ 投诉 Complaint        → Webhooks → subscribed=false │
  ├──────────────────────────────────────────────────────────┤
  │ 用户自助（通过链接）                                        │
  │   ├─ POST /contacts/public/:id/subscribe   → true        │
  │   └─ POST /contacts/public/:id/unsubscribe → false       │
  ├──────────────────────────────────────────────────────────┤
  │ API / 操作面板（单条）                                      │
  │   ├─ ContactService.create      → ?? true                │
  │   ├─ ContactService.update      → 请求体决定              │
  │   ├─ ContactService.upsert      → 请求体 + defaultSub    │
  │   └─ POST /v1/track (Actions)   → 请求体决定              │
  ├──────────────────────────────────────────────────────────┤
  │ 批量操作（队列异步）                                        │
  │   ├─ bulk-subscribe   → bulk-contact-processor → true   │
  │   ├─ bulk-unsubscribe → bulk-contact-processor → false  │
  │   └─ CSV import       → import-processor → 按列解析      │
  ├──────────────────────────────────────────────────────────┤
  │ 工作流步骤                                                 │
  │   └─ UPDATE_CONTACT step → subscriptionAction 决定       │
  └──────────────────────────────────────────────────────────┘
```

---

## 六、硬退/投诉对四类发送场景的完整影响分支

硬退/投诉造成的核心变化是：`contact.subscribed = false`  + （若超限则）`project.disabled = true`。下面按四类发送场景逐一展开影响分支。

### 6.1 营销活动（Campaign，非事务性）

假设某 Contact 在活动开始时处于订阅状态，在第 1 批发出邮件后立即因硬退被退订。完整的影响链路：

```
第 1 批 processBatch
  ├─ buildRecipientWhereAsync → subscribed=true → Contact 命中
  ├─ sendCampaignEmail → 创建 Email(PENDING) → 入 emailQueue
  └─ Worker 发送 → SES 接受 → SENT → SNS Delivery 推送 → DELIVERED
       └─ 但收件方 MTA 返回硬退 → SNS 推送 Bounce(Permanent)
            ├─ Webhooks: Email.status → BOUNCED
            ├─ Webhooks: Contact.subscribed → false   ★ 关键分叉点
            └─ Webhooks: 安全阈值检查
                 └─ 超限? → YES → project.disabled=true
                           └─ NO  → 项目继续运行

第 2 批 processBatch（稍后执行）
  ├─ buildRecipientWhereAsync
  │    └─ subscribed=true 条件 + Contact 已退订
  │         └─ Contact 不在结果集中 → **不会为其创建 Email 记录**
  └─ 继续处理其他联系人

第 N 批（如果项目被禁用）
  └─ 所有已入队的 PENDING 邮件被 SecurityService 批量标 FAILED
```

| 影响项 | 未超限 | 超限（项目被禁） |
|---|---|---|
| 后续批次 | 自动排除退订 Contact，其他联系人流畅发送 | 所有 PENDING 邮件 FAILED，取消队列 job，Campaign 终止 |
| 本批次已发出的邮件 | 无法撤回，该 Contact 仍收到本批次内容 | 同左 |
| 活动统计 | BOUNCED 计入 bounceRate；正常邮件按 SENT 统计 | 活动状态变为 FAILED（SecurityService 终止 Campaign） |
| 退信率 | BOUNCED 会被 SecurityService 纳入 7 天/全局计算 | 同上，但项目已被冻结 |

### 6.2 营销工作流（Workflow，非事务性模板）

工作流的执行是连续步骤链式推进，每个 SEND_EMAIL 步骤单独检查订阅状态。假设某 Contact 在工作流的步骤 1（确认邮件）后因回复硬退被退订：

```
工作流执行链：
  Step 1: SEND_EMAIL (营销模板)
    ├─ sendWorkflowEmail → sourceType=WORKFLOW, 无recipientEmail
    ├─ 检查 subscribed=true → ✅ 通过
    ├─ 创建 Email(PENDING) → 入队 → 发送 → SENT
    └─ 硬退 SNS 推送 → Contact.subscribed=false ★

  Step 2: CONDITION (例：3 天后检查是否打开)
    └─ 不涉及订阅 → 正常评估条件

  Step 3: SEND_EMAIL (营销模板, 跟进邮件)
    ├─ sendWorkflowEmail → 检查 subscribed
    │   └─ subscribed=false → ❌ 拦截
    │        ├─ 创建 Email(FAILED, error='Contact is unsubscribed from marketing emails')
    │        ├─ 不入 emailQueue，Worker 不会处理
    │        └─ 步骤状态 → completed（继续执行，不中断流程）
    └─ 工作流流转到 Step 4

  Step 4: SEND_EMAIL (事务性模板, 如账户通知)
    ├─ sourceType=TRANSACTIONAL → 不检查订阅
    ├─ 创建 Email(PENDING) → 入队 → 发送
    └─ ✅ 即使退订也正常送达

  Step 5: UPDATE_CONTACT (subscriptionAction='subscribe', 如用户点击确认)
    └─ subscribed 被重新设为 true → 后续营销步骤恢复可发送
```

| 影响项 | 结果 |
|---|---|
| SEND_EMAIL（营销） | 静默跳过，创建 FAILED，工作流不中断 |
| SEND_EMAIL（事务性） | 照常发送，不检查订阅 |
| UPDATE_CONTACT | 可重新订阅，解除后续拦截 |
| 自定义收件人（`recipientEmail`） | **完全绕过 Contact 订阅检查**（见 6.4） |
| 工作流执行本身 | 订阅拦截不导致工作流中止或失败 |

### 6.3 事务性邮件（API /v1/send）

事务性邮件没有"营销活动/工作流"的批量上下文，每封邮件独立。硬退/投诉的影响取决于后续是否继续调用 `/v1/send`。

**事务性 API 的订阅分支**（[EmailService.sendTransactionalEmail](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L31-L105)）：

```
POST /v1/send
  └─ 选择模板
       ├─ 无模板 或 模板.type=TRANSACTIONAL
       │    └─ 完全不检查 subscribed
       │         ├─ 创建 Email(PENDING) → 入队 → 发送
       │         └─ 即使退订也会发送（故意设计：验证码、订单通知等必须送达）
       │
       └─ 模板.type=MARKETING
            └─ 检查 subscribed
                 ├─ subscribed=true → ✅ 发送
                 └─ subscribed=false → ❌ 抛 400 "This email is unsubscribed."
                      ├─ 不创建 Email 记录
                      └─ 不入队，直接返回错误
```

**新收件人特殊分支**（[Actions.ts L274](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Actions.ts#L274)）：
- `/v1/send` 调用 `ContactService.upsert(..., defaultSubscribed=false)`，**新收件人默认退订**
- 但如果传的是事务性模板 → 即使默认退订也发送（因为不检查订阅）
- 第一次调用会创建一个 `subscribed=false` 的 Contact，后续用营销模板调用会被拒
- 如果需要新收件人可接收营销邮件，必须先通过 `/v1/track` 或 `/contacts` API 显式传 `subscribed: true`

| 场景 | 是否检查订阅 | 退订后行为 |
|---|---|---|
| 无模板事务性邮件 | 否 | 正常发送 |
| 事务性模板 | 否 | 正常发送 |
| 营销模板 | 是 | 返回 400，不发 |
| 项目被安全机制禁用（不管什么模板） | Worker 层检查 | 创建了 PENDING 也会被 Worker 标 FAILED |

### 6.4 自定义收件人发送（recipientEmail 覆盖）

这是工作流的一个特殊功能：在 SEND_EMAIL 步骤配置中通过 `recipientEmail` 字段指定一个**不同于 Contact.email** 的收件地址（例如发送到联系人的备用邮箱、或 CC 某个审批人）。

**实现机制**（[EmailService.sendWorkflowEmail](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L198-L206) + [EmailService.sendRawEmail](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L380-L387)）：

```
sendWorkflowEmail
  └─ params.recipientEmail 存在？
       ├─ YES → 跳过 subscribed 检查（因为收件人可能不在 Contact 列表中）
       │    ├─ 创建 Email 记录，contactId 仍绑定到原 Contact（用于审计/统计）
       │    ├─ recipientEmail 写入 headers['X-Plunk-Recipient-Override']
       │    └─ 入 emailQueue
       │
       └─ NO  → 正常检查 subscribed（见 6.2）

Worker email-processor 实际调 SES 时：
  ├─ 读取 headers['X-Plunk-Recipient-Override']
  ├─ 若存在 → to: [recipientEmail]（覆盖 contact.email）
  ├─ 发送前删除该内部 header，不暴露给收件人
  └─ 若不存在 → to: [contact.email]
```

**硬退/投诉对自定义收件人的影响**：

| 影响维度 | 结果 |
|---|---|
| 订阅检查 | `recipientEmail` 存在时**完全绕过** `contact.subscribed` 检查。即使 Contact 已因硬退退订，后续使用自定义收件人的步骤仍会发送 |
| 退信归因 | SNS Bounce 事件通过 SES messageId 关联回 Email 记录 → 关联到原 contactId → **原 Contact 被退订**。自定义收件人邮箱本身不在 Contact 表中，退信惩罚落到 Contact 身上 |
| 自定义收件人自身硬退 | 该自定义邮箱下次再被使用时**仍可发送**——因为它没有独立的订阅/退信状态，不受影响 |
| 退信率计算 | 退信仍计入项目总体退信率（因为 `projectId` 是关联的），可能触发安全阈值 |

**风险提示**：自定义收件人功能允许向非 Contact 列表中的地址发送。如果滥用此功能（频繁向第三方地址发送营销内容且被退信），虽然不会退订任何 Contact，但会累积项目退信率，严重时会被安全机制禁用项目。

---

## 七、清单清洗策略

### 7.1 退订触发条件

以下两种 SNS 事件会自动将 Contact 的 `subscribed` 设为 `false`：

| 场景 | 触发位置 | 退订后效果 |
|---|---|---|
| 硬退 (Permanent Bounce) | [Webhooks.ts L385-L388](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L385-L388) | 后续营销活动和工作流不再发送 |
| 投诉 (Complaint) | [Webhooks.ts L436-L439](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L436-L439) | 同上 |
| 软退 (Transient Bounce) | — | **不退订** |
| 未知退信类型 | [Webhooks.ts L409-L427](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L409-L427) | **当作硬退 → 退订** |

### 7.2 退订后对进行中活动的影响

一个营销活动在发送过程中（状态 SENDING），按批（BATCH_SIZE=500）处理联系人。**每批的受众查询都是实时从 DB 读取 `subscribed: true` 的 Contact**。

这意味着：如果第 1 批处理时 Contact A 收到了邮件，随后因硬退被退订（`subscribed → false`），那么到第 2 批查询时 Contact A 已经不在受众列表中——**退订是即时生效的**，不会等到活动结束。

但已经发出的邮件不可撤回，该 Contact 仍会在 SES 层面收到那封邮件。

### 7.3 退订后对进行中工作流的影响

工作流的 SEND_EMAIL 步骤在执行时调用 `sendWorkflowEmail`，此时才检查 `contact.subscribed`。如果 Contact 在工作流执行中途被退订：
- 下一个 SEND_EMAIL 步骤检查到 `!subscribed` → 创建 FAILED 记录（error='Contact is unsubscribed from marketing emails'），**工作流不中断**，继续执行后续步骤（如 DELAY、CONDITION 等）
- 只有事务性工作流（模板类型为 TRANSACTIONAL）会忽略退订状态继续发送

### 7.4 退订的重新订阅路径

硬退/投诉只能将 `subscribed` 设为 `false`，不能自动恢复。**所有能将 subscribed 重新设为 `true` 的入口**：

| 入口 | 方式 |
|---|---|
| 用户点击邮件中的订阅链接 | `POST /contacts/public/:id/subscribe` → [ContactService.subscribe](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/ContactService.ts#L427-L437) 强制 `true` |
| 手动调用 REST API | `PATCH /contacts/:id` 传 `{"subscribed": true}` 或 `POST /contacts` 传 `subscribed=true` upsert |
| 事件追踪 API | `POST /v1/track` 请求体传 `subscribed: true` |
| 工作流步骤 | `UPDATE_CONTACT` 步骤的 `subscriptionAction='subscribe'` |
| 批量操作 | `POST /contacts/bulk-subscribe`（ids 或 query 模式） |
| CSV 导入 | CSV 中 `subscribed` 列传 "true"/"1"/"yes" |

### 7.5 安全阈值与项目停用

[SecurityService.ts L30-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/SecurityService.ts#L30-L69) 定义了双层阈值体系：

**老项目（>30天）— 仅看比率**：

| 指标 | 警告阈值 | 严重阈值 | 最低数量门槛 |
|---|---|---|---|
| 7 天退信率 | 5% | 10% | 5/10 次 |
| 全局退信率 | 4% | 8% | 5/10 次 |
| 7 天投诉率 | 0.075% | 0.15% | 3/5 次 |
| 全局投诉率 | 0.03% | 0.12% | 3/5 次 |

> 仅硬退计入退信率。软退和 transientBounce 不参与计算。

**新项目（≤30天）— 额外看绝对数量天花板**：

| 指标 | 警告天花板 | 严重天花板 |
|---|---|---|
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

## 八、完整状态流转图

```
  三条发送入口                              全部汇入
  ─────────────                            ────────

  ┌─────────────────┐
  │ API /v1/send     │
  │ (事务性邮件)      │
  │ sendTransactional│
  └───────┬─────────┘
          │ 营销模板→检查subscribed
          │ 事务性模板→不检查
          ▼
  ┌─────────────────┐                    ┌──────────────────┐
  │ CampaignService  │                    │  BullMQ emailQueue│
  │ .processBatch()  │                    │  3次重试,指数退避   │
  │ ┌───────────────┐│                    └────────┬─────────┘
  │ │受众查询:       ││                             │
  │ │subscribed=true ││                             ▼
  │ │+ SEGMENT/FILTER││                    ┌──────────────────┐
  │ └───────┬───────┘│                    │  email-processor   │
  │ │sendCampaignEmail│                    │  status≠PENDING?  │──YES→ return
  │ │创建Email(PENDING)│──────────────────→│  项目disabled?    │──YES→ FAILED
  └───────┴─────────┘                     │  钓鱼检查?        │──YES→ FAILED+禁项目
                                          └────────┬─────────┘
  ┌─────────────────┐                              │
  │ WorkflowExecution│                              ▼
  │ .executeSendEmail│                    ┌──────────────────┐
  │ sendWorkflowEmail│                    │  SENDING          │
  │ ┌───────────────┐│                    │  SES sendRawEmail │
  │ │营销→检查sub   ││                    └────────┬─────────┘
  │ │事务性→不检查   ││                         ┌────┴────┐
  │ │自定义收件→不查 ││                      成功        失败
  │ │退订→FAILED不入队│                        │          │
  │ └───────┬───────┘│                         ▼          ▼
  └─────────┴─────────┘                    ┌────────┐  ┌──────────────┐
          │                                │  SENT  │  │  FAILED      │
          └───────────────────────────────→│ +msgId │  │  +error      │
                                          └───┬────┘  │  throw→BullMQ│
                                              │       │  重试(但status│
                                              │       │  已非PENDING, │
                                              │       │  实际不会重发)│
                                              │       └──────────────┘
                                              │
                         ┌────────────────────┼──────────────────┐
                         │                    │                  │
                    SES SNS              SES SNS             SES SNS
                    Delivery              Bounce            Complaint
                         │                    │                  │
                         ▼               ┌────┴────┐            ▼
                    ┌──────────┐          │         │     ┌───────────┐
                    │ DELIVERED│          │         │     │ COMPLAINED│
                    └────┬─────┘          │         │     │ 退订Contact│
                         │                ▼         ▼     │ 安全检查   │
                    SES SNS       ┌──────────┐  ┌──────────────┐  └───────────┘
                    Open          │ BOUNCED  │  │(不更新状态)    │
                         │        │ 退订Contact│  │仅记录事件     │
                         ▼        │ 安全检查  │  │bounceType=   │
                    ┌──────────┐  └──────────┘  │Transient     │
                    │ OPENED   │                │不退订Contact  │
                    └────┬─────┘                │不触发安全检查  │
                         │                      └──────────────┘
                    SES SNS
                    Click                 未知bounceType → 当作硬退
                         │
                         ▼
                    ┌──────────┐
                    │ CLICKED  │
                    └──────────┘
```

---

## 九、关键发现与注意点

1. **软退不留痕**：软退（Transient Bounce）不改变 Email.status，不退订 Contact，不计入退信率。仅以 `email.bounce` 事件（含 `transientBounce: true`）留存记录。如果需要"软退 N 次后退订"策略，当前代码不支持。

2. **重试实质失效**：BullMQ 配置了 3 次重试，但 email-processor 在 catch 块中先将 Email 标为 FAILED 再 throw——下次 Worker 取到 job 后发现 `status !== PENDING` 直接 return。**等于说重试机制形同虚设**，第一次失败后不会再实际重发。

3. **两条退信处理路径**：
   - **主路径**：SNS Webhook → [Webhooks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts) — 区分硬退/软退，精细处理
   - **遗留路径**：[EmailService.handleWebhookEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L462-L584) — 不区分软硬退，一律退订。此方法目前未被实际调用（Webhook 控制器直接操作 DB），但代码仍存在

4. **安全检查仅限硬退+投诉**：[Webhooks.ts L465-L468](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Webhooks.ts#L465-L468) 的 `isPermanentBounce` 条件确保软退不触发安全阈值检查，不会误杀项目。

5. **Campaign 终结条件**：[CampaignService.finalizeIfDone](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/CampaignService.ts#L617-L663) 在每封邮件 SENT/FAILED 后检查是否所有邮件都已到达终态（`sentAt != null || status === FAILED`），满足则将 Campaign 从 SENDING 改为 SENT。退信（BOUNCED）不阻碍 Campaign 终结——因为 BOUNCED 的邮件必然已经 SENT 过。

6. **SES 没有 Delay 事件**：AWS SES 的事件通知类型为 Send/Delivery/Bounce/Complaint/Open/Click/Rendering Failure，没有"延迟"类型。SES 在 MTA 返回 4xx 时自行重试，最终要么 Delivery 要么 Bounce（此时 bounceType=Transient）。

7. **活动受众筛选是实时查询而非快照**：[buildRecipientWhereAsync](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/CampaignService.ts#L866-L906) 在每个 batch 处理时执行，`subscribed: true` 条件确保硬退/投诉被退订的 Contact 不会被后续批次选中。这是营销活动路径最核心的清洗机制——**退订即时生效**。

8. **Worker 层不检查订阅状态，不存在"兜底拦截"**：[email-processor](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/jobs/email-processor.ts#L73-L304) 只检查 `status === PENDING`、项目 disabled、钓鱼内容，不检查 `contact.subscribed`。订阅拦截完全依赖各路径的入口 Service 层（活动受众查询 / 工作流入口检查 / 事务性 API 营销模板检查）。[EmailService.sendEmail](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/services/EmailService.ts#L290-L456) 中的订阅检查是**死代码**，生产路径不调用此方法。

9. **活动路径的退订拦截最彻底**：退订 Contact 根本不会产生 Email 记录，而工作流路径只是创建 FAILED 记录但不入队。两者都不会消耗 SES 发送配额，但活动路径更干净——不会在 Email 表中留下 FAILED 垃圾记录。

10. **事务性类型不受退订限制是故意设计，不是遗漏**：三条路径中，只要邮件类型判定为事务性（sourceType=TRANSACTIONAL 或 campaign.type=TRANSACTIONAL），就跳过所有订阅检查。唯一例外是事务性 API 使用了营销模板（此时入口层抛 400）。验证码、密码重置、订单通知等场景本就不应该因为退订而停止送达。

11. **自定义收件人是订阅检查的"旁路"**：工作流的 `recipientEmail` 覆盖机制完全绕过 `contact.subscribed` 检查，通过 `X-Plunk-Recipient-Override` 内部 header 路由到自定义地址。自定义收件人自身没有独立的订阅/退信状态，**硬退惩罚会落到原 Contact 身上**（Contact 被退订），但下次继续用该自定义邮箱仍可发送。

12. **事务性 API 的新收件人默认退订是反垃圾措施**：`/v1/send` 调用 `ContactService.upsert` 时显式传 `defaultSubscribed=false`（[Actions.ts L274](file:///d:/fz/0601-2/solo-dogfeeding/code/15-plunk/apps/api/src/controllers/Actions.ts#L274)），符合反垃圾邮件"默认不营销"原则。但由于事务性模板不检查订阅，**第一次事务性邮件不会受阻**——真正受影响的是后续用该 Contact 发营销模板邮件的场景。

13. **CSV 导入的 subscribed 数字形式会被误判为 false**：import-processor 只识别字符串形式的 "1"，不识别数字 `1`。如果 CSV 导出工具将 subscribed 序列化为数字 0/1，则全部会被当作 false，可能导致导入后联系人全退订。

14. **重新订阅路径共 6 个，且都需要显式触发**：硬退/投诉后 subscribed 不会自动恢复。需要通过：订阅链接、REST API、事件追踪 API、工作流 UPDATE_CONTACT 步骤、批量订阅操作、CSV 导入，其中任何一种方式显式设为 true 才能解除退订状态。

15. **批量操作的游标机制对 delete 安全，但对 subscribe/unsubscribe 也有副作用**：query 模式用 `id < lastId` 倒序分批，是为了避免 delete 后行消失导致的游标跳跃。对 subscribe 来说，如果在操作过程中有新的订阅变更，可能不会被覆盖（因为游标已过）。但批量操作设计上就是"某一时刻的快照"，这个边界可接受。
