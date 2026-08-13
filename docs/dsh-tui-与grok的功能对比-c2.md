# DSH TUI vs grok-build feature comparison — C2

English | [中文](dsh-tui-与grok的功能对比-c2.zh.md)

> **Tag**: C2 (first in the grok-build TUI benchmarking series; for C1 see [dsh-tui-与claude的对比-c1.md](dsh-tui-与claude的对比-c1.md)) **Date**: 2026-08-11 **Purpose**: use the grok-build TUI as the reference to pin down fix plans for five DSH TUI gaps

## 1. The local grok-build repository

**Path**: `~/checkouts/grok-build` **Remote**: https://github.com/xai-org/grok-build **Local version**: `b13fa52 Synced from monorepo` (pulled 2026-08-11)

To update:
```sh
cd ~/checkouts/grok-build
git pull --ff-only origin main
```

## 2. Directory index of grok-build TUI-related files

grok-build is a Rust monorepo. The TUI is concentrated in a single crate, `xai-grok-pager` (ratatui + crossterm, talking to the agent over ACP).

**Crate root**: `crates/codegen/xai-grok-pager/`

### Related-file inventory (grouped by this round's five gaps)

| Gap | grok file (absolute path) | Description |
|---|---|---|
| **Approval diff** | `…/xai-grok-pager/src/diff.rs` | Diff engine: the `similar` crate (Myers LCS) + ±3 context + line-number gutters on both sides |
| | `…/xai-grok-pager/src/app/acp_handler/permissions.rs` | Approval-request enqueueing + title/description construction (reads `raw_input` for file_path/command) |
| | `…/xai-grok-pager/src/views/permission_view.rs` | Approval-overlay rendering (**note: grok puts no diff in the approval modal**) |
| | `…/xai-grok-pager/src/scrollback/blocks/tool/edit.rs` | Diff-block rendering for the edit tool (after execution) + full-screen viewer |
| | `…/xai-grok-pager/src/app/dispatch/permissions.rs` | Approval-reply dispatch (select/followup/cancel) |
| **History search** | `…/xai-grok-pager/src/scrollback/search.rs` | Scrollback search: `ScrollbackSearchIndex` + `SearchDaemon` (background thread) + query coalescing |
| | `…/xai-grok-pager/src/scrollback/state/mod.rs` | `ScrollbackState`: `entries: IndexMap<EntryId, ScrollbackEntry>` + the `content_generation` cache key |
| | `…/xai-grok-pager/src/scrollback/state/nav.rs` | Paging navigation: `page_up/down`, `half_page_up/down`, `goto_top/bottom`, `next_turn/prev_turn` |
| | `…/xai-grok-pager/src/search/matcher.rs` | `TextMatcher`: smart-case + Substring/Regex |
| | `…/xai-grok-pager/src/views/history_search.rs` | Prompt history search (up-arrow recall + `/history`) |
| | `…/xai-grok-pager/src/app/agent_view/panes.rs` | Scrollback-search wiring (`open_scrollback_search`, `handle_scrollback_search_key`) |
| **Output folding** | `…/xai-grok-pager/src/scrollback/types.rs` | The three-state `DisplayMode { Collapsed, Truncated, Expanded }` enum |
| | `…/xai-grok-pager/src/scrollback/entry.rs` | `ScrollbackEntry` + `display_mode`/`display_mode_pinned` + `toggle_fold` |
| | `…/xai-grok-pager/src/scrollback/blocks/tool/read.rs` | read tool: `FIRST_LINES=5, LAST_LINES=3`; the truncated state shows the first 5 and last 3 lines |
| | `…/xai-grok-pager/src/scrollback/blocks/tool/execute.rs` | bash tool: truncation config + user commands expanded in full by default |
| | `…/xai-grok-pager/src/scrollback/state/selection.rs` | `toggle_fold_selected` + scroll-anchor capture/restore |
| **Keyboard shortcuts** | `…/xai-grok-pager/src/actions/defaults.rs` | `default_actions()`: the single source of every key-binding declaration (**hardcoded, not configurable**) |
| | `…/xai-grok-pager/src/actions/mod.rs` | `ActionDef { id, default_key, alt_keys, context, category }` + 3-layer dispatch (pane→agent→global) |
| | `…/xai-grok-pager/src/input/key.rs` | `KeyShortcut { code, modifiers }` + the `key!()` macro |
| **Model switching** | `…/xai-grok-pager/src/slash/commands/model.rs` | `/model <name> [effort]`: bare name → `SetDefaultModel`; name+effort → `SwitchModel` |
| | `…/xai-grok-pager/src/app/dispatch/settings/setters.rs` | `set_default_model`: optimistic update + `Effect::SwitchModel` (ACP hot swap) |
| | `…/xai-grok-pager/src/app/effects/mod.rs` | `Effect::SwitchModel`: sends `acp::SetSessionModelRequest` to the running agent |
| | `…/xai-grok-pager/src/app/dispatch/session/lifecycle.rs` | Switch-completion handling: on success render to the transcript / on agent-type mismatch open a modal / on failure roll back |
| **Settings panel** | `…/xai-grok-pager/src/settings/defs.rs` | `default_settings()`: ~60 `SettingMeta` entries, grouped by category |
| | `…/xai-grok-pager/src/settings/registry.rs` | `SettingKind { Bool, String, Enum, Int, DynamicEnum, Group }` |
| | `…/xai-grok-pager/src/views/settings_modal/` | Settings modal UI (four states: browse/filter/pick/edit) |

### Supporting crates

| crate | Path | Description |
|---|---|---|
| `xai-grok-pager-render` | `crates/codegen/xai-grok-pager-render/` | Theme/appearance primitives; `AppearanceConfig` truncation defaults (`first_lines:2, last_lines:3`) |
| `xai-grok-tools` | `crates/codegen/xai-grok-tools/` | Tool type definitions (`SearchReplaceEditDetail`, `BashToolInput`) |
| `xai-grok-shell` | `crates/codegen/xai-grok-shell/` | Configuration (`UiConfig`), agent runtime, bash highlighting |
| `xai-grok-config` | `crates/codegen/xai-grok-config/` | Configuration schema |
| `xai-ratatui-inline` | `crates/codegen/xai-ratatui-inline/` | Inline rendering widget |
| `xai-ratatui-textarea` | `crates/codegen/xai-ratatui-textarea/` | Prompt text-editing widget |

## 3. Feature comparison (DSH today vs grok-build)

### Approval diff preview

| Dimension | grok-build | DSH today |
|---|---|---|
| Diff inside the approval modal? | **No** (the diff renders only after execution) | No diff; a bare `⚠ 允许执行 X？[y/N]` |
| Diff algorithm | Real Myers LCS (the `similar` crate) | None (`format/diff.ts` only handles unified-diff text; it cannot produce a diff from two strings) |
| ±context trimming | ±3 context + line-number gutters on both sides | None |
| Data path | Reads arguments from `req.tool_call.fields.raw_input` | `PendingApprovalRequest` drops `callId` (present at runtime, never wired into the type) |

**Key difference**: grok's stance is that approval needs no diff — reviewing it after execution is enough. DSH's user pain point is exactly the opposite: **blind approval is a trust break**. We go the other way and put an inline diff above the approval prompt.

### History search / scrollback pager

| Dimension | grok-build | DSH today |
|---|---|---|
| Conversation search | Yes: background thread + query coalescing + smart-case + regex | `scrollback-transcript.ts` has pure search functions (`searchTranscript`/`findNextMatch`), **but nothing wires them up** |
| Paging | Yes: PageUp/Down, half-page Ctrl-U/D, goto top/bottom, jumps between turns | None (relies on the terminal's native scrollback) |
| Prompt history | Yes: up-arrow recall + `/history` + nucleo fuzzy matching | Yes: up/down history on the input line (basic) |
| Search thread | Background `SearchDaemon` (for million-line multi-agent workloads) | Not needed (DSH's single-session scale is small; synchronous main-thread search suffices) |

**Key difference**: grok's background index serves multi-agent, million-line workloads. DSH's single-session scale is far smaller, so **synchronous search on the main thread is enough** — the parsing layer is ready; only the overlay container + key routing are missing.

### Output folding

| Dimension | grok-build | DSH today |
|---|---|---|
| Fold states | Three-state `Collapsed|Truncated|Expanded` | Two-state collapse/expand (built into tool-card) |
| read tool | `FIRST_LINES=5, LAST_LINES=3`; the truncated state shows the first 5 and last 3 lines | **None** (large read_file output floods the screen) |
| bash tool | `first_lines:2, last_lines:3`; user commands expanded in full by default | Has `collapsed-bash.ts` (group folding across multiple calls, not single-output truncation) |
| Search/list | Per-tool configuration | None |
| Expand interaction | Toggle fold + anchor preservation | Yes (two-state tool-card toggle) |

**Key difference**: DSH has group folding (merging multiple tool calls) but lacks **truncation of a single tool's large output**. A read_file that returns 500 lines floods the screen outright.

### Keyboard shortcuts

| Dimension | grok-build | DSH today |
|---|---|---|
| Configurable? | **Not configurable** (the docs state it explicitly: "Bindings are built in and cannot currently be remapped") | Not configurable (hardcoded switch statements) |
| Architecture | An `ActionDef` registry feeds all three consumers (shortcuts-bar / command-palette / key-dispatch) | switch statements scattered across input-handler / app.ts |
| vim mode | `[ui].vim_mode` toggle (j/k/h/l scrollback navigation) | vim state machine built into the input line (normal/insert/visual) |

**Key finding**: grok itself also hardcodes bindings with no configurability, but it uses an `ActionDef` registry to eliminate the scattered switch statements — an architectural improvement, not added functionality. **Not in the first batch** (the user confirmed its ROI trails the other four items).

### Model switching

| Dimension | grok-build | DSH today |
|---|---|---|
| Switches the current session? | Yes (hot swap via ACP `SetSessionModelRequest`) | **No** (only switches the default via `saveSelection`, affecting new sessions) |
| Hot-swap timing | `model_switch_pending` holds the queue; takes effect once the current request completes | No hot-swap mechanism |
| agent-type mismatch | Opens a modal asking whether to start a new session | Does not distinguish agent types |
| reasoning effort | `/model <name> <effort>` accepts an effort | No effort hot swap |

**Key difference**: grok hot-swaps through the ACP protocol. DSH has no ACP, but `ModelSelectionRef` (model-selection.ts:20) is a **mutable object** — hold the ref, mutate `ref.current`, and the change takes effect automatically on the next agent step (prompt assembly). **Pure assembly layer; agent-loop untouched**.

## 4. Fix plans for the five gaps

### Item 1: approval diff preview (inline above y/N) ✅ user confirmed inline — **✅ done (2026-08-11)**

**Goal**: when approving `edit`/`write_file`, render an old/new diff block of ≤12 lines above `⚠ 允许执行 X？[y/N]`.

**Design decisions**:
- Invert grok's approach (grok puts no diff in the approval modal), because DSH's pain point is blind approval
- Use a real LCS diff algorithm (matching grok's Myers); in TS, use `diffLines` from the `diff` npm package
- Data path: add `callId?` to `PendingApprovalRequest` → transcript `view.tools.find(t => t.callId === callId)` → `parseToolArguments` → yields old/new
- Keys are locked during approval (only y/N/Esc/Ctrl+C) → the diff must be **fully visible without paging**, hard-capped at 12 lines

**Files changed**:
1. Add the `diff` npm package as a dependency
2. Create `src/format/permission-diff.ts` (~140 lines)
   - `formatPermissionDiff(input): string[] | null`
   - edit class → `formatEditDiff`: `diffLines` → Change[] → coloring (+green/-red/=dim) → ±3 context → gutters on both sides → fold at 12 lines
   - write class → `formatWritePreview`: path + first 4 lines
   - non-edit tools → null
3. Modify `src/ui/app.ts`: add `callId?` to `PendingApprovalRequest`; add the diff lookup to the approval line block in `renderLive`
4. Create `tests/permission-diff.spec.ts`

### Item 2: history search overlay ✅ user confirmed needed — **✅ done (2026-08-11)**

**Goal**: `/search` or `Ctrl+F` → full-screen alt-screen overlay, smart-case search over the conversation history, n/N to jump.

**Design decisions**:
- **No Worker introduced** (DSH's single-session scale is small enough; grok's background index serves million-line multi-agent workloads)
- Reuse the pure functions already in `scrollback-transcript.ts` (`searchTranscript`/`findNextMatch`/`estimateMessageRows`)
- The overlay uses the existing `OverlayController` (already used for command-palette/keymap)
- smart-case: a query containing uppercase → case-sensitive, otherwise insensitive (grok's one-line rule)

**Files changed**:
1. Create `src/format/history-search-overlay.ts` (~120 lines) implementing `OverlayRenderer`
   - State: query / matches / current / messages
   - `render`: search bar (query + match count N/M) + transcript scrolling (current match highlighted and centered) + key hints
2. Modify `src/ui/app.ts`: register the overlay + key routing (printable → query; Enter → search; n/N → jump; Esc → exit)
3. Create `tests/history-search-overlay.spec.ts`

**Scope limits**: synchronous search; no regex (smart-case substring first)

### Item 3: read/search tool output folding — **⏸️ no implementation needed (confirmed by investigation, 2026-08-11)**

**Investigation conclusion**: DSH already covers this gap — `read_file` (the read family) truncates by default to the first 3 lines + a marker + the last 5 (tool-card.ts getDefaultMaxLines read=8 lines), `grep/glob` (the find family) to 6 lines; the "… +N 行 · ctrl+o 展开" marker in `truncation-marker.ts` and the `isToolCardTruncated` expansion check are both in place, and tool-card.spec.ts already tests the head/tail preview. The C2 draft's "large read_file output floods the screen" does not match today's behavior (the existing 8-line threshold is more aggressive than the draft's proposed 50). Per the horizontal-reuse discipline, this is not reimplemented.

**Design decisions**:
- DSH already has two-state folding (built into tool-card); do not introduce grok's three-state enum
- Replicate the `LiveRegionLine` return shape of `collapsed-bash.ts` (zero adaptation in the assembly layer)
- Per-tool thresholds: read_file >50 lines, grep/glob >20 entries

**Files changed**:
1. Create `src/format/collapsed-read-search.ts` (~150 lines)
   - `shouldCollapseReadSearch(toolName, outputLength): boolean`
   - `formatCollapsedReadSearch(opts): string[]` (first N + "… +X 行" + last N)
   - `READ_THRESHOLDS: Record<string, { lines, first, last }>`
2. Modify `src/format/tool-card.ts`: when the result region hits the threshold and is collapsed → use `formatCollapsedReadSearch`
3. Create `tests/collapsed-read-search.spec.ts`

### Item 4: /model hot swap for the current session — **✅ done (2026-08-11)**

**Goal**: `/model <provider/model>` hot-swaps the model of the session currently running.

**Design decisions**:
- DSH has no ACP, but `ModelSelectionRef` is a mutable object — hold the ref and mutate `ref.current`
- `installModelSelection` reads `selection.current` on every prompt assembly → the change takes effect automatically on the next agent step
- Current problem: app.ts:615/652 pass the literal `{ current: selection, assembled: undefined }` and never hold the ref

**Files changed**:
1. Modify `src/ui/app.ts`:
   - Add `private modelRef: ModelSelectionRef | null = null`
   - newSession/switchSession: replace the literal with a held object `this.modelRef = { current: selection, assembled: undefined }`
   - Add `switchLiveModel(selection)`: `if (this.modelRef) this.modelRef.current = selection`
2. Modify `src/commands/registry.ts`: the `/model` handler additionally calls the injected `switchLiveModel` callback
3. Modify `tests/commands.spec.ts`

**Scope limits**: the hot swap takes effect on the next agent step (no immediate interruption); no effort hot swap and no agent-type mismatch modal

### Item 5: configurable keyboard shortcuts — moved out of this batch ❌ user confirmed not doing it

grok-build itself hardcodes bindings with no configurability. The ROI trails the other four items. Left for later.

## 5. Commit sequence (as executed, 2026-08-11)

1. `15173d4` `feat(tui): C2-1 approval diff preview — inline unified diff above y/N via callId` (item 1)
2. `0645886` `feat(tui): C2-4 /model hot-swap current session via ModelSelectionRef` (item 4)
3. `ec724ac` `feat(tui): C2-2 history search overlay — Ctrl+F smart-case search with n/N jump` (item 2)
4. Item 3: investigation confirmed the existing tool-card truncation covers it; not implemented (see above)
5. Item 5: the user confirmed it is not being done

**Acceptance status (2026-08-11)**: user-level behavioral acceptance of the three implemented items is **blocked** — the execution environment has no interactive TTY (raw mode unavailable, byte injection unreliable). Real-assembly probe evidence is in hand: TUI profile startup ✓, resumed-session list and fork lineage rendering ✓, Ctrl+F alt-screen enter/exit sequences (?1049h/?1049l) ✓. Feature-level verification rests on the package tests (RED→GREEN); full user acceptance on the real assembly awaits a terminal environment.

## 6. Cross-item discipline (execution results)

- **SOURCE-MAP.md**: not updated — permission-diff.ts is an original module (not a Tianshu port) whose file header declares the grok reference source; it is outside SOURCE-MAP's mapping scope
- **Dependencies**: only `diff@^9.0.0` added (item 1; ships its own types); items 2/4 add zero new dependencies ✅
- **Tests**: 7 cases in the item 1 spec + 6 for item 4 + 11 for item 2, all RED→GREEN ✅
- **Coverage back-fill (2026-08-11, `de64c35`)**: permission-diff.spec grew to 16 cases (every write_file/edit_file branch, missing-argument combinations, valid JSON that is not an object, missing muted, unknown tools); history-search-overlay.spec grew to 19 cases (clear/onDeactivate/empty text/explicit theme/no-match render/small heights/empty message sets). Final numbers (vitest v8): permission-diff **statements 100% + branches 100%** (up from 68.75%/52.08%); history-search-overlay **statements 100%** (up from 88.13%) with branches at 96.9% — 1 implicit branch unhit, every entry in that file's branchMap lacks a loc (v8+esbuild loses sourcemap attribution for TS classes), and 3 test scenarios plus an if-break rewrite failed to pin it down, so it is recorded as a known tooling limitation (the perFile branches gate was already red from parallel sessions' uncommitted files, so this adds no new blocker)
- **Types**: per-item scratch checks report 0 errors (a full tsc -b is blocked by parallel-session contention and pre-existing errors)

## 7. Summary of grok-design adoption decisions

| grok design | Adopted by DSH? | Rationale |
|---|---|---|
| No diff in the approval modal | ❌ Inverted | DSH's pain point is blind approval; an inline diff builds trust |
| Real Myers LCS diff | ✅ Adopted | Matched in TS with the `diff` npm package |
| Background-thread search | ❌ Simplified | DSH's single-session scale is small; synchronous main-thread search suffices |
| smart-case matching | ✅ Adopted | A one-line rule and the behavior users expect |
| Three-state fold enum | ❌ Simplified | Two states are enough for DSH and match the existing tool-card |
| Per-tool first/last thresholds | ⏸️ Already covered | Investigation confirmed DSH's existing tool-card truncation (read 8 lines / find 6 lines) is more aggressive than grok's per-tool configuration; no new implementation needed |
| `ActionDef` registry | ⏸️ Left for later | The first batch does not touch keyboard shortcuts |
| ACP model hot swap | ❌ Unavailable | DSH has no ACP, but the mutable `ModelSelectionRef` object achieves the same effect |
| agent-type mismatch modal | ❌ Not adopted | DSH does not distinguish agent types |
| Settings-panel registry | ⏸️ Left for later | DSH already has the /config panel (T3.2); shallow but sufficient |
