# Project rules

Drop your project's convention files here as `.md` or `.mdc` files — for example, copies of your `.cursor/rules/*.mdc` files or any internal style guide. The `review-changes` skill reads every file in this directory (except this README) and checks each changed file against them.

How it works:

- **One file = one findings group.** The report groups findings per rule file, so split rules by topic (e.g. `react.md`, `project-structure.md`, `data-access.md`).
- **Be checkable, not aspirational.** Write rules a reviewer can verify against a diff: where files belong, naming conventions, required patterns, forbidden patterns.
- **No need to duplicate `CLAUDE.md` / `AGENTS.md`.** The skill already reads those from the repository root if they exist.

## Example rule file

```markdown
# React conventions

| Category | What to check |
|----------|---------------|
| Project structure | Components in `components/`, hooks in `hooks/use-*.ts`, query factories in `query-options/`, shared constants in `constants/`. |
| Naming | Kebab-case filenames. Named exports for components. `isX`/`hasX` boolean names. |
| Forms | Forms use TanStack Form with Zod schemas — no uncontrolled ad-hoc form state. |
| Styling | Tailwind only; no inline style objects except for dynamic values. |
| Comments | No comments in code. |
```

Plain prose rules work too — tables are just easy to scan. Anything in this directory is treated as authoritative project convention during review.
