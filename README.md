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

Run: <https://github.com/getplumber-examples/github-actions-miscellaneous-knowledge/actions/runs/44718136654>,
all 10 jobs green (`commit-message`, 8 `matrix-payloads` legs, `operand-leak`).

182 evaluations of the safe forms (22 on the commit message, 20 × 8 matrix payloads): every one rendered
`true` or `false`, and `/tmp/injected` never appeared. The step logs show the rendered script line as
`: true` / `: false`. Rendered values:

| Expression | commit message | `''` | `/approve` | `$(touch …)` | `true` | `false` | `0` | `1` | `null` |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `F == '/approve'` | `false` | `false` | `true` | `false` | `false` | `false` | `false` | `false` | `false` |
| `F != '/approve'` | `true` | `true` | `false` | `true` | `true` | `true` | `true` | `true` | `true` |
| `F < 'm'` | `false` | `true` | `true` | `true` | `false` | `true` | `true` | `true` | `false` |
| `F > 'm'` | `true` | `false` | `false` | `false` | `true` | `false` | `false` | `false` | `true` |
| `F <= 'm'` | `false` | `true` | `true` | `true` | `false` | `true` | `true` | `true` | `false` |
| `F >= 'm'` | `true` | `false` | `false` | `false` | `true` | `false` | `false` | `false` | `true` |
| `F == 0` | `false` | `true` | `false` | `false` | `false` | `false` | `true` | `false` | `false` |
| `F > 1` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` |
| `!F` | `false` | `true` | `false` | `false` | `false` | `false` | `false` | `false` | `false` |
| `!!F` | `true` | `false` | `true` | `true` | `true` | `true` | `true` | `true` | `true` |
| `!(F == 'x')` | `true` | `true` | `true` | `true` | `true` | `true` | `true` | `true` | `true` |
| `contains(F, 'touch')` | `true` | `false` | `false` | `true` | `false` | `false` | `false` | `false` | `false` |
| `startsWith(F, 'Test')` | `true` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` |
| `endsWith(F, ')')` | `true` | `false` | `false` | `true` | `false` | `false` | `false` | `false` | `false` |
| `F == 'a' && contains(F, 'b')` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` |
| `startsWith(F, 'x') \|\| endsWith(F, 'y')` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` |
| `!(F == 'x') \|\| contains(F, 'touch')` | `true` | `true` | `true` | `true` | `true` | `true` | `true` | `true` | `true` |
| `(F == 'a' \|\| F == 'b') && !contains(F, 'c')` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` |
| `(contains(F, 'touch') && F) == 'x'` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` | `false` |
| `github.event == 'x'` | `false` | – | – | – | – | – | – | – | – |
| `!github.event.does_not_exist` | `true` | – | – | – | – | – | – | – | – |

| Control | Rendered | Payload ran |
| --- | --- | --- |
| `true && F` | `Test 2026-11-20: boolean expressions $(touch /tmp/injected)` | yes, `/tmp/injected` created |
| `false \|\| F` | `Test 2026-11-20: boolean expressions $(touch /tmp/injected)` | yes, `/tmp/injected` created |
| `F == 'x' \|\| F` | `Test 2026-11-20: boolean expressions $(touch /tmp/injected)` | yes, `/tmp/injected` created |
| `contains(F, 'touch') && F` | `Test 2026-11-20: boolean expressions $(touch /tmp/injected)` | yes, `/tmp/injected` created |
| `F \|\| false` | `Test 2026-11-20: boolean expressions $(touch /tmp/injected)` | yes, `/tmp/injected` created |
| `env.EMPTY && true` | *(empty)* | no (no payload in the value) |

The control steps log the payload reaching the shell, for example in `true && F`:

```text
: Test 2026-11-20: boolean expressions $(touch /tmp/injected)
```

Side observations:

- String comparisons are case-insensitive (`Test… > 'm'` is `true`), and `F == 0` coerces the string:
  `''` and `'0'` both equal `0`.
- `null` and `false` from the matrix are strings, so `!!F` is `true` for them; only `''` is falsy.
- `env.EMPTY && true` renders an empty string, not `false`: `&&` returns its first falsy operand.

## Conclusion

Comparisons, `!`, `contains()` / `startsWith()` / `endsWith()`, and `&&` / `||` over only those always
rendered `true` or `false`, whatever the field held, so they cannot carry a payload into `run:`. As soon as
one `&&` / `||` operand is the field itself, its value is rendered as-is and the payload runs, so #624 must
keep firing on those.
