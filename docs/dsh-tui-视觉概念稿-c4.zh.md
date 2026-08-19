# DSH TUI 视觉概念稿 — C4

[English](dsh-tui-视觉概念稿-c4.md) | 中文

> **Tag**: C4（DSH TUI 系列第 4 篇；C1 对比 Claude Code、C2 对比 grok-build、C3 增强方案、next-phase 功能演进） **Date**: 2026-08-12 **目的**: 面向用户可见部分（首屏欢迎页、输入框、主界面布局、状态区）产出 3 个概念设计方案，供选型。 **参考实现**: grok-build `crates/codegen/xai-grok-pager`（本地 `~/checkouts/grok-build`，版本 b13fa52）、claude-code-haha（本地 `~/checkouts/claude-code-haha`，Claude Code CLI 的 React/Ink 重实现）。 **状态**: 概念设计文档，未实施。所有对参考项目的描述为源码阅读结论（未运行验证）；对本仓库的描述基于当前工作树。

## 0. 设计约束（从现状继承）

概念稿必须落在 dsh TUI 既有架构内，不打破以下不变量：

- **渲染三层单向依赖**：`ui/app.ts`（装配）→ `render/ + format/`（纯函数，快照→ANSI 行）→ `engine/`（CommitEngine append-only scrollback / LiveEngine 底部 live 区增量重绘）。新视觉只落在 format/ 纯函数层与 app.ts 组装层（tui-controllers.md、projection-layer.md 的纪律）。
- **滚动转录区不可擦除**：已提交内容不能重绘（commit-engine.ts）。流式内容只能走 live 区或稳定前缀提交。
- **纯函数可测**：任何新视觉输入必须是窄输入（width/modelName/cwd/…），输出 ANSI 行数组，无 IO 无时钟。
- **宽度守恒**：任何输入下每行显示宽度 ≤ 终端宽度（welcome.ts 已示范）。
- **主题系统**：所有颜色走 `RivetTheme`（theme.ts），不引入硬编码色值。
- **legacy 终端降级**：`term-caps.ts` 的 ASCII 字形集（ui-glyphs.ts 已示范双档字形）。
- **现有能力不重复**：C1/C2/C3 已落地的能力（slash 命令、steer、@补全、外部编辑器、vim、工具家族色/计时、并行折叠、审批 diff、历史搜索 Ctrl+F、模型热切换、/fork、plan/auto 模式、7 面板、btw 侧问、会话 tab、主题定制、fluency 控制）是概念稿的**底座**而非新设计。

### 现有可见部分快照（2026-08-12 工作树）

| 区域 | 现状 | 文件 |
|---|---|---|
| 欢迎页 | 4 行迷你框（╭─ DSH ─ ╯）+ 环境检查行（renderWelcomeEnvCheck：API Key / Git / 终端） | `src/format/welcome.ts` |
| 输入框 | 单行输入 + 前缀提示 + vim 模式标签；Tab 补全 / @mention / 图片附件（[Image #N] chip） | `src/engine/input-line.ts` |
| 消息区 | markdown 渲染 + 工具折叠卡（家族色/计时/截断 marker）+ 并行组折叠 | `src/format/markdown.ts`、`tool-card.ts` |
| 底部 live 区 | glance 状态行 + 7 面板（tasks/status/delegation/workflow/config/skills）+ 会话 tab + btw 侧问 + 审批/提问行 + 流式尾巴 | `src/render/live-panels.ts`、`src/ui/app.ts` renderLive |
| 状态行 | idle/running + [plan]/[plan…] badge + 权限 preset/policy + 会话 id | `src/statusline.ts` |

### 参考项目提取的关键设计语言（侦察摘要，附证据）

**grok-build 欢迎页**（`views/welcome/mod.rs` L679-700 骨架；L263-347 双布局）：
- 垂直骨架：顶边距 1 → 顶部栏 1 行（分支 + tilde 折叠 cwd，`top_bar.rs`）→ 内容区垂直居中 → 底边距 1。
- 双布局二选一：**hero box**（宽 ≥90、非 compact、高足够）圆角边框内左 logo 右 version/subtitle/info/menu（`hero_box.rs` L14 HERO_BOX_MIN_WIDTH=90）；否则 **stacked** 堆叠：top_pad → logo → error → menu → changelog → flex gap → tip → prompt → version。
- Braille 点阵 logo，高度三档（<22 无 logo、22-25 小号、≥26 全尺寸，`logo.rs` L17-26）；shimmer 动画 12fps（可选装饰）。
- 菜单行：label 左对齐 BOLD + 快捷键右对齐 gray_bright，选中整行 bg_highlight（`menu.rs` L30-63）。菜单项：ctrl+w New worktree / ctrl+s Resume session / ctrl+q Quit / ctrl+i Import Claude settings。
- prompt 复用共享 PromptWidget，placeholder "Type a message..."，固定 3 行，水平 inset 2 列（`welcome/prompt.rs` L35-51）。
- toast：单行 `[ msg ]`，prompt 上方右对齐（`toast.rs` L17-62）。
- 退出键按终端品牌自适应（VS Code/xterm.js 家族 ctrl+d，否则 ctrl+q，`mod.rs` L54-60）。

**grok-build 主界面**（`views/agent.rs` AgentViewLayout L103-250；`app/agent_view/render.rs`）：
- 垂直区域：状态栏(1) → 启动警告 → 面板区(tasks/catalog/todo/queue) → scrollback → btw → turn_status（braille spinner + 阶段标签 Thinking…/Responding；等待输入时 pulsing ◆）→ banner → follow_ups → voice → prompt → shortcuts。
- 消息按 RenderBlock 分块：UserPrompt/AgentMessage/ToolCall(Execute/Read/Edit/Search…)/Thinking/System/BgTask/Subagent/Workflow（`scrollback/block.rs` L371-405）。
- prompt 形态由 PromptStyle 控制（focused/compact/chrome/prefix_override/placeholder/image_preview），footer 快捷键随模式变化（plan-approval：Enter save / Esc cancel / a approve / Tab plan，`render.rs` L926-1028）。
- 窄终端：SHORT_TERMINAL_ROWS=16 硬降级、AUTO_COMPACT_MAX_ROWS=20 自动 compact（`views/agent.rs` L95-119）。

**claude-code-haha 输入框**（`components/PromptInput/`）：
- 前缀符号 ❯（默认）/ `!`（bash 模式），loading 时 dim（`PromptInputModeIndicator.tsx` L63-93）。
- 多行输入；全屏 maxVisibleLines = max(3, floor(rows/2)-5)；图片附件 [Image #N] chip 与文本双向绑定（`PromptInput.tsx` L1999/L2170）。
- 边框 round 只画底边，颜色随模式（bash→bashBorder、默认 promptBorder）（L2244-2271）。
- footer 左右分栏（<80 列纵排）：左侧权限模式（symbol+名称+on）+ 快捷键 + tasks pill；右侧 Notifications（token 用量、模型名、apiKeyStatus、autoUpdater）（`PromptInputFooter.tsx` L139-150）。
- 占位符三级：teammate 提示 → 队列提示 → 首次示例命令（`usePromptInputPlaceholder.ts` L25-80）。
- 建议列表：overlay 上限 5 项 / 内联 min(6, rows-3)，图标+文件/◇ MCP/* agent；inlineGhostText 补全（`PromptInputFooterSuggestions.tsx` L17-18）。

**claude-code-haha 欢迎/onboarding**：WelcomeV2 字符画 logo（Clawd 图标 + "Welcome to Claude Code vX" + 虚线框，Apple_Terminal 变体）+ 分步向导（preflight → theme → api-key → oauth → security → terminal-setup）（`LogoV2/WelcomeV2.tsx`、`Onboarding.tsx`）。

---

## 1. 概念 A「航图」— grok 血统，信息密度优先

**一句话**：把 dsh TUI 从"对话滚动"升级为"工作台视图"——欢迎页是功能入口，主界面按信息优先级堆叠，一切状态可见。

### 1.1 欢迎页（首屏）

```
┌─ dsh 航图 ───────────────────────────────────────────── 2026-08-12 ─┐
│                                                                      │
│        ████████                                                        │
│        ██ dsh ██      Tianshu Harness  v0.x                          │
│        ████████      中文 / English · 主题 graphite · API key ✓        │
│                                                                      │
│        ╭──────────────────────────────────────────────────────╮       │
│        │ 新会话              ctrl+n                            │       │
│        │ 恢复最近会话        ctrl+s                            │       │
│        │ 从 .rivet 导入设置  ctrl+i                            │       │
│        │ 退出                ctrl+q                            │       │
│        ╰──────────────────────────────────────────────────────╯       │
│                                                                      │
│  Type a message…   （或 ctrl+p 打开命令面板）                           │
└──────────────────────────────────────────────────────────────────────┘
```

**设计决策**：
- **双布局**（对齐 grok `mod.rs` L263-347）：宽 ≥90 且非 compact 时用 hero 框（左 logo 右信息+菜单）；否则 stacked 堆叠（logo → 环境检查 → 菜单 → tip → prompt）。当前 `welcome.ts` 的 4 行迷你框保留为 **stacked 的 compact 档**（rows <8 / width <48 时）。
- **菜单即入口**：菜单行复用 `menu.rs` 形态——label 左对齐 BOLD、快捷键右对齐、选中整行高亮。dsh 版菜单项映射到现有能力：`新会话`（newSession）、`恢复会话`（renderRestorableSessions 已有列表，改为可选中交互）、`导入设置`（若 .rivet 存在）、`退出`。
- **brand 可替换**：顶框品牌名走 `FormatWelcomeInput.brand`（已有字段），主题色 brandColor。
- **环境检查并进欢迎页**：`renderWelcomeEnvCheck`（API Key / Git / 终端）作为 stacked 的 error/info slot（grok 的 error 槽位），不单独占屏。
- **toast 槽**：一次性消息（如"会话已恢复"）在 prompt 上方右对齐单行，复用 `toast.rs` 形态。
- **prompt 固定 3 行**：欢迎页输入框与主界面同一组件（共享 PromptWidget 语义 = dsh 的 `InputLine` + `InputController`），placeholder 为 `Type a message…`（或 `输入消息…`，主题变量）。

### 1.2 主界面布局

```
┌─ main ✗ ✓ ─ 分支 main · ~/proj/dsh ─────────────────────────────────┐  ← top bar（grok top_bar.rs）
│  ⏳ understand → research → decompose → implement → verify → wrap    │  ← 工作流阶段指示（已有）
│  ┌─ 消息区 scrollback（已提交，不可擦除）─────────────────────────┐    │
│  │ 用户 ▌ …                                                        │    │
│  │ 助手 markdown …                                                 │    │
│  │ 🛠 bash 12.3s · file 家族色 · 折叠卡 …                          │    │
│  └──────────────────────────────────────────────────────────────┘    │
│  ◆ 等待输入 / ⏳ Thinking… / ⏳ Responding…                          │  ← turn_status（grok turn_status.rs）
│  ────────────────────────────────────────────────────────────────     │
│  ❯ Type a message…                                                   │  ← prompt（3 行固定）
│  Enter 发送 · Shift+Enter 换行 · / 命令 · @ 文件 · ctrl+p 面板       │  ← shortcuts 行
└──────────────────────────────────────────────────────────────────────┘
```

**设计决策**：
- **顶部栏**（新建 `format/top-bar.ts`）：分支 icon+分支名（DIM）+ tilde 折叠 cwd（gray_dim）+ 可选 announcement 第二行。dsh 数据源：cwd 与 git 分支来自 `ctx`（git 信息走缓存不阻塞）。
- **turn_status 行**（新建 `format/turn-status.ts`）：braille spinner（已有 `braille-spinner.ts`）+ 阶段标签（复用 `activity-status.ts` 的 ActivityPhase）+ 等待输入时 pulsing ◆。这替换当前 glance 状态行的部分职责——`activity-status` 模型已就绪（projection-layer.md 记录 App 尚未驱动它），概念 A 正好驱动它。
- **shortcuts 行**：模式相关快捷键（plan 模式显示 `Enter save / Esc cancel / a approve`，对齐 grok `render.rs` L926-1028 的 footer hints）。数据源：模式状态（plan/auto）已投影。
- **面板区**：7 面板保留但**降级为弹层**——grok 的面板区（tasks/catalog/todo/queue）常驻顶部；dsh 的面板是 /命令触发。概念 A 建议：常驻区只放 tasks + workflow 两个活跃度最高的，其余保持 overlay（Ctrl+P / /status 等现有路径）。

### 1.3 与现有架构的映射

| 新视觉 | 落点 | 复用/新建 |
|---|---|---|
| 欢迎页 hero/stacked 双布局 | `format/welcome.ts` 扩展 | 复用 formatWelcome 的 compact 档；新建 hero 分支（纯函数） |
| 菜单行 | `format/welcome.ts` 内新 `formatWelcomeMenu` | 新建纯函数；交互走现有 slash/命令注册 |
| top bar | 新 `format/top-bar.ts` | 新建；CommitEngine 首行提交 |
| turn_status | 新 `format/turn-status.ts` | 复用 braille-spinner、activity-status |
| shortcuts 行 | `render/live-panels.ts` 追加面板或 renderLive 直渲染 | 复用模式投影 |
| toast | 新 `format/toast.ts` | 新建纯函数；app.ts 一次性消息队列 |

**增量**：约 5 个纯函数模块 + app.ts 组装 + 对应 spec。零后端改动。

---

## 2. 概念 B「深潜」— claude 血统，对话沉浸

**一句话**：极简对话流 + 一个强大的多行输入框——输入框是全部 UI 的核心，信息都在输入框的 footer 里。

### 2.1 首屏欢迎

```
   ██████████████████
   ██   d s h      ██     Tianshu Harness v0.x
   ██  ██████████  ██
   ██████████████████     ·····································
                          首次使用？三步就绪：
                          1. 选择主题 → 2. 配置 API Key → 3. 开始
                          （Enter 开始引导 · Esc 跳过直接进入）

   ❯ Type a message…
```

**设计决策**：
- **品牌字符画 + 一句欢迎**（对齐 WelcomeV2）：字符画 logo（dsh 三字母方块，Unicode 半块字符 █▄▀，legacy 终端降级为纯文本行），下方品牌名 + 版本（dim）。
- **分步引导**（对齐 Onboarding.tsx 的 step 序列）：dsh 版三步——主题（复用 `ThemePicker`/`theme-detect`）、API Key（检测 `DEEPSEEK_API_KEY`，对齐 `renderWelcomeEnvCheck` 的 hasApiKey）、安全提示。每步 Enter 继续 / Esc 跳过。首次（无可恢复会话）才出现；有可恢复会话时直接进入会话恢复列表（现有 `renderRestorableSessions`）。
- **无菜单**：不提供入口菜单——一切通过输入框（/ 命令）到达，保持首屏纯净。

### 2.2 主界面布局

```
  ··································································
  ▌ 重构 package 拆分的接口边界

  > 我先读一下现有模块划分，然后给出拆分方案。

  🛠  read_file  src/packages/index.ts  2.1s

  ··································································
  ❯ 继续，把方案写进 docs/            （多行输入，图片 [Image #1] chip）
  ─────────────────────────────────────────────────────────────────
  normal · plan off     Enter 发送 · Shift+Enter 换行 · / 命令
                                    ↑
  模式/快捷键（左）                token 用量 · 模型 · API 状态（右）
```

**设计决策**：
- **输入框**（对齐 PromptInput）：
  - 前缀符号 ❯（默认）/ `!`（bash 输入模式——dsh 已有输入模式概念可扩展）；loading 时 dim。
  - **多行**：maxVisibleLines = max(3, floor(rows/2)-5)。当前 `input-line.ts` 是单行 + 历史/undo，多行是新增能力（换行语义需要确认：Shift+Enter 换行）。
  - **只画底边圆角框**：输入区上下留白、下边框圆角（└─ 形态），色随模式。
  - 图片附件 [Image #N] chip 已存在（input-line images 参数），视觉上保留。
- **footer 一行**（对齐 PromptInputFooter）：左——模式（normal/plan/[plan…]，复用 statusline badge 词汇）+ 快捷键提示（Enter/Shift+Enter//）+ tasks pill；右——token 用量（formatGlanceBar 已有 token 段）+ 模型名 + API key 状态（renderWelcomeEnvCheck 的 hasApiKey 状态）。窄终端（<80 列）纵排两行。
- **消息区保持现状**：markdown + 工具卡已足够，概念 B 不重设计消息视觉，只强化"对话流纯净"——工具卡默认更紧凑（`/density` 已存在）。
- **建议补全**（对齐 PromptInputFooterSuggestions）：内联 ghost text（输入后文）+ 输入 `/` 或 `@` 时在输入框上方弹建议列表（图标 + 文件/◇ MCP/* agent）。dsh 已有 `/` 命令补全和 `@` 文件补全，缺的是视觉形态与 ghost text。
- **状态行弱化**：glance 状态行并入 footer 右区（token/模型），7 面板全部保持 overlay 弹层，屏幕上只剩"对话 + 输入框"。

### 2.3 与现有架构的映射

| 新视觉 | 落点 | 复用/新建 |
|---|---|---|
| 品牌字符画 | `format/welcome.ts` 新 `formatBrandWelcome` | 新建（字符画为静态常量，legacy 降级） |
| 分步引导 | app.ts attach 流程扩展 | 复用 theme-detect、renderWelcomeEnvCheck |
| 多行输入 | `engine/input-line.ts` 扩展 | 行缓冲/光标/undo 已有，加换行折叠 |
| footer 行 | 新 `format/prompt-footer.ts` | 复用 glance-bar、statusline badge、model 名（已有投影） |
| ghost text 建议 | `format/` 新 `format-suggestions.ts` + input-controller 扩展 | 复用 file-completer、slash 注册表 |
| 消息区 | 不变 | — |

**增量**：输入框多行 + footer + 建议形态（输入层扩展，风险高于概念 A）+ 2 个新 format 模块。零后端改动。

---

## 3. 概念 C「工作站」— 融合，dsh 特色会话工作台

**一句话**：grok 的入口效率 + claude 的输入框质感 + dsh 独有的多会话/btw/审批/面板——面向重度多会话用户。

### 3.1 欢迎页（会话恢复优先）

```
┌─ dsh ────────────────────────────────────────────────────────────┐
│                                                                   │
│   ██ dsh ██   Tianshu Harness v0.x · graphite                    │
│                                                                   │
│   ╭─ 恢复会话 ────────────────────────────────╮                    │
│   │ ◇ 昨天 · 重构 tui 拆分 (fork: main)  [1]  │                    │
│   │ ◇ 3 天前 · meridian wave5 收尾      [2]   │                    │
│   │ ◇ 上周 · 前缀缓存基线                [3]   │                    │
│   ╰──────────────────────────────────────────╯                    │
│   新会话 ctrl+n · 导入设置 ctrl+i · 退出 ctrl+q                    │
│                                                                   │
│   ❯ 输入消息开始新会话…                                            │
│   API key ✓ · Git ✓ · 24×80 · 3 个会话可恢复                       │  ← 环境检查一行
└───────────────────────────────────────────────────────────────────┘
```

**设计决策**：
- **会话恢复是主角**：dsh 的多会话（SessionStore + fork 谱系）是区别于 grok/claude 的核心资产。欢迎页主体 = 可恢复会话列表（数字键选择，复用 `renderRestorableSessions` 的数据 + 新的可选择渲染），grok 的 Resume session 菜单项升级为**首屏列表**。
- **fork 谱系可见**：列表项显示 fork 谱系（parentSession，restore-session.ts 已有该数据），`[1][2][3]` 数字键直达。
- **环境检查压缩为一行**：`API key ✓ · Git ✓ · 24×80`（renderWelcomeEnvCheck 的紧凑档），不占屏。
- **底部一行入口**：新会话/导入/退出（快捷键右对齐，菜单行形态），输入框常驻可立即输入。

### 3.2 主界面布局

```
┌─ main ✗ ✓ · ~/proj/dsh ──────────── [会话1] [会话2] [会话3] ────┐  ← top bar + 会话 tab（已有）
│  ⏳ 实现 wave 3 · 工具卡 2/5 并行 · 12.3s                         │  ← glance 行（已有，扩展阶段）
│  ┌─ scrollback ──────────────────────────────────────────────┐    │
│  │ …                                                          │    │
│  └───────────────────────────────────────────────────────────┘    │
│  ◇ btw 侧问: 这个方案和 C1 文档的对比？                           │  ← btw 行（已有）
│  ⚠ 允许执行 edit_file src/x.ts？[y/N]   ┌─ diff 预览（≤12 行）─┐ │  ← 审批行（已有 + C2-1 diff）
│  ────────────────────────────────────────────────────────────     │
│  ❯ 输入消息…                                        [Image #1]    │  ← prompt（3 行）
│  normal [plan] · Enter 发送 · / 命令 · ctrl+p 面板                │  ← footer（模式 + 快捷键）
│  token 12.3k · deepseek-chat · 缓存命中 96.8%                     │  ← metrics 行（glance-bar 已有）
└───────────────────────────────────────────────────────────────────┘
```

**设计决策**：
- **三行底部区**（概念 C 的特色）：prompt（3 行）+ 模式/快捷键 footer + metrics 行。metrics 行放 glance-bar 的 token/缓存命中段——dsh 的缓存命中率（cache-hit-baseline 已有数据）是模型层特色，值得常驻。
- **会话 tab 上移与 top bar 合并**：`renderSessionTabs` 已有，从 live 区并入顶部栏一行，多会话切换不占底部空间。
- **btw 侧问行**（已有 btw-controller + `format/btw-panel`）保留在 scrollback 与 prompt 之间，这是 dsh 独有的交互。
- **审批行**（已有 approval-controller + C2-1 的 permission-diff）保持 y/N 内联 + diff 预览，概念 C 只做视觉统一（与 prompt 同宽度、同前缀色）。
- **面板/overlay 全部保留现状**：/status /config /skills /subagents /workflow /tasks 走 overlay（已有），屏幕只保留高活跃行。

### 3.3 与现有架构的映射

| 新视觉 | 落点 | 复用/新建 |
|---|---|---|
| 会话恢复列表欢迎页 | `format/welcome.ts` 新 `formatSessionPicker` + app.ts | 复用 restore-session 数据；新建可选择渲染 + 数字键路由 |
| top bar + tab 合并 | 新 `format/top-bar.ts` + 调整 renderSessionTabs | 半新建（tab 已有） |
| 三行底部区 | renderLive 重排 + 新 `format/prompt-footer.ts` | 复用 glance-bar、statusline |
| 环境检查紧凑行 | `format/welcome.ts` 新紧凑档 | 复用 renderWelcomeEnvCheck 数据 |
| btw/审批视觉统一 | `format/btw-panel.ts`、approval 渲染微调 | 已存在 |

**增量**：概念 A 的 top-bar/turn_status + 概念 B 的 footer 形态 + 会话选择器。是三案中最大的，但全部落在纯函数层与 app.ts 组装层。

---

## 4. 三案对比

| 维度 | A「航图」 | B「深潜」 | C「工作站」 |
|---|---|---|---|
| 视觉血统 | grok | claude code | 融合 |
| 首屏 | 菜单入口（hero/stacked 双布局） | 品牌字符画 + 分步引导 | 会话恢复列表 |
| 输入框 | 3 行固定，无 footer | 多行 + footer 左右分栏 + ghost 建议 | 3 行 + 双 footer（模式 + metrics） |
| 信息密度 | 高（一切状态可见） | 低（对话纯净） | 中（高活跃行常驻） |
| dsh 特色 | 面板 overlay 化 | 弱化 | 会话 tab/btw/缓存命中常驻 |
| 主要新增 | 5 个 format 纯函数 | 输入层多行改造（风险点） | 前两者之和 + 会话选择器 |
| 与现状差距 | 中 | 中（输入层） | 大 |
| 实现风险 | 低（纯渲染） | 中（输入模型改动） | 中 |
| 对重度用户的收益 | 高（一切可见） | 中（输入效率） | 最高（多会话工作流） |

### 推荐路径

**分阶段混合**：
1. **Wave 1（概念 A 的骨架 + 概念 C 的会话恢复）**：top bar + turn_status + 欢迎页双布局（菜单/会话列表二选一，先做 stacked 档——当前 4 行迷你框已是最简档，加菜单行即可）。纯渲染，风险最低，收益立现。
2. **Wave 2（概念 C 的底部三行区）**：prompt footer（模式/快捷键）+ metrics 行（glance-bar 已有段，仅重排）+ 会话 tab 上移。零新数据源。
3. **Wave 3（概念 B 的输入框增强）**：多行输入 + 底边框视觉 + ghost text 建议。输入层改动，独立验证（input-line.spec 已有覆盖基础）。
4. 欢迎页 hero 布局（宽终端专属）与品牌字符画作为**可选装饰**，排在 Wave 3 之后——grok 的 hero box 需要 ≥90 列且其价值主要是品牌感，stacked 档已覆盖功能。

**决策点（需要用户确认）**：
- 欢迎页首屏主角：菜单入口（A）vs 会话恢复列表（C）vs 纯品牌引导（B）？
- 输入框多行化是否进入本次范围（影响 Wave 3 优先级）？
- metrics 行（token/缓存命中）是否常驻底部（C）还是并入 footer 右区（B）？

---

## 5. 反证/复现（设计定稿前需验证的断言）

概念稿中的关键论断多为对参考项目的源码阅读结论（evidenceStatus: unverified），实施前逐条核验：

| 断言 | 来源 | 核验方式 |
|---|---|---|
| grok 欢迎页双布局阈值（宽≥90、compact 双标志） | `grok-build/.../welcome/mod.rs` L263-347 | cargo 构建 + pty e2e（仓库已有 welcome_screen.rs 测试）或截图 |
| grok menu 行视觉（label BOLD + key 右对齐 + bg_highlight） | `menu.rs` L30-63 | 同上 |
| claude footer 左右分栏与窄终端纵排（<80 列） | `PromptInputFooter.tsx` L105-150 | 起 dev server 截图（browser_debug），80/120 列两档 |
| dsh `input-line.ts` 单行 → 多行的改造面 | 本仓库 input-line.ts | 写最小多行渲染 spec（RED→GREEN） |
| `activity-status.ts` 可驱动 turn_status 行 | projection-layer.md（App 尚未驱动） | 事件注入 spec 已存在（activity-status.spec.ts），补 app 装配测试 |

**反目标**：不引入 ratatui/ink 等新渲染依赖（引擎层已定 CommitEngine/LiveEngine）；不改变会话事件词汇（投影纪律）；不在概念稿阶段改任何引擎代码。

## 6. 参考文件索引

- grok-build：`crates/codegen/xai-grok-pager/src/views/welcome/{mod,hero_box,logo,menu,prompt,toast,top_bar,workspace_mode}.rs`、`src/views/agent.rs`（AgentViewLayout）、`src/app/agent_view/render.rs`（PromptStyle/footer hints）、`src/scrollback/block.rs`（RenderBlock）
- claude-code-haha：`src/components/PromptInput/{PromptInput,PromptInputFooter,PromptInputFooterLeftSide,PromptInputFooterSuggestions,PromptInputModeIndicator,usePromptInputPlaceholder}.tsx`、`src/components/LogoV2/WelcomeV2.tsx`、`src/components/Onboarding.tsx`
- 本仓库：`packages/tui/tui/src/format/welcome.ts`、`engine/input-line.ts`、`render/live-panels.ts`、`statusline.ts`、`ui/app.ts`（renderLive L1764）、`activity-status.ts`
- 系列文档：`docs/dsh-tui-与claude的对比-c1.md`、`docs/dsh-tui-与grok的功能对比-c2.md`、`docs/dsh-tui-增强方案-c3.md`、`docs/dsh-tui-next-phase.md`

---

## 7. 决策记录（2026-08-12，用户选型 + Wave 1+2 实施）

用户对第 4 节三个决策点的答复：**① 欢迎页首屏主角 = 菜单入口（A）；② 输入框多行化 = 放到下一批；③ metrics 行 = 常驻底部（C）**。据此实施推荐路径 Wave 1 + Wave 2：

| 决策 | 实施 | 提交 |
|---|---|---|
| 菜单入口 | `formatWelcomeMenu`（label BOLD 左 + keyHint 右对齐，宽度守恒）+ ctrl+n/s/q 键路由（input-handler 补 ctrl_s/ctrl_q 键名与 0x11/0x13 解析） | `abe58e6` |
| 顶部栏 | `formatTopBar`（📁 cwd → model → (branch)，ascii 档 `~`）替换内联 context bar；快捷键提示移出（归 footer） | `7300b33`（纯函数）/ `72471e6`（装配） |
| 状态行 | `formatTurnStatus`（braille spinner / pulsing ◆）替换 glance 纯文本；`glanceStatus` 放宽 `string \| null` | `72471e6` |
| 三行底部区 | `formatPromptFooter`（模式徽标 + 快捷键）+ metrics 行从 glance 面板移出、输入行下常驻 | `72471e6` |
| 多行输入 | 未实施（用户决策：下一批） | — |
| hero 布局 / 品牌字符画 | 未实施（Wave 3 之后的可选装饰） | — |

验证：TUI 全量 1326 passed（75 文件）、`tsc --noEmit` 0 错误；Agent Note `.agents/notes/implemented/feature/2026-08-12-tui-c4-concepts-w12.zh.md`。

## 8. 决策记录（2026-08-12，B 布局 Wave：输入框底边线 / 欢迎页会话限高 / footer 合并）

用户反馈：欢迎页会话列表过多占满（一般不进旧会话）、输入框无横线；选型 **概念 B「深潜」（claude code 布局）**——四周无包围框线，先做 B 的样式。

| 决策 | 实施 |
|---|---|
| 输入框底边圆角线 | `formatInputDivider`（新）：`└─` 形态、色随模式（normal secondary / plan warning / auto error）、ascii 降级 `+`/`-`，在输入行下方渲染（B「只画底边圆角框」） |
| 欢迎页会话列表 | `formatRestorableSessions` 加 `maxRows` 选项；app.ts 传 1——只展示最近 1 个可恢复会话，超出折叠为「… 还有 N 个会话」 |
| footer 合并一行 | `formatPromptFooter` 加 `rightSegments`：宽终端（≥ `FOOTER_RIGHT_MERGE_MIN_WIDTH`=80 列）右侧状态段（token/模型/API ✓✗）右对齐合并进同一行；窄终端纵排两行（metrics 独立行保持） |

验证：TUI 全量 1354 passed / 2 todo（76 文件）、`tsc --noEmit` 0 错误、lint 与 verify-export-jsdoc 0 错误；Agent Note `.agents/notes/implemented/feature/2026-08-12-tui-c4-b-layout-bottom-bar.zh.md`。

## 9. 决策记录（2026-08-12，slash 命令下拉菜单——grok slash_dropdown 移植）

用户反馈：`/` 不展示命令列表、不能上下滚动命令；参考 grok-build（xai-grok-pager `views/slash_dropdown.rs` + `slash/mod.rs` + `prompt.rs` 键路由）设计。阶段 1 实施：

| 决策 | 实施 |
|---|---|
| 匹配数据源 | `InputController.refreshSlash`（onChange 驱动）：`/` 开头且有匹配 → 打开（孤立 `/` 全量列表）；前缀优先 + 子串兜底；query 不变时 carry 保持选中 |
| 列表渲染 | 新 `format/slash-menu.ts`：label 列对齐 + 描述截断；选中 `❯` + primary + bold（主题无 bg 槽）；maxRows=8 滚动窗口 + 「↑↓ 还有 N 项」；ascii 降级；宽度守恒 |
| 键路由 | 菜单打开时：↑↓ 移动（环绕）/ PageUp/Down 翻页 / Tab 接受 / Enter 提交（精确命令直接发且清空输入行）/ Esc 关闭（Ctrl+P/N 已被占用，不照搬 grok 键位） |
| 参数命令 | argsHint 命令补全到 `cmd ` 留参数位；参数链式补全留待下一批 |

验证：TUI 全量 1382 passed / 2 todo（77 文件）；`tsc --noEmit` 0 错误；lint/jsdoc gate 0 错误；Agent Note `.agents/notes/implemented/feature/2026-08-12-tui-slash-menu-dropdown.zh.md`。

## 10. 决策记录（2026-08-12，slash 菜单阶段 2：MRU / 参数占位 ghost / 输入行 ghost 预览）

| 决策 | 实施 |
|---|---|
| MRU 排序 | `InputController.slashMru`（上限 10）+ `recordSlashUse`（runSlash 成功后记录）；匹配组内按最近使用稳定排序 |
| 参数占位 ghost | `/cmd `（完整命令名+尾空格）且带 argsHint → 菜单保持打开 + 输入行 ghost 显示参数占位（`<name>`）；Enter 提交完整行 |
| 输入行 ghost 预览 | `InputLine.setGhost`：光标在末尾且无选区时，█ 右侧 dim 显示补全剩余（`/th` → `eme`）；wrap 路径光标行插入并截断 |

不做：mid-text slash token 高亮（需动 InputLine 核心渲染，价值低，留待验收后评估）。

验证：TUI 全量 1402 passed / 2 todo（78 文件）；`tsc --noEmit` 0 错误；coverage gate 0 ERROR；Agent Note `.agents/notes/implemented/feature/2026-08-12-tui-slash-menu-dropdown.zh.md`。

## 11. 决策记录（2026-08-12，subagent 对话流状态行——grok SubagentBlock 移植）

| 决策 | 实施 |
|---|---|
| 运行中 | live 区动态行 `⠋ 子代理 <label>`（braille spinner 帧；ascii 降级 `*`） |
| 终态 | end 事件结算耗时并提交 scrollback：completed `✓`（success）/ aborted `◌`（muted）/ error/max-tokens/refusal/未知 `✗`（error）+ ` (reason)` 后缀 |
| 数据 | `subagentRuns`（runId → label/startedAt）；label 取委派树缓存（滞后回退 id 短哈希）；未配对 end 免疫（跨会话安全）；事件声明修正为真实契约（runId/id/stopReason） |
| 不做 | Enter 打开子会话全屏视图（dsh 无子会话视图概念，留待后续） |

验证：TUI 全量 1416 passed / 2 todo（79 文件）；`tsc --noEmit` 0 错误；lint/jsdoc 0 错误；subagent-line.ts 覆盖率 100%；Agent Note `.agents/notes/implemented/feature/2026-08-12-tui-slash-menu-dropdown.zh.md`。
