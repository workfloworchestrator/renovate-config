# About `required-checks.json`

GitHub only blocks a merge on *required* checks; a failing check that is not required still allows the merge.
This ruleset makes the `CI result` job (see the README) required on `main`. It has no bypass actors, so
neither Renovate nor anyone else can merge a PR whose `CI result` is not green.

The free tier of Github has no organization wide rulesets, so these must be created in each repository.

## Installation

Merge the CI workflow first: a required check that never reports blocks every PR.

From the root of this repository use the `gh` tool to create the main branch ruleset in `REPOSITORY`:

```bash
REPOSITORY=nwa-stdlib
gh api -X POST "repos/workfloworchestrator/${REPOSITORY}/rulesets" --input docs/rulesets/required-checks.json
```

To update an existing ruleset, look up its id by name and replace it:

```bash
REPOSITORY=nwa-stdlib
RULESET_ID=$(gh api "repos/workfloworchestrator/${REPOSITORY}/rulesets" --jq '.[] | select(.name == "Required checks") | .id')
gh api -X PUT "repos/workfloworchestrator/${REPOSITORY}/rulesets/${RULESET_ID}" --input docs/rulesets/required-checks.json
```

## About the ruleset

- `CI result` is the job name; it must match the job's `name:` exactly.
- integration_id `15368` is the GitHub Actions app, so only a workflow can report the check, not another app.
- Without bypass actors, repository admins can't merge a red PR either. Branch protection allows that while
  "Do not allow bypassing the above settings" is off.
- `strict_required_status_checks_policy` is off: a bypass covers every rule in a ruleset, so requiring an
  up-to-date branch here would stop admins from merging an out-of-date PR too. Keep "Require branches to be
  up to date before merging" on in the branch protection of `main` instead, where admins can bypass it.
