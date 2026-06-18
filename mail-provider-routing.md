# 邮件多 Provider 选路、降级黑名单与队列协作（v2 纠正版）

> 本文顺着代码重新核清 Plunk 的邮件发送链路。上一版把“发送服务入口（`EmailService.sendEmail`）”与“发送 worker（`email-processor`）”的职责混淆，把退订拦截、发件域校验错误地归给了 worker。本版按代码事实重新划分职责。
>
> 结论先行：**Plunk 出站只有 AWS SES 一个 transport，没有多 Provider 运行时选路**。所谓“选路”是 BullMQ `priority` 氃度；所谓“黑名单/降级”是多处闸门；而“发送”本身有**两条并存但职责不同的代码路径**——一条是仅测试调用的 `EmailService.sendEmail`，另一条才是生产真正使用的 `email-processor` worker。

---

## 0. 最重要的纠正：`EmailService.sendEmail` ≠ 生产发送路径

全仓搜索 `EmailService.sendEmail(` 的调用点，结果只有 [apps/api/src/services/\_\_tests\_\_/EmailService.test.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/__tests__/EmailService.test.ts)（L354/L372/L398/L422/L703）。**生产代码中没有任何地方调用它**。

- `EmailService.sendEmail` 定义在 [apps/api/src/services/EmailService.ts#L290](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L290)。它是一个“自包含的同步发送方法”：内含退订拦截（L311）、发件域校验（L331）、模板渲染、`sendRawEmail`、落 `SENT`、`EventService.trackEvent`。
- 生产发送路径不经过它。邮件先由入口方法（`sendTransactionalEmail`/`sendCampaignEmail`/`sendWorkflowEmail`）写成 `status=PENDING` 的 `Email` 记录入队，再由 [apps/api/src/jobs/email-processor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts) 的 worker 消费、内联完成渲染与发送。worker **不调用** `EmailService.sendEmail`，二者各自实现了一套发送逻辑。

因此下文凡涉及“发送时”的闸门，必须区分是“入口/收件人筛选阶段”还是“worker 消费阶段”，不能笼统说“发送时拦截”。

---

## 1. 全景架构（修正后）

```
 入口（写 Email 记录 PENDING + queueEmail）                   收件人筛选（退订拦截①）
 ┌───────────────────────────────────────────┐                ┌──────────────┐
 │ Actions.send(/v1/send)  → sendTransactionalEmail            │ campaign:    │
 │ SMTP Relay → /v1/send   → sendTransactionalEmail            │ buildRecipient│
 │ 平台通知 packages/email → /v1/send                          │ WhereAsync   │
 │ 营销广播 CampaignService.send → sendCampaignEmail           │ subscribed:true│
 │ 工作流 WorkflowExecutionService → sendWorkflowEmail(退订拦截②)│ └──────────────┘
 └───────────────────────────┬───────────────┘                     │
                             ▼                                      │
        BillingLimitService.checkLimit（计费闸门，入队前）           │
                             │                                      │
                             ▼                                      │
   ┌──── BullMQ `email` 队列（priority 选路：TRANSACTIONAL=1<WORKFLOW=5<CAMPAIGN=10）────┐
   │  jobId=email-{id} 幂等；limiter=rate/s；retry=3×exp 2s                              │
   └───────────────────────────────────┬──────────────────────────────────────────────┘
                                       ▼
   email-processor.ts（Worker，生产发送路径）
   ┌─────────────────────────────────────────────┐
   │ 1. status!==PENDING → return（已被处理）       │
   │ 2. project.disabled → FAILED（终态）         │
   │ 3. 渲染 + compileHTML + shouldTrackEmail      │
   │ 4. SecurityService.checkPhishingContent（采样）│ ← 退订拦截/域校验都不在这里
   │ 5. sendRawEmail → AWS SES（唯一出站 transport）│
   │ 6. 落 SENT + EventService.trackEvent + Meter  │
   │ 7. CampaignService.finalizeIfDone（campaign）│
   │ catch → FAILED + rethrow（触发 BullMQ 重试）  │
   └─────────────────────────────────────────────┘
```

发件域校验、退订拦截都不在 worker，而是分散在**入口层**和**收件人筛选层**。详见第 3 节。

---

## 2. 选路算法（Selection / Routing）

### 2.1 入口归一：四条触发路径 → 三个 EmailService 入口方法 → 一个 `email` 队列

| 触发路径 | 入口方法 | 来源类型 | 入口位置 |
| --- | --- | --- | --- |
| API `/v1/send`（含 SMTP Relay、平台通知转发） | `sendTransactionalEmail` | `TRANSACTIONAL`（用 marketing 模板时按订阅拦截②见下） | [apps/api/src/services/EmailService.ts#L52](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L52) |
| 营销广播分批 | `sendCampaignEmail` | `CAMPAIGN`（模板为 TRANSACTIONAL 时升级） | [apps/api/src/services/EmailService.ts#L119](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L119) |
| 自动化工作流 | `sendWorkflowEmail` | `WORKFLOW`（模板为 TRANSACTIONAL 时升级） | [apps/api/src/services/EmailService.ts#L183](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L183) |

三个入口都先过 `BillingLimitService.checkLimit`（超限抛 429），再写 `Email(status=PENDING)`，最后 `QueueService.queueEmail`。

### 2.2 队列内选路：BullMQ `priority`（唯一的“选路算法”）

[apps/api/src/services/QueueService.ts#L177](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L177) 的 `emailPriorityFor`：

```ts
TRANSACTIONAL → 1   // 最高，登录/验证码优先
WORKFLOW      → 5
CAMPAIGN      → 10  // 最低
default       → 5
```

投递在 [apps/api/src/services/QueueService.ts#L207](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L207)。要点：
- 所有来源共用一个 `email` 队列（[apps/api/src/services/QueueService.ts#L47](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L47)），靠 priority 排序，事务邮件不被营销洪流阻塞。
- `jobId = email-${id}` 天然幂等，重复入队不产生重复任务。

### 2.3 出站选路：只有 SES，按 `tracking` 选 Configuration Set

[apps/api/src/services/SESService.ts#L87](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SESService.ts#L87) 的 `sendRawEmail` 按 `tracking` 在两个 SES 配置集间选路（[apps/api/src/services/SESService.ts#L211](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SESService.ts#L211)）：`tracking=false` 且开关启用时用 `*_NO_TRACKING` 配置集。`tracking` 由 `shouldTrackEmail` 推导（[apps/api/src/services/EmailService.ts#L645](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L645)）：`MARKETING_ONLY` 模式下事务邮件不追踪。

---

## 3. 降级与黑名单（闸门）——按所在阶段重新归类

**核心纠正**：退订拦截与发件域校验都不在 worker。下表给出每道闸门真正所在的代码层与阶段。

| 闸门 | 真实位置 | 阶段 | 代码 |
| --- | --- | --- | --- |
| ① 退订拦截（campaign） | `buildRecipientWhereAsync` 的 SQL `WHERE` 加 `subscribed:true` | 收件人筛选（入队前） | [apps/api/src/services/CampaignService.ts#L866](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/CampaignService.ts#L866) |
| ② 退订拦截（workflow） | `sendWorkflowEmail` 落库前检查，未订阅营销联系人直接写 `FAILED` 占位 | 入口层（入队前） | [apps/api/src/services/EmailService.ts#L200](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L200) |
| ③ 退订拦截（transactional） | `sendTransactionalEmail` 仅在使用 marketing 模板时检查订阅；纯事务邮件不拦 | 入口层（入队前） | [apps/api/src/services/EmailService.ts#L62](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/EmailService.ts#L62) |
| ④ 发件域校验（API） | `Actions.send` controller 入队前调用 `verifyEmailDomain` | 入口层（入队前） | [apps/api/src/controllers/Actions.ts#L256](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/controllers/Actions.ts#L256) |
| ⑤ 发件域校验（test 邮件） | `CampaignService.sendTest` 调用 `verifyEmailDomain` | 入口层 | [apps/api/src/services/CampaignService.ts#L775](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/CampaignService.ts#L775) |
| ⑥ 项目 disable 拦截 | worker 取出任务后检查 `project.disabled` → `FAILED` | worker 消费阶段 | [apps/api/src/jobs/email-processor.ts#L103](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L103) |
| ⑦ 钓鱼采样拦截 | worker 发送前采样 `checkPhishingContent` | worker 消费阶段 | [apps/api/src/jobs/email-processor.ts#L185](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L185) |
| ⑧ 计费配额拦截 | `BillingLimitService.checkLimit`，超限抛 429 不入队 | 入口层（入队前） | [apps/api/src/services/BillingLimitService.ts#L149](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/BillingLimitService.ts#L149) |

### 3.1 退订黑名单的真实形态：联系人 `subscribed` 单一布尔位

- **没有独立的黑名单表/收件人地址黑名单**，抑制名单就是 `contact.subscribed`。
- 退订的触发源都在回执链路 [apps/api/src/controllers/Webhooks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/controllers/Webhooks.ts)：
  - SNS `Bounce` 且 `bounceType==='Permanent'` → `subscribed=false`（L374-L396）；
  - `Complaint` → `subscribed=false`（L431-L447）；
  - `bounceType==='Transient'` 软退信**不退订**，仅记录事件（L397-L408）。
- 一旦 `subscribed=false`：
  - campaign 路径在①处被 SQL 过滤掉，根本不入队；
  - workflow 路径在②处写 `FAILED` 占位（不发）；
  - transactional 纯事务邮件**仍会发送**（事务邮件语义上不受退订影响），只有用 marketing 模板时才在③被拦。

> ⚠️ 注意 campaign 的退订过滤依赖 `campaign.type !== TRANSACTIONAL`（①处 L873）。事务型 campaign 会发给未订阅者，这是设计行为。

### 3.2 项目级降级（disable = 整项目拉黑）

worker 在⑥处拦截 `disabled=true` 的项目。`disabledReason` 三类来源：

1. **邮件声誉超阈**（`EMAIL_REPUTATION`）：硬退信/投诉触发 [apps/api/src/services/SecurityService.ts#L272](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L272) → [disableProject](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L654)。
2. **钓鱼检测**（`PHISHING_DETECTED`）：worker ⑦处采样命中后 [disableProjectForPhishing](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L946)。
3. **支付失败**（`PAYMENT_FAILED`）：Stripe `invoice.payment_failed`（[apps/api/src/controllers/Webhooks.ts#L591](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/controllers/Webhooks.ts#L591)）。

自托管可用 `AUTO_PROJECT_DISABLE=false` 关闭自动拉黑（[apps/api/src/app/constants.ts#L125](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/app/constants.ts#L125)）。

### 3.3 发送失败的降级：重试而非换 Provider

worker catch 后把邮件置 `FAILED` 并 `rethrow` 触发 BullMQ 重试（[apps/api/src/jobs/email-processor.ts#L266](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L266)），默认 3 次、指数退避起始 2s（[apps/api/src/services/QueueService.ts#L49](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L49)）。**重试仍是同一个 SES，没有二级 Provider 兜底**。

---

## 4. 发送服务入口 vs 发送 Worker 职责差异（本版重点）

两者都包含“渲染 + sendRawEmail + 落 SENT + trackEvent”，但闸门职责**几乎互补**：

| 职责 | `EmailService.sendEmail`（仅测试用） | `email-processor` worker（生产路径） |
| --- | --- | --- |
| `status===PENDING` 检查 | ✅ | ✅ |
| `project.disabled → FAILED` | ❌ | ✅（⑥） |
| 退订拦截 `contact.subscribed` | ✅（L311） | ❌（改在入口/筛选层①②③） |
| `verifyEmailDomain` 发件域校验 | ✅（L331） | ❌（改在入口层④⑤） |
| 钓鱼采样 `checkPhishingContent` | ❌ | ✅（⑦） |
| `MeterService` 计量记账 | ❌ | ✅ |
| `CampaignService.finalizeIfDone` | ❌ | ✅ |
| 失败 rethrow 触发重试 | ✅(throw) | ✅(throw) |
| 生产调用方 | 仅测试 | BullMQ worker |

> 这正是上一版“对不上”的根源：把 sendEmail 的退订拦截/域校验误记到了 worker 名下。实际上 worker 只负责 disabled + 钓鱼 + 计量 + campaign 收尾，**信任**入口层已做完退订与域校验。

补充：`CampaignService.send` 正式广播链路（`send → startSending → processBatch → sendCampaignEmail`）中**未见 `verifyEmailDomain` 调用**；campaign 的发件域约束靠 `from` 在创建/配置阶段保证，发送链路本身不重复校验。只有 `sendTest`（⑤）显式校验域。

---

## 5. 队列协作（Queue Collaboration）

### 5.1 队列拓扑与邮件链路

10 个 BullMQ 队列共享一条 Redis 连接（[apps/api/src/services/QueueService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts)），由 [apps/api/src/jobs/worker.ts#L25](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/worker.ts#L25) 统一拉起。邮件相关协作链：

```
scheduled ─(到点)─► CampaignService.startSending ─► campaign 队列(分批)
campaign  ─(每批 N 联系人)─► sendCampaignEmail ─► email 队列
workflow  ─(SEND_EMAIL 步)─► sendWorkflowEmail ─► email 队列
email     ─(worker)─► AWS SES ─► SNS 回执 ─► /webhooks/sns（退订/声誉回流）
```

### 5.2 campaign ↔ email 链式批处理

- campaign 队列并发 5（[apps/api/src/jobs/campaign-processor.ts#L13](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/campaign-processor.ts#L13)）。
- [processBatch](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/CampaignService.ts#L513) 游标分页，每联系人 `sendCampaignEmail → email 队列`；当前批处理完**自链投递下一批**（[apps/api/src/services/CampaignService.ts#L587](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/CampaignService.ts#L587)）。
- 末批 reconcile `totalRecipients` 后 [finalizeIfDone](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/CampaignService.ts#L617) 推进 campaign 状态，`SENT` 与 `FAILED` 都算终态，避免部分失败时卡 `SENDING`。

### 5.3 email 队列限速/并发（由 SES 配额推导）

[apps/api/src/jobs/email-processor.ts#L30](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L30)：
- 限速优先级：`EMAIL_RATE_LIMIT_PER_SECOND`(env) > `SES getSendQuota()` > 安全默认 14/s；
- 并发 = `rate * 0.5`，clamp 到 `[5, EMAIL_WORKER_MAX_CONCURRENCY=50]`（[deriveWorkerConcurrency#L62](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L62)）；
- BullMQ `limiter:{max, duration:1000}` 令牌桶（[apps/api/src/jobs/email-processor.ts#L284](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/jobs/email-processor.ts#L284)）。

### 5.4 项目拉黑时的跨队列级联清理

[cancelAllProjectJobs](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L545) 扫 `scheduled/email/campaign/workflow` 的 `waiting|delayed` 任务比对 `projectId` 后 `remove()`；把 `PENDING` 邮件置 `FAILED`；对 `SENDING` 中的 campaign reconcile 并 `finalizeIfDone` 收尾。声誉/钓鱼 disable 都会调用它（[SecurityService#L694](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L694)、[SecurityService#L983](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/SecurityService.ts#L983)）。

### 5.5 运维协作

[pauseAll/resumeAll](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L480)（维护期整体暂停/恢复）、[cleanOldJobs](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L516)（completed 24h、failed 7 天、meter 30 天）、[getStats](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/services/QueueService.ts#L438)（聚合计数）。

---

## 6. 关键环境变量速查

| 变量 | 作用 | 默认 |
| --- | --- | --- |
| `AWS_SES_*` | 唯一出站 transport 凭证 | 必填 |
| `EMAIL_RATE_LIMIT_PER_SECOND` | 覆盖 SES 限速 | 取 SES 配额 |
| `EMAIL_WORKER_CONCURRENCY` / `EMAIL_WORKER_MAX_CONCURRENCY` | worker 并发 | 推导 / 50 |
| `SES_CONFIGURATION_SET(_NO_TRACKING)` | 追踪开/关的配置集选路 | plunk-configuration-set[-no-tracking] |
| `AUTO_PROJECT_DISABLE` | 声誉/钓鱼超阈是否自动拉黑 | true |
| `PHISHING_DETECTION_*` | 钓鱼采样率/置信阈/累计阈 | 0.1 / 95 / 3 |
| `PLUNK_API_KEY` / `PLUNK_FROM_ADDRESS` | 平台通知邮件自反调用 `/v1/send` | 空=禁用 |
| `SMTP_*` | 入站 SMTP Relay | 默认 localhost |

定义集中在 [apps/api/src/app/constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/33-plunk/apps/api/src/app/constants.ts)。

---

## 7. 一句话总结（修正版）

- **选路** = 多入口归一进单 `email` 队列 + BullMQ `priority` 按来源类型排序 + 出站仅 SES（按 tracking 选 Configuration Set）。
- **退订拦截**不在 worker：campaign 走收件人筛选 SQL（①），workflow 走入口落库前（②），transactional 仅 marketing 模板在入口拦（③）。
- **发件域校验**不在 worker：在 `Actions.send`（④）与 `sendTest`（⑤）；正式广播链路未见显式校验。
- **发送 worker 职责** = disabled 拦截 + 钓鱼采样 + 渲染发送 + 计量 + campaign 收尾 + 重试；**信任**入口层已做完退订与域校验。
- `EmailService.sendEmail` 是与 worker 并存的另一套发送逻辑，但**仅测试调用**，不参与生产发送。
- **降级**无二级 Provider，失败只靠 BullMQ 3 次重试；项目拉黑时跨队列级联清理。
