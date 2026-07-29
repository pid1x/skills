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

## The evidence gate

The leading idea is the **evidence gate**: a claim of success must carry evidence only the real drive could produce. `confirmed ✓` is postable only with a proof block from that run — the live URL(s) driven, the driver used, and the observed artifact.

**Litmus:** if the evidence contains no running-app hostname, it was not verify — it cannot be `confirmed`. Test-runner output is never verify evidence. **Second litmus:** if the change's own code path did not execute — because the fix sits behind an off feature flag, a config gate, or missing provisioning, so the app ran the *old* path — it is `not verified — <fix's path gated off>`, never `confirmed`; driving the flow is necessary but not sufficient. The wording is never upgraded: without a proof block, the only truthful postings are `diverged` or `not verified`. An honest gap is actionable; an overstated ✓ is a false green light.

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
