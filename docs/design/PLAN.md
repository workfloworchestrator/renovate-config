# Plan

## Goal

One shared Renovate configuration for all repositories in workfloworchestrator and BandwidthOnDemand,
whatever the language:

- **Less noise.** Auto-merge what can't affect users, group what moves together, and schedule what isn't
  urgent. In the WFO Python libraries, Renovate made 162 of 365 commits to `main` from 2026-07-29 to
  2026-10-29, mostly CI-only bumps.
- **One place to change policy**, instead of copying the same rules into every `renovate.json`. The NSI
  repos repeat the same 5–6 rules today.
- **The same experience for maintainers** in every repository: same schedule, same labels, same
  auto-merge rules, one tool.

## Design: one preset per question

Combining languages (Python, JS, JVM) with project types (library, application) would mean six presets
with a lot of duplication. Instead, each repository answers a few independent questions and extends
one preset per answer:

| Question | Presets | Status |
|---|---|---|
| Base (always) | `default` | ✅ |
| Which language? | `python` | ✅ |
| | `javascript`, `jvm` | Planned |
| Is the lock file shipped? | `library` (no, lock file maintenance auto-merges), `application` (yes, needs review) | ✅ |
| Release notes from PR labels? | `release-notes` | ✅ |
| Security PRs from GitHub alerts? | `security-alerts` (opt-in; requires verified alert access) | ✅ |
| Helm chart with our own image? | `helm` | Planned |

### Repository layout

The base stays at the root as `default.json`, so the bare reference
`local>workfloworchestrator/renovate-config` and org-level onboarding keep working. Every other preset
lives in a folder named after the question it answers, and repositories reach it with Renovate's
`//path` syntax:

```
default.json                  # base, always
languages/{python,javascript,jvm}.json
projects/{library,application}.json
extras/{release-notes,security-alerts,helm}.json
```

For example `local>workfloworchestrator/renovate-config//languages/python` loads `languages/python.json`.
Folders are for readability: `//path` is only needed once a preset leaves the root, and a flat repository
would work the same way.

The library/application split works for every ecosystem, because it is about what users install:

| Ecosystem | Library: lock file not shipped | Application: lock file ships | Dev-only dependency type | Watch out for |
|---|---|---|---|---|
| Python/uv | Packages on PyPI, tools run with `uvx` | Images built with `uv sync`, `uv run` from a checkout | `dependency-groups` | Packages in a group *and* in the runtime dependencies or an extra |
| npm | Packages on npm (`package-lock.json` isn't published) | Bundled apps, images | `devDependencies` | Bundlers put `devDependencies` into the build, so they aren't dev-only in apps |
| Maven/sbt | No lock file | No lock file | `test` scope | Dependencies only on GitHub Packages are invisible to Renovate |

Hybrids, like a library that also ships a container, combine presets and add repo-level rules. Rules in
a repository's own `renovate.json` come after the presets, so they win.

### Planned presets

- **`javascript`**
  - Group Node.js across the image and CI, and leave Node majors alone (nsi-orchestrator-ui does this today).
  - Auto-merge only dev tooling (`packages:linters`, `packages:jsUnitTest`, `@types/**`), not all
    `devDependencies`.
  - Decide on `rangeStrategy`: applications pin (nsi-orchestrator-ui uses `pin`), published packages keep ranges.
- **`jvm`**
  - Leave `java-version` majors alone.
  - Add the sbt-plugin → Maven Central workaround, currently copied in nsi-safnari and play-nsi-support.
- **`helm`**
  - Don't pin digests for our own images in `helm-values`: `ghcr.io/workfloworchestrator/**` and
    `ghcr.io/bandwidthondemand/**`. Every NSI/BoD repo has this rule, each with its own image name.

## Repositories

Target `extends` per repository. Everything starts with `default`.

| Repository | Today | Target |
|---|---|---|
| nwa-stdlib, oauth2-lib, pydantic-forms | Renovate + Dependabot (security) | `python`, `library`, `release-notes` |
| orchestrator-core | Renovate + Dependabot (security) | `python`, `library`, `release-notes`; repo rules for `click`, `sqlalchemy`, `pytest`, `testcontainers` and `urllib3-mock` (also runtime or in the `testing` extra), a `sqlalchemy` + `alembic` group, release age 0 for own libraries |
| orchestrator-shell, lso | Renovate | `python`, `library`, `security-alerts`; shell adds Ruff/tests before broader automerge; LSO keeps lock maintenance manual because runtime-imported Starlette can move transitively |
| orchestrator-cookiecutter | Renovate | `python`, `library`, `security-alerts`; root-only automation with literal `cookiecutter.package_name` ignore and recurring manual review of rendered Threadpool/Celery variants |
| aruba-client | None | `python`, `library`, `security-alerts`; explicit `requirements/docs.txt` manager pattern and manual runtime/extra updates |
| workfloworchestrator.github.io | None | `python`, `application`, `security-alerts`; deployment consumes the lock, so lock maintenance stays manual |
| example-orchestrator, orchestrator-agent | None | `python`, `application`, `security-alerts`; runtime and application lock updates stay manual |
| nsi-orchestrator, nsi-aura, nsi-auth, nsi-dds-proxy, nsi-aggregator-proxy, nsi-mgmt-info, ana-automation-ui | Renovate (NSI style) | `python`, `application`, `helm` |
| orchestrator-ui-library, pydantic-forms-ui | Dependabot (orchestrator-ui-library) / none | `javascript`, `library`; orchestrator-ui-library's `apps/*` share the lock file, so check whether `library` fits |
| example-orchestrator-ui, orchestrator-ui-app, nsi-orchestrator-ui | None / Renovate (NSI style) | `javascript`, `application` (+ `helm`) |
| nsi-dds, nsi-pce, nsi-safnari | Renovate (NSI style) | `jvm`, `application`, `helm` |
| nsi-common-lib, nsi-dds-lib, play-nsi-support | Renovate (NSI style) | `jvm`, `library` |
| gnmic (Go), gnmic-cluster-chart, pynso-restconf, orchestrator-ui | Dependabot or none | To decide: move over, or archive if unmaintained |

## Security updates

**For now, Dependabot handles security PRs in the WFO Python libraries.** nwa-stdlib, oauth2-lib,
pydantic-forms and orchestrator-core have a `dependabot.yml` with `open-pull-requests-limit: 0` and the
`kind/security` label. That turns off version updates but keeps security PRs. To avoid duplicate PRs, the
base preset turns off Renovate's own security PRs (`vulnerabilityAlerts`, `osvVulnerabilityAlerts`).

Repositories without Dependabot security updates extend `extras/security-alerts` as the **last** preset
instead, which enables `vulnerabilityAlerts` with `schedule: ["at any time"]`, `minimumReleaseAge: null`,
the `kind/security` label and `automerge: false`. `vulnerabilityAlerts` is a repo-level object that
merges with the base preset's `{ "enabled": false }`, so it has to come after the base; a repository can
still override a single key, such as `automerge: true`, in its own `renovate.json`. It is opt-in, so a
repository that keeps Dependabot security updates keeps the base behaviour and no duplicate PRs are
opened. The procedure for a fix the release-age cutoff blocks is in
[docs/security-updates.md](../security-updates.md).

`automerge: false` is deliberate: the base github-actions/pre-commit automerge rule can match a
vulnerability PR, so without it a security fix could merge on PR CI alone. Security bumps change the version
of a package that is known to be broken and deserve a human review.

Things to check before enabling it:

- The dependency graph is on, Dependabot **alerts** are on, and the Mend-hosted Renovate app has `read`
  access to Dependabot alerts in every repository. This is a live check per repository (alerts API and
  hosted job log), not something the preset can prove. A personal `gh api` success does not prove app
  access, and an empty alert list is a normal result rather than an access failure.
- Renovate always creates vulnerability PRs regardless of `schedule` and the hourly limits, so it reacts
  on its next run, every few hours on the hosted app, instead of within minutes.
- Python: `uv lock` still respects `exclude-newer`, so a fix younger than 3 days needs an
  `exclude-newer-package` exemption, whichever tool opens the PR. A direct requirement capped by an exact
  pin or a transitive package held by `override-dependencies` needs a manifest change as well.
- Cookiecutter's Jinja template is outside root extraction: alerts on the generator root do not cover the
  dependencies of rendered projects. Review both rendered variants manually and propagate fixes to the
  template until rendered automation exists.
- Alert access, the ten-repository dry-runs and the `exclude-newer` rehearsal are not verified yet; they
  are evidence still to be produced, not results of this change.

## Moving repositories off Dependabot

Repositories that rely completely on Dependabot get more out of Renovate:

- shared presets instead of a separate `dependabot.yml` per repository
- per-package rules, groups and schedules
- built-in auto-merge and lock file maintenance
- the same experience as every other repository

Steps per repository:

1. Add a `renovate.json` that extends the presets.
2. Remove Dependabot version updates, or set `open-pull-requests-limit: 0` while it still handles
   security PRs. Otherwise both tools open PRs for the same bumps.
3. Check workflows for `dependabot/**` branch filters and add `renovate/**` next to them. nsi-dds, nsi-pce
   and nsi-safnari only ignore `dependabot/**` in their container build-and-push workflow.

## Open decisions

1. **How much auto-merges.** The NSI repos auto-merge all minor and patch updates in one group, including
   runtime dependencies and application lock files, protected only by CI and an 8-day release age. The
   WFO repos review every runtime change. Options: one policy for all, or an opt-in preset such as
   `automerge-non-major`.
2. **Release age: 3 or 8 days.** Python repos must keep `[tool.uv] exclude-newer` in sync.
3. **Schedule.** The NSI repos run whenever Renovate runs; the WFO repos run in nightly and weekend
   windows.
4. **`config:best-practices` or `config:recommended`.** best-practices includes action digest pinning,
   adds Docker digest pinning, and pins npm `devDependencies` to exact versions. That suits applications
   but needs checking for published npm packages. The NSI repos use it already.
5. **How the NSI repos merge.** They leave `platformAutomerge` at its default, so GitHub merges once the
   *required* checks pass. If tests aren't required checks there, a PR can merge before its tests finish.
   The base preset avoids this with `platformAutomerge: false`.
6. **Preset versioning.** Repositories currently follow `main`, so every change applies everywhere at
   once. The alternative is tagged releases (`…:python#v1.0.0`) that Renovate itself bumps per repository.
   That's safer, but slower to roll out.

## Roadmap

1. **This repository**: presets distilled from nwa-stdlib.
2. **Pilot**: oauth2-lib replaced its own rules with `extends` on 2026-10-29, pinned to the
   `initial-presets` branch (see the README). Renovate picked the presets up with no config errors.
   nwa-stdlib follows once its auto-merge setup has run for a few weeks.
3. **WFO Python repositories**, including orchestrator-core with its repo rules and the repos without a
   config. Each repo needs the Renovate approve apps and the review ruleset, see the onboarding
   checklist in the README.
4. **Decide 1–5**, then the Python applications (NSI repos, example-orchestrator, orchestrator-agent) and
   the `helm` preset.
5. **`javascript` preset** and the JS/TS repositories. orchestrator-ui-library drops its Dependabot
   version updates.
6. **`jvm` preset** and the BandwidthOnDemand repositories.
7. **Security updates** per repository: verify alert access, rehearse the `exclude-newer` procedure from
   [docs/security-updates.md](../security-updates.md), then replace Dependabot security updates with
   `extras/security-alerts`. Cookiecutter's template review stays manual.
8. **Dashboard across repositories**: a nightly workflow here that lists open Renovate PRs per
   repository (failing, waiting for review, older than 14 days, config errors) in one issue or page.

## Notes

- **`labels` vs `addLabels`.** A later rule replaces `labels`, but `addLabels` accumulates and can't be
  undone. `release-notes` therefore sets the `dev-dependencies` label with `labels`, so a repository can
  take it off packages that are also runtime dependencies.
- **Python version scope.** `languages/python` disables the pep621 `python` update, which is the
  `requires-python` field. The base preset disables the other Python version sources it can reach: the
  pyenv manager (on by default, `/(^|/)\.python-version$/`) and the github-actions `uses-with` inputs
  named `python` (such as `python-version`). Docker `python:*` images and `.tool-versions` (asdf) are not
  covered; Docker major/minor handling stays a repository rule, constrained to the supported
  `requires-python`.
- **Dependency groups.** The Python preset schedules and automerges `[dependency-groups]` but does not
  group them. If a `groupName` is added later, each runtime-overlap override must set
  `groupName: null`, or one manual package makes the whole group manual.
- **Schedules.** `updateNotScheduled: false` means Renovate does nothing to a PR outside its schedule,
  including rebasing it. GitHub (`platformAutomerge`) still merges a green, up to date PR at any time. The
  automerged updates each get their own weekend slot (see the README), because the branch protection
  requires PRs to be up to date: two automerged PRs open at once make one wait hours for a rebase.
  Vulnerability alerts are the documented exception: `extras/security-alerts` sets
  `vulnerabilityAlerts.schedule: ["at any time"]`, and Renovate always creates vulnerability PRs
  regardless of schedule and hourly limits.
- **Runner images.** The base preset automerges the `GitHub Actions` group, but `github-runner`
  dependencies (`ubuntu-24.04` in `runs-on`) are excluded: a separate `GitHub runner images` group,
  `automerge: false`, nightly window (no weekend slot). A new runner image changes the VM every job runs
  on, including publish and release workflows that PR CI does not exercise, so a maintainer merges it.
- **Cross-org use.** `local>` means "same platform", not "same org". This repository is public, so
  BandwidthOnDemand repositories can extend it too. The ruleset templates are per repository, and the
  approve apps are per organization: WFO installs them org-wide, so a repository in another organization
  needs its own installation (see the README).
- **Validation.** `renovate-config-validator` treats files given on the command line as *global*
  self-hosted config unless you pass `--no-global`. This repository, nwa-stdlib and oauth2-lib pass it.

## Housekeeping for this repository

- Add a LICENSE (the other WFO repositories use Apache-2.0).
- Protect `main`: require reviews and make `Validate presets` a required check. A broken preset stops
  Renovate in every repository.
- Optionally add a CODEOWNERS file, so the maintainers of the org's Renovate setup review every change.
