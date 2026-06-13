# 模板下发链路：展示一致性与工作流发送时序 · 代码理解

本文在上一份 `code-understanding-56.md` 的基础上，深入追踪模板从「编辑器保存」到「邮件客户端渲染」的完整下发链路，重点剖析：

1. juice 内联 CSS 路径（Landing 工具页）vs Dashboard 直接发送路径的 CSS 投递差异及其在 Outlook / Gmail 的呈现分歧
2. `sendWorkflowEmail` 的完整方法调用顺序——退订拦截、format/compile 处理、队列投递、BullMQ Worker 实际发送、SES 异常处理
3. Email 记录的状态机：从 PENDING 到最终态的所有路径与回退逻辑

---

## 一、两条 CSS 投递路径的代码源点

### 1.1 Dashboard 路径：`<style>` 块 + class 选择器

**代码入口**：[EmailService.compile()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L1039-L1134)

```
compile() → detectCustomHtmlPatterns(content) ?
              content（原样） :
              wrapWithEmailStyles(content)（追加 <style> 块）
```

[wrapWithEmailStyles()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L708-L1033) 生成的是一个完整的 HTML 文档，核心结构为：

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    /* ~330 行 Tailwind Typography prose 样式 */
    .prose { color: #374151; max-width: 600px; }
    .prose a { color: #3b82f6; text-decoration: underline; font-weight: 500; }
    .prose h1 { font-size: 2.25em; font-weight: 800; ... }
    /* ... */
  </style>
</head>
<body>
  <div class="prose prose-sm max-w-none">
    ${htmlBody}
  </div>
</body>
</html>
```

**关键特征**：所有样式定义在 `<style>` 标签中，通过 `.prose` 类选择器级联。`htmlBody` 是编辑器产出的 HTML 片段（如 `<p>Hello</p>`），被包裹在 `<div class="prose prose-sm max-w-none">` 容器内，依赖 class 选择器生效。

### 1.2 Landing 工具页路径：juice 内联 CSS

**代码入口**：[convertToEmailHtml()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/landing/src/lib/emailHtmlConverter.ts#L9-L139)

```typescript
const wrappedHtml = `<html><head><style>
  body { font-family: ...; font-size: 16px; ... }
  h1 { font-size: 32px; font-weight: 700; ... }
  p { margin: 0 0 16px 0; }
  /* ... 约 80 行样式 ... */
</style></head><body>${html}</body></html>`;

const inlined = juice(wrappedHtml, {
  preserveMediaQueries: false,
  preserveFontFaces: false,
  removeStyleTags: true,    // 删除 <style> 标签
  applyStyleTags: true,     // 将 <style> 中的规则内联到每个元素
});
```

**关键特征**：juice 将 `<style>` 中的每条 CSS 规则解析后，以 `style="..."` 属性的形式直接写入匹配的 HTML 元素上，然后**删除** `<style>` 标签。结果是每个元素自带完整内联样式，不依赖任何 class 选择器或 `<style>` 块。

---

## 二、Outlook / Gmail 客户端呈现差异深度分析

### 2.1 Outlook（桌面版，Word 渲染引擎）

Outlook 桌面版（Windows）使用 **Microsoft Word 的 HTML 渲染引擎**，这是邮件客户端中兼容性最差的渲染器。

| CSS 特性 | Dashboard `<style>` + class 路径 | Landing juice 内联路径 |
|----------|----------------------------------|----------------------|
| `<style>` 标签 | ✅ 支持（仅 `<head>` 内） | N/A（已删除） |
| class 选择器 | ✅ 支持 | N/A（样式已内联） |
| `max-width` | ⚠️ 部分不支持（`<div>` 上的 `max-width` 被忽略） | ✅ 内联在 `<div>` 上同样被忽略，但样式不丢失 |
| `margin` 值 `em` 单位 | ⚠️ 不完全支持 | ⚠️ 同样不完全支持 |
| `border-left` (blockquote) | ✅ | ✅ |
| `box-sizing: border-box` | ✅（`<head>` `<style>` 内） | ✅（内联到 `*` 选择器时，juice 会给每个元素加 `box-sizing`） |
| `font-family` 系统字体栈 | ✅ | ✅ |
| 伪元素 `::before`/`::after` | ❌ **完全不支持**（`.prose code::before` 中的反引号不会出现） | ❌ 同样不支持（juice 无法内联伪元素） |

**最关键的差异点**：Dashboard 路径依赖 `.prose` class 级联，当 Outlook **忽略某些 class 组合选择器**（如 `.prose h1`）时，整个排版链断裂。juice 内联路径中每个 `<h1>` 直接带 `style="font-size: 32px; font-weight: 700; ..."` ，不依赖级联，因此在 Outlook 中**更可靠**。

**具体断裂场景（Dashboard 路径）**：

1. `.prose h1` 不被识别 → H1 降级为 Word 默认标题样式（可能巨大、加粗、不同颜色）
2. `.prose a` 不被识别 → 链接丢失蓝色和下划线
3. `.prose blockquote` 不被识别 → 引用块丢失左边框和斜体
4. `.prose-sm` 的字号缩放丢失 → 所有文字大小回退到 Word 默认

### 2.2 Gmail（Web / 移动端）

Gmail 使用自己的渲染管线，对 CSS 有独特的处理方式：

| CSS 特性 | Dashboard `<style>` + class 路径 | Landing juice 内联路径 |
|----------|----------------------------------|----------------------|
| `<style>` 标签 | ✅ 支持（但有限制，见下） | N/A |
| class 选择器 | ✅ 支持 | N/A |
| `<head>` 中多个 `<style>` | ⚠️ Gmail 会**合并并截断**超过 ~8192 字符的 `<style>` 内容 | N/A |
| `max-width: 600px` | ✅ 支持 | ✅ 支持（内联） |
| 伪元素 `::before`/`::after` | ❌ **不支持**（`.prose code::before` 反引号不出现） | ❌ 同样不支持 |
| `data-*` 属性选择器 | ❌ 不支持 | N/A |

**Gmail 的 `<style>` 截断风险**：

Dashboard 路径的 `wrapWithEmailStyles()` 产出约 **330 行 CSS**，加上退订脚注表格 + Plunk Badge 表格的行内样式，总 HTML 体积约 **8-10KB**。Gmail 的 `<style>` 标签截断阈值约为 **8192 字符**（仅计算 `<style>` 标签内的文本）。如果 `htmlBody` 中还包含 Custom HTML 的 `<style>` 块，则累积内容很可能**超过截断线**，导致后半段 CSS 规则被丢弃。

**juice 路径无此风险**：`<style>` 标签已被 juice 完全删除，所有样式转为内联，不存在截断问题。

### 2.3 Apple Mail / iOS Mail

Apple 系列邮件客户端对 CSS 支持最完善，两条路径在 Apple Mail 中**几乎无差异**。`<style>` 块、class 选择器、伪元素全部正常工作。

### 2.4 差异总结矩阵

| 邮件客户端 | Dashboard `<style>`+class | Landing juice 内联 | 哪个更可靠 |
|-----------|--------------------------|-------------------|-----------|
| **Outlook 桌面** | ⚠️ class 级联可能断裂 | ✅ 内联样式可靠 | **juice** |
| **Gmail Web/Mobile** | ⚠️ `<style>` 可能截断 | ✅ 无截断风险 | **juice** |
| **Apple Mail** | ✅ 完全支持 | ✅ 完全支持 | 持平 |
| **Yahoo Mail** | ⚠️ 部分 class 不识别 | ✅ 内联可靠 | **juice** |
| **Outlook.com** | ✅ 支持 `<style>` | ✅ 内联 | 持平 |

### 2.5 代码层面的具体原因

Dashboard 路径之所以不使用 juice，可以从 [email-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L73-L303) 的设计意图推断：

1. **HTML 体积膨胀**：juice 内联后，每 `<p>` 都带 `style="margin-top: 1.25em; margin-bottom: 1.25em; font-size: 0.875rem; line-height: 1.7142857; color: #374151;"` ，批量发送时总 HTML 体积可增加 2-3 倍，在 campaign 大批量场景下不可忽略
2. **Custom HTML 兼容**：用户自己写的 HTML 可能已经做了内联优化，再用 juice 处理会产生冲突或重复属性
3. **代码维护成本**：juice 是一个有已知 bug 的库（如 `!important` 处理、伪元素丢失），在生产邮件管道中引入它会增加维护负担
4. **退订脚注 + Badge 已使用内联**：注意 [compile()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L1054-L1121) 中退订表格和 Plunk Badge 的样式**全部是行内 `style="..."`**——这说明 Plunk 团队清楚行内样式对邮件兼容性的重要性，但在 body 内容层面选择了「prose 级联」以换取体积优势

---

## 三、sendWorkflowEmail 完整方法调用时序

### 3.1 触发入口：WorkflowExecutionService.executeSendEmail

[executeSendEmail()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/WorkflowExecutionService.ts#L526-L593) 是工作流引擎中 SEND_EMAIL 步骤的执行器。它的调用链如下：

```
WorkflowExecutionService.processStepExecution()     // L47
  → executeStep()                                    // L467 → switch(step.type)
    → executeSendEmail()                             // L526
```

executeSendEmail 内部做了**首次变量替换**：

```typescript
// L553-L562: 组装变量映射
const variables = {
  id: contact.id,
  email: contact.email,
  ...contactData,        // 扁平化自定义字段
  ...executionContext,    // workflow 执行上下文（如触发事件的数据）
  data: contactData,
  unsubscribeUrl: `${DASHBOARD_URI}/unsubscribe/${contact.id}`,
  subscribeUrl: ...,
  manageUrl: ...,
};

// L564-L565: 替换模板中的变量
const renderedSubject = this.renderTemplate(step.template.subject, variables);
const renderedBody    = this.renderTemplate(step.template.body, variables);
```

**注意**：这里的 `renderTemplate` 调用是 `@plunk/shared` 的共享函数，与后端 `EmailService.format()` 使用的是**完全相同的函数**。但执行位置不同——WorkflowExecutionService 在入队前替换了一次，而 EmailService.sendEmail() 在 Worker 内又替换一次。这意味着 **变量被替换了两次**，但由于第一次替换后 `{{var}}` 已变成具体值，第二次替换时正则无匹配，不会产生错误。

### 3.2 EmailService.sendWorkflowEmail：三层拦截

[sendWorkflowEmail()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L183-L284) 的完整拦截链：

```
① 模板类型判定 → sourceType 决策
② 退订拦截 → marketing 邮件 vs unsubscribed 联系人
③ 账单限额拦截 → BillingLimitService.checkLimit()
④ Email 记录创建 → status = PENDING
⑤ 计数器递增 → BillingLimitService.incrementUsage()
⑥ 入队 → QueueService.queueEmail()
```

**① 模板类型判定**（L185-L198）：

```typescript
let sourceType = EmailSourceType.WORKFLOW;  // 默认

if (params.templateId) {
  const template = await prisma.template.findUnique({where: {id: params.templateId}, select: {type: true}});
  if (template?.type === 'TRANSACTIONAL') {
    sourceType = EmailSourceType.TRANSACTIONAL;  // 升级为事务性
  }
}
```

这一步决定了后续所有分支的走向。关键决策点：**TRANSACTIONAL 模板即使在 workflow 中也被视为事务性邮件**，不受退订限制、不添加退订脚注。

**② 退订拦截**（L200-L235）：

```typescript
if (sourceType !== EmailSourceType.TRANSACTIONAL && !params.recipientEmail) {
  const contact = await prisma.contact.findUnique({
    where: {id: params.contactId},
    select: {subscribed: true},
  });

  if (!contact?.subscribed) {
    signale.info(`[WORKFLOW] Skipping marketing email to unsubscribed contact...`);
    return await prisma.email.create({
      data: {
        ...params,
        status: EmailStatus.FAILED,   // ← 直接标记为 FAILED
        error: 'Contact is unsubscribed from marketing emails',
      },
    });
  }
}
```

**核心行为**：退订联系人的 marketing 邮件被**静默跳过**——不抛异常，不阻塞 workflow 执行，只创建一条 FAILED 记录。这与 `sendTransactionalEmail()` 中的营销模板退订拦截**不同**：

| 场景 | sendTransactionalEmail | sendWorkflowEmail |
|------|----------------------|-------------------|
| 营销模板 + 退订联系人 | 抛 `HttpException(400)` | 静默创建 FAILED 记录 |
| 原因 | API 调用需要明确的错误反馈 | workflow 需要继续执行后续步骤 |

**③ 账单限额拦截**（L237-L250）：

```typescript
const limitCheck = await BillingLimitService.checkLimit(params.projectId, sourceType);
if (!limitCheck.allowed) {
  throw new HttpException(429, limitCheck.message);
}
```

这里**会抛异常**——429 表示限流，调用方（WorkflowExecutionService）的 catch 块会捕获并标记 workflow 执行为 FAILED。

**④-⑥ Email 记录创建 + 计数 + 入队**（L252-L283）：

```typescript
// 如果有自定义收件人，存入内部 header
const emailHeaders = params.headers ? {...params.headers} : {};
if (params.recipientEmail) {
  emailHeaders['X-Plunk-Recipient-Override'] = params.recipientEmail;
}

const email = await prisma.email.create({
  data: {
    ...params,
    status: EmailStatus.PENDING,   // 初始状态
    headers: emailHeaders,
  },
});

await BillingLimitService.incrementUsage(params.projectId, sourceType);
await this.queueEmail(email.id, sourceType);  // → emailQueue.add()
```

### 3.3 BullMQ 队列投递与优先级

[QueueService.queueEmail()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L202-L216) 将 job 投入 `email` 队列，带优先级：

```typescript
const job = emailQueue.add('send-email', {emailId}, {
  delay,
  jobId: `email-${emailId}`,
  priority: emailPriorityFor(sourceType),
});
```

[优先级映射](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L177-L188)：

| sourceType | priority | 含义 |
|-----------|----------|------|
| TRANSACTIONAL | 1 | 最高——登录验证码、密码重置等延迟敏感邮件 |
| WORKFLOW | 5 | 中等 |
| CAMPAIGN | 10 | 最低——大批量营销邮件不抢资源 |

队列配置了**3 次重试 + 指数退避**（[QueueService.ts L49-L55](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L49-L55)）：

```typescript
attempts: 3,
backoff: { type: 'exponential', delay: 2000 },  // 2s → 4s → 8s
```

### 3.4 Email Worker：实际发送与异常处理

[email-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L73-L303) 是 BullMQ Worker，处理 `email` 队列中的 job。完整执行流程：

```
Worker 消费 job
  │
  ├─ ① 查询 Email + Contact + Project + Template + Campaign
  │     └─ email.status !== PENDING → return（防重复发送）
  │
  ├─ ② 项目禁用检查
  │     └─ project.disabled → 标记 FAILED + finalizeIfDone()
  │
  ├─ ③ 更新 status = SENDING
  │
  ├─ ④ EmailService.format() → 变量替换（第二次，见 3.1 注）
  │
  ├─ ⑤ EmailService.compile() → 样式包装 + 退订脚注 + Badge
  │     ├─ detectCustomHtmlPatterns? → 原样 / wrapWithEmailStyles()
  │     ├─ includeUnsubscribe? → 注入退订表格
  │     └─ STRIPE_ENABLED && 无订阅? → 注入 Plunk Badge
  │
  ├─ ⑥ SecurityService.checkPhishingContent() → 钓鱼内容检测
  │     └─ shouldDisable → 禁用项目 + 标记 FAILED
  │
  ├─ ⑦ sendRawEmail() → SES 发送
  │     ├─ 构造 MIME 消息（multipart/alternative 或 multipart/mixed）
  │     ├─ 检测 unsubscribe 链接 → 注入 List-Unsubscribe header
  │     ├─ 行长度合规处理（breakLongLines, 500 字符/行）
  │     └─ SES ConfigurationSet 选择（tracking vs no-tracking）
  │
  ├─ ⑧ 成功 → status = SENT + sentAt + messageId
  │     ├─ MeterService.recordEmailSent() → 计费（附件双倍计费）
  │     ├─ EventService.trackEvent('email.sent') → 事件追踪（可触发后续 workflow）
  │     └─ CampaignService.finalizeIfDone() → campaign 完成检查
  │
  └─ ⑨ 失败 → status = FAILED + error 字段
        └─ throw error → BullMQ 重试
```

**关键细节——Worker 中有而 EmailService.sendEmail() 中没有的步骤**：

1. **项目禁用检查**（L104-L120）：Worker 会再次检查 `project.disabled`，因为从入队到出队之间项目可能已被禁用
2. **钓鱼内容检测**（L185-L212）：SecurityService 在 Worker 中才运行，避免在入队路径上做重量级 AI 分析
3. **计费记录**（L244-L248）：只在成功发送后扣费，有附件则按 2 封计费
4. **Campaign 完成检查**（L263-L265）：每封邮件发送后检查 campaign 是否所有邮件都已处理完

### 3.5 sendEmail() 方法（EmailService 内部的同步发送路径）

[EmailService.sendEmail()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L290-L456) 是另一个发送路径，在 `sendTransactionalEmail` 中直接调用（不走 Worker）。它与 email-processor.ts 的逻辑**几乎完全一致**，区别在于：

| 方面 | email-processor.ts (Worker) | EmailService.sendEmail() |
|------|---------------------------|--------------------------|
| 执行环境 | BullMQ Worker 进程 | API 请求线程 |
| 退订拦截 | sendWorkflowEmail 已在入队前拦截 | 退订联系人直接标记 FAILED |
| 钓鱼检测 | 有 | 无 |
| 计费 | MeterService.recordEmailSent() | 无（由 queueEmail 路径处理） |
| 重试 | BullMQ 自动重试 3 次 | 无重试（catch 中 throw 向上传播） |

---

## 四、Email 记录状态机全路径

### 4.1 EmailStatus 枚举定义

[schema.prisma#L777-L788](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/db/prisma/schema.prisma#L777-L788)：

```
PENDING    // 已创建，等待发送
SENDING    // Worker 正在处理
SENT       // SES 返回 MessageId
DELIVERED  // SES webhook 确认送达
RECEIVED   // 入站邮件（反向）
OPENED     // 收件人打开
CLICKED    // 收件人点击链接
BOUNCED    // 退信
COMPLAINED // 垃圾邮件投诉
FAILED     // 发送失败
```

### 4.2 状态转换图

```
                          ┌─────────────────────────────────────┐
                          │          创建 Email 记录             │
                          └──────────────┬──────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼                    ▼                    ▼
             退订联系人            账单超限(429)          正常入队
          (sendWorkflowEmail)   (sendWorkflowEmail)         │
                    │                  抛异常               │
                    ▼               (不创建记录)             ▼
              ┌──────────┐                            ┌──────────┐
              │  FAILED  │                            │  PENDING │
              │ error=   │                            └─────┬────┘
              │ "Contact │                                  │
              │  is unsub│                     ┌────────────┤
              │  scribed"│                     │            │
              └──────────┘                     ▼            ▼
                                        项目已禁用     Worker 消费
                                             │              │
                                             ▼              ▼
                                        ┌──────────┐  ┌──────────┐
                                        │  FAILED  │  │ SENDING  │
                                        │ error=   │  └────┬─────┘
                                        │ "Project │       │
                                        │  is      │  ┌────┴─────────────┐
                                        │ disabled"│  │                  │
                                        └──────────┘  ▼                  ▼
                                              钓鱼检测成功        SES 发送
                                                  │              │       │
                                                  ▼              ▼       ▼
                                            ┌──────────┐    ┌────────┐ ┌────────┐
                                            │  FAILED  │    │  SENT  │ │ FAILED │
                                            │ error=   │    └───┬────┘ │ error= │
                                            │ "project │        │      │ errMsg │
                                            │  disabl" │        │      └───┬────┘
                                            └──────────┘        │          │
                                                         SES webhook     BullMQ
                                                         事件到达          重试?
                                                                │          │
                                                    ┌───────────┼──────┐   │
                                                    ▼           ▼      ▼   ▼
                                              ┌──────────┐ ┌────────┐ ┌────────┐
                                              │ DELIVERED│ │ OPENED │ │ BOUNCED│
                                              └──────────┘ └───┬────┘ └───┬────┘
                                                                │          │
                                                                ▼          ▼
                                                           ┌────────┐ ┌───────────┐
                                                           │CLICKED │ │ COMPLAINED│
                                                           └────────┘ └───────────┘
```

### 4.3 各路径触发条件与代码位置

| 状态转换 | 触发条件 | 代码位置 |
|---------|---------|---------|
| → PENDING | 正常创建 Email 记录 | [sendWorkflowEmail L258-L275](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L258-L275) |
| → FAILED (退订) | sourceType ≠ TRANSACTIONAL && !contact.subscribed | [sendWorkflowEmail L215-L233](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L215-L233) |
| → FAILED (项目禁用) | project.disabled 在 Worker 中检查 | [email-processor L106-L111](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L106-L111) |
| PENDING → SENDING | Worker 开始处理 | [email-processor L124-L127](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L124-L127) |
| SENDING → SENT | SES sendRawEmail 返回 MessageId | [email-processor L232-L239](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L232-L239) |
| SENDING → FAILED | SES 异常 / 钓鱼检测 | [email-processor L267-L278](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L267-L278) |
| SENT → DELIVERED | SES webhook `delivered` 事件 | [handleWebhookEvent L479-L481](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L479-L481) |
| SENT → OPENED | SES webhook `opened` 事件 | [handleWebhookEvent L485-L489](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L485-L489) |
| OPENED → CLICKED | SES webhook `clicked` 事件 | [handleWebhookEvent L492-L497](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L492-L497) |
| SENT → BOUNCED | SES webhook `bounced` 事件 | [handleWebhookEvent L500-L513](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L500-L513) |
| SENT → COMPLAINED | SES webhook `complained` 事件 | [handleWebhookEvent L516-L529](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L516-L529) |

### 4.4 SES 异常处理与重试机制

**BullMQ 重试配置**（[QueueService.ts L49-L55](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L49-L55)）：

```typescript
attempts: 3,
backoff: { type: 'exponential', delay: 2000 },
// 重试间隔: 2s → 4s → 8s
```

**重试行为细节**：

1. Worker catch 块中 `throw error`（[email-processor.ts L278](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/jobs/email-processor.ts#L278)），BullMQ 捕获后按 backoff 策略调度重试
2. 重试前 Email 状态已经是 FAILED（由 catch 块中的 update 设置）
3. **但重试时 Worker 先检查 `email.status !== PENDING → return`**——这意味着**实际上 BullMQ 的重试对 FAILED 状态的邮件无效**

这是一个重要的设计意图：一旦 Email 记录被标记为 FAILED，即使 BullMQ 重试 job，Worker 也会跳过。真正的重试需要通过其他机制（如手动重新发送）。

### 4.5 Bounced / Complained 的连带效应

[handleWebhookEvent()](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L462-L584) 中，BOUNCED 和 COMPLAINED 事件触发**自动退订**：

```typescript
case 'bounced':
  await prisma.contact.update({
    where: {id: email.contactId},
    data: {subscribed: false},
  });
  await EventService.trackEvent(..., 'contact.unsubscribed', ..., {reason: 'bounce'});
  break;

case 'complained':
  await prisma.contact.update({
    where: {id: email.contactId},
    data: {subscribed: false},
  });
  await EventService.trackEvent(..., 'contact.unsubscribed', ..., {reason: 'complaint'});
  break;
```

**连锁反应**：退订事件被追踪后，如果项目中有监听 `contact.unsubscribed` 事件的 workflow，该 workflow 可能被触发——形成「bounce → 退订 → 触发 workflow」的联动。

---

## 五、sendWorkflowEmail 全链路时序图

```
WorkflowExecutionService
  │
  ├─ processStepExecution(executionId, stepId)
  │    │
  │    ├─ 查询 execution + workflow + steps + contact
  │    ├─ 检查 project.disabled → 取消
  │    ├─ 创建/更新 stepExecution (RUNNING)
  │    │
  │    └─ executeStep(step, execution, stepExecution)
  │         │
  │         └─ executeSendEmail(step, execution, stepExecution)
  │              │
  │              ├─ 1. 解析 stepConfig (WorkflowStepConfigSchemas.sendEmail)
  │              ├─ 2. 组装 variables (id, email, contactData, executionContext, URLs)
  │              ├─ 3. renderTemplate(subject, variables)   ← 首次变量替换
  │              ├─ 4. renderTemplate(body, variables)      ← 首次变量替换
  │              ├─ 5. 判定 recipientEmail (CONTACT / CUSTOM)
  │              │
  │              └─ EmailService.sendWorkflowEmail(params)
  │                   │
  │                   ├─ 6. 查询 template.type → 判定 sourceType
  │                   │     ├─ TRANSACTIONAL → sourceType = TRANSACTIONAL
  │                   │     └─ 其他 → sourceType = WORKFLOW
  │                   │
  │                   ├─ 7. 退订拦截 (sourceType ≠ TRANSACTIONAL && !recipientEmail)
  │                   │     ├─ contact.subscribed = false → 创建 FAILED 记录, return
  │                   │     └─ contact.subscribed = true → 继续
  │                   │
  │                   ├─ 8. 账单限额检查
  │                   │     └─ !allowed → 抛 HttpException(429)
  │                   │
  │                   ├─ 9. 处理 X-Plunk-Recipient-Override header
  │                   ├─ 10. 创建 Email 记录 (status=PENDING)
  │                   ├─ 11. BillingLimitService.incrementUsage()
  │                   │
  │                   └─ 12. QueueService.queueEmail(emailId, sourceType)
  │                        │
  │                        └─ emailQueue.add('send-email', {emailId}, {priority})
  │
  │  ← 返回 email 记录
  │
  ├─ 标记 stepExecution (COMPLETED)
  └─ processNextSteps() → 处理下一个 step

═══════════════════════════════════════════════════════

Email Worker (email-processor.ts)
  │
  ├─ 消费 job {emailId}
  │
  ├─ A. 查询 email + contact + project + template + campaign
  │     └─ status ≠ PENDING → return
  │
  ├─ B. project.disabled → FAILED + finalizeIfDone
  │
  ├─ C. status → SENDING
  │
  ├─ D. EmailService.format(subject, body, contactData)
  │     └─ renderTemplate() ← 第二次变量替换（幂等）
  │
  ├─ E. EmailService.compile(content, contact, project, includeUnsubscribe)
  │     ├─ detectCustomHtmlPatterns(content)?
  │     │   ├─ true → 原样 content
  │     │   └─ false → wrapWithEmailStyles(content)  ← <style> + class 级联
  │     ├─ includeUnsubscribe → 注入退订脚注表格 (i18n)
  │     └─ STRIPE_ENABLED && !subscription → 注入 Plunk Badge
  │
  ├─ F. SecurityService.checkPhishingContent()
  │     └─ shouldDisable → 禁用项目 + FAILED
  │
  ├─ G. sendRawEmail()
  │     ├─ 检测 unsubscribe 链接 → List-Unsubscribe header
  │     ├─ 构造 MIME (multipart/alternative 或 multipart/mixed)
  │     ├─ breakLongLines(html, 500)
  │     ├─ 选择 ConfigurationSet (tracking / no-tracking)
  │     └─ ses.sendRawEmail()
  │
  ├─ H. 成功 → SENT + sentAt + messageId
  │     ├─ MeterService.recordEmailSent()
  │     ├─ EventService.trackEvent('email.sent') ← 可能触发新 workflow
  │     └─ CampaignService.finalizeIfDone()
  │
  └─ I. 失败 → FAILED + error
       └─ throw → BullMQ 重试（但 PENDING 检查使重试无效）
```

---

## 六、compile() 中的展示一致性保障细节

### 6.1 退订脚注的 i18n 实现

[compile() L1054-L1086](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L1054-L1086)：

```typescript
const contactLocale = contact.data?.locale ?? null;
const translator = createTranslatorSync(contactLocale || project.language || 'en');
const unsubscribeText = translator.t('email.footer.unsubscribeText', {projectName: project.name});
```

**优先级链**：`contact.data.locale` → `project.language` → `'en'`（硬编码兜底）

这意味着同一项目发送给不同 locale 的联系人时，退订脚注的文字会**自动切换语言**——这是展示一致性中唯一的个性化差异，且是有意为之的合规设计。

### 6.2 退订脚注的 HTML 表格结构

退订脚注使用 **`<table>` + `role="presentation"`** 布局：

```html
<table align="center" width="100%" style="max-width: 480px; ..."
       role="presentation">
  <tbody><tr><td>
    <hr style="border: none; border-top: 1px solid #eaeaea; ...">
    <p style="font-size: 12px; line-height: 24px; ...">
      {unsubscribeText} <a href=".../unsubscribe/{contactId}">{updatePreferencesText}</a>.
    </p>
  </td></tr></tbody>
</table>
```

**为什么用 table？** 因为 Outlook 桌面版不支持 `margin: auto` 居中，`<table align="center">` 是唯一可靠的居中方案。`role="presentation"` 确保 screen reader 不把它当数据表格读。

**为什么全部用行内 style？** 因为脚注是后端拼接的 HTML 片段，直接插入到 `</body>` 前，不会被 juice 处理，也不受 `<style>` 块中的 `.prose` 规则影响。行内样式是唯一可靠的样式投递方式。

### 6.3 Plunk Badge 的表格嵌套

[compile() L1090-L1121](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L1090-L1121) 使用了 **三层嵌套表格**：

```
外层 <table> (width:100%)
  └─ <td> (direction:ltr; text-align:center)
       └─ <div> (mj-column-per-100 class — 实际上此 class 无 CSS 定义，不影响)
            └─ <table> (vertical-align:top)
                 └─ <td> (align:center; word-break:break-word)
                      └─ <table> (border-collapse:collapse)
                           └─ <td> (width:180px)
                                └─ <a href="landing?ref=badge">
                                     └─ <img src="badge.png" width="180">
```

这个嵌套深度（4 层 table）是为了兼容 **Outlook 的 Word 渲染引擎**——Outlook 不支持 `display: inline-block` 和 `margin: auto`，必须用 table 布局来实现居中的 badge 图片。

### 6.4 脚注插入位置

[compile() L1127-L1131](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L1127-L1131)：

```typescript
if (html.includes('</body>')) {
  html = html.replace('</body>', `${footerHtml}</body>`);
} else {
  html = `${html}${footerHtml}`;
}
```

当 `wrapWithEmailStyles()` 产出了完整 HTML 文档（含 `</body>`）时，脚注插在 `</body>` 前——位于 `<div class="prose">` 容器**之外**，不受 `.prose` 样式影响。

当 Custom HTML 路径产出的是没有 `</body>` 的片段时，脚注追加在末尾。

---

## 七、前端预览与实际发送的一致性差距

### 7.1 前端预览流程（回顾）

[EmailEditor.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/EmailEditor.tsx) 中预览流程：

```
用户选择联系人
  → getPreviewHtml()
    → renderTemplate(body, contactData)          // 变量替换
    → wrapEmailWithStyles(replaced)              // prose 包装（或原样）
    → 注入 iframe 渲染
```

### 7.2 实际发送流程（Worker）

```
EmailService.format()                             // 变量替换
  → EmailService.compile()
    → wrapWithEmailStyles() / 原样               // prose 包装
    → 注入退订脚注                               // ← 预览没有
    → 注入 Plunk Badge                           // ← 预览没有
  → sendRawEmail()
    → breakLongLines(html, 500)                  // ← 预览没有
    → MIME 封装                                  // ← 预览没有
```

### 7.3 差异清单

| 维度 | 前端预览 | 实际发送 | 影响 |
|------|---------|---------|------|
| 变量替换 | ✅ renderTemplate() | ✅ renderTemplate() | **一致** |
| prose 样式包装 | ✅ wrapEmailWithStyles() | ✅ wrapWithEmailStyles() | **一致**（代码拷贝） |
| Custom HTML 检测 | ✅ detectCustomHtmlPatterns() | ✅ detectCustomHtmlPatterns() | **一致**（代码拷贝） |
| 退订脚注 | ❌ 不显示 | ✅ marketing 邮件注入 | 预览短于实际邮件 |
| Plunk Badge | ❌ 不显示 | ✅ 免费版注入 | 预览短于实际邮件 |
| List-Unsubscribe header | ❌ 不显示 | ✅ SES 注入 | 客户端可能显示「退订」按钮 |
| 行长度折行 | ❌ 不折行 | ✅ 500 字符/行 | 极端情况可能有细微空格差异 |
| MIME 封装 | ❌ 直接 HTML | ✅ multipart/alternative | 不影响渲染 |
| 钓鱼检测 | ❌ 不检测 | ✅ SecurityService | 预览不拦截 |
| CSS 内联 | ❌ 依赖 `<style>` + class | ❌ 同样依赖 `<style>` + class | **一致**（都未 juice） |

### 7.4 一致性风险点

1. **`detectCustomHtmlPatterns` 的两份拷贝**：[emailStyles.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/lib/emailStyles.ts#L10-L56) 和 [EmailService.ts#L669-L701](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L669-L701)。如果只修改一端，预览看到的样式路径与实际发送不同——这是当前架构中**最大的隐含风险**

2. **`wrapEmailWithStyles` 的两份拷贝**：[emailStyles.ts#L58-L387](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/lib/emailStyles.ts#L58-L387) 和 [EmailService.ts#L708-L1033](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L708-L1033)。330 行 CSS 的手动拷贝，任何一端修改后另一端必须同步

3. **退订脚注不可预览**：用户只能在发送测试邮件后才能看到退订脚注的最终效果。如果脚注的 `<hr>` 或 `<p>` 与 body 内容产生视觉冲突（如 body 底部也是灰色分割线），在预览中无法发现

---

## 八、关键洞察

1. **juice 内联与 Dashboard 直接发送的分歧是有意的技术取舍**，不是遗漏。juice 换取的是邮件客户端兼容性（特别是 Outlook），代价是 HTML 体积膨胀和伪元素丢失；Dashboard 路径换取的是代码简洁和体积可控，代价是 Outlook 和旧版 Gmail 的部分样式丢失。两条路径服务不同场景：Landing 工具页是「一次生成、导出即用」，Dashboard 是「批量发送、体积累积」。

2. **变量替换的两次执行不会出错但浪费算力**：WorkflowExecutionService.executeSendEmail() 和 EmailService.sendEmail() 都调用了 renderTemplate()。第二次执行是幂等的（正则不再匹配），但在大批量 workflow 场景下这是可优化的点。

3. **BullMQ 重试对 FAILED 邮件实质无效**：Worker 的 PENDING 检查使重试跳过已标记 FAILED 的邮件。这是一个防重复发送的安全机制，但也意味着一旦 SES 临时故障导致 SENDING→FAILED，该邮件**不会自动恢复**——需要手动干预或 campaign 层面的重发。

4. **退订脚注和 Badge 的 HTML 使用了「table 布局 + 行内样式」的双重保守策略**，这是邮件开发中对抗 Outlook 的经典模式。与之形成对比的是 body 内容的「`<style>` + class」现代模式——两个模式在同一个 HTML 文档中共存，这是 Plunk 在兼容性和代码可维护性之间的折中。

5. **前端预览与实际发送最大的不一致性是退订脚注 + Badge 的缺失**。解决方向有两种：a) 在预览中也注入脚注（加一个开关）；b) 在预览中显示一个「发送后将在底部添加退订链接」的占位提示。当前代码选择了不做——简化了预览逻辑，但增加了用户对最终效果的认知负担。
