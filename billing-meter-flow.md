# 账单计量 — 代码边界深度分析

## 全局总览（含 Dead-Letter 设计）

### 双轨计量体系

```
┌─────────────────────────────────────────────────────────────────────┐
│                         限制判断（内部用）                              │
│                                                                      │
│  Email 创建 (PENDING)                                                │
│   → BillingLimitService.incrementUsage()                             │
│     → redis.incr(billing:usage:{project}:{sourceType}:{YYYY-MM})    │
│                                                                      │
│   数据源: prisma.email.count() + Redis 缓存(5min TTL)                │
│   触发: checkLimit() 邮件入队前同步检查                                │
│   失败策略: Fail-Open (错误时 allowed:true)                          │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         计费计量（Stripe 账单）                         │
│                                                                      │
│  邮件发送成功 (SES 返回 messageId)                                   │
│   → email-processor.ts#L241-L248                                     │
│     → MeterService.recordEmailSent(customerId, count, idempotencyKey)│
│       → QueueService.queueMeterEvent(...)                            │
│         → BullMQ meterQueue (attempts:10, backoff:5s指数)           │
│           → meter-processor Worker (并发5, 限流50/秒)                │
│             → stripe.billing.meterEvents.create()                    │
│                                                                      │
│  ★ 幂等键: email_{emailId} / batch_{batchId}                         │
│  ★ 附件加倍: 有附件计 2 封                                           │
│  ★ 无 Dead-Letter Queue: 10次重试全失败后                          │
│     └─ Job 留在 meter:failed 集合（removeOnFail:10000）              │
│     └─ 没有补偿/重放机制                                             │
│     └─ 幂等键随 job.data 保留，但永远不会再被处理                     │
│     → 后果: 单边漏账（邮件已发但未计费）                                │
└─────────────────────────────────────────────────────────────────────┘
```

### BullMQ 队列配置一览（与计量/限制相关）

| Queue | attempts | backoff | removeOnFail | DLQ | 备注 |
|---|---|---|---|---|---|
| emailQueue | 3 | 2s 指数 | 5000 | ❌ 无 | 邮件发送 |
| meterQueue | **10** | 5s 指数 | 10000 | ❌ 无 | Stripe 计量事件 |
| campaignQueue | 3 | 5s 指数 | 500 | ❌ 无 | 营销活动批量 |
| workflowQueue | 3 | 2s 指数 | 5000 | ❌ 无 | 工作流步骤 |

所有队列**均未配置死信队列**，重试耗尽后作业留在 `{queue}:failed` Redis 集合中，后续完全依赖人工排查或清理。

---

## 计划限制

### 两种计费模式的切换逻辑

`BillingLimitService` 的核心判断见 [BillingLimitService.ts#L200-L227](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L200-L227)：

```typescript
public static getPlanTierLimit(
  project: { subscription: string | null; billingLimitWorkflows: number | null; ... },
  sourceType: EmailSourceType,
): number | null {
  const hasCustomLimits =
    project.billingLimitWorkflows !== null ||
    project.billingLimitCampaigns !== null ||
    project.billingLimitTransactional !== null ||
    project.billingLimitInbound !== null;

  // 免费版判定：STRIPE_ENABLED && 无订阅 && 无自定义限额
  if (STRIPE_ENABLED && !project.subscription && !hasCustomLimits) {
    return FREE_TIER_TOTAL_LIMIT;  // 常量 = 1000
  }

  // 付费版：按 sourceType 返回对应字段（null = unlimited）
  const limitMap: Record<EmailSourceType, number | null> = {
    [EmailSourceType.WORKFLOW]:      project.billingLimitWorkflows,
    [EmailSourceType.CAMPAIGN]:      project.billingLimitCampaigns,
    [EmailSourceType.TRANSACTIONAL]: project.billingLimitTransactional,
    [EmailSourceType.INBOUND]:       project.billingLimitInbound,
  };
  return limitMap[sourceType];
}
```

### 免费 vs 付费模式关键差异

| 维度 | 免费版 | 付费版 |
|---|---|---|
| 触发条件 | `!subscription && 四个 billingLimit 全 null` | `subscription 存在` 或 `任意 billingLimit 非 null` |
| 限额模式 | 共享额度：WORKFLOW + CAMPAIGN + TRANSACTIONAL + INBOUND 合计 1000 封/月 | 四类独立限额，各自独立计数 |
| 用量统计 | `getTotalUsage(projectId)` → `prisma.email.count({ where: { projectId, createdAt in 本月 } })` | `getUsage(projectId, sourceType)` → 按 sourceType 分开 count |
| limit 为 null 的含义 | N/A（免费版不可能为 null） | Unlimited，`checkLimit` 直接返回 allowed=true 且 usage 硬编码为 0 |

### Project 模型字段

[schema.prisma#L48-L60](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/packages/db/prisma/schema.prisma#L48-L60)：

```prisma
customer           String?  @unique
subscription       String?  @unique

// Billing Limits (per calendar month, null = unlimited)
billingLimitWorkflows     Int?
billingLimitCampaigns     Int?
billingLimitTransactional Int?
billingLimitInbound       Int?
```

### 限制判断时机的四个检查点

| 检查点 | 代码位置 | 触发条件 |
|---|---|---|
| **Transactional API** | [EmailService.ts#L93](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/EmailService.ts#L93) | 每封 Transactional 邮件入队前 |
| **Workflow 步骤** | workflow 执行器每次触发发送时 | 每封 Workflow 邮件入队前 |
| **Campaign 预检查** | [CampaignService.ts#L320-L336](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/CampaignService.ts#L320-L336) | Campaign 发送前预判 `usage + recipientCount` 是否超限 |
| **Campaign 单封循环** | campaign-processor.ts 每封发送前 | 单封邮件入队前兜底检查 |

Campaign 预检查示例：
```typescript
if (limitCheck.limit !== null) {
  const projectedUsage = limitCheck.usage + recipientCount;
  if (projectedUsage > limitCheck.limit) {
    throw new HttpException(403, 'Campaign recipients exceed monthly billing limit');
  }
}
```

### 告警阈值

常量定义在 [BillingLimitService.ts#L29-L30](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L29-L30)：
```typescript
WARNING_THRESHOLD = 0.8;   // 80% 触发警告
LIMIT_THRESHOLD   = 1.0;   // 100% 阻断
```

---

## 发送量计数

### 双轨计量架构

```
                ┌── Email.create (PENDING)
                │     → BillingLimitService.incrementUsage()
                │       ├── redis.incr(billing:usage:{project}:{sourceType}:{YYYY-MM})
                │       └── (失败时静默，不阻断邮件)
                │
                │   ★ 用途：内部限制判断
                │   ★ 时机：邮件创建时（PENDING 状态）
                │   ★ 粒度：按 project + sourceType + 自然月
                │
                └── SES 返回 messageId（SENT 状态）
                      → email-processor.ts#L241-L248
                        → MeterService.recordEmailSent(customerId, count, idempotencyKey)
                          → QueueService.queueMeterEvent(...)
                            → BullMQ meterQueue
                              → meter-processor Worker
                                → stripe.billing.meterEvents.create()

                      ★ 用途：Stripe 账单计费
                      ★ 时机：邮件发送成功时（SENT 状态）
                      ★ 粒度：按 Stripe customerId
```

### 内部用量计数（限制判断用）

[BillingLimitService.ts#L102-L135](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L102-L135)：

```typescript
public static async incrementUsage(projectId: string, sourceType: EmailSourceType): Promise<void> {
  try {
    const cacheKey = this.getCacheKey(projectId, sourceType);
    const exists = await redis.exists(cacheKey);
    if (exists) {
      await redis.incr(cacheKey);          // 缓存命中：直接 +1
    }
    // 缓存未命中：什么都不做，等下次 getUsage 时从 DB 回填
  } catch (error) {
    signale.warn(`[BILLING_LIMIT] Failed to increment usage cache:`, error);
    // Fail-Open：redis 挂了也不阻断业务
  }
}
```

`getUsage` 逻辑 [BillingLimitService.ts#L61-L100](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L61-L100)：
- 先查 Redis 缓存（TTL 300 秒 = 5 分钟）
- 缓存 miss 时查 DB `prisma.email.count()`，并回写 Redis（含 `CACHE_TTL`）

### Stripe 计量（计费用）

[MeterService.ts#L19-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/MeterService.ts#L19-L53)：

```typescript
public static async recordEmailSent(
  customerId: string,
  value = 1,
  idempotencyKey?: string,
): Promise<void> {
  await QueueService.queueMeterEvent({ customerId, value, idempotencyKey });
}
```

两个计量场景：
1. **Transactional/Workflow/Campaign 单封**：`recordEmailSent(customerId, count, 'email_' + emailId)`
   - 附件加倍：`count = hasAttachments ? 2 : 1`
   - 幂等键：`email_{emailId}`
2. **Campaign 批处理**：`recordEmailBatch(customerId, recipientCount, 'batch_' + batchId)`
   - 幂等键：`batch_{batchId}`

Worker 实际发送 [meter-processor.ts#L20-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/jobs/meter-processor.ts#L20-L58)：
- 并发 5，`groupKey: customerId` 保证同一 customer 的事件顺序
- 全局限流 50 次/秒
- Stripe SDK 自带 `idempotencyKey` 参数，防止重复计费

---

## 前端呈现

### 数据获取 Hooks

| Hook | 刷新频率 | 用途 |
|---|---|---|
| [useBillingLimits.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/lib/hooks/useBillingLimits.ts) | SWR 30 秒自动刷新 | 获取四类独立限额 + 用量，编辑限额 |
| [useBillingConsumption.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/lib/hooks/useBillingConsumption.ts) | SWR 60 秒自动刷新 | 获取 Stripe 账单消费、账户余额、即将到来的发票（付费用户专用） |

### BillingLimits 组件

[BillingLimits.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/components/BillingLimits.tsx) 核心渲染分支：

```
data = useBillingLimits()
  │
  ├── isFreeTier === true
  │     └── 单卡片展示：Total Emails（1000 封共享额度）
  │           颜色: <80% 灰✓ / 80-100% 琥珀⚠ / ≥100% 红✕
  │           底部 CTA: "Upgrade to Pro"
  │
  └── isFreeTier === false
        └── 四卡片展示：Workflows / Campaigns / Transactional / Inbound
                    每卡独立三态颜色编码
                    数值编辑（数字输入框）
                    保存按钮 → PUT /billing-limits
```

颜色三态映射：
```typescript
// 安全：灰底 + Check Icon
if (category.percentage < 0.8)  →  variant: 'default',  icon: Check

// 警告：琥珀底 + AlertTriangle
if (percentage >= 0.8 && < 1.0) →  variant: 'warning',  icon: AlertTriangle

// 阻断：红底 + AlertCircle
if (percentage >= 1.0)          →  variant: 'destructive', icon: AlertCircle
```

### BillingConsumption 组件（付费用户账单面板）

[BillingConsumption.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/components/BillingConsumption.tsx)：
- Total Usage：本月总发送量（四类合计）
- Account Balance：Stripe 账户余额（负数 = 欠款）
- Upcoming Invoice：预计下次发票金额

### 后端返回结构

API 类型定义在 [billing.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/packages/types/src/api/billing.ts)：

```typescript
interface CategoryUsage {
  sourceType: 'WORKFLOW' | 'CAMPAIGN' | 'TRANSACTIONAL' | 'INBOUND';
  usage: number;
  limit: number | null;
  percentage: number;     // 0~1，limit=null 时为 0
  isWarning: boolean;     // percentage >= 0.8
  isBlocked: boolean;     // percentage >= 1.0
}

interface BillingLimitsResponse {
  isFreeTier: boolean;
  categories: CategoryUsage[];
  totalUsage?: number;    // 免费版：共享额度用量
  totalLimit?: number;    // 免费版：1000
}
```

---

## 失败降级

### 全链路 Fail-Open 设计哲学

系统的核心哲学：**计量/限制/通知全部是可失败的，邮件发送本身不能因为辅助系统故障而中断。**

| 组件 | 失败模式 | 降级行为 | 代码位置 |
|---|---|---|---|
| Redis 用量缓存读 | 网络故障/超时 | 直接查 DB，用完即使缓存写失败也不报错 | [BillingLimitService.ts#L61-L100](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L61-L100) |
| Redis 用量缓存写 | `redis.incr` 抛异常 | 打 warn 日志，继续执行邮件入队 | [BillingLimitService.ts#L127-L134](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L127-L134) |
| `checkLimit` 整体异常 | DB/Redis 都挂 | 返回 `{ allowed: true, warning: false }`，完全放行 | [BillingLimitService.ts#L340-L346](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L340-L346) |
| `clearNotificationCache` | `redis.del` 失败 | 打 warn 日志，不返回错误给调用方（→ 旧 SETNX 持续到月底） | [BillingLimitService.ts#L561-L563](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L561-L563) |
| 告警邮件发送 | `sendPlatformEmail` 抛异常 | 打 warn 日志，不影响限制判断 | BillingLimitService.sendWarningEmail / sendLimitExceededEmail |
| Ntfy 推送 | 网络异常 | 打 warn 日志，静默失败 | NtfyService 所有方法 |
| Meter Queue 入队 | `meterQueue.add` 失败 | 打 warn 日志，邮件已经发出，只能寄望于后续补偿 | [MeterService.ts#L34-L40](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/MeterService.ts#L34-L40) |
| Meter Worker 10 次重试全失败 | Stripe API 长时间不可用 | Job 留在 `meter:failed` 集合，没有 DLQ / 没有补偿 / 幂等键永不复用 → **单边漏账** | [meter-processor.ts#L49-L51](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/jobs/meter-processor.ts#L49-L51) |
| INBOUND webhook 处理异常 | 解析/DB/Redis 任何环节抛异常 | 返回 HTTP 200 给 SNS（防止重投风暴），邮件默默丢失 | [Webhooks.ts#L280-L284](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Webhooks.ts#L280-L284) |

### checkLimit Fail-Open 具体代码

```typescript
public static async checkLimit(projectId: string, sourceType: EmailSourceType)
  : Promise<LimitCheckResult> {
  try {
    // ... 正常判断逻辑 ...
  } catch (error) {
    signale.error(`[BILLING_LIMIT] Error checking limit for ${projectId}:`, error);
    // Fail-Open: 限制系统出问题时宁可放行也不误伤
    return {
      allowed: true,
      warning: false,
      usage: 0,
      limit: null,
      percentage: 0,
      message: null,
    };
  }
}
```

### Fail-Open 的取舍分析

| 维度 | 影响 |
|---|---|
| **发送率** | 不受基础设施影响，用户感知最佳 |
| **限制准确性** | 可能短暂超量（缓存 miss + DB 挂时） |
| **收入损失** | Meter Worker 失败会导致漏账（用量收不到钱） |
| **运维风险** | 告警静默失败意味着用户超量但运营不知道 |

---

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

---

## 七、clearNotificationCache fail-soft 导致旧 SETNX key 持续到月底

### 代码证据

[BillingLimitService.ts#L516-L563](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L516-L563)：

```typescript
public static async clearNotificationCacheForChangedLimits(...) {
  try {
    // ... 构建 keysToDelete ...
    if (keysToDelete.length > 0) {
      await Promise.all(keysToDelete.map(key => redis.del(key)));
    }
  } catch (error) {
    signale.warn(`[BILLING_LIMIT] Failed to clear notification cache for ${projectId}:`, error);
    // ↑ 这里是 fail-soft：打个 warn 日志就完了，没有重试、没有补偿
  }
}
```

### SETNX key 的生命周期

```
发送告警邮件时：
  redis.set(key, '1', 'EX', ttl, 'NX')  // [BillingLimitService.ts#L610]
                              ↑
                     ttl = 月底时间戳 - 当前时间
                     = 剩余天数（最多 31 天）
```

SETNX key 的 TTL 是**到月底（end of month）**，不是固定的 24 小时。参考 [BillingLimitService.ts#L606-L610](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L606-L610)：

```typescript
const endOfMonth = new Date(now.getFullYear(), now.getMonth() + 1, 1);
const ttl = Math.floor((endOfMonth.getTime() - now.getTime()) / 1000);
```

### 问题场景

| 时间 | 事件 |
|---|---|
| 6/1 10:00 | 用户触发了 80% 警告 → SETNX key `billing:warning_email:...` 写入，TTL 到 7/1 00:00 |
| 6/15 14:00 | 用户把 limit 从 1000 调到 500 → 调用 `clearNotificationCacheForChangedLimits` |
| 6/15 14:00:02 | **Redis 恰好分区/网络抖动** → `redis.del` 抛异常被 catch → fail-soft |
| 6/15 14:00:03 | DB 已更新为 500，但 warning/limit 的 SETNX key 还在 Redis 里 |
| 6/15 ~ 6/30 | 用户再发邮件 → checkLimit 调 sendWarningEmail → 因为 SETNX key 还在，直接 return → **整整半个月再也收不到告警** |
| 7/1 00:00 | SETNX key TTL 到期自动过期 |

### 修复建议

- 单个 key 删除失败时应针对失败的 key 做**重试**
- 或者用 `redis.unlink`（异步非阻塞删除）减少失败概率
- 或者在 PUT 限额返回前**再读取一次这些 key** 确认已删除，没删掉就补删一次

---

## 八、八 key Promise.all 非 MULTI，部分成功部分失败

### 当前实现

[BillingLimitService.ts#L555-L556](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L555-L556)：

```typescript
await Promise.all(keysToDelete.map(key => redis.del(key)));
```

### 四类 × 两种 = 最多 8 个独立命令

一次调用可能要清 8 个 key：

| 类别 | Warning key | Limit key |
|---|---|---|
| WORKFLOWS | `billing:warning_email:proj:WORKFLOW:2026-06` | `billing:limit_email:proj:WORKFLOW:2026-06` |
| CAMPAIGNS | `billing:warning_email:proj:CAMPAIGN:2026-06` | `billing:limit_email:proj:CAMPAIGN:2026-06` |
| TRANSACTIONAL | `billing:warning_email:proj:TRANSACTIONAL:2026-06` | `billing:limit_email:proj:TRANSACTIONAL:2026-06` |
| INBOUND | `billing:warning_email:proj:INBOUND:2026-06` | `billing:limit_email:proj:INBOUND:2026-06` |

这 8 个 `redis.del` 是 **8 个独立的 TCP 命令**，用 Promise.all 并发发出，**没有 MULTI/EXEC 事务**。

### 可能的中间态

| 场景 | 结果 |
|---|---|
| 全部成功 | ✅ 预期行为 |
| 前 4 个成功、后 4 个失败 | ✗ 一半 key 清了、一半没清 |
| warning 清了、limit 没清 | ✗ 用户可能还会收到"超限"邮件但收不到"警告"邮件 |
| Promise.all 中某一个 reject | ✗ 整个 Promise.all reject，进入 catch，后面的 del 被放弃 |

### 对比：应该怎么做？

```typescript
// ❌ 当前：8 条独立命令，可能部分成功
await Promise.all(keysToDelete.map(key => redis.del(key)));

// ✅ 方案 A：MULTI 事务，要么全删要么全不删
const multi = redis.multi();
keysToDelete.forEach(key => multi.del(key));
await multi.exec();

// ✅ 方案 B：DEL 支持可变参数，一条命令搞定（推荐）
await redis.del(...keysToDelete);
```

`redis.del(key1, key2, key3)` 是**单条命令原子执行**，不存在中间态。并且把 8 条命令减少到 1 条，还能省 7 个 RTT。

---

## 九、INBOUND 同步处理触发 SNS 30 秒超时

### AWS SNS 对 HTTP/S 订阅的超时约束

SNS 要求订阅端点必须在 **30 秒**内返回 HTTP 200/202/204，否则 SNS 会判定投递失败并开始重试（指数退避，最多重试 100,015 次，持续 23 天 21 小时 1 分钟）。

### INBOUND 处理链路中的同步阻塞点

[Webhooks.ts#L111-L284](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Webhooks.ts#L111-L284) 在一个请求里**同步串行**执行以下操作：

```
for (每个 recipient)
  │
  ├── prisma.domain.findMany()           // 1 次 DB 查询
  │
  ├── 解析邮件内容（base64 + simpleParser + sanitizeHtml）
  │     └── sanitizeHtml 处理大量 HTML 可能耗时数百毫秒
  │
  └── for (每个 domainRecord)            // ★ 串行！
        │
        ├── BillingLimitService.checkLimit()
        │     ├── prisma.project.findUnique()    // DB 查 project
        │     ├── 免费版: prisma.email.count()   // DB count
        │     └── 付费版: getUsage() → redis.get 或 prisma.email.count()
        │
        ├── ContactService.upsert()        // DB upsert（可能 insert）
        │
        ├── prisma.email.create()          // DB insert
        │
        ├── BillingLimitService.incrementUsage()  // redis.incr
        │
        ├── MeterService.recordEmailSent()        // BullMQ addJob（redis L/RPUSH）
        │
        └── EventService.trackEvent()             // 可能触发 workflow 步骤、DB 写入...
```

所有操作**完全同步串行**，没有 `await Promise.all()` 批处理。

### 放大超时的因子

| 因子 | 说明 |
|---|---|
| 多收件人邮件 | 如群发邮件 `To: a@x.com, b@x.com, c@x.com` → 外层循环 3 次 |
| 共享域名多项目 | 1 个域名被 5 个项目验证 → 内层循环 5 次 → **每封邮件 15 次完整处理** |
| 大邮件/多附件 | `body.content` 可能几百 KB，base64 decode + simpleParser + sanitizeHtml 阻塞 |
| DB 慢查询 | `prisma.email.count()` 带复合索引在大数据量下可能 >100ms |
| free tier 检查 | 免费版 INBOUND 会调 `getTotalUsage` → count 所有 sourceType 的 Email |

### 后果

- **SNS 超时后重试投递** → 如果超时是因为量大，重试时再处理一遍又会超时 → **恶性循环**
- 最严重的问题：SNS 重试意味着同一封入站邮件会被**重复处理多次** → 下一节会讲的重复计数问题

### 建议架构

```
POST /webhooks/sns (Received 分支)
  │
  ├── 立刻从 body.mail.messageId 做幂等检查（Redis SETNX）
  ├── 仅解析少量元数据（sender, recipients, subject, domainRecords 列表）
  ├── 投递到 inboundQueue（BullMQ，异步）
  └── 立即返回 HTTP 200  ← 整个过程 <100ms

inboundWorker:
  └── 慢慢处理：完整解析 + checkLimit + create Email + metering + trackEvent
```

---

## 十、共享域名 N customer 触发 N 次独立 metering

### 代码位置

内层循环 [Webhooks.ts#L192-L276](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Webhooks.ts#L192-L276)：

```typescript
for (const domainRecord of domainRecords) {
  // ...
  await prisma.email.create({ projectId: domainRecord.projectId, ... });
  await BillingLimitService.incrementUsage(domainRecord.projectId, INBOUND);

  // ★ 每个项目独立 metering
  if (domainRecord.project.customer) {
    await MeterService.recordEmailSent(
      domainRecord.project.customer,   // ← 不同项目不同 customerId
      1,
      `email_${inboundEmail.id}`,      // ← inboundEmail.id 是各自独立的 PK
    );
  }
  // ...
}
```

### 场景说明

假设域名 `company.com` 同时被 Project A（customerId: cus_A）和 Project B（customerId: cus_B）验证。

| 路径 | 结果 |
|---|---|
| Project A | 创建 Email A.id=1001，调用 `stripe.meterEvents.create({ customer: cus_A, value: 1, identifier: email_1001 })` |
| Project B | 创建 Email B.id=2047，调用 `stripe.meterEvents.create({ customer: cus_B, value: 1, identifier: email_2047 })` |

**结论**：同一封真实邮件被两个项目各接收一次，**Stripe 会对 cus_A 和 cus_B 各收一次费用**。

### 这是 bug 还是 feature？

这是**有意设计**，因为：
1. 两个项目各创建了一份独立的 Email 记录，各触发了一次 workflow，各占用了各自项目的额度
2. `inboundEmail.id` 不一样，幂等键不冲突
3. 用户选择把同一个域名在两个项目里验证，自然期望两边都能收到邮件

### 需要注意的边界

- Stripe meter event 是 **按量计费**，多项目共享域名意味着发送端的一封邮件 → 平台对 N 个 Stripe customer 各收一次费
- AWS SES 入站接收**按每封邮件（每个收件人）计费一次**，不会因为多个项目处理而加倍
- 因此**共享域名场景下，平台的毛利率高于独占域名场景**（成本 1 次，收入 N 次）

---

## 十一、无 SNS messageId 幂等，SNS 重投递导致 INBOUND 重复计数

### 关键发现：INBOUND 处理完全没有任何幂等保护

对比 OUTBOUND 事件（Bounce/Delivery/Open...）的处理：

```typescript
// OUTBOUND: [Webhooks.ts#L288-L293]
const messageId = body.mail?.messageId;
if (!messageId) {
  return res.status(400).json({success: false, error: 'No messageId found'});
}
const email = await prisma.email.findUnique({ where: { messageId } });
```

OUTBOUND 至少用了 `messageId` 去**查 Email 是否存在**（隐式幂等）。而 INBOUND 呢：

```typescript
// INBOUND: [Webhooks.ts#L111-L284]
// 整个 Received 分支里：
// - 没有读 req.body['MessageId']（SNS 级的重投递去重用）
// - 没有用 body.mail.messageId 查是否已创建过 Email
// - 没有 Redis SETNX 去重
// - 什么都没有！
```

### 两种来源的重投递

| 来源 | 触发条件 | 标识字段 |
|---|---|---|
| **SNS 重投递** | 上一节说的 30 秒超时，或任何非 2xx 响应 | `req.body['MessageId']`（SNS 消息级唯一） |
| **SES 重复投递** | SES 内部机制偶发同一邮件投递两次 | `body.mail?.messageId`（SES mail 级唯一） |

### 最坏情况

- 因为处理链路长 → SNS 超时 → SNS 重投 → 处理链路再超时 → 重投 → ...（循环）
- **每一次重投都会**：
  1. 为每个项目 `prisma.email.create()` 一条重复的 INBOUND Email 记录
  2. `incrementUsage()` 把每个项目的 Redis 计数各加一遍
  3. 为每个项目的 customerId 各生成一条 Stripe meter event → **对用户重复计费**
  4. 触发 `email.received` 事件 → 可能重复触发自动化 workflow

### 快速定位字段

SNS 消息结构：
```json
{
  "Type": "Notification",
  "MessageId": "77ea5252-4925-444d-9152-xxxxxxxxxxxx",  // ← SNS 消息 ID
  "Message": "{\"notificationType\":\"Received\",\"mail\":{\"messageId\":\"ses-message-id-123\",...}}"
}
```

- 外层 `MessageId` 用 `req.body.MessageId` 取到 → 防 SNS 重复投递
- 内层 `body.mail.messageId`（SES 生成的 mail message ID）→ 防 SES 层面的重复

### 最小修复建议

在 INBOUND Received 分支**最开头**加上：

```typescript
const snsMessageId = req.body.MessageId;
if (snsMessageId) {
  const dedupKey = `sns:inbound:${snsMessageId}`;
  const wasSet = await redis.set(dedupKey, '1', 'EX', 86400, 'NX');
  if (!wasSet) {
    signale.info(`[WEBHOOK] Duplicate SNS message ${snsMessageId} skipped`);
    return res.status(200).json({success: true, message: 'Duplicate skipped'});
  }
}
```

---

## 十二、customer.subscription.deleted 覆盖写 null 触发 updatedAt

### Prisma @updatedAt 语义

Project 模型定义 [schema.prisma#L77-L79](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/packages/db/prisma/schema.prisma#L77-L79)：

```prisma
model Project {
  // ...
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt   // ← Prisma 自动管理
}
```

`@updatedAt` 表示**任何字段的 update 操作**（即使写入值和原值完全相同）都会把这个字段刷新为当前时间戳。

### 覆盖写 null 的场景

`customer.subscription.deleted` 处理 [Webhooks.ts#L645-L672](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Webhooks.ts#L645-L672)：

```typescript
case 'customer.subscription.deleted': {
  // ...
  await prisma.project.update({
    where: { id: project.id },
    data: {
      subscription: null,  // ← 即使本来就是 null，也会触发 updatedAt
    },
  });
  break;
}
```

**覆盖写** = 不管原值是什么，无条件写 `null`。Prisma 对这种写法的行为：

| 场景 | 原 subscription 字段 | 写入值 | updatedAt 是否更新？ |
|---|---|---|---|
| 正常取消订阅 | `sub_abc123` | `null` | ✅ **更新**（正确，因为字段确实变了） |
| Stripe 重投递 webhook | 已经是 `null` 了 | `null` | ✅ **仍然更新**！（这是问题所在） |

**注意**：Prisma 的 `@updatedAt` 行为是「执行 update SQL 就更新」，不是「字段实际变化才更新」。`UPDATE project SET subscription = NULL, "updatedAt" = NOW() WHERE id = ...` 这个 SQL 不管原值是什么都要执行，所以 `updatedAt` 每次都会被刷新。

### 对比 checkout.session.completed

```typescript
await prisma.project.update({
  data: {
    customer: customerId,
    subscription: subscriptionId,
  }
});
```

同样是覆盖写，如果 customerId / subscriptionId 重复，也会一样无意义地刷 updatedAt。

### 实际影响

- 审计上有误导：查询「最近有修改的项目」时，重复收到的 webhook 会把项目顶上来，但用户其实什么操作都没做
- 如果有业务逻辑依赖 updatedAt 排序列表或做增量同步（比如「拉取最近 24h 有更新的项目」），会包含大量噪声数据
- 建议：加前置判断 `if (project.subscription !== null)` 再写 update（就像 `invoice.paid` 里 `if (project.disabled && ...)` 那样的模式）

---

## 十三、checkout.session.completed 无 client_reference_id 与 customer 一致性校验

### 当前代码

[Webhooks.ts#L513-L531](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Webhooks.ts#L513-L531)：

```typescript
case 'checkout.session.completed': {
  const session = event.data.object;
  const customerId = session.customer as string;
  const subscriptionId = session.subscription as string;
  const projectId = session.client_reference_id;

  if (!projectId) {
    signale.warn('[WEBHOOK] No client_reference_id in checkout session');
    break;
  }

  // 直接写！没有任何一致性校验：
  // 1. 不校验 project.customer === session.customer（如果 project 已有 customer）
  // 2. 不校验 project.subscription 是否本来就是 null
  // 3. 不校验 session.customer 是否为空
  const updatedProject = await prisma.project.update({
    where: { id: projectId },
    data: {
      customer: customerId,
      subscription: subscriptionId,
    },
  });
```

### `client_reference_id` 的信任问题

`client_reference_id` 是前端传进去的 [Users.ts#L227-L230](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Users.ts#L227-L230)：

```typescript
const session = await stripe.checkout.sessions.create({
  // ...
  client_reference_id: project.id,  // ← 后端写入 Stripe
});
```

虽然是后端写入的，但因为 Stripe 只是把它原样放在 webhook 里回传，**签名只保证这个字段确实是 Stripe 看到的那份，不保证没被用户用恶意手段改了**（其实签名了所以字段不会被篡改——不过这里要验证的是另一个维度）。

### 真正的风险：Session 劫持

想象以下攻击场景（实际上需要 Stripe 被攻破，概率极低，但属于纵深防御问题）：

| 步骤 | 事件 |
|---|---|
| 1 | 用户 A 合法 checkout，`client_reference_id=proj_A`，`customer=cus_A` |
| 2 | 攻击者 B 截获这个 webhook（需要突破 Stripe 签名，假设发生），把 `client_reference_id` 改成 `proj_C`（属于受害者 C），但保留 `customer=cus_A` |
| 3 | 后端执行 `prisma.project.update({ where: { id: proj_C }, data: { customer: cus_A, subscription: sub_A } })` |
| 4 | **proj_C 的 customer 被篡改为 cus_A** |
| 5 | proj_C 后续发邮件时，`meterQueue` 会把使用量记到 **cus_A 的 Stripe 账上**，proj_C 的主人凭空被换成陌生人 |

### 更现实的风险：字段错乱

`client_reference_id` 是 `project.id`，但 `session.customer` 是 Stripe 的 `cus_xxx`。两者之间的正确绑定应该在创建 checkout session 时由后端一起设置：

```typescript
// Users.ts createCheckoutSession
const session = await stripe.checkout.sessions.create({
  customer: project.customer ?? undefined,  // ← A: 这里写了 cus_A 或 undefined
  client_reference_id: project.id,          // ← B: 这里写了 proj_A
});
```

如果 project.customer 已经是 `cus_old`，而 Stripe 返回了 `customer=cus_new`，webhook 就会覆盖写。正常情况下不会发生，但如果创建 checkout session 和 webhook 到达之间有并发事件（比如另一个 checkout session 走了不同的 project），就会出现不一致。

### 应该加的校验

```typescript
// 查找 project 时，把当前 customer/subscription 也带出来
const project = await prisma.project.findUnique({ where: { id: projectId } });

if (!project) break;

// 如果 project 已经有 customer，必须和 session.customer 一致
if (project.customer && project.customer !== customerId) {
  signale.error(`[WEBHOOK] Customer mismatch for project ${projectId}: ` +
    `expected ${project.customer}, got ${customerId}`);
  break;
}

// 如果已经有 subscription 且不是同一个，拒绝（防止覆盖）
if (project.subscription && project.subscription !== subscriptionId) {
  signale.error(`[WEBHOOK] Subscription already exists for project ${projectId}: ` +
    `existing ${project.subscription}, new ${subscriptionId}`);
  break;
}
```

