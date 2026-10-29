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
extras/{release-notes,helm}.json
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
| orchestrator-shell, lso | Renovate | `python`, `library` |
| orchestrator-cookiecutter | Renovate | `python`, `library`; repo rule keeping `{{ cookiecutter.package_name }}/**` manual |
| aruba-client, workfloworchestrator.github.io | None | `python`, `library` (the site's lock file only builds the docs) |
| example-orchestrator, orchestrator-agent | None | `python`, `application` |
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

**Later**, this can move to Renovate:

- Keep GitHub's Dependabot *alerts* on as the detection source, and turn off Dependabot security
  *updates*.
- Enable `vulnerabilityAlerts` in `default`, with `schedule: ["at any time"]`,
  `minimumReleaseAge: null` and the `kind/security` label.
- Remove the `dependabot.yml` files.

Things to check first:

- The Renovate app needs read access to Dependabot alerts in every repository.
- Renovate reacts on its next run, every few hours on the hosted app, instead of within minutes.
- Python: `uv lock` still respects `exclude-newer`, so a fix younger than 3 days needs an
  `exclude-newer-package` exemption, whichever tool opens the PR.

A repository without Dependabot can already opt in; see the README.

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
2. **Pilot**: nwa-stdlib replaces its own rules with `extends`, once its auto-merge setup has run for a
   few weeks.
3. **WFO Python repositories**, including orchestrator-core with its repo rules and the repos without a
   config. Each repo needs the Renovate app as a review-bypass actor.
4. **Decide 1–5**, then the Python applications (NSI repos, example-orchestrator, orchestrator-agent) and
   the `helm` preset.
5. **`javascript` preset** and the JS/TS repositories. orchestrator-ui-library drops its Dependabot
   version updates.
6. **`jvm` preset** and the BandwidthOnDemand repositories.
7. **Dependabot-only repositories**, and security updates to Renovate.
8. **Dashboard across repositories**: a nightly workflow here that lists open Renovate PRs per
   repository (failing, waiting for review, older than 14 days, config errors) in one issue or page.

## Notes

- **`labels` vs `addLabels`.** A later rule replaces `labels`, but `addLabels` accumulates and can't be
  undone. `release-notes` therefore sets the `dev-dependencies` label with `labels`, so a repository can
  take it off packages that are also runtime dependencies.
- **Schedules.** `updateNotScheduled: false` means Renovate does nothing to a PR outside its schedule,
  merging included. Every schedule must therefore fall inside the nightly `automergeSchedule`. The weekly
  windows (Saturday and Sunday night) give a PR a second chance if a flaky check stalls the first night.
- **Cross-org use.** `local>` means "same platform", not "same org". This repository is public, so
  BandwidthOnDemand repositories can extend it too.
- **Validation.** `renovate-config-validator` treats files given on the command line as *global*
  self-hosted config unless you pass `--no-global`. This repository passes it. The hook in nwa-stdlib
  should too.

## Housekeeping for this repository

- Add a LICENSE (the other WFO repositories use Apache-2.0).
- Protect `main`: require reviews and make `Validate presets` a required check. A broken preset stops
  Renovate in every repository.
- Optionally add a CODEOWNERS file, so the maintainers of the org's Renovate setup review every change.
