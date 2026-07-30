Quickstart:

```bash
npx skills add pid1x/skills --skill=verify-live
```

```bash
npx skills update verify-live
```

[Source](https://github.com/pid1x/skills/tree/main/skills/engineering/verify-live)

## What it does

`verify-live` runtime-verifies **one change** — a branch, PR, ticket, or path — by driving the flow the change affects against the **running app** and observing what actually happens. A green test suite proves the tests pass; `verify-live` proves the feature works.

It reports exactly one evidence-gated status: `confirmed ✓` (with a proof block), `diverged — <url>`, or `not verified — <why>`. On first run it detects how to reach the app and caches it in a local `.verify-live.json` (ignored via `.git/info/exclude` by default — never committed). That cache holds only **pointers** to credentials — an env var name, a secret-manager key, a login route — never a secret itself: it is a convenience cache, not a credential store.

## When to reach for it

Type `/verify-live`, pass a branch/PR/ticket/path as an argument, or let the agent reach for it when a change has a runtime surface and you want proof it works — not just a passing suite.

Reach for it after a change that a user or caller would exercise at runtime: an endpoint, a screen, a job. Skip it for docs-, config-, or test-only diffs — there is nothing to drive.

## Logging in is part of the drive

An unauthenticated drive reaches a login form, not the change — so getting a session is a **step, not a blocker**, while the credential stays with the human or the environment and never with the agent. It works down four routes and stops at the first that lands: reuse an authenticated state that already exists (saved `storageState`, cookie jar, live browser session) → let the project's own e2e fixture or login helper log in, consuming its secret from an env var the agent names but never reads → **ask the user to type the password** in the automated browser and resume from the authenticated page → or drive an unauthenticated surface and report it per-surface.

Four things it will never do, even when handed the secret or told to: type or paste a password, mint or forge a session (`Auth::login` scripts, hand-written `sessions` rows, self-signed cookies), read or reset credential columns, or create an account to log in as. A refused credential operation is an answer, not something to split into smaller calls. And in an interactive session it **offers the password handoff before concluding** — reporting `not verified — needs credentials` while a human is sitting there, never having asked, is quitting one question early.

## The evidence gate

The leading idea is the **evidence gate**: a claim of success must carry evidence only the real drive could produce. `confirmed ✓` is postable only with a proof block from that run — the live URL(s) driven, the driver used, and the observed artifact.

The wordings are a **decision table** — evidence in, wording out, nothing outside it. `confirmed ✓` needs a live URL, proof the change's *own* path ran, and an observed artifact. A flow that ran the *old* path (off flag, config gate, missing provisioning) is `not verified — <fix's path gated off>`, never `confirmed`. A `not verified` whose blocker was only inferred or recalled — rather than observed, with the command and what it returned — degrades to `not verified — drive not completed`, because the reason is itself a published claim about the author's environment.

Three guards sit alongside the table: test-runner output is never evidence (no running-app hostname, no `confirmed`); a blocker the standing OK or an auth route already covers is a **step**, not a blocker; and a recalled mechanism is a hypothesis until confirmed in the tree in front of you. Rows are never upgraded — an overstated ✓ is a false green light, and `not verified` is not a free pass either, since a fabricated blocker is equally false and additionally buries a change that was verifiable. There is no `partially confirmed`: a split surface reports one row per surface.

## It's working if

- It drives the actual flow against the running app, not the test suite.
- A `confirmed ✓` always carries a proof block with a running-app hostname; a run that could only execute tests reports `not verified`, never `confirmed`.
- It confirms the change's own code path actually ran — a flow gated off by an off feature flag or missing provisioning reports `not verified`, never `confirmed`, and reaching a gated path needs the user's OK (it never mutates the env silently). That OK can be a standing, scoped pre-authorization from the caller — an autonomous routine may pre-authorize flag-on and provisioning in a throwaway worktree (never credentials or shared config) — so unattended runs stay honest instead of stalling on `not verified`.
- When a change spans more than one runtime surface, it reports one canonical wording per surface (`confirmed ✓ (contract) · not verified (core flow — no creds)`) rather than a blended `partially confirmed` that hides which half is real.
- A diff with no runtime surface is declared `no runtime surface` and stops before touching the app.
- It never writes source files — dirty tree means review from the diff; clean tree means check out the PR head and restore afterwards.
- Every run ends on a single tagged `[verify]` bottom-line, so its verdict folds into a fuller review without re-reading the report.

## Where it fits

`verify-live` is one lens of a fuller review. It pairs with `mutation-check` (does the suite catch regressions?) and a general review engine (is the diff sound?); `verify-live` answers the separate question a static pass cannot — does the change actually behave correctly against a running app? Where a project ships its own `verify` skill with real login and routes, `verify-live` delegates the drive to it.
