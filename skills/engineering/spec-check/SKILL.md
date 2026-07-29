---
name: spec-check
description: Check a change against the spec it was meant to satisfy — the originating ticket's acceptance criteria — for missing/partial requirements, scope creep, and implemented-but-wrong behaviour. Severity is judged independently of the author's framing. Use when reviewing whether a PR/branch/diff actually does what was asked.
argument-hint: "<PR url/number | ticket key | branch>  — omit to derive the ticket from the current branch"
---

Passing review is not the same as doing what was asked. `spec-check` holds the change against its **spec** — the originating ticket's expected behaviour and acceptance criteria — and names where the two diverge. Its one hard rule: **severity is the skill's own call.** An author's "known limitation" is context for the judgement, never permission to lower it. That is what stops a disclosed regression from being waved through.

## 1. Get the spec — the source of truth to judge against
Resolve the target → its change, then find the **spec** it is meant to satisfy. `spec-check` is tracker-agnostic — take the spec from whatever source is reachable, in order:

- **Ticket key** (or derivable from the branch/PR title) **+ a connected tracker** (Jira/Linear/GitHub Issues MCP) → fetch the ticket's description, acceptance criteria, **and its comments** — a requirement is often clarified, narrowed, or waived in the discussion, not the description.
- **No tracker, but a PR** → use the PR description **and its review/comment thread** as the stated intent.
- **Neither** → ask the user to paste the acceptance criteria.

**Read the clarifications, not just the description.** A gap in the written spec may already be resolved in the ticket's comments — the **reporter or spec-owner** narrowing or waiving a requirement. Such a clarification **amends the spec you judge against**: if the requirement-owner says English-only is acceptable, then English-only conforms and there is no gap. This is distinct from the **author's** self-disclosure of a shortfall (step 3), which is context, not permission. The difference is *who spoke*: the requirement-owner amends the requirement; the implementer disclosing a limitation does not. Attribute each clarification before you rely on it.

`.spec-check.json` is an optional **local per-project cache** of the spec source (`{ "specSource": "mcp:jira" | "github-issues" | "pr-description" }`) — not committed, and it **must be ignored by git**: if `git check-ignore .spec-check.json` fails, add it to `.git/info/exclude` (local-only). Record no ticket keys or tracker project names in it — the source *kind* only. `--reconfigure` redoes source detection.

A change with **no clear spec or acceptance criteria** anywhere is the **one valid skip**: report `no spec to check against` and stop. Judging a change against a spec you had to invent is worse than not judging it.

**Done when:** the acceptance criteria / expected behaviour **and any clarifying comments** are in hand from a named source, with reporter/spec-owner amendments folded into the spec; or the change is declared `no spec to check against`.

## 2. Find the gaps — three kinds, quote the line
Read the diff against the spec and surface every divergence as one of exactly three kinds. **Quote the spec line** each finding answers to:

- **Missing / partial** — a requirement the spec asks for that the change does not deliver, or delivers only part of.
- **Scope creep** — behaviour in the diff the spec never asked for.
- **Implemented-but-wrong** — a requirement the change appears to address but gets wrong.

Read the actual behaviour, not the author's summary of it — a PR title claiming "adds X" is a claim to check against the diff, not a fact.

**Done when:** every divergence is listed as one of the three kinds with the spec line it answers to; a change that fully conforms is stated as `conforms ✓`.

## 3. Assign severity independently — the author's framing is context, not permission
Judge each finding's severity yourself, from the **residual behaviour** — what the code actually does after the change — not from how the author labelled it.

- An author's **"known limitation" / "follow-up ticket" / "tracked partial"** note is **context, not permission.** A disclosed limitation is still a limitation the human reviewer gets to reject. Judge the residual behaviour as if the note were absent, then note that it was disclosed. (A **reporter/spec-owner** clarification is the opposite — it amends the spec itself; fold it in at step 1, don't treat it as a gap here.)
- **Meeting the literal acceptance criteria is NOT a clean bill of health.** If the change satisfies the written AC but leaves **persistent state wrong** — a stored record (ledger, inventory, accounting, serial/expiry, saved config) left in a state that misleads downstream logic — that is a **blocking-class finding**, even when disclosed and even when the literal AC passes.
- **LITMUS — "is this partial actually shippable?"** Residual **persistent-wrong-state → blocking**; a merely cosmetic or label gap → minor. **NEVER downgrade a correctness or data-integrity regression** to fit the author's framing or the literal AC.

Severity is the skill's own call — this is not about merge authority (the skill never approves or blocks a merge), it is about not softening the language.

**Done when:** every finding carries a severity set from the residual behaviour; each finding touching persistent state has been tested against the litmus; no finding inherited its severity from an author disclosure.

## 4. Report
Per finding: **kind · the quoted spec line · severity · the residual behaviour** in one line. Lead with the blocking-class findings. Give the single honest read: is the change shippable against its spec, or not — and if a disclosed limitation drove a blocking severity, say so plainly.

**Bottom line.** Close the report with one tagged status line — the single line a reader or an orchestrator folds first: `[spec] conforms ✓` or `[spec] N gaps · M blocking` (name the blocking driver, e.g. `persistent-wrong-state`). Add `· K notes` if disclosed-but-non-blocking observations were recorded.

**Done when:** every finding has all four fields; blocking-class findings are surfaced as such; and the report ends with the `[spec]` bottom-line status (or `[spec] conforms ✓`).

## When nothing fits
- **No spec** → no reachable ticket/AC/PR-description and none pasted → `no spec to check against`, stop (step 1).
- **Fully conforms** → no divergence in any of the three kinds → `conforms ✓`, and say what spec it was judged against.
- **Author disclosed the gap** → not a reason to skip or soften — record the disclosure, judge the residual behaviour on its own (step 3).
