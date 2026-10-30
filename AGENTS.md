# AGENTS.md

This repository collects knowledge and experiments about GitHub Actions. It is not a real project.

## Layout

- `main`: `README.md` (index of every test), this file, `CLAUDE.md`. No workflows.
- One branch per test: `test/YYYY-MM-DD-<short-slug>`, created from `main`, dated the day the test is created.

## Adding a test

1. `git switch main && git pull && git switch -c test/YYYY-MM-DD-<short-slug>`
2. Add the workflow(s) under `.github/workflows/`.
   - Trigger on `push` to the test branch itself, plus `workflow_dispatch`. `workflow_dispatch` alone
     only shows up once the file exists on the default branch, so pushing is what triggers the run.
     Add `paths-ignore: ['**.md']` so pushing the README result does not re-run the test.
   - Set top-level `permissions: {}` and give each job only the permissions it needs.
   - Pin third-party actions to a full commit SHA. Prefer the preinstalled `gh` CLI where possible.
   - When a step is expected to fail, use `continue-on-error: true` and write the outcome to
     `$GITHUB_STEP_SUMMARY`, so the run shows the result without digging through logs.
   - Include a control job (same steps, enough permissions) when testing a restriction, to prove the
     failure comes from the thing under test.
3. Replace the branch `README.md` with:
   - **Question**: the one thing the test answers.
   - **Setup**: what the workflow does, and the trigger.
   - **How to run**: push to the branch, or re-run from the Actions tab.
   - **Expected**: the hypothesis, written before the run.
   - **Result**: what actually happened, with the run URL and the exact error messages.
   - **Conclusion**: the takeaway in one or two sentences.
4. Push, wait for the run (`gh run watch`), then fill in Result and Conclusion from the real logs.
   Never write a result before the run has finished.
5. On `main`, add a row at the top of the **Tests** table in `README.md`:
   `| YYYY-MM-DD | <name> | [<branch>](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/tree/<branch>) | <question> | <result> |`
6. Clean up side effects (close issues or PRs the test opened) with a comment linking the run.

## Rules

- Never merge a test branch into `main`.
- Never use real secrets. Use dummy values if a test needs a secret.
