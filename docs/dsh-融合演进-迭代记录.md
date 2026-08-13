# DSH × Tianshu Fusion Evolution Iteration Record (from the 2026-08-09 snapshot baseline)

English | [中文](dsh-融合演进-迭代记录.zh.md)

> **Positioning**: this file records the local evolution starting from the snapshot branch `snapshots/20260809T140917Z-a6bb5a95ba` (the 2026-08-09 official snapshot), for later repository documentation. **Nature**: an iteration capability inventory with evolution evidence (commit hashes are verification records), not decision records (→ Agent Notes) and not a system map (→ [architecture.md](architecture.md)). **Coverage**: 140 local commits (2026-08-09 → 2026-08-11), 4 new packages + core session-model extensions. **Related**: [roadmap](dsh-tui-next-phase.md) | [C1 comparison](dsh-tui-与claude的对比-c1.md) | [C2 comparison](dsh-tui-与grok的功能对比-c2.md) | [C3 enhancement plan](dsh-tui-增强方案-c3.md) | [C4 split plan](dsh-tui-拆分方案-c4.md) | [handoff doc](../.agents/notes/implemented/process/2026-08-10-tui-handoff.md)

## Baseline profile (snapshot branch = 2026-08-09)

The evolution starting point is a harness of **headless agent core + capability plugin families**:

- **Spine** (`packages/core/`): sessions / system-prompt / tools / agents / agent-default-model / agent-loop
- **Capability families** (capability seams): llm, bash, subprocess, pty, fs (fs/fs-local/fs-policy/fs-sandbox/tool-fs/tool-fs-search/tool-str-replace-editor), lsp, skill, web, compact, subagent, workflow, tasks, plan, goal, todo, code-runtime, sandbox
- **Data plane**: session (JSONL/SQLite persistence), session-query, session-title, settings, credentials, storage, workspace
- **Frontend plane**: only the ACP automation server, the Web GUI (host/client), and scaffold (the CLI launcher) — **no interactive terminal frontend**
- **guard group**: only 2 packages (repeat-tool-guard reminders, timeout-policy timeouts) — behavioral hygiene amounts to "reminders" only, with no discipline mechanism
- **Session model**: the session log is **append-only** (the only operation is appending; no rollback/truncation)

## Evolution overview (140 commits = 4 new packages + core extensions)

| Addition | Location | Essence | Evolution evidence |
|---|---|---|---|
| `dsh-tui` | packages/tui/tui | a brand-new third frontend plane (interactive terminal UI) | 457 commits since `f825fd1` ported the Tianshu (天枢) rendering core |
| `dsh-fs-snapshot` | packages/fs/fs-snapshot | workspace file-state snapshots (the physical substrate for rewind) | `277657e`/`c6764f5`/`42c4da3` |
| `dsh-evidence-gate` | packages/guard/evidence-gate | behavioral discipline mechanism (RED→GREEN verification gate) | S1–S6 series, from `1db1b35` |
| `dsh-agent-router` | packages/guard/agent-router | behavioral discipline mechanism (failure prediction → routing escalation → native subagent dispatch) | S7–S8 series, from `d458e36` |
| `Session.truncate` | packages/session | core model: append-only → rewindable | `62d1e76` |
| `deleteFrom`/`truncateStored` | packages/session/session-persistence | cross-reload truncation at the persistence layer | `e4a057e` |
| examples/tui + headless-agent e2e | examples/ | real-assembly verification surfaces | 20 changes from `4ca8b68` |

Commit distribution (by top-level path): tui 457, guard 70, fs 13, session 10, examples/headless-agent 20, a handful across workflow/subagent/scaffold/hooks/boot, and 44 Agent Notes.

## 1. Frontend plane: TUI (the largest capability block)

### Architectural position

The TUI layer of `dsh --profile`; the bundle patch rides on top of dsh-base (stable plugin id `tui-runner`); the rendering core is ported from the Tianshu terminal engine (Apache-2.0, per-file provenance in SOURCE-MAP). **A pure presentation layer**: all state arrives through session events and the projection bus, it invents no new event types and contains no agent logic, honoring "Model-visible ⟺ logged". Layering: `src/engine/` (the ported terminal rendering engine), `src/ui/app.ts` (TuiApp assembly), `src/adapter/` (adapting dsh services to `TuiPort`).

### Filling the interaction plane (a baseline gap)

- Registers a `userInteraction` provider → in-terminal answering (question panel, plan-review 🧭 decision card)
- Subscribes to `approval/request` → pending approval card + y/N/Ctrl+C settlement + approval diff preview (inline unified diff)
- Commands register on `ctx.commands`: the slash-command system is discovered through the Cordis service registry, no model turn required

### Panorama of in-terminal capabilities

- **Internal-state visualization** (Phase 5): six-stage workflow indicator, live activity labels, metrics glance-bar (model/turn/elapsed)
- **Interaction efficiency** (Phase 6): the slash-command system, mid-turn steering (the `agent.steer()` entry), @-path completion, external editor (Ctrl+O), Vim mode (normal/insert/visual), Ctrl+F history search (smart-case + n/N jumps)
- **Tool visualization** (Phase 7): tool run timers, family coloring, parallel-group folding, streaming tail, turn summary
- **Session management**: /fork /branch (`ctx.sessions.fork`), the restore-session panel, /model hot-switching, **/rewind two-phase rollback** (message list → granularity, coordinating fs-snapshot + Session.truncate)
- **Mode system**: Shift+Tab three-state cycling (normal/plan/always-approve), the plan approval gate + request-changes feedback path (the `f` key enters feedback text; this fixed the shape break between the TUI submit shape and the provider contract — under real assembly it used to throw a TypeError)
- **Experience polish** (Phase 9): @mention parsing (user-side summary expansion, cwd boundary), fluency control (tiered on-screen stale hints), command palette (Ctrl+P), CJK width awareness

### Architectural pattern evolution (the C4 split)

app.ts as a 1727-line monolith → Wave 1 extracted the Question/Approval controllers (with unit tests) → Wave 2 turned live-panels into pure functions (the renderLive combinator) → Wave 3 converged orphaned controllers (compared case by case against the inline logic, deleted on semantic mismatch, leaving nothing "half-extracted") + dispose/detach lifecycle discipline (subscription-ledger balance assertions: every `ctx.on` collects its disposer, mount/unmount release symmetrically, the `??` short-circuit trap is fixed). Accompanied by the `scripts/verify-source-budgets.ts` line-count ratchet gate (app.ts budget: 1831 lines).

## 2. Behavioral hygiene: the guard family, 2 → 4 packages

The baseline guard only "reminds"; the evolution adds **mechanism-level discipline**, turning prompt-layer requirements into plugin-layer facts:

- **evidence-gate (verification gate)**: RED→GREEN discipline for bugfix tasks — the L1 edit gate (blocks the first edit without a RED reproduction + precise probe suggestions: test path and expected result), the three RED rules (failed records evidence; passed requires a prior RED; blocked ≠ proven), the TDD gate (≥3 consecutive edits without verification → suggest/enforce), the L2 final gate (continue_once / honest_blocked disclosure); **verification accounting is detected automatically from tool/call→tool/result pairs on `session/event`** (command-text heuristics, zero test-framework coupling); adapted to the native `str_replace_editor` (write commands intercepted, view reads pass through)
- **agent-router (routing layer)**: a prediction accumulator (sliding window of 10; error rates 0.4/0.6/0.8 → three intervention tiers hint/gate/escalate; 3 consecutive correct → tipping-point reset) + a deterministic routing table (escalate → delegate verifier; gate + probe cooldown exhausted → delegate code_scout; obligations unresolved + zero verification → self writes a probe first) + **native dsh subagent dispatch** (`ctx.agents.create` → followup injects the task → whenIdle waits → dispose; results are accounted back to evidence-gate automatically via session/event, zero new channels); configurable degradation (dispatchEnabled: false decides only, without echoing)

Both are verified through the `examples/headless-agent` real-assembly e2e (Loader load, task-boundary wiring, tools via reflect.get, exit-code failure detection), and each package ships an `invariant.ts` runtime-invariant companion.

## 3. Core model extension: append-only → rewindable

- `Session.truncate`: event-log rollback + derived-state reset (surface/header/context folds)
- `session-persistence`: the `deleteFrom` backends (JSONL/SQLite) + the `truncateStored` coordinator — **truncation persists across reloads**

This extends the session from "append only" to "rewindable and replayable", the foundation of /rewind (file+session dual-track rollback).

## 4. File-state snapshots: fs-snapshot

- The `tools/execute` waterfall injects the trackEdit hook: a full-text snapshot **before** each write tool executes (`str_replace_editor` write commands / `write` / `edit`) (FileHistory ported from opencode-tui, Apache-2.0)
- **Orthogonal** to checkpoint-policy: checkpoint is event-log persistence (against losing a turn on crash); this plugin is file-content snapshots (for rewind's file rollback)
- Consumption surface: the `HISTORIES_KEY` per-session FileHistory index, `rewindToBoundary(boundaryId)` restore (`backupFileName === null` means the file did not exist then → unlink); snapshots capped at 100, evicting the oldest on overflow
- Declared boundaries: direct bash writes are invisible (the same blind spot as evidence-gate's edit gate), the snapshot index is in-memory (not rebuilt across restarts; a persisted index is deferred work), and the default backup directory is in tmpdir

## 5. Real-assembly verification surfaces

- `examples/tui`: a runnable TUI composition (settings + credentials + the real DeepSeek adapter + a pre-created `main` agent + the `tui-runner` plugin, with fs-snapshot wiring)
- `examples/headless-agent`: became the guard family's real-assembly e2e proving ground (evidence-gate S5/S6, agent-router S7/S8: subagent mock round-trip, verifier real-turn dispatch, fail-loop mock)
- A repo-wide coverage-gate push: 119 violations → about 10; app.ts coverage gaps zeroed; **typecheck debt 86 → 0** (the test layers of five packages — tui/subagent/workflow/hooks-claude/fs-snapshot — with zero behavior change)

## 6. Architectural judgments

1. **Everything goes through the plugin mechanism, with zero agent-loop changes** — the new frontend (TUI), the new discipline (guard×2), and the new persistence (fs-snapshot) are all consumers of existing extension points (`ctx.on` / waterfall / registerProvider / inject), consistent with "everything is a plugin".
2. **The frontend goes from "two planes" to "three planes"**: ACP (automation) / Web (browser) / TUI (terminal); the TUI is the only frontend that fills both pending channels, `userInteraction` and `approval/request`.
3. **Behavioral hygiene upgrades from "hints" to "mechanism"**: evidence-gate + agent-router form the "failure prediction → verification discipline → routing escalation → event accounting" loop, with accounting reusing the session event stream (zero new channels).
4. **The data model extends from append-only to rewindable**: truncate lands at three levels — session, persistence, and files.

## 7. Type-aware lint debt: 183 → 0 and the lint-budgets ratchet

The type-aware oxlint debt carried by the TUI line (183 diagnostics at the 2026-08-11 inventory, 171 by measurement time) is now **zero**: `pnpm run lint` exits 0 and the full-repo oxlint pass reports no errors. The cleanup ran by rule family (one adjudication per family — unbound-method 42, no-unnecessary-condition 22, no-unsafe-\* 43, restrict-plus-operands 11, no-unnecessary-type-conversion 16, and the small families), fixing the *types* where boundaries were real (terminal capability / environment / model tool JSON) and deleting guards only where the types were honest (`9a76921`). The baseline `ctx.emit` overload breakage in guard/hooks/core tests (17 diagnostics present at the snapshot) was adjudicated alongside: test harnesses emit loosely by design, expressed with `@ts-expect-error` + rationale instead of `as any` (blocked by `typescript/no-explicit-any: error`).

To keep the debt from regressing silently, `scripts/verify-lint-budgets.ts` + `scripts/lint-budgets.manifest.json` ratchet the `typescript/*` family per file — an empty manifest today means **zero tolerance everywhere**, and any intentional new debt must add a manifest entry in the same PR (`3c82af2`). The ratchet attributes violations per file (the full-repo lint gate stays the authority for all diagnostics) and is verified by `scripts/verify-lint-budgets.spec.ts`.

## Current state and outstanding debt

- Test baseline: TUI suite 1292 passed / 2 todo (2026-08-12); the lint-touched packages (tui, subagent, guard, hooks-claude, core/scope) together 1776 passed / 2 todo, 101 files
- **Working tree clean**; the T2.3 listener-lifecycle wrap-up, source-budgets gate wiring, fs-snapshot → tui example wiring, and tui-group invariant/tsconfig inclusion all landed earlier (abccfde/e37003b and neighbors)
- **174 local commits unpushed** (branch `snapshots/20260809T140917Z-a6bb5a95ba` is ahead of origin)
- **Documentation debt cleared**: the `packages/guard/README.md` group table lists agent-router/evidence-gate and `docs/architecture.md` reflects truncate and the guard extensions (both since `e37003b`); this iteration record is the live remainder
- About 10 coverage-gate residues; ⚠ `tsc -b --force` hangs on this machine (suspected corrupted tsbuildinfo cache; clear `*.tsbuildinfo` first, and `-b` must run serially to prevent deadlock); plain `tsc -b tsconfig.host.json` passes once the cache is cleared
- Known boundary: user-level TTY acceptance is blocked (the execution environment has no interactive terminal); behavioral evidence rests on unit tests and real composition tests
