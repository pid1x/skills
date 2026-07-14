---
name: debrief
description: End-of-day debrief — pull 3–5 anonymized notes from today's agent sessions worth keeping.
disable-model-invocation: true
argument-hint: "[YYYY-MM-DD] — omit for today"
---

Go through today's sessions and pull out anything worth writing down as a note for myself.

## 1. Gather — all of today, not just what's open

Include **all** of today's sessions, both active and archived/closed ones — most of the finished work is in the closed sessions, not the one currently open.

Discover session transcripts by searching every known storage pattern on this machine (any harness may use one or more):

```bash
find \
  ~/.cursor/projects/*/agent-transcripts \
  ~/.claude/projects \
  ~/.codex \
  ~/.agents \
  -name '*.jsonl' -newermt '<date> 00:00:00' 2>/dev/null
```

Use the argument date if given; otherwise today in local time.

If nothing turns up, broaden the search: look under `~/.*` and `$HOME` for `.jsonl` files in directories named `agent-transcripts`, `projects`, `sessions`, or `conversations` modified on the target date.

Read **parent session** files first — a transcript at the session root (`<id>.jsonl` or `<id>/<id>.jsonl`), not nested paths (`subagents/`, `workflows/`, `journal`). Only read nested transcripts if a parent session is thin on lessons.

Skim for outcomes, surprises, and course-corrections — not tool output or code.

**Done when:** every parent-session transcript from the target date has been checked across every storage pattern found, not only the current conversation.

## 2. Extract — by priority

Context: I work hands-on as an engineering manager, mostly on agentic coding workflows in a large legacy codebase.

Prioritise, in this order:

1. **False green** — looked fine but wasn't: a green result that was actually wrong, a metric that lied, a tool that reported success while doing nothing
2. Something that went against my expectation, or that I changed my mind about
3. A limit or friction I hit with an agent, and what I did about it
4. A tool, skill or workflow that clearly did or didn't work

Skip anything that is just a tool observation without a lesson behind it.
Skip generic advice.

When more than 5 qualify, keep the highest-priority tier first; within a tier, prefer the biggest surprise or the lesson most likely to recur.

**Done when:** every candidate note is ranked against this list; skipped items have a one-line reason (no lesson, or generic); the final pick is capped at 5.

## 3. Anonymize — before writing anything

IMPORTANT — keep the notes free of any of the following:

- Ticket IDs, ticket titles, or ticket content
- Customer names, customer data, or anything customer-specific
- Product names, project names, repo names, service names, internal tooling names
- Team members, team size, org structure
- Infrastructure details, hostnames, environments
- Any code snippets from the codebase

Describe the situation in generic terms instead. For example, not "the PROJ-1234 import broke for customer X" but "a batch import kept failing silently and the agent didn't catch it".

**Done when:** every note passes the exclusion list; nothing identifiable remains.

## 4. Output

Open with one line: `Scanned N sessions from <date> (M sources).`

Then 3-5 notes, English. Each note: 2-4 sentences, covering what I expected, what actually happened, and how I found out. Raw observation, no polish, no headline.

At the end, if several notes share a common thread, name it in one line.

If nothing today was worth noting, say so instead of padding the list.

**Done when:** the scan line is present; the output is 3-5 notes (or an explicit "nothing worth noting today"), each 2-4 sentences, plus an optional one-line common thread.

## 5. Save — only if asked

Default: output stays in chat. If I ask to save, persist via whatever notes skill or vault I have configured on this machine.

**Done when:** notes are in chat, or also saved when explicitly requested.
