# Engineering skills

Daily code work.

### User-invoked

_(none yet)_

### Model-invoked

- **[review-queue](./review-queue/SKILL.md)** — Drive a review queue unattended: pick the oldest ticket awaiting review, claim an isolated environment, hand the review to `deep-review`, mark, report, hand the environment back. One ticket per pass; dry-run by default.
- **[deep-review](./deep-review/SKILL.md)** — Orchestrate a full PR review (engine + spec/security/mutation/verify lenses) into one loop-style write-up; dry-run by default, opt-in `post` writes a comment-only review to the PR.
- **[mutation-check](./mutation-check/SKILL.md)** — Mutation-test a change and report the tests that let mutants survive.
- **[verify-live](./verify-live/SKILL.md)** — Runtime-verify a change by driving the real flow against the running app; reports one evidence-gated status.
- **[spec-check](./spec-check/SKILL.md)** — Check a change against the spec it was meant to satisfy; severity judged independently of the author's framing.
- **[security-check](./security-check/SKILL.md)** — Security-review a change against a vuln-class taxonomy, tracing tainted input from entry point to sink; reachability sets severity.
