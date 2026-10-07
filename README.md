# claude-skills

Personal, reusable [Claude Code](https://claude.com/claude-code) skills — installed at `~/.claude/skills/` so they're available across every project, not scoped to one repo.

## Skills

- **[agentic-development](agentic-development/SKILL.md)** — plans, implements, verifies, and reviews a repository change through a specification-driven workflow; runs `owasp-check` at its review step.
- **[implement](implement/SKILL.md)** — implements one bounded task end-to-end, then validates it against its acceptance criteria or Definition of Done before calling it complete.
- **[juliopy](juliopy/SKILL.md)** — scaffolds a production-grade Python backend: Docker, Postgres, Django or FastAPI, uv, ruff, mypy, pytest, CI, pre-commit, typed config, health checks. Always resolves current latest-stable versions instead of hardcoding them.
- **[owasp-check](owasp-check/SKILL.md)** — checks a diff or a codebase against the OWASP Top 10:2025 (plus the API Security Top 10:2023 for HTTP APIs): every category ends Finding, Clean, or N/A, each finding is confirmed with `file:line` evidence and an attack path, and the report gives severity and fixes. Run by `agentic-development` at its review step.
- **[ship-pr](ship-pr/SKILL.md)** — ships a change as a PR via GitHub flow: feature branch off the default branch, secrets and `.gitignore` validated, only the relevant files staged, then pushed and opened as a PR.
- **[spec-driven-development](spec-driven-development/SKILL.md)** — writes `spec.md`, `plan.md`, and `tasks.md` under `specs/<slug>/` with human sign-off between stages, then implements against `tasks.md`.
- **[tdd](tdd/SKILL.md)** — test-driven development via red-green-refactor, for one feature or bug fix at a time; used by `spec-driven-development` for its test-first loop.

## Usage

Clone this repo to `~/.claude/skills/` (or symlink individual skill directories into it) to make these skills available in Claude Code globally.
