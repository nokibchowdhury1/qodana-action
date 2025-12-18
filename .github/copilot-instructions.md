# Qodana Action: AI Agent Developer Guide

## Repository Architecture

This is a **monorepo** containing Qodana integrations for 4 CI/CD platforms:
- **GitHub Actions** (`scan/`) - Main TypeScript action using `action.yaml`
- **Azure Pipelines** (`vsts/`) - Azure DevOps extension
- **GitLab CI** (`gitlab/`) - GitLab integration
- **CircleCI Orb** (`orb/`) - YAML-based CircleCI orb
- **Gradle Plugin** (`plugin/`) - Kotlin-based Gradle plugin

All TypeScript integrations share **common code** in `common/` (core Qodana CLI interaction logic).

## Development Workflow

### Build & Test Commands
```bash
# Install dependencies (root + all workspaces)
npm ci

# Build all TypeScript packages
npm run build

# Lint all packages
npm run lint

# Test all packages  
npm run test

# Full CI check (lint + build + test + package)
npm run all
```

### Package-specific operations
```bash
# GitHub Action
cd scan && npm run build && npm run package  # Creates scan/dist/index.js

# Azure Extension  
cd vsts && npm run build && npm run package  # Creates vsts/QodanaScan/

# GitLab Integration
cd gitlab && npm run build && npm run package  # Creates gitlab/dist/

# Gradle Plugin
./gradlew :plugin:test -PtestGradleVersion=8.5
```

## Critical Patterns

### 1. Distributable Files Must Be Committed
- `scan/dist/index.js` - **Must be committed** after changes to `scan/src/`
- `vsts/QodanaScan/` - **Must be committed** after changes to `vsts/src/`
- CI enforces this: `.github/workflows/node.yml` auto-commits if dist is stale

### 2. Qodana CLI Version Synchronization
- All integrations use the same Qodana CLI version defined in `common/cli.json`
- Update via automated PRs when new CLI releases (see `common/update-cli.js`)
- CLI download logic: `common/qodana.ts` (`getQodanaUrl()`, `getQodanaSha256()`)

### 3. Multi-platform Support
- Qodana CLI runs on: `windows`, `linux`, `darwin` × `x86_64`, `arm64`
- Platform detection: `common/qodana.ts` (`getProcessPlatformName()`, `getProcessArchName()`)

### 4. Test Strategy
- Unit tests: Jest in `__tests__/` directories
- Integration tests: `.github/workflows/node.yml` test-action matrix
  - Uses `JetBrains/code-analytics-examples` for real-world scenarios
  - Tests multiple Qodana linters (JVM, Python, .NET, Go, JS, PHP, etc.)
  - Validates both PR mode (`pr-mode: true`) and full scan modes

### 5. GitHub Action Specifics
- Entry point: `scan/src/main.ts` → compiled to `scan/dist/index.js`
- Inputs defined: `action.yaml` (must stay in sync with `scan/src/utils.ts`)
- Quick-fixes workflow: `pushQuickFixes()` in `scan/src/utils.ts`
- PR annotations: `scan/src/annotations.ts` using GitHub Check Runs API

## Key Files & Directories

| Path | Purpose |
|------|---------|
| `common/qodana.ts` | Core Qodana CLI constants, exit codes, platform logic |
| `common/cli.json` | **Single source of truth** for CLI version & checksums |
| `scan/src/main.ts` | GitHub Action entry point |
| `scan/src/utils.ts` | GitHub-specific utilities (caching, artifacts, git ops) |
| `vsts/src/main.ts` | Azure Pipelines entry point |
| `action.yaml` | GitHub Action metadata (inputs, outputs, branding) |
| `.github/workflows/node.yml` | CI for TypeScript packages |
| `.github/workflows/gradle.yml` | CI for Gradle plugin |

## Versioning & Release

- **CLI-driven versioning**: Extensions follow [qodana-cli releases](https://github.com/JetBrains/qodana-cli/releases)
- **Release trigger**: Push git tag `vX.X.X` → `.github/workflows/release.yml` publishes to:
  - GitHub Marketplace, Azure DevOps Marketplace, CircleCI Orb Registry, Gradle Plugin Portal
- **No manual version bumps** except Azure extension files (`vsts/vss-extension.json`, `vsts/QodanaScan/task.json`)

## Code Style

- **Commit messages**: Use [Gitmoji](https://gitmoji.dev) (e.g., `:sparkles:` for features)
- **Linting**: ESLint config in `.github/linters/.eslintrc.yml` + Prettier
- **Node.js version**: Pinned in `.node-version`
- **TypeScript**: Strict mode, shared config in `tsconfig.base.json`

## Common Debugging Scenarios

### Dist out of sync
Run `npm run package` in `scan/` or `vsts/` before committing.

### Test failures in CI
Check `.github/workflows/node.yml` matrix for specific linter/OS combinations.
View logs at `${{ runner.temp }}/qodana/results/log/idea.log`.

### Gradle plugin issues
Test with different Gradle versions: `./gradlew :plugin:test -PtestGradleVersion=7.6`
