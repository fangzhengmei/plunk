# Plunk — Campaign 创建与联系人选择：代码理解

## 一、业务对象还原

### 1.1 核心数据模型（Prisma Schema）

| 模型 | 文件位置 | 核心字段 | 说明 |
|------|----------|----------|------|
| **Campaign** | [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/db/prisma/schema.prisma#L280-L332) | `status`, `type`, `audienceType`, `audienceCondition`, `segmentId`, `totalRecipients`, 统计字段 | 一次性广播邮件 |
| **Contact** | [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/db/prisma/schema.prisma#L129-L157) | `email`, `data`(Json), `subscribed` | 联系人，自定义字段存在 data JSON 列 |
| **Segment** | [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/db/prisma/schema.prisma#L196-L251) | `type`(DYNAMIC/STATIC), `condition`(Json), `trackMembership`, `memberCount` | 受众分组 |
| **SegmentMembership** | [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/db/prisma/schema.prisma#L253-L274) | `contactId`, `segmentId`, `enteredAt`, `exitedAt` | 静态分组成员关系 |
| **Email** | [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/db/prisma/schema.prisma#L517-L594) | `campaignId`, `status`, 投递/打开/点击/退回时间戳 | 邮件追踪记录 |
| **Domain** | [schema.prisma](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/db/prisma/schema.prisma#L101-L123) | `domain`, `verified` | 发件域名验证 |

### 1.2 关键枚举

| 枚举 | 值 | 用途 |
|------|----|------|
| `CampaignStatus` | `DRAFT → SCHEDULED → SENDING → SENT / CANCELLED` | 活动生命周期 |
| `CampaignAudienceType` | `ALL / FILTERED / SEGMENT` | 受众选择模式 |
| `TemplateType` | `MARKETING / TRANSACTIONAL / HEADLESS` | 邮件类型，影响订阅检查与页脚 |
| `SegmentType` | `DYNAMIC / STATIC` | 分组求值方式 |
| `Role` | `OWNER / ADMIN / MEMBER` | 项目成员角色 |

### 1.3 类型定义

| 类型 | 文件位置 | 说明 |
|------|----------|------|
| `CreateCampaignData` | [campaign.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/types/src/api/campaign.ts#L11-L23) | 创建活动入参 |
| `UpdateCampaignData` | [campaign.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/types/src/api/campaign.ts#L28-L40) | 更新活动入参 |
| `FilterCondition` | [segments/index.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/types/src/segments/index.ts#L46-L49) | `logic + groups[]`，递归嵌套 |
| `FilterGroup` | [segments/index.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/types/src/segments/index.ts#L41-L44) | `filters[] + conditions?`，可嵌套 |
| `SegmentFilter` | [segments/index.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/packages/types/src/segments/index.ts#L34-L39) | `field + operator + value? + unit?` |

---

## 二、用户操作还原

### 2.1 Campaign 生命周期操作

| 用户操作 | 前端入口 | API 端点 | 服务方法 | 约束 |
|----------|----------|----------|----------|------|
| 创建活动 | [create.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/create.tsx) | `POST /campaigns` | [CampaignService.create](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L26-L87) | 需验证域名、段存在性；初始 status=DRAFT |
| 编辑草稿 | [[id].tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx) | `PUT /campaigns/:id` | [CampaignService.update](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L92-L159) | 仅 DRAFT/SCHEDULED 可编辑 |
| 立即发送 | [id].tsx 发送对话框](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx#L147-L155) | `POST /campaigns/:id/send` | [CampaignService.send](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L305-L378) | 校验计费限额、收件人数>0 |
| 定时发送 | [id].tsx 调度对话框](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx#L157-L188) | `POST /campaigns/:id/send {scheduledFor}` | 同上 | 时间须在未来 |
| 发送测试 | [id].tsx 测试对话框](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx#L190-L211) | `POST /campaigns/:id/test` | [CampaignService.sendTest](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L678-L727) | 仅项目成员可接收 |
| 取消活动 | [id].tsx 取消按钮](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx#L127-L135) | `POST /campaigns/:id/cancel` | [CampaignService.cancel](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L592-L622) | 仅 SCHEDULED/SENDING |
| 删除草稿 | [id].tsx 删除按钮](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx#L137-L145) | `DELETE /campaigns/:id` | [CampaignService.delete](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L236-L261) | 仅 DRAFT |
| 复制活动 | API only | `POST /campaigns/:id/duplicate` | [CampaignService.duplicate](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L266-L300) | 复制为 DRAFT，重置统计 |
| 查看统计 | [id].tsx 统计区](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx) | `GET /campaigns/:id/stats` | [CampaignService.getStats](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L627-L673) | 从 Email 表实时聚合 |

### 2.2 联系人选择操作

| 操作 | 组件 | 说明 |
|------|------|------|
| 搜索联系人 | [ContactPicker 搜索模式](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/components/ContactPicker.tsx#L136-L197) | Popover 内输入，300ms 防抖后调 `GET /contacts?search=` |
| 批量粘贴 | [ContactPicker 粘贴模式](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/components/ContactPicker.tsx#L235-L286) | 文本区输入，400ms 防抖后调 `POST /contacts/lookup` 预览已有/新建 |
| 选择分段 | [create.tsx 分段下拉](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/create.tsx#L357-L391) | 从 `GET /segments` 列表选，显示 memberCount |
| 筛选条件构建 | [SegmentFilterBuilder](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/components/SegmentFilterBuilder.tsx#L809-L817) | 递归 FilterCondition 编辑器，用于段创建/编辑页面 |
| 从已有活动复制 | [CampaignSelectionDialog](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/components/CampaignSelectionDialog.tsx#L46-L93) | 两步：选活动 → 选复制字段（含受众设置） |

---

## 三、数据流理解

### 3.1 活动配置数据流

```
前端表单状态 (useState)
    ↓ network.fetch('POST', '/campaigns', body)
Zod 校验 (CampaignSchemas.create)
    ↓ 解析 + 附加校验
Controller: requireAuth + requireEmailVerified
    ↓ DomainService.verifyEmailDomain(from, projectId)
CampaignService.create(projectId, data)
    ├── 若 audienceType=SEGMENT → 校验 segment 归属项目
    ├── 若 audienceType=FILTERED → SegmentService.validateCondition
    ├── prisma.campaign.create (status=DRAFT, totalRecipients=0)
    ├── getRecipientCount → buildRecipientWhereAsync → prisma.contact.count
    ├── prisma.campaign.update(totalRecipients=实际值)
    └── NtfyService.notifyCampaignCreated
```

### 3.2 受众筛选数据流

`CampaignService.buildRecipientWhereAsync` 是受众筛选的核心路由：

```
buildRecipientWhereAsync(projectId, campaign)
    ├── baseWhere = { projectId, subscribed: true }  // MARKETING/HEADLESS
    │              或 { projectId }                     // TRANSACTIONAL
    │
    ├── audienceType = ALL
    │   └── return baseWhere
    │
    ├── audienceType = SEGMENT
    │   └── buildSegmentWhereAsync(projectId, segmentId, baseWhere)
    │       ├── segment.type = STATIC
    │       │   └── { ...baseWhere, segmentMemberships: { some: { segmentId, exitedAt: null } } }
    │       └── segment.type = DYNAMIC
    │           └── SegmentService.buildConditionClause(condition) → Prisma where
    │
    └── audienceType = FILTERED
        └── SegmentService.buildConditionClause(audienceCondition) → Prisma where
            └── 递归处理 FilterCondition
                ├── groups 以 logic=AND/OR 组合
                ├── 每个 group 内 filters AND 组合
                ├── 每个 filter 按 field 前缀路由：
                │   ├── data.*     → buildJsonFieldCondition
                │   ├── event.*    → buildEventCondition
                │   ├── email.*    → buildEmailActivityCondition
                │   ├── segment.*  → 递归解析引用段（含环形引用检测）
                │   └── email/subscribed/createdAt/updatedAt → 标准字段
                └── group.conditions 嵌套递归
```

### 3.3 邮件投递数据流

```
CampaignService.send / startSending
    ├── 校验 status + recipientCount + 计费限额
    ├── prisma.campaign.update(status=SENDING)
    ├── QueueService.queueCampaignBatch({batchNumber:1, offset:0, limit:500})
    │
    ↓ BullMQ Worker (campaign-processor.ts)
CampaignService.processBatch
    ├── getRecipientsCursor (cursor-based pagination)
    ├── 逐 contact 渲染模板变量 → EmailService.sendCampaignEmail
    ├── hasMore → QueueService.queueCampaignBatch(nextBatch)
    └── !hasMore → reconciliate totalRecipients + finalizeIfDone
        └── 所有 Email 进入终态 → status=SENT
```

### 3.4 前端选择组件数据流

**ContactPicker（用于静态分段添加成员）**：

```
用户输入 → 防抖 → SWR fetch
    搜索模式: GET /contacts?search= → CursorPaginatedResponse<Contact>
    粘贴模式: POST /contacts/lookup → { found[], notFound[] }
    ↓
选中/提交 → onAdd(emails[], subscribed) → SegmentService.addContacts
```

**CampaignSelectionDialog（从已有活动复制）**：

```
SWR: GET /campaigns?page=&pageSize=10&status= → PaginatedResponse<Campaign>
    ↓ 选择活动 + 勾选字段
onSelectCampaign(campaign, selectedFields) → 调用方将字段回填到 create.tsx 表单
```

**SegmentFilterBuilder（用于段条件编辑）**：

```
useAvailableOptions hook:
    并行请求 → GET /contacts/fields + GET /events/names + GET /segments
    ↓
    构建 FieldOption[]（含 Contact Fields / Custom Data / Events / Email Activity / Segments）
    ↓
FilterRow 组件：
    字段选择 → 自动切换可用操作符 → 值输入（按类型渲染不同控件）
FilterGroupComponent：
    filters[] + 可嵌套 conditions
FilterConditionComponent：
    groups[] 以 AND/OR 组合
```

---

## 四、数据一致性缺口

### 4.1 totalRecipients 与实际收件人不一致

**问题**：`totalRecipients` 在创建/更新时计算，但发送时 `buildRecipientWhereAsync` 重新求值，动态段成员可能在间隔期变化。代码中已有 `processBatch` 末尾的对账逻辑（[CampaignService.ts#L519-L533](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L519-L533)），但 **SCHEDULED 状态的活动在到达发送时间前不会重新计算 totalRecipients**，计费限额检查使用的仍是旧值。

**风险**：计费限额可能低估实际发送量，或 totalRecipients 与最终实际发送数偏差较大时影响前端展示。

### 4.2 Segment.memberCount 缓存过期

**问题**：`memberCount` 是缓存值，由后台 job（[segment-count-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/jobs/segment-count-processor.ts)）定期刷新。前端创建活动选择段时显示的联系人数量可能不是实时值。

**影响范围**：前端 [create.tsx#L156-L161](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/create.tsx#L156-L161) 和 [[id].tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx) 中 `segment.memberCount` 展示。

### 4.3 前端创建活动时 FILTERED 类型未传递 audienceCondition

**问题**：[create.tsx#L143-L145](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/create.tsx#L143-L145) 中：

```typescript
audienceFilter: audienceType === CampaignAudienceType.FILTERED ? [] : undefined,
```

前端发送的字段名是 `audienceFilter`（且为空数组），但后端 Zod schema 期望的字段名是 `audienceCondition`（类型为 `FilterCondition`）。这意味着 **FILTERED 受众类型在创建页面实际无法工作**，前端也没有提供 SegmentFilterBuilder 组件来构建条件。创建页面仅支持 ALL 和 SEGMENT 两种受众类型。

### 4.4 update 端点未对 audienceCondition 做 Zod 校验

**问题**：[Campaigns.ts#L112-L149](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/controllers/Campaigns.ts#L112-L149) 中 update 方法直接从 `req.body` 解构字段，**未使用 `CampaignSchemas.update.parse()` 做 Zod 校验**。而 create 方法使用了 `CampaignSchemas.create.parse(req.body)`。这意味着 update 端点的入参校验弱于 create。

### 4.5 Campaign 与 Segment 删除竞态

**问题**：[SegmentService.delete](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/SegmentService.ts#L218-L259) 中检查段是否被活动使用：

```typescript
const campaignsUsingSegment = await prisma.campaign.count({
  where: { segmentId, status: { in: ['DRAFT', 'SCHEDULED', 'SENDING'] } }
});
```

但检查与删除之间没有事务保护，并发情况下可能在检查通过后、删除前有新的活动关联该段。同时，Campaign 模型的 `segmentId` 字段没有外键约束的 `onDelete` 行为（仅 `@relation` 无 `onDelete: Cascade` 或 `SetNull`），删除段后活动记录会留下悬挂外键。

---

## 五、权限校验缺口

### 5.1 所有 Campaign 端点缺少角色区分

**现状**：所有 Campaign 端点仅使用 `[requireAuth, requireEmailVerified]` 中间件。[auth.ts#L227-L328](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/middleware/auth.ts#L227-L328) 中的 `requireAuth` 仅检查用户是否为项目成员（任何角色均可），**不区分 OWNER / ADMIN / MEMBER**。

**缺口**：
- MEMBER 角色用户可以创建、发送、取消、删除活动
- MEMBER 角色用户可以发送测试邮件
- 缺少对敏感操作（发送、取消、删除）的权限提升检查

### 5.2 API Key 认证无角色概念

**现状**：API Key 认证（`requireAuth` 中 Authorization header 分支）仅验证 secret key 有效性，直接赋予项目级权限，无用户级角色概念。任何持有 secret key 的调用者可执行所有操作。

### 5.3 Campaign 归属项目校验依赖 `findFirst` 而非唯一约束

**现状**：[CampaignService.get](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L164-L180) 使用 `findFirst({ where: { id, projectId } })` 校验活动属于项目。这是正确的做法，但 `delete` 方法使用的是 `findUnique({ where: { id, projectId } })`（[CampaignService.ts#L237-L244](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L237-L244)），注意 Prisma `findUnique` 的 `where` 必须是唯一约束字段，`{ id, projectId }` 不是唯一约束组合（id 本身是主键），某些 Prisma 版本可能报错或忽略 projectId 条件。

### 5.4 公开端点缺少速率限制

**现状**：[Contacts.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/controllers/Contacts.ts#L198-L276) 中的公开端点 `GET /contacts/public/:id`、`POST /contacts/public/:id/subscribe`、`POST /contacts/public/:id/unsubscribe` 无认证也无速率限制，可被滥用。

---

## 六、可继续核查的缺口

### 6.1 FILTERED 受众类型的端到端完整性

- 前端创建页面不支持 FILTERED（无 SegmentFilterBuilder），需确认编辑页面 [[id].tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx) 是否支持
- CampaignSelectionDialog 复制受众设置时，FILTERED 的 `audienceCondition` 是否被正确传递
- 需核查是否有 API 用户通过直接调用 API 创建 FILTERED 类型活动

### 6.2 计费限额与并发发送

- [BillingLimitService.checkLimit](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/BillingLimitService.ts) 在 `CampaignService.send` 中检查，但多个活动并发发送时可能同时通过限额检查，导致实际超限
- 需核查限额检查是否有原子性保障（如 Redis INCR 或数据库行锁）

### 6.3 批量处理容错与重试

- [campaign-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/jobs/campaign-processor.ts) 中 `concurrency: 5`，但单个 contact 发送失败仅 log 不重试（[CampaignService.ts#L503-L506](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/CampaignService.ts#L503-L506)）
- 需核查 BullMQ job 级别的重试策略是否覆盖批次级失败（如数据库连接中断导致整批失败）

### 6.4 Segment 引用的递归深度与循环引用

- [SegmentService.buildFilterCondition](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/services/SegmentService.ts#L636-L717) 中使用 `visitedSegments` Set 检测环形引用，但没有递归深度限制
- 恶意构造的深层嵌套条件可能导致查询超时或栈溢出
- 需核查是否有递归深度保护

### 6.5 Campaign 级联删除行为

- Campaign 删除（仅 DRAFT）时，是否需要清理已创建的 Email 记录？
- Prisma schema 中 `Email.campaignId` 设有 `onDelete: Cascade`，但 Campaign 删除限制为 DRAFT 状态，此时理论上不会有 Email 记录
- 需确认 SCHEDULED 状态活动在取消后是否能被删除，以及取消后的 Email 记录清理

### 6.6 update 端点的 from 字段校验时机

- [Campaigns.ts#L128-L130](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/api/src/controllers/Campaigns.ts#L128-L130) 中 `if (from) { await DomainService.verifyEmailDomain(...) }`，这仅在 from 非空时校验
- 但 `CampaignSchemas.update` 中 `from` 是 `z.string().optional()`，可以传入空字符串或无效值绕过域名验证
- 需核查更新逻辑是否在 `from` 为空字符串时跳过域名验证

### 6.7 联系人 data 字段查询的性能边界

- `ContactService.getAvailableFields` 和 `getUniqueFieldValues` 使用 PostgreSQL 原生 SQL 查询 JSON 字段
- 项目有 GIN 索引支持（`@@index([data(ops: JsonbOps)], type: Gin)`）
- 但超大规模联系人（>100万）时需确认查询是否仍在可接受时间范围内
- 需核查是否有查询超时保护

### 6.8 前端 audienceType 切换时的数据残留

- [create.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/create.tsx) 切换 audienceType 时 `segmentId` 仅在 SEGMENT 类型下发送
- 但 [[id].tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/52-plunk/apps/web/src/pages/campaigns/[id].tsx) 编辑页面切换到 ALL 后再切回 SEGMENT，之前选择的 segmentId 是否保留？
- 需核查前端状态管理在类型切换时是否正确清理/恢复关联字段

---

## 七、架构总结

```
┌─────────────────────────────────────────────────────────┐
│                    前端 (Next.js + SWR)                  │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ create   │  │ [id] 编辑    │  │ CampaignSelection │  │
│  │ .tsx     │  │ /统计/发送   │  │ Dialog            │  │
│  └────┬─────┘  └──────┬───────┘  └────────┬──────────┘  │
│       │               │                    │             │
│  ┌────┴─────┐  ┌──────┴───────┐  ┌────────┴──────────┐  │
│  │Segment   │  │ ContactPicker│  │ SegmentFilter     │  │
│  │Select    │  │ 搜索/粘贴    │  │ Builder           │  │
│  └──────────┘  └──────────────┘  └───────────────────┘  │
└─────────────────────────┬───────────────────────────────┘
                          │ network.fetch
┌─────────────────────────┴───────────────────────────────┐
│                    API 层 (Express + OvernightJS)        │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Middleware: requireAuth + requireEmailVerified   │    │
│  │ - JWT: 验证用户+项目成员关系                      │    │
│  │ - API Key: 验证 secret key → projectId           │    │
│  │ - 项目禁用检查 (写操作拦截)                        │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │Campaigns │  │Contacts  │  │Segments  │  Controller   │
│  │Controller│  │Controller│  │Controller│              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │                    │
│  ┌────┴─────┐  ┌─────┴─────┐  ┌────┴─────┐            │
│  │Campaign  │  │Contact    │  │Segment   │  Service     │
│  │Service   │  │Service    │  │Service   │              │
│  └────┬─────┘  └───────────┘  └────┬─────┘            │
│       │          Prisma             │                   │
│       └──────────┬──────────────────┘                   │
│                  │                                      │
│  ┌───────────────┴──────────────┐                      │
│  │  BullMQ (campaign-processor) │  Background Jobs     │
│  │  批量发送 → EmailService     │                      │
│  └──────────────────────────────┘                      │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────┴───────────────────────────────┐
│                    数据层 (PostgreSQL + Redis)            │
│  Prisma ORM + Redis 缓存 (Membership/Domain)            │
└─────────────────────────────────────────────────────────┘
```
