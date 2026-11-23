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
  "labels": ["kind/security"]
}
```

`vulnerabilityAlerts.schedule` replaces the default mode schedule for alert PRs, so they do not wait for
the nightly or weekend windows. `minimumReleaseAge: null` only lifts
*Renovate's* release age; it does not lift uv's `exclude-newer` cutoff (see below).

Repositories that keep Dependabot security updates (`dependabot.yml` with
`open-pull-requests-limit: 0`) must **not** extend this preset: both tools would open a PR for the same
advisory.

## Alert access requirement

Renovate detects vulnerabilities only through GitHub's Dependabot **alerts**. Before the preset goes
into a repository's `renovate.json`:

1. Enable Dependabot alerts for the repository (Settings → Code security, or the dependency graph).
   Alerts are a separate switch from Dependabot *security updates*: alerts can stay on while security
   updates stay off.
2. Confirm the Mend-hosted Renovate app can read them. If it cannot, the job log shows no vulnerability
   data or an access error and no security PR ever appears, even though the config validates.
3. Record the result per repository in the rollout notes.

Read-only check of the alert inventory (requires `gh` with a token that can see security alerts):

```shell
gh api --paginate repos/workfloworchestrator/REPOSITORY/dependabot/alerts \
  -f state=open -q '.[] | [.security_advisory.ghsa_id, .dependency.package.name, .security_vulnerability.vulnerable_version_range] | @tsv'
```

An empty result, a `403`, or a `404` is an alert-access finding, not privacy: check it against the
repository settings before enabling the preset. The hosted Renovate job log is the second source: it
should list the alert source it read, without an authorization error.

## Fresh fix blocked by the uv cutoff

Python repositories keep `[tool.uv] exclude-newer = "3 days"` in `pyproject.toml` so `uv lock` ignores
releases younger than three days. Renovate's `minimumReleaseAge: null` changes only Renovate's own
filter. The lock resolution is still done by `uv`, so a fix published hours ago fails during the lock
step with an artifact or lock error such as:

```
Because there is no version of PACKAGE==X.Y.Z and ...
no solution found when resolving dependencies
```

(`uv` describes it as "no solution" because the future release is excluded by `exclude-newer`; the
Renovate job log shows the `uv lock` command and its output.)

### Procedure on the remediation branch

Do this on the PR branch, not on `main`, and only for the package the advisory names:

1. Identify the package and the version the fix needs (direct, or the transitive dependency `uv` would
   need to move).
2. Add an exemption next to the cutoff in `pyproject.toml`:

   ```toml
   [tool.uv]
   exclude-newer = "3 days"
   exclude-newer-package = { PACKAGE = false }
   ```

   `false` disables the cutoff for that package only. Use the normalized PyPI name.
3. Regenerate the lock from the repository root: `uv lock`. Commit `pyproject.toml` and `uv.lock` (and
   `evals/uv.lock` where the repository has a second project) on the remediation branch.
4. Run the repository's full CI gate, not just the lock step. A security bump can change behaviour.
5. Merge through the normal review path; security PRs are not automerged by these presets.
6. Remove the `exclude-newer-package` entry once the fixed version is older than three days, in a
   follow-up commit or PR, and regenerate the lock again. Leaving exemptions behind silently weakens the
   release-age policy for that package.

Dependabot uses the same `exclude-newer` setting when it runs `uv lock`, so switching tools is not a
bypass. If both tools hit the same cutoff, use this procedure rather than adding a second bot.

## Packages capped by a pin or an override

An exemption only helps when the fix is *allowed* by the manifest. It does nothing when another
constraint excludes the fixed version: an exact `==` pin, an upper bound, or
`[tool.uv] override-dependencies`. Those need a manual manifest or override change, reviewed like any
other compatibility change:

- **Website (`workfloworchestrator.github.io`)**: FastAPI, uvicorn and Redis constraints may cap the
  package an advisory affects. Check which constraint produces the resolved version before adding an
  exemption.
- **Agent (`orchestrator-agent`)**: Pydantic and pydantic-settings constraints, and the production
  compatibility overrides, can hold a package at a vulnerable version.
- **LSO (`lso`)**: exact runtime pins and the Redis cap keep the public API stable. A security fix may
  require raising the pin deliberately, with a test run against Redis.

Treat each of these as an example to verify, not a reason to bypass. The procedure:

1. Find the constraint: exact pin, extra, or `override-dependencies`.
2. Decide whether the fixed version is compatible. If not, the advisory cannot be closed by Renovate
   alone; plan the code change.
3. Change the pin or override in the manifest, regenerate the lock, run CI, and record the reason in the
   PR description.
4. Only then use the `exclude-newer-package` exemption if the fix is still younger than three days.

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

These documents describe the procedure. The following steps still have to be performed against the real
repositories and recorded, and are **not** covered by the preset change itself:

- **Live alert check** for each repository before `extras/security-alerts` lands in its `renovate.json`:
  the `gh api … /dependabot/alerts` result and the hosted job-log check.
- **Rehearsal**: in a scratch repository or disposable branch — never a production rollout branch — pin
  a known-vulnerable version, enable the alert source, observe the security PR or the `exclude-newer`
  artifact error, apply the exemption, and confirm a usable remediation PR with green CI. Record the
  outcome and any limitations found.
