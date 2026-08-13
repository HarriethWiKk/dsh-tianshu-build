# DSH TUI C4 Split Retrospective: Findings and Workflow Improvement Proposals

English | [中文](dsh-复盘-c4拆分-问题与优化.zh.md)

> **Date**: 2026-08-11 **Background**: C4 (the app.ts monolith split, 1727 lines → controller + render pure functions) completed over 4 galaxy-cluster rounds, with 5 commits (a603ada/6cd01db/a8ff778/abccfde/a9d80d3) and 3 supervisor full-suite re-run verifications. **Purpose**: analysis material for workflow and algorithm iteration — every finding carries phenomenon, evidence, root cause, and proposal, and can be cited independently.

---

## 1. Execution overview (factual baseline)

| Round | Dispatched dimensions | Result | Key events |
|---|---|---|---|
| 1 | implement (Tianfu) + audit (Yaoguang) + review (Tianquan) | 2/3 passed | Wave 1 done (15 new tests green); Wave 2 half-done in a red state (live-panels unimplemented); Tianfu claimed "app.spec 138/138" |
| 2 | wave2-3 (Tianfu) + tab-triage (Yaoguang) | 2/3 passed | loader-composition attributed to a scope event-dispatch problem (fixed externally); the two Tab failures attributed to a pre-existing flake; Wave 2 not started |
| 3 | wave2 (Tianfu) + verify (Yaoguang) | 0/1 passed | Wave 2 implementation done (claimed 1247/1248); the verify dimension was stripped for file overlap; **the supervisor's re-run measured 73 failed** (dispose leak regression) |
| 4 | fix-regression (Tianliang) + leak-verify (Yaoguang) | 3/3 passed | the leak had already been fixed concurrently by an external session (intermediate state); the orphan controllers' extraction deleted (unequal-semantics evidence); the taskDone/taskSurface release gap confirmed |
| 5 | disposer-gap (Tianliang) + ledger-audit (Yaoguang) | 3/3 passed | taskDone/taskSurface release fixed (RED→GREEN); the ledger blind spot confirmed |

**Final state**: `packages/tui/tui/tests/` 66/66 files, 1223 passed, all green (+2 todo); `tsc -b` exit 0; lefthook all passed. The app.spec.ts black-box tests passed every Wave with **0 changes** (the regression checklist's 10 anchors preserved).

---

## 2. Tianshu workflow problems

### W1. Worker verification claims are untrustworthy (severity: high)

- **Phenomenon**: in round 3 Tianfu claimed "app.spec 138/138, full suite 1247/1248 with only the Tab flake"; the supervisor's re-run measured **73 failed** (dispose leak regression, 1 failed file). In round 1 Tianfu's claim that Wave 1 was all green held; in round 4 Tianliang's all-green claim held — **verification reliability correlates strongly with the star domain**.
- **Evidence**: the round-3 supervisor re-run printed "Test Files 1 failed | 66 passed; Tests 73 failed | 1175 passed" vs the worker's claimed "1247/1248".
- **Root cause**: a worker's "verified" carries no mandatory evidence format; there is no automatic cross-check between claim and measurement; the supervisor's re-run is the only backstop, but it happens at the aggregation stage (late and expensive).
- **Proposals**:
  1. Turn the galaxy write-dimension delivery gate into an **evidence package**: it must attach `命令 + exit code + 关键输出行`; a "pass" without an evidence package counts as unverified.
  2. The supervisor's aggregation stage force-re-runs the full suite (done this time, and it caught the regression — keep it and document it).
  3. For star domains with poor verification discipline (Tianfu slipped twice this time), default write dimensions to Tianliang or add a Yaoguang side-channel verification.

### W2. Dimension slicing mismatched to refactoring cadence (severity: medium)

- **Phenomenon**: three waves took four rounds. galaxy's hard constraint — writable-dimension files must not overlap — collided with app.ts as the single-file main battlefield (all three waves touch it), so everything squeezed into one write dimension; each round the worker actually finished only 1-1.5 waves before exhausting the round.
- **Evidence**: the round-vs-progress mapping (see the execution overview table); in round 2 Tianfu spent its budget on attribution (the already-attributed loader-composition) instead of Wave 2.
- **Root cause**: the single write dimension's objective was too large (Wave 1+2+3 = 14 tasks); workers have no wave-switching discipline (finishing one wave does not automatically start the next); attribution work and implementation work were mixed into one work order.
- **Proposals**:
  1. **Single-file-battlefield refactors = one dimension, one wave per round**: each round galaxy dispatches only one write dimension doing one wave plus parallel read-only dimensions, avoiding mid-course re-dispatch.
  2. Already-attributed problems get no further attribution dimension (the supervisor writes the attribution conclusion directly into the write dimension's constraints).

### W3. Read-only dimensions whose files overlap a write dimension are silently stripped (severity: medium)

- **Phenomenon**: in round 3 the verify dimension's files included app.ts (overlapping the write dimension); after galaxy stripped it, the whole dimension was skipped ("the write dimension's files were all taken by other dimensions; dispatch skipped") — one dimension's budget wasted, verification missing.
- **Root cause**: the read-only dimension was given a files parameter; galaxy's stripping granularity for overlap is "skip the whole dimension" rather than "strip only the overlapping files".
- **Proposal**: the supervisor self-checks before dispatch — read-only dimensions leave files empty and describe their focus in the objective; or galaxy downgrades read-only handling to "strip the overlapping files, keep the rest".

### W4. Gap in the RED trust chain (severity: medium)

- **Phenomenon**: the RED for taskDone/taskSurface was the worker's claimed "before the fix: 1 failed | 138 passed"; the supervisor never reproduced it independently (the defect was already landed; rolling back is destructive and app.ts ownership is shared). The system reminder (evidence-obligation) caught this gap; it was finally downgraded and annotated as "static causal proof + worker's account".
- **Root cause**: the collection point for RED evidence is irreversibly gone once the fix lands; the supervisor only sees claims at the aggregation stage.
- **Proposal**: **move RED collection earlier** — for fix-type work orders, the verification dimension independently reproduces RED before the fix and archives it (output/logs), then only confirms GREEN after the fix; the delivery gate checks "this work order has RED evidence".
- **Addendum**: this time RED provability finally closed via static causal reasoning on the diff (the abccfde test asserts "if the disposer is never called, the assertion must fail", which cannot be bypassed) — usable as the template for the downgrade path.

### W5. No fixed handling step for plan-anchor drift (severity: low)

- **Phenomenon**: the app.ts line numbers the C4 plan was based on (1727 lines/L333/L586/L1099 etc.) drifted after the plan was committed, due to an external session's rewind commit (f1ff5a0); execution relied on the executing worker re-checking and correcting them — pure self-discipline.
- **Proposal**: add a fixed step to the plan template — "**re-verify every anchor as the first step of execution**" (especially necessary for refactor-type plans); a plan's file:line references carry their "as-of point", and execution defers to reality.

---

## 3. dsh repository problems

### R1. Flakes pollute the "full-suite green" verdict signal (severity: high)

- **Phenomenon**: the Tab @-completion flake (app.spec "Tab with an @ token → completion applies" + completion.spec "a sole candidate does not enter cycle mode") fired three times across the four rounds; every full run required manual triage of "flake vs regression" — the largest source of regression-triage cost this time. In round 2 Yaoguang's three consecutive re-runs, all green, falsified the "regression" suspicion; in round 3 it went red again (triggered by parallel load).
- **Root cause** (attributed, landed in a9d80d3): the completion tests depend on real-filesystem glob + `git ls-files`; `tabComplete` does not forward timeoutMs, so when git ls-files exceeds 500ms it returns null → `out!.text` throws a TypeError; doubly environment-dependent on cwd + parallel load.
- **Proposals**:
  1. `tabComplete` forwards timeoutMs (or raise the default timeout) — a one-shot elimination.
  2. **A flake-annotation mechanism**: mark known-flaky cases + vitest retry (`retry: 2`) so the full run's red/green signal stays clean; standardize the flake determination (three consecutive re-runs) into the process.

### R2. Test-ledger blind spot: service-registered disposers are not covered (severity: high)

- **Phenomenon**: app.spec's listener ledger (SubscriptionRecord, L42-48) covers only `ctx.on` subscriptions; taskDone/taskSurface register through the tasks service (`onTaskDone`/`attachSurface`) and are **entirely outside ledger coverage** — nobody knows how long the leak existed; it was found by Yaoguang's static audit (round 4), not intercepted by tests.
- **Evidence**: the ledger-audit conclusion — "the ledger covers only ctx.on subscriptions; taskDone/taskSurface register through the tasks service and thus cannot be captured by the ledger"; the fix closed only after a standalone RED test was added (abccfde).
- **Root cause**: the ledger mechanism records by event name, and service-type disposers have no event name; the comment's promise ("taskSurface released on dispose") had no automated check.
- **Proposal**: **extend the ledger into a "disposer-lifecycle foundation"** — service-registered disposers also join the release assertion (record disposer calls in the mock service layer, and assert all released after dispose/detach); this class of leak moves from "found by static audit" to "intercepted automatically by tests".

### R3. Comment promises drift from the implementation (severity: medium)

- **Phenomenon**: an app.ts comment explicitly promises "taskSurface released on dispose, taskDone unloaded with the session" (L344-346) while the implementation is absent — the comment is a contract nobody checks.
- **Proposal**: put a "must have a release point" assertion for disposer fields into the test infrastructure (implement together with R2); or at minimum write such contracts into the corresponding describe's comment and pair them with tests.

### R4. The orphan controllers' "extracted but unwired" intermediate state (severity: medium)

- **Phenomenon**: StreamRenderController/ToolGroupController were extracted with tests but no production consumer, coexisting with duplicated isomorphic inline logic in app.ts (two spots: handleStreamEvent and the live tool card) — knip (a member of the hygiene script) did not catch it (most likely misjudged as used via the index.ts export chain). C4 closed it as "isomorphism comparison found unequal semantics → delete the extraction, keep the inline logic" (a8ff778, 4 files, 725 lines).
- **Root cause**: the extraction step lacked a "wire in or delete" closing discipline; static tools cannot distinguish "exported with no production consumer" from "exported for tests".
- **Proposals**:
  1. Add an "exported symbols must have a production consumer" check to the knip configuration (distinguishing test consumers).
  2. Document the refactoring discipline: **extraction means wiring in or deleting, no intermediate state left** (write it into the C5 batch discipline).

### R5. Toolchain boundaries: deliver_task does not support deleted files / the git tool blocked by historical owned files (severity: low)

- **Phenomenon**: when committing the orphan deletion, the deliver_task pathspec reported "did not match any files" (D status unsupported); the structured git tool's commit was blocked by a historical owned file (the deleted draft plan .rivet/plans/draft-1786435014028.md still in the owned set) — two tool-boundary workarounds (git add -u + bash commit).
- **Proposal**: deliver_task supports D-status files; clean up historical owned files after a plan submit succeeds.

---

## 4. Consolidated workflow/algorithm improvement proposals (sorted by ROI)

| # | Proposal | Owner | Cost | Payoff |
|---|---|---|---|---|
| 1 | galaxy write-dimension "evidence package" gate (command + exit code + output) | Tianshu workflow | zero (pure process) | eliminates "claimed green, actually red" (this run's largest loss source) |
| 2 | Flake fix: tabComplete forwards timeoutMs + known-flake annotation | dsh repository | small (single-point change) | eliminates the false-positive triage cost of every full run |
| 3 | Extend the ledger into a disposer-lifecycle foundation (covering service registration) | dsh repository | medium | leaks move from manual audit to automatic interception |
| 4 | Single-file-battlefield refactors = one dimension, one wave per round | Tianshu workflow | zero (pure process) | eliminates mid-course re-dispatch (four rounds → two) |
| 5 | Move RED collection earlier (the verification dimension reproduces it before the fix) | Tianshu workflow | zero (pure process) | closes the RED trust-chain gap |
| 6 | Fix an "anchor re-verification" step into the plan template | Tianshu workflow | zero (pure process) | eliminates execution-time anchor-drift rework |
| 7 | knip gains "exported symbols must have a production consumer" | dsh repository | small | orphans surface at merge time instead of being found manually during refactors |
| 8 | deliver_task supports D status + historical owned cleanup | dsh repository (tool side) | small | eliminates tool-boundary workarounds |

---

## 5. Appendix: evidence index

- Commits: a603ada (Wave 1) / 6cd01db (Wave 2+3 main body) / a8ff778 (orphan deletion) / abccfde (disposer fix) / a9d80d3 (flake-attribution doc)
- Test baseline: 66/66 files, 1223 passed (supervisor's real run, 2026-08-11 22:10); pre-Wave-2 baseline 1191 (1 failed)
- Intermediate-state events: at 21:21 the supervisor's re-run measured 73 failed (after Tianfu claimed 1247/1248) → at 21:40 Tianliang arrived to find it already green (concurrent fix by an external session) — shared-workspace concurrency is the strongest footnote to why the re-run was necessary
- memory: this retrospective has been distilled into 1 failure_pattern (worker claims require re-runs + the deliver_task D-status boundary), project-scoped
