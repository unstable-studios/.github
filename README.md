# .github

Org-level reusable GitHub Actions workflows for Unstable Studios.

## Reusable Workflows

| Workflow | Purpose | Key Inputs |
|----------|---------|------------|
| [`ci.yml`](.github/workflows/ci.yml) | Lint, typecheck, build, test | `node-version`, `enable-matrix`, `doppler-project`, scripts |
| [`ci-apple.yml`](.github/workflows/ci-apple.yml) | Run an Apple app's test script on a Mac with Xcode | `script`, `matrix`, `pull-request-matrix` |
| [`release-please.yml`](.github/workflows/release-please.yml) | Automated versioning and changelogs | `config-file`, `manifest-file`, `mark-latest` |
| [`release-pr-check.yml`](.github/workflows/release-pr-check.yml) | Lightweight gate for release PRs (skip full CI) | `manifest-file` |
| [`commitlint.yml`](.github/workflows/commitlint.yml) | Conventional commit enforcement | `config-file`, `registry-url` |
| [`publish-npm.yml`](.github/workflows/publish-npm.yml) | Publish to GitHub Packages | `package-filter`, `registry-url` |
| [`deploy-cloudflare.yml`](.github/workflows/deploy-cloudflare.yml) | Workers deploy + migrations | `migration-command`, `doppler-project`, scripts |
| [`preview-cloudflare.yml`](.github/workflows/preview-cloudflare.yml) | Workers PR preview upload + URL comment | `wrangler-subcommand`, `build-script`, `comment-on-pr` |
| [`deploy-terraform.yml`](.github/workflows/deploy-terraform.yml) | Plan on PR, apply on tag | `working-directory`, `plan-only`, `doppler-project` |
| [`publish-relay.yml`](.github/workflows/publish-relay.yml) | Publish release artifacts to Relay | `org`, `product`, `version`, `artifact-name`, `artifact-path`, `policy-type` |
| [`publish-testflight.yml`](.github/workflows/publish-testflight.yml) | Archive an iOS app, sign it and upload it to TestFlight | `project`, `scheme`, `version` |
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

`release-pr-check.yml` passes a title that is `chore(release): …`, or `chore: release …` with any scope or none. That covers release-please's default (`chore(main): release 1.12.0`) and the patterns the org's repos set: `chore: release ${version}`, `chore(release): ${version}` and `chore(${component}): release ${version}`.

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

The new release has the mark for a few seconds before it is moved back, since Release Please has no setting for it. The first release of a repository keeps it, there being no earlier release to give it to.

Marking `$TAG` assumes one build at a time. If builds can overlap or be re-run for an old tag, mark the newest release that has its files instead, so the mark never moves backwards.

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

### CI for an Apple App

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  pull-requests: read

jobs:
  test:
    uses: unstable-studios/.github/.github/workflows/ci-apple.yml@main
```

The repo provides `scripts/test`, which takes an iOS major version (`scripts/test 27`), picks a simulator of that version and runs the tests and builds. The workflow sets `DEVELOPER_DIR` to the matrix entry's Xcode.

- Every run tests each entry of `matrix`: by default Xcode 27.2 on iOS 27 and Xcode 26.6 on iOS 26. Pass your own JSON list to change that, or `pull-request-matrix` to test less on pull requests.
- The Mac is the org's macOS runner pool when the org variable `RUNS_ON_MACOS` is set, which private repos see. What the pool has installed is listed in `unstable-studios/echo-core` under `runners/macos`: a matrix entry must name an Xcode and an iOS that are there. Without the variable, each entry's `hosted` label picks one of GitHub's macOS runners: `xcode-27` for Xcode 27.2 and `macos-26` for Xcode 26.6 (GitHub's `xcode-27` image has no iOS 26 simulator). An entry can name a different Xcode for GitHub's runner with `hosted-xcode`.
- The caller must grant `contents: read` and `pull-requests: read` (the docs-only check lists the pull request's files); without them the run fails at startup.
- A pull request that only touches docs skips the tests and still reports success. A newer push to a pull request cancels the run before it.
- It's one job per matrix entry: on GitHub's runners every job is billed a whole minute at the macOS rate.
- The pool's runners aren't fresh machines. A job there shares a home folder, build caches and simulators with earlier jobs, and a pull request's own scripts run as the runner's account. That's why only private repos see the variable, and why `publish-testflight.yml` never uses it: the signing key would be on a runner that pull requests' code runs on.

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

The job always runs in the calling repo's `testflight` environment, which isn't an input, and reads the App Store Connect key from it: `ASC_KEY_ID` and `ASC_ISSUER_ID` as variables, `ASC_KEY_P8` as a secret. Keep the key in Doppler, in a config of its own, with a GitHub sync to that environment and "Sync unmasked secrets as variables" on. The secret has to be passed by name even though it only exists in the environment ([reusing workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)).

- The archive is unsigned (`CODE_SIGNING_ALLOWED=NO`); `xcodebuild -exportArchive -allowProvisioningUpdates` signs it, with the export options' automatic signing. That is cloud signing: no distribution certificate on the runner, and no Apple Development certificate for the archive. The key is a team key with the Admin role. Apple's engineers have said API keys only support exporting archives, not building them ([forum report, cited in a community PR](https://github.com/T0mSIlver/starbridge/pull/1416)), which is why the archive doesn't sign in.
- Xcode embeds entitlements only when it signs, and the export signs with the entitlements it finds on the archived binaries, so an unsigned archive would lose App Groups without an error ([a community report of exactly that](https://github.com/Nicolapps/let-it-go/pull/5)). The job therefore signs the app and each `PlugIns/*.appex` ad hoc (`codesign --sign -`: no certificate, no keychain) with their entitlements files, children first. It finds each target in the scheme's `xcodebuild -showBuildSettings -json` output by bundle identifier, reads `CODE_SIGN_ENTITLEMENTS`, expands `$(VAR)` references from that target's build settings, and fails if `com.apple.security.application-groups` isn't on the archived binary afterward. Everything nested is signed ad hoc too, children first (`Frameworks/*.framework` and `*.dylib`, each extension, then the app), because `codesign` needs nested code signed before its container; only targets with an entitlements file get `--entitlements`. The job fails if the app's own target resolves none (it asks for the `iphoneos` SDK, since a project can set `CODE_SIGN_ENTITLEMENTS[sdk=iphoneos*]`). This was chosen over archiving with `CODE_SIGN_IDENTITY=-`, which I found no report of working for a device archive and which keeps Xcode signing during the archive, the step the key is reported not to support; it is untried.
- The export runs twice from the same archive: first with the options' `destination` set to `export`, into a folder, where the job reads each signed binary's entitlements from the `.ipa` and fails if an application group is missing; then with the caller's options, whose `upload` sends it. Nothing reaches App Store Connect unless the check passes. (`xcrun altool --upload-app` would upload the checked file itself, but it needs the key in `~/.appstoreconnect/private_keys` or `API_PRIVATE_KEYS_DIR`, and its documentation is `man altool` only; a second `-exportArchive` reuses the flow that works.)
- The key file is written after the archive and the entitlements step, just before the export, so the key isn't on disk when the caller's build phases and package plugins run (a process they start could outlive the archive step).
- Limit the environment to the branch the caller runs on (`main` when chained off release-please). Whoever can run a workflow there can use the key.
- The environment is fixed because its branch rule is also what keeps other branches off the publish runner: the runner group limits the workflow and the repos, not the caller's branch. If a caller could name another environment, a workflow on any branch could run the job on the publish runner without the key and change the account's files before the next real upload.
- Create the `testflight` environment with its `main` branch rule before adding a repo to the `testflight-publish` runner group; GitHub creates a missing environment on first use with no protection rules ([managing environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)).
- With no `version`, it reads the root package's version from `.release-please-manifest.json`. A `workflow_dispatch` caller then uploads whatever is on the branch it ran for, under the last released version, as a new build.
- The build number is the time in UTC to the minute (`202611251630`), so it rises across re-runs, renamed workflows and several callers alike.
- The key's ID and issuer ID show in the job's log, because `xcodebuild` prints its command line. They are no use without the key file. In a public repo they would be public.
- The repo provides an export options plist (default `Config/TestFlight-ExportOptions.plist`) with `method` `app-store-connect`, `destination` `upload`, `teamID` (the unsigned archive records no team, so the export takes it from here; the job fails early without it), and its Info.plist answers the export compliance question (`ITSAppUsesNonExemptEncryption`).
- The Mac is the runner the org variable `RUNS_ON_MACOS_PUBLISH` names, and GitHub's `xcode-27` when it isn't set, with Xcode 27.2 on either. `runs-on` and `xcode-path` override both. The publish runner is one only publish jobs use, outside the CI pool (`RUNS_ON_MACOS`), so no pull request's build can leave a process behind there that reads the key. The job fails early on a CI pool runner (`echo-macaroni-<n>`).
- One runner per publish account: the keychain list and provisioning profiles are per account, so two runners sharing an account would undo each other's.
- The org runner group `testflight-publish` keeps other workflows off the publish runner, not its label: selected repositories tabstash-apple and rabbithole-apple, and workflows limited to `unstable-studios/.github/.github/workflows/publish-testflight.yml@refs/heads/main`. The calling repo's own code on the allowed branch still runs there during the archive (build phases, package plugins); the key is written only after it. `vars.RUNS_ON` doesn't apply: it names a Linux runner.
- A self-hosted runner keeps its home folder, so there the job signs with a keychain it creates in `$RUNNER_TEMP` and makes the only one on the account's search list. Its last steps (`if: always()`) delete that keychain, restore the account's keychain list and default, and remove any provisioning profile the job downloaded. On GitHub's runners signing uses the login keychain, as before.
- On a self-hosted runner the job also removes anything new in `~/Library/Keychains`. The first `xcodebuild` call can make macOS create the account's data protection keychain there (`<hardware UUID>/keychain-2.db`), which the search list doesn't cover and which may hold an Apple Development key if signing made one. `secd`, the account's keychain daemon, stays running between jobs with that database open, and its scheduled tasks can start it again after a job. So at the start the job stops `secd`, waits for it to exit and removes any data protection keychain an earlier job left; at the end it stops `secd` again around removing the new entries. launchd starts it again on demand. The step fails if a removal fails (unlinking an open SQLite database [can corrupt it](https://www.sqlite.org/howtocorrupt.html), which is why it waits for `secd` to exit first). The start-of-job list of what's already in `~/Library/Keychains` leaves out `<UUID>` folders, so one recreated after the removal is still deleted at the end.
- Cleanup on a self-hosted runner also deletes Xcode's URL caches and HTTP stores: `com.apple.dt.xcodebuild`, `com.apple.dt.Xcode.ITunesSoftwareService` and `com.apple.itunes.swinfo` under `~/Library/Caches` and `~/Library/HTTPStorages`, and `~/Library/Application Support/com.apple.dt.Xcode.ITunesSoftwareService`. A URL cache keeps request headers, which can include the App Store Connect token `xcodebuild` makes from the key ([valid up to 20 minutes](https://developer.apple.com/documentation/appstoreconnectapi/generating-tokens-for-api-requests)).
- What a self-hosted job leaves in the account's home: SwiftPM's caches (`~/Library/Caches/org.swift.swiftpm`, `~/Library/org.swift.swiftpm`, and `~/.swiftpm`, which links to them). They hold clones of the app's package dependencies, their fingerprints and parsed manifests, and no credentials; a private dependency's source would be there. Also Xcode's preferences, and `~/Library/Developer/CoreSimulator`, `~/Library/Developer/DVTDownloads`, `~/Library/Caches/com.apple.cds*`, `~/Library/Logs/CoreSimulator` and `~/Library/Containers/com.apple.CoreDevice.*`, none of which holds credentials.
- On both, the build goes to `$RUNNER_TEMP` (`-derivedDataPath`, and `TMPDIR` for distribution logs and export staging), and the key file, archive and build are deleted at the end.
- On a self-hosted runner each run's keychain is new. Signing the archive with `-allowProvisioningUpdates` used to create an Apple Development certificate ("Created via API") each run, with its private key in the job keychain, and Apple emailed the account holder about each one. Once cleanup deletes that key, such a certificate can't sign, and while it exists the next run fails ("Your account already has an Apple Development signing certificate for this machine, but its private key is not installed in your keychain"). The archive is now unsigned, so the job should make none. The step "Revoke the job's Apple Development certificate" (`if: always()`, self-hosted only) stays as a fallback in case the export still makes one. It runs before the key and the keychain are deleted, even when an earlier step failed:
  - It takes the certificates whose private key is in the job keychain (`security find-identity`), so only ones this job made, and reads their serial numbers with `openssl x509 -serial`.
  - Before the export it also lists the account's `DEVELOPMENT` certificates and keeps their ids. After, a certificate that is new and wasn't revoked above (its key went elsewhere, the data protection keychain say) fails the job with its id, instead of passing silently; it isn't revoked, since nothing shows it is the job's. With none new, the log says so.
  - It lists the account's `DEVELOPMENT` certificates ([`GET /v1/certificates`](https://developer.apple.com/documentation/appstoreconnectapi/get-v1-certificates)) and matches serials as hex, ignoring case and leading zeros: a serial is a positive integer ([RFC 5280, 4.1.2.2](https://www.rfc-editor.org/rfc/rfc5280#section-4.1.2.2)), and `openssl` prints it a byte at a time, so it can start with `0`.
  - It revokes each match ([`DELETE /v1/certificates/{id}`](https://developer.apple.com/documentation/appstoreconnectapi/delete-v1-certificates-_id_)). The account's other certificates, by name or date, are never touched. A certificate in the keychain without its key there gets a warning and stays. With no Apple Development key in the job keychain nothing is revoked, which is the expected case.
  - The token is an ES256 JWT made with `/usr/bin/openssl` and the job's key, expiring in 10 minutes ([generating tokens](https://developer.apple.com/documentation/appstoreconnectapi/generating-tokens-for-api-requests)). It's masked, sent only to `api.appstoreconnect.apple.com` and never printed.
  - If revocation fails, the job fails with an error naming the serial, and the rest of cleanup still runs. Revoke that certificate in Certificates, Identifiers & Profiles before the next upload.
- Uploads from one repo run one at a time (the concurrency group is per repo).

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
- **Runner**: All workflows respect `vars.RUNS_ON`, defaulting to `ubuntu-latest`. It's set at the org level for private repos (`unstable-studios/echo-core`, decision 0007). `ci-apple.yml` needs a Mac and uses `vars.RUNS_ON_MACOS` the same way. `publish-testflight.yml` uses `vars.RUNS_ON_MACOS_PUBLISH`, a runner only publish jobs use, and takes a `runs-on` input to override it.
- **Caching**: the pnpm store and Terraform plugin caches use GitHub's cache on GitHub-hosted runners only (`if: runner.environment == 'github-hosted'`). On self-hosted runners (echo-turbine) uploading to GitHub's cache took minutes per job; the runner keeps the pnpm store on the host instead. Follow the same rule for new `actions/cache` steps and `setup-*` `cache:` inputs.
- **Concurrency**: CI cancels in-progress; deploys and releases do not.
- **Release PRs**: Use `release-pr-check.yml` instead of full CI — the code has already passed CI on the constituent commits.
- **Secrets**: Use `secrets: inherit` to pass all secrets from the caller. Doppler integration is opt-in via `doppler-project` input.
- **Conventional commits**: Enforced via `commitlint.yml`. Repos should have `@commitlint/config-conventional` as a dev dependency.
- **Database migrations**: Use [Drizzle ORM](https://orm.drizzle.team/) with `drizzle-kit` for D1 schema management. Define schema in TypeScript, generate migrations with `drizzle-kit generate`, and run them in CI via the `migration-command` input on `deploy-cloudflare.yml`. This replaces hand-written SQL migrations and `wrangler d1 migrations apply`.
