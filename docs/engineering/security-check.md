Quickstart:

```bash
npx skills add pid1x/skills --skill=security-check
```

```bash
npx skills update security-check
```

[Source](https://github.com/pid1x/skills/tree/main/skills/engineering/security-check)

## What it does

`security-check` security-reviews **one change** — a branch, PR, ticket, or path — against the known vuln-class taxonomy, then traces the data-flow that separates a real finding from noise. A dangerous-looking sink is only a finding if tainted input can actually reach it.

It walks the diff against the vuln classes (injection, XSS, SSRF, IDOR, auth bypass, unsafe deserialization, path traversal, hardcoded secrets, and more), then follows each candidate from its external entry point to the sink across files. If the repo carries a `claude-security-guidance.md`, it applies those org rules on top of the taxonomy.

## When to reach for it

Type `/security-check`, pass a PR/branch/ticket/path as an argument, or let the agent reach for it when a change touches input handling, auth, queries, file paths, deserialization, or secrets.

Reach for it on any change with a security surface. Skip it when the diff touches none — docs, styling, internal refactors with no external input — where there is nothing to trace.

## Reachability sets severity

The leading idea is **reachability**: severity comes from whether tainted input can reach the sink, not from how alarming the sink looks. A candidate with a traced path from an external boundary is a finding; a candidate no external input can reach is informational, and stays out of the findings column.

**Litmus:** if there is no traced path from an external entry point, it is not a finding. The trace is the verdict — a scary sink with no reachable path is a note, not an alarm. This is what keeps the review honest instead of a wall of false positives.

## It's working if

- Every candidate is traced from an external entry point to its sink across files, not judged at the changed line.
- Reachable candidates become findings with severity; unreachable ones stay informational, with the reason they cannot be reached.
- Findings and informational notes are reported separately — an unreachable sink is never promoted to a finding.
- A diff with no security surface is declared `no security surface` rather than scanned for its own sake.
- Every run ends on a single tagged `[security]` bottom-line, so its verdict folds into a fuller review without re-reading the report.

## Where it fits

`security-check` is the vulnerability lens of a fuller review. It pairs with `spec-check` (does the change do what was asked?), `verify-live` (does it behave correctly at runtime?), and `mutation-check` (does the suite catch regressions?). Where the built-in `security-review` covers pending local changes broadly, `security-check` scopes to one change and makes reachability the gate on every finding.
