# GitVersion versioning for .NET clients

## TL;DR

The .NET client repos extend [`pipelines/templates/dotnetclient.yml`](../pipelines/templates/dotnetclient.yml).
They were **pinned to GitVersion 5.11.x** because a naive bump to GitVersion 6
silently breaks versioning:

* untagged `main` builds publish **released** versions (`0.15.0`) instead of
  CI pre-releases (`0.15.0-ci.<n>`), and
* release tags such as `v0.15.0-beta.1` are **not** honoured — they finalize to
  a stable `0.15.0`.

A **validated permanent fix** now exists (see
[Permanent fix on GitVersion 6](#permanent-fix-on-gitversion-6)). In short:
GitVersion computes **branch (CI) builds** only; **tag builds take the version
verbatim from the tag name** and skip GitVersion entirely. This is the only way
to satisfy all rows below on GitVersion 6 — a maintainer has
[confirmed](https://github.com/GitTools/GitVersion/discussions/4696) that a single
GitVersion config cannot produce both `ci`-on-`main` **and** verbatim pre-release
tags. The template change must be rolled out **together** with the per-consumer
migration (`gitversion.yml` v6 schema + `GitVersion.MsBuild` 6.x), because builds
resolve the template from `build`'s default branch at queue time.

The desired behaviour is:

| Situation                                   | Version produced        |
| ------------------------------------------- | ----------------------- |
| untagged commit on `main`                   | `X.Y.Z-ci.<n>`          |
| latest reachable tag is a pre-release       | continues that lineage, e.g. `X.Y.Z-beta.<n>` |
| build of a release tag `vX.Y.Z-beta.1`      | `X.Y.Z-beta.1` (verbatim) |
| build of a release tag `vX.Y.Z`             | `X.Y.Z` (stable)        |

## Background: there are *two* GitVersions in play

1. **The pipeline task** `gitversion/execute` — sets the Azure build number and
   the `GitVersion_*` pipeline variables. Controlled by `versionSpec` in the
   template.
2. **`GitVersion.MsBuild`** — a `PackageReference` in each client's
   `src/Directory.Build.props`. This runs during `dotnet build`/`pack` and is
   what actually **stamps the package version** (the pipeline packs with
   `versioningScheme: off`). It also feeds `$(GitVersion_MajorMinorPatch)` /
   `$(GitVersion_NuGetPreReleaseTag)` into the PowerShell-module packaging
   (`build/Prepare-PsModule.ps1` in the client repos).

Both must agree on a major version. Bumping only one of them produces a build
number that disagrees with the published package version.

## What broke (2026-05, regression and fix)

On 2026-05-12 the template was bumped to the GitVersion **6.6.x** task
(`eryph-org/build` commits `8a15ef9`, `64d3cd4`, `d594f9f`). The client repos
still carried GitVersion-5-style `gitversion.yml`:

```yaml
assembly-versioning-scheme: MajorMinor
mode: ContinuousDeployment
branches: {}
```

Under GitVersion 6 this regressed every consumer (e.g. `dotnet-computeclient`
build 6440 onward produced stable `0.15.0`). Root causes:

1. **Root-level `mode` was removed in v6.** It became a per-branch property
   (`mode` in 6.6.x, renamed to `deployment-mode` in 6.7.0). The old root
   `mode: ContinuousDeployment` is ignored, so `main` falls back to the v6
   default (empty pre-release label) → stable versions.
2. **The deployment-mode values were redefined.** GitVersion 5
   `ContinuousDeployment` (auto-incrementing pre-release) is **GitVersion 6
   `ContinuousDelivery`**. GitVersion 6 `ContinuousDeployment` now emits a
   *stable* version per commit (it strips the pre-release label).
3. **The `NuGet*` output variables were removed in v6**
   (`GitVersion_NuGetVersion`, `GitVersion_NuGetPreReleaseTag`, …). The
   PowerShell-module packaging depends on `NuGetPreReleaseTag` because a PS
   module prerelease must be alphanumeric (`beta0020`), not the SemVer2
   `beta.20`.
4. **Pre-release tags are not honoured on `main` in v6.** A *stable* tag on the
   current commit (`v0.16.0` → `0.16.0`) is used verbatim, but a *pre-release*
   tag (`v0.15.0-beta.1`) is treated as a hint and finalized to stable / relabelled.
   GitVersion 5 returns the pre-release tag verbatim — which is how the
   `beta.N`/`preview.N` releases were produced.

**Fix:** the template was pinned back to `gitversion/setup@3.1.11` +
`versionSpec: '5.11.x'`, matching the repos that never regressed (`eryph`,
`guest-services`, `dotnet-genepoolclient`, which run their own pipelines on
5.11.x). Each client repo keeps `GitVersion.MsBuild` 5.11.1 and the original
v5 `gitversion.yml`.

> Builds resolve this template from `build`'s default branch **at queue time**,
> so the template must be fixed *before* re-running a consumer build.

## Clean reproduction on a test repo

This reproduces the divergence locally without Azure. Requires the .NET SDK.

A `dotnet` tool manifest holds only **one** version per tool, so install each
GitVersion CLI in its **own** manifest folder:

```powershell
$work = Join-Path $env:TEMP "gv-repro"
New-Item -ItemType Directory -Force -Path $work | Out-Null

foreach ($v in '5.11.1', '6.6.2') {
    $m = Join-Path $work "gv-$v"
    New-Item -ItemType Directory -Force -Path $m | Out-Null
    Push-Location $m
    dotnet new tool-manifest --force | Out-Null
    dotnet tool install GitVersion.Tool --version $v | Out-Null
    Pop-Location
}
```

Two helpers — one to evaluate a chosen GitVersion version, one to spin up a
minimal repo. Use a **separate repo per scenario** so the latest tag is always
the one under test; checking out an older commit in a repo that also has *newer*
tags makes GitVersion (in detached HEAD) pick up the highest tag and gives
misleading results.

```powershell
function Get-RepoVersion([string]$Version, [string]$Repo) {
    Push-Location (Join-Path $work "gv-$Version")
    $v = dotnet tool run dotnet-gitversion $Repo /showvariable FullSemVer 2>$null
    Pop-Location
    $v.Trim()
}

function New-TestRepo([string]$Name) {
    $r = Join-Path $work $Name
    New-Item -ItemType Directory -Force -Path $r | Out-Null
    Push-Location $r
    git init -q -b main
    git config core.autocrlf false      # avoid CRLF churn blocking checkout
    git config user.email t@t.t; git config user.name t
    # GitVersion-5-style config, as the client repos have it
    Set-Content gitversion.yml "assembly-versioning-scheme: MajorMinor`nmode: ContinuousDeployment`nbranches: {}`n"
    "a" > f.txt; git add .; git commit -q -m c1; git tag v0.14.0   # baseline stable release
    Pop-Location
    return $r
}
```

Scenario 1 — untagged commit on `main` (a normal CI build, evaluated on the branch):

```powershell
$r1 = New-TestRepo "sc1"
Push-Location $r1; "b" > f.txt; git commit -q -am c2; Pop-Location
"untagged main : v5=$(Get-RepoVersion '5.11.1' $r1)  v6=$(Get-RepoVersion '6.6.2' $r1)"
```

Scenario 2 — a pre-release **tag build** (Azure checks the tag out in detached HEAD):

```powershell
$r2 = New-TestRepo "sc2"
Push-Location $r2; "b" > f.txt; git commit -q -am c2; git tag v0.15.0-beta.1
git checkout -q --detach v0.15.0-beta.1; Pop-Location
"beta tag      : v5=$(Get-RepoVersion '5.11.1' $r2)  v6=$(Get-RepoVersion '6.6.2' $r2)"
```

Scenario 3 — a stable **tag build**:

```powershell
$r3 = New-TestRepo "sc3"
Push-Location $r3; "b" > f.txt; git commit -q -am c2; git tag v0.16.0
git checkout -q --detach v0.16.0; Pop-Location
"stable tag    : v5=$(Get-RepoVersion '5.11.1' $r3)  v6=$(Get-RepoVersion '6.6.2' $r3)"
```

Verified results:

| Scenario                                    | GitVersion 5.11.1 | GitVersion 6.6.2  |
| ------------------------------------------- | ----------------- | ----------------- |
| untagged commit on `main` after `v0.14.0`   | `0.14.1-ci.1`     | `0.14.1` (stable) |
| tag build of `v0.15.0-beta.1` (detached)    | `0.15.0-beta.1`   | `0.15.0` (stable) |
| tag build of `v0.16.0` (detached, stable)   | `0.16.0`          | `0.16.0`          |

The first two rows are the regression: v6 drops the `-ci` CI label and finalizes
the `-beta.1` pre-release tag to a stable release. (With a branch `label`
configured for `main`, v6 instead rewrites the tag's label, e.g.
`0.15.0-ci.<n>`.) This matches the real pipeline: build 6755 (tag
`v0.15.0-beta.20`) under the restored 5.11.x produced `FullSemVer
0.15.0-beta.20`, package `Eryph.ComputeClient.0.15.0-beta.20.nupkg`, and PS
module prerelease `beta0020`.

> **Note on the Azure agent.** When `TF_BUILD=true`, GitVersion uses its
> AzurePipelines agent, which requires a git *remote* (it errors with "0
> remote(s) have been detected" otherwise) and reads `BUILD_SOURCEBRANCH`. To
> reproduce the exact agent behaviour, add a bare `origin`, `git push` the
> branch and tags, set `TF_BUILD=true` and `BUILD_SOURCEBRANCH=refs/tags/v…`,
> then run GitVersion. The plain (LocalBuild) runs above already demonstrate the
> version contrast without that setup.

## Permanent fix on GitVersion 6

The shared template now targets **GitVersion 6.7.x** using a split that matches
the GitHub-flow release model (CI pre-releases on `main`, named tags = releases):

* **Branch (CI) builds** — GitVersion computes the version. With the v6 config
  below, `main` produces `X.Y.Z-ci.<n>`.
* **Tag builds (`refs/tags/v*`)** — the version is taken **verbatim from the tag
  name** (minus the leading `v`). GitVersion is **not** run at all: the
  `gitversion/execute` task is skipped, and `GitVersion.MsBuild` is disabled
  during `pack` via `-p:DisableGitVersionTask=true -p:Version=<tag>`.

### Why tag builds must bypass GitVersion

This is **by design** in GitVersion 6, not a misconfiguration on our side:

* A pre-release tag's label is only honoured when it **matches the branch's
  configured `label`** (v6
  [BREAKING_CHANGES](https://github.com/GitTools/GitVersion/blob/main/BREAKING_CHANGES.md)).
  So `label: ci` (needed for CI builds) makes v6 **relabel** a `beta` tag to
  `ci`, or finalise it to stable. `label: ''`/`null` loses the `ci` label.
  Verified with all three label modes — none satisfy the matrix.
* A GitVersion maintainer has
  [confirmed](https://github.com/GitTools/GitVersion/discussions/4696) (Oct 2025)
  that producing `ci` on untagged commits **and** verbatim pre-release tags from
  one config is *"not possible"*.
* On Azure DevOps, tag builds check out a **detached HEAD** with
  `BUILD_SOURCEBRANCH=refs/tags/v*`; GitVersion 6 then falls back to `-no-branch-`
  ([#4534](https://github.com/GitTools/GitVersion/issues/4534), closed without a
  fix). Bypassing GitVersion for tags sidesteps this entirely.

This combination (CI-on-`main` + tag-as-release-version on Azure DevOps) has been
reported upstream repeatedly since 2020 (e.g.
[#2231](https://github.com/GitTools/GitVersion/issues/2231),
[#2362](https://github.com/GitTools/GitVersion/issues/2362),
[#4015](https://github.com/GitTools/GitVersion/issues/4015)) and never got a
complete upstream fix; deriving the version from the tag is the accepted
pragmatic solution.

> **Alternative considered — `label: beta`.** Setting `main`'s `label: beta`
> reproduces v5 behaviour (verbatim `beta` tags *and* beta lineage) with no
> pipeline logic, but labels untagged CI builds `beta.<n>` instead of `ci.<n>`
> and silently relabels any non-`beta` pre-release tag (`alpha`, `rc`). Rejected
> because we want `ci` for CI builds and a label-agnostic release path.

### Validation

Proven end-to-end on real Azure DevOps builds (GitVersion 6.7.0), at both the
pipeline level (build number) and the package level (`dotnet pack` output):

| Scenario                               | Checkout      | Version produced  |
| -------------------------------------- | ------------- | ----------------- |
| commit on `main` (untagged)            | normal        | `X.Y.Z-ci.<n>`    |
| tag `vX.Y.Z-beta.1`                    | detached HEAD | `X.Y.Z-beta.1`    |
| tag `vX.Y.Z`                           | detached HEAD | `X.Y.Z`           |

Note: untagged CI builds after a pre-release tag now carry `-ci.<n>` (not the v5
`-beta.<n>` lineage). NuGet ordering is preserved (`beta` < `ci`), and CI builds
go to the internal feed while releases come from tags, so they do not collide.

### Per-consumer migration (required, all consumers together)

The template change must land **with** these per-consumer changes, since builds
resolve the template from `build`'s default branch at queue time:

1. Rewrite each `gitversion.yml` to the v6 schema:
   ```yaml
   assembly-versioning-scheme: MajorMinor
   branches:
     main:
       deployment-mode: ContinuousDelivery   # v5 'ContinuousDeployment' == v6 'ContinuousDelivery'
       label: ci
   ```
2. Bump `GitVersion.MsBuild` to **6.7.x** in every `Directory.Build.props` (or, for
   repos without one, in each project file — e.g. `dotnet-commonclient`).

That is the **entire** per-consumer change. The PowerShell-module versioning
(`withCmdlet` repos) is handled **centrally by the template**: GitVersion 6 dropped
the `GitVersion_NuGet*` outputs, so the template reconstructs `NuGetPreReleaseTag`
(`label` + 4-digit zero-padded number, e.g. `ci0001`, `beta0020`) and `MajorMinorPatch`
for **both** build kinds and injects them —

* as MSBuild properties (`-p:GitVersion_MajorMinorPatch=… -p:GitVersion_NuGetPreReleaseTag=…`)
  for repos that stamp the module during build via `Prepare-PsModule.ps1`
  (`dotnet-computeclient`, `dotnet-clientruntime`, `dotnet-identityclient`), and
* as `GITVERSION_*` pipeline/environment variables for repos whose cmdlet step reads
  them directly (`dotnet-commonclient`).

So `Prepare-PsModule.ps1` / `build-cmdlet.ps1` keep working unchanged. On tag builds
this matters doubly: GitVersion.MsBuild is disabled there, so without the injected
`GitVersion_MajorMinorPatch` the module build would fail its version validation.

The repro harness in [Clean reproduction on a test repo](#clean-reproduction-on-a-test-repo)
above validates the matrix locally before any consumer is changed.
