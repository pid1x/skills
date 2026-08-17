# Posting mechanics

Read this only when actually posting (the `post` argument). The rules that decide **whether** and **what** to
post live in `SKILL.md` §5–§6; this file is only *how*.

## Pass every body by FILE, never inline

A review body is large markdown — backticks, `$`, quotes, code fences. Passed inline
(`gh … -f body="…"` / `--body "…"`) it breaks shell quoting and posts an **empty `~` body**: a silent,
content-less review that nobody sees fail. Always write the body to a temp file and pass it by reference.

## The main review — a COMMENT-type review

```
gh api repos/{owner}/{repo}/pulls/{n}/reviews --input <payload.json>
```

`payload.json` carries `"event": "COMMENT"` and the body. Never `-f body="…"`.

The body is led by the `🤖 deep-review · r<N> · <lenses>` header — the dedupe key lives **inside the review**,
not in a separate comment, because that is what a later round matches on.

## Anchoring findings inline

`SKILL.md` §6 carries the rule (**anchor every finding with a resolvable `file:line`**); this is the mechanism.

Inline comments ride in the same review payload, as a `comments` array alongside `event` and `body`:

```json
{ "event": "COMMENT", "body": "<summary body>",
  "comments": [ { "path": "src/orders/release.ts", "line": 214, "side": "RIGHT", "body": "<the finding>" } ] }
```

`line` must fall inside the diff hunks for `path`, or the whole call is rejected — so resolve positions from
`gh pr diff` first. For a finding on a line the PR does not touch (a pre-existing defect, a caller elsewhere),
the position will not resolve: that one belongs in the summary body, with its `file:line` written out.

## The runtime follow-up — a separate PR comment

```
gh pr comment <pr> --body-file <file>
```

Again by file. It is a separate post because `[mutation]` and `[verify]` finish after the static pass — the
same reason the automated loop splits the two.

## Verify the post landed

After sending, **re-read the review or comment on the PR** and confirm the body is non-empty and complete. A
silent empty post is worse than no post. If the body is empty or truncated the send failed: re-post from the
file, or report the failure plainly. Never leave an empty review standing.

## Own PR

GitHub blocks a review on your own PR. Fall back to a single plain `gh pr comment` carrying the write-up, or
report to chat if even that isn't wanted.
