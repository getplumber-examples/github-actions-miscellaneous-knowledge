# `always()` after a skipped gate job

Test from 2026-11-27, part of [github-actions-miscellaneous-knowledge](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge).

## Question

A `gate` job only runs for the repository's own branches (`if:` compares the head repository with
`github.repository`), so a pull request from a fork skips it. Which of the jobs that need `gate` still run for the
fork, and what do they read in `needs.gate.result` and `needs.gate.outputs.*`?

Background: [getplumber/plumber#641](https://github.com/getplumber/plumber/issues/641). Plumber lost the gate behind
`if: always() && needs.gate.result == 'success'` and reported the jobs after it as open to anyone. The same model reads
`always()` as unknown.

## Setup

GitHub reads a `pull_request_target` workflow from the default branch only
([changelog, in effect since 2025-12-08](https://github.blog/changelog/2025-11-07-actions-pull_request_target-and-environment-branch-protections-changes/)).
So the workflow of this test is on `main`:
[`always-after-skipped-gate.yml`](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/blob/main/.github/workflows/always-after-skipped-gate.yml),
on `pull_request_target` into this branch. This branch is the base of the test pull requests and holds `case.txt`,
which contains `base`.

The workflow has `permissions: {}`. No job checks out anything or reads a secret. Each job only prints its name and,
where it can, what it reads from `gate`.

| Job | `needs` | `if:` |
| --- | ------- | ----- |
| `gate`, writes the output `go=true` | none | `github.event.pull_request.head.repo.full_name == github.repository` |
| `plain` (control) | `gate` | none |
| `always` | `gate` | `always()` |
| `always-own-name` | `gate` | `always() && github.repository == 'getplumber-examples/github-actions-miscellaneous-knowledge'` |
| `always-result-success` | `gate` | `always() && needs.gate.result == 'success'` |
| `always-output` | `gate` | `always() && needs.gate.outputs.go == 'true'` |
| `always-result-skipped` | `gate` | `always() && needs.gate.result == 'skipped'` |
| `behind-closed` | `always-result-success` | none |
| `behind-always` | `always` | none |

`always`, `always-own-name` and `always-result-skipped` print `needs.gate.result` and `needs.gate.outputs.go`.

Two pull requests into this branch, each changing only `case.txt`:

| Case | Head | `case.txt` |
| ---- | ---- | ---------- |
| Fork | `Exypte/github-actions-miscellaneous-knowledge`, branch `always-after-skipped-gate-fork` | `fork` |
| Same repository (control) | this repository, branch `always-after-skipped-gate-same-repo` | `same-repo` |

## How to run

Push a new commit that changes `case.txt` to one of the two head branches, with its pull request into this branch
open. Each push starts the workflow.

## Expected

GitHub's documentation says a job is skipped when a job it needs is skipped, "unless the jobs use a conditional
expression that causes the job to continue", and that `always()` "returns `true`, even when canceled". So `always()`
should let a job outlive the skipped `gate`, and the rest of its condition should then decide, with
`needs.gate.result` equal to `skipped` and the outputs of `gate` empty.

| Job | Fork PR | Same-repository PR |
| --- | ------- | ------------------ |
| `gate` | skipped | runs |
| `plain` | skipped | runs |
| `always` | runs, result `skipped`, output empty | runs, result `success`, output `true` |
| `always-own-name` | runs, result `skipped`, output empty | runs, result `success`, output `true` |
| `always-result-success` | skipped | runs |
| `always-output` | skipped | runs |
| `always-result-skipped` | runs, result `skipped`, output empty | skipped |
| `behind-closed` | skipped | runs |
| `behind-always` | no prediction: it runs because `always` succeeded, or it is skipped because `gate` was skipped further up | runs |

## Result

Every row came out as expected. `behind-always`, the row with no prediction, was skipped for the fork.

| Job | Fork PR [#26](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/pull/26), head `Exypte/github-actions-miscellaneous-knowledge` at `8d3f803` (run [45994252328](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/45994252328)) | Same-repository PR [#27](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/pull/27), head `always-after-skipped-gate-same-repo` at `2387d8f` (run [45994256043](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/45994256043)) |
| --- | --- | --- |
| `gate` | skipped | ran |
| `plain` | skipped | ran |
| `always` | ran, `needs.gate.result` = `skipped`, `needs.gate.outputs.go` empty | ran, `needs.gate.result` = `success`, `needs.gate.outputs.go` = `true` |
| `always-own-name` | ran, `needs.gate.result` = `skipped`, `needs.gate.outputs.go` empty | ran, `needs.gate.result` = `success`, `needs.gate.outputs.go` = `true` |
| `always-result-success` | skipped | ran |
| `always-output` | skipped | ran |
| `always-result-skipped` | ran, `needs.gate.result` = `skipped`, `needs.gate.outputs.go` empty | skipped |
| `behind-closed` | skipped | ran |
| `behind-always` | skipped | ran |

Both runs ended in `success`. A skipped job is not a failure.

`behind-always` needs only `always`, which succeeded in the fork run, and it has no `if:`. It was skipped all the
same: `gate`, two steps up its `needs` chain, was skipped, and without a status function the default `success()`
looks at the whole chain.

### Limit

The fork belongs to `Exypte`, who has admin access to this repository. The repository asks for approval before running
workflows from outside contributors (`all_external_contributors`), and that approval was not exercised here.

## Conclusion

`always()` is true and lets a job run after a skipped `gate`, but only that job: the rest of its `if:` still decides.
For the fork, `needs.gate.result` was `skipped` and the output of `gate` was empty, so
`always() && needs.gate.result == 'success'` and `always() && needs.gate.outputs.go == 'true'` kept the fork out, and
so did a job with no condition behind them. `if: always()` alone, or next to a condition on the repository's own name,
let the fork in.

A job with no `if:` is skipped when any job up its `needs` chain was skipped, even when the job it needs directly
ran and succeeded.
