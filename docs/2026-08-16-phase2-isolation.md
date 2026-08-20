# Phase 2: Home Isolation + Migration + Renaming — Implementation Record

[中文](2026-08-16-phase2-isolation.zh.md) | English

Date: 2026-08-16 · Repository: oh-my-tianshu (formerly tianshu-public) · Goal: coexist with the official dsh and the `dsh-tianshu-tui` plugin installed side by side without interference (fully isolated user data); unify naming to remove semantic confusion.

## Change 1: Default home isolation (core)

`packages/util/paths/src/index.ts`:
- `DSH_HOME_DIR_NAME`: `.dsh` → `.dsh-tianshu`
- `DEFAULT_DSH_HOME_DISPLAY` follows automatically (`~/.dsh-tianshu`)
- Precedence unchanged: explicit config > `$DSH_HOME` > default (power-user / CI compatible)

Synchronized:
- `tests/paths.spec.ts` assertions updated (`.dsh` → `.dsh-tianshu`)
- `README.zh.md` / `README.md` (default-directory description)
- Hardcoded `.dsh` references in other packages cleaned up (settings-local comments etc.)
- `lib/` rebuilt (index.js tracked)

## Change 2: `migrate-home` command

New `apps/cli` subcommand `migrate-home`:
- Detection: old default home (`~/.dsh`) exists, new default home (`~/.dsh-tianshu`) does not
- Action: copy `~/.dsh` → `~/.dsh-tianshu` wholesale (`node:fs cpSync` recursive, **old kept** — conservative)
- Output: migration manifest + hint (old home retained; user may clean up after confirming)
- Idempotent: prompts and skips when the new home already exists

## Change 3: Naming unification (command + npm package)

The repository is already named `oh-my-tianshu` (remote verified). Unified to:
- **bin command**: `tianshu` → `oh-my-tianshu` (`apps/cli/package.json` bin field)
- **npm package name**: `@huiliyi37/dsh-tianshu` → `@huiliyi37/oh-my-tianshu`
- Synchronized: root package.json scripts, README/documentation command and package references, `args.ts` help copy
- The old package `@huiliyi37/dsh-tianshu` gets an npm deprecate later (at publish time)

## Change 4: Local rollout

- Run `migrate-home` (or manually) to move the current `~/.dsh` (already taken over by oh-my-tianshu) to `~/.dsh-tianshu`
- Restore the official `tui` profile: `npx -y @deepseek-ai/dsh plugin --profile tui add @huiliyi37/dsh-tianshu-tui`
- Verify: `oh-my-tianshu tui` (old) and `DSH_HOME=~/.dsh-tianshu oh-my-tianshu tui` each use their own home

## Order and verification

1. Change 1 → paths.spec all green + typecheck
2. Change 2 → cli tests (if any) + manual verification
3. Change 3 → full typecheck + tests; **confirm with the user before npm publish and git push** (publish-level action)
4. Change 4 → run locally (explain before touching `~/.dsh`)

## Execution record (after 2026-08-16 implementation)

1. **Default home isolation**: `dsh-paths` `DSH_HOME_DIR_NAME` → `.dsh-tianshu`; paths.spec 6/6, workspace-context 143/143, settings-local all green. Two pitfalls fixed during investigation: ① stale build artifacts left in paths `src/` (index.js resolved first by vite, making tests read old values) — deleted the leftovers + added the `@huiliyi37/dsh-paths → src` mapping in tsconfig.base.json (project convention, unified module instance); ② workspace-context cases relying on `node:os` mocks were fragile under dual instances — switched to an explicit dshHome (default-home resolution is covered by paths.spec).
2. **migrate-home**: new subcommand (copies the old `~/.dsh` to the new default home wholesale, keeps the old, idempotent); honors `$DSH_HOME` (verified the target follows when overridden by env).
3. **Naming unification**: command `tianshu` → `oh-my-tianshu`, npm package `@huiliyi37/dsh-tianshu` → `@huiliyi37/oh-my-tianshu`; repo-wide package-name references + documentation command usage replaced (84 files); brand/product identifiers (Tianshu/天枢/tianshu-harness) and the plugin name `dsh-tianshu-tui` retained; historical baseline snapshots (.artifacts) untouched.
4. **Local rollout**: `~/.dsh` data merged into `~/.dsh-tianshu` (add-only, never deletes); official `tui` profile restored (bundles = @deepseek-ai/dsh-base + dsh-tianshu-tui).

### Remaining (publish-level, pending confirmation)

- Push to github (3 commits: 1c738040 home isolation / 92425431 renaming)
- npm publish the new package name `@huiliyi37/oh-my-tianshu` + deprecate the old `@huiliyi37/dsh-tianshu`
- Global command reinstall (`npm i -g @huiliyi37/oh-my-tianshu`); the old `tianshu` command is uninstalled by the user
- Installed profiles depending on `@huiliyi37/dsh-tianshu` need reinstalling under the new package name
