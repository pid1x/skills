Quickstart:

```bash
npx skills add pid1x/skills --skill=deep-review
```

```bash
npx skills update deep-review
```

[Source](https://github.com/pid1x/skills/tree/main/skills/engineering/deep-review)

## What it does

`deep-review` orchestrates a full review of **one change** — a branch, PR, or ticket — by running five lenses and folding them into a single write-up shaped the way a PR already expects. The lenses are a general **engine review** (bug / quality / performance / concurrency) plus the four specialised ones: **[spec]**, **[security]**, **[mutation]**, **[verify]**.

It orchestrates; it does not re-implement. Each lens owns its own rules — `deep-review` invokes `spec-check`, `security-check`, `mutation-check`, `verify-live`, and a delegated engine review, then folds their tagged bottom-lines into one report. That keeps a single source of truth per lens and no drift.

## When to reach for it

Type `/deep-review` with a PR, branch, or ticket. It is model-invoked — you can run it by hand, and a review routine or agent can delegate its per-PR review to it — but it triggers only on a deliberate whole-PR review across all axes, not on a casual "review" mention; for a single axis, reach for that lens skill directly. It is **dry-run by default** (writes a PR-ready review into the chat, posts nothing); add **`post`** to write the review to the PR as a comment-only review, in the same shape the automated review loop posts — a COMMENT-type review plus a runtime follow-up comment, with delta-dedupe against a prior own review. It never approves, requests changes, or merges.

Reach for it when you want the whole review in one pass rather than running the lenses by hand. The specialised lenses give targeted depth (evidence-gated runtime checks, reachability-scoped security, independent spec severity); the engine review covers the general quality ground — performance and concurrency bugs — the specialised four structurally miss.

## Two disciplines keep the fold honest

- **No silent skips.** Every one of the five lenses carries an explicit status, even when that status is a degraded reason (`no spec to check against`, `app not runnable`). A lens that can't run says so; it never vanishes.
- **No dressed-up non-runs.** Before composing, every claim is audited against its evidence and downgraded to the weaker true claim where it outruns the proof (`confirmed ✓` → `not verified`, a real score → `no result`). An overstated ✓ is a false green light on a real PR.

## It's working if

- All five lenses run or degrade honestly, each with an explicit status; capability is detected first and cached in a git-ignored `.deep-review.json`.
- Cross-lens links are surfaced — a runtime lens confirming or contradicting a static finding makes the report stronger, not just longer.
- **A re-review is a delta, not a re-issue.** The main comment runs fixed-header → one-clause verdict → open findings only, each with its inline `→ ask:` → asks owned by someone else → closed findings as **one line of keys and ticks** → a single trailing lens status line; the runtime follow-up carries only its own findings and proof block. Closed work is never re-explained (the PR is the journal), 🟢 "this is correct" entries are never posted, and a lens gets body text only when it has a finding — the status line already proves it ran.
- **The budget is per round, not per comment** — ≈2,500 characters for a re-review, ≈4,000 for the first — because two comments that each pass a per-comment cap still land twice the text on the author. Over budget means cut, never append.
- It never writes source files — the one sanctioned exception is `mutation-check`'s hand-applied pass, whose edits are transient and unconditionally reverted — never approves or blocks a merge, and rides a `⚠️ static-only` header whenever the app couldn't be driven.
- By default it posts nothing (dry-run); with `post` it writes a COMMENT-type review plus a runtime follow-up comment, confirms the target PR before sending, passes the body by file (never inline, so a large markdown body can't post as an empty `~`), and re-reads the post to confirm a non-empty body landed.
- On a delta re-review it re-posts when there are new commits or its verdict changed (a stale-verdict correction), and carries a lens result forward only when the head SHA is byte-identical — a moved SHA, even a merge, re-runs the runtime lenses.
- Its posts are headed `🤖 deep-review` — a distinct engine tag, so its dedupe matches only its own prior posts and never "corrects" another review engine's post on the same PR.

## Where it fits

`deep-review` is the orchestrator over the four base lenses and a delegated engine review. It is the manual front door to the whole review suite — dry-run by default, opt-in `post` to write the review onto the PR; the individual lenses (`spec-check`, `security-check`, `mutation-check`, `verify-live`) remain usable on their own when you want just one axis.
