---
name: git-commit
description: Use when asked to stage, commit, or otherwise create a git commit in this repo. Defines the required workflow: inspect status/diff/log, stage only intended files, never commit secrets, use Conventional Commits, and NEVER push without explicit user approval.
---

# Git commit workflow

This repo uses Conventional Commits and strict guardrails around git operations. Apply this skill whenever a commit is requested or implied.

## Hard rules

1. **Never `git push` on your own.** Pushing requires the user to say "push" (or equivalent) explicitly — a commit request alone is NOT permission to push. If a commit completes and no push was requested, stop and ask.
2. **Never commit secrets.** Check `git diff` for credentials, tokens, `.env` files, or anything in the `VPN_PASS`/`DOMAIN` category before staging. Real values live in the Coolify UI, never in the repo.
3. **Only commit when the user asks.** A request to "commit" is required. Do not commit merely because work is finished.

## Steps

1. Inspect state first:
   ```bash
   git status
   git diff
   git log --oneline -10
   ```
2. Review what would be committed. Stage only intended files:
   ```bash
   git add <file> ...
   ```
   (Never `git add -A` or `git add .` without review.)
3. Write a Conventional Commits message matching repo style:
   - Types: `feat:`, `fix:`, `chore:`, `refactor:`, `docs:`
   - No emoji. Imperative, lowercase after the type, e.g. `fix: drop MEDIA_DIR default in volumes for coolify compat`.
4. Commit:
   ```bash
   git commit -m "<type>: <summary>"
   ```
5. If the commit fails or a hook rejects it, fix the issue and create a NEW commit; do not amend a failed commit.
6. After committing, if the user has not explicitly asked to push, report the commit and ask whether to push.

## Push

Only run `git push` after the user explicitly requests it. The commit message or intent of a task ("commit this") does not imply push permission.
