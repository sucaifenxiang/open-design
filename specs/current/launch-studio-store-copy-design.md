# Launch Studio 商店文案（Store Copy）第一阶段设计规格

## 文档状态

- 日期：2026-08-05
- 状态：设计规格草稿，进入实施计划
- 产品：Launch Studio
- 依据：`doc/Launch Studio PRD v1.md`（四、应用商店文案生成；七、文案编辑器；八、品牌语气系统）、`doc/Launch Studio PRD v2.md`（Content Studio）、`doc/Launch Studio 技术方案.md`（10.3 Copy Document、13 文案生成技术方案）
- 上游基线：与商店截图第一阶段相同（`nexu-io/open-design` `main`，提交 `f52fda29` 之后已合入商店截图第一阶段）
- 目标平台：macOS 和 Windows 桌面端（沿用 Open Design 桌面 + Daemon + Web 架构）

## 1. 目标

在 Launch Studio 现有产品项目、品牌体系与商店截图基建之上，交付一条完整的 App Store / Google Play 英文商店文案工作流：

- 根据 Product Profile 一次生成当前平台的完整文案字段；
- 每个字段可独立编辑、重新生成，重新生成不覆盖已确认或已锁定的字段；
- 字符数与平台限制由确定性程序实时计算并展示；
- 文案读取当前 Brand Profile 语气，执行确定性合规检查（禁用词、绝对化表述、关键词堆砌）；
- 字段级锁定保护产品名、关键词与专有名词；
- 截图标题与完整介绍保持一致（字段可绑定截图）；
- 版本快照与恢复；
- 单语言导出（JSON / Markdown / ZIP），导出结构预留多语言扩展；
- UI、HTTP API、`od` CLI 三端能力对等。

## 2. 产品与界面方向

在 Open Design 的左右分栏中，右侧 `FileWorkspace` 提供商店文案工作台：

```text
Store Copy Workspace
├── 平台标签（App Store / Google Play）
├── 字段列表（按平台 Schema 固定顺序）
│   ├── 字段名 + 当前值
│   ├── 字符数 / 平台上限（超限红色提示）
│   ├── 锁定开关（锁定的字段显示 🔒）
│   └── 操作：重新生成 / 确认 / 恢复上一版本
├── 底部操作栏
│   ├── AI 生成整套（CopyPlan 预览 → 应用）
│   ├── 合规检查结果（错误 / 警告）
│   └── 导出（JSON / Markdown / ZIP）
└── 版本历史侧栏（快照对比与恢复）
```

交互原则沿用商店截图第一阶段：

- AI 只生成结构化 `StoreCopyPlan`，经 ChangeSet 校验后应用；
- 锁定字段在任何生成、改写、应用路径中都不被修改；
- 每个字段的修改都产生可回滚的版本快照；
- 无 Provider 时手工编辑、合规检查、锁定与导出完整可用。

## 3. 范围

### 3.1 第一阶段包含

- 平台：App Store（iOS）、Google Play（Android）。
- 语言：`en-US` 生成与校验；导出结构支持多语言目录。
- App Store 字段：`name`、`subtitle`、`promotionalText`、`description`、`keywords`、`whatsNew`、`screenshotTitle[1..10]`。
- Google Play 字段：`name`、`shortDescription`、`fullDescription`、`featureGraphicText`、`releaseNotes`、`tags`、`screenshotTitle[1..8]`。
- 字段锁定、确认与逐字段重新生成。
- 确定性字符限制与平台合规检查。
- 品牌语气：读取 Brand Profile 的 voice 规则（禁用表达、标点/Emoji 规则、CTA 风格）做确定性检查；AI 改写必须携带 voice 上下文。
- 版本快照与恢复（沿用项目目录 `store-copy/versions/` 约定）。
- 导出：`store-copy/exports/<job-id>/` 下 manifest + `en-US.json` + `en-US.md` + ZIP。
- 三端：HTTP API、Web 工作台、`od store-copy` CLI。

### 3.2 第一阶段不包含

- 多语言 AI 本地化生成（schema 与导出目录预留 `locale`，但 AI 翻译延后到第二阶段）。
- In-App Purchase、活动推广、Mac App Store / Microsoft Store / Chrome Web Store 等扩展字段与平台。
- Product Hunt / X / LinkedIn / Reddit / Instagram / Email / 官网等非商店平台文案。
- 完整 AI 品牌一致性评分（仅确定性规则 + 锁定保护；AI 语义评分延后）。
- 与商店截图素材的双向智能映射（第一阶段只提供截图标题字段，不做自动配图绑定）。
- 自动上传商店、发布与计划任务。

## 4. 平台输出目标

| 平台 | 字段 | 上限（确定性计算） |
| --- | --- | --- |
| App Store | Name | 30 |
| App Store | Subtitle | 30 |
| App Store | Promotional Text | 170 |
| App Store | Description | 4000 |
| App Store | Keywords | 100（逗号分隔，含逗号） |
| App Store | What's New | 170 |
| App Store | Screenshot Title | 40（本阶段约定上限） |
| Google Play | App Name | 50 |
| Google Play | Short Description | 80 |
| Google Play | Full Description | 4000 |
| Google Play | Feature Graphic 文案 | 80（本阶段约定上限） |
| Google Play | Release Notes | 500（本阶段约定上限） |
| Google Play | Tags | 每个标签 30，最多 5 个（本阶段约定） |
| Google Play | Screenshot Title | 40（本阶段约定上限） |

字符数按 Unicode code point 计算（`Array.from(value).length`），换行计入；确定性程序计算，不由模型判断。

## 5. 总体架构

```text
packages/store-copy
  ├── src/schema.ts        领域 Schema 与类型（Zod）
  ├── src/platforms.ts     平台规格：字段、上限、必填、数量规则
  ├── src/compliance.ts    确定性合规检查（长度/必填/关键词/禁用表达）
  ├── src/changeset.ts     ChangeSet 校验与应用（锁定保护）
  ├── src/derive.ts        平台规格 → 空文档 / 模板初始文档
  └── src/export.ts        文档 → JSON / Markdown / manifest 渲染

apps/daemon/src/store-copy
  ├── persistence.ts       SQLite 索引 + 项目目录快照（store-copy/document.json + versions/）
  ├── planner.ts           AI StoreCopyPlan 生成（结构化 JSON，复用现有 provider）
  ├── service.ts           业务编排（生成/应用/导出/版本）
  ├── routes.ts            HTTP 路由
  └── cli.ts               od store-copy 子命令

apps/web/src/features/store-copy
  ├── api.ts
  ├── StoreCopyWorkspace.tsx
  ├── CopyFieldEditor.tsx
  ├── CopyPlanReview.tsx
  ├── CompliancePanel.tsx
  └── version-history.tsx（复用商店截图 VersionHistory 模式）
```

依赖方向与商店截图一致：纯领域包不依赖浏览器、Electron、SQLite、文件系统；daemon 是唯一持久化与 AI 编排方；web 与 CLI 只消费 HTTP API。

## 6. 规范业务文档

`StoreCopyDocument` 是唯一事实来源，存放于项目目录：

```text
store-copy/
  document.json
  versions/000001.json
  exports/<job-id>/
    manifest.json
    en-US.json
    en-US.md
    launch-studio-store-copy.zip
```

### 6.1 文档 Schema

```ts
interface StoreCopyDocument {
  id: string;
  projectId: string;
  platform: 'app-store' | 'google-play';
  locale: string;          // 第一阶段固定 'en-US'
  strategy: CopyStrategy;  // 文案策略 id
  fields: Record<string, StoreCopyField>;
  updatedAt: string;
  version: number;         // 当前快照序号
}

interface StoreCopyField {
  id: string;
  value: string;
  locked: boolean;         // 锁定字段 AI 不可改
  confirmed: boolean;      // 用户确认过
  source: 'initial' | 'generated' | 'edited' | 'restored';
}
```

字段 Schema 随平台规格派生（`app-store` 字段集 vs `google-play` 字段集），缺失字段按必填/可选规则校验。

### 6.2 文案策略

`CopyStrategy` 枚举（PRD 4.4）：

```text
feature-focused | value-focused | efficiency-focused |
problem-solving | professional | minimal | storytelling | conversion
```

文档记录当前策略；重新生成整套时用户可切换策略，生成器把策略写入 `strategy` 并在 plan 中使用对应提示词。

### 6.3 锁定语义

- 锁定只对 AI 生效：`locked: true` 的字段在生成、改写、应用 ChangeSet 时一律跳过；
- 用户手工编辑锁定字段始终允许（锁定约束的是“自动修改”，不是“人工修改”）；
- `od store-copy lock <field>` 与 UI 开关等价；
- 导出与合规检查不受锁定影响。

## 7. AI 生成

### 7.1 StoreCopyPlan

生成器返回结构化 `StoreCopyPlan`（复用现有结构化 JSON provider 通道，与 ScreenshotPlan 同一模式）：

```ts
interface StoreCopyPlan {
  strategy: CopyStrategy;
  fields: Record<string, { value: string; rationale: string }>;
}
```

plan 只含生成目标平台的字段；必填字段缺失视为校验失败；任意字段超长由合规检查在应用前拦截。

### 7.2 应用语义

```text
plan → 校验（Schema + 平台规格 + 合规）→ 生成 ChangeSet（只含非锁定字段的差异）
      → 应用 → 新版本快照 → 返回变更摘要
```

应用后返回每个字段的 `before/after`，供 UI 预览；与截图工作台相同，用户确认前不落最终文档。

### 7.3 降级路径

- 无 Provider：整套生成不可用，手工编辑、合规、锁定、导出完整可用；
- Provider 失败：返回精确错误（沿用 `PROVIDER_NOT_CONFIGURED` / provider 错误透传），不写半成品；
- 单个字段超限：整 plan 拒绝（避免部分应用），错误信息列出超限字段与字符数。

## 8. 编辑与锁定

- 每个字段独立编辑（UI 输入框 / CLI `od store-copy patch --field <id> --value <text>`）；
- 编辑写入后立即重算字符数并展示合规状态；
- 字段可 `confirm`（标记已确认）、`lock`（锁定）、`unlock`；
- 版本快照在每次字段变更、AI 应用、恢复后自动产生（节流：同一秒内连续编辑合并为一个快照）。

## 9. 合规检查（确定性）

`compliance.ts` 输出 `ComplianceReport`：

```text
[
  { severity: 'error', field: 'description', code: 'EXCEEDS_LIMIT', message: '4100/4000' },
  { severity: 'error', field: 'keywords',   code: 'KEYWORD_TOO_LONG', ... },
  { severity: 'warning', field: 'description', code: 'FORBIDDEN_TERM', term: 'revolutionary' },
  { severity: 'warning', field: 'description', code: 'ABSOLUTE_CLAIM', term: 'guaranteed' },
  { severity: 'warning', field: 'name', code: 'LOCKED_FIELD_EMPTY', ... }
]
```

规则集：

- 必填字段非空；
- 字符数不超过平台上限；
- Keywords：逗号分隔去重、每词 ≤ 30、总长 ≤ 100、不含品牌词（Product Profile 中的品牌词排除）；
- 禁用表达：Brand Profile voice 的 `forbiddenPhrases` 命中即 warning；
- 绝对化承诺：内置 deny-list（guaranteed / best in the world / revolutionary / completely effortless 等）+ Brand Profile 追加词；
- 空锁定字段：locked 且空 → warning（提示用户锁定的字段没有内容）；
- 重复内容：description 与 shortDescription 显著重复（≥80% 相同）→ warning。

错误（error）阻塞导出；警告（warning）不阻塞但写入 manifest。

## 10. 持久化与版本管理

- `store-copy/document.json`：当前文档；
- `store-copy/versions/<seq>.json`：只读快照，`<seq>` 从 1 递增；
- SQLite 表 `store_copy_documents`：`(id, project_id, platform, locale, updated_at)` 索引 + `store_copy_versions` 计数，与商店截图同一 db；
- 恢复 = 从快照生成新版本（不破坏历史），恢复后 `source: 'restored'`；
- 损坏的 document.json：读取失败时提供最近快照提示，不静默覆盖。

## 11. API 与 CLI

路由前缀：`/api/projects/:projectId/store-copy`

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/documents` | 列出平台文档 |
| POST | `/documents` | 创建平台文档（缺省策略） |
| GET | `/documents/:platform` | 读取文档 |
| POST | `/documents/:platform/generate` | AI 生成 plan（长 prompt 走 `--prompt-file`） |
| POST | `/documents/:platform/apply` | 应用 plan / ChangeSet |
| PATCH | `/documents/:platform/fields/:field` | 编辑 / 锁定 / 确认单个字段 |
| GET | `/documents/:platform/versions` | 版本列表 |
| POST | `/documents/:platform/restore` | 恢复到指定版本 |
| POST | `/documents/:platform/export` | 导出（返回 job id / 下载 URL） |
| GET | `/documents/:platform/compliance` | 合规检查结果 |

CLI `od store-copy` 子命令（`SUBCOMMAND_MAP` 注册，`--json` 输出）：

```text
od store-copy create <platform> [--strategy <id>]
od store-copy get <platform>
od store-copy generate <platform> [--prompt-file <path|->]
od store-copy apply <platform> --plan-file <path|->        # 或管道 stdin
od store-copy patch <platform> --field <id> --value <text> [--lock|--unlock|--confirm]
od store-copy compliance <platform>
od store-copy versions <platform>
od store-copy restore <platform> --version <n>
od store-copy export <platform>
```

所有写操作返回变更摘要；所有读操作支持 `--json`。

## 12. 导出结构

```text
exports/<job-id>/
  manifest.json
  en-US/
    app-store.json
    google-play.json
    app-store.md
    google-play.md
  launch-studio-store-copy.zip
```

manifest 记录：`jobId`、`platforms`、`locale`、`exportedAt`、`fields`（每平台字段数）、`compliance`（错误/警告计数）、`hashes`（每个文件的 SHA-256）。

Markdown 渲染按平台字段顺序输出标题 + 内容 + 字符数 + 合规状态。

## 13. 测试方案

### 13.1 领域包（packages/store-copy）

- schema：文档/字段/plan 校验，锁定语义；
- platforms：字段集与上限常量，数量规则；
- changeset：生成差异、锁定字段跳过、无效 plan 拒绝；
- compliance：全部规则与错误/警告分类；
- export：JSON/Markdown/manifest 确定性渲染与 hash。

### 13.2 daemon

- persistence：文档/版本/索引、损坏恢复；
- service：生成→应用→恢复→导出的编排与错误路径；
- routes：HTTP 契约（成功/拒绝/400）；
- CLI：stub server 验证 `od store-copy` 各子命令的请求/响应（沿用 cli-templates 测试模式）。

### 13.3 web

- api.ts 请求契约；
- StoreCopyWorkspace 渲染与交互（字段编辑、字符数、锁定、合规面板、导出）；
- CopyPlanReview 应用/拒绝；
- 版本历史恢复。

### 13.4 e2e

- 手工路径：创建 → 编辑 → 锁定 → 合规 → 导出；
- AI 路径（Provider 可用时）：生成 → 预览 → 应用 → 锁定字段不变 → 恢复。

## 14. 验收标准

- 从新建入口可创建 App Store / Google Play 文案文档；
- 一次生成完整字段套件（AI 路径）或手工填写（无 Provider 路径）；
- 每个字段可独立编辑与重新生成；重新生成不覆盖已确认/锁定字段；
- 字符数与平台限制实时显示，超限标红；
- 合规检查结果可读，错误阻塞导出；
- 锁定字段在 AI 生成、应用、恢复任何路径下保持不变；
- 版本快照可列出、对比、恢复；
- 导出 JSON/Markdown/ZIP，manifest hash 与实际文件一致；
- HTTP、UI、CLI 三端能力对等；
- 单元、契约、集成、e2e 测试通过。

## 15. 参考资料

- `doc/Launch Studio PRD v1.md` 四/七/八
- `doc/Launch Studio PRD v2.md` 二/三/十四/十五/十六
- `doc/Launch Studio 技术方案.md` 10.3/13
- `specs/current/launch-studio-store-screenshot-design.md`（架构与三端模式复用）
