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
