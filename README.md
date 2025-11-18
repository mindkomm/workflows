# GitHub Actions Workflows

A collection of reusable GitHub Actions workflows for PHP and Node.js projects, designed to streamline CI/CD processes for web development projects.

## Workflows

### Core Workflows

- `release.yml`: Creates releases using Google's release-please action with automatic versioning and changelog generation.
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

## Requirements

- PHP projects require a valid `composer.json` and `.php-version` file. Validate your composer.json file using `composer validate` before using this workflow.
- Node.js projects require a valid `package.json`.
- Appropriate secrets configured in your repository settings.

## Configuration

Most workflows accept input parameters for customization. Check individual workflow files for available options and defaults.
