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

## 九、核心结论

1. **认证与授权分离**：OAuth/密码登录负责确认"你是谁"（User 身份），Membership 负责"你能做什么"（Project 权限），两者通过 JWT + X-Project-Id 的组合桥接。

2. **权限粒度为项目级**：没有系统级管理员角色，所有权限都在项目边界内。OWNER 角色是项目内的最高权限，不可被修改或移除，但没有跨项目管理能力。

3. **资源严格归属于项目**：所有业务数据通过 `projectId` 外键绑定，用户通过 Membership 间接访问。API Key 是项目的直接凭证，绕过用户身份。

4. **跨项目隔离不完整**：项目禁用的惩罚会通过 User 维度传播（禁止创建新项目），使得一个项目的安全问题可能影响用户在其他项目中的操作能力，这是一个以安全优先但牺牲了多租户隔离性的设计决策。

5. **缓存与一致性权衡**：Membership 查询使用 Redis 缓存（1 分钟 TTL），在性能与权限一致性之间选择了性能，引入了短暂的权限泄漏窗口。

6. **API Key 是高风险凭证**：sk_* 拥有项目的完全控制权，无用户级审计追踪，泄露后仅能通过重置 Key 来止损，且无法定位泄露源。
