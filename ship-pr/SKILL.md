---
name: ship-pr
description: Ship a change as a pull request via GitHub flow — feature branch off the default branch, secrets and .gitignore validated, only the relevant files staged, then pushed and opened as a PR. Use when the user asks to push changes, ship/land a change, open or create a PR, or wants a proper git workflow before pushing.
---

# Ship PR

GitHub flow: branch, validate, stage narrow, push, open PR. Run the steps in order — each gates the next.

## 1. Branch off the default branch

Find the default branch (`git symbolic-ref refs/remotes/origin/HEAD`, usually `main`). If already on a non-default branch with the target changes, skip to step 2.

Otherwise, before staging anything:
1. Sync it: `git fetch origin` then fast-forward (`git merge --ff-only origin/<default>`).
2. Cut a feature branch named for the change, kebab-case (`git checkout -b fix/event-feed-sse`).

Never commit or push directly to the default branch.

Done when: `git branch --show-current` prints the new feature branch.

## 2. Validate secrets

List what would be staged (`git status --porcelain`, `git add -A --dry-run`) and check every file in that list, not just the ones you edited:

- Grep for secret shapes: AWS keys (`AKIA[0-9A-Z]{16}`), private key blocks (`-----BEGIN ... PRIVATE KEY-----`), provider tokens (`sk-`, `ghp_`, `xox[baprs]-`), hardcoded `password=`/`secret=`/`api_key=` assignments with real-looking values.
- Check filenames even when the grep is clean: `.env` (not `.env.example`), `*.pem`, `*.key`, `id_rsa*`, `credentials.json`, `service-account*.json`.

A hit means stop — do not stage or commit that file. Tell the user what was found and where before doing anything else.

Done when: the grep and the filename sweep are both clean, or every hit has been resolved (removed, or confirmed safe by the user) and re-checked.

## 3. Validate ignore files

Read the effective `.gitignore`(s) — repo root and any nested package ones — and confirm they exclude:

- dependency dirs: `node_modules`, `.venv`/`venv`, `vendor`
- build output: `dist`, `build`, `.next`, `out`, `target`
- env files: `.env*` (excluding `*.example`)
- OS/editor cruft: `.DS_Store`, `.idea`, `.vscode`
- caches: `.turbo`, `coverage`, `__pycache__`

Patch `.gitignore` for any gap, then re-run the dry-run staging.

Done when: `git add -A --dry-run` contains none of the above categories.

## 4. Stage only what's needed

Stage the paths the change actually touches with targeted `git add <paths>`. Reach for `git add -A`/`git add .` only when the working tree is already clean of unrelated untracked files — a dirty tree makes blanket staging sweep in stray work that isn't part of this change.

Review `git diff --cached --stat` before committing. Anything that isn't part of the requested change: unstage it (`git restore --staged <path>`) and ask the user whether it belongs.

Done when: `git diff --cached --stat` lists only files implementing the requested change.

## 5. Commit, push, open the PR

Commit and open the PR following your standard commit-message and pull-request conventions (including any attribution footer your instructions specify). Push the feature branch, not the default branch:

```
git push -u origin <branch>
gh pr create --title "..." --body "..."
```

Report the PR URL back to the user.
