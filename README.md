# Dependabot and unmerged branches

## Question

1. If an action repo has an unmerged `v1` branch and no `v1` tag, does Dependabot propose it to consumers,
   for a tag pin and for a SHA pin?
2. If it does, does the `V1` code run, and where?
3. Can a `v2` branch on a fork influence the update, on its own or through a PR into the action repo?

This comes from [getplumber/getplumber.io#536](https://github.com/getplumber/getplumber.io/pull/536), where
Dependabot bumped `getplumber/plumber@v0.7.18` to `@v1` and `v1` was a branch.
[The first run of that PR](https://github.com/getplumber/getplumber.io/actions/runs/41944645543) downloaded
the branch and ran it.

## Setup

This test uses separate repos, because Dependabot reads its config from the default branch and test
branches here are never merged. This branch only holds the write-up.

| Repo | Role |
| ---- | ---- |
| [print-version-demo](https://github.com/getplumber-examples/print-version-demo) | Composite action that prints its version. `main` and tag `v0.1.0` print `V0`. |
| [print-version-demo-consumer-tag](https://github.com/getplumber-examples/print-version-demo-consumer-tag) | Uses `print-version-demo@v0.1.0`. |
| [print-version-demo-consumer-sha](https://github.com/getplumber-examples/print-version-demo-consumer-sha) | Uses `print-version-demo@f95b8e2... # v0.1.0`. |
| `Exypte/print-version-demo` | Fork, with a `v2` branch printing `V2`. |

Both consumers run on `push` to `main`, `pull_request` and `workflow_dispatch`, with `permissions: {}`.
Dependabot checks `github-actions` daily, with the cooldown excluded for this action.

The stages run in this order, with a Dependabot check after each one:

1. Control: only `v0.1.0` exists.
2. Fork A: `v2` branch on the fork.
3. Fork B: PR from the fork's `v2` into the action repo, never merged.
4. PR from a `v1` branch on the action repo into `main`, never merged. No `v1` tag is created.

## How to run

Push the stage's change to the action repo or the fork, then run "Check for updates" under Insights >
Dependency graph > Dependabot on each consumer, or wait for the daily run.

## Expected

Written before the fork and the `v1` branch existed. It comes from reading dependabot-core:
`git_commit_checker.rb` only considers tags when the pinned SHA has a tag, and considers branches too
otherwise. `git_metadata_fetcher.rb` keeps only `refs/heads` and `refs/tags` of the action repo.

- Control: no PR on either consumer.
- Fork A and B: no PR on either consumer.
- `v1` branch, tag consumer: PR to `@v1`, and its run prints `V1`.
- `v1` branch, SHA consumer: no PR.
- `main` of both consumers keeps printing `V0`.

## Result

Pending.

## Conclusion

Pending.
