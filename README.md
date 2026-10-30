# Create an issue with `permissions: contents: read`

Test from 2026-10-30, part of [github-actions-miscellaneous-knowledge](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge).

## Question

Can a workflow open an issue and comment on it when its `GITHUB_TOKEN` only has `contents: read`?

Short answer: no. The token also needs `issues: write`.

## Setup

Three workflows run on every push to this branch (Markdown changes are ignored) and on `workflow_dispatch`.
Each job runs the same script with `GH_TOKEN: ${{ github.token }}`: `gh issue create`, then `gh issue comment`
on the new issue. The script writes each exit code and output to the step summary. If a command fails, it
logs an `::error::` annotation and exits with gh's exit code, so the job goes red.

| Workflow | Job | Permissions |
| -------- | --- | ----------- |
| [`create-issue-contents-read.yml`](.github/workflows/create-issue-contents-read.yml) | `contents-read` | `contents: read` on the job, `{}` at the top level |
| [`create-issue-contents-read.yml`](.github/workflows/create-issue-contents-read.yml) | `issues-write` | `contents: read` and `issues: write` on the job. Control job that shows the script works |
| [`create-issue-top-level-contents-read.yml`](.github/workflows/create-issue-top-level-contents-read.yml) | `top-level-contents-read` | `contents: read` at the top level, inherited by the job |
| [`create-issue-no-permissions.yml`](.github/workflows/create-issue-no-permissions.yml) | `no-permissions` | Not set anywhere, so the repository default applies |

`no-permissions` has its own file because inside the first workflow it would inherit `permissions: {}`.

## How to run

Push a non-Markdown change to this branch, or re-run the workflows from the Actions tab. The passing jobs open
real issues, so close them afterwards.

## Expected

Once `permissions` is set, every scope it doesn't list is `none`. So:

- `contents-read` and `top-level-contents-read` fail with `Resource not accessible by integration`.
- `issues-write` opens the issue and comments on it.
- `no-permissions` gets the repository default. For this repository,
  `GET /repos/{owner}/{repo}/actions/permissions/workflow` returns `default_workflow_permissions: write`,
  so the job should get `Issues: write` and succeed.

## Result

Runs: [top-level contents read](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41038921943),
[contents read](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41038921691),
[no permissions](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41038921417).
Token permissions come from the "Set up job" step.

| Job | Token permissions | `gh issue create` | `gh issue comment` | Job |
| --- | ----------------- | ----------------- | ------------------ | --- |
| `contents-read` | `Contents: read`, `Metadata: read` | ❌ `GraphQL: Resource not accessible by integration (createIssue)` | not reached | ❌ exit 1 |
| `top-level-contents-read` | `Contents: read`, `Metadata: read` | ❌ same error | not reached | ❌ exit 1 |
| `issues-write` | `Contents: read`, `Issues: write`, `Metadata: read` | ✅ [#6](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/issues/6) | ✅ | ✅ |
| `no-permissions` | `write` on almost every scope | ✅ [#5](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/issues/5) | ✅ | ✅ |

The `no-permissions` token had `write` on Actions, Contents, Issues, Packages, Pages, PullRequests,
SecurityEvents, Statuses and the remaining scopes, and `read` on Metadata, Models and VulnerabilityAlerts.

Earlier runs gave the same outcomes, but back then a failed command didn't fail the job, so every job was green:
[41036507896](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41036507896)
(issue [#1](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/issues/1),
[comment](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/issues/1#issuecomment-6519879217)),
[41037979580](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41037979580),
[41037979599](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41037979599)
(issue [#4](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/issues/4),
[comment](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/issues/4#issuecomment-6520088693)).

## Conclusion

A token with only `contents: read` can't open an issue, so it can't comment on one either. Opening and
commenting both need `issues: write`.

Other things the runs showed:

- Declaring `contents: read` at the top level or on the job gives the same token.
- A job-level `permissions` block replaces the top-level one instead of merging with it. That's how
  `issues-write` gets `Issues: write` under a top-level `permissions: {}`.
- With no `permissions` key at all, the job gets the repository default. Here that's `write` on almost
  everything, so it could also push code or publish packages.
- `Metadata: read` is always granted, even when not listed.
- Every job logs `Cache mode: write`, including the `contents: read` ones. The `permissions` block doesn't
  seem to control the Actions cache.
- `gh issue create` uses the GraphQL API, so the failure shows up as a GraphQL error. The REST API returns
  a 403 with the same message.

Set `permissions: {}` at the top level and give `issues: write` only to the jobs that need it.
