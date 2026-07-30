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
{ "baseUrl": "http://localhost:3000", "start": "<dev-server command>", "auth": "<pointer to creds: env var name / secret-manager key / login route — never the secret>" }
```

- **`baseUrl`** — a running-app hostname (recognisable dev server, `.test` host, container port).
- **`start`** — the command that brings it up, if it is not already running.
- **`auth`** — a **pointer** to how a request authenticates: the env var name, the secret-manager key, or the login route plus which local dev account to use — consumed by step 3. **Never the secret itself** — no tokens, passwords, cookies or PATs in this file. It is a convenience cache, not a credential store; it sits untracked on disk, so a leaked cache would be a leaked credential. If a drive needs a real secret, read it from the environment at run time.

If the app **cannot be made to run** — no dev server, no runnable target — the drive cannot happen. That is not a skip and not a `confirmed`: report `not verified — app not runnable` and mark the review with a **`⚠️ static-only — not runtime-verified`** header, so no reader mistakes a static pass for a runtime one.

**Done when:** the app responds at `baseUrl` (or a registered `verify` skill owns the drive); `.verify-live.json` is written or read; and `git check-ignore .verify-live.json` passes. If the app is not runnable, the `static-only` header is set and the run continues to report `not verified`.

## 3. Get an authenticated session — most real flows need one
An unauthenticated drive reaches a **login form, not the change**. Authentication is therefore **part of the drive, not a blocker to report** — but the credential is always held by the human or the environment, **never by you**. Work down this order and stop at the first that works:

1. **Reuse an authenticated state that already exists.** A saved browser `storageState`, a cookie jar from an earlier drive, a session still live in the automated browser. Free when present — always look first.
2. **Let the project's own harness log in.** An e2e fixture, a login helper, or a seeded-session command already encodes the real login (e.g. a `performLogin` helper that reads its password from an env var). Invoke it and let **it** consume the secret: you pass a **variable name**, never a value. This is the route that works unattended.
3. **Hand the keyboard to the user (interactive only).** Open the login page in the automated browser, fill only the non-secret fields (the username), then **ask the user to type the password and confirm**, and resume from the authenticated page. They keep the secret; you never see it.
4. **Drive an unauthenticated surface instead**, where the change has one — a public route, or an endpoint whose token comes from a sanctioned mint. Report it **per surface** (step 5) so no reader thinks the authenticated flow was covered.

**NEVER obtain a session any other way.** These are refusals, not obstacles to be worked around, and they hold even when the user supplies the secret or authorizes it outright:

- **never type or paste a password** into a login field — including one the user pasted into the chat;
- **never mint or forge a session** — no `Auth::login` script, no hand-written `sessions` row, no self-signed cookie;
- **never read, reset or migrate password / credential columns**;
- **never create an account** to authenticate as.

And **never split a refused call into smaller steps to slip it past a guard** — a blocked credential operation is an answer, not a puzzle.

**In an interactive session, OFFER route 3 before you conclude.** Reporting `not verified — needs credentials` while a human is sitting right there, having never asked them to log in, is the lens quitting one question early. Ask, then drive.

**Done when:** the drive holds an authenticated session via one of routes 1–4 — or, after routes 1, 2 and (if interactive) 3 were genuinely attempted, `not verified — no authenticated session available (<what was tried>)`, which step 5's third litmus accepts as a demonstrated blocker.

## 4. Drive the actual flow
Exercise the flow from step 1 against the running app: hit the endpoint with an authenticated HTTP client, or drive the UI with browser automation. Observe the **real** result — the response body, the rendered output, or the persisted state read back after the action.

**Running the test suite is NOT this.** A green suite never substitutes for driving the flow. Integration tests through the framework's HTTP kernel are still the test suite — they do not touch the running app. If the only thing you can run is tests, the flow was **not driven**: fix the drive (start the app, find the route, get auth via step 3), then drive it.

**Drive the change's own code path, not just the flow.** A running app can execute the flow while a **feature flag, config gate, or unprovisioned dependency routes execution *around* your change** — the old path runs, the drive looks successful, and the fix never executed. Before observing, trace that the change's path is actually reached: the flag is on, the provisioning exists, and the observed behaviour is the **new** behaviour, not a gated-off fallback. Reaching it may need flipping a flag or seeding provisioning — that is **env mutation: do it only with the user's OK, never silently** (a stub config file replaced blindly can break login or other flags).

The OK need not be a live prompt: a **standing, scoped authorization from the caller or context counts** — e.g. an autonomous routine that pre-authorizes, *in a throwaway worktree with its own DB*, turning on the PR's own feature flag and seeding provisioning rows (and migrations/deps), while never authorizing credentials or shared config. That keeps an unattended run honest — a documented OK, not a bypass — instead of forcing every flag-gated change to `not verified` because no human was watching. Absent any authorization (live or standing), the gated path stays `not verified`; and no standing OK ever covers writing credentials or touching shared config. Never assume a fix's path ran because the surrounding flow did.

**Checkout policy (conservative):** a review must **never write source files**.
- **Dirty working tree** → do not check anything out; drive against the PR diff as applied, or note that the flow could not be reached without a checkout.
- **Clean tree** → check out the PR head, drive, then restore the original branch afterwards.

**Done when:** the affected flow was exercised against the running app, the **change's own code path was confirmed reached** (not routed around by an off flag / gate / missing provisioning), and its real response / output / persisted state was observed and captured — or the drive failed and step 5 reports the honest negative.

## 5. Gate the claim on evidence
The wording is decided by the evidence you hold, not by how the run felt. **Match your evidence to a row —
no wording exists outside this table:**

| Evidence you actually hold | Wording |
|---|---|
| Live URL driven · the change's **own** path provably ran · observed artifact (response, screenshot, record read back) | `confirmed ✓` |
| Flow ran, behaved wrong | `diverged — <url>` |
| Flow ran but took the **old** path — off flag, config gate, missing provisioning | `not verified — <fix's path gated off>` |
| Not driven; blocker **observed** — the command you ran and what it returned | `not verified — <blocker>` |
| Not driven; blocker only inferred, recalled, or never actually attempted | `not verified — drive not completed (<what was tried>)` |

**NEVER UPGRADE A ROW.** Audit the claim against its evidence before posting and take the weaker true row
wherever evidence is missing. An overstated ✓ is a false green light on a real change — but `not verified` is
**not a free pass** either: a fabricated blocker is equally a false claim, and it does extra damage by burying a
change that was verifiable. Downgrade the **verdict** when evidence is thin, never the **rigour of the reason**.

**Three guards the table cannot carry:**

1. **Test-runner output is never evidence.** No running-app hostname → no `confirmed`, ever. Integration tests
   through the framework's HTTP kernel never touched the running app.
2. **A blocker the standing OK (§4) or an auth route (§3) already covers is a STEP, not a blocker** — seeding
   provisioning or fixtures, flipping the PR's own flag, migrations. Over-classifying an authorized step as
   out-of-scope is the most common way this lens quits early on a change it could have verified.
3. **A recalled mechanism is a hypothesis.** Confirm it in the tree in front of you or do not name it — a
   confidently-named gate that does not exist is a false claim on someone else's PR, and a negative verdict
   does not make it safe.

**One row per runtime surface, never a blend.** There is **no `partially confirmed`**: a change spanning an
endpoint *contract* and a *core flow* that don't share a fate reports both —
`confirmed ✓ (contract) · not verified (core flow — no creds)`.

**Bottom line.** Close the report with one tagged status line — the single line a reader or an orchestrator folds first: `[verify] confirmed ✓ — <url>` / `[verify] diverged — <url>` / `[verify] not verified — <why>`. When the surface splits, compose the canonical wordings per surface on the one line: `[verify] confirmed ✓ (contract) · not verified (core flow — no creds)`.

**Done when:** the report carries exactly one row per surface; a `confirmed ✓` carries its proof block; any `static-only` header from step 2 is present; and the report ends with the `[verify]` bottom-line.

## When nothing fits
- **No runtime surface** → docs/config/test-only diff → `no runtime surface`, stop before detecting an app (step 1).
- **App not runnable** → `not verified — app not runnable` + the `⚠️ static-only` header; never `confirmed`.
- **Only the test suite runs** → the flow was not driven → fix the drive, or report `not verified — <what blocked the drive>`. A green suite is not evidence.
- **Change gated off** → the app runs and the flow is drivable, but the fix's path sits behind an off feature flag / config gate / missing provisioning → `not verified — <fix's path gated off>`. Reaching it needs flipping the flag or seeding provisioning — **env mutation, only with the user's OK**, never silently. Where a live or standing OK **does** cover the flip, reaching the path is part of the drive: `not verified — gated off` is correct only when no authorization covers it.
- **Flow needs a login** → not a blocker: work step 3's routes (reuse saved state → let the project's harness log in → ask the user to type it). Only after those are actually attempted is `not verified — no authenticated session available` true. Never type a password, forge a session, or read credential columns to get there.
- **A blocker you cannot demonstrate** → do not name it. Either show the command and its output, or report `not verified — drive not completed (<what you actually tried>)` and let the gap read as a gap rather than as a fact about the author's setup.
