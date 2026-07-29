# Changelog

All notable changes to this repo are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Changed

- `engineering/deep-review` — end the write-up on **asks, never a tally**: every finding carries a one-line `→ ask:`, and the comment closes with a consolidated `## Asks` block grouped Must / Should / **Decide** (a human judgement, not a code change) / FYI (deliberately no action) — a closing statistic leaves the reader to derive the work. Plus a length budget, because a wall of text gets skimmed and a skimmed review changes nothing: ≈3.500 characters for the main comment, ≈400 / three sentences per finding, one line per FYI; cut the investigation narration, the restated ticket/PR, and confirmations first
- all four `engineering/` lenses — close each report with a single tagged `[verify]` / `[spec]` / `[security]` / `[mutation]` bottom-line status, so a reader (or the future `deep-review` orchestrator) folds the verdict without reading the full report
- `engineering/mutation-check` — report one of exactly three outcomes (`score N% · M survivors` / `no covered tests in diff` / `no result — <reason>`); a zero-output abort is `no result`, never `partial`; add pre-flight feasibility (integration-only coverage → `no result` upfront) and a watchdog on hangs
- `engineering/mutation-check` — add an agent-executable hand-applied fallback for when the tool can't target the changed code (legacy classes, string-literal predicates) or returns a hollow artifact score (single-mutant `100%`, all-timeout); a hollow artifact is never reported as a real score; provably-equivalent mutants are excluded from the score. The hand-applied pass is the one sanctioned source write and carries a hard revert guarantee — unconditional (fires on watchdog abort mid-mutant, test error, or interrupt), the tree is verified clean before reporting, a failed revert is named at the top, and it refuses to hand-apply on a dirty tree (`no result — dirty tree`); a timeout after some mutants scored is a bounded `score`, never a `partial`
- `engineering/spec-check` — read the ticket's comments, not just its description/AC; a reporter/spec-owner clarification amends the spec (a waived requirement no longer a gap), distinct from an author's self-disclosure which stays context-not-permission
- `engineering/verify-live` — add a second litmus: driving the flow is not enough if a feature flag / config gate / missing provisioning routes execution around the change; a flow that ran the old path is `not verified — <fix's path gated off>`, never `confirmed`; reaching a gated path is env mutation and needs the user's OK, never done silently — where the OK may be a standing, scoped pre-authorization from the caller (e.g. an autonomous routine authorizing flag-on + provisioning in a throwaway worktree, never credentials or shared config), so unattended runs stay honest instead of forcing every flag-gated change to `not verified`. When a change spans multiple runtime surfaces, report one canonical wording per surface (`confirmed ✓ (contract) · not verified (core)`) instead of a blended `partially confirmed`. The `.verify-live.json` cache holds only pointers to credentials (an env var name, a secret-manager key, a login route), never a secret itself — an untracked cache carrying a real secret would be a leaked credential; secrets are read from the environment at run time
- `docs/engineering/mutation-check.md` — synced with the above

### Added

- `engineering/review-queue` — the workflow half around `deep-review`: drive a review queue unattended (oldest ticket first, **one per pass**, the cadence drains the backlog), resolve its PR, claim an isolated environment, hand the review over, mark, report and hand the environment back. Two rules make unattended runs safe: the static review is **posted before** the slow runtime lenses start, and the runtime follow-up comment is the completion marker, so a truncated pass is resumed rather than redone (the ticket is only marked once both halves landed). No isolated environment → the pass defers instead of posting a runtime-less review that looks complete; `isolation.strategy: none` still runs the static lenses and labels the review `⚠️ static-only`. A refused tracker write is retried once, then degrades visibly — the review still ships and the brief names every skipped write and its consequence, and a denial is never routed around with another credential. All instance-specific values live in a git-ignored `.review-queue.json`, so the skill body is identical across repos; isolation defaults to [wts](https://github.com/pid1x/wts) with `fixed-worktree` / `fresh-clone` / `none` as alternatives

- `engineering/verify-live` — runtime-verify a change by driving the real flow against the running app; reports one evidence-gated status (`confirmed ✓` / `diverged` / `not verified`), never the test suite
- `docs/engineering/verify-live.md`
- `engineering/spec-check` — check a change against its originating spec/acceptance criteria (missing/partial, scope creep, implemented-but-wrong); severity judged independently of the author's framing, persistent-wrong-state is blocking
- `docs/engineering/spec-check.md`
- `engineering/security-check` — security-review a change against a vuln-class taxonomy, tracing tainted input from external entry point to sink across files; reachability sets severity (unreachable sink is informational, not a finding)
- `docs/engineering/security-check.md`
- `engineering/deep-review` — model-invoked orchestrator (reachable by hand or delegated to by a review routine) that folds five lenses (delegated engine review + spec/security/mutation/verify) into one loop-style write-up; capability detection with honest degradation, cross-lens audit before compose; dry-run by default, opt-in `post` writes a comment-only review to the PR (COMMENT-type review + runtime follow-up comment, delta-dedupe, own-PR fallback) matching the automated loop's shape; the body is passed by file (`--input` / `--body-file`), never inline, to avoid the empty-`~`-body shell-quoting failure, and the post is re-read to confirm a non-empty body landed; a delta re-review re-posts on new commits OR a verdict change (a stale-verdict correction, not a duplicate), and carries a lens result forward only on a byte-identical head SHA — a moved SHA (even a main-merge) re-runs the runtime lenses. Posts are headed `🤖 deep-review · <lenses>` — a distinct engine tag so the dedupe matches only its own prior posts and never mistakes another review engine's post on the same PR for its own
- `docs/engineering/deep-review.md`

## [0.0.2] — 2026-07-23

### Changed

- `engineering/mutation-check` — fetch `origin/<base>` before scoping a branch diff (avoids stale local base); ignore `.mutation-check.json` via `.git/info/exclude` instead of editing the repo's `.gitignore`
- `docs/engineering/mutation-check.md` — synced with the above

## [0.0.1] — 2026-07-14

Initial release.

### Added

- `engineering/mutation-check` — scoped mutation testing on a diff (branch, PR, ticket, or path)
- `personal/debrief` — end-of-day debrief; pull anonymized notes from today's agent sessions
- `docs/engineering/mutation-check.md`
- Repo scaffold: `engineering/`, `productivity/`, `personal/` buckets
- `scripts/link-skills.sh`, `scripts/list-skills.sh`
- Claude Code plugin manifest (`.claude-plugin/plugin.json`)
