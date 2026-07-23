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

**Time budget** — mutation is slow; bail loudly, never hang:
- **PR / review context** (PR arg, ticket key, or invoked from a review lens): **10 minutes**
- **Standalone** (branch, path, or no arg): **30 minutes**

On timeout: report whatever survivors you have so far, then **`skipped — time budget`** — never a silent drop or an indefinite hang.

**Done when:** the run finished within budget, or timed out with partial results and an explicit `skipped — time budget` line.

## 4. Report — a surviving mutant names a weak test
Per survivor, state four things: **file · line · the mutation** (e.g. "`>` → `>=`", "call removed") **· the test that should have killed it but didn't.** Give the mutation score. Rank by **blast radius** — survivors in the most-depended-on code first (changed files with the most callers, then by mutated-line count). Close with the single test worth writing first.

**Done when:** every survivor has all four fields; the report ends with one recommended test (or states zero survivors).

## When nothing fits
- **No source files in the diff** → after filtering (step 1) nothing is left to mutate (e.g. docs- or config-only change) → report "no testable source changed" and stop before detecting a tool.
- **Tool not installed** → name the right one for the detected stack + the one-line install. Do not invent a run.
- **Stack not recognized** → say so; ask for the mutate command, then record it in `.mutation-check.json`.
- **No covered lines in the diff** → report "no tested code changed" — that is itself a finding.
