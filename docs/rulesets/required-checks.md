# About `required-checks.json`

GitHub only blocks a merge on *required* checks; a failing check that is not required still allows the merge.
This ruleset makes the `CI result` job (see the README) required on `main`. It has no bypass actors, so
neither Renovate nor anyone else can merge a PR whose `CI result` is not green.

The free tier of Github has no organization wide rulesets, so these must be created in each repository.

## Installation

Merge the CI workflow first: a required check that never reports blocks every PR.

From the root of this repository use the `gh` tool to set the main branch ruleset on the given `REPOSITORY`:

```bash
gh api -X POST repos/workfloworchestrator/REPOSITORY/rulesets --input docs/rulesets/required-checks.json
```

## About the ruleset

- `CI result` is the job name; it must match the job's `name:` exactly.
- actor_id `15368` is the GitHub Actions app, so only a workflow can report the check, not another app.
- `strict_required_status_checks_policy` requires the PR branch to be up to date with `main`, the same as
  the "Require branches to be up to date" setting of branch protection.
- Without bypass actors, repository admins can't merge a red PR either. Branch protection allows that while
  "Do not allow bypassing the above settings" is off.
