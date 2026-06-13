# 账单限制与用量计量 —— 代码理解分析

## 一、系统总览

本系统是一个邮件营销平台（Plunk），围绕**计划限制**、**发送量计数**、**前端用量展示**三大核心模块构建了完整的账单计量与限制体系。系统同时支持免费版（Free Tier）与付费订阅版，并通过 Stripe 进行按量计费（pay-per-email）。

核心代码分布：

| 层 | 关键文件 |
|---|---|
| 后端服务 | [BillingLimitService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts) |
| 后端服务 | [MeterService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/MeterService.ts) |
| 发送服务 | [EmailService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/EmailService.ts) |
| 发送服务 | [CampaignService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/CampaignService.ts) |
| 后台任务 | [email-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/jobs/email-processor.ts) |
| 后台任务 | [meter-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/jobs/meter-processor.ts) |
| 控制器 | [Users.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Users.ts) |
| 前端组件 | [BillingLimits.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/components/BillingLimits.tsx) |
| 前端组件 | [BillingConsumption.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/components/BillingConsumption.tsx) |
| 前端 Hooks | [useBillingLimits.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/lib/hooks/useBillingLimits.ts) |
| 前端 Hooks | [useBillingConsumption.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/lib/hooks/useBillingConsumption.ts) |
| 数据模型 | [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/packages/db/prisma/schema.prisma) |

---

## 二、计划限制（Billing Limits）

### 2.1 两类计划模式

#### 免费版（Free Tier）
- 触发条件：`STRIPE_ENABLED && !project.subscription && !hasCustomLimits`（参考 [BillingLimitService.ts#L203](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L203-L203)）
- 限制值：`FREE_TIER_TOTAL_LIMIT = 1000` 封邮件/月，**所有邮件类别共享此额度**
- 共享类别：WORKFLOW + CAMPAIGN + TRANSACTIONAL + INBOUND 四类统一计数

#### 付费版（Paid Tier）
- 触发条件：有 `subscription` 或设置了**任意**自定义 per-category 限制
- 四类独立限制（可独立设置为 `null` 表示无限）：
  - `billingLimitWorkflows` —— 工作流邮件
  - `billingLimitCampaigns` —— 营销活动邮件
  - `billingLimitTransactional` —— 事务性邮件
  - `billingLimitInbound` —— 入站邮件

### 2.2 限制判断时机（Check Timing）

限制检查发生在**邮件入队前**，而非实际发送时。这是关键的业务约束边界：

| 发送路径 | 检查位置 | 说明 |
|---|---|---|
| 事务性 API | [EmailService.sendTransactionalEmail#L78](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/EmailService.ts#L78-L82) | 调用 API 时同步检查 |
| 营销活动 | [CampaignService.send#L323](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/CampaignService.ts#L323-L336) | **活动级别预判**：发送前先 `projectedUsage = usage + recipientCount` 做整体校验 |
| 工作流 | [EmailService.sendWorkflowEmail#L238](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/EmailService.ts#L238-L245) | 工作流每步执行时检查 |
| 单封活动邮件 | [EmailService.sendCampaignEmail#L139](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/EmailService.ts#L139-L146) | 批量循环中每封检查 |

**关键设计**：CampaignService 在活动发送前做了一次"**整体预检查**"（`usage + recipientCount` 对比 limit），避免中途部分发送部分失败。但注意，这只是检查，并没有原子地"预占额度"。

### 2.3 限制数据存储

限制字段直接存储在 Project 模型中（[schema.prisma#L53-L57](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/packages/db/prisma/schema.prisma#L53-L57)）：

```prisma
model Project {
  billingLimitWorkflows     Int?
  billingLimitCampaigns     Int?
  billingLimitTransactional Int?
  billingLimitInbound       Int?
}
```

---

## 三、发送量计数（Usage Metering）

系统采用**双轨计量**：内部用量计数（限制判断用）+ Stripe Meter Event（计费用）。

### 3.1 内部用量计数（限制判断用）

#### 数据来源
通过统计 `Email` 表中当前自然月的记录数，依赖复合索引 `(projectId, sourceType, createdAt)` 加速查询（[schema.prisma#L588](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/packages/db/prisma/schema.prisma#L588-L588)）。

#### 计数缓存策略（Redis）
- **缓存 Key**：`billing:usage:{projectId}:{sourceType}:{YYYY}-{MM}`，定义于 [keys.ts#L37-L39](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/keys.ts#L37-L39)
- **TTL**：300 秒（5 分钟），见 [BillingLimitService.ts#L32](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L32-L32)
- **写入路径**：
  1. 首次查询时从 DB 拉取并缓存（`getUsage`）
  2. 每成功创建一封 Email 记录后调用 `incrementUsage` 做 `redis.incr`（[EmailService.sendTransactionalEmail#L108](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/EmailService.ts#L108-L108)）
- **缓存失效条件**：只有当 key 已存在时才 increment，否则等下一次 getUsage 从 DB 回填

#### 月份范围
使用自然月（calendar month）：本月 1 日 00:00:00 至下月 1 日 00:00:00（[BillingLimitService.ts#L579-L584](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L579-L584)）。

### 3.2 Stripe 计费计量（账单用）

#### 数据流
```
Email 发送成功 (SES 返回 messageId)
  → email-processor.ts#L244-L248
    → MeterService.recordEmailSent(customerId, count, idempotencyKey)
      → QueueService.queueMeterEvent(...)  // 写入 BullMQ meterQueue
        → meter-processor.ts Worker
          → stripe.billing.meterEvents.create(...)
```

#### 关键细节
- **幂等 Key**：`email_{emailId}`，防止重试重复计费（[email-processor.ts#L247](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/jobs/email-processor.ts#L247-L247)）
- **附件加倍计费**：含附件邮件计 `2` 封（[email-processor.ts#L245-L246](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/jobs/email-processor.ts#L245-L246)）
- **批量活动**：`recordEmailBatch` 使用 `batch_{batchId}` 做幂等键
- **异步可靠性**：meterQueue 配置 10 次重试 + 指数退避（[QueueService.ts#L164-L175](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/QueueService.ts#L164-L175)）
- **限流**：50 次/秒（Stripe 安全余量），Worker 并发 5

---

## 四、并发计数与限制判断的竞争条件

### 4.1 并发竞态分析（Race Conditions）

这是整个系统最关键的边界问题。测试用例 [BillingLimitService.test.ts#L405-L450](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/__tests__/BillingLimitService.test.ts#L405-L450) 专门测试了这个场景：

```
现有 8/10 → 并发发送 5 封 → 可能超过限制
```

**存在的问题**：

1. **检查与写入分离（TOCTOU）**：`checkLimit` 读 usage，`prisma.email.create` 写 Email，`incrementUsage` 写缓存，三者之间没有事务或分布式锁
2. **Redis increment 非原子组合**：`getUsage` 与 `checkLimit` 之间可能被其他请求插入
3. **缓存 TTL 窗口**：缓存 miss 回源 DB 的窗口内，并发请求都会读到旧值

**系统当前策略**：
- **Fail-Open（宽松）**：错误时允许发送（[BillingLimitService.ts#L343-L351](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L343-L351)），避免服务中断
- **容忍少量超发**：测试断言 `totalEmails <= 15`（限制 10 的情况下允许 5 封超发）
- **Campaign 级预检查**：对营销活动做整体 `usage + recipientCount` 预判，降低大规模超发风险

### 4.2 通知去重的并发保护

告警邮件和通知使用 **Redis SETNX**（SET if Not eXists）实现原子去重，这是正确的设计：

- **告警邮件**：`billing:warning_email:{projectId}:{sourceType}:{YYYY}-{MM}`，TTL 到月底（[BillingLimitService.ts#L610](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L610-L610)）
- **超限邮件**：`billing:limit_email:{projectId}:{sourceType}:{YYYY}-{MM}`，同上
- **Ntfy 警告通知**：`ntfy:billing:warning:{projectId}:{sourceType}`，24 小时 TTL（[NtfyService.ts#L610](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/NtfyService.ts#L610-L610)）
- **Ntfy 超限通知**：`ntfy:billing:exceeded:{projectId}:{sourceType}`，24 小时 TTL

SETNX 保证了即使高并发下也只会发送一次通知。

---

## 五、前端呈现与告警边界

### 5.1 数据获取层（Hooks）

| Hook | 接口 | 刷新频率 | 适用场景 |
|---|---|---|---|
| `useBillingLimits` | `/users/@me/projects/{id}/billing-limits` | 30 秒（[useBillingLimits.ts#L19](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/lib/hooks/useBillingLimits.ts#L19-L19)） | 显示用量与限制配置 |
| `useBillingConsumption` | `/users/@me/projects/{id}/billing-consumption` | 60 秒（[useBillingConsumption.ts#L47](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/lib/hooks/useBillingConsumption.ts#L47-L47)） | 付费用户账单预览（Stripe Invoice） |

### 5.2 BillingLimits 组件呈现逻辑

[BillingLimits.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/web/src/components/BillingLimits.tsx) 的关键分支：

```
billingEnabled == false → 不渲染
├── tier == 'free'
│   └── 单卡展示 "Total Emails (All Categories)"，共享 1000 额度
│       └── 所有类别显示相同的 usage/limit/percentage（后端 getLimitsAndUsage 已统一处理）
└── tier == 'paid'
    └── 四卡独立展示 Workflows / Campaigns / Transactional / Inbound
        └── 支持编辑（PUT /billing-limits）
```

#### 视觉状态编码

| 状态 | 条件 | 颜色 | 图标 |
|---|---|---|---|
| 正常 | `usage < 80% limit` 或 `limit == null` | 中性灰 | Check ✓ |
| 警告 | `80% ≤ usage < 100%` | 琥珀色 | AlertTriangle ⚠ |
| 阻断 | `usage ≥ 100%` | 红色 | AlertCircle ✕ |

进度条 `Math.min(percentage, 100)` 夹紧到 100%，避免超限时视觉溢出。

### 5.3 告警阈值与通知通道

#### 阈值（WARNING_THRESHOLD = 0.8，见 [BillingLimitService.ts#L33](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L33-L33)）

- **80%**：警告级别，Ntfy DEFAULT 优先级通知 + 警告邮件（每月一封）
- **100%**：阻断级别，Ntfy MAX 优先级通知 + 超限邮件（每月一封）+ 返回 HTTP 429

#### 通知通道

1. **站内邮件**：通过 `sendPlatformEmail` 发送给项目所有成员（[MembershipService.getMembers](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L616-L616)）
2. **Ntfy.sh 推送**：[NtfyService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/NtfyService.ts)，可选配置
3. **服务端日志**：`signale.warn/error` 输出

### 5.4 限制变更时的通知缓存清理

当用户修改 billing limits 时（`PUT /billing-limits`），系统调用 `clearNotificationCacheForChangedLimits` 删除对应类别的 warning/limit 邮件缓存 Key，确保新阈值下仍能触发通知（[Users.ts#L367-L376](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/controllers/Users.ts#L367-L376)）。

---

## 六、核心数据结构

### 6.1 后端响应类型

```typescript
// 来自 @plunk/types 的 BillingLimitsResponse
interface BillingLimitsResponse {
  workflows:     CategoryUsage;
  campaigns:     CategoryUsage;
  transactional: CategoryUsage;
  inbound:       CategoryUsage;
  currency:      string | null;
}

interface CategoryUsage {
  limit:      number | null;  // null = unlimited
  usage:      number;
  percentage: number;         // 0-100
  isWarning:  boolean;        // ≥ 80%
  isBlocked:  boolean;        // ≥ 100%
}

interface LimitCheckResult {
  allowed:    boolean;
  warning:    boolean;
  usage:      number;
  limit:      number | null;
  percentage: number;
  message?:   string;
}
```

### 6.2 免费版的特殊处理

`getLimitsAndUsage` 对免费版会把四类的 usage 都设为**总和值**、limit 都设为 1000（[BillingLimitService.ts#L427-L451](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/BillingLimitService.ts#L427-L451)），前端只展示"Total Emails"一张卡。

---

## 七、失败处理与降级策略

### 7.1 Redis 故障
- **读失败**：fallback 到 DB 查询，记录 warn 日志
- **写失败（increment）**：静默忽略，下次 getUsage 从 DB 回填

### 7.2 DB 故障
- `getUsage/getTotalUsage` 返回 `0`，**不阻断邮件发送**
- `checkLimit` 最外层 try-catch 返回 `{allowed: true}`（fail-open）

### 7.3 Stripe 故障
- Meter event 通过 BullMQ 队列异步重试，最多 10 次
- 计费计量不影响限制判断（两条独立管道）

### 7.4 邮件发送失败
- SES 发送失败后 Email 状态变为 FAILED，但**不回滚 Meter 计数**（因为 Email 记录已存在）
- BullMQ retry 机制依赖 idempotency key 保证最多计费一次

---

## 八、关键 API 端点

| 方法 | 路径 | 功能 | 权限 |
|---|---|---|---|
| GET | `/users/@me/projects/{id}/billing-limits` | 获取用量和限制 | 项目成员 |
| PUT | `/users/@me/projects/{id}/billing-limits` | 更新限制（需有 subscription） | Admin/Owner |
| GET | `/users/@me/projects/{id}/billing-consumption` | 获取 Stripe 当前账单预览 | 项目成员 |
| GET | `/users/@me/projects/{id}/billing-invoices` | 获取历史发票列表 | 项目成员 |
| POST | `/users/@me/projects/{id}/checkout` | 创建 Stripe Checkout Session | Admin/Owner |
| POST | `/users/@me/projects/{id}/billing-portal` | 创建 Stripe Billing Portal | Admin/Owner |

---

## 九、测试覆盖要点

[BillingLimitService.test.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/60-plunk/apps/api/src/services/__tests__/BillingLimitService.test.ts) 覆盖了以下关键场景：

1. **收入保护**：三类邮件（Transactional/Campaign/Workflow）超限时阻断
2. **独立限制**：per-category 互不影响
3. **Unlimited**：`limit = null` 时无上限
4. **80% 警告阈值**
5. **缓存一致性**：incrementUsage 与 DB 写入配合
6. **Redis 故障降级**
7. **月度重置**：只统计当月 Email
8. **免费版共享额度**：1000 封跨类别共享
9. **并发竞态**：`Promise.allSettled` 批量发送下允许少量超发
10. **限制变更**：付费版自定义限制优先级高于免费版

---

## 十、系统边界总结

```
                    ┌─────────────────────────────────┐
                    │   Email Entry (Transactional/   │
                    │   Campaign/Workflow API/SMTP)    │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │  BillingLimitService.checkLimit  │
                    │  (同步检查，fail-open 策略)       │
                    └──────────────┬──────────────────┘
                      allowed=true  │  allowed=false → 429
                    ┌──────────────▼──────────────────┐
                    │  prisma.email.create (PENDING)   │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │ BillingLimitService.incrementUsage│
                    │    (redis.incr, 仅当 key 存在)   │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │   QueueService.queueEmail        │
                    │   (BullMQ emailQueue, 按优先级)  │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │  email-processor Worker          │
                    │  ├── SES 发送成功 → SENT         │
                    │  └── MeterService.recordEmailSent│
                    │       └── meterQueue             │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │  meter-processor Worker          │
                    │  └── stripe.billing.meterEvents  │
                    └─────────────────────────────────┘
```

**核心边界原则**：
1. **限制在入口判断**，发送链路不再重复检查
2. **计量在发送成功后**，避免发送失败也要付费
3. **限制与计费解耦**：限制基于 Email 表 + Redis 缓存，计费基于 Stripe Meter Events
4. **Fail-Open**：系统异常时宁可不限制也不阻断业务
5. **通知去重**：SETNX 保证告警风暴不会发生
6. **容忍竞态**：通过 Campaign 级预判 + 合理超发容忍空间换取性能
