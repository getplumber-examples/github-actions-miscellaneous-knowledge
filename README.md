# Test 2026-11-20: boolean expressions in `run:`

## Question

When a `${{ }}` expression over an attacker-controlled field is a comparison (`==`, `!=`, `<`, `>`, `<=`,
`>=`), a `!` negation, a `contains()` / `startsWith()` / `endsWith()` call, or an `&&` / `||` combination
of only those, does it always render `true` or `false` into the script, whatever the field holds? And do
`&&` / `||` still leak an operand when one of them is the field itself?

Context: [getplumber/plumber#624](https://github.com/getplumber/plumber/issues/624) (ISSUE-207 false
positive on expressions that evaluate to a boolean).

## Setup

[`.github/workflows/boolean-expressions.yml`](.github/workflows/boolean-expressions.yml), triggered on push
to this branch. Every step renders the same expression twice:

- inline, as `: ${{ <expr> }}` in the script (the pattern ISSUE-207 flags). If the rendered value carries
  `$(touch /tmp/injected)`, the shell runs it;
- through `env: V`, to read the value safely and assert on it.

Jobs:

- `commit-message`: `F` = `github.event.head_commit.message`. The commit that pushed the workflow has the
  message `Test 2026-11-20: boolean expressions $(touch /tmp/injected)`. Each safe form must render
  `true` / `false` and leave no `/tmp/injected`. Also `github.event == 'x'` (object operand) and
  `!github.event.does_not_exist` (null operand).
- `matrix-payloads`: the same safe forms with `F` = `matrix.payload`, over `''`, `/approve`,
  `$(touch /tmp/injected)`, `true`, `false`, `0`, `1`, `null`, to cover empty, number-like and boolean-like
  strings.
- `operand-leak` (control): `true && F`, `false || F`, `F == 'x' || F`, `contains(F, 'touch') && F`,
  `F || false` must render the message and run the payload. `env.EMPTY && true` with an empty value must
  render something that is not a boolean.

Each step's rendered value goes to the job summary.

## How to run

Push to this branch (with the payload as the head commit message), or re-run the workflow from the Actions
tab.

## Expected

- `commit-message` and `matrix-payloads`: every step renders `true` or `false`, the payload never runs.
- `operand-leak`: every injection case renders the commit message and creates `/tmp/injected`;
  `env.EMPTY && true` renders an empty string. So an operand that is not itself boolean-producing keeps
  the expression injectable, even next to a comparison.

## Result

_Pending the run._

## Conclusion

_Pending the run._
