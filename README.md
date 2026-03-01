# Renovate Configuration

This repository contains shared Renovate Bot configuration for the Pinguteca organization. It provides a reusable, centrally-managed configuration for automating dependency updates across multiple repositories.

## Overview

[Renovate](https://www.renovatebot.com/) is a tool that automatically detects, updates, and tests dependency updates in your repositories. This config uses best practices and security-focused defaults.

## Files

### `default.json`

The main Renovate configuration file that extends Renovate's best practices preset and applies the following settings:

**Base Configuration:**

- Extends `config:best-practices` for recommended defaults

**Commit & PR Behavior:**

- Enables semantic commits for consistent commit messages
- Creates draft PRs
- Uses `renovate/` branch prefix with strict naming
- Applies `[skip ci]` to pin update commit bodies
- Uses `immediate` PR creation for quick feedback
- Automerges non-major and non-minors updates after passing checks

**Update Strategy:**

- Maintains a 21-day minimum release age before updating
- Uses `bump` range strategy for version constraints
- Does not separate minor/patch updates
- Separate major updates into their own PRs

**Dependency Labels & Workflow:**

- Labels all PRs with `renovate` tag
-
- Auto-merges type: `pr` for straightforward updates

**Security Features:**

- Security vulnerability alerts automatically merged
- Vulnerability fix strategy set to `lowest` for minimal disruption
- Security-related PRs labeled with `security` tag
- Strict internal checks filtering enabled

**Network:**

- HTTP/2 support enabled for faster package registry requests

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
     "schedule": ["after 9am before 5pm"],
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

Waits 21 days after a package release before creating update PRs following Snyk recommendation. This reduces the risk of updating to versions with undiscovered bugs.

### Draft PRs

Renovate creates draft PRs instead of ready-for-review PRs, saving compute and early unnecessary CI checks.

### Security Auto-merge

Security vulnerability fixes are automatically merged after passing checks even if they do not meet the [minimum release age](#minimum-release-age), ensuring timely security patching.

### Lock File Maintenance

Keeps lock files fresh by running on a schedule.

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
- [Mise Documentation](https://mise.jdx.dev/getting-started.html)
- [Prek Documentation](https://prek.j178.dev/)
- [Snyk 21 day cooldown strategy](https://snyk.io/blog/shai-hulud-post-mortem/#the-21-day-cooldown-strategy)
