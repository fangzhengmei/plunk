# 模板编辑器与 HTML 转换 · 代码理解

本文以代码为主线，系统梳理 **Plunk** 项目中「富文本邮件模板编辑器 → HTML 转换 → 变量替换 → 存储 → 邮件发送」的完整运转机制。

---

## 一、架构总览与关键文件定位

Plunk 是一个邮件营销平台，采用 Monorepo 结构。模板编辑器与 HTML 转换涉及以下模块：

| 层级 | 位置 | 职责 |
|------|------|------|
| 前端-模板编辑页 | [templates/create.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/pages/templates/create.tsx) / [templates/[id].tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/pages/templates/%5Bid%5D.tsx) | 模板的创建/编辑页面，承载表单 + 编辑器 |
| 前端-编辑器组件 | [EmailEditor.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/EmailEditor.tsx) | 富文本编辑器主控组件：双模式切换、变量弹窗、预览 |
| 前端-可视化编辑器(Tiptap) | EmailEditor.tsx + [Toolbar.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/Toolbar.tsx) + [ResizableImage.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/ResizableImage.tsx) | 所见即所得的可视化编辑 |
| 前端-HTML 代码编辑器 | [HtmlEditor.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/HtmlEditor.tsx) | 基于 CodeMirror 的原始 HTML 代码编辑 |
| 前端-变量扩展 | [VariableExtension.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/VariableExtension.ts) + [VariableMention.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/VariableMention.ts) | 变量高亮装饰 + `{{` 自动补全 |
| 前端-邮件样式 | [emailStyles.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/lib/emailStyles.ts) | Custom HTML 检测 + 预览时的 Prose 样式包装 |
| Landing-工具页编辑器 | [MarkdownEmailEditor.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/landing/src/components/tools/MarkdownEmailEditor.tsx) + [emailHtmlConverter.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/landing/src/lib/emailHtmlConverter.ts) | 官网「Markdown→邮件」工具（与 Dashboard 编辑器不同的简化版本） |
| 共享-模板渲染引擎 | [template.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/shared/src/template.ts) | `renderTemplate()`：`{{var}}` 语法变量替换 |
| 共享-退订信号检测 | [unsubscribe.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/shared/src/unsubscribe.ts) | `detectUnsubscribeSignal()`：检测 HEADLESS 模板是否含退订链接 |
| 共享-Zod Schema 校验 | [schemas/index.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/shared/src/schemas/index.ts) | `TemplateSchemas.create / update`：服务端入参校验 |
| 后端-HTTP 接口 | [Templates.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Templates.ts) | `@Controller('templates')`：RESTful CRUD 路由 |
| 后端-业务服务 | [TemplateService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/TemplateService.ts) | 模板读写 + 删除时的 workflow 引用校验 |
| 后端-邮件发送服务 | [EmailService.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts) | `format()` 变量替换 + `compile()` 样式包装与退订脚注注入 + 实际 SES 发送 |
| 数据库模型 | [schema.prisma#L159-L190](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/db/prisma/schema.prisma#L159-L190) | `Template` 表结构定义 |

---

## 二、富文本邮件编辑器：双模式运转机制

核心组件是 [EmailEditor.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/EmailEditor.tsx)，它提供 **Visual（可视化）** 和 **HTML（源代码）** 两种编辑模式。

### 2.1 两种模式的切换逻辑

```typescript
// EmailEditor.tsx L55-L172
const initialMode = detectCustomHtmlPatterns(value) ? 'html' : 'visual';
const [mode, setMode] = useState<'visual' | 'html'>(initialMode);
```

**初始化判断**：根据传入的 HTML 是否包含「Custom HTML 特征」决定启动模式。如果用户之前在 HTML 模式下写了 `<table>`、`<style>`、自定义 class 等不被 Tiptap 识别的内容，则直接进入 HTML 模式，避免 Tiptap 解析时丢失信息。

**切换保护**：
- Visual → HTML：无风险，直接把 `editor.getHTML()` 复制给 `htmlContent` 即可
- HTML → Visual：**必须**经过 `detectCustomHtmlPatterns()` 检查。如果包含自定义内容，弹出「Custom HTML Detected」确认对话框（[EmailEditor.tsx L697-L736](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/EmailEditor.tsx#L697-L736)），用户必须选择「Stay in HTML Mode」或「Switch Anyway」（破坏性操作）

### 2.2 Visual 模式：Tiptap 扩展栈

[EmailEditor.tsx L88-L130](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/EmailEditor.tsx#L88-L130) 中 `useEditor` 配置的完整扩展链：

| 扩展 | 功能 |
|------|------|
| `StarterKit` | 段落、H1-H3、加粗/斜体/删除线、有序无序列表、引用、分割线等基础能力 |
| `TextAlign` | 段落对齐（左/中/右/两端） |
| `Color` + `TextStyle` | 文字颜色，序列化后为 `<span style="color: ...">` |
| `Link` | 超链接插入/编辑，带 `rel="noopener noreferrer"` |
| `ResizableImage` | **自定义节点**：可拖拽四角调整尺寸的图片（见 2.4 节） |
| `Variable` | **自定义节点/装饰**：对 `{{...}}` 文本自动高亮蓝色背景（见 3.2 节） |
| `VariableMention` | 基于 `@tiptap/extension-mention` 的变量自动补全（见 3.3 节） |
| `Placeholder` | 空编辑器时的占位提示文字 |

编辑器的 `onUpdate` 回调实时把 Tiptap 内部 ProseMirror 文档序列化为 HTML 字符串：

```typescript
// EmailEditor.tsx L123-L129
onUpdate: ({editor}) => {
  const html = editor.getHTML();
  onChange(html);          // 上抛给 create/[id] 页面
  setHtmlContent(html);    // 同步给 HTML 模式缓冲区
  setPreviewUpdateTrigger(); // 触发预览刷新
},
```

### 2.3 HTML 模式：CodeMirror 代码编辑

[HtmlEditor.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/HtmlEditor.tsx) 使用 `@uiw/react-codemirror` + `@codemirror/lang-html`，启用了：

- 行号、高亮当前行、括号匹配、自动关闭标签
- 代码折叠、搜索替换、自动补全、lint
- **行换行**（`EditorView.lineWrapping`）：避免水平滚动条

直接通过 `onChange` 上抛原始 HTML 字符串。

### 2.4 自定义节点：可调整尺寸的图片

[ResizableImage.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/ResizableImage.tsx) 使用 Tiptap 的 `ReactNodeViewRenderer` 机制，把 ProseMirror 中的 `image` 节点渲染成一个 React 组件，该组件在选中时显示 4 个角的拖拽手柄：

```typescript
// ResizableImage.tsx L68-L83 拖动开始时记录起始位置与原始尺寸
handleResizeStart = (e, direction) => {
  startPos.current = {x: e.clientX, y: e.clientY};
  startSize.current = {width: rect.width, height: rect.height};
  setIsResizing(true);
};

// L22-L66 根据方向计算新宽度，固定长宽比，调用 updateAttributes 回写节点
updateAttributes({ width: Math.round(newWidth), height: Math.round(newHeight) });
```

序列化到 HTML 时变成带有 `width`/`height` 属性的 `<img>`：

```html
<img src="..." width="400" height="300" class="email-image">
```

### 2.5 工具栏命令映射

[Toolbar.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/Toolbar.tsx) 的所有按钮都对应 `editor.chain().focus().XXX().run()` 命令模式。按钮激活态通过 `editor.isActive(...)` 查询 ProseMirror 当前选区的 Mark/Node 状态，并通过 `data-active=true` 的 Tailwind 条件样式高亮。

按钮分为几组（从左到右）：
1. **历史**：Undo / Redo
2. **文本格式**：Bold / Italic / Strike / Code
3. **标题**：H1 / H2 / H3
4. **列表**：Bullet List / Ordered List / Blockquote
5. **对齐**：Left / Center / Right / Justify
6. **颜色**：56 色调色板 + 自定义 HEX 输入
7. **链接**：URL 输入弹窗 + 删除链接
8. **图片**：URL 上传 / S3 文件上传（受 `features.storage.s3Enabled` 控制）
9. **变量**：打开 Insert Variable 对话框

---

## 三、变量系统：定义、补全、高亮、替换

### 3.1 变量语法

支持两种语法，定义在共享包中：

```typescript
// packages/shared/src/template.ts
{{variable}}                      // 基础语法
{{variable ?? defaultValue}}      // 带默认值（fallback）
{{data.firstName}}                // 嵌套属性访问（点路径）
```

### 3.2 变量高亮装饰（Variable 扩展）

[VariableExtension.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/VariableExtension.ts) 提供了两种机制：

**1) 节点级 Node（atom）**：`span[data-variable]`，这是遗留设计——允许通过命令 `insertVariable(name)` 插入一个原子性节点。渲染为：

```html
<span data-variable="email" class="variable-placeholder">{{email}}</span>
```

**2) ProseMirror Plugin 装饰器（主力方案）**：通过正则 `/\{\{([^}]+)\}\}/g` 扫描文档所有文本节点，对匹配到的 `{{...}}` 片段**动态添加 `.variable-highlight` class**，从而在视觉编辑器里把所有变量渲染为蓝底蓝字的代码块样式。这与 CSS 中的定义相呼应：

```css
/* EmailEditor.tsx L817-L835 */
.variable-highlight {
  background-color: #dbeafe;
  color: #1e40af;
  padding: 2px 6px;
  border-radius: 3px;
  font-family: 'Courier New', monospace;
}
```

这是一种**无侵入式方案**——变量仍然是纯文本 `{{email}}`，只是在显示时加了装饰层，序列化到 HTML 时不会产生多余的 DOM 结构。

### 3.3 `{{` 自动补全（VariableMention 扩展）

[VariableMention.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/VariableMention.ts) 基于 `@tiptap/extension-mention` 改造，但没有使用默认的 mention 节点渲染方式，而是：

- 触发字符：`{{`（默认的 Mention 是 `@`）
- 候选源：`id / email / unsubscribeUrl / subscribeUrl / manageUrl / locale` + `useContactFields()` 拉取的项目自定义字段（通过顶层 `availableVariables` 闭包变量传递）
- 按 query 前缀过滤，最多显示 10 项
- 选中后，`command` 会先删除触发用的 `{{`，然后**插入纯文本** `{{${props.id}}}`：
  ```typescript
  // VariableMention.ts L131-L135
  editor.chain().focus().deleteRange(range).insertContent(`{{${props.id}}}`).run();
  ```

UI 层用 **Tippy.js** 渲染一个跟随光标浮动的下拉面板，支持键盘上下箭头 + Enter 选中，滚动时通过 `sticky` 插件重定位。

### 3.4 Insert Variable 显式弹窗

除了 `{{` 隐式补全，[Toolbar.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/Toolbar.tsx) 的变量按钮打开一个显式 Modal（[EmailEditor.tsx L605-L695](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/EmailEditor.tsx#L605-L695)），提供三列：

1. **Common variables**（硬编码 5 个：id/email/unsubscribeUrl/subscribeUrl/manageUrl）
2. **Contact fields**（useContactFields 动态拉取）
3. **Custom variable**（任意名字 + 可选 default value，最终生成 `{{name ?? default}}`）

### 3.5 替换引擎 renderTemplate

[template.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/shared/src/template.ts) 中的 `renderTemplate` 是前后端**共享的纯函数**，通过一次正则替换完成所有变量：

```typescript
export function renderTemplate(template: string, variables: Record<string, unknown>): string {
  return template.replace(/\{\{(.*?)\}\}/g, (match, key) => {
    const [mainKey, defaultValue] = key.split('??').map(s => s.trim());

    // 三层查找策略
    const value =
      getValue(variables, mainKey) ||           // ① 作为嵌套路径：data.firstName
      variables[mainKey] ||                      // ② 作为顶层属性
      (variables.data)?.[mainKey];               // ③ 从 data 对象里兜底查找

    if (Array.isArray(value)) {
      return value.map(e => `<li>${e}</li>`).join('\n');  // 数组→列表项
    }
    return value ?? defaultValue ?? '';          // 三级回退：值 → defaultValue → 空串
  });
}
```

**注意**：这是一种「宽松查找」策略，三层 OR 意味着允许用户不写 `data.` 前缀也能访问联系人自定义字段，便于模板作者编写。`??` 是代码层面的字符串分割，不是 JS 语法。

---

## 四、HTML 保存与转换链路

### 4.1 Custom HTML 模式检测算法

[emailStyles.ts#L10-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/lib/emailStyles.ts#L10-L56) 的 `detectCustomHtmlPatterns` 是整个链路的关键判别函数，决定了 HTML 的「处理路径」。它检查 5 类特征，任意满足即判定为 Custom：

| 特征 | 正则/算法 | 理由 |
|------|-----------|------|
| 自定义 class | 提取所有 `class="..."`，逐个检查是否以 `prose / variable- / email-image / ProseMirror / resizable-image / selected / resize-handle` 开头，出现不在白名单中的 class 即判 Custom | Tiptap 只产生上述前缀的 class，其他 class 必然是用户手写的 |
| 自定义属性 | `/<[a-z][^>]*?[\s"'](?:data-\|aria-\|role=\|id=)/i` | Tiptap 不会给标签加 `data-*` / `aria-*` / `id` / `role` |
| 不可往返元素 | `/<(div\|section\|article\|...\|table\|tr\|td\|...\|form\|input\|iframe\|video\|...)\b/i` | 这组元素在当前 Tiptap 扩展集中没有对应 Node，解析会丢结构 |
| 媒体查询 | `/@media/i` | Tiptap 不会产出 `<style>` 里的媒体查询 |
| `<style>` 标签 | `/<style[^>]*>/i` | 同上 |

**重要**：`<span>` 和 `style="color:#xxx"` **不被视为 Custom**——因为 Color/TextStyle 扩展天然会产出 `<span style="...">`，这些能完美往返。

这个算法在 EmailService 中有一份**完全拷贝**（[EmailService.ts#L669-L701](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L669-L701) 的 `detectCustomHtmlPatterns` 私有方法），避免前后端判定不一致。

### 4.2 转换链路全景图

```
         ┌─────────────────────────────────────────────────────────────────────┐
         │                        前端（apps/web）                             │
         │                                                                     │
  用户输入 │  Visual 模式                         HTML 模式                      │
  ───────►│  Tiptap ProseMirror ◄──────────┐   CodeMirror ◄──── 用户粘贴 HTML    │
         │       │                         │       │                           │
         │       │ editor.getHTML()        │       │ onChange(rawHtml)          │
         │       ▼                         │       ▼                           │
         │   HTML 字符串（fragment）◄──────┘   htmlContent 状态                │
         │       │                                 │                           │
         │       └─────────────┬───────────────────┘                           │
         │                     ▼                                               │
         │             <EmailEditor onChange>                                  │
         │                     │                                               │
         │                     ▼                                               │
         │   create.tsx / [id].tsx → editedTemplate.body                       │
         │                     │                                               │
         │         [Preview 模式: 选中联系人后]                                │
         │                     │                                               │
         │                     ├─► renderTemplate() 变量替换                   │
         │                     ├─► wrapEmailWithStyles() 添加 prose 样式       │
         │                     └─► 注入 iframe 内 document 渲染预览             │
         │                                                                     │
         └──────────────────────────────────┬──────────────────────────────────┘
                                            │ PATCH /templates/:id 或 POST /templates
                                            ▼
         ┌─────────────────────────────────────────────────────────────────────┐
         │                       后端（apps/api）                              │
         │                                                                     │
  请求体   │  { name, subject, body: "<p>Hello {{email}}</p>", from, ... }      │
  ───────►│                                                                     │
         │  ① Zod Schema 校验（TemplateSchemas.create / update）                │
         │  ② DomainService.verifyEmailDomain() 校验发件域名所有权+验证状态    │
         │  ③ TemplateService.create/update() → Prisma 写入                    │
         │     └─ body 以 TEXT 原样存入 templates.body 字段                     │
         │                                                                     │
         │  ── 邮件发送时（campaign / workflow / transactional API）──        │
         │                                                                     │
         │  ④ EmailService.format()                                            │
         │        subject = renderTemplate(subject, contactData)               │
         │        body    = renderTemplate(body, contactData)                  │
         │  ⑤ EmailService.compile()                                           │
         │        ├─ detectCustomHtmlPatterns ?                                │
         │        │     content 原样 : wrapWithEmailStyles(content)           │
         │        ├─ 注入 i18n 本地化的 unsubscribe 表格 footer                │
         │        └─ 免费套餐注入 "Powered by Plunk" badge 表格                │
         │  ⑥ SES Service.sendRawEmail() 通过 AWS SES 发送                    │
         │                                                                     │
         └─────────────────────────────────────────────────────────────────────┘
```

### 4.3 wrapEmailWithStyles / wrapWithEmailStyles

**前端预览**（[emailStyles.ts#L58-L387](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/lib/emailStyles.ts#L58-L387)）和**后端发送**（[EmailService.ts#L708-L1033](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L708-L1033)）这两份 CSS **完全相同**——都是 Tailwind Typography 风格的 `prose prose-sm max-w-none` 基础样式，外加编辑器特定的 variable / resizable-image 样式。

关键设计：**Custom HTML 路径不包装**。如果检测到用户手写了完整的布局/样式，Plunk 就把 body 原封不动发送，由用户对渲染结果负责。

### 4.4 Landing 工具页的 juice 内联路径

官网工具页 [Markdown-to-Email](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/landing/src/lib/emailHtmlConverter.ts) 走了另一条路——使用 **juice** 库将 `<style>` 中的 CSS 全部转为 `style="..."` 内联属性（因为很多邮件客户端不支持 `<style>` 只支持内联样式）：

```typescript
// landing/src/lib/emailHtmlConverter.ts L120-L126
const inlined = juice(wrappedHtml, {
  preserveMediaQueries: false,
  preserveFontFaces: false,
  removeStyleTags: true,
  applyStyleTags: true,
});
// 然后提取 <body> 内容返回 email-friendly HTML
```

Dashboard 编辑器路径**当前不做 juice 内联**，依赖现代邮件客户端对 `<style>` 的支持。两条路径的差异是有意识的技术选择。

### 4.5 数据库中的存储格式

[schema.prisma#L159-L190](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/db/prisma/schema.prisma#L159-L190) 的 `Template` 模型：

```prisma
model Template {
  id          String       @id @default(uuid())
  name        String
  description String?
  subject     String                  // 标题行，也支持 {{variable}}
  body        String                  // HTML 片段（不是完整 <html> 文档）
  from        String                  // 发件邮箱
  fromName    String?
  replyTo     String?
  type        TemplateType @default(MARKETING)  // MARKETING / TRANSACTIONAL / HEADLESS
  projectId   String
  // ...
}
```

`body` 存储的是**编辑器输出的 HTML 片段**（Visual 模式是 Tiptap 序列化的 `<p>/<h1>/<a>/<img>...`，HTML 模式是用户写的任意内容），**不包含** `<html>/<head>/<body>` 包装，也**不执行**变量替换。模板 = 未烘烤的生面团。

---

## 五、服务端模板接口：CRUD + 校验

### 5.1 接口一览

[Templates.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Templates.ts) 共 7 个端点，全部挂在 `@Controller('templates')` 下，并使用 `@Middleware([requireAuth, requireEmailVerified])` 做项目级鉴权 + 用户邮箱已验证拦截。

| Method | Path | 行为 | 校验 |
|--------|------|------|------|
| GET | `/templates` | 分页列表 + 搜索 + 按类型过滤 | auth |
| GET | `/templates/:id` | 取单条 | auth + 归属项目（TemplateService.get 会校验 projectId） |
| POST | `/templates` | 创建 | name/subject/body/from 必填（**Controller 内手写校验**）+ 域名校验 |
| PATCH | `/templates/:id` | 更新 | 域名校验（仅当 from 变更时） |
| DELETE | `/templates/:id` | 删除 | 被 workflow step 引用则 409 Conflict（见 5.3） |
| POST | `/templates/:id/duplicate` | 复制（name 加 " (Copy)" 后缀） | auth |
| GET | `/templates/:id/usage` | 统计：workflow 引用数 + 已发送邮件数 | auth |

### 5.2 两级校验：Controller 手写 + Zod Schema

**Controller 层手动校验**（[Templates.ts L61-L76](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/controllers/Templates.ts#L61-L76)）：

```typescript
if (!name)    return res.status(400).json({error: 'Name is required'});
if (!subject) return res.status(400).json({error: 'Subject is required'});
if (!body)    return res.status(400).json({error: 'Body is required'});
if (!from)    return res.status(400).json({error: 'From address is required'});
```

前端也有一层对应的本地校验（[validation.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/lib/validation.ts) 的 `EmailFormValidator.validateTemplate`），拦截空值避免 HTTP 请求浪费。

**共享 Zod Schema**（[schemas/index.ts#L185-L206](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/shared/src/schemas/index.ts#L185-L206)）——注意 Template Controller 实际上**没有显式调用** Zod parse（Campaign Controller 用了），但前端 `network.fetch()` 在发出请求前会用这些 schema 做**客户端类型约束**（TemplateSchemas.create / update 作为泛型参数传入）：

```typescript
TemplateSchemas.create = z.object({
  name: z.string().min(1).max(100),
  subject: z.string().min(1),
  body: z.string().min(1),
  from: z.string().email(),         // 这里的 email() 格式校验比 Controller 的非空检查更严格
  fromName: z.string().max(100).nullish(),
  replyTo: z.string().email().nullish(),
  type: z.nativeEnum(TemplateType).default('MARKETING'),
});
```

### 5.3 删除时的引用完整性

[TemplateService.ts#L138-L160](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/TemplateService.ts#L138-L160) 的 `delete` 方法：

```typescript
const workflowSteps = await prisma.workflowStep.count({
  where: { templateId, workflow: { projectId } },
});
if (workflowSteps > 0) {
  throw new HttpException(409,
    'Cannot delete template: it is currently used in workflow steps. Remove it from workflows first.'
  );
}
```

这是一种**数据库前置约束**，等价于「软外键检查」。注意 Prisma 中 `WorkflowStep.templateId` 是可空的外键，所以 DB 本身不会阻止删除，必须靠业务代码保护。

### 5.4 发件域名校验

[DomainService.verifyEmailDomain](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/DomainService.ts) 会在创建/更新模板（以及 campaign / send-email 动作）时检查：

1. from 邮箱的域名部分必须已在当前 project 下注册
2. 该域名的 `verified === true`（DKIM DNS 记录已通过）

失败则抛出 `HttpException`，返回给前端 400 错误并展示在 toast 中。

---

## 六、前后协作完整流程

### 6.1 创建模板（create.tsx）

```
[用户填写表单 + 编辑 body]
        │
        ▼
handleSubmit()
  │
  ├─► EmailFormValidator.validateTemplate()        // 前端非空校验（name/subject/body/from）
  │      ├─ 失败 → toast.error('Name is required') 并 return
  │
  ├─► network.fetch('POST', '/templates', payload)
  │      │
  │      ├─► TemplateSchemas.create 类型检查        // 泛型参数约束
  │      │
  │      └─► HTTP 请求
  │             │
  │             └─► Templates.create()
  │                    ├─ requireAuth + requireEmailVerified
  │                    ├─ 手写非空校验（400）
  │                    ├─ DomainService.verifyEmailDomain(from)
  │                    │      └─ 失败 → 抛 HttpException
  │                    └─ TemplateService.create() → prisma.template.create
  │
  ├─ 成功 → toast.success + router.push(`/templates/${id}`)
  └─ 失败 → toast.error(message)
```

### 6.2 编辑模板（[id].tsx）

与创建不同点：
1. 用 `useSWR('/templates/${id}')` 拉取现有数据填充表单
2. `useMemo` 计算 `hasChanges` 标记脏状态 → [useChangeTracking](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/lib/hooks/useChangeTracking.ts) 拦截浏览器 beforeunload + Next.js routeChangeStart，弹出「你有未保存的更改」
3. 底部有 StickySaveBar，滚动时也能看到「保存/有未保存/已保存」状态
4. PATCH 请求，`network.fetch()` 使用 `TemplateSchemas.update` 作约束
5. 保存成功后调用 `mutate()` 静默更新 SWR 缓存，不弹 toast

### 6.3 预览机制

EmailEditor 内部集成了完整预览：

1. 用户从顶部 `Preview as` 下拉选择一个联系人（取前 50 个）
2. `getPreviewHtml()` 组装 contactData 对象（[EmailEditor.tsx#L247-L255](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/EmailEditor/EmailEditor.tsx#L247-L255)）：
   ```typescript
   const contactData = {
     email, unsubscribed,
     unsubscribeUrl: `${origin}/unsubscribe/${id}`,
     subscribeUrl, manageUrl,
     data: contact.data,
     ...contact.data,          // 扁平化，以便 renderTemplate 第 ②③ 层查找
   };
   ```
3. `replaceVariables()` → 调用共享 `renderTemplate(text, contactData)`
4. `wrapEmailWithStyles()` → 把 HTML 片段包装成完整文档
5. 直接 `iframeDoc.write(fullHtml)` 注入沙盒 iframe
6. 通过 `setTimeout(adjustHeight, 100/300)` + `load` 事件三段式调整 iframe 高度，消除高度抖动
7. 设备预览按钮（375/768/1200px）仅调整容器宽度触发 CSS 响应式，不重绘 DOM

Subject 行也经过**同样的** `renderTemplate` 替换，保证预览 Subject 与实际发送一致。

### 6.4 工作流 Send Email 步骤

[SendEmailStepDialog.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/web/src/components/workflow-steps/SendEmailStepDialog.tsx) 展示了模板在 workflow 中的使用方式：

1. workflow step 只存储 `templateId`，不存 body 副本
2. 运行时 WorkflowExecutionService 从模板表读取最新内容
3. 收件人类型可选 `CONTACT`（触发 workflow 的人）或 `CUSTOM`（硬编码邮箱）
4. 模板可随时编辑，下一次 workflow 触发时立即生效，无需重新配置 step

---

## 七、校验失败与异常场景全景

| 场景 | 在哪一层拦截 | 表现 |
|------|-------------|------|
| **name / subject / body / from 为空** | 前端 EmailFormValidator + 后端 Controller 双重 | 前端 toast 红条；后端返回 400 JSON |
| **from 邮箱格式非法** | 共享 Zod Schema `.email()` + network 类型层 | 类型检查提前拦截；若绕过则后端域名校验阶段抛 |
| **from 域名未在项目注册** | 后端 DomainService.verifyEmailDomain | 400 Error，toast 显示具体原因 |
| **from 域名 DNS 未验证通过** | 同上 | 同上 |
| **replyTo 邮箱格式非法** | Zod `.email().nullish()` | 类型拦截 |
| **name / fromName 超长（>100）** | Zod `.max(100)` | 类型拦截 |
| **删除模板时被 workflow step 引用** | TemplateService.delete | 409 Conflict，提示先从 workflow 移除 |
| **HEADLESS 模板无退订信号** | 前端 detectUnsubscribeSignal + 黄色警告框（不阻塞） | UI 提示用户「你需要自行提供退订链接」，给出 `{{unsubscribeUrl}}` 等变量 |
| **营销模板发送给退订联系人** | EmailService.sendWorkflowEmail / sendTransactionalEmail | workflow 中：静默跳过 + Email 记录 FAILED 状态 + error 字段；transactional API：直接抛 400 异常 |
| **Billing 超限** | BillingLimitService.checkLimit | 429 Too Many Requests，带 message |
| **自定义 HTML → Visual 模式切换** | 前端 detectCustomHtmlPatterns 弹窗 | 二次确认，防止 Tiptap 解析破坏结构 |
| **变量在渲染时找不到值** | renderTemplate 三级回退 | 返回空串 `''` 或 fallback 默认值，不报错（这是有意的静默失败设计） |
| **邮件实际发送时 SES 出错** | catch (error) in sendEmail | Email 记录标记 FAILED，error 字段写堆栈，事件不追踪 |
| **退订联系人的营销邮件最终兜底** | EmailService.sendEmail() 发送前最终检查 | 即使通过上层也可能被拦截，保证合规 |

---

## 八、展示一致性保障机制

为了「编辑器里看到的效果」与「用户邮箱里收到的效果」尽可能一致，系统做了多层保障：

### 8.1 前后端 wrap 样式完全一致

`apps/web/src/lib/emailStyles.ts` 里的 `wrapEmailWithStyles()` 和 `apps/api/src/services/EmailService.ts` 里的 `wrapWithEmailStyles()` **一字不差**（目测拷贝约 330 行的 `<style>` 块）。这保证了预览 iframe 里看到的排版与最终邮件里 prose 样式一致。

### 8.2 Custom HTML 分支「零改造」策略

如果 `detectCustomHtmlPatterns` 为真，两端都跳过包装，**完全信任用户代码**。不会出现「前端预览有自定义样式 → 后端发送时又套了 prose 导致样式冲突」的情况。

### 8.3 变量替换同一函数

Subject 和 Body 的变量替换，前端预览调用 `renderTemplate()`，后端发送也调用同一个 `renderTemplate()`（都来自 `@plunk/shared`）。替换顺序、三层查找策略、默认值处理完全一致。

### 8.4 Subject 行同步替换

预览顶部的 Subject 行**不是**直接显示原始模板字符串，而是经过 `getPreviewSubject()` 同样的变量替换，确保「标题中变量值是否正确」也能被验证。

### 8.5 三档设备尺寸预览

通过容器宽度调整为 375 / 768 / 1200px，用户可以直接在编辑器内检验响应式表现，而不是发送测试邮件。

### 8.6 数组值特殊渲染

`renderTemplate` 对数组值会自动包 `<li>`，但只有在用户**显式**把数组放进 contact.data 才触发——普通字符串/数字走正常文本替换，不会引入意外的 HTML 标签。

### 8.7 一致性例外（有意识的技术债务）

- **Landing 工具页使用 juice 内联，Dashboard 发送路径不使用**：两条路径的邮件客户端兼容性可能有差异。可能的原因是 juice 会使 HTML 体积膨胀，Dashboard 路径更偏向「代码整洁」优先，而工具页偏向「邮件客户端兼容性」优先。
- **退订脚注不参与前端预览**：EmailService.compile() 注入的 unsubscribe footer 与 Plunk badge，在 EmailEditor 的预览里看不到——用户必须发送测试邮件才能最终校验脚注效果。这是预览与真实发送之间**最显著的差异点**。
- **邮件客户端差异不可控**：Outlook/Apple Mail/Gmail 对 CSS 支持差异很大，ProseMirror 样式虽然以 email-safe 原则编写，但不能保证所有客户端的像素级一致。

---

## 九、Template 类型语义（MARKETING vs TRANSACTIONAL vs HEADLESS）

模板的 `type` 字段驱动了多个分支判断：

| 类型 | 退订检查 | 自动注入退订脚注 | 自动注入 Plunk Badge | 典型用途 |
|------|---------|-----------------|---------------------|---------|
| MARKETING | ✅ 发送前检查 subscribed，退订用户拒收 | ✅ | ✅（免费版） | 营销邮件、周报、公告 |
| TRANSACTIONAL | ❌ 不管订阅状态一律可发 | ❌ | ✅（免费版） | 密码重置、订单通知 |
| HEADLESS | ✅ 同 MARKETING | ❌（用户自行提供） | ✅（免费版） | 想要自定义脚注样式的营销邮件 |

三处核心判断：
1. [EmailService.sendTransactionalEmail L62](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L62) —— 用营销模板时即使走 transactional API 也必须对方订阅
2. [EmailService.sendWorkflowEmail L203](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L203) —— workflow 中 TRANSACTIONAL 模板绕过 subscribed 检查
3. [EmailService.compile L364-L367](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/apps/api/src/services/EmailService.ts#L364-L367) —— 是否注入退订脚注

配合前端 [unsubscribe.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/56-plunk/packages/shared/src/unsubscribe.ts) 的 `detectUnsubscribeSignal()`，对 HEADLESS 类型做「建议性警告」而非强制性报错，兼顾合规与灵活度。

---

## 十、关键洞察总结

1. **双编辑器策略**是应对「邮件模板既要易用又要灵活」的经典解法：普通用户用 Visual，高级用户切 HTML。模式切换时的 Custom 检测是关键保护措施。

2. **变量系统采用「纯文本 + 装饰层」而非自定义 Node**，保证序列化结果就是最干净的 `{{var}}`，后端替换完全无感知。这种 Text + Decoration 的组合在 Tiptap 里比自定义 Atom 节点更健壮。

3. **三层变量查找策略**（嵌套路径 → 顶层 → data 对象）虽然不是最高效，但极大降低了模板作者的心智负担——不用关心字段属于顶层还是 data 内部。

4. **`detectCustomHtmlPatterns` 是整个数据流的中枢判别函数**，前端决定初始模式、切换警告、预览样式；后端决定是否包装 prose 样式。它在前后端各有一份拷贝，这是一致性的基石，也是日后修改时最容易「只改一端」而出 bug 的地方。

5. **模板与发送解耦**：模板只存 HTML 片段 + 变量占位，真正的「烘烤」发生在发送瞬间的 `format()` + `compile()` 两步。这允许：a) 同一模板发送给 1000 人各有不同的变量值；b) 事后修改模板立即生效，无需同步 workflow 配置。

6. **合规性保护**分布在多层：发件域名校验（接口层）、订阅状态校验（发送服务）、HEADLESS 退订信号提示（编辑期提醒）——形成「越提前拦截成本越低」的递进防护。
