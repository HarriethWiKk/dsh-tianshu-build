# DSH 能力复盘 + TUI 渲染对标 — C6

[English](dsh-能力复盘与对标-c6.md) | 中文

> **Tag**: C6（综合复盘；合并并更新 C1/C2/C5——Claude Code 基准 [C5](dsh-与claude的对比-c5.md)、grok 功能对比 [C2](dsh-tui-与grok的功能对比-c2.md)、首个 Claude TUI 对比 [C1](dsh-tui-与claude的对比-c1.md)）
>
> **日期**: 2026-08-12
>
> **范围**: DSH 当前能力（harness + TUI）对标 Claude Code（官方 64 个斜杠命令）与 grok-build（`xai-grok-pager` crate，本地 checkout `~/checkouts/grok-build`，Rust/ratatui/crossterm，ACP 传输）。本文档为合并后的最新图景——C5 之后的新项标注 **NEW**，已完成项划除并附提交。

## 1. DSH 当前能力总览

### 1.1 Harness（packages/，45 个包组）

**执行**: bash/pwsh（前台+后台）、bwrap/Landlock/Seatbelt 沙箱（三种按调用策略）、托管子进程树、持久 PTY（`terminal_*` 六工具）、fs（read/write/edit/glob/grep/file_info + 读前编辑门禁 + 原子写 + 快照回滚）、run_code（worker 隔离）、web_search/web_fetch、E2B 远程沙箱 POC、LSP 四语义工具、spill 溢出。

**Agent 编排**: agent-loop（turn/step/串并行/重试/取消）、subagent（六 provider 类 + 可恢复后台 subagent）、tasks 后台协议、workflow（模型编写 JS 编排 + ralph）、plan mode、goal/todo、compact、send/followup/steer/inject/cancel 输入通道。

**模型优化 — NEW**: `deepseek-spark` provider route 在 wire 层把 assistant 推理截断为尾部 N token（flash 300 / pro 需显式开启），与 `dsh-spark-anchors` 排除路径锚点回灌成对（`d17b414`、`3bcae85`、`f336a60`）；确定性退化分词（字节稳定、缓存友好）；共用 API key、`/model spark-flash|spark-pro` 别名（`360adc3`）。

**交互/审批**: user-approval（allowed-once/rejected/cancelled + waterfall answerer + 审计）、权限预设（workspace-write/danger-full-access + `/permission`）、ask_user_question、插件命令注册、skill 目录加载。

**上下文/知识**: AGENTS.md/CLAUDE.md 分层注入、memory 跨会话记忆（recall/remember）、semantic_search + repo_graph（meridian 代码图）、session_search/read 检索、credentials/settings 分层配置。

**会话/持久化**: 事件源 Session（append-only + fork/truncate）、JSONL(zstd)/SQLite 双后端 + 投影 + 标题 + OTel 遥测。

**协议/集成**: MCP client、ACP server、hooks（Claude Code/Codex hooks.json 桥）、TypeRT RPC 网关 + Web host、TS/Python SDK + stdio JSON-RPC。

**治理/质量**: invariants 运行时断言、guard 家族（重复工具提醒/超时预算/evidence-gate/agent-router）、自我修改（热运行时变更）。

### 1.2 TUI（`packages/tui/tui/`，22 个斜杠命令）

命令: `/theme /session /fork /branch /clear /compact /steer /model /tasks /density /goal /status /subagents /workflow /config /skills /rewind /btw /doctor /mcp /remember /memory` + `/permission` 预设切换。**NEW**: `/model spark-flash|spark-pro` 别名（`360adc3`）。

渲染面: 顶部栏（cwd/branch/model）、三行底部区（输入→footer→metrics）、markdown + 数学（LaTeX→Unicode）+ CJK 感知宽度、tool-card 渲染（家族着色/耗时/并行组折叠/头尾截断 + 展开标记）、审批内联 diff（Myers LCS via `diff` 包、±3 上下文、≤12 行折叠）、subagent spinner→✓/✗/◌ scrollback 行、turn-status 行、欢迎页（会话恢复列表 + 菜单）、slash 下拉（MRU + ghost 预览）、Ctrl+F 历史搜索 overlay、`/rewind` 两阶段 overlay、btw 侧问面板、task/goal/memory/subagent/workflow 面板（5 域投影总线）、vim 模式（normal/insert/visual）、外部编辑器（Ctrl+O）、主题系统（检测/定制/调色板，22KB 调色板模块）、性能监控 + 写批处理。**NEW**: 已结算工具卡实时提交进 scrollback，消费 harness presenter 渲染意图（`presentCall`/`presentResult` → 结构化 diff/terminal 卡、generic 回落；结算 diff 与审批预览共享 `renderFileDiff`）；think 推理通道——live shimmer 头行 + 暗色尾巴，段结束折叠头行落底 scrollback（正文默认收起，Ctrl+O 在 live 区展开查看），resume 经同一条桥按事件 seq 交错重放。

## 2. Claude Code 基准（C5 合并，更新）

### 2.1 命令映射（Claude 64 × DSH 22）

**已有等价**: `/clear /compact /config /model /theme /vim /status /memory /skills /mcp /hooks /permission /plan /fork /branch /rewind /btw /tasks /doctor /exit`。

**仍缺失**（C5 §5，未变）: `/init /effort /fast /passes /context /cost /usage /insights /stats /copy /diff /keybindings /permissions（规则级）/sandbox /add-dir /pr-comments /security-review /debug /feedback /rename /resume /voice /color /statusline /stickers /privacy-settings /upgrade /login /logout` + 整个 Remote 类。（T3 已落地 `/export`，已从缺失列表移除。）

### 2.2 Harness 层差距（C5 §3，状态更新）

| # | 差距 | 状态 |
|---|---|---|
| H1 | 面向模型的 git 工具 | ✅ 完成（`7313021` git 接缝 + `5b31656` tool-git：git_status/diff/log/commit） |
| H2 | 规则级权限白名单 | 未做（中） |
| H3 | /init 项目记忆生成 | 未做（低） |
| H4 | 会话 checkpoint/restore 工具 | 未做（中；fs-snapshot/truncate 原语已存在） |
| H5 | 声明式命令元数据（frontmatter） | 未做（低） |
| H6 | 代码审查工具 | 未做（低） |
| H7 | Web SSRF/私网保护 | 未做（中，安全） |
| H8 | Windows 沙箱后端 | 未做（平台） |

### 2.3 TUI 层差距（C5 §4，状态更新）

| # | 差距 | 状态 |
|---|---|---|
| T1 | /effort 切换 | ✅ 完成（`a1ae54b`：`/model <p/m\|别名> [off\|high\|max]`、别名组合、effort 回显） |
| T2 | 交互式 diff 导航 | 未做（中） |
| T3 | /export /copy 会话导出 | ✅ 部分完成（/export 已落地，本批；/copy 依赖剪贴板面，列后续） |
| T4 | /context 分布可视化 | 未做（中） |
| T5 | 全屏转录查看器 | 未做（中） |
| T6 | 可定制键位 | 未做（高；grok 也是硬编码——C2 项 5，用户确认不做） |
| T7 | /usage /cost 命令 | 未做（低） |
| T8 | 多行输入 | 未做（中，已计划） |
| T9 | 任务完成铃 | 未做（低） |
| T10 | /debug /feedback | 未做（低） |

## 3. grok-build 基准（C2 合并 + 渲染面清单）

### 3.1 C2 五差距项闭环记录（全部已决）

| C2 项 | 结果 |
|---|---|
| 审批 diff 预览 | ✅ 完成 `15173d4`（反转 grok：内联 diff 置于 y/N 上方，信任构建） |
| 历史搜索 overlay | ✅ 完成 `ec724ac`（同步——DSH 规模无需后台守护） |
| 输出折叠 | ⏸️ 无需实现（既有 tool-card 截断 read 8 / find 6 行比 grok 的按工具配置更激进） |
| /model 热切 | ✅ 完成 `0645886`（可变 `ModelSelectionRef` 替代 ACP） |
| 可定制键位 | ❌ 不做（grok 同样硬编码；用户确认） |

### 3.2 grok TUI 面清单（xai-grok-pager）——新交叉核对

grok crate 中存在的特性（文件名证据 + C2 索引；未逐文件重读）与 DSH 对应：

| grok 特性（文件） | DSH 对应 | 差距 |
|---|---|---|
| `diff.rs` — Myers LCS diff、±3 上下文、双 gutter | 审批 diff + 工具编辑 diff（`format/diff.ts` + `permission-diff.ts`） | ✅ 覆盖 |
| `export_cmd.rs` — 会话导出 | `/export`（`format/export.ts` 纯函数事件渲染 + TuiApp 接线写盘） | ✅ 覆盖（T3） |
| `doctor_cmd.rs` | `/doctor` | ✅ 覆盖 |
| `mcp_cmd.rs` | `/mcp` | ✅ 覆盖 |
| `git_info.rs` — 仓库信息 | 顶部栏 cwd/branch | ✅ 基本覆盖 |
| `fs_size.rs` / `disk_usage_cmd/` — 磁盘用量 | 无 | **未做**（低） |
| `headless.rs` — 无头运行 | `tianshu run` | ✅ 覆盖 |
| `hyperlink_route.rs` — OSC-8 超链接 | 无 | **未做**（低，外观） |
| `inline_media_ffmpeg.rs` — 内联媒体（ffmpeg 帧） | `image-tool`/`term-image`/`image-attach`（kitty/ANSI 图形） | ⚠️ 路径不同——DSH 原生终端图像；grok 提取视频帧 |
| `config_toml_edit.rs` — TOML 配置编辑 | `/config` 面板（settings/permission/credentials） | ⚠️ 形态不同——DSH 面板 vs grok TOML 文件编辑 |
| `input_log.rs` — 输入日志 | session log（一切皆记录） | ✅ 覆盖（更强） |
| `diagnostics/` — 诊断 | `/doctor` | ✅ 覆盖 |
| `scrollback/search.rs` + `SearchDaemon` — 后台搜索 | Ctrl+F overlay（同步） | ✅ 覆盖（按规模简化） |
| `scrollback/state/nav.rs` — 翻页（PageUp/Down、半页、goto top/bottom、轮次跳转） | 终端原生 scrollback | **未做**（中；T5 全屏查看器可吸收） |
| `scrollback/blocks/tool/read.rs` — read 首尾阈值 | tool-card 截断 | ✅ 覆盖（更激进） |
| `views/settings_modal/` — 四态设置弹窗 | `/config` 面板 | ✅ 覆盖 |
| `slash/commands/model.rs` — `/model <name> [effort]` | `/model` + 别名 | ✅ 覆盖（+effort 热切未做，T1） |

### 3.3 渲染维度对比

| 维度 | grok（xai-grok-pager） | DSH TUI |
|---|---|---|
| 框架 | Rust ratatui + crossterm | TS 自研 ANSI 引擎（移植自 Tianshu）+ 写批处理 |
| 传输 | ACP → agent | 进程内 cordis 事件 + 投影总线 |
| Diff | similar（Myers）+ ±3 ctx + gutter | `diff`（Myers）+ ±3 ctx + gutter + ≤12 行审批折叠 |
| 搜索 | 后台守护（多 agent 规模） | 同步主线程 overlay（单会话规模） |
| 折叠 | 三态枚举（Collapsed/Truncated/Expanded） | 两态 tool-card + 组折叠 |
| 键位 | ActionDef 注册表，硬编码 | 散落 switch，硬编码（C2 项 5：注册表重构留待后续） |
| 图像/媒体 | ffmpeg 内联帧 | 原生终端图形（kitty 协议等） |
| 配置面 | TOML 文件编辑 | 类型化设置面板（热加载） |
| 会话日志 | 输入日志 | 全事件源日志（更强） |

## 4. 合并差距矩阵与优先级

### 高价值未做项（建议下一步）

1. **T3 /copy 剪贴板导出**（低成本）——/export 已落地（本批）；copy 依赖 TUI 剪贴板面（osc52 已有写入面，读入面在 C6 矩阵后续），列下一批。
2. **T1 /effort 切换**（低成本）——llm 层已按模型暴露 `reasoningEffort`；`/model spark-flash high` 或 `/effort` 命令只需命令 + settings 接线。
3. **H1 面向模型的 git 工具**（中）——Claude 原生 git 工具是日常面；DSH 今天经 bash 手搓。
4. **T5 全屏转录查看器**（中）——吸收 grok 翻页导航（PageUp/Down、goto top/bottom、轮次跳转）；与 Ctrl+F overlay 互补。
5. **H2 规则级权限白名单**（中、信任）——在既有审批/预设接缝上加 `Bash(npm test:*)` 式持久规则。

### 明确不做（已记录）

- 可定制键位（grok 同样硬编码；C2 项 5 用户确认）
- 三态折叠（两态覆盖 DSH 规模；C2 项 3）
- 后台搜索守护（同步足够；C2 项 2）
- Remote 类（desktop/mobile/slack——DSH 是终端工具；E2B 是种子）

## 5. 位置总结

- **Harness**: 与 Claude Code 核心面（执行/审批/上下文/会话）特性持平，另加模型优化（spark）与 Claude 不暴露为插件的治理层（invariants/guards）；诚实差距是日常便利项（git 工具、规则级权限、审查命令）。
- **TUI**: 22 命令 vs Claude 64（多为管理/生态命令）；渲染面与 grok 在 diff/搜索/折叠持平、图像渲染领先；工具结算卡、结构化 presenter diff 与折叠推理通道（Ctrl+O 展开）对齐 Claude Code 的会话视图。具体下一步赢点是导出、effort 切换、全屏转录查看器。
- **证据基础**: 磁盘证据（packages/、TUI 测试 1418/1418 绿、C1/C2/C5 文档、grok-build 文件清单）。grok 侧特性语义由文件名 + C2 索引推断，未逐文件重读。尚无交互 TTY 验收（C1/C2 已记录）。
