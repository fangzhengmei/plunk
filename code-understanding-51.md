# 邮件发送队列与处理器 — 代码理解

## 一、整体架构概览

邮件发送系统采用 **"生产者-消费者"** 异步架构，核心链路如下：

```
业务入口 (Transactional/Campaign/Workflow)
       ↓
EmailService (创建 Email 记录 + 计费校验)
       ↓
QueueService.queueEmail (写入 BullMQ 队列)
       ↓
Redis (队列存储)
       ↓
email-processor Worker (BullMQ Worker 消费)
       ↓
SESService.sendRawEmail (AWS SES 发送)
       ↓
EventService / MeterService / CampaignService (后置处理)
```

关键文件：
- [QueueService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/QueueService.ts) — 队列管理与任务入队
- [email-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts) — Worker 消费与邮件发送流程
- [EmailService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts) — Email 记录创建、模板编译、Webhook 事件处理
- [SESService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/SESService.ts) — AWS SES Provider 封装
- [worker.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/worker.ts) — Worker 进程入口

---

## 补充章节：同步直发链路 vs 队列处理器链路 分工对比

### 链路定位与代码位置

系统中存在两条并行的邮件发送执行链路，它们共享模板编译和 SES Provider 发送，但责任边界有显著差异：

| 维度 | 同步直发链路 | 队列处理器链路（主路径） |
|------|-------------|------------------------|
| **入口方法** | [EmailService.sendEmail(emailId)](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L290-L456) | [email-processor Worker handler](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L82-L279) |
| **执行上下文** | 调用线程同步执行，阻塞请求 | BullMQ Worker 异步消费，不阻塞业务入口 |
| **生产调用方** | 生产代码中**无直接调用**（仅被单元测试调用，见 `EmailService.test.ts`） | 三条入口：`sendTransactionalEmail` / `sendCampaignEmail` / `sendWorkflowEmail` 均通过 `QueueService.queueEmail` 入队后由 Worker 消费 |
| **注释定位** | 注释 "Actually send the email via AWS SES / This is called by the email processor worker" | 注释为 BullMQ Worker 的处理函数 |

> **关键事实**：`EmailService.sendEmail` 在当前生产代码中是"遗留/测试辅助"路径，线上实际发送走的是 email-processor Worker。两条链路逻辑高度相似但不完全一致，这是理解分工的基础。

---

### 1. Provider 发送（两条链路均负责，但参数构造有差异）

两条链路最终都调用 [sendRawEmail](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/SESService.ts#L87-L231)（AWS SES Provider），区别只在 `to` 字段的格式：

| 项目 | 同步链路 [EmailService.ts#L405-L419](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L405-L419) | Worker 链路 [email-processor.ts#L215-L229](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L215-L229) |
|------|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| `to` 参数 | 只传 `[recipientEmail]` 字符串数组（[EmailService.ts#L381-L411](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L381-L411)） | 从 `email.toName` 字段取收件人姓名，优先传 `[{name: email.toName, email: recipientEmail}]` 对象数组（携带收件人姓名，邮件客户端显示 "张三 <zhang@example.com>"）；无 `toName` 时降级为字符串数组（[email-processor.ts#L177-L179](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L177-L179)、[L220](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L220)） |
| `replyTo` | `email.replyTo \|\| undefined`（[EmailService.ts#L415](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L415)） | 与同步链路一致，`email.replyTo \|\| undefined`（[email-processor.ts#L225](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L225)），无 `project.replyTo` 回退 |
| 附件计费逻辑 | 不区分 | 有附件时 Meter 计费 **×2**（[email-processor.ts#L245-L248](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L245-L248)） |

**结论**：Worker 链路对 Provider 参数做了更完善的处理，是"生产级"实现；同步链路相对简陋，可能是早期实现被保留用于测试。

---

### 2. 域名校验（仅同步链路负责，Worker 链路缺失）

这是两条链路**最关键的职责差异**：

| 项目 | 同步链路 [EmailService.ts#L329-L331](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L329-L331) | Worker 链路 email-processor.ts |
|------|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|
| `DomainService.verifyEmailDomain` | ✅ **有**。发送前显式调用，校验三项：<br>① 域名在 DB 存在<br>② 域名属于当前 project<br>③ `verified === true`<br>不满足抛 403 异常 | ❌ **无**。整个 Worker handler 中没有调用 `verifyEmailDomain` 或任何 DomainService 方法 |
| 拦截效果 | 未验证域名的邮件 **SENDING 状态之前**就会失败，不会调用 SES | 未验证域名的邮件会被 **直接送入 SES**，由 AWS 侧抛出错误（通常是 `Email address is not verified`），catch 块再标记 FAILED |

**风险分析**：Worker 链路缺少域名校验存在以下隐患：
1. **无效 SES 调用消耗配额**：未验证域名的邮件会实际打到 SES API 才失败，占用每秒发送速率配额
2. **错误信息不一致**：同步链路抛明确的中文业务错误，Worker 链路收到的是 AWS 原始英文错误
3. **安全漏洞**：若入队后项目被回收了域名所有权（域名被另一个 project 占用），Worker 链路仍会以该域名发信

---

### 3. 订阅拦截（两条链路均负责，但时机和力度不同）

订阅状态检查在链路中存在**双重校验**，但层级不同：

| 层级 | 位置 | 检查内容 | 失败动作 |
|------|------|---------|---------|
| **第一层：入队前**（共享） | `sendTransactionalEmail` [L55-L75](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L55-L75)<br>`sendWorkflowEmail` [L203-L235](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L203-L235) | MARKETING 模板 + 已退订联系人 | **不入队**，直接返回或创建 FAILED Email |
| **第二层：同步链路发送前** | [EmailService.sendEmail#L309-L326](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L309-L326) | `!email.contact.subscribed` 且非 TRANSACTIONAL | 跳过 SES 调用，直接 `status=FAILED` + `error='Contact is unsubscribed...'`，**不 throw**（return 静默结束） |
| **第二层：Worker 链路发送前** | email-processor.ts | ❌ **无对应检查** | — |

**设计意图解读**：
- 同步链路的二次校验是"防御性"的：防止 PENDING Email 记录在入队后、实际发送前，联系人刚好退订导致违规（GDPR/反垃圾邮件合规）
- Worker 链路**缺失这道防线**：若入队到消费之间存在时间差（高并发队列积压几秒到几分钟），期间退订的联系人仍会收到邮件
- 注意：`sendCampaignEmail` 本身**不做入队前订阅检查**（由上游 Campaign 批处理 chain 统一在 `recipientIds` 构造时过滤），Worker 链路消费 Campaign 邮件时同样缺少二次校验

---

### 4. 状态更新（两条链路均负责，Worker 链路覆盖更全）

状态更新职责对比：

| 状态变化 | 同步链路 EmailService.sendEmail | Worker 链路 email-processor.ts |
|---------|-------------------------------|-------------------------------|
| **PENDING (初始)** | 不负责（由 `sendXxxEmail` 入口创建） | 不负责 |
| **PENDING → SENDING** | ✅ [L334-L337](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L334-L337) | ✅ [L124-L127](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L124-L127) |
| **PENDING → FAILED (域名未验证)** | ✅ 由 `verifyEmailDomain` throw 进入 catch 块 | ❌ 无此路径（由 SES 报错触发，见下） |
| **PENDING → FAILED (未订阅)** | ✅ [L317-L323](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L317-L323)，直接 return 不 throw | ❌ **无** |
| **PENDING → FAILED (项目禁用)** | ❌ **无检查** | ✅ [L104-L120](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L104-L120)，额外 `finalizeIfDone` campaign |
| **SENDING → SENT** | ✅ [L422-L429](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L422-L429)，仅写 `sentAt + messageId` | ✅ [L232-L239](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L232-L239)，字段一致 |
| **SENDING → FAILED (catch)** | ✅ [L446-L452](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L446-L452) | ✅ [L270-L276](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L270-L276) |
| **POST-SENT 事件** | ✅ 调 `EventService.trackEvent('email.sent')` [L432-L441](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L432-L441) | ✅ 同样调 `EventService.trackEvent` [L254-L260](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L254-L260) |
| **计费上报 (Meter)** | ❌ **无** | ✅ `MeterService.recordEmailSent` [L244-L248](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L244-L248)，含附件×2 计费 |
| **Campaign 收敛** | ❌ **无** | ✅ `CampaignService.finalizeIfDone` [L263-L265](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L263-L265) |
| **钓鱼检测命中禁用项目** | ❌ **无** | ✅ 禁用项目 + 调 `QueueService.cancelAllProjectJobs` [L193-L212](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L193-L212) |

**结论**：Worker 链路的状态更新是"完整版"，包含安全、计费、campaign 联动；同步链路只做了最小可用的状态流转。

---

### 5. 失败处理（两条链路结构相似，Worker 链路多一层 BullMQ 重试）

| 阶段 | 同步链路 | Worker 链路 |
|------|---------|------------|
| **catch 结构** | try-catch 包裹整个发送流程 [L328-L455](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L328-L455) | try-catch 包裹整个发送流程 [L128-L279](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L128-L279) |
| **catch 动作** | ① `signale.error` 日志<br>② `status=FAILED` + 写 `error` 字段<br>③ `throw error` 向上抛出 | ① `signale.error` 日志（带 attemptId）<br>② 取 `attemptsMade + 1 === maxAttempts` 做 `isFinalAttempt` 判断（但**实际未使用**该变量）<br>③ `status=FAILED` + 写 `error` 字段<br>④ `finalizeIfDone` campaign（仅 campaign 邮件）<br>⑤ `throw error` 向上抛出 |
| **BullMQ 重试** | ❌ 无（同步调用无队列机制） | ✅ 有（队列配置 `attempts: 3` + 指数退避），但因 catch 块已先置 FAILED，**重试会被 PENDING guard 拦截为空转**（详见第五章 5.4） |
| **未订阅处理** | 不抛异常，return 静默结束（不触发任何重试/补偿） | ❌ 无此场景（不做检查） |
| **项目禁用处理** | ❌ 不检查 | 不抛异常，return 静默结束 + finalize campaign |
| **钓鱼检测处理** | ❌ 不检查 | `phishingResult.shouldDisable` 时先禁用项目 + 取消所有作业，再抛异常触发 FAILED 写入 |

---

### 6. 两条链路独有的职责清单

| 功能 | 仅同步链路有 | 仅 Worker 链路有 |
|------|:-----------:|:--------------:|
| SES 发送前域名有效性校验 (`verifyEmailDomain`) | ✅ | ❌ |
| SES 发送前订阅状态二次校验 | ✅ | ❌ |
| BullMQ 速率限制 (rate × 每秒) + 并发控制 | ❌ | ✅ |
| 项目禁用状态检查 (`project.disabled`) | ❌ | ✅ |
| LLM 钓鱼内容检测 (`checkPhishingContent`) + 采样率 | ❌ | ✅ |
| Stripe 计费 MeterEvent 上报（含附件加权） | ❌ | ✅ |
| Campaign finalize 状态收敛 | ❌ | ✅ |
| 收件人姓名注入 `to.name` | ❌ | ✅ |
| `project.replyTo` 作为 replyTo 回退值 | ❌ | ✅ |
| 项目禁用级联清理 (`cancelAllProjectJobs`) | ❌ | ✅ |
| `X-Plunk-Recipient-Override` header 处理 | ✅ (公共 header 处理一致) | ✅ (公共 header 处理一致) |

---

### 7. 架构设计意图推断

从代码差异可以推断出**演进路径**和**设计分工**：

```
阶段 1 (MVP):
  EmailService.sendEmail ← 同步直发，所有逻辑都在这里
  包含：域名校验 + 订阅校验 + 状态更新 + SES 调用

阶段 2 (规模化):
  引入 BullMQ → 抽取 Email + QueueService
  Worker 内"重写"了一份 sendEmail 流程（email-processor.ts）
  为生产场景新增：项目禁用检查 / 钓鱼检测 / 计费 / campaign 联动 / 速率控制
  但遗漏了同步链路原有的：域名校验 / 订阅二次校验 ← 技术债

现状:
  同步链路 → 测试专用 / 历史遗留 / 文档上说是 Worker 调它但实际 Worker 没调
  Worker 链路 → 真正的生产路径
```

> **证据**：`EmailService.sendEmail` 的注释写着 "This is called by the email processor worker"，但 [email-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts) 全程没有调用 `EmailService.sendEmail()`，而是内联实现了整个流程。注释与实现不一致，说明中间重构时 Worker 从"调用同步方法"改成了"内联重写"，但同步方法没被删除、注释没更新，形成了当前"两条并行链路"的状态。

---

## 二、邮件任务入队流程

### 2.1 三条入队路径

邮件通过三种业务场景入队，最终都汇聚到 `emailQueue`：

| 场景 | 入口方法 | SourceType | 优先级 |
|------|---------|------------|--------|
| Transactional API | [EmailService.sendTransactionalEmail](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L52-L114) | `TRANSACTIONAL` | 1 (最高) |
| Campaign 广播 | [EmailService.sendCampaignEmail](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L119-L178) | `CAMPAIGN` / `TRANSACTIONAL` | 10 (最低) |
| Workflow 自动化 | [EmailService.sendWorkflowEmail](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L183-L284) | `WORKFLOW` / `TRANSACTIONAL` | 5 |

优先级由 [emailPriorityFor](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/QueueService.ts#L177-L188) 函数定义，BullMQ 中数字越小优先级越高。Transactional 邮件（登录验证码、密码重置）优先于营销邮件。

### 2.2 入队前置检查（EmailService 层）

每条邮件在入队前必须经过以下检查：

1. **计费限额检查** — [BillingLimitService.checkLimit](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/BillingLimitService.ts#L149-L353)
   - 免费版：总限额 1000 封/月（跨所有类型共享）
   - 付费版：按 `billingLimitTransactional` / `billingLimitCampaigns` / `billingLimitWorkflows` 分别限制
   - 超限直接抛 `HttpException(429)`，不入队

2. **订阅状态检查**（仅营销类邮件）
   - Workflow 场景：未订阅联系人直接创建 FAILED 状态 Email 记录（不入队），见 [sendWorkflowEmail#L203-L235](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L203-L235)
   - Transactional 场景：使用 MARKETING 模板时校验订阅，见 [sendTransactionalEmail#L55-L75](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L55-L75)

3. **模板类型推断**
   - 模板类型为 `TRANSACTIONAL` 时，即使通过 Campaign/Workflow 入口也会被提升为 `TRANSACTIONAL` sourceType，用于后续取消退订页脚和优先出队

### 2.3 Email 记录创建 + 入队（原子性）

```
prisma.email.create (status=PENDING)
       ↓
BillingLimitService.incrementUsage (Redis 缓存计数 +1)
       ↓
QueueService.queueEmail (BullMQ add)
```

三个步骤**非事务性**，但顺序保证：先落库再入队。若在中间失败，会留下 PENDING 状态的孤儿邮件，无自动补偿机制。

### 2.4 队列配置

[emailQueue](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/QueueService.ts#L47-L58) 的默认参数：

- **重试次数**：`attempts: 3`（BullMQ 层面最多重试 3 次）
- **退避策略**：`exponential`，初始延迟 2000ms（2s → 4s → 8s）
- **保留策略**：完成的保留最近 1000 条，失败的保留最近 5000 条
- **Job ID**：`email-${emailId}`（保证幂等，重复入队同一 emailId 会被 BullMQ 去重）

---

## 三、Provider 发送与外部依赖

### 3.1 Worker 启动

[createEmailWorker](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L73-L304) 启动时：

1. 通过 [getEmailRateLimit](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L30-L53) 确定发送速率：
   - 优先级：环境变量 `EMAIL_RATE_LIMIT_PER_SECOND` > AWS SES `getSendQuota()` > 默认 14（沙箱限额）
2. 根据速率计算 Worker 并发数：[deriveWorkerConcurrency](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L62-L71)，`ceil(rate × 0.5)`，夹在 `[5, EMAIL_WORKER_MAX_CONCURRENCY]` 之间
3. BullMQ limiter 配置：`{max: rateLimit, duration: 1000}` — 每秒最多发送 rateLimit 封

### 3.2 Worker 处理流程（核心链路）

[email-processor.ts#L82-L279](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L82-L279) 的执行步骤：

```
Step 1: 预检查 (5 项)
  ├─ Email 记录存在性校验
  ├─ status === PENDING (非 PENDING 直接 return)
  ├─ project.disabled 检查 (禁用则 FAILED + finalize campaign)
  ├─ status PENDING → SENDING (DB 更新)
  └─ SecurityService.checkPhishingContent (LLM 钓鱼检测)

Step 2: 内容编译
  ├─ EmailService.format (变量替换: {{firstName}} 等)
  └─ EmailService.compile (HTML 包装 + 退订页脚 + Powered by 徽章)

Step 3: SES 发送
  └─ SESService.sendRawEmail (构造 MIME → AWS SES API)

Step 4: 成功后置处理
  ├─ status SENDING → SENT + sentAt + messageId
  ├─ MeterService.recordEmailSent (Stripe 计费, idempotencyKey=emailId)
  ├─ EventService.trackEvent ('email.sent' → 触发 workflow)
  └─ CampaignService.finalizeIfDone (若为 campaign 邮件)
```

### 3.3 外部依赖清单

| 依赖 | 用途 | 关键配置 |
|------|------|----------|
| **Redis (ioredis)** | BullMQ 队列存储 + 缓存 | `REDIS_URL` |
| **AWS SES** | 邮件实际发送 | `AWS_SES_ACCESS_KEY_ID` / `AWS_SES_SECRET_ACCESS_KEY` / `AWS_SES_REGION` |
| **OpenRouter API** | 钓鱼内容检测 (LLM) | `OPENROUTER_API_KEY` / `OPENROUTER_MODEL`，采样率由 `PHISHING_DETECTION_SAMPLE_RATE` 控制 |
| **Stripe (MeterEvent)** | 按邮件量计费 | `STRIPE_ENABLED`，走 `meterQueue` 异步上报 |
| **PostgreSQL (Prisma)** | Email / Campaign / Contact 持久化 | `DATABASE_URL` |

### 3.4 SES 发送细节

[sendRawEmail](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/SESService.ts#L87-L231) 负责：

- **MIME 构造**：支持 `multipart/alternative`（仅 HTML）、`multipart/mixed`（带附件）、`multipart/related`（带内联图片）三种嵌套结构
- **退订头**：自动提取 HTML 中的 unsubscribe 链接，注入 `List-Unsubscribe` Header
- **跟踪配置集**：根据 `shouldTrackEmail` 决定使用 `SES_CONFIGURATION_SET` 还是 `SES_CONFIGURATION_SET_NO_TRACKING`
- **长行处理**：[breakLongLines](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/SESService.ts#L54-L82) 按 RFC 规范将 HTML 每行截断到 500 字符，Base64 附件截断到 76 字符

---

## 四、状态更新机制与状态流转

### 4.1 EmailStatus 枚举

[schema.prisma#L777-L788](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/packages/db/prisma/schema.prisma#L777-L788) 定义了 10 种状态：

```
PENDING → SENDING → SENT → DELIVERED → OPENED → CLICKED
                         ↘ BOUNCED
                         ↘ COMPLAINED
            ↘ FAILED
```

### 4.2 状态变化责任分工

| 状态转换 | 触发方 | 位置 |
|---------|--------|------|
| **PENDING (初始)** | EmailService | create 时默认值 |
| **PENDING → SENDING** | email-processor Worker | [email-processor.ts#L124-L127](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L124-L127) |
| **SENDING → SENT** | email-processor Worker | [email-processor.ts#L232-L239](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L232-L239)，SES 返回 messageId 后 |
| **SENDING → FAILED** | email-processor Worker (catch 块) | [email-processor.ts#L266-L279](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L266-L279) |
| **SENT → DELIVERED** | SES Webhook → EmailService.handleWebhookEvent | [EmailService.ts#L479-L482](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L479-L482) |
| **SENT/DELIVERED → OPENED** | SES Webhook / 跟踪像素 | [EmailService.ts#L484-L490](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L484-L490)，累计 opens 计数 |
| **OPENED → CLICKED** | 点击跟踪链接 | [EmailService.ts#L492-L498](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L492-L498)，累计 clicks 计数 |
| **SENT → BOUNCED** | SES Webhook | [EmailService.ts#L500-L514](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L500-L514)，同时自动退订联系人 |
| **SENT → COMPLAINED** | SES Webhook (垃圾邮件投诉) | [EmailService.ts#L516-L530](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/EmailService.ts#L516-L530)，同时自动退订联系人 |
| **PENDING → FAILED (项目禁用)** | email-processor / QueueService.cancelAllProjectJobs | [email-processor.ts#L104-L120](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L104-L120) 或 [QueueService.ts#L609-L612](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/QueueService.ts#L609-L612) |

### 4.3 Campaign 状态联动

每封 Campaign 邮件成功/失败后，都会调用 [CampaignService.finalizeIfDone](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/CampaignService.ts#L541-L587)：

- 终止条件：`processedCount >= totalRecipients`（processedCount = SENT + FAILED 的邮件数）
- 满足条件则 Campaign 从 `SENDING` → `SENT`，写入 `sentCount`
- 设计意图：FAILED 也算作"已处理"，避免因个别邮件失败导致 Campaign 永远卡在 SENDING

---

## 五、失败处理与重试机制

### 5.1 分层失败处理

系统采用 **三层失败处理**：

#### 第 1 层：BullMQ 自动重试（配置层）
- 位置：[QueueService.ts#L49-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/QueueService.ts#L49-L54)
- 策略：指数退避 `2s → 4s → 8s`，最多 3 次
- 触发条件：Worker handler 抛出异常（email-processor catch 块末尾 `throw error`）
- **副作用**：每次重试前，DB 中 status 已被标记为 FAILED（见第 2 层），下次重试时 Worker 会因 `status !== PENDING` 直接跳过。这意味着 **BullMQ 的重试实际上无效** — 第一次失败后 status 已是 FAILED，后续重试会被 guard 拦截。

#### 第 2 层：Worker 内标记 FAILED（业务层）
- 位置：[email-processor.ts#L266-L279](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L266-L279)
- 行为：`prisma.email.update({status: FAILED, error: message})` + `throw error`
- 错误信息持久化到 `email.error` 字段

#### 第 3 层：非重试性失败的即时终止（预检查层）
以下情况**不触发 BullMQ 重试**（直接 return 或 throw 后 status 已是 FAILED）：

| 场景 | 处理方式 | 位置 |
|------|---------|------|
| Email 记录不存在 | `throw Error` (但 status 不会被更新，因为找不到记录) | [email-processor.ts#L95-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L95-L97) |
| status !== PENDING | 静默 return | [email-processor.ts#L99-L101](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L99-L101) |
| project.disabled | FAILED + finalize campaign + return (不 throw) | [email-processor.ts#L104-L120](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L104-L120) |
| 钓鱼检测命中 + shouldDisable | 禁用项目 + FAILED + throw | [email-processor.ts#L193-L212](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/jobs/email-processor.ts#L193-L212) |

### 5.2 项目禁用时的级联清理

当项目因安全原因被禁用（退信率过高 / 钓鱼检测）时：

1. [SecurityService.disableProject](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/SecurityService.ts#L654-L727) → 调用 `QueueService.cancelAllProjectJobs`
2. [QueueService.cancelAllProjectJobs](file:///d:/fz/0601-1/solo-dogfeeding/code/51-plunk/apps/api/src/services/QueueService.ts#L545-L640) 执行：
   - 遍历 `scheduledQueue` / `emailQueue` / `campaignQueue` / `workflowQueue` 的 waiting+delayed jobs
   - 逐个查询 DB 校验归属项目后移除
   - **批量** `prisma.email.updateMany({status: FAILED, error: "Project is disabled"})`（将所有 PENDING 邮件置为 FAILED）
   - 对 SENDING 状态的 Campaigns：修正 `totalRecipients` → `finalizeIfDone`

### 5.3 幂等性设计

| 场景 | 幂等保证 |
|------|---------|
| **重复入队** | Job ID = `email-${emailId}`，BullMQ 同 ID 的 job 不会重复入队 |
| **重复计费** | MeterEvent 使用 `email_${emailId}` 作为 idempotencyKey，Stripe 端去重 |
| **重复处理** | Worker 开头检查 `status !== PENDING` 直接 return，防止 SENDING/SENT/FAILED 邮件被重复消费 |

### 5.4 已知设计要点 / 潜在风险

1. **BullMQ 重试与 FAILED 状态冲突**：Worker catch 块先将 status 置为 FAILED 再 throw，触发 BullMQ 重试；但重试时 status !== PENDING 直接 return。重试机制实际上是"空转"。若希望真正重试，应在 FAILED 更新前判断是否还有剩余 attempts，或不在 catch 中更新 status（让 BullMQ 控制状态）。

2. **非原子入队**：`prisma.email.create` → `QueueService.queueEmail` 无事务包裹。若 Redis 写入失败，会留下 PENDING 状态的孤儿邮件。

3. **计费计数时机**：`BillingLimitService.incrementUsage` 在入队前调用（基于 Email 记录创建成功），而非实际发送成功时。若 SES 发送失败，计费计数已增加但邮件未实际发出。

---

## 六、责任分工总结

| 模块 | 核心职责 | 不负责 |
|------|---------|--------|
| **QueueService** | 队列实例管理、任务入队（带优先级/Job ID）、队列统计、项目级作业取消、旧作业清理 | 不关心邮件内容、不做业务校验、不更新 DB 状态 |
| **EmailService** | Email 记录 CRUD、模板变量替换、HTML 编译（含退订页脚/徽章）、Webhook 事件分发（DELIVERED/OPENED/CLICKED/BOUNCED/COMPLAINED）、三种发送入口的前置校验 | 不直接调用 SES（由 Worker 或 SESService 承担）、不管理队列 |
| **email-processor (Worker)** | 消费 emailQueue、执行发送流程（预检查→编译→SES→后置）、更新 SENDING/SENT/FAILED 状态、触发计费与事件、关联 campaign 收尾 | 不创建 Email 记录、不做入队、不处理投递后事件 |
| **SESService** | AWS SES SDK 封装、MIME 邮件构造、发送速率配额查询、域名验证相关 API | 不关心邮件业务状态、不操作 DB、不处理重试 |
| **BillingLimitService** | 发送前限额校验、Redis 缓存使用量、超限告警通知（Ntfy + Email） | 不实际扣款（由 MeterService + Stripe 处理）、不阻止已入队邮件 |
| **MeterService** | 发送成功后的 Stripe MeterEvent 上报（走 meterQueue 异步） | 不做限额判断、不直接操作 Stripe API |
| **SecurityService** | 钓鱼内容检测（LLM + 采样）、退信/投诉率监控、项目自动禁用、SNS 签名验证 | 不直接处理邮件发送流程 |
| **EventService** | `email.sent` 等事件持久化、触发 workflow 监听、唤醒 WAIT_FOR_EVENT 步骤 | 不关心邮件发送本身 |
| **CampaignService** | Campaign 批处理链驱动、finalize 状态收敛 | 不处理单封邮件发送 |
