# DSH × 天枢融合演进迭代记录（2026-08-09 快照基线起）

[English](dsh-融合演进-迭代记录.md) | 中文

> **定位**：本文件记录从快照分支 `snapshots/20260809T140917Z-a6bb5a95ba`（2026-08-09 官方快照）开始的本地演进，供后续仓库说明使用。**性质**：迭代能力清单与演进证据（提交哈希为验证记录），不是决策记录（→ Agent Notes）也不是系统地图（→ [architecture.md](architecture.md)）。**覆盖**：140 个本地提交（2026-08-09 → 2026-08-11），新增 4 个包 + 核心会话模型扩展。**关联**：[路线图](dsh-tui-next-phase.md) ｜ [C1 对标](dsh-tui-与claude的对比-c1.md) ｜ [C2 对标](dsh-tui-与grok的功能对比-c2.md) ｜ [C3 增强方案](dsh-tui-增强方案-c3.md) ｜ [C4 拆分方案](dsh-tui-拆分方案-c4.md) ｜ [交接文档](../.agents/notes/implemented/process/2026-08-10-tui-handoff.md)

## 基线画像（快照分支 = 2026-08-09）

演进起点是**无头 agent 核心 + 能力插件族**的 harness：

- **Spine**（`packages/core/`）：sessions / system-prompt / tools / agents / agent-default-model / agent-loop
- **能力族**（capability seams）：llm、bash、subprocess、pty、fs（fs/fs-local/fs-policy/fs-sandbox/tool-fs/tool-fs-search/tool-str-replace-editor）、lsp、skill、web、compact、subagent、workflow、tasks、plan、goal、todo、code-runtime、sandbox
- **数据平面**：session（持久化 JSONL/SQLite）、session-query、session-title、settings、credentials、storage、workspace
- **前端平面**：仅 ACP 自动化服务端、Web GUI（host/client）、scaffold（CLI 启动器）——**无交互式终端前端**
- **guard 组**：仅 2 包（repeat-tool-guard 提醒、timeout-policy 超时）——行为卫生只有"提醒"，无纪律机制
- **会话模型**：session log **append-only**（唯一操作是追加；无回退/截断）

## 演进总览（140 提交 = 4 新包 + 核心扩展）

| 新增 | 位置 | 本质 | 演进证据 |
|---|---|---|---|
| `dsh-tui` | packages/tui/tui | 全新第三前端平面（交互式终端 UI） | `f825fd1` 移植天枢渲染核心起，457 次提交改动 |
| `dsh-fs-snapshot` | packages/fs/fs-snapshot | 工作区文件状态快照（rewind 物理支撑） | `277657e`/`c6764f5`/`42c4da3` |
| `dsh-evidence-gate` | packages/guard/evidence-gate | 行为纪律机制（RED→GREEN 验证门） | S1–S6 系列，`1db1b35` 起 |
| `dsh-agent-router` | packages/guard/agent-router | 行为纪律机制（失败预测→路由升级→原生子代理派发） | S7–S8 系列，`d458e36` 起 |
| `Session.truncate` | packages/session | 核心模型：append-only → 可回退 | `62d1e76` |
| `deleteFrom`/`truncateStored` | packages/session/session-persistence | 持久化层跨 reload 截断 | `e4a057e` |
| examples/tui + headless-agent e2e | examples/ | 真实装配验证面 | `4ca8b68` 起 20 次改动 |

提交分布（按顶层路径）：tui 457、guard 70、fs 13、session 10、examples/headless-agent 20、workflow/subagent/scaffold/hooks/boot 少量、Agent Notes 44 条。

## 一、前端平面：TUI（最大能力块）

### 架构定位

`oh-my-tianshu --profile` 的 TUI 层，bundle patch 骑在 dsh-base 之上（稳定插件 id `tui-runner`）；渲染核心移植自天枢 Tianshu 终端引擎（Apache-2.0，逐文件溯源见 SOURCE-MAP）。**纯展示层**：所有状态经 session 事件与投影总线到达，不发明新事件类型、不含 agent 逻辑，遵守 "Model-visible ⟺ logged"。分层：`src/engine/`（终端渲染引擎移植层）、`src/ui/app.ts`（TuiApp 装配）、`src/adapter/`（dsh 服务到 `TuiPort` 的适配）。

### 交互平面补齐（基线缺口）

- 注册 `userInteraction` provider → 终端内答题（问题面板、plan-review 🧭 决策卡）
- 订阅 `approval/request` → 挂起审批卡片 + y/N/Ctrl+C 结算 + 审批 diff 预览（inline unified diff）
- 命令注册于 `ctx.commands`：slash 命令系统经 Cordis 服务注册表发现，无需 model turn

### 终端内能力全景

- **内部状态可视化**（Phase 5）：六阶段工作流指示器、实时活动标签、metrics glance-bar（model/turn/耗时）
- **交互效率**（Phase 6）：slash 命令系统、中轮转向（`agent.steer()` 入口）、@-路径补全、外部编辑器（Ctrl+O）、Vim 模式（normal/insert/visual）、Ctrl+F 历史搜索（smart-case + n/N 跳转）
- **工具可视化**（Phase 7）：工具运行计时、家族着色、并行分组折叠、流式尾巴、turn 摘要
- **会话管理**：/fork /branch（`ctx.sessions.fork`）、restore-session 恢复面板、/model 热切换、**/rewind 两阶段回退**（消息列表 → 粒度，联动 fs-snapshot + Session.truncate）
- **模式系统**：Shift+Tab 三态循环（normal/plan/always-approve）、plan 审批门 + request-changes 反馈路径（`f` 键输入反馈文本；修复了 TUI 提交形状与 provider 契约的断层——真实装配下原会 TypeError）
- **体验增强**（Phase 9）：@mention 解析（用户侧摘要展开、cwd 边界）、流利度控制（stale 提示分级上屏）、命令面板（Ctrl+P）、CJK 宽度感知

### 架构模式演进（C4 拆分）

app.ts 1727 行单体 → Wave 1 Question/Approval controller 提取（带单测）→ Wave 2 live-panels 纯函数化（renderLive 组合器）→ Wave 3 孤儿 controller 收敛（与内联逻辑逐 case 对比，语义不等则删除，不留"提取一半"）+ dispose/detach 生命周期纪律（订阅台账平衡断言：`ctx.on` 全部收集 disposer、挂载卸载对称释放、`??` 短路陷阱已修）。配套 `scripts/verify-source-budgets.ts` 行数棘轮门禁（app.ts 预算 1831 行）。

## 二、行为卫生：guard 家族 2 → 4 包

基线 guard 只做"提醒"；演进增加**机制性纪律**，把 prompt 层要求变成插件层事实：

- **evidence-gate（验证门）**：bugfix 任务的 RED→GREEN 纪律——L1 编辑门（无 RED 复现拦首次编辑 + 精准探针建议：测试路径与期望结果）、RED 三规则（failed 记证据；passed 需先有 RED；blocked ≠ 已证）、TDD 门（连续编辑 ≥3 次无验证 → suggest/enforce）、L2 final gate（continue_once / honest_blocked 披露）；**验证归账自动从 `session/event` 的 tool/call→tool/result 配对检测**（命令文本启发式，零测试框架耦合）；适配原生 `str_replace_editor`（写命令拦截、view 读放行）
- **agent-router（路由层）**：prediction 累计器（窗口 10 滑动，错误率 0.4/0.6/0.8 → hint/gate/escalate 三级干预；连续 3 次正确 → tipping point 重置）+ 确定性路由表（escalate → delegate verifier；gate + 探针冷却耗尽 → delegate code_scout；义务未决 + 零验证 → self 先写探针）+ **dsh 原生子代理派发**（`ctx.agents.create` → followup 注入任务 → whenIdle 等待 → dispose；结果经 session/event 自动归账回 evidence-gate，零新通道）；配置化降级（dispatchEnabled: false 只决策不回显）

两者通过 `examples/headless-agent` 真实装配 e2e（Loader load、任务边界接线、tools via reflect.get、exit-code 失败检测）验证，每包配 `invariant.ts` 运行时不变量 companion。

## 三、核心模型扩展：append-only → 可回退

- `Session.truncate`：事件日志回退 + 派生状态重置（surface/header/context folds）
- `session-persistence`：`deleteFrom` 后端（JSONL/SQLite）+ `truncateStored` 协调器——**跨 reload 持久化截断**

把 session 从"只能追加"扩展为"可回退重放"，是 /rewind 的地基（file+session 双轨回退）。

## 四、文件状态快照：fs-snapshot

- `tools/execute` waterfall 注入 trackEdit 钩子：写工具（`str_replace_editor` 写命令 / `write` / `edit`）执行**前**全文快照（FileHistory 移植自 opencode-tui，Apache-2.0）
- 与 checkpoint-policy **正交**：checkpoint 是事件日志持久化（防崩溃丢 turn），本插件是文件内容快照（供 rewind 文件回退）
- 消费面：`HISTORIES_KEY` per-session FileHistory 索引、`rewindToBoundary(boundaryId)` 恢复（`backupFileName === null` 表示当时不存在 → unlink）；快照上限 100 条，溢出淘汰最旧
- 已声明边界：bash 直写不可见（与 evidence-gate 编辑门同款盲区）、快照索引内存态（跨重启不重建，持久化索引为延期工作）、默认备份目录在 tmpdir

## 五、真实装配验证面

- `examples/tui`：可运行 TUI 组合（settings + credentials + 真实 DeepSeek adapter + 预创建 `main` agent + `tui-runner` 插件，含 fs-snapshot 接线）
- `examples/headless-agent`：成为 guard 家族真实装配 e2e 验证场（evidence-gate S5/S6、agent-router S7/S8：子代理 mock round-trip、verifier 真实 turn 派发、fail-loop mock）
- 覆盖率门全仓攻坚：119 违规 → 约 10 条；app.ts 覆盖缺口清零；**typecheck 债务 86 → 0**（tui/subagent/workflow/hooks-claude/fs-snapshot 五包测试层，零行为变化）

## 六、架构判断

1. **全部走插件机制，agent-loop 零改动**——新前端（TUI）、新纪律（guard×2）、新持久化（fs-snapshot）都是既有扩展点（`ctx.on` / waterfall / registerProvider / inject）的消费者，符合 "everything is a plugin"。
2. **前端从"双平面"变"三平面"**：ACP（自动化）/ Web（浏览器）/ TUI（终端）；TUI 是唯一同时补齐 `userInteraction` 与 `approval/request` 双挂起通道的前端。
3. **行为卫生从"提示"升级为"机制"**：evidence-gate + agent-router 形成"失败预测 → 验证纪律 → 路由升级 → 事件归账"闭环，归账复用 session 事件流（零新通道）。
4. **数据模型从 append-only 扩展为可回退**：truncate 落到会话、持久化、文件三个层面。

## 7. 类型感知 lint 债：183 → 0 与 lint-budgets 棘轮

TUI 线的类型感知 oxlint 债（2026-08-11 盘点 183 处，实测时 171 处）现已**清零**：`pnpm run lint` exit 0，全仓 oxlint 无错误。清理按规则族推进（每族一个裁定——unbound-method 42、no-unnecessary-condition 22、no-unsafe-\* 43、restrict-plus-operands 11、no-unnecessary-type-conversion 16 及小族），边界真实处修类型（终端能力/环境/模型工具 JSON），类型诚实处才删守卫（`9a76921`）。基线 `ctx.emit` 重载在 guard/hooks/core 测试的 17 处错（快照即有）一并裁定：测试替身按设计宽松派发，用 `@ts-expect-error` + 理由注释表达（`as any` 被 `typescript/no-explicit-any: error` 拦截）。

为防债务静默回归，`scripts/verify-lint-budgets.ts` + `scripts/lint-budgets.manifest.json` 对 `typescript/*` 族逐文件设棘轮——当前空表即**全仓零容忍**，任何有意新增债须在同一 PR 加 manifest 条目（`3c82af2`）。棘轮按文件归因违规（全仓 lint 门禁仍是全部诊断的权威），由 `scripts/verify-lint-budgets.spec.ts` 验证。

## 当前状态与遗留债务

- 测试基线：TUI 套件 1292 通过 / 2 todo（2026-08-12）；lint 涉及的包（tui、subagent、guard、hooks-claude、core/scope）合计 1776 通过 / 2 todo，101 文件
- **工作区干净**；T2.3 监听器生命周期收尾、source-budgets 门禁接线、fs-snapshot → tui example 接线、tui 组 invariant/tsconfig 收录均已落地（abccfde/e37003b 及相邻提交）
- **174 个本地提交未推送**（分支 `snapshots/20260809T140917Z-a6bb5a95ba` 领先 origin）
- **文档欠账已清**：`packages/guard/README.md` 组表已收录 agent-router/evidence-gate，`docs/architecture.md` 已反映 truncate 与 guard 扩展（均自 `e37003b`）；本迭代记录是存续的剩余项
- 覆盖率门约 10 条残留；⚠ `tsc -b --force` 本机卡死（疑 tsbuildinfo 缓存损坏，需先清 `*.tsbuildinfo`，`-b` 必须串行防死锁）；清缓存后普通 `tsc -b tsconfig.host.json` 通过
- 已知边界：用户级 TTY 验收受阻（执行环境无交互式终端），行为证据以单测与真实组合测试为准
