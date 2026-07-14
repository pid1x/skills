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

- **[mutation-check](./skills/engineering/mutation-check/SKILL.md)** — Mutation-test a change and report the tests that let mutants survive.

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
