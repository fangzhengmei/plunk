# Plunk 代码理解：OAuth 登录与项目成员权限体系

## 一、系统架构概览

Plunk 是一个邮件平台（Email Marketing + Transactional），采用 monorepo 结构，核心认证与授权代码位于 `apps/api`，前端 Dashboard 位于 `apps/web`，数据模型定义在 `packages/db/prisma/schema.prisma`。

### 关键数据模型关系

```
User ──1:N──> Membership ──N:1──> Project
                   │
                   └── role: OWNER | ADMIN | MEMBER
```

- [User](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/packages/db/prisma/schema.prisma#L16-L33)：核心字段 `email`（唯一）、`type`（AuthMethod 枚举：PASSWORD / GOOGLE_OAUTH / GITHUB_OAUTH）、`emailVerified`
- [Membership](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/packages/db/prisma/schema.prisma#L84-L99)：联合主键 `[userId, projectId]`，`role` 字段控制权限层级
- [Project](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/packages/db/prisma/schema.prisma#L35-L82)：拥有 `public`（pk_*）和 `secret`（sk_*）两套 API Key，`disabled` 状态控制项目可用性

---

## 二、OAuth 登录流程分析（请求→服务→状态流转）

### 2.1 三种认证方式

| 方式 | 入口 | 用户类型 | 邮箱验证 |
|------|------|----------|----------|
| 密码登录 | `POST /auth/login` | `PASSWORD` | 需要验证（若 PLUNK_ENABLED） |
| 密码注册 | `POST /auth/signup` | `PASSWORD` | 发送验证邮件 |
| GitHub OAuth | `GET /oauth/github/outbound` → `callback` | `GITHUB_OAUTH` | 自动验证 |
| Google OAuth | `GET /oauth/google/outbound` → `callback` | `GOOGLE_OAUTH` | 自动验证 |

### 2.2 OAuth 请求流转（以 GitHub 为例）

```
浏览器                    API Server                    GitHub
  │                          │                            │
  │  GET /oauth/github/outbound                           │
  │─────────────────────────>│                            │
  │                          │  302 → github.com/authorize│
  │<─────────────────────────│                            │
  │  redirect to GitHub      │                            │
  │───────────────────────────────────────────────────── >│
  │                          │                            │
  │  GitHub 授权回调 code     │                            │
  │<─────────────────────────────────────────────────────│
  │  GET /oauth/github/callback?code=xxx                  │
  │─────────────────────────>│                            │
  │                          │  POST access_token         │
  │                          │───────────────────────────>│
  │                          │  access_token              │
  │                          │<───────────────────────────│
  │                          │  GET /user/emails          │
  │                          │───────────────────────────>│
  │                          │  [{email, primary}]        │
  │                          │<───────────────────────────│
  │                          │                            │
  │                          │  UserService.email(email)  │
  │                          │  → 查找/创建 User          │
  │                          │  → jwt.sign(user.id)       │
  │                          │  → set-cookie: next_token  │
  │  302 → DASHBOARD_URI     │                            │
  │<─────────────────────────│                            │
```

### 2.3 OAuth 核心逻辑（[Github.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Oauth/Github.ts#L38-L114) / [Google.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Oauth/Google.ts#L33-L104)）

**身份解析策略**：
1. GitHub：请求 `/user/emails` API，取 `primary: true` 的邮箱
2. Google：请求 `oauth2/v3/userinfo`，直接取 `email` 字段

**用户查找/创建**：
```typescript
let user = await UserService.email(email); // 大小写不敏感查找
if (!user) {
  user = await prisma.user.create({
    data: { email, type: 'GITHUB_OAUTH', emailVerified: true }
  });
}
```

**认证方式互斥校验**：
```typescript
if (user.type !== 'GITHUB_OAUTH') {
  return res.redirect(DASHBOARD_URI + '/auth/login?message=You used another form of authentication');
}
```

### 2.4 密码登录流转（[Auth.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Auth.ts#L34-L63)）

```
POST /auth/login { email, password }
  │
  ├── UserService.email(email)           // 查找用户
  ├── AuthService.verifyCredentials()     // bcrypt 验证
  ├── redis.set(user)                    // 缓存用户信息
  ├── jwt.sign(user.id)                  // 签发 JWT
  └── set-cookie: next_token             // 写入 HttpOnly Cookie
```

### 2.5 JWT 机制（[auth.ts middleware](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/middleware/auth.ts#L25-L59)）

- **签名载荷**：`{ id: userId }`，仅包含用户 ID
- **有效期**：168h（7 天）
- **Cookie 名**：`next_token`，HttpOnly、SameSite 策略根据环境动态调整
- **解析**：`parseJwt()` 从 Cookie 提取 token → `jwt.verify()` 解出 userId

---

## 三、用户身份与项目成员关系

### 3.1 身份模型

```
┌──────────────────────────────────────────────┐
│                   User                        │
│  id: uuid                                     │
│  email: unique (case-insensitive)             │
│  type: PASSWORD | GOOGLE_OAUTH | GITHUB_OAUTH│
│  emailVerified: boolean                       │
│                                               │
│  ──1:N──> Membership[]                        │
│              ├── projectId                    │
│              ├── role: OWNER|ADMIN|MEMBER     │
│              └── createdAt                    │
└──────────────────────────────────────────────┘
```

**关键设计**：
- 一个 User 可以属于多个 Project（通过 Membership 多对多）
- 同一邮箱只能注册一个 User（`email` 字段 unique）
- 认证方式 `type` 一旦确定不可混用（OAuth 回调会做互斥校验）
- OAuth 用户自动标记 `emailVerified: true`

### 3.2 项目创建时的 Owner 自动绑定（[Users.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Users.ts#L56-L101)）

```typescript
const project = await prisma.project.create({
  data: {
    name,
    public: publicKey,    // pk_*
    secret: secretKey,    // sk_*
    members: {
      create: {
        userId: auth.userId,
        role: 'OWNER',    // 创建者自动成为 OWNER
      },
    },
  },
});
```

### 3.3 成员管理 API（[Projects.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Projects.ts)）

| 操作 | 端点 | 权限要求 |
|------|------|----------|
| 查看成员 | `GET /projects/:id/members` | 任意成员 |
| 添加成员 | `POST /projects/:id/members` | ADMIN 或 OWNER |
| 修改角色 | `PATCH /projects/:id/members/:userId` | ADMIN 或 OWNER |
| 移除成员 | `DELETE /projects/:id/members/:userId` | ADMIN 或 OWNER |

### 3.4 角色层级与约束

```
OWNER > ADMIN > MEMBER
  │       │       │
  │       │       └── 只读访问项目资源
  │       └────────── 可管理成员、修改项目设置
  └────────────────── 不可被修改角色、不可被移除
```

[MembershipService](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/MembershipService.ts) 的硬性约束：
- **OWNER 角色不可被修改**（`updateRole` 方法第 247-249 行校验）
- **OWNER 不可被移除**（`removeMember` 方法第 291-293 行校验）
- **不可移除自己**（`removeMember` 控制器层第 252-254 行校验）

---

## 四、权限上下文：请求鉴权全链路

### 4.1 双通道认证架构

```
┌─────────────────────────────────────────────────────┐
│                  请求认证入口                         │
│                                                     │
│  ┌──────────────┐         ┌──────────────┐         │
│  │  JWT Cookie   │         │  API Key     │         │
│  │  next_token   │         │  Bearer sk_* │         │
│  └──────┬───────┘         └──────┬───────┘         │
│         │                        │                  │
│         ▼                        ▼                  │
│  parseJwt() → userId      ProjectService.secret()  │
│         │                  → projectId              │
│         │                        │                  │
│         ▼                        ▼                  │
│  X-Project-Id header       res.locals.auth =        │
│  + MembershipService        { type: 'apiKey',       │
│    .getMembership()           projectId }           │
│         │                                           │
│         ▼                                           │
│  res.locals.auth =                                  │
│    { type: 'jwt', userId, projectId }               │
└─────────────────────────────────────────────────────┘
```

### 4.2 中间件层级

[requireAuth](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/middleware/auth.ts#L227-L328) 是最核心的鉴权中间件，处理逻辑：

1. **优先检查 Authorization Header** → 走 API Key 通道
2. **否则走 JWT 通道**：
   - 解析 Cookie 中的 `next_token` → 获取 userId
   - **必须提供 `X-Project-Id` Header** → 获取 projectId
   - **验证 Membership 关系** → `MembershipService.getMembership(userId, projectId)`
   - 无成员关系 → 403 `PROJECT_ACCESS_DENIED`
3. **项目禁用检查**：写入操作（POST/PUT/PATCH/DELETE）在项目 disabled 时被 403 拦截

[isAuthenticated](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/middleware/auth.ts#L19-L23)：轻量级中间件，仅解析 JWT 设置 `res.locals.auth`，不校验项目成员关系（用于 `/users/@me` 等用户级接口）。

[requireEmailVerified](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/middleware/auth.ts#L337-L378)：在认证后追加验证邮箱（API Key 通道自动跳过；OAuth 用户视为已验证）。

[requirePublicKey](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/middleware/auth.ts#L88-L148) / [requireSecretKey](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/middleware/auth.ts#L157-L217)：专用于公开 API（`/v1/track`、`/v1/send`），直接从 API Key 推导 projectId，无用户上下文。

### 4.3 前端项目上下文传递

[ActiveProjectProvider](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/web/src/lib/contexts/ActiveProjectProvider.tsx)：
- 从 `localStorage` 读取 `activeProjectId`
- 切换项目时写入 `localStorage` 并刷新 SWR 缓存

[network.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/web/src/lib/network.ts#L34-L43)：
- 每次 API 请求自动从 `localStorage` 取 `activeProjectId`
- 作为 `X-Project-Id` Header 发送到后端
- 与 Cookie 中的 JWT 一起构成完整的认证上下文

```
前端请求 → Cookie: next_token (JWT) + Header: X-Project-Id
         → 后端 requireAuth → 解析 userId + projectId → 验证 Membership → 放行
```

---

## 五、权限传递分析

### 5.1 权限传播路径

```
User (通过 Membership.role)
  └──→ Project 级操作权限
        ├── MEMBER: 读取项目资源（联系人、模板、活动等）
        ├── ADMIN: 读取 + 成员管理 + 项目设置 + API Key 重置
        └── OWNER: ADMIN 全部权限 + 角色不可变 + 不可被移除

API Key (sk_* / pk_*)
  └──→ Project 级 API 权限
        ├── pk_*: 仅限 /v1/track（追踪事件）
        └── sk_*: 全部 /v1/* API（发送邮件、管理联系人等）
```

**权限不传播**：Membership 角色仅作用于 Dashboard 操作，不传递到 API Key 权限。API Key 的权限范围由 key 类型（pk/sk）决定，与持有者的 Membership 角色无关。

### 5.2 细粒度权限控制分布

| 操作 | MEMBER | ADMIN/OWNER | 实现位置 |
|------|--------|-------------|----------|
| 查看项目资源 | ✅ | ✅ | MembershipService.requireAccess |
| 查看成员列表 | ✅ | ✅ | MembershipService.requireAccess |
| 添加/修改/移除成员 | ❌ | ✅ | MembershipService.requireAdminAccess |
| 修改项目设置 | ❌ | ✅ | MembershipService.requireAdminAccess |
| 重置 API Key | ❌ | ✅ | MembershipService.requireAdminAccess |
| 删除/重置项目 | ❌ | ✅ | MembershipService.requireAdminAccess |
| 管理账单 | 读取✅ / 写入❌ | ✅ | requireAccess / requireAdminAccess |

---

## 六、资源归属模型

### 6.1 资源所有权

所有业务资源（Contact、Template、Campaign、Email、Event、Domain 等）均通过 `projectId` 外键直接归属于 Project：

```
Project ──1:N──> Contact
Project ──1:N──> Template
Project ──1:N──> Campaign
Project ──1:N──> Email
Project ──1:N──> Event
Project ──1:N──> Domain
Project ──1:N──> Workflow
```

**关键设计**：
- 资源不属于 User，而属于 Project
- User 通过 Membership 间接获得对 Project 资源的访问权
- API Key 直接绑定 Project，绕过用户身份，是 Project 级凭证
- Project 删除时级联删除所有资源（`onDelete: Cascade`）

### 6.2 API Key 与资源的映射

```
sk_xxx ──查找──> Project.id ──关联──> 所有 Project 资源
pk_xxx ──查找──> Project.id ──关联──> 仅限 /v1/track
```

[ProjectService](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/ProjectService.ts) 通过 Redis 缓存 Key → Project 映射，API Key 即是 Project 的身份令牌。

---

## 七、跨项目风险标注

### ⚠️ 风险 1：项目禁用的级联影响

**代码位置**：[SecurityService.disableProject](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/SecurityService.ts#L654-L727)

**风险描述**：当一个项目被安全系统自动禁用（退信率/投诉率超标、钓鱼检测），该项目的 **所有成员** 都会受到牵连：

1. 项目禁用后，所有成员的写入操作被阻断
2. [创建项目限制](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Users.ts#L66-L74)：如果用户属于任何禁用项目，**该用户不能创建新项目**

```typescript
const {hasDisabledProject} = await SecurityService.userHasDisabledProject(auth.userId);
if (hasDisabledProject) {
  throw new HttpException(403, `You cannot create new projects at this time.`);
}
```

**跨项目影响**：一个项目被禁用 → 该项目的 MEMBER（可能只是普通成员）在其他项目中也失去创建新项目的能力。这是以 User 为中心的惩罚，而非以 Membership 为中心的隔离。

### ⚠️ 风险 2：ADMIN 可添加任意已注册用户

**代码位置**：[Projects.addMember](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Projects.ts#L129-L173)

**风险描述**：ADMIN 可以通过邮箱将任何已注册用户添加到项目中，无需被添加者确认。这导致：
- 用户可能被强制加入不相关的项目
- 被添加的用户会立即获得对项目资源的访问权
- 如果该项目后续被禁用，被强制加入的用户也会受到"禁止创建新项目"的连带影响

### ⚠️ 风险 3：Membership 缓存导致权限变更延迟

**代码位置**：[MembershipService](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/MembershipService.ts#L24-L84)

**风险描述**：`hasAccess`、`hasAdminAccess`、`getMembership` 均使用 Redis 缓存（TTL 1 分钟）。当成员角色被变更或成员被移除后，最多有 1 分钟的窗口期：
- 被移除的成员仍可访问项目资源
- 被降级的 ADMIN 仍可执行管理操作

虽然 CRUD 操作后会调用 `invalidateCache`，但在高并发或缓存失效失败时，存在权限泄漏窗口。

### ⚠️ 风险 4：API Key 无用户审计追踪

**代码位置**：[requireSecretKey](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/middleware/auth.ts#L157-L217)

**风险描述**：通过 API Key（sk_*）发起的请求仅解析出 `projectId`，不关联 `userId`。这意味着：
- API Key 泄露后无法追踪具体操作者（项目内多个成员共享同一组 Key）
- 审计日志中 `authType: 'apiKey'` 的请求无法关联到具体用户
- 任何持有 sk_* 的人（包括被移除的前成员）仍可操作项目资源，直到 Key 被重置

### ⚠️ 风险 5：OAuth 认证方式锁定

**代码位置**：[Github.ts#L101-L103](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Oauth/Github.ts#L101-L103) / [Google.ts#L91-L93](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Oauth/Google.ts#L91-L93)

**风险描述**：用户首次注册的认证方式（PASSWORD/GOOGLE_OAUTH/GITHUB_OAUTH）被锁定，不可切换。如果用户先通过密码注册，后来尝试用同一邮箱的 GitHub 登录，会收到 "You used another form of authentication" 错误。这种设计：
- 阻止了账户接管（安全收益）
- 但用户无法合并多种登录方式到同一账户（体验代价）
- 没有账户关联/合并机制

### ⚠️ 风险 6：前端 localStorage 的项目上下文可被篡改

**代码位置**：[network.ts#L34-L43](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/web/src/lib/network.ts#L34-L43)

**风险描述**：`activeProjectId` 存储在 `localStorage` 中，作为 `X-Project-Id` Header 发送。虽然后端 `requireAuth` 中间件会验证 Membership 关系，但如果用户手动修改 `localStorage` 为一个他们没有权限的 projectId，后端会正确返回 403。然而，在 `isAuthenticated` 中间件保护的端点（如 `/users/@me`），不校验项目归属，这些接口不受影响。总体风险可控，但需注意 `isAuthenticated` 保护的接口不应依赖 `X-Project-Id`。

---

## 八、状态流转总结

### 8.1 用户生命周期

```
[未注册] ──OAuth/注册──> [已创建] ──邮箱验证──> [已验证]
  │                        │
  │    DISABLE_SIGNUPS     │  type 互斥校验
  │    可阻断注册           │  (OAuth vs PASSWORD)
  ▼                        ▼
[注册被拒]              [认证失败]
```

### 8.2 项目生命周期

```
[创建] ──OWNER 自动绑定──> [活跃]
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        [EMAIL_REPUTATION] [PHISHING]  [PAYMENT_FAILED]
              │              │              │
              └──────┬───────┘              │
                     ▼                      │
                  [disabled=true]  ◄────────┘
                     │
                     ├── 写入操作被 403 拦截
                     ├── 所有成员禁止创建新项目
                     └── 读取操作仍可执行
```

### 8.3 请求认证状态机

```
请求进入
  │
  ├── Authorization Header?
  │     ├── YES → requireSecretKey/requirePublicKey
  │     │         → API Key → projectId
  │     │         → type: 'apiKey'
  │     │
  │     └── NO  → parseJwt(Cookie: next_token)
  │               → userId
  │               ├── X-Project-Id Header?
  │               │     ├── YES → MembershipService.getMembership()
  │               │     │         → type: 'jwt', userId, projectId
  │               │     └── NO  → 400 BAD_REQUEST
  │               │
  │               └── Cookie 无效 → 401 NOT_AUTHENTICATED
  │
  └── 项目禁用?
        ├── 写入操作 → 403 PROJECT_DISABLED
        └── 读取操作 → 放行
```

---

---

## 十、后台任务权限链路：群发、工作流触发、邮件队列消费

### 10.1 工作进程获取项目上下文的方式

后台任务全部基于 **BullMQ（Redis Queue）** 架构，使用 `Worker` 消费队列消息。与 HTTP 请求链路不同，Worker **完全绕过 JWT/Membership 中间件体系**，直接从队列 Job 的数据负载（payload）中提取业务实体 ID，再通过数据库关联查询反推出 `projectId` 与 `disabled` 状态。

```
                        ┌──────────────────────────┐
                        │    BullMQ Redis Queue    │
                        └────────────┬─────────────┘
                                     │ job.data
                                     ▼
┌──────────────────────────────────────────────────────────────┐
│                       Worker Handler                          │
│                                                              │
│  campaign:   job.data.campaignId                             │
│                  → prisma.campaign.findUnique                │
│                  → include: { project: true }                │
│                  → campaign.projectId / project.disabled     │
│                                                              │
│  email:      job.data.emailId                                │
│                  → prisma.email.findUnique                   │
│                  → include: { project: true, campaign, ... } │
│                  → email.projectId / project.disabled        │
│                                                              │
│  workflow:   job.data.executionId                            │
│                  → prisma.workflowExecution.findUnique       │
│                  → include: { workflow: { project: {...} } } │
│                  → workflow.projectId / project.disabled     │
│                                                              │
│  scheduled:  job.data.campaignId                             │
│                  → prisma.campaign.findUnique                │
│                  → include: { project: { disabled, id } }    │
│                  → campaign.projectId / project.disabled     │
└──────────────────────────────────────────────────────────────┘
```

### 10.2 各 Worker 的项目禁用检查一览

| Worker | 文件入口 | 禁用检查执行点 | 检查方式 | 处理策略 |
|--------|----------|---------------|----------|----------|
| **Email Processor** | [email-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/jobs/email-processor.ts#L82-L120) | 加载 email 后，**发送前**（line 103-120） | `email.project.disabled` | ✅ 标记 Email 为 FAILED（"Project is disabled"）；若属 Campaign 则调用 `finalizeIfDone` 避免卡死 |
| **Campaign Batch Processor** | [CampaignService.processBatch](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/CampaignService.ts#L437-L533) | 加载 campaign 后（line 455） | ❌ 仅校验 `campaign.status !== SENDING`，**不检查 `project.disabled`** | ⚠️ 若项目在 Campaign 进入 SENDING 状态后才被禁用，此处会继续调用 `EmailService.sendCampaignEmail` 入队下游 Email。**下游 Email Processor 才会拦截发送**，但 Campaign 自身仍会推进到下一批 |
| **Scheduled Campaign** | [scheduled-processor.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/jobs/scheduled-processor.ts#L38-L50) | **开始执行前**（line 38-44） | `campaign.project.disabled` | ✅ 将 Campaign 状态由 SCHEDULED 改为 CANCELLED，不入队 |
| **Workflow Step (主路径)** | [WorkflowExecutionService.processStepExecution](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/WorkflowExecutionService.ts#L91-L105) | 加载 execution 后，**步骤执行前**（line 91-105） | `workflow.project.disabled` | ✅ 将 WorkflowExecution 标记为 CANCELLED，exitReason="Project disabled" |
| **Workflow Timeout** | [WorkflowExecutionService.processTimeout](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/WorkflowExecutionService.ts#L305-L396) | 方法入口处 | ❌ **完全不检查** `project.disabled` | ⚠️ 会继续完成超时步骤，然后递归调用 `processStepExecution` 推进下一步——**下游方法才会检查 disabled**。实际发送邮件仍会被拦截，但状态流转逻辑不受控 |
| **Workflow handleEvent** | [WorkflowExecutionService.handleEvent](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/WorkflowExecutionService.ts#L401-L462) | 方法入口处 | ❌ **完全不检查** `project.disabled` | ⚠️ 事件到达后会继续推进后续步骤，同样依赖 `processStepExecution` 的下游检查 |

### 10.3 项目禁用时的队列清理兜底

[QueueService.cancelAllProjectJobs](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/QueueService.ts#L545-L640) 是项目禁用后的兜底操作：

1. **遍历所有队列**（scheduled、email、campaign、workflow）中 `waiting/delayed` 状态的 Job
2. **每个 Job 都要再查一次数据库**（N+1 查询）以确认是否属于被禁用的 projectId
3. 匹配的 Job 被 `remove()` 出队
4. **额外将仍处于 PENDING 的 Email 批量 updateMany 为 FAILED**（line 609-612）
5. **将仍处于 SENDING 状态的 Campaign 通过 `finalizeIfDone` 收口**

> ⚠️ 缺陷：已处于 `active`（正在执行中）状态的 Job 无法被取消。Campaign batch、Workflow step、Email send 正在执行中的作业会继续走完。

### 10.4 后台任务权限模型的关键特性

**与 HTTP 鉴权完全隔离：**
- 无 Membership 校验：Worker 不会检查"当前用户"是否有执行该项目操作的权限（因为本就没有"当前用户"概念）
- 权限信任来源是 **Job payload 中的业务实体 ID**。若攻击者能向 Redis 队列注入伪造的 Job（即获得 Redis 写入权限），即可触发任意项目的邮件发送——这比 JWT/API Key 体系的防线更外层
- 无 `userId` 审计：后台任务执行结果（Email 发送记录、Campaign 状态变更、Workflow 执行）**均不关联操作人 ID**，仅能追溯到 projectId

**双层防御结构：**
```
HTTP 入队时（requireAuth + Membership）
    ↓ 校验通过才会写队列
BullMQ Job 执行
    ↓ 无法再校验用户
项目 disabled 检查（部分路径缺失）
    ↓
Email 实际发送（phishing 内容二次检查 + SES 风控）
```

### 10.5 ⚠️ 后台任务链路的风险点

1. **Campaign Batch Processor 不检查项目禁用**：如果项目在 Campaign 已切换为 SENDING 并入队若干 batch 后才被禁用（例如正在发送时触发钓鱼检测），剩余 batch 仍会继续生产 Email 记录并推入 email 队列。虽然 Email Processor 最终会拦截这些 Email，但：
   - 会产生大量 FAILED 状态的 Email 垃圾记录
   - 批次链的最后一次 `finalizeIfDone` 才能收口，中间有延迟
   - 额外消耗队列吞吐

2. **Workflow Timeout/Event 不做禁用检查**：WAIT_FOR_EVENT 的超时时长可能很长（数天甚至更久），若项目禁用期间 timeout 触发，会在"无项目禁用检查"的状态下完成 stepExecution 更新，再递归到 `processStepExecution` 时才被拦住。状态机流转（step 标记完成、execution 切回 RUNNING 再到 CANCELLED）会产生不必要的写操作。

3. **清理逻辑的 N+1 查询**：`cancelAllProjectJobs` 遍历每个 Job 再查 DB 确认归属，在队列深度较大时可能成为性能瓶颈。Job payload 中应直接携带 projectId，避免 DB 回查。

---

## 十一、OAuth state 参数与 CSRF 机制分析

### 11.1 OAuth 2.0 state 参数的作用

OAuth 2.0 授权码流程中，`state` 参数是防止 **CSRF（Cross-Site Request Forgery）攻击** 的标准机制：
1. 客户端在发起授权请求前生成一个**不可预测的随机值 state**
2. 将 state 通过 URL 参数发送到授权服务器
3. 同时将 state 保存在用户的会话（Cookie/Session）中
4. 授权服务器回调时原封不动地回传 state
5. 客户端比对回调中的 state 与会话中保存的 state，**不一致则拒绝**

攻击者若想构造一个伪造的回调链接让受害者登录，必须知道受害者会话中保存的 state 值——这被浏览器同源策略阻止。

### 11.2 现状分析：state 参数完全缺失

#### GitHub OAuth

[Github.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Oauth/Github.ts) 的 outbound 与 callback：

```typescript
// outbound [line 22-34]
const OAUTH_QS = new URLSearchParams({
  client_id: GITHUB_OAUTH_CLIENT,
  redirect_uri: `${API_URI}/oauth/github/callback`,
  response_type: 'code',
  scope: 'user:email',
  // ❌ 完全没有 state 参数
});
return res.redirect(`https://github.com/login/oauth/authorize?${OAUTH_QS.toString()}`);

// callback [line 37-114]
const {code} = req.query; // ❌ 只解构 code，不读 state
// ❌ 没有任何 state 校验逻辑
```

#### Google OAuth

[Google.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Oauth/Google.ts) 的 outbound 与 callback：

```typescript
// outbound [line 22-29]
return res.redirect(
  `https://accounts.google.com/o/oauth2/v2/auth?scope=...&prompt=select_account&response_type=code&redirect_uri=...&client_id=...`
  // ❌ 完全没有 state 参数
);

// callback [line 32-104]
const {code} = req.query; // ❌ 只解构 code，不读 state
// ❌ 没有任何 state 校验逻辑
```

### 11.3 两种具体的 CSRF 攻击路径

#### 攻击 1：登录 CSRF（最常见，危害中等）

```
攻击者（控制 attacker@github.com）                受害者浏览器
        │                                          │
        │  1. 自己发起 GitHub OAuth               │
        │  2. GitHub 授权，拿到 code=XXXX         │
        │  3. 构造 URL：                           │
        │     /oauth/github/callback?code=XXXX     │
        │  4. 诱导受害者点击（img/src/iframe）     │
        │─────────────────────────────────────────>│
        │                                          │ 5. 受害者浏览器带着 Cookie
        │                                          │    上下文请求 callback
        │                                          │ 6. 后端用 code=XXXX 换到
        │                                          │    attacker@github.com 的 token
        │                                          │ 7. 后端签发 attacker 的 JWT
        │                                          │    写入受害者的 Cookie
        │                                          │ 8. 受害者登录到了攻击者账户
        │                                          │    （浑然不知，可能写入
        │                                          │     敏感联系人/数据）
```

#### 攻击 2：授权劫持（危害高，针对新用户注册场景）

如果攻击者为 `victim@github.com` 账户预先注册（通过密码注册）了一个 PASSWORD 类型的 Plunk 账户，虽然 `user.type` 互斥校验会阻止直接关联，但如果 `victim@github.com` 尚未有 Plunk 账户，攻击 1 会直接在受害者的浏览器下创建攻击者 OAuth 账户，然后受害者可能手动退出再去自己走 OAuth——此时账户已被占用（邮箱唯一），需要用"其他认证方式"错误提示回退。

### 11.4 已有防护措施与剩余风险

| 防线 | 是否存在 | 说明 |
|------|----------|------|
| `state` 参数校验 | ❌ 完全缺失 | 核心 CSRF 防线未建立 |
| PKCE（Proof Key for Code Exchange） | ❌ 未使用 | 公共客户端的推荐方案，机密客户端可增强 |
| `redirect_uri` 精确匹配 | ✅ 服务端硬编码 | `redirect_uri` 通过常量拼接，不接受用户输入，可防止回调 URL 污染 |
| `SameSite` Cookie | ✅ 动态配置 | `UserService.cookieOptions()` 根据生产/开发环境配置 SameSite，能一定程度降低登录 CSRF 风险（顶级导航跳转场景仍可能绕过） |
| OAuth `code` 一次性使用 | ✅ GitHub/Google 原生保证 | 授权码通常是一次性的，攻击者先消费则受害者回调会失败——但仍有竞态窗口 |

### 11.5 风险评估（高）

- **直接后果**：攻击者可以将自己的 OAuth 账户登录到受害者的浏览器，诱导受害者操作攻击者的项目（上传联系人数据、设置发件域名等）
- **修复成本**：低。只需在两个 OAuth Controller 中增加：
  1. outbound 时生成 `crypto.randomBytes(32).toString('hex')` → 存入 Redis/Cookie（关联用户会话）→ 拼接到授权 URL 的 `state` 参数
  2. callback 时从 query 读 state → 从 Redis/Cookie 取原值 → 严格相等比较 → 不匹配则 400 拒绝 → 无论成功失败都消费（删除）已存的 state

---

## 十二、成员缓存失效触发点与遗漏路径

### 12.1 缓存体系概览

[MembershipService](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/MembershipService.ts) 通过 Redis 缓存四组键（TTL 定义在 line 38/62/83/185）：

| 缓存键模式 | 对应方法 | TTL | 读频率 |
|-----------|---------|-----|--------|
| `membership:access:{userId}:{projectId}` | `hasAccess()` | 1 min | **每次 JWT 请求**（requireAuth 中间件） |
| `membership:admin:{userId}:{projectId}` | `hasAdminAccess()` | 1 min | 每次管理类操作前 |
| `membership:full:{userId}:{projectId}` | `getMembership()` | 1 min | requireAuth 中间件 + requireAdminAccess |
| `membership:owner:{projectId}` | `getOwner()` | 5 min | 低频（通知、管理员查询等） |

所有写操作通过私有方法 `invalidateCache(projectId, userId?)`（line 348-367）执行失效，策略是 **精确删除（DELETE）** 而非 TTL 等待。

### 12.2 正常触发缓存失效的路径 ✅

成员 CRUD 全部集中在 `MembershipService`，三个公共方法写操作后均调用 `invalidateCache`：

| 操作 | 方法 | 代码位置 | 失效的键 |
|------|------|----------|---------|
| **添加成员** | `addMember()` | [line 221](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/MembershipService.ts#L221) | access/admin/full（新成员） + owner |
| **修改角色** | `updateRole()` | [line 265](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/MembershipService.ts#L265) | access/admin/full（被改角色的用户） + owner |
| **移除成员** | `removeMember()` | [line 306](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/services/MembershipService.ts#L306) | access/admin/full（被移除用户） + owner |

> 注意：以上三个操作**始终失效 owner 缓存**，即使改动与 OWNER 无关（例如 MEMBER ↔ ADMIN 互转）。这属于过度失效（owner 5 分钟 TTL 也会到期），但不影响正确性。

### 12.3 缓存失效遗漏路径 ❌

#### 遗漏 1：项目创建时的嵌套 Member 创建

**代码位置**：[Users.ts:88-93](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Users.ts#L88-L93)

```typescript
const project = await prisma.project.create({
  data: {
    name, public: publicKey, secret: secretKey,
    members: {
      create: { userId: auth.userId, role: 'OWNER' },  // ← 直接嵌套创建 Membership
    },
  },
});
// ❌ 此处未调用 MembershipService.invalidateCache()
```

**影响评估**：低风险。这是全新 Project 与其首个 Membership，对应的四个缓存键在 Redis 中尚不存在，没有脏数据问题。但在以下边缘场景可能出问题：
- 单测中复用 userId + projectId 组合（先删 Project 再重建），旧缓存仍在 TTL 窗口内
- 分布式环境下 Project 创建后，另一个节点极快地对该 (userId, projectId) 发起查询并缓存了 `null` 值（读穿竞态）

#### 遗漏 2：项目删除时的级联 Membership 清除

**代码位置**：[Users.ts:730-732](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Users.ts#L730-L732)

```typescript
await prisma.project.delete({ where: { id } });
// ❌ Prisma Cascade onDelete: Cascade 删除了该项目所有 Membership
// ❌ 但未对这些 Membership 对应的 Redis 键执行失效
```

**影响评估**：中风险。
- 项目删除后，被删除的 (userId, projectId) 组合在 1 分钟 TTL 内仍可能返回 `true`/旧 Membership——但后续业务查询会因为 Project 不存在而报 404（`requireAuth` 中校验 Membership 为 true 后，后续 CRUD 会查不到 Project），因此**没有实际权限泄漏，仅浪费 Redis 内存**
- `membership:owner:{projectId}` 键在 5 分钟 TTL 内会返回已删除项目的 Owner 信息

#### 遗漏 3：项目重置操作

**代码位置**：[Users.ts:632-672](file:///d:/fz/0601-1/solo-dogfeeding/code/58-plunk/apps/api/src/controllers/Users.ts#L632-L672)

项目重置会删除 email/event/campaign/workflow/segment/contact/template/apiRequest，但 **不触碰 Membership 表**。

**评估**：✅ 无缓存失效需求，Membership 未变更。

#### 遗漏 4：测试代码中的直接 Membership 创建

多个集成测试文件（如 `__tests__/integration/domains.test.ts`、`__tests__/services/DomainService.test.ts`）直接使用：

```typescript
await prisma.membership.create({ data: { userId, projectId, role } });
```

**评估**：仅影响测试环境。生产代码通过 Grep 确认所有 Membership 写入都集中在 MembershipService 三个方法 + 上述遗漏 1/2 的两处。

#### 遗漏 5：`invalidateCache` 未失效 `hasAccess`/`hasAdminAccess` 对应的间接调用方

虽然 `invalidateCache` 精确删除了 access/admin/full 三个用户维度键，但需注意：`requireAuth` 中间件实际调用的是 `getMembership()`（full 键），然后从返回值读取 role。`hasAccess()` 和 `hasAdminAccess()` 在其他直接调用点仍命中各自独立的键。`invalidateCache` 对这三个键**都删除**了，所以是完整的。

### 12.4 缓存失效一致性的完整性检查矩阵

| 写操作 | 代码路径 | access 键 | admin 键 | full 键 | owner 键 |
|--------|---------|-----------|----------|---------|----------|
| 新增成员 | MembershipService.addMember | ✅ 删 | ✅ 删 | ✅ 删 | ✅ 删 |
| 角色变更 | MembershipService.updateRole | ✅ 删 | ✅ 删 | ✅ 删 | ✅ 删 |
| 移除成员 | MembershipService.removeMember | ✅ 删 | ✅ 删 | ✅ 删 | ✅ 删 |
| 创建项目（嵌套创建 OWNER）| Users.ts 嵌套 create | ❌ 不删 | ❌ 不删 | ❌ 不删 | ❌ 不删 |
| 删除项目（级联删除所有 Members）| Users.ts prisma.project.delete | ❌ 不删 | ❌ 不删 | ❌ 不删 | ❌ 不删 |
| 重置项目 | Users.ts deleteMany（非 Membership 表）| N/A | N/A | N/A | N/A |

### 12.5 ⚠️ 额外隐患：失效操作是"尽力而为"

`invalidateCache` 使用 `redis.del(...keysToDelete)`，**没有数据库事务保障**。如果 Prisma 写入成功但 Redis DEL 失败（网络闪断、Redis 临时不可用），脏数据将在 TTL 内持续存在。补救方式：
- 将 Redis DEL 改为 Lua 脚本或 pipeline，增加可靠性
- 或使用订阅/发布机制主动广播失效事件
- 可考虑在鉴权中间件中增加"当缓存结果与 DB 查询在极小概率冲突时"的兜底双写校验（仅用于关键 admin 权限）

---

## 十三、核心结论（补充版）

1. **认证与授权分离**：OAuth/密码登录负责确认"你是谁"（User 身份），Membership 负责"你能做什么"（Project 权限），两者通过 JWT + X-Project-Id 的组合桥接。

2. **权限粒度为项目级**：没有系统级管理员角色，所有权限都在项目边界内。OWNER 角色是项目内的最高权限，不可被修改或移除，但没有跨项目管理能力。

3. **资源严格归属于项目**：所有业务数据通过 `projectId` 外键绑定，用户通过 Membership 间接访问。API Key 是项目的直接凭证，绕过用户身份。

4. **跨项目隔离不完整**：项目禁用的惩罚会通过 User 维度传播（禁止创建新项目），使得一个项目的安全问题可能影响用户在其他项目中的操作能力，这是一个以安全优先但牺牲了多租户隔离性的设计决策。

5. **缓存与一致性权衡**：Membership 查询使用 Redis 缓存（1 分钟 TTL），在性能与权限一致性之间选择了性能，引入了短暂的权限泄漏窗口。项目创建/删除两条路径遗漏了缓存失效调用。

6. **API Key 是高风险凭证**：sk_* 拥有项目的完全控制权，无用户级审计追踪，泄露后仅能通过重置 Key 来止损，且无法定位泄露源。

7. **后台任务权限模型不同质**：HTTP 请求链路走 JWT+Membership 双层校验，BullMQ Worker 直接信任 Job payload 中的业务实体 ID，仅在部分路径检查 `project.disabled`。Campaign Batch Processor 和 Workflow Timeout/handleEvent 的项目禁用检查缺失，依赖下游环节兜底。

8. **OAuth CSRF 防御缺位**：GitHub 和 Google 两个 OAuth 流程均未实现 `state` 参数校验，存在登录 CSRF 漏洞。攻击者可将自己的 OAuth 账户绑定到受害者浏览器，实现账户混淆。SameSite Cookie 和 `redirect_uri` 硬编码提供了部分防护，但不足以完全阻断攻击。

9. **队列清理兜底逻辑代价高**：`QueueService.cancelAllProjectJobs` 采用遍历队列 + N+1 DB 回查的方式清理禁用项目的积压作业。若 Job payload 直接携带 projectId，可将复杂度从 O(N) 降为 O(1)。

10. **Worker 无操作人审计**：后台触发的 Email 发送、Campaign 状态变更、Workflow 执行记录均不关联 userId。如果出现误操作或恶意队列注入，无法追溯责任人。
