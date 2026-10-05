# pnpm 11 & 12 test examples

Test fixtures for Dependabot's pnpm 11 and pnpm 12 support. Each numbered directory is an
independent project with a real pnpm-generated `pnpm-lock.yaml` and at least one
intentionally out-of-date dependency. Every directory has its own entry in
`.github/dependabot.yml`.

- pnpm 11 fixtures were generated with **pnpm 11.28.2** (Node 22.17). Their lockfiles are a single YAML document.
- pnpm 12 fixtures were generated with **pnpm 12.9.1** (`13-*` uses **12.4.1**). Their lockfiles are **two YAML documents**: first an env document with `packageManagerDependencies`, then the regular lockfile.

At the time of writing, dependabot-core lists pnpm 7–11 in `PnpmPackageManager::SUPPORTED_VERSIONS`, ships pnpm 11.17 in the image, and does not read `devEngines.packageManager`.

## Feature coverage (01–12)

| Directory | pnpm | Setup | What it exercises |
| --- | --- | --- | --- |
| `01-pnpm11-basic` | 11 | `packageManager`, lodash 4.17.20, is-odd 3.0.0 | Baseline version updates and a lodash security update. |
| `02-pnpm11-workspace-settings` | 11 | `allowBuilds` (esbuild) and `overrides` in `pnpm-workspace.yaml` | `strictDepBuilds`/`allowBuilds` (pnpm 11 defaults) don't block updates; overrides are respected. |
| `03-pnpm11-legacy-config` | 11 | `pnpm.overrides` in package.json and non-auth settings in `.npmrc` | pnpm 11 ignores both (the lockfile has no `overrides:`); Dependabot shouldn't rely on them. |
| `04-pnpm11-monorepo-named-catalogs` | 11 | `packages/*`, default `catalog:` and named `catalog:legacy`, `workspace:*` | Catalog updates are written back to `pnpm-workspace.yaml`. |
| `05-pnpm11-engines-only` | 11 | No `packageManager`; `engines.pnpm: ">=11 <12"` | Version detection falls back to `engines`. |
| `06-pnpm12-basic` | 12 | `packageManager: pnpm@12.9.1` | Core pnpm 12 check; PRs must keep the env lockfile document intact. |
| `07-pnpm12-devengines` | 12 | Only `devEngines.packageManager` (`^12.0.0`, `onFail: download`) | Dependabot doesn't read devEngines and falls back to lockfile inference. |
| `08-pnpm12-monorepo-catalog` | 12 | Workspace, `catalog:` and `workspace:*` | Workspace and catalog handling under pnpm 12. |
| `09-pnpm12-min-release-age` | 12 | `minimumReleaseAge: 4320`, `minimumReleaseAgeExclude: [is-odd]`, plus Dependabot `cooldown` | Release-age gate and cooldown interaction. |
| `10-pnpm12-git-dependency` | 12 | `github:jonschlinkert/is-odd#3.0.0` (3.0.1 exists) | pnpm 12 canonical HTTPS/codeload git resolution; git tag update. |
| `11-pnpm12-allowbuilds-overrides` | 12 | Same as `02` under pnpm 12 | pnpm 12's stricter settings validation. |
| `12-pnpm12-unrecognized-setting` ⚠️ | 12 | Misspelled `minimumReleaseAg` in `pnpm-workspace.yaml` | **Expected failure**: `ERR_PNPM_UNRECOGNIZED_WORKSPACE_SETTINGS`. Should surface as a clear user error. |

## Sentry-informed reproductions (13–20)

Each directory reproduces a high-volume pnpm failure seen in Sentry (`github/deltaforce`). ⚠️ means a failure is expected today; locally verified error codes are listed.

| Directory | pnpm | Setup | Sentry signal |
| --- | --- | --- | --- |
| `13-pnpm12-self-switch-download` ⚠️ | 12.4.1 | Pinned to an older pnpm 12 release | `Could not download the pnpm 12.x binary … @pnpm/exe.linux-x64 … fetch failed`: pnpm 11 in the image tries to self-switch to the pinned 12.x (~16k events/30d across 12.0–12.4). |
| `14-pnpm11-comment-only-workspace-yaml` ⚠️ | 11 | `pnpm-workspace.yaml` that contains only comments (parses to `nil`) | `NoMethodError: undefined method '[]' for nil` in `FileFetcher#fetch_pnpm_workspace_package_jsons`: [DELTAFORCE-1F1W](https://github.sentry.io/issues/DELTAFORCE-1F1W). pnpm itself is fine. |
| `15-pnpm11-settings-only-workspace-yaml` | 11 | `pnpm-workspace.yaml` with settings but no `packages` (pnpm 11 style) | `ERR_PNPM_INVALID_WORKSPACE_CONFIGURATION packages field missing or empty`. pnpm itself is fine. |
| `16-pnpm12-min-release-age-strict` ⚠️ | 12 | `minimumReleaseAge` of 10 years, `minimumReleaseAgeStrict: true`, `trustLockfile: true` | `ERR_PNPM_STRICT_MIN_RELEASE_AGE_REQUIRES_SAVE`: strict mode can't be combined with Dependabot's `--no-save` (top pnpm error code). Verified locally. |
| `17-pnpm11-url-tarball-no-integrity` ⚠️ | 11 | `xlsx` from `https://cdn.sheetjs.com/...tgz`, `integrity` stripped from the lockfile | `ERR_PNPM_MISSING_TARBALL_INTEGRITY`. Verified locally. |
| `18-pnpm11-runtime-pin` ⚠️ | 11 | `devEngines.runtime` node `>=26.0.0`, `onFail: error` | `ERR_PNPM_BAD_RUNTIME_VERSION` / Unsupported engine ([DELTAFORCE-1JSS](https://github.sentry.io/issues/DELTAFORCE-1JSS)). Verified locally. |
| `19-pnpm11-overrides-lockfile-mismatch` ⚠️ | 11 | `overrides` in `pnpm-workspace.yaml` (`is-number: 7.0.0`) differ from the lockfile (`6.0.0`) | `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH`. Verified locally with `--frozen-lockfile`. |
| `20-pnpm12-transitive-security` | 12 | `express@4.17.1` → vulnerable `path-to-regexp@0.1.7`, `qs@6.7.0`, `send@0.17.1`, … | Transitive security updates under pnpm 12 (`ERR_PNPM_UPDATE_VERSION_ON_INDIRECT_DEP`). |

## Regenerating a lockfile

```sh
# pnpm 11 requires Node >= 22.13
pnpm_config_manage_package_manager_versions=false pnpm install --lockfile-only --ignore-scripts
```

Re-apply the intentional breakage afterwards. In `12`, add the bad key after generating. In `16`, generate with `minimumReleaseAge: 0`. In `17`, strip `integrity`. In `18`, temporarily set `onFail: ignore`. In `19`, change the override after generating.
