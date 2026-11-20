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

Pending.

## Conclusion

Pending.
