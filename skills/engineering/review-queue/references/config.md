# `.review-queue.json` — the per-instance config

Read this on the first run in a repo, or when a field is missing or unclear. It lives at the **repo root** and
is **never committed** (the values are per-reviewer). `SKILL.md` §0 carries the operative rule: **missing file →
say what is needed and stop; never guess a queue query.**

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

## Fields

| Field | Notes |
|---|---|
| `tracker` | Which issue tracker. The queries below are that tracker's query language. |
| `queue` | The candidate query, oldest first. Never invented — if it is absent, stop. |
| `reviewStatuses` | The review lane. §1's stale-marker reset keys on *leaving* it. |
| `assignTo` | Who the ticket is assigned to at §4. **Not** used by §1 — see §1 on why assignee is not evidence of marker ownership. |
| `marker.label` | The done-marker. **A shared name**: other people's queues write it too. |
| `marker.header` | The review header the dedupe matches. **Fixed** — renaming it breaks the delta. |
| `notify` | Where the operator brief goes. |
| `vcsIdentity` | Your VCS login, so §3 can silently skip your own PRs. |
| `isolation` | `wts` (default) / `fixed-worktree` / `fresh-clone` / `none`. `prefer` names a favoured slot; `parkingBranch` is where §7 parks it. |
| `env.baseUrl` | Where the claimed environment answers. |
| `env.repair` | Project-specific recovery steps for §5, tried in order. |
| `env.generatedArtifacts` | The **only** paths §7's restore may revert — regenerable machine output, never work. |
| `envMutation.authorized` | The standing, scoped OK the runtime lenses rely on unattended. |
| `envMutation.forbidden` | Absolute. Credentials and shared config, never — not even to complete a lens. |
| `reviewRules` | Repo review-convention files handed to the engine lens. |
| `workHours` | When an unattended pass may run. |
