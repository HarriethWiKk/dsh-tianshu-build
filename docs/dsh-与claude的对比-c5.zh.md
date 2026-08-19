# DSH 与 Claude Code 双层对标（harness + TUI）— C5

[English](dsh-与claude的对比-c5.md) | 中文

> **标记**：C5（与 Claude Code 对标系列第 5 份；harness 层第一份）

> **日期**：2026-08-12

> **基准**：Claude Code 官方 64 个 slash 命令（11 类）+ 扩展面（hooks/MCP/skills/agents/frontmatter）对比 DSH harness（packages/ 45 组包）+ TUI（`packages/tui/tui/`，22 个 slash 命令）

> **来源**：Claude Code 侧为官方文档（commands）+ 第三方命令全表（ThamJiaHe/claude-code-handbook command-and-config-reference）；DSH 侧为磁盘证据（packages/README、各组包 README、docs/tool-catalog、C1 对比文档 docs/dsh-tui-与claude的对比-c1.md）

## 一、Claude Code 能力基准

64 个官方 slash 命令分 11 类：**Auth**（login/logout/upgrade）、**Config**（color/config/keybindings/permissions/privacy-settings/sandbox/statusline/stickers/terminal-setup/theme/vim/voice）、**Context**（context/cost/extra-usage/insights/stats/status/usage）、**Debug**（doctor/feedback/help/release-notes/tasks）、**Export**（copy/export）、**Extensions**（agents/chrome/hooks/ide/mcp/plugin/reload-plugins/skills）、**Memory**（memory）、**Model**（effort/fast/model/passes/plan）、**Project**（add-dir/diff/init/pr-comments/security-review）、**Remote**（desktop/install-github-app/install-slack-app/mobile/remote-control/remote-env/schedule）、**Session**（branch/btw/clear/compact/exit/rename/resume/rewind）。

扩展面：命令/技能/agent 共享 13 字段 frontmatter（paths 自动激活、allowed-tools、model/effort 覆盖、context: fork 隔离、hooks 挂载）；官方捆绑 5 技能（simplify/batch/debug/loop/claude-api）；RPI（research→plan→implement 三阶段分 agent 分权限）工作流。

## 二、DSH harness 层现状（能力域摘要）

**执行**：bash/pwsh（前台+后台）、bwrap/Landlock/Seatbelt 沙箱（三档 per-call 策略）、受管子进程树、持久 PTY（terminal_* 六工具）、fs（read/write/edit/glob/grep/file_info + read-before-edit 门 + 原子写 + 快照回卷）、run_code（worker 隔离）、web_search/web_fetch、E2B 远程沙箱 POC、LSP 四语义工具、spill 溢出。

**agent 编排**：agent-loop（turn/step/串并行/重试/取消）、subagent（六类 provider + 可续后台子代理）、tasks 后台协议、workflow（模型写 JS 编排 + ralph）、plan 模式、goal/todo、compact 压缩、send/followup/steer/inject/cancel 输入通道。

**交互/审批**：user-approval（allowed-once/rejected/cancelled + waterfall answerer + 审计）、permission 预设表（workspace-write/danger-full-access + `/permission` + Settings 持久化）、ask_user_question、插件化命令注册、skill 目录加载。

**上下文/知识**：AGENTS.md/CLAUDE.md 分层注入、memory recall/remember 跨会话存储、semantic_search + repo_graph（meridian 代码图）、session_search/read 等检索、credentials/settings 分层配置。

**会话/持久化**：事件溯源 Session（append-only + fork/truncate）、JSONL(zstd)/SQLite 双后端 + 投影 + 标题 + OTel 遥测。

**协议/集成**：MCP 客户端（mcp__server__tool）、ACP 服务端、hooks（Claude Code/Codex hooks.json 桥）、TypeRT RPC 网关 + Web host、TS/Python SDK + stdio JSON-RPC。

**治理/质量**：invariants 运行时断言、guard 族（重复工具提醒/超时预算/evidence-gate/agent-router）、self-modification（热改运行时）。

## 三、Harness 层差距（8 项）

| # | 缺口 | Claude Code 形态 | DSH 现状 | 成本 |
|---|---|---|---|---|
| H1 | 模型面 git 工具 | git diff/commit/log 原生工具 | 无（全靠 bash 手搓） | 中 |
| H2 | 规则级权限 allowlist | /permissions 按规则持久 allow（如 `Bash(npm test:*)`） | 预设表 + 一次性审批；无模式匹配规则 | 中 |
| H3 | /init 项目记忆生成 | 从项目生成 CLAUDE.md/记忆 | 有 memory 服务与指令注入；无生成命令 | 低 |
| H4 | checkpoint/restore 工具 | /rewind 会话级回滚点 | 有 fs-snapshot/truncate 原语；无会话 checkpoint 工具 | 中 |
| H5 | 声明式命令元数据 | frontmatter（paths/allowed-tools/model/effort/context:fork） | slash 命令仅 name/description/argsHint | 中 |
| H6 | 代码审查工具 | /code-review /security-review | 无 | 低 |
| H7 | Web SSRF/私网保护 | 默认私网隔离 | web 包 README 明确推迟 | 中（安全项） |
| H8 | Windows 沙箱后端 | — | 仅 POSIX（bwrap/Landlock/Seatbelt） | 平台项 |

## 四、TUI 层差距（C1 基线更新）

相对 C1 基线（2026-08-11）的新增：rewind/memory/doctor/mcp/btw 命令、/permission、slash 下拉菜单（阶段 1+2）、subagent 对话流行、B 布局输入框。当前 22 命令 vs Claude 64。

| # | 缺口 | 现状 | 成本 |
|---|---|---|---|
| T1 | /effort 切换 | llm 层已有 reasoningEffort（glance 段在渲染）；缺切换命令 | 低 |
| T2 | diff 交互导航 | format/diff.ts 纯渲染；无逐 hunk 导航/展开 | 中 |
| T3 | /export /copy 会话导出 | 无 | 低 |
| T4 | /context 分布可视化 | 有 token/缓存占比；无 per-message 分布 | 中 |
| T5 | 全屏 transcript viewer | 无 | 中 |
| T6 | 自定义 keybindings | 键位硬编码；keymap-panel 只读 | 高 |
| T7 | /usage /cost 命令 | glance 有 token/cost 段；无命令 | 低 |
| T8 | 多行输入（Wave 3） | 单行 | 中（已计划） |
| T9 | 任务完成通知（bell） | 无 | 低 |
| T10 | /debug /feedback | 无 | 低 |

## 五、命令对照（Claude 64 类 × DSH 22）

**已有等价**：/clear /compact /config /model /theme /vim /status /memory /skills /mcp /hooks（桥）/permission /plan /fork /branch /rewind /btw /tasks /doctor /exit。

**缺失**：/init /effort /fast /passes /context /cost /usage /insights /stats /export /copy /diff /keybindings /permissions（规则级）/sandbox /add-dir /pr-comments /security-review /debug /feedback /rename /resume /voice /color /statusline /stickers /privacy-settings /upgrade /login /logout，以及整个 Remote 类（desktop/mobile/remote-env/schedule——DSH 属终端工具，E2B 为雏形）。

**DSH 独有**：/steer /density /subagents /workflow /goal /remember /session（fork 谱系 + 多会话恢复，会话管理强于 Claude）。

## 六、优先级建议

**第一梯队（补齐基本盘）**：H1 git 工具 → T1 /effort → H2 规则级 allowlist → H3 /init。

**第二梯队（体验闭环）**：T2 diff 导航 → T3 /export → T4 context 可视化 → H5 frontmatter。

**第三梯队（按需）**：H4 checkpoint、T5 全屏 viewer、T6 keybindings、T7 /usage、H6 code-review。

## 七、证据与局限

DSH 侧结论均有磁盘证据（包 README、命令注册表 commands/registry.ts + ui/app.ts、C1 文档）；harness 能力域来自 packages/README 与各组包 README 的系统梳理。Claude Code 侧命令全表来自第三方整理的官方 64 命令清单，个别命令的交互细节未逐一核对；Remote 类对 DSH 无对标意义（终端工具定位）。本文只做对标与排序，不承诺实施范围；实施跟随独立规划文档。
