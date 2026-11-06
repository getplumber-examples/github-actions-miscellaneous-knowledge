# Dependabot and unmerged branches

## Question

1. If an action repo has an unmerged `v1` branch and no `v1` tag, does Dependabot propose it to consumers,
   for a tag pin and for a SHA pin?
2. If it does, does the `V1` code run, and where?
3. Can a `v2` branch on a fork influence the update, on its own or through a PR into the action repo?

This comes from [getplumber/getplumber.io#536](https://github.com/getplumber/getplumber.io/pull/536), where
Dependabot bumped `getplumber/plumber@v0.7.18` to `@v1` and `v1` was a branch.
[The first run of that PR](https://github.com/getplumber/getplumber.io/actions/runs/41944645543) downloaded
the branch and ran it.

## Setup

This test uses separate repos, because Dependabot reads its config from the default branch and test
branches here are never merged. This branch only holds the write-up.

| Repo | Role |
| ---- | ---- |
| [print-version-demo](https://github.com/getplumber-examples/print-version-demo) | Composite action that prints its version. `main` and tag `v0.1.0` print `V0`. |
| [print-version-demo-consumer-tag](https://github.com/getplumber-examples/print-version-demo-consumer-tag) | Uses `print-version-demo@v0.1.0`. |
| [print-version-demo-consumer-sha](https://github.com/getplumber-examples/print-version-demo-consumer-sha) | Uses `print-version-demo@f95b8e2... # v0.1.0`. |
| `Exypte/print-version-demo` | Fork, with a `v2` branch printing `V2`. |

Both consumers run on `push` to `main`, `pull_request` and `workflow_dispatch`, with `permissions: {}`.
Dependabot checks `github-actions` daily, with the cooldown excluded for this action.

The stages run in this order, with a Dependabot check after each one:

1. Control: only `v0.1.0` exists.
2. Fork A: `v2` branch on the fork.
3. Fork B: PR from the fork's `v2` into the action repo, never merged.
4. PR from a `v1` branch on the action repo into `main`, never merged. No `v1` tag is created.

## How to run

Push the stage's change to the action repo or the fork, then change the comment on the first line of
`.github/dependabot.yml` on each consumer. Pushing that file starts a Dependabot check within seconds, so
there is no need to wait for the daily run.

## Expected

Written before the fork and the `v1` branch existed. It comes from reading dependabot-core:
`git_commit_checker.rb` only considers tags when the pinned SHA has a tag, and considers branches too
otherwise. `git_metadata_fetcher.rb` keeps only `refs/heads` and `refs/tags` of the action repo.

- Control: no PR on either consumer.
- Fork A and B: no PR on either consumer.
- `v1` branch, tag consumer: PR to `@v1`, and its run prints `V1`.
- `v1` branch, SHA consumer: no PR.
- `main` of both consumers keeps printing `V0`.

## Result

| Stage | Tag consumer | SHA consumer |
| ----- | ------------ | ------------ |
| Control | [No PR](https://github.com/getplumber-examples/print-version-demo-consumer-tag/actions/runs/42201566689) | [No PR](https://github.com/getplumber-examples/print-version-demo-consumer-sha/actions/runs/42201567585) |
| Fork A: `v2` on the fork | [No PR](https://github.com/getplumber-examples/print-version-demo-consumer-tag/actions/runs/42203702510) | [No PR](https://github.com/getplumber-examples/print-version-demo-consumer-sha/actions/runs/42203706486) |
| Fork B: [PR from the fork](https://github.com/getplumber-examples/print-version-demo/pull/1) | [No PR](https://github.com/getplumber-examples/print-version-demo-consumer-tag/actions/runs/42203885066) | [No PR](https://github.com/getplumber-examples/print-version-demo-consumer-sha/actions/runs/42203887678) |
| [`v1` branch and PR](https://github.com/getplumber-examples/print-version-demo/pull/2) | [PR to `@v1`](https://github.com/getplumber-examples/print-version-demo-consumer-tag/actions/runs/42204001806) | [No PR](https://github.com/getplumber-examples/print-version-demo-consumer-sha/actions/runs/42204007205) |

All four fork-stage logs say:

```
Latest version is 0.1.0
No update needed for getplumber-examples/print-version-demo 0.1.0
```

None of them mention the fork, even though the fork PR had created `refs/pull/1/head` on the action repo.

At the `v1` stage, the tag consumer's log says:

```
Latest version is 1
Updating getplumber-examples/print-version-demo from 0.1.0 to 1
```

Dependabot opened [consumer-tag#1](https://github.com/getplumber-examples/print-version-demo-consumer-tag/pull/1),
"Bump getplumber-examples/print-version-demo from 0.1.0 to 1":

```diff
-      - uses: getplumber-examples/print-version-demo@v0.1.0
+      - uses: getplumber-examples/print-version-demo@v1
```

[The `pull_request` run of that PR](https://github.com/getplumber-examples/print-version-demo-consumer-tag/actions/runs/42204071970)
ran the unmerged code:

```
Download action repository 'getplumber-examples/print-version-demo@v1' (SHA:62fd2ca978b7b2e6e61a02583cd70605527f50f0)
V1
```

`main` kept printing `V0` on [the tag consumer](https://github.com/getplumber-examples/print-version-demo-consumer-tag/actions/runs/42203995648)
and [the SHA consumer](https://github.com/getplumber-examples/print-version-demo-consumer-sha/actions/runs/42204001383).

The Marketplace listing was not published when these stages ran, so this result doesn't depend on it.
Only one case of each pin was run, and a pin to a SHA with no tag was not tested.

## Conclusion

A version-like branch on the action repo is enough for Dependabot to propose it to consumers pinned by tag,
and that unreleased code runs in the PR's CI, then on `main` if the PR is merged. A SHA pin on a tagged commit
was not offered the branch, and a fork's branch had no effect, even through a PR.

## Lockfile

### Question

With a [lockfile](https://docs.github.com/en/actions/how-tos/secure-your-work/lock-workflow-dependencies),
does Dependabot still propose the `v1` branch, and does the PR run `V1`? If `v1` then moves to new code, does
the PR keep running the locked commit?

### Setup

[print-version-demo-consumer-lock](https://github.com/getplumber-examples/print-version-demo-consumer-lock) is
the tag consumer plus `.github/workflows/actions.lock`, generated with `gh actions-lock`. The print action is
its only dependency. `v1` already exists when this repo is created, so Dependabot's first check sees it.

After the Dependabot PR has run, `v1` moves to a commit printing `V1-moved`, and the PR runs on the lock and
tag consumers are re-run. The tag consumer is the control: it must print the moved code.

### Expected

Written before the repo existed.

- `main` prints `V0`, and `gh actions-lock --verify` passes.
- Dependabot opens a PR to `@v1` that changes both `print.yml` and `actions.lock`.
- The PR's run prints `V1`, and `--verify` passes on the PR branch. The lockfile does not block it.
- After `v1` moves, the re-run on the lock consumer still prints `V1`, and the re-run on the tag consumer
  prints `V1-moved`.

### Result

`main` [printed `V0`](https://github.com/getplumber-examples/print-version-demo-consumer-lock/actions/runs/42207476505),
and `gh actions-lock --verify` reported `All 1 workflow valid`.

[Dependabot's first check](https://github.com/getplumber-examples/print-version-demo-consumer-lock/actions/runs/42207484035)
opened [consumer-lock#1](https://github.com/getplumber-examples/print-version-demo-consumer-lock/pull/1) to `@v1`.
It changed `print.yml` and regenerated `actions.lock`, pinning the unmerged branch:

```diff
-    'getplumber-examples/print-version-demo@v0.1.0':
-        ref: 'v0.1.0'
-        commit: 'sha1-f95b8e245021b29ca2ed35c1f14589a12142b068'
+    'getplumber-examples/print-version-demo@v1':
+        ref: 'v1'
+        commit: 'sha1-62fd2ca978b7b2e6e61a02583cd70605527f50f0'
```

`--verify` also passed on the PR branch, and the PR's run printed `V1`:

```
Using locked action versions from the workflow's lockfile
Download action repository 'getplumber-examples/print-version-demo@v1' (SHA:62fd2ca978b7b2e6e61a02583cd70605527f50f0)
V1
```

Then `v1` moved to `9f55230`, which prints `V1-moved`, and both PR runs were re-run (attempt 2):

| Consumer | Downloaded SHA | Printed |
| -------- | -------------- | ------- |
| [Lock](https://github.com/getplumber-examples/print-version-demo-consumer-lock/actions/runs/42207565928) | `62fd2ca` | `V1` |
| [Tag](https://github.com/getplumber-examples/print-version-demo-consumer-tag/actions/runs/42204071970) | `9f55230` | `V1-moved` |

`--verify` still reported `All 1 workflow valid` on the lock PR branch after the move, and both runs passed.

`gh actions-lock` warned `Dependabot cooldown include/exclude filters are not supported and were ignored`.
Dependabot itself still opened the PR on its first check, minutes after `v1` was created.

No **Require lockfile** policy was set, and `--verify` ran locally, not in CI.

### Conclusion

A lockfile doesn't stop Dependabot from proposing an unmerged branch: it locks that branch's commit in the same
PR, `--verify` passes, and the code runs. What it adds is that later commits on the branch don't run until the
lockfile changes, so the commit to review is the one in the PR diff.
