---
name: security-check
description: Security-review a change against a vuln-class taxonomy, tracing tainted input from external entry point to sink across files — reachability sets severity, so a sink unreachable from a boundary is informational, not a finding. Use when a PR/branch/diff touches input handling, auth, queries, file paths, deserialization, or secrets.
argument-hint: "<PR url/number | ticket key | branch | path>  — omit for the current working diff"
---

A vulnerability is only real if tainted input can reach the sink. `security-check` reviews a change against the known **vuln-class taxonomy**, then does the work that separates a finding from noise: it **traces the data-flow** from an external entry point to the sink, across files. **Reachability sets severity** — a dangerous-looking sink no external input can reach is informational, not a finding. That is what keeps the review honest instead of a wall of false positives.

## 1. Scope to the security surface — and load org rules
Resolve the target → its changed files, then decide whether the change touches a security-relevant surface at all: input handling, auth/permission checks, queries, file paths, deserialization, external requests, or secrets. A diff that touches **none** of these is the **one valid skip**: report `no security surface` and stop.

If the repo carries a **security-guidance file** — `claude-security-guidance.md` or a `.claude/security-*.md` in the repo or `~/.claude/` — load it and apply its project-specific rules on top of the taxonomy below. These are git-tracked org rules, not a local cache; there is no per-project detection to persist for this skill.

**Done when:** the change is declared `no security surface`, or its security-relevant surfaces are named and any security-guidance file has been loaded.

## 2. Check the taxonomy against the diff
Walk the changed lines against the vuln classes. For each security-relevant change, name the candidate class:

- **Injection** — SQL, command, LDAP, template.
- **XSS** — reflected, stored, DOM.
- **SSRF** — server-side request forgery.
- **IDOR / broken access control** — object references or actions with no ownership check.
- **Auth / permission bypass** — missing or wrong authz on a protected path.
- **Unsafe deserialization** — untrusted data into an object graph.
- **Path traversal** — user-influenced file paths.
- **Hardcoded secrets** — keys, tokens, credentials in the diff.
- **Open redirect · SSTI · XXE · insecure crypto · mass assignment · CSRF** — check as the surface warrants.

**Done when:** every security-relevant change has a candidate vuln class named, or is cleared with a one-line reason.

## 3. Trace reachability — external entry point → sink, across files
This is the step that sets truth. For each candidate, do **not** stop at the changed line — trace the **tainted input** from where it enters the system (an HTTP handler, a queue message, a CLI arg, an uploaded file) to the sink, following it **across files** with Read / Grep / Glob.

The trace reads the surrounding code, so the change must be present locally: a PR not checked out → check out its head first (restore the original branch afterwards); a dirty tree → trace against the working copy as-is. A review never writes source files.

- A sink with a **traced path from an external boundary** → a real finding; its severity rises with how directly untrusted input reaches it and what the sink can do.
- A sink with **no reachable path from any external boundary** (guarded by validation, constant input, internal-only) → **informational, not a finding.** Say why it is unreachable.

**LITMUS: no traced path from an external entry point → it is not a finding.** A scary-looking sink is a candidate, not a verdict — the trace is the verdict.

**Done when:** every candidate has either a traced entry-point → sink path (→ finding) or a stated reason it is unreachable (→ informational).

## 4. Report
Per finding: **vuln class · the traced path (entry point → sink, the files it crosses) · severity set by reachability · the fix direction** in one line. Separate the **findings** (reachable) from the **informational** notes (unreachable) — never inflate the latter into the former. Lead with the highest-severity reachable findings.

**Bottom line.** Close the report with one tagged status line — the single line a reader or an orchestrator folds first: `[security] no security surface` / `[security] no reachable findings · K informational` / `[security] N findings · <highest severity> · K informational`.

**Done when:** every finding carries all four fields; reachable findings and informational notes are separated; and the report ends with the `[security]` bottom-line status.

## When nothing fits
- **No security surface** → diff touches no input/auth/query/path/deserialization/secret surface → `no security surface`, stop (step 1).
- **Candidate but unreachable** → keep it as an **informational** note with the reason it cannot be reached — never promote it to a finding.
- **No reachable findings** → say so plainly, and name the surfaces that were traced and cleared.
