# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) presets for the workfloworchestrator repositories
(and related orgs such as BandwidthOnDemand). See [docs/design/PLAN.md](docs/design/PLAN.md) for the design and roadmap.

## Presets

Pick one preset per question: always the base, then the language, the project type, and any extras.
The base is `default.json` at the repository root; the rest live in folders named after the question they
answer.

| Preset | Use when | What it does |
|---|---|---|
| `local>workfloworchestrator/renovate-config` | Always | Nightly schedule (22:00–06:00 Europe/Amsterdam), 3-day release age, GitHub merges once the required checks pass, GitHub Actions and pre-commit bumps grouped, weekly and automerged, runner images never automerged (see [Weekend schedule](#weekend-schedule)) |
| `…//languages/python` | uv / `pyproject.toml` | `uv` toolchain grouped weekly, `[dependency-groups]` proposed weekly and automerged, `requires-python` left alone. Only the pep621 `requires-python` update is disabled: a `python` version used by github-actions, pyenv/asdf or a Docker `python:*` image is not covered by this preset |
| `…//projects/library` | Published to a registry (PyPI, npm, Maven) | Lock file maintenance weekly and automerged: users don't install from the lock file |
| `…//projects/application` | Deployed from the repo (image, `uv run`, bundled UI) | Lock file maintenance needs review: the lock file is what runs |
| `…//extras/release-notes` | Release notes are generated from PR labels | Adds `kind/dependencies`, `kind/ci` and `dev-dependencies` labels |
| `…//extras/security-alerts` | No Dependabot security updates (see [Security updates](docs/security-updates.md)) | Renovate opens security PRs from GitHub vulnerability alerts, at any time, without the release age, labelled `kind/security` |

`…//path/name` is Renovate's syntax for "preset `name.json` in folder `path`"; see
[Preset File Naming](https://docs.renovatebot.com/config-presets/#preset-file-naming).

Main dependencies, extras and anything not listed above always need a review.

### Weekend schedule

Automerged updates are proposed in consecutive weekend slots (Europe/Amsterdam), one kind at a time.
GitHub requires a PR to be up to date with `main` before it merges, so two automerged PRs open at the
same time make one of them wait for Renovate's next run to rebase it. Each slot is 6-8 hours, so it
spans about two Renovate runs: a second chance for a PR that fell behind or hit a flaky check.

| Slot                      | Preset    | Update                                                                                                          |
|---------------------------|-----------|-----------------------------------------------------------------------------------------------------------------|
| Saturday 10:00–18:00      | base      | `GitHub Actions` group: all action bumps, majors and digests included, but not runner images |
| Saturday 18:00–24:00      | `python`  | `uv` group: `setup-uv`, its `version` input and `uv-pre-commit`                                                 |
| Sunday 00:00–08:00        | base      | `pre-commit hooks` group (majors in a separate PR)                                                              |
| Sunday 08:00–16:00        | `python`  | `[dependency-groups]`                                                                                           |
| Sunday 16:00–22:00        | base      | `noisy pre-commit hooks` group: hooks that release several times a week, such as `renovatebot/pre-commit-hooks` |
| Sunday 22:00–Monday 06:00 | `library` | Lock file maintenance, last so it builds on the other bumps                                                     |

`GitHub runner images` (`github-runner`, e.g. `ubuntu-24.04` to `ubuntu-26.04`) is not in the table: the
base preset groups those bumps separately and never automerges them. They keep the nightly window
instead of a weekend slot, so a maintainer can pick them up any weekday. A runner change makes every
job run on a different VM, including publish and release workflows that PR CI does not exercise.

`updateNotScheduled` is off, so Renovate doesn't create, update or rebase a PR outside its slot. A PR
that is still open when its slot ends waits for the next weekend. The one exception is a PR from
`extras/security-alerts`: `vulnerabilityAlerts.schedule` is `at any time`, and Renovate applies that
setting to alert PRs outside the base schedule.

### Dependency groups

`languages/python` schedules and automerges `[dependency-groups]` updates but does *not* group them: each
package keeps its own PR, so a failure points at one dependency. Runtime dependencies and extras are never
automerge, and a package that is in a group *and* a runtime dependency needs a repository-level
`automerge: false` rule (see [Overriding](#overriding)). If a shared `groupName` is ever added for these
updates, every such override must also set `groupName: null`; otherwise one manual package turns the whole
group manual.

### Python version updates

The `[disable]` rule for `python` is scoped to `matchManagers: ["pep621"]`, so it only stops Renovate from
bumping `requires-python` (managed by hand together with the CI test matrix). A version of `python` found
elsewhere is *not* disabled by this preset:

| Source | Handled by | What a repository still has to decide |
|---|---|---|
| `requires-python` in `pyproject.toml` | pep621 rule in `languages/python` | The shared preset disables it |
| `python-version` input of `astral-sh/setup-uv`/`actions/setup-python` | github-actions manager | Nothing shared: GitHub Actions bumps are grouped and automerged on Saturday |
| `.python-version`, `pyenv`, `.tool-versions` (asdf) | pyenv/asdf managers when enabled | Whether the runtime version should follow `requires-python` |
| Docker `python:*` image | dockerfile/docker-compose managers | Repository rule: disable major/minor image updates, or keep them within the supported `requires-python` range |

Scoping the rule to pep621 does not make Docker handling automatic; the last row is a per-repository
decision.

## Security updates

Repositories that let Renovate open security PRs extend `extras/security-alerts`, which reads GitHub
vulnerability (Dependabot) *alerts* and turns them into PRs with `schedule: at any time`,
`minimumReleaseAge: null` and the `kind/security` label. The base preset keeps `vulnerabilityAlerts` and
`osvVulnerabilityAlerts` off, so a repository that keeps Dependabot security updates (such as
`pydantic-forms`) does not get duplicate PRs.

Before enabling the preset:

1. Dependabot alerts must be enabled for the repository. Renovate reads them through the Mend-hosted app,
   which needs read access to alerts; check that the job log does not report a missing or denied alert
   source.
2. A fix newer than three days still hits uv's `exclude-newer` cutoff. See
   [docs/security-updates.md](docs/security-updates.md) for the remediation procedure.

Cookiecutter repositories: alerts on the generator's root manifest do not cover the shipped Jinja
template. That review stays manual; see [docs/security-updates.md](docs/security-updates.md#cookiecutter-template).

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

Repositories that should get security PRs add
`"local>workfloworchestrator/renovate-config//extras/security-alerts"` after verifying alert access (see
[Security updates](#security-updates)), and drop their Dependabot security updates so the two don't each
open a PR.

### Onboarding checklist

In order to use this config in another repository, ensure the following is in place:

1. Let Renovate's PRs pass the review requirement by enabling the `renovate-approve` and
   `renovate-approve-2` apps for the repository and configuring a [ruleset](./docs/rulesets/pull-request-reviews.md).
   WFO installs both org-wide; another organization needs its own installation.
2. Define a [CI workflow](#ci-workflow) that includes all tests/checks and ends with `"CI result"`
3. Make `CI result` a required check by configuring a [ruleset](./docs/rulesets/required-checks.md)
4. Optional: for security PRs, enable Dependabot alerts, verify the Renovate app can read them, then
   extend `//extras/security-alerts` (see [Security updates](#security-updates))

Rulesets are per repository and app installations are per organization; the presets cannot set them up.


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
| Let Renovate open security PRs (no Dependabot in this repo) | Extend `local>workfloworchestrator/renovate-config//extras/security-alerts`, or copy `"vulnerabilityAlerts": { "enabled": true, "schedule": ["at any time"], "minimumReleaseAge": null, "labels": ["kind/security"] }` and verify alert access first (see [Security updates](#security-updates)) |
| Different lock file maintenance day | `"lockFileMaintenance": { "schedule": ["before 3am on Wednesday"] }` |
| Skip the release age for your own packages | Package rule: `{ "matchPackageNames": ["nwa-stdlib"], "minimumReleaseAge": "0 days" }` |
| Ignore a file | `"ignorePaths": ["Dockerfile"]` |

Python repos: keep `[tool.uv] exclude-newer` in `pyproject.toml` equal to `minimumReleaseAge` (3 days).
A Renovate security PR does not bypass that cutoff; when the fix is younger than three days, follow
[docs/security-updates.md](docs/security-updates.md).

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

Schema validation proves the JSON is valid and every key is known; it does not prove package extraction,
grouping or schedules. The template boundary and the security procedures in
[docs/security-updates.md](docs/security-updates.md) still need a live alert check and a rehearsal in a
scratch repository before the rollout is complete.
