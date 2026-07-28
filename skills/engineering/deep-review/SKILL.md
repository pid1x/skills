---
name: deep-review
description: Orchestrate a full multi-lens review of one change (PR / branch / ticket) — a general engine review plus the spec, security, mutation, and verify lenses, folded into one loop-style write-up; dry-run by default, opt-in `post` writes a comment-only PR review. Use when the user wants a whole-PR review across all axes at once (a deep or full review of a PR), or when a review routine delegates its per-PR review to this engine; for a single axis, use that lens skill (spec-check / security-check / mutation-check / verify-live) directly.
argument-hint: "<PR url/number | ticket key | branch> [post]  — omit target for the working diff; add post to write the review to the PR (dry-run by default)"
---

A real review is more than one pass. `deep-review` runs five lenses over **one change** — a general **engine review** plus **[spec]**, **[security]**, **[mutation]**, **[verify]** — and folds them into a single write-up in the shape a PR already expects. It **orchestrates; the lenses own their rules** — this skill never restates them, so there is one source of truth per lens and no drift. Two disciplines hold the fold honest: **no silent skips** (every lens gets an explicit status) and **no dressed-up non-runs** (every claim is downgraded to what its evidence supports before it ships).

## 1. Resolve the target + detect what can run
Resolve the target → its changed files (path / branch / PR / ticket, as the lenses do). Then detect which lenses this repo can run, and **degrade honestly** — a lens that can't run says so in one line; it never vanishes.

Capability map, cached in `.deep-review.json` (git-ignored like the lenses' own caches — if `git check-ignore` fails, add it to `.git/info/exclude`; record capability *kinds* only, no ticket keys or hostnames):

| Lens | Can-run check | If it can't |
|---|---|---|
| engine review | a review skill/plugin installed (e.g. `code-review`)? | fall back to built-in review instructions |
| [spec] | a tracker reachable, or a PR / pasted spec? | `[spec] no spec to check against` |
| [security] | always (static) | — |
| [mutation] | `mutation-check` reachable + a mutation tool detected? | `[mutation] no result — no tool` + one-line install hint |
| [verify] | app runnable (a `.verify-live.json` or a recognisable dev server)? | `[verify] not verified — app not runnable` + a `⚠️ static-only` header on the review |

**Done when:** the changed files are listed; each of the five lenses is marked runnable or degraded-with-reason; `.deep-review.json` is written or read and `git check-ignore` passes.

## 2. Checkout policy — never write source
The conservative rule the lenses share: **dirty tree** → review from the diff, don't check out; **clean tree** → check out the PR head, restore the original branch afterwards. A review **never writes source files**. Runtime lenses ([mutation]/[verify]) need the app stood up; static lenses (engine/[spec]/[security]) don't.

**Done when:** the change is present to review (checked-out head or working diff), and the runtime lenses have a stood-up app or a recorded reason they don't.

## 3. Run the five lenses
Invoke each runnable lens — **do not restate its rules, the lens owns them:**

- **engine review** (delegated skill, else built-in) — general bug / quality / **performance / concurrency** over the diff. This is the lens that catches what the four specialised ones structurally miss.
- **spec-check · security-check · mutation-check · verify-live** — each returns its `[lens]` bottom-line plus detail.

Static lenses (engine/[spec]/[security]) may run first; runtime lenses ([mutation]/[verify]) run against the stood-up app. Every lens completes on its own bottom-line — **that line is the fold surface.**

**Delta re-review — carry forward only on an identical head SHA.** Re-reviewing a PR you already reviewed, you may carry a lens's prior result forward instead of re-running it — labelled `carried forward from <SHA>` — **only when the head SHA is byte-identical** to that prior run. If the SHA moved at all — new commits *or a main-merge into the branch* — **re-run the runtime lenses** ([mutation]/[verify]): a merge changes the runtime even when the PR's own diff didn't, so a prior verification no longer covers this head. Carrying a result forward across a moved SHA is a dressed-up non-run.

**Done when:** every runnable lens has produced its `[lens]` bottom-line + findings (or a `carried forward from <SHA>` label valid only for an identical SHA); every degraded lens has its honest status line.

## 4. Cross-lens audit — before you compose
Two passes over the collected lens outputs, both mandatory:

- **Corroborate.** A runtime lens often confirms or contradicts a static one — `verify-live` reading a persisted value that pins a `mutation` survivor, or diverging exactly where `spec` predicted a gap. Surface these links: a finding two lenses agree on is stronger; one they contradict must be resolved before it ships.
- **Audit every claim against its evidence.** Where a claim outruns its evidence, **downgrade it to the weaker true claim** (`confirmed ✓` → `not verified`, a real score → `no result`). An honest gap is actionable; an overstated ✓ is a false green light on a real PR. And **no silent skips** — every one of the five lenses carries an explicit status, even when that status is a degraded reason.

**Done when:** cross-lens corroborations and contradictions are named; every posted claim is backed by this-run evidence or downgraded; all five lenses have an explicit status.

## 5. Compose — the loop-style write-up
Write the review in the shape a PR already expects, so it reads the same whoever produced it. **Two comments, never blurred:**

**Main review comment** — led by the header `🤖 deep-review · <lenses that ran>` (the same two-comment shape the automated review loop posts; the `🤖 deep-review` tag names this engine distinctly so a reader can tell it from a loop review, and so the dedupe below matches only its **own** prior posts, never another engine's), then:
- **My read:** `approvable` / `minor changes` / `blocking concern` — deep-review's own call, folding the lens severities.
- **Prior findings** (delta re-review only) — per prior finding, `addressed ✓` / `still open`; never re-raise a resolved thread.
- **Engine review** — the general findings, numbered.
- **[security]** and **[spec]** — folded in with their inline tags and bottom-lines.

**Runtime follow-up comment** — `[mutation]` and `[verify]` with their proof blocks (they finish later than the static pass, so they can't fold into the main comment).

Each lens keeps its `[lens]` bottom-line. If [verify] couldn't run, the `⚠️ static-only — not runtime-verified` header rides on the main comment.

**Done when:** the write-up carries both comments in loop shape; every lens's bottom-line is present; the `My read` verdict leads the main comment.

## 6. Deliver — dry-run by default, `post` writes to the PR
**Dry-run (default, no `post` argument):** report the composed write-up into the chat, PR-ready. Nothing is sent. This is the safe default — posting is outward-facing and hard to undo.

**`post`:** send the review to the PR, in the automated loop's shape. **Confirm the target PR number before sending.**

**Pass the body by FILE, never inline.** A review body is large markdown — backticks, `$`, quotes, code fences. Passed inline (`gh … -f body="…"` / `--body "…"`) it breaks shell quoting and posts an **empty `~` body** — a silent, content-less review. Always write the body to a temp file and pass it by reference:
- The **main review** goes up as a GitHub **COMMENT-type review** — `gh api repos/{owner}/{repo}/pulls/{n}/reviews --input <payload.json>` (a JSON file with `"event":"COMMENT"` and the body), never `-f body="…"`. Its body is led by the `🤖 deep-review · <lenses>` header (the dedupe key, inside the review, not a separate comment). Anchor findings that carry a `file:line` as inline review comments where the position resolves; fold the rest into the summary body.
- The **runtime follow-up** ([mutation]/[verify], with proof blocks) goes up as a **separate** PR comment — `gh pr comment <pr> --body-file <file>` (again by file), because it finishes after the static pass, the same reason the loop splits the two.
- **Comment-only, always.** Never `event=APPROVE` or `REQUEST_CHANGES`, never merge. Severity is deep-review's own call — a confirmed correctness or data-integrity regression is `blocking concern` regardless of author framing — but the merge verdict stays with the human.

**Verify the post landed.** After sending, re-read the review/comment on the PR and confirm the body is **non-empty and complete** — a silent empty (`~`) post is worse than no post, and nobody sees it fail. If the body is empty or truncated, the send failed: re-post from the file, or report the failure plainly; never leave an empty review standing.

**Delta re-review** (a prior own `🤖 deep-review` review already on the PR — match on the `🤖 deep-review` tag, **not** a bare `🤖 skill-review`, so another engine's review on the same PR is never mistaken for your own and "corrected"): dedupe on it. Re-post when **either** there are new commits since it **or** this run's verdict differs from the prior review's. **No new commits *and* the same verdict** → don't re-post; say the prior review still stands. When re-posting a delta: per prior finding `addressed ✓` / `still open`, plus new issues; never re-raise a resolved or acknowledged thread. **A verdict change with no new commits is a correction, not a duplicate** — the author is otherwise sitting on a stale verdict (e.g. still reads `blocking` after the blocker was withdrawn); the no-new-commits rule does not cover it, so post it.

**Own PR:** GitHub blocks a review on your own PR — fall back to a single plain `gh pr comment` carrying the write-up, or report to chat if even that isn't wanted.

**Done when:** dry-run → the write-up is in the chat; `post` → the COMMENT review + runtime follow-up are on the PR **with their bodies confirmed non-empty** (or the delta / own-PR / no-new-commits path is taken with its reason stated), and nothing was ever approved, requested-changes, or merged.

## When nothing fits
- **No runnable lens** — a docs-only diff (no spec, no runtime surface, no security surface, nothing to mutate) → say so plainly; there is nothing to review.
- **Only static lenses run** (app not runnable) → the review ships with the `⚠️ static-only` header so no reader mistakes it for runtime-verified.
- **A lens errors** → its status line carries the error; the other four still compose. One lens failing never sinks the review.
