---
name: commit-this
description: 'Commit all currently implemented changes using conventional commit practices. Analyzes staged/unstaged changes, constructs a proper conventional commit message, and executes the commit.'
---

### Instructions

When the user invokes this skill, commit all current changes following the Conventional Commits specification. Do not ask for confirmation — just analyze and commit.

For commit message structure, types, examples, and validation rules, follow the `/conventional-commit` skill at `.claude/skills/conventional-commit/SKILL.md`.


Run `git status` after committing to verify success.

### Rules

- If changes span multiple unrelated concerns, create separate commits for each logical unit.
- Do not push to remote unless explicitly asked.
