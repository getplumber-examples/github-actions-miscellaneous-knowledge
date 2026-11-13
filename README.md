# Required check source pinning

Test from 2026-11-13, part of [github-actions-miscellaneous-knowledge](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge).

## Question

A required status check can be tied to a source app ("expected source", `integration_id` in a ruleset). If it is tied to
GitHub Actions, can a workflow still clear it by posting a fake `success` with its own `GITHUB_TOKEN`?

Short answer: yes. Tying the check to GitHub Actions rejects a status posted with a personal token, but a workflow posts
as GitHub Actions, so both a commit status and a check run from its `GITHUB_TOKEN` clear the check.

Background: in the wfx-loot campaign (2026-11-10 to 2026-11-12), hijacked accounts posted fake `success` statuses
under the names of required checks so branch protection let changes through
([Aikido](https://www.aikido.dev/blog/compromised-github-accounts-push-ci-secret-stealer-to-tens-of-thousands-of-repositories),
[Endor Labs](https://www.endorlabs.com/learn/github-actions-supply-chain-attack-targeting-ci-secrets-at-scale)).

## Setup

- Branch `test/2026-11-13-required-check-source-pinning-base`, created from `main` and never written to.
- Ruleset [test 2026-11-13 required-check source pinning](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/rules/27206333)
  on that branch only: one rule, `required_status_checks`, requiring a check named `fake-ci`. No bypass actors.
  It is left in place, tied to GitHub Actions (`integration_id: 15368`).
- PR [#12](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/pull/12) from this branch into
  the base branch. Nothing ever runs a real `fake-ci`, so the PR is mergeable only if someone fakes it.
- Workflow [`forge-required-check.yml`](.github/workflows/forge-required-check.yml), on push to this branch (Markdown
  ignored) and on `workflow_dispatch`, `permissions: {}` at the top level:

| Job | Runs when the head commit message contains | Permissions | Posts `fake-ci` as |
| --- | ------------------------------------------ | ----------- | ------------------ |
| `forge-status` | `[post-status]` | `statuses: write`, `pull-requests: read` | commit status (`POST /repos/{owner}/{repo}/statuses/{sha}`) |
| `forge-check` | `[post-check]` | `checks: write`, `pull-requests: read` | check run (`POST /repos/{owner}/{repo}/check-runs`) |

After posting, the job polls PR #12 until `mergeable_state` settles for its commit, and writes the post result and the
state to the step summary. A failed API call, no open PR, or a state that never settles fails the job with `::error::`.

The GitHub Actions app id (15368) comes from the `app.id` of a check run created by a workflow job in this repository.

## How to run

Each case needs its own head commit on this branch. Reopen PR #12 first.

- Baseline, A and B are manual, with a personal token:
  - A: set the ruleset's check to any source (remove `integration_id`), then
    `gh api repos/getplumber-examples/github-actions-miscellaneous-knowledge/statuses/<head sha> -f context=fake-ci -f state=success`.
  - B: put `"integration_id": 15368` back on the check. The status from A stays on the commit.
- C: push a commit that changes `case.txt` with `[post-status]` in the message.
- D: push a commit that changes `case.txt` with `[post-check]` in the message.

Read the result with `gh pr view 12 --json mergeStateStatus` or in the job summary.

## Expected

- Baseline: BLOCKED, `fake-ci` is missing.
- A: CLEAN. With any source, a status from anyone with write access counts.
- B: BLOCKED. A status posted with a personal token comes from no app.
- C and D: unknown before the run. `GITHUB_TOKEN` is an installation token of the GitHub Actions app, so what it posts
  may count as coming from GitHub Actions.

## Result

| Case | Check tied to | Head commit | Who posted `fake-ci: success` | Posted as | PR #12 |
| ---- | ------------- | ----------- | ----------------------------- | --------- | ------ |
| Baseline | any source | `ab0652e` | nobody | — | BLOCKED |
| A | any source | `ab0652e` | personal token, commit status | `Exypte (User)` | CLEAN |
| B | GitHub Actions (15368) | `ab0652e` | same status as A | `Exypte (User)` | BLOCKED |
| C | GitHub Actions (15368) | `dcd217c` | workflow `GITHUB_TOKEN`, commit status ([run](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/43442864003)) | `github-actions[bot] (Bot)` | `unstable` during the run, CLEAN after |
| D | GitHub Actions (15368) | `638b99b` | workflow `GITHUB_TOKEN`, check run ([run](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/43442962455)) | app `github-actions` (id 15368) | `unstable` during the run, CLEAN after |

Baseline, A and B were read with `gh pr view 12 --json mergeStateStatus` and `GET /repos/{owner}/{repo}/pulls/12`
(`mergeable_state`: `blocked`, `clean`, `blocked`). In C and D the job's own summary shows the post result and
`mergeable_state: unstable`. That means mergeable while a check that isn't required (the job itself) is still running.
Once the run ended, the PR read `clean`.

The token permissions logged in "Set up job" were `PullRequests: read` and `Statuses: write` for C, and
`Checks: write` and `PullRequests: read` for D. The commit for D has no commit status at all, only check runs, so the
check run alone satisfied the rule.

An earlier commit, `a7947c8` ("case C [post-status]"), was empty and started no run. With `paths-ignore` set, a push
that changes no file doesn't match the filter. That's why each case commit changes `case.txt`.

## Conclusion

Tying a required check to GitHub Actions stops a stolen personal token from faking it (B), but not a workflow. Anyone
who can push a branch can add a workflow that asks for `statuses: write` or `checks: write` and clears the check with
its own `GITHUB_TOKEN` (C, D). Setting the repository's default token to read-only doesn't help, because the workflow
asks for these permissions itself.

To stop that too, tie the check to an app that workflows can't post as (an external CI, a dedicated GitHub App), or
require something a token can't fake: an approval from someone else and code-owner review on `.github/workflows/**`.
