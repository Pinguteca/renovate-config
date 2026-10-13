# Renovate Configuration

Shared Renovate configuration for the Pinguteca organization. One place to set dependency update policy for every repository.

## Overview

[Renovate](https://www.renovatebot.com/) finds outdated dependencies, opens pull requests to update them, and lets CI test the result. This config holds the defaults the organization uses.

## Files

### `default.json`

The main Renovate configuration file. It extends Renovate's best practices preset, then layers the per-ecosystem presets in `presets/` on top of it.

> [!NOTE]
> Several options in `default.json` restate a current Renovate default. That is on purpose. Pinning them means an upstream change to a default shows up here as a diff in a pull request, not as a silent change in behaviour.

**Base Configuration:**

- Extends `config:best-practices` for recommended defaults

**Commit & PR Behavior:**

- Enables semantic commits for consistent commit messages
- Creates draft PRs, except where a PR is eligible for automerge
- Uses `renovate/` branch prefix with strict naming
- Applies `[skip ci]` to pin update commit bodies
- Uses `immediate` PR creation for quick feedback
- Adds a warning banner to the PR body on major updates

**Update Strategy:**

- Maintains a 7-day minimum release age before updating
- Uses the `auto` range strategy, letting Renovate pick per ecosystem
- Does not separate minor and patch updates, so they share one PR
- Separates major updates into their own PR, including out of ecosystem groups
- Separates each major version into its own PR when a dependency is several majors behind

**Dependency Labels & Workflow:**

- Labels every PR with `renovate`
- Adds an ecosystem label such as `python` or `container` via the presets in `presets/`
- Adds a `git-hooks` label across pre-commit, prek and hk updates, so hook tooling is visible at a glance without being forced into one group
- Automerges minor, patch, pin and digest updates once checks pass
- Automerges `devDependencies` regardless of update type
- Pausable per-PR by applying the `have-a-break-renovate` label

**Security Features:**

- Security vulnerability alerts automatically merged
- Vulnerability fix strategy set to `lowest` for minimal disruption
- Security-related PRs labeled with `security` tag
- OSV vulnerability alerts enabled, widening coverage beyond GitHub advisories
- Unresolved OSV vulnerabilities summarised on the Dependency Dashboard
- Strict internal checks filtering enabled
- OpenSSF Scorecard badges via `security:openssf-scorecard`

**Network:**

- HTTP/2 support enabled for faster package registry requests

### `presets/`

Each file is a preset you can extend on its own. They all follow the same two-rule shape:

1. A **label** rule matching all update types, so the ecosystem label is present on every PR.
2. A **group** rule matching only `minor`, `patch`, `digest` and `pin`, so major updates fall out of the group and get their own PR.

| Preset | Covers | Group name |
| --- | --- | --- |
| `container-config` | Container images in Dockerfiles, Compose, Quadlet, devcontainers, buildpacks | Container image updates |
| `kubernetes-config` | Helm charts, Kubernetes manifests, Kustomize, Flux, Argo CD, Fleet, vendir | Kubernetes/Helm updates |
| `infrastructure-as-code-config` | Terraform, OpenTofu, Crossplane, Bicep | per tool |
| `configuration-as-code-config` | Ansible, Ansible Galaxy, Puppet | per tool |
| `github-actions-config` | GitHub Actions, pinned to digests with a semver comment | GitHub Actions updates |
| `azure-pipelines-config` | Azure Pipelines tasks, `resources.containers` images, `resources.repositories` tags | Azure Pipelines updates |
| `azure-devops-config` | Azure DevOps platform settings, ignored elsewhere | n/a |
| `azure-pipelines-agent-latest` | Opt-in migration of pinned `vmImage` to `-latest`, not in `default.json` | n/a |
| `mise-config` | Tool versions in `mise.toml` and friends | Mise updates |
| `hk-config` | The `hk` tool and the `hk.pkl` Pkl package, kept in sync | hk updates |
| `pre-commit-config` | `.pre-commit-config.yaml` hooks, plus `prek.toml` via a stopgap custom manager | Pre-commit updates |
| `nix-config` | Nix flakes and packages | Nix updates |
| `python-config` | Python packages and Copier templates | Python / Copier updates |
| `javascript-config` | JS/TS packages, plus Node and package-manager pins and the `engines` floor | JavaScript updates |
| `go-config` | Go modules, plus the `go` directive floor in `go.mod` | Go updates |
| `buf-config` | Pinned Buf remote plugins, paired with the runtime they generate against | ConnectRPC / protobuf codegen |
| `java-config` | Maven, Gradle, their wrappers, plugins and parent POMs | Java updates |
| `dotnet-config` | NuGet, Cake, Unity3D, plus `global.json` and `Directory.*.props` | .NET updates |
| `rust-config` | Cargo | Rust updates |
| `ruby-config` | Bundler, `.ruby-version` | Ruby updates |
| `php-config` | Composer | PHP updates |

> [!IMPORTANT]
> Preset order in `default.json` is significant, because later `packageRules` win:
>
> - `container-config` must come **before** `kubernetes-config`, so images declared in Kubernetes manifests keep the `container` label but are regrouped alongside the chart that ships them.
> - `hk-config` must stay **last**, so its grouping overrides `mise-config` for `hk` dependencies.
> - `container-config` must also come **before** `azure-pipelines-config`, so images declared in a pipeline are regrouped with that pipeline.

### Floors versus selectors

Every version line in a repository answers one of two questions, and the answer decides how Renovate should treat it.

A **selector** says *what this repository builds with*. It is local, nothing downstream reads it, and a mistake shows up in the next CI run. Selectors should move fast and agree with each other.

A **floor** says *the oldest toolchain a consumer of this package may use*. It is part of the published contract, and raising one forces an upgrade on everybody downstream. You cannot take it back once a version is tagged, because a Go module on the proxy cannot be amended. Floors should move slowly and on purpose.

| Language | Floor (published contract) | Renovate sees it? | Selector (what this repo builds with) |
| --- | --- | --- | --- |
| Go | `go` directive in each `go.mod` | yes, dependency-driven | `go` in `mise.toml`, with `go.work` at or below it |
| .NET | `TargetFramework` in the project file | no | `sdk.version` in `global.json` |
| Node | `engines` in `package.json` | yes | `volta`, `packageManager`, `.nvmrc`, `.node-version`, `.bun-version` |
| Rust | `rust-version` in `Cargo.toml` | no | `rust-toolchain.toml` |
| Python | `requires-python` in `pyproject.toml` | no | `.python-version`, or `python` in `mise.toml` |

Selectors are grouped so they cannot disagree. Floors get their own pull request, never automerge, and sit behind `dependencyDashboardApproval`, so a floor bump appears on the Dependency Dashboard instead of opening a pull request on its own. An application, whose floor nobody reads, can drop the approval gate.

The split follows what the managers actually emit, checked with `renovate --platform=local`. It is not guesswork about file names.

> [!IMPORTANT]
> Renovate does not raise a Go floor by itself, and that is on purpose. The stock `go-mod-directive` versioning only bumps the `go` directive when a dependency demands it. In testing, a `go 1.19` directive produced no update at all, with Go 1.27 already out. Leave it that way. `go mod tidy` already raises the directive when a dependency needs more, and raises it to the **lowest** version that satisfies the need. Adding `connectrpc.com/connect/v2` to a `go 1.24` module rewrote it to `go 1.26.0`, not to the newest release. That is the right trigger and the right target. Overriding the versioning to chase the newest release would force every consumer onto the latest Go.
>
> `minimumReleaseAge` is not a way around this. On the `golang-version` datasource it had no effect at 180 or 365 days. Both still proposed a Go 1.27 that was five days old.
>
> A floor can still move inside an ordinary dependency pull request, which may be automerged. Checking that the floor still builds, and that it is still inside the language's support window, belongs in CI. Use a job that builds and tests at the floor version as well as at the one mise selects.

**Avoid two selectors for one toolchain.** Where the same version is pinned in two places, delete one instead of grouping them. Grouping keeps Renovate's own edits in step but does nothing about a hand edit. mise can read the ecosystem's own file instead of repeating the version, which leaves one selector:

```toml
[settings]
idiomatic_version_file_enable_tools = ["dotnet", "rust", "python"]
```

With that in place, `mise current dotnet` resolves from `global.json` and `mise current rust` from `rust-toolchain.toml`. It is off by default in mise and opts in per tool, so you have to name each one. Put it in the repository's own `mise.toml` to make it portable. Setting it per machine brings back the drift it was meant to remove.

> [!NOTE]
> `go.work` is not a floor, and Renovate does not track it. Nothing published reads it, and under the default `GOTOOLCHAIN=auto` a `go` line above the installed toolchain makes Go quietly download another one. Keep it equal to the highest module floor and set `GOTOOLCHAIN=local`, so an overshoot fails with a clear error.

> [!NOTE]
> Three of those floors are not automated at all. No Renovate manager extracts `TargetFramework`, `rust-version` or `requires-python`. Dry runs over a `.csproj`, a `Cargo.toml` and a `pyproject.toml` returned their package references and nothing else. Those rows are review conventions, not something the presets act on. Only the Go and Node floors are visible to Renovate.

### .NET

The `nuget` manager already covers more than project files: `global.json` (both the SDK pin and MSBuild SDKs), `Directory.Build.props`, and `Directory.Packages.props` for Central Package Management. No custom manager is needed for any of them.

The preset separates three things that would otherwise land in one pull request:

- **Runtime-aligned packages** (`Microsoft.AspNetCore.*`, `Microsoft.EntityFrameworkCore.*`, `Microsoft.Extensions.*`, `System.*`) are grouped together, majors included. These ship in lockstep with the .NET major, so a pull request that moves some to the next major and leaves the rest behind will not build.
- **The SDK pin** in `global.json` leaves the package groups. It is the .NET toolchain selector, not a package update. Where a repository also repeats the version as a `dotnet` entry in `mise.toml`, a custom manager resolves that entry and both files share a **.NET SDK toolchain** group, majors included. The custom manager is needed because the `mise` manager extracts the entry but has no datasource for it, and skips it as `unsupported-datasource`, so it would never move. The better end state is to delete the `mise.toml` entry and let mise read `global.json`. The group is a guard for repositories that have not migrated yet.
- **MSBuild SDKs** likewise, as build infrastructure.
- **gRPC code generation.** `Grpc.Tools` generates the C# from the protos at build time, and `Grpc.*` and `Google.Protobuf` are the runtime that generated code calls into. They share a **gRPC codegen and runtime** group, majors included, so a major cannot advance the generator and leave the runtime a major behind. The pull request carries a note reminding whoever merges it to move the matching `buf.build/protocolbuffers/csharp` and `buf.build/grpc/csharp` pins, which Renovate cannot see.

> [!WARNING]
> `/^Grpc\./` is broad on purpose. Every package it matches today sits on the grpc-dotnet 2.x release line, including `Grpc.Tools`, `Grpc.StatusProto` and `Grpc.AspNetCore.HealthChecks`. A repository that still pulls the legacy `Grpc.Core` (2.46.x, deprecated) would be dragged into the group behind a version train it does not follow. Exclude it there if that comes up.

A group that holds a major does not automerge. A branch automerges only if every update in it would, and the shared config automerges minor and patch only. These rules are insurance against splitting a generator from its runtime, not a new route to unattended majors.

### Java

`gradle-wrapper` and `maven-wrapper` pin the build tool itself, so they share a **Java build tool updates** group. Majors are included on purpose. The `maven-wrapper` manager emits both the Maven distribution and the wrapper jar, and those two must move together.

Maven build plugins (`build`) and Gradle plugins (`plugin`) group separately from application dependencies as **Java build plugin updates**.

A Maven `parent` is never grouped. Bumping a parent POM changes the managed version of everything it governs, so it gets its own pull request.

> [!NOTE]
> Where a Maven version lives in a property, Renovate names the dependency after the property instead of the artifact, so expect a branch like `major-guava.version`.

### JavaScript

The runtime is typically pinned in more than one place at once: `volta`, and a `.nvmrc` or `.node-version` file. The package manager is pinned in `packageManager`. If those drift apart the toolchain contradicts itself, so they share one **JavaScript toolchain updates** group and move together.

`engines` is kept out of that group. It is the floor, the oldest Node a consumer may install this package with. Grouping it with the selectors would raise the published contract every time the local toolchain moved. It gets its own **JavaScript engines floor** pull request behind dashboard approval.

Ordinary packages are unaffected and keep the normal non-major grouping.

### Code generation (Buf and ConnectRPC)

Renovate has no manager for `buf.yaml`, `buf.gen.yaml` or `buf.lock`, and there is no BSR datasource. Asking for one returns `Unknown datasource "buf"`. A dry run over a repository containing all three matched only `go.mod` and `mise.toml`.

So generated code can drift away from the runtime it calls into. The generator plugin and the runtime are released together. If Renovate walks `connectrpc.com/connect` from v1.18.1 to v2.0.0 while `buf.gen.yaml` still pins `buf.build/connectrpc/go:v1.18.1`, the generated code may not compile.

`buf-config` closes that by mapping the plugin onto the package Renovate can already resolve. It works because the plugin tag and the runtime version are the same string:

| Pinned in `buf.gen.yaml` | Resolved as | Datasource |
| --- | --- | --- |
| `buf.build/connectrpc/go` | `connectrpc.com/connect` | `go` |
| `buf.build/protocolbuffers/go` | `google.golang.org/protobuf` | `go` |

Both halves then share a group, so the plugin and the runtime move in one pull request, majors included.

> [!WARNING]
> A remote plugin written without a version is not pinned, and floats to whatever the registry serves on each generation. That is worse than a frozen pin and `buf-config` cannot see it: an unversioned `buf.build/protocolbuffers/csharp` entry was not extracted at all in testing. Pin every plugin before expecting any of this to apply.

> [!NOTE]
> The same generator-and-runtime pairing exists in NuGet, where both halves are visible to Renovate. It is handled in `dotnet-config`, not here. See the gRPC entry in the .NET section.

Only the Go plugins are covered. The protobuf runtimes for C#, Java, Kotlin and Python do not share the plugin's version string. A `v29.3` plugin pairs with `3.29.3` on NuGet and `4.29.3` on Maven. Each needs its own mapping, added once the plugins are pinned and the pairing is confirmed against a real run. Module dependencies in `buf.lock` cannot be tracked at all, so use a scheduled `buf dep update` for those.

> [!NOTE]
> Renovate should not edit generated output. Where `buf generate` writes into the repository, add that path to `ignorePaths`, or Renovate and the generator will take turns rewriting the same lines.
>
> This does **not** apply to Copier instances. `copier update` does a three-way merge instead of regenerating, and it does not manage `go.mod` or `go.work`, so updates Renovate makes in an instance survive. Copier instances stay under Renovate, floors included.

### Conda tools (mise)

A mise conda tool is written with a bare package name (`conda:verapdf`), because mise rejects a channel in the name. The channel lives in a separate `channel` property and defaults to conda-forge. Renovate's mise manager passes only the bare name to the conda datasource, which cannot resolve a package without a channel and reports a `no-result` lookup failure. The preset routes bare conda names from the mise manager through the prefix.dev conda-forge channel, which indexes conda-forge and matches names case-insensitively.

The rule matches only names without a slash. mise rejects a channel in the tool name, so it only ever emits a bare name, and any channel-qualified conda spec such as `bioconda/samtools` comes from pixi reading `pixi.toml` instead. Scoping the rule to the mise manager keeps it off those pixi entries, so it never forces the conda-forge registry onto a spec that already carries its own channel and doubles it.

> [!NOTE]
> The rule assumes a bare mise conda tool is a conda-forge package. A mise conda tool on another channel (set through the `channel` property, for example `channel = "bioconda"`) installs correctly, but Renovate still looks it up on conda-forge and fails. The channel property is invisible to Renovate, so this cannot be fixed in the shared preset. Add a per-repo override for that tool:
>
> Match on `packageName`, which is the bare name the datasource queries and the same field the preset rule matches. Matching `depName` would work too but is easier to get wrong, since it carries the `conda:` prefix (`conda:samtools`).
>
> ```json
> {
>   "matchManagers": ["mise"],
>   "matchDatasources": ["conda"],
>   "matchPackageNames": ["samtools"],
>   "registryUrls": ["https://prefix.dev/bioconda"]
> }
> ```

### Azure DevOps

`azure-devops-config` sets `azureWorkItemType` to `Task`. Renovate has no issue concept on Azure DevOps, so it stores the Dependency Dashboard as a work item. The default type is `Issue`, which only exists in the `Basic` process. `Task` exists in `Basic`, `Agile` and `Scrum` alike.

> [!WARNING]
> Set this before Renovate's first run on a repository. Renovate keeps an existing dashboard work item when the type changes, so switching later strands the old one.

> [!TIP]
> If a branch policy requires linked work items, Renovate pull requests cannot complete until one is linked. Set `azureWorkItemId` per repository to the id of an existing work item. It takes a specific id, so it cannot be shared from this preset.

`azure-pipelines-config` enables a manager that is off by default, because Renovate cannot tell whether a task version has reached your Azure DevOps instance yet. It covers YAML pipelines only. Classic pipelines live in the Azure DevOps database, not in the repository, so there is no file for Renovate to read.

It updates pipeline tasks, images in `resources.containers`, and GitHub repository resources pinned to a tag. It does **not** update `pool.vmImage`, Azure-hosted repositories in `resources.repositories`, or the root `container:` element of a container job.

File patterns add to the defaults instead of replacing them, so this preset adds `ci/` on top of the built-in `azure-pipelines.yml` and `.azure-pipelines/**` patterns.

#### Agent images

Prefer `ubuntu-latest`, `windows-latest` and `macos-latest` over a pinned `vmImage`. Renovate cannot maintain a pinned one. The `-latest` alias is the only thing that tracks the signal that matters, because Microsoft moves it when the image is actually available on their hosted agents.

To migrate existing pinned images, extend `azure-pipelines-agent-latest`. It opens a replacement pull request rewriting `ubuntu-22.04` to `ubuntu-latest`, and leaves images already on an alias alone.

> [!WARNING]
> That preset also opens version-chasing pull requests it cannot suppress, roughly two per pinned Ubuntu or macOS image, proposing OS releases that Microsoft may not ship an agent for yet. Close them and merge only the replacement. Both stop permanently once the file no longer pins a version, since the custom manager then matches nothing. Windows is unaffected.

It is a migration helper, not a standing policy, so it is left out of `default.json`. Extend it per repository and stop once the migration is done.

Do not try to track agent versions instead. `endoflife-date` knows when an OS release exists, not when Microsoft ships an agent for it, and the two are months apart. It also only covers Ubuntu and macOS, returning Windows Server build numbers such as `10.0.20348.2582` instead of the `2019`/`2022`/`2025` labels a pipeline uses. Tracking `actions/runner-images` does not help either, even though Microsoft builds the hosted images there. Its tags and releases are per-image builds such as `win25/20260913.261`, not the `windows-2025` labels `vmImage` accepts.

Presets can also be extended individually, without `default.json`:

```json
{
  "extends": [
    "config:recommended",
    "github>Pinguteca/renovate-config//presets/go-config"
  ]
}
```

## Usage

To use this configuration in your repository:

**Reference in your repository's `renovate.json`:**

```json
{
  "extends": [
    "github>Pinguteca/renovate-config"
  ]
}
```

**Or extend with your own settings:**

```json
{
  "extends": [
    "github>Pinguteca/renovate-config",
    ":dependencyDashboard"
  ],
  "schedule": ["after 9am and before 5pm"]
}
```

## Pre-commit Configuration (`prek.toml`)

Pre-commit hooks run through Prek, locally and in CI:

**Hooks Included:**

- **pre-commit-hooks**: Standard file validation
  - Large file detection
  - Case conflict checking
  - Executable shebang validation
  - YAML/JSON validation
  - Trailing whitespace cleanup
  - Line ending fixes (LF)

- **Local hooks**: Project-specific validation
  - Cocogitto commit message verification (conventional commits)

- **renovate-config-validator**: Validates Renovate configuration
  - Runs in strict mode
  - Validates all `.json` files
  - Executes on pre-push

> [!NOTE]
> Renovate's `pre-commit` manager reads `.pre-commit-config.yaml` only, and does not parse `prek.toml`. Until native prek support lands upstream, `pre-commit-config` carries a small `customManagers` regex that tracks the `repo`/`rev` pairs in `prek.toml` and groups them alongside ordinary pre-commit hooks. Drop that block once Renovate reads `prek.toml` natively.

**Development Setup:**

Install dependencies with [Mise](https://mise.jdx.dev/)

```bash
mise install
```

Followed by installing git hooks with [Prek](https://prek.j178.dev/):

```bash
prek install --install-hooks
```

## Configuration Details

### Semantic Commits

Commit messages follow the conventional commits format, so changelogs and version bumps can be generated from the commit type. Renovate uses chore and fix unless told otherwise.

### Minimum Release Age

Waits 7 days after a package release before opening update PRs. That lowers the risk of picking up a release with bugs or vulnerabilities nobody has found yet.

Snyk's [21 day cooldown strategy](https://snyk.io/blog/shai-hulud-post-mortem/#the-21-day-cooldown-strategy) argues for a longer window. 7 days is the trade-off taken here between supply-chain exposure and carrying known-fixed bugs for three weeks. Raise `minimumReleaseAge` in a consuming repository if it warrants a more conservative window.

### Draft PRs

Renovate opens PRs as drafts, so they read as not-yet-for-review while checks settle. The `automerge` rules set `draftPR: false` wherever a PR is expected to merge on its own. GitHub refuses to merge a draft PR, so an automerge-eligible update that stayed a draft would stall.

> [!NOTE]
> Draft status does not by itself stop GitHub Actions from running. Workflows run on draft PRs unless they opt out, for example with `if: github.event.pull_request.draft == false`. Use `prCreation: "not-pending"` instead if the goal is to avoid spending CI minutes.

### Major Updates

Major updates are kept out of the ecosystem groups, so a breaking change arrives in its own PR with its own checks instead of buried among patches. Major PRs also carry a warning banner in the PR body, and are never automerged.

`hk-config` is the one exception: the `hk` tool and the `hk.pkl` package must move together or the setup breaks, so that group includes majors.

### Schedules

There is no schedule window. Non-major updates are created as soon as a version qualifies and merge as soon as checks pass.

An office-hours merge window was tried and removed. Its premise was that an automerged update should land while someone is around to notice breakage, but nothing watches a merge happen. The real safety net is CI refusing to merge a broken change and, for deployed workloads, the GitOps controller rolling back to the previous commit. Against that, the window only added latency. A weekend pull request waited until Monday morning, on top of the seven-day [minimum release age](#minimum-release-age).

> [!NOTE]
> `platformAutomerge` is left at its default of `true`, so the platform merges the moment required checks pass instead of waiting for the next Renovate run.
>
> It has to be `false` for `automergeSchedule` to work at all, because Renovate hands the pull request to the platform's native automerge at creation time and, as the docs put it, "the schedule specified in `automergeSchedule` cannot be followed". If a merge window is ever reintroduced, that setting has to change with it or the window is silently decoration.

Majors are unaffected: they are never automerged, so they wait for a human regardless.

> [!TIP]
> If pull request volume during working hours becomes a problem for CI capacity rather than for review, extend `schedule:nonOfficeHours` to batch branch creation into evenings and weekends. That limits when work is *created* without reintroducing a merge window, and leaves `platformAutomerge` alone.

### Digests and the minimum release age

> [!WARNING]
> The [minimum release age](#minimum-release-age) does **not** apply to container digest updates. Digest updates carry no release timestamp, so Renovate has nothing to age them against and lets them through immediately, even under `internalChecksFilter: "strict"`. This was verified with a dry-run: a ten-year cooldown fails to hold a digest refresh.

Since digest updates are also automerged, a compromised upstream tag would be picked up and merged once CI passes, without waiting out the cooldown. Digest pinning plus digest automerge is therefore close to following the mutable tag, except that each move is a reviewable commit instead of an invisible change.

This is an accepted trade-off, not an oversight. To close it, drop `digest` from the `matchUpdateTypes` of the automerge rule in `default.json`, so digest refreshes become pull requests a person reviews. Expect real volume if you do: base images are rebuilt often, and unreviewed pull requests that pile up turn into stale images, which is its own security problem.

Narrowing the exemption to a curated list of trusted registries was considered and rejected. Keeping such a list current is its own maintenance burden, and it would buy little: digest refreshes already bypass the cooldown for every registry, and those are how vendor CVE fixes ship. A list would only accelerate version bumps by seven days.

### Security Auto-merge

Security fixes merge automatically once checks pass, even if they do not meet the [minimum release age](#minimum-release-age).

### Lock File Maintenance

Refreshes lock files on a schedule and automerges the result.

> [!NOTE]
> Renovate commits a manifest change and its lock file in the same commit, so the two do not drift under normal operation. They can drift when the artifact update itself fails, for example a missing toolchain or a private registry the runner cannot authenticate against. Renovate reports that as a warning in the PR body instead of failing the PR, so consuming repositories should enforce it in CI with a locked install such as `npm ci`, `uv lock --check`, `go mod verify` or `dotnet restore --locked-mode`.

## Extending This Configuration

Repositories can extend this config and override any setting:

```json
{
  "extends": ["github>Pinguteca/renovate-config"],
  "schedule": ["before 3am"],
  "lockFileMaintenance": {
    "enabled": false
  }
}
```

Every standard Renovate option can be overridden.

## Resources

- [Renovate Documentation](https://docs.renovatebot.com/)
- [Renovate Configuration Reference](https://docs.renovatebot.com/configuration-options/)
- [Renovate Presets](https://docs.renovatebot.com/config-presets/)
- [Mise Documentation](https://mise.jdx.dev/getting-started.html)
- [Prek Documentation](https://prek.j178.dev/)
- [Snyk 21 day cooldown strategy](https://snyk.io/blog/shai-hulud-post-mortem/#the-21-day-cooldown-strategy)
