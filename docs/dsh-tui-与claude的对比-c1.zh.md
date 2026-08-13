# DSH TUI 与 Claude Code 能力对比 — C1

[English](dsh-tui-与claude的对比-c1.md) | 中文

> **标记**：C1（与 Claude Code 对标系列第 1 份）

> **日期**：2026-08-11

> **基准**：DSH TUI（`packages/tui/tui/`，snapshots/20260809T140917Z 分支）对比 Claude Code CLI v2.1.x

> **来源**：DSH 侧为磁盘证据（src 文件清单 / grep 行号）；Claude Code 侧为官方文档（commands、interactive-mode、terminal-config）+ wil.dev TUI guide

## 一、DSH TUI 现状盘点

`packages/tui/tui/src/` 79 个源文件，13 个内置 slash 命令（`/theme /session /model /clear /compact /goal /tasks /subagents /workflow /steer /status /config /skills`）。

| 层 | 已有能力 | 关键文件 |
|---|---|---|
| 渲染 | stream-renderer、block-stream-writer、write-batcher、live-engine、markdown、ANSI、term-image 图片、latex | `engine/stream-renderer.ts`、`engine/block-stream-writer.ts`、`engine/term-image.ts` |
| 输入 | vim 状态机（内建）、@-path Tab 补全、@mention 展开、外部编辑器 Ctrl+O、20+ ctrl 快捷键 | `engine/input-line.ts`、`completion/file-completer.ts`、`mention-parser.ts`、`external-editor.ts` |
| 面板 | status/config/question/skill/delegation/workflow/task/keymap/command-palette/restore-session | `status-panel.ts`、`config-panel.ts`、`delegation-panel.ts`、`workflow-panel.ts` |
| 投影 | 活动状态机、turn 统计、token/缓存命中、statusline、metrics glance | `activity-status.ts`、`turn-summary.ts`、`cache-telemetry.ts`、`statusline.ts` |
| 工具可视化 | 家族着色、计时、并行分组折叠、collapsed-bash、流利度控制 | `format/tool-family.ts`、`tool-elapsed.ts`、`engine/tool-group-controller.ts` |
| 主题 | theme/theme-detect/theme-custom/theme-palettes、term-caps、width | `theme*.ts`、`term-caps.ts` |
| 审批 | Phase 8 y/N 内联 answerer（waterfall next() 委托） | `ui/app.ts` L505 区域 |

## 二、与 Claude Code 的差距（三分类）

### A 类：底层能力已有，缺 TUI 接线（ROI 最高）

| # | 差距 | Claude Code 形态 | DSH 现状 | 缺口 |
|---|---|---|---|---|
| A1 | 操作模式系统 | normal / plan / auto 三模式，状态栏显示 | ✅ 已完成（2026-08-11）——slash 通道 fallback 到 CommandService（`/plan` 可达）+ statusline `[plan…]` pending 显示 | 无模式切换快捷键（Tab 被补全占用）；auto 三态未做（DSH plan off = 常规） |
| A2 | 权限 allowlist 记忆 | y/N 审批 + "总是允许"持久记忆 | Phase 8 y/N answerer 已接（`ui/app.ts` L505）；`packages/interaction/permission/` 无 allow/remember 机制（grep 0 命中） | 后端缺口（permission 包），不只是接线 |
| A3 | 会话分叉 | `/fork` `/branch` | ✅ 已完成（2026-08-11）——`/fork` `/branch` 命令（TuiApp.forkSession：`ctx.sessions.fork` + switchSession 切换），分叉继承历史与 parentSession 血缘 | 无分支树 UI（/session list 可看血缘）；确认对话框未做（命令即切，可后续加） |
| A4 | 上下文占用可视化 | `/context` 显示窗口占用分布 | cache-telemetry 有 token 用量投影 | 无"按消息/工具占多少"分布视图 |

### B 类：真缺失，需新建（中等工作量）

| # | 差距 | Claude Code 形态 | 说明 |
|---|---|---|---|
| B1 | 可定制快捷键 | `keybindings.json` + `/keybindings` | DSH keymap-panel 只读展示，快捷键硬编码在 input-handler/app.ts switch |
| B2 | 内联 diff 交互导航 | diff viewer 逐块导航/展开折叠 | `format/diff.ts` 是纯渲染（注释自述"基础版直移"） |
| B3 | 全屏 transcript viewer | fullscreen 模式，`?` 快捷键帮助 | 有 scrollback-transcript 无全屏查看器 |
| B4 | 推理强度切换 | `/effort` | DSH 无 effort 概念 |
| B5 | 后台 detach | `/background` 会话脱机继续跑 | tasks 有后台任务，无 detach 整个会话 |

### C 类：管理/生态命令（Claude Code 有，DSH 大多没有；多为小命令）

`/init`（生成项目记忆）、`/memory`、`/mcp`、`/permissions`、`/doctor`、`/debug`、`/usage`、`/cost`、`/rewind`（回滚）、`/code-review`、`/security-review`、`/btw`、`/prompt-color`、`/output-style`。

DSH 已有等价物：`/config`（对应 /config + /permissions 部分）、`/compact`、`/clear`、`/model`、`/status`。

Claude Code 特色而 DSH 未接：**终端通知**（任务完成 bell）、**Shift+Enter 换行**（DSH 输入行单行，多行输入需确认）。

## 三、优先级建议

1. **A1 模式系统**（plan/auto 切换）——plan-mode 包已存在，纯接线，用户价值最高，Claude Code 心智已建立
2. **A3 会话分叉**（/fork /branch）——session 层已支持 fork，TUI 加两个命令 + 确认对话框
3. **B1 可定制快捷键**——改动面大（switch 分发改查表），"手感"级差异
4. **A2 allowlist**——需动 permission 包（后端契约变更），跨包改动，单独评估
5. **B2 diff 导航**——在已有 format/diff.ts 上加交互层
6. C 类命令按需补，多为薄包装

## 四、证据与局限

- DSH 侧结论均有磁盘证据：src 文件清单（glob）、命令注册表（`commands/registry.ts` L215-432 + `ui/app.ts` L405-433）、plan 接线（`ui/app.ts` L77）、permission allowlist（grep 0 命中）、diff 纯渲染（`format/diff.ts` L2 注释）
- Claude Code 快捷键表格只抓到官方文档片段（interactive-mode 页截断），具体键位未逐条核对；能力面清单来自官方 commands + interactive-mode 文档
- 本文档只做对比，不承诺实现范围；实现以独立计划文档为准

## 五、实现与验收记录（2026-08-11）

| 项 | 状态 | 提交 |
|---|---|---|
| A1 模式系统（slash fallback + [plan…] pending） | ✅ 已完成 | `4dca7d4` |
| A3 会话分叉（/fork /branch） | ✅ 已完成 | `ee23cd9` |

**验收状态**：A1/A3 与 C2 系列（审批 diff / 历史搜索 / /model 热切，见`docs/dsh-tui-与grok的功能对比-c2.md`）的用户级行为验收 **blocked**——执行环境无交互式 TTY。已获真实装配探针证据：TUI profile 启动 ✓、恢复会话列表与 fork 血缘渲染 ✓（A3 的 parentSession 元数据真实可见）、Ctrl+F alt-screen 进出序列 ✓。完整用户验收待终端环境执行（验收步骤见各 Agent Note 的 Risks）。

## 后续（本系列待补）

- C2：与天枢 TUI（opencode-tui）对比——已有内容见 `docs/dsh-tui-next-phase.md`，可摘录归并
- C3：实现优先级与波次计划（待 A/B 类拍板后）
