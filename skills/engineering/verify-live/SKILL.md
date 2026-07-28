---
name: verify-live
description: Runtime-verify a change by driving the real flow against the running app and observing actual behaviour — never the test suite. Reports one evidence-gated status: confirmed, diverged, or not verified. Use when a PR/branch/diff has a runtime surface and you need proof it works, not a green suite.
argument-hint: "<branch | PR url/number | ticket key | path>  — omit for the current working diff"
---

A green suite proves the tests pass, not that the feature works. `verify-live` **drives the real flow** — hits the endpoint or drives the UI against the **running app** — and reports what it actually observed. The whole skill hangs on one rule: a claim of success must carry **evidence only the real drive could produce**. That makes honesty the cheapest path, and an honest negative a legitimate outcome — not a failure to paper over.

## 1. Scope to the runtime surface
Resolve the target → its changed files, then decide whether there is anything to drive at all.

- **Path / glob** → use as-is.
- **Branch** (or no argument) → `git fetch origin <base>`, then `git diff --name-only origin/<base>...HEAD` (`base` = `main`/`master`).
- **PR** (url or number) → `gh pr diff <pr> --name-only`.
- **Ticket key** → resolve its PR (`gh pr list --search "<KEY> in:title"`), then treat as a PR.

Name the **flow(s)** the change affects — the endpoint, route, screen, or job a user or caller would exercise. A diff with **genuinely no runtime surface** — docs, config, or pure-test-only — is the **one valid skip**: report `no runtime surface` and stop. Everything else has a flow to drive.

**Done when:** either the affected flow(s) are named, or the diff is declared `no runtime surface` with the reason (docs / config / test-only).

## 2. Detect how to run the app — once per project, then remember
`.verify-live.json` is a **local per-project cache** — not committed, and it **must be ignored by git**. First run in a repo: no file exists → detect, write it, and ensure git ignores it — if `git check-ignore .verify-live.json` fails, add it to `.git/info/exclude` (local-only, never touches tracked files); only edit the repo's `.gitignore` if the user asks. Later runs read it and skip detection. `--reconfigure` redoes detection.

**If a `verify` skill is registered in this session, delegate the drive to it** — it holds the project's real login, routes, and seeding. Its availability varies per session; when it is absent, drive directly using the fields below. A missing `verify` skill is NEVER a skip reason.

Detect and record how to reach the app:

```json
{ "baseUrl": "http://localhost:3000", "start": "<dev-server command>", "auth": "<how to authenticate>" }
```

- **`baseUrl`** — a running-app hostname (recognisable dev server, `.test` host, container port).
- **`start`** — the command that brings it up, if it is not already running.
- **`auth`** — how a request authenticates (login creds path, token, session).

If the app **cannot be made to run** — no dev server, no runnable target — the drive cannot happen. That is not a skip and not a `confirmed`: report `not verified — app not runnable` and mark the review with a **`⚠️ static-only — not runtime-verified`** header, so no reader mistakes a static pass for a runtime one.

**Done when:** the app responds at `baseUrl` (or a registered `verify` skill owns the drive); `.verify-live.json` is written or read; and `git check-ignore .verify-live.json` passes. If the app is not runnable, the `static-only` header is set and the run continues to report `not verified`.

## 3. Drive the actual flow
Exercise the flow from step 1 against the running app: hit the endpoint with an authenticated HTTP client, or drive the UI with browser automation. Observe the **real** result — the response body, the rendered output, or the persisted state read back after the action.

**Running the test suite is NOT this.** A green suite never substitutes for driving the flow. Integration tests through the framework's HTTP kernel are still the test suite — they do not touch the running app. If the only thing you can run is tests, the flow was **not driven**: fix the drive (start the app, find the route, get auth), then drive it.

**Drive the change's own code path, not just the flow.** A running app can execute the flow while a **feature flag, config gate, or unprovisioned dependency routes execution *around* your change** — the old path runs, the drive looks successful, and the fix never executed. Before observing, trace that the change's path is actually reached: the flag is on, the provisioning exists, and the observed behaviour is the **new** behaviour, not a gated-off fallback. Reaching it may need flipping a flag or seeding provisioning — that is **env mutation: do it only with the user's OK, never silently** (a stub config file replaced blindly can break login or other flags).

The OK need not be a live prompt: a **standing, scoped authorization from the caller or context counts** — e.g. an autonomous routine that pre-authorizes, *in a throwaway worktree with its own DB*, turning on the PR's own feature flag and seeding provisioning rows (and migrations/deps), while never authorizing credentials or shared config. That keeps an unattended run honest — a documented OK, not a bypass — instead of forcing every flag-gated change to `not verified` because no human was watching. Absent any authorization (live or standing), the gated path stays `not verified`; and no standing OK ever covers writing credentials or touching shared config. Never assume a fix's path ran because the surrounding flow did.

**Checkout policy (conservative):** a review must **never write source files**.
- **Dirty working tree** → do not check anything out; drive against the PR diff as applied, or note that the flow could not be reached without a checkout.
- **Clean tree** → check out the PR head, drive, then restore the original branch afterwards.

**Done when:** the affected flow was exercised against the running app, the **change's own code path was confirmed reached** (not routed around by an off flag / gate / missing provisioning), and its real response / output / persisted state was observed and captured — or the drive failed and step 4 reports the honest negative.

## 4. Gate the claim on evidence
The claim you post is decided by the evidence you hold, not by how the run felt. **Exactly three wordings are allowed:**

- **`confirmed ✓`** — the flow behaves as the change intends. Postable **ONLY** with a **proof block from THIS run**: the exact live URL(s) driven, the driver used (browser automation / authenticated HTTP client), and the observed artifact (response snippet, screenshot, or record read back after the action).
- **`diverged — <url>`** — the flow ran but behaved wrong. Name the live URL and what you saw versus expected.
- **`not verified — <why + what was tried>`** — the flow could not be driven. State why and what you attempted.

**LITMUS: if the evidence contains no running-app hostname, it was not verify — it cannot be `confirmed`.** PHPUnit/Pest/Jest output is never verify evidence.

**SECOND LITMUS: if the change's own code path did not execute, it was not verified.** A flow that ran the **old** path — because the fix sits behind an off feature flag, a config gate, or missing provisioning — is `not verified — <the fix's path is gated off>`, never `confirmed`. Driving the flow is necessary but not sufficient; the change's path must have actually run.

**More than one runtime surface → one wording per surface, never a blend.** When a change spans distinct surfaces that don't share a fate — e.g. an endpoint *contract* confirmed live but the *core flow* unreachable without credentials — report **each surface with its own canonical wording**: `confirmed ✓ (contract) · not verified (core flow — no creds)`. There is **no `partially confirmed`** — the three wordings compose per surface; a blended wording hides which half is real.

**NEVER UPGRADE THE WORDING.** Without a proof block, `confirmed ✓` is unavailable — the only truthful postings are `diverged` or `not verified`. Before reporting, audit the claim against its evidence; where evidence is missing, downgrade to the weaker true claim. An honest gap is actionable; an overstated ✓ is a false green light on a real change — strictly worse than a `not verified`.

**Bottom line.** Close the report with one tagged status line — the single line a reader or an orchestrator folds first: `[verify] confirmed ✓ — <url>` / `[verify] diverged — <url>` / `[verify] not verified — <why>`. When the surface splits, compose the canonical wordings per surface on the one line: `[verify] confirmed ✓ (contract) · not verified (core flow — no creds)`.

**Done when:** the report carries exactly one of the three wordings; a `confirmed ✓` is accompanied by its proof block containing a running-app hostname **and evidence the change's own path ran**; any `static-only` header from step 2 is present; and the report ends with the `[verify]` bottom-line status.

## When nothing fits
- **No runtime surface** → docs/config/test-only diff → `no runtime surface`, stop before detecting an app (step 1).
- **App not runnable** → `not verified — app not runnable` + the `⚠️ static-only` header; never `confirmed`.
- **Only the test suite runs** → the flow was not driven → fix the drive, or report `not verified — <what blocked the drive>`. A green suite is not evidence.
- **Change gated off** → the app runs and the flow is drivable, but the fix's path sits behind an off feature flag / config gate / missing provisioning → `not verified — <fix's path gated off>`. Reaching it needs flipping the flag or seeding provisioning — **env mutation, only with the user's OK**, never silently.
