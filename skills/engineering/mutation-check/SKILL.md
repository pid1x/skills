---
name: mutation-check
description: Mutation-test a change — detect the project's mutation tool, mutate only the diff (a branch, PR, or ticket), and report the tests that let mutants live. Use to prove tests catch regressions, not just pass.
argument-hint: "<branch | PR url/number | ticket key | path>  — omit for the current branch vs its base"
---

A green suite proves nothing. Mutation testing **breaks the code on purpose** and asks one question: *did a test notice?* A **surviving mutant** is a hole — logic your tests don't actually pin. This skill hunts those holes on **one change**, never the whole repo.

## 1. Scope to the change — never the whole suite
Mutate only what the change touched. That is what makes this fast enough to run per PR. Resolve the target → its changed files:

- **Path / glob** → use as-is.
- **Branch** (or no argument) → `git fetch origin <base>`, then `git diff --name-only origin/<base>...HEAD` (`base` = `main`/`master`; `...` diffs from the merge-base, and the fetch avoids comparing against a stale local base).
- **PR** (url or number) → `gh pr diff <pr> --name-only`; check out its head first if not local.
- **Ticket key** (e.g. `ABC-123`) → resolve its PR (`gh pr list --search "<KEY> in:title"`), then treat as a PR.

Keep source files only — drop tests, config, generated code. Mutate **those files**, nothing else.

**Done when:** every changed source file is listed; every dropped path has a one-line reason (test, config, or generated).

## 2. Detect the tool — once per project, then remember
`.mutation-check.json` is a **local per-project cache** — not committed, and it **must be ignored by git**. First run in a repo: no file exists → detect, write it, and ensure git ignores it — if `git check-ignore .mutation-check.json` fails, add it to `.git/info/exclude` (local-only, never touches tracked files); only edit the repo's `.gitignore` if the user asks. Later runs in the same repo read it and skip detection. `--reconfigure` redoes detection.

Read the manifest, confirm the binary is installed, pick the tool:

| Manifest | Tool | Scoped run |
|---|---|---|
| `composer.json` | Pest `--mutate` (or Infection) | `pest --mutate --covered-only <paths>` |
| `package.json` | Stryker | `stryker run --mutate <paths>` |
| `pyproject.toml` / `setup.py` | `mutmut` (or `cosmic-ray`) | `mutmut run --paths-to-mutate <paths>` |
| `Cargo.toml` | `cargo-mutants` | `cargo mutants --file <paths>` |
| `pom.xml` / gradle | PIT | target the changed classes |
| `Gemfile` | `mutant` | `mutant run <subjects>` |

Schema — tool, the scoped-run command template (`<paths>` substituted at runtime), and where it was detected:

```json
{ "tool": "pest", "command": "pest --mutate --covered-only <paths>", "detectedFrom": "composer.json" }
```

**Done when:** the tool binary is verified (`--version` or equivalent); `.mutation-check.json` is written or read; and `git check-ignore .mutation-check.json` passes (via existing ignore rules or `.git/info/exclude`, added on first write if missing).

## 3. Run — covered, scoped, parallel
Run the detected tool on the scoped paths. Always: **covered code only** (an untested line's mutant is noise, not signal), **parallel** if supported.

**Pre-flight feasibility** — before launching, identify which tests cover the diff. If the covered lines are reached **only by heavy integration tests** (e.g. through an HTTP kernel), the coverage baseline is effectively infeasible → declare **`no result — integration-only coverage`** upfront instead of starting a run that will hang.

**Watchdog** — mutation is slow; bail loudly, never hang:
- **PR / review context** (PR arg, ticket key, or invoked from a review lens): **10 minutes**
- **Standalone** (branch, path, or no arg): **30 minutes**

No mutant results by the deadline → **abort** → **`no result — baseline timed out`**. If the deadline hits *after* some mutants already scored, that is a **`score`** — report it with the **bounded scope stated** (e.g. `score 50% · 2 survivors (4 of ~30 mutants — budget)`), so the number cannot be read as a coverage verdict. There are **exactly three** outcomes (step 4) and no fourth wording: never invent `partial`.

**Hand-applied fallback — when the tool can't produce a real score.** Two signals that the tool's number is not real: it **can't target the changed code** (legacy classes, or string-literal predicates inside a query that a mutation tool won't mutate), or it returns an **artifact** — a single generated mutant, or all-timeout / 0-tested dressed up as `100%`. In either case the tool's score is hollow; **do not report it as a score.** Fall back to a **hand-applied pass** on the changed lines: pick the highest-value mutants on the diff's own logic — a boundary flip (`<` → `<=`), a dropped clause (remove `AND rechnung != ''`), an inverted ternary or boolean (`? 0 : 1` → `? 1 : 0`) — then for each: **apply → run the covering test → revert.** A survivor is any mutant the covering test still passes on. This is agent-executable and pins the logic the tool couldn't reach.

Bound it to the same watchdog: hand-applying is only worthwhile when the covering test is **fast enough to re-run a handful of times within budget.** If the covering tests are heavy integration tests (each run near the whole budget), even a bounded hand-applied pass busts it → report `no result — integration-only coverage` instead. A hand-applied score reports like any other, tagged **`(hand-applied)`**.

**The one sanctioned source write — and its revert guarantee.** A review otherwise **never writes source files** (`deep-review` step 2). The hand-applied pass is the single, explicit exception: each mutant is a **transient edit to already-checked-out source**, reverted before the next one. Make that revert **unconditional, not best-effort** — run it the way a `finally` would, so it also fires when the **watchdog aborts mid-mutant**, when a test errors, or when the run is interrupted. Then **verify the tree is clean** (`git diff --quiet` over the touched files) *before* reporting. If a revert cannot be completed, say so **at the top of the report** and name the file left modified — an unreverted mutant is a corrupted working tree, far worse than a missing score. And never hand-apply on a **dirty** tree: you could not tell your edit from the user's, so report `no result — dirty tree` instead.

**Done when:** the run produced a mutation score (tool or hand-applied), or it declared exactly one `no result — <reason>` (infeasible baseline or timeout) — never a silent drop, an indefinite hang, a hollow artifact score reported as real, or a zero-output abort dressed up as a score. Any hand-applied mutant is reverted and the tree verified clean.

## 4. Report — one of exactly three outcomes
Every run ends on **exactly one** of these, never an improvised wording:

- **`score N% · M survivors`** — a **real** run completed (M may be 0), from the tool or the hand-applied fallback; tag the latter `(hand-applied)`. A hollow tool artifact (single-mutant `100%`, all-timeout) is **not** a score — it routes to the fallback or `no result`, never reported as `score`. Per survivor, state four things: **file · line · the mutation** (e.g. "`>` → `>=`", "call removed") **· the test that should have killed it but didn't.** Rank by **blast radius** — survivors in the most-depended-on code first (changed files with the most callers, then by mutated-line count). Exclude provably-equivalent mutants from the score and say so. Close with the single test worth writing first.
- **`no covered tests in diff`** — the changed source has no covering tests to mutate. That is itself a finding, not a skip.
- **`no result — <specific reason>`** — no score could be produced (`integration-only coverage`, `baseline timed out`, `dirty tree`, tool error). A zero-output abort is `no result` — never dressed up as a score, and never a fourth wording. If some mutants *did* score before the budget ran out, that is `score` with its bounded scope stated (step 3).

**Bottom line.** Close the report with one tagged status line — the single line a reader or an orchestrator folds first: `[mutation] score N% · M survivors` / `[mutation] no covered tests in diff` / `[mutation] no result — <reason>`.

**Done when:** the report carries exactly one of the three outcomes; a `score` outcome gives every survivor all four fields and ends with one recommended test; and the report ends with the `[mutation]` bottom-line status.

## When nothing fits
- **No source files in the diff** → after filtering (step 1) nothing is left to mutate (e.g. docs- or config-only change) → report "no testable source changed" and stop before detecting a tool.
- **Tool not installed** → name the right one for the detected stack + the one-line install. Do not invent a run.
- **Stack not recognized** → say so; ask for the mutate command, then record it in `.mutation-check.json`.
- **No covered lines in the diff** → **`no covered tests in diff`** — that is itself a finding (step 4).
