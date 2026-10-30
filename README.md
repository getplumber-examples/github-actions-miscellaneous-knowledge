# github-actions-miscellaneous-knowledge

Notes and experiments about GitHub Actions.

Each experiment gets its own branch, named `test/YYYY-MM-DD-<short-slug>`. The branch holds the workflows
and a `README.md` with the question, the setup, and the result, with links to the runs. Test branches are
never merged. `main` only keeps this index and general notes.

[AGENTS.md](AGENTS.md) describes how to add a test.

## Tests

Newest first.

| Date | Test | Branch | Question | Result |
| ---- | ---- | ------ | -------- | ------ |
| 2026-10-30 | Create issue with `contents: read` | [test/2026-10-30-create-issue-with-contents-read](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/tree/test/2026-10-30-create-issue-with-contents-read) | Can a `contents: read` token open an issue and comment on it? | ❌ No, it needs `issues: write`. With no `permissions` key at all, the repo default (`write`) lets it through. |
