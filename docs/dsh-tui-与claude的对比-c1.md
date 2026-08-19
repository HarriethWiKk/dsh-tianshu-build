# DSH TUI vs Claude Code capability comparison — C1

English | [中文](dsh-tui-与claude的对比-c1.zh.md)

> **Tag**: C1 (first in the Claude Code benchmarking series)

> **Date**: 2026-08-11

> **Baseline**: DSH TUI (`packages/tui/tui/`, the snapshots/20260809T140917Z branch) versus Claude Code CLI v2.1.x

> **Sources**: the DSH side is on-disk evidence (src file inventory / grep line numbers); the Claude Code side is the official docs (commands, interactive-mode, terminal-config) + the wil.dev TUI guide

## 1. DSH TUI current-state inventory

`packages/tui/tui/src/` has 79 source files and 13 built-in slash commands (`/theme /session /model /clear /compact /goal /tasks /subagents /workflow /steer /status /config /skills`).

| Layer | Existing capabilities | Key files |
|---|---|---|
| Rendering | stream-renderer, block-stream-writer, write-batcher, live-engine, markdown, ANSI, term-image images, latex | `engine/stream-renderer.ts`, `engine/block-stream-writer.ts`, `engine/term-image.ts` |
| Input | vim state machine (built-in), @-path Tab completion, @mention expansion, external editor Ctrl+O, 20+ ctrl shortcuts | `engine/input-line.ts`, `completion/file-completer.ts`, `mention-parser.ts`, `external-editor.ts` |
| Panels | status/config/question/skill/delegation/workflow/task/keymap/command-palette/restore-session | `status-panel.ts`, `config-panel.ts`, `delegation-panel.ts`, `workflow-panel.ts` |
| Projection | activity state machine, turn statistics, token/cache hits, statusline, metrics glance | `activity-status.ts`, `turn-summary.ts`, `cache-telemetry.ts`, `statusline.ts` |
| Tool visualization | family coloring, timing, parallel group folding, collapsed-bash, fluency control | `format/tool-family.ts`, `tool-elapsed.ts`, `engine/tool-group-controller.ts` |
| Theming | theme/theme-detect/theme-custom/theme-palettes, term-caps, width | `theme*.ts`, `term-caps.ts` |
| Approval | Phase 8 y/N inline answerer (waterfall next() delegation) | `ui/app.ts` around L505 |

## 2. Gaps versus Claude Code (three categories)

### Category A: underlying capability exists, TUI wiring missing (highest ROI)

| # | Gap | Claude Code shape | DSH status | Missing |
|---|---|---|---|---|
| A1 | Operating mode system | three modes normal / plan / auto, shown in the status bar | ✅ Done (2026-08-11) — slash channel falls back to CommandService (`/plan` reachable) + statusline `[plan…]` pending display | No mode-switch shortcut (Tab is taken by completion); the auto tri-state is not built (DSH plan off = normal) |
| A2 | Permission allowlist memory | y/N approval + persistent "always allow" memory | The Phase 8 y/N answerer is wired (`ui/app.ts` L505); `packages/interaction/permission/` has no allow/remember mechanism (grep: 0 hits) | A backend gap (the permission package), not just wiring |
| A3 | Session forking | `/fork` `/branch` | ✅ Done (2026-08-11) — `/fork` `/branch` commands (TuiApp.forkSession: `ctx.sessions.fork` + switchSession switch); a fork inherits history and parentSession lineage | No branch-tree UI (/session list shows lineage); no confirmation dialog (the command switches immediately; can be added later) |
| A4 | Context usage visualization | `/context` shows the window usage distribution | cache-telemetry has a token usage projection | No "how much per message/tool" distribution view |

### Category B: genuinely missing, needs new build (medium effort)

| # | Gap | Claude Code shape | Notes |
|---|---|---|---|
| B1 | Customizable keybindings | `keybindings.json` + `/keybindings` | The DSH keymap-panel is a read-only display; shortcuts are hardcoded in the input-handler/app.ts switch |
| B2 | Inline diff interactive navigation | diff viewer with per-hunk navigation and expand/collapse | `format/diff.ts` is pure rendering (its comment describes itself as a "basic straight port") |
| B3 | Fullscreen transcript viewer | fullscreen mode, `?` shortcut help | scrollback-transcript exists but there is no fullscreen viewer |
| B4 | Reasoning effort switching | `/effort` | DSH has no effort concept |
| B5 | Background detach | `/background` keeps the session running detached | tasks has background tasks; there is no detach of the whole session |

### Category C: management/ecosystem commands (Claude Code has them, DSH mostly doesn't; mostly small commands)

`/init` (generates project memory), `/memory`, `/mcp`, `/permissions`, `/doctor`, `/debug`, `/usage`, `/cost`, `/rewind` (rollback), `/code-review`, `/security-review`, `/btw`, `/prompt-color`, `/output-style`.

DSH already has equivalents: `/config` (covering /config plus part of /permissions), `/compact`, `/clear`, `/model`, `/status`.

Claude Code specialties DSH has not wired: **terminal notifications** (a bell on task completion) and **Shift+Enter newline** (the DSH input line is single-line; multi-line input needs confirmation).

## 3. Priority recommendations

1. **A1 mode system** (plan/auto switching) — the plan-mode package already exists, pure wiring, highest user value, and the Claude Code mental model is already established
2. **A3 session forking** (/fork /branch) — the session layer already supports fork; the TUI adds two commands plus a confirmation dialog
3. **B1 customizable keybindings** — a large change surface (switch dispatch becomes table lookup); a "feel"-level difference
4. **A2 allowlist** — requires touching the permission package (a backend contract change); cross-package, assess separately
5. **B2 diff navigation** — adds an interaction layer on top of the existing format/diff.ts
6. Category C commands as needed; mostly thin wrappers

## 4. Evidence and limitations

- Every DSH-side conclusion has on-disk evidence: the src file inventory (glob), the command registry (`commands/registry.ts` L215-432 + `ui/app.ts` L405-433), plan wiring (`ui/app.ts` L77), permission allowlist (grep: 0 hits), diff as pure rendering (the `format/diff.ts` L2 comment)
- The Claude Code shortcut table captured only a fragment of the official docs (the interactive-mode page was truncated), so individual key positions were not checked one by one; the capability inventory comes from the official commands + interactive-mode docs
- This document only compares and commits to no implementation scope; implementation follows standalone planning documents

## 5. Implementation and acceptance record (2026-08-11)

| Item | Status | Commit |
|---|---|---|
| A1 mode system (slash fallback + [plan…] pending) | ✅ Done | `4dca7d4` |
| A3 session forking (/fork /branch) | ✅ Done | `ee23cd9` |

**Acceptance status**: user-level behavior acceptance for A1/A3 and the C2 series (approval diff / history search / /model hot swap; see `docs/dsh-tui-与grok的功能对比-c2.md`) is **blocked** — the execution environment has no interactive TTY. Real-assembly probe evidence obtained: TUI profile startup ✓, restore-session list and fork lineage rendering ✓ (A3's parentSession metadata is genuinely visible), Ctrl+F alt-screen entry/exit sequence ✓. Full user acceptance awaits execution in a terminal environment (acceptance steps live in each Agent Note's Risks).

## Follow-ups (pending in this series)

- C2: comparison with Tianshu (天枢) TUI (opencode-tui) — existing content lives in `docs/dsh-tui-next-phase.md` and can be excerpted and merged in
- C3: implementation priorities and wave plan (after Category A/B decisions are made)
