# 联系人导入与分段 - 代码结构理解

## 概述

本文档从代码结构切入，梳理联系人导入（CSV Import）与分段（Segments）的完整协作流程。全文分为五个环节和三大隐性协作关系的深度分析。

---

## 一、导入文件处理

### 1.1 HTTP 上传入口

- **Controller**: [Contacts.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/controllers/Contacts.ts) `importCsv` 方法（第 306-334 行）
- **路由**: `POST /contacts/import`
- **中间件**: `requireAuth` + `multer` 内存存储

处理流程：
1. multer 接收 CSV 文件（最大 5MB，仅允许 text/csv 或 .csv 后缀）
2. 将文件 buffer 转换为 base64 字符串（`req.file.buffer.toString('base64')`）
3. 调用 `QueueService.queueImport()` 将任务入队
4. 返回 202 Accepted，附带 jobId

### 1.2 队列层

- **QueueService**: [QueueService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/QueueService.ts) `queueImport` 方法（第 308-320 行）
- **队列名**: `import`
- **Job 数据结构**: `ContactImportJobData`
  - `projectId`: 项目 ID
  - `csvData`: base64 编码的 CSV 内容
  - `filename`: 原始文件名
- **Job ID**: `import-${projectId}-${Date.now()}`

### 1.3 后台 Worker 处理

- **Worker**: [import-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/import-processor.ts) `createImportWorker`
- **并发数**: 2（最多同时处理 2 个导入任务）
- **批次大小**: 100 条/批

处理步骤：
1. 解码 base64 CSV 数据为 UTF-8 字符串
2. 使用 `csv-parse/sync` 解析，表头统一转小写
3. 校验：非空、必须包含 `email` 列
4. 逐批处理，每条记录依次执行：邮箱校验 → 字段提取 → 去重 upsert
5. 通过 `job.updateProgress()` 更新进度
6. 完成后通过 `NtfyService` 发送通知

### 1.4 状态查询

- **路由**: `GET /contacts/import/:jobId`
- **安全性**: 校验 job 所属 projectId 与请求者一致

---

## 二、联系人去重

### 2.1 去重策略：按 email + projectId 唯一约束

- **数据库层面**: [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/packages/db/prisma/schema.prisma) 第 152 行 `@@unique([projectId, email])`
- **服务层**: [ContactService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/ContactService.ts) `upsert` 方法（第 261-327 行）

### 2.2 Upsert 流程

```
findFirst({ projectId, email })
       │
       ├─ 存在 → update（合并 data + 可选更新 subscribed）
       │        └─ 订阅状态变化时，追踪 contact.subscribed / contact.unsubscribed 事件
       │
       └─ 不存在 → create（新联系人默认 subscribed=true，除非显式指定）
```

关键细节：
- **导入时的去重判断**: import-processor 第 133-134 行先调用 `findByEmail` 标记 isUpdate，再调用 `upsert`
- **订阅状态保留**: 对已有联系人，若 CSV 中无 `subscribed` 列，则不改变原有订阅状态（`subscribed` 参数为 `undefined` 时跳过更新）
- **数据合并**: 使用 `mergeContactData` 方法进行增量合并，而非全量覆盖

### 2.3 数据合并规则（mergeContactData）

位置: [ContactService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/ContactService.ts) 第 145-177 行

合并规则：
- `null` 值 → 删除该 key
- 空字符串 → 忽略（不更新）
- 保留字段（过滤）: `plunk_id`, `plunk_email`, `id`, `email`, `unsubscribeUrl`, `subscribeUrl`, `manageUrl`
- `{value, persistent: false}` 格式的非持久化字段 → 跳过
- `locale` 字段必须是字符串

### 2.4 批量去重查询

- **lookup 方法**: [ContactService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/ContactService.ts) 第 82-94 行
- **路由**: `POST /contacts/lookup`
- 单条查询最多 500 个邮箱，返回 `{found[], notFound[]}`

---

## 三、字段映射

### 3.1 CSV 表头映射

位置: [import-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/import-processor.ts) 第 57-62 行

```
CSV 表头 → 全部 toLowerCase() → 作为 record 的 key
```

### 3.2 特殊字段处理

| CSV 列名 | 映射目标 | 处理逻辑 |
|---------|---------|---------|
| `email` | `contact.email` | 必填，做格式校验 |
| `subscribed` | `contact.subscribed` | 解析布尔值: `true/1/yes` → true；`false/0/no` → false；空值/缺失 → undefined（不更新） |
| 其他列 | `contact.data.*` | 作为自定义字段存储到 JSON 列 |

### 3.3 自定义字段类型推断

位置: `coerceCustomValue` 函数（import-processor.ts 第 242-248 行）

类型推断顺序：
1. **布尔型**: `true/yes` → `true`；`false/no` → `false`（不区分大小写，去首尾空格）
2. **数字型**: 匹配正则 `/^-?(0|[1-9]\d*)(\.\d+)?$/` → `Number(value)`
   - 合法: `0`, `42`, `-42`, `3.14`
   - 拒绝: `007`, `+42`, `1.2.3`, `1e5`（保留为字符串，避免 ID 类字段被误转）
3. **字符串**: 以上都不匹配则原样返回

> 注意: `1` 和 `0` 在 `coerceCustomValue` 中会被转为数字，但 `subscribed` 字段有独立的解析逻辑（import-processor.ts 第 118-122 行），其中 `'1'` 被视为 true。

### 3.4 字段发现与类型推断（查询侧）

- **getAvailableFields**: [ContactService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/ContactService.ts) 第 450-543 行
- 使用 PostgreSQL 原生 SQL + `jsonb_object_keys` 提取所有自定义字段
- 通过 `jsonb_typeof` + 样本值推断类型（string/number/boolean/date）
- 计算字段覆盖率（有多少联系人拥有该字段）

---

## 四、Segment 更新路径

### 4.1 Segment 数据模型

- **Segment 表**: [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/packages/db/prisma/schema.prisma) 第 196-251 行
  - `type`: `DYNAMIC`（动态筛选）或 `STATIC`（静态手动管理）
  - `condition`: JSON 格式的筛选条件（仅 DYNAMIC）
  - `trackMembership`: 是否追踪成员变化（启用后生成 entry/exit 事件）
  - `memberCount`: 缓存的成员数量

- **SegmentMembership 表**: 第 253-274 行
  - `enteredAt`: 进入时间
  - `exitedAt`: 退出时间（null 表示当前在段内）
  - 复合主键 `[contactId, segmentId]`

### 4.2 动态 Segment 条件结构

```typescript
FilterCondition {
  logic: "AND" | "OR"
  groups: FilterGroup[]
}

FilterGroup {
  filters: SegmentFilter[]     // 组内过滤器
  conditions?: FilterCondition // 嵌套条件（递归）
}

SegmentFilter {
  field: string     // 如 "email", "subscribed", "data.plan", "event.purchase", "segment.xxx"
  operator: string  // equals, contains, greaterThan, within, triggered, memberOfSegment, ...
  value?: unknown
  unit?: "days" | "hours" | "minutes"
}
```

### 4.3 条件编译（buildWhereClause）

位置: [SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts) 第 885-893 行（入口），递归展开

支持的字段类型：
- **标准字段**: `email`, `subscribed`, `createdAt`, `updatedAt`
- **JSON 字段**: `data.xxx`（通过 PostgreSQL jsonb 路径查询）
- **事件字段**: `event.xxx`（通过 events 关联表 + some/none 查询）
- **邮件活动**: `email.opened`, `email.clicked` 等（通过 emails 关联表）
- **Segment 嵌套**: `segment.<segmentId>`（递归解析引用的 segment）

### 4.4 静态 Segment 手动管理

- **添加成员**: `POST /segments/:id/members` → `SegmentService.addContacts()`
  - 支持 `createMissing` 选项：不存在的邮箱自动创建联系人
  - 已有 membership 记录但已退出的 → 重新激活（exitedAt = null）
- **移除成员**: `DELETE /segments/:id/members` → `SegmentService.removeContacts()`
  - 软删除：设置 exitedAt = 当前时间

---

## 五、Segment 刷新入口全景：四个入口，三个真正落地

Segment 重算有**四个入口**，其中一个有实现没调用，真正落到生产的是三个。每个入口负责不同的场景。

| 入口 | 触发方式 | 调用路径 | 影响范围 | 落到生产？ |
|-----|---------|---------|---------|-----------|
| 定时轮询 | 每 5 分钟自动 | `app.ts L473` 直接 `segmentCountQueue.add()` | 全平台所有项目所有 segment | ✅ |
| 手动 refresh | `POST /segments/:id/refresh` | `Segments.refresh` → `SegmentService.refreshMemberCount()` | 单个 segment | ✅ |
| 手动 compute | `POST /segments/:id/compute` | `Segments.compute` → `SegmentService.computeMembership()` | 单个 segment | ✅ |
| queueSegmentCountUpdate | 预留 API | `QueueService.queueSegmentCountUpdate()` → `segmentCountQueue.add()` | 全项目或单个项目 | ❌ 业务零调用 |

### 5.1 入口一：定时轮询（生产主链路）

**定位：后台兜底刷新，保证所有数据最终一致。**

#### 触发方式

- 位置: [app.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/app.ts) 第 473-482 行
- 频率: 每 5 分钟一次（BullMQ repeatable job）
- **调用方式**: 直接 `segmentCountQueue.add()`，**没有走 `QueueService.queueSegmentCountUpdate()` 封装**
- Job data: `{}`（空对象，无 projectId）
- Job ID: `segment-count-repeatable`（固定 ID 防重复）

#### 处理链路

```
segmentCountQueue (队列)
    │
    ▼
segment-count-processor.ts → processSegmentCountUpdate()
    │  job.data = {}  （空，无 projectId）
    │
    └─ 遍历所有活跃项目（每批 10 个，批间延迟 2s）
         │
         ▼
    processProjectSegments(projectId)
         │
         ├─ trackMembership=true  → SegmentService.computeMembership()  重链路
         │    （全量重算 + 写 membership + entry/exit 事件）
         │
         └─ trackMembership=false → SegmentService.refreshAllMemberCounts()  轻链路
              （仅 COUNT() + 更新 memberCount）
```

位置: [segment-count-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/segment-count-processor.ts)

#### Worker 限制

- 并发数: 1（串行处理避免 DB 过载）
- 限流器: 每分钟最多 1 个 job

#### 覆盖场景

1. 导入新联系人后的最终一致（导入链路不主动刷新，靠定时兜底）
2. 联系人属性变化后的成员变更
3. segment 条件被修改后未手动 compute 时的增量追赶
4. 事件触发导致的成员进出（如「近 7 天未登录」条件随时间变化）

---

### 5.2 入口二：POST /segments/:id/refresh（手动轻量刷新）

**定位：用户主动触发，只想立刻看最新人数。**

#### 调用链路

- 路由: [Segments.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/controllers/Segments.ts) 第 219-236 行
- 服务层: `SegmentService.refreshMemberCount(projectId, segmentId)`
- 位置: [SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts) 第 265-284 行

#### 行为

```
1. 读 segment 记录（type + condition）
2. STATIC → COUNT SegmentMembership 表
   DYNAMIC → buildWhereClause + COUNT contact 表
3. UPDATE segment SET memberCount = 结果
4. 返回 memberCount 数字
```

#### 关键特性

- **单 segment**：只刷新指定 ID 的 segment
- **轻量**：只做 COUNT 查询 + UPDATE，不扫描全表，不写 membership，不触发事件
- **不区分 trackMembership**：对所有 segment 只做计数，哪怕开了 trackMembership 也不重算成员关系

#### 覆盖场景

- 刚导入完一批联系人，用户等不及 5 分钟定时，主动点「刷新」按钮看人数
- segment 列表页 memberCount 显示的是缓存值，用户想确认当前真实人数

---

### 5.3 入口三：POST /segments/:id/compute（手动全量重算）

**定位：用户主动触发，要立刻重算所有成员并触发事件。**

#### 调用链路

- 路由: [Segments.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/controllers/Segments.ts) 第 199-216 行
- 服务层: `SegmentService.computeMembership(projectId, segmentId)`
- 位置: [SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts) 第 449-631 行

#### 完整流程（与定时任务中的重链路完全相同）

1. 用游标分页（1000 条/批）获取所有符合条件的 contactId，存入 Set
2. 用游标分页获取当前活跃成员的 contactId，存入 Set
3. 计算差集：`toAdd`（新加入）、`toRemove`（需退出）
4. 批量新增/更新 SegmentMembership 记录（500 条/批）
5. 为每个变动的联系人生成 `segment.<slug>.entry` 或 `segment.<slug>.exit` 事件
6. 更新 segment.memberCount

> 注意: STATIC 类型的 segment 不做联系人扫描，直接从 membership 表计数。

#### 关键特性

- **单 segment**：只重算指定 ID 的 segment
- **全量重算**：不管有没有变化，都重新扫描所有联系人
- **写 membership**：新增/更新 SegmentMembership 记录
- **触发生成事件**：entry/exit 事件会驱动 Workflow 引擎
- **同步等待**：API 同步调用，大 segment 可能需要几秒甚至几十秒才返回

#### 覆盖场景

- 刚改完 segment 的筛选条件，想立刻让新条件生效并触发相关 workflow
- 修复了 segment 条件 bug 后，想立刻重算历史成员
- 紧急营销活动前，确认 segment 成员是最新的并触发必要的自动化流程

---

### 5.4 入口四：queueSegmentCountUpdate（预留 API，零调用）

**定位：有完整实现，但业务代码零调用，属于预留的触发接口。**

#### 代码位置

定义: [QueueService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/QueueService.ts) 第 425-433 行

```typescript
public static async queueSegmentCountUpdate(projectId?: string): Promise<Job<SegmentCountJobData>> {
  return segmentCountQueue.add(
    'update-segment-counts',
    { projectId },   // ← 支持传 projectId 只处理单个项目
    {
      jobId: projectId ? `segment-count-${projectId}-${Date.now()}` : `segment-count-all-${Date.now()}`,
    },
  );
}
```

#### 核实结果：全代码库零业务调用

通过全代码库 grep `queueSegmentCountUpdate` 确认：
- **唯一出现位置**：[QueueService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/QueueService.ts#L425) 的定义本身
- **定时任务**没有走这个封装——[app.ts L473](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/app.ts#L473) 直接 `segmentCountQueue.add()`
- **导入完成后**（import-processor）没有调用
- **ContactService.upsert()** 没有调用
- **测试文件**中也没有引用

#### 推测设计意图

这个方法原本可能计划作为「按需触发」的标准入口，例如导入完成后调用 `queueSegmentCountUpdate(projectId)` 只刷新该项目的 segment。但最终没有接入，推测原因：
1. 导入后立即重算对 DB 压力太大（10 万条导入后立刻全表扫描）
2. 5 分钟窗口的不一致对大多数场景可接受
3. 用户真的急可以手动点 refresh/compute 按钮
4. 即使需要按项目触发，app.ts 里已经直接操作 `segmentCountQueue`，绕过了这个封装

**这个方法目前是一个死代码入口。**

---

### 5.5 三大生产入口的场景分工图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     什么场景该走哪个入口？                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  「我不想管，数据最终一致就行」                                          │
│      └─→ 定时轮询 (每5分钟)  ← 默认行为，无需操作                       │
│                                                                         │
│  「导入完了，人数看着不对」                                              │
│      └─→ POST /segments/:id/refresh  ← 只 COUNT，秒级返回              │
│                                                                         │
│  「改了条件，要让 Workflow 立刻生效」                                    │
│      └─→ POST /segments/:id/compute  ← 全量重算+事件，可能耗时较长      │
│                                                                         │
│  「我想在代码里按项目触发重算」                                          │
│      └─→ ❌ queueSegmentCountUpdate  ← 死代码，未接入                   │
│          临时方案：直接 segmentCountQueue.add() 或手动 compute            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 补充：隐式即时更新场景

除了以上三个显式入口，segment 在以下场景会即时更新 memberCount（不需要等 5 分钟定时）：

| 场景 | 代码位置 | 行为 |
|-----|---------|------|
| 创建 segment 时 | [SegmentService.create](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts#L118-L166) 第 140 行 | 同步 `COUNT()` 计算初始 memberCount |
| 修改 DYNAMIC segment 条件时 | [SegmentService.update](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts#L171-L213) 第 202 行 | 同步 `COUNT()` 更新 memberCount |
| STATIC segment 添加成员后 | [SegmentService.addContacts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts#L331-L404) 第 399 行 | `COUNT(membership)` 更新 memberCount |
| STATIC segment 移除成员后 | [SegmentService.removeContacts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts#L409-L443) 第 438 行 | `COUNT(membership)` 更新 memberCount |

> 注意：创建/修改 segment 时只做 `COUNT()` 更新 memberCount，**不做** `computeMembership()` 全量重算。也就是说新 segment 即使开了 trackMembership，也需要等到下次定时轮询或手动 compute 时才会生成 membership 记录和 entry/exit 事件。

---

## 六、三大隐性协作关系深度分析

### 6.1 协作关系 A：导入 → Segment 的时序断层

**核心结论：导入完成后不会立即刷新 Segment，完全依赖定时轮询（最坏延迟约 5 分钟）。**

#### 代码证据

1. **import-processor 导入完成后无刷新钩子**
   - 位置: [import-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/import-processor.ts) 第 166-177 行
   - 导入完成仅执行：`NtfyService.notifyContactImportCompleted()` 发送通知 + `return result`
   - **没有任何调用** `QueueService.queueSegmentCountUpdate()` 或 `SegmentService.refreshMemberCount()` 的逻辑

2. **ContactService.upsert() 也不触发 Segment 刷新**
   - 位置: [ContactService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/ContactService.ts) 第 261-327 行
   - upsert 方法只做：写入 contact 表 + 订阅状态变化事件追踪
   - **无任何 Segment 相关调用**

3. **queueSegmentCountUpdate 是死代码**（见 5.4 节）

#### 实际刷新时序

```
T=0min  用户上传 CSV → importQueue → 导入 worker 开始处理
T=1min  导入完成 → 仅通知用户，Segment memberCount 仍是旧值
        ├─ 用户不急 → 等到 T=5min 定时轮询自动刷新
        └─ 用户很急 → 点「刷新」按钮 → POST /segments/:id/refresh → 立刻 COUNT
                      或 点「重算」按钮 → POST /segments/:id/compute → 全量重算 + 事件
T=5min  定时 repeatable job 触发 → segmentCountQueue 入队
T=5min+ segment-count-processor 执行
        → STATIC/DYNAMIC nonTracked: COUNT() 更新 memberCount
        → DYNAMIC tracked: 全量 computeMembership()
T=5min+ Segment 数据与导入结果一致（视项目大小和 segment 数量有延迟）
```

#### 设计权衡

| 方案 | 优点 | 缺点 |
|-----|-----|-----|
| **当前方案：定时兜底 + 手动按需** | 1. 导入性能不受影响（大导入不会被 segment 重算拖慢）<br>2. 多次导入在 5 分钟窗口内被合并为一次重算<br>3. 代码解耦<br>4. 紧急场景可手动触发 | 不主动干预时最坏 5 分钟不一致 |
| 导入后自动调用 queueSegmentCountUpdate | 实时性好 | 1. 大批量导入后立即 segment 重算对 DB 造成突发压力<br>2. 连续多次导入触发多次重复重算<br>3. 导入和 segment 强耦合 |

---

### 6.2 协作关系 B：trackMembership 分叉链路

**核心原因：成本差异巨大 + 不同场景下对「精确历史」和「事件触发」的需求不同。**

#### 两条链路的成本对比

位置: [segment-count-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/segment-count-processor.ts) 第 20-72 行

```
processProjectSegments(projectId)
    │
    ├─ trackMembership=true  →  computeMembership()   （重链路）
    │
    └─ trackMembership=false →  refreshAllMemberCounts()  （轻链路）
```

| 维度 | `refreshAllMemberCounts()` 轻链路 | `computeMembership()` 重链路 |
|-----|----------------------------------|------------------------------|
| **位置** | [SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts#L290-L326) | [SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts#L449-L631) |
| **DB 读取量** | 每个 segment 1 次 `COUNT()` 查询（用 GIN 索引走 jsonb 条件） | 2 次游标全表扫描：<br>① 所有匹配条件的 contactId（1000 条/批）<br>② 所有活跃 membership contactId（1000 条/批） |
| **内存占用** | 几乎为 0（COUNT 结果单个数字） | 两个 Set，大小等于「匹配人数」和「当前成员数」（最坏 O(N)） |
| **DB 写入量** | 每个 segment 1 次 `UPDATE segment SET memberCount` | 每有一个变动：<br>① INSERT/UPDATE SegmentMembership（500/批）<br>② INSERT Event（**逐条**，每个进出都有 entry/exit 事件） |
| **生成产物** | 仅 `segment.memberCount`（一个数字） | ① `SegmentMembership` 记录（精确历史，含 enteredAt/exitedAt）<br>② `segment.<name>.entry` 和 `segment.<name>.exit` 事件 |
| **典型耗时** | 每项目 ~几百 ms（10 个 segment 以内） | 数万联系人的项目可达数秒至数十秒 |

#### 为什么需要重链路？三个关键能力

**1. Workflow 触发能力**

Segment entry/exit 事件是 Workflow 系统的重要触发源：
- 「当联系人进入『VIP客户』segment 时，自动发欢迎邮件」→ 监听 `segment.vip-customers.entry`
- 「当联系人退出『活跃用户』segment 时，启动挽回流程」→ 监听 `segment.active-users.exit`

如果只更新 memberCount（一个数字），Workflow 引擎无法知道具体是哪个联系人进出了 segment。

**2. 精确的成员关系历史审计**

`SegmentMembership` 表记录了 `enteredAt` 和 `exitedAt`，支持：
- 「这个用户什么时候成为 VIP 的？什么时候降级的？」
- 「过去 30 天有多少人进出了某 segment？」（营销漏斗分析）
- 静态 segment 的成员管理（加入时间、软删除重激活）

**3. 嵌套 segment 引用的性能优化**

当 segment A 的条件引用 segment B 时（`field: "segment.<B的id>", operator: "memberOfSegment"`），如果 B 开启了 trackMembership，查询可以直接走 `SegmentMembership` 表的索引（`WHERE exitedAt IS NULL`），而不需要递归展开 B 的条件再做一次全量扫描。

位置: [SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts#L636-L685) `buildFilterCondition` 中对 `segment.` 前缀的处理：
```
引用的 segment 开启了 trackMembership 或是 STATIC
    → 直接查 SegmentMembership（一次简单索引查询）

否则（未开启 trackMembership 的 DYNAMIC segment）
    → 递归解析该 segment 的 condition
    → 构造嵌套的 WHERE 子句（可能触发多层递归扫描）
```

#### 为什么需要轻链路？覆盖 80% 的场景

大多数用户使用 segment 只是为了「给 campaign 选人群」和「看列表里的人数」：
- 发 campaign 时动态执行条件查询（不需要预先存 membership）
- 列表页显示一个大致准确的 memberCount

对于这些场景，开启 trackMembership 会造成不必要的 DB 负担：
- 项目有数十个 segment，但只有 2-3 个需要做 workflow 触发
- 联系人数在 10 万+ 级别，全量扫描代价显著
- 每 5 分钟一次的重链路对 DB 造成持续压力

> **设计总结**：这是典型的「按需付费」架构。默认关闭 trackMembership，成本最低；需要高级功能（事件触发、历史审计、嵌套优化）的用户主动开启，用额外计算资源换取更强能力。

---

### 6.3 协作关系 C：自定义字段 → Segment 条件判断

**完整链路：CSV 文本 → PostgreSQL jsonb → Prisma 路径查询 → Segment 结果集**

#### 阶段 1：字段落地（导入阶段）

```
CSV 行: Email,Plan,SignupDate,FirstName
        john@example.com,Pro,2025-01-15,John
           │
           ▼  [import-processor.ts L57-62]
        headers.toLowerCase() → ['email','plan','signupdate','firstname']
           │
           ▼  [import-processor.ts L124-L130]
        解构: email 和 subscribed 单独处理
        剩余字段 → customData = { plan: "Pro", signupdate: "2025-01-15", firstname: "John" }
           │
           ▼  [import-processor.ts L127-L130]
        Object.entries(customData).map(([k,v]) => [k, coerceCustomValue(v)])
           │  "Pro"→"Pro"(str), "2025-01-15"→"2025-01-15"(str), "John"→"John"(str)
           ▼
        ContactService.upsert(projectId, email, { plan:"Pro", signupdate:"2025-01-15", firstname:"John" })
           │
           ▼  [ContactService.ts L261-L327] upsert() → mergeContactData()
        合并到 contact.data (JSONB 列)
           │
           ▼  [PostgreSQL]
        contacts 表一行:
          id: "uuid-xxx"
          projectId: "uuid-yyy"
          email: "john@example.com"
          data: { "plan": "Pro", "signupdate": "2025-01-15", "firstname": "John" }
                              ↑
          GIN 索引: @@index([data(ops: JsonbOps)], type: Gin)  [schema.prisma L155]
```

#### 阶段 2：字段被发现（前端构建 Segment 条件时）

用户在 Segment 构建器界面添加过滤器时，需要可选字段列表：

```
GET /contacts/fields
     │
     ▼  [ContactService.ts L450-L543] getAvailableFields()
  PostgreSQL 原生 SQL:
    1. jsonb_object_keys(data) → 提取所有 data 列的 key
    2. 对每个 key: jsonb_typeof(data->key) + 采样值 → 推断类型
    3. COUNT(*) → 计算覆盖率
     │
     ▼
  返回: [
    { field: "email", type: "string", coverage: 100 },
    { field: "subscribed", type: "boolean", coverage: 100 },
    { field: "data.firstname", type: "string", coverage: 87 },   ← 导入的字段
    { field: "data.plan", type: "string", coverage: 87 },        ← 导入的字段
    { field: "data.signupdate", type: "date", coverage: 63 },    ← 导入的字段
  ]
     │
     ▼
  前端展示下拉菜单，用户选 "data.plan" + "equals" + "Pro"
```

#### 阶段 3：条件编译（Segment 查询时）

假设用户构建的 segment 条件是：「plan 等于 Pro 且已订阅」

```typescript
// 用户保存到 segment.condition 的 JSON:
{
  logic: "AND",
  groups: [{
    filters: [
      { field: "data.plan", operator: "equals", value: "Pro" },
      { field: "subscribed", operator: "equals", value: true }
    ]
  }]
}
```

当需要查询 segment 成员时（`SegmentService.getContacts()` 或 `refreshAllMemberCounts()` 中的 COUNT），经过以下编译链：

```
FilterCondition (上面的 JSON)
     │
     ▼  [SegmentService.ts L885-L893] buildWhereClause(projectId, condition)
  { projectId, ... 展开的条件 }
     │
     ▼  [SegmentService.ts L763-L776] buildConditionClause()  处理 AND/OR 逻辑
  { AND: [ groupClause1, groupClause2, ... ] }
     │
     ▼  [SegmentService.ts L898-L927] buildGroupClause()  组内 AND
  { AND: [ filterClause1, filterClause2, ... ] }
     │
     ├─ filter: { field: "subscribed", operator: "equals", value: true }
     │     ▼  [SegmentService.ts L706-L717] buildFilterCondition() → switch case "subscribed"
     │     ▼  [SegmentService.ts L1043-L1056] buildBooleanFieldCondition()
     │  生成 Prisma where: { subscribed: true }
     │
     └─ filter: { field: "data.plan", operator: "equals", value: "Pro" }
           ▼  [SegmentService.ts L699-L703] buildFilterCondition() → field.startsWith("data.")
           ▼  截取 jsonPath = "plan"（substring(5)）
           ▼  [SegmentService.ts L932-L1020] buildJsonFieldCondition(jsonPath, "equals", "Pro")
              → jsonPath.split('.') → ['plan']
              → operator "equals" → 检查是否日期字符串（这里不是）
              → 生成 Prisma where:
                   { data: { path: ['plan'], equals: "Pro" } }
                              ↑
          Prisma 将此编译为 PostgreSQL: data @> '{"plan": "Pro"}' 或 data->>'plan' = 'Pro'
          命中 schema.prisma 第 155 行的 GIN 索引
     │
     ▼  合并最终 Prisma WhereInput
  {
    projectId: "uuid-yyy",
    AND: [
      { subscribed: true },
      { data: { path: ['plan'], equals: "Pro" } }
    ]
  }
     │
     ▼  prisma.contact.count({ where })  或  prisma.contact.findMany({ where })
  返回匹配的联系人或计数
```

#### 关键技术点：Prisma + PostgreSQL jsonb 路径查询

位置: [SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts#L932-L1020) `buildJsonFieldCondition`

所有 data.xxx 字段的条件最终都通过 Prisma 的 `data: { path: [...], operator: value }` 语法编译为 PostgreSQL jsonb 操作，利用 [schema.prisma L155](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/packages/db/prisma/schema.prisma#L155) 的 GIN 索引。

支持的 data 字段操作符：`equals` / `notEquals` / `contains` / `notContains` / `greaterThan` / `lessThan` / `greaterThanOrEqual` / `lessThanOrEqual` / `exists` / `notExists` / `within`（X 时间内）/ `olderThan`（X 时间前）。

> **嵌套路径支持**: `buildJsonFieldCondition` 第 938 行 `const path = jsonPath.split('.')` 支持多层嵌套。例如 CSV 列 `address.city` 会被导入为 `data.address.city`，条件编译时 `jsonPath = "address.city"` split 为 `['address', 'city']`，正确命中嵌套的 JSON 字段。

---

## 七、协作环节总览图

```
┌──────────────────────────────────────────────────────────────────────┐
│  1. 导入文件处理                                                     │
│  ContactsController → QueueService.queueImport → import-processor     │
└────────────────────────┬─────────────────────────────────────────────┘
                         │ CSV 解析 / 邮箱校验 / 类型推断
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  2. 联系人去重                                                       │
│  findByEmail 判重 → upsert() → mergeContactData(增量合并)            │
└────────────────────────┬─────────────────────────────────────────────┘
                         │ 写入 contact 表 (data JSONB 列 + GIN 索引)
                         │
                         │ ┌────────────────────────────────────────────┐
                         │ │  协作关系 A: 导入→Segment 时序断层         │
                         │ │  ⚠ 导入后不立即刷新，等定时轮询(5min)      │
                         │ │  用户急 → 手动 refresh/compute             │
                         │ │  queueSegmentCountUpdate 是死代码          │
                         │ └────────────────────────────────────────────┘
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  3. 字段映射                                                         │
│  email/subscribed 特殊处理                                           │
│  其他列 → coerceCustomValue(布尔/数字/字符串推断) → contact.data.*   │
└────────────────────────┬─────────────────────────────────────────────┘
                         │
                         │ ┌────────────────────────────────────────────┐
                         │ │  协作关系 C: 自定义字段→条件判断           │
                         │ │  ① 落地: data JSONB 列 + GIN 索引         │
                         │ │  ② 发现: getAvailableFields → 前端可选列表│
                         │ │  ③ 编译: buildJsonFieldCondition          │
                         │ │    data.xxx → Prisma path 查询 → SQL GIN  │
                         │ └────────────────────────────────────────────┘
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  4. Segment 更新（四大入口）                                         │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │ 定时轮询     │  │ 手动refresh  │  │ 手动compute  │  │ 预留API  │ │
│  │ (每5min)     │  │ (COUNT)      │  │ (全量重算)   │  │ (死代码) │ │
│  │ 全平台       │  │ 单segment    │  │ 单segment    │  │          │ │
│  │ ✅ 生产主链路│  │ ✅ 看人数    │  │ ✅ 触发事件  │  │ ❌ 零调用│ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────┘ │
│         │                                                            │
│         │ ┌────────────────────────────────────────────────────────┐ │
│         │ │  协作关系 B: trackMembership 分叉                      │ │
│         │ │  ├─ true  → computeMembership() 重链路                │ │
│         │ │  │    全扫描+差集+写membership+entry/exit事件          │ │
│         │ │  └─ false → refreshAllMemberCounts() 轻链路           │ │
│         │ │       仅 COUNT() + 更新 memberCount                    │ │
│         │ └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 导入入口 | [controllers/Contacts.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/controllers/Contacts.ts) | HTTP 接口、文件上传、队列调度 |
| 导入 Worker | [jobs/import-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/import-processor.ts) | CSV 解析、批量处理、进度通知 |
| 联系人服务 | [services/ContactService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/ContactService.ts) | CRUD、upsert、数据合并、字段发现 |
| 队列服务 | [services/QueueService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/QueueService.ts) | 所有 BullMQ 队列管理（含 queueSegmentCountUpdate 死代码） |
| Segment 服务 | [services/SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts) | Segment CRUD、条件编译、成员计算 |
| Segment Worker | [jobs/segment-count-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/segment-count-processor.ts) | 定时更新 segment 计数和成员关系 |
| Segment 控制器 | [controllers/Segments.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/controllers/Segments.ts) | Segment HTTP 接口（含 refresh/compute） |
| 应用启动 | [app.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/app.ts) | 定时任务注册（segment-count repeatable job） |
| 数据模型 | [packages/db/prisma/schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/packages/db/prisma/schema.prisma) | Contact / Segment / SegmentMembership 表结构 |
| Worker 入口 | [jobs/worker.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/worker.ts) | 所有后台 Worker 统一启动 |
| 导入类型 | [packages/types/src/jobs/import.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/packages/types/src/jobs/import.ts) | ContactImportJobData 等类型定义 |
