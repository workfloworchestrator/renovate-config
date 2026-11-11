# About `required-checks.json`

GitHub only blocks a merge on *required* checks; a failing check that is not required still allows the merge.
This ruleset makes the `CI result` job (see the README) required on `main`. The `core-maintainers` team may
bypass it; nobody else can merge a PR whose `CI result` is not green.

The bypass exists because a required check can never report on a *direct push*: GitHub evaluates the check
against the pushed commit before the `on: push` workflow starts, so the push is rejected for the lack of a
status that only that push could have produced. Without a bypass, no release version bump could be pushed
to `main` at all, and the only way to land one would be to temporarily disable the ruleset.

The free tier of Github has no organization wide rulesets, so these must be created in each repository.

## Installation

1. Merge the CI workflow first: a required check that never reports blocks every PR.

2. From the root of this repository use the `gh` tool to create the main branch ruleset in `REPOSITORY`:

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

3. Set `bypass_actors` to the team that maintains the repository. `4138115` is `core-maintainers`, which
   covers the WFO repositories. A team in another organization needs its own id:

   ```bash
   gh api "orgs/BandwidthOnDemand/teams/OTHER_TEAM" --jq .id
   ```

   A repository whose maintainers never push to `main` can leave the list empty: a required check is
   enforced for everyone then, which is the strictest option.

## About the ruleset

- `CI result` is the job name; it must match the job's `name:` exactly.
- integration_id `15368` is the GitHub Actions app, so only a workflow can report the check, not another app.
- actor_id `4138115` is the `core-maintainers` team, the same bypass the review ruleset gives it. It lets
  maintainers push a commit to `main` directly, which a required check would otherwise always block, and
  merge a red PR in an emergency. A bypass covers every rule in its ruleset, so both are possible and both
  are recorded in the repository's audit log.
- Renovate is not a bypass actor: its PRs still wait for a green `CI result`.
- `strict_required_status_checks_policy` is off: a bypass covers every rule in a ruleset, so requiring an
  up-to-date branch here would stop maintainers from merging an out-of-date PR too. Keep "Require branches
  to be up to date before merging" on in the branch protection of `main` instead, where admins can bypass it.
