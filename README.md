# Secret reachability from a branch, a fork PR and environments

Test from 2026-11-13, part of [github-actions-miscellaneous-knowledge](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge).

## Question

Which secrets can a workflow read when someone pushes it to an arbitrary branch, or opens it from a fork? And does
putting a secret in an environment restricted to `main` keep it away from that workflow?

Short answer: a workflow pushed to any branch reads every repository secret. A fork PR reads none. An environment
secret stays out of reach: a job without `environment:` gets it empty, and a job that asks for the environment from a
branch it doesn't allow is rejected before it starts.

Background: in the wfx-loot campaign (2026-11-10 to 2026-11-12), hijacked accounts pushed workflows to throwaway
branches and dumped `toJSON(secrets)`
([Endor Labs](https://www.endorlabs.com/learn/github-actions-supply-chain-attack-targeting-ci-secrets-at-scale)).

## Setup

Secrets and environments in this repository:

| Name | Where | Value |
| ---- | ----- | ----- |
| `MY_OWN_SECRET` | repository secret | set by hand, never printed |
| `MY_OWN_ENVIRONMENTS_SECRET` | environment `release`, deployment branches: `main` only | set by hand, never printed |
| `TEST_ENV_SECRET` | environment `test-secret-reachability`, deployment branches: `test/2026-11-13-secret-reachability` only | `dummy-not-a-real-secret` |

Workflow [`secret-reachability.yml`](.github/workflows/secret-reachability.yml) runs on a push to this branch, on a
`pull_request` targeting it (Markdown ignored), and on `workflow_dispatch`, with `permissions: {}`. Each step only
tests whether one secret is empty (`secrets.X != ''`) and never prints a value. It writes `reachable=true/false` to the
log and the step summary. An empty secret fails the step with an `::error::` annotation. Later steps run anyway
(`if: !cancelled()`), so every secret gets reported.

| Job | `environment` | Secrets checked |
| --- | ------------- | --------------- |
| `no-environment` | none | `MY_OWN_SECRET`, `MY_OWN_ENVIRONMENTS_SECRET` |
| `release-environment` | `release` | `MY_OWN_ENVIRONMENTS_SECRET`, `MY_OWN_SECRET` |
| `allowed-environment` | `test-secret-reachability`. Control job: an environment that does allow this branch | `TEST_ENV_SECRET`, `MY_OWN_SECRET` |

## How to run

- Same repository: push a non-Markdown change to this branch, or re-run from the Actions tab.
- Fork: push a branch with at least one extra commit to a fork, and open a PR from it into this branch.
  `fork-case.txt` was that commit for the run below.

## Expected

- Push to this branch: `no-environment` reads `MY_OWN_SECRET` but not `MY_OWN_ENVIRONMENTS_SECRET`.
  `release-environment` is rejected, because `release` only allows `main`. `allowed-environment` reads both of its
  secrets.
- Fork PR: GitHub doesn't pass secrets to workflows from forks, so `no-environment` reads nothing. The ref is
  `refs/pull/N/merge`, which neither environment allows, so both environment jobs are rejected.

## Result

Runs: [push](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/43445960823)
(commit `9cd120a`), [fork PR](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/43446037612)
([#14](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/pull/14) from
`Exypte/github-actions-miscellaneous-knowledge`, commit `2c5d41f`).

| Job | Push to this branch | Fork PR |
| --- | ------------------- | ------- |
| `no-environment` | `MY_OWN_SECRET` ✅ reachable, `MY_OWN_ENVIRONMENTS_SECRET` ❌ empty | `MY_OWN_SECRET` ❌ empty, `MY_OWN_ENVIRONMENTS_SECRET` ❌ empty |
| `release-environment` | ❌ rejected, never started | ❌ rejected, never started |
| `allowed-environment` | `TEST_ENV_SECRET` ✅ reachable, `MY_OWN_SECRET` ✅ reachable | ❌ rejected, never started |

The rejections, from the job annotations:

- Push: `Branch "test/2026-11-13-secret-reachability" is not allowed to deploy to release due to environment protection rules.`
- Fork PR: `Branch "refs/pull/14/merge" is not allowed to deploy to release due to environment protection rules.` and the
  same message for `test-secret-reachability`.
- Each followed by `The deployment was rejected or didn't satisfy other protection rules.`

A rejected job has no steps at all, so it never sees any secret, not even the repository one.

History: the same question was first explored on a `demo/secret-reachability-proof` branch, through PRs into `main`, and
gave the same answers. From a same-repository PR (#8), `MY_OWN_SECRET` was reachable
([43428217416](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/43428217416),
[43438122402](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/43438122402)).
From a fork PR (#10) it wasn't
([43428949240](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/43428949240)).
With `environment: release` the job was rejected for `refs/pull/8/merge`
([43439531016](https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/43439531016)).

## Conclusion

Anyone who can push a branch can read every repository secret: one workflow on any branch is enough. Fork PRs get
none. A secret in an environment whose deployment branches are limited to `main` can't be read from another branch or
from a PR. A job without `environment:` gets it empty, and a job that asks for the environment is rejected. The control
job shows that the environment mechanism itself works when the branch is allowed.

Keep deploy and publish secrets in an environment restricted to the default branch (and protect that branch), not as
repository secrets.
