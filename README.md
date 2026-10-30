# github-actions-miscellaneous-knowledge

Knowledge base and experiments around GitHub Actions.

Each experiment lives on its own branch, named `test/YYYY-MM-DD-<short-slug>`. That branch holds the
workflow(s) and a `README.md` describing the question, setup, and result, with a link to the run.
`main` only holds this index and general notes. Test branches are never merged.

Conventions for humans and agents: [AGENTS.md](AGENTS.md).

## Tests

Newest first.

| Date | Test | Branch | Question | Result |
| ---- | ---- | ------ | -------- | ------ |
| 2026-10-30 | Create issue with `contents: read` | [test/2026-10-30-create-issue-with-contents-read](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/tree/test/2026-10-30-create-issue-with-contents-read) | Can a `contents: read` token open an issue and comment on it? | ❌ No: `Resource not accessible by integration`. Needs `issues: write`. ([run](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41036507896)). With **no** `permissions` key, the repo default (`write`) applies and it works ([run](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41037979599)). Same failure with `contents: read` set at the top level ([run](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/41038921943)). |
