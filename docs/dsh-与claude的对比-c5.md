# DSH vs Claude Code two-layer benchmark (harness + TUI) — C5

English | [中文](dsh-与claude的对比-c5.zh.md)

> **Tag**: C5 (fifth in the Claude Code benchmarking series; first covering the harness layer)

> **Date**: 2026-08-12

> **Baseline**: Claude Code's official 64 slash commands (11 categories) plus the extension surface (hooks/MCP/skills/agents/frontmatter) versus DSH harness (packages/ 45 package groups) and TUI (`packages/tui/tui/`, 22 slash commands)

> **Sources**: the Claude Code side is the official docs (commands) plus a third-party full command table (ThamJiaHe/claude-code-handbook command-and-config-reference); the DSH side is on-disk evidence (packages/README, per-group package READMEs, docs/tool-catalog, the C1 comparison doc docs/dsh-tui-与claude的对比-c1.md)

## 1. Claude Code capability baseline

The 64 official slash commands fall into 11 categories: **Auth** (login/logout/upgrade), **Config** (color/config/keybindings/permissions/privacy-settings/sandbox/statusline/stickers/terminal-setup/theme/vim/voice), **Context** (context/cost/extra-usage/insights/stats/status/usage), **Debug** (doctor/feedback/help/release-notes/tasks), **Export** (copy/export), **Extensions** (agents/chrome/hooks/ide/mcp/plugin/reload-plugins/skills), **Memory** (memory), **Model** (effort/fast/model/passes/plan), **Project** (add-dir/diff/init/pr-comments/security-review), **Remote** (desktop/install-github-app/install-slack-app/mobile/remote-control/remote-env/schedule), **Session** (branch/btw/clear/compact/exit/rename/resume/rewind).

Extension surface: commands/skills/agents share a 13-field frontmatter (paths auto-activation, allowed-tools, model/effort overrides, context: fork isolation, hooks attachment); 5 bundled skills (simplify/batch/debug/loop/claude-api); the RPI (research→plan→implement) workflow with per-phase agents and permissions.

## 2. DSH harness current state (capability summary)

**Execution**: bash/pwsh (foreground+background), bwrap/Landlock/Seatbelt sandbox (three per-call policies), managed subprocess tree, persistent PTY (terminal_* six tools), fs (read/write/edit/glob/grep/file_info + read-before-edit gate + atomic writes + snapshot rollback), run_code (worker-isolated), web_search/web_fetch, E2B remote sandbox POC, LSP four-semantic tools, spill overflow.

**Agent orchestration**: agent-loop (turn/step/serial-parallel/retry/cancel), subagent (six provider classes + resumable background subagents), tasks background protocol, workflow (model-written JS orchestration + ralph), plan mode, goal/todo, compact, send/followup/steer/inject/cancel input channels.

**Interaction/approval**: user-approval (allowed-once/rejected/cancelled + waterfall answerer + audit), permission presets (workspace-write/danger-full-access + `/permission` + Settings persistence), ask_user_question, plugin command registration, skill directory loading.

**Context/knowledge**: AGENTS.md/CLAUDE.md tiered injection, memory recall/remember cross-session store, semantic_search + repo_graph (meridian code graph), session_search/read retrieval, credentials/settings layered configuration.

**Session/persistence**: event-sourced Session (append-only + fork/truncate), JSONL(zstd)/SQLite dual backends + projections + titles + OTel telemetry.

**Protocol/integration**: MCP client (mcp__server__tool), ACP server, hooks (Claude Code/Codex hooks.json bridges), TypeRT RPC gateway + Web host, TS/Python SDK + stdio JSON-RPC.

**Governance/quality**: invariants runtime assertions, guard family (repeat-tool reminder/timeout budget/evidence-gate/agent-router), self-modification (hot runtime changes).

## 3. Harness-layer gaps (8 items)

| # | Gap | Claude Code shape | DSH status | Cost |
|---|---|---|---|---|
| H1 | Model-facing git tools | native git diff/commit/log tools | none (all hand-rolled through bash) | medium |
| H2 | Rule-level permission allowlist | /permissions persistent per-rule allow (e.g. `Bash(npm test:*)`) | presets + one-shot approval; no pattern rules | medium |
| H3 | /init project memory generation | generates CLAUDE.md/memory from a project | memory service + instruction injection exist; no generation command | low |
| H4 | checkpoint/restore tools | /rewind session-level rollback point | fs-snapshot/truncate primitives exist; no session checkpoint tool | medium |
| H5 | Declarative command metadata | frontmatter (paths/allowed-tools/model/effort/context:fork) | slash commands carry only name/description/argsHint | medium |
| H6 | Code review tools | /code-review /security-review | none | low |
| H7 | Web SSRF/private-network protection | private-network isolation by default | deferred per the web package README | medium (security) |
| H8 | Windows sandbox backend | — | POSIX only (bwrap/Landlock/Seatbelt) | platform item |

## 4. TUI-layer gaps (C1 baseline updated)

Additions since the C1 baseline (2026-08-11): rewind/memory/doctor/mcp/btw commands, /permission, slash dropdown menu (phases 1+2), subagent conversation-stream lines, B-layout input box. Current: 22 commands vs Claude's 64.

| # | Gap | Status | Cost |
|---|---|---|---|
| T1 | /effort switching | llm layer already has reasoningEffort (glance segment rendered); no switching command | low |
| T2 | Interactive diff navigation | format/diff.ts is pure rendering; no per-hunk navigation/expand | medium |
| T3 | /export /copy session export | none | low |
| T4 | /context distribution visualization | token/cache ratios exist; no per-message distribution | medium |
| T5 | Fullscreen transcript viewer | none | medium |
| T6 | Customizable keybindings | keys hardcoded; keymap-panel is read-only | high |
| T7 | /usage /cost commands | glance has token/cost segments; no commands | low |
| T8 | Multiline input (Wave 3) | single-line | medium (planned) |
| T9 | Task-completion notification (bell) | none | low |
| T10 | /debug /feedback | none | low |

## 5. Command mapping (Claude 64 categories × DSH 22)

**Equivalents present**: /clear /compact /config /model /theme /vim /status /memory /skills /mcp /hooks (bridge) /permission /plan /fork /branch /rewind /btw /tasks /doctor /exit.

**Missing**: /init /effort /fast /passes /context /cost /usage /insights /stats /export /copy /diff /keybindings /permissions (rule-level) /sandbox /add-dir /pr-comments /security-review /debug /feedback /rename /resume /voice /color /statusline /stickers /privacy-settings /upgrade /login /logout, plus the entire Remote category (desktop/mobile/remote-env/schedule — DSH is a terminal tool; E2B is a seed).

**DSH-only**: /steer /density /subagents /workflow /goal /remember /session (fork lineage + multi-session resume; session management is stronger than Claude's).

## 6. Priority recommendations

**Tier 1 (fill the base)**: H1 git tools → T1 /effort → H2 rule-level allowlist → H3 /init.

**Tier 2 (experience loop)**: T2 diff navigation → T3 /export → T4 context visualization → H5 frontmatter.

**Tier 3 (on demand)**: H4 checkpoint, T5 fullscreen viewer, T6 keybindings, T7 /usage, H6 code-review.

## 7. Evidence and limitations

Every DSH-side conclusion has on-disk evidence (package READMEs, the command registries commands/registry.ts + ui/app.ts, the C1 doc); the harness capability summary comes from a systematic pass over packages/README and per-group package READMEs (subagent research). The Claude Code command table comes from a third-party consolidation of the official 64 commands; individual command interaction details were not checked one by one. The Remote category carries no benchmark meaning for DSH (terminal-tool positioning). This document only benchmarks and ranks; it commits to no implementation scope — implementation follows standalone planning documents.
