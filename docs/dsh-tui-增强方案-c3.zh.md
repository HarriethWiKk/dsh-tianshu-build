# DSH TUI 四项增强方案 — C3

[English](dsh-tui-增强方案-c3.md) | 中文

> **标记**：C3（DSH TUI 增强系列第 3 份；C1 对标 Claude Code、C2 对标 grok-build）**日期**：2026-08-11（初稿）/ 2026-08-11 修订（加 rewind 天枢搬运 + Shift+Tab）**范围**：rewind 回退（抄天枢）、plan 审批门、/fork 增强、Shift+Tab 模式切换**参考**：[dsh-tui-与claude的对比-c1.md](dsh-tui-与claude的对比-c1.md)、[dsh-tui-与grok的功能对比-c2.md](dsh-tui-与grok的功能对比-c2.md)

## 重要前置发现：四项的真实状态差异巨大

调研后确认，四项**不是同一性质的工作**：

| 项 | 调研前假设 | 调研后事实 | 性质 |
|---|---|---|---|
| rewind UI | 底层有 checkpoint，只差 UI | checkpoint 只是 flush；**但天枢 `FileHistory` 核心 rewind 路径自包含可搬**（≈314 行，核心路径 ~200 行零外部依赖，diff 统计部分依赖 worker pool 需剥离），让 rewind 变成移植+适配 | 🟡 移植 + 一处后端缺口（SessionStore.truncate） |
| plan 审批门 | 底层有 plan-mode，差审批 UI | 链路存在但**形状断层**（诊断修订：TUI settle 提交 `{questionId,value}`/`{cancelled}`，user-interaction 直通 provider 无转换，exit_plan_mode 读 `answer.answers` 会 TypeError）——先修形状 + 再加反馈路径 | 🔴 修断层 + 纯 TUI 增强 |
| /fork 入口 | 底层 forkable，TUI 无命令 | **`/fork` 命令已存在且可用**，只缺可选参数 | 🟢 纯 TUI 增强 |
| Shift+Tab 模式切换 | 需新建 | **键解析已就绪**（`shift_tab` 已在 KeyName），plan-mode API 现成；缺 always-approve 第二轴 + handleKey 接线 | 🟡 TUI + 最小本地状态 |

**结论**：plan 是「修形状断层 + 补最后一公里」（诊断修订：链路并非端到端可用）；fork 是「补完最后一公里」；Shift+Tab 是「接线 + 加本地标志位」；rewind 是「移植天枢 + 加 SessionStore.truncate」。四项均可做，工作量从 ~35 行到跨包移植不等。

---

## 项 1：plan 审批门增强（纯 TUI，最小工作量）

### 现状：链路存在但形状断层（诊断修订 2026-08-11）

**诊断修订**：C3 初稿称「端到端已存在」是静态乐观断言。巡天侦察 + 独立核验发现链路在**形状层面断裂**：

- TUI `settleQuestion` 提交 `{ questionId, value: option.label }`（app.ts:1215）或 `{ cancelled: true }`（app.ts:1210）——TUI 层自己的契约，被 app.spec.ts:2183 固化；
- `user-interaction` 的 `ask()` 直通 provider、无任何形状转换（user-interaction/src/index.ts:153）；
- plan-mode `exit_plan_mode` 期望 `{ answers: [{ id, selected[], custom? }] }`（plan-mode/src/index.ts:365 `answer.answers.filter(...)`）——TUI 返回物没有 `.answers`，**真实装配下抛 TypeError**；
- 取消路径同样断裂：`exit_plan_mode` 期望 reject `UserInteractionError('...', 'ASK_CANCELLED')`，TUI 是正常 resolve `{cancelled:true}`，落不进取消分类；
- 两套测试各自固化自己的形状（app.spec.ts:2183 vs user-interaction.spec.ts:16/38），集成点无测试覆盖——宽松 mock 掩盖的典型断层。

**修复方向**（实施项 1 的前置步骤）：在 TUI 提交侧把答案包装为 provider 契约形状 `{ answers: [{ id, selected: [label], custom? }] }`，取消路径改为 reject `UserInteractionError`（code `ASK_CANCELLED`）；或按 AGENTS.md「Trust TypeScript at typed same-process boundaries」在服务边界做一次显式转换（选型在实施批次定）。

完整链路（修形状后）：

```
agent 调 exit_plan_mode(plan)
  → PlanModeService 调 ctx.userInteraction.ask({ intent: { kind: 'plan-review' }, detail: plan, options: [Approve, Keep planning] })
    → TUI 的 handleQuestionRequest 挂起为 pendingQuestion（app.ts:577-595）
      → question-panel.ts 识别 intent.kind === 'plan-review'（L111），渲染 🧭 决策卡（计划正文 + ✓/✗ 选项）
        → 用户数字键选 → settleQuestion（L1215）→ exit_plan_mode 返回 { approved } 或抛反馈
```

**已确认存在的文件**：
- `packages/plan/plan-mode/src/index.ts:302-389` — `exit_plan_mode` 工具 + plan-review intent
- `packages/tui/tui/src/question-panel.ts:111` — `intent?.kind === 'plan-review'` 渲染分支（🧭 卡 + ✓/✗ 标记；detail 扁平缩进在 L120-124）
- `packages/tui/tui/src/ui/app.ts:1207-1219` — pendingQuestion 键处理（数字键 1-9 选 / Esc 取消；settle 提交 L1215；挂起 L577-595）
- `packages/interaction/user-interaction/src/types.ts` — AskUserQuestionIntent 含 `plan-review` kind

### 缺口

0. **形状断层（前置，见上）**：TUI 提交形状与 provider 契约不匹配，真实装配下 `exit_plan_mode` 抛 TypeError——必须先修，否则现有「批准」路径都不可用。

1. **无「带反馈的 request-changes」路径**：`exit_plan_mode` 读 `item.custom` 作反馈文本（plan-mode/index.ts:368-371，`const feedback = item?.custom ?? ''`），但 TUI 的 handleKey 只发 `value` 或 `cancelled`——「keep planning」不带反馈文本。用户无法在 TUI 里说「这个计划的第 3 步不对，改成 X」。

2. **计划正文是扁平缩进行**：question-panel.ts 把 `detail`（计划 markdown）按行缩进渲染，无 markdown 结构（标题/列表/代码块）。长计划在 live 区占满屏，不可滚动。

### 方案

#### 改动 1：加 request-changes 反馈路径

**改 `packages/tui/tui/src/ui/app.ts`**（handleKey 的 pendingQuestion 分支，L1207-1219）：
- 现状：数字键 1-9 → 选选项；Esc/Ctrl+C → cancelled
- **前置：settleQuestion 提交形状改为 provider 契约** `{ answers: [{ id, selected: [label], custom? }] }`，取消路径改 reject `UserInteractionError('...', 'ASK_CANCELLED')`（修复形状断层，见「现状」节）
- 新增：`f` 键（feedback）→ 进入「反馈输入模式」，输入框接收文本，Enter 提交为 `{ answers: [{ id, selected: ['Keep planning'], custom: <输入文本> }] }`
- 复用 input-line 的编辑能力（已有 vim/光标/历史），只加一个「反馈态」标志位 + 渲染提示

**改 `packages/tui/tui/src/question-panel.ts`**：
- plan-review 卡底部加 key hints：`[1] 批准  [2] 继续规划  [f] 反馈修改  [Esc] 取消`

#### 改动 2（可选，第二批次）：计划正文 markdown 渲染

**改 `packages/tui/tui/src/question-panel.ts`**：
- detail 渲染从「按行缩进」改为复用 `format/markdown.ts`（已有标题/列表/代码块渲染）
- 长计划超 N 行时折叠 + 提示「完整计划见 alt-screen viewer」（后续接 overlay 全屏查看）

### 范围与工作量
- **第一批 = 修形状断层 + 改动 1**（feedback 路径）：形状转换 ~15 行 app.ts（settleQuestion/取消路径）+ ~40 行 app.ts + ~10 行 question-panel.ts + spec + **集成测试**（补 app.spec 与 user-interaction 契约对齐断言，覆盖「TUI 提交 → plan-mode 读到 answers」整链）
- 改动 2 留后续批次（markdown 渲染 + 全屏 viewer 是独立工作量）

---

## 项 2：/fork 增强（纯 TUI，最小工作量）

### 现状：命令已存在

**已确认存在的完整接线**：
- `packages/tui/tui/src/commands/registry.ts:262-267` — `/fork` 命令（调 `deps.forkSession()`；branch 别名紧随其后）
- `packages/tui/tui/src/ui/app.ts:671-680` — `forkSession()`：`ctx.sessions.fork(activeSessionId)` → `switchSession(child.id)`
- `packages/core/session/src/index.ts:1095` — `SessionStore.fork(source, boundary?, childSessionId?)`：seed = `events[0..boundary]`（默认到最后一个事件），meta 含 `parentSession`/`seedLength`
- `packages/tui/tui/src/restore-session.ts:63` — 可恢复会话列表已渲染 fork 血缘（`fork: <parentSession>`）

### 缺口

当前 `/fork` 无参数、无 boundary 选择。grok 的 `/fork` 支持：
- `<directive>` — fork 后的首条 prompt（分叉探索方向）
- `--worktree` / `--no-worktree` — git worktree 隔离（**DSH 无 worktree 概念，不做**）
- `--at <turn>` — 从指定 turn 分叉（DSH 的 `fork(source, boundary)` 已支持 seq boundary）

### 方案

#### 改动：/fork 加可选 directive + boundary

**改 `packages/tui/tui/src/ui/app.ts`**：
- `forkSession()` → `forkSession(opts?: { directive?: string; atSeq?: number })`
- `atSeq` 透传给 `ctx.sessions.fork(activeSessionId, atSeq)`
- `directive`：fork + switchSession 后，经 `this.controls?.followup(directive)` 作为首条消息

**改 `packages/tui/tui/src/commands/registry.ts`**：
- `/fork` 的 run handler 解析参数：
  - 纯文本 → directive
  - `/fork <text>` → fork + 提交 text 为首条消息
  - 无参数 → 现状（纯 fork）
- `/fork at <seq> <text>` → 带 boundary（高级用法，第一批可只做 directive、boundary 留后续）

**改 `packages/tui/tui/src/adapter/sessions.ts`**：
- `forkSession(ctx, source, boundary?, childSessionId?)` 已存在且支持 boundary，无需改

### 范围与工作量
- 第一批只做 directive（`/fork 探索另一种方案` → fork 后自动提交）：~20 行 app.ts + ~15 行 registry.ts + spec
- boundary 选择 UI（picker）留后续——需要 turn 列表投影，工作量独立
- worktree 永不做（DSH 无此概念，session-level fork 已够用）

---

## 项 3：rewind 回退（🔴 需后端新工作）

### 现状：底层完全缺失

这是三项里**唯一真正的缺口**，且缺口在底层、不在 TUI。

#### checkpoint 的真相

`packages/session/session-checkpoint-policy/src/index.ts`（≈78 行）做的核心事情是 `ctx.sessions.flush(session)`——把内存里的 append-only 事件日志**持久化到磁盘**（防崩溃丢 turn）。**「checkpoint」= flush，不是快照**（tools/execute 分支另有 `exec.signal.aborted` 检查返回 canonical 错误，非纯 flush）。

三个 flush 时机：
- `llm/stream` 前（首个 chunk 发出前）
- `tools/execute` 前（顶层工具 body 执行前）
- `agent/pre-step` 前

**没有任何文件快照。** flush 的产物是 JSONL/SQLite 事件日志，不是可回退的文件状态。

#### 缺失的四个组件

| 组件 | grok 有 | DSH 现状 |
|---|---|---|
| 文件快照存储 | `RewindPointInfo.num_file_snapshots` + 服务端快照 | ❌ `str_replace_editor` 读 `before` 但**丢弃**（只用于算 diff，不持久化） |
| 每轮变更文件索引 | `has_file_changes` | ❌ 无 `file/changed` 事件；变更文件只能从 tool/call+tool/result 对推断 |
| rewind API | 服务端 `rewind(sessionId, promptIndex)` 返回 `{ reverted_files, clean_files, conflicts }` | ❌ `Session` 只有 append/create/restore/fork，**无 truncate/rewind/revert** |
| rewind UI | `RewindPhase` 状态机（Loading→Picker→Confirm→Executing→Result）~900 行 | ❌ 仅 live-engine.ts:668 有个「rewind 需全重绘」注释，无实现 |

#### 已预留但未实现的痕迹

类型系统**预留了** rewind，但没人产出：
- `ConversationContextOriginKind = 'compaction' | 'rewind' | 'rewrite'`（client/runtime）
- `history-fold.ts` 识别 `source.plugin === 'rewind'` 的 user/message 开新上下文分支
- **但没有任何代码 emit `plugin: 'rewind'`**——compact 有（`compact/checkpoint.ts` 写 `plugin: 'compact'`），rewind 没有

### 方案：分两阶段

#### 阶段 1（第一批不做，出设计方案）：会话级 rewind（不含文件回退）

**最小可行**：rewind = 在目标 prompt 处 fork 新会话 + 切过去。利用已有 `SessionStore.fork(source, boundary)`。

- 优点：纯装配层，不碰 session 核心、不改持久化
- 缺点：**不回退文件**（agent 改的文件还在）、遗弃原会话 id（fork 产生新 id）
- 适用：用户想「从这里重新开始对话」，不介意文件状态

实现：`/rewind` 命令 → turn 列表 picker（从 transcript 投影）→ 选目标 turn → `ctx.sessions.fork(activeSessionId, turnEndSeq)` → `switchSession(child.id)` → 用 `plugin: 'rewind'` 标记 user/message（复用 history-fold 已预留的分类）

#### 阶段 2（后端专项，独立 PR）：文件快照 + 真 rewind

这是 grok 级 rewind 的硬工作，**不在第一批**：

1. **新建文件快照包**（`packages/snapshot/` 或 `packages/fs/fs-snapshot/`）：
   - 在工具执行前（`tools/execute` waterfall）对被改文件做 full-file before-image 快照
   - 快照存储：内存 Map + 可选持久化（JSONL/SQLite）
   - `str_replace_editor` / `tool-fs` 的 write/edit 路径要注入快照点

2. **新建 rewind API**（`packages/core/session/` 或新包）：
   - `ctx.sessions.rewind(sessionId, promptIndex): Promise<RewindResult>`
   - `RewindResult = { revertedFiles: string[]; cleanFiles: string[]; conflicts: RewindConflict[] }`
   - 截断事件日志到目标 seq（需新增 `SessionStore.truncate(id, atSeq)` + 持久化后端 `deleteFrom(atSeq)`）
   - 从快照恢复文件 + 检测冲突（文件在快照后被外部改过）

3. **TUI rewind UI**（参考 grok 的 `RewindPhase` 状态机）：
   - `/rewind` → turn picker → 确认（显示 `N 个文件将回退`）→ 执行 → 结果（回退 X 文件 / Y 文件无快照需手动处理 / Z 冲突）

**工作量评估**：阶段 2 是跨 3 个包的后端工程（快照存储 + session truncate + 持久化 deleteFrom），不是 TUI 接线，应作独立专项。

### 项 3 修订：抄天枢——rewind 变得可搬

C3 初稿基于「DSH 底层无文件快照」判断 rewind 是后端工程。**复查天枢源码后推翻此判断**：天枢的 `FileHistory` 自包含、可移植，让 rewind 从「从零建后端」变成「移植 + 适配」。

#### 天枢 rewind 的核心机制（可搬部分）

天枢用**两套独立机制**，精确路径是 TUI rewind 用的：

**精确路径（FileHistory）——抄这个：**
- `src/agent/file-history.ts`（314 行）：`FileHistory` 类，**只依赖 `node:fs/promises` + `node:crypto`**，自包含
- 快照时机：`tools/execute` waterfall 里，**每个 `write_file`/`edit_file` 执行前**调 `trackEdit(filePath, toolUseId)`，读当前全文写磁盘备份
- 存储：`<sessionDir>/<sessionId>/backups/<sha256(path)[:16]>@v<N>`（全文，按路径内容寻址）；索引内存 + `file-history.json` 持久化；上限 100 快照
- 回退：`rewindToBoundary(postBoundaryIds: Set<string>)`——找边界后每个文件最早的备份（=边界前状态），`backupFileName === null` → 文件是新建的 → unlink
- 索引：`MAX_SNAPSHOTS = 100`，溢出淘汰最旧

**粗路径（checkpoint.ts，git 基线）——不抄：**
- 每轮首个变更工具前做 git 快照，覆盖 bash 改的文件；DSH 若只经 `str_replace_editor`/`tool-fs` 改文件，精确路径已够

**关键算法（`collectPostBoundaryEditIds`，file-history.ts:18）**：扫描 `messages[messageIndex..]` 收集所有 `write_file`/`edit_file` 的 `tool_use` id——这是边界的唯一真相源，CLI 和 server 共用。

#### DSH 适配点

| 天枢 | DSH 对应 | 适配 |
|---|---|---|
| `tool-pipeline.ts` L1290 `trackEdit` 钩子 | `packages/core/tools/src/index.ts:149` `tools/execute` waterfall | 在 `next()` **前**加快照钩子（同模式） |
| `write_file`/`edit_file` 工具名 | DSH 注册名：`str_replace_editor`（command enum = `view`/`create`/`str_replace`/`insert`，**无 write**）+ `write` + `edit`（tool-fs） | **collectPostBoundaryEditIds 须判 3 个注册名**（str_replace_editor/write/edit），漏判则快照缺采；TUI 显示层的 `write_file`/`edit_file` 只是映射，勿混淆 |
| `ctx.session.rewindToMessages(msgs.slice(0, n))` | DSH `Session` 是 append-only，**无 truncate** | **真缺口**：需加 `SessionStore.truncate(id, atSeq)` + 持久化 `deleteFrom` |
| `messages[messageIndex]` OAI 消息模型 | DSH `SessionEvent[]` 事件日志 | `collectPostBoundaryEditIds` 适配为扫 `tool/call` 事件的 callId |
| `tuiApp.setInput(content)` 回填输入 | DSH `this.inputLine.setValue(content)` | 直接对应 |
| `<sessionDir>/<sessionId>/backups/` | **DSH 侧 backups 约定零存在**（packages/session 全目录 grep 无匹配） | 新增约定：照搬天枢 `<backupDir>/<sessionId>/<sha256(path)[:16]>@v<N>` 形态，backupDir 由 TUI 数据目录传入 |

**唯一的真后端缺口**：`SessionStore.truncate(id, atSeq)`。天枢的 `rewindToMessages` 是原地截断内存数组；DSH 的 `Session` 是 append-only 事件日志，截断需要：
1. `SessionStore` 加 `truncate(id, atSeq)` 方法（内存：`events.splice(atSeq+1)`）
2. 持久化后端加 `deleteFrom(id, atSeq)`（JSONL：重写截断后的文件；SQLite：`DELETE WHERE seq > atSeq`）
3. 截断后重置派生状态（token 估算、turn 计数等——参考天枢 context.ts:355 的全量 reset）

#### 项 3 分两阶段

**阶段 1（本批，会话 + 文件双回退）**：
1. 移植 `file-history.ts`（≈314 行）→ `packages/fs/fs-snapshot/src/`（新包，或挂 `packages/session/` 下）——**注意**：天枢版依赖面含 cpu-pool/cpu-tasks（`getDiffStats` 的 worker pool），「仅 node:fs/crypto」只对核心 rewind 路径成立；建议精简约搬核心 ~200 行（trackEdit/rewind/rewindToBoundary/getBoundaryFiles），diff 统计后补或复用 DSH 现有 diff 设施
2. 在 `tools/execute` waterfall 注入 `trackEdit` 钩子（DSH 工具名判定）
3. 移植 `format/rewind.ts`（183 行）→ `packages/tui/tui/src/format/rewind-overlay.ts`（纯渲染，落地文件名）
4. TUI 接 rewind overlay（复用 `OverlayController`，双阶段：消息列表 → 回退粒度选择）
5. 加 `SessionStore.truncate(id, atSeq)` + JSONL/SQLite `deleteFrom`（**唯一的后端改动**；落点在 session-persistence-jsonl / session-persistence-sqlite 两个实现包，事务边界需专项设计——append-only 日志的唯一破坏性改动，需谨慎 + 测试）

**阶段 2（后续）**：summarize 模式（`summarize-from`/`summarize-to`）——需要 LLM 摘要器，工作量大，留后续

#### RewindMode（天枢 5 种，第一批搬 3 种）

| 模式 | 会话 | 文件 | 第一批 |
|---|---|---|---|
| `convo` | 截断 | 不动 | ✅ |
| `code` | 不动 | 回退 | ✅ |
| `both` | 截断 | 回退 | ✅ |
| `summarize-from` | 边界后替换为摘要 | 不动 | ❌ 留后续 |
| `summarize-to` | 边界前替换为摘要 | 不动 | ❌ 留后续 |

#### 天枢 rewind 文件索引（搬运源）

| 天枢文件 | 行数 | 角色 | 搬运策略 |
|---|---|---|---|
| `opencode-tui/src/agent/file-history.ts` | 314 | 数据层（快照/回退/索引） | 近乎原样移植，改工具名判定 + 目录约定 |
| `opencode-tui/src/agent/file-history-persist.ts` | 43 | 持久化（file-history.json） | 原样 |
| `opencode-tui/src/tui/format/rewind.ts` | 183 | 渲染（双阶段面板） | 原样，改 overlay-frame 引用 |
| `opencode-tui/src/agent/context.ts:355` | — | `rewindToMessages` 派生态全量 reset | 参考契约，DSH 适配为事件日志截断 |
| `opencode-tui/src/tui/__tests__/format-rewind.test.ts` | 107 | 渲染测试 | 原样移植 |
| `opencode-tui/src/agent/__tests__/file-history.test.ts` | 155 | 数据层测试 | 原样移植 |

---

## 项 4：Shift+Tab 模式切换（新增）

### 目标

Shift+Tab 在 Normal → Plan → Always-Approve → Normal 间循环（对齐 grok/Claude Code 心智）。

### 现状：基础设施基本就绪

| 能力 | 状态 | 证据 |
|---|---|---|
| Shift+Tab 键解析 | ✅ 已就绪 | `input-handler.ts:75` 有 `'shift_tab'`，ANSI `[Z` 映射（L174），CSI `;2` shift 解析（L482） |
| plan-mode 服务 | ✅ 已就绪 | `ctx.planMode`，`set(agent, active)` 返回 `'committed'|'queued'|'cancelled'|'noop'`，带 pending 边界刷新 |
| plan 状态投影 | ✅ 已就绪 | TUI 经投影总线收 `{active, pending}`，渲染 `[plan]`/`[plan…]` 徽章 |
| `[plan]` 徽章 | ✅ 已就绪 | `statusline.ts:212-217` |
| **handleKey 接 shift_tab** | ❌ 缺 | `handleKey` 无 `shift_tab` 分支，键被丢弃进 InputLine |
| **always-approve 模式** | ❌ 缺 | DSH 无 session 级「全批准」概念；审批是 per-tool-call（`approval/request`） |
| permission 服务 | 🟡 存在 | `ctx.reflect.get('permission')`（app.ts:882）——`PermissionService`（**packages/interaction/permission**，非 core/）有预设表，但**无「全批准」档**：DSH 策略词汇仅 `'ask'|'never'`，never=确定性拒绝（fail-closed），不是全批准 |

### 设计决策：两轴分离（对齐 grok，非 Claude Code）

**采用 grok 模型，不采用 Claude Code 模型**：
- grok：plan 与 permission 是**正交两轴**，Shift+Tab 在一个小环上一次触一个轴（Normal→Plan→Always-Approve→Normal）
- Claude Code：所有都是「permission mode」单轴（default→acceptEdits→plan→auto）

**理由**：DSH 的 plan-mode 模块**明确定义了不变量**——「plan mode 独立于 sandbox 模式和审批策略」（plan-mode/src/index.ts:7）。把它们合并成单轴会违反这个不变量。

### 三态循环

```
Normal → Plan → Always-Approve → Normal
```

- **Normal → Plan**：`ctx.planMode.set(agent, true)`（已有 API）
- **Plan → Always-Approve**：`ctx.planMode.set(agent, false)` + 开 always-approve
- **Always-Approve → Normal**：关 always-approve

### 缺口：always-approve 第二轴

DSH 没有 session 级「全批准」。需新建（最小方案）：

**方案 A（推荐，最小改动）**：TUI 本地标志位
- `TuiApp` 加 `private alwaysApprove = false`
- `handleApprovalRequest`（app.ts:1060，注册点 `ctx.on('approval/request')` L538-540）开头加：`if (this.alwaysApprove) return 'allowed-once'`（短路，不经挂起提示）
- Shift+Tab 循环时切换此标志 + 渲染状态行指示
- **不持久化、不跨会话**——纯 TUI 本地，退出即失（与 grok 的 yolo_mode 类似，但更简）
- **适用范围（诊断修订）**：仅对 `approval='ask'` 策略会话生效——`'never'` 策略（danger-full-access 预设）在 `decide()` 的 waterfall 分发前就确定性返回 'rejected'，answerer 根本不被调用，TUI 短路不生效；且 `handleApprovalRequest` 非唯一消费点（apiproxy.ts:919 也注册 answerer 把审批转发远端 RPC），本地 TUI 运行时不受影响但需知悉

**方案 B（后端，留后续）**：新 cordis 服务 `ctx.approvalPolicy`
- session 级策略，持久化到 settings
- approval answerer 读此服务决定是否短路
- 更重，但跨会话持久

**第一批用方案 A**——纯 TUI、零后端改动、立即可用。

### 改动文件

1. **改 `packages/tui/tui/src/ui/app.ts`**（核心）：
   - 加 `private alwaysApprove = false`
   - `handleKey` 开头加 `if (key.name === 'shift_tab') { this.cycleMode(); return }`
   - 新增 `private cycleMode(): void`：
     ```
     当前 plan 关 + alwaysApprove 关 → plan 开（Normal→Plan）
     当前 plan 开 → plan 关 + alwaysApprove 开（Plan→Always-Approve）
     当前 alwaysApprove 开 → alwaysApprove 关（Always-Approve→Normal）
     ```
   - `handleApprovalRequest` 加短路：`if (this.alwaysApprove) return Promise.resolve('allowed-once')`
   - `renderLive` 状态行加 always-approve 指示（如 `[auto]` 徽章，对齐 `[plan]`）

2. **改 `packages/tui/tui/src/statusline.ts`**：
   - `formatStatusLine` 支持 `[auto]` 徽章（always-approve 激活时）

3. **改 `packages/tui/tui/src/engine/input-handler.ts`**（可选，kitty 协议兼容）：
   - `resolveEscapeSequence` 加 `\x1B[9;2u`（kitty Tab+SHIFT）→ `'shift_tab'`
   - 对齐 grok 的 `shift_tab_keys()` 三编码覆盖

4. **新建 `tests/mode-cycle.spec.ts`**：
   - 三态循环转换、always-approve 短路审批、plan 切换调 `ctx.planMode.set`

### 范围限制
- always-approve 不持久化（方案 A，退出即失）
- 不做 auto 模式（grok 的 classifier 审批，需分类器模型，留后续）
- 不做 acceptEdits（Claude Code 的「只自动批编辑」细分，留后续）

---

## 提交序列（修订：四项）

基于复查，第一批扩展为四项（原 plan + fork 两项 + rewind 移植 + Shift+Tab）：

1. `feat(tui): fix question answer shape + plan-review request-changes feedback`（项 1：形状断层修复 + feedback 路径，~65 行）
2. `feat(tui): /fork optional directive as first prompt`（项 2，~35 行）
3. `feat(tui): shift-tab mode cycling normal/plan/always-approve`（项 4，~80 行）
4. `feat(session+tui): rewind via ported FileHistory + overlay`（项 3，跨包——最大）

项 3 最大（移植 file-history ≈314 行 + rewind 渲染 ≈184 行 + SessionStore.truncate + overlay 接线），但天枢源让它从「设计未知」变成「按图施工」。

## 跨项纪律

- **项 1/2/4 不碰后端**：plan-mode / session / agent 零改动
- **项 3 碰后端**：加 `SessionStore.truncate` + 持久化 `deleteFrom`（唯一的 append-only 日志改动，需谨慎 + 测试）
- **测试**：每项独立 spec，`pnpm vitest run packages/tui/tui/tests/`（**基线实测 1135 tests / 3 failed**，诊断时点 2026-08-11：app.spec ×2 状态行/工具卡渲染、term-width isCjkLocale——实施前必须先修基线，否则无法区分回归）
- **类型**：复用现有 `QuestionRequestInput` / `SessionStore.fork` / `ctx.planMode.set` 签名
- **天枢搬运**：file-history.ts / format/rewind.ts 标注 Apache-2.0 来源（SOURCE-MAP.md——存在性未核验，搬运时确认 opencode-tui 根 LICENSE）

---

## 诊断修订记录（2026-08-11，/scout 巡天侦察蜂群 + 独立核验 + 测试实跑）

本修订基于轻量并行只读诊断（4 路 code_scout + 主代理独立核验）。scout findings 均为待核验假设，下述修正点均经 read_file/grep 独立复核。

### 修正点

| # | 原断言 | 实测事实 | 证据 |
|---|---|---|---|
| 1 | 项 1「端到端已存在」 | **形状断层**：TUI settle 提交 `{questionId,value}`/`{cancelled}`，user-interaction `ask()` 直通 provider，plan-mode 读 `answer.answers` → 真实装配 TypeError | app.ts:1215 / user-interaction/src/index.ts:153 / plan-mode/src/index.ts:365-368 |
| 2 | rewind 工具判定「str_replace_editor(write/replace) + write_file/edit_file」 | 注册名 = `str_replace_editor`（enum: view/create/str_replace/insert）+ `write` + `edit`，判 3 名 | tool-str-replace-editor/src/index.ts:423-430 / tool-fs/src/write.ts:70 / tool-fs/src/edit.ts:84 |
| 3 | 「复用 session-persistence 的目录约定」放 backups | **backups 约定零存在**，需新增 | packages/session grep 'backups' = 0 matches |
| 4 | 「960+ 全绿」 | **1135 tests / 3 failed**（app.spec ×2 + term-width ×1，60 文件；1132 passed） | `pnpm vitest run packages/tui/tui/tests/` 2026-08-11 实测（vitest 报告 3 failed） |
| 5 | app.ts:644 forkSession / :1027 handleApprovalRequest / :849 permission / :1140-1152 键处理 | 实际 671-680 / 1060 / 882 / 1207-1219（registry /fork 262-267；question-panel plan-review L111） | grep + read_file（2026-08-11 工作区） |
| 6 | permission 包在 core/ | 实际 `packages/interaction/permission` | glob packages/**/permission |
| 7 | 天枢 file-history.ts「仅 node:fs/crypto」 | getDiffStats 依赖 cpu-pool/cpu-tasks worker pool；核心 rewind 路径 ~200 行自包含 | 天枢 file-history.ts import 段 |
| 8 | 方案 A always-approve 无限制 | 仅对 `approval='ask'` 会话生效；'never' 策略在 decide() 分发前确定性拒绝；apiproxy.ts:919 亦监听 approval/request | user-approval decide() / api-proxy.ts:919 |
| 9 | checkpoint-policy 84 行「唯一事情是 flush」 | ≈78 行；tools/execute 分支另有 aborted 检查 | session-checkpoint-policy/src/index.ts:66-75 |

### 已坐实（无需改）的断言

- plan-mode exit_plan_mode L323/L331-345/L368、Keep planning value='Keep planning'；`fork(source, boundary?, childSessionId?)` 签名；input-handler shift_tab L75/L174/L481-482；statusline [plan] L215-218；Session append-only 无 truncate；DSH rewind 预留痕迹（ConversationContextOriginKind / history-fold.ts:56 / compact checkpoint.ts:19 / live-engine.ts:668）；天枢 tool-pipeline.ts:1290 trackEdit、context.ts:355-368 全量 reset、rewind.ts ≈184 行、persist ≈42 行。

### 未验证项（实施前需补）

- `controls.followup()` 行为（项 2 directive 依赖）
- apiproxy 审批转发的实际形态（项 4 边界）
- 天枢 opencode-tui 根 LICENSE / SOURCE-MAP.md 存在性
- `deleteFrom` 在 jsonl/sqlite 两实现包的落点与事务边界
- 3 个测试失败的具体归因（app.spec ×2 疑与未提交改动相关；term-width 是 Intl 兜底 vs env 优先的语义分歧，macOS 中文系统环境下稳定触发）

---

## 后续（独立专项，不在本批）

| 项 | 性质 | 建议文档 |
|---|---|---|
| rewind summarize 模式（LLM 摘要器） | compact 包 + LLM 调用 | 随压缩重附着批次 |
| plan 正文 markdown 渲染 + 全屏 viewer | TUI 渲染层 | 随 overlay 框架批次 |
| /fork boundary picker（从指定 turn 分叉） | TUI 交互层 | 随 history-search 批次 |
| always-approve 持久化（方案 B） | 新 cordis 服务 + settings | 独立调研 |
| 压缩后 skill/指令重附着 | compact 包改动 | 独立调研 |
| auto 模式（classifier 审批） | 分类器模型 + 审批管线 | 独立调研 |

## 参考文件索引

### 天枢 TUI（rewind 搬运源）

路径：`~/checkouts/opencode-tui`

| 功能 | 天枢文件 | 说明 |
|---|---|---|
| rewind 数据层 | `src/agent/file-history.ts`（314 行） | `FileHistory` 类：`trackEdit`/`rewindToBoundary`/`collectPostBoundaryEditIds`，自包含（node:fs/crypto） |
| rewind 持久化 | `src/agent/file-history-persist.ts`（43 行） | `file-history.json` 读写（1MB/50 条上限） |
| rewind 渲染 | `src/tui/format/rewind.ts`（183 行） | 双阶段面板（消息列表 → 回退粒度），`RewindMode` 5 种 |
| rewind 派生态 reset | `src/agent/context.ts:355` | `rewindToMessages`：截断 + token/turn/缓存全量 reset |
| rewind 钩子 | `src/agent/tool-pipeline.ts:1290` | `tools/execute` 里 `write_file`/`edit_file` 前 `trackEdit` |
| rewind 测试 | `src/tui/__tests__/format-rewind.test.ts`（107 行） | 渲染测试，原样可搬 |
| | `src/agent/__tests__/file-history.test.ts`（155 行） | 数据层测试，原样可搬 |
| undo 工具 | `src/tools/undo.ts`（82 行） | agent 自调的单步撤销（独立于 overlay rewind） |

### grok-build（设计参照）

路径：`~/checkouts/grok-build`（版本 `b13fa52`，2026-08-11 拉取）

| 功能 | grok 文件 | 说明 |
|---|---|---|
| rewind | `crates/codegen/xai-grok-pager/src/app/dispatch/rewind.rs` | rewind 分发（~600 行），`RewindResponse` 结构 |
| | `…/src/views/rewind.rs` | rewind UI（`RewindPhase` 状态机） |
| plan 审批 | `…/src/views/plan_approval_view.rs` | plan 审批 UI（Approve/Request-changes/Quit） |
| fork | `…/src/slash/commands/fork.rs` | `/fork` 参数解析 |
| | `…/src/app/dispatch/session/fork.rs` | fork 分发 |
| Shift+Tab 模式 | `…/src/app/dispatch/modes.rs:631` | `dispatch_cycle_mode_inner`：Normal→Plan→Always-Approve 循环 |
| | `…/src/app/actions.rs:440` | `CycleMode` action |
| | `…/src/input/key.rs:313` | `shift_tab_keys()` 三编码（BackTab / BackTab+SHIFT / Tab+SHIFT） |
