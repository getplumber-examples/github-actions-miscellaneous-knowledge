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
| 2026-11-13 | Required approval bypass | [test/2026-11-13-required-approval-bypass](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/tree/test/2026-11-13-required-approval-bypass) | If a branch requires a check *and* an approval, can one workflow clear both and merge? | ❌ Not protected here. With "Actions can approve PRs" on and a write token, the workflow posted the check, approved as `github-actions`, and merged. `require_last_push_approval` didn't stop the bot approval. |
| 2026-11-13 | Secret reachability | [test/2026-11-13-secret-reachability](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/tree/test/2026-11-13-secret-reachability) | Which secrets can a workflow pushed to any branch, or opened from a fork, read? Does an environment restricted to `main` protect a secret? | ⚠️ A branch workflow reads every repository secret; a fork PR reads none. An environment secret limited to `main` stays out of reach: empty without `environment:`, and the job is rejected when it asks for the environment. |
| 2026-11-13 | Required check source pinning | [test/2026-11-13-required-check-source-pinning](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/tree/test/2026-11-13-required-check-source-pinning) | Does tying a required check to GitHub Actions stop a fake `success`? | ⚠️ It stops a personal token, not a workflow. A commit status or check run posted with the workflow's own `GITHUB_TOKEN` clears the check. |
| 2026-11-06 | Dependabot and unmerged branches | [test/2026-11-06-dependabot-unmerged-branches](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/tree/test/2026-11-06-dependabot-unmerged-branches) | Does Dependabot propose an unmerged `v1` branch, does that code run, and do a fork's branch or a lockfile change that? | ⚠️ Yes for a tag pin, and the PR's CI ran the unmerged code. No for a SHA pin on a tagged commit. A fork's `v2` branch had no effect. A lockfile doesn't block it, but it holds the PR to the locked commit if the branch moves. Deleting the branch fails the PR at Set up job either way. |
| 2026-10-30 | Create issue with `contents: read` | [test/2026-10-30-create-issue-with-contents-read](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/tree/test/2026-10-30-create-issue-with-contents-read) | Can a `contents: read` token open an issue and comment on it? | ❌ No, it needs `issues: write`. With no `permissions` key at all, the repo default (`write`) lets it through. |
