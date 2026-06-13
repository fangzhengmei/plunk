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
- `triggerType`: 触发类型，枚举值 `EVENT | MANUAL | SCHEDULE`
  - `EVENT`: 事件触发（已实现，主路径）
  - `MANUAL`: 手动 API 触发（已实现）
  - `SCHEDULE`: 定时调度触发（**已声明未实现**，仅有枚举值和 schema 注释，无调度执行逻辑）
- `triggerConfig`: 触发配置，如 `{eventName: "user.signup"}` 或注释中的 `{schedule: "0 9 * * *"}`
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
| `CANCELLED` | 手动取消或项目禁用自动取消 |

**exitReason 的取值来源**（不同终态各异，部分终态为空）：
- `COMPLETED`: 不设置 exitReason（null）
- `EXITED`: 取 EXIT 步骤的 `config.reason`，无配置时默认 `"exit_step"`
- `CANCELLED`（用户手动）: `"Cancelled by user"` 或 `"Cancelled by user (bulk cancel)"`
- `CANCELLED`（项目禁用）: `"Project disabled"`
- `FAILED`: 不设置 exitReason，错误信息记录在 StepExecution.error 中
- 代码依据：[WorkflowExecutionService.ts#L816](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L816)

**执行上下文（context）的写入来源**：
`WorkflowExecution.context` 字段存储触发工作流的原始数据，仅在**创建 Execution 时一次性写入**，后续所有步骤执行中不会被修改。共有两个写入入口：

| 写入入口 | 调用方 | context 来源 | 代码位置 |
|---------|--------|-------------|---------|
| 事件触发启动 | EventService.startWorkflowForContact() | `trackEvent()` 传入的 `data` 参数（事件载荷） | [EventService.ts#L470-L477](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/EventService.ts#L470-L477) |
| 手动 API 启动 | WorkflowService.startExecution() | 调用方传入的可选 `context` 参数 | [WorkflowService.ts#L924-L931](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L924-L931) |

**重要**：context 一旦写入不再变更。即使后续 WAIT_FOR_EVENT 步骤被新的事件唤醒，该新事件的 data 只会记录在 StepExecution.output 中，不会更新 Execution.context。

#### WorkflowStepExecution（单步执行记录）
[schema.prisma#L475-L511](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/db/prisma/schema.prisma#L475-L511)

```prisma
model WorkflowStepExecution {
  id             String                    @id @default(uuid())
  executionId    String
  stepId         String
  status         StepExecutionStatus       @default(PENDING)
  output         Json?                     // 步骤输出（因类型而异）
  error          String?                   // 错误信息
  startedAt      DateTime                  @default(now())
  completedAt    DateTime?
  executeAfter   DateTime?                 // 延迟/超时执行时间
}
```

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
  operator: string;             // 十个操作符之一
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
  }>;  // 1~20 个分支
  // 注意：Schema 中没有 default 配置，default 是运行时的隐式兜底
}
```

**CONDITION 条件字段的五个根命名空间**：
条件表达式中的 `field` 通过点号从 `fieldData` 对象解析。顶层共有五个命名空间：

| 命名空间 | 含义 | 包含内容 | 代码位置 |
|---------|------|---------|---------|
| `contact` | 联系人基础属性 | `email`, `subscribed`（仅两个字段） | [WorkflowExecutionService.ts#L740-L743](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L740-L743) |
| `data` | 联系人自定义数据 | `contact.data` 展开的所有字段 | [WorkflowExecutionService.ts#L744](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L744) |
| `workflow` | 执行上下文（别名） | `execution.context` 展开的所有字段 = 触发事件的 payload | [WorkflowExecutionService.ts#L745](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L745) |
| `event` | 执行上下文（正名） | 与 `workflow` 完全相同，指向同一个 `execution.context` 对象 | [WorkflowExecutionService.ts#L746](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L746) |
| `contact.data` | 兼容格式（解析时转换） | 特殊的前缀匹配规则：`contact.data.X` 解析前自动去前缀，转为 `data.X` 再查找 | [WorkflowExecutionService.ts#L1186-L1188](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1186-L1188) |

**示例字段路径**：
- `contact.email` → 联系人邮箱
- `data.firstName` → 联系⼈自定义属性 firstName
- `workflow.orderId` → 触发事件中的 orderId（事件 payload）
- `event.orderId` → 同上，event 是 workflow 的别名
- `contact.data.plan` → 兼容旧写法，自动转换为 `data.plan`

**CONDITION 十个操作符及未知操作符处理**：
操作符定义在 Zod Schema 的 enum 中，并在 `evaluateCondition()` 用 switch 实现。遇到未识别操作符会**直接抛错中断执行**：

| 操作符 | 说明 | null/undefined 处理 |
|-------|------|-------------------|
| `equals` | 严格相等 (`===`) | 可匹配 null/undefined（双方一致时） |
| `notEquals` | 严格不等 (`!==`) | 字段为 null/undefined 时返回 false |
| `contains` | 字符串包含 | 字段为 null/undefined 时返回 false |
| `notContains` | 字符串不包含 | 字段为 null/undefined 时返回 false |
| `greaterThan` | 数值大于 `>` | 字段为 null/undefined 时返回 false |
| `lessThan` | 数值小于 `<` | 字段为 null/undefined 时返回 false |
| `greaterThanOrEqual` | 数值大于等于 `>=` | 字段为 null/undefined 时返回 false |
| `lessThanOrEqual` | 数值小于等于 `<=` | 字段为 null/undefined 时返回 false |
| `exists` | 字段存在（非 null 非 undefined） | 返回 false |
| `notExists` | 字段不存在 | 返回 true |

**未知操作符**：switch 的 default 分支 `throw new Error(\`Unknown operator: ${operator}\`)`，属于**硬错误**，会导致该步骤及整个 Execution 标记为 FAILED。
代码依据：[WorkflowExecutionService.ts#L1207-L1263](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1207-L1263)

**CONDITION 多分支模式 default 兜底**：
`mode='multi'` 时，逐个遍历 `branches` 数组按 operator 评估，第一个匹配即返回其对应的 `branch.id`。如果所有 branches 都不匹配，**隐式返回 `branch='default'`**：
- Schema 层面没有 default 的配置项，default 是执行层的内置兜底
- 返回值：`{matchedBranch: 'default', branch: 'default', actualValue, field, mode: 'multi'}`
- 对应 Transition 的 `condition.branch` 需配置为 `"default"` 才能被选中
- 代码依据：[WorkflowExecutionService.ts#L749-L773](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L749-L773)

**WEBHOOK 配置**：
```typescript
{
  url: string;                  // Webhook URL（支持模板变量渲染）
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';  // 默认 POST，不参与模板渲染
  headers?: Record<string, string>;  // header 值支持模板变量渲染
  body?: Json;                  // 请求体（支持模板变量深度渲染）
}
```

**WEBHOOK 模板变量域的展开覆盖陷阱**：
变量 scope 是 SEND_EMAIL scope 的超集。构造方式为「浅层对象字面量 + 展开运算符」，**展开顺序决定覆盖优先级**，后面展开的同名字段会覆盖前面的：

```typescript
const variables = {
  id: contact.id,              // (1) 系统变量：联系人 ID
  email: contact.email,        // (2) 系统变量：联系人邮箱
  ...contactData,              // (3) 展开联系人自定义数据 —— 如果自定义数据含 id/email，会覆盖 (1)(2)
  ...executionContext,         // (4) 展开执行上下文（事件payload）—— 优先级最高，会覆盖 (1)(2)(3)
  data: contactData,           // (5) data 命名空间
  event: context,              // (6) event 命名空间（WEBHOOK 独有）
  unsubscribeUrl: `${DASHBOARD_URI}/unsubscribe/${contact.id}`,  // (7) 系统URL
  subscribeUrl:   `${DASHBOARD_URI}/subscribe/${contact.id}`,    //    同(7)
  manageUrl:      `${DASHBOARD_URI}/manage/${contact.id}`,       //    同(7)
};
```

**覆盖陷阱总结**：
- 第 (3)(4) 步的展开是**浅拷贝**，任意顶层键冲突都会被后面的值覆盖
- 风险场景：
  - 联系人自定义数据中有 `id`、`email`、`unsubscribeUrl` 等键 → 覆盖系统变量
  - 触发事件 payload 中有上述键 → 进一步覆盖前两者
- SEND_EMAIL 的 scope 构造逻辑相同，存在同样的覆盖陷阱
- 代码依据：[WorkflowExecutionService.ts#L944-L954](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L944-L954)

**WEBHOOK 的 method 不参与模板渲染**：
`url`、`headers` 的每个值、`body` 中所有字符串值均会被模板渲染，但 `method` 字段**直接传递给 fetch，不做任何模板替换**（防止模板注入导致非预期的 HTTP 方法）。
代码依据：[WorkflowExecutionService.ts#L956-L989](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L956-L989)

**WEBHOOK 未配请求体时的默认载荷**：
当配置中未提供 `body` 字段时，引擎会**自动构造一个标准化的默认 JSON 载荷**，包含四类信息：

```json
{
  "contact": {
    "email": "user@example.com",     // 联系人邮箱
    "subscribed": true,              // 订阅状态
    "data": { /* 联系⼈自定义数据 */ }   // contact.data 原样嵌入
  },
  "workflow": {
    "id": "wf_xxx",                  // 工作流 ID
    "name": "Onboarding Series"      // 工作流名称
  },
  "execution": {
    "id": "exec_xxx",                // 执行实例 ID
    "startedAt": "2025-06-13T09:00:00Z"  // 启动时间
  },
  "event": { /* 触发事件的 payload */ }   // execution.context
}
```

**重要细节**：
- 即便配置了 `body`，对于 `GET` 请求也**不会发送请求体**（`method !== 'GET'` 时才会将 payload 转为 JSON 字符串）
- 默认载荷完全不经过模板渲染，是直接从运行时对象构造的字面量
- 代码依据：[WorkflowExecutionService.ts#L964-L989](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L964-L989)

**UPDATE_CONTACT 配置**：
```typescript
{
  updates?: Record<string, any>;         // 字段更新
  subscriptionAction?: 'none' | 'subscribe' | 'unsubscribe';
  // 至少需要提供一项
}
```

**模板渲染的三大坑**（SEND_EMAIL 和 WEBHOOK 共用）：
模板渲染函数 [renderTemplate()](file:///d:/fz/0601-1\solo-dogfeeding\code\54-plunk\packages\shared\src\template.ts#L11-L38) 存在三处容易踩坑的设计：

**坑 1：`||` 对零和空串等假值短路**
变量查找使用 `||` 运算符串联三层查找，**所有 JavaScript 假值（`0`、`""`、`false`）都会被短路**，不会被当作有效值使用：

```typescript
const value =
  getValue(variables, mainKey) ||      // 第 1 层：嵌套路径查找
  variables[mainKey] ||                // 第 2 层：顶层键查找
  (variables.data as Record<string, unknown>)?.[mainKey];  // 第 3 层：data 回落
```

| 变量实际值 | 是否被当作有效值 | 说明 |
|---------|----------------|------|
| `"hello"` | ✅ 是 | 非空字符串 |
| `0` | ❌ 否 | 数值 0 被 `||` 跳过 |
| `""` | ❌ 否 | 空串被跳过 |
| `false` | ❌ 否 | 布尔 false 被跳过 |
| `null`/`undefined` | ❌ 否 | 空值被跳过 |

> 注意：最后返回用 `??`，只有 null/undefined 才会回退到 defaultValue

**坑 2：数组字段被自动包成 `<li>` 列表**
如果变量值是数组类型，会被自动 map 成 HTML `<li>` 列表，**无法作为普通字符串使用：

```typescript
if (Array.isArray(value)) {
  return value.map((e: string) => `<li>${e}</li>`).join('\n');
}
```

- 例如 `tags: ["a", "b", "c"]` 会被渲染成：
```html
<li>a</li>
<li>b</li>
<li>c</li>
```

**坑 3：缺命中时自动回落到 `data` 命名空间**
前两层查找都失败时，**自动尝试从 `variables.data[mainKey]` 再找一遍。这意味着 `variables = { id, email, ...contactData, ...executionContext, data: contactData }` 的结构下，即便顶层键不存在，还会在 `data.键` 再找一次。

三层查找顺序：
1. `getValue(variables, "firstName")` → 嵌套路径（如 `data.firstName`）
2. `variables["firstName"]` → 顶层键（因 `...contactData` 展开，通常能命中）
3. `variables.data["firstName"]` → 回落到 data 命名空间

代码依据：[template.ts#L26-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/shared/src/template.ts#L26-L37)

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
  │           ├─ 校验 re-entry 规则
  │           ├─ 创建 WorkflowExecution（context = event.data）
  │           └─ processStepExecution(triggerStep)  // 异步不等待
  └─ handleEvent() → 唤醒 WAIT_FOR_EVENT 步骤
```

#### 方式 2：手动 API 触发
入口：[WorkflowService.startExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L860-L941)

**执行流程**：
```
API 请求 → startExecution(projectId, workflowId, contactId, context?)
  ├─ 校验 Workflow 已 enabled
  ├─ 校验 Contact 存在
  ├─ 校验 re-entry 规则
  ├─ 创建 WorkflowExecution（context = 传入参数 or null）
  └─ processStepExecution(triggerStep)  // fire-and-forget，不 await
```

#### 方式 3：定时调度触发（未实现）
`WorkflowTriggerType.SCHEDULE` 仅在枚举和 schema 注释中声明，**没有任何实际调度执行逻辑**：
- 无 cron 调度器
- 无定时扫描任务
- 无对应的队列 worker 或 job processor
- 代码依据：[schema.prisma#L681-L685](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/db/prisma/schema.prisma#L681-L685)

### 3.2 Re-entry 规则详解

[WorkflowService.startExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowService.ts#L885-L914) 和 [EventService.startWorkflowForContact()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/EventService.ts#L434-L460)

| allowReentry | 已有历史执行 | 已有 RUNNING 执行 | 结果 |
|-------------|------------|------------------|------|
| false | 有（任意状态） | 任意 | 禁止进入 |
| false | 无 | 无 | 允许进入 |
| true | 任意 | 无 | 允许进入 |
| true | 任意 | 有 | 禁止进入 |

---

## 四、执行状态推进主链路

### 4.1 核心入口：processStepExecution()
[WorkflowExecutionService.processStepExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L70-L298)

**执行流程**：
```
processStepExecution(executionId, stepId)
  │
  ├─ 1. 前置检查
  │   ├─ 查询 Execution（含 workflow、steps、contact）
  │   ├─ 状态校验：COMPLETED/FAILED/CANCELLED → 直接返回
  │   ├─ 项目禁用检查 → 标记 CANCELLED，exitReason="Project disabled"
  │   └─ Workflow 禁用 → 允许继续（仅阻止新启动）
  │
  ├─ 2. 状态恢复
  │   └─ WAITING → 设置回 RUNNING（延迟步骤或等待步骤恢复）
  │
  ├─ 3. 创建/获取 StepExecution 记录
  │   ├─ 存在 PENDING/RUNNING → 复用
  │   └─ 不存在 → 创建新的 PENDING 记录
  │
  ├─ 4. 标记 StepExecution 为 RUNNING
  │
  ├─ 5. 调用 executeStep() 执行步骤逻辑
  │   └─ 返回 StepResult（包含分支信息、输出数据等）
  │
  ├─ 6. 成功处理
  │   ├─ 标记 StepExecution 为 COMPLETED，写入 output
  │   └─ 调用 processNextSteps() 推进下一步
  │
  └─ 7. 失败处理
      ├─ 标记 StepExecution 为 FAILED，记录 error
      ├─ 标记 Execution 为 FAILED，记录 completedAt
      ├─ 发送 Ntfy 失败通知
      └─ throw error → BullMQ 重试（最多 3 次）
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

### 4.4 同步递归长链的事务超时风险

**执行模型**：`processNextSteps()` → `processStepExecution()` → `executeStep()` → `processNextSteps()` 形成**同步递归调用链**。
- 没有分拆事务，没有队列投递，没有异步切分
- 一长串同步步骤（如 TRIGGER → CONDITION → UPDATE_CONTACT → CONDITION → SEND_EMAIL）会在同一个调用栈内连续执行
- 每个步骤独立更新数据库，但整体调用链是同步阻塞的

**风险点**：
- 步骤数多、外部调用慢（WEBHOOK、邮件发送）时，单请求/单任务耗时可能过长
- 可能触及 HTTP 请求超时、数据库连接池占用时间过长等边界问题
- 仅在遇到 `DELAY` 或 `WAIT_FOR_EVENT` 步骤时才会中断同步链，通过队列异步推进
- 代码依据：[WorkflowExecutionService.ts#L1165-L1168](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1165-L1168)

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

**延迟上限**：
- Schema 层面限制最大 365 天
- 按单位分别校验：`minutes` ≤ 525600，`hours` ≤ 8760，`days` ≤ 365
- 代码依据：[index.ts#L277-L292](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/shared/src/schemas/index.ts#L277-L292)

**时区问题**：
- 延迟计算使用 `Date.now() + delayMs`，基于服务器本地时间的毫秒时间戳
- **不涉及时区转换**，也不考虑夏令时、工作日/自然日等语义
- 如果配置为 "1 day"，实际是精确的 24 × 60 × 60 × 1000 毫秒，而非日历意义上的「一天」
- 代码依据：[WorkflowExecutionService.ts#L606-L623](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L606-L623)

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
  ├─ 标记 StepExecution 为 WAITING（stepExecution.executeAfter = timeoutDate）
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

**WAIT_FOR_EVENT 超时双重防御机制**：
WAIT_FOR_EVENT 的「事件到达 vs 超时」两个唤醒路径存在竞态，引擎通过两层防御避免重复推进：

| 防御层 | 实现机制 | 时机 | 代码位置 |
|-------|---------|------|---------|
| 第一层（队列层） | 事件到达时调用 `QueueService.cancelWorkflowTimeout(stepExecutionId)`，尝试从 BullMQ 中移除尚未执行的超时 job | handleEvent 中 | [WorkflowExecutionService.ts#L455-L456](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L455-L456) |
| 第二层（状态校验层） | `processTimeout()` 在处理超时前**再次查询并校验** `stepExecution.status === WAITING`，若状态已变（事件先到了）则直接 return，不推进 | processTimeout 开头 | [WorkflowExecutionService.ts#L326-L329](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L326-L329) |

**第一层失效场景**：
- 超时 job 刚好已经开始执行（BullMQ 已从队列取出，cancel 无法终止正在运行的 job）
- 网络抖动导致 Redis cancel 操作未生效
- 上述情况下必须依赖第二层防御

**两层防御的协作逻辑**：
```
事件到达路径 (handleEvent)                超时路径 (processTimeout)
       │                                          │
       ├─ 更新 status = COMPLETED (DB)            │
       ├─ cancelWorkflowTimeout (第一层)           │
       │    └─ 最好情况：超时job未开始 → 成功取消   │
       │                                          │
       │                      超时 job 已在处理中  │
       │                          ├─ 查 status = COMPLETED (非 WAITING)
       │                          └─ return（第二层拦截成功）
       │                                          │
       ▼                                          ▼
   推进下游步骤                             不推进，安全退出
```

> 注意：虽然超时侧有双重防御，但**事件到达侧（handleEvent）缺乏同等的原子性保护**，详见 6.7 节。

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

## 六、错误恢复与边界风险

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

**拦截的 IPv4 网段**：
| 网段 | 说明 |
|------|------|
| `127.0.0.0/8` | 回环地址 |
| `10.0.0.0/8` | 私有地址 A 类 |
| `172.16.0.0/12` | 私有地址 B 类 |
| `192.168.0.0/16` | 私有地址 C 类 |
| `169.254.0.0/16` | 链路本地 / 云元数据 |
| `100.64.0.0/10` | 运营商共享地址空间（CGNAT） |
| `0.0.0.0/8` | 本网络 |
| `224.0.0.0+` | 组播及以上保留段 |

**拦截的 IPv6 网段**：
| 前缀 | 说明 |
|------|------|
| `::1` | 回环地址 |
| `fe80:` | 链路本地地址 |
| `fc:` / `fd:` | 唯一本地地址（ULA） |
| `ff:` | 组播地址 |
| `::ffff:` | IPv4 映射地址（会转为 IPv4 校验） |

代码依据：[WorkflowExecutionService.ts#L831-L861](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L831-L861)

### 6.7 事件唤醒的原子性风险

**问题描述**：`handleEvent()` 函数在唤醒 WAIT_FOR_EVENT 步骤时**缺乏事务保护**，存在并发重复推进的风险。

**执行流程**（非原子）：
```
1. 查询所有 status=WAITING 的 StepExecution 列表（读）
2. for 循环逐个处理：
   a. 更新 StepExecution 为 COMPLETED（写）
   b. 取消超时任务
   c. 调用 processNextSteps() 推进（写 + 后续递归）
```

**风险场景**：
- 同一事件并发到达（如多次上报、重试风暴），两个请求同时读取到同一个 WAITING 的 StepExecution
- 两者都会更新为 COMPLETED，并都调用 `processNextSteps()` 推进
- 可能导致同一个等待步骤被重复推进，产生重复的下游步骤执行（如重复发邮件、重复调用 webhook）

**与超时侧的对比**：
- 超时路径（processTimeout）有「二次状态校验」作为兜底
- 事件到达路径（handleEvent）没有同等的状态检查，直接从 WAITING 更新为 COMPLETED 后立即推进
- 这是一种**不对称的防御**

**缓解措施**：
- 步骤执行的状态更新是独立数据库操作，但整体流程无事务包裹
- 幂等性依赖后续步骤自身的业务逻辑，而非引擎层面保证
- 代码依据：[WorkflowExecutionService.ts#L401-L462](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L401-L462)

### 6.8 UPDATE_CONTACT 的订阅副作用

**问题描述**：执行 `UPDATE_CONTACT` 步骤时，如果 `subscriptionAction` 导致订阅状态实际改变，会**触发额外的事件**，可能间接启动其他 workflow。

**副作用链路**：
```
executeUpdateContact()
  ├─ 更新 contact.subscribed 字段
  └─ 订阅状态变化时 → EventService.trackEvent()
      ├─ 事件名：contact.subscribed 或 contact.unsubscribed
      ├─ 该事件可能被其他 Workflow 的 WAIT_FOR_EVENT 步骤捕捉
      └─ 该事件也可能触发新的 Workflow 启动（triggerType=EVENT 且 eventName 匹配）
```

**注意**：
- 这种副作用是隐式的，在 workflow 编排层面不可见
- 可能引发意料之外的级联触发
- 代码依据：[WorkflowExecutionService.ts#L1069-L1076](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1069-L1076)

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

### ❗ 误判点 11：TRIGGER 步骤的入边与 EXIT 步骤的出边校验
**误解**：TRIGGER 步骤不允许有入边、EXIT 步骤不允许有出边，系统会强制校验。
**实际**：代码层面**没有对步骤的入出边数量做强制校验**。TRIGGER 步骤理论上可以接入边，EXIT 步骤理论上可以接出边（但出边永远不会被遍历到，因为 EXIT 步骤执行后直接标记执行结束，不会调用 processNextSteps）。
- TRIGGER 步骤仅作为执行入口被 `startExecution()` 使用
- EXIT 步骤执行后直接返回 `{exited: true}`，后续 transition 不会被处理
**代码依据**：[WorkflowExecutionService.ts#L796-L825](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L796-L825)

### ❗ 误判点 12：exitReason 的取值来源与默认值
**误解**：exitReason 只有用户取消时才设置，或者所有退出状态都有统一的默认值。
**实际**：exitReason 的取值因退出方式而异，且部分终态不设置 exitReason：
- `COMPLETED`: 不设置 exitReason（null）
- `EXITED`: 取 EXIT 步骤的 `config.reason`，无配置时默认 `"exit_step"`
- `CANCELLED`（用户手动）: `"Cancelled by user"` 或 `"Cancelled by user (bulk cancel)"`
- `CANCELLED`（项目禁用）: `"Project disabled"`
- `FAILED`: 不设置 exitReason，错误信息记录在 StepExecution.error 中
**代码依据**：[WorkflowExecutionService.ts#L816](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L816)

### ❗ 误判点 13：调度触发类型的可用性
**误解**：SCHEDULE 触发类型已经实现，可以配置 cron 定时触发工作流。
**实际**：`WorkflowTriggerType.SCHEDULE` 仅在枚举和 schema 注释中声明，**没有任何实际调度执行逻辑**——没有 cron 调度器、没有定时扫描任务、没有对应的队列 worker。
**代码依据**：[schema.prisma#L681-L685](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/packages/db/prisma/schema.prisma#L681-L685)

### ❗ 误判点 14：WEBHOOK 的 method 字段支持模板变量
**误解**：WEBHOOK 配置的所有字段都支持 `{{变量}}` 模板渲染。
**实际**：`url`、`headers` 的值、`body` 的字符串值都支持模板渲染，但 `method` 字段**不参与模板渲染**，始终作为字面 HTTP 动词使用（防止模板注入）。
**代码依据**：[WorkflowExecutionService.ts#L940-L954](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L940-L954)

### ❗ 误判点 15：UPDATE_CONTACT 步骤仅修改联系人数据
**误解**：UPDATE_CONTACT 步骤只会更新联系人的属性和订阅状态，不会产生其他影响。
**实际**：当 `subscriptionAction` 导致订阅状态实际改变时，会**隐式触发 `contact.subscribed` 或 `contact.unsubscribed` 事件**，可能唤醒其他工作流的 WAIT_FOR_EVENT 步骤，甚至启动新的工作流实例。
**代码依据**：[WorkflowExecutionService.ts#L1069-L1076](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1069-L1076)

### ❗ 误判点 16：事件唤醒是原子操作
**误解**：`handleEvent()` 唤醒等待步骤是原子的，并发情况下也不会重复执行。
**实际**：`handleEvent()` 先查询 WAITING 列表再逐个更新推进，**没有数据库事务包裹**。同一事件并发到达时，两个请求可能同时读到同一个 WAITING 的 StepExecution，导致重复推进下游步骤。
**代码依据**：[WorkflowExecutionService.ts#L401-L462](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L401-L462)

### ❗ 误判点 17：DELAY 的 "days" 是自然日
**误解**：配置 "1 day" 延迟就是到下一天的同一时间（考虑时区和夏令时）。
**实际**：延迟使用 `Date.now() + delayMs` 计算，`1 day = 24 × 60 × 60 × 1000` 毫秒，**不涉及时区转换，不考虑夏令时**，就是精确的 24 小时时长。
**代码依据**：[WorkflowExecutionService.ts#L606-L623](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L606-L623)

### ❗ 误判点 18：CONDITION 操作符未知时会安全跳过
**误解**：配置了拼写错误的操作符（如 `equal` 少打了 s），系统会优雅降级或跳过该条件。
**实际**：`evaluateCondition()` 的 switch 遇到未知操作符会**直接 `throw new Error`**，这是硬错误，会导致整个步骤和执行实例标记为 FAILED。
**代码依据**：[WorkflowExecutionService.ts#L1260-L1262](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1260-L1262)

### ❗ 误判点 19：CONDITION multi 模式必须配置 default 分支
**误解**：多分支（switch/case）模式需要在配置中显式声明 default 分支，否则所有 case 都不匹配时会报错。
**实际**：Schema 中**没有 default 的配置项**，default 是执行层的**隐式兜底**——遍历完所有 branches 仍无匹配时自动返回 `branch='default'`。只需在 Transition 层配置一条 `condition.branch='default'` 的出边即可。
**代码依据**：[WorkflowExecutionService.ts#L765-L772](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L765-L772)

### ❗ 误判点 20：WEBHOOK 的变量 id/email 始终是联系人的真实值
**误解**：模板变量中的 `{{id}}` 和 `{{email}}` 永远等于联系人 ID 和邮箱。
**实际**：`variables` 对象使用「字面量在前、展开在后」的构造方式。如果 `contact.data`（自定义数据）或 `execution.context`（事件 payload）中**恰好包含同名顶层键**，会通过展开运算符**覆盖**前面的系统变量。
**代码依据**：[WorkflowExecutionService.ts#L944-L954](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L944-L954)

### ❗ 误判点 21：WEBHOOK 未配置 body 时请求为空
**误解**：不配置 `body` 字段，请求体就是空的（或空对象 `{}`）。
**实际**：未配置 `body` 时，引擎会自动构造一个**标准化默认载荷**，包含 `contact`（email/subscribed/data）、`workflow`（id/name）、`execution`（id/startedAt）、`event`（触发事件 payload）四大部分。此外，对 `GET` 请求即使配置了 body 也不会发送。
**代码依据**：[WorkflowExecutionService.ts#L964-L989](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L964-L989)

### ❗ 误判点 22：执行上下文会被后续事件更新
**误解**：工作流被中途的 WAIT_FOR_EVENT 步骤的事件唤醒后，新事件的数据会更新到 `execution.context` 中，后续步骤可以读取最新值。
**实际**：`execution.context` 仅在**创建 Execution 时（启动瞬间）一次性写入**，之后永不修改。后续唤醒事件的 data 只写入该 WAIT_FOR_EVENT 步骤的 `stepExecution.output.eventData` 字段，不影响全局 context。
**代码依据**：[EventService.ts#L470-L477](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/EventService.ts#L470-L477)

### ❗ 误判点 23：超时和事件到达都有同等的防御
**误解**：WAIT_FOR_EVENT 的两条唤醒路径（超时、事件到达）都有同样的原子性保障。
**实际**：超时路径有「队列取消 + 二次状态校验」的**双重防御**，而事件到达路径（handleEvent）**没有同等的二次校验**——直接从 DB 读 WAITING 列表就推进，存在并发重复推进的风险（即 6.7 节的原子性问题）。
**代码依据**：[WorkflowExecutionService.ts#L326-L329](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L326-L329) 对比 [L401-L462](file:///d:/fz/0601-1/solo-dogfeeding/code/54-plunk/apps/api/src/services/WorkflowExecutionService.ts#L401-L462)

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
                ├─ 无下一步 → COMPLETED（exitReason 空）
                ├─ EXIT 步骤 → EXITED（exitReason = config.reason || "exit_step"）
                ├─ 异常 → FAILED（exitReason 空，错误在 step.error）
                └─ 用户取消 / 项目禁用 → CANCELLED（exitReason 有明确值）
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
`Event → Workflow 匹配 → Execution 创建（写入 context，之后不变）→ 递归执行 Steps → Transitions 路由 → 完成/退出`

### 异步边界
1. **DELAY**：立即完成 Step，通过 BullMQ 延迟调度下一步
2. **WAIT_FOR_EVENT**：进入 WAITING，双路径唤醒（事件到达 / 超时）
   - 超时侧：双重防御（队列取消 + 状态二次校验）
   - 事件侧：缺乏原子性，有并发重复推进风险
3. **startExecution**： fire-and-forget 模式，不等待执行结果

### 错误恢复
- BullMQ 指数退避重试（3 次）
- 项目禁用自动取消
- 失败状态持久化 + 通知
- Webhook SSRF 多层防护（IPv4+IPv6 + 运营商共享段）
- WAIT_FOR_EVENT 超时侧的双重防御

### 已知风险与限制
| 风险点 | 说明 |
|--------|------|
| 同步递归长链 | 多步骤同步执行可能超时，遇 DELAY/WAIT_FOR_EVENT 才中断 |
| 事件唤醒非原子 | handleEvent 无事务，并发可能重复推进（不对称：超时侧有防御，事件侧无） |
| UPDATE_CONTACT 副作用 | 改订阅会触发事件，可能级联启动其他 workflow |
| SCHEDULE 未实现 | 仅声明枚举，无实际调度逻辑 |
| DELAY 无时区 | 按毫秒精确计算，不考虑夏令时和自然日 |
| 变量展开覆盖 | SEND_EMAIL/WEBHOOK 的 variables 浅展开可能被 contact.data 或 context 覆盖系统变量 |
| 未知操作符硬错误 | CONDITION 配置错误操作符直接抛错，标记 FAILED |
| 上下文只读 | execution.context 启动后不再更新，新唤醒事件数据仅存 StepExecution.output |

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
| SCHEDULE 触发 | 仅声明未实现，无调度器/扫描器/worker |
| WEBHOOK method | 不参与模板渲染，始终为字面量动词 |
| WEBHOOK 未配 body | 自动发送标准化默认载荷（contact+workflow+execution+event） |
| WEBHOOK GET 请求 | 即使配置了 body 也不发送请求体 |
| UPDATE_CONTACT | 改订阅会触发额外事件，可能级联启动其他 workflow |
| 事件唤醒 | 非原子操作，并发有重复推进风险（超时侧有防御，事件侧无） |
| DELAY days | 精确 24 小时毫秒数，无时区/夏令时/自然日概念 |
| CONDITION 未知操作符 | 直接 throw Error，步骤和执行均标记 FAILED |
| CONDITION multi default | 隐式兜底，无需在 Schema 中显式声明 default |
| SEND_EMAIL/WEBHOOK 变量 | 浅展开覆盖，contact.data/context 可能覆盖 id/email 等系统键 |
| execution.context | 启动时一次性写入，后续永不修改；新唤醒事件数据存 StepExecution.output |
| WAIT_FOR_EVENT 唤醒防御 | 不对称：超时侧双重防御，事件侧无原子性保障 |
| TRIGGER/EXIT 入出边 | 无强制校验，EXIT 出边不执行 |
| exitReason 取值 | 不同终态来源不同，COMPLETED/FAILED 为空 |
