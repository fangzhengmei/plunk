# 邮件多 Provider 选路、降级黑名单与队列协作（v3 再次纠正）

> 本文顺着代码重新核清 Plunk 的邮件发送链路。v1 把发送 service 入口与 worker 职责混淆；v2 又遗漏了域校验在"创建/更新"阶段做、workflow 自定义收件人绕过退订、三个入口退订占位与计费顺序不一致等细节。本版逐一核对代码事实，纠正上述偏差。
>
> 结论先行：**Plunk 出站只有 AWS SES 一个 transport，没有多 Provider 运行时选路**。所谓"选路"是 BullMQ `priority` 维度；所谓"黑名单/降级"是多处闸门；而"发送"本身有**两条并存但职责不同的代码路径**——一条是仅测试调用的 `EmailService.sendEmail`，另一条才是生产真正使用的 `email-processor` worker。

---

## 0. 最重要的纠正：`EmailService.sendEmail` ≠ 生产发送路径

全仓搜索 `EmailService.sendEmail(` 的调用点，结果只有 [apps/api/src/services/\_\_tests\_\_/EmailService.test.ts](apps/api/src/services/__tests__/EmailService.test.ts)（L354/L372/L398/L422/L703）。**生产代码中没有任何地方调用它**。

- `EmailService.sendEmail` 定义在 [apps/api/src/services/EmailService.ts#L290](apps/api/src/services/EmailService.ts#L290)。它是一个"自包含的同步发送方法"：内含退订拦截（L311）、发件域校验（L331）、模板渲染、`sendRawEmail`、落 `SENT`、`EventService.trackEvent`。
- 生产发送路径不经过它。邮件先由入口方法（`sendTransactionalEmail`/`sendCampaignEmail`/`sendWorkflowEmail`）处理：**正常情况下写一条 `status=PENDING` 的 `Email` 记录并 `queueEmail` 入队**，再由 [apps/api/src/jobs/email-processor.ts](apps/api/src/jobs/email-processor.ts) 的 worker 消费、内联完成渲染与发送。但有**一个例外**：`sendWorkflowEmail` 对未订阅的营销联系人会写一条 `status=FAILED` 的占位记录后**直接 return**——不写 `PENDING`、不计费、不入队（见 §3.2⑤b）。worker **不调用** `EmailService.sendEmail`，二者各自实现了一套发送逻辑。

因此下文凡涉及"发送时"的闸门，必须区分"入口/收件人筛选阶段"与"worker 消费阶段"，不能笼统说"发送时拦截"。

---

## 1. 全景架构（v3 修正后）

```
 ┌──────────── 资源创建/更新阶段（controller 做域校验①②）────────────┐
 │ Templates.create / Templates.update（from 字段）                    │
 │ Campaigns.create / Campaigns.update（from 字段）                    │
 │   └─► DomainService.verifyEmailDomain   ←—— 入口 controller 层做     │
 └──────────────────────────────────────────────────────────────────┘
                              │创建时保证 from 合法
                              ▼
 入口（正常：PENDING + queueEmail；workflow 未订阅营销：FAILED 占位不入队）   收件人筛选（退订拦截③）
 ┌───────────────────────────────────────────┐     ┌──────────────┐
 │ Actions.send(/v1/send)  → sendTransactionalEmail│ campaign:    │
 │ SMTP Relay → /v1/send   → sendTransactionalEmail│ buildRecipient│
 │ 平台通知 packages/email → /v1/send              │ WhereAsync   │
 │ 营销广播 CampaignService.send → sendCampaignEmail│ subscribed:true│
 │ 工作流 SEND_EMAIL step → sendWorkflowEmail       └──────────────┘
 │   (recipient=CUSTOM 时绕过退订⑤b；未订阅营销写 FAILED 占位不入队⑤b)        │
 └───────────────────────────┬───────────────┘
                             ▼
       入口各自的退订拦截⑤/计费⑥先后顺序（三个入口不一致，见 §3）
                             ▼
   ┌──── BullMQ `email` 队列（priority 选路：TRANSACTIONAL=1<WORKFLOW=5<CAMPAIGN=10）────┐
   │  jobId=email-{id} 幂等；limiter=rate/s；retry=3×exp 2s                              │
   └───────────────────────────────────┬──────────────────────────────────────────────┘
                                       ▼
   email-processor.ts（Worker，生产发送路径）
   ┌─────────────────────────────────────────────┐
   │ 1. status!==PENDING → return（已被处理）       │
   │ 2. project.disabled → FAILED（终态）⑦          │
   │ 3. 渲染 + compileHTML + shouldTrackEmail      │
   │ 4. SecurityService.checkPhishingContent（采样）⑧│
   │ 5. sendRawEmail → AWS SES（唯一出站 transport）│ ← sendTest 发送前还会再 verifyDomain⑨
   │ 6. 落 SENT + EventService.trackEvent + Meter  │
   │ 7. CampaignService.finalizeIfDone（campaign）│
   │ catch → FAILED + rethrow（触发 BullMQ 重试）  │
   └─────────────────────────────────────────────┘
```

**worker 不做**：退订拦截、发件域校验——前者入口层/筛选层已做，后者资源创建/更新时已保证。

> ⚠️ "入口写 PENDING 入队"有一个例外：`sendWorkflowEmail` 对未订阅的营销联系人走 `if (!contact?.subscribed)` 分支，`return await prisma.email.create({... status: FAILED ...})` 后直接返回（[apps/api/src/services/EmailService.ts#L209-L234](apps/api/src/services/EmailService.ts#L209-L234)）——**不写 PENDING、不调用 checkLimit、不 incrementUsage、不 queueEmail**。这条 FAILED 占位记录留在库里作为"已跳过"的审计痕迹，但永远不会进 `email` 队列、不会被 worker 消费。详见 §3.2⑤b。

---

## 2. 选路算法（Selection / Routing）

### 2.1 入口归一：四条触发路径 → 三个 EmailService 入口方法 → 一个 `email` 队列

| 触发路径 | 入口方法 | 来源类型 | 入口位置 |
| --- | --- | --- | --- |
| API `/v1/send`（含 SMTP Relay、平台通知转发） | `sendTransactionalEmail` | `TRANSACTIONAL`（marketing 模板时按订阅拦截⑤a） | [apps/api/src/services/EmailService.ts#L52](apps/api/src/services/EmailService.ts#L52) |
| 营销广播分批 | `sendCampaignEmail` | `CAMPAIGN`（模板 TRANSACTIONAL 时升级） | [apps/api/src/services/EmailService.ts#L119](apps/api/src/services/EmailService.ts#L119) |
| 自动化工作流 SEND_EMAIL step | `sendWorkflowEmail` | `WORKFLOW`（模板 TRANSACTIONAL 时升级；CUSTOM 收件人绕过退订⑤b） | [apps/api/src/services/EmailService.ts#L183](apps/api/src/services/EmailService.ts#L183) |

"归一"指：无论哪条触发路径，最终都由 `EmailService` 的某个入口方法处理，且正常发送的邮件都汇入**同一个** `email` 队列（而非每条路径各自一个队列）。但**不是每个入口调用都一定写 PENDING 并入队**——三条入口在"未订阅营销联系人"上的行为分岔：

| 入口方法 | 未订阅营销联系人时的处理 | 是否写 PENDING | 是否入队 |
| --- | --- | --- | --- |
| `sendTransactionalEmail` | throw 400（不写记录） | ❌ | ❌ |
| `sendCampaignEmail` | 不在此处判断（信任③的 SQL 预筛选，未订阅者根本到不了这里） | — | — |
| `sendWorkflowEmail` | 写 `status=FAILED` 占位后**直接 return** | ❌（写 FAILED） | ❌ |

因此准确说法是：**正常路径下**三个入口写 `Email(status=PENDING)` 后 `QueueService.queueEmail`；`sendWorkflowEmail` 的退订占位分支是例外——它写 `FAILED` 占位后 `return`，既不写 `PENDING` 也不入队（见 §3.2⑤b）。

### 2.2 队列内选路：BullMQ `priority`（唯一的"选路算法"）

[apps/api/src/services/QueueService.ts#L177](apps/api/src/services/QueueService.ts#L177) 的 `emailPriorityFor`：

```ts
TRANSACTIONAL → 1   // 最高，登录/验证码优先
WORKFLOW      → 5
CAMPAIGN      → 10  // 最低
default       → 5
```

投递在 [apps/api/src/services/QueueService.ts#L207](apps/api/src/services/QueueService.ts#L207)。要点：
- 所有来源共用一个 `email` 队列（[apps/api/src/services/QueueService.ts#L47](apps/api/src/services/QueueService.ts#L47)），靠 priority 排序，事务邮件不被营销洪流阻塞。
- `jobId = email-${id}` 天然幂等，重复入队不产生重复任务。

### 2.3 出站选路：只有 SES，按 `tracking` 选 Configuration Set

[apps/api/src/services/SESService.ts#L87](apps/api/src/services/SESService.ts#L87) 的 `sendRawEmail` 按 `tracking` 在两个 SES 配置集间选路（[apps/api/src/services/SESService.ts#L211](apps/api/src/services/SESService.ts#L211)）：`tracking=false` 且开关启用时用 `*_NO_TRACKING` 配置集。`tracking` 由 `shouldTrackEmail` 推导（[apps/api/src/services/EmailService.ts#L645](apps/api/src/services/EmailService.ts#L645)）：`MARKETING_ONLY` 模式下事务邮件不追踪。

---

## 3. 降级与黑名单（闸门）——按阶段重新归类（v3 核心纠正）

**v2 的偏差**：把发件域校验笼统归为"入口层"（发送时才做），实际上 campaign/template 创建/更新时 controller 已校验；把三个入口的"退订/计费顺序"混为一谈，实际上 sendTransactionalEmail 抛异常、sendWorkflowEmail 写 FAILED 占位、sendCampaignEmail 信任收件人筛选。

### 3.1 发件域校验（4 道闸门，分散在 3 个阶段）

发件域校验**不是"发送时才做"**，资源创建/更新时已保证合法性，发送路径仅做最后一道兜底：

| 编号 | 阶段 | 位置 | 触发条件 | 代码 |
| --- | --- | --- | --- | --- |
| ①a | 资源创建（template） | `TemplatesController.create` | 任何 template 创建 | [apps/api/src/controllers/Templates.ts#L88](apps/api/src/controllers/Templates.ts#L88) |
| ①b | 资源更新（template） | `TemplatesController.update` | `from` 字段被修改时（`if (from)`） | [apps/api/src/controllers/Templates.ts#L122](apps/api/src/controllers/Templates.ts#L122) |
| ②a | 资源创建（campaign） | `CampaignsController.create` | 任何 campaign 创建 | [apps/api/src/controllers/Campaigns.ts#L36](apps/api/src/controllers/Campaigns.ts#L36) |
| ②b | 资源更新（campaign） | `CampaignsController.update` | `from` 字段被修改时（`if (from)`） | [apps/api/src/controllers/Campaigns.ts#L169](apps/api/src/controllers/Campaigns.ts#L169) |
| ④ | 发送入口（API） | `Actions.send` controller | `/v1/send` 调用时（非 template/campaign 场景，from 可能动态传入） | [apps/api/src/controllers/Actions.ts#L256](apps/api/src/controllers/Actions.ts#L256) |
| ⑨ | 发送入口（test 邮件） | `CampaignService.sendTest` | 仅 campaign 测试邮件，直接走 `sendRawEmail` 前 | [apps/api/src/services/CampaignService.ts#L775](apps/api/src/services/CampaignService.ts#L775) |

**关键修正**：
- `CampaignService.create/update` 和 `TemplateService.create/update` 的 service 方法内**不做** `verifyEmailDomain`，校验在 controller 层调用 service**之前**完成。
- `CampaignService.send` 正式广播链路（send→startSending→processBatch→sendCampaignEmail）中**没有再调用 `verifyEmailDomain`**——因为创建/更新 campaign 时已经保证 `from` 合法，发送链路信任数据库里的 `from`。
- 只有 **API 直发送**（`/v1/send`，from 可能动态指定，未经资源创建阶段）和 **test 邮件**（直接 sendRawEmail，不走 campaign 发送链路）才在发送前再做一次校验。

### 3.2 退订拦截（3 个入口 + 收件人筛选，顺序和行为不一致）

退订拦截**也不在 worker**，而是分散在收件人筛选层和三个发送入口里，且三个入口的"拦截形式"与"和计费检查的先后顺序"都不一致。

**（③）收件人筛选层（campaign 广播，最严格）**
- `buildRecipientWhereAsync` 直接在 SQL `WHERE` 加 `subscribed:true`，未订阅者根本不进队列：
  ```ts
  const baseWhere: Prisma.ContactWhereInput = {
    projectId,
    ...(campaign.type !== TemplateType.TRANSACTIONAL && {subscribed: true}),
  };
  ```
- 位置：[apps/api/src/services/CampaignService.ts#L866](apps/api/src/services/CampaignService.ts#L866)（L873 是 TRANSACTIONAL 类型豁免分支）。
- 事务型 campaign 不做订阅过滤，发给所有联系人——这是设计行为。

**（⑤a）sendTransactionalEmail：抛异常，先退订后计费**
- 仅在用 **MARKETING 模板**时才检查订阅（纯事务邮件不拦）。
- 未订阅时 **throw 400 HttpException**，**不写 Email 记录**。
- **顺序：先退订检查（L62） → 后 `checkLimit`（L78）**。未订阅不会占用计费配额。
- 位置：[apps/api/src/services/EmailService.ts#L62-L82](apps/api/src/services/EmailService.ts#L62-L82)。

**（⑤b）sendWorkflowEmail：写 FAILED 占位，先后顺序有分支**
- 条件：`sourceType !== TRANSACTIONAL && !params.recipientEmail`。
  - `sourceType !== TRANSACTIONAL`：事务模板不拦。
  - `!params.recipientEmail`：**CUSTOM 收件人（自定义邮件地址）跳过退订检查**——这类收件人"不在联系人列表中"（代码注释：`Custom recipient emails also bypass subscription checks (they're not in the contact list)`），由用户自行保证合规。CUSTOM 收件人通过 header `X-Plunk-Recipient-Override` 写入后实际发送给目标地址，`contactId` 仍保留原执行联系人用于追踪。
- 未订阅时**写一条 `status=FAILED, error='Contact is unsubscribed from marketing emails'` 的 Email 记录**，然后 return——后续计费和入队都不执行。
- **顺序：先退订检查/写占位（L200-L234） → 后 `checkLimit`（L237）**。FAILED 占位**不计费**、不 incrementUsage、不入队。
- CUSTOM 收件人路径：`recipientConfig.type === 'CUSTOM'` → `recipientEmail = customEmail` → 传给 sendWorkflowEmail → 条件 `!params.recipientEmail` 为 false → **绕过退订**；但仍会执行计费检查、落 `PENDING` 记录、入队发送。
- 位置：
  - 入口逻辑 [apps/api/src/services/EmailService.ts#L200-L284](apps/api/src/services/EmailService.ts#L200-L284)；
  - CUSTOM 收件人的来源（工作流 step 配置）[apps/api/src/services/WorkflowExecutionService.ts#L538-L585](apps/api/src/services/WorkflowExecutionService.ts#L538-L585)（L569/L584 是 recipient 选择与传递）。

**（⑤c）sendCampaignEmail：信任收件人筛选，不做订阅检查**
- **完全不做** `contact.subscribed` 检查——campaign 路径的 `buildRecipientWhereAsync`（③）已经在 SQL 层过滤掉未订阅者，入口方法信任上游。
- 顺序：`checkLimit`（L138）→ `prisma.email.create(PENDING)` → `incrementUsage` → `queueEmail`。没有退订分支，直接到计费。
- 位置：[apps/api/src/services/EmailService.ts#L119-L177](apps/api/src/services/EmailService.ts#L119-L177)。

> ⚠️ 注意上述"退订/计费顺序"的不对称性：三个入口中只有 campaign 入口**先计费后落库**，因为它在入口层信任上游筛选；transactional 先退订后计费（不订阅的不扣费）；workflow 先写 FAILED 占位后计费（占位不计费）。

**退订的触发源**（硬退信/投诉）都在 [apps/api/src/controllers/Webhooks.ts](apps/api/src/controllers/Webhooks.ts) 的 SNS 回执链路：
- SNS `Bounce` 且 `bounceType==='Permanent'` → `subscribed=false`（L374-L396）；
- `Complaint` → `subscribed=false`（L431-L447）；
- `bounceType==='Transient'` 软退信**不退订**，仅记录事件（L397-L408）。

### 3.3 计费闸门（⑥）——入口层入队前做

[apps/api/src/services/BillingLimitService.ts#L149](apps/api/src/services/BillingLimitService.ts#L149) 的 `checkLimit` 在每个入口写 PENDING Email **前**调用（workflow 是在退订占位**之后**调用）。超限抛 429，不写 Email、不入队。`incrementUsage` 在写 Email 记录之后、入队之前调用。

**与退订检查的先后顺序总结表**：

| 入口方法 | 退订检查存在？ | 退订拦截形式 | 与计费 checkLimit 的先后 |
| --- | --- | --- | --- |
| `sendTransactionalEmail` | 仅 marketing 模板 | throw 400，不落库 | **先退订，后计费** |
| `sendCampaignEmail` | 否（信任③） | - | **直接计费** |
| `sendWorkflowEmail` | 非事务模板 + 非 CUSTOM 收件人 | 写 FAILED 占位（不计费不入队） | **先退订占位，后计费** |
| campaign 的 recipients 筛选（③） | 是（SQL WHERE） | 不入 sendCampaignEmail | —（发生在更早阶段） |

### 3.4 项目级降级（disable = 整项目拉黑）——worker 阶段⑦

worker 在取出邮件后检查 `project.disabled` → 直接 `FAILED`（[apps/api/src/jobs/email-processor.ts#L103](apps/api/src/jobs/email-processor.ts#L103)）。`disabledReason` 三类来源：
1. **邮件声誉超阈**（`EMAIL_REPUTATION`）：硬退信/投诉触发 [apps/api/src/services/SecurityService.ts#L272](apps/api/src/services/SecurityService.ts#L272) → `disableProject`（L654）。
2. **钓鱼检测**（`PHISHING_DETECTED`）：worker ⑧处采样命中 → `disableProjectForPhishing`（L946）。
3. **支付失败**（`PAYMENT_FAILED`）：Stripe `invoice.payment_failed`（[apps/api/src/controllers/Webhooks.ts#L591](apps/api/src/controllers/Webhooks.ts#L591)）。

自托管可用 `AUTO_PROJECT_DISABLE=false` 关闭自动拉黑（[apps/api/src/app/constants.ts#L125](apps/api/src/app/constants.ts#L125)）。

### 3.5 钓鱼采样拦截——worker 阶段⑧

worker 发送前按采样率（默认 10%）调用 `checkPhishingContent`，命中则拉黑项目（[apps/api/src/jobs/email-processor.ts#L185](apps/api/src/jobs/email-processor.ts#L185)）。

### 3.6 发送失败的降级：重试而非换 Provider

worker catch 后把邮件置 `FAILED` 并 `rethrow` 触发 BullMQ 重试（[apps/api/src/jobs/email-processor.ts#L266](apps/api/src/jobs/email-processor.ts#L266)），默认 3 次、指数退避起始 2s（[apps/api/src/services/QueueService.ts#L49](apps/api/src/services/QueueService.ts#L49)）。**重试仍是同一个 SES，没有二级 Provider 兜底**。

---

## 4. 发送服务入口 vs 发送 Worker 职责差异（v3 重申）

两者都包含"渲染 + sendRawEmail + 落 SENT + trackEvent"，但闸门职责**几乎互补**：

| 职责 | `EmailService.sendEmail`（仅测试用） | `email-processor` worker（生产路径） |
| --- | --- | --- |
| `status===PENDING` 检查 | ✅ | ✅ |
| `project.disabled → FAILED` | ❌ | ✅（⑦） |
| 退订拦截 `contact.subscribed` | ✅（L311） | ❌（改在收件人筛选③ / 入口⑤a⑤b⑤c） |
| `verifyEmailDomain` 发件域校验 | ✅（L331） | ❌（改在资源创建①② / 发送入口④⑨） |
| 钓鱼采样 `checkPhishingContent` | ❌ | ✅（⑧） |
| `MeterService` 计量记账 | ❌ | ✅ |
| `CampaignService.finalizeIfDone` | ❌ | ✅ |
| 失败 rethrow 触发重试 | ✅(throw) | ✅(throw) |
| 生产调用方 | 仅测试 | BullMQ worker |

> 可以把 worker 理解为"**信任入口层已完成所有前置校验**的最终执行器"：worker 不做退订/域校验，因为 campaign 的 SQL 筛选、template/campaign 创建时的 verifyDomain、workflow 入口的退订占位，都已保证合法性。worker 只处理那些"即使入口层也无法提前预知"的情况（项目被动态拉黑、内容钓鱼），以及 SES 调用本身。

---

## 5. 队列协作（Queue Collaboration）

### 5.1 队列拓扑与邮件链路

10 个 BullMQ 队列共享一条 Redis 连接（[apps/api/src/services/QueueService.ts](apps/api/src/services/QueueService.ts)），由 [apps/api/src/jobs/worker.ts#L25](apps/api/src/jobs/worker.ts#L25) 统一拉起。邮件相关协作链：

```
scheduled ─(到点)─► CampaignService.startSending ─► campaign 队列(分批)
campaign  ─(每批 N 联系人)─► sendCampaignEmail ─► email 队列
workflow  ─(SEND_EMAIL 步，recipient=CUSTOM 绕过退订)─► sendWorkflowEmail ─► email 队列
email     ─(worker)─► AWS SES ─► SNS 回执 ─► /webhooks/sns（退订/声誉回流）
```

### 5.2 campaign ↔ email 链式批处理

- campaign 队列并发 5（[apps/api/src/jobs/campaign-processor.ts#L13](apps/api/src/jobs/campaign-processor.ts#L13)）。
- `processBatch`（[apps/api/src/services/CampaignService.ts#L513](apps/api/src/services/CampaignService.ts#L513)）游标分页，每联系人 `sendCampaignEmail → email 队列`；当前批处理完**自链投递下一批**（[apps/api/src/services/CampaignService.ts#L587](apps/api/src/services/CampaignService.ts#L587)）。
- 末批 reconcile `totalRecipients` 后 `finalizeIfDone`（[apps/api/src/services/CampaignService.ts#L617](apps/api/src/services/CampaignService.ts#L617)）推进 campaign 状态，`SENT` 与 `FAILED` 都算终态，避免部分失败时卡 `SENDING`。

### 5.3 email 队列限速/并发（由 SES 配额推导）

[apps/api/src/jobs/email-processor.ts#L30](apps/api/src/jobs/email-processor.ts#L30)：
- 限速优先级：`EMAIL_RATE_LIMIT_PER_SECOND`(env) > `SES getSendQuota()` > 安全默认 14/s；
- 并发 = `rate * 0.5`，clamp 到 `[5, EMAIL_WORKER_MAX_CONCURRENCY=50]`（`deriveWorkerConcurrency` L62）；
- BullMQ `limiter:{max, duration:1000}` 令牌桶（L284）。

### 5.4 项目拉黑时的跨队列级联清理

`cancelAllProjectJobs`（[apps/api/src/services/QueueService.ts#L545](apps/api/src/services/QueueService.ts#L545)）扫 `scheduled/email/campaign/workflow` 的 `waiting|delayed` 任务比对 `projectId` 后 `remove()`；把 `PENDING` 邮件置 `FAILED`；对 `SENDING` 中的 campaign reconcile 并 `finalizeIfDone` 收尾。声誉/钓鱼 disable 都会调用它（[SecurityService#L694](apps/api/src/services/SecurityService.ts#L694)、[SecurityService#L983](apps/api/src/services/SecurityService.ts#L983)）。

### 5.5 运维协作

`pauseAll/resumeAll`（[apps/api/src/services/QueueService.ts#L480](apps/api/src/services/QueueService.ts#L480)，维护期整体暂停/恢复）、`cleanOldJobs`（[apps/api/src/services/QueueService.ts#L516](apps/api/src/services/QueueService.ts#L516)，completed 24h、failed 7 天、meter 30 天）、`getStats`（[apps/api/src/services/QueueService.ts#L438](apps/api/src/services/QueueService.ts#L438)，聚合计数）。

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

定义集中在 [apps/api/src/app/constants.ts](apps/api/src/app/constants.ts)。

---

## 7. 一句话总结（v3 修正版）

- **选路** = 多入口归一到单一 `email` 队列体系（正常邮件写 PENDING 入队）+ BullMQ `priority` 按来源类型排序 + 出站仅 SES（按 tracking 选 Configuration Set）。
- **入口归一的例外**：`sendWorkflowEmail` 对未订阅营销联系人写 `FAILED` 占位后直接 return，**不写 PENDING、不计费、不入队**；transactional 未订阅则 throw 400 不写记录；campaign 未订阅者在 SQL 预筛选阶段就被剔除，到不了入口。
- **发件域校验**不是发送时做：campaign/template **创建/更新时 controller 层**校验（①②），只有 `/v1/send` 直发送（④）和 campaign 测试邮件（⑨）才在发送前补做；**正式广播链路信任创建期校验，不复检**。
- **退订拦截三个入口不一致**：campaign 走 SQL WHERE 预筛选（③），transactional 先退订后计费（抛 400），workflow 先退订写 FAILED 占位后 return（占位不计费不入队）+ **CUSTOM 收件人条件 `!params.recipientEmail` 绕过退订**；sendCampaignEmail 信任上游筛选，不做二次检查。
- **Worker 职责**（与 sendEmail 互补）= disabled 拦截 + 钓鱼采样 + 渲染发送 + 计量 + campaign 收尾 + 重试；**信任**入口层/创建期已做完退订与域校验。
- `EmailService.sendEmail` 是与 worker 并存的另一套发送逻辑，但**仅测试调用**，不参与生产发送。
- **降级无二级 Provider**，失败只靠 BullMQ 3 次重试；项目拉黑时跨队列级联清理。
