# API Key Routing Customization Handoff

## Scope

This repository maintains a downstream Cockpit Tools customization on
`feature/api-key-routing-usage`.

The customization provides:

- A selectable Codex account pool for each client API Key.
- Inheritance from the service account pool when a Key has no explicit
  account IDs.
- Per-Key request count, token usage, success rate, and estimated cost.
- End-to-end `accountIds` propagation across TypeScript, Tauri commands, and
  Rust persistence.
- Sidecar enforcement, per-Key session-affinity isolation, account-scope
  validation, and compact token display added by later hardening commits.

Do not record passwords, API Keys, OAuth tokens, signing keys, or other
credentials in this document, patches, commits, logs, or pull requests.

## Archived State

State checked before the upstream fetch on 2026-07-11:

- Working branch: `feature/api-key-routing-usage`.
- Initial working tree: clean. `src-tauri/Cargo.toml` had no text diff.
- Initial `HEAD`: `75dbb614` (`docs: add downstream customization maintenance guide`).
- Branch upstream: `fork/feature/api-key-routing-usage`.
- Official remote: `origin` -> `https://github.com/jlcodes99/cockpit-tools.git`.
- Personal fork remote: `fork` -> `https://github.com/kin001/cockpit-tools.git`.
- Official tracking commit before fetch: `93d939b7`.
- Custom package version before fetch: `1.1.7`.
- Requested historical core commit: `2629dc16`.
- Rebased equivalent of that core change in the current branch: `1911dc3d`.

`2629dc16` is not an ancestor of the current branch because the branch was
previously rebased. Its patch is archived at:

```text
docs/maintenance/patches/0001-Add-API-key-account-routing-controls.patch
```

That patch contains only the original five-file implementation. The complete
current behavior also depends on the later commits from `03bd0d67` through
`96a8e096`; restoring only the archived patch does not restore all hardening,
sidecar, compatibility, or display changes.

Local untracked material under `docs/superpowers/plans` is not part of the
official contribution and must not be deleted or included in an upstream PR.

## Key Files

- `src-tauri/src/commands/codex.rs`
- `src-tauri/src/models/codex_local_access.rs`
- `src-tauri/src/modules/codex_local_access.rs`
- `src/services/codexLocalAccessService.ts`
- `src/types/codexLocalAccess.ts`
- `src/pages/CodexApiServicePage.tsx`
- `src/pages/CodexApiServicePage.css`
- `src/utils/codexApiKeyAccountScope.ts`
- `sidecars/cockpit-cliproxy/main.go`
- `sidecars/cockpit-cliproxy/cdk/CLIProxyAPI/sdk/cliproxy/auth/selector.go`

## Recovery

Preferred full recovery uses the personal fork because it contains the whole
hardened branch:

```powershell
git fetch fork
git switch -c feature/api-key-routing-usage --track fork/feature/api-key-routing-usage
```

To restore only the original core change onto a compatible clean checkout:

```powershell
git switch -c recover/api-key-routing origin/main
git am docs/maintenance/patches/0001-Add-API-key-account-routing-controls.patch
```

If `git am` conflicts, abort it before retrying a manual port:

```powershell
git am --abort
```

For disaster recovery of the exact pre-fetch state, retain the commit ID
`75dbb614` and create a branch without resetting the current worktree:

```powershell
git branch recovery/api-key-routing-pre-fetch 75dbb614
```

Never use `git reset --hard`, `git clean`, or checkout-based file restoration
when uncommitted or untracked files may be present.

## Upstream Upgrade Procedure

1. Inspect and preserve local state:

   ```powershell
   git status --short --branch
   git diff --stat
   git diff
   git log --oneline --decorate -12
   git branch --show-current
   ```

2. Refresh official history and inspect it before merging:

   ```powershell
   git fetch origin
   git log --oneline --decorate HEAD..origin/main
   git diff --stat HEAD...origin/main
   git diff --name-status HEAD...origin/main
   ```

3. Create a backup reference, then merge the official branch:

   ```powershell
   git branch backup/pre-upstream-sync-$(Get-Date -Format yyyyMMdd-HHmmss)
   git merge origin/main
   ```

4. Resolve conflicts while preserving both upstream behavior and every item
   in the Scope section. Pay particular attention to the key files above and
   to version metadata in `package.json`, `package-lock.json`,
   `src-tauri/Cargo.toml`, `src-tauri/tauri.conf.json`, and `Cargo.lock`.

5. Review the resolved merge before committing:

   ```powershell
   git status --short
   git diff --check
   git diff --cached --stat
   ```

Do not use the official in-app updater for this checkout. It installs an
official binary without the downstream behavior even though stored settings
remain on disk.

## Verification

Run the required TypeScript and Rust regression checks from the repository
root:

```powershell
npm run typecheck

$env:COCKPIT_SKIP_CLIPROXY_BUILD='1'
Set-Location src-tauri
cargo test -q custom_api_key_scope_filters_duplicates_and_updates_manifest_scope --lib
Set-Location ..
```

The historical test name
`update_api_key_account_scope_filters_duplicates_and_updates_manifest_scope`
no longer matches a registered test after the previous hardening refactor. It
returns success with `0 tests`; do not treat that as regression evidence. The
current equivalent name above must report `1 passed`.

Also run the focused JavaScript tests when their files are present:

```powershell
npm run test:codex-api-key-scope
node --test src/utils/codexApiServiceCompatibility.test.ts
```

Go `1.26.5` is available on the current maintenance machine. Verify both the
Cockpit wrapper and the affected embedded CLIProxyAPI packages before a local
release:

```powershell
Set-Location sidecars/cockpit-cliproxy
go test ./...
Set-Location cdk/CLIProxyAPI
go test ./internal/watcher ./internal/runtime/executor ./sdk/api/handlers/openai
Set-Location ../../../..
```

The retained Windows sidecar binary is located at:

```text
sidecars/cockpit-cliproxy/bin/cockpit-cliproxy-x86_64-pc-windows-msvc.exe
```

## Packaging

There is no `TAURI_SIGNING_PRIVATE_KEY` on this machine. Keep the committed
release configuration unchanged and disable updater artifacts only for the
local build by using the existing CI override:

```powershell
Remove-Item Env:COCKPIT_SKIP_CLIPROXY_BUILD -ErrorAction SilentlyContinue
npm run tauri -- build --config src-tauri/tauri.ci.conf.json
```

`src-tauri/tauri.ci.conf.json` sets `bundle.createUpdaterArtifacts=false` and
therefore avoids the updater-signing requirement. With Go available, do not
skip the sidecar build after merging upstream sidecar source changes. Set
`COCKPIT_SKIP_CLIPROXY_BUILD=1` only as an explicit recovery fallback when a
known-compatible binary has already been verified. This workspace uses the
repository-level Cargo target directory. Windows installers are written under:

```text
target/release/bundle/nsis
target/release/bundle/msi
```

After building, record the installer name, size, timestamp, and SHA-256 hash:

```powershell
Get-ChildItem target\release\bundle\nsis,target\release\bundle\msi
Get-FileHash target\release\bundle\nsis\*.exe,target\release\bundle\msi\*.msi -Algorithm SHA256
```

## Publishing

Push the feature branch to the personal fork, never directly to the official
repository:

```powershell
git push fork feature/api-key-routing-usage
```

If history was intentionally rebased, use `--force-with-lease`, not a plain
force push. Open the pull request from
`kin001:feature/api-key-routing-usage` to `jlcodes99:main`. Before publishing,
exclude local planning files and scan the diff for credentials and generated
artifacts.

## Latest Sync Result

Sync performed on 2026-07-11:

- Fetched official commit: `c9e4c856` (`chore(homebrew): update cask for
  v1.1.5 (#1506)`).
- New official tag: `v1.1.5`.
- Merge parents: downstream `75dbb614` and official `c9e4c856`.
- Backup reference: `backup/pre-origin-main-merge-20260711-112957`.
- Version retained: downstream `1.1.7`, newer than official `1.1.5`.
- Conflicts resolved in both changelogs, `Cargo.lock`, `package.json`,
  `sidecars/cockpit-cliproxy/main.go`, `src-tauri/Cargo.toml`,
  `src-tauri/src/modules/codex_local_access.rs`, and
  `src-tauri/tauri.conf.json`.
- Resolution preserved per-Key account scopes, inherited empty scopes, usage
  statistics, sidecar scope enforcement, session isolation, and compact token
  display while adding the official OAuth quota-reserve behavior and release
  updates.
- One official Rust test initializer required the downstream
  `inherit_account_pool` field after the merge.

Verification results:

- `npm run typecheck`: passed.
- `npm run test:codex-api-key-scope`: 5 passed.
- `node --test src/utils/codexApiServiceCompatibility.test.ts`: 2 passed.
- Historical Rust command: compiled successfully but ran 0 tests because the
  old name is no longer registered.
- Current Rust equivalent
  `custom_api_key_scope_filters_duplicates_and_updates_manifest_scope`: 1
  passed.
- `cargo fmt --check`: passed after formatting the merged test imports.
- Go tests: not run because Go is unavailable on this machine.
- Tauri build with `COCKPIT_SKIP_CLIPROXY_BUILD=1` and
  `src-tauri/tauri.ci.conf.json`: passed.

Packaged artifacts:

```text
target/release/bundle/nsis/Cockpit Tools_1.1.7_x64-setup.exe
size: 26325870 bytes
time: 2026-07-11 12:06:12 +08:00
sha256: 559499239038037A209D643C89D7737649509A9085757535F28D876DBA23A527

target/release/bundle/msi/Cockpit Tools_1.1.7_x64_en-US.msi
size: 35926016 bytes
time: 2026-07-11 12:05:14 +08:00
sha256: 86196B21ABCD883556FBEFFD7B6EE11F1E5A91572713CC0D5152E573EA48D3E8
```

The reused sidecar was not rebuilt:

```text
sidecars/cockpit-cliproxy/bin/cockpit-cliproxy-x86_64-pc-windows-msvc.exe
size: 19257344 bytes
time: 2026-07-10 18:09:55 +08:00
sha256: A3DF9E4165E5A15A5684F1F84593DF873013BF6103AFBD02682E19C9B4770DF9
```

Because this binary predates the fetched 2026-07-11 upstream source and Go is
unavailable, this local package does not prove that newly merged upstream
sidecar changes are present in the bundled executable. Rebuild and rerun the
Go tests on a Go-enabled machine before treating the package as a complete
release candidate. The existing binary does contain the previously packaged
downstream API Key scope enforcement.

The build emitted existing Rust unused/dead-code warnings and a Tauri warning
that `__TAURI_BUNDLE_TYPE` was not found while patching MSI/NSIS binaries.
Both local bundles were still produced successfully; updater artifacts were
disabled for this unsigned local build.

## Calendar Period Fix And Local Package 1.1.10

Follow-up performed on 2026-07-11:

- Replaced rolling 24-hour, 7-day, and 30-day usage ranges with local calendar
  periods: today from local midnight, the current week from Monday at local
  midnight, and the current month from its first day at local midnight.
- Centralized the boundary calculation for empty snapshots, request-log range
  filters, in-memory window recomputation, and request-log loading from disk.
- Updated client-key range labels to Today, This week, and This month, with
  Simplified and Traditional Chinese translations.
- Added boundary regression coverage for events immediately before and after
  day, week, and month starts, plus refresh behavior after crossing midnight.
- Published the fix to the official contribution branch in commit `ff6ce7a7`.
  Pull request: <https://github.com/jlcodes99/cockpit-tools/pull/1514>.

Verification results on `feature/api-key-routing-usage`:

- `npm run typecheck`: passed.
- `npm run test:codex-api-key-scope`: 5 passed.
- `node --test src/utils/codexApiServiceCompatibility.test.ts`: 2 passed.
- `api_key_usage_stats_are_isolated_by_time_window`: 1 passed.
- `custom_api_key_scope_filters_duplicates_and_updates_manifest_scope`: 1
  passed.
- `rustfmt --edition 2021 --check
  src-tauri/src/modules/codex_local_access.rs`: passed.
- Tauri build with `COCKPIT_SKIP_CLIPROXY_BUILD=1` and
  `src-tauri/tauri.ci.conf.json`: passed.

Packaged artifacts:

```text
target/release/bundle/nsis/Cockpit Tools_1.1.10_x64-setup.exe
size: 26353318 bytes
time: 2026-07-11 20:23:45 +08:00
sha256: 9CF947D2A7C625ECB15C4FCE48D87D4E6677526482AB1D70E6DEAFC180442EF1

target/release/bundle/msi/Cockpit Tools_1.1.10_x64_en-US.msi
size: 35921920 bytes
time: 2026-07-11 20:22:50 +08:00
sha256: AD2171C1C6218D474B302B6140E42065D0F478FF8E359A47C9697DD7F125D3D8
```

The existing compatible sidecar was reused without a Go rebuild:

```text
sidecars/cockpit-cliproxy/bin/cockpit-cliproxy-x86_64-pc-windows-msvc.exe
size: 19277824 bytes
time: 2026-07-11 12:29:46 +08:00
sha256: 007AF7AA6D49F2A8DFAA6EDABD3A8FF4F0A95566060988EAF95C70F6C74C8ED7
```

Local installation verification:

- NSIS silent installer exit code: `0`.
- Installed product version: `1.1.10`.
- Installed application SHA-256:
  `F8936EE5418E2AD350EA33671FFF03DDA5E7274F1574B6DC6670AA510375EBA1`.
- Restarted application process ID: `16860`.
- API sidecar port `54548`: listening.
- Unauthenticated `GET /v1/models`: HTTP `401`, confirming the restarted
  service is reachable and enforcing authentication without recording a
  client API Key in the verification log.

## Upstream 1.3.0 Sync

Sync started on 2026-07-13:

- Fetched official `origin/main` at `a015fcd3` and release tag `v1.3.0` at
  `da0deca4`.
- Created rollback reference
  `backup/pre-origin-main-1.3.0-sync-20260713-095845` before merging.
- Adopted upstream version `1.3.0` in `package.json`, `package-lock.json`,
  `src-tauri/Cargo.toml`, `src-tauri/tauri.conf.json`, and `Cargo.lock`.
- Resolved conflicts in both changelogs, version metadata, the API Service
  page, and `codex_local_access.rs`.
- Preserved per-Key account scopes, inherited empty scopes, sidecar account
  filtering, Key-scoped affinity, Key usage statistics, and local calendar
  day/week/month boundaries. Upstream request-log display names, custom-route
  backup accounts, and upstream Base URL recovery were retained too.

Verification before packaging:

- `npm run typecheck`: passed.
- `npm run test:codex-api-key-scope`: 5 passed.
- `node --test src/utils/codexApiServiceCompatibility.test.ts`: 2 passed.
- `api_key_usage_stats_are_isolated_by_time_window`: 1 passed.
- `custom_api_key_scope_filters_duplicates_and_updates_manifest_scope`: 1
  passed.
- `cargo fmt --check` under the current local Rust toolchain reports
  formatting differences in pre-existing upstream 1.3.0 files. It did not
  modify sources and is not treated as a functional test failure.

The 1.3.0 local package must reuse the checked-in sidecar binary because Go is
not installed. Record its installer names, checksums, installation result, and
runtime port check here after packaging; do not include credentials.

Local 1.3.0 package verification:

```text
target/release/bundle/nsis/Cockpit Tools_1.3.0_x64-setup.exe
size: 27360759 bytes
time: 2026-07-13 10:24:20 +08:00
sha256: A1357C483ABB6E9071400B0112F9983E5153CD7D40F6D0C0906F6C482C36DE3C

target/release/bundle/msi/Cockpit Tools_1.3.0_x64_en-US.msi
size: 37269504 bytes
time: 2026-07-13 10:23:20 +08:00
sha256: A3B3BA20255F7702040D9D99593756D22AFABF539C047365766D459453987870
```

- The compatible checked-in Windows sidecar was reused without a Go rebuild.
- The NSIS silent installer exited with `0`.
- Installed product version: `1.3.0`.
- Installed application SHA-256:
  `F1B0861E725E5E4C306073F49D18307E6698364039772CACC26F93110CDD285F`.
- Restarted application process ID: `65972`.
- API sidecar port `54548`: listening.
- Unauthenticated `GET /v1/models`: HTTP `401`, confirming the restarted
  API service is reachable and enforces authentication without exposing a
  client API Key in the verification record.

## Upstream 1.3.2 Sync And Archive

Sync completed on 2026-07-15:

- Fetched official `origin/main` at `0a5a1d46` and release tag `v1.3.2` at
  `a84a97cb`.
- Merged `origin/main` into `feature/api-key-routing-usage` as `b6c2b054`
  (`merge: sync upstream 1.3.2`).
- Adopted version `1.3.2` consistently in the npm, Cargo, and Tauri metadata.
- Preserved the pre-merge line-ending-only worktree state in the stash named
  `pre-v1.3.2 worktree line-ending state`; those files had no text diff.
- Resolved the only content conflicts in
  `src-tauri/src/modules/codex_local_access.rs` and `src/types/codex.ts`.

The merged branch retains all downstream behavior:

- Per-client-Key inherited or explicit Codex account pools.
- Ordered per-Key account priorities, immediate priority hot reload, existing
  session affinity, and fallback to the next usable account.
- Per-Key request, token, success-rate, and estimated-cost reporting.
- Local calendar periods: today begins at local midnight, the week begins on
  Monday, and the month begins on its first day.
- Spark quota visibility, Spark model selection, and a complete local Codex
  profile model catalog containing `gpt-5.3-codex-spark`.

The branch also retains official 1.3.2 behavior, including random routing,
immediate SSE response configuration, current model pricing and historical
repricing, Codex local import changes, and the Windsurf-to-Devin path update.
During the merge, official historical-repricing code still referenced rolling
24-hour, 7-day, and 30-day constants removed by the calendar-period fix. It
was changed to use `calendar_stats_window_starts` so repriced costs use the
same day/week/month boundaries as the visible Key statistics.

GitHub thread `#1470` records the maintainer response that the contribution
had merge conflicts and required time to review and validate. This sync
addresses the requested first step: the contribution is now based on the
latest official code and has fresh verification evidence. Do not merge PR
`#1540` wholesale into this branch; its useful Spark quota visibility change
is already present here, while its broader history is unrelated to the Key
routing feature and it does not solve Codex profile model-catalog discovery.

Verification after the 1.3.2 merge:

- `npm run typecheck`: passed.
- `go test ./...` in `sidecars/cockpit-cliproxy`: passed.
- `custom_api_key_scope_filters_duplicates_and_updates_manifest_scope`: passed.
- `api_key_priority_queue_respects_custom_scope_session_affinity_and_fallbacks`:
  passed.
- `api_key_usage_stats_are_isolated_by_time_window`: passed.
- `local_access_takeover_writes_a_complete_model_catalog`: passed.
- `random_routing_serializes_and_keeps_all_candidates`: passed.
- `cargo fmt --all -- --check` reports formatting differences in official
  1.3.2 files. It did not modify the worktree and is not a functional failure.
- Tauri release build with `COCKPIT_SKIP_CLIPROXY_BUILD=1` and
  `src-tauri/tauri.ci.conf.json`: passed.

Go `1.26.5` is now available on this machine. The Windows sidecar was rebuilt
from the merged source instead of reusing the older binary:

```text
sidecars/cockpit-cliproxy/bin/cockpit-cliproxy-x86_64-pc-windows-msvc.exe
size: 19335680 bytes
time: 2026-07-15 10:10:12 +08:00
sha256: E29B90B89D0242D8BFB4A568411A9E6F4BF967BC884EC96BE57D8DC0F0DD786B
```

Packaged artifacts:

```text
target/release/bundle/nsis/Cockpit Tools_1.3.2_x64-setup.exe
size: 27919295 bytes
time: 2026-07-15 10:26:22 +08:00
sha256: FA6F70FCA47D1E74D03C848B91E1E8247E66D99204262C1E685B542F715C57AD

target/release/bundle/msi/Cockpit Tools_1.3.2_x64_en-US.msi
size: 38150144 bytes
time: 2026-07-15 10:25:30 +08:00
sha256: 328A55A13C4A3BF5C0C0148FF71E45BBE06D444F41B8DA90372B17596143FC00

target/release/cockpit-tools.exe
size: 83955200 bytes
time: 2026-07-15 10:25:14 +08:00
sha256: C927C4C260F17C180A9B249BF4BAB1BEF6574CA8ABC0311F1AC0C38CC80BE69E
```

The 1.3.2 package was built but not installed during this archive operation,
so the running Cockpit Tools process and active API sessions were not stopped.

The full downstream delta from official commit `0a5a1d46` is archived at:

```text
docs/maintenance/patches/cockpit-tools-1.3.2-api-key-routing-usage.patch
```

Restore that snapshot on the exact official base with:

```powershell
git switch -c recover/api-key-routing-1.3.2 0a5a1d46
git apply --3way docs/maintenance/patches/cockpit-tools-1.3.2-api-key-routing-usage.patch
```

Preferred recovery remains fetching the complete feature branch from the
personal fork because it preserves commit history and merge decisions.

## 1.3.3 Responses Lite Tool Compatibility Fix

On 2026-07-15 the first custom 1.3.3 install was rolled back to 1.3.2 after
Responses Lite requests failed with an unsupported `image_generation` tool.
The failure was reproduced in the sidecar SSE path with `gpt-5.6-sol`.
Official 1.3.3 had removed the old image-generation disable projection, while
the sidecar injected the hosted image tool after payload configuration. The
Responses Lite endpoint accepts only function tools, custom tools, and
client-executed `tool_search` tools.

The fix is present in:

- `src-tauri/src/modules/codex_protocol.rs`
- `src-tauri/src/modules/codex_local_access.rs`
- `sidecars/cockpit-cliproxy/cdk/CLIProxyAPI/internal/runtime/executor/helps/payload_helpers.go`
- `sidecars/cockpit-cliproxy/cdk/CLIProxyAPI/internal/runtime/executor/codex_executor.go`
- `sidecars/cockpit-cliproxy/cdk/CLIProxyAPI/internal/runtime/executor/codex_websockets_executor.go`
- `sidecars/cockpit-cliproxy/cdk/CLIProxyAPI/sdk/api/handlers/openai/openai_responses_handlers.go`

Lite protection now detects the feature header by presence, including an
empty value, and also recognizes the catalog models `gpt-5.6-sol`,
`gpt-5.6-terra`, and `gpt-5.6-luna`. It filters top-level, additional-tool,
nested-response, Chat Completions conversion, and WebSocket declarations;
only `function`, `custom`, and `tool_search` with `execution: client` remain.
Hosted `image_generation` injection is blocked, while ordinary models retain
their official image-generation behavior and client function tools remain
available.

Verification:

- `npm run typecheck`: passed.
- `go test ./...` in `sidecars/cockpit-cliproxy`: passed.
- Affected nested CLIProxyAPI packages (`helps`, `executor`, and OpenAI
  handlers): passed.
- Rust Responses Lite regression set: 7 passed, including Chat Completions,
  HTTP, WebSocket follow-up/model switching, and hosted-tool preservation.
- The full Rust library run reached 584 passed; 11 unrelated existing Windows
  tests failed because of shared config assertions and temporary-file locks.
- Release build with `src-tauri/tauri.ci.conf.json` and
  `bundle.createUpdaterArtifacts=false`: passed.
- Installed API service: version `1.3.3`, port `54548`, unauthenticated
  `/v1/models` returns 401. Authenticated model listing includes Sol, Terra,
  Luna, and Spark. Authenticated Sol requests with no tools and with a valid
  mixed tool declaration both returned HTTP 200 without the prior error.

Final artifacts:

```text
target/release/bundle/nsis/Cockpit Tools_1.3.3_x64-setup.exe
sha256: CC6AF19487C153E99A1FBF45FD3A5F3C0A4C6AA8C2D6030F877444229114B42F

target/release/bundle/msi/Cockpit Tools_1.3.3_x64_en-US.msi
sha256: A25AC6025FF009C1BDD55A490FD3A3F467978B279B5B03DE568D7E00C9FC895E

sidecars/cockpit-cliproxy/bin/cockpit-cliproxy-x86_64-pc-windows-msvc.exe
sha256: 8F11EBC9519BF6A1D5FF0E6F9DE00199FA622B2EF81384E5D97F3A87C03B3C95
```

Before installation, the prior 1.3.2 program directory was backed up to:

```text
target/install-backups/pre-1.3.3-lite-fix-20260715-122735
```

To roll back, exit Cockpit Tools and restore that directory to
`%LOCALAPPDATA%\Cockpit Tools`, or reinstall the previously retained 1.3.2
installer. User configuration and account data were not deleted or replaced.

## PR #1577 Locale Check Repair

On 2026-07-15, the first CI run for the Responses Lite compatibility PR failed
before compilation. Build Matrix run `29389549341` failed in `Preflight` at
`node scripts/check_locales.cjs`:

- `en-US.json` was the 4,999-key baseline.
- `en.json` and `zh-CN.json` contained five account-priority keys that were
  absent from the baseline.
- Fourteen other locale files were missing 17 API Service statistics and
  account-scope keys.
- No Rust, Go, or sidecar build failure was involved; all build jobs were
  skipped after Preflight.

The locale set was synchronized in commit `e32020a1` (`fix(i18n): sync API
key routing locale keys`) on branch `fix/responses-lite-after-1470`. All 18
locale files now contain 5,004 matching keys, including localized account
scope, account priority, usage range, and calendar statistics labels. The
commit was pushed to:

```text
fork/fix/responses-lite-after-1470
https://github.com/jlcodes99/cockpit-tools/pull/1577
```

The recovery patch is archived at:

```text
docs/maintenance/patches/0001-fix-i18n-sync-API-key-routing-locale-keys.patch
```

To replay only this locale repair on the PR parent:

```powershell
git switch -c recover/responses-lite-locale 2622a3fc
git am docs/maintenance/patches/0001-fix-i18n-sync-API-key-routing-locale-keys.patch
```

Verification for `e32020a1`:

- `node scripts/check_locales.cjs`: passed; all 18 files match and no
  blocking English-value reuse was reported.
- `npm run typecheck`: passed.
- `custom_api_key_scope_filters_duplicates_and_updates_manifest_scope`: 1
  passed. The older similarly named command ran 0 tests and is not valid
  regression evidence.
- GitHub Actions Build Matrix run `29390209527`: Preflight and all six
  platform builds passed.
- GitHub Actions CodeQL run `29390209499`: passed.

After the push, the local worktree was returned to
`feature/api-key-routing-usage` at `667a390b`. The only remaining local
modification is the pre-existing line-ending-only state of
`src-tauri/Cargo.toml`; it was not included in the locale commit or patch.

## v1.3.4 Sync and API Service Corrections

Sync performed on 2026-07-15:

- Pre-merge downstream HEAD: `2742e6b0`.
- Official head: `466e3f6d`; release tag: `v1.3.4` at `2d8f0fc2`.
- Official API Key routing merge: `9bb510d5` (PR #1470).
- Backup branch:
  `backup/pre-origin-main-1.3.4-sync-20260715-161817`.
- External byte backup:
  `%LOCALAPPDATA%\Temp\cockpit-tools-pre-1.3.4-20260715-161817`.
- Superseded downstream Responses Lite source changes were retired in
  `14041290` before the merge. The maintenance document and locale recovery
  patch were retained.
- Merge commit: `2c9c4c5b`. The merge completed without conflicts and kept the
  local untracked `docs/prototypes/` directory outside version control.

Two post-merge defects were corrected:

1. API service speed changes now rely on CLIProxyAPI's config watcher. The
   hot-reloadable `payload` defaults are excluded from the process-restart
   fingerprint, so changing Standard/Fast affects new requests without killing
   an in-flight stream. Port, bind address, account routing, and other process
   configuration changes still trigger the existing restart path.
2. Existing request-log databases now add the `service_tier` column in place
   when opened with the current schema. Existing rows remain intact and receive
   the empty default value.
3. Quota-pool summaries now aggregate the real quota windows returned for each
   account. A 10,080-minute primary window is displayed as weekly instead of
   being hard-coded as `5h`. Legacy 5-hour plus weekly data remains supported,
   as do missing metadata, exhausted 0% windows, and mixed old/new account
   pools.

Live state before installation confirmed that the saved API service speed is
`fast` and the running sidecar config injects `service_tier=priority`. The
existing 1.3.3 request database did not contain the `service_tier` column,
which reproduced the migration defect without exposing request content or
credentials.

Focused verification before packaging:

- `npm run typecheck`: passed.
- Quota-pool regression file via temporary `tsx`: 6 passed.
- Existing Codex quota test: 1 passed.
- Existing API Key scope tests: 5 passed.
- Current Rust API Key scope test: 1 passed.
- Rust `codex_local_access` module: 179 passed.
- New speed-fingerprint and log-schema migration tests: passed.
- Cockpit Go sidecar wrapper: passed.
- CLIProxyAPI watcher, Codex executor, and OpenAI Responses packages: passed.
- The complete Rust library baseline ran 603 tests. Failures were confined to
  unchanged `origin/main` files (`codex_account`, Windows SQLite cleanup, and
  file timestamp tests); the relevant files and Cargo metadata have identical
  Git object hashes to `origin/main`.
- `cargo fmt --check` reports broad formatting differences already present in
  `origin/main`; do not bulk-format unrelated source during this maintenance
  release.

Release build with a freshly rebuilt Go sidecar and
`src-tauri/tauri.ci.conf.json` passed. Artifacts:

```text
target/release/bundle/nsis/Cockpit Tools_1.3.4_x64-setup.exe
size: 27974121 bytes
sha256: 2FA9C171C981D8D1B089B77E2D02559681D0FB75A7A1874681119C1644E235E8

target/release/bundle/msi/Cockpit Tools_1.3.4_x64_en-US.msi
size: 38150144 bytes
sha256: 6188A7B42B693DC2720D0E7469C2CA4955CA5181A0936E1E50DD493D29B5C60F

sidecars/cockpit-cliproxy/bin/cockpit-cliproxy-x86_64-pc-windows-msvc.exe
size: 19347456 bytes
sha256: FA319B0416BB690B4FD16A0DDDFC9D71CE3CCE7492A707A9C0D7D58EC81ECB9D
```

The pre-install program-directory backup is:

```text
target/install-backups/pre-1.3.4-sync-20260715-165950
```

It contains the same 100 files as the installed directory at backup time.
Account data under `%USERPROFILE%\.antigravity_cockpit` is separate and is not
replaced by the installer.

### Deferred Usage Activity Design

No activity-dashboard code is included in this release. The agreed discussion
direction is:

- Place the activity view at the top of the existing Codex API Service page,
  not in a new product shell.
- Add a compact selector to the daily Token activity heatmap for `all client
  Keys` or one specific client Key.
- Keep the Token trend to one total-Token series rather than simultaneous input,
  output, and cached lines.
- Preserve Key-level and account-level attribution in the data model so later
  provider expansion is possible, but do not claim cross-provider support until
  those providers expose comparable usage events.

This deferred design must not be bundled into the fast-mode/quota-window bugfix
commit or its pull request.

## v1.3.5 Sync, Remaining Delta, and Updater Policy

Sync and archive completed on 2026-07-16:

- Pre-merge downstream HEAD: `b6bc7f5b`.
- Official release tag: `v1.3.5` at `2c6412c0`.
- Official head after the Homebrew update: `24617f5c`.
- Backup branch:
  `backup/pre-origin-main-1.3.5-sync-20260716-100132`.
- Merge commit: `dca4239f` (`merge: sync upstream 1.3.5`).
- The only textual conflict was this maintenance document. The resolution kept
  the official PR #1577 record and the downstream v1.3.4 correction archive.
- The untracked `docs/prototypes/` directory was not deleted or committed.

Official 1.3.5 now includes the previously contributed Responses Lite fix from
PR #1577, including Lite search forwarding, Responses WebSocket support, and
hosted-tool filtering. Official 1.3.4 already included the core API Key account
pool, ordered priority, per-Key usage statistics, calendar periods, and Spark
support from PR #1470.

The remaining functional downstream delta from `origin/main` is now limited to
two correction groups:

1. API service Standard/Fast changes hot-reload sidecar payload defaults without
   restarting the process or interrupting active streams. Existing request-log
   databases also migrate the `service_tier` column in place.
2. Quota-pool UI components render the actual windows reported by each account,
   so a weekly-only primary window is not mislabeled as `5h`.

Those corrections are proposed upstream in PR #1587 but are not present in
official head `24617f5c`. Maintenance-only differences also remain in this
document and the recovery patches; they do not affect the packaged application.

The focused recovery patch for the remaining code delta is:

```text
docs/maintenance/patches/cockpit-tools-1.3.5-api-service-fixes.patch
```

It is based on official head `24617f5c`. Preferred recovery remains fetching
the complete maintained branch from the personal fork. To replay only the two
correction groups, make the patch available in a checkout based on `24617f5c`
and run:

```powershell
git apply --3way docs/maintenance/patches/cockpit-tools-1.3.5-api-service-fixes.patch
```

Verification after the merge:

- `npm run typecheck`: passed, including the production build preflight.
- `node scripts/check_locales.cjs`: passed; all 18 locale files contain 5,012
  matching keys.
- API Key account-scope tests: 5 passed.
- Quota-window and quota-error tests: 9 passed.
- API service compatibility tests: 2 passed.
- Rust `codex_local_access` module: 186 passed.
- Cockpit sidecar wrapper `go test ./...`: passed.
- CLIProxyAPI watcher, executor, payload helper, OpenAI handler, and auth
  classification packages: passed.
- Release build completed with Go sidecar compilation enabled and
  `src-tauri/tauri.ci.conf.json`; updater artifacts remained disabled because
  no Tauri signing private key is configured.

Release artifacts:

```text
target/release/bundle/nsis/Cockpit Tools_1.3.5_x64-setup.exe
size: 28051709 bytes
time: 2026-07-16 10:26:29 +08:00
sha256: 6DED548F3EE0BF5FBC0EBEC126A8CFA3FEC326A9E2E0DB038ACCEA69A505AA66

target/release/bundle/msi/Cockpit Tools_1.3.5_x64_en-US.msi
size: 38191104 bytes
time: 2026-07-16 10:25:34 +08:00
sha256: 3CD973C23F767F991CB72819B4EC599DDFD6DBAB6BDEB9C564DBDB299DD5EE32

target/release/cockpit-tools.exe
size: 83884032 bytes
time: 2026-07-16 10:25:19 +08:00
sha256: 186E6DB8CF841F57E200E65772C9953E788020A36E62D24D577344F2A75BA060

sidecars/cockpit-cliproxy/bin/cockpit-cliproxy-x86_64-pc-windows-msvc.exe
size: 19633152 bytes
time: 2026-07-16 10:13:05 +08:00
sha256: 53FD4EFE64B6B3A8629D66899D655F4A69C8A12EECF2269EDBEC604315BE9193
```

The package was built but not installed during this sync. The running installed
application remains official/custom 1.3.4 with the prior sidecar, so merely
restarting it does not activate the 1.3.5 Lite fixes.

Do not use the in-app official updater while PR #1587 is absent from the target
release if the hot-reload and accurate-window corrections must be retained. The
official updater replaces the application binaries with the official build,
although account and API service data remain on disk. Continue using a locally
merged package for releases until the official branch contains those remaining
code changes. Once a released official version includes PR #1587 and the
downstream diff contains only maintenance documentation or patches, normal
in-app updates can replace this manual packaging workflow.

The build emitted existing unused/dead-code warnings, a Vite chunk-size warning,
and the known Tauri `__TAURI_BUNDLE_TYPE` patch warning. Both unsigned local
installers were still produced successfully. After the build,
`src-tauri/Cargo.toml` briefly appeared modified with no text diff; its worktree
and index object hashes were identical, and refreshing the index cleared the
status without changing file content.
