# GitHub Actions Workflows

A collection of reusable GitHub Actions workflows for PHP and Node.js projects, designed to streamline CI/CD processes for web development projects.

## Workflows

### Core Workflows

- `release.yml`: Creates releases using Google's release-please action with automatic versioning and changelog generation.
- `release-gate.yml`: Reports a commit status on the release-please PR so a ruleset can block merging it until assets are built, see [Release gate](#release-gate-block-the-release-pr-while-assets-are-building).
- `publish.yml`: Publishes NPM packages to GitHub Packages registry with configurable Node.js versions and release types.

### PHP Workflows

- `php-coding-standards.yml`: Runs PHP_CodeSniffer on changed PHP files to enforce coding standards.
- `php-static-analysis.yml`: Performs static code analysis using PHPStan on changed PHP files.

### Package Release Workflows

#### `theme-release.yml`

Creates packaged releases for WordPress themes or similar projects.

**.github/workflows/release.yml**

```yaml
on:
  push:
    branches:
      - main
    tags:
      - '*'

name: Release

jobs:
  release:
    uses: mindkomm/workflows/.github/workflows/theme-release.yml@main
    with:
      # Optional package name.
      # package_name: 'mind'
    secrets:
      COMPOSER_AUTH_JSON: ${{ secrets.COMPOSER_AUTH_JSON }}
      READ_PACKAGES_TOKEN: ${{ secrets.READ_PACKAGES_TOKEN }}```
```

#### `plugin-release.yml`

Similar to theme-release but with enhanced cleanup capabilities.

**.github/workflows/release.yml**

```yaml
on:
  push:
    branches:
      - main

name: CI

jobs:
  release:
    uses: mindkomm/workflows/.github/workflows/plugin-release.yml@main
    with:
      # Optional package name.
      # package_name: 'plugin-name'

      # Optional path to cleanup configuration file.
      # cleanup_file: 'cleanup.txt'
    secrets:
      COMPOSER_AUTH_JSON: ${{ secrets.COMPOSER_AUTH_JSON }}
      READ_PACKAGES_TOKEN: ${{ secrets.READ_PACKAGES_TOKEN }}
```

##### Excluding Files from the Release

There are two ways to exclude files from the release .zip file:

**1. Using `.gitattributes` (Recommended for source files)**

Create a `.gitattributes` file in your repository root to exclude files during the `git archive` step. This is ideal for excluding development files that should never be in the release.

```gitattributes
# Exclude development and CI files
.github export-ignore
.gitattributes export-ignore
.gitignore export-ignore
tests/ export-ignore
phpunit.xml export-ignore
.php-cs-fixer.php export-ignore
phpstan.neon export-ignore
```

**2. Using a cleanup file (For build artifacts and dependencies)**

Create a `cleanup.txt` file (or specify a custom path with the `cleanup_file` input) to remove files after the build process. This is useful for removing build artifacts and dependencies that are needed during the build but not in the final release.

The cleanup file supports:
- **Glob patterns**: `node_modules/**/*.map`, `*.config.js`
- **Simple paths**: `node_modules`, `webpack.config.js`
- **Comments**: Lines starting with `#` are ignored
- **Empty lines**: Ignored

**cleanup.txt example:**

```
# Remove build dependencies and source files
node_modules
assets

# Remove configuration files
webpack.config.js
webpack.mix.js
package.json
package-lock.json
babel.config.json

# Remove source maps
**/*.map
```

If no cleanup file is found, the workflow falls back to default cleanup patterns that remove common development files.

#### Release gate: block the release PR while assets are building

`release.yml` runs release-please on every push to the release branch. Repositories that also build and commit assets or translations on that branch (`assets-build-commit.yml` and friends) have a race: the release PR is mergeable while the build is still running, and the bot commit then lands *after* the release, which cuts a release without fresh assets and immediately opens a new patch release PR.

`release-gate.yml` (and the `release-gate` action it wraps) closes that window with a commit status on the head of the open release-please PR. Report `pending` before the build, run release-please only after the build committed, then report the final result. Make the status a required check on the release branch and the PR cannot be merged until the run finished. Statuses are posted through the API because workflows triggered by the release-please bot's own PR never run (they end up as `action_required`), so a regular `pull_request` check cannot do this job.

**.github/workflows/release.yml**

```yaml
name: Release

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

# Queue runs instead of cancelling: a run must report its own pending/success
# pair, otherwise an earlier run could mark the PR green while a later push is
# still building.
concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

permissions:
  contents: write
  pull-requests: write
  statuses: write
  issues: write
  packages: write

jobs:
  gate-pending:
    if: github.event_name == 'push'
    uses: mindkomm/workflows/.github/workflows/release-gate.yml@main
    with:
      state: pending

  build-assets:
    needs: gate-pending
    uses: mindkomm/workflows/.github/workflows/assets-build-commit.yml@main
    secrets:
      READ_PACKAGES_TOKEN: ${{ secrets.READ_PACKAGES_TOKEN }}
      DEPLOY_KEY: ${{ secrets.RELEASE_DEPLOY_KEY }}

  release:
    needs: build-assets
    uses: mindkomm/workflows/.github/workflows/release.yml@main
    secrets: inherit

  gate-result:
    if: always() && github.event_name == 'push'
    needs: [gate-pending, build-assets, release]
    uses: mindkomm/workflows/.github/workflows/release-gate.yml@main
    with:
      state: ${{ needs.release.result == 'success' && 'success' || 'failure' }}

  # Human PRs need the required status too: report success on their head.
  gate-pull-request:
    if: github.event_name == 'pull_request'
    uses: mindkomm/workflows/.github/workflows/release-gate.yml@main
    with:
      state: success
```

Then add a ruleset on the release branch (Settings → Rules → Rulesets, or `gh api`) with **Require status checks to pass** and `release-gate` as the required check. Leave "Require branches to be up to date" off, release-please rebases its PR itself. Do not add admins or teams to the bypass list, otherwise they can still merge past the gate.

A required status check also rejects direct pushes of commits that lack the status, so the bot commits (assets, translations) cannot be pushed with `GITHUB_TOKEN` any more (`GH013: Repository rule violations`), and GitHub does not accept the GitHub Actions app as a bypass actor of a repository ruleset. Push them with a deploy key instead and list deploy keys as bypass actors:

1. Create an SSH key, add the public part as a deploy key with write access, store the private part as a repository secret (say `RELEASE_DEPLOY_KEY`).
2. Pass it to `assets-build-commit.yml` as the `DEPLOY_KEY` secret, and check out with `ssh-key: ${{ secrets.RELEASE_DEPLOY_KEY }}` in any other job that commits to the branch. Add `[skip ci]` to the body of those commit messages: unlike `GITHUB_TOKEN`, deploy key pushes trigger workflows.
3. Add `{ "actor_type": "DeployKey", "bypass_mode": "always" }` to the ruleset's `bypass_actors`.

How it behaves:

- A push to `main` marks the open release PR `pending` within seconds. The PR stays blocked until assets are committed and release-please refreshed it, then the status is reported on the new PR head.
- If the build fails, the PR is marked `failure` and stays blocked. A PR head without any status is blocked too ("Expected"), so a crashed run fails closed.
- When release-please does not update the PR (nothing changed in the release notes), the status is reported on the existing head.
- Inputs: `state` (required), `sha`, `target-branch`, `head-branch` (defaults to `release-please--branches--<target-branch>`), `context` (defaults to `release-gate`), `description`.

Do not bump the required check on other branches: the status is only ever reported for PRs targeting the release branch.

## Requirements

- PHP projects require a valid `composer.json` and `.php-version` file.
- Node.js projects require a valid `package.json`.
- Appropriate secrets configured in your repository settings.

## Configuration

Most workflows accept input parameters for customization. Check individual workflow files for available options and defaults.
