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
| `mise-config` | Tool versions in `mise.toml` and friends | Mise updates |
| `hk-config` | The `hk` tool and the `hk.pkl` Pkl package, kept in sync | hk updates |
| `pre-commit-config` | `.pre-commit-config.yaml` hooks, plus `prek.toml` via a stopgap custom manager | Pre-commit updates |
| `nix-config` | Nix flakes and packages | Nix updates |
| `python-config` | Python packages and Copier templates | Python / Copier updates |
| `javascript-config` | JavaScript and TypeScript packages | JavaScript updates |
| `go-config` | Go modules, plus the `go` directive in `go.work` | Go updates |
| `java-config` | Maven, Gradle | Java updates |
| `dotnet-config` | NuGet | .NET updates |
| `rust-config` | Cargo | Rust updates |
| `ruby-config` | Bundler, `.ruby-version` | Ruby updates |
| `php-config` | Composer | PHP updates |

> [!IMPORTANT]
> Preset order in `default.json` is significant, because later `packageRules` win:
>
> - `container-config` must come **before** `kubernetes-config`, so images declared in Kubernetes manifests keep the `container` label but are regrouped alongside the chart that ships them.
> - `hk-config` must stay **last**, so its grouping overrides `mise-config` for `hk` dependencies.

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
