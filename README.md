# Agent Skills

The agent skills I use every day with AI coding assistants (Claude Code and compatible tools).

## Skills

| Skill | Description |
|-------|-------------|
| [`commit-this`](skills/commit-this/SKILL.md) | Analyzes current changes and commits them with a proper conventional commit message — no confirmation, splits unrelated changes into separate commits. |
| [`conventional-commit`](skills/conventional-commit/SKILL.md) | Structured workflow for writing commit messages following the [Conventional Commits](https://www.conventionalcommits.org/) specification, with examples and validation rules. |
| [`review-changes`](skills/review-changes/SKILL.md) | Reviews uncommitted changes against your project's own rule files and a built-in React best-practices checklist, then conditionally spawns a "deslop" subagent to clean up AI-generated noise. |

## Usage

Copy a skill directory into your project's `.claude/skills/` folder (or `~/.claude/skills/` to make it available globally):

```bash
cp -r skills/review-changes /path/to/your-project/.claude/skills/
```

Then invoke it in your session, e.g. `/review-changes`.

### Customizing `review-changes`

The review skill is project-agnostic. Drop your project's convention files (`.md`/`.mdc`, e.g. your `.cursor/rules`) into its `rules/` directory and every finding will cite the specific rule violated — see [`rules/README.md`](skills/review-changes/rules/README.md).

## License

[MIT](LICENSE)
