# DSH TUI next-phase enhancement plan

English | [中文](dsh-tui-next-phase.zh.md)

> Comparison baseline: the capabilities Tianshu (天枢) TUI (opencode-tui/src/tui/) evolved independently. DSH TUI is an early fork, and Tianshu has since accumulated three generations of TUI capabilities.

## Non-goals

- No multi-agent orchestration visualization (council/team/worker panels) — DSH is a single-agent design
- No desktop features (Tauri/updater) — DSH is terminal-only
- No changes to the Cordis plugin architecture — every enhancement stays self-contained inside the TUI plugin

> **Evolution record (2026-08-11)**: the first non-goal expired when agent-router arrived. Once agent-router (S7/S8) brought verifier subagent scheduling to DSH, the "single-agent design" premise no longer held, and TUI Phase 2 (`9a038de`) shipped the matching /subagents delegation tree and /workflow runtime panel. Adding further multi-agent visualization is no longer treated as a non-goal violation; the porting scope of Phases 5-9 in this document (single-agent experience enhancements) is unaffected.

## Phase 5: agent internal-state visualization

DSH's statusline currently shows only idle/running. Tianshu's cockpit lets users see the internal state of the agent's cognitive process.

### 5.1 Six-phase workflow indicator

> ✅ Done — `1c7c0e4` feat(tui): workflow phase indicator and live activity projection

Tianshu's `phase-tracker.ts` shows in real time which workflow step the agent is in (understand / research / decompose / implement / verify / wrap up). DSH's agent loop has the corresponding phase concept (session events); a phase indicator needs to be added beside the statusline.

Implementation: consume `agent/status` events plus the turn structure in the session log to infer the current phase. No agent-side changes needed — a pure TUI projection.

### 5.2 Live activity label

> ✅ Done — `1c7c0e4` (same commit as 5.1)

Tianshu's `activity-status.ts` shows dynamic labels such as "reading file…", "searching…", "running bash…". DSH's `tool/call` events already carry the tool name and arguments, so the TUI can show the current activity on the statusline while a tool runs.

### 5.3 Metrics glance

> ✅ Done — `5abf6e2`+`1ae0af3`+`9775e93`: glance-bar (formatTokenCount / segment assembly / narrow-width drop) + metrics-glance-controller (first push / throttling) + TuiApp assembly (status/error line derivation + metrics line: model/turn/elapsed). The token data source is not in the TUI session event stream (council as counter-evidence); only obtainable segments are rendered.

Tianshu's `metrics-glance-controller.ts` shows token usage, cache hit rate, and current-turn elapsed time at the bottom of the TUI or in a sidebar. DSH already has the session log and LLM usage data; only the TUI-layer display is needed.

## Phase 6: interaction efficiency

### 6.1 Slash command system

> ✅ Done — `426607d` feat(tui): slash command system with Cordis service registry

Tianshu: `slash-commands.ts` + `command-palette.ts`. The DSH TUI input box is currently plain text. Add a `/` prefix that triggers a command palette:

- `/model` — switch models
- `/theme` — switch themes
- `/session new|list|switch` — session management
- `/steer <text>` — mid-turn steering (see 6.2)
- `/clear` — clear the current conversation
- `/compact` — trigger context compaction

Implementation: detect the `/` prefix in `input-controller.ts` and pop up an overlay panel. The command registry uses the Cordis service pattern, so other plugins can register custom commands.

### 6.2 Mid-turn steering

> ✅ Done — `553c0f3` feat(tui): mid-turn steering entry and statusline production mount

Tianshu: `steer-buffer.ts` + `steer-intent.ts`. DSH's `agent.steer()` API already exists (`adapter/send.ts`), but the TUI exposes no entry point. Needed:

- Triggered by `/steer <text>` in the input box or a keyboard shortcut (e.g. Ctrl+T)
- Steering text shown in the conversation stream with a distinct color/prefix
- Priority support (now/next/later)

### 6.3 File path autocompletion

> ✅ Done — `34f3787` feat(tui): @-path tab completion (Phase 6.3)

Tianshu: `file-completer.ts`. DSH's `.rivet/tui-source/tui/file-completer.ts` keeps the original implementation but is excluded. It needs to be restored and wired into the input box's Tab completion.

### 6.4 External editor

> ✅ Done — `92c2d06` feat(tui): external editor (Phase 6.4) — Ctrl+O opens $EDITOR on input line

Tianshu: `external-editor.ts`. Ctrl+O opens `$EDITOR` (configurable via Config editorKey) to edit the current input; on save-and-exit the content returns to the input box. Ctrl+E conflicts with input-line's moveEnd, hence Ctrl+O as the default.

### 6.5 Vim mode

> ✅ Done — `2c82467` feat(tui): vim mode wiring (Phase 6.5) — mode label in live region

Tianshu: `vim-mode.ts`. input-line.ts has a built-in vim state machine (normal/insert/visual); the TuiRunnerConfig vimEnabled switch is wired, plus mode label rendering (-- NORMAL -- / -- VISUAL --).

## Phase 7: tool execution visualization depth

### 7.1 Tool run timing

> ✅ Done — `59a6591` feat(tui): tool family coloring and run timing

Tianshu: `tool-elapsed.ts`. Shows a live timer during a tool call: "bash running… 12.3s". DSH's `tool/call` and `tool/result` events carry timestamps, so the TUI can render the timer in between.

### 7.2 Tool family coloring

> ✅ Done — `59a6591` (same commit as 7.1)

Tianshu: `tool-family.ts`. Colors tools by functional family: file operations (blue), shell (yellow), search (green), editing (purple), network (cyan). DSH's `format/tool-card.ts` already renders tool cards; adding family information is enough.

### 7.3 Parallel tool group folding

> ✅ Done — `da0adac` feat(tui): parallel tool group folding (Phase 7.3)

Tianshu: `tool-group-controller.ts` + `tool-accumulator.ts`. When the agent issues several independent tool calls at once, DSH's TUI currently shows them one by one. Tianshu folds them into a single group showing "3 tools running in parallel…", and users can expand it to inspect each one.

## Phase 8: approval flow UI

> ✅ Done — `1ae0af3` (overlay / command palette) + `2acc509` (approval answerer: registered via ctx.on('approval/request'), y/N inline confirmation, waterfall next() delegation)

Tianshu: `approval-intent-controller.ts`. DSH's approvals go through a Cordis policy plugin, and the TUI answerer is registered: when policy intercepts a tool, it renders an inline "⚠ allow running `rm -rf ./build`? [y/N]" prompt — y allows, n denies, Ctrl+C cancels — without blocking the terminal render stream.

## Phase 9 (optional): other experience enhancements

- **@mention parsing** ✅ Done — `5abf6e2` (src/mention-parser.ts) + `fccbe4b` (assembly: user-side summary expansion; cwd boundary / truncation / degradation)
- **Session restore panel** ✅ Done — `5abf6e2` (src/restore-session.ts projection) + `af73fa2` (scrollback lists restorable sessions on attach)
- **Turn summary** ✅ Implementation landed — `5abf6e2`+`1ae0af3` (turn-summary model + format dual paths, per the staged contract)
- **Fluency control** ✅ Done — `51feb85` (fluency-policy/hook port, ActivityPhase adaptation) + `835638e` (TuiApp assembly: tool events → tiered stale hints on screen)

- **@mention parsing** (`mention-parser.ts`): typing `@filename` auto-expands a summary of the file's contents
- **Session restore panel** (`restore-session.ts`): lists restorable sessions at startup
- **Turn summary** (`turn-summary.ts`): a tool-call statistics summary after each conversation turn
- **Fluency control** (`fluency-hook.ts`): prevents overly fast LLM output from causing terminal flicker

## Architecture constraints

- All enhancements are implemented inside `packages/tui/tui/src/`; the core/agent/session packages are untouched
- Data sources: published session events plus the Agent public API (followup/steer/cancel/whenIdle)
- Approval flow exception: a TUI-visible pending state must be negotiated with the Cordis policy system (policy currently returns allow/deny synchronously, with no pending state)
- No new event types — every display projects from existing events

## Priority order

1. **Phase 6.2 mid-turn steering** — the Agent API already exists and only the TUI entry point is missing; highest ROI
2. **Phase 5.1 phase indicator + 5.2 activity label** — the improvement users perceive most directly
3. **Phase 6.1 slash commands** — changes the interaction paradigm
4. **Phase 7.1 tool timing + 7.2 tool family coloring** — visual depth
5. **Phase 6.3 file completion** — saves users time every day
6. **Phase 8 approval flow UI** — a security improvement
7. **Others** — as needed

## Ecosystem integration track (Phases 1-3, 2026-08-11)

> The second track, parallel to Phases 5-9 (the experience-enhancement port): connect the TUI with the DSH guard system (evidence-gate / agent-router) at the interaction layer. No standalone planning document exists; the commit history is authoritative.

| Phase | Commit | Content |
|---|---|---|
| Phase 1 | `6c9cf7c` | native wiring — projection bus (5 domains), /status panel, /goal command, [plan] badge |
| Phase 2 | `9a038de` | multi-agent visualization — /subagents delegation tree, /workflow runtime panel, /tasks background tasks (paired with agent-router S7/S8; see the non-goals evolution record) |
| Phase 3 | `ae268fa` | interaction seams — structured questions, /config panel, /skills browser |

## Benchmark integration track (A1/A3/C2 series, 2026-08-11)

> The third track: fill TUI interaction gaps benchmarked against Claude Code (the C1 document) and grok-build (the C2 document). The gap list and per-item decisions live in `docs/dsh-tui-与claude的对比-c1.md` and `docs/dsh-tui-与grok的功能对比-c2.md`.

| Item | Commit | Content |
|---|---|---|
| A1 mode system | `4dca7d4` | slash channel falls back to CommandService (/plan reachable) + statusline [plan…] pending |
| A3 session forking | `ee23cd9` | /fork /branch commands (SessionStore.fork + switchSession, parentSession lineage) |
| C2-1 approval diff | `15173d4` | inline unified diff above the approval y/N (reuses createTwoFilesPatch + formatDiff; adds the diff dependency) |
| C2-4 /model hot swap | `0645886` | ModelSelectionRef mutable ref; takes effect on the next agent step |
| C2-2 history search | `ec724ac` | Ctrl+F fullscreen overlay, smart-case, n/N cyclic jumps |
| C2-3 output folding | — (no implementation needed) | research confirmed the existing tool-card truncation (read 8 lines / find 6 lines) already covers it |
| C2-5 keybinding configuration | — (won't do) | user-confirmed; grok itself also hardcodes |

**Acceptance status (2026-08-11)**: user-level behavior acceptance is blocked (the execution environment has no interactive TTY); real-assembly probe evidence: TUI profile startup ✓, fork lineage rendering ✓, Ctrl+F alt-screen sequence ✓. Full acceptance steps live in the individual Agent Notes (`.agents/notes/implemented/feature/2026-08-11-tui-*.md`).
