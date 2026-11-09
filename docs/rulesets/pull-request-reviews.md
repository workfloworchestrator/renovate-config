# About `pull-request-reviews.json`

This ruleset holds the review rule for `main`. Renovate has no bypass: it relies on the
`renovate-approve` and `renovate-approve-2` apps, which approve a PR that is configured for automerge.
With `required_approving_review_count` set to 2, both apps are needed. GitHub then merges the PR once the
required checks pass, without a human approval.

The ruleset holds only the review rule, because a bypass covers every rule in its ruleset. Keep required
status checks in a separate branch protection rule or ruleset, so that these stay mandatory for everyone.

The free tier of Github has no organization wide rulesets, so these must be created in each repository.

## Installation

1. Make sure the [renovate-approve](https://github.com/apps/renovate-approve) and
   [renovate-approve-2](https://github.com/apps/renovate-approve-2) apps are installed on the repository's
   organization. They only approve Renovate PRs that have automerge enabled.

   Both are installed organization-wide on `workfloworchestrator`, so its repositories can skip this step.
   Another organization needs its own installation (org-wide, or on the repository): an app installed in
   `workfloworchestrator` cannot approve PRs in `BandwidthOnDemand`, for example.
2. Set the review parameters in `pull-request-reviews.json` to what the repository's branch protection of
   `main` requires now, e.g. `required_approving_review_count` and `require_code_owner_review`. The
   approval count must match the number of approve apps you installed, so 2 for both.
3. From the root of this repository use the `gh` tool to create the main branch ruleset in `REPOSITORY`:

   ```bash
   REPOSITORY=nwa-stdlib
   gh api -X POST "repos/workfloworchestrator/${REPOSITORY}/rulesets" --input docs/rulesets/pull-request-reviews.json
   ```

   To update an existing ruleset, look up its id by name and replace it:

   ```bash
   REPOSITORY=nwa-stdlib
   RULESET_ID=$(gh api "repos/workfloworchestrator/${REPOSITORY}/rulesets" --jq '.[] | select(.name == "Pull request reviews") | .id')
   gh api -X PUT "repos/workfloworchestrator/${REPOSITORY}/rulesets/${RULESET_ID}" --input docs/rulesets/pull-request-reviews.json
   ```

4. Turn off "Require a pull request before merging" in the branch protection of `main`. GitHub combines
   branch protection and rulesets and applies the strictest, so a review requirement left there applies on
   top of the ruleset.

## About the ruleset

- actor_id `4138115` is the `core-maintainers` team, the same bypass the branch protection gives it today.
  It keeps the ability to merge without the two approvals.
- The renovate-approve apps approve only PRs with automerge enabled. A Renovate PR that needs review, such as a
  runtime dependency, still waits for a human.
