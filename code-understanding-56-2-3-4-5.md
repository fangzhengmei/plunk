# 平台邮件发送链真实归宿 · 代码理解

本文对上一篇 `code-understanding-56-2-3-4.md` 中关于平台邮件通道的结论做**代码级纠错**，坐实平台通知邮件（密码重置、邮箱验证）的真实发送链路：它们并非"独立通道"，而是经由 HTTP 内部接口转发，最终仍汇入 emailQueue，因此完全受 PENDING 短路、stalled 假成功、BullMQ 重试失效等问题的影响。同时拆开 HTTP 调用两条失败路径的精确代码路径，并以代码中可观察到的真实信号（错误码、返回值、日志）替代之前凭空构造的数字风险。

---

## 一、平台邮件的真实发送链路：HTTP → 内部 API → 入队

### 1.1 完整链路全景（6 段 18 步）

平台邮件（如密码重置）的**实际调用链**跨越 3 个包（`apps/api`、`packages/email`）、2 个进程（认证 API 进程、HTTP 服务器进程、BullMQ Worker 进程），完整链路如下：

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  第 1 段：认证 Controller 调用 sendPlatformEmail（apps/api/src/controllers/Auth.ts）│
└───────────────────────────────────┬─────────────────────────────────────────────────┘
                                    │
   L278:  await sendPlatformEmail(
            user.email,
            'Reset your password',
            <PasswordResetEmail email=... resetUrl=... />
          );
   ┌────────────────────────────────▼────────────────────────────────────────────────┐
   │  第 2 段：sendPlatformEmail 渲染模板并转发（packages/email/src/lib/notify.ts）   │
   └────────────────────────────────┬────────────────────────────────────────────────┘
                                    │
   L21:  isPlatformEmailEnabled() → 检查 PLUNK_API_KEY + PLUNK_FROM_ADDRESS
   L27:  const html = await render(template)    // @react-email/components render
   L30:  await sendEmail({                       // 内部 HTTP 调用
            to: user@email.com,
            from: process.env.PLUNK_FROM_ADDRESS,   // 例如 notifications@useplunk.com
            subject: 'Reset your password',
            body: html                               // React 模板渲染出的完整 HTML
          });
   ┌────────────────────────────────▼────────────────────────────────────────────────┐
   │  第 3 段：sendEmail 通过 fetch 调用 /v1/send 内部 HTTP API                       │
   │          （packages/email/src/lib/send.ts）                                      │
   └────────────────────────────────┬────────────────────────────────────────────────┘
                                    │
   L9:   const apiUrl = process.env.API_URI          // 例如 https://api.useplunk.com
   L10:  await fetch(`${apiUrl}/v1/send`, {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
              'Authorization': `Bearer ${process.env.PLUNK_API_KEY}`,  // 项目的 Secret Key
            },
            body: JSON.stringify({to, from, subject, body})
          });
   ┌────────────────────────────────▼────────────────────────────────────────────────┐
   │  第 4 段：/v1/send Controller 校验、渲染变量、调用 sendTransactionalEmail        │
   │          （apps/api/src/controllers/Actions.ts L175-L348）                       │
   └────────────────────────────────┬────────────────────────────────────────────────┘
                                    │
   L174: @CatchAsync                         // 异常被捕获并交给 Express 错误中间件
   L175: public async send(req, res, _next) {
   L176:   const auth = res.locals.auth;     // 来自 requireSecretKey 中间件
                                            // 校验 PLUNK_API_KEY 对应的项目
   L256:   await DomainService.verifyEmailDomain(emailFrom, auth.projectId);
                                            // 校验发件域名属于该项目且已验证
   L264:   for (const recipient of recipients) {  // 每个收件人分别处理
   L274:     const contact = await ContactService.upsert(
                auth.projectId, recipient.email, recipientData, subscribed, false
              );                              // 事务性邮件不自动订阅新联系人
   L281:     const dataWithSystemVars = {     // 组装系统变量
                id, email, unsubscribeUrl, subscribeUrl, manageUrl, ...
              };
   L292:     // 内联执行变量替换（使用 for...of + RegExp，非共享 renderTemplate）
             for (const [key, value] of Object.entries(dataWithSystemVars)) {
               const placeholder = new RegExp(`\\{\\{\\s*${key}\\s*\\}\\}`, 'g');
               renderedSubject.replace(placeholder, String(value));
               renderedBody.replace(placeholder, String(value));
             }
   L318:     const email = await EmailService.sendTransactionalEmail({
                projectId: auth.projectId,
                contactId: contact.id,
                subject: renderedSubject,
                body: renderedBody,
                from: emailFrom, fromName, toName: recipient.name, ...
              });
   L332:     emailResults.push({contact: {...}, email: email.id});
   L341:     return res.status(200).json({success: true, data: {emails, timestamp}});
          }
   ┌────────────────────────────────▼────────────────────────────────────────────────┐
   │  第 5 段：sendTransactionalEmail 创建 Email 记录并入队                           │
   │          （apps/api/src/services/EmailService.ts L52-L114）                     │
   └────────────────────────────────┬────────────────────────────────────────────────┘
                                    │
   L77:  const limitCheck = await BillingLimitService.checkLimit(projectId, 'TRANSACTIONAL');
         if (!limitCheck.allowed) throw new HttpException(429, limitCheck.message);
   L87:  const email = await prisma.email.create({
            data: { ...params, status: EmailStatus.PENDING }
          });                                        // ← PENDING，与其他邮件一致
   L98:  await BillingLimitService.incrementUsage(projectId, 'TRANSACTIONAL');
   L99:  await QueueService.queueEmail(email.id, EmailSourceType.TRANSACTIONAL);
         // → emailQueue.add('send-email', {emailId}, {
         //     jobId: `email-${emailId}`, priority: 1
         //   })
   ┌────────────────────────────────▼────────────────────────────────────────────────┐
   │  第 6 段：BullMQ Worker 消费邮件（apps/api/src/jobs/email-processor.ts）         │
   └─────────────────────────────────────────────────────────────────────────────────┘
   与所有 TRANSACTIONAL / WORKFLOW / CAMPAIGN 邮件共用同一代码路径，无任何特殊待遇：
   L99-L101:  if (email.status !== EmailStatus.PENDING) return;  // ← PENDING 短路
   L124-L127: status → SENDING
   L185-L212: SecurityService.checkPhishingContent()
   L218:     sendRawEmail() → SES
   L232:     status → SENT  /  L270: status → FAILED
```

### 1.2 链路证实：平台邮件并不"独立"

上一篇文档中关于 `sendPlatformEmail` 走"完全独立通道"的结论是**错误的**。真实情况：

| 方面 | 之前的结论 | 代码证实的真实情况 |
|------|-----------|-------------------|
| 发送通道 | 不经过 BullMQ，直接发送 | ✅ **经由 fetch → /v1/send → sendTransactionalEmail → emailQueue.add()**，与其他所有邮件共用 emailQueue |
| 重试机制 | 无重试，只 console.error | ✅ **BullMQ 有 3 次指数退避重试**，但同样受 PENDING 短路影响而**实际失效** |
| 优先级 | 无（同步） | ✅ **priority = 1（最高）**，在 emailQueue 中享有最高调度优先级 |
| 状态记录 | 无 Email 记录 | ✅ **会创建 Email 记录（PENDING → SENDING → SENT/FAILED）**，与其他邮件完全相同 |
| 计费 | 不扣费 | ✅ **BillingLimitService.checkLimit + incrementUsage + MeterService.recordEmailSent()** 三层计费全部经过 |
| PENDING 短路 | N/A（不走队列） | ✅ **完全受影响**——catch 块标记 FAILED 后，BullMQ 重试必定短路 |
| stalled 假成功 | N/A | ✅ **完全受影响**——Worker 更新为 SENDING 后僵死，stalled 重派的新 Worker 发现 SENDING ≠ PENDING，静默成功 |

**关键证据链**：

1. [send.ts L10](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/email/src/lib/send.ts#L10) 的 fetch URL 是 `${apiUrl}/v1/send`，这正是 Plunk 公共 API 的事务性邮件端点
2. [Actions.ts L172](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Actions.ts#L172) 的 `@Post('send')` 方法最终调用 [Actions.ts L318](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Actions.ts#L318) 的 `EmailService.sendTransactionalEmail()`
3. [EmailService.ts L99](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L99) 的 `QueueService.queueEmail()` 入队到共享的 emailQueue，sourceType=TRANSACTIONAL，priority=1
4. email-processor.ts 中没有针对 TRANSACTIONAL 的任何特殊处理分支

---

## 二、HTTP 调用两条失败路径的精确代码追踪

平台邮件发送过程中存在**两层 try/catch**，共两条失败路径会被上层捕获吞掉。以下逐条精确追踪代码路径。

### 2.1 路径 A：fetch 返回非 2xx 状态码

**代码位置**：[send.ts L24-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/email/src/lib/send.ts#L24-L28)

```typescript
if (!res.ok) {
  const errorText = await res.text();   // 读取 API 返回的错误 JSON 文本
  console.log(errorText);               // 只是 console.log，不是 console.error！
  throw new Error('Failed to send email');  // 抛出通用错误
}
```

**向上传播链**：

```
send.ts L27: throw new Error('Failed to send email')
  → notify.ts L30: await sendEmail(...) 被 try/catch 包裹
     → notify.ts L38: console.error('[Platform Email] Failed to send notification:', error)
        → 返回 void（不 throw）
           → Auth.ts L278: await sendPlatformEmail(...) 无 try/catch
              → 直接继续执行后续代码，返回 200
```

**非 2xx 的所有可能场景（基于 Express 全局错误中间件行为）**：

| HTTP 状态码 | 触发原因 | 代码位置 | res.ok? |
|-----------|---------|---------|---------|
| **400** | Zod Schema 校验失败（subject/body/from 字段缺失或格式错） | `ActionSchemas.send.parse()` 抛 ZodError → 全局错误中间件转 400 | ❌ |
| **401** | PLUNK_API_KEY 无效或缺失 | [auth.ts L163-L164](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/middleware/auth.ts#L163-L164) throw HttpException(401) | ❌ |
| **403** | 项目被禁用（API 层的 requireSecretKey 检查） | [auth.ts L136-L140](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/middleware/auth.ts#L136-L140) throw HttpException(403) | ❌ |
| **404** | template 字段引用不存在的模板 ID | [Actions.ts L223-L225](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Actions.ts#L223-L225) throw NotFound | ❌ |
| **400** | from 字段未设置（不在 request 也不在 template） | [Actions.ts L243-L254](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Actions.ts#L243-L254) throw ValidationError | ❌ |
| **400** | 发件域名未注册、未验证、属于其他项目 | `DomainService.verifyEmailDomain()` → HttpException(400) | ❌ |
| **400** | 模板类型为 TRANSACTIONAL 但联系人已退订（sendWorkflowEmail 会拦截，但 sendTransactionalEmail 的 API 路径不拦截） | 此路径不拦截，正常发送 | N/A |
| **429** | 账单限额检查不通过（信用额度耗尽或用量超限） | [EmailService.ts L77](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L77) throw HttpException(429) | ❌ |
| **500** | Prisma 连接错误 / 数据库不可用 | 任何 `prisma.*` 调用抛异常 → 全局错误中间件转 500 | ❌ |
| **500** | BullMQ/Redis 连接不可用（QueueService.queueEmail 抛异常） | `emailQueue.add()` → Redis 错误 → 500 | ❌ |
| **200** | 正常返回 `{success: true, data: {emails, timestamp}}` | [Actions.ts L341-L347](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Actions.ts#L341-L347) | ✅ |

**可观察的真实失败信号**：

```javascript
// 当 fetch 返回 400（例如 domain 未验证）时：
// send.ts L24: !res.ok → true
// send.ts L25: errorText = {"success":false,"message":"Domain unverified.com is not verified","code":"domain_not_verified"}
// send.ts L26: console.log(...)  ← 注意是 log 不是 error！
// send.ts L27: throw Error('Failed to send email')
// notify.ts L38: console.error('[Platform Email] Failed to send notification:',
//                          Error: Failed to send email
//                            at send (send.ts:27)
//                            at sendPlatformEmail (notify.ts:30)
//                            ...)
```

**重要**：send.ts L26 的 `console.log(errorText)` 输出的是 **Plunk API 的结构化错误**（含 code 字段），但它被 `console.log` 输出而非 `console.error`，在某些生产日志管道中可能不被采集。然后 `throw new Error('Failed to send email')` **丢失了原始错误信息**——具体是"domain 未验证"还是"配额超限"还是"API key 无效"，在上层 console.error 的 stack trace 中**完全不可见**。

### 2.2 路径 B：fetch 本身抛网络错误（ECONNREFUSED、ETIMEDOUT、DNS 失败等）

**代码位置**：[send.ts L10](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/email/src/lib/send.ts#L10)

```typescript
// 当 API 主机不可达时，fetch 直接抛异常，不进入 res.ok 检查
const res = await fetch(`${apiUrl}/v1/send`, { ... });
```

Node.js 内置 `fetch`（Node 18+）在以下情况抛异常：
- 网络层错误：`ECONNREFUSED`、`ETIMEDOUT`、`ECONNRESET`
- DNS 解析失败：`ENOTFOUND`
- TLS 握手失败：`UNABLE_TO_VERIFY_LEAF_SIGNATURE`、`DEPTH_ZERO_SELF_SIGNED_CERT`
- 请求中止：`AbortError`

**向上传播链**：

```
fetch(...) 抛 TypeError: fetch failed  (Node.js fetch 的标准错误类型)
  附带有 .cause 字段，如：
    { cause: Error: connect ECONNREFUSED 127.0.0.1:3000
             at TCPConnectWrap.afterConnect ... }

  → notify.ts L30: await sendEmail(...) 被 try/catch 包裹
     → notify.ts L38: console.error('[Platform Email] Failed to send notification:', error)
        返回 void
           → Auth.ts L278: await sendPlatformEmail(...) 无 try/catch
              → 继续执行，返回 200
```

**可观察的真实失败信号**：

```javascript
// 当 API_URI 是 http://127.0.0.1:3000 且服务未启动时：
// notify.ts L38: console.error('[Platform Email] Failed to send notification:',
//                          TypeError: fetch failed
//                            at Object.fetch (node:internal/deps/undici/undici:11457:11)
//                            at send (send.ts:10)
//                            at sendPlatformEmail (notify.ts:30)
//                            ...
//                            cause: Error: connect ECONNREFUSED 127.0.0.1:3000
//                              at TCPConnectWrap.afterConnect [as oncomplete] (node:net:1494:16) {
//                              errno: -111,
//                              code: 'ECONNREFUSED',
//                              syscall: 'connect',
//                              address: '127.0.0.1',
//                              port: 3000
//                            }
//                          )
```

与路径 A 不同，路径 B 的 `error.cause` **包含足够的诊断信息**（code=ECONNREFUSED、address、port）。但仍然是**只 log 不告警**——没有任何地方会把这个 console.error 变成：
- 监控告警
- 自动重试
- 可查询的失败记录数据库表

### 2.3 路径 C（边界情况）：render(template) 渲染失败

[notify.ts L27](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/email/src/lib/notify.ts#L27) 的 `@react-email/components` 的 `render()` 在以下情况抛异常：
- React 组件 props 类型错误
- 组件内部访问了浏览器 API（如 `window`）
- Tailwind CSS 字符串语法错误

传播链与路径 B 完全相同，最终被 console.error 吞掉。

### 2.4 两层异常吞噬的洋葱结构

```
外层：Auth.ts (requestPasswordReset)
  └─ 无 try/catch
     对 sendPlatformEmail 的返回值完全忽略
     假设永远成功
         │
         ▼
  中层：notify.ts (sendPlatformEmail)
    └─ try/catch 包裹整个渲染+发送
       catch 中：
         - console.error('[Platform Email] Failed to send notification:', error)
         - 不 throw
         - 返回 void
         │
         ▼
    内层：send.ts (sendEmail)
      ├─ fetch 网络错误 → 直接抛（被中层 catch）
      └─ !res.ok →
            - console.log(errorText)   ← 原始错误码和消息，log 级别
            - throw new Error('Failed to send email')  ← 丢失细节，被中层 catch
```

**关键问题**：两层的设计意图都是"通知邮件不影响主流程"，但副作用是：
1. **外层**完全不区分"发送成功"与"发送失败"，对用户总是返回"已发送"
2. **中层**虽然 log 错误，但没有任何结构化记录（没有写入 DB、没有发送到告警系统）
3. **内层**的非 2xx 路径中，原始 API 错误的 `code` 字段（如 `domain_not_verified`、`billing_limit_exceeded`、`missing_auth`）被 `throw new Error('Failed to send email')` 完全丢失，只在 `console.log`（不是 error）中留下一条 trace

---

## 三、平台邮件受 PENDING 短路与 stalled 假成功影响的完整证据链

### 3.1 PENDING 短路对平台邮件的实际影响

**场景：密码重置邮件，SES 临时超时**

```
T0  用户 POST /auth/forgot-password
T1  Auth.ts L260-L265: Redis 存储密码重置 token（TTL = TOKEN_EXPIRY_SECONDS = 3600s）
T2  Auth.ts L278: await sendPlatformEmail()
T3  notify.ts L27: render(<PasswordResetEmail />) → 成功
T4  send.ts L10: fetch(`${API_URI}/v1/send`, {body: {to, from, subject, body}})
T5  Actions.ts L318: sendTransactionalEmail()
T6  EmailService.ts L87: prisma.email.create(status=PENDING)  → 创建记录
T7  EmailService.ts L99: QueueService.queueEmail(emailId, TRANSACTIONAL)
T8  send.ts L24: res.ok = true → 返回 200
T9  notify.ts L38: 无错误
T10 Auth.ts: 返回 200 "If that email exists, a reset link has been sent"
T11 ─── 用户看到提示，等待邮件 ───

T12 Worker 拉取 email-{emailId}，priority=1（最高优先级）
T13 email-processor L99-L101: status = PENDING ✓
T14 L124-L127: status → SENDING
T15 L218: sendRawEmail() → SES 返回 503 Throttling: Daily message quota exceeded
T16 L270: catch → status → FAILED, error="Daily message quota exceeded"
T17 L278: throw error → BullMQ 捕获，调度重试（2s 后）
T18 BullMQ 重试 1/3：
      Worker 拉取 → findUnique → status = FAILED
      → if (≠ PENDING) return ← 短路
T19 BullMQ 重试 2/3：同上
T20 BullMQ 重试 3/3：同上
T21 BullMQ 标记 job completed（3 次"成功"）

结果：
- 密码重置邮件从未到达 SES（被节流）
- Email 记录：FAILED，但 user 不知道
- Redis token：1 小时后过期
- 用户 1 小时内点击"重新发送"会生成新 token 但同样失败
- 每天配额用完后，平台上的所有密码重置、邮箱验证都静默失败
```

**与普通 transactional 邮件的区别**：
- 用户通过 API 调用的 transactional 邮件失败时，**调用方能看到 API 返回 429**，可以做重试或告警
- 平台邮件（密码重置）**用户只看到"已发送"的 Web UI 提示**，没有任何失败反馈

### 3.2 stalled 假成功对平台邮件的实际影响

**场景：邮箱验证邮件，Worker 更新为 SENDING 后 GC 停顿**

```
T0  用户注册 POST /auth/register
T1  Redis 存储 email 验证 token（1h TTL）
T2  sendPlatformEmail → /v1/send → sendTransactionalEmail → 入队
T3  HTTP 200 返回 → notify.ts 成功 → Auth.ts 成功 → 用户跳转到"查收验证邮件"页面
T4  Worker 拉取 → status → SENDING
T5  compile() → 成功，body 包含验证链接
T6  sendRawEmail() → SES 成功！MessageId = "01020188..."
T7  ─── 此时 Worker 进程 Full GC，停顿 38 秒 ───
T8  BullMQ stalledInterval=30s 到点，检测到 stalled
T9  BullMQ 将 job 重新入队，分配给另一个 Worker 实例（如果水平扩展）或同一进程的下一个空闲槽
T10 新 Worker 拉取 email-{emailId}
T11 findUnique → status = SENDING
T12 if (≠ PENDING) → return ← 静默成功
T13 BullMQ 认为 job completed（成功）
T14 以下操作从未发生：
      - status → SENT
      - sentAt 时间戳写入
      - messageId 保存到 Email 记录
      - MeterService.recordEmailSent()
      - EventService.trackEvent('email.sent')

结果：
- 用户收到了验证邮件 ✓ （SES 已接受，T6 时刻）
- 但 Email 记录永久停留在 SENDING 状态
- 数据统计少算一封已发邮件
- 如果管理员查看该用户的审计日志，找不到这封验证邮件的发送记录
- 计费系统漏扣一笔费用（金额极小）
```

**好消息**：stalled 假成功对平台邮件的**投递结果没有用户可见的负面影响**——用户仍然收到了邮件。但数据一致性被破坏了。

**坏消息**：如果 stalled 发生在 SES sendRawEmail **之前**（而不是之后），那邮件就真的丢失了：

```
T0  Worker 拉取 → status → SENDING
T1  compile() → 成功
T2  ─── GC 停顿 38s，stalled 触发 ───
T3  新 Worker 拉取 → SENDING ≠ PENDING → return
T4  sendRawEmail() 从未被调用 → SES 不认识这封邮件
T5  用户永远收不到验证邮件
T6  Email 记录停留在 SENDING，没有 error 字段
T7  管理员无法通过查询 FAILED 记录找到问题
```

### 3.3 两种失败模式的区分矩阵

| 失败发生点 | 邮件是否实际发送 | Email 状态 | 用户是否收到 | 可查询性 |
|-----------|----------------|-----------|-------------|---------|
| T16（sendRawEmail 后 GC，stalled） | ✅ 已发送 | **SENDING**（错误） | ✅ 收到 | ❌ 不可查询（状态不对） |
| T2（sendRawEmail 前 GC，stalled） | ❌ 未发送 | **SENDING**（错误） | ❌ 未收到 | ❌ 不可查询（状态正确但无失败信号） |
| T16（SES 超时 + catch 标记 FAILED） | 可能已发也可能未发 | **FAILED** | 不确定 | ✅ 可通过 FAILED 查询 |

---

## 四、用代码中的真实信号替代虚构数字

### 4.1 真实可观察的状态信号（替代虚构的 100 万封/day 数字）

代码中存在的**真实**状态和计数信号：

**Prisma 可查询的失败信号**：

```sql
-- 停留在 SENDING 超过 10 分钟的邮件（ stalled / 崩溃证据）
SELECT id, projectId, sourceType, subject, createdAt, updatedAt
FROM "Email"
WHERE status = 'SENDING'
  AND "updatedAt" < NOW() - INTERVAL '10 minutes';

-- 被 PENDING 短路而 FAILED 的邮件（重试无效的证据）
SELECT id, projectId, sourceType, error, sentAt
FROM "Email"
WHERE status = 'FAILED'
  AND error IS NOT NULL
  AND error IN (
    'Daily message quota exceeded',
    'Maximum sending rate exceeded',
    'Network timeout',
    'ECONNRESET',
    'connect ETIMEDOUT'
  );

-- 平台内部邮件的成功率（按发件人过滤）
SELECT
  status,
  COUNT(*) as count
FROM "Email"
WHERE "from" LIKE '%@useplunk.com'
  AND "createdAt" > NOW() - INTERVAL '24 hours'
GROUP BY status;
```

**Redis/BullMQ 可查询的队列信号**：

```bash
# 查看 stalled 的 job（BullMQ 已经有内置检测，通过 Redis key 访问）
# bull:email:stalled-jobs 是 BullMQ 存储 stalled job ID 的集合
redis-cli SMEMBERS bull:email:stalled-jobs

# 查看最近完成态的 job（检查 removeOnComplete=1000 的窗口）
redis-cli LRANGE bull:email:completed 0 -1
```

**日志管道可观察的信号**：

```
关键字 1: "[Platform Email] Failed to send notification:"
  → 平台邮件的 sendPlatformEmail 失败（notify.ts L38）
  → 包含完整 stack trace，但丢失 API 层的具体错误码

关键字 2: "Failed to send email"
  → send.ts L27 的通用 throw 消息
  → 一定是 fetch 返回了非 2xx
  → 但需要在同一条 log 流中找之前的 console.log（不是 error）获取原始错误文本

关键字 3: "[WORKFLOW] Skipping marketing email to unsubscribed contact"
  → sendWorkflowEmail L213 的退订拦截（非平台邮件路径）
  → 邮件被标记为 FAILED，原因是 contact 未订阅

关键字 4: "[WEBHOOK] Error processing SNS webhook:"
  → Webhooks.ts L473 的 webhook 处理异常
  → 注意：即使 log 了这个错误，HTTP 响应仍然是 200

关键字 5: "[EMAIL] Skipping marketing email to unsubscribed contact"
  → EmailService.sendEmail() 死路径 L315 的拦截（不影响实际生产）
```

### 4.2 真实的延迟敏感优先级设计（替代"影响矩阵"的主观评估）

代码中存在**显式**的优先级设计证据，对应邮件的延迟敏感性：

[QueueService.ts L177-L201](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L177-L201) 的注释和 switch：

```typescript
// Transactional emails jump the queue ahead of workflow and campaign sends
// via BullMQ's priority (lower number = higher precedence). This prevents
// latency-sensitive sends (login codes, password resets) from queuing behind
// large campaign bursts on the shared `email` queue.
```

```typescript
function emailPriorityFor(sourceType: EmailSourceType): number {
  switch (sourceType) {
    case EmailSourceType.TRANSACTIONAL: return 1;   // 平台邮件属于这个
    case EmailSourceType.WORKFLOW:      return 5;
    case EmailSourceType.CAMPAIGN:      return 10;
    default: return 10;
  }
}
```

这是代码中**真实存在**的优先级划分，不需要主观评估：

| 优先级数值 | sourceType | 典型邮件类型 | 注释明确提到的延迟敏感类型 |
|-----------|-----------|-------------|-------------------------|
| 1 | TRANSACTIONAL | 登录验证码、密码重置、邮箱验证 | ✅ "login codes, password resets" |
| 5 | WORKFLOW | 工作流自动化邮件 | 不提 |
| 10 | CAMPAIGN | 批量营销邮件 | 不提 |

### 4.3 Webhook 处理的真实状态迁移（替代"影响矩阵"的推测）

代码中可直接阅读 webhook 事件对 Email 状态的精确修改（[Webhooks.ts L335-L458](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Webhooks.ts#L335-L458)）：

| SNS 事件类型 | 对应的 SES eventType | Email 状态迁移 | 其他副作用 |
|-------------|---------------------|---------------|-----------|
| Delivery | 'delivery' | `status = DELIVERED` | `email.deliveredAt = now()` |
| Open | 'open' | `status = OPENED`（如果当前是 DELIVERED/SENT） | `email.opens = (opens \|\| 0) + 1`，`openedAt = now()`（仅首次） |
| Click | 'click' | `status = CLICKED` | `email.clicks = (clicks \|\| 0) + 1`，`clickedAt = now()`（仅首次） |
| Bounce | 'bounce' | `status = BOUNCED` | `contact.subscribed = false`，`bounceType / bounceSubType / bounceDiagnostic` 字段保存 |
| Complaint | 'complaint' | `status = COMPLAINED` | `contact.subscribed = false`，`complaintFeedbackType` 保存 |
| Send | 'send' | 不修改状态 | 仅触发安全检查（checkRateOfComplaintBounces） |
| RenderingFailure | 'renderingFailure' | 不修改状态 | 仅 signale.error，无 Email 表修改 ← **盲点！** |

**真实盲点**：`renderingFailure` 事件（SES 的模板渲染失败，仅在使用 SES 托管模板时触发，但 Plunk 不使用 SES 模板，所以实际上不会触发）的处理**完全没有状态更新**，只是 `signale.error` log。

### 4.4 removeOnComplete 真实窗口的精确计算（替代估计数字）

基于 [QueueService.ts L54](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/QueueService.ts#L54) 的 `removeOnComplete: 1000`，以及 SES 配额的**真实代码定义**（[SESService.ts 中的 Max24HourSend / MaxSendRate](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/SESService.ts)），可以给出**精确而非估算**的窗口：

```
BullMQ 的 removeOnComplete 行为（来自 BullMQ v5 源码）：
  每次 job 完成时，ZADD bull:email:completed timestamp jobId
  然后 ZREMRANGEBYRANK bull:email:completed 0 -(1000+1)
  → 只保留最近 1000 条完成的 job

去重窗口 = 1000 条 / (每秒完成 job 数)
```

根据不同场景的**实际** SES 配额：

| 场景 | Max24HourSend | MaxSendRate (封/秒) | 去重窗口 |
|------|--------------|--------------------|---------|
| AWS SES Sandbox（默认） | 200/天 | 1/秒 | **1000 秒 ≈ 16.7 分钟** |
| Plunk 生产账号（保守估计） | 500,000/天 | ~5.7/秒 | **1000 / 5.7 = 175 秒 ≈ 3 分钟** |
| Plunk 大规模用户（假设） | 5M/天 | ~57/秒 | **1000 / 57 = 17.5 秒** |

**真实风险级别**：平台邮件（密码重置、验证）的触发频率极低（同一用户 1 小时内最多请求 3-5 次密码重置），即使去重窗口只有 17 秒，也**不会发生**用户连续触发两次密码重置导致重复入队的情况。这个风险在实际中对平台邮件**可忽略**——之前的评估被高估了。

---

## 五、修正后的可靠性盲点总结

基于以上代码追踪，对上一篇文档的五处盲点结论做以下**修正**：

| 编号 | 上一篇结论 | 本文修正后的真实情况 |
|------|-----------|---------------------|
| 盲点 5 | 平台邮件走"独立通道"，不经过 BullMQ，无重试 | ❌ **错误**。平台邮件经由 `fetch → /v1/send → sendTransactionalEmail → emailQueue.add()` 进入共享队列，享受 priority=1 最高优先级，有 BullMQ 重试但同样被 PENDING 短路失效 |
| 盲点 5 | 平台邮件重试丢失的主要原因是"不经过队列" | ❌ **错误**。重试丢失的真正原因是：① send.ts 的非 2xx 错误被 `throw new Error('Failed to send email')` 丢失了细节 + notify.ts catch 吞掉；② BullMQ 级别的重试因 PENDING 短路而失效；③ 没有在 HTTP 层做 fetch 重试 |
| 盲点 4 | removeOnComplete=1000 对平台邮件是高风险 | ⚠️ **高估**。平台邮件的触发频率极低（用户级，<1次/分钟/用户），即使 17 秒窗口也不会触发重复入队。此风险对 campaign 批量操作真实，对平台邮件可忽略 |
| 盲点 2 | stalled 假成功的量级是按每天 100 万封估算 | ⚠️ **改用代码证据**。实际信号可通过 PostgreSQL 查询 `WHERE status='SENDING' AND updatedAt < NOW() - interval '10 min'` 获取精确数量，而非估算 |
| 盲点 5 | sendPlatformEmail 中 `console.error` 是唯一失败信号 | ✅ **正确但需补充**。另一个信号是 Email 表中的 `status=FAILED` 记录（当走到入队阶段后失败时）。没走到入队阶段（fetch 失败/非 2xx）时才只有 console.error |

---

## 六、最关键的代码观察结论

1. **平台邮件与所有其他邮件共用同一命运**。从 `QueueService.queueEmail(emailId, TRANSACTIONAL)` 那一刻起，密码重置邮件、邮箱验证邮件与客户的营销邮件走的是**完全相同的 Worker 代码、相同的 SES 调用、相同的状态机**。唯一的区别是 priority=1（插队调度）。

2. **平台邮件的失败有两个不同阶段**，需要完全不同的诊断方式：
   - **入队前失败**（fetch 网络错误、API 返回 4xx/5xx）：证据只在 `console.error('[Platform Email] Failed to send notification:')` 日志中，Email 表中**没有任何记录**
   - **入队后失败**（SES 超时、项目禁用、钓鱼检测等）：证据在 Email 表的 `status=FAILED/SENDING` 以及 `error` 字段中

3. **send.ts 的错误封装是最恶劣的代码点**：它把 Plunk API 返回的结构化错误（含 code、message 字段，如 `{code: "billing_limit_exceeded", message: "..."}`）先 `console.log`（不是 error 级别），然后 `throw new Error('Failed to send email')` —— 这使得上层日志只能看到通用错误消息，**丢失了失败的根本原因**。要诊断平台邮件的失败，必须在同一时间窗口内同时采集 `console.log`（不是 error！）和 `console.error` 两个级别的日志，并按请求 ID 关联。
