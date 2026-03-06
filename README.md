# .github

Org-level reusable GitHub Actions workflows for Unstable Studios.

## Reusable Workflows

| Workflow | Purpose | Key Inputs |
|----------|---------|------------|
| [`ci.yml`](.github/workflows/ci.yml) | Lint, typecheck, build, test | `node-version`, `enable-matrix`, `doppler-project`, scripts |
| [`release-please.yml`](.github/workflows/release-please.yml) | Automated versioning and changelogs | `config-file`, `manifest-file` |
| [`release-pr-check.yml`](.github/workflows/release-pr-check.yml) | Lightweight gate for release PRs (skip full CI) | `manifest-file` |
| [`commitlint.yml`](.github/workflows/commitlint.yml) | Conventional commit enforcement | `config-file` |
| [`publish-npm.yml`](.github/workflows/publish-npm.yml) | Publish to GitHub Packages | `package-filter`, `registry-url` |
| [`deploy-cloudflare.yml`](.github/workflows/deploy-cloudflare.yml) | Workers deploy + migrations | `migration-command`, `doppler-project`, scripts |
| [`deploy-terraform.yml`](.github/workflows/deploy-terraform.yml) | Plan on PR, apply on tag | `working-directory`, `plan-only`, `doppler-project` |
| [`stale.yml`](.github/workflows/stale.yml) | Auto-close stale issues and PRs | `days-before-stale`, `days-before-close` |

## Usage

Call from any repo in the org:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main]

jobs:
  # Skip full CI for release PRs — just validate the PR itself
  release-check:
    if: startsWith(github.head_ref, 'release-please') || github.actor == 'release-please[bot]'
    uses: unstable-studios/.github/.github/workflows/release-pr-check.yml@main

  # Full CI for everything else
  ci:
    if: "!startsWith(github.head_ref, 'release-please') && github.actor != 'release-please[bot]'"
    uses: unstable-studios/.github/.github/workflows/ci.yml@main
    secrets: inherit
```

### With Doppler + Matrix

```yaml
jobs:
  ci:
    uses: unstable-studios/.github/.github/workflows/ci.yml@main
    with:
      enable-matrix: true
      doppler-project: my-app
      doppler-config: ci
    secrets: inherit
```

### Release + Publish

```yaml
# .github/workflows/release-please.yml
name: Release
on:
  push:
    branches: [main]

jobs:
  release:
    uses: unstable-studios/.github/.github/workflows/release-please.yml@main
    secrets: inherit

# .github/workflows/publish.yml
name: Publish
on:
  push:
    tags: ["v*"]

jobs:
  publish:
    uses: unstable-studios/.github/.github/workflows/publish-npm.yml@main
    secrets: inherit
```

### Deploy on Release Tag

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    tags: ["v*"]

jobs:
  cloudflare:
    uses: unstable-studios/.github/.github/workflows/deploy-cloudflare.yml@main
    with:
      migration-command: "pnpm drizzle-kit migrate"
      doppler-project: my-app
    secrets: inherit

  terraform:
    uses: unstable-studios/.github/.github/workflows/deploy-terraform.yml@main
    with:
      plan-only: false
    secrets: inherit
```

### Terraform Plan on PR

```yaml
jobs:
  terraform-plan:
    uses: unstable-studios/.github/.github/workflows/deploy-terraform.yml@main
    with:
      plan-only: true
      post-plan-comment: true
    secrets: inherit
```

### Custom Overrides (e.g. Electron)

For repos needing custom steps (like Electron cross-platform builds), use the shared workflows for the common parts and add repo-specific jobs:

```yaml
jobs:
  ci:
    uses: unstable-studios/.github/.github/workflows/ci.yml@main
    with:
      node-version: "24"
      build-script: "pnpm run typecheck && electron-vite build"
      post-build-script: |
        test -d out/main && test -d out/preload && test -d out/renderer
    secrets: inherit

  # Repo-specific: cross-platform Electron builds
  build-release:
    needs: ci
    strategy:
      matrix:
        os: [macos-latest, windows-latest, ubuntu-latest]
    runs-on: ${{ matrix.os }}
    steps:
      # ... electron-builder steps
```

## Conventions

- **Package manager**: pnpm everywhere
- **Runner**: All workflows respect `vars.RUNS_ON`, defaulting to `ubuntu-latest`. Set the repo/org variable for self-hosted runners.
- **Concurrency**: CI cancels in-progress; deploys and releases do not.
- **Release PRs**: Use `release-pr-check.yml` instead of full CI — the code has already passed CI on the constituent commits.
- **Secrets**: Use `secrets: inherit` to pass all secrets from the caller. Doppler integration is opt-in via `doppler-project` input.
- **Conventional commits**: Enforced via `commitlint.yml`. Repos should have `@commitlint/config-conventional` as a dev dependency.
- **Database migrations**: Use [Drizzle ORM](https://orm.drizzle.team/) with `drizzle-kit` for D1 schema management. Define schema in TypeScript, generate migrations with `drizzle-kit generate`, and run them in CI via the `migration-command` input on `deploy-cloudflare.yml`. This replaces hand-written SQL migrations and `wrangler d1 migrations apply`.
