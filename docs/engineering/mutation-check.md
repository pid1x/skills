Quickstart:

```bash
npx skills add pid1x/skills --skill=mutation-check
```

```bash
npx skills update mutation-check
```

[Source](https://github.com/pid1x/skills/tree/main/skills/engineering/mutation-check)

## What it does

`mutation-check` mutation-tests **one change** — a branch, PR, ticket, or path — and reports the tests that let mutants survive. A green suite only proves the tests pass; mutation testing breaks the code on purpose and asks whether any test noticed.

It detects the project's mutation tool on first run (Pest, Stryker, mutmut, cargo-mutants, PIT, mutant, …), caches the choice in a local `.mutation-check.json` (ignored via `.git/info/exclude` by default — never committed), and scopes every run to covered lines in the diff — never the whole repo. Branch scopes fetch `origin/<base>` first so the diff isn't against a stale local base.

## When to reach for it

Type `/mutation-check`, pass a branch/PR/ticket/path as an argument, or let the agent reach for it when you want to prove tests catch regressions — not just pass.

Reach for it after testable logic changes, especially on a PR, when you want to know which assertions are actually pinning behaviour. Pair it with a test-first workflow when you want to verify the suite is sharp, not just green.

## Scoped diff, not the whole suite

The leading idea is **scoped mutation**: only files the change touched, only lines already covered by tests. That keeps per-PR runs fast enough to be useful. Survivors name weak tests — file, line, the mutation, and the test that should have killed it but didn't.

## It's working if

- It mutates only the diff's source files, not the whole codebase.
- It runs on covered code only and ends on exactly one of three outcomes — `score N% · M survivors` / `no covered tests in diff` / `no result — <reason>`. A run that hangs is aborted by a watchdog (10 min in PR/review context, 30 min standalone) and reported as `no result — baseline timed out`, never dressed up as `partial`.
- When the tool can't target the changed code (legacy classes, string-literal predicates) or returns a hollow artifact score (single-mutant `100%`, all-timeout), it falls back to a hand-applied pass — apply each high-value mutant, run the covering test, revert — tagged `(hand-applied)`, or reports `no result` if the covering tests are too heavy to re-run within budget. A hollow artifact is never reported as a real score.
- The hand-applied pass is the **only** place a review touches source, and the revert is guaranteed rather than best-effort: it also runs when the watchdog aborts mid-mutant, when a test errors, or when the run is interrupted, and the tree is verified clean before reporting — a revert that fails is named at the top of the report. It refuses to hand-apply on a dirty tree (`no result — dirty tree`), where its own edit couldn't be told from the user's.
- The report ranks survivors by blast radius, excludes provably-equivalent mutants, and names the single test worth writing first.
- Every run ends on a single tagged `[mutation]` bottom-line, so its verdict folds into a fuller review without re-reading the report.

## Where it fits

`mutation-check` complements the test feedback loop — test-first development builds behaviour with a red-green loop; `mutation-check` stress-tests whether those tests would catch a regression if the implementation drifted.
