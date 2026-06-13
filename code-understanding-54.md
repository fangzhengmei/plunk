# Workflow 执行引擎代码理解

## 一、概述

Workflow 执行引擎是一个基于事件驱动的自动化工作流系统，支持多步骤、条件分支、延迟等待、外部 Webhook 调用等功能。核心设计围绕「事件触发 → 步骤执行 → 状态推进 → 完成/退出」的主链路展开，同时通过 BullMQ 队列处理异步延迟和超时场景。

**核心代码位置**：

| 模块 | 文件路径 |
|------|---------|
| 数据模型 | [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/db/prisma/schema.prisma#L338-L511) |
| Workflow 管理服务 | [WorkflowService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts) |
| 执行引擎核心 | [WorkflowExecutionService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts) |
| 事件触发服务 | [EventService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/EventService.ts) |
| 队列管理 | [QueueService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/QueueService.ts) |
| Worker 入口 | [worker.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/jobs/worker.ts) |
| Workflow Worker | [workflow-processor-queue.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/jobs/workflow-processor-queue.ts) |
| Step 配置 Schema | [index.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/shared/src/schemas/index.ts#L250-L360) |

---

## 二、Workflow 定义与数据模型

### 2.1 核心数据模型

#### Workflow（工作流定义）
[schema.prisma#L338-L367](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/db/prisma/schema.prisma#L338-L367)

```prisma
model Workflow {
  id            String                @id @default(uuid())
  name          String
  description   String?
  enabled       Boolean               @default(false)
  triggerType   WorkflowTriggerType   // EVENT | MANUAL | SCHEDULE
  triggerConfig Json?                 // { eventName: "user.signup" }
  allowReentry  Boolean               @default(false)
  projectId     String
  steps         WorkflowStep[]
  executions    WorkflowExecution[]
}
```

**关键字段解读**：
- `triggerType`: 触发类型，目前主要使用 `EVENT`（事件触发）
- `triggerConfig`: 触发配置，如 `{eventName: "user.signup"}`
- `allowReentry`: 是否允许同一联系人重复进入工作流
  - `false`: 联系人一旦执行过（无论最终状态），永远不能再进入
  - `true`: 联系人可以重复进入，但同一时间只能有一个 `RUNNING` 实例

#### WorkflowStep（工作流步骤）
[schema.prisma#L369-L409](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/db/prisma/schema.prisma#L369-L409)

```prisma
model WorkflowStep {
  id                  String              @id @default(uuid())
  type                WorkflowStepType    // 步骤类型
  name                String
  position            Json                // 可视化位置 {x, y}
  config              Json                // 步骤配置（因类型而异）
  workflowId          String
  templateId          String?             // SEND_EMAIL 专用
  outgoingTransitions WorkflowTransition[] @relation("FromStep")
  incomingTransitions WorkflowTransition[] @relation("ToStep")
}
```

**步骤类型枚举** `WorkflowStepType`：
| 类型 | 说明 |
|------|------|
| `TRIGGER` | 入口点，每个 Workflow 有且仅有一个 |
| `SEND_EMAIL` | 发送邮件 |
| `DELAY` | 等待指定时长 |
| `WAIT_FOR_EVENT` | 等待特定事件发生（支持超时） |
| `CONDITION` | 条件分支（if/else 或 switch/case） |
| `EXIT` | 提前退出工作流 |
| `WEBHOOK` | 调用外部 Webhook |
| `UPDATE_CONTACT` | 更新联系人属性 |

#### WorkflowTransition（步骤转移）
[schema.prisma#L411-L432](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/db/prisma/schema.prisma#L411-L432)

```prisma
model WorkflowTransition {
  id          String  @id @default(uuid())
  fromStepId  String
  toStepId    String
  condition   Json?   // null=无条件, {branch: "yes"}=条件分支
  priority    Int     @default(0)  // 评估顺序（小值优先）
}
```

#### WorkflowExecution（工作流执行实例）
[schema.prisma#L434-L473](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/db/prisma/schema.prisma#L434-L473)

```prisma
model WorkflowExecution {
  id             String                    @id @default(uuid())
  workflowId     String
  contactId      String
  status         WorkflowExecutionStatus   @default(RUNNING)
  currentStepId  String?
  exitReason     String?
  context        Json?                     // 执行上下文（事件数据）
  startedAt      DateTime                  @default(now())
  completedAt    DateTime?
}
```

**执行状态枚举** `WorkflowExecutionStatus`：
| 状态 | 说明 |
|------|------|
| `RUNNING` | 正在执行 |
| `WAITING` | 等待中（DELAY 或 WAIT_FOR_EVENT） |
| `COMPLETED` | 正常完成 |
| `EXITED` | 通过 EXIT 步骤提前退出 |
| `FAILED` | 执行失败 |
| `CANCELLED` | 被用户手动取消 |

#### WorkflowStepExecution（单步执行记录）
[schema.prisma#L475-L511](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/db/prisma/schema.prisma#L475-L511)

```prisma
model WorkflowStepExecution {
  id             String               @id @default(uuid())
  executionId    String
  stepId         String
  status         StepExecutionStatus  @default(PENDING)
  scheduledFor   DateTime?            // DELAY 步骤专用
  executeAfter   DateTime?            // WAIT_FOR_EVENT 超时时间
  output         Json?                // 步骤执行结果
  error          String?              // 错误信息
  startedAt      DateTime?
  completedAt    DateTime?
}
```

**单步状态枚举** `StepExecutionStatus`：
| 状态 | 说明 |
|------|------|
| `PENDING` | 待执行 |
| `SCHEDULED` | 已调度（DELAY） |
| `WAITING` | 等待事件中（WAIT_FOR_EVENT） |
| `RUNNING` | 执行中 |
| `COMPLETED` | 已完成 |
| `SKIPPED` | 已跳过 |
| `FAILED` | 执行失败 |

### 2.2 Step 配置 Schema 详解

各步骤类型的配置 Schema 定义在 [WorkflowStepConfigSchemas](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/shared/src/schemas/index.ts#L250-L360)：

**SEND_EMAIL 配置**：
```typescript
{
  templateId: string;           // 邮件模板 ID
  recipient?: {
    type: 'CONTACT' | 'CUSTOM';
    customEmail?: string;       // type=CUSTOM 时必填
  }
}
```

**DELAY 配置**：
```typescript
{
  amount: number;               // 时长数值
  unit: 'minutes' | 'hours' | 'days';  // 时长单位
  // 最大 365 天
}
```

**WAIT_FOR_EVENT 配置**：
```typescript
{
  eventName: string;            // 等待的事件名
  timeout?: number;             // 超时秒数（最大 365 天）
}
```

**CONDITION 配置**（支持两种模式）：
```typescript
// 1. 二元模式 (if/else)
{
  field: string;                // 字段路径，如 "data.plan"
  operator: string;             // equals, notEquals, contains, greaterThan 等
  value?: any;                  // 比较值
}

// 2. 多分支模式 (switch/case)
{
  mode: 'multi';
  field: string;
  branches: Array<{
    id: string;
    name: string;
    operator: string;
    value?: any;
  }>;
}
```

**WEBHOOK 配置**：
```typescript
{
  url: string;                  // Webhook URL
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';  // 默认 POST
  headers?: Record<string, string>;
  body?: Json;                  // 请求体（支持模板变量）
}
```

**UPDATE_CONTACT 配置**：
```typescript
{
  updates?: Record<string, any>;         // 字段更新
  subscriptionAction?: 'none' | 'subscribe' | 'unsubscribe';
  // 至少需要提供一项
}
```

---

## 三、触发条件与启动机制

### 3.1 触发方式

#### 方式 1：事件自动触发（主路径）
触发入口：[EventService.trackEvent()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/EventService.ts#L22-L47) → [triggerWorkflows()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/EventService.ts#L351-L408)

**执行流程**：
```
事件发生 → trackEvent()
  ├─ 创建 Event 记录
  ├─ triggerWorkflows() → 查询匹配的 Workflow
  │   └─ 对每个匹配的 Workflow:
  │       └─ startWorkflowForContact()
  │           ├─ 检查 re-entry 规则
  │           ├─ 创建 WorkflowExecution (status=RUNNING)
  │           └─ 调用 processStepExecution() 开始执行
  └─ handleEvent() → 唤醒正在等待此事件的 WAIT_FOR_EVENT 步骤
```

**缓存机制**：
- 已启用的 Workflow 列表缓存于 Redis 5 分钟
- 缓存 Key: `Keys.Workflow.enabled(projectId)`
- Workflow 启用/禁用/更新时通过 [invalidateWorkflowCache()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/EventService.ts#L53-L60) 失效缓存

#### 方式 2：手动 API 触发
触发入口：[WorkflowService.startExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L860-L941)

**执行流程**：
```
API 请求 → startExecution()
  ├─ 验证 Workflow 已启用
  ├─ 验证联系人存在
  ├─ 检查 re-entry 规则
  ├─ 创建 WorkflowExecution (status=RUNNING)
  └─ 异步调用 processStepExecution()（不 await）
```

### 3.2 Re-entry 规则详解
[WorkflowService.startExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L886-L914)

| allowReentry | 检查逻辑 | 结果 |
|--------------|---------|------|
| `false` | 查询是否存在 **任何** 执行记录（无论状态） | 存在则拒绝 |
| `true` | 查询是否存在 **RUNNING** 状态的执行记录 | 存在则拒绝 |

> **重要**：`allowReentry=true` 时，已完成/失败/退出的执行不阻止新的执行，但同一时间只能有一个运行中的实例。

---

## 四、执行状态推进主链路

### 4.1 核心入口：processStepExecution()
[WorkflowExecutionService.processStepExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L47-L299)

这是整个执行引擎的核心函数，所有步骤执行都从此进入。

**执行流程图**：
```
processStepExecution(executionId, stepId)
  │
  ├─ 前置检查
  │   ├─ 加载 Execution + Workflow + Steps + Transitions
  │   ├─ 状态检查：
  │   │   ├─ WAITING → 继续（延迟步骤恢复）
  │   │   ├─ RUNNING → 继续
  │   │   └─ 其他状态 → 直接返回（已完成/取消/失败）
  │   ├─ 项目禁用检查 → 若禁用，标记 CANCELLED 并返回
  │   └─ Workflow 禁用检查 → 仅记录日志，**允许继续执行**
  │
  ├─ 状态校正（从 WAITING 恢复时）
  │   └─ 更新 Execution 状态为 RUNNING
  │
  ├─ 创建/更新 StepExecution
  │   ├─ 存在 PENDING/RUNNING → 更新为 RUNNING
  │   └─ 不存在 → 创建新 StepExecution (status=RUNNING)
  │
  ├─ 执行步骤（executeStep）
  │   └─ 根据 step.type 分派到不同处理器
  │
  ├─ 步骤执行后处理
  │   ├─ 检查 StepExecution 是否为 WAITING（WAIT_FOR_EVENT）
  │   │   └─ 是 → 直接返回（不推进）
  │   ├─ 检查 Execution 是否为 WAITING（DELAY 刚设置）
  │   │   └─ 是 → 直接返回（不推进，由队列唤醒）
  │   ├─ 标记 StepExecution 为 COMPLETED
  │   └─ processNextSteps() → 推进到下一步
  │
  └─ 异常处理
      ├─ 标记 StepExecution 为 FAILED
      ├─ 标记 Execution 为 FAILED
      ├─ 发送失败通知
      └─ 抛出异常
```

### 4.2 步骤处理器：executeStep()
[WorkflowExecutionService.executeStep()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L467-L502)

根据 `step.type` 分派到不同处理函数：

| Step 类型 | 处理函数 | 核心行为 |
|-----------|---------|---------|
| `TRIGGER` | [executeTrigger()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L507-L521) | 空操作，仅记录触发事件 |
| `SEND_EMAIL` | [executeSendEmail()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L526-L593) | 渲染模板 + 调用 EmailService 发送 |
| `DELAY` | [executeDelay()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L598-L668) | 计算延迟时间 + 排队下一步 + 标记 WAITING |
| `WAIT_FOR_EVENT` | [executeWaitForEvent()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L673-L712) | 标记 WAITING + 排队超时任务 |
| `CONDITION` | [executeCondition()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L717-L791) | 评估条件 + 返回匹配分支 |
| `EXIT` | [executeExit()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L796-L825) | 标记 Execution 为 EXITED |
| `WEBHOOK` | [executeWebhook()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L920-L1006) | SSRF 安全检查 + HTTP 请求 |
| `UPDATE_CONTACT` | [executeUpdateContact()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1033-L1085) | 更新联系人属性 + 可选订阅操作 |

### 4.3 下一步选择：processNextSteps()
[WorkflowExecutionService.processNextSteps()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1090-L1168)

**转移选择逻辑**：
```
遍历 transitions（按 priority 升序）:
  1. 无条件（condition=null）→ 选中，break
  2. 有条件 + CONDITION 步骤结果 → 匹配 branch → 选中，break
  3. 其他条件 → evaluateTransitionCondition() → 满足 → 选中，break

无匹配 transition → 标记 Execution 为 COMPLETED
有匹配 transition →
  ├─ 更新 Execution.currentStepId = nextStep.id
  ├─ 更新 Execution.status = RUNNING
  └─ 递归调用 processStepExecution() 执行下一步
```

> **注意**：`evaluateTransitionCondition()` 当前始终返回 `false`，只有 branch 匹配和无条件转移生效。

---

## 五、异步与延迟边界

### 5.1 DELAY 步骤的异步处理
[executeDelay()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L598-L668)

**执行流程**：
```
executeDelay()
  ├─ 解析配置：{amount, unit} → 计算 delayMs
  ├─ 立即标记 StepExecution 为 COMPLETED（记录 resumeAt）
  ├─ 标记 Execution 为 WAITING
  ├─ 查询下一 Step
  └─ QueueService.queueWorkflowStep(executionId, nextStepId, delayMs)
      └─ 加入 BullMQ 队列，延迟 delayMs 后执行
```

**队列消费**：
- Worker: [createWorkflowWorker()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/jobs/workflow-processor-queue.ts#L13-L50)
- 延迟到达后 → 调用 `processStepExecution(executionId, stepId)`
- processStepExecution 检测到 `status=WAITING` → 恢复执行

### 5.2 WAIT_FOR_EVENT 的异步处理
[executeWaitForEvent()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L673-L712)

**执行流程**：
```
executeWaitForEvent()
  ├─ 解析配置：{eventName, timeout?}
  ├─ 标记 StepExecution 为 WAITING
  ├─ 标记 Execution 为 WAITING
  └─ timeout 存在时 → QueueService.queueWorkflowTimeout(..., timeoutMs)
      └─ 延迟 timeoutMs 后执行 processTimeout()
```

**两条唤醒路径**：

**路径 A：事件到达（正常路径）**
```
EventService.trackEvent(eventName)
  └─ WorkflowExecutionService.handleEvent(projectId, eventName, contactId, data)
      ├─ 查询所有 status=WAITING 且 type=WAIT_FOR_EVENT 的 StepExecution
      ├─ 匹配 eventName
      ├─ 标记 StepExecution 为 COMPLETED（记录事件数据）
      ├─ 取消超时任务（cancelWorkflowTimeout）
      └─ processNextSteps() → 推进下一步
```

**路径 B：超时触发**
```
BullMQ 超时任务到期 → processTimeout(executionId, stepId, stepExecutionId)
  ├─ 检查 StepExecution 仍为 WAITING（事件可能已先到）
  ├─ 标记 StepExecution 为 COMPLETED（output={timedOut: true}）
  ├─ 查找 timeout/fallback 分支的 transition
  ├─ 找到 → 推进到对应分支
  ├─ 未找到但有 transitions → 推进到第一个
  └─ 无 transitions → 标记 COMPLETED
```

### 5.3 队列与并发配置
[QueueService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/QueueService.ts#L73-L84)

```typescript
// workflowQueue 配置
{
  attempts: 3,                     // 失败重试 3 次
  backoff: { type: 'exponential', delay: 2000 },  // 指数退避
  concurrency: 10,                 // 同时处理 10 个任务
  removeOnComplete: 1000,          // 保留最近 1000 个完成任务
  removeOnFail: 5000,              // 保留最近 5000 个失败任务
}
```

**作业类型**：
| Job 类型 | Job ID 格式 | 说明 |
|---------|-----------|------|
| process-step | `workflow-${executionId}-${stepId}` | 延迟步骤执行 |
| timeout | `workflow-timeout-${stepExecutionId}` | WAIT_FOR_EVENT 超时 |

---

## 六、错误恢复机制

### 6.1 步骤执行失败
[processStepExecution() 异常处理](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L254-L298)

```
try {
  executeStep()
} catch (error) {
  ├─ 标记 StepExecution.status = FAILED
  ├─ 记录 error = error.message
  ├─ 标记 Execution.status = FAILED
  ├─ 记录 completedAt = now()
  ├─ 发送 Ntfy 失败通知
  └─ throw error → 由 BullMQ 重试（最多 3 次）
}
```

### 6.2 BullMQ 自动重试
- 队列配置 `attempts: 3`，步骤执行失败会自动重试
- 重试间隔：指数退避（2s, 4s, 8s...）
- **注意**：重试时会再次执行整个 `processStepExecution()`，包括：
  - 重新查询 Execution 状态
  - 重新创建/更新 StepExecution
  - 重新执行步骤逻辑
- 已标记为 FAILED 的 Execution 不会被重试（状态检查会直接返回）

### 6.3 项目禁用处理
[processStepExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L92-L105)
- 每次执行前检查 `project.disabled`
- 若禁用 → 标记 Execution 为 `CANCELLED`，`exitReason="Project disabled"`

### 6.4 Workflow 禁用处理
[processStepExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L111-L116)
- Workflow 禁用后，**已启动的 Execution 允许继续执行完成**
- 仅阻止新的 Execution 启动（在 `startExecution()` 中检查）

### 6.5 手动取消
- [WorkflowService.cancelExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L1045-L1061) - 取消单个执行
- [WorkflowService.cancelAllExecutions()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L1066-L1086) - 批量取消
- 状态更新为 `CANCELLED`，`exitReason="Cancelled by user"`

### 6.6 WEBHOOK SSRF 防护
[safeFetch()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L871-L908)
- DNS 解析后检查 IP 不落在私有/保留网段
- 手动处理重定向，每跳重新验证 IP
- 支持最多 5 次重定向
- 10 秒请求超时
- 仅允许 http/https 协议

---

## 七、人工容易误判的点

### ❗ 误判点 1：Workflow 禁用后的行为
**误解**：禁用 Workflow 会中断正在运行的执行。
**实际**：禁用仅阻止新执行启动，已运行的执行会继续完成。
**代码依据**：[WorkflowExecutionService.ts#L111-L116](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L111-L116)

### ❗ 误判点 2：allowReentry 的语义
**误解**：`allowReentry=true` 允许多个执行同时运行。
**实际**：仅允许历史执行过的联系人再次进入，但同一时间只能有一个 RUNNING 实例。
**代码依据**：[WorkflowService.ts#L902-L914](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L902-L914)

### ❗ 误判点 3：DELAY 步骤的状态变化
**误解**：DELAY 步骤会在等待结束后才标记为 COMPLETED。
**实际**：DELAY 步骤立即标记为 COMPLETED，等待是在 Transition 层面通过队列延迟实现的。
**代码依据**：[WorkflowExecutionService.ts#L626-L637](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L626-L637)

### ❗ 误判点 4：WAIT_FOR_EVENT 超时的分支选择
**误解**：超时后走默认分支。
**实际**：优先查找 `branch="timeout"` 或 `fallback=true` 的 transition，找不到才走第一个 transition。
**代码依据**：[WorkflowExecutionService.ts#L351-L358](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L351-L358)

### ❗ 误判点 5：CONDITION 字段解析路径
**误解**：`contact.data.plan` 会从 `contact.data.data.plan` 取值。
**实际**：`contact.data.` 前缀会被特殊处理，去掉 `contact.` 后解析，实际路径是 `data.plan`。
**代码依据**：[WorkflowExecutionService.ts#L1186-L1188](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1186-L1188)

### ❗ 误判点 6：notEquals 等运算符对 null/undefined 的处理
**误解**：`notEquals "abc"` 在字段不存在时返回 true。
**实际**：`notEquals`、`notContains`、`greaterThan` 等运算符在字段为 null/undefined 时返回 false，避免误匹配。
**代码依据**：[WorkflowExecutionService.ts#L1214-L1255](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1214-L1255)

### ❗ 误判点 7：Step 删除的级联行为
**误解**：删除 Step 只会删除该 Step 本身。
**实际**：删除 Step 会级联删除所有下游 Steps（BFS 遍历可达节点）。
**代码依据**：[WorkflowService.ts#L604-L643](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L604-L643)

### ❗ 误判点 8：Workflow 修改的并发保护
**误解**：Workflow 启用后可以随意修改步骤配置。
**实际**：有活跃执行时，禁止修改 trigger 配置和 step 的 config/templateId，仅允许修改名称和位置。
**代码依据**：[WorkflowService.ts#L179-L195](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L179-L195)

### ❗ 误判点 9：执行失败后的重试
**误解**：执行失败标记为 FAILED 后会自动重试。
**实际**：标记为 FAILED 后状态检查会直接返回，BullMQ 重试也不会继续。只有步骤执行过程中抛出未捕获异常才会触发重试。
**代码依据**：[WorkflowExecutionService.ts#L84-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L84-L89)

### ❗ 误判点 10：startExecution 的调用方式
**误解**：`startExecution()` 是同步调用，会等待第一个步骤完成。
**实际**：调用 `processStepExecution()` 时没有 await，异步在后台执行，立即返回 Execution 对象。
**代码依据**：[WorkflowService.ts#L936-L938](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L936-L938)

---

## 八、关键状态流转图

### 8.1 WorkflowExecution 状态流转
```
          startExecution()
                │
                ▼
           RUNNING ◄───────────┐
                │              │
                ├─ 步骤成功 → processNextSteps()
                │              │
                ├─ DELAY → WAITING ── 队列延迟 ──┘
                │              │
                ├─ WAIT_FOR_EVENT → WAITING ─┬─ 事件到达 ─┐
                │                              └─ 超时 ────┘
                │
                ├─ 无下一步 → COMPLETED
                ├─ EXIT 步骤 → EXITED
                ├─ 异常 → FAILED
                └─ 用户取消 / 项目禁用 → CANCELLED
```

### 8.2 StepExecution 状态流转
```
PENDING → RUNNING ─┬─ 成功 → COMPLETED
                   ├─ DELAY（立即完成）→ COMPLETED
                   ├─ WAIT_FOR_EVENT → WAITING ─┬─ 事件 → COMPLETED
                   │                              └─ 超时 → COMPLETED
                   └─ 失败 → FAILED
```

---

## 九、测试覆盖要点

从 [WorkflowExecutionService.test.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/__tests__/WorkflowExecutionService.test.ts) 和 [WorkflowExecutionService.integration.test.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/__tests__/WorkflowExecutionService.integration.test.ts) 可见测试覆盖：

1. **基础流程**：完整执行链路、状态追踪
2. **条件分支**：YES/NO 分支、多分支模式、嵌套条件
3. **延迟处理**：DELAY 步骤排队与恢复
4. **事件等待**：WAIT_FOR_EVENT 正常唤醒、超时处理、事件先到的幂等性
5. **退订处理**：营销邮件跳过、事务邮件不跳过、中途退订
6. **Webhook**：SSRF 防护、变量渲染
7. **并发控制**：re-entry 规则验证
8. **边界场景**：项目禁用、Workflow 禁用、手动取消

---

## 十、总结

### 主链路
`Event → Workflow 匹配 → Execution 创建 → 递归执行 Steps → Transitions 路由 → 完成/退出`

### 异步边界
1. **DELAY**：立即完成 Step，通过 BullMQ 延迟调度下一步
2. **WAIT_FOR_EVENT**：进入 WAITING，双路径唤醒（事件到达 / 超时）
3. **startExecution**： fire-and-forget 模式，不等待执行结果

### 错误恢复
- BullMQ 指数退避重试（3 次）
- 项目禁用自动取消
- 失败状态持久化 + 通知
- Webhook SSRF 多层防护

### 易误判点速查
| 行为 | 实际表现 |
|------|---------|
| Workflow 禁用 | 已启动的继续跑，仅阻新的 |
| allowReentry=true | 历史可重入，但并发仅 1 个 |
| DELAY 状态 | Step 立即 COMPLETED，Execution WAITING |
| 字段 `contact.data.X` | 自动去前缀，解析为 `data.X` |
| notEquals null | 返回 false，不匹配空值 |
| 删除 Step | 级联删除所有下游 Steps |
| 活跃时修改 | 仅允许改名称和位置 |
| FAILED 后 | 不会自动重试，状态为终态 |
