# Agent Skills

The agent skills I use every day with AI coding assistants (Claude Code and compatible tools).

## Skills

| Skill | Description |
|-------|-------------|
| [`commit-this`](skills/commit-this/SKILL.md) | Analyzes current changes and commits them with a proper conventional commit message — no confirmation, splits unrelated changes into separate commits. |
| [`conventional-commit`](skills/conventional-commit/SKILL.md) | Structured workflow for writing commit messages following the [Conventional Commits](https://www.conventionalcommits.org/) specification, with examples and validation rules. |
| [`plan-feature-driven-refactor`](skills/plan-feature-driven-refactor/SKILL.md) | Produces a complete, executable plan for migrating a codebase from a layer-by-type layout (flat `components/`, `hooks/`, `services/`...) to feature-driven architecture (`features/<feature>/` + `shared/`) — move-only, verified by the type checker, with execution starting only after you approve the plan. |
| [`review-changes`](skills/review-changes/SKILL.md) | Reviews uncommitted changes against your project's own rule files and a built-in React best-practices checklist, then conditionally spawns a "deslop" subagent to clean up AI-generated noise. |
| [`to-linear-issues`](skills/to-linear-issues/SKILL.md) | Turns a feature discussion into Linear sub-tasks split by layer — DB schema (e.g. Drizzle), DAL, API, and UI (Mobile/Web) — with dependency order, ticket format, and MCP creation workflow. |
| [`update-linear-status`](skills/update-linear-status/SKILL.md) | Updates the active Linear issue status via MCP — resolves the ticket from context, branch, or explicit ID, then transitions it (`In Progress`, `In Review`, `Done`, etc.)|

## Installation

Install a skill with the [Skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add jjxmonster/skills --skill commit-this
```

Then invoke it in your session, e.g. `/commit-this`.

### Customizing `review-changes`

The review skill is project-agnostic. Drop your project's convention files (`.md`/`.mdc`, e.g. your `.cursor/rules`) into its `rules/` directory and every finding will cite the specific rule violated — see [`rules/README.md`](skills/review-changes/rules/README.md).

### Customizing `to-linear-issues`

The skill ships with generic placeholders. After installing, open [`skills/to-linear-issues/SKILL.md`](skills/to-linear-issues/SKILL.md) and replace them with your project's values before first use:

| Placeholder | Replace with |
|-------------|--------------|
| `PROJ` | Your Linear issue key prefix (e.g. `ACME`) |
| `Your Team` / `Your Project` | Your Linear team and project names |
| `Backlog` | Default issue state, if different |
| Labels (`Mobile`, `Web`, `API`, `Database`) | Labels your team actually uses |
| Area prefixes (`Schema:`, `DAL:`, `API:`, …) | Prefixes that match your stack — rename or drop rows you do not need |
| Folder paths (`/db/schema/`, `/data-access/`, `/api`, …) | Real paths in your repo |

You only need to edit the skill file once per project (or per repo if you install it locally). The agent reads those values when splitting a conversation into tickets and creating them via Linear MCP.

## License

[MIT](LICENSE)
