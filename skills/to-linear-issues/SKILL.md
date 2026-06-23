---
name: to-linear-issues
description: >-
  Turn a feature discussion into Linear issues split by DB schema, DAL, API, and UI. Use when the user asks to create Linear tickets from a
  conversation, plan a feature as issues, or run the to-linear-issues workflow.
---

# To Linear Issues

Convert the current conversation into Linear sub-tasks under the relevant epic. **English only.** Be compact in ticket bodies — no filler.

## Project configuration

Edit these placeholders in this file for your repo:

| Setting | Placeholder |
| ------- | ----------- |
| Issue key prefix | `PROJ` |
| Parent epic example | `PROJ-81` |
| Reference ticket format | `PROJ-88` |
| Linear team | `Your Team` |
| Linear project | `Your Project` |
| Default state | `Backlog` |
| Labels | `Feature` + context (`Mobile`, `Web`, `API`, `Database`) |

Rename area prefixes (e.g. `DAL:` → `DB:`), folder paths in section 2, or drop/reorder rows to match your stack.

## 1. Extract from conversation

Before creating issues, lock in:

- **Feature goal** (one sentence)
- **Product decisions** (scope, status rules, what's in/out of MVP)
- **Parent epic** (e.g. `PROJ-81`) — ask if missing
- **Platforms** — Mobile, Web, or both (separate UI tickets)
- **Schema changes** — new/altered tables, columns, indexes, constraints (e.g. Drizzle schema + Postgres migration)
- **Dependencies** — what must ship before what

If decisions are still open, ask the user first. Do not guess on scope rules.

## 2. Split into areas

Create **one issue per layer**, in dependency order:

| Order | Title prefix | Scope |
| ----- | ------------ | ----- |
| 1 | `Schema:` | `/db/schema/` Drizzle table definitions, relations, enums; generated/applied migrations (`drizzle-kit`). No DAL/router/UI. |
| 2 | `DAL:` | `/data-access/`, pure builders, unit tests. No router/UI. |
| 3 | `API:` | `/packages/api` Zod schemas + contract, `/api` router handlers, auth guards. |
| 4 | `Mobile:` | `/mobile/` — screens, hooks, query-options. |
| 5 | `Web:` | `/web/features/` — pages, components, query-options, i18n. |

Skip layers not needed (e.g. API-only change → no Schema/Mobile/Web). Add **Schema** when the feature needs new or changed tables. Split Mobile and Web when both are in scope.

**Blocked-by chain:** Schema → DAL → API → Mobile/Web (DAL blocks on Schema when present; UI tickets block on API, not DAL directly).

## 3. Ticket format (match PROJ-88)

```markdown
[Area prefix]: [concise feature slice]

## Scope
* Bullet decisions from conversation (data sources, rules, exclusions)

## Changes
* Concrete files/modules to add or extend

## Acceptance criteria
* Testable outcomes

## Blocked by
* PROJ-XX (previous layer) — or "Nothing"
```

**Title examples:** `Schema: add progress_snapshots table`, `DAL: per-student progress aggregation`, `API: per-student progress procedures`, `Mobile: instructor per-student progress screens`

## 4. Linear defaults

| Field | Value |
| ----- | ----- |
| team | `Your Team` |
| project | `Your Project` |
| state | `Backlog` |
| labels | `Feature` + context (`Mobile`, `Web`, `API`, `Database`) |

Set `parentId` to the epic. Read MCP tool schemas before calling.

## 5. Create issues (MCP)

Server: `plugin-linear-linear`, tool: `save_issue`.

1. Create **Schema** first when DB changes are needed (no `blockedBy`).
2. Create **DAL** with `blockedBy: ["PROJ-XX"]` (Schema id) when Schema exists; otherwise no `blockedBy`.
3. Create **API** with `blockedBy: ["PROJ-YY"]` (DAL id).
4. Create **Mobile** / **Web** with `blockedBy: ["PROJ-ZZ"]` (API id).
5. If `blockedBy` was omitted on create, update with `save_issue` + `id`.

Use literal newlines in `description` (not `\n` escapes). Link blockers in the `## Blocked by` section.

## 6. Report to user

Return a short summary:

- Feature goal + key decisions
- Dependency diagram (`Schema → DAL → API → UI` — omit Schema when not needed)
- Table: ID, title, area, URL
- Suggested start ticket (usually Schema or DAL)

## Rules

- Do not create tickets for undecided scope.
- Do not combine Schema + DAL + API + UI in one ticket.
- Do not update epic description unless asked.
- Embed conversation decisions in tickets — agents implementing later may not have chat context.
- Reference existing code paths/patterns from the discussion when known.
