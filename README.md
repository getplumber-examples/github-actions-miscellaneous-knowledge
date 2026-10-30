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

## How to run

Push to this branch, or re-run the workflow from the Actions tab.

## Expected

When a `permissions` block is set, every scope it doesn't list is `none`. So `issues` is `none` in
`contents-read`, and `gh issue create` should fail with HTTP 403
`Resource not accessible by integration`. `issues-write` should open the issue and comment on it.

## Result

_Pending: filled in from the first run._

## Conclusion

_Pending._
