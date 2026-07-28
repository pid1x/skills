Quickstart:

```bash
npx skills add pid1x/skills --skill=spec-check
```

```bash
npx skills update spec-check
```

[Source](https://github.com/pid1x/skills/tree/main/skills/engineering/spec-check)

## What it does

`spec-check` holds **one change** — a branch, PR, or ticket — against the spec it was meant to satisfy, and names where the two diverge. Passing a code review is not the same as doing what was asked; `spec-check` answers the second question.

It surfaces divergences as three kinds — missing/partial requirements, scope creep, and implemented-but-wrong behaviour — and quotes the spec line each finding answers to. It is tracker-agnostic: the spec comes from a connected tracker (Jira/Linear/GitHub Issues) if reachable, else the PR description, else pasted acceptance criteria. It reads the ticket's **comments** too — a reporter or spec-owner clarification often narrows or waives a requirement, and that amends the spec rather than leaving a gap to flag.

## When to reach for it

Type `/spec-check`, pass a PR/branch/ticket as an argument, or let the agent reach for it when reviewing whether a change actually delivers what its ticket asked for.

Reach for it on any change with a stated intent — an acceptance-criteria ticket, a spec, a described PR. Skip it when there is no clear spec to judge against; inventing one to score against is worse than not scoring.

## Independent severity

The leading rule is **independent severity**: the skill judges each finding's weight from the residual behaviour — what the code actually does after the change — not from how the author framed it.

An author's "known limitation" or "follow-up ticket" note is context, not permission — a disclosed limitation is still a limitation a reviewer gets to reject. And meeting the literal acceptance criteria is not a clean bill of health: a change that satisfies the written AC but leaves **persistent state wrong** — a stored record that misleads downstream logic — is a blocking-class finding, disclosed or not.

**Litmus:** *is this partial actually shippable?* Residual persistent-wrong-state → blocking; a cosmetic gap → minor. A correctness or data-integrity regression is never downgraded to fit the framing.

## It's working if

- Every finding is one of the three kinds and quotes the spec line it answers to.
- Severity is set from the residual behaviour, and no finding inherits its severity from an author disclosure.
- A change with no reachable spec is declared `no spec to check against` rather than scored against an invented one.
- The report leads with blocking-class findings and ends with an honest shippable / not-shippable read.
- Every run ends on a single tagged `[spec]` bottom-line, so its verdict folds into a fuller review without re-reading the report.

## Where it fits

`spec-check` is the intent lens of a fuller review. It pairs with `verify-live` (does the change behave correctly at runtime?) and `mutation-check` (does the suite catch regressions?); `spec-check` answers what neither can — does the change do what was actually asked, and is any shortfall shippable? Its severity call is comment-level: it names blocking concerns but never approves or blocks a merge.
