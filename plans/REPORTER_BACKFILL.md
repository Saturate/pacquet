# Reporter Backfill Plan

Scope: pnpm/pacquet#347. Engine and `pnpm:stage` smoke test landed in
pnpm/pacquet#345. This plan enumerates the remaining channels from
[`@pnpm/core-loggers`](https://github.com/pnpm/pnpm/tree/086c5e91e8/core/core-loggers/src)
that pacquet needs to emit so its NDJSON output renders identically to
pnpm's under `@pnpm/cli.default-reporter`.

Upstream paths in this document are relative to the
[`pnpm/pnpm`](https://github.com/pnpm/pnpm) repo. Permalinks pin commit
`086c5e91e8` so they don't drift; resolve a fresh SHA with
`git ls-remote https://github.com/pnpm/pnpm.git refs/heads/main` and
update this doc when re-auditing.

## Per-PR conventions

Each backfill PR lands one channel, end-to-end:

1. **Add the variant.** Extend the `LogEvent` enum in `crates/reporter`
   with the channel and its payload. Mirror the upstream TS shape from
   `@pnpm/core-loggers` byte-for-byte (camelCase field names, `status`
   string sets, etc.) so `@pnpm/cli.default-reporter` accepts it.
2. **Reference the upstream permalink** (pinned SHA per the rule in
   `AGENTS.md`) for the emit site you're matching, in code comments, the
   commit message, and the PR description. This makes the parity check
   trivial for reviewers.
3. **Wire the emit site.** Find the pacquet location that mirrors the
   upstream call. Thread `R: Reporter` through the call chain if it
   doesn't already reach there; the trait monomorphises away, so the
   ergonomic cost is a turbofish at the production entry point and a
   generic on intermediate fns. See the engine PR (#345) for the pattern.
4. **Throttle on the emit side** for high-volume channels
   (`pnpm:fetching-progress`, `pnpm:progress`) — pnpm's reporter coalesces
   per-package events to 200ms, so emitting every byte of progress wastes
   serialization and pipe cost.
5. **Test.** Use the recording-fake DI pattern from #339: a unit-struct
   reporter declared inside the `#[test]` body, recording into a `static
   Mutex<Vec<LogEvent>>` declared in the same body. Assert the captured
   sequence.
6. **Verify the test catches the regression.** Temporarily delete the
   emit site, run the test, confirm it fails for the right reason, then
   restore. Same discipline `plans/TEST_PORTING.md` calls for.
7. **Check off the item below** as part of the PR.

## Channels in scope

Ordered roughly by user-visible value: download progress and per-package
status are what users see first when running `pacquet install`, so they
land before counts and summary.

### `pnpm:fetching-progress` — tarball download progress

- Upstream type:
  [`core/core-loggers/src/fetchingProgressLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/fetchingProgressLogger.ts).
  Two payloads: `{ status: 'started', attempt, packageId, size }` and
  `{ status: 'in_progress', downloaded, packageId }`.
- Upstream emit:
  [`installing/package-requester/src/packageRequester.ts:560`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/package-requester/src/packageRequester.ts#L560)
  fires `started` once per fetch attempt and `in_progress` for each chunk.
- Pacquet target: `crates/tarball/src/lib.rs`. The retry loop
  (`fetch_and_extract_once` and the surrounding driver) already tracks
  `attempt` and the response body — the natural emit site is right after
  the HEAD response (for `started`) and inside the body-streaming loop
  (for `in_progress`).
- Notes: this channel is per-byte. Throttle dedup per-package-id within
  a small window before emitting; pnpm's reporter throttles to 200ms and
  the JS side will drop the overflow anyway.
- [ ] Backfill `pnpm:fetching-progress`.

### `pnpm:progress` — per-package status transitions

- Upstream type:
  [`core/core-loggers/src/progressLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/progressLogger.ts).
  Status values: `resolved`, `fetched`, `found_in_store`, `imported`. The
  first three carry `{ packageId, requester }`; `imported` carries
  `{ method, requester, to }`.
- Upstream emits:
  - `resolved` —
    [`installing/deps-resolver/src/resolveDependencies.ts:1586`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/deps-resolver/src/resolveDependencies.ts#L1586).
  - `fetched` / `found_in_store` —
    [`installing/package-requester/src/packageRequester.ts:435`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/package-requester/src/packageRequester.ts#L435).
  - `imported` —
    [`installing/deps-installer/src/install/link.ts:498`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/deps-installer/src/install/link.ts#L498).
- Pacquet targets:
  - `resolved` — `crates/package-manager/src/install_without_lockfile.rs`
    (resolver path) and the lockfile-walk in
    `crates/package-manager/src/install_frozen_lockfile.rs` (each lockfile
    snapshot is "already resolved" and could emit `resolved`
    immediately).
  - `fetched` / `found_in_store` — `crates/package-manager/src/install_package_from_registry.rs`
    and the prefetch path; the `MemCache` already distinguishes hits and
    misses.
  - `imported` — `crates/package-manager/src/create_virtual_dir_by_snapshot.rs`
    after `create_cas_files` completes. The chosen import method is in
    scope at this site.
- Notes: `requester` is the project root (same value as `pnpm:stage`'s
  `prefix` — see the `findWorkspaceDir` TODO already in `Install::run`).
- [ ] Backfill `pnpm:progress` (`resolved`, `fetched`, `found_in_store`,
  `imported`).

### `pnpm:stage` — `resolution_started` / `resolution_done`

- Upstream type:
  [`core/core-loggers/src/stageLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/stageLogger.ts).
  `importing_started` / `importing_done` already wired in #345.
- Upstream emits:
  - `resolution_started` —
    [`installing/deps-installer/src/install/index.ts:1232`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/deps-installer/src/install/index.ts#L1232).
  - `resolution_done` —
    [`installing/deps-installer/src/install/index.ts:1375`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/deps-installer/src/install/index.ts#L1375).
- Pacquet target: `crates/package-manager/src/install_without_lockfile.rs`
  brackets resolution; the frozen-lockfile path skips this stage by
  design (the lockfile *is* the resolution).
- Notes: emit only when resolution actually runs. `--frozen-lockfile`
  installs proceed straight from `importing_started` → `importing_done`,
  matching pnpm.
- [ ] Backfill `pnpm:stage` `resolution_started` / `resolution_done`.

### `pnpm:context` — install start metadata

- Upstream type:
  [`core/core-loggers/src/contextLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/contextLogger.ts).
  Payload: `{ currentLockfileExists, storeDir, virtualStoreDir }`.
- Upstream emit:
  [`installing/context/src/index.ts:196`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/context/src/index.ts#L196)
  (also fires from a second site in the same file at `:359`).
- Pacquet target: `Install::run` in `crates/package-manager/src/install.rs`,
  immediately before the existing `importing_started` emit.
- Notes: `currentLockfileExists` should reflect
  `node_modules/.pnpm/lock.yaml` once that's being read/written; until
  then, hard-code `false` and add a TODO so the emit doesn't go stale.
- [x] Backfill `pnpm:context`.

### `pnpm:stats` — added / removed counts

- Upstream type:
  [`core/core-loggers/src/statsLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/statsLogger.ts).
  Two payloads: `{ prefix, added }` and `{ prefix, removed }`.
- Upstream emits:
  [`installing/deps-installer/src/install/link.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/deps-installer/src/install/link.ts)
  and
  [`installing/deps-restorer/src/index.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/deps-restorer/src/index.ts)
  emit after the link phase completes.
- Pacquet target: `crates/package-manager/src/create_virtual_store.rs` —
  count packages newly linked vs. removed in the cleanup pass.
- Notes: `removed` is currently always 0 because pacquet doesn't prune
  yet. Emit `removed: 0` to keep the wire shape stable; revisit when
  pruning lands.
- [ ] Backfill `pnpm:stats`.

### `pnpm:summary` — end-of-install summary

- Upstream type:
  [`core/core-loggers/src/summaryLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/summaryLogger.ts).
  Payload: `{ prefix }`. The reporter combines this with the accumulated
  `pnpm:root` events to render the final "+N -M" block.
- Upstream emit:
  [`installing/deps-installer/src/install/index.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/deps-installer/src/install/index.ts)
  and
  [`installing/deps-restorer/src/index.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/deps-restorer/src/index.ts).
- Pacquet target: `Install::run` after the existing `importing_done`
  emit.
- Notes: must come after `pnpm:root` emits so the reporter can render
  the diff block.
- [x] Backfill `pnpm:summary`.

### `pnpm:package-import-method` — clone / hardlink / copy decision

- Upstream type:
  [`core/core-loggers/src/packageImportMethodLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/packageImportMethodLogger.ts).
  Payload: `{ method: 'clone' | 'hardlink' | 'copy' }`.
- Upstream emit:
  [`fs/indexed-pkg-importer/src/index.ts:32`](https://github.com/pnpm/pnpm/blob/086c5e91e8/fs/indexed-pkg-importer/src/index.ts#L32)
  fires once when the importer is constructed; the `auto` branch fires
  the actual method that succeeded.
- Pacquet target: `crates/package-manager/src/create_virtual_store.rs`
  (line ~232) where `import_method = config.package_import_method` is
  resolved. Emit once per install when the chosen method is known.
- Notes: pacquet's `PackageImportMethod` enum already has `Auto`, `Clone`,
  `Hardlink`, `Copy`, `Reflink`. `Reflink` doesn't have a pnpm equivalent
  — emit `clone` for `Reflink` (it's the closest concept upstream
  understands) and add a TODO so we revisit if upstream gains a `reflink`
  variant.
- [ ] Backfill `pnpm:package-import-method`.

### `pnpm:request-retry` — HTTP retry events

- Upstream type:
  [`core/core-loggers/src/requestRetryLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/requestRetryLogger.ts).
  Payload: `{ attempt, error, maxRetries, method, timeout, url }`.
- Upstream emits:
  - Tarball fetch retries —
    [`fetching/tarball-fetcher/src/remoteTarballFetcher.ts:106`](https://github.com/pnpm/pnpm/blob/086c5e91e8/fetching/tarball-fetcher/src/remoteTarballFetcher.ts#L106).
  - Generic network fetch retries —
    [`network/fetch/src/fetch.ts:90`](https://github.com/pnpm/pnpm/blob/086c5e91e8/network/fetch/src/fetch.ts#L90).
  - Registry-side retries —
    [`resolving/npm-resolver/src/fetch.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/resolving/npm-resolver/src/fetch.ts).
- Pacquet targets:
  - Tarball — `crates/tarball/src/lib.rs`'s retry driver
    (`fetch_and_extract_once` callers, around the existing `RetryOpts`).
  - Registry / generic — `crates/registry/src/package_distribution.rs`
    and `crates/registry/src/package_version.rs`. Pacquet currently has
    no retry loop in those paths; the channel only fires when retries
    happen, so wire it once retries land. Track the retry-loop work
    separately if needed.
- Notes: `error` is a structured object, not a string. Mirror the field
  set (`name`, `message`, `status`, `code`, `errno`) — the reporter
  decides which to display.
- [ ] Backfill `pnpm:request-retry` (tarball path).
- [ ] Backfill `pnpm:request-retry` (registry path; gated on retry-loop
  parity).

### `pnpm:lifecycle` — script execution

- Upstream type:
  [`core/core-loggers/src/lifecycleLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/lifecycleLogger.ts).
  Three payload shapes: stdio (`{ depPath, stage, wd, line, stdio }`),
  exit (`{ depPath, stage, wd, exitCode, optional }`), script
  (`{ depPath, stage, wd, script, optional }`).
- Upstream emit:
  [`exec/lifecycle/src/runLifecycleHook.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/exec/lifecycle/src/runLifecycleHook.ts)
  emits `script` at start, streams `stdio` per output line, and emits
  `exit` at the end.
- Pacquet target: `crates/executor/src/lib.rs`'s `execute_shell` is the
  current entry point but it's too thin for streaming output. This
  channel needs the executor to grow line-by-line capture — track
  separately as part of the lifecycle-scripts feature.
- Notes: `stage` here means the npm-script name (`preinstall`, `postinstall`,
  …), not the install pipeline phase. Don't confuse with `pnpm:stage`.
- [ ] Backfill `pnpm:lifecycle` (gated on streaming-executor work).

### `pnpm:root` — top-level dependency add / remove

- Upstream type:
  [`core/core-loggers/src/rootLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/rootLogger.ts).
  Payload: `{ prefix, added: { id, name, realName, version, dependencyType, latest, linkedFrom } }`
  or `{ prefix, removed: { name, version, dependencyType } }`.
- Upstream emit:
  [`installing/linking/direct-dep-linker/src/linkDirectDeps.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/linking/direct-dep-linker/src/linkDirectDeps.ts).
- Pacquet target: `crates/package-manager/src/symlink_direct_dependencies.rs`
  fires after each direct symlink is created.
- Notes: powers the "+N -M" output the user sees at the end of an
  install, paired with `pnpm:summary`.
- [ ] Backfill `pnpm:root`.

### `pnpm:link` — symlink to store

- Upstream type:
  [`core/core-loggers/src/linkLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/linkLogger.ts).
  Payload: `{ target, link }`.
- Upstream emit:
  [`fs/symlink-dependency/src/index.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/fs/symlink-dependency/src/index.ts).
- Pacquet target: `crates/package-manager/src/symlink_package.rs` and
  `link_file.rs`.
- Notes: low priority — the reporter doesn't render this channel by
  default in pnpm. Backfill last.
- [ ] Backfill `pnpm:link`.

### `pnpm:package-manifest` — manifest read / write

- Upstream type:
  [`core/core-loggers/src/packageManifestLogger.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/core/core-loggers/src/packageManifestLogger.ts).
  Two payloads: `{ prefix, initial }` and `{ prefix, updated }`.
- Upstream emit:
  [`installing/context/src/index.ts`](https://github.com/pnpm/pnpm/blob/086c5e91e8/installing/context/src/index.ts)
  emits `initial` at install start; deps-installer / restorer emit
  `updated` after `pacquet add` mutates the manifest.
- Pacquet targets:
  - `initial` — `Install::run` entry, alongside the `pnpm:context` emit.
  - `updated` — `Add::run` after the manifest has been re-saved.
- [ ] Backfill `pnpm:package-manifest`.

## Channels gated on feature work

These channels exist in `@pnpm/core-loggers` but only fire from code
paths pacquet hasn't built yet. They land alongside the feature, not as
part of this sweep:

- **`pnpm:install-check`** — fires from
  [`config/package-is-installable`](https://github.com/pnpm/pnpm/blob/086c5e91e8/config/package-is-installable/src/index.ts);
  pacquet doesn't enforce engine / cpu / os filters yet.
- **`pnpm:peer-dependency-issues`** — peer-dep resolution.
- **`pnpm:deprecation`** — fires when the resolver sees a deprecated
  package; pacquet doesn't surface deprecation metadata yet.
- **`pnpm:hook`** — `.pnpmfile.cjs` hook execution. No pnpmfile support
  yet.
- **`pnpm:scope`** — workspace scope. No workspace support yet.
- **`pnpm:ignored-scripts`** — `--ignore-scripts` and the build-allow-list.
  Gated on lifecycle-script support.
- **`pnpm:installing-config-deps`** — config-dep installation; pacquet
  doesn't have a config-deps feature.
- **`pnpm:update-check`** — version-update notifier; pacquet has no
  self-update path.
- **`pnpm:execution-time`** — overall install timing; nice-to-have, no
  hard dependency, but not a parity blocker.
- **`pnpm:registry`** — never directly emitted upstream; `@pnpm/logger`
  exposes the channel but no live caller writes to it. Skip.
