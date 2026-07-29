---
name: review-queue
description: Drive a review queue unattended — take the oldest ticket awaiting review, resolve its PR, claim an isolated environment, hand the review to deep-review, mark the ticket, report, and hand the environment back. One ticket per pass; dry-run by default, `live` writes.
argument-hint: "[live]  — omit for dry-run (reports what it would do, writes nothing)"
---

A review engine is only half a workflow. `review-queue` is the other half: it decides **which** change gets
reviewed, **where** it gets reviewed, and **who hears about it** — then hands the actual reviewing to
[`deep-review`](../deep-review/SKILL.md) and never second-guesses its findings.

It is built for **unattended** runs (a scheduled task, several times a day). Two rules make that safe:
**one ticket per pass** — the cadence, not a batch loop, drains the queue — and **deliver before you
finish**, so a pass that dies partway still shipped something useful and the next pass can resume it.

## 0a. DRY-RUN unless the argument is `live` — the gate over every write
**Without the `live` argument nothing leaves this machine.** Dry-run is the default because the writes are
outward-facing and hard to undo: a review on someone else's PR cannot be unseen.

There are exactly **five** writes, and **every one of them is LIVE-only**. In dry-run, do the *work* — poll,
resolve, claim, stand up, run the lenses, compose — then **report what you would have written** and stop:

| Step | Write | Dry-run instead |
|---|---|---|
| §1 | remove stale marker labels | list the tickets whose marker *would* be reset |
| §3 | re-add a lost marker (self-heal) | name it |
| §4 | assign the ticket | say who it *would* be assigned to |
| §6 | **`deep-review` with `post`** — the review on the PR | invoke `deep-review` **without** `post`: it composes and reports, posts nothing |
| §7 | add the marker label · send the brief | print the brief, prefixed `[DRY-RUN]` |

**LITMUS: in dry-run, `deep-review` must never receive `post`.** That single argument is the difference
between a rehearsal and a review on a stranger's pull request.

Everything else — reading the queue, claiming an environment, standing it up, running all five lenses — is
identical in both modes. That is the point: a dry-run rehearses the real pass, it does not simulate it.

**Done when:** the mode is established from the argument, and if it is dry-run, every step below treats its
write as a *report*.

## 0. Read the configuration
All instance-specific values live in **`.review-queue.json`** at the repo root, so this skill stays
identical across repos. Missing file → say what is needed and stop; never guess a queue query.

```json
{
  "tracker": "jira",
  "queue": "(assignee = currentUser() OR ((project = ABC OR labels = ABC) AND assignee IS EMPTY)) AND status in (\"Ready for review\", \"In Review\") ORDER BY created ASC",
  "reviewStatuses": ["Ready for review", "In Review"],
  "assignTo": "someone@example.com",
  "marker": { "label": "agent-reviewed", "header": "🤖 deep-review" },
  "notify": { "via": "slack-dm", "target": "@me" },
  "vcsIdentity": "your-gh-login",
  "isolation": { "strategy": "wts", "tool": "path to wts (vendored copy, or just `wts` on PATH)", "prefer": "review", "parkingBranch": "<name>-main" },
  "env": { "baseUrl": "https://<worktree>.test", "repair": ["project-specific recovery steps"], "generatedArtifacts": ["paths a real flow may regenerate — revertable on restore"] },
  "envMutation": { "authorized": ["feature flags", "provisioning rows", "migrations", "deps"], "forbidden": ["credentials", "shared config"] },
  "reviewRules": ["repo review-convention files to pass to the engine lens, e.g. .github/instructions/*.instructions.md"],
  "workHours": "when an unattended pass may run, e.g. weekdays 08:00-18:00 Europe/Berlin"
}
```

**Done when:** the config is loaded and the queue query, marker, notify target and isolation strategy are known.

### When the tracker refuses writes
Reads and writes are separate permissions. A run can often **read** the queue while every **write** is denied
(label edits, assignment) — an unattended run in particular. That is a **degraded mode, not a failure**, and
it must be visible rather than silent:

- **Denials are per-call — retry once, later.** A refusal is not necessarily a standing permission state:
  the same call can be blocked early in a pass and succeed minutes later. So on a denial, carry
  on and **re-attempt that write once, later in the pass** (e.g. at the marking step). Only if the retry is
  refused too do you declare degraded mode. **Never work around a denial** — no direct API call with a
  personal token, no alternate credential. A refused permission is an answer; a retry is not a workaround.
- **The review still ships.** Reviewing is a VCS write, unaffected by the tracker. Never abandon a pass over
  bookkeeping.
- **The dedupe survives untouched**, because the source of truth is the **review header on the PR**, not the
  label. The marker is only a cache of it — so a marker-less pass can still tell reviewed from unreviewed and
  will not review the same change twice.
- **Say exactly what did not happen** in the brief, with the ticket keys: *"tracker writes denied — the marker
  was not set on `<KEY>`, so it returns next pass; stale markers on `<KEY…>` could not be reset, so those
  tickets stay out of the queue until someone clears the label by hand."* Never report a marker as set when it
  was not — a bookkeeping claim is a claim like any other.

**Done when:** either every tracker write succeeded, or the pass ran degraded and the brief names each write
that was refused and the consequence of it.

## 1. Reset stale markers — before the poll
The marker label means exactly: *in a review status **and** assigned to the reviewer **and** already fully
reviewed.* **Reset it the moment any of those breaks** — a status transition **or** a re-assignment:

```
labels = "<marker.label>" AND (status not in (<reviewStatuses>) OR assignee != <assignTo> OR assignee IS EMPTY)
```

**(LIVE only — in dry-run, list the tickets instead.)** Remove the label from every match (targeted remove — never overwrite the label list). A ticket handed back
to a developer went back for rework, so the marker must clear and let it re-enter the queue cleanly.

**Compare against `assignTo`, not the querying account.** They are often different (a routine may run under one
identity and assign to another); using the runner's identity here makes §1 strip the very label §4 just set —
label churn on every pass. The condition above is the *semantics*; the JQL is one tracker's spelling of it —
express it in whatever query language `tracker` uses.

**Done when:** no ticket outside the review statuses (or off the reviewer's name) still carries the marker — or, if the remove was refused, the affected keys are recorded for the brief (see *When the tracker refuses writes*). A marker that cannot be cleared keeps that ticket **out of the queue**, so it must be named, not shrugged off.

## 2. Load the queue, oldest first
Run `queue` from the config, excluding the marker. Nothing matched → **end quietly**, no notification.

A pass is **single-shot**: never loop, never schedule your own wake-up. The scheduler owns the cadence.

**Done when:** the candidate list is in hand, oldest first, or the pass ended quietly.

## 3. Pick exactly ONE ticket
Walk oldest-first. **Never batch.** Per candidate:

- **Resolve its PR** by ticket key (`gh`, excluding PRs authored by `vcsIdentity` — verify `author.login`).
  - **Own PR** → **silent skip**, no tracker write → next candidate. You cannot review your own work.
  - **No PR at all** → notify **once per day** (a review-status ticket without a PR is a real anomaly, but
    one reminder is enough) → next candidate.
- **Dedupe:** a prior review whose body carries `marker.header` exists **and no new commits since it** →
  **skip**; if the ticket lost its marker, re-add it so it stops re-entering the walk **(LIVE only)**. **New commits since
  it → not a skip:** the change came back for another look, so `deep-review` handles it as a delta.
- An existing **approval or plain human review NEVER skips** — this workflow's lenses (spec, security,
  mutation, runtime verification) surface findings a plain review does not contain.

**Done when:** exactly one reviewable ticket is selected, or every candidate was skipped with its reason.

## 4. Claim an isolated environment — then take ownership
Only now, and **before** assigning anything. The runtime lenses need a working app that is not the
reviewer's own workspace.

| `isolation.strategy` | How |
|---|---|
| `wts` *(default)* | `<isolation.tool> -j` → pick a **GREEN** worktree, preferring `prefer`; never the session's own cwd. `isolation.tool` is the path to [wts](https://github.com/pid1x/wts) — a repo-vendored copy or just `wts` on `PATH` |
| `fixed-worktree` | one dedicated worktree; free only if its tree is clean |
| `fresh-clone` | clone into a scratch dir per pass (slow, always free) |
| `none` | no isolation — runtime lenses will degrade to `static-only`, honestly labelled |

**Nothing free → STOP the pass before assigning:** no review, no ownership change. Notify once
(`⛔ review deferred — <TICKET> waiting, no environment free`). The next pass **auto-resumes** when one
frees; no manual signal. **No environment = no full review, and a half-baked review is worse than none.**

Then take ownership **(LIVE only — dry-run says who it would go to)**: unassigned → assign to `assignTo`; already theirs → leave it. Assignment refused → carry on and note it (see *When the tracker refuses writes*); ownership is bookkeeping, not a precondition for reviewing.

**Done when:** an isolated environment is claimed (or the pass stopped and said why) and the ticket is owned.

## 5. Stand the environment up — to the head, not to the diff
`git fetch` → check out the **PR head** → **reconcile the environment to that head**: run **all** pending
migrations, install dependencies to match the **checked-out** lockfiles, clear caches, build assets if the
change touches UI. A free environment is not necessarily a *current* one — it may be far behind the default
branch, and reconciling only the PR's own diff leaves it subtly wrong.

**Environment not ready is a REPAIR task, never a reason to degrade the review.** Work through
`env.repair` from the config, then confirm the app answers at `env.baseUrl`. Only if it still refuses after
that, say so loudly in the brief and let the lenses report their own honest degraded status.

**Standing env-mutation authorization.** `verify-live` needs consent before mutating the environment to
reach a change's own code path (turning on its feature flag, seeding provisioning). Unattended, nobody can
grant it — so `envMutation.authorized` **is** that consent, scoped to the claimed throwaway environment,
and `envMutation.forbidden` is absolute: **never** write credentials or touch shared config, not even to
complete a lens. Pass this along, and list every mutation performed in the brief.

**Done when:** the app answers at `baseUrl` on the PR head, or the repair steps are exhausted and that is stated.

## 6. Hand the review to deep-review — in two stages
Invoke `deep-review` with the PR — **and `post` ONLY in live mode.** In dry-run invoke it *without* `post`, so it composes the write-up and reports it without touching the PR (see §0a). Give it: the head is already checked out and the app is up
(**you** own checkout and restore), the originating ticket key, the standing env-mutation authorization,
and any repo review rules it should apply (e.g. matching `.github/instructions/*.instructions.md`, which a
generic engine would otherwise miss).

**Deliver in two stages** — this deliberately re-orders `deep-review`'s compose-then-deliver ending,
because one unattended context may not hold five deep lenses:

1. **Static → post immediately.** Engine review + `[spec]` + `[security]` → post the main review **before**
   touching a runtime lens, noting that the runtime pass is still to come. From here the pass has delivered.
2. **Runtime → follow-up comment.** `[mutation]` + `[verify]` with their proof blocks, plus the cross-lens
   links. Only when this lands is the pass complete.

**Take every lens bottom-line verbatim** — never paraphrase, re-grade or soften it. And **never hand-roll a
lens**: if one could not run *as its skill*, say so in its status line rather than substituting your own
version and reporting it as the lens.

**Done when:** the main review is posted and either the runtime follow-up is posted or its absence is stated.

## 7. Mark, report, restore
- **Mark** *(LIVE only)* — add `marker.label` **only after stage 2 landed** (refused → say so in the brief with the key, and expect the ticket back next pass; the header-dedupe keeps it from being reviewed twice). A static-only pass stays unmarked so the next
  pass resumes it: a prior review **without** a runtime follow-up counts as **incomplete** → run only the
  runtime stage, do not redo the static one. The PR itself is the journal; no extra state.
- **Report** — notify via `notify` (in dry-run, print the brief prefixed `[DRY-RUN]` instead of sending it), folding `deep-review`'s verdict and each lens bottom-line verbatim, plus
  the diff size, the PR link and any environment mutations performed.
- **Restore — always, even after a failure.** Dirty tree (unexpected: a review must not write source) →
  **discard nothing**, report the file list, leave it.
  - **Exception — named generated artifacts.** Driving a real flow can *unavoidably* regenerate machine
    artifacts (font caches from a PDF render, compiled assets). Left behind they make the environment read
    *busy* forever and shrink the pool — the workflow would block itself. So `env.generatedArtifacts` lists
    those paths explicitly, and **only** they may be reverted (`git checkout --` / delete), because they are
    regenerable output, not work. **Everything outside that list still stops the restore.** The allowlist is
    a config decision, never a judgement call at runtime; report which artifacts were reverted.
  - Otherwise park the environment back on its
  `parkingBranch` and fast-forward it to the default branch. Left on the reviewed branch it looks *busy* to
  the next pass and shrinks the pool of claimable environments.

**Done when:** the ticket is marked (or deliberately left unmarked), the brief is out, and the environment
is parked clean — the last of these runs even when the review failed.

## Scheduling it
Run `/review-queue` by hand first (**dry-run** — it writes nothing) until a pass looks right, then wire it to
a scheduler with `live`, a few times a day on working days. Cadence is the throughput knob: one ticket per
pass means N tickets need N passes. Repeated passes are cheap and idempotent — the marker and the dedupe
keep an already-reviewed change from being reviewed twice.

## When nothing fits
- **No `.review-queue.json`** → say which fields are needed and stop. Never invent a queue query.
- **Tracker writes denied, reads fine** → run degraded: review and post as usual, skip the label/assignment
  writes, and name every skipped write plus its consequence in the brief. Never route around the denial with
  another credential.
- **Empty queue** → end quietly. No notification: nothing was deferred, nothing failed.
- **No environment free** → defer with one notification and auto-resume next pass. Never review without one.
- **`isolation.strategy: none`** → static lenses only, and the review says `⚠️ static-only` so no reader
  mistakes it for runtime-verified.
