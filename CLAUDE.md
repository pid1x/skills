Skills are organized into bucket folders under `skills/`:

- `engineering/` — daily code work
- `productivity/` — daily non-code workflow tools

Both buckets are **promoted**: every skill in them must have a reference in the top-level `README.md` and an entry in `.claude-plugin/plugin.json`. Each skill entry in the top-level `README.md` must link the skill name to its `SKILL.md`.

Each bucket folder has a `README.md` that lists every skill in the bucket with a one-line description, with the skill name linked to its `SKILL.md`. Group entries into **User-invoked** and **Model-invoked**.

Skills also have a human-facing docs page at `docs/<bucket>/<skill-name>.md` (the docs tree mirrors the bucket folders under `skills/`). When you add, rename, or change the behaviour of a skill, create or re-sync its docs page and add an entry to [`CHANGELOG.md`](./CHANGELOG.md).

Every `SKILL.md` is either user-invoked (`disable-model-invocation: true`, reachable only by the human) or model-invoked (model- or user-reachable). See `skills/productivity/writing-great-skills` in the [aihero.dev/skills](https://github.com/mattpocock/skills) repo for the conventions this repo follows for writing a good `SKILL.md`.

If a skill isn't ready to promote yet, or is too tied to one personal setup to share, add an `in-progress/` or `personal/` bucket for it when the need actually arises — don't create them speculatively.

To (re)link every skill into the local harness skill directories (`~/.agents/skills`, `~/.cursor/skills`, `~/.claude/skills`, `~/.codex/skills`), run `scripts/link-skills.sh`. Each entry is a symlink into this repo, so a `git pull` keeps installed skills current; re-run the script after adding, removing, or renaming a skill.
