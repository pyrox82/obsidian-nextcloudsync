# AGENTS.md

Guidance for AI coding agents working in this repository.

## Project

**Nextcloud Sync** (`id: nextcloud-sync`) is an Obsidian community plugin that syncs a vault
bidirectionally with Nextcloud over WebDAV (desktop and mobile; `isDesktopOnly: false`).
Data safety comes first: no corrupted files, no endless conflict copies, no unintended loss.
Plain WebDAV servers are a documented fallback (`StandardWebDAVClient`), not a supported target.

- Language: TypeScript (strict), bundled with esbuild into a single CJS `main.js` (target `es2018`).
- Runtime deps are deliberately few: `node-diff3`, `reconcile-text`, `semver`.
- Package manager: **pnpm only** (version pinned in `package.json` → `packageManager`). Never add
  `package-lock.json` / `yarn.lock`. Node 26.

## Sources of truth — read before changing behaviour

| File                                            | Role                                                                                                                                                                  |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `docs/spec.md`                                  | **WHAT** the plugin does. Single source of truth for behaviour, defaults and test clauses (`CN-*`, `CF-*`, `WSF-*`, …). Rewritten in full whenever behaviour changes. |
| `docs/plan.md`                                  | **HOW** it is implemented: components, phase order, module conventions (§17), dependency direction, layer structure (§19).                                            |
| `tests/README.md`                               | Test layers, Docker runner, dedup rule, spec tagging.                                                                                                                 |
| `tests/a-no-nextcloud/spec-coverage/clauses.ts` | Catalog of every spec clause the suite must cover.                                                                                                                    |

When behaviour changes (docs/spec.md §19): **(1)** update `docs/spec.md`, **(2)** update the clause in
`clauses.ts`, **(3)** add/update tests tagged with that clause, **(4)** make `pnpm test` green
(the coverage meta-test fails on uncovered clauses or unknown tags).

## Commands

```bash
pnpm install            # also sets git core.hooksPath=.githooks
pnpm dev                # esbuild watch (OUTDIR=<vault>/.obsidian/plugins/nextcloud-sync to build into a vault)
pnpm build              # tsc --noEmit + production bundle
pnpm lint               # eslint src (mirrors the Obsidian community-directory reviewer)
pnpm test               # layer "a": pure logic + spec-coverage meta-test (the only layer in CI)
pnpm exec jest path/to/file.test.ts   # single test file
bash tests/docker/run.sh <a|b1|b2|b3|b4|all>      # full Docker-based suite
bash tests/docker/run.sh --changed <layer|all>    # skip layers already passed on this exact tree
```

Before declaring a change done, run at least `pnpm lint`, `pnpm build` and `pnpm test`.

## Repository layout

```
src/
  main.ts              plugin entry: init, DI, lifecycle, commands
  auth/                Nextcloud Login Flow v2
  data/                LocalAdapter (atomic writes, IgnoreList), StateDB, MergeBaseStore, SyncHistoryStore
  network/             IWebDAVClient, WebDAVFactory, NextcloudClient, StandardWebDAVClient, dav/ (pure PROPFIND parsing)
  sync/
    SyncEngine.ts      orchestration + per-file decision (composition root)
    ConflictResolver.ts  pure conflict decision (no I/O)
    policy/            pure decision functions
    scan/ transfer/ conflict/ resolution/ deletion/ directory/ watch/ mirror/ versions/ session/ ctime/
    merge/             MergeEngine → ReconcileTextStrategy → Diff3Strategy, FrontmatterMergeStrategy
    upload/            Simple vs Chunked upload strategy
    plugins/           community-plugin sync (mostly pure helpers + PluginSyncCoordinator)
  settings/            SettingTab, settingDefinitions (declarative settings API), slider limits
  ui/                  modals, status bar, ribbon
  util/                limits.ts (tuning constants), fixedSyncConfig.ts, retry, mutex, hashing, …
tests/
  a-no-nextcloud/        *.test.ts      pure logic, no network/UI (default, CI)
  b1-nextcloud-headless/ *.b1.test.ts   live Nextcloud in Docker
  b2-nextcloud-ui/       *.b2.test.ts   real desktop Obsidian via wdio
  b3-android-ui/         *.b3.test.ts   real Obsidian on Android (Redroid, API 33)
  b4-plain-webdav/       *.b4.test.ts   Apache mod_dav — deliberately NOT Nextcloud
  docker/                run.sh + compose for all layers
```

## Architecture rules (docs/plan.md §17)

- **Separate decision from execution.** Put decisions in pure functions (`sync/policy/`, `network/dav/`,
  `sync/directory/classify.ts`, `ConflictResolver.decide`, `sync/plugins/*` helpers) and keep I/O in services.
- **Do not import `SyncEngineOptions`** in modules; each module declares the minimal interface it needs.
- `IWebDAVClient` / `IUploadStrategy` are **passed as arguments, never stored as fields** (clients are
  lazily recreated by `ensureClient`).
- Values that change during the engine's lifetime (settings, `isNextcloud`, times, `isCancelled`) are
  passed as **accessors**, not values. Lazy evaluation is part of the contract.
- Mutable state lives with its owner: core-loop/session state stays in `SyncEngine`; state only a module
  reads belongs to that module.
- Only `watch/` calls back into the engine, via the injected `processFile` port (no import cycles).
  Watch never decides on its own — it goes through the same classifier as a full sync.
- Services never import `PluginSyncCoordinator`; they receive narrow ports (`PluginPathGate`,
  `LocalWritePort`, `PluginSessionPort`).
- Server capability detection is based on **probe response contents**, never on whether an exception was thrown.
- Local writes are atomic (tmp → rename). Keep it that way; see docs/spec.md §9.
- Tuning constants go in `src/util/limits.ts`; hidden fixed values in `src/util/fixedSyncConfig.ts`.
- Clients read the app password on every request (`AppPasswordReader`) — do not cache it.

## Testing rules

- **One behaviour, one layer.** Pick the first that can express it: a (pure) → b1 (live server) →
  b2 (desktop UI) → b3 (only Capacitor/Android-specific behaviour). Do not test the same behaviour in both a and b1.
- Reference spec clauses **only** via a `[SPEC:<id>]` tag or the `spec()` helper:
  ```ts
  import { spec } from "../support/specRef";
  it(`${spec("CF-2", "FR-008")} same-line conflict skips`, () => {
    /* ... */
  });
  ```
  A bare `CF-2` does not count; skipped tests (`it.skip`, `xit`) do not count as coverage.
- Directly extracted modules get untagged direct tests; clause tags stay on the `SyncEngine`-level tests
  (don't double-tag a clause).
- In layer a, `obsidian` is mocked by `tests/a-no-nextcloud/support/obsidian.ts`.
- Live layers need Docker; agents usually cannot run b2/b3 (b3 needs host binder + an AppArmor profile).
  Say so instead of claiming they passed.

## Code conventions

- `eslint-plugin-obsidianmd` recommended + project overrides (`eslint.config.mjs`). Notable:
  - no `any` (`@typescript-eslint/no-explicit-any: error`);
  - use `createDiv()` / `createSpan()`, never `createEl('div'|'span')`;
  - `console.log` is banned; `console.warn` / `console.error` are allowed;
  - every `eslint-disable` needs a `-- reason`;
  - UI strings are **sentence case** (brand words like Nextcloud, WebDAV are allowlisted).
- Comments pointing at docs must use the form `docs/spec.md §N` / `docs/plan.md §N`; the `DOC-1` test
  checks the section exists. `DOC-2` forbids private paths (`specs/`, `report/`) and feature-number
  history markers in comments.
- Do not add `js-yaml` (frontmatter is parsed with Obsidian's API). Anything not imported from `src/`
  goes in `devDependencies`. All version specifiers use `^`; the lockfile does the pinning.
- Held versions: TypeScript **6.0.3** (no 7.x), `@wdio/*` **9.31.6**. Don't bump them casually (DEP-1 test).

## Language rules (enforced by git hooks)

- Commit messages, code comments, and all files **except `*.ja*`** must be **English** — no CJK characters.
- `README.ja.md` and `CHANGELOG.ja.md` are the Japanese translations; keep them in sync with their
  English counterparts when you change user-facing docs.

## Git hooks (`.githooks/`)

- **pre-commit**: secretlint on staged files; English-only content; optional author gate — it only
  runs when the local git config sets `hooks.allowedAuthorEmail`, and then rejects any other author
  email. No identity is hard-coded in the repo. Do not bypass hooks with `--no-verify` unless the
  user explicitly asks.
- **commit-msg**: rejects CJK in the message.
- **pre-push**: `pnpm lint`, `pnpm build`, README/README.ja must name the current `manifest.json`
  version in the "What's new" heading and mention `minAppVersion`; stable versions need a
  `[x.y.z]` entry in both CHANGELOGs; beta versions run test layer a.

## Personal fork

This repository is a **personal fork**. Its changes are never contributed back.

- Remotes: `origin` = the fork (push, PRs, releases), `upstream` = the original project
  (`siosig/obsidian-nextcloudsync`), **fetch only**. Never push to `upstream`, and never open
  issues or PRs there.
- Open PRs against the fork itself: `gh pr create --repo <origin owner>/obsidian-nextcloudsync --base main`
  (the fork is the `gh` default repo; check with `gh repo set-default --view`).
- Pulling upstream changes in: `git fetch upstream` and merge `upstream/main` into a branch of the fork, then PR it.
- Upstream links in README/CHANGELOG/SECURITY/LICENSE and the `author` fields stay as they are
  (MIT attribution and upstream history).

## Commits, branches, releases

- Conventional commits: `fix(sync): …`, `feat(…)`, `chore(release): bump version to x.y.z`.
- Feature/fix branches are named `<issue-number>-<slug>`, merged into `develop`, released from `main`.
- Versions must match in `manifest.json`, `package.json` and `versions.json`.
- Pushing a bare version tag (`1.0.18`, no `v`) triggers `.github/workflows/release.yml`; tags with `-`
  (e.g. `1.0.19-beta.1`) become pre-releases (BRAT). Betas are not listed in the CHANGELOG.
- A beta release requires `bash tests/docker/run.sh --changed all` to pass on the released tree.
- Only commit, push, tag or release when the user asks.

## Do not

- Commit `.env*`, `.secretlintrc.local.json`, `main.js`, or anything under `specs/`, `report/`, `refs/`,
  `.specify/` (gitignored, private).
- Weaken data-safety invariants (docs/spec.md §18.1) to make a test pass — fix the code or update the
  spec clause deliberately.
- Disable TLS verification in tests; the suite uses a per-run CA.
