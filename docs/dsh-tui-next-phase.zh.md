# DSH TUI 下阶段增强计划

[English](dsh-tui-next-phase.md) | 中文

> 对比基准：天枢 TUI（opencode-tui/src/tui/）独立演进的功能DSH TUI 是早期分叉版本，此后天枢累积了三个代际的 TUI 能力

## 反目标

- 不做多 agent 编排可视化（council/team/worker 面板）—— DSH 是单 agent 设计
- 不做桌面端功能（Tauri/updater）—— DSH 是纯终端
- 不改变 Cordis 插件架构——所有增强在 TUI 插件内闭环

> **演进记录（2026-08-11）**：反目标第一条已随 agent-router 引入而过期。agent-router（S7/S8）为 DSH 带来 verifier 子代理调度后，"单 agent 设计"前提不再成立，TUI Phase 2（`9a038de`）配套实现了 /subagents 委派树与/workflow 运行时面板。后续新增多 agent 可视化不再视为反目标违反；本文档 Phase 5-9 的移植范围（单 agent 体验增强）不受影响。

## Phase 5：Agent 内部状态可视化

DSH 当前 statusline 只显示 idle/running。天枢的 cockpit 让用户看到 agent 认知过程的内部状态。

### 5.1 六阶段工作流指示器

> ✅ 已完成 — `1c7c0e4` feat(tui): workflow phase indicator and live activity projection

天枢 `phase-tracker.ts` 实时展示 agent 处于工作流的哪一步（理解/调研/拆解/实施/验证/收尾）。DSH 的 agent 循环有对应的阶段概念（session events），需要在 statusline 旁边加一个阶段指示器。

实现：消费 `agent/status` 事件 + session log 中的 turn 结构，推断当前阶段。不需要 agent 侧修改——纯 TUI 投影。

### 5.2 实时活动标签

> ✅ 已完成 — `1c7c0e4`（与 5.1 同提交）

天枢 `activity-status.ts` 显示 "正在读取文件…" "正在搜索…" "正在执行 bash…" 等动态标签。DSH 的 `tool/call` 事件已经包含工具名和参数，TUI 可以在工具执行时在 statusline 展示当前活动。

### 5.3 Metrics glance

> ✅ 已完成 — `5abf6e2`+`1ae0af3`+`9775e93`：glance-bar（formatTokenCount/段组装/窄宽 drop）+ metrics-glance-controller（首推/节流）+ TuiApp 装配（状态/错误行派生 + metrics 行：model/turn/耗时）。token 数据源不在 TUI 会话事件流（council 反证），可得段才渲染。

天枢 `metrics-glance-controller.ts` 在 TUI 底部或侧栏展示 token 用量、缓存命中率、本轮耗时。DSH 的 session log 和 LLM usage 数据已有，只需 TUI 层展示。

## Phase 6：交互效率

### 6.1 Slash 命令系统

> ✅ 已完成 — `426607d` feat(tui): slash command system with Cordis service registry

天枢 `slash-commands.ts` + `command-palette.ts`。DSH TUI 输入框目前是纯文本。加入 `/` 前缀触发命令面板：

- `/model` — 切换模型
- `/theme` — 切换主题
- `/session new|list|switch` — 会话管理
- `/steer <text>` — 中轮转向（见 6.2）
- `/clear` — 清空当前对话
- `/compact` — 触发上下文压缩

实现：`input-controller.ts` 中检测 `/` 前缀，弹出 overlay 面板。命令注册表用 Cordis 服务模式，其他插件可注册自定义命令。

### 6.2 中轮转向

> ✅ 已完成 — `553c0f3` feat(tui): mid-turn steering entry and statusline production mount

天枢 `steer-buffer.ts` + `steer-intent.ts`。DSH 的 `agent.steer()` API 已存在（`adapter/send.ts`），但 TUI 没有暴露入口。需要：

- 输入框内 `/steer <text>` 或快捷键（如 Ctrl+T）触发
- 转向文本以不同颜色/前缀展示在对话流中
- 支持优先级（now/next/later）

### 6.3 文件路径自动补全

> ✅ 已完成 — `34f3787` feat(tui): @-path tab completion (Phase 6.3)

天枢 `file-completer.ts`。DSH 的 `.rivet/tui-source/tui/file-completer.ts` 保留了原始实现但被排除。需要恢复并接入输入框的 Tab 补全。

### 6.4 外部编辑器

> ✅ 已完成 — `92c2d06` feat(tui): external editor (Phase 6.4) — Ctrl+O opens $EDITOR on input line

天枢 `external-editor.ts`。按 Ctrl+O 打开 `$EDITOR`（Config editorKey 可配）编辑当前输入内容，保存退出后内容回到输入框。Ctrl+E 与 input-line 的 moveEnd 冲突，故缺省 Ctrl+O。

### 6.5 Vim 模式

> ✅ 已完成 — `2c82467` feat(tui): vim mode wiring (Phase 6.5) — mode label in live region

天枢 `vim-mode.ts`。input-line.ts 内建 vim 状态机（normal/insert/visual），TuiRunnerConfig vimEnabled 开关接线 + 模式标签渲染（-- NORMAL -- / -- VISUAL --）。

## Phase 7：工具执行可视化深度

### 7.1 工具运行计时

> ✅ 已完成 — `59a6591` feat(tui): tool family coloring and run timing

天枢 `tool-elapsed.ts`。工具调用时展示实时计时器："bash 运行中… 12.3s"。DSH 的 `tool/call` 和 `tool/result` 事件有时间戳，TUI 可以在期间渲染计时。

### 7.2 工具家族着色

> ✅ 已完成 — `59a6591`（与 7.1 同提交）

天枢 `tool-family.ts`。按功能家族给工具配色：文件操作（蓝）、shell（黄）、搜索（绿）、编辑（紫）、网络（青）。DSH 的 `format/tool-card.ts` 已经渲染工具卡片，加入家族信息即可。

### 7.3 并行工具分组折叠

> ✅ 已完成 — `da0adac` feat(tui): parallel tool group folding (Phase 7.3)

天枢 `tool-group-controller.ts` + `tool-accumulator.ts`。当 agent 同时发起多个独立工具调用时，DSH 的 TUI 目前逐个展示。天枢将它们分组折叠为一组，显示 "3 个工具并行执行中…"，用户可以展开查看每个。

## Phase 8：审批流 UI

> ✅ 已完成 — `1ae0af3`（overlay/命令面板）+ `2acc509`（approval answerer：ctx.on('approval/request') 注册，y/N 内联确认，waterfall next() 委托）

天枢 `approval-intent-controller.ts`。DSH 的审批通过 Cordis 策略插件处理，TUI answerer 已注册：工具被策略拦截时渲染 "⚠ 允许执行 `rm -rf ./build`？[y/N]" 内联提示，y 放行 / n 拒绝 / Ctrl+C 取消，不阻断终端渲染流。

## Phase 9（可选）：其他体验增强

- **@mention 解析** ✅ 已完成 — `5abf6e2`（src/mention-parser.ts）+ `fccbe4b`（装配：用户侧摘要展开，cwd 边界/截断/降级）
- **会话恢复面板** ✅ 已完成 — `5abf6e2`（src/restore-session.ts 投影）+ `af73fa2`（attach 时 scrollback 列出可恢复会话）
- **Turn 摘要** ✅ 实现落地 — `5abf6e2`+`1ae0af3`（turn-summary model + format 双路径，按 staged 契约）
- **流利度控制** ✅ 已完成 — `51feb85`（fluency-policy/hook 移植，ActivityPhase 适配）+ `835638e`（TuiApp 装配：tool 事件 → stale 提示分级上屏）

- **@mention 解析**（`mention-parser.ts`）：输入 `@filename` 自动展开文件内容摘要
- **会话恢复面板**（`restore-session.ts`）：启动时列出可恢复的会话
- **Turn 摘要**（`turn-summary.ts`）：每轮对话后的工具调用统计摘要
- **流利度控制**（`fluency-hook.ts`）：防止 LLM 输出过快导致终端闪烁

## 架构约束

- 所有增强在 `packages/tui/tui/src/` 内实现，不修改 core/agent/session 包
- 数据源：已发布的 session events + Agent public API（followup/steer/cancel/whenIdle）
- 审批流例外：需与 Cordis 策略系统协商一个 TUI 可见的挂起状态（当前策略是同步返回 allow/deny，无挂起态）
- 不发明新的事件类型——所有展示从已有事件投影

## 优先级排序

1. **Phase 6.2 中轮转向** — Agent API 已有，只缺 TUI 入口，ROI 最高
2. **Phase 5.1 阶段指示器 + 5.2 活动标签** — 用户最直观感受到的提升
3. **Phase 6.1 Slash 命令** — 改变交互范式
4. **Phase 7.1 工具计时 + 7.2 工具家族着色** — 视觉深度
5. **Phase 6.3 文件补全** — 每天省用户时间
6. **Phase 8 审批流 UI** — 安全性提升
7. **其他** — 按需

## 生态整合线（Phase 1-3，2026-08-11）

> 与 Phase 5-9（体验增强移植）并行的第二条线：把 TUI 与 DSH 守卫系统（evidence-gate / agent-router）在交互层接通。规划文档未单独成册，以提交记录为权威。

| 阶段 | 提交 | 内容 |
|---|---|---|
| Phase 1 | `6c9cf7c` | native wiring — projection bus（5 域）、/status 面板、/goal 命令、[plan] 徽章 |
| Phase 2 | `9a038de` | multi-agent visualization — /subagents 委派树、/workflow 运行时面板、/tasks 后台任务（配套 agent-router S7/S8，见反目标演进记录） |
| Phase 3 | `ae268fa` | interaction seams — structured questions、/config 面板、/skills 浏览器 |

## 对标整合线（A1/A3/C2 系列，2026-08-11）

> 第三条线：以 Claude Code（C1 文档）与 grok-build（C2 文档）为对标，补 TUI 交互缺口。缺口清单与逐项决策见 `docs/dsh-tui-与claude的对比-c1.md`、`docs/dsh-tui-与grok的功能对比-c2.md`。

| 项 | 提交 | 内容 |
|---|---|---|
| A1 模式系统 | `4dca7d4` | slash 通道 fallback 到 CommandService（/plan 可达）+ statusline [plan…] pending |
| A3 会话分叉 | `ee23cd9` | /fork /branch 命令（SessionStore.fork + switchSession，parentSession 血缘） |
| C2-1 审批 diff | `15173d4` | 审批 y/N 上方内联 unified diff（createTwoFilesPatch + formatDiff 复用，+diff 依赖） |
| C2-4 /model 热切 | `0645886` | ModelSelectionRef 可变 ref，下一次 agent 步进生效 |
| C2-2 历史搜索 | `ec724ac` | Ctrl+F 全屏 overlay，smart-case，n/N 循环跳转 |
| C2-3 输出折叠 | —（无需实现） | 调研确认既有 tool-card 截断（read 8 行/find 6 行）已覆盖 |
| C2-5 快捷键配置 | —（不做） | 用户确认；grok 自身也硬编码 |

**验收状态（2026-08-11）**：用户级行为验收 blocked（执行环境无交互式 TTY）；真实装配探针证据：TUI profile 启动 ✓、fork 血缘渲染 ✓、Ctrl+F alt-screen 序列 ✓。完整验收步骤见各 Agent Note（`.agents/notes/implemented/feature/2026-08-11-tui-*.md`）。
