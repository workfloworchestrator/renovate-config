# About `pull-request-reviews.json`

A ruleset can limit Renovate's bypass to merging PRs. The bypass option of branch protection also lets
Renovate push to `main` without a PR, which is not desirable. This ruleset holds only the
review rule, because a bypass covers every rule in its ruleset. Keep required status checks in a separate branch
protection rule or ruleset without a Renovate bypass, so that these are still mandatory.

The free tier of Github has no organization wide rulesets, so these must be created in each repository.

## Installation

1. Set the review parameters in `pull-request-reviews.json` to what the repository's branch protection of
   `main` requires now, e.g. `required_approving_review_count` and `require_code_owner_review`.
2. From the root of this repository use the `gh` tool to set the main branch ruleset on the given `REPOSITORY`:

   ```bash
   gh api -X POST repos/workfloworchestrator/REPOSITORY/rulesets --input docs/rulesets/pull-request-reviews.json
   ```

3. Turn off "Require a pull request before merging" in the branch protection of `main`. This also removes
   the bypass list of that setting, which the ruleset takes over. GitHub combines branch protection and
   rulesets and applies the strictest, so a review requirement left there would still block Renovate, and
   a Renovate bypass left there would still let it push to `main` without a PR.

## About the ruleset

- actor_id `2740` is the Renovate app; `pull_request` lets it bypass when merging a PR, not when pushing.
- actor_id `4138115` is the `core-maintainers` team, the same bypass the branch protection gives it today.
