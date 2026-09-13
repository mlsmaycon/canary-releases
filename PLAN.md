# Canary releases for NetBird: investigation and test plan

Goal: build **and sign** Windows and macOS packages from NetBird pull requests
(and optionally from `main`) using the existing GoReleaser + `sign-pipelines`
machinery, without touching what a real `vX.Y.Z` release does today.

This document records what the current pipeline does (section 1), the options
for producing a canary from a non-tag ref, including moving the signing jobs
into the main repository (section 2), and a phased test implementation for this
repository (section 3). Section 4 sketches what would change in
`netbirdio/netbird` and `netbirdio/sign-pipelines` once an option is chosen, and
section 5 lists the decisions that are still open.

Investigated at: `netbirdio/netbird@7914010` (main), `netbirdio/sign-pipelines@ed09e23` (= `v0.1.8`, the version `release.yml` pins), `netbirdio/packages-proxy@b66cbdf` (main), `netbirdio/shared-actions` (`parse-semver`).

> **Decision (2026-09-13): Option A** — canaries are prerelease tags on
> `netbirdio/netbird`, built and signed by the existing pipeline. Sections
> 2.7 and 2.8 record the two checks that decision depended on (tag format
> and `packages-proxy` compatibility); section 3 is scoped to A.

---

## 1. What exists today

### 1.1 `netbird` Release workflow (`.github/workflows/release.yml`)

Triggers: `push` tags `v*`, `push` to `main` / `release-*`, and `pull_request`.
GoReleaser `v2.16.0` via `goreleaser-action`.

| Job | Runner | Config | Notes |
| --- | --- | --- | --- |
| `release` | ubuntu-24.04-8-core | `.goreleaser.yaml` | client (all OS), server components, nfpm deb/rpm, docker images, brew, deb/yum uploads |
| `release_ui` | ubuntu-latest | `.goreleaser_ui.yaml` | GTK4 Linux UI + Windows UI cross-compiled with mingw / llvm-mingw (CGO) |
| `release_ui_gtk3` | ubuntu-22.04 | `.goreleaser_ui_gtk3.yaml` | legacy Linux UI |
| `release_ui_darwin` | macos-latest | `.goreleaser_ui_darwin.yaml` | macOS UI, amd64 + arm64 + universal |
| `test_windows_installer` | windows-2022 | – | builds unsigned NSIS + MSI from the `release` and `release-ui` artifacts, sanity only |
| `comment_release_artifacts` | ubuntu | – | PR comment with artifact links (same-repo PRs only) |
| `trigger_signer` | ubuntu | – | **tags only**: `workflow_dispatch` of `sign-pipelines` `Sign bin and installer` @ `SIGN_PIPE_VER` with `{tag: github.ref, skipRelease: false}` using `SIGN_GITHUB_TOKEN` |

Behaviour by ref type:

- **Non-tag refs (PRs, main, release-\*)**: `flags=--snapshot`. GoReleaser only
  builds; nothing is published. `dist/` is uploaded as workflow artifacts
  (`release`, `release-ui`, `release-ui-gtk3`, `release-ui-darwin`, plus
  `linux-packages` / `windows-packages` / `macos-packages`), retention 3–7
  days. amd64 Docker images are re-tagged `pr-N` / `main` / `sha-*` and pushed
  to GHCR for same-repo PRs and branch pushes. The snapshot version is
  GoReleaser's default `{{ .Version }}-SNAPSHOT-{{ .ShortCommit }}` (e.g.
  `0.71.4-SNAPSHOT-7914010`); no config sets `snapshot.version_template`.
- **Tag refs**: full `goreleaser release`. All four configs upload into the
  **same** GitHub release (`make_latest: false`, `prerelease: auto`). `SKIP_PUBLISH`
  becomes `false` only when the tag has **no prerelease suffix and the repo is
  `netbirdio/netbird`**; it gates brew, deb, yum uploads. `SKIP_DOCKER_PUSH` is
  `true` only for forks, so a prerelease tag on `netbirdio/netbird` still pushes
  `netbirdio/*:{version}` images to Docker Hub and GHCR (never `latest`).
- Version source: `netbirdio/shared-actions/actions/parse-semver` on `github.ref`
  (outputs `major`, `minor`, `patch`, `fullversion`, `prerelease`). Windows
  `.syso` version info uses `fullversion` (`X.Y.Z.0`).
- `sync-tag.yml` fans out to android/ios/dashboard on `v*` tags but only when
  `!contains(github.ref_name, '-')`, so prerelease tags are already ignored there.

### 1.2 `sign-pipelines` (`.github/workflows/sign.yml`, `v0.1.8`)

A `workflow_dispatch`-only workflow. Inputs:

| Input | Default | Meaning |
| --- | --- | --- |
| `tag` | `refs/tags/vX.Y.Z` | parsed with `parse-semver`; version = `fullversion` + `-prerelease` when present |
| `Repository` | `netbirdio/netbird` | "Custom name only if running tests" — used for checkout, asset download **and** asset upload |
| `skipRelease` | `true` | skip uploading signed assets to the release **and** skip pushing `.sig` files to `netbirdio/public-keys` |
| `skipWindowsRun` / `skipMacosRun` / `skipMacosCaskRun` | `false` | job switches |

What it needs from the **source repository at tag `v{version}`** (it does
`actions/checkout` of `Repository` at that ref):

- `go.mod` — reads the `github.com/wailsapp/wails/v3` version to install `wails3` and stage `client/MicrosoftEdgeWebview2Setup.exe`
- `client/installer.nsis` (references `ui\build\windows\icon.ico`, `ui\build\banner.bmp`, `..\LICENSE`, `..\dist\netbird_windows_{arch}\`, `..\client\ui\assets\netbird.png`)
- `client/netbird.wxs` (references `dist\netbird_windows_{arch}\{netbird.exe,netbird-ui.exe,wintun.dll}`, `client\ui\assets\netbird.png`)
- `client/ui/build/darwin/icons.icns`
- `release_files/darwin-ui-installer.sh`, `release_files/darwin-ui-uninstaller.sh`, `release_files/darwin_pkg/{preinstall,postinstall}`
- `client/ui/netbird-ui.rb.tmpl` (cask; only when `Repository == netbirdio/netbird`)

What it downloads from the **GitHub release `v{version}` of `Repository`**
(exact names, via `robinraju/release-downloader` with `CI_GITHUB_TOKEN`):

| Asset | Produced by |
| --- | --- |
| `netbird_{V}_windows_{amd64,arm64}.tar.gz` | `.goreleaser.yaml`, build `netbird`, default archive template |
| `netbird-ui-windows_{V}_windows_{amd64,arm64}.tar.gz` | `.goreleaser_ui.yaml`, archive id `windows-arch` |
| `netbird_{V}_darwin_{amd64,arm64,all}.tar.gz` | `.goreleaser.yaml`, `universal_binaries` adds `all` |
| `netbird-ui_{V}_darwin_{amd64,arm64,all}.tar.gz` | `.goreleaser_ui_darwin.yaml` |

What it produces:

| Output | Where |
| --- | --- |
| `netbird_{V}_windows_{arch}_signed.tar.gz`, `netbird_installer_{V}_windows_{arch}.exe`, `.msi` | release assets when `!skipRelease`; always as workflow artifacts `windows-signed-files-{arch}` (3 days) |
| `netbird-ui_{V}_darwin_{amd64,arm64,all}_signed.zip`, `netbird_{V}_darwin{,_amd64,_arm64}.pkg` | release assets when `!skipRelease`; always as workflow artifacts `macos-signed-files*` |
| `*.sig` for every exe/msi/pkg (NetBird artifact key, `client/cmd/signer`) | pushed to `netbirdio/public-keys` under `artifact-signatures/tag/v{V}/` unless `skipRelease` |
| cask `netbird-ui.rb` | pushed to `netbirdio/homebrew-tap` only when `!skipRelease && prerelease == ''` |

Signing mechanics: Windows uses AzureSignTool against Key Vault (`netbird.exe`,
`netbird-ui.exe`, the NSIS exe, the MSI; verified with `signtool verify /pa`).
macOS imports two Developer ID certificates, codesigns with a hard-coded SHA-1
identity, builds the pkg with `pkgbuild`/`productbuild` (`--version fullversion`),
notarizes both the zip and the pkg with `notarytool --wait`. Uploads use
`svenstaro/upload-release-action` with `overwrite: false`.

**Key observation.** Everything that must *not* happen for a canary is already
gated: `mark_release_latest`, cask push, and the choco dispatch require
`prerelease == ''`; `generate_cask` requires `Repository == netbirdio/netbird`;
brew/deb/yum require `SKIP_PUBLISH=false`, which requires no prerelease suffix.
Feeding the existing pipeline a **prerelease tag** already yields signed,
notarized Windows and macOS installers on a GitHub prerelease. The only
unguarded side effects of a prerelease tag are:

1. Docker Hub / GHCR pushes of `{version}`-tagged images (`SKIP_DOCKER_PUSH` is only `true` for forks).
2. `.sig` files pushed to `netbirdio/public-keys` (gated on `skipRelease` only, not on `prerelease`).
3. The tag and the prerelease itself living on `netbirdio/netbird`.

### 1.3 Constraints from the client

- The in-app updater downloads installers from hard-coded URLs
  (`client/internal/updater/installer/installer_run_{windows,darwin}.go`):
  `https://github.com/netbirdio/netbird/releases/download/v%version/...`, and
  signatures from `https://publickeys.netbird.io/artifact-signatures`. A canary
  published to **any other repository cannot be installed by the updater**. That
  is fine for "testers download and install manually", and it is the deciding
  factor if a future "canary update channel" is wanted.
- `version.IsDevelopmentVersion` only recognises `development`, `ci-*`, `dev-*`.
  A canary version like `0.71.5-canary.pr7450.3` is treated as a real version by
  management and the dashboard, which is what we want for testing.
- MSI `ProductVersion` is `fullversion` only (`X.Y.Z`); `MajorUpgrade` has
  `AllowSameVersionUpgrades='yes'` and a downgrade error. So the canary base
  version must be **≥ the installed stable** (otherwise the MSI refuses to
  install), and the eventual real `X.Y.Z` installs over a canary `X.Y.Z-canary.*`
  fine. `CFBundleVersion` strips the prerelease suffix the same way. The NSIS
  `APPVER` embeds `github.run_id` and is unique per run.

### 1.4 Side findings (not needed for canaries)

- `sign.yml` "Copy install files" references `env.PackageWorkdirARM`, which is
  never defined (the `cp` lands in `dist/`; harmless leftover).
- `sign.yml` `sign_windows` hard-codes `go-version: '1.24'` while netbird's
  `go.mod` requires 1.26; it works only through Go's toolchain auto-download.
- `actions/setup-go@v5` (sign.yml, release.yml `test_windows_installer`) and
  `actions/setup-node@v4` (release.yml `release_ui`, `release_ui_darwin`) are
  not SHA-pinned while everything else is.

---

## 2. Options for producing a canary from a PR or `main`

All options share one requirement inherited from `sign.yml`: the artifacts must
sit on a GitHub release whose tag is `v<semver>` and whose assets follow the
table in 1.2, and the source tree must be checkable-out at that tag.

### Option A — prerelease tag on `netbirdio/netbird`, pipeline untouched (chosen)

Push `v0.71.5-canary.pr7450.3` at the PR head (by hand, or from a small
`canary.yml` that runs on a `canary` label / `workflow_dispatch` and pushes the
tag with a PAT so `push: tags` fires). The existing Release workflow builds,
publishes a prerelease, and dispatches the signer.

- **+** Zero pipeline changes; identical to a real release; updater-compatible (a real canary channel later).
- **−** Tags and prereleases accumulate on the public repo (needs a cleanup job); Docker Hub gets canary image tags; `.sig` files for canaries land in `public-keys`; changelog noise; whoever can create tags can trigger signing.

### Option B — separate canary repository as the release target

A canary job in netbird pushes the PR head to a canary repository **as the tag**
(`git push <canary-repo> HEAD:refs/tags/v…`, which carries the netbird tree so
`sign.yml` can check it out), runs GoReleaser with `release.github.owner/name`
templated from env so the release is created there, and dispatches
`sign-pipelines` with `Repository: <canary-repo>`. `sign.yml` already uses
`Repository` for checkout, download and upload, and the cask / latest jobs skip
non-netbird repositories.

- **+** `netbirdio/netbird` stays clean; the canary repo can be wiped at will; `sign-pipelines` needs at most the `.sig` gate (2.x below).
- **−** Needs a token with `contents: write` on the canary repo in netbird's secrets; `CI_GITHUB_TOKEN` in sign-pipelines must be able to read and write releases there; the updater cannot install these builds; first tree push is large (subsequent pushes are incremental).

### Option C — hand over workflow artifacts, no GitHub release

PR snapshot builds already upload `dist/`. Extend `sign.yml` with a
`source: release|workflow-run` mode (`run_id`, `sha`, `version` inputs),
download via `actions/download-artifact` with `repository`/`run-id`/`github-token`,
check out `Repository@sha`, and publish signed installers as workflow artifacts.

- **+** No tags or releases anywhere; reuses the build that already runs on every PR.
- **−** Real changes to `sign.yml` (version composition, download, upload paths); artifacts expire in 3–7 days; testers need a GitHub login to download; exercises the least GoReleaser behaviour.

### Option D — rolling canary per PR (`mode: replace`)

A variant of A or B: one moving tag per target (`v0.71.5-canary.pr7450`,
`v0.71.5-canary.main`), GoReleaser `release.replace_existing_artifacts: true`
and `release.mode: replace` (release notes), signer re-uploads.

- **+** Bounded pollution: one release per PR.
- **−** Force-moved tags; `sign.yml` uploads with `overwrite: false` and would collide on the second run; build/sign races need a concurrency group.

### Option E — bring signing into `netbirdio/netbird`

A–D keep `sign-pipelines` as a separate repository and differ only in how the
build hands artifacts to it. E removes the hand-off: the Windows and macOS
signing jobs move into netbird (as jobs in `release.yml` / `canary.yml`, or a
`workflow_call` reusable workflow that both call), run **after** the build jobs
in the same workflow run, and take their inputs from the `dist/` **workflow
artifacts** with `needs` + `download-artifact`. The GitHub release and its tag
are then only needed for *distribution*, not for the hand-off, and only the
final upload step touches it.

What it removes:

- the cross-repo `workflow_dispatch`, `SIGN_PIPE_VER` pinning, `SIGN_GITHUB_TOKEN`, and the `CI_GITHUB_TOKEN` round trip (download-by-exact-name from the release, re-upload with `overwrite: false`);
- the `Repository` test hook and the duplicated `test_windows_installer` job (it becomes the real installer build);
- the "tag must exist before the signer checks it out" ordering, so canaries no longer need a tag until the moment they are published, and Option C's only drawback (rewriting `sign.yml`) disappears because the rewrite is the point.

What it costs, and the mitigations:

- **Signing secrets in a public repository with PR workflows.** Put every
  signing job behind a GitHub **Environment** (`signing`) with *required
  reviewers* and *deployment branches and tags* limited to `main`,
  `release-*` and `v*`. Environment secrets are only handed to a job that
  references the environment, only after approval, and `pull_request` runs
  from forks never receive secrets at all. The approval click doubles as the
  "sign this canary?" trigger, which is the cost control we wanted anyway.
  Add `CODEOWNERS` for `.github/workflows/` and `.github/actions/`.
- **Azure Key Vault credentials** can stop being secrets: `azure/login` with
  OIDC (`id-token: write`) plus AzureSignTool's managed-identity mode
  (`-kvm`) removes `AZURE_CLIENT_SECRET` entirely; the federated credential
  is bound to the environment, so a leaked workflow cannot reuse it elsewhere.
- **Apple certificates** stay `.p12` secrets (no OIDC path). Notarization
  should move from `AC_USERNAME`/`AC_PASSWORD` to an App Store Connect API
  key (`notarytool --key`), which is revocable and scoped.
- **`NB_ARTIFACT_PRIV_KEY` is different in kind.** It signs the `.sig` files
  the in-app updater trusts (`publickeys.netbird.io`); a leaked code-signing
  certificate is revocable, a leaked artifact key means re-rooting the
  updater. Keep artifact signing out of the canary path completely, and
  either leave it in `sign-pipelines` for `v*` tags only or give it its own
  environment with a stricter reviewer set.
- macOS runner minutes move into netbird's Actions bill; the environment gate
  keeps them opt-in for PRs.

- **+** One workflow, one repo, no tokens between repos, canaries need no tag until publish, signing steps version together with `installer.nsis` / `netbird.wxs`.
- **−** Secret exposure surface moves to the main repo (mitigated above); a larger `release.yml`; the environment approval is a manual step per canary run unless the reviewer list is a bot-free team that accepts the noise.

### 2.7 Tag format for canaries

Proposed: `v<MAJOR>.<MINOR>.<PATCH>-canary.<source>.<run>`, e.g.
`v0.71.5-canary.pr7450.3` for a PR and `v0.71.5-canary.main.912` for `main`,
where `<MAJOR>.<MINOR>.<PATCH>` is the last non-canary tag reachable from the
commit with the patch bumped. No `+build` metadata: `sign.yml` composes
`fullversion-prerelease` and drops anything after `+`, so asset names would
stop matching; `+` is also awkward in URLs.

Semver 2.0.0 compliance: the pre-release part is a dot-separated list of
identifiers made of `[0-9A-Za-z-]` with no leading zeros in numeric ones
(§9); `canary`, `pr7450`, `3` all qualify. Precedence (§11) is compared
identifier by identifier, numerics numerically, alphanumerics in ASCII order,
so `canary.*` sorts **below** `rc.*` for the same base and above `alpha`/`beta`.
Verified against the parsers on the path (`go run` with
`hashicorp/go-version v1.7.0`, the module netbird's `version` package and the
updater use; the `parse-semver` bash regex; the `packages-proxy` njs regex):

| Check | Result |
| --- | --- |
| `0.71.5-canary.pr7450.3 > 0.71.4` | true → a canary is "newer" than the stable it was cut after, so MSI/pkg upgrades install and the update prompt stays quiet |
| `0.71.5-canary.pr7450.3 < 0.71.5` | true → the real release later replaces it and triggers the update prompt |
| `0.71.5-canary.pr7450.3 < 0.71.5-rc.1` | true |
| `canary.pr7450.3 < canary.pr7450.10` | true (numeric identifiers) |
| `parse-semver` on `refs/tags/v0.71.5-canary.pr7450.3` | `fullversion=0.71.5`, `prerelease=canary.pr7450.3` → `sign.yml` composes `0.71.5-canary.pr7450.3`, matching GoReleaser's `{{ .Version }}` |
| GoReleaser `prerelease: auto` | non-empty `.Prerelease` → GitHub prerelease, never `latest` |
| `packages-proxy` `parseRcVersion` | does not match `-canary.*` (regex requires `-rc`) |

Ordering across PRs (`pr123` vs `pr7450`) is lexical and meaningless, which is
fine: nothing compares two canaries from different PRs. Docker tags, nfpm
versions (`0.71.5~canary.pr7450.3` for deb) and `CFBundleShortVersionString`
all accept the string; `CFBundleVersion`, MSI `ProductVersion` and `pkgbuild
--version` use the stripped `0.71.5`.

### 2.8 `netbirdio/packages-proxy` compatibility

`pkgs.netbird.io` is an nginx cache in front of the GitHub API
(`docker/common.conf`, `docker/github_releases.js`). Three code paths matter:

| Endpoint family | Upstream call | Canary impact |
| --- | --- | --- |
| `/releases/latest/version`, `/windows/*`, `/macos/*`, `/wasm/client`, install scripts | `GET /repos/netbirdio/netbird/releases/latest` | **None.** GitHub's `latest` never points at a prerelease, GoReleaser sets `make_latest: false`, and only `sign-pipelines` `mark_release_latest` (gated on `prerelease == ''`) flips it. Consumers of this path: `version/update.go` (client update prompt), `management/server/instance/manager.go`, `release_files/install.sh`. |
| `/releases/rc/*`, `/windows/*/rc`, `/macos/*/rc`, `/wasm/client/rc` | `GET /repos/netbirdio/netbird/releases?per_page=30`, then `parseRcVersion` (`^v?(\d+)\.(\d+)\.(\d+)-rc(?:\.)?(\d+)?`) picks the highest | **Filter is safe** (canary tags do not match), **but the window is not**: only the 30 most recent releases are inspected. Once 30 canaries have been published after the newest `-rc` release, every rc endpoint returns `404 no rc release found` until the next RC is cut. |
| `/releases/<tag>`, `/wasm/client/<tag>/…` | `GET /repos/netbirdio/netbird/releases/tags/<tag>` | None; a canary tag simply becomes resolvable by name. |

So Option A does not break `latest` and does not mis-select a canary as an
RC, but it can **starve the rc channel** through the 30-release window.
Mitigations, in order of preference:

1. `packages-proxy`: raise `per_page=30` to `per_page=100` (GitHub's maximum)
   in `location = /releases/rc/raw`, and bump `subrequest_output_buffer_size`
   accordingly (the 30-release body already needs 4m). One-line change,
   widens the window to a level canaries are unlikely to fill between RCs.
2. Canary retention: delete canary releases and tags when the PR closes and
   sweep anything older than 14 days (already in the plan). Keeps the number
   of canaries in the window small regardless of 1.
3. Optional hardening in `github_releases.js`: when no RC is found in the
   page, follow the `Link: rel="next"` header once before returning 404.

Also noticed while reading `docker/common.conf`: the GitHub API token used
for the `Authorization` header is committed in clear text in that file (three
`proxy_set_header Authorization` lines). It should move to an environment
variable or a mounted file and be rotated; the value is not reproduced here.

### Comparison

| | A: tag on netbird | B: canary repo | C: artifacts | D: rolling | E: in-repo signing |
| --- | --- | --- | --- | --- | --- |
| netbird repo pollution | high | none | none | medium | as A/B (publish target is a separate choice) |
| `sign-pipelines` changes | `.sig` gate (optional) | `.sig` gate (optional) | significant | `overwrite` input | retired, or kept for `.sig` on `v*` only |
| netbird changes | none / tiny | canary workflow + templated `release.github` | none | as A/B | port sign jobs, environment, OIDC |
| Cross-repo tokens | `SIGN_GITHUB_TOKEN`, `CI_GITHUB_TOKEN` | + `CANARY_REPO_TOKEN` | `SIGN_GITHUB_TOKEN` + `actions: read` | as A/B | none |
| updater can install | yes | no | no | as A/B | as A/B |
| Cost control | tag permission | label / dispatch | label / dispatch | as A/B | environment approval |
| Cleanup | releases + tags + images | wipe repo | expiry | per-PR delete | as A/B |

**Recommendation and decision.** Two independent choices are hiding in the
table: *where the signing runs* (external `sign-pipelines` vs. E) and *where
canaries are published* (netbird prereleases A, a canary repo B, or nothing
but artifacts C). **A was chosen** for publishing: it needs no pipeline
changes, is the only option the in-app updater and `pkgs.netbird.io` can
serve, and its two risks are contained (2.7 shows the tag sorts correctly,
2.8 shows `latest` is unaffected and the rc window has a one-line fix). E
stays open as a later simplification of the signing half; it is orthogonal
to A and nothing in section 3 depends on it.

---

## 3. Test implementation in this repository

`mlsmaycon/canary-releases` plays two roles: the *source* (stand-in for
netbird, with a tiny Go module that mirrors the artifact naming) and the
*release target*. No real signing secrets are used until Phase 3, and Phase 3
starts with `skipRelease: true`.

### Phase 0 — scaffold a netbird-shaped stub

Files:

```text
go.mod                         module github.com/mlsmaycon/canary-releases
client/main.go                 prints version/commit/date from -X ldflags   -> binary "netbird"
client/ui/main.go              same                                          -> binary "netbird-ui"
.goreleaser.yaml               client: linux/darwin/windows, amd64/arm64, darwin universal
.goreleaser_ui.yaml            windows UI, archive id windows-arch
.goreleaser_ui_darwin.yaml     darwin UI amd64/arm64/universal, own checksum name
```

Both goreleaser configs must reproduce the names from 1.2 exactly. Draft of the
client config (the shape is what matters; the UI configs differ only in ids,
archive templates and `checksum.name_template`, as in netbird):

```yaml
version: 2
project_name: netbird
env:
  - CANARY={{ if index .Env "CANARY" }}{{ .Env.CANARY }}{{ else }}false{{ end }}
  - RELEASE_OWNER={{ if index .Env "RELEASE_OWNER" }}{{ .Env.RELEASE_OWNER }}{{ else }}mlsmaycon{{ end }}
  - RELEASE_REPO={{ if index .Env "RELEASE_REPO" }}{{ .Env.RELEASE_REPO }}{{ else }}canary-releases{{ end }}

builds:
  - id: netbird
    dir: client
    binary: netbird
    env: [CGO_ENABLED=0]
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]
    ldflags:
      - -s -w -X main.version={{.Version}} -X main.commit={{.Commit}} -X main.date={{.CommitDate}}
    mod_timestamp: "{{ .CommitTimestamp }}"

universal_binaries:
  - id: netbird            # adds netbird_{V}_darwin_all.tar.gz, like netbird

archives:
  - builds: [netbird]      # default template -> netbird_{V}_{os}_{arch}.tar.gz

snapshot:
  version_template: "{{ .Version }}-SNAPSHOT-{{ .ShortCommit }}"

changelog:
  disable: "{{ .Env.CANARY }}"   # canary releases carry no changelog

release:
  github:
    owner: "{{ .Env.RELEASE_OWNER }}"   # Option B: release into another repo
    name: "{{ .Env.RELEASE_REPO }}"
  make_latest: false
  prerelease: auto
  target_commitish: "{{ .FullCommit }}"
  replace_existing_artifacts: true      # Option D re-runs
  mode: replace                         # release notes only
  name_template: '{{ if eq .Env.CANARY "true" }}Canary {{ end }}v{{ .Version }}'
```

Things this phase verifies by simply running `goreleaser check` and a
`--snapshot` build locally: config validity under v2.16, and that
`release.github.owner/name` accept templates (GoReleaser documents them as
templatable; if not, fall back to a second config or a `yq` patch step).

### Phase 1 — GoReleaser canary mechanics (no signing)

Add `.github/workflows/canary.yml`:

- **Triggers**: `pull_request` (`labeled`, `synchronize`, `reopened`) gated on
  `contains(github.event.pull_request.labels.*.name, 'canary')` and on
  `head.repo.full_name == github.repository`; `workflow_dispatch` with
  `base_version` (optional `X.Y.Z` override) and `sign` (boolean, Phase 2);
  `schedule` nightly for `main`. Not on every push to `main`: that would create
  a release per merge.
- **Concurrency**: `canary-${{ github.event.pull_request.number || github.ref }}`, cancel in progress.
- **Job `version`**: check out the PR **head sha** (not the merge ref; the tag
  must point at a commit developers recognise) with `fetch-depth: 0`, compute:

  ```bash
  base="${INPUT_BASE_VERSION:-}"
  if [ -z "$base" ]; then
    last=$(git describe --tags --abbrev=0 --match 'v[0-9]*' --exclude '*-canary*' 2>/dev/null || echo v0.0.0)
    IFS=. read -r maj min pat <<<"${last#v}"
    base="$maj.$min.$((pat + 1))"          # canary of "what comes after the last tag"
  fi
  id="main"; [ -n "$PR_NUMBER" ] && id="pr${PR_NUMBER}"
  echo "tag=v${base}-canary.${id}.${GITHUB_RUN_NUMBER}" >> "$GITHUB_OUTPUT"
  ```

  Dotted prerelease identifiers (`canary.pr12.34`) are valid semver; `sign.yml`
  already composes `fullversion-prerelease` since `v0.1.8`.
- **Job `tag`** (Option A): push the tag so that the repository's own
  Release workflow runs on it, exactly as a human-pushed `v*` tag would. Two
  sub-variants to compare, because events created with `GITHUB_TOKEN` do not
  start workflows:
  - **A1**: `git push origin "$tag"` with a PAT / GitHub App token
    (`CANARY_TAG_TOKEN`). `on: push: tags: v*` fires; zero changes to
    `release.yml`.
  - **A2**: push the tag with `GITHUB_TOKEN`, then `gh workflow run release.yml
    --ref "$tag"`. Needs a `workflow_dispatch:` trigger added to
    `release.yml`, but no long-lived token.
- **Job `build`** (stand-in for netbird's Release workflow, in
  `release.yml` of this repo): `on: push: tags: v*` plus `workflow_dispatch`,
  `goreleaser release --clean` three times (client, ui, ui-darwin) with
  `GITHUB_TOKEN`, `CANARY=true` when the tag contains `-canary.`. GoReleaser
  builds whatever the tag points at; the PR head does not have to be on `main`.
- **Job `verify`**: with `gh api`, assert the release is `prerelease: true`,
  `releases/latest` did not move, the tag exists and points at the head sha,
  and the asset list equals the contract table in 1.2 (fail the job on any
  missing or extra name). Post the asset list as a PR comment (same
  marker-comment pattern as `comment_release_artifacts`).

Also add `canary-cleanup.yml` (`pull_request: closed` deletes
`v*-canary.pr{N}.*` releases with `gh release delete --cleanup-tag`; weekly
schedule deletes canaries older than 14 days) and a stand-in `sync-tag.yml`
that echoes, guarded with netbird's `!contains(github.ref_name, '-')`, to
confirm the fan-out workflows ignore canary tags.

Variants to run and record in a results table (one workflow input `variant`):

| Variant | What it tests | Expected |
| --- | --- | --- |
| 1a **A1**: tag pushed with a PAT from `canary.yml` | the stand-in Release workflow starts on `push: tags` | one Release run for the tag; prerelease created, `latest` unchanged |
| 1b **A2**: tag pushed with `GITHUB_TOKEN` + `gh workflow run release.yml --ref $tag` | the no-PAT path | no run from the push itself; exactly one run from the dispatch, on the tag ref |
| 1c GitHub `latest` after several canaries | `make_latest: false` + `prerelease: auto` | `releases/latest` still returns the last stable; `/releases/latest/version` semantics preserved |
| 1d rolling tag (`v…-canary.pr{N}` moved with `-f`) + second run | Option D on top of A | second run succeeds thanks to `replace_existing_artifacts`; notes replaced |
| 1e 30+ canaries then `GET releases?per_page=30` | the `packages-proxy` rc window from 2.8 | an `-rc` release present before the canaries drops out of the page; confirms the need for `per_page=100` and cleanup |
| 1f fork PR / unlabeled PR | gating | job skipped, no token exposure |
| 1g `canary-cleanup.yml` on PR close | retention | releases and tags for that PR are gone; stable and rc releases untouched |

Success for Phase 1: the contract assets exist on a prerelease that is not
`latest`, the Release stand-in ran exactly once per canary tag through A1 or
A2, cleanup removes what it should, and the rc-window behaviour from 2.8 is
reproduced and documented.

### Phase 2a — external signer hand-off, against a mock signer

Add `.github/workflows/sign-mock.yml` with **the same `workflow_dispatch`
inputs as `sign.yml`** (`tag`, `Repository`, `skipRelease`, `skipWindowsRun`,
`skipMacosRun`, `skipMacosCaskRun`) and the same first steps: `parse-semver` on
`tag`, compose `version`, `actions/checkout` of `Repository` at `v{version}`,
`robinraju/release-downloader` for each asset name from 1.2. Instead of
AzureSignTool / codesign it repackages: adds a `SIGNED-BY-MOCK` marker into
`netbird_{V}_windows_{arch}_signed.tar.gz`, produces placeholder
`netbird_installer_{V}_windows_{arch}.{exe,msi}`, `netbird-ui_{V}_darwin_*_signed.zip`
and `netbird_{V}_darwin*.pkg`, and uploads them with
`svenstaro/upload-release-action` (`overwrite: false`, as the real one) when
`!skipRelease`. It keeps the real gate expressions (`prerelease == ''`,
`Repository == netbirdio/netbird`) on no-op steps so the evaluation is visible
in the run.

Extend the stand-in `release.yml` with a `trigger_signer` job (tags only,
mirroring netbird's) that uses `benc-uk/workflow-dispatch` with
`workflow: "Sign bin and installer (mock)"`, `repo: mlsmaycon/canary-releases`,
`inputs: {tag: "refs/tags/<tag>", Repository: "mlsmaycon/canary-releases", skipRelease: false}`.
Inside one repository `GITHUB_TOKEN` may dispatch (needs `actions: write`), and
`workflow_dispatch` runs started by `GITHUB_TOKEN` do execute, so no PAT is
needed here.

Verifies: `parse-semver` output for a dotted canary prerelease; the exact
asset-name round trip; downloading assets from a **prerelease** by tag; how a
second dispatch collides on `overwrite: false` (input for Option D); that the
`skipRelease` and `prerelease` gates evaluate as expected for canaries.

### Phase 2b — in-repo signing model (Option E), against mock signing steps

Add `sign_windows` and `sign_macos` jobs to `canary.yml` itself, shaped like
the ones in `sign.yml` but fed from workflow artifacts:

- `needs: [build]`, `runs-on: windows-2022` / `macos-latest` with the same
  arch matrices, `environment: signing` (create the environment in this repo
  with yourself as required reviewer and deployment tags limited to
  `v*`; the job must not start before approval).
- `actions/download-artifact` of the `dist/` artifacts the build job uploaded
  (`release`, `release-ui`, `release-ui-darwin`, as netbird already names
  them), then the real packaging steps from `sign.yml`: stage the tarballs
  into `dist/netbird_windows_{arch}`, wintun download, `wails3 generate
  webview2bootstrapper`, `makensis`, `wix build`; and on macOS the
  `NetBird.app` tree, `Info.plist`, `pkgbuild`, `productbuild`. This needs the
  vendored installer sources from Phase 3 in the tree, so vendor them here
  already.
- Signing itself is mocked: Windows uses a throwaway certificate
  (`New-SelfSignedCertificate` + `signtool sign /f`) so `signtool verify`
  runs against the right file set; macOS uses ad-hoc `codesign -s -` and skips
  notarization. Read the secrets the real steps would need
  (`AZURE_*`, `CERTIFICATES_P12`, ...) as *environment* secrets holding dummy
  values, so the run proves they are only available after approval.
- A `publish` job (`needs: [sign_windows, sign_macos]`) uploads the signed
  outputs to the canary release with `gh release upload --clobber`, then the
  existing `verify` assertions run against the final asset list.

Verifies: that the tag/release can be created *after* signing (the build job
only needs the version string; the release upload is last); the approval gate
UX and timing; that fork PRs and unlabeled PRs never reach the environment;
`download-artifact` handoff sizes and durations vs. the release round trip;
`gh release upload --clobber` as the answer to the `overwrite: false`
collision from 2a. Record wall-clock and runner minutes for 2a vs. 2b in
`docs/results.md`.

### Phase 3 — real `sign-pipelines` dry-run against this repository

Under Option A the production signer receives real netbird tags, so this
phase only validates that `sign-pipelines` handles a *canary-shaped* prerelease
tag end to end (version composition, asset names, gates). `sign.yml` needs the
installer sources at the tag, so vendor them into this repo at a pinned netbird
commit: `LICENSE`, `client/installer.nsis`,
`client/netbird.wxs`, `client/ui/build/windows/icon.ico`,
`client/ui/build/banner.bmp`, `client/ui/build/darwin/icons.icns`,
`client/ui/assets/netbird.png`, `release_files/darwin-ui-*.sh`,
`release_files/darwin_pkg/*`, and a `github.com/wailsapp/wails/v3` requirement
in `go.mod` (the Windows job greps it to install `wails3` for the WebView2
bootstrapper). Record the source commit in a `VENDORED_FROM` file. The stub
binaries are real PE/Mach-O files, so AzureSignTool, NSIS, WiX, `codesign`,
`pkgbuild` and notarization all run for real.

Then dispatch the real workflow from the Actions tab of `netbirdio/sign-pipelines`:

1. `tag: refs/tags/v0.0.X-canary.prN.M`, `Repository: mlsmaycon/canary-releases`,
   `skipRelease: true`, `skipMacosCaskRun: true`. Signed outputs appear only as
   workflow artifacts in sign-pipelines; nothing is uploaded, no `.sig` push.
   Verify `signtool verify`, `codesign --verify`, `pkgutil --check-signature`
   and notarization pass on stub binaries.
2. `skipRelease: false` **only after** `sign.yml` gains a way to skip the
   `.sig` push for non-netbird repositories (see 4.2). Otherwise this step
   writes `artifact-signatures/tag/v0.0.X-canary…/` into `netbirdio/public-keys`.
   `CI_GITHUB_TOKEN` also needs write access to this repository for the upload;
   an org-owned scratch repository may be simpler than a personal one.

The cheapest real end-to-end proof for A is different and needs no vendoring:
push one canary-shaped tag (for example `v0.71.5-canary.test.1`) to
`netbirdio/netbird` at a commit that is already released, let the existing
Release workflow and signer run, verify the prerelease, `pkgs.netbird.io`
`latest` and `rc` endpoints, then delete the release and tag. That single run
answers most of Phase 3 for A; the vendored dry-run above is only needed if
the team wants to iterate on `sign.yml` without touching netbird.

If Option E wins instead, Phase 3 is not run against `sign-pipelines` at all.
Real certificates never belong in this personal repository, so the in-repo
flavour stops at Phase 2b here and continues as Phase 4.3 in netbird: the
first real run is a `v*-rc.*` tag on netbird with the ported jobs behind the
`signing` environment, comparing its outputs byte-for-byte (signature
identity, notarization ticket, MSI/pkg contents) with what `sign-pipelines`
produces for the same tag.

### Suggested layout after Phases 0–3

```text
.github/workflows/canary.yml           build + release canary, verify; 2a dispatches the mock signer, 2b signs in-repo behind the `signing` environment
.github/workflows/canary-cleanup.yml   delete canary releases/tags on PR close + weekly
.github/workflows/release.yml          stand-in: proves tag filters / no recursion
.github/workflows/sign-mock.yml        contract double of sign-pipelines
.goreleaser.yaml / .goreleaser_ui.yaml / .goreleaser_ui_darwin.yaml
client/, client/ui/                    stub binaries
client/installer.nsis, client/netbird.wxs, release_files/, LICENSE   vendored for Phase 3
docs/results.md                        the variant table from Phase 1 with run links
```

---

## 4. Production sketch (after the test)

### 4.1 `netbirdio/netbird` (Option A)

- New `.github/workflows/canary.yml`: `pull_request` (`labeled`) gated on the
  `canary` label and `head.repo.full_name == github.repository`, plus
  `workflow_dispatch` (`pr` number or `sha`, optional `base_version`). One
  job: compute the tag as in 2.7 and push it at the PR head (A1 with
  `CANARY_TAG_TOKEN`, or A2 with `GITHUB_TOKEN` + `gh workflow run
  release.yml --ref`), then comment the future release URL on the PR. The
  existing Release workflow and `trigger_signer` do everything else.
- `release.yml`: no functional change required. Recommended small edits:
  `SKIP_DOCKER_PUSH=true` when `prerelease` starts with `canary` (GHCR `pr-N`
  images already exist for PRs); skip `release_freebsd_port` and
  `release_ui_gtk3` for canary tags; `workflow_dispatch:` if A2 is chosen.
- GoReleaser configs: `changelog.disable` templated on a `CANARY` env so
  canary releases do not carry a changelog since the last stable.
- New `.github/workflows/canary-cleanup.yml`: on `pull_request: closed`
  delete the PR's canary releases and tags; weekly sweep of canaries older
  than 14 days. This is also what keeps the `packages-proxy` rc window healthy.
- Tag protection: if rulesets restrict who may create `v*` tags, allow the
  token used by `canary.yml`.
- Secrets: `CANARY_TAG_TOKEN` for A1 only; A2 needs none.

### 4.2 `netbirdio/sign-pipelines`

- Nothing is required for A. Decide whether canary `.sig` files should reach
  `public-keys` (they make the updater able to install a canary when pointed
  at it); if not, gate `sign_all_artifacts` on `prerelease` not starting with
  `canary`.
- Optional: `overwrite` input for rolling releases (Option D).
- Optional: `pr_number` / `notify_repo` inputs so the signer can comment back
  on the originating PR with the signed asset links.
- `CI_GITHUB_TOKEN` needs `contents: write` on the canary repository.
- Tidy-ups from 1.4 while there.

### 4.3 If signing moves into `netbirdio/netbird` (Option E)

- Port `sign_windows` and `sign_macos` from `sign.yml` into a reusable
  `.github/workflows/sign.yml` (`on: workflow_call`, inputs `version`,
  `publish`, `artifact_prefix`; `secrets: inherit`). `release.yml` calls it
  after the build jobs for `v*` tags; `canary.yml` calls it after the canary
  build. Replace the release-downloader steps with `download-artifact`; replace
  `svenstaro/upload-release-action` with `gh release upload --clobber` in a
  final `publish` job that also flips `make_latest` for stable tags.
- Create the `signing` environment: required reviewers (release managers),
  deployment branches/tags `main`, `release-*`, `v*`; move `CERTIFICATES_P12*`,
  `DEVELOPER_ID_INSTALLER_*`, the notarization credential and, if Azure OIDC is
  not adopted immediately, the `AZURE_*` values into it as environment
  secrets. Nothing signing-related stays a repository-level secret.
- Azure: register a federated credential for
  `repo:netbirdio/netbird:environment:signing`, give the workflow
  `id-token: write`, `azure/login` before AzureSignTool, and sign with
  `-kvm` (managed identity / DefaultAzureCredential) instead of `-kvi`/`-kvs`.
- Apple: switch `notarytool` to an App Store Connect API key
  (`--key`, `--key-id`, `--issuer`) stored in the environment.
- `NB_ARTIFACT_PRIV_KEY`: keep `sign_all_artifacts` in `sign-pipelines`,
  dispatched only for `v*` tags, or give it a second environment
  (`artifact-signing`) with a smaller reviewer set. Canaries never produce
  `.sig` files.
- `CODEOWNERS` entries for `.github/workflows/**` and `.github/actions/**`;
  branch protection already applies to `main`.
- Delete `test_windows_installer` (the real installer job replaces it) and
  `trigger_signer`; retire `SIGN_GITHUB_TOKEN` and, if `sign_all_artifacts`
  also moves, `sign-pipelines` itself.

### 4.4 `netbirdio/packages-proxy`

- `docker/common.conf`, `location = /releases/rc/raw`: `per_page=30` →
  `per_page=100`, and raise `subrequest_output_buffer_size` (see 2.8).
- Optional: follow one `Link: rel="next"` page in `withLatestRc` before
  returning 404.
- Independent of canaries: move the committed GitHub token out of
  `docker/common.conf` and rotate it.

---

## 5. Open decisions

1. ~~Where canaries live~~ **Resolved: A**, prerelease tags on `netbirdio/netbird`.
2. **Where signing runs**: keep `sign-pipelines` (as A assumes) or later move the
   Windows and macOS jobs into netbird behind a protected environment (E).
   Orthogonal to A; Phase 2b stays optional.
2a. **A1 or A2** for pushing the tag: a stored PAT/App token that fires
   `push: tags`, or `GITHUB_TOKEN` plus a `workflow_dispatch` on `release.yml`.
3. **Trigger**: `canary` label, `/canary` comment, or maintainer `workflow_dispatch`;
   and whether every subsequent push to a labelled PR rebuilds and re-signs
   (macOS notarization is the slow, paid part: 3 zips + 3 pkgs per run).
4. **Version base**: last reachable tag + patch bump (default above) vs explicit input.
5. **Unique tag per run** (simple, more releases) vs **rolling tag per PR** (D).
6. **`.sig` files for canaries**: only meaningful if the updater should ever
   install them (A). Otherwise gate them off.
7. **Retention**: delete on PR close plus a weekly sweep (proposed 14 days).
8. **Phase 3 shape**: one throwaway canary-shaped tag on `netbirdio/netbird`
   (fast, real) vs the vendored dry-run in this repo (isolated, more setup).
9. **If E**: who sits on the `signing` environment's reviewer list, and whether
   `NB_ARTIFACT_PRIV_KEY` moves at all.
