# review-queue

**Drive a review queue unattended.** `deep-review` reviews one change well; `review-queue` is the half that
decides *which* change, *where* it gets reviewed, and *who hears about it* — then hands the reviewing over and
never second-guesses the findings.

Built for a scheduled task running a few times a day.

## What it does in one pass

1. **Reset stale markers** — the done-marker label means *this change was already fully reviewed*; once the
   ticket leaves the review lane the label is cleared so it can re-enter cleanly. It resets on **status
   only**: the label is a shared name that other people's queues write too, so a ticket's assignee is never
   treated as proof of ownership, and a marker this queue cannot prove it set is left alone.
2. **Load the queue, oldest first**, and pick **exactly one** ticket. Never a batch loop.
3. **Resolve its PR** — skip your own PRs silently, notify once a day about a review-status ticket that has no
   PR at all, and skip a change already carrying this engine's review header *unless* new commits followed
   (then it is a delta re-review). An approval or a plain human review never skips it: the lenses see things a
   plain review does not.
4. **Claim an isolated environment** — the runtime lenses need a running app that is not the reviewer's own
   workspace. Nothing free → the pass **defers** and the next one auto-resumes.
5. **Stand it up to the PR head** — all pending migrations, deps from the *checked-out* lockfiles, caches,
   assets. A free environment is not necessarily a current one.
6. **Hand off to `deep-review`**, then mark, report, and hand the environment back.

## It's working if

- **One ticket per pass, and the cadence drains the queue.** A bad pass costs one review, never the backlog.
- **It delivers before it finishes.** The static review is posted *before* the slow runtime lenses start, so a
  pass that runs out of room has still shipped something. The runtime follow-up comment is the completion
  marker: a prior review without one is *incomplete*, so the next pass resumes exactly the missing half
  instead of redoing the work. The ticket is only marked once both halves landed.
- **No environment, no review.** It defers rather than posting a runtime-less review that looks complete.
  With `isolation.strategy: none` it still runs the static lenses, and the review carries `⚠️ static-only`.
- **A refused tracker write degrades visibly, never silently.** Denials can be per-call, so it retries once
  later; if the write is truly blocked the review still ships and the brief names each skipped write and its
  consequence. It never routes around a denial with another credential — and never reports a marker as set
  when it was not.
- **It hands the environment back clean.** Dirty tree → discards nothing and reports. The one exception is the
  explicitly configured `env.generatedArtifacts` — paths a real flow unavoidably regenerates (font caches from
  a PDF render, compiled assets); left behind, they make the environment look busy forever and the workflow
  would block its own pool.
- **It never re-implements a lens.** If one cannot run as its skill, that is stated — not substituted with a
  hand-rolled version reported as the real thing.

## Configuration

Everything instance-specific lives in `.review-queue.json` at the repo root (git-ignore it — the values are
per-reviewer), so the skill body is identical in every repo: the queue query and review statuses, the account
to assign to, the marker label and review header, the notify target, your VCS login (used to skip your own
PRs), the isolation strategy, the base URL plus project-specific env-repair steps and regenerable artifact
paths, and the standing env-mutation authorization. See the schema at the top of
[the skill](../../skills/engineering/review-queue/SKILL.md).

**Isolation** defaults to [`wts`](https://github.com/pid1x/wts) — "which worktree is free for an agent?" —
with `isolation.tool` pointing at it (a repo-vendored copy or just `wts` on `PATH`). `fixed-worktree`,
`fresh-clone` and `none` are the alternatives, so no worktree fleet is required to use the workflow.

## Scheduling

Run it by hand first — **dry-run is the default** and writes nothing — until a pass looks right, then wire it
to a scheduler with `live`. A scheduled task itself cannot be version-controlled, so the schedule stays local
while the workflow is versioned here.
