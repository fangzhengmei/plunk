# 账单计量 — 六处遗漏边界分析

## 一、Stripe checkout 完成事件写 subscription 入口

### 完整数据流

```
用户点击"升级订阅"按钮
  → POST /users/@me/projects/{id}/checkout
    [Users.ts#L163-L249]
    │
    ├── 检查: 必须有 admin/owner 权限
    ├── 检查: project.subscription 已存在 → 400 拒绝（防止重复开通）
    ├── 构造 line_items: onboarding 费(可选) + metered usage 单价(必填)
    ├── billing_cycle_anchor = 下月 1 日（所有订阅锚定自然月）
    ├── client_reference_id = project.id（webhook 回填用）
    └── stripe.checkout.sessions.create() → 返回 session.url
        │
        ▼
用户在 Stripe 完成支付
  → Stripe POST /webhooks/incoming/stripe (event.type = checkout.session.completed)
    [Webhooks.ts#L513-L558]
    │
    ├── projectId = session.client_reference_id
    ├── customerId  = session.customer
    ├── subscriptionId = session.subscription
    │
    ├── prisma.project.update({
    │     where: { id: projectId },
    │     data: {
    │       customer: customerId,
    │       subscription: subscriptionId   ← 写 subscription 的唯一入口
    │     }
    │   })
    │
    ├── stripe.customers.update(customerId, { name, balance: -100 或 -300 })
    │   └── credit: 退 1 刀卡验证费；SWITCH 优惠码再退 2 刀
    │
    └── NtfyService.notifySubscriptionStarted()
```

### 关键细节

- **写 subscription 的唯一入口**：只有 `checkout.session.completed` webhook 会写入 `project.subscription` 和 `project.customer` 字段。
- **前端保护**：调用 `createCheckoutSession` 时若 `project.subscription` 已存在，返回 400（[Users.ts#L188-L191](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Users.ts#L188-L191)）。
- **webhook 端无保护**：`checkout.session.completed` 处理没有检查是否已存在 subscription，直接覆盖写入。

### 与 tier 切换的关系

subscription 被写入后，`checkLimit` 中条件 `!project.subscription` 不再成立 → **立即退出免费版模式**，开始按付费版（per-category 或 unlimited）判断限制。由于默认付费版四类 limit 全为 null，实际上用户刚升级完是 **unlimited 状态**，需要在 UI 里手动设置限额才会生效。

---

## 二、Paid tier limit 为 null 时 checkLimit 返回 0

### 代码路径

[BillingLimitService.ts#L266-L275](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L266-L275)：

```typescript
// If no limit set for paid tier, allow unlimited
if (limit === null) {
  return {
    allowed: true,
    warning: false,
    usage: 0,       // ← 这里是硬编码的 0
    limit: null,
    percentage: 0,  // ← 这里也是硬编码的 0
  };
}

// 只有 limit !== null 才走到这里
const usage = await this.getUsage(projectId, sourceType);
```

### 语义分析

付费版用户如果某类邮件的 limit 是 `null`（默认状态），`checkLimit` **完全跳过查 DB/Redis 的 usage**，直接返回 `usage: 0`。这是一种性能优化——因为无限额就不需要知道用了多少。

### 影响范围

| 调用方 | limit === null 时拿到 usage=0 的影响 |
|---|---|
| **EmailService（限制判断）** | ✅ 无影响——反正 allowed=true，usage 不用于判断 |
| **CampaignService 预检查** | ✅ 无影响——`limitCheck.limit === null` 时跳过 projectedUsage 判断 |
| **getLimitsAndUsage（前端展示）** | ⚠️ **不影响**——它直接调 `getUsage()` 逐个算，不经过 `checkLimit` 的这个分支 |
| **告警通知** | ✅ 无影响——limit 为 null 永远不会触发 warning/exceeded |

### 隐藏边界

`checkLimit` 是整个限制体系里唯一可能返回"假 usage=0"的地方。只要调用方不把 `checkLimit().usage` 当真值用于展示或统计，就没有问题。当前所有调用方都只关心 `allowed` 和 `warning`，用法是安全的。

---

## 三、PUT 限额时 update DB 与 clearCache 并发窗口

### 当前执行顺序

[Users.ts#L334-L376](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Users.ts#L334-L376)：

```
T0: project = prisma.project.findUnique()  // 读旧限额快照
T1: prisma.project.update(data)            // ① 写新限额到 DB
T2: clearNotificationCacheForChangedLimits(
      project,  // 用的是 T0 读的旧快照
      data,     // 新限额
    )
    │
    └── if (old.workflows !== new.workflows)
          redis.del(warningEmail_key, limitEmail_key)  // ② 清通知缓存
T3: getLimitsAndUsage()  // 返回给前端
```

### 并发窗口分析

#### 窗口 1：T0 与 T1 之间

如果另一个请求在 T0~T1 之间也修改了限额，`oldLimits`（T0 快照）和真实的旧值不一致，可能导致该清的 key 没清、不该清的被清。

| 场景 | 初始 | 请求 A（T0 读） | 请求 B（抢先提交） | 请求 A（T1 写+T2 清） | 结果 |
|---|---|---|---|---|---|
| workflows | 10000 | 读 old=10000, new=1000 | 改成 5000 | 写 1000，old=10000≠1000 → 清缓存 | ✅ 缓存清了，但 B 的修改被 A 覆盖 |
| campaigns | 10000 | 读 old=10000, new=10000（不变） | 改成 5000 | 写 10000（不变），old=new → 不清缓存 | ❌ B 改到了 5000 但缓存没清 |

更严重的是**写冲突本身**（A 覆盖 B），缓存只是衍生问题。当前没有乐观锁（没有 `updatedAt` 或 version 字段）。

#### 窗口 2：T1 与 T2 之间

DB 已经是新限额，但 Redis 里的 warning/limit notification key 还在。这期间如果恰好有邮件触发 `checkLimit` 并越过新阈值，可能**因为 SETNX key 已存在而不发告警邮件**。

最坏时长 = `clearNotificationCacheForChangedLimits` 的执行延迟（通常 < 10ms），加上后续 `redis.del` 的网络延迟。

#### 窗口 3：T2 之后用量缓存仍未清

`clearNotificationCacheForChangedLimits` **只清通知缓存**，不清 `billing:usage:*` 用量缓存。如果旧限额很大、新限额很小，而用量缓存的值恰好介于两者之间：

```
实际 usage: 500
旧 limit: 10000, 缓存 usage = 500 (OK)
新 limit: 100, 但缓存还在 → getUsage() 返回 500（超过新 limit 100）
```

实际上这不是 bug——因为 `getUsage` 缓存的是**用量数字**本身，和限额无关。500 就是 500，改不改限额它都没变。所以不清 usage 缓存是正确的。

### 顺序合理性

先写 DB 再清缓存（Cache-Aside 模式的写操作）是标准正确顺序。反过来先清缓存再写 DB，窗口期间读请求会把旧值回填缓存，导致脏读更久。

---

## 四、调小限额到已用量之下，整月无发送则永不触发告警

### 告警触发条件

告警邮件（warning + exceeded）**只在 `checkLimit` 调用时触发**：

```typescript
// checkLimit 内 [BillingLimitService.ts#L307-L330]
if (isWarning) {
  await NtfyService.notifyBillingLimitApproaching(...);
  await this.sendWarningEmail(...);  // ← 只有这里会发
}
if (usage >= limit) {
  await NtfyService.notifyBillingLimitExceeded(...);
  await this.sendLimitExceededEmail(...);  // ← 只有这里会发
}
```

### 场景复现

| 时间 | 事件 | 结果 |
|---|---|---|
| 6/1 | 用户升级付费，默认 workflows limit = null | unlimited，无告警 |
| 6/1 ~ 6/14 | 陆续发送 500 封 workflow 邮件 | 正常 |
| 6/14 10:00 | 用户把 workflows limit 从 null 调到 100 | DB 更新 + 清通知缓存。`getLimitsAndUsage` 返回 `{ usage: 500, limit: 100, isBlocked: true }`，前端显示红色 ✕ |
| 6/14 10:00 ~ 7/1 | 用户没再发邮件 | **checkLimit 从未被调用** → 超限邮件**永远不发** |

### 两条通知路径的对比

| 路径 | 是否触发 | 原因 |
|---|---|---|
| **前端 UI 展示** | ✅ 立即显示红色超限 | `getLimitsAndUsage` 独立计算，不需要走 checkLimit |
| **Ntfy 推送** | ❌ 不发 | 只在 checkLimit 内调 notifyBillingLimitExceeded |
| **邮件通知** | ❌ 不发 | 只在 checkLimit 内调 sendLimitExceededEmail |

### 业务后果

用户调小限额后看到前端红了，以为系统会通知他，但如果他之后不发邮件，就**永远收不到邮件和 Ntfy 告警**。如果用户本人不常看 Dashboard，超限状态可能持续整月都无人知晓。

### 对比 PUT /billing-limits 的「设计意图」

`clearNotificationCacheForChangedLimits` 设计目标是「允许新阈值下**再到阈值时**可以重发」，隐含假设是「用户调完限额后还会继续发邮件」。但这个假设在调小限额（立即已超限）的场景下不成立。

---

## 五、INBOUND webhook 跨多 project 的 for+continue 计数

### 代码结构

[Webhooks.ts#L124-L277](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Webhooks.ts#L124-L277)：

```
一封入站邮件到达 SNS
  → 解析 recipients（可能多个邮箱，如 info@ + support@）
  → for (const recipient of recipients)           // 外层循环：按收件人
      │
      ├── 解析域名 domain = recipient.split('@')[1]
      ├── domainRecords = prisma.domain.findMany({ where: { domain, verified: true }, include: { project: true } })
      │     └── ★ 同一个域名可能被多个项目验证（共享域名）
      │
      └── for (const domainRecord of domainRecords)  // 内层循环：按项目
            │
            ├── BillingLimitService.checkLimit(projectId, INBOUND)
            │     if (!allowed) continue;              // ★ 这个项目超了，跳过，继续下一个项目
            │
            ├── prisma.email.create(sourceType: INBOUND)
            ├── BillingLimitService.incrementUsage(projectId, INBOUND)
            ├── if (project.customer) MeterService.recordEmailSent(...)
            └── EventService.trackEvent('email.received', ...)
```

### 核心设计：按项目独立计数、独立限流

同一个域名 `company.com` 被 Project A 和 Project B 同时验证时，一封发到 `info@company.com` 的邮件会**同时被两个项目各处理一次**：

- Project A 限额够 → 创建 Email + incrementUsage + 触发 workflow
- Project B 限额已用完 → `continue` 跳过，什么都不发生
- 两者互不干扰

### 边界场景

| 场景 | 行为 |
|---|---|
| 同一收件人域名对应 N 个项目 | 每个项目独立 checkLimit + 计数，一封邮件最多收 N 份（N 个项目各存一份 Email 记录） |
| 外层有多个 recipients（info@ + support@） | **串行处理**，每个 recipient 各自走完整的 domainRecords 循环 |
| 某项目超限 | `continue` 只跳过该项目，不影响其他项目，也不中断外层 recipient 循环 |
| 某项目 DB/Redis 挂了 | `checkLimit` fail-open 返回 `allowed:true` → 仍然计数（fail-open 设计） |
| 免费版 + INBOUND | 计入共享 1000 封额度（免费版 getTotalUsage 是所有 sourceType 合计） |

### 业务公平性

对于**共享域名的多个项目**，先处理的项目不占用后处理项目的额度——因为每个项目的 INBOUND 计数是独立的。但免费版是个例外：免费版所有类别共享 1000 额度，如果 Project A（免费）的 INBOUND 触发了 900 封，Project B（也免费）再收到 INBOUND 时会发现 totalUsage 已超过 1000 而被拦。

**注意**：这里 `getTotalUsage` 是**按 projectId 聚合**的，不是跨项目共享。Project A 和 Project B 即使都免费也各有各的 1000 额度。上面的例子是我理解错了——每个 project 独立计数。真正的跨项目影响只发生在**同一个项目内**不同 sourceType 之间。

---

## 六、重复 payment_failed 与 checkout_completed 幂等

### 结论先行：都没有幂等保护，重复触发会产生副作用。

### invoice.payment_failed 的重复处理

[Webhooks.ts#L591-L642](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Webhooks.ts#L591-L642)：

```typescript
case 'invoice.payment_failed': {
  // 唯一过滤条件: billing_reason === 'subscription_create' 跳过
  // 没有: if (project.disabled && project.disabledReason === 'PAYMENT_FAILED') break;

  await prisma.project.update({       // 每次都写
    data: { disabled: true, disabledReason: 'PAYMENT_FAILED' },
  });

  await NtfyService.notifyProjectDisabledForPayment(...);  // 每次都推

  // 给所有成员发邮件
  await Promise.all(
    emails.map(email => sendPlatformEmail(email, 'Project Disabled - Payment Failed', template)),
  );
  // 没有 SETNX 去重！
}
```

Stripe 的 `invoice.payment_failed` 会**连续发送多次**（默认 dunning 周期是 4 次重试，每次都会发 webhook）。所以：

- **DB update**：重复写同样的 `disabled:true`，副作用不大，但会产生无意义的写入
- **Ntfy 推送**：会重复推送 4 次（NtfyService.notifyProjectDisabledForPayment 内部没有 SETNX 去重，参考 [NtfyService.ts#L176-L182](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/NtfyService.ts#L176-L182)）
- **邮件通知**：会重复发 4 次"Project Disabled"邮件，轰炸用户

### checkout.session.completed 的重复处理

[Webhooks.ts#L513-L558](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Webhooks.ts#L513-L558)：

```typescript
case 'checkout.session.completed': {
  // 没有检查: if (project.subscription) break;

  await prisma.project.update({
    data: { customer: customerId, subscription: subscriptionId },
  });

  await stripe.customers.update(customerId, {
    name: updatedProject.name,
    balance: creditBalance,   // ★ -100 或 -300 重复写入！
  });

  await NtfyService.notifySubscriptionStarted(...);  // 重复推送
}
```

Stripe 保证 webhook **at-least-once 送达**，所以 `checkout.session.completed` 可能重复投递。后果：

- **DB subscription 字段**：重复写同样的值，幂等
- **Stripe customer.balance**：⚠️ **每次都会减 100（或 300）美分**。如果 webhook 投递 3 次，用户凭空多拿了 2~6 刀 credit。这是真正的资金漏洞。
- **Ntfy 推送**：重复通知

### 对比 invoice.paid 的幂等保护

`invoice.paid` 的 re-enable 逻辑**有前置条件判断**（[Webhooks.ts#L578-L584](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Webhooks.ts#L578-L584)）：

```typescript
if (project.disabled && project.disabledReason === 'PAYMENT_FAILED') {
  await prisma.project.update({ data: { disabled: false, disabledReason: null } });
}
```

只有满足条件才更新，天然幂等。但 `NtfyService.notifyInvoicePaid` 和邮件通知仍然会重复发送。

### 三个 webhook 的幂等对比表

| Webhook 事件 | DB 写入 | Stripe API 调用 | Ntfy 推送 | 邮件通知 |
|---|---|---|---|---|
| `checkout.session.completed` | ✅ 幂等（覆盖写同值） | ❌ **重复扣 credit（资金风险）** | ❌ 重复 | — |
| `invoice.paid` | ✅ 幂等（有前置判断） | — | ❌ 重复 | ❌ 重复 |
| `invoice.payment_failed` | ✅ 幂等（覆盖写同值） | — | ❌ 重复 | ❌ **重复发 4 次** |
| `customer.subscription.deleted` | ✅ 幂等（写 null） | — | ❌ 重复 | — |
| `customer.subscription.updated` | —（不写 DB） | — | ❌ 重复 | — |

### Stripe 官方建议的幂等方式

Stripe webhook 头里带有 `Stripe-Signature`，验证了签名但**没有用 event.id 做幂等去重**。正确做法是把处理过的 `event.id` 存到 Redis/DB，重复事件直接 ack。
