# claude-skills

Personal, reusable [Claude Code](https://claude.com/claude-code) skills — installed at `~/.claude/skills/` so they're available across every project, not scoped to one repo.

## Skills

- **[juliopy](juliopy/SKILL.md)** — scaffolds a production-grade Python backend: Docker, Postgres, Django or FastAPI, uv, ruff, mypy, pytest, CI, pre-commit, typed config, health checks. Always resolves current latest-stable versions instead of hardcoding them.
- **[ship-pr](ship-pr/SKILL.md)** — ships a change as a PR via GitHub flow: feature branch off the default branch, secrets and `.gitignore` validated, only the relevant files staged, then pushed and opened as a PR.

## Usage

Clone this repo to `~/.claude/skills/` (or symlink individual skill directories into it) to make these skills available in Claude Code globally.
