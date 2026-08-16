# DSH capability review + TUI rendering benchmark — C6

English | [中文](dsh-能力复盘与对标-c6.zh.md)

> **Tag**: C6 (consolidated review; merges and updates C1/C2/C5 — Claude Code benchmark [C5](dsh-与claude的对比-c5.md), grok feature comparison [C2](dsh-tui-与grok的功能对比-c2.md), first Claude TUI comparison [C1](dsh-tui-与claude的对比-c1.md))
>
> **Date**: 2026-08-12
>
> **Scope**: DSH current capability (harness + TUI) versus Claude Code (official 64 slash commands) and grok-build (`xai-grok-pager` crate, local checkout `~/checkouts/grok-build`, Rust/ratatui/crossterm, ACP transport). This document is the merged, up-to-date picture — new items since C5 are marked **NEW**; completed items are struck through with their commit.

## 1. DSH current capability summary

### 1.1 Harness (packages/, 45 package groups)

**Execution**: bash/pwsh (foreground+background), bwrap/Landlock/Seatbelt sandbox (three per-call policies), managed subprocess tree, persistent PTY (`terminal_*` six tools), fs (read/write/edit/glob/grep/file_info + read-before-edit gate + atomic writes + snapshot rollback), run_code (worker-isolated), web_search/web_fetch, E2B remote sandbox POC, LSP four-semantic tools, spill overflow.

**Agent orchestration**: agent-loop (turn/step/serial-parallel/retry/cancel), subagent (six provider classes + resumable background subagents), tasks background protocol, workflow (model-written JS orchestration + ralph), plan mode, goal/todo, compact, send/followup/steer/inject/cancel input channels.

**Model optimization — NEW**: `deepseek-spark` provider route truncates assistant reasoning to the tail N tokens on the wire (flash 300 / pro opt-in), paired with `dsh-spark-anchors` excluded-path anchor re-injection (`d17b414`, `3bcae85`, `f336a60`); deterministic degenerate tokenizer (byte-stable, cache-preserving); same-API-key, `/model spark-flash|spark-pro` aliases (`360adc3`).

**Interaction/approval**: user-approval (allowed-once/rejected/cancelled + waterfall answerer + audit), permission presets (workspace-write/danger-full-access + `/permission`), ask_user_question, plugin command registration, skill directory loading.

**Context/knowledge**: AGENTS.md/CLAUDE.md tiered injection, memory recall/remember cross-session store, semantic_search + repo_graph (meridian code graph), session_search/read retrieval, credentials/settings layered configuration.

**Session/persistence**: event-sourced Session (append-only + fork/truncate), JSONL(zstd)/SQLite dual backends + projections + titles + OTel telemetry.

**Protocol/integration**: MCP client, ACP server, hooks (Claude Code/Codex hooks.json bridges), TypeRT RPC gateway + Web host, TS/Python SDK + stdio JSON-RPC.

**Governance/quality**: invariants runtime assertions, guard family (repeat-tool reminder/timeout budget/evidence-gate/agent-router), self-modification (hot runtime changes).

### 1.2 TUI (`packages/tui/tui/`, 22 slash commands)

Commands: `/theme /session /fork /branch /clear /compact /steer /model /tasks /density /goal /status /subagents /workflow /config /skills /rewind /btw /doctor /mcp /remember /memory` + `/permission` preset switcher. **NEW**: `/model spark-flash|spark-pro` aliases (`360adc3`).

Rendering surface: top bar (cwd/branch/model), three-row bottom area (input → footer → metrics), markdown + math (LaTeX→Unicode) + CJK-aware width, tool-card rendering (family coloring, run timing, parallel group folding, head/tail truncation with expand marker), approval inline diff (Myers LCS via the `diff` package, ±3 context, ≤12-line fold), subagent spinner→✓/✗/◌ scrollback lines, turn-status line, welcome page (session restore list + menu), slash dropdown (MRU + ghost preview), Ctrl+F history-search overlay, `/rewind` two-phase overlay, btw side-question panel, task/goal/memory/subagent/workflow panels via the 5-domain projection bus, vim mode (normal/insert/visual), external editor (Ctrl+O), theme system (detect/custom/palettes, 22KB palette module), perf monitor + write batching. **NEW**: settled tool cards commit into scrollback in real time consuming the harness presenter intent (`presentCall`/`presentResult` → structured diff/terminal cards, generic fallback; the settled diff shares `renderFileDiff` with the approval preview), and the think/reasoning channel — live shimmer header + dim tail, a folded header committed to scrollback at segment end (full text hidden by default, `Ctrl+O` reveals it in the live area), resume replays through the same bridge interleaved by event seq.

## 2. Claude Code benchmark (C5 consolidated, updated)

### 2.1 Command mapping (Claude 64 × DSH 22)

**Equivalents present**: `/clear /compact /config /model /theme /vim /status /memory /skills /mcp /hooks /permission /plan /fork /branch /rewind /btw /tasks /doctor /exit`.

**Still missing** (C5 §5, unchanged): `/init /effort /fast /passes /context /cost /usage /insights /stats /export /copy /diff /keybindings /permissions (rule-level) /sandbox /add-dir /pr-comments /security-review /debug /feedback /rename /resume /voice /color /statusline /stickers /privacy-settings /upgrade /login /logout` + the entire Remote category.

### 2.2 Harness-layer gaps (C5 §3, status update)

| # | Gap | Status |
|---|---|---|
| H1 | Model-facing git tools | ✅ done (`7313021` git seam + `5b31656` tool-git: git_status/diff/log/commit) |
| H2 | Rule-level permission allowlist | open (medium) |
| H3 | /init project memory generation | open (low) |
| H4 | Checkpoint/restore session tools | open (medium; fs-snapshot/truncate primitives exist) |
| H5 | Declarative command metadata (frontmatter) | open (low) |
| H6 | Code review tools | open (low) |
| H7 | Web SSRF/private-network protection | open (medium, security) |
| H8 | Windows sandbox backend | open (platform) |

### 2.3 TUI-layer gaps (C5 §4, status update)

| # | Gap | Status |
|---|---|---|
| T1 | /effort switching | ✅ done (`a1ae54b`: `/model <p/m\|alias> [off\|high\|max]`, alias combo, effort display) |
| T2 | Interactive diff navigation | open (medium) |
| T3 | /export /copy session export | open (low) |
| T4 | /context distribution visualization | open (medium) |
| T5 | Fullscreen transcript viewer | open (medium) |
| T6 | Customizable keybindings | open (high; grok also hardcodes — C2 item 5, user confirmed not doing) |
| T7 | /usage /cost commands | open (low) |
| T8 | Multiline input | open (medium, planned) |
| T9 | Task-completion bell | open (low) |
| T10 | /debug /feedback | open (low) |

## 3. grok-build benchmark (C2 consolidated + rendering-surface inventory)

### 3.1 C2 five-gap closure record (all resolved)

| C2 item | Outcome |
|---|---|
| Approval diff preview | ✅ done `15173d4` (inverted grok: inline diff above y/N, trust-builder) |
| History search overlay | ✅ done `ec724ac` (synchronous — DSH scale needs no background daemon) |
| Output folding | ⏸️ no implementation needed (existing tool-card truncation read 8 / find 6 lines is more aggressive than grok's per-tool config) |
| /model hot swap | ✅ done `0645886` (mutable `ModelSelectionRef` instead of ACP) |
| Configurable keybindings | ❌ not doing (grok hardcodes too; user confirmed) |

### 3.2 grok TUI surface inventory (xai-grok-pager) — new cross-check

Features present in grok's crate (file-name evidence + C2 index; not re-read per file) and DSH counterpart:

| grok feature (file) | DSH counterpart | Gap |
|---|---|---|
| `diff.rs` — Myers LCS diff, ±3 context, dual gutters | approval diff + tool edit diff (`format/diff.ts` + `permission-diff.ts`) | ✅ covered |
| `export_cmd.rs` — session export | none | **open** (C5 T3) |
| `doctor_cmd.rs` | `/doctor` | ✅ covered |
| `mcp_cmd.rs` | `/mcp` | ✅ covered |
| `git_info.rs` — repo info | top-bar cwd/branch | ✅ mostly covered |
| `fs_size.rs` / `disk_usage_cmd/` — disk usage | none | **open** (low) |
| `headless.rs` — headless run | `oh-my-tianshu run` | ✅ covered |
| `hyperlink_route.rs` — OSC-8 hyperlinks | none | **open** (low, cosmetic) |
| `inline_media_ffmpeg.rs` — inline media (ffmpeg frames) | `image-tool`/`term-image`/`image-attach` (kitty/ANSI graphics) | ⚠️ different approaches — DSH renders images natively in terminal; grok extracts video frames |
| `config_toml_edit.rs` — TOML config editing | `/config` panel (settings/permission/credentials) | ⚠️ different shape — DSH panel vs grok TOML file editing |
| `input_log.rs` — input logging | session log (everything logged) | ✅ covered (stronger) |
| `diagnostics/` — diagnostics | `/doctor` | ✅ covered |
| `scrollback/search.rs` + `SearchDaemon` — background search | Ctrl+F overlay (synchronous) | ✅ covered (simplified by scale) |
| `scrollback/state/nav.rs` — paging (PageUp/Down, half-page, goto top/bottom, next/prev turn) | terminal native scrollback | **open** (medium; T5 fullscreen viewer would absorb it) |
| `scrollback/blocks/tool/read.rs` — read first/last thresholds | tool-card truncation | ✅ covered (more aggressive) |
| `views/settings_modal/` — 4-state settings modal | `/config` panel | ✅ covered |
| `slash/commands/model.rs` — `/model <name> [effort]` | `/model` + aliases | ✅ covered (+effort hot-swap open, T1) |

### 3.3 Rendering-dimension comparison

| Dimension | grok (xai-grok-pager) | DSH TUI |
|---|---|---|
| Framework | Rust ratatui + crossterm | TS, custom ANSI engine (ported from Tianshu) + write batching |
| Transport | ACP to agent | in-process cordis events + projection bus |
| Diff | similar (Myers) + ±3 ctx + gutters | `diff` (Myers) + ±3 ctx + gutters + ≤12-line approval fold |
| Search | background daemon (multi-agent scale) | synchronous main-thread overlay (single-session scale) |
| Fold | three-state enum (Collapsed/Truncated/Expanded) | two-state tool-card + group folding |
| Keybindings | ActionDef registry, hardcoded | scattered switches, hardcoded (C2 item 5: registry refactor left for later) |
| Image/media | ffmpeg inline frames | native terminal graphics (kitty protocol etc.) |
| Config surface | TOML file editing | typed settings panel (hot-reload) |
| Session log | input log | full event-sourced log (stronger) |

## 4. Consolidated gap matrix and priorities

### High-value open items (recommended next)

1. **T3 /export /copy** (low cost, visible) — grok has `export_cmd.rs`; Claude has both; DSH logs are event-sourced so a transcript renderer is mostly reuse (`scrollback-transcript.ts`).
2. **T1 /effort switching** (low cost) — llm layer already exposes `reasoningEffort` per model; a `/model spark-flash high` shape or `/effort` command needs only command + settings wiring.
3. **H1 model-facing git tools** (medium) — Claude's native git tools are a daily-use surface; DSH hand-rolls through bash today.
4. **T5 fullscreen transcript viewer** (medium) — absorbs grok's paging nav (PageUp/Down, goto top/bottom, next/prev turn); complements Ctrl+F overlay.
5. **H2 rule-level permission allowlist** (medium, trust) — `Bash(npm test:*)`-style persistent rules on top of the existing approval/preset seams.

### Deliberately not doing (recorded)

- Configurable keybindings (grok hardcodes too; C2 item 5 user confirmation)
- Three-state fold (two-state covers DSH scale; C2 item 3)
- Background search daemon (synchronous suffices; C2 item 2)
- Remote category (desktop/mobile/slack — DSH is a terminal tool; E2B is the seed)

## 5. Position summary

- **Harness**: feature-parity with Claude Code's core surface (execution/approval/context/session) plus model optimization (spark) and a governance layer (invariants/guards) Claude does not expose as plugins; the honest gaps are daily-use conveniences (git tools, rule-level permissions, review commands).
- **TUI**: 22 commands vs Claude 64 (mostly management/ecosystem commands), rendering surface is at parity with grok on diff/search/fold and ahead on image rendering; tool settlement cards, structured presenter diffs, and the folded reasoning channel (Ctrl+O to reveal) match Claude Code's session view. The concrete next wins are export, effort switching, and a fullscreen transcript viewer.
- **Evidence basis**: on-disk sources (packages/, TUI tests 1418/1418 green, C1/C2/C5 documents, grok-build file inventory). grok-side feature semantics inferred from file names + C2 index, not re-read per file. No interactive TTY acceptance yet (recorded in C1/C2).
