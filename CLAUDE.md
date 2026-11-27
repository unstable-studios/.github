# CLAUDE.md

## Repository Overview

This is the **org-level `.github` repo** for Unstable Studios. It contains reusable GitHub Actions workflows called by all repos in the org.

## Structure

```
.github/workflows/     # Reusable workflows (workflow_call trigger)
.github/actions/       # Composite actions shared by the workflows (e.g. setup-pnpm)
README.md              # Usage docs and examples
```

## Conventions

- All workflows use `on: workflow_call` — they are never triggered directly
- Every workflow respects `vars.RUNS_ON` with a default of `ubuntu-latest`, except the two that need a Mac with Xcode: `ci-apple.yml` uses `vars.RUNS_ON_MACOS` and falls back to the GitHub runner named in its matrix, and `publish-testflight.yml` uses `vars.RUNS_ON_MACOS`, falling back to `xcode-27`, unless its `runs-on` input names a runner. Jobs on the macOS pool share a home folder with earlier jobs: anything a job writes outside `$RUNNER_TEMP` (keychains, DerivedData, profiles) it must remove in an `always()` step
- GitHub's cache (`actions/cache`, `setup-*` `cache:`) only on GitHub-hosted runners: `if: runner.environment == 'github-hosted'`. Uploading from the self-hosted echo-turbine runners takes minutes
- pnpm is the standard package manager across the org
- Concurrency: CI jobs cancel in-progress; deploy/release jobs do not
- All user-controlled inputs (PR titles, branch names) are passed via `env:` variables, never interpolated directly in `run:` blocks
- Doppler integration is opt-in via `doppler-project` input
- Release PRs skip full CI and use `release-pr-check.yml` instead

## Editing Workflows

When modifying workflows:
- Ensure all `workflow_call` inputs have descriptions and sensible defaults
- Never introduce required secrets that aren't clearly documented
- Test with `act` or a test repo before pushing to main
- Keep backwards compatibility — adding inputs is safe, removing/renaming is breaking
