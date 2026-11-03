# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) presets for the workfloworchestrator repositories
(and related orgs such as BandwidthOnDemand). See [docs/design/PLAN.md](docs/design/PLAN.md) for the design and roadmap.

## Presets

Pick one preset per question: always the base, then the language, the project type, and any extras.
The base is `default.json` at the repository root; the rest live in folders named after the question they
answer.

| Preset | Use when | What it does |
|---|---|---|
| `local>workfloworchestrator/renovate-config` | Always | Nightly schedule (22:00–06:00 Europe/Amsterdam), 3-day release age, GitHub merges once the required checks pass, GitHub Actions and pre-commit bumps automerge, action digests batched weekly |
| `…//languages/python` | uv / `pyproject.toml` | `uv` toolchain grouped weekly, `[dependency-groups]` proposed weekly and automerged, `requires-python` left alone |
| `…//projects/library` | Published to a registry (PyPI, npm, Maven) | Lock file maintenance automerges: users don't install from the lock file |
| `…//projects/application` | Deployed from the repo (image, `uv run`, bundled UI) | Lock file maintenance needs review: the lock file is what runs |
| `…//extras/release-notes` | Release notes are generated from PR labels | Adds `kind/dependencies`, `kind/ci` and `dev-dependencies` labels |

`…//path/name` is Renovate's syntax for "preset `name.json` in folder `path`"; see
[Preset File Naming](https://docs.renovatebot.com/config-presets/#preset-file-naming).

Main dependencies, extras and anything not listed above always need a review.

## Usage

Python library or tool:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "local>workfloworchestrator/renovate-config",
    "local>workfloworchestrator/renovate-config//languages/python",
    "local>workfloworchestrator/renovate-config//projects/library",
    "local>workfloworchestrator/renovate-config//extras/release-notes"
  ]
}
```

Python application:

```json
{
  "extends": [
    "local>workfloworchestrator/renovate-config",
    "local>workfloworchestrator/renovate-config//languages/python",
    "local>workfloworchestrator/renovate-config//projects/application"
  ]
}
```

JavaScript/TypeScript and Java/Scala (no language preset yet, see [docs/design/PLAN.md](docs/design/PLAN.md)):

```json
{
  "extends": [
    "local>workfloworchestrator/renovate-config",
    "local>workfloworchestrator/renovate-config//projects/library"
  ]
}
```

### Onboarding checklist

In order to use this config in another repository, ensure the following is in place:

1. Allow Renovate to merge PRs without approvals by configuring a [ruleset](./docs/rulesets/pull-request-reviews.md)
2. Define a [CI workflow](#ci-workflow) that includes all tests/checks and ends with `"CI result"`
3. Make `CI result` a required check by configuring a [ruleset](./docs/rulesets/required-checks.md)


#### CI workflow

Renovate automerges once all checks are green, but GitHub only blocks a merge on *required* checks.
To keep the GitHub ruleset configuration simple, we define one `"CI result"` check.

Add a workflow `.github/workflows/ci.yml` as the single entrypoint and make it include all your other unit tests and checks:

```yaml
# .github/workflows/ci.yml
name: CI

# Single PR/push entry point for CI
# For PRs the final 'CI result' job must pass in order to merge
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    name: Unit tests
    uses: ./.github/workflows/test-package.yml
    # CODECOV_TOKEN for the coverage upload
    secrets: inherit

  linting-tests:
    name: Linting tests
    uses: ./.github/workflows/run-linting-tests.yml

  gate:
    name: CI result
    needs:  # every job that must pass
      - unit-tests
      - linting-tests
    runs-on: ubuntu-slim
    if: always()  # also report when a needed job failed or was cancelled
    steps:
      - name: Decide whether the needed jobs succeeded or failed
        uses: re-actors/alls-green@b5b5b37504aa4183270bd3d855c52a67f212be35 # v1.3.0
        with:
          jobs: ${{ toJSON(needs) }}
```

**Important**:

1. In the included workflow files, remove the `on: push:` and `on: pull_request:` triggers to prevent
duplicate checks. They should only (or at least) have `on: workflow_call` so they can be executed by `ci.yml`
2. Merge this workflow first before creating the ruleset that makes this mandatory

## Overriding

Rules in your own `renovate.json` come after the presets, so they win.

Keep a package that is in a dependency group *and* a runtime dependency or extra out of the weekly
automerge, and in the release notes:

```json
"packageRules": [
  {
    "description": "[automerge] Also a runtime dependency; review it and propose it nightly",
    "matchManagers": ["pep621"],
    "matchPackageNames": ["click", "sqlalchemy"],
    "automerge": false,
    "schedule": ["* 22-23 * * *", "* 0-5 * * *"],
    "labels": ["kind/dependencies"]
  }
]
```

Other common overrides:

| Goal | Setting |
|---|---|
| Let Renovate open security PRs (no Dependabot in this repo) | `"vulnerabilityAlerts": { "enabled": true, "schedule": ["at any time"], "minimumReleaseAge": null, "labels": ["kind/security"] }` |
| Different lock file maintenance day | `"lockFileMaintenance": { "schedule": ["before 3am on Wednesday"] }` |
| Skip the release age for your own packages | Package rule: `{ "matchPackageNames": ["nwa-stdlib"], "minimumReleaseAge": "0 days" }` |
| Ignore a file | `"ignorePaths": ["Dockerfile"]` |

Python repos: keep `[tool.uv] exclude-newer` in `pyproject.toml` equal to `minimumReleaseAge` (3 days).

## Testing a change

A preset reference can carry a `#ref`. Renovate resolves it through the platform's file API, which
accepts a *branch* as well as a tag, so one repository can try a preset change before it lands on
`main`:

```json
{
  "extends": [
    "local>workfloworchestrator/renovate-config#my-branch",
    "local>workfloworchestrator/renovate-config//languages/python#my-branch",
    "local>workfloworchestrator/renovate-config//projects/library#my-branch"
  ]
}
```

The `#ref` is not inherited, so put it on every entry. The branch has to exist on the remote *before*
the consuming repository's next Renovate run, and Renovate never bumps the ref itself: replace
`#my-branch` with nothing (follow `main`) once the change is merged. A repository pinned to a branch
breaks if that branch is deleted, so don't leave a pin behind.

## Contributing

A change on `main` applies to every repository at its next Renovate run. Run `prek run --all-files`
(CI does the same), and try changes on one repository first.

Every package rule's `description` starts with one tag: `[automerge]`, `[cooldown]`, `[disable]`,
`[group]`, `[labels]` or `[schedule]`.
