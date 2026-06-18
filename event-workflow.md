# Event → Workflow 完整执行脉络

本文档基于代码梳理事件从入站到触发、规则匹配、节点调度、失败恢复的完整流程。
所有路径均为仓库相对路径（基于 `apps/api` 等模块根）。

---

## 1. 总体架构图

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │                  会触发 workflow 的事件源                            │
   ├────────────────────────────┬────────────────────────────────────────┤
   │   外部 HTTP 入口            │   内部服务触发                         │
   │   ───────────────          │   ──────────────                       │
   │   • POST /v1/track         │   • ContactService (订阅状态变化)       │
   │     (Actions, 公钥)        │     → contact.subscribed/.unsubscribed  │
   │   • POST /events/track     │   • EmailService.sendEmail             │
   │     (Events, JWT)          │     → email.sent                       │
   │   • POST /webhooks/sns     │   • email-processor.ts (BullMQ worker)│
   │     (SES 回调)             │     → email.sent                       │
   │                            │   • EmailService.handleWebhookEvent    │
   │                            │     → contact.unsubscribed             │
   │                            │   • SegmentService (成员变动)           │
   │                            │     → segment.{slug}.entry/.exit       │
   │                            │   • WorkflowExecutionService           │
   │                            │     .executeUpdateContact              │
   └────────────┬───────────────┴───────────────────┬───────────────────┘
                │                                    │
                ▼                                    ▼
       ┌──────────────────────────────────────────────────────────┐
       │             EventService.trackEvent()                    │
       │  1. 持久化 Event 记录（写库）                              │
       │  2. triggerWorkflows() → 启动监听该事件的新 workflow      │
       │  3. handleEvent()      → 唤醒 WAIT_FOR_EVENT 等待者       │
       └─────────┬──────────────────────────────────┬──────────────┘
                 │                                  │
                 ▼                                  ▼
   ┌───────────────────────────┐    ┌──────────────────────────────────┐
   │ triggerWorkflows()         │    │ handleEvent()                    │
   │ • Redis 缓存 enabled 流    │    │ • 查询所有 WAITING 的            │
   │ • 匹配 triggerConfig.      │    │   WAIT_FOR_EVENT step executions │
   │   eventName === eventName  │    │ • 匹配 step.config.eventName      │
   │ • 检查 re-entry 规则       │    │ • 取消超时 job                   │
   │ • 创建 WorkflowExecution   │    │ • processNextSteps() 继续         │
   └──────────┬────────────────┘    └─────────────┬────────────────────┘
              │                                    │
              ▼                                    ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │       WorkflowExecutionService.processStepExecution()              │
   │  • 状态校验 (RUNNING/WAITING, project.disabled, workflow.enabled)  │
   │  • 创建/复用 StepExecution 记录                                    │
   │  • executeStep() 按 step.type 分派到 8 种处理器                    │
   │  • processNextSteps() 按 transition 推进                          │
   └─────────────────────────┬────────────────────────────────────────┘
                             │
              ┌──────────────┴───────────────────┐
              │      8 种 step 类型分派           │
              ├──────────────────────────────────┤
              │ TRIGGER        │ 入口占位        │
              │ SEND_EMAIL     │ 渲染+发邮件     │
              │ DELAY          │ 入队 BullMQ     │
              │ WAIT_FOR_EVENT │ 等待+超时       │
              │ CONDITION      │ 条件分支        │
              │ EXIT           │ 终止工作流      │
              │ WEBHOOK        │ SSRF-safe 调用  │
              │ UPDATE_CONTACT │ 更新联系人      │
              └──────────────────────────────────┘


   ┌──────────────────────────────────────────────────────────────────────┐
   │   不触发 workflow 的入口（仅内部账务/状态更新）                       │
   ├──────────────────────────────────────────────────────────────────────┤
   │   • POST /webhooks/incoming/stripe  (Webhooks.receiveStripeWebhook)  │
   │     → 仅更新 Project.customer/subscription/disabled 状态              │
   │     → 发 Ntfy 通知，发邮件给项目成员                                   │
   │     → 全程不调用 EventService.trackEvent，不进入事件系统                │
   └──────────────────────────────────────────────────────────────────────┘
```

---

## 2. 事件入站入口：会启动 workflow 的路径

所有真正进入事件系统的入口都最终汇聚到 `EventService.trackEvent()`。

### 2.1 外部 HTTP 入口

| HTTP 路径 | Controller 方法 | 鉴权 | 触发的事件名 |
|-----------|-----------------|------|--------------|
| `POST /v1/track` | [Actions.track](apps/api/src/controllers/Actions.ts#L49-L102) | Public API Key | 业务方自定义事件（如 `user.signup`, `purchase.completed`） |
| `POST /events/track` | [Events.track](apps/api/src/controllers/Events.ts#L14-L28) | JWT + 邮箱验证 | 后台手动追踪，事件名由请求体 `name` 字段指定 |
| `POST /webhooks/sns` | [Webhooks.receiveSNSWebhook](apps/api/src/controllers/Webhooks.ts#L36-L477) | SNS 签名验证 | `email.delivery` / `email.open` / `email.click` / `email.bounce` / `email.complaint` / `email.received` |

#### 2.1.1 `/v1/track` 公共 API 流程

[Actions.track()](apps/api/src/controllers/Actions.ts#L49-L102) 是业务方最常用入口：

1. **Zod 校验**请求体（event, email, data 等），见 [ActionSchemas.track](packages/shared/src/schemas/index.ts#L453-L459)
2. **保留事件检查**：[EventService.isReservedEvent()](apps/api/src/services/EventService.ts#L327-L345) 阻止手动追踪 `email.*`、`contact.subscribed`、`contact.unsubscribed`、`segment.*.entry/.exit`
3. **联系人 upsert**：[ContactService.upsert()](apps/api/src/services/ContactService.ts#L273-L339) — 仅将 persistent=true 的字段写入 `Contact.data`
4. **事件追踪**：[EventService.trackEvent()](apps/api/src/services/EventService.ts#L22-L47) — 完整 data（含 non-persistent）作为 workflow context

#### 2.1.2 SNS Webhook 流程

[Webhooks.receiveSNSWebhook()](apps/api/src/controllers/Webhooks.ts#L36-L477) 处理两类 SES 通知：

- **SubscriptionConfirmation**：仅校验 SubscribeURL 域名（限制 `sns.<region>.amazonaws.{com|eu}`），fetch 确认订阅，**不产生事件**
- **Notification**：
  - **入站邮件**（`body.notificationType === 'Received'`）：逐 recipient 查已验证 domain → upsert contact → 创建 Email 记录 → [EventService.trackEvent()](apps/api/src/controllers/Webhooks.ts#L265-L271) 触发 `email.received` 事件
  - **出站邮件事件**（Delivery/Open/Click/Bounce/Complaint）：按 SES messageId 查 Email → 更新 Email 状态 → [EventService.trackEvent()](apps/api/src/controllers/Webhooks.ts#L461) 触发 `email.{eventType.toLowerCase()}` 事件
  - Bounce/Complaint 还会**额外**修改 Contact.subscribed = false，但**不**直接调用 trackEvent 触发 `contact.unsubscribed`（仅触发 `email.bounce` / `email.complaint` 事件）

### 2.2 内部服务触发的事件

这些不是 HTTP 入口，而是业务流程中由其他服务调用 `EventService.trackEvent()` 触发的事件：

| 服务 / 文件 | 触发条件 | 事件名 |
|------------|---------|--------|
| [ContactService.update](apps/api/src/services/ContactService.ts#L228-L235) | 联系人订阅状态从 false→true 或 true→false | `contact.subscribed` / `contact.unsubscribed` |
| [ContactService.upsert](apps/api/src/services/ContactService.ts#L304-L311) | 同上（upsert 路径） | `contact.subscribed` / `contact.unsubscribed` |
| [ContactService.subscribe / unsubscribe](apps/api/src/services/ContactService.ts#L432-L452) | 单个订阅/退订 | `contact.subscribed` / `contact.unsubscribed` |
| [ContactService.trackEventsSequentially](apps/api/src/services/ContactService.ts#L864-L885) | 批量订阅/退订（bulkSubscribe/bulkUnsubscribe 调用） | `contact.subscribed` / `contact.unsubscribed` |
| [EmailService.sendEmail](apps/api/src/services/EmailService.ts#L290-L441) | 服务内直接发送邮件成功后 | `email.sent` |
| [email-processor.ts](apps/api/src/jobs/email-processor.ts#L250-L261) | BullMQ worker 成功发送邮件后 | `email.sent` |
| [EmailService.handleWebhookEvent](apps/api/src/services/EmailService.ts#L462-L531) | bounce/complaint 时修改订阅状态 | `contact.unsubscribed`（含 reason 字段） |
| [SegmentService](apps/api/src/services/SegmentService.ts#L569-L614) | 动态 segment 重算成员，contact 进入/离开 segment | `segment.{slug}.entry` / `segment.{slug}.exit` |
| [WorkflowExecutionService.executeUpdateContact](apps/api/src/services/WorkflowExecutionService.ts#L1069-L1076) | workflow 内 UPDATE_CONTACT 步骤改变订阅状态 | `contact.subscribed` / `contact.unsubscribed` |

> ⚠️ **递归风险点**：`UPDATE_CONTACT` 步骤触发 `contact.subscribed` 事件，该事件可能又触发监听该事件的其他 workflow，形成链式调用。`ContactService` 触发的 `contact.subscribed` 同理可能触发新的 workflow execution。

### 2.3 不触发 workflow 的入口

**`POST /webhooks/incoming/stripe`**（[Webhooks.receiveStripeWebhook](apps/api/src/controllers/Webhooks.ts#L483-L789)）是计费回调，**完全不进入事件系统**：

| Stripe 事件类型 | 处理行为 |
|----------------|---------|
| `checkout.session.completed` | 更新 Project.customer / subscription；refund 卡验证费用；可选应用 SWITCH promo |
| `invoice.paid` | 若 project 因 `PAYMENT_FAILED` 被禁用则重新启用 |
| `invoice.payment_failed` | （非首次订阅时）禁用 project，置 `disabledReason = PAYMENT_FAILED`，发邮件通知成员 |
| `customer.subscription.deleted` | 清空 Project.subscription |
| `customer.subscription.updated` | 仅记录日志 + Ntfy 通知 |
| `radar.early_fraud_warning.created` | refund charge，将 card fingerprint/email 加入 Stripe Radar blocklist |

整个过程**不调用 `EventService.trackEvent()`**，不写入 Event 表，不会启动或唤醒任何 workflow。它的副作用仅限于 `Project` 表的状态字段和外部 Stripe API。

> 注意：`EmailService.handleWebhookEvent()` 末尾会直接 `prisma.event.create()` 写入 `email.{eventType}`，但没有调用 `EventService.trackEvent()`，因此这条直接写库路径不会启动或唤醒 workflow；真正能触发 workflow 的邮件状态事件来自 SNS webhook 的 `EventService.trackEvent()` 调用。

---

## 3. 事件处理核心：EventService.trackEvent()

[EventService.trackEvent()](apps/api/src/services/EventService.ts#L22-L47) 是所有事件源的汇聚点，执行三件事：

```typescript
public static async trackEvent(
  projectId: string,
  eventName: string,
  contactId?: string,
  emailId?: string,
  data?: Record<string, unknown>,
): Promise<Event> {
  // 1. 持久化事件
  const event = await prisma.event.create({ ... });

  // 2. 触发监听该事件的新 workflow（启动新 execution）
  await this.triggerWorkflows(projectId, eventName, contactId, data);

  // 3. 唤醒正在 WAIT_FOR_EVENT 的已运行 workflow
  await WorkflowExecutionService.handleEvent(projectId, eventName, contactId, data);

  return event;
}
```

### 3.1 数据模型：Event 表

[schema.prisma](packages/db/prisma/schema.prisma#L600-L630)

| 字段 | 说明 |
|------|------|
| `name` | 事件名，如 `email.opened`、`user.signup` |
| `data` | JSON 事件载荷，带 GIN 索引 |
| `projectId` | 所属项目 |
| `contactId` | 关联联系人（可选，project-level 事件为 null） |
| `emailId` | 关联邮件（仅 `email.*` 系列事件） |

---

## 4. 规则匹配：triggerWorkflows()

[EventService.triggerWorkflows()](apps/api/src/services/EventService.ts#L351-L408) 负责找出哪些 workflow 需要被这个事件**新启动**。

### 4.1 工作流缓存策略

```typescript
const cacheKey = Keys.Workflow.enabled(projectId);  // "workflows:enabled:${projectId}"
let workflows;

// 先查 Redis 缓存
const cached = await redis.get(cacheKey);
if (cached) {
  workflows = JSON.parse(cached);
}

// 缓存未命中则查库：enabled=true 且 triggerType='EVENT' 的工作流
if (!workflows) {
  workflows = await prisma.workflow.findMany({
    where: { projectId, enabled: true, triggerType: 'EVENT' },
    include: { steps: { where: { type: 'TRIGGER' } } },
  });
  await redis.setex(cacheKey, 300, JSON.stringify(workflows));  // TTL 5 分钟
}
```

**缓存失效时机**：workflow 被创建/更新/删除/启用/禁用时，[EventService.invalidateWorkflowCache()](apps/api/src/services/EventService.ts#L53-L60) 会 `redis.del(cacheKey)`。Key 模板见 [keys.ts](apps/api/src/services/keys.ts#L71-L75)。

### 4.2 事件名匹配 + 重入检查

```typescript
for (const workflow of workflows) {
  if (workflow.triggerConfig?.eventName === eventName) {  // 精确字符串匹配
    if (contactId) {
      await this.startWorkflowForContact(workflow.id, contactId, data);
    } else {
      // project-level 事件无 contactId，不会启动 workflow
      signale.info(`[EVENT] Event ${eventName} triggered workflow ${workflow.id}, but no contact specified`);
    }
  }
}
```

> ⚠️ **关键限制**：即便 workflow 的 trigger 监听了某事件，若事件没有 `contactId`（如纯 project-level 事件），workflow 也**不会启动**。这是当前实现的一个隐含约束。

[startWorkflowForContact()](apps/api/src/services/EventService.ts#L413-L489) 中的重入规则：

| `allowReentry` | 检查逻辑 |
|---|---|
| `false`（默认） | 联系人**任何状态**的 execution 已存在 → 跳过 |
| `true` | 仅当联系人有 **RUNNING** 状态的 execution → 跳过 |

### 4.3 启动 Workflow Execution

通过检查后：

1. 创建 `WorkflowExecution` 记录：`status=RUNNING`, `currentStepId=triggerStep.id`, `context=event.data`
2. 同步调用 [WorkflowExecutionService.processStepExecution()](apps/api/src/services/WorkflowExecutionService.ts#L47-L299) 开始执行

---

## 5. 节点调度：processStepExecution()

[WorkflowExecutionService.processStepExecution()](apps/api/src/services/WorkflowExecutionService.ts#L47-L299) 是 workflow 执行引擎的核心。

### 5.1 前置校验（按顺序）

| 检查项 | 行为 | 代码行 |
|--------|------|--------|
| Execution 是否存在 | 不存在抛 404 | [L50-L76](apps/api/src/services/WorkflowExecutionService.ts#L50-L76) |
| Execution 状态 | 非 RUNNING 且非 WAITING → 直接 return | [L80-L89](apps/api/src/services/WorkflowExecutionService.ts#L80-L89) |
| Project 是否 disabled | 是 → 标记 CANCELLED + exitReason | [L91-L105](apps/api/src/services/WorkflowExecutionService.ts#L91-L105) |
| Workflow 是否 enabled | 禁用时**允许已开始的执行继续**（只阻止新 execution） | [L111-L116](apps/api/src/services/WorkflowExecutionService.ts#L111-L116) |

> ⚠️ **重要设计**：Workflow 被禁用时，已在运行中的 execution 会继续跑完，不会被中断。只有 `project.disabled` 才会立即取消运行中的 execution。

### 5.2 WAITING 恢复（DELAY 场景）

如果 execution 处于 `WAITING` 状态（DELAY 步骤通过 BullMQ 延迟触发的回调）：

```typescript
if (initialExecution.status === WorkflowExecutionStatus.WAITING) {
  // 状态改回 RUNNING，currentStepId 设置为待执行的 step
  await prisma.workflowExecution.update({
    where: { id: executionId },
    data: { status: WorkflowExecutionStatus.RUNNING, currentStepId: stepId },
  });
}
```

### 5.3 Step Execution 记录

查找或创建 `WorkflowStepExecution` 记录：

```typescript
let stepExecution = await prisma.workflowStepExecution.findFirst({
  where: { executionId, stepId, status: { in: [PENDING, RUNNING] } },
});

if (!stepExecution) {
  stepExecution = await prisma.workflowStepExecution.create({
    data: { executionId, stepId, status: RUNNING, startedAt: new Date() },
  });
}
```

### 5.4 执行步骤 → 推进下一步

```
try {
  result = await executeStep(step, execution, stepExecution);

  // 特殊情况 A：WAIT_FOR_EVENT 把自己标记为 WAITING，直接 return
  if (updatedStepExecution?.status === WAITING) return;

  // 特殊情况 B：DELAY 把 workflow 设为 WAITING 并已入队，直接 return
  if (!isResumingFromDelay && updatedExecution?.status === WAITING) return;

  // 正常完成：标记 step COMPLETED
  await prisma.workflowStepExecution.update({ status: COMPLETED, output: result });

  // 根据 transitions 决定下一步
  await processNextSteps(execution, step, result);
} catch (error) {
  // 标记 step 和 workflow FAILED，发 Ntfy 通知，重抛
}
```

---

## 6. 步骤类型详解（executeStep）

[executeStep()](apps/api/src/services/WorkflowExecutionService.ts#L467-L502) 按 `step.type` 分派到 8 种处理函数。

### 6.1 TRIGGER（入口节点）

[executeTrigger()](apps/api/src/services/WorkflowExecutionService.ts#L507-L521) 仅返回 `{ triggered: true, eventName, timestamp }`，无副作用。

### 6.2 SEND_EMAIL（发送邮件）

[executeSendEmail()](apps/api/src/services/WorkflowExecutionService.ts#L526-L593)

1. Zod 校验 step 配置（[WorkflowStepConfigSchemas.sendEmail](packages/shared/src/schemas/index.ts#L274-L294)）
2. 组装模板变量作用域：
   ```
   {
     id, email,                          // 联系人基础字段
     ...contactData,                     // Contact.data 自定义字段
     ...executionContext,                // WorkflowExecution.context（触发事件的 data）
     data: contactData,                  // 兼容 {{data.fieldName}} 写法
     unsubscribeUrl, subscribeUrl, manageUrl  // 系统链接
   }
   ```
3. [renderTemplate()](packages/shared/src/template.ts) 渲染 subject/body（支持 `{{var}}` 和 `{{var ?? default}}`）
4. `EmailService.sendWorkflowEmail()` 发送（内部走 BullMQ `emailQueue`）

### 6.3 DELAY（等待时间）

[executeDelay()](apps/api/src/services/WorkflowExecutionService.ts#L598-L668)

关键行为：**不是让线程 sleep，而是立即完成当前 step、把 workflow 置为 WAITING，通过 BullMQ 延迟调度下一步**。

```
1. StepExecution → COMPLETED（记录 resumeAt）
2. WorkflowExecution → WAITING
3. 找第一条 outTransition，把 toStep 入队 workflowQueue，delay = 计算出的毫秒数
```

入队实现见 [QueueService.queueWorkflowStep()](apps/api/src/services/QueueService.ts#L230-L243)，BullMQ worker 在 [workflow-processor-queue.ts](apps/api/src/jobs/workflow-processor-queue.ts) 消费。

### 6.4 WAIT_FOR_EVENT（等待特定事件）

[executeWaitForEvent()](apps/api/src/services/WorkflowExecutionService.ts#L673-L712)

```
1. StepExecution → WAITING，executeAfter = timeoutDate
2. WorkflowExecution → WAITING
3. 如果配置了 timeout（秒），入队超时 job：QueueService.queueWorkflowTimeout()
```

**事件到达时的唤醒**：[handleEvent()](apps/api/src/services/WorkflowExecutionService.ts#L401-L462)

```typescript
// 查询所有 WAITING 的 WAIT_FOR_EVENT step executions
const waitingExecutions = await prisma.workflowStepExecution.findMany({
  where: {
    status: WAITING,
    execution: { workflow: { projectId }, ...(contactId ? { contactId } : {}) },
    step: { type: 'WAIT_FOR_EVENT' },
  },
  include: { execution: { include: { contact, workflow } }, step: { include: { outgoingTransitions } } },
});

for (const stepExecution of waitingExecutions) {
  if (stepExecution.step.config?.eventName === eventName) {
    // 标记 StepExecution COMPLETED，记录事件数据
    // 取消超时 job：QueueService.cancelWorkflowTimeout(stepExecution.id)
    // 继续下一步：processNextSteps(execution, step, { eventReceived: true })
  }
}
```

**超时触发**：[processTimeout()](apps/api/src/services/WorkflowExecutionService.ts#L305-L396)，BullMQ 超时 job 到期后调用：

- 检查 step 是否仍为 WAITING（事件可能刚好到达）
- 标记 COMPLETED，output = `{ timedOut: true, eventName }`
- 找 `branch: 'timeout'` 或 `fallback: true` 的 transition；没有则走第一条；都没有则 complete workflow

### 6.5 CONDITION（条件分支）

[executeCondition()](apps/api/src/services/WorkflowExecutionService.ts#L717-L791)

支持两种模式：

**1) Legacy 二元模式（if/else）**：返回 `{ branch: 'yes' | 'no', ... }`

**2) Multi 模式（switch/case）**：遍历 branches，第一个匹配的返回 `{ branch: branch.id, matchedBranch: branch.name }`；无匹配返回 `branch: 'default'`

**字段解析**：[resolveField()](apps/api/src/services/WorkflowExecutionService.ts#L1181-L1202) 支持点号路径，作用域为：
```typescript
{
  contact: { email, subscribed },
  data: contactData,           // Contact.data
  workflow: executionContext, // 触发事件的 data
  event: executionContext,     // 同上（别名）
}
```
兼容 legacy `contact.data.plan` → 自动转换为 `data.plan`。

**支持的运算符**（[evaluateCondition()](apps/api/src/services/WorkflowExecutionService.ts#L1207-L1263)）：

| 运算符 | null/undefined 行为 |
|--------|---------------------|
| equals | 可以匹配 null/undefined |
| notEquals | false（不存在不匹配） |
| contains / notContains | false |
| greaterThan / lessThan / >= / <= | false |
| exists / notExists | 按字面判断 |

### 6.6 EXIT（终止工作流）

[executeExit()](apps/api/src/services/WorkflowExecutionService.ts#L796-L825)：将 WorkflowExecution 标记为 `EXITED`，写入 `exitReason`。

### 6.7 WEBHOOK（调用外部接口）

[executeWebhook()](apps/api/src/services/WorkflowExecutionService.ts#L920-L1006)

**SSRF 防护**（[safeFetch()](apps/api/src/services/WorkflowExecutionService.ts#L871-L908)）：
- 仅允许 http/https 协议
- DNS 解析后校验 IP 不在内网/回环/链路本地/共享地址/云 metadata 范围（[isPrivateIp()](apps/api/src/services/WorkflowExecutionService.ts#L831-L865)）
- 手动跟随 3xx 重定向，每跳重新做 DNS + IP 校验
- 最多 5 跳，10 秒超时

**变量渲染**：URL、header values、body 全部经过 `renderTemplate()` / [renderJsonTemplate()](apps/api/src/services/WorkflowExecutionService.ts#L1013-L1028)，`method` 不渲染。作用域比 SEND_EMAIL 多一个 `event` 命名空间（即触发事件的 data）。

### 6.8 UPDATE_CONTACT（更新联系人）

[executeUpdateContact()](apps/api/src/services/WorkflowExecutionService.ts#L1033-L1085)

- 合并 `updates` 到 Contact.data
- 处理 `subscriptionAction`（subscribe/unsubscribe）
- 若订阅状态改变，会额外触发 [EventService.trackEvent()](apps/api/src/services/WorkflowExecutionService.ts#L1069-L1076) 发出 `contact.subscribed` / `contact.unsubscribed` 事件（可能递归触发其他 workflow）

---

## 7. Transition 与下一步调度（processNextSteps）

[processNextSteps()](apps/api/src/services/WorkflowExecutionService.ts#L1090-L1168)

```
transitions = step.outgoingTransitions（按 priority asc 排序）

for transition of transitions:
  1. transition.condition == null → 无条件跟随，break
  2. CONDITION step 的分支匹配：condition.branch === stepResult.branch → break
  3. evaluateTransitionCondition()（当前未实现，恒返回 false）

若无匹配 transition → WorkflowExecution → COMPLETED
若匹配 → 更新 currentStepId，递归调用 processStepExecution()
```

> ⚠️ 递归调用是**同步**的（非入队），意味着同一个 workflow 的连续步骤会在同一个调用栈里顺序执行，直到遇到 DELAY / WAIT_FOR_EVENT / EXIT / 末尾。

---

## 8. 数据模型与状态机

### 8.1 核心表关系

```
Workflow (1) ──→ (N) WorkflowStep
      │                    │
      │                    ├──→ (N) WorkflowTransition (fromStep → toStep)
      │                    │
      │                    └──→ (N) WorkflowStepExecution
      │                              │
      └──→ (N) WorkflowExecution ───┘
                  │
                  └── Contact (N:1)
```

见 [schema.prisma](packages/db/prisma/schema.prisma#L338-L511)。

### 8.2 WorkflowExecution 状态

| 状态 | 说明 |
|------|------|
| RUNNING | 正在执行或待执行下一步 |
| WAITING | 等待 DELAY 到期或 WAIT_FOR_EVENT 事件到达 |
| COMPLETED | 正常走完所有步骤 |
| EXITED | 被 EXIT 步骤终止 |
| FAILED | 步骤执行抛出异常 |
| CANCELLED | 用户手动取消或项目被禁用 |

### 8.3 WorkflowStepExecution 状态

| 状态 | 说明 |
|------|------|
| PENDING | 待开始 |
| RUNNING | 执行中 |
| WAITING | WAIT_FOR_EVENT 等待中 |
| COMPLETED | 成功完成 |
| SKIPPED | 条件不满足跳过 |
| FAILED | 执行失败 |

---

## 9. 失败恢复与可靠性

### 9.1 BullMQ 队列重试策略

[QueueService](apps/api/src/services/QueueService.ts#L73-L84) 中 workflowQueue 配置：

```typescript
defaultJobOptions: {
  attempts: 3,                                   // 最多重试 3 次
  backoff: { type: 'exponential', delay: 2000 }, // 指数退避：2s, 4s, 8s
  removeOnComplete: 1000,
  removeOnFail: 5000,
}
```

worker 并发数为 10，见 [workflow-processor-queue.ts](apps/api/src/jobs/workflow-processor-queue.ts#L33)。

### 9.2 步骤级 try-catch

[processStepExecution() catch 块](apps/api/src/services/WorkflowExecutionService.ts#L254-L298)：

```typescript
catch (error) {
  // 1. StepExecution → FAILED，记录 error message
  await prisma.workflowStepExecution.update({ status: FAILED, error: error.message });

  // 2. WorkflowExecution → FAILED
  const failedExecution = await prisma.workflowExecution.update({ status: FAILED, completedAt: now });

  // 3. Ntfy 通知：workflow 名、项目名、联系人邮箱、错误信息
  await NtfyService.notifyWorkflowExecutionFailed(...);

  // 4. 重抛异常 → BullMQ 根据 attempts 决定是否重试
  throw error;
}
```

> ⚠️ **重试语义**：BullMQ 重试的是**整个 job**（即 `processStepExecution(executionId, stepId)`），不是单步骤。重试时 step 已被标记 FAILED，[L176-L209](apps/api/src/services/WorkflowExecutionService.ts#L176-L209) 的 findFirst 不会找到 PENDING/RUNNING 的 stepExecution，会创建新的 RUNNING 记录重新执行。这意味着 step 可能被多次执行——side-effect（如发邮件、调 webhook）需要业务方自行幂等。

### 9.3 Worker 优雅关闭

[worker.ts](apps/api/src/jobs/worker.ts#L86-L121) 监听 SIGINT / SIGTERM / uncaughtException / unhandledRejection，对每个 worker 调用 `worker.close()` 等待当前 job 完成后退出。

### 9.4 项目禁用时的清理

[QueueService.cancelAllProjectJobs()](apps/api/src/services/QueueService.ts#L545-L640)：

- 扫描 scheduled/email/campaign/workflow 四个队列中 pending/delayed job
- 校验属于该 project 后移除
- 将仍处于 PENDING 的 Email 标记为 FAILED
- 将处于 SENDING 的 Campaign finalize

同时 `processStepExecution()` 每次执行前检查 `project.disabled`，是则把 execution 标记为 CANCELLED。

---

## 10. 关键文件索引

| 功能 | 文件 |
|------|------|
| 核心执行引擎 | [WorkflowExecutionService.ts](apps/api/src/services/WorkflowExecutionService.ts) |
| 事件服务 + workflow 触发 | [EventService.ts](apps/api/src/services/EventService.ts) |
| Workflow CRUD + 手动启动 | [WorkflowService.ts](apps/api/src/services/WorkflowService.ts) |
| BullMQ 队列管理 | [QueueService.ts](apps/api/src/services/QueueService.ts) |
| Workflow Worker | [workflow-processor-queue.ts](apps/api/src/jobs/workflow-processor-queue.ts) |
| Worker 总入口 | [worker.ts](apps/api/src/jobs/worker.ts) |
| 邮件发送 Worker | [email-processor.ts](apps/api/src/jobs/email-processor.ts) |
| 公共 API：/v1/track | [Actions.ts](apps/api/src/controllers/Actions.ts) |
| 内部 API：/events/track | [Events.ts](apps/api/src/controllers/Events.ts) |
| SNS/SES + Stripe Webhook | [Webhooks.ts](apps/api/src/controllers/Webhooks.ts) |
| ContactService（订阅事件） | [ContactService.ts](apps/api/src/services/ContactService.ts) |
| EmailService（email.sent 等） | [EmailService.ts](apps/api/src/services/EmailService.ts) |
| SegmentService（segment entry/exit） | [SegmentService.ts](apps/api/src/services/SegmentService.ts) |
| Step Config Zod Schemas | [schemas/index.ts](packages/shared/src/schemas/index.ts) |
| 模板变量渲染 | [template.ts](packages/shared/src/template.ts) |
| 数据模型 | [schema.prisma](packages/db/prisma/schema.prisma) |
| Redis Key 定义 | [keys.ts](apps/api/src/services/keys.ts) |
