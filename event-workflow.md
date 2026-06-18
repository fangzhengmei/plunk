# Event → Workflow 完整执行脉络

本文档基于代码梳理事件从入站到触发、规则匹配、节点调度、失败恢复的完整流程。

---

## 1. 总体架构图

```
                    ┌──────────────────────────────────────────────┐
                    │              事件入站入口                      │
                    ├──────────────────────────────────────────────┤
                    │  • /v1/track        (Actions API, 公钥)       │
                    │  • /events/track    (Events API, JWT)        │
                    │  • /webhooks/sns    (AWS SES 邮件回调)        │
                    │  • /webhooks/incoming/stripe (支付回调)       │
                    │  • 内部服务调用     (e.g. UPDATE_CONTACT)    │
                    └────────────────────┬─────────────────────────┘
                                         │
                                         ▼
                    ┌──────────────────────────────────────────────┐
                    │         EventService.trackEvent()             │
                    │  1. 写入 Event 表（持久化）                    │
                    │  2. triggerWorkflows() → 触发新 workflow      │
                    │  3. handleEvent()      → 唤醒 WAIT_FOR_EVENT  │
                    └─────────┬───────────────────┬────────────────┘
                              │                   │
                              ▼                   ▼
              ┌───────────────────────┐   ┌──────────────────────────┐
              │ triggerWorkflows()    │   │ handleEvent()            │
              │ • Redis 缓存工作流     │   │ • 查找 WAITING 的步骤     │
              │ • 匹配 eventName       │   │ • 匹配 eventName         │
              │ • 检查 re-entry 规则   │   │ • 取消超时 job           │
              │ • 创建 Execution       │   │ • 继续下一步             │
              └───────────┬───────────┘   └────────────┬─────────────┘
                          │                            │
                          ▼                            ▼
              ┌─────────────────────────────────────────────────────┐
              │     WorkflowExecutionService.processStepExecution()  │
              │  • 状态校验 (RUNNING/WAITING, project/workflow)      │
              │  • 创建/更新 StepExecution                           │
              │  • executeStep() → 按 step.type 分派                │
              │  • processNextSteps() → 按 transition 推进           │
              └──────────────────────┬──────────────────────────────┘
                                     │
                    ┌────────────────┴────────────────┐
                    │         步骤类型分派              │
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
```

---

## 2. 事件入站（Event Ingestion）

事件有 5 个主要入站渠道，最终都汇聚到 `EventService.trackEvent()`。

### 2.1 入站渠道对比

| 渠道 | 文件 | 鉴权 | 典型事件 |
|------|------|------|----------|
| `/v1/track` | [Actions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/controllers/Actions.ts#L49-L102) | Public API Key | 自定义业务事件（user.signup, purchase.completed 等） |
| `/events/track` | [Events.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/controllers/Events.ts#L14-L28) | JWT + 邮箱验证 | 后台手动追踪事件 |
| `/webhooks/sns` | [Webhooks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/controllers/Webhooks.ts#L36-L477) | SNS 签名验证 | email.sent, email.delivered, email.opened, email.clicked, email.bounce, email.complaint, email.received |
| `/webhooks/incoming/stripe` | [Webhooks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/controllers/Webhooks.ts#L483-L789) | Stripe 签名 | 支付相关（不触发 workflow，仅内部账务） |
| 内部服务调用 | [WorkflowExecutionService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1070-L1076) | 内部调用 | contact.subscribed / contact.unsubscribed（UPDATE_CONTACT 步骤触发） |

### 2.2 `/v1/track` 公共 API 流程

[Actions.track()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/controllers/Actions.ts#L49-L102) 是业务方最常用的入口：

1. **Zod 校验** 请求体（event, email, data 等）
2. **保留事件检查**：`EventService.isReservedEvent()` 阻止手动追踪 `email.*`、`contact.*`、`segment.*.entry/exit`
3. **联系人 upsert**：`ContactService.upsert()` - 仅将 persistent=true 的字段写入 Contact.data
4. **事件追踪**：`EventService.trackEvent()` - 将完整 data（含 non-persistent）传入作为 workflow context

### 2.3 SNS Webhook 流程

[Webhooks.receiveSNSWebhook()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/controllers/Webhooks.ts#L36-L477) 处理两类 SES 通知：

- **SubscriptionConfirmation**：自动请求 SubscribeURL 确认订阅（有 SSRF 防护，仅允许 AWS SNS 域名）
- **Notification**：
  - 入站邮件（notificationType === 'Received'）：逐 recipient 查找已验证 domain → upsert contact → 创建 Email 记录 → 触发 `email.received` 事件
  - 出站邮件事件（Delivery/Open/Click/Bounce/Complaint）：按 SES messageId 查 Email → 更新 Email 状态 → 触发 `email.xxx` 事件

---

## 3. 事件处理核心：EventService.trackEvent()

[EventService.trackEvent()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/EventService.ts#L22-L47) 是所有事件的汇聚点，执行三件事：

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

  // 2. 触发监听该事件的新 workflow
  await this.triggerWorkflows(projectId, eventName, contactId, data);

  // 3. 唤醒正在 WAIT_FOR_EVENT 的已运行 workflow
  await WorkflowExecutionService.handleEvent(projectId, eventName, contactId, data);

  return event;
}
```

### 3.1 数据模型：Event 表

[schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/packages/db/prisma/schema.prisma#L600-L630)

| 字段 | 说明 |
|------|------|
| `name` | 事件名，如 "email.opened" |
| `data` | JSON 事件载荷，带 GIN 索引 |
| `projectId` | 所属项目 |
| `contactId` | 关联联系人（可选） |
| `emailId` | 关联邮件（可选） |

---

## 4. 规则匹配：triggerWorkflows()

[EventService.triggerWorkflows()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/EventService.ts#L351-L408) 负责找出哪些 workflow 需要被这个事件新启动。

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

**缓存失效时机**：workflow 被创建/更新/删除/启用/禁用时，`EventService.invalidateWorkflowCache()` 会 `redis.del(cacheKey)`。见 [keys.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/keys.ts#L71-L75)。

### 4.2 事件名匹配 + 重入检查

```typescript
for (const workflow of workflows) {
  if (workflow.triggerConfig?.eventName === eventName) {  // 规则匹配
    if (contactId) {
      await this.startWorkflowForContact(workflow.id, contactId, data);
    }
  }
}
```

[startWorkflowForContact()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/EventService.ts#L413-L489) 中的重入规则：

| `allowReentry` | 检查逻辑 |
|---|---|
| `false`（默认） | 联系人**任何状态**的 execution 已存在 → 跳过 |
| `true` | 仅当联系人有 **RUNNING** 状态的 execution → 跳过 |

### 4.3 启动 Workflow Execution

通过检查后：

1. 创建 `WorkflowExecution` 记录：`status=RUNNING`, `currentStepId=triggerStep.id`, `context=event.data`
2. 同步调用 `WorkflowExecutionService.processStepExecution(execution.id, triggerStep.id)` 开始执行

---

## 5. 节点调度：processStepExecution()

[WorkflowExecutionService.processStepExecution()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L47-L299) 是 workflow 执行引擎的核心。

### 5.1 前置校验（按顺序）

| 检查项 | 行为 | 代码行 |
|--------|------|--------|
| Execution 是否存在 | 不存在抛 404 | [L50-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L50-L76) |
| Execution 状态 | 非 RUNNING 且非 WAITING → 直接 return | [L80-L89](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L80-L89) |
| Project 是否 disabled | 是 → 标记 CANCELLED + exitReason | [L91-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L91-L105) |
| Workflow 是否 enabled | 禁用时**允许已开始的执行继续**（只阻止新 execution） | [L111-L116](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L111-L116) |

> ⚠️ **重要设计**：Workflow 被禁用时，已在运行中的 execution 会继续跑完，不会被中断。

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

  // 特殊情况：WAIT_FOR_EVENT 把自己标记为 WAITING，直接 return
  if (updatedStepExecution?.status === WAITING) return;

  // 特殊情况：DELAY 把 workflow 设为 WAITING 并已入队，直接 return
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

[executeStep()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L467-L502) 按 `step.type` 分派到 8 种处理函数。

### 6.1 TRIGGER（入口节点）

```typescript
// [WorkflowExecutionService.ts#L507-L521]
// 仅返回 { triggered: true, eventName, timestamp }，无副作用
```

### 6.2 SEND_EMAIL（发送邮件）

[executeSendEmail()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L526-L593)

1. Zod 校验 step 配置（[WorkflowStepConfigSchemas.sendEmail](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/packages/shared/src/schemas/index.ts#L274-L294)）
2. 组装模板变量作用域：
   ```
   {
     id, email,                          // 联系人基础字段
     ...contactData,                     // Contact.data 自定义字段
     ...executionContext,                // WorkflowExecution.context（即触发事件的 data）
     data: contactData,                  // 兼容 {{data.fieldName}} 写法
     unsubscribeUrl, subscribeUrl, manageUrl  // 系统链接
   }
   ```
3. `renderTemplate()` 渲染 subject/body（支持 `{{var}}` 和 `{{var ?? default}}`，见 [template.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/packages/shared/src/template.ts)）
4. `EmailService.sendWorkflowEmail()` 发送（内部走 BullMQ `emailQueue`）

### 6.3 DELAY（等待时间）

[executeDelay()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L598-L668)

关键行为：**不是让线程 sleep，而是立即完成当前 step、把 workflow 置为 WAITING，通过 BullMQ 延迟调度下一步**。

```
1. StepExecution → COMPLETED（记录 resumeAt）
2. WorkflowExecution → WAITING
3. 找第一条 outTransition，把 toStep 入队 workflowQueue，delay = 计算出的毫秒数
```

入队实现见 [QueueService.queueWorkflowStep()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/QueueService.ts#L230-L243)，BullMQ worker 在 [workflow-processor-queue.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/jobs/workflow-processor-queue.ts) 消费。

### 6.4 WAIT_FOR_EVENT（等待特定事件）

[executeWaitForEvent()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L673-L712)

```
1. StepExecution → WAITING，executeAfter = timeoutDate
2. WorkflowExecution → WAITING
3. 如果配置了 timeout（秒），入队超时 job：QueueService.queueWorkflowTimeout()
```

**事件到达时的唤醒**：`handleEvent()` 在 [WorkflowExecutionService.ts#L401-L462](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L401-L462)

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

**超时触发**：`processTimeout()` 在 [WorkflowExecutionService.ts#L305-L396](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L305-L396)，BullMQ 超时 job 到期后调用：

- 检查 step 是否仍为 WAITING（事件可能刚好到达）
- 标记 COMPLETED，output = `{ timedOut: true, eventName }`
- 找 `branch: 'timeout'` 或 `fallback: true` 的 transition；没有则走第一条；都没有则 complete workflow

### 6.5 CONDITION（条件分支）

[executeCondition()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L717-L791)

支持两种模式：

**1) Legacy 二元模式（if/else）**：返回 `{ branch: 'yes' | 'no', ... }`

**2) Multi 模式（switch/case）**：遍历 branches，第一个匹配的返回 `{ branch: branch.id, matchedBranch: branch.name }`；无匹配返回 `branch: 'default'`

**字段解析**：支持点号路径，作用域为：
```typescript
{
  contact: { email, subscribed },
  data: contactData,           // Contact.data
  workflow: executionContext,  // 触发事件的 data
  event: executionContext,     // 同上（别名）
}
```
兼容 legacy `contact.data.plan` → 自动转换为 `data.plan`。见 [resolveField()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1181-L1202)。

**支持的运算符**（[evaluateCondition()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1207-L1263)）：

| 运算符 | null/undefined 行为 |
|--------|---------------------|
| equals | 可以匹配 null/undefined |
| notEquals | false（不存在不匹配） |
| contains / notContains | false |
| greaterThan / lessThan / >= / <= | false |
| exists / notExists | 按字面判断 |

### 6.6 EXIT（终止工作流）

[executeExit()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L796-L825)：将 WorkflowExecution 标记为 `EXITED`，写入 `exitReason`。

### 6.7 WEBHOOK（调用外部接口）

[executeWebhook()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L920-L1006)

**SSRF 防护**（[safeFetch()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L871-L908)）：
- 仅允许 http/https 协议
- DNS 解析后校验 IP 不在内网/回环/链路本地/共享地址/云 metadata 范围（[isPrivateIp()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L831-L865)）
- 手动跟随 3xx 重定向，每跳重新做 DNS + IP 校验
- 最多 5 跳，10 秒超时

**变量渲染**：URL、header values、body 全部经过 `renderTemplate()` / `renderJsonTemplate()`，`method` 不渲染。作用域比 SEND_EMAIL 多一个 `event` 命名空间（即触发事件的 data）。

### 6.8 UPDATE_CONTACT（更新联系人）

[executeUpdateContact()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1033-L1085)

- 合并 `updates` 到 Contact.data
- 处理 `subscriptionAction`（subscribe/unsubscribe）
- 若订阅状态改变，会额外触发 `contact.subscribed` / `contact.unsubscribed` 事件（递归进入 EventService.trackEvent）

---

## 7. Transition 与下一步调度（processNextSteps）

[processNextSteps()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1090-L1168)

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

见 [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/packages/db/prisma/schema.prisma#L338-L511)。

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

[QueueService](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/QueueService.ts#L73-L84) 中 workflowQueue 配置：

```typescript
defaultJobOptions: {
  attempts: 3,                                   // 最多重试 3 次
  backoff: { type: 'exponential', delay: 2000 }, // 指数退避：2s, 4s, 8s
  removeOnComplete: 1000,
  removeOnFail: 5000,
}
```

worker 并发数为 10，见 [workflow-processor-queue.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/jobs/workflow-processor-queue.ts#L33)。

### 9.2 步骤级 try-catch

[processStepExecution() catch 块](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts#L254-L298)：

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

### 9.3 Worker 优雅关闭

[worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/jobs/worker.ts#L86-L121) 监听 SIGINT / SIGTERM / uncaughtException / unhandledRejection，对每个 worker 调用 `worker.close()` 等待当前 job 完成后退出。

### 9.4 项目禁用时的清理

[QueueService.cancelAllProjectJobs()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/QueueService.ts#L545-L640)：

- 扫描 scheduled/email/campaign/workflow 四个队列中 pending/delayed job
- 校验属于该 project 后移除
- 将仍处于 PENDING 的 Email 标记为 FAILED
- 将处于 SENDING 的 Campaign finalize

同时 `processStepExecution()` 每次执行前检查 `project.disabled`，是则把 execution 标记为 CANCELLED。

---

## 10. 关键文件索引

| 功能 | 文件 |
|------|------|
| 核心执行引擎 | [WorkflowExecutionService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowExecutionService.ts) |
| 事件服务 + workflow 触发 | [EventService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/EventService.ts) |
| Workflow CRUD + 手动启动 | [WorkflowService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/WorkflowService.ts) |
| BullMQ 队列管理 | [QueueService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/QueueService.ts) |
| Workflow Worker | [workflow-processor-queue.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/jobs/workflow-processor-queue.ts) |
| Worker 总入口 | [worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/jobs/worker.ts) |
| 公共 API：/v1/track | [Actions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/controllers/Actions.ts) |
| 内部 API：/events/track | [Events.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/controllers/Events.ts) |
| SNS/SES + Stripe Webhook | [Webhooks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/controllers/Webhooks.ts) |
| Step Config Zod Schemas | [schemas/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/packages/shared/src/schemas/index.ts) |
| 模板变量渲染 | [template.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/packages/shared/src/template.ts) |
| 数据模型 | [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/packages/db/prisma/schema.prisma) |
| Redis Key 定义 | [keys.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/34-plunk/apps/api/src/services/keys.ts) |
