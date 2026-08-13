# DSH TUI 与 grok-build 功能对比 — C2

[English](dsh-tui-与grok的功能对比-c2.md) | 中文

> **标记**：C2（与 grok-build TUI 对标系列第 1 份；C1 见 [dsh-tui-与claude的对比-c1.md](dsh-tui-与claude的对比-c1.md)）**日期**：2026-08-11 **目的**：以 grok-build TUI 为参照，定位 DSH TUI 五项缺口的修复方案

## 一、grok-build 本地仓库

**路径**：`/Users/banxia/app/deepseek-tui/grok-build` **远程**：https://github.com/xai-org/grok-build **本地版本**：`b13fa52 Synced from monorepo`（2026-08-11 拉取）

更新方式：
```sh
cd /Users/banxia/app/deepseek-tui/grok-build
git pull --ff-only origin main
```

## 二、grok-build TUI 关联文件目录索引

grok-build 是 Rust monorepo。TUI 集中在单个 crate `xai-grok-pager`（ratatui + crossterm，通过 ACP 与 agent 通信）。

**crate 根**：`crates/codegen/xai-grok-pager/`

### 关联文件清单（按本次五项缺口分类）

| 缺口 | grok 文件（绝对路径） | 说明 |
|---|---|---|
| **审批 diff** | `…/xai-grok-pager/src/diff.rs` | diff 引擎：`similar` crate（Myers LCS）+ ±3 context + 双侧行号 gutter |
| | `…/xai-grok-pager/src/app/acp_handler/permissions.rs` | 审批请求入队 + title/description 构建（读 `raw_input` 取 file_path/command） |
| | `…/xai-grok-pager/src/views/permission_view.rs` | 审批 overlay 渲染（**注意：grok 审批 modal 里不放 diff**） |
| | `…/xai-grok-pager/src/scrollback/blocks/tool/edit.rs` | edit 工具 diff 块渲染（执行后）+ 全屏查看器 |
| | `…/xai-grok-pager/src/app/dispatch/permissions.rs` | 审批回复分发（select/followup/cancel） |
| **历史搜索** | `…/xai-grok-pager/src/scrollback/search.rs` | scrollback 搜索：`ScrollbackSearchIndex` + `SearchDaemon`（后台线程）+ query coalescing |
| | `…/xai-grok-pager/src/scrollback/state/mod.rs` | `ScrollbackState`：`entries: IndexMap<EntryId, ScrollbackEntry>` + `content_generation` 缓存键 |
| | `…/xai-grok-pager/src/scrollback/state/nav.rs` | 翻页导航：`page_up/down`、`half_page_up/down`、`goto_top/bottom`、`next_turn/prev_turn` |
| | `…/xai-grok-pager/src/search/matcher.rs` | `TextMatcher`：smart-case + Substring/Regex |
| | `…/xai-grok-pager/src/views/history_search.rs` | prompt 历史搜索（up-arrow recall + `/history`） |
| | `…/xai-grok-pager/src/app/agent_view/panes.rs` | scrollback 搜索接线（`open_scrollback_search`、`handle_scrollback_search_key`） |
| **输出折叠** | `…/xai-grok-pager/src/scrollback/types.rs` | `DisplayMode { Collapsed, Truncated, Expanded }` 三态枚举 |
| | `…/xai-grok-pager/src/scrollback/entry.rs` | `ScrollbackEntry` + `display_mode`/`display_mode_pinned` + `toggle_fold` |
| | `…/xai-grok-pager/src/scrollback/blocks/tool/read.rs` | read 工具：`FIRST_LINES=5, LAST_LINES=3`，截断态首5尾3 |
| | `…/xai-grok-pager/src/scrollback/blocks/tool/execute.rs` | bash 工具：truncation 配置 + 用户命令默认全展开 |
| | `…/xai-grok-pager/src/scrollback/state/selection.rs` | `toggle_fold_selected` + 滚动锚点捕获/恢复 |
| **快捷键** | `…/xai-grok-pager/src/actions/defaults.rs` | `default_actions()`：所有键位声明的单一来源（**硬编码不可配**） |
| | `…/xai-grok-pager/src/actions/mod.rs` | `ActionDef { id, default_key, alt_keys, context, category }` + 3 层 dispatch（pane→agent→global） |
| | `…/xai-grok-pager/src/input/key.rs` | `KeyShortcut { code, modifiers }` + `key!()` 宏 |
| **模型切换** | `…/xai-grok-pager/src/slash/commands/model.rs` | `/model <name> [effort]`：bare name → `SetDefaultModel`；name+effort → `SwitchModel` |
| | `…/xai-grok-pager/src/app/dispatch/settings/setters.rs` | `set_default_model`：乐观更新 + `Effect::SwitchModel`（ACP 热切） |
| | `…/xai-grok-pager/src/app/effects/mod.rs` | `Effect::SwitchModel`：发 `acp::SetSessionModelRequest` 到运行中的 agent |
| | `…/xai-grok-pager/src/app/dispatch/session/lifecycle.rs` | 切换完成处理：成功上屏 / agent-type mismatch 开 modal / 失败回滚 |
| **设置面板** | `…/xai-grok-pager/src/settings/defs.rs` | `default_settings()`：~60 项 `SettingMeta`，按 category 分组 |
| | `…/xai-grok-pager/src/settings/registry.rs` | `SettingKind { Bool, String, Enum, Int, DynamicEnum, Group }` |
| | `…/xai-grok-pager/src/views/settings_modal/` | 设置 modal UI（browse/filter/pick/edit 四态） |

### 支撑 crate

| crate | 路径 | 说明 |
|---|---|---|
| `xai-grok-pager-render` | `crates/codegen/xai-grok-pager-render/` | 主题/外观原语，`AppearanceConfig` 截断默认值（`first_lines:2, last_lines:3`） |
| `xai-grok-tools` | `crates/codegen/xai-grok-tools/` | 工具类型定义（`SearchReplaceEditDetail`、`BashToolInput`） |
| `xai-grok-shell` | `crates/codegen/xai-grok-shell/` | 配置（`UiConfig`）、agent runtime、bash 高亮 |
| `xai-grok-config` | `crates/codegen/xai-grok-config/` | 配置 schema |
| `xai-ratatui-inline` | `crates/codegen/xai-ratatui-inline/` | 内联渲染 widget |
| `xai-ratatui-textarea` | `crates/codegen/xai-ratatui-textarea/` | prompt 文本编辑 widget |

## 三、功能对比（DSH 现状 vs grok-build）

### 审批 diff 预览

| 维度 | grok-build | DSH 现状 |
|---|---|---|
| 审批 modal 里放 diff？ | **不放**（diff 只在执行后渲染） | 无 diff，裸 `⚠ 允许执行 X？[y/N]` |
| diff 算法 | 真 Myers LCS（`similar` crate） | 无（`format/diff.ts` 只处理 unified-diff 文本，不能从两串生成 diff） |
| ±context 裁剪 | ±3 context + 双侧行号 gutter | 无 |
| 数据通路 | `req.tool_call.fields.raw_input` 取参数 | `PendingApprovalRequest` 把 `callId` 丢了（运行时有、类型没接） |

**关键差异**：grok 认为「审批时不需要 diff，执行后看就行」。DSH 的用户痛点恰恰相反——**盲批是信任断点**。我们反其道，在审批提示上方放内联 diff。

### 历史搜索 / scrollback pager

| 维度 | grok-build | DSH 现状 |
|---|---|---|
| 对话搜索 | 有：后台线程 + query coalescing + smart-case + 正则 | `scrollback-transcript.ts` 有搜索纯函数（`searchTranscript`/`findNextMatch`），**但没接线** |
| 翻页 | 有：PageUp/Down、半页 Ctrl-U/D、goto top/bottom、turn 间跳转 | 无（靠终端原生 scrollback） |
| prompt 历史 | 有：up-arrow recall + `/history` + nucleo 模糊 | 有：input-line 的 up/down 历史（基础） |
| 搜索线程 | 后台 `SearchDaemon`（百万行多 agent 场景） | 不需要（DSH 单会话规模小，主线程同步够） |

**关键差异**：grok 的后台索引是为多 agent 百万行场景。DSH 单会话规模远小，**主线程同步搜索即可**——解析层已就绪，只差 overlay 容器 + key routing。

### 输出折叠

| 维度 | grok-build | DSH 现状 |
|---|---|---|
| 折叠态 | 三态 `Collapsed|Truncated|Expanded` | 二态折叠/展开（tool-card 内建） |
| read 工具 | `FIRST_LINES=5, LAST_LINES=3`，截断态首5尾3 | **无**（read_file 大输出全刷屏） |
| bash 工具 | `first_lines:2, last_lines:3`，用户命令默认全展开 | 有 `collapsed-bash.ts`（多调用 group 折叠，非单输出截断） |
| 搜索/列表 | per-tool 配置 | 无 |
| 展开交互 | toggle fold + 锚点保持 | 有（tool-card 二态切换） |

**关键差异**：DSH 有 group 折叠（多个工具调用合并），缺**单工具大输出截断**。read_file 读回 500 行直接刷屏。

### 快捷键

| 维度 | grok-build | DSH 现状 |
|---|---|---|
| 可配置？ | **不可配**（文档明确：「Bindings are built in and cannot currently be remapped」） | 不可配（硬编码 switch） |
| 架构 | `ActionDef` 注册表统一三处消费（shortcuts-bar / command-palette / key-dispatch） | switch 散落在 input-handler / app.ts |
| vim 模式 | `[ui].vim_mode` 开关（scrollback 导航 j/k/h/l） | input-line 内建 vim 状态机（normal/insert/visual） |

**关键发现**：grok 自己也硬编码不可配。但它用 `ActionDef` 注册表消除了散落的 switch——这是架构改进，不是功能增加。**第一批不做**（用户确认 ROI 不如其余四项）。

### 模型切换

| 维度 | grok-build | DSH 现状 |
|---|---|---|
| 切当前会话？ | 是（ACP `SetSessionModelRequest` 热切） | **否**（只切默认 `saveSelection`，影响新会话） |
| 热切时机 | `model_switch_pending` 阻塞队列，当前请求完成后生效 | 无热切机制 |
| agent-type 不匹配 | 开 modal 问是否新开会话 | 不区分 agent type |
| reasoning effort | `/model <name> <effort>` 可带 effort | 不支持热切 effort |

**关键差异**：grok 通过 ACP 协议热切。DSH 无 ACP，但 `ModelSelectionRef`（model-selection.ts:20）是**可变对象**——持有 ref、改 `ref.current`，下一次 agent 步进（prompt assembly）自动生效。**纯装配层，不碰 agent-loop**。

## 四、五项缺口修复方案

### 项 1：审批 diff 预览（内联在 y/N 上方）✅ 用户确认内联 — **✅ 已完成（2026-08-11）**

**目标**：审批 `edit`/`write_file` 时，在 `⚠ 允许执行 X？[y/N]` 上方渲染 ≤12 行 old/new diff 块。

**设计决策**：
- 反 grok 之道（grok 审批 modal 不放 diff），因 DSH 痛点是盲批
- diff 算法用真 LCS（对齐 grok 的 Myers），TS 用 `diff` npm 包 `diffLines`
- 数据通路：`PendingApprovalRequest` 加 `callId?` → transcript `view.tools.find(t => t.callId === callId)` → `parseToolArguments` → 得 old/new
- 审批期间键锁（只 y/N/Esc/Ctrl+C）→ diff 必须**无翻页全可见**，硬限 12 行

**改动文件**：
1. 加依赖 `diff` npm 包
2. 新建 `src/format/permission-diff.ts`（~140 行）
   - `formatPermissionDiff(input): string[] | null`
   - edit 类 → `formatEditDiff`：`diffLines` → Change[] → 染色（+绿/-红/=dim）→ ±3 context → 双 gutter → 12 行折叠
   - write 类 → `formatWritePreview`：path + 前4行
   - 非编辑工具 → null
3. 改 `src/ui/app.ts`：`PendingApprovalRequest` 加 `callId?`；`renderLive` 审批行块加 diff 查找
4. 新建 `tests/permission-diff.spec.ts`

### 项 2：历史搜索 overlay ✅ 用户确认需要 — **✅ 已完成（2026-08-11）**

**目标**：`/search` 或 `Ctrl+F` → 全屏 alt-screen overlay，smart-case 搜对话历史，n/N 跳转。

**设计决策**：
- **不引入 Worker**（DSH 单会话规模够小；grok 的后台索引是为百万行多 agent）
- 复用 `scrollback-transcript.ts` 已有纯函数（`searchTranscript`/`findNextMatch`/`estimateMessageRows`）
- overlay 用现有 `OverlayController`（已用于 command-palette/keymap）
- smart-case：查询含大写→敏感，否则不敏感（grok 的一行规则）

**改动文件**：
1. 新建 `src/format/history-search-overlay.ts`（~120 行）implements `OverlayRenderer`
   - 状态：query / matches / current / messages
   - `render`：搜索栏（query + 匹配数 N/M）+ transcript 滚动（匹配高亮居中）+ key hints
2. 改 `src/ui/app.ts`：注册 overlay + key routing（可打印→query；Enter→搜索；n/N→跳转；Esc→退出）
3. 新建 `tests/history-search-overlay.spec.ts`

**范围限制**：同步搜索、不做正则（先 smart-case 子串）

### 项 3：read/search 工具输出折叠 — **⏸️ 无需实现（2026-08-11 调研确认）**

**调研结论**：DSH 现状已覆盖该缺口——`read_file`（read 族）默认截断为头 3 + 标记 + 尾 5（tool-card.ts getDefaultMaxLines read=8 行），`grep/glob`（find 族）6 行，`truncation-marker.ts` 的「… +N 行 · ctrl+o 展开」标记与 `isToolCardTruncated` 展开判定齐备，tool-card.spec.ts 已有头尾预览测试。C2 草案的"read_file 大输出全刷屏"与现状不符（现有阈值 8 行比草案建议的 50 行更激进）。按水平复用纪律不重复实现。

**设计决策**：
- DSH 已有二态折叠（tool-card 内建），不引入 grok 的三态枚举
- 复刻 `collapsed-bash.ts` 的 `LiveRegionLine` 返回形态（装配层零适配）
- per-tool 阈值：read_file >50 行、grep/glob >20 条

**改动文件**：
1. 新建 `src/format/collapsed-read-search.ts`（~150 行）
   - `shouldCollapseReadSearch(toolName, outputLength): boolean`
   - `formatCollapsedReadSearch(opts): string[]`（首 N + 「… +X 行」+ 尾 N）
   - `READ_THRESHOLDS: Record<string, { lines, first, last }>`
2. 改 `src/format/tool-card.ts`：result 区命中阈值且折叠态 → 用 `formatCollapsedReadSearch`
3. 新建 `tests/collapsed-read-search.spec.ts`

### 项 4：/model 当前会话热切 — **✅ 已完成（2026-08-11）**

**目标**：`/model <provider/model>` 热切当前正在跑的会话的模型。

**设计决策**：
- DSH 无 ACP，但 `ModelSelectionRef` 是可变对象——持有 ref、改 `ref.current`
- `installModelSelection` 每次 prompt assembly 读 `selection.current` → 下一次 agent 步进自动生效
- 当前问题：app.ts:615/652 传字面量 `{ current: selection, assembled: undefined }`，没持有 ref

**改动文件**：
1. 改 `src/ui/app.ts`：
   - 加 `private modelRef: ModelSelectionRef | null = null`
   - newSession/switchSession：字面量改为持有对象 `this.modelRef = { current: selection, assembled: undefined }`
   - 新增 `switchLiveModel(selection)`：`if (this.modelRef) this.modelRef.current = selection`
2. 改 `src/commands/registry.ts`：`/model` handler 加调注入的 `switchLiveModel` 回调
3. 改 `tests/commands.spec.ts`

**范围限制**：热切在下一次 agent 步进生效（非立即中断）；不做 effort 热切、不做 agent-type mismatch modal

### 项 5：快捷键可配置 — 移出本批 ❌ 用户确认不做

grok-build 自己也硬编码不可配。ROI 不如其余四项。留后续。

## 五、提交序列（实际执行，2026-08-11）

1. `15173d4` `feat(tui): C2-1 approval diff preview — inline unified diff above y/N via callId`（项 1）
2. `0645886` `feat(tui): C2-4 /model hot-swap current session via ModelSelectionRef`（项 4）
3. `ec724ac` `feat(tui): C2-2 history search overlay — Ctrl+F smart-case search with n/N jump`（项 2）
4. 项 3：调研确认被既有 tool-card 截断覆盖，未实现（见上）
5. 项 5：用户确认不做

**验收状态（2026-08-11）**：三项已实现功能的用户级行为验收 **blocked**——执行环境无交互式 TTY（raw-mode 不可用，字节注入不可靠）。已获真实装配探针证据：TUI profile启动 ✓、恢复会话列表与 fork 血缘渲染 ✓、Ctrl+F alt-screen 进出序列（?1049h/?1049l）✓。功能级验证以包级测试为准（RED→GREEN），真实装配的完整用户验收待终端环境执行。

## 六、跨项纪律（执行结果）

- **SOURCE-MAP.md**：未更新——permission-diff.ts 为原创模块（非 Tianshu 移植），文件头声明 grok 参考来源；不在 SOURCE-MAP 映射范围
- **依赖**：仅加 `diff@^9.0.0`（项 1，自带类型）；项 2/4 零新依赖 ✅
- **测试**：项 1 spec 7 用例 + 项 4 6 用例 + 项 2 11 用例，全 RED→GREEN ✅
- **覆盖率补账（2026-08-11，`de64c35`）**：permission-diff.spec 扩至 16 用例（write_file/edit_file 全分支、缺参组合、合法 JSON 非对象、muted 缺失、未知工具），history-search-overlay.spec 扩至 19 用例（clear/onDeactivate/空 text/theme 显式/无匹配 render/小高度/空消息集）。终值（vitest v8）：permission-diff **statements 100% + branches 100%**（原 68.75%/52.08%）；history-search-overlay **statements 100%**（原 88.13%），branches 96.9%——1 个隐式分支未命中，该文件 branchMap 全部无 loc（v8+esbuild 对 TS class 的 sourcemap 归因丢失），尝试 3 测试场景 + if-break改写无法定位，记录为已知工具限制（perFile branches 门禁受并行会话未提交文件影响本就红着，非本项新增阻塞）
- **类型**：逐项 scratch 单查 0 错误（全量 tsc -b 受并行会话占用与既有错误阻塞）

## 七、grok-design 借鉴决策总表

| grok 设计 | DSH 是否采用 | 理由 |
|---|---|---|
| 审批 modal 不放 diff | ❌ 反其道 | DSH 痛点是盲批，内联 diff 建立信任 |
| 真 Myers LCS diff | ✅ 采用 | TS 用 `diff` npm 包对齐 |
| 后台线程搜索 | ❌ 简化 | DSH 单会话规模小，主线程同步够 |
| smart-case 匹配 | ✅ 采用 | 一行规则，用户期望行为 |
| 三态折叠枚举 | ❌ 简化 | DSH 二态够用，与现有 tool-card 一致 |
| per-tool first/last 阈值 | ⏸️ 已覆盖 | 调研确认 DSH 既有 tool-card 截断（read 8 行/find 6 行）比 grok per-tool 配置更激进，无需新实现 |
| `ActionDef` 注册表 | ⏸️ 留后续 | 第一批不动快捷键 |
| ACP 热切模型 | ❌ 不可用 | DSH 无 ACP，但 `ModelSelectionRef` 可变对象达成同效果 |
| agent-type mismatch modal | ❌ 不做 | DSH 不区分 agent type |
| 设置面板 registry | ⏸️ 留后续 | DSH 已有 /config 面板（T3.2），深度不足但够用 |
