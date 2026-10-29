# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) presets for the workfloworchestrator repositories
(and related orgs such as BandwidthOnDemand). See [docs/design/PLAN.md](docs/design/PLAN.md) for the design and roadmap.

## Presets

Pick one preset per question: always the base, then the language, the project type, and any extras.
The base is `default.json` at the repository root; the rest live in folders named after the question they
answer.

| Preset | Use when | What it does |
|---|---|---|
| `local>workfloworchestrator/renovate-config` | Always | Nightly schedule (22:00–06:00 Europe/Amsterdam), 3-day release age, Renovate merges only when all checks pass, GitHub Actions and pre-commit bumps automerge, action digests batched weekly |
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

### Required repository settings

Renovate can't approve PRs. If `main` requires reviews, add the Renovate app as a review-bypass actor,
otherwise automerge never happens.

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

The first pilot, [oauth2-lib#144](https://github.com/workfloworchestrator/oauth2-lib/pull/144) on
2026-10-29, pinned `python`, `library` and `release-notes` to `initial-presets` this way. The next
Renovate run picked the presets up with no config errors, autoclosed the three PRs the presets now
group or schedule, and left the rest of the dashboard as expected.

## Contributing

A change on `main` applies to every repository at its next Renovate run. Run `prek run --all-files`
(CI does the same), and try changes on one repository first.

Every package rule's `description` starts with one tag: `[automerge]`, `[cooldown]`, `[disable]`,
`[group]`, `[labels]` or `[schedule]`.
