# Test 2026-10-30: create an issue with `permissions: contents: read`

Part of [github-actions-miscellaneous-knowledge](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge).

## Question

Can a workflow whose `GITHUB_TOKEN` is limited to `permissions: contents: read` open an issue and
write a comment in it?

## Setup

[`.github/workflows/create-issue-contents-read.yml`](.github/workflows/create-issue-contents-read.yml),
triggered on `push` to this branch and on `workflow_dispatch`. Top-level `permissions: {}`. Two jobs
run the same script (YAML anchor), which calls `gh issue create` and then `gh issue comment` with
`GH_TOKEN: ${{ github.token }}`, and writes each exit code and output to the step summary:

| Job | Permissions | Role |
| --- | ----------- | ---- |
| `contents-read` | `contents: read` | Experiment |
| `issues-write` | `contents: read`, `issues: write` | Control: proves the script works |

A second workflow, [`.github/workflows/create-issue-no-permissions.yml`](.github/workflows/create-issue-no-permissions.yml),
has the same trigger and script but **no `permissions` key at all**, neither top-level nor on its job.
It has to live in its own file: inside the first workflow, a job without `permissions` would inherit
the top-level `permissions: {}`.

| Job | Permissions | Role |
| --- | ----------- | ---- |
| `no-permissions` | not set: repository/org default | What you get when you don't think about it |

## How to run

Push to this branch, or re-run the workflows from the Actions tab.

## Expected

When a `permissions` block is set, every scope it doesn't list is `none`. So `issues` is `none` in
`contents-read`, and `gh issue create` should fail with HTTP 403
`Resource not accessible by integration`. `issues-write` should open the issue and comment on it.

`no-permissions` gets the default from **Settings → Actions → Workflow permissions**. For this repository it is
`default_workflow_permissions: write` (from `GET /repos/{owner}/{repo}/actions/permissions/workflow`),
so it should get `Issues: write` and succeed.

## Result

Run: https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41036507896

| Job | Token permissions (from "Set up job") | `gh issue create` | `gh issue comment` |
| --- | ------------------------------------- | ----------------- | ------------------ |
| `contents-read` | `Contents: read`, `Metadata: read` | ❌ `GraphQL: Resource not accessible by integration (createIssue)` | skipped |
| `issues-write` | `Contents: read`, `Issues: write`, `Metadata: read` | ✅ [#1](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/issues/1) | ✅ [comment](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/issues/1#issuecomment-6519879217) |

`gh issue create` goes through the GraphQL API, so the error comes back as a GraphQL error and not
as an HTTP 403. The REST API would return 403 with the same message.

## Conclusion

No. With `permissions: contents: read`, the `GITHUB_TOKEN` cannot open an issue (and so cannot write in
one): `issues` is implicitly `none`. You need `issues: write` for both creating and commenting.
`Metadata: read` is always granted, even when not listed.
