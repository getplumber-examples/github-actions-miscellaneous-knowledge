# Case sensitivity of `uses:`

Test from 2026-11-20, part of [github-actions-miscellaneous-knowledge](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge).

## Question

Does GitHub resolve `uses:` case-sensitively? If an action or a reusable workflow is referenced with a
different case in the owner, the repository, the path or the ref, does the job still get it?

## Setup

All workflows run on every push to this branch (Markdown changes are ignored) and on `workflow_dispatch`, with
top-level `permissions: {}`.

[`action-case.yml`](.github/workflows/action-case.yml) has one job per variant. Each job runs `actions/checkout`
spelled differently. The action is downloaded in "Set up job", so a reference GitHub can't resolve fails that
job there, and the other jobs still run.

| Job | `uses:` |
| --- | ------- |
| `control` | `actions/checkout@11d5960a326750d5838078e36cf38b85af677262` |
| `owner-upper` | `Actions/checkout@11d5960a326750d5838078e36cf38b85af677262` |
| `repo-upper` | `actions/Checkout@11d5960a326750d5838078e36cf38b85af677262` |
| `all-upper` | `ACTIONS/CHECKOUT@11d5960a326750d5838078e36cf38b85af677262` |
| `tag-ref-upper` | `actions/checkout@V4` (the tag is `v4`) |

Reusable workflows are resolved when the run starts, and a reference that doesn't resolve invalidates the whole
workflow file. So each variant has its own caller. All of them call
[`reusable.yml`](.github/workflows/reusable.yml), which only echoes a line, at this branch.

| Workflow | `uses:` |
| -------- | ------- |
| [`reusable-control.yml`](.github/workflows/reusable-control.yml) | `getplumber-examples/github-actions-miscellaneous-knowledge/.github/workflows/reusable.yml@<branch>` |
| [`reusable-owner-repo-upper.yml`](.github/workflows/reusable-owner-repo-upper.yml) | `GetPlumber-Examples/GitHub-Actions-Miscellaneous-Knowledge/.github/workflows/reusable.yml@<branch>` |
| [`reusable-path-upper.yml`](.github/workflows/reusable-path-upper.yml) | `getplumber-examples/github-actions-miscellaneous-knowledge/.github/workflows/Reusable.yml@<branch>` |

`control` and `reusable-control` show that the reference works when spelled correctly, so any failure comes from
the case.

## How to run

Push a non-Markdown change to this branch, or re-run the workflows from the Actions tab.

## Expected

Owner and repository names on GitHub are case-insensitive (the API and `git clone` accept any case), while a
ref and a file path are case-sensitive git names. So:

- `owner-upper`, `repo-upper`, `all-upper` and `reusable-owner-repo-upper` pass like their controls.
- `tag-ref-upper` fails in "Set up job" because `V4` isn't a ref.
- `reusable-path-upper` is rejected because `Reusable.yml` doesn't exist.

## Result

Runs: [action case](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/44713707806), [reusable control](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/44713708194),
[reusable owner repo upper](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/44713707958), [reusable path upper](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/44713707750).

Actions. The "Set up job" log line is quoted as GitHub printed it.

| Job | `uses:` | Result |
| --- | ------- | ------ |
| `control` | `actions/checkout@<sha>` | ✅ `Download action repository 'actions/checkout@11d5960…' (SHA:11d5960…)` |
| `owner-upper` | `Actions/checkout@<sha>` | ✅ `Download action repository 'Actions/checkout@11d5960…' (SHA:11d5960…)` |
| `repo-upper` | `actions/Checkout@<sha>` | ✅ `Download action repository 'actions/Checkout@11d5960…' (SHA:11d5960…)` |
| `all-upper` | `ACTIONS/CHECKOUT@<sha>` | ✅ `Download action repository 'ACTIONS/CHECKOUT@11d5960…' (SHA:11d5960…)` |
| `tag-ref-upper` | `actions/checkout@V4` | ❌ `` Unable to resolve action `actions/checkout@V4`, unable to find version `V4` `` |

All four spellings downloaded and ran the same commit. The log and the Node.js 20 deprecation annotation keep
the spelling from the workflow, so the runner doesn't rewrite the name.

Reusable workflows:

| Workflow | Result |
| -------- | ------ |
| `reusable-control` | ✅ |
| `reusable-owner-repo-upper` | ✅ The called job logs `Uses: getplumber-examples/github-actions-miscellaneous-knowledge/.github/workflows/reusable.yml@refs/heads/test/2026-11-20-uses-case-sensitivity (b461474…)`: GitHub rewrote the reference to the canonical lower-case name. |
| `reusable-path-upper` | ❌ The run never starts. `Invalid workflow file`: `error parsing called workflow -> "getplumber-examples/github-actions-miscellaneous-knowledge/.github/workflows/Reusable.yml@test/2026-11-20-uses-case-sensitivity": failed to fetch workflow: workflow was not found.` |

The hypothesis held for every variant.

## Conclusion

The owner and repository in `uses:` are case-insensitive, for actions and reusable workflows alike:
`ACTIONS/CHECKOUT` is `actions/checkout`. The ref (`V4` ≠ `v4`) and the workflow path (`Reusable.yml` ≠
`reusable.yml`) are case-sensitive. A tool that matches `uses:` against an allow or deny list must fold the case
of the owner/repo part only.
