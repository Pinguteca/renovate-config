# Renovate Configuration

This repository contains shared Renovate Bot configuration for the Pinguteca organization. It provides a reusable, centrally-managed configuration for automating dependency updates across multiple repositories.

## Overview

[Renovate](https://www.renovatebot.com/) is a tool that automatically detects, updates, and tests dependency updates in your repositories. This config uses best practices and security-focused defaults.

## Files

### `default.json`

The main Renovate configuration file. It extends Renovate's best practices preset, then layers the per-ecosystem presets in `presets/` on top of it.

> [!NOTE]
> Several options in `default.json` restate a current Renovate default. This is deliberate: pinning them explicitly means that if a default changes upstream, it shows up here as a diff in a PR rather than as a silent change in behaviour.

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

Each file is an independently extendable preset. Every one of them follows the same two-rule shape:

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
| `javascript-config` | JS/TS packages, plus Node and package-manager pins | JavaScript updates |
| `go-config` | Go modules, plus the `go` directive in `go.work` | Go updates |
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

### Toolchain versus dependencies

Three presets split a repository's *toolchain* away from its *dependencies*, because the two carry different risk and want different reviewers. A dependency bump is routine; a change to the compiler, runtime or build tool is not, and it should not arrive buried in a group of forty patches.

The split is driven by what the managers actually emit, checked with `renovate --platform=local`, rather than by guesswork about file names.

### .NET

The `nuget` manager already covers more than project files: `global.json` (both the SDK pin and MSBuild SDKs), `Directory.Build.props`, and `Directory.Packages.props` for Central Package Management. No custom manager is needed for any of them.

The preset separates three things that would otherwise land in one pull request:

- **Runtime-aligned packages** (`Microsoft.AspNetCore.*`, `Microsoft.EntityFrameworkCore.*`, `Microsoft.Extensions.*`, `System.*`) are grouped together, majors included. These ship in lockstep with the .NET major, so a pull request that moves some to the next major and leaves the rest behind will not build.
- **The SDK pin** in `global.json` gets its own pull request. It is a toolchain change, not a package update.
- **MSBuild SDKs** likewise, as build infrastructure.

### Java

`gradle-wrapper` and `maven-wrapper` pin the build tool itself, so they share a **Java build tool updates** group. Majors are included deliberately: the `maven-wrapper` manager emits both the Maven distribution and the wrapper jar, and those two must move together.

Maven build plugins (`build`) and Gradle plugins (`plugin`) group separately from application dependencies as **Java build plugin updates**.

A Maven `parent` is never grouped. Bumping a parent POM changes the managed version of everything it governs, so it gets its own pull request.

> [!NOTE]
> Where a Maven version lives in a property, Renovate names the dependency after the property rather than the artifact, so expect a branch like `major-guava.version`.

### JavaScript

The runtime is typically pinned in more than one place at once: `engines`, `volta`, and a `.nvmrc` or `.node-version` file. The package manager is pinned in both `engines` and `packageManager`. If those drift apart the toolchain contradicts itself, so all of them share one **JavaScript toolchain updates** group and move together.

Ordinary packages are unaffected and keep the normal non-major grouping.

### Azure DevOps

`azure-devops-config` sets `azureWorkItemType` to `Task`. Renovate has no issue concept on Azure DevOps, so it stores the Dependency Dashboard as a work item. The default type is `Issue`, which only exists in the `Basic` process; `Task` exists in `Basic`, `Agile` and `Scrum` alike.

> [!WARNING]
> Set this before Renovate's first run on a repository. Renovate keeps an existing dashboard work item when the type changes, so switching later strands the old one.

> [!TIP]
> If a branch policy requires linked work items, Renovate pull requests cannot complete until one is linked. Set `azureWorkItemId` per repository to the id of an existing work item. It takes a specific id, so it cannot be shared from this preset.

`azure-pipelines-config` enables a manager that is off by default, because Renovate cannot tell whether a task version has reached your Azure DevOps instance yet. It covers YAML pipelines only: classic pipelines are defined in the Azure DevOps database rather than in the repository, so there is no file for Renovate to read.

It updates pipeline tasks, images in `resources.containers`, and GitHub repository resources pinned to a tag. It does **not** update `pool.vmImage`, Azure-hosted repositories in `resources.repositories`, or the root `container:` element of a container job.

File patterns are additive rather than replaced, so this preset adds `ci/` on top of the built-in `azure-pipelines.yml` and `.azure-pipelines/**` patterns.

#### Agent images

Prefer `ubuntu-latest`, `windows-latest` and `macos-latest` over a pinned `vmImage`. Renovate cannot maintain a pinned one, and the `-latest` alias is the only thing that tracks the signal that matters: Microsoft moves it when the image is actually available on their hosted agents.

To migrate existing pinned images, extend `azure-pipelines-agent-latest`. It opens a replacement pull request rewriting `ubuntu-22.04` to `ubuntu-latest`, and leaves images already on an alias alone.

> [!WARNING]
> That preset also opens version-chasing pull requests it cannot suppress, roughly two per pinned Ubuntu or macOS image, proposing OS releases that Microsoft may not ship an agent for yet. Close them and merge only the replacement. Both stop permanently once the file no longer pins a version, since the custom manager then matches nothing. Windows is unaffected.

It is a migration helper, not a standing policy, so it is deliberately left out of `default.json`. Extend it per repository and stop once the migration is done.

Do not try to track agent versions instead. `endoflife-date` knows when an OS release exists, not when Microsoft ships an agent for it, and the two are months apart. It also only covers Ubuntu and macOS, returning Windows Server build numbers such as `10.0.20348.2582` rather than the `2019`/`2022`/`2025` labels a pipeline uses. Tracking `actions/runner-images` does not help either, even though Microsoft builds the hosted images there: its tags and releases are per-image builds such as `win25/20260913.261`, not the `windows-2025` labels `vmImage` accepts.

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

This repository includes pre-commit hooks using Prek for local development and CI/CD validation:

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

Commit messages follow the conventional commits format, enabling automated changelog generation and version bumping based on commit type. Renovate uses chore and fix unless specified otherwise.

### Minimum Release Age

Waits 7 days after a package release before creating update PRs. This reduces the risk of updating to versions with undiscovered bugs and vulnerabilities.

Snyk's [21 day cooldown strategy](https://snyk.io/blog/shai-hulud-post-mortem/#the-21-day-cooldown-strategy) argues for a longer window. 7 days is the deliberate trade-off taken here between supply-chain exposure and carrying known-fixed bugs for three weeks. Raise `minimumReleaseAge` in a consuming repository if it warrants a more conservative window.

### Draft PRs

Renovate opens PRs as drafts, so they read as not-yet-for-review while checks settle, and the `automerge` rules set `draftPR: false` wherever a PR is expected to merge on its own. This matters because GitHub refuses to merge a draft PR: any automerge-eligible update that stayed a draft would simply stall.

> [!NOTE]
> Draft status does not by itself stop GitHub Actions from running. Workflows run on draft PRs unless they opt out, for example with `if: github.event.pull_request.draft == false`. Use `prCreation: "not-pending"` instead if the goal is to avoid spending CI minutes.

### Major Updates

Major updates are deliberately excluded from the ecosystem groups, so a breaking change arrives in its own PR with its own checks rather than buried among patches. Major PRs also carry a warning banner in the PR body, and are never automerged.

`hk-config` is the one exception: the `hk` tool and the `hk.pkl` package must move together or the setup breaks, so that group includes majors.

### Security Auto-merge

Security vulnerability fixes are automatically merged after passing checks even if they do not meet the [minimum release age](#minimum-release-age), ensuring timely security patching.

### Lock File Maintenance

Keeps lock files fresh by running on a schedule, and automerges the result.

> [!NOTE]
> Renovate commits a manifest change and its lock file in the same commit, so the two do not drift under normal operation. They can drift when the artifact update itself fails, for example a missing toolchain or a private registry the runner cannot authenticate against. Renovate reports that as a warning in the PR body rather than failing the PR, so consuming repositories should enforce it in CI with a locked install such as `npm ci`, `uv lock --check`, `go mod verify` or `dotnet restore --locked-mode`.

## Extending This Configuration

Repositories can extend this config and override any settings:

```json
{
  "extends": ["github>Pinguteca/renovate-config"],
  "schedule": ["before 3am"],
  "lockFileMaintenance": {
    "enabled": false
  }
}
```

All standard Renovate configuration options are available for customization.

## Resources

- [Renovate Documentation](https://docs.renovatebot.com/)
- [Renovate Configuration Reference](https://docs.renovatebot.com/configuration-options/)
- [Renovate Presets](https://docs.renovatebot.com/config-presets/)
- [Mise Documentation](https://mise.jdx.dev/getting-started.html)
- [Prek Documentation](https://prek.j178.dev/)
- [Snyk 21 day cooldown strategy](https://snyk.io/blog/shai-hulud-post-mortem/#the-21-day-cooldown-strategy)
