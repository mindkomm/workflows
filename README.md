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
    secrets:
      COMPOSER_AUTH_JSON: ${{ secrets.COMPOSER_AUTH_JSON }}
      READ_PACKAGES_TOKEN: ${{ secrets.READ_PACKAGES_TOKEN }}
```

## Requirements

- PHP projects require a valid `composer.json` and `.php-version` file. Validate your composer.json file using `composer validate` before using this workflow.
- Node.js projects require a valid `package.json`.
- Appropriate secrets configured in your repository settings.

## Configuration

Most workflows accept input parameters for customization. Check individual workflow files for available options and defaults.
