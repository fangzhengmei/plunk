# 联系人导入与分段 - 代码结构理解

## 概述

本文档从代码结构切入，梳理联系人导入（CSV Import）与分段（Segments）的完整协作流程，将整个链路拆分为四个核心协作环节：**导入文件处理**、**联系人去重**、**字段映射**、**Segment 更新**。

---

## 一、导入文件处理环节

### 1.1 入口：HTTP 上传

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

## 二、联系人去重环节

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

## 三、字段映射环节

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

### 4.3 Segment 计数更新（定时轮询）

#### 触发方式

- **定时任务**: [app.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/app.ts) 第 473-482 行
- **频率**: 每 5 分钟一次（BullMQ repeatable job）
- **Job ID**: `segment-count-repeatable`（固定 ID 防重复）

#### 处理链路

```
segmentCountQueue (队列)
    │
    ▼
segment-count-processor.ts → processSegmentCountUpdate()
    │
    ├─ 有 projectId → 处理单个项目
    │
    └─ 无 projectId → 遍历所有活跃项目（每批 10 个，批间延迟 2s）
         │
         ▼
    processProjectSegments()
         │
         ├─ trackedSegments → SegmentService.computeMembership() 全量重算 + 事件
         │
         └─ nonTrackedSegments → SegmentService.refreshAllMemberCounts() 仅更新计数
```

位置: [segment-count-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/segment-count-processor.ts)

#### Worker 限制

- 并发数: 1
- 限流器: 每分钟最多 1 个 job

### 4.4 成员关系计算（computeMembership）

位置: [SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts) 第 449-631 行

流程：
1. 用游标分页（1000 条/批）获取所有符合条件的 contactId，存入 Set
2. 用游标分页获取当前活跃成员的 contactId，存入 Set
3. 计算差集：`toAdd`（新加入）、`toRemove`（需退出）
4. 批量新增/更新 SegmentMembership 记录（500 条/批）
5. 为每个变动的联系人生成 `segment.<slug>.entry` 或 `segment.<slug>.exit` 事件
6. 更新 segment.memberCount

> 注意: STATIC 类型的 segment 不做联系人扫描，直接从 membership 表计数。

### 4.5 条件编译（buildWhereClause）

位置: SegmentService.ts 第 885-893 行（入口），递归展开

支持的字段类型：
- **标准字段**: `email`, `subscribed`, `createdAt`, `updatedAt`
- **JSON 字段**: `data.xxx`（通过 PostgreSQL jsonb 路径查询）
- **事件字段**: `event.xxx`（通过 events 关联表 + some/none 查询）
- **邮件活动**: `email.opened`, `email.clicked` 等（通过 emails 关联表）
- **Segment 嵌套**: `segment.<segmentId>`（递归解析引用的 segment）

### 4.6 静态 Segment 手动管理

- **添加成员**: `POST /segments/:id/members` → `SegmentService.addContacts()`
  - 支持 `createMissing` 选项：不存在的邮箱自动创建联系人
  - 已有 membership 记录但已退出的 → 重新激活（exitedAt = null）
- **移除成员**: `DELETE /segments/:id/members` → `SegmentService.removeContacts()`
  - 软删除：设置 exitedAt = 当前时间

### 4.7 手动触发更新

- `POST /segments/:id/refresh` → 刷新 memberCount
- `POST /segments/:id/compute` → 全量重算 membership（仅 trackMembership 的 segment）

---

## 五、协作环节总览图

```
┌─────────────────────┐
│  1. 导入文件处理     │
│  ContactsController  │
│  → QueueService      │
│  → import-processor  │
└─────────┬───────────┘
          │ CSV 解析 + 逐行处理
          ▼
┌─────────────────────┐
│  2. 联系人去重       │
│  ContactService      │
│  .upsert()           │
│  .mergeContactData() │
└─────────┬───────────┘
          │ 写入/更新 contact 表
          ▼
┌─────────────────────┐
│  3. 字段映射         │
│  coerceCustomValue   │
│  email/subscribed    │
│  特殊处理 + data 合并 │
└─────────┬───────────┘
          │ 持久化到 JSON 列
          │
          │  ┌──────────────────────┐
          │  │  4. Segment 更新      │
          └─▶│  (异步/定时触发)      │
             │  segment-count-       │
             │  processor (每5分钟)  │
             │  → computeMembership  │
             │  → refreshMemberCount │
             └──────────────────────┘
```

---

## 六、关键文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 导入入口 | [controllers/Contacts.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/controllers/Contacts.ts) | HTTP 接口、文件上传、队列调度 |
| 导入 Worker | [jobs/import-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/import-processor.ts) | CSV 解析、批量处理、进度通知 |
| 联系人服务 | [services/ContactService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/ContactService.ts) | CRUD、upsert、数据合并、字段发现 |
| 队列服务 | [services/QueueService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/QueueService.ts) | 所有 BullMQ 队列管理 |
| Segment 服务 | [services/SegmentService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/services/SegmentService.ts) | Segment CRUD、条件编译、成员计算 |
| Segment Worker | [jobs/segment-count-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/segment-count-processor.ts) | 定时更新 segment 计数和成员关系 |
| Segment 控制器 | [controllers/Segments.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/controllers/Segments.ts) | Segment HTTP 接口 |
| 数据模型 | [packages/db/prisma/schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/packages/db/prisma/schema.prisma) | Contact / Segment / SegmentMembership 表结构 |
| Worker 入口 | [jobs/worker.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/apps/api/src/jobs/worker.ts) | 所有后台 Worker 统一启动 |
| 导入类型 | [packages/types/src/jobs/import.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/53-plunk/packages/types/src/jobs/import.ts) | ContactImportJobData 等类型定义 |
