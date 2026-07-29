# Skills

My own agent skills and workflows. They work with any coding agent that supports the Agent Skills standard — Claude Code, Codex, Cursor, and others — and with any model.

Conventions for buckets, promotion, and `SKILL.md` structure follow [Matt Pocock's skills](https://github.com/mattpocock/skills). This repo ships my own skills only — install Matt's set separately via `npx skills add mattpocock/skills`.

## Quickstart

```bash
npx skills@latest add pid1x/skills
```

Pick the skills you want, and which coding agents to install them on.

## Skills

See [CLAUDE.md](./CLAUDE.md) for conventions when adding skills.

### Engineering

Daily code work. See [skills/engineering/README.md](./skills/engineering/README.md).

#### User-invoked

_(none yet)_

#### Model-invoked

- **[review-queue](./skills/engineering/review-queue/SKILL.md)** — Drive a review queue unattended: pick the oldest ticket awaiting review, claim an isolated environment, hand the review to `deep-review`, mark, report, hand the environment back. One ticket per pass; dry-run by default.
- **[deep-review](./skills/engineering/deep-review/SKILL.md)** — Orchestrate a full PR review (engine + spec/security/mutation/verify lenses) into one loop-style write-up; dry-run by default, opt-in `post` writes a comment-only review to the PR.
- **[mutation-check](./skills/engineering/mutation-check/SKILL.md)** — Mutation-test a change and report the tests that let mutants survive.
- **[verify-live](./skills/engineering/verify-live/SKILL.md)** — Runtime-verify a change by driving the real flow against the running app; reports one evidence-gated status.
- **[spec-check](./skills/engineering/spec-check/SKILL.md)** — Check a change against the spec it was meant to satisfy; severity judged independently of the author's framing.
- **[security-check](./skills/engineering/security-check/SKILL.md)** — Security-review a change against a vuln-class taxonomy, tracing tainted input from entry point to sink; reachability sets severity.

### Productivity

Daily non-code workflow tools. See [skills/productivity/README.md](./skills/productivity/README.md).

_(none yet)_

### Personal

Not promoted in the plugin — tied to my own setup. See [skills/personal/README.md](./skills/personal/README.md).

#### User-invoked

- **[debrief](./skills/personal/debrief/SKILL.md)** — End-of-day debrief — pull anonymized notes from today's agent sessions worth keeping.

#### Model-invoked

_(none yet)_

## Local development

Run `scripts/link-skills.sh` to symlink every skill in this repo into `~/.agents/skills`, `~/.cursor/skills`, `~/.claude/skills`, and `~/.codex/skills`. Re-run it after adding, removing, or renaming a skill.

## Changelog

See [CHANGELOG.md](./CHANGELOG.md) for skill additions and changes.

## License

[MIT](./LICENSE)
