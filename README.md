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
| [`repository-name-condition-workflow-run.yml`](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/blob/main/.github/workflows/repository-name-condition-workflow-run.yml) | `workflow_run`, when the trigger workflow completes |

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

The condition stopped nothing. On the fork pull request, `own-name` ran for both events, checked out the fork's
commit and read the secret.

Fork pull request [#21](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/pull/21), head `Exypte/github-actions-miscellaneous-knowledge` at `5bfe545`:

| Job | `pull_request_target` ([45973998287](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/45973998287)) | `workflow_run` ([45974025370](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/45974025370)) |
| --- | --- | --- |
| `own-name` | ran, `case.txt` = `fork`, `MY_OWN_SECRET` reachable | ran, `case.txt` = `fork`, `MY_OWN_SECRET` reachable |
| `no-condition` | ran, `case.txt` = `fork`, `MY_OWN_SECRET` reachable | ran, `case.txt` = `fork`, `MY_OWN_SECRET` reachable |
| `head-repo` | skipped | skipped |
| `other-name` | skipped | skipped |

Same-repository pull request [#22](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/pull/22), head `repository-name-condition-same-repo` at `4adabc4`:

| Job | `pull_request_target` ([45973557021](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/45973557021)) | `workflow_run` ([45973579994](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/45973579994)) |
| --- | --- | --- |
| `own-name` | ran, `case.txt` = `same-repo`, `MY_OWN_SECRET` reachable | ran, `case.txt` = `same-repo`, `MY_OWN_SECRET` reachable |
| `no-condition` | ran, `case.txt` = `same-repo`, `MY_OWN_SECRET` reachable | ran, `case.txt` = `same-repo`, `MY_OWN_SECRET` reachable |
| `head-repo` | ran, `case.txt` = `same-repo`, `MY_OWN_SECRET` reachable | ran, `case.txt` = `same-repo`, `MY_OWN_SECRET` reachable |
| `other-name` | skipped | skipped |

What the `context` job printed, identical for `pull_request_target` and `workflow_run`:

| Value | Fork PR | Same-repository PR |
| ----- | ------- | ------------------ |
| `github.repository` | `getplumber-examples/github-actions-miscellaneous-knowledge` | `getplumber-examples/github-actions-miscellaneous-knowledge` |
| head repository | `Exypte/github-actions-miscellaneous-knowledge` | `getplumber-examples/github-actions-miscellaneous-knowledge` |
| head repository is a fork | `true` | `false` |
| `GITHUB_REF` | `refs/heads/main` | `refs/heads/main` |
| own-name condition | `true` | `true` |
| head-repo condition | `false` | `true` |

`GITHUB_SHA` was the head of `main` each time (`81798ce` for the fork runs, `52272ea` for the same-repository runs),
never the pull request's commit.

The first fork run ([45973551637](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/45973551637), head `353c922`) gave the same `pull_request_target`
column as above.

### A `branches:` filter on `workflow_run` and a fork

The `workflow_run` workflow first had `branches: ['repository-name-condition-*']`. With it, the workflow started for
the same-repository pull request but not for the fork one, although the fork's trigger run
([45973552005](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/45973552005)) completed with `head_branch` = `repository-name-condition-fork`.
After [#23](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/pull/23) removed the filter, a new commit on the fork branch started it
([45974025370](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/45974025370)). This is one observation and the test did not look further into it.

### Limit

The fork belongs to `Exypte`, who has admin access to this repository. The repository asks for approval before running
workflows from outside contributors (`all_external_contributors`), and that approval was not exercised here.

## Conclusion

`if: github.repository == 'owner/repo'` with the repository's own name is true for every run in that repository,
fork pull requests included, on `pull_request_target` and on `workflow_run`. It only stops the job in a copy of the
repository that has another name. It gave the fork the same access as no condition at all: the job checked out the
fork's commit with the repository secret available.

To keep fork pull requests out of such a job, compare the head repository with `github.repository`
(`github.event.pull_request.head.repo.full_name` or `github.event.workflow_run.head_repository.full_name`). That
condition skipped the job for the fork and ran it for the same-repository branch.
