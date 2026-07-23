# Changelog

All notable changes to this repo are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/).

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
