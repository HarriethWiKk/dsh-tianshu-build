# DSH orchestration mechanisms vs Claude Code — C7

English | [中文](dsh-编排机制对标-claude-c7.zh.md)

> **Tag**: C7 (design-decision comparison of orchestration mechanisms — incremental from [C6](dsh-能力复盘与对标-c6.md))
>
> **Date**: 2026-08-16
>
> **Scope**: the design decisions behind Claude Code's orchestration mechanisms per its official documentation (code.claude.com/docs/en, current as of 2026-08), measured against DSH harness reality with `path:line` evidence. C6 inventories capabilities; this volume compares mechanism contracts — how context flows and which layer enforces each constraint. Every Claude Code claim carries an official URL; every DSH claim carries a code location.

## 1. Method and drift signals

Method: eight official Claude Code pages were fetched (sub-agents, hooks, hooks-guide, permissions, permission-modes, skills, memory, output-styles), corroborated by the tools page and issues in the official repository; the DSH side was spot-checked in code, and items already settled in C6 are inherited without re-adjudication.

Drift signals (comparisons must follow the current pages; earlier secondary sources are stale): the Task tool is now called Agent; custom slash commands have been merged into skills ("Custom commands have been merged into skills"); permission modes grew from 4 to 6; hook events grew from an early 9 to 31.

## 2. Seven-dimension comparison matrix

| Dimension | Claude Code design decision | DSH current state | Verdict |
|---|---|---|---|
| plan mode | A permission mode: edits hard-blocked, read-only exploration, research delegated to the Plan subagent, ExitPlanMode approval | Monotonic guard hard-blocks the mutation families at execution (C7-1 landed) + plan-review intent approval + persisted plan file | Aligned (constraint hardness); subagent roles tracked in C7-5 |
| subagents | User-authored markdown role definitions + description-driven auto-delegation + built-in read-only Explore/Plan + safety scan of returned reports | 7-provider execution surface (spawn/fork/inprocess/acp/claude-code/codex/dsh-sdk) + durable background children + cross-agent messaging | Mixed: execution surface ahead, role-definition surface missing |
| hooks | Declarative shell hooks: 31 events, 5 handler kinds, full JSON protocol (updatedInput/systemMessage/continue), layered settings merge with hot reload | CC-compatible subset: 7 events, command-only, protocol subset, process-level one-shot load | Behind (protocol and discovery coverage) |
| skills | Progressive disclosure + allowed-tools pre-approval + context:fork + re-attach after compaction | Progressive disclosure + multi-level discovery + `/name` gesture | Parity |
| slash commands | Merged into skills: a markdown command is equivalent to a SKILL.md, skill wins on name clash | 27 builtins + dual registries + skill `/name` gesture | Parity (CC's merger falsifies the standalone markdown-command gap) |
| memory | Four-level CLAUDE.md concatenation + @import 4 hops + auto memory on by default (index-capped loading) | Walk-up AGENTS.md layering + nested projection + auto memory (substring retrieval) | Parity, slightly ahead |
| permissions | 6 modes + deny→ask→allow rule layers + "always allow" persisted to settings.local.json + circuit breakers | OS-level sandbox tri-mode (Seatbelt/bwrap/Landlock) × one-shot approvals | Sandbox ahead, rule layer behind (deepens C6 H2) |

## 3. Per-dimension contract details

### 3.1 plan mode

Claude Code: plan is a permission mode, not a feature toggle (three entries: Shift+Tab cycle, `/plan` prompt prefix, `--permission-mode plan`); edits are blocked until approval (bypass sessions excepted); `useAutoModeDuringPlan` defaults on, routing shell commands through a classifier; the approval UI offers three choices (approve and switch to auto / approve with per-action review / keep planning); subagents have EnterPlanMode stripped by default. (https://code.claude.com/docs/en/permission-modes)

DSH: plan mode states it is "independent of sandbox mode and approval policy" (`packages/plan/plan-mode/src/index.ts:5-7`); approval flows through the `plan-review` intent (index.ts:331-347).

Gap (closed): Claude Code enforces read-only in the execution layer; DSH used to advise it in the prompt layer. **C7-1 has landed**: a monotonic `ctx.tools.guard` denies the mutation families at execution (write/edit/str_replace_editor mutating commands/git_commit/terminal_*; bash/pwsh stay allowed per CC semantics), and every `exit_plan_mode` call persists the plan under `$DSH_HOME/plans/…` with a log-only `plan/file` event (see the plan-mode README and Agent Note 2026-08-16-plan-mode-hard-readonly-and-plan-file).

### 3.2 subagents

Claude Code: a subagent is a user-authored markdown frontmatter role definition (`.claude/agents/*.md`), with `description` dedicated to driving automatic delegation; built-in Explore/Plan are read-only, fast, and cheap (skipping CLAUDE.md and git status) — the design intent is keeping exploration output out of the main context; returned reports are safety-scanned (text mimicking `<system-reminder>`/`Human:` is escaped); context isolation carries only the subagent's own system prompt + the delegation message + the CLAUDE.md hierarchy + a git-status snapshot; concurrency cap defaults to 20, nesting depth to 3. (https://code.claude.com/docs/en/sub-agents)

DSH: the execution surface is well ahead — 7 providers, fork inheriting a parent-turn prefix, durable background children, `send_message`/`interrupt_agent` cross-agent messaging, maxDepth recursion limits (`packages/subagent/tool-subagent/src/index.ts:246-383`, `packages/subagent/subagent/src/depth.ts`). But a DSH subagent is a provider type, not a role definition: there is no user-authored markdown agent surface, no description-driven delegation routing, and no built-in read-only explore role. No counterpart to the report safety scan was found; recorded as unverified (§7).

### 3.3 hooks (the widest divergence)

Claude Code: 31 event points in three cadences (per session / per turn / per tool call); five handler kinds (command/http/mcp_tool/prompt/agent); exit 2 is the only JSON-insurmountable block; `hookSpecificOutput.permissionDecision` (allow/deny/ask/defer) + `updatedInput` wholesale input replacement + `additionalContext` injection; `~/.claude/settings.json` and project `.claude/settings.json` merge across layers with a hot-reload watcher, and deny cannot be overridden by allow across layers; hooks can only tighten, never loosen — command hooks run with the user's full permissions and no sandbox, and no hook executes in an interactive session before workspace trust is accepted. (https://code.claude.com/docs/en/hooks)

DSH: the `hooks-claude` bridge runs unmodified Claude Code command hooks — 7 events (`packages/hooks/hooks-claude/src/config.ts:11-19`), exit-2 block, `additionalContext` injection, and Stop-deny-to-steer are all aligned; `systemMessage` now rides the durable `hook/result` event and is surfaced by clients (muted `[hook]` line in the TUI — first half of C7-2 landed); `updatedInput` remains parsed-but-not-honored, project-level per-session config discovery is a TODO, and configuration is read once at load time, process-wide.

Verdict: the backbone is deliberately homologous (CC-ecosystem compatibility is the right call); the gap is protocol coverage and config discovery.

### 3.4 skills

Claude Code: only name+description stay resident (truncated to 1,536 characters after merging when_to_use); `allowed-tools` is a pre-approval, not a whitelist (it waives prompts for that turn only); `context: fork` runs the skill in an isolated context; after compaction the most recent invocation of each skill is re-attached (5,000 tokens each, 25,000 total budget). (https://code.claude.com/docs/en/skills)

DSH: progressive disclosure (a pre-step `<available_skills>` catalog, full text via the `skill` tool) + four-level discovery (project `.dsh/skills` → customSkillDirs → `~/.dsh/skills` → bundled) + the `/name` user gesture (`packages/skill/tool-skill/src/index.ts:178-205`) + aligned `disable-model-invocation`/`user-invocable` fields. The only deltas are the allowed-tools pre-approval semantics and the compaction re-attach contract.

### 3.5 slash commands

Claude Code has merged custom commands into skills: `.claude/commands/deploy.md` behaves identically to `.claude/skills/deploy/SKILL.md`, the skill winning on a name clash; command names come from the file or directory name. (https://code.claude.com/docs/en/skills)

DSH's skill `/name` user gesture is already the equivalent channel. **Recommend closing C6 H5 (declarative command metadata)**: Claude Code's own evolution falsifies the value of a standalone markdown-command surface — parameterized templates belong to skills, not a separate command layer.

### 3.6 memory

Claude Code: four CLAUDE.md levels concatenate wide-to-narrow (no overriding); `@path` imports recurse 4 hops, and a project-level file importing outside the working directory prompts once for approval; auto memory is on by default, writing to `~/.claude/projects/<project>/memory/`, and the `MEMORY.md` index loads only its first 200 lines or 25KB per session; the docs state plainly that memory is context, not an enforcement layer — hard constraints must go through a PreToolUse hook or permissions.deny. (https://code.claude.com/docs/en/memory)

DSH: walk-up layering + nested projection (`packages/context/workspace-context`) + an auto memory store (global/session scopes, `packages/memory/memory`); retrieval is plain substring matching. Gap: the auto-memory "index file + capped loading" anti-bloat contract is worth borrowing.

### 3.7 permissions

Claude Code: 6 modes (default/acceptEdits/plan/bypassPermissions/auto/dontAsk); rules evaluate deny→ask→allow in that fixed order, first match wins, regardless of specificity; `Tool(specifier)` syntax (`Bash(npm run *)`), with gitignore path syntax for Read/Edit; Bash approvals persist to `.claude/settings.local.json` (compound commands split into per-subcommand rules, capped at 5); a bare-name deny removes the tool from the model's context entirely; no layer's deny can be overridden by another layer's allow; grant-type configuration is gated behind workspace trust. (https://code.claude.com/docs/en/permissions)

DSH: an OS-level sandbox tri-mode (read-only/workspace-write/danger-full-access, hard-enforced via Seatbelt/bwrap/Landlock) × an approval axis with only ask/never, one-shot per call, and no persistent rules (already listed as C6 H2). Philosophy: Claude Code runs "rule surface + sandbox defense-in-depth"; DSH runs "sandbox alone".

## 4. The two through-lines (the filter for what to borrow)

**Context economics**: subagent isolation, skill progressive disclosure, lazy loading of nested CLAUDE.md files, auto-memory caps — every design keeps information out of the main context. DSH already runs the same line (spark truncation, skill disclosure, the projection layer).

**The two-layer safety model**: settings/hooks enforce × CLAUDE.md advises, and every grant-type configuration passes through workspace trust. DSH's two layers are "sandbox enforces × prompt advises" — the rule layer in between is missing. Borrowing priority follows this: fill the rule layer > fill protocol coverage > fill the role surface.

## 5. Borrow list (value/cost order)

| # | Item | Claude Code mechanism essence | DSH landing spot | Cost |
|---|---|---|---|---|
| C7-1 | Hard read-only plan mode + plan file | Edit blocking at the tool layer; plan persisted for review | ✅ Landed: monotonic plan-mode guard (denies write/edit/git_commit/terminal_*, str_replace_editor discriminated by subcommand, bash allowed, `blockedTools` to extend) + `$DSH_HOME/plans/…` persistence + log-only `plan/file` event | Medium |
| C7-2 | Hook protocol completion | Make updatedInput effective; surface systemMessage | Half landed: systemMessage rides `hook/result` and is surfaced (both bridges + TUI); updatedInput awaits the pre-identity rewrite transaction (proposed note on file) | Medium |
| C7-3 | Permission rule persistence (deepens C6 H2) | Tool(specifier) syntax, deny→ask→allow evaluation, approvals written back to settings.local.json | interaction/user-approval + the settings layers | Medium-high |
| C7-4 | Project-level hook discovery + more events | Project settings merge with hot reload; Notification/SessionEnd/PreCompact | hooks-claude config surface + session lifecycle event points | Medium |
| C7-5 | Role-based subagent definition surface | Markdown agents + description-driven delegation routing + a built-in read-only explore | subagent service + tool-subagent; discovery can mirror the skill surface | High |
| C7-6 | Auto-memory index cap contract | MEMORY.md index + 200-line/25KB injection ceiling | The memory store's injection section | Low |

## 6. Not worth borrowing, and a C6 update

- Skip standalone markdown custom commands: Claude Code has merged commands into skills, and DSH's skill `/name` gesture is equivalent; recommend closing C6 H5 (to be annotated at C6's next revision).
- Skip the 6-mode cycle: DSH's orthogonal "mode tri-state × sandbox tri-mode" is clearer; acceptEdits ≈ workspace-write already has an equivalent.
- Skip the unanchored-regex matcher trap (Claude Code's `Edit.*` matches `NotebookEdit`): if DSH adds matchers, anchor the whole string.
- Output styles are low value: DSH already has the `systemPrompt.section` assembly surface.

## 7. Unverified items

- Whether DSH safety-scans subagent reports Claude Code-style (`<system-reminder>` escaping) — no counterpart found; confirm before implementing C7-5.
- The Claude Code Agent tool's full schema and the plan-file path convention: not in the official pages; issues point to `~/.claude/plans/`.
- Two official pages contradict each other on whether `allowed-tools` is gated by workspace trust; verify empirically.
