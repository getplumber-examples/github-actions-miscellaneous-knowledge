# Required approval bypass from a workflow token

Test from 2026-11-13, part of [github-actions-miscellaneous-knowledge](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge).

## Question

The [source-pinning test](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/tree/test/2026-11-13-required-check-source-pinning)
showed a workflow can clear a required status check with its own `GITHUB_TOKEN`. If the branch *also* requires an
approval from someone else, does one compromised account (acting through a pushed workflow) still get blocked? Or can
the workflow approve its own PR and merge it?

Short answer: with this repository's settings, the workflow clears everything and merges. The automatic `GITHUB_TOKEN`
posted the check, approved the PR as `github-actions`, and merged it. `require_last_push_approval` did not stop the
bot approval.

Background: the wfx-loot campaign used hijacked write access to push workflows. This test measures how much a single
such account can do on its own.

## Setup

- Base branch `test/2026-11-13-required-approval-bypass-base`, a throwaway stand-in for a protected default branch.
  Nothing targets `main`.
- Ruleset [test 2026-11-13 required approval bypass](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/rules/27212710)
  on the base branch, two rules:
  - `required_status_checks`: requires a check named `fake-ci`, any source.
  - `pull_request`: `required_approving_review_count: 1`, `require_last_push_approval: true`,
    `require_code_owner_review: false`. No bypass actors.
- Repository settings that matter: **Allow GitHub Actions to create and approve pull requests** is on
  (`can_approve_pull_request_reviews: true`), and the default workflow token is `write`.
- PR [#16](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/pull/16), test branch -> base.
- Workflow [`approve-and-merge.yml`](.github/workflows/approve-and-merge.yml), job `drive` (runs when the head commit
  message contains `[drive]`), top-level `permissions: {}`, job permissions `contents: write`, `pull-requests: write`,
  `checks: write`. Using only `gh` and `GITHUB_TOKEN`, it: reads the PR state, posts `fake-ci` success, submits an
  APPROVE review, re-reads the state, then calls `PUT /pulls/{n}/merge`. Each step's result goes to the step summary.

## How to run

Reopen PR #16 (or recreate the base branch, the test branch and the ruleset), then push a commit that changes
`case.txt` with `[drive]` in the message.

## Expected

Baseline: BLOCKED, REVIEW_REQUIRED. The unknown is the approval: does a review submitted by `GITHUB_TOKEN` count as an
approval, and does `require_last_push_approval` reject it because the bot wasn't the last pusher? If the approval counts
and last-push-approval doesn't catch it, the workflow merges end to end.

## Result

Run: [43456344950](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/43456344950)
(head commit `c29b358`). From the step summary and the PR/review APIs:

| Step | Outcome |
| ---- | ------- |
| Baseline (before `[drive]`) | `mergeStateStatus=BLOCKED`, `reviewDecision=REVIEW_REQUIRED` |
| Post `fake-ci` check run | `fake-ci=success`, app `github-actions` (id 15368) |
| Approve PR with `GITHUB_TOKEN` | review `state=APPROVED` by `github-actions` (Bot) |
| After approval | `reviewDecision=APPROVED`, reviews `[github-actions:APPROVED]` |
| Merge (`PUT /pulls/16/merge`) | merged |
| Final | `state=MERGED`, `mergedBy=app/github-actions`, merge commit `185edaa`; the base branch head moved to it |

One workflow, one token, no second person: the check and the approval were both satisfied by `github-actions`, and the
PR merged into the protected base branch. `require_last_push_approval: true` did not block a review from the Actions
bot.

## Conclusion

A required approval is **not** a real second-person gate while **Allow GitHub Actions to create and approve pull
requests** is on and the workflow token is `write`: a pushed workflow approves and merges its own PR. For #746 this
means "require an approval" can't be counted as protection on its own.

What actually stops a single hijacked write token:

- Turn **off** "Allow GitHub Actions to create and approve pull requests" at the repository and organization level, so a
  bot review can't satisfy the requirement.
- Set the default workflow token to **read-only**, so a pushed workflow can't merge or approve without explicitly asking
  for the scope (which a reviewer would see in the diff).
- Require **code-owner review on `.github/workflows/**`** with a real `CODEOWNERS` (this repository's `main` has none),
  so any change to a workflow needs a named human.
- Pin the required check to a source workflows can't post as (external CI or a dedicated GitHub App whose key is not a
  repository secret).
