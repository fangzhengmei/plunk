# Webhook 配置与投递 — 模块协作梳理

> 分析范围：事件产生、Webhook 注册（即 Workflow WEBHOOK 步骤配置）、队列投递、错误记录
> 架构分层：前端/入口层 ↔ 服务层 ↔ 队列/工作进程层 ↔ 存储层

---

## 一、总体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           前端 / 入口层                                  │
│  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────────┐  │
│  │ WebhookStepDialog│   │ Workflows API    │   │ Events/Webhooks Ctrl │  │
│  │ (配置 WEBHOOK    │   │ (增删改工作流、   │   │ (接收SNS/Stripe回   │  │
│  │  步骤的URL/方法/ │   │  步骤、连线)      │   │  调、用户track API)  │  │
│  │  Headers)        │   │                  │   │                      │  │
│  └────────┬─────────┘   └────────┬─────────┘   └──────────┬───────────┘  │
└───────────┼──────────────────────┼─────────────────────────┼──────────────┘
            │                      │                         │
            ▼                      ▼                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                             服务层                                       │
│  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────────┐  │
│  │ WorkflowService  │   │ EventService     │   │ WorkflowExecution-   │  │
│  │ (CRUD工作流+步骤) │──▶│ (事件入库+触发)  │──▶│ Service              │  │
│  │                  │   │                  │   │ (步骤引擎/模板渲染/  │  │
│  └──────────────────┘   └──────────────────┘   │ safeFetch SSRF防护) │  │
│                                                └──────────┬───────────┘  │
└───────────────────────────────────────────────────────────┼──────────────┘
                                                            │
                                  ┌─────────────────────────┤
                                  ▼                         ▼
                        ┌──────────────────┐     ┌──────────────────────┐
                        │   QueueService   │     │   NtfyService        │
                        │ (BullMQ队列封装) │     │ (失败/异常通知)      │
                        └────────┬─────────┘     └──────────────────────┘
                                 │
                                 ▼
                ┌──────────────────────────────────────┐
                │         队列 / 工作进程层             │
                │  workflow-processor-queue.ts (Worker)│
                │  worker.ts (统一启动所有Worker)      │
                └──────────────────┬───────────────────┘
                                   │
                                   ▼
                ┌──────────────────────────────────────┐
                │             存储层 (PostgreSQL)       │
                │  Workflow / WorkflowStep / Transition│
                │  WorkflowExecution / StepExecution   │
                │  Event / Email / ApiRequest          │
                │  (Redis: 工作流缓存 + BullMQ 队列)   │
                └──────────────────────────────────────┘
```

---

## 二、事件产生

### 事件产生的三条路径

| 路径 | 产生位置 | 典型事件 |
|------|---------|---------|
| **外部 Webhook 回调** | 入口层 Controllers | `email.delivered`, `email.opened`, `email.clicked`, `email.bounce`, `email.complaint`, `email.received` |
| **用户 REST API 调用** | 入口层 `Events.track` | 任意自定义事件（如 `user.signup`, `purchase.completed`） |
| **工作流内部触发** | 服务层 WorkflowExecutionService | `contact.subscribed`, `contact.unsubscribed` |

### 前端/入口层逻辑

**文件：[Webhooks.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Webhooks.ts)**

- `POST /webhooks/sns`（[L36-L477](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Webhooks.ts#L36-L477)）
  - SNS 签名验证（`SecurityService.verifySnsSignature`）
  - SNS SubscriptionConfirmation 自动确认（+ SSRF 防护：URL 必须是 `sns.<region>.amazonaws.(com|eu)` 的 HTTPS）
  - SES 邮件事件解析：Delivery / Open / Click / Bounce / Complaint → 对应 `email.*` 事件
  - SES 入站邮件解析：Received → `email.received` 事件，附带邮件正文、安全检测结果
  - 业务副作用：Permanent Bounce/Complaint 退订联系人、更新 Email 状态、计费计数

- `POST /webhooks/incoming/stripe`（[L483-L789](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Webhooks.ts#L483-L789)）
  - Stripe webhook 签名校验（`stripe.webhooks.constructEvent`）
  - 处理结账、发票、订阅、欺诈预警等事件（**不产生 Event 记录，只更新业务表**）

**文件：[Events.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Events.ts)**

- `POST /events/track`（[L14-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Events.ts#L14-L28)）
  - 需 JWT 认证 + 邮箱验证
  - 接收 `{name, contactId?, emailId?, data?}`，直接调用 `EventService.trackEvent`

### 服务层逻辑

**文件：[EventService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/EventService.ts)**

核心方法 `trackEvent`（[L22-L47](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/EventService.ts#L22-L47)）：
1. **写入 Event 表**：`projectId`, `name`, `contactId`, `emailId`, `data(JSON)`
2. **触发匹配的工作流**：`triggerWorkflows()` → 从 Redis 缓存/DB 查 `triggerType=EVENT` 且 `triggerConfig.eventName === 当前事件名` 的启用工作流 → `startWorkflowForContact`
3. **唤醒等待该事件的工作流**：`WorkflowExecutionService.handleEvent()` → 找到所有 `WAIT_FOR_EVENT` 且状态为 WAITING 的步骤执行

工作流内部产生事件（UPDATE_CONTACT 步骤）：
- [WorkflowExecutionService.ts L1070-L1076](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1070-L1076)：订阅状态变更时 `EventService.trackEvent('contact.subscribed'/'contact.unsubscribed')`

### 存储侧逻辑

**Prisma Model：Event**（[schema.prisma L600-L630](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/packages/db/prisma/schema.prisma#L600-L630)）

关键字段：
- `name`：事件名（如 `email.opened`、`user.signup`）
- `data Json?`：任意载荷（带 GIN 索引便于 JSON 查询）
- `projectId / contactId / emailId`：关联外键
- 索引：`(projectId, name, createdAt)`、`(contactId, name, createdAt)`、`data` 的 GIN 索引

Redis 缓存：
- `Keys.Workflow.enabled(projectId)`：缓存项目的启用工作流列表，5 分钟 TTL（[EventService L351-L391](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/EventService.ts#L351-L391)）

---

## 三、Webhook 注册（Workflow WEBHOOK 步骤配置）

> 注：此处"Webhook 注册"指用户在工作流中添加 WEBHOOK 类型步骤，定义当工作流运行到该步时向哪个外部 URL 发送什么请求。非平台对外暴露的 webhook 端点。

### 前端层逻辑

**文件：[WebhookStepDialog.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/web/src/components/workflow-steps/WebhookStepDialog.tsx)**

UI 组成：
1. **步骤名称**（`name`，用户可编辑）
2. **请求预览折叠面板**（展示默认 payload 结构示例 + 文档链接）
3. **URL 输入**（支持 `{{变量}}` 模板，如 `https://api.example.com/{{userId}}`）
4. **Method 下拉**：GET / POST / PUT / PATCH / DELETE（默认 POST）
5. **Headers 键值对编辑器**：可动态增删，Value 支持 `{{变量}}` 模板

数据提交（[L32-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/web/src/components/workflow-steps/WebhookStepDialog.tsx#L32-L56)）：
- 空值校验：URL 必填
- Headers 过滤掉空 key/空 value 的条目
- 通过 `useStepUpdate` hook 调用 `PATCH /workflows/:id/steps/:stepId`

### 入口层逻辑

**文件：[Workflows.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Workflows.ts)**

相关端点：
- `POST /workflows`（[L78-L102](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Workflows.ts#L78-L102)）：创建工作流，自动生成 TRIGGER 步骤
- `POST /workflows/:id/steps`（[L176-L202](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Workflows.ts#L176-L202)）：添加 WEBHOOK 步骤，body 含 `{type: 'WEBHOOK', name, position, config, autoConnect}`
- `PATCH /workflows/:id/steps/:stepId`（[L208-L229](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Workflows.ts#L208-L229)）：更新 WEBHOOK 步骤配置

全部要求：JWT 认证 + 邮箱验证。

### 服务层逻辑

**文件：[WorkflowService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowService.ts)**

- `addStep()`（[L382-L443](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowService.ts#L382-L443)）：写入 `WorkflowStep`，若 `autoConnect=true` 自动从最后一个叶子步骤创建 Transition
- `updateStep()`（[L448-L507](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowService.ts#L448-L507)）：工作流启用中有活跃执行时，**禁止修改 config/templateId**（只能改 name/position），防止运行中行为变化

**配置 Schema 验证：[@plunk/shared/schemas](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/packages/shared/src/schemas/index.ts#L343-L348)**

```typescript
webhook: z.object({
  url: z.string().url(),                              // 合法 URL 校验
  method: z.enum(['GET','POST','PUT','PATCH','DELETE']).default('POST'),
  headers: z.record(z.string()).optional(),           // {key: value} 字符串映射
  body: jsonSchema.optional(),                        // 任意 JSON，支持 {{vars}}
})
```

### 存储侧逻辑

**Prisma Model：WorkflowStep**（[schema.prisma L369-L409](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/packages/db/prisma/schema.prisma#L369-L409)）
- `type: WorkflowStepType` 枚举值 `WEBHOOK`
- `config Json`：存储 `{url, method, headers?, body?}`
- `workflowId` 外键关联工作流
- `incomingTransitions / outgoingTransitions`：DAG 图的边（WorkflowTransition 表）

---

## 四、队列投递

分为两个层次：
1. **工作流步骤的异步队列投递**（BullMQ + Redis）——用于延迟步骤、超时回调
2. **WEBHOOK 步骤执行时的 HTTP 投递**（`WorkflowExecutionService.executeWebhook` + SSRF-safe fetch）

### 队列/工作进程层逻辑

**文件：[QueueService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/QueueService.ts)**

`workflowQueue`（BullMQ Queue，[L73-L84](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/QueueService.ts#L73-L84)）：
- 连接 Redis，3 次重试 + 指数退避（初始 2 秒）
- 保留最近 1000 条完成 / 5000 条失败 Job

核心投递方法：
- `queueWorkflowStep(executionId, stepId, delay?)`（[L230-L243](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/QueueService.ts#L230-L243)）：DELAY 步骤用 `delay` 参数在指定时间后唤醒
- `queueWorkflowTimeout(executionId, stepId, stepExecutionId, timeoutMs)`（[L248-L262](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/QueueService.ts#L248-L262)）：WAIT_FOR_EVENT 步骤的超时兜底
- `cancelWorkflowTimeout(stepExecutionId)`（[L267-L275](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/QueueService.ts#L267-L275)）：事件到达后取消对应超时 Job

**文件：[workflow-processor-queue.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/jobs/workflow-processor-queue.ts)**

BullMQ Worker（并发 10，[L13-L35](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/jobs/workflow-processor-queue.ts#L13-L35)）：
- Job type = `process-step`：调用 `WorkflowExecutionService.processStepExecution(executionId, stepId)`
- Job type = `timeout`：调用 `WorkflowExecutionService.processTimeout(executionId, stepId, stepExecutionId)`
- 监听 `completed` / `failed` / `error` 事件，输出 signale 日志

**文件：[worker.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/jobs/worker.ts)**
- 作为独立进程启动，初始化包括 workflowWorker 在内的 10 个 Worker
- 注册 SIGINT/SIGTERM 优雅关闭钩子

### WEBHOOK HTTP 投递（服务层核心）

**文件：[WorkflowExecutionService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts)**

`executeWebhook()`（[L920-L1006](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L920-L1006)）：

**Step 1 — 配置解析**
```typescript
const {url, method, headers, body} = WorkflowStepConfigSchemas.webhook.parse(config);
```

**Step 2 — 变量作用域**（比 SEND_EMAIL 多一个 `event` 命名空间）
```
variables = {
  id, email, ...contactData, ...executionContext,  // 同 SEND_EMAIL
  data: contactData,
  event: context,                // ✅ webhook 独有：触发事件的原始 payload
  unsubscribeUrl, subscribeUrl, manageUrl
}
```
- `method` **故意不渲染**（防止模板注入篡改 HTTP 动词）
- `url`、每个 header value、body 中所有字符串叶子节点都走 `renderTemplate` / `renderJsonTemplate`（递归遍历数组/对象）

**Step 3 — 默认 payload**（用户未指定 body 时自动发送）
```json
{
  "contact": { "email", "subscribed", "data": {...} },
  "workflow": { "id", "name" },
  "execution": { "id", "startedAt" },
  "event": { /* 触发事件的 data */ }
}
```

**Step 4 — SSRF-safe HTTP 请求**（`safeFetch` [L871-L908](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L871-L908)）
1. 只允许 `http:` / `https:` 协议
2. 手工 `dns.lookup(hostname)` 获取 IP，通过 `isPrivateIp()`（[L831-L865](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L831-L865)）拒绝：
   - IPv4：127.0.0.0/8、10.0.0.0/8、172.16.0.0/12、192.168.0.0/16、169.254.0.0/16（云元数据）、100.64.0.0/10、0.0.0.0/8、224.0.0.0+
   - IPv6：`::1`、`fe80:*`（link-local）、`fc*`/`fd*`（unique local）、`ff*`（多播）
   - 先剥掉 `::ffff:` IPv4 映射前缀
3. `redirect: 'manual'`，最多跟随 5 次重定向，**每一跳都重新做 DNS + IP 校验**
4. `AbortSignal.timeout(10_000)` 10 秒超时

**Step 5 — 返回值**：`{url, method, statusCode, success, response}`（响应体 JSON 解析失败则原样返回文本）

---

## 五、错误记录

### 入口层：外部系统回调的错误处理

**[Webhooks.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Webhooks.ts)**

| 场景 | 处理方式 | 理由 |
|------|---------|------|
| SNS 签名验证失败 | 返回 403 | 拒绝非法请求 |
| SNS SubscribeURL SSRF 校验失败 | 返回 400 | 防止被利用打内网 |
| SES 事件处理异常 | **返回 200**（[L473-L476](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Webhooks.ts#L473-L476)） | SNS 会无限重试 4xx/5xx，必须 200 确认收到 |
| Stripe 签名验证失败 | 返回 400 | Stripe 有有限重试 + 签名必须严格验证 |
| Stripe 事件处理异常 | 返回 400 | Stripe 会重试，允许排查后重试 |

**REST API 层：[Events.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Events.ts) / [Workflows.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/controllers/Workflows.ts)**
- 全局 `@CatchAsync` 装饰器（`asyncHandler`）捕获异常，统一转为 HTTP 错误响应
- 参数缺失返回 400，资源不存在返回 404，并发冲突返回 409

### 服务层：步骤执行错误记录

**[WorkflowExecutionService.processStepExecution()](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L254-L298)**

```
try { executeStep() }
catch (error) {
  1. WorkflowStepExecution.status = FAILED
     .error = error.message
     .completedAt = now
  2. WorkflowExecution.status = FAILED
     .completedAt = now
  3. NtfyService.notifyWorkflowExecutionFailed()  // 管理员通知
  4. throw error  // 触发 BullMQ 重试
}
```

**WEBHOOK 步骤的错误（未单独 try-catch，走父级）：**
- URL Schema 错误（非 http/https）
- DNS 解析失败
- 解析到内网 IP（SSRF 拒绝）
- 重定向次数超限（>5 次）
- 10 秒超时
- 目标返回任何 HTTP 状态码都**不抛错**——`response.ok` 只作为 `success` 字段写入 output，2xx 和 5xx 都算步骤"完成"

### 队列层：BullMQ 重试机制

QueueService 中 `workflowQueue` 默认策略（[L75-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/QueueService.ts#L75-L83)）：
- `attempts: 3`：最多执行 4 次（1 次初始 + 3 次重试）
- `backoff: { type: 'exponential', delay: 2000 }`：第 1 次失败等 2s，第 2 次等 4s，第 3 次等 8s
- 超过次数后 Job 进入 `failed` 状态，保留在 Redis 中最多 5000 条

Worker 事件监听（[workflow-processor-queue.ts L37-L47](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/jobs/workflow-processor-queue.ts#L37-L47)）：
- `completed` / `failed` / `worker error` 全部 signale 日志输出（可接入 ELK/CloudWatch）

### 存储侧：错误持久化位置

| 错误类型 | 存储位置 | 字段 |
|---------|---------|------|
| **工作流步骤执行错误** | `WorkflowStepExecution` | `status=FAILED`, `error`(String), `completedAt` |
| **工作流执行失败** | `WorkflowExecution` | `status=FAILED`, `exitReason`, `completedAt` |
| **WEBHOOK 请求状态** | `WorkflowStepExecution.output`(Json) | `{statusCode, success, response}` |
| **邮件发送错误** | `Email` | `status=FAILED`, `error` |
| **API 请求错误** | `ApiRequest` | `statusCode`, `errorCode`, `errorMessage`, `duration` |
| **事件 + 邮件历史** | `Event` + `Email` | 全量保留，便于排查 WEBHOOK payload 是否正确 |

---

## 六、分层对照表

| 关注点 | 前端/入口层 | 服务层 | 队列/Worker 层 | 存储层 |
|--------|------------|--------|---------------|--------|
| **事件产生** | Webhooks.ts（SNS/Stripe 解析）<br>Events.ts（POST /events/track）<br>✅ 签名验证、参数校验 | EventService.trackEvent()<br>✅ 写 Event 表<br>✅ 触发新工作流<br>✅ 唤醒等待步骤 | 不涉及 | Event 表 (PostgreSQL)<br>工作流缓存 (Redis) |
| **Webhook 注册** | WebhookStepDialog.tsx（UI 表单）<br>Workflows.ts（POST/PATCH 步骤端点）<br>✅ 参数格式 | WorkflowService.addStep/updateStep<br>✅ 活跃执行保护<br>✅ autoConnect 连线<br>WorkflowStepConfigSchemas.webhook（Zod 验证） | 不涉及 | WorkflowStep.config(Json)<br>WorkflowTransition（DAG 边） |
| **队列投递** | 不涉及 | QueueService.queueWorkflowStep / Timeout<br>✅ BullMQ Job 构造<br>✅ delay/cancel<br>WorkflowExecutionService.executeWebhook<br>✅ 变量渲染<br>✅ SSRF-safe fetch | workflow-processor-queue.ts<br>✅ 拉取 Job<br>✅ 调用 processStepExecution / processTimeout<br>worker.ts（进程管理） | BullMQ = Redis Streams/Hashes<br>WorkflowExecution.status<br>StepExecution.status/output |
| **错误记录** | Webhooks.ts（SNS 返回 200 防重试）<br>@CatchAsync（HTTP 错误码映射） | processStepExecution catch 块<br>✅ FAILED 状态更新<br>✅ NtfyService 通知<br>safeFetch（防 SSRF 抛错） | BullMQ attempts+backoff 重试<br>completed/failed 日志 | WorkflowStepExecution.error<br>WorkflowExecution.status=FAILED<br>Email.error<br>ApiRequest.error* |

---

## 七、关键安全设计

1. **SSRF 双重防护**（[WorkflowExecutionService.ts L831-L908](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L831-L908)）
   - SNS SubscribeURL 阶段：正则限制主机名
   - WEBHOOK 阶段：DNS 解析后逐 IP 范围比对，每跳重定向重新校验

2. **活跃执行保护**（[WorkflowService.ts L472-L488](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowService.ts#L472-L488)）
   - 工作流启用时有 RUNNING/WAITING 执行 → 禁止修改步骤 config/templateId，删除步骤/连线需先 disable workflow

3. **方法不模板化**：`method` 字段故意不走 `renderTemplate`，避免 `{{shell_injection}}` 扩展出非预期 HTTP 动词

4. **SNS 幂等响应**：业务处理失败也返回 200，避免 SNS 重打压垮服务（错误记入 signale 日志）

---

## 八、WAIT_FOR_EVENT：事件唤醒 vs 队列超时取消的并发协调

### 两条竞态路径

WAIT_FOR_EVENT 步骤从进入 WAITING 到最终推进，存在两条并行竞争的唤醒路径，两者共享同一份 `StepExecutionStatus.WAITING` 的守卫。

```
     ┌──事件到达路径──┐            ┌──超时路径──┐
     │ EventService    │            │ BullMQ     │
     │ .handleEvent()  │            │ Worker     │
     └────────┬────────┘            └─────┬──────┘
              ▼                           ▼
     prisma.findUnique(stepExId)   prisma.findUnique(stepExId)
              │                           │
              └──────────┬────────────────┘
                         ▼
              status === WAITING ?
              (竞态消解的核心守卫)
               ┌────┴────┐
               ▼         ▼
              YES        NO → return（对方已处理）
               ▼
         prisma.update(status=COMPLETED)
         ┌────────────┬────────────────────┐
         │ handleEvent│ processTimeout     │
         │ (事件赢)   │ (超时赢)           │
         │ output: {  │ output: {          │
         │  eventName,│  timedOut: true,   │
         │  eventData │  eventName }       │
         │ }          │                    │
         └─────┬──────┴──────────┬─────────┘
               ▼                 ▼
     cancelWorkflowTimeout  按 timeout/fallback
     (L456)                 transition 推进
        │                     (L351-L358)
        ▼                          ▼
    processNextSteps ─── 语义等价，但代码不共用
```

### 代码落点

**路径 A：事件唤醒（先发）**
[WorkflowExecutionService.handleEvent()](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L401-L462)

```
L408 查 waitingExecutions 时已带条件 status=WAITING
     ↓ 循环内 eventName 精确匹配
L442 update StepExecution: status=COMPLETED, output={eventName, eventData, receivedAt}
L456 cancelWorkflowTimeout(stepExecutionId)  → QueueService L267-L275
     用 jobId=workflow-timeout-${stepExecutionId} 去 BullMQ 里 remove()
L459 processNextSteps(execution, step, {eventReceived: true})
```

**路径 B：超时唤醒（后发）**
[WorkflowExecutionService.processTimeout()](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L305-L396)

```
L307-L320 findUnique 拉 stepExecution 及其 outgoingTransitions
L327  if (status !== WAITING) return;   // 事件先到则直接退出
L332 update StepExecution: status=COMPLETED, output={timedOut: true, eventName}
L350-L358 找带 {branch: 'timeout'} 或 {fallback: true} 的 transition
L360-L395 若有 timeout/fallback 分支走它；否则按优先级走第一条；否则 COMPLETED 退出
           * 这里不调用 processNextSteps，而是内联推进 *
```

### 竞态消解的三重防线

| 层次 | 机制 | 位置 | 效果 |
|------|------|------|------|
| 第 1 层 | **状态守卫** | handleEvent L410（WHERE WAITING）+ processTimeout L327（IF WAITING） | 只有一个路径会进入"真正干活"的代码 |
| 第 2 层 | **cancelWorkflowTimeout** | handleEvent L456 对应 QueueService L267-L275 | 事件赢了之后撤掉超时 Job，防止未来再触发（对 **尚未开始执行** 的 timeout Job 有效） |
| 第 3 层 | **status 幂等** | processTimeout L327 兜底检查 | 即使 timeout Job 已经在执行中，remove() 无法阻止执行中的进程，也会因 status 已变 COMPLETED 安全退出 |

### 仍存在的边界（非 Bug，但要心中有数）

1. **TOCTOU 双写窗口**：findUnique(WAITING) 和 update(COMPLETED) 之间没有事务锁。若两条路径都过了 L327 的检查点（精确同帧的极度并发），后者的 update 会覆盖前者的 `output` 字段（`{eventData}` vs `{timedOut: true}`），但状态机推进最终只走一条（先到的调 processNextSteps/内联推进，后到的只会做无意义的 update）。
2. **cancel 对"已开始的 Job"无效**：BullMQ `job.remove()` 仅对 `delayed`/`waiting` 状态生效。如果 timeout Worker 已经在 `processTimeout` 的"前半段"，cancel 不会中止它，但第 3 层守卫保证了安全。
3. **JobId 精确性**：`workflow-timeout-${stepExecutionId}` 绑定的是 **步骤执行实例**（不是 workflow execution），支持多 WAIT_FOR_EVENT 步骤互不干扰。

---

## 九、工作流缓存 5 分钟刷新窗口与事件触发的一致性边界

### 缓存结构（只服务于"工作流启动"链路）

[EventService.triggerWorkflows()](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/EventService.ts#L351-L408)

| 项 | 值 | 说明 |
|----|----|------|
| Key | `Keys.Workflow.enabled(projectId)` | 项目级别隔离 |
| Value | `enabled=true AND triggerType=EVENT` 的 workflow[]，带 TRIGGER steps | JSON 序列化 |
| TTL | 300 s（5 分钟） | 写回用 `redis.setex`（L387） |
| 读取降级 | redis 失败 → 直接走 DB | 静默降级，warn 日志 |

### 主动失效的触发点（工作流变更时主动清缓存）

**[WorkflowService](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowService.ts)**：

| 操作 | 触发条件 | 代码位置 |
|------|---------|---------|
| `create()` | 创建时 `enabled=true` | L142-L144 |
| `update()` | enabled 字段有变化 OR workflow 是 enabled 且 triggerConfig 有变化 | L251-L258 |
| 其他（addStep / updateStep / deleteStep / addTransition 等） | **无显式失效** | — |

### 两条"触发工作流"路径的读一致性差异

```
                  新事件到达
                      │
           ┌──────────┴───────────┐
           ▼                      ▼
   triggerWorkflows()       handleEvent()
   (启动新工作流)            (唤醒 WAIT_FOR_EVENT)
           │                      │
           ▼                      ▼
    查 Redis 缓存 (TTL 300s)     直接查 DB
      命中→用缓存                实时读 workflowStepExecution
      未命中→查 DB→写回缓存       无缓存层
           │                      │
           └───────一致性边界不同──┘
```

**关键：** 工作流启动链路走缓存，步骤唤醒链路完全绕过缓存。这意味着：
- ✅ 更改步骤内的 `WAIT_FOR_EVENT.config.eventName` 后立即生效（不经过缓存）
- ❌ 更改 workflow 顶层的 `triggerConfig.eventName` 可能有 5 分钟延迟（缓存）
- ✅ workflow **禁用**的动作通常立刻生效（update 会 invalidate）
- ⚠️ 但分布式下有"读写并发写回"窗口（见下文）

### 一致性窗口的具体边界

| 场景 | 一致性行为 | 最坏延迟/风险 |
|------|-----------|-------------|
| 禁用工作流（enabled=false） | WorkflowService.update() 调 invalidate，缓存被清，下次 trigger 读 DB | 分布式下约 0~20ms 可生效 |
| 改 trigger 事件名（同 enabled=true） | WorkflowService.update() 调 invalidate，同上 | 同上 |
| A 实例: `setex(旧数据)` 与 B 实例: `del(缓存)` 交错 | **A del → B setex(旧)** 模式下旧数据可存活 5 分钟 | 最多 5 分钟的"已禁用/已改名的工作流仍被触发" |
| 新事件触发后 startWorkflowForContact（二次校验） | L420-L427 再读 DB 取 workflow 详情，但 **只检查有 TRIGGER step，不检查 enabled** | 已禁用的 workflow 只要缓存命中就真的会启动执行 |
| WAIT_FOR_EVENT 步骤的 eventName 变更 | handleEvent 实时查 DB，无缓存 | 0 ms |

---

## 十、步骤转移判断与状态机推进：执行落点 + 条件分支求值

### 步骤推进的三条"代码入口"

WORKFLOW 状态机的推进（currentStepId 前进）分散在三处代码，**不共用 processNextSteps**：

| 入口 | 场景 | 推进方式 | 代码位置 |
|------|------|---------|---------|
| **processStepExecution** 尾部 | 普通步骤（SEND_EMAIL/WEBHOOK/CONDITION/TRIGGER/UPDATE_CONTACT）执行完 | 调 `processNextSteps()` | [L253](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L253) |
| **handleEvent** 尾部 | WAIT_FOR_EVENT 因事件匹配而完成 | 调 `processNextSteps(execution, step, {eventReceived: true})` | [L459](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L459) |
| **processTimeout** 中部 | WAIT_FOR_EVENT 因超时而完成 | **内联推进（不调 processNextSteps）**：找 timeout/fallback 分支，手动 update execution，直接调 processStepExecution | [L350-L395](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L350-L395) |

**重要的语义不对称**：第三条路径（processTimeout）不经过 CONDITION 的 branch 匹配逻辑。如果 WAIT_FOR_EVENT 后面紧跟 CONDITION 节点，只有事件到达时能正常走分支判断；超时路径会按 timeout/fallback→firstTransition→COMPLETED 的优先级推进，不会执行 CONDITION stepResult.branch 匹配。

### processNextSteps 里的 Transition 选择逻辑

[WorkflowExecutionService.processNextSteps()](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1090-L1168)

```
L1095 取 outgoingTransitions（按 priority 升序）
L1097 transitions 为空 → workflow COMPLETED
L1113 for 循环按 priority 遍历
  ├─ L1117 transition.condition 为空 → 选中，break
  ├─ L1124 stepResult.branch 存在 && transition.condition.branch === stepResult.branch
  │     → CONDITION 步骤的分支匹配，选中，break
  ├─ L1137 evaluateTransitionCondition(condition, stepResult, execution)
  │     → 目前是占位函数，永远 false（L1268-L1276）
L1143 循环结束未选中 → workflow COMPLETED（无有效出口 = 结束）
L1156 update execution: {currentStepId, status=RUNNING}
L1167 processStepExecution(nextStepId)   ← 继续深度推进
```

### CONDITION 步骤的条件求值全链路

条件求值发生在 **executeCondition**（执行阶段）→ 返回 branch 字符串 → **processNextSteps**（转移阶段）用 branch 串匹配 transition.condition.branch。

**阶段 1：构造条件字段命名空间** [L733-L747](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L733-L747)

```typescript
fieldData = {
  contact: {email, subscribed},   // 只暴露核心字段
  data: contactData,              // 完整自定义 data（扁平对象）
  workflow: context,              // 其实是 execution.context（事件 payload）
  event: context,                 // 别名，双写
}
```

注意：这里不把字段"平展到顶层"（不像 SEND_EMAIL/WEBHOOK 的 variables scope）。条件求值要求显式写 `contact.email`、`data.plan`、`event.amount`，避免字段名覆盖冲突。

**阶段 2：字段路径解析 + 历史兼容层**
[resolveField()](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1181-L1202)

```
L1186 if (field.startsWith('contact.data.')) {
        normalizedField = field.substring(8)  // "contact.data.plan" → "data.plan"
      }
L1190 点号切分，逐层走对象引用
L1194 任何一层 undefined → 整个求值返回 undefined（不是抛错）
```

**阶段 3：条件运算符集合**
[evaluateCondition()](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1207-L1263)

核心原则：**"字段不存在则不匹配"**（`equals` 和 `exists` 系列除外）。

| 运算符 | 字段为 null/undefined 时 | 非空时逻辑 |
|--------|-------------------------|-----------|
| equals | `null===null` true，其余 false | `actual === expected`（严格相等） |
| notEquals | **直接 false** | `actual !== expected` |
| contains / notContains | **直接 false** | `String(x).includes(String(y))` |
| greaterThan/lessThan/≥/≤ | **直接 false** | `Number(x) op Number(y)` |
| exists | false | true |
| notExists | true | false |

**阶段 4：两种 CONDITION 模式**

- **Multi 模式（switch/case）** [L750-L773]：遍历 `parsed.branches[]` 按声明顺序逐个 evaluateCondition，**第一个命中**返回其 `branch.name`；全不命中返回 `branch = "default"`。
- **Binary 模式（if/else）** [L776-L790]：单条件，命中返回 `branch = "yes"`，未命中返回 `branch = "no"`。`actualValue=undefined` 时 JSON 里强制转 `null` 保存。

### 条件分支与 Transition 的绑定方式

分支结果通过 `stepResult.branch` 字符串流转：
- executeCondition 返回 `{..., branch: branch_id_or_name}`
- processNextSteps L1124 匹配 `transition.condition.branch === stepResult.branch`
- 前端在 Transition 上写 `condition: {branch: "yes"}` / `{branch: "premium"}` / `{branch: "timeout"}`

而 `evaluateTransitionCondition` [L1268-L1276] 目前是 **预留的占位钩子**，永远返回 false。这意味着除了"分支名精确匹配"，所有更复杂的"transition 级表达式"在当前版本都未启用。

---

## 十一、WEBHOOK 默认载荷四块拆分 & 变量作用域编排

### 默认载荷的四象限结构

当 webhook 步骤未配置 `body`（用户没自定义 payload）时，使用 [executeWebhook L964-L979](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L964-L979) 构造默认 payload：

```json
{
  "contact":   { "email", "subscribed", "data": {...} },   ← 接收方视角（给谁）
  "workflow":  { "id", "name" },                            ← 配置视角（哪条工作流）
  "execution": { "id", "startedAt" },                       ← 运行时视角（本次运行）
  "event":     execution.context                            ← 因果视角（为什么触发）
}
```

### 四块拆分的设计理由

| 拆分块 | 职责定位 | 典型使用场景 |
|--------|---------|-------------|
| **contact** | 邮件平台第一公民对象（= 订阅者） | `contact.email` 做收件人校验；`contact.data.plan` 读业务属性；`contact.subscribed` 做订阅状态去重 |
| **workflow** | 配置对象元信息（版本不变式） | 外部系统做"工作流级 webhook 统计报表"，按 workflow.id 聚合，name 做人读日志 |
| **execution** | 单次执行实例（版本递增） | 外部系统回查 execution.id 调 Plunk 的 API 查详情；startedAt 测"事件 → 投递"链路延迟 |
| **event** | 原始触发原因（因果链起点） | 订单 webhook 场景中，event.amount / event.orderId 是真正想传递给第三方的业务数据 |

**拆分的边界含义**：contact/workflow/execution/event 是语义正交的四个维度。event.amount 和 contact.data.amount 即使重名也不冲突，因为各自在独立命名空间。

### 三处"变量作用域"的编排对比

在 SEND_EMAIL（邮件模板渲染）、WEBHOOK（URL/Headers/Body 模板渲染）、CONDITION（条件字段取值）中，同一系统里有 **三种 scope 定义**，反映各步骤的"便利 vs 精确"取舍：

| scope 条目 | SEND_EMAIL `variables` [L553-L562] | WEBHOOK `variables` [L944-L954] | CONDITION `fieldData` [L739-L747] |
|-----------|------|------|------|
| 顶层 `id`, `email` | ✅ 直接 | ✅ 直接 | ❌（需写 `contact.email`） |
| 顶层平展 contactData | ✅ `...contactData`（`{{firstName}}` 直接写） | ✅ `...contactData` | ❌（需写 `data.firstName`） |
| 顶层平展 executionContext | ✅ `...executionContext` | ✅ `...executionContext` | ❌（需写 `event.xxx` / `workflow.xxx`） |
| `data.*` 命名空间（完整 contactData） | ✅ | ✅ | ✅ |
| `event.*` 命名空间（触发 payload） | ❌ | ✅ `event: context` | ✅ `event: context` |
| `workflow.*` 命名空间（触发 payload 别名） | ❌ | ❌（默认 payload 里有，渲染变量里没有） | ✅ `workflow: context` |
| subscribeUrl / unsubscribeUrl / manageUrl | ✅ | ✅ | ❌ |

#### 不对称背后的设计意图

1. **SEND_EMAIL 无 `event.*` scope**：邮件模板通常引用联系人信息；WEBHOOK 是"把触发事件广播给第三方"，必须暴露 event。这体现两种步骤语义差异：邮件是"和联系人沟通"，webhook 是"和系统同步"。
2. **CONDITION fieldData 不平展**：条件判断追求"显式和无歧义"，禁止 `{{email}}` 这种写法防止与 contact.data.email（同名自定义字段）冲突。
3. **WEBHOOK 渲染变量里无 `workflow.*`**：用户想在 URL/Headers 里插工作流 ID，不能写 `{{workflow.id}}`，但默认 payload 里有 workflow 块。渲染变量 vs 默认 payload 的不一致是 **当前版本的一个缺口**——如需要得用 `{{id}}`（当前 scope 是 contact.id，不推荐）或在前端自定义 body 时自己拼。
4. **顶层平展 + 命名空间双写并存**：`...contactData` 和独立 `data: contactData` 共存，既兼容旧模板 `{{firstName}}`，又提供 `{{data.firstName}}` 的规范写法，是渐进迁移的兼容层。

### 变量渲染的范围与深度

**执行顺序（executeWebhook L956-L979）：**

```
L956  renderedUrl      = renderTemplate(url, variables)            ← 只对字符串
L957-L961 renderedHeaders = Object.map(value → renderTemplate)    ← 只对 value，key 原样
L962  renderedBody     = renderJsonTemplate(body, variables)      ← 递归深渲染
L964  payload = renderedBody 或 默认的{contact,workflow,execution,event}
L988  GET 方法：body 无条件不发送（即使配置了 body 也静默丢弃）
```

**renderJsonTemplate 深度递归策略** [L1013-L1028](file:///d:/fz/0601-1/solo-dogfeeding/code/57-plunk/apps/api/src/services/WorkflowExecutionService.ts#L1013-L1028)：

```
string  → renderTemplate(字符串, vars)
array   → map 每项递归
object  → 键保留，值递归（键本身不做模板渲染）
number/bool/null → 原样返回（不做模板化，保证类型不变）
```

特别地，`method` 字段 **完全不走任何渲染**（L926 解析后直接透传到 safeFetch）。这是防止 `{{malicious}}` 模板被扩展成非预期 HTTP 动词的安全决策，在 WEBHOOK 的设计注释 L913-L918 中明确标出。
