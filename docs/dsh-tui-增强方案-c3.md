# DSH TUI Four Enhancements — C3

English | [中文](dsh-tui-增强方案-c3.zh.md)

> **Tag**: C3 (3rd in the DSH TUI enhancement series; C1 benchmarks Claude Code, C2 benchmarks grok-build) **Date**: 2026-08-11 (initial draft) / revised 2026-08-11 (added the rewind Tianshu (天枢) port + Shift+Tab) **Scope**: rewind rollback (copied from Tianshu), plan approval gate, /fork enhancement, Shift+Tab mode cycling **References**: [dsh-tui-与claude的对比-c1.md](dsh-tui-与claude的对比-c1.md), [dsh-tui-与grok的功能对比-c2.md](dsh-tui-与grok的功能对比-c2.md)

## Key up-front finding: the four items differ hugely in actual status

Investigation confirmed that the four items are **not the same kind of work**:

| Item | Pre-investigation assumption | Post-investigation fact | Nature |
|---|---|---|---|
| rewind UI | checkpoints exist underneath, only the UI is missing | checkpoint is just flush; **but Tianshu's `FileHistory` core rewind path is self-contained and portable** (≈314 lines; the core path is ~200 lines with zero external dependencies, while the diff-stat part depends on a worker pool and must be stripped), turning rewind into port + adapt | 🟡 Port + one backend gap (SessionStore.truncate) |
| plan approval gate | plan-mode exists underneath, only the approval UI is missing | the chain exists but has a **shape break** (diagnostic revision: TUI settle submits `{questionId,value}`/`{cancelled}`, user-interaction passes straight through to the provider with no conversion, and exit_plan_mode reading `answer.answers` throws a TypeError) — fix the shape first + then add the feedback path | 🔴 Fix the break + pure TUI enhancement |
| /fork entry | forkable underneath, TUI has no command | **the `/fork` command already exists and works**; only optional arguments are missing | 🟢 Pure TUI enhancement |
| Shift+Tab mode cycling | needs building from scratch | **key parsing is ready** (`shift_tab` is already in KeyName), the plan-mode API is available; missing the always-approve second axis + handleKey wiring | 🟡 TUI + minimal local state |

**Conclusion**: plan is "fix the shape break + close the last mile" (diagnostic revision: the chain is not usable end to end); fork is "close the last mile"; Shift+Tab is "wiring + a local flag"; rewind is "port Tianshu + add SessionStore.truncate". All four are feasible, with effort ranging from ~35 lines to a cross-package port.

---

## Item 1: plan approval gate enhancement (pure TUI, minimal effort)

### Current state: the chain exists but the shape is broken (diagnostic revision 2026-08-11)

**Diagnostic revision**: the C3 initial draft's claim that "the end-to-end chain already exists" was a static, optimistic assertion. The sky-survey scout swarm + independent verification found the chain broken **at the shape level**:

- TUI `settleQuestion` submits `{ questionId, value: option.label }` (app.ts:1215) or `{ cancelled: true }` (app.ts:1210) — the TUI layer's own contract, pinned by app.spec.ts:2183;
- `user-interaction`'s `ask()` passes straight through to the provider with no shape conversion of any kind (user-interaction/src/index.ts:153);
- plan-mode `exit_plan_mode` expects `{ answers: [{ id, selected[], custom? }] }` (plan-mode/src/index.ts:365 `answer.answers.filter(...)`) — the TUI's return value has no `.answers`, and **under real assembly this throws a TypeError**;
- the cancellation path is broken the same way: `exit_plan_mode` expects a rejection with `UserInteractionError('...', 'ASK_CANCELLED')`, while the TUI resolves normally with `{cancelled:true}`, which never lands in the cancellation classification;
- the two test suites each pin their own shape (app.spec.ts:2183 vs user-interaction.spec.ts:16/38), and the integration point has no test coverage — a textbook break masked by loose mocks.

**Fix direction** (a prerequisite step for implementing item 1): on the TUI submit side, wrap the answer into the provider-contract shape `{ answers: [{ id, selected: [label], custom? }] }`, and change the cancellation path to reject with `UserInteractionError` (code `ASK_CANCELLED`); or, per AGENTS.md "Trust TypeScript at typed same-process boundaries", perform one explicit conversion at the service boundary (the choice is decided in the implementation batch).

The full chain (after the shape fix):

```
agent 调 exit_plan_mode(plan)
  → PlanModeService 调 ctx.userInteraction.ask({ intent: { kind: 'plan-review' }, detail: plan, options: [Approve, Keep planning] })
    → TUI 的 handleQuestionRequest 挂起为 pendingQuestion（app.ts:577-595）
      → question-panel.ts 识别 intent.kind === 'plan-review'（L111），渲染 🧭 决策卡（计划正文 + ✓/✗ 选项）
        → 用户数字键选 → settleQuestion（L1215）→ exit_plan_mode 返回 { approved } 或抛反馈
```

**Files confirmed to exist**:
- `packages/plan/plan-mode/src/index.ts:302-389` — the `exit_plan_mode` tool + the plan-review intent
- `packages/tui/tui/src/question-panel.ts:111` — the `intent?.kind === 'plan-review'` rendering branch (🧭 card + ✓/✗ markers; detail rendered as flat indentation at L120-124)
- `packages/tui/tui/src/ui/app.ts:1207-1219` — pendingQuestion key handling (digit keys 1-9 select / Esc cancels; settle submits at L1215; suspension at L577-595)
- `packages/interaction/user-interaction/src/types.ts` — AskUserQuestionIntent includes the `plan-review` kind

### Gaps

0. **Shape break (prerequisite, see above)**: the TUI's submit shape does not match the provider contract, and `exit_plan_mode` throws a TypeError under real assembly — this must be fixed first, otherwise even the existing "approve" path is unusable.

1. **No "request-changes with feedback" path**: `exit_plan_mode` reads `item.custom` as the feedback text (plan-mode/index.ts:368-371, `const feedback = item?.custom ?? ''`), but the TUI's handleKey only sends `value` or `cancelled` — "keep planning" carries no feedback text. The user cannot say in the TUI "step 3 of this plan is wrong, change it to X".

2. **The plan body renders as flat indented lines**: question-panel.ts renders `detail` (the plan markdown) as line-by-line indentation, with no markdown structure (headings/lists/code blocks). A long plan fills the live region and cannot scroll.

### Approach

#### Change 1: add the request-changes feedback path

**Change `packages/tui/tui/src/ui/app.ts`** (handleKey's pendingQuestion branch, L1207-1219):
- Current: digit keys 1-9 → pick an option; Esc/Ctrl+C → cancelled
- **Prerequisite: change the settleQuestion submit shape to the provider contract** `{ answers: [{ id, selected: [label], custom? }] }`, and change the cancellation path to reject `UserInteractionError('...', 'ASK_CANCELLED')` (fixes the shape break, see the "Current state" section)
- New: the `f` key (feedback) → enters "feedback input mode"; the input box accepts text, and Enter submits it as `{ answers: [{ id, selected: ['Keep planning'], custom: <输入文本> }] }`
- Reuse input-line's editing capabilities (vim/cursor/history already exist); add only one "feedback state" flag + a rendering hint

**Change `packages/tui/tui/src/question-panel.ts`**:
- Add key hints at the bottom of the plan-review card: `[1] 批准  [2] 继续规划  [f] 反馈修改  [Esc] 取消`

#### Change 2 (optional, second batch): markdown rendering of the plan body

**Change `packages/tui/tui/src/question-panel.ts`**:
- Switch detail rendering from "line-by-line indentation" to reusing `format/markdown.ts` (heading/list/code-block rendering already exists)
- When a long plan exceeds N lines, fold it + hint "see the full plan in the alt-screen viewer" (a follow-up wires up the overlay full-screen viewer)

### Scope and effort
- **First batch = fix the shape break + Change 1** (the feedback path): ~15 lines of shape conversion in app.ts (settleQuestion/cancellation path) + ~40 lines app.ts + ~10 lines question-panel.ts + spec + an **integration test** (add contract-alignment assertions between app.spec and user-interaction, covering the whole "TUI submit → plan-mode reads answers" chain)
- Change 2 stays for a later batch (markdown rendering + the full-screen viewer are independent effort)

---

## Item 2: /fork enhancement (pure TUI, minimal effort)

### Current state: the command already exists

**Complete wiring confirmed to exist**:
- `packages/tui/tui/src/commands/registry.ts:262-267` — the `/fork` command (calls `deps.forkSession()`; the branch alias follows right after)
- `packages/tui/tui/src/ui/app.ts:671-680` — `forkSession()`: `ctx.sessions.fork(activeSessionId)` → `switchSession(child.id)`
- `packages/core/session/src/index.ts:1095` — `SessionStore.fork(source, boundary?, childSessionId?)`: seed = `events[0..boundary]` (defaults to the last event), meta carries `parentSession`/`seedLength`
- `packages/tui/tui/src/restore-session.ts:63` — the restorable-session list already renders fork lineage (`fork: <parentSession>`)

### Gaps

The current `/fork` takes no arguments and offers no boundary selection. grok's `/fork` supports:
- `<directive>` — the first prompt after the fork (the direction to explore on the branch)
- `--worktree` / `--no-worktree` — git worktree isolation (**DSH has no worktree concept; skip**)
- `--at <turn>` — fork from a given turn (DSH's `fork(source, boundary)` already supports a seq boundary)

### Approach

#### Change: add optional directive + boundary to /fork

**Change `packages/tui/tui/src/ui/app.ts`**:
- `forkSession()` → `forkSession(opts?: { directive?: string; atSeq?: number })`
- Pass `atSeq` through to `ctx.sessions.fork(activeSessionId, atSeq)`
- `directive`: after fork + switchSession, submit it as the first message via `this.controls?.followup(directive)`

**Change `packages/tui/tui/src/commands/registry.ts`**:
- The `/fork` run handler parses arguments:
  - plain text → directive
  - `/fork <text>` → fork + submit text as the first message
  - no arguments → current behavior (plain fork)
- `/fork at <seq> <text>` → with a boundary (advanced usage; the first batch can ship directive only and leave boundary for later)

**Change `packages/tui/tui/src/adapter/sessions.ts`**:
- `forkSession(ctx, source, boundary?, childSessionId?)` already exists and supports boundary; no change needed

### Scope and effort
- The first batch ships directive only (`/fork 探索另一种方案` → auto-submit after fork): ~20 lines app.ts + ~15 lines registry.ts + spec
- The boundary-selection UI (picker) stays for later — it needs a turn-list projection, an independent chunk of work
- worktree is never done (DSH has no such concept; session-level fork suffices)

---

## Item 3: rewind rollback (🔴 needs new backend work)

### Current state: the underlying layer is entirely missing

This is the **only genuine gap** among the three, and the gap sits in the backend, not the TUI.

#### The truth about checkpoint

The core thing `packages/session/session-checkpoint-policy/src/index.ts` (≈78 lines) does is `ctx.sessions.flush(session)` — **persisting** the in-memory append-only event log to disk (against losing a turn on crash). **"checkpoint" = flush, not a snapshot** (the tools/execute branch additionally has an `exec.signal.aborted` check returning a canonical error, so it is not pure flush).

Three flush points:
- before `llm/stream` (before the first chunk is emitted)
- before `tools/execute` (before a top-level tool body runs)
- before `agent/pre-step`

**There is no file snapshot of any kind.** What flush produces is the JSONL/SQLite event log, not a revertible file state.

#### The four missing components

| Component | grok has | DSH current state |
|---|---|---|
| file snapshot storage | `RewindPointInfo.num_file_snapshots` + server-side snapshots | ❌ `str_replace_editor` reads `before` but **discards it** (used only to compute the diff, never persisted) |
| per-turn changed-file index | `has_file_changes` | ❌ no `file/changed` event; changed files can only be inferred from tool/call+tool/result pairs |
| rewind API | server-side `rewind(sessionId, promptIndex)` returning `{ reverted_files, clean_files, conflicts }` | ❌ `Session` only has append/create/restore/fork, **no truncate/rewind/revert** |
| rewind UI | the `RewindPhase` state machine (Loading→Picker→Confirm→Executing→Result), ~900 lines | ❌ only a "rewind needs a full repaint" comment at live-engine.ts:668, no implementation |

#### Reserved-but-unimplemented traces

The type system **reserved** rewind, but nothing produces it:
- `ConversationContextOriginKind = 'compaction' | 'rewind' | 'rewrite'` (client/runtime)
- `history-fold.ts` recognizes a user/message with `source.plugin === 'rewind'` as opening a new context branch
- **but no code emits `plugin: 'rewind'`** — compact does (`compact/checkpoint.ts` writes `plugin: 'compact'`), rewind does not

### Approach: two phases

#### Phase 1 (not in the first batch; produce a design doc): session-level rewind (no file rollback)

**Minimum viable**: rewind = fork a new session at the target prompt + switch to it. Uses the existing `SessionStore.fork(source, boundary)`.

- Pros: pure assembly layer; touches neither the session core nor persistence
- Cons: **does not roll back files** (files the agent changed remain), and abandons the original session id (fork produces a new id)
- Fits: a user who wants to "restart the conversation from here" and does not mind file state

Implementation: the `/rewind` command → a turn-list picker (projected from the transcript) → pick the target turn → `ctx.sessions.fork(activeSessionId, turnEndSeq)` → `switchSession(child.id)` → mark the user/message with `plugin: 'rewind'` (reusing the classification history-fold already reserves)

#### Phase 2 (backend project, separate PR): file snapshots + true rewind

This is the hard work of grok-grade rewind, **not in the first batch**:

1. **A new file-snapshot package** (`packages/snapshot/` or `packages/fs/fs-snapshot/`):
   - Take a full-file before-image snapshot of each file about to change, before tool execution (the `tools/execute` waterfall)
   - Snapshot storage: an in-memory Map + optional persistence (JSONL/SQLite)
   - The write/edit paths of `str_replace_editor` / `tool-fs` must inject snapshot points

2. **A new rewind API** (`packages/core/session/` or a new package):
   - `ctx.sessions.rewind(sessionId, promptIndex): Promise<RewindResult>`
   - `RewindResult = { revertedFiles: string[]; cleanFiles: string[]; conflicts: RewindConflict[] }`
   - Truncate the event log to the target seq (requires a new `SessionStore.truncate(id, atSeq)` + a persistence-backend `deleteFrom(atSeq)`)
   - Restore files from snapshots + detect conflicts (a file externally modified after its snapshot)

3. **The TUI rewind UI** (modeled on grok's `RewindPhase` state machine):
   - `/rewind` → turn picker → confirm (showing `N 个文件将回退`) → execute → result (X files reverted / Y files without snapshots need manual handling / Z conflicts)

**Effort assessment**: Phase 2 is backend engineering across 3 packages (snapshot storage + session truncate + persistence deleteFrom), not TUI wiring; it should be an independent project.

### Item 3 revision: copy Tianshu — rewind becomes portable

The C3 initial draft judged rewind to be backend engineering, based on "DSH has no file snapshots underneath". **Re-reading the Tianshu source overturned that judgment**: Tianshu's `FileHistory` is self-contained and portable, turning rewind from "build a backend from scratch" into "port + adapt".

#### The core mechanism of Tianshu rewind (the portable part)

Tianshu uses **two independent mechanisms**; the precise path is the one the TUI rewind uses:

**The precise path (FileHistory) — copy this:**
- `src/agent/file-history.ts` (314 lines): the `FileHistory` class, **depends only on `node:fs/promises` + `node:crypto`**, self-contained
- Snapshot timing: in the `tools/execute` waterfall, **before every `write_file`/`edit_file` executes**, call `trackEdit(filePath, toolUseId)`, which reads the current full text and writes a disk backup
- Storage: `<sessionDir>/<sessionId>/backups/<sha256(path)[:16]>@v<N>` (full text, content-addressed by path); index in memory + `file-history.json` persistence; capped at 100 snapshots
- Rollback: `rewindToBoundary(postBoundaryIds: Set<string>)` — for each file, find the earliest backup after the boundary (= the pre-boundary state); `backupFileName === null` → the file was newly created → unlink
- Index: `MAX_SNAPSHOTS = 100`, evicting the oldest on overflow

**The coarse path (checkpoint.ts, git baseline) — do not copy:**
- Takes a git snapshot before each turn's first mutating tool, covering files changed by bash; if DSH changes files only via `str_replace_editor`/`tool-fs`, the precise path already suffices

**The key algorithm (`collectPostBoundaryEditIds`, file-history.ts:18)**: scan `messages[messageIndex..]` and collect the `tool_use` ids of every `write_file`/`edit_file` — this is the single source of truth for the boundary, shared by the CLI and the server.

#### DSH adaptation points

| Tianshu | DSH counterpart | Adaptation |
|---|---|---|
| `tool-pipeline.ts` L1290 `trackEdit` hook | `packages/core/tools/src/index.ts:149` `tools/execute` waterfall | Add the snapshot hook **before** `next()` (same pattern) |
| the `write_file`/`edit_file` tool names | DSH registered names: `str_replace_editor` (command enum = `view`/`create`/`str_replace`/`insert`, **no write**) + `write` + `edit` (tool-fs) | **collectPostBoundaryEditIds must check 3 registered names** (str_replace_editor/write/edit); missing one means missed snapshots. The TUI display layer's `write_file`/`edit_file` are just a mapping — do not confuse them |
| `ctx.session.rewindToMessages(msgs.slice(0, n))` | DSH `Session` is append-only, **no truncate** | **The real gap**: needs `SessionStore.truncate(id, atSeq)` + persistence `deleteFrom` |
| `messages[messageIndex]` OAI message model | DSH `SessionEvent[]` event log | Adapt `collectPostBoundaryEditIds` to scan the callId of `tool/call` events |
| `tuiApp.setInput(content)` refills the input | DSH `this.inputLine.setValue(content)` | Direct correspondence |
| `<sessionDir>/<sessionId>/backups/` | **the backups convention has zero presence on the DSH side** (grep of the whole packages/session tree finds no match) | New convention: copy Tianshu's `<backupDir>/<sessionId>/<sha256(path)[:16]>@v<N>` shape, with backupDir passed in from the TUI data directory |

**The only genuine backend gap**: `SessionStore.truncate(id, atSeq)`. Tianshu's `rewindToMessages` truncates an in-memory array in place; DSH's `Session` is an append-only event log, so truncation requires:
1. Add a `truncate(id, atSeq)` method to `SessionStore` (in memory: `events.splice(atSeq+1)`)
2. Add `deleteFrom(id, atSeq)` to the persistence backends (JSONL: rewrite the truncated file; SQLite: `DELETE WHERE seq > atSeq`)
3. Reset derived state after truncation (token estimates, turn counters, etc. — see the full reset in Tianshu context.ts:355)

#### Item 3 in two phases

**Phase 1 (this batch; session + file dual rollback)**:
1. Port `file-history.ts` (≈314 lines) → `packages/fs/fs-snapshot/src/` (a new package, or under `packages/session/`) — **note**: the Tianshu version's dependency set includes cpu-pool/cpu-tasks (the worker pool behind `getDiffStats`); "only node:fs/crypto" holds only for the core rewind path. Recommendation: port a trimmed core of ~200 lines (trackEdit/rewind/rewindToBoundary/getBoundaryFiles) and add diff stats later or reuse DSH's existing diff facilities
2. Inject the `trackEdit` hook into the `tools/execute` waterfall (with DSH tool-name detection)
3. Port `format/rewind.ts` (183 lines) → `packages/tui/tui/src/format/rewind-overlay.ts` (pure rendering; final filename)
4. Wire the rewind overlay into the TUI (reusing `OverlayController`, two phases: message list → rollback-granularity selection)
5. Add `SessionStore.truncate(id, atSeq)` + JSONL/SQLite `deleteFrom` (**the only backend change**; it lands in the session-persistence-jsonl / session-persistence-sqlite implementation packages, and the transaction boundary needs dedicated design — the only destructive change to the append-only log, so proceed carefully + test)

**Phase 2 (later)**: summarize modes (`summarize-from`/`summarize-to`) — needs an LLM summarizer, large effort, deferred

#### RewindMode (5 in Tianshu; the first batch ports 3)

| Mode | Session | Files | First batch |
|---|---|---|---|
| `convo` | truncated | untouched | ✅ |
| `code` | untouched | reverted | ✅ |
| `both` | truncated | reverted | ✅ |
| `summarize-from` | post-boundary replaced by a summary | untouched | ❌ deferred |
| `summarize-to` | pre-boundary replaced by a summary | untouched | ❌ deferred |

#### Tianshu rewind file index (port source)

| Tianshu file | Lines | Role | Port strategy |
|---|---|---|---|
| `opencode-tui/src/agent/file-history.ts` | 314 | data layer (snapshot/rollback/index) | near-verbatim port; change tool-name detection + directory convention |
| `opencode-tui/src/agent/file-history-persist.ts` | 43 | persistence (file-history.json) | as-is |
| `opencode-tui/src/tui/format/rewind.ts` | 183 | rendering (two-phase panel) | as-is; update the overlay-frame reference |
| `opencode-tui/src/agent/context.ts:355` | — | the full derived-state reset in `rewindToMessages` | reference contract; DSH adapts it to event-log truncation |
| `opencode-tui/src/tui/__tests__/format-rewind.test.ts` | 107 | rendering tests | port as-is |
| `opencode-tui/src/agent/__tests__/file-history.test.ts` | 155 | data-layer tests | port as-is |

---

## Item 4: Shift+Tab mode cycling (new)

### Goal

Shift+Tab cycles Normal → Plan → Always-Approve → Normal (aligned with the grok/Claude Code mental model).

### Current state: the infrastructure is largely ready

| Capability | Status | Evidence |
|---|---|---|
| Shift+Tab key parsing | ✅ ready | `input-handler.ts:75` has `'shift_tab'`; ANSI `[Z` mapping (L174); CSI `;2` shift parsing (L482) |
| plan-mode service | ✅ ready | `ctx.planMode`; `set(agent, active)` returns `'committed'|'queued'|'cancelled'|'noop'`, with pending boundary refresh |
| plan state projection | ✅ ready | the TUI receives `{active, pending}` over the projection bus and renders the `[plan]`/`[plan…]` badges |
| the `[plan]` badge | ✅ ready | `statusline.ts:212-217` |
| **handleKey wired to shift_tab** | ❌ missing | `handleKey` has no `shift_tab` branch; the key falls through into InputLine |
| **always-approve mode** | ❌ missing | DSH has no session-level "approve everything" concept; approval is per-tool-call (`approval/request`) |
| permission service | 🟡 exists | `ctx.reflect.get('permission')` (app.ts:882) — `PermissionService` (**packages/interaction/permission**, not core/) has a preset table, but **no "approve everything" tier**: DSH's policy vocabulary is only `'ask'|'never'`, and never = deterministic rejection (fail-closed), not approve-everything |

### Design decision: two separate axes (aligned with grok, not Claude Code)

**Adopt the grok model, not the Claude Code model**:
- grok: plan and permission are **two orthogonal axes**; Shift+Tab touches one axis at a time around a small cycle (Normal→Plan→Always-Approve→Normal)
- Claude Code: everything is a single "permission mode" axis (default→acceptEdits→plan→auto)

**Rationale**: DSH's plan-mode module **explicitly declares an invariant** — "plan mode is independent of sandbox mode and approval policy" (plan-mode/src/index.ts:7). Merging them into a single axis would violate that invariant.

### The three-state cycle

```
Normal → Plan → Always-Approve → Normal
```

- **Normal → Plan**: `ctx.planMode.set(agent, true)` (existing API)
- **Plan → Always-Approve**: `ctx.planMode.set(agent, false)` + turn always-approve on
- **Always-Approve → Normal**: turn always-approve off

### Gap: the always-approve second axis

DSH has no session-level "approve everything". It needs building (minimal approach):

**Option A (recommended, smallest change)**: a TUI-local flag
- Add `private alwaysApprove = false` to `TuiApp`
- At the top of `handleApprovalRequest` (app.ts:1060; registration point `ctx.on('approval/request')` L538-540) add: `if (this.alwaysApprove) return 'allowed-once'` (short-circuit, bypassing the pending prompt)
- The Shift+Tab cycle toggles this flag + renders a status-line indicator
- **No persistence, no cross-session carry-over** — purely TUI-local, gone on exit (similar to grok's yolo_mode, but simpler)
- **Applicability (diagnostic revision)**: effective only for sessions with the `approval='ask'` policy — under the `'never'` policy (the danger-full-access preset), `decide()` deterministically returns 'rejected' before the waterfall dispatch, the answerer is never invoked, and the TUI short-circuit has no effect; also `handleApprovalRequest` is not the only consumer (apiproxy.ts:919 also registers an answerer forwarding approvals to remote RPC) — local TUI runs are unaffected, but be aware

**Option B (backend, deferred)**: a new cordis service `ctx.approvalPolicy`
- A session-level policy, persisted to settings
- The approval answerer reads this service to decide whether to short-circuit
- Heavier, but persists across sessions

**The first batch uses Option A** — pure TUI, zero backend changes, immediately usable.

### Files to change

1. **Change `packages/tui/tui/src/ui/app.ts`** (core):
   - Add `private alwaysApprove = false`
   - At the top of `handleKey`, add `if (key.name === 'shift_tab') { this.cycleMode(); return }`
   - Add `private cycleMode(): void`:
     ```
     当前 plan 关 + alwaysApprove 关 → plan 开（Normal→Plan）
     当前 plan 开 → plan 关 + alwaysApprove 开（Plan→Always-Approve）
     当前 alwaysApprove 开 → alwaysApprove 关（Always-Approve→Normal）
     ```
   - Add the short-circuit to `handleApprovalRequest`: `if (this.alwaysApprove) return Promise.resolve('allowed-once')`
   - Add an always-approve indicator to the `renderLive` status line (e.g. an `[auto]` badge, aligned with `[plan]`)

2. **Change `packages/tui/tui/src/statusline.ts`**:
   - `formatStatusLine` supports the `[auto]` badge (while always-approve is active)

3. **Change `packages/tui/tui/src/engine/input-handler.ts`** (optional, kitty protocol compatibility):
   - Add `\x1B[9;2u` (kitty Tab+SHIFT) → `'shift_tab'` to `resolveEscapeSequence`
   - Matches grok's `shift_tab_keys()` triple-encoding coverage

4. **New `tests/mode-cycle.spec.ts`**:
   - Three-state cycle transitions, always-approve short-circuiting approvals, plan toggling calling `ctx.planMode.set`

### Scope limits
- always-approve is not persisted (Option A; gone on exit)
- No auto mode (grok's classifier-based approval needs a classifier model; deferred)
- No acceptEdits (Claude Code's "auto-approve edits only" refinement; deferred)

---

## Commit sequence (revised: four items)

Based on the re-check, the first batch expands to four items (the original plan + fork pair, plus the rewind port and Shift+Tab):

1. `feat(tui): fix question answer shape + plan-review request-changes feedback` (item 1: shape-break fix + feedback path, ~65 lines)
2. `feat(tui): /fork optional directive as first prompt` (item 2, ~35 lines)
3. `feat(tui): shift-tab mode cycling normal/plan/always-approve` (item 4, ~80 lines)
4. `feat(session+tui): rewind via ported FileHistory + overlay` (item 3, cross-package — the largest)

Item 3 is the largest (porting file-history ≈314 lines + rewind rendering ≈184 lines + SessionStore.truncate + overlay wiring), but the Tianshu source turns it from "unknown design" into "building to a blueprint".

## Cross-item discipline

- **Items 1/2/4 do not touch the backend**: zero changes to plan-mode / session / agent
- **Item 3 touches the backend**: adds `SessionStore.truncate` + persistence `deleteFrom` (the only change to the append-only log; proceed carefully + test)
- **Tests**: an independent spec per item, `pnpm vitest run packages/tui/tui/tests/` (**measured baseline: 1135 tests / 3 failed**, at the 2026-08-11 diagnosis point: app.spec ×2 status-line/tool-card rendering, term-width isCjkLocale — the baseline must be fixed before implementation, otherwise regressions cannot be distinguished)
- **Types**: reuse the existing `QuestionRequestInput` / `SessionStore.fork` / `ctx.planMode.set` signatures
- **Tianshu port**: file-history.ts / format/rewind.ts must carry Apache-2.0 source attribution (SOURCE-MAP.md — existence unverified; confirm the opencode-tui root LICENSE when porting)

---

## Diagnostic revision record (2026-08-11, /scout sky-survey scout swarm + independent verification + real test run)

This revision is based on a lightweight parallel read-only diagnosis (4 code_scout lanes + independent verification by the main agent). All scout findings are hypotheses pending verification; every correction below was independently re-checked via read_file/grep.

### Corrections

| # | Original claim | Measured fact | Evidence |
|---|---|---|---|
| 1 | item 1 "the end-to-end chain already exists" | **Shape break**: TUI settle submits `{questionId,value}`/`{cancelled}`, user-interaction `ask()` passes straight through to the provider, plan-mode reads `answer.answers` → TypeError under real assembly | app.ts:1215 / user-interaction/src/index.ts:153 / plan-mode/src/index.ts:365-368 |
| 2 | rewind tool detection "str_replace_editor(write/replace) + write_file/edit_file" | registered names = `str_replace_editor` (enum: view/create/str_replace/insert) + `write` + `edit`; check 3 names | tool-str-replace-editor/src/index.ts:423-430 / tool-fs/src/write.ts:70 / tool-fs/src/edit.ts:84 |
| 3 | "reuse session-persistence's directory convention" for backups | **the backups convention has zero presence**; needs adding | packages/session grep 'backups' = 0 matches |
| 4 | "960+ all green" | **1135 tests / 3 failed** (app.spec ×2 + term-width ×1, 60 files; 1132 passed) | `pnpm vitest run packages/tui/tui/tests/` measured 2026-08-11 (vitest reports 3 failed) |
| 5 | app.ts:644 forkSession / :1027 handleApprovalRequest / :849 permission / :1140-1152 key handling | actually 671-680 / 1060 / 882 / 1207-1219 (registry /fork 262-267; question-panel plan-review L111) | grep + read_file (2026-08-11 working tree) |
| 6 | the permission package is under core/ | actually `packages/interaction/permission` | glob packages/**/permission |
| 7 | Tianshu file-history.ts "only node:fs/crypto" | getDiffStats depends on the cpu-pool/cpu-tasks worker pool; the core rewind path is ~200 self-contained lines | the Tianshu file-history.ts import section |
| 8 | Option A always-approve unrestricted | effective only for `approval='ask'` sessions; the 'never' policy rejects deterministically before the decide() dispatch; apiproxy.ts:919 also listens to approval/request | user-approval decide() / api-proxy.ts:919 |
| 9 | checkpoint-policy at 84 lines, "the only thing it does is flush" | ≈78 lines; the tools/execute branch additionally checks aborted | session-checkpoint-policy/src/index.ts:66-75 |

### Claims confirmed solid (no change needed)

- plan-mode exit_plan_mode L323/L331-345/L368, Keep planning value='Keep planning'; the `fork(source, boundary?, childSessionId?)` signature; input-handler shift_tab L75/L174/L481-482; statusline [plan] L215-218; Session is append-only with no truncate; DSH's reserved rewind traces (ConversationContextOriginKind / history-fold.ts:56 / compact checkpoint.ts:19 / live-engine.ts:668); Tianshu tool-pipeline.ts:1290 trackEdit, context.ts:355-368 full reset, rewind.ts ≈184 lines, persist ≈42 lines.

### Unverified items (fill in before implementation)

- `controls.followup()` behavior (item 2's directive depends on it)
- the actual shape of the apiproxy approval forwarding (item 4 boundary)
- existence of the Tianshu opencode-tui root LICENSE / SOURCE-MAP.md
- where `deleteFrom` lands in the jsonl/sqlite implementation packages, and the transaction boundary
- specific attribution of the 3 test failures (app.spec ×2 likely related to uncommitted changes; term-width is a semantic split between Intl fallback and env priority, reliably triggered on a Chinese-locale macOS system)

---

## Follow-ups (independent projects, not in this batch)

| Item | Nature | Suggested doc |
|---|---|---|
| rewind summarize modes (LLM summarizer) | compact package + LLM calls | with the compaction re-attachment batch |
| plan body markdown rendering + full-screen viewer | TUI rendering layer | with the overlay framework batch |
| /fork boundary picker (fork from a given turn) | TUI interaction layer | with the history-search batch |
| always-approve persistence (Option B) | new cordis service + settings | independent investigation |
| post-compaction skill/instruction re-attachment | compact package changes | independent investigation |
| auto mode (classifier-based approval) | classifier model + approval pipeline | independent investigation |

## Reference file index

### Tianshu TUI (rewind port source)

Path: `~/checkouts/opencode-tui`

| Feature | Tianshu file | Notes |
|---|---|---|
| rewind data layer | `src/agent/file-history.ts` (314 lines) | the `FileHistory` class: `trackEdit`/`rewindToBoundary`/`collectPostBoundaryEditIds`, self-contained (node:fs/crypto) |
| rewind persistence | `src/agent/file-history-persist.ts` (43 lines) | `file-history.json` read/write (1MB/50-entry caps) |
| rewind rendering | `src/tui/format/rewind.ts` (183 lines) | two-phase panel (message list → rollback granularity), 5 `RewindMode`s |
| rewind derived-state reset | `src/agent/context.ts:355` | `rewindToMessages`: truncation + full token/turn/cache reset |
| rewind hook | `src/agent/tool-pipeline.ts:1290` | `trackEdit` before `write_file`/`edit_file` inside `tools/execute` |
| rewind tests | `src/tui/__tests__/format-rewind.test.ts` (107 lines) | rendering tests, portable as-is |
| | `src/agent/__tests__/file-history.test.ts` (155 lines) | data-layer tests, portable as-is |
| undo tool | `src/tools/undo.ts` (82 lines) | agent-invoked single-step undo (independent of the overlay rewind) |

### grok-build (design reference)

Path: `~/checkouts/grok-build` (version `b13fa52`, pulled 2026-08-11)

| Feature | grok file | Notes |
|---|---|---|
| rewind | `crates/codegen/xai-grok-pager/src/app/dispatch/rewind.rs` | rewind dispatch (~600 lines), the `RewindResponse` structure |
| | `…/src/views/rewind.rs` | rewind UI (the `RewindPhase` state machine) |
| plan approval | `…/src/views/plan_approval_view.rs` | plan approval UI (Approve/Request-changes/Quit) |
| fork | `…/src/slash/commands/fork.rs` | `/fork` argument parsing |
| | `…/src/app/dispatch/session/fork.rs` | fork dispatch |
| Shift+Tab modes | `…/src/app/dispatch/modes.rs:631` | `dispatch_cycle_mode_inner`: the Normal→Plan→Always-Approve cycle |
| | `…/src/app/actions.rs:440` | the `CycleMode` action |
| | `…/src/input/key.rs:313` | `shift_tab_keys()` triple encodings (BackTab / BackTab+SHIFT / Tab+SHIFT) |
