# 邮件多 Provider 选路、降级黑名单与队列协作

> 本文顺着代码梳理 Plunk 中“邮件如何被选路、何时被降级/拉黑、队列之间如何协作”的真实逻辑。
> 结论先行：**Plunk 并没有一个“在多个发送 Provider 之间做运行时选路”的抽象层**，出站只有 AWS SES 一个 transport；但它存在**多条入站/触发路径**、**按来源类型的优先级选路**、**多道降级/黑名单闸门**以及**多个 BullMQ 队列的协作链路**。下面把这四件事分别讲清楚。

---

## 1. 全景架构

```
                       ┌──────────────────────── 入站/触发路径（多） ─────────────────────────┐
                       │                                                                      │
  外部 SMTP 客户端 ──► apps/smtp (SMTP Relay) ──► POST /v1/send ──┐                            │
  外部 API 调用   ──► POST /v1/send (Actions.send) ───────────────┤                            │
  平台通知        ──► packages/email ──► POST /v1/send ──────────┤                            │
  营销广播        ──► CampaignService.send (campaign 队列分批) ───┤                            │
  自动化          ──► WorkflowExecutionService (workflow 队列) ──┤                            │
  SES 收信        ──► SNS Webhook (/webhooks/sns, notificationType=Received) ── 直接落库(不入发送队列)│
                       │                                                                      │
                       ▼                                                                      │
              EmailService.send*Email()                                                       │
              ┌───────────────────────────────────────┐                                       │
              │ 1. 来源类型判定 (TRANSACTIONAL/CAMPAIGN/WORKFLOW)                              │
              │ 2. 计费闸门 BillingLimitService.checkLimit                                      │
              │ 3. 写 Email 记录 (status=PENDING)                                              │
              │ 4. QueueService.queueEmail(emailId, sourceType)                                │
              └───────────────────┬───────────────────────┘                                       │
                                  ▼                                                              │
                       ┌─────────── BullMQ `email` 队列（按 priority 选路）──────────┐          │
                       │  priority: TRANSACTIONAL=1 < WORKFLOW=5 < CAMPAIGN=10         │          │
                       │  limiter: max=rateLimit/s, duration=1000ms                  │          │
                       │  retry: attempts=3, exponential backoff 2s                  │          │
                       └───────────────────┬───────────────────────────────────────┘          │
                                           ▼                                                   │
                  jobs/email-processor.ts (Worker)                                              │
                  ┌───────────────────────────────────────────────┐                          │
                  │ A. project.disabled? → FAILED (终态)            │                          │
                  │ B. contact.subscribed? + 非事务 → FAILED        │                          │
                  │ C. DomainService.verifyEmailDomain 闸门          │                          │
                  │ D. SecurityService.checkPhishingContent (采样)   │                          │
                  │ E. sendRawEmail → AWS SES（唯一出站 transport） │                          │
                  └───────────────────────────────────────────────┘                          │
                                           │                                                   │
                                           ▼                                                   │
                  AWS SES → 收件人                                                          │
                  回执通过 SNS → /webhooks/sns 回流：Delivery/Open/Click/Bounce/Complaint      │
                  → 触发 SecurityService.checkAndEnforceSecurityLimits（可能再次拉黑）         │
```

**唯一的出站 transport 是 [SESService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SESService.ts) 里的 `sendRawEmail`**，客户端在 [SESService.ts#L18-L25](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SESService.ts#L18-L25) 初始化。所谓“多 provider 选路”更准确地说是“多入口 + 单出口 + 多级闸门”。

---

## 2. 选路算法（Selection / Routing）

### 2.1 入口选择：四条触发路径汇入同一个 `email` 队列

无论从哪条路径进来，最终都由 [EmailService](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts) 的三个入口方法创建 `Email` 记录（`status=PENDING`）并调用 `queueEmail`：

| 触发路径 | 入口方法 | 来源类型 | 代码位置 |
| --- | --- | --- | --- |
| API `/v1/send`（含 SMTP Relay、平台通知转发的请求） | `sendTransactionalEmail` | `TRANSACTIONAL` | [EmailService.ts#L52-L114](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L52-L114) |
| 营销广播分批 | `sendCampaignEmail` | `CAMPAIGN`（模板为 TRANSACTIONAL 时升级为事务） | [EmailService.ts#L119-L178](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L119-L178) |
| 自动化工作流 | `sendWorkflowEmail` | `WORKFLOW`（模板为 TRANSACTIONAL 时升级为事务） | [EmailService.ts#L183-L284](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L183-L284) |

来源类型枚举定义在 [schema.prisma#L770-L775](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/packages/db/prisma/schema.prisma#L770-L775)（`TRANSACTIONAL / CAMPAIGN / WORKFLOW / INBOUND`）。`INBOUND` 是 SES 收信落库，不走发送队列。

> 三条入口在写库前都先做 **计费闸门** `BillingLimitService.checkLimit`，超限直接抛 429，不入队；详见第 3 节。

### 2.2 队列内选路：BullMQ `priority`（核心“选路算法”）

真正意义上的“选路”发生在 [QueueService.queueEmail](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L202-L216)。它把同一封邮件以 BullMQ `priority`（数值越小越先出队）投递进共享的 `email` 队列：

```ts
function emailPriorityFor(sourceType: EmailSourceType): number {
  switch (sourceType) {
    case EmailSourceType.TRANSACTIONAL: return 1;   // 最高
    case EmailSourceType.WORKFLOW:      return 5;
    case EmailSourceType.CAMPAIGN:      return 10;  // 最低
    default:                            return 5;
  }
}
```

见 [QueueService.ts#L177-L188](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L177-L188) 与 [QueueService.ts#L207-L216](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L207-L216)。

**算法要点**：
- 所有来源共用**一个 `email` 队列**（[QueueService.ts#L47-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L47-L58)），靠 priority 区分先后。这样登录验证码、密码重置等时延敏感邮件不会被大批营销邮件阻塞。
- `jobId` 固定为 `email-${emailId}`，**天然幂等**：同一封邮件重复入队不会产生重复任务。
- 支持 `delay` 参数（用于工作流的 DELAY 步骤延迟投递）。

### 2.3 出站选路：只有 AWS SES，但按“是否追踪”选择 Configuration Set

出站没有第二 provider，但 [sendRawEmail](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SESService.ts#L87-L231) 会根据 `tracking` 标志在两个 SES 配置集之间“选路”：

```ts
const configurationSetName =
  TRACKING_TOGGLE_ENABLED && !tracking
    ? SES_CONFIGURATION_SET_NO_TRACKING
    : SES_CONFIGURATION_SET;
```

见 [SESService.ts#L211-L224](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SESService.ts#L211-L224)。`tracking` 又由项目追踪模式 + 来源类型推导（`shouldTrackEmail`，[EmailService.ts#L645-L657](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L645-L657)）：`MARKETING_ONLY` 模式下事务邮件不追踪。

### 2.4 域名闸门（出站前的硬选路）

入队前在 controller 层（[Actions.send#L256](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/controllers/Actions.ts#L256)）以及 worker 内（[email-processor 实际未再校验，由 `EmailService.sendEmail` 校验](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L328-L331)）都会调用 [DomainService.verifyEmailDomain](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/DomainService.ts#L349-L389)：
- 域名必须已注册、属于当前项目、且 `verified=true`，否则 403 拒绝。这是“从哪个发件域发出”的选路约束。

---

## 3. 降级与黑名单（Degradation / Suppression）

Plunk 没有传统意义上的“主 provider 故障后切到备用 provider”的降级链。它的“降级/黑名单”是**多层闸门**：未通过即不发送（或入队后被拦下标记 FAILED）。

### 3.1 联系人级黑名单（订阅态 = 抑制名单）

- **退订联系人不发营销邮件**：worker 取出邮件后，若 `contact.subscribed === false` 且邮件非事务型，直接标记 `FAILED` 并写 `error: 'Contact is unsubscribed from marketing emails'`，见 [email-processor.ts#L99-L120](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts) 与 [EmailService.sendEmail#L309-L326](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L309-L326)。
- 工作流入口在落库前就跳过未订阅营销联系人，直接写一条 `FAILED` 占位记录，见 [EmailService.sendWorkflowEmail#L200-L235](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L200-L235)。
- **永久退信（硬退信）→ 自动退订**：SNS `Bounce` 事件中 `bounceType === 'Permanent'` 时把联系人置为 `subscribed=false`，见 [Webhooks.ts#L374-L396](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/controllers/Webhooks.ts#L374-L396)。
- **投诉（Complaint）→ 自动退订**：[Webhooks.ts#L431-L447](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/controllers/Webhooks.ts#L431-L447)。
- **瞬时退信（软退信）不拉黑**：`bounceType === 'Transient'` 只记录事件，不改状态、不退订，不计入退信率，见 [Webhooks.ts#L397-L408](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/controllers/Webhooks.ts#L397-L408)。

> 注意：抑制名单就是 `contact.subscribed` 这一个布尔位，**没有独立的黑名单表/收件人地址黑名单**。一旦退订，后续营销邮件在闸门处被拦。

### 3.2 项目级降级（disable = 整个项目被拉黑）

项目 `disabled=true` 时，worker 取到邮件会立即标记 `FAILED: 'Project is disabled'` 并 `finalizeIfDone`，见 [email-processor.ts#L103-L120](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L103-L120)。项目被 disable 的三种原因（`disabledReason`）：

1. **邮件声誉超阈**（`EMAIL_REPUTATION`）：永久退信/投诉后触发 [SecurityService.checkAndEnforceSecurityLimits](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L272-L332) → [disableProject](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L654-L727)。阈值体系见 [SecurityService.ts#L30-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L30-L69)，区分 7 天/全量、新项目绝对计数上限、最小样本数等多重判定，避免误杀。
2. **钓鱼检测**（`PHISHING_DETECTED`）：worker 发送前采样调用 [SecurityService.checkPhishingContent](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L734-L941)（OpenRouter LLM，默认 10% 采样）。满足“单次高置信 ≥ 阈值”或“窗口内累计 ≥ 阈值”即 [disableProjectForPhishing](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L946-L1016)。worker 内联拦截见 [email-processor.ts#L185-L212](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L185-L212)。
3. **支付失败**（`PAYMENT_FAILED`）：Stripe `invoice.payment_failed` 触发，见 [Webhooks.ts#L591-L643](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/controllers/Webhooks.ts#L591-L643)；支付成功后可被重新启用。

> 自托管者可用 `AUTO_PROJECT_DISABLE=false` 关闭自动 disable，只告警不拉黑（[constants.ts#L125](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/app/constants.ts#L125)）。

### 3.3 计费级降级（配额 = 软拉黑）

[BillingLimitService.checkLimit](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/BillingLimitService.ts#L149-L353) 在每次入队前校验：
- **免费层**（启用计费且无订阅、无自定义限额）：所有来源共享 1000 封/月总配额，超限 `allowed=false` → 入口方法抛 429，见 [BillingLimitService.ts#L203-L264](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/BillingLimitService.ts#L203-L264)。
- **付费层**：按 `sourceType` 分别校验 `billingLimitWorkflows/Campaigns/Transactional/Inbound`，超限同样阻断，见 [BillingLimitService.ts#L266-L305](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/BillingLimitService.ts#L266-L305)。
- 达到 80% 触发告警邮件/ntfy。

### 3.4 发送失败的降级行为：重试，而非换 provider

worker 内 `try/catch` 失败后把邮件置为 `FAILED` 并 **重新抛出** 以触发 BullMQ 重试（[email-processor.ts#L266-L279](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L266-L279)）。队列默认 **3 次尝试、指数退避起始 2s**（[QueueService.ts#L49-L57](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L49-L57)）。**重试仍是同一个 SES**，没有二级 provider 兜底。

---

## 4. 队列协作（Queue Collaboration）

### 4.1 队列拓扑：10 个 BullMQ 队列共享一条 Redis 连接

全部定义在 [QueueService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts)，由统一的 [worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/worker.ts#L25-L84) 进程拉起所有 worker。

与邮件直接相关的协作链：

```
scheduled 队列 ──(到点)──► CampaignService.send ──► campaign 队列(分批)
workflow 队列 ──(SEND_EMAIL 步)──► EmailService.sendWorkflowEmail ──► email 队列
campaign 队列 ──(每批 N 联系人)──► EmailService.sendCampaignEmail ──► email 队列
email 队列 ──(Worker)──► AWS SES
```

### 4.2 campaign ↔ email 的链式批处理协作

- 营销广播先入 `campaign` 队列按批处理（[campaign-processor.ts#L13-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/campaign-processor.ts#L13-L44)，并发 5）。
- 每批在 [CampaignService.processBatch#L513-L609](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/CampaignService.ts#L513-L609) 内为每个联系人调用 `sendCampaignEmail` → 进入 `email` 队列。
- 游标分页处理完当前批后，**自链投递下一批**（[CampaignService.ts#L587-L593](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/CampaignService.ts#L587-L593)），直到没有更多联系人。
- 最后一批会 reconcile `totalRecipients` 并调用 [finalizeIfDone](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/CampaignService.ts#L617) 推进 campaign 状态。`finalizeIfDone` 把 `SENT` 与 `FAILED` 都算作终态，避免部分失败时 campaign 永远卡在 `SENDING`。

### 4.3 email 队列自身的并发与限速协作

worker 启动时根据 SES 配额推导限速与并发（[email-processor.ts#L30-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L30-L79)）：

- **限速优先级**：`EMAIL_RATE_LIMIT_PER_SECOND`（env）> AWS SES `getSendQuota()` > 安全默认 14/s。
- **并发推导**：`rate * 0.5` 给约 2× 余量，clamp 到 `[5, EMAIL_WORKER_MAX_CONCURRENCY=50]`，见 [deriveWorkerConcurrency#L62-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L62-L71)。
- BullMQ `limiter: { max: rateLimit, duration: 1000 }` 做令牌桶限速，[email-processor.ts#L284-L288](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L284-L288)。

### 4.4 项目 disable 时的级联清理（队列协作的关键）

一旦项目被拉黑（声誉/钓鱼/支付），[QueueService.cancelAllProjectJobs](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L545-L640) 会：

1. 扫描 `scheduled / email / campaign / workflow` 四个队列里 `waiting|delayed` 的任务，逐个查库比对 `projectId` 后 `job.remove()`；
2. 把仍处于 `PENDING` 的邮件批量置为 `FAILED: 'Project is disabled'`；
3. 对 `SENDING` 中的 campaign 重新 reconcile `totalRecipients` 并 `finalizeIfDone`，让 campaign 以部分发送量收尾，而不是永远卡住。

这保证了“拉黑”这一降级动作能跨队列一致传播，避免悬挂任务。SecurityService / 钓鱼 disable 都会调用它（[SecurityService.ts#L694-L700](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L694-L700)、[SecurityService.ts#L983-L989](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L983-L989)）。

### 4.5 运维协作

- [pauseAll / resumeAll](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L480-L511)：维护期整体暂停/恢复。
- [cleanOldJobs](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L516-L539)：completed 保留 24h，failed 保留 7 天（计费类 meter 保留 30 天用于审计）。
- [getStats](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L438-L475)：聚合所有队列的 waiting/active/completed/failed/delayed 计数。

---

## 5. 关键环境变量速查

| 变量 | 作用 | 默认 |
| --- | --- | --- |
| `AWS_SES_*` | 唯一出站 transport 凭证 | 必填 |
| `EMAIL_RATE_LIMIT_PER_SECOND` | 覆盖 SES 限速 | 取 SES 配额 |
| `EMAIL_WORKER_CONCURRENCY` | 固定 worker 并发 | 按 rate 推导 |
| `EMAIL_WORKER_MAX_CONCURRENCY` | 推导并发上限 | 50 |
| `SES_CONFIGURATION_SET` / `SES_CONFIGURATION_SET_NO_TRACKING` | 追踪开/关的配置集选路 | plunk-configuration-set[-no-tracking] |
| `AUTO_PROJECT_DISABLE` | 声誉/钓鱼超阈是否自动拉黑项目 | true |
| `PHISHING_DETECTION_*` | 钓鱼采样率/置信阈/累计阈 | 0.1 / 95 / 3 |
| `PLUNK_API_KEY` / `PLUNK_FROM_ADDRESS` | 平台通知邮件自反调用 `/v1/send` | 空=禁用 |
| `SMTP_*` | 入站 SMTP Relay | 默认 localhost |

定义集中在 [constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/app/constants.ts)。

---

## 6. 一句话总结

- **选路** = 入口归一（多路径 → `EmailService` → 单 `email` 队列）+ 队列内 BullMQ `priority` 按来源类型排序 + 出站仅 SES（按 tracking 选 Configuration Set）+ 发件域 `verifyEmailDomain` 闸门。
- **降级/黑名单** = 联系人 `subscribed` 抑制位（退订/硬退信/投诉触发）+ 项目 `disabled`（声誉/钓鱼/支付触发）+ 计费配额阻断；**没有二级 provider 兜底**，失败只靠 BullMQ 重试。
- **队列协作** = `scheduled→campaign→email` 与 `workflow→email` 的链式投递，限速/并发由 SES 配额推导，项目拉黑时跨队列级联清理并推进 campaign 收尾。
