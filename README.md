# A condition on the repository's own name

Test from 2026-11-27, part of [github-actions-miscellaneous-knowledge](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge).

## Question

A job runs on `pull_request_target` (or `workflow_run`), checks out the pull request's head commit and reads a secret.
It carries `if: github.repository == 'owner/repo'`, with the repository's own name. Does that condition keep a pull
request from a fork out of the job?

Background: [getplumber/plumber#640](https://github.com/getplumber/plumber/issues/640). Plumber read this condition
as a maintainer gate and lowered the finding from critical to medium.

## Setup

GitHub reads `pull_request_target` and `workflow_run` workflows from the default branch only
([changelog, in effect since 2025-12-08](https://github.blog/changelog/2025-11-07-actions-pull_request_target-and-environment-branch-protections-changes/)).
So the three workflows of this test are on `main`, not on this branch. This branch is the base of the test pull
requests and holds `case.txt`, which contains `base`.

| Workflow (on `main`) | Trigger |
| -------------------- | ------- |
| [`repository-name-condition.yml`](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/blob/main/.github/workflows/repository-name-condition.yml) | `pull_request_target` into this branch |
| [`repository-name-condition-trigger.yml`](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/blob/main/.github/workflows/repository-name-condition-trigger.yml) | `pull_request` into this branch. It echoes one line and exists to fire the next one |
| [`repository-name-condition-workflow-run.yml`](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/blob/main/.github/workflows/repository-name-condition-workflow-run.yml) | `workflow_run`, when the trigger workflow completes for a head branch named `repository-name-condition-*` |

The `pull_request_target` and `workflow_run` workflows have the same five jobs, with `permissions: {}` at the top and
`contents: read` on the jobs that check out:

| Job | `if:` |
| --- | ----- |
| `context` | none. Prints `github.repository`, the head repository, `GITHUB_REF`, `GITHUB_SHA` |
| `own-name` | `github.repository == 'getplumber-examples/github-actions-miscellaneous-knowledge'` |
| `no-condition` (control) | none |
| `head-repo` (control, a condition that does compare the fork) | `github.event.pull_request.head.repo.full_name == github.repository`, or `github.event.workflow_run.head_repository.full_name == github.repository` |
| `other-name` (control, the comparison can be false) | `github.repository == 'getplumber-examples/some-other-repository'` |

The last four jobs run the same steps: check out the head commit (`github.event.pull_request.head.sha` or
`github.event.workflow_run.head_sha`) into `pr/` with `persist-credentials: false`, print `pr/case.txt`, and test
whether the repository secret `MY_OWN_SECRET` is empty. Nothing from the pull request is executed and the secret is
never printed. A missing marker or an empty secret fails the job.

Two pull requests into this branch, each changing only `case.txt`:

| Case | Head | `case.txt` |
| ---- | ---- | ---------- |
| Fork | `Exypte/github-actions-miscellaneous-knowledge`, branch `repository-name-condition-fork` | `fork` |
| Same repository (control) | this repository, branch `repository-name-condition-same-repo` | `same-repo` |

## How to run

Push a new commit that changes `case.txt` to one of the two head branches, with its pull request into this branch
open. Each push starts the three workflows.

## Expected

`github.repository` is the repository the workflow runs in. For both events that is the base repository, whoever
opened the pull request. So the condition should be true in every case and stop nothing.

| Job | Fork PR | Same-repository PR |
| --- | ------- | ------------------ |
| `own-name` | runs, reads `fork`, secret reachable | runs, reads `same-repo`, secret reachable |
| `no-condition` | same as `own-name` | same as `own-name` |
| `head-repo` | skipped | runs |
| `other-name` | skipped | skipped |

The same table is expected for `pull_request_target` and for `workflow_run`.

## Result

Not run yet.

## Conclusion

Not run yet.
