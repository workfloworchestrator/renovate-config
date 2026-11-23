# Security updates

How a security update reaches a repository that extends `extras/security-alerts`, and what to do when
the three-day release policy blocks a fresh fix.

The base preset leaves Renovate's own security PRs off (`vulnerabilityAlerts: false`,
`osvVulnerabilityAlerts: false`). A repository opts in by extending
`local>workfloworchestrator/renovate-config//extras/security-alerts`, which sets:

```json
"vulnerabilityAlerts": {
  "enabled": true,
  "schedule": ["at any time"],
  "minimumReleaseAge": null,
  "labels": ["kind/security"],
  "automerge": false
}
```

**Extend it after the base preset**, in the same order as every other preset:

```json
{
  "extends": [
    "local>workfloworchestrator/renovate-config",
    "local>workfloworchestrator/renovate-config//languages/python",
    "local>workfloworchestrator/renovate-config//projects/library",
    "local>workfloworchestrator/renovate-config//extras/security-alerts"
  ]
}
```

`vulnerabilityAlerts` is a repo-level object that Renovate merges with the base preset's
`{ "enabled": false }`. A later `extends` entry wins, and so does a `vulnerabilityAlerts` object in the
repository's own `renovate.json`, which is why the preset has to come after the base and why a repository
can still override a single key.

What the preset does and does not do:

- Renovate always creates vulnerability PRs regardless of `schedule`, `prHourlyLimit`,
  `commitHourlyLimit` or the concurrent limits; only the concurrent limits can be set inside
  `vulnerabilityAlerts`. `schedule: ["at any time"]` is therefore documentation of that behaviour.
- `minimumReleaseAge: null` removes *Renovate's* release age for these PRs. It does not lift uv's
  `exclude-newer` cutoff (see below).
- `automerge: false` is a deliberate safety choice. Without it, the base "[automerge] Merge github
  actions and pre-commit bumps" rule can match a vulnerability PR when the vulnerable package is a
  github-actions or pre-commit dependency, and other shared automerge rules can match it by package.
  A security bump changes the resolved version of a package that is known to be broken, so it is reviewed
  by a person. Repositories that want Renovate to automerge security PRs opt in explicitly in their own
  `renovate.json` (`"vulnerabilityAlerts": { "automerge": true }`) and accept the risk of a bad or
  looping fix.

Repositories that keep Dependabot security updates (`dependabot.yml` with
`open-pull-requests-limit: 0`) must **not** extend this preset: both tools would open a PR for the same
advisory.

## Alert access requirement

Renovate reads GitHub's Dependabot **alerts**. All three of these must hold before the preset goes into a
repository's `renovate.json`:

1. **Dependency graph** is on for the repository (Settings → Code security → Dependency graph). Without
   it GitHub produces no alerts at all.
2. **Dependabot alerts** are on (Settings → Code security → Dependabot alerts). This is a separate switch
   from Dependabot *security updates*: alerts can stay on while security updates stay off.
3. **The hosted Renovate app can read them.** On github.com/apps/renovate → Configure, the permissions
   list must include `read` access to "Dependabot alerts". The org-level installation propagates to
   repositories that opt in, so a single repository can also be missing the app entirely.

Checking alerts with the GitHub API:

```shell
gh api --method GET repos/workfloworchestrator/REPOSITORY/dependabot/alerts -f state=open \
  --jq '.[] | [.security_advisory.ghsa_id, .dependency.package.name, .security_vulnerability.vulnerable_version_range] | @tsv'
```

or with the filter in the path, `gh api "repos/workfloworchestrator/REPOSITORY/dependabot/alerts?state=open" --jq ...`.

- Use `--method GET` (or put `?state=open` in the path). Plain `gh api -f state=open ...` switches the
  request to `POST`, which is not what you want here.
- An **empty list is a normal result**, not an access failure: a repository can have no open alerts.
- A **successful personal check does not prove the app can read alerts.** `gh` uses your own token, which
  usually has a wider scope than the app installation. It proves that alerts exist and that the API works,
  nothing about the Mend-hosted job.
- Failures are `403` (token lacks the `security_events`/alerts scope, or you cannot see the repository)
  and `404` (alerts disabled, or not accessible to the token). Record the exact response per repository;
  a `404` with alerts visibly enabled in the UI is an app or token access finding.

The second source is the **hosted Renovate job log** for the repository:

- With alert access, a run over a repository that has open alerts logs vulnerability lookups and, when a
  fix exists, creates the `[SECURITY]` PRs; the dependency dashboard can list vulnerability counts
  (`dependencyDashboard.osvVulnerabilitySummary` covers OSV, not Dependabot alerts).
- Without alert access, the log shows no vulnerability data at all and no security PR ever appears, even
  though the config validates. A config validation run proves nothing about alert access.

Record dependency graph, alerts and app permission per repository before enabling the preset, and treat
the live check and the rehearsal below as evidence that still has to be produced.

## Fresh fix blocked by the uv cutoff

Python repositories keep `[tool.uv] exclude-newer = "3 days"` in `pyproject.toml` so `uv lock` ignores
distributions uploaded in the last three days. Renovate's `minimumReleaseAge: null` changes only
Renovate's own filter. The lock resolution is still done by `uv`, and `uv lock` fails with:

```
error: No solution found when resolving dependencies
  cause: Because there is no version of idna==3.4 and your project depends on idna==3.4, we can conclude that your project's requirements are unsatisfiable.

hint: `idna` was filtered by `exclude-newer` to only include packages uploaded before 2021-01-01T00:00:00Z. The requested version, v3.4, was published at 2022-09-14T00:24:23.22Z. Consider using `exclude-newer-package` to override the cutoff for this package.
```

(Reproduced locally with uv 0.13.4 on a project pinned to a version older than its `exclude-newer`
date; the names and dates differ per case.) The `hint` line is the marker: it names the package that the
cutoff filtered. Renovate reports this as a branch/artifact error.

An artifact error is not always attached to a PR. If Renovate cannot produce an update at all, the
failure appears in the dependency dashboard or the job log (as an "artifact update" failure) without a
PR branch, so there is nothing to amend. If a PR branch exists but is red, use it.

`uv` answers to `--locked`/`--frozen` in CI: the lock must be regenerated, not just edited. The
`exclude-newer-package = { package = false }` form (a package set to `false` is exempt from the global
cutoff) is supported by current uv releases; the repository's pinned uv version must be new enough to
accept it, which is the uv baseline from the rollout's uv-upgrade task. Do not assume an older uv
understands `false`; upgrade first if `uv lock` reports the key or value as invalid.

### Procedure

1. **Decide where to work.** Preferred: a branch from `main` carrying the lock change, reviewed like any
   other PR. Alternatively, take over the Renovate branch (`git fetch origin && git checkout
   renovate/PACKAGE-vulnerability`), which is useful when the PR exists and only the artifact is red. Do
   not create the exemption on `main` directly.
2. **Identify the package** named in the `exclude-newer` hint: the direct dependency, or the transitive
   dependency uv needs to move.
3. **Merge into the existing exemption table** if one is already there; do not replace it, or you remove
   earlier exemptions:

   ```toml
   [tool.uv]
   exclude-newer = "3 days"
   exclude-newer-package = { package_a = false, PACKAGE = false }
   ```

   A package that also has a *date* entry (`PACKAGE = "2026-11-01T00:00:00Z"`) is changed, not appended:
   TOML cannot list the same key twice.
4. **Regenerate the lock** from the repository root: `uv lock`. Commit `pyproject.toml` and `uv.lock`
   (and `evals/uv.lock` where the repository has a second project).
5. **Run the full CI gate**, not just the lock step. A security bump can change behaviour.
6. **Merge through the normal review path.** The preset does not automerge security PRs.
7. **Remove the exemption** once the fixed version is older than three days, in a follow-up commit or PR,
   and regenerate the lock again. Leaving exemptions behind weakens the release-age policy for that
   package.

Dependabot runs `uv lock` with the same `exclude-newer` configuration, so it hits the same cutoff;
switching tools is not a bypass.

## Packages capped by a pin or an override

An exemption only helps when the fixed version is *allowed* by the manifest. It does nothing when a
direct requirement excludes it: an exact `==` pin, an upper bound, an extra that pins, or
`[tool.uv] override-dependencies` (overrides are absolute and replace the requirements of the
transitive packages that depend on it). Those need a reviewed manifest/override change:

1. Find the constraint: the direct requirement, an extra, or `override-dependencies`.
2. Decide whether the fixed version is compatible. If not, the advisory cannot be closed by Renovate
   alone; plan the code change.
3. Change the pin or override, regenerate the lock, run CI, record the reason in the PR description.
4. Only then use `exclude-newer-package` if the fix is still younger than three days.

Examples to verify in the repository, not assumptions to bypass:

- **Website (`workfloworchestrator.github.io`)**: FastAPI and uvicorn are constrained here; check which
  requirement or extra caps the package an advisory names. Redis in the site stack has no version cap
  worth relying on for security purposes - do not add an exemption on the assumption that it does.
- **Agent (`orchestrator-agent`)**: Pydantic and pydantic-settings constraints, plus the production
  compatibility overrides, can hold a package at a vulnerable version.
- **LSO (`lso`)**: exact runtime pins and a Redis version chosen for compatibility with the LSO
  API/tests. A security fix may require raising the pin deliberately, with a test run against Redis.

A constraint on a *transitive* package is a different case: it is resolved by `exclude-dependencies`,
`override-dependencies` or a newer parent, and no manifest pin needs to move.

## Cookiecutter template

`orchestrator-cookiecutter` ships a Jinja template that is not a manifest Renovate can parse. Alerts on
the generator's root `pyproject.toml`/`uv.lock` cover the generator itself, **not** the dependencies the
template declares for generated projects. Until rendered-project automation exists:

- Perform a recurring manual review (at least monthly, and after each generator release) of both
  rendered executor variants: render the template with each supported executor option, inspect their
  dependency files and lock files, and check them against open advisories.
- Record the date, the reviewer, the rendered variants, and any findings in the generator repository.
- Propagate remediations back to the template (and its constraints) rather than only into the rendered
  output.
- Do not describe template coverage as automated in the final audit: this is a documented, manual
  boundary.

## Verification status

This document describes the procedure. The following evidence is still outstanding and is **not**
produced by the preset change:

- **Live alert check** per repository: dependency graph on, Dependabot alerts on, app permission with
  `read` on Dependabot alerts, the `gh api ... dependabot/alerts` result, and the hosted job-log check.
- **Rehearsal** in a scratch repository or disposable branch (never a production rollout branch): pin a
  known-vulnerable version, enable the alert source, observe the security PR or the `exclude-newer`
  artifact error, apply the exemption, and confirm a usable remediation PR with green CI. Record the
  outcome and any limitations.
- **Ten-repository dry-runs** of the shared presets, with extraction and effective rules inspected.
