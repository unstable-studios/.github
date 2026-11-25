# .github

Org-level reusable GitHub Actions workflows for Unstable Studios.

## Reusable Workflows

| Workflow | Purpose | Key Inputs |
|----------|---------|------------|
| [`ci.yml`](.github/workflows/ci.yml) | Lint, typecheck, build, test | `node-version`, `enable-matrix`, `doppler-project`, scripts |
| [`release-please.yml`](.github/workflows/release-please.yml) | Automated versioning and changelogs | `config-file`, `manifest-file`, `mark-latest` |
| [`release-pr-check.yml`](.github/workflows/release-pr-check.yml) | Lightweight gate for release PRs (skip full CI) | `manifest-file` |
| [`commitlint.yml`](.github/workflows/commitlint.yml) | Conventional commit enforcement | `config-file`, `registry-url` |
| [`publish-npm.yml`](.github/workflows/publish-npm.yml) | Publish to GitHub Packages | `package-filter`, `registry-url` |
| [`deploy-cloudflare.yml`](.github/workflows/deploy-cloudflare.yml) | Workers deploy + migrations | `migration-command`, `doppler-project`, scripts |
| [`preview-cloudflare.yml`](.github/workflows/preview-cloudflare.yml) | Workers PR preview upload + URL comment | `wrangler-subcommand`, `build-script`, `comment-on-pr` |
| [`deploy-terraform.yml`](.github/workflows/deploy-terraform.yml) | Plan on PR, apply on tag | `working-directory`, `plan-only`, `doppler-project` |
| [`publish-relay.yml`](.github/workflows/publish-relay.yml) | Publish release artifacts to Relay | `org`, `product`, `version`, `artifact-name`, `artifact-path`, `policy-type` |
| [`publish-testflight.yml`](.github/workflows/publish-testflight.yml) | Archive an iOS app, sign it and upload it to TestFlight | `project`, `scheme`, `version`, `environment` |
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

### Cloudflare PR Preview

Uploads a Worker version without deploying it, then posts (and updates on each
push) a comment with the versioned preview URL. Nothing on the live routes
changes.

```yaml
# .github/workflows/preview.yml
name: PR Preview
on:
  pull_request:

jobs:
  preview:
    uses: unstable-studios/.github/.github/workflows/preview-cloudflare.yml@main
    secrets: inherit
```

`CLOUDFLARE_ACCOUNT_ID` is optional here — wrangler reads `account_id` from
`wrangler.jsonc`/`wrangler.toml` when it isn't set. The preview URL is also
exposed as a `preview-url` output for downstream jobs (smoke tests, Lighthouse):

```yaml
jobs:
  preview:
    uses: unstable-studios/.github/.github/workflows/preview-cloudflare.yml@main
    secrets: inherit

  smoke:
    needs: preview
    runs-on: ubuntu-latest
    steps:
      - run: curl -fsS "$URL" > /dev/null
        env:
          URL: ${{ needs.preview.outputs.preview-url }}
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

### Release whose files are attached later

GitHub marks a release Latest the moment it is created. If installers are built and attached afterwards, anything that reads the Latest release (an in-app updater, a download link) finds a release with nothing in it for a few minutes.

Set `mark-latest: false` to leave the mark on the release that had it. The job that attaches the files then marks the new release:

```yaml
jobs:
  release:
    uses: unstable-studios/.github/.github/workflows/release-please.yml@main
    with:
      mark-latest: false
    secrets: inherit
```

```yaml
# In the workflow that builds the files, after they are attached:
- run: gh release edit "$TAG" --repo "$GITHUB_REPOSITORY" --latest
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    TAG: ${{ github.event.release.tag_name }}
```

The new release has the mark for the second or two before it is moved back, since Release Please has no setting for it. The first release of a repository keeps it, there being no earlier release to give it to.

### Publish Artifact to Relay

The publish workflow expects a GitHub Actions artifact (from `actions/upload-artifact`). A prior job must build the file and upload it:

```yaml
jobs:
  release:
    uses: unstable-studios/.github/.github/workflows/release-please.yml@main
    secrets: inherit

  build:
    needs: release
    if: needs.release.outputs.release_created == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm install --frozen-lockfile && pnpm build
      - uses: actions/upload-artifact@v4
        with:
          name: my-app-release
          path: dist/my-app.zip

  publish:
    needs: [release, build]
    uses: unstable-studios/.github/.github/workflows/publish-relay.yml@main
    with:
      org: "unstable-studios"
      product: "my-app"
      version: ${{ needs.release.outputs.version }}
      artifact-name: "my-app-release"
      artifact-path: "my-app.zip"
      policy-type: "unlimited"
    secrets:
      RELAY_API_SECRET: ${{ secrets.RELAY_API_SECRET }}
```

Outputs `download-url`, `artifact-id`, and `link-id` for downstream jobs. Set `RELAY_API_SECRET` as a repo or org secret.

### Publish an iOS App to TestFlight

Chained off release-please, so merging the release PR uploads that version:

```yaml
jobs:
  release:
    uses: unstable-studios/.github/.github/workflows/release-please.yml@main
    secrets: inherit

  testflight:
    needs: release
    if: needs.release.outputs.release_created == 'true'
    uses: unstable-studios/.github/.github/workflows/publish-testflight.yml@main
    with:
      project: MyApp.xcodeproj
      scheme: MyApp
      version: ${{ needs.release.outputs.version }}
    secrets:
      ASC_KEY_P8: ${{ secrets.ASC_KEY_P8 }}
```

Pass `ref` too when the caller has the release's commit (release-please's `sha` output), so the build is the tagged commit even if `main` has moved on.

The job runs in the calling repo's `testflight` environment and reads the App Store Connect key from it: `ASC_KEY_ID` and `ASC_ISSUER_ID` as variables, `ASC_KEY_P8` as a secret. Keep the key in Doppler, in a config of its own, with a GitHub sync to that environment and "Sync unmasked secrets as variables" on. The secret has to be passed by name even though it only exists in the environment ([reusing workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)).

- Signing is cloud signing: no certificate or profile on the runner. The key is a team key with the Admin role.
- Limit the environment to the branch the caller runs on (`main` when chained off release-please). Whoever can run a workflow there can use the key.
- With no `version`, it reads the root package's version from `.release-please-manifest.json`. A `workflow_dispatch` caller then uploads whatever is on the branch it ran for, under the last released version, as a new build.
- The build number is the time in UTC to the minute (`202611251630`), so it rises across re-runs, renamed workflows and several callers alike.
- The key's ID and issuer ID show in the job's log, because `xcodebuild` prints its command line. They are no use without the key file. In a public repo they would be public.
- The repo provides an export options plist (default `Config/TestFlight-ExportOptions.plist`) with `method` `app-store-connect` and `destination` `upload`, and its Info.plist answers the export compliance question (`ITSAppUsesNonExemptEncryption`).
- The runner is `xcode-27` with Xcode 27.2 unless `runs-on` and `xcode-path` say otherwise. `vars.RUNS_ON` doesn't apply: it names a Linux runner.

First used by `rabbithole-apple`.

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
- **Runner**: All workflows respect `vars.RUNS_ON`, defaulting to `ubuntu-latest`. Set the repo/org variable for self-hosted runners. `publish-testflight.yml` needs a Mac and takes a `runs-on` input instead.
- **Caching**: the pnpm store and Terraform plugin caches use GitHub's cache on GitHub-hosted runners only (`if: runner.environment == 'github-hosted'`). On self-hosted runners (echo-turbine) uploading to GitHub's cache took minutes per job; the runner keeps the pnpm store on the host instead. Follow the same rule for new `actions/cache` steps and `setup-*` `cache:` inputs.
- **Concurrency**: CI cancels in-progress; deploys and releases do not.
- **Release PRs**: Use `release-pr-check.yml` instead of full CI — the code has already passed CI on the constituent commits.
- **Secrets**: Use `secrets: inherit` to pass all secrets from the caller. Doppler integration is opt-in via `doppler-project` input.
- **Conventional commits**: Enforced via `commitlint.yml`. Repos should have `@commitlint/config-conventional` as a dev dependency.
- **Database migrations**: Use [Drizzle ORM](https://orm.drizzle.team/) with `drizzle-kit` for D1 schema management. Define schema in TypeScript, generate migrations with `drizzle-kit generate`, and run them in CI via the `migration-command` input on `deploy-cloudflare.yml`. This replaces hand-written SQL migrations and `wrangler d1 migrations apply`.
