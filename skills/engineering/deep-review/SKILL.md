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
The conservative rule the lenses share: **dirty tree** → review from the diff, don't check out; **clean tree** → check out the PR head, restore the original branch afterwards. A review **never writes source files** — with exactly one sanctioned exception: `mutation-check`'s hand-applied fallback makes **transient** edits to already-checked-out source, each reverted unconditionally (including when its watchdog aborts mid-mutant) with the tree verified clean afterwards; see that skill's revert guarantee. Nothing else writes source. Runtime lenses ([mutation]/[verify]) need the app stood up; static lenses (engine/[spec]/[security]) don't.

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

## 5. Compose — delta-first
**On a re-review, FIRST check whether you may post at all — before composing a word.** Compare this head SHA
against the prior `🤖 deep-review` post's. **Same SHA**, and this run reaches the **same verdict**, and it
carries **no finding the prior review didn't already have** → **do not post.** Say the prior review still
stands, and stop. This gate also exists at delivery (§6), but by then the text is written, and "don't post"
starts to read as discarding work — so it belongs here, ahead of the writing.

**"I now have better evidence" is NOT a trigger.** Proving a standing finding harder is not a new round; it is
the previous round restated at length. If the original finding was under-evidenced, that was a fault in the
original — and the remedy is a short reply in its existing thread, not a fresh review.

**A re-review is a DELTA, not a re-issue.** The PR is the journal — never re-write it. A closed finding was
already explained in the round that found it, and that text is still on the page one scroll up; restating it
spends the whole budget telling the author what they already did.

**Main review comment**, in this order — only step 3 gets body text:

1. **Header** — `🤖 deep-review · r<N> · <lenses that ran>`. The tag is **FIXED — never rename it between rounds.** The dedupe matches on this exact string, so a renamed header stops it recognising its own prior posts and the next round re-issues a full review instead of a delta (and a reader can no longer tell one engine's thread from another's).
2. **Verdict** — `approvable` / `minor changes` / `blocking concern`, plus **one clause** of why. Not a paragraph.
3. **Open** — each finding still live, with its severity and its `→ ask:` on the following line.
4. **Not the author** — asks owned by someone else (file the follow-up, a SEC ticket, a product decision), one line each, owner named.
5. **Closed since r\<N-1>** — **one line, keys and ticks only**: `1 release guard ✓ · 2 reachability ✓ · 5 mutation survivors ✓ · 3–4 declined, still correct`. No table, no `file:line`, no re-explanation.
6. **Lens status line** — the `[lens]` bottom-lines folded onto one trailing line.

**Runtime follow-up comment** — `[mutation]` and `[verify]` land later than the static pass, so they cannot
fold into the main comment. It carries **only its own findings and proof block**: what the drive showed, the
URL, the survivor list. It is not a second review — no verdict restatement, no prior-findings recap.

If [verify] couldn't run, the `⚠️ static-only — not runtime-verified` header rides on the main comment.

### Every finding carries its ask
A review that never approves and never blocks has exactly one lever: **how clearly it asks.** Findings are
diagnoses; an author needs the translation into work. So **every open finding carries its ask on the following
line** (`→ ask: <verb the author can act on>`), and asks that are **not the author's** get their own short
block with the owner named.

Severity still sets the register — **Must** (correctness, data integrity, a reachable security path; name the
`file:line` to change) · **Should** (worth doing now, not release-blocking) · **Decide** (no code change: a
judgement only a human can make, and never signed off by the author alone) · **FYI** (deliberately no action —
unreachable sinks, pre-existing patterns — one line, so the author knows they may skip it).

The consolidated trailing `## Asks` block is **gone**: with each ask inline and the owner-asks in their own
block, re-listing them at the end was the same work stated twice, and it was the single largest duplicated
section. The **lens status line may now be last** precisely because every ask already appears above it — a
trailing statistic is acceptable only when the reader has nothing left to derive from it. An all-clear review
still says so in words (`approvable — nothing open`), never by falling silent.

### Keep it short enough to be read
Length is not thoroughness. A wall of text gets skimmed, and a skimmed review changes nothing — so the budget
is part of the format, not a nicety. **Budget the ROUND, not the comment:** two comments that each pass a
per-comment cap still land twice the text on the author.

| | Budget |
|---|---|
| **A whole round** (main + runtime follow-up) | **≈2,500 characters** for a re-review · **≈4,000** for the first review, where everything is genuinely news |
| One finding | **3 sentences / ≈400 characters**: the claim · the evidence (`file:line`) · the consequence |
| An `FYI` / informational item | **one line** |
| The closed-findings recap | **one line, for all of them together** |

**Measure the round before posting, and if it is over, COMPRESS — never drop a finding, and never grant
yourself an exception.** *"Cut, or lose a real finding"* is a false choice, and it is the excuse behind every
overrun: a 3,400-character finding compresses to 400 without losing its claim, its `file:line` or its
consequence. Compress, measure again, then post — an overrun you noticed and shipped anyway is a decision, not
an accident.

What to cut, in this order; all of it is process, not finding:

- **Closed findings, beyond the one-line key.** The most expensive habit by far and the first thing to go.
- **🟢 "this is correct" entries — never post one.** Reassurance is not a finding. A 🟢 carrying a nit becomes an **ask**; a 🟢 carrying nothing is deleted. A claim that needed checking and checked out is one clause of the verdict, or it is already what the lens bottom-line says.
- **The same problem described more than once.** One finding, one description. A blocker does not need a paragraph in the verdict *and* a numbered finding *and* a `[spec]` bullet *and* an ask — it needs the finding and its ask.
- **Per-lens sections with no finding in them.** `### [security] — no new risk surface` followed by three clearance bullets is the lens proving it ran; the status line already does that. A lens earns body text only when it has a finding.
- **How you found it.** The `file:line` *is* the evidence; the trace that led there is not. Never narrate the investigation ("I then checked X, which showed Y, so I looked at Z").
- **Restating the author's own words.** Do not quote the PR description or re-explain the ticket back to them — they wrote it. Quote a spec line only where the divergence turns on its exact wording.
- **Re-deriving what the change obviously does.** Findings are about what is *wrong or missing*.

_(Reference point, measured on one real PR: six rounds ran **~45,200 characters** across 11 comments — round 6
alone was 10,240 (4,942 static + 5,298 runtime), against a then-budget of 3,500 for the main comment that every
round overran while the follow-up went unbudgeted. The same round in delta shape is ~1,150 with nothing
actionable lost. For scale, the human reviewer on that PR approved in 243 characters, and the author's replies
ran 1,111–2,125.)_

**Done when:** the main comment carries the fixed header, the one-clause verdict, the open findings, the owner-asks, the one-line closed recap and the trailing lens status line, in that order; every lens's bottom-line is present; **every open finding has its `→ ask:`**; no 🟢 entry and no findingless lens section was posted; and the round is inside its character budget — measured across **both** comments, not each.

## 6. Deliver — dry-run by default, `post` writes to the PR
**Dry-run (default, no `post` argument):** report the composed write-up into the chat, PR-ready. Nothing is sent. This is the safe default — posting is outward-facing and hard to undo.

**`post`:** send the review to the PR, in the automated loop's shape. **Confirm the target PR number before sending.**

**Pass the body by FILE, never inline.** A review body is large markdown — backticks, `$`, quotes, code fences. Passed inline (`gh … -f body="…"` / `--body "…"`) it breaks shell quoting and posts an **empty `~` body** — a silent, content-less review. Always write the body to a temp file and pass it by reference:
- The **main review** goes up as a GitHub **COMMENT-type review** — `gh api repos/{owner}/{repo}/pulls/{n}/reviews --input <payload.json>` (a JSON file with `"event":"COMMENT"` and the body), never `-f body="…"`. Its body is led by the `🤖 deep-review · <lenses>` header (the dedupe key, inside the review, not a separate comment). Anchor findings that carry a `file:line` as inline review comments where the position resolves; fold the rest into the summary body.
- The **runtime follow-up** ([mutation]/[verify], with proof blocks) goes up as a **separate** PR comment — `gh pr comment <pr> --body-file <file>` (again by file), because it finishes after the static pass, the same reason the loop splits the two.
- **Comment-only, always.** Never `event=APPROVE` or `REQUEST_CHANGES`, never merge. Severity is deep-review's own call — a confirmed correctness or data-integrity regression is `blocking concern` regardless of author framing — but the merge verdict stays with the human.

**Settle the findings BEFORE posting — never post an addendum essay.** A review is one main comment plus its
runtime follow-up; a third post correcting the first ("I understated the scope") means the round shipped before
its own severities were settled. Fix the severity and the scope while the text is still a draft. If something
genuinely has to be corrected after the fact, it is **one line** in the existing thread, never a new section.

**Verify the post landed.** After sending, re-read the review/comment on the PR and confirm the body is **non-empty and complete** — a silent empty (`~`) post is worse than no post, and nobody sees it fail. If the body is empty or truncated, the send failed: re-post from the file, or report the failure plainly; never leave an empty review standing.

**Delta re-review** (a prior own `🤖 deep-review` review already on the PR — match on the `🤖 deep-review` tag, **not** a bare `🤖 skill-review`, so another engine's review on the same PR is never mistaken for your own and "corrected"): dedupe on it. Re-post when **either** there are new commits since it **or** this run's verdict differs from the prior review's. **No new commits *and* the same verdict** → don't re-post; say the prior review still stands — and **"I now have better evidence" is not a third trigger** (see §5's gate: extra proof for a standing finding is a reply in its thread, never a fresh review). When re-posting a delta: per prior finding `addressed ✓` / `still open`, plus new issues; never re-raise a resolved or acknowledged thread. **A verdict change with no new commits is a correction, not a duplicate** — the author is otherwise sitting on a stale verdict (e.g. still reads `blocking` after the blocker was withdrawn); the no-new-commits rule does not cover it, so post it.

**Own PR:** GitHub blocks a review on your own PR — fall back to a single plain `gh pr comment` carrying the write-up, or report to chat if even that isn't wanted.

**Done when:** dry-run → the write-up is in the chat; `post` → the COMMENT review + runtime follow-up are on the PR **with their bodies confirmed non-empty** (or the delta / own-PR / no-new-commits path is taken with its reason stated), and nothing was ever approved, requested-changes, or merged.

## When nothing fits
- **No runnable lens** — a docs-only diff (no spec, no runtime surface, no security surface, nothing to mutate) → say so plainly; there is nothing to review.
- **Only static lenses run** (app not runnable) → the review ships with the `⚠️ static-only` header so no reader mistakes it for runtime-verified.
- **A lens errors** → its status line carries the error; the other four still compose. One lens failing never sinks the review.
