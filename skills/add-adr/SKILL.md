---
name: add-adr
description: >-
  Capture an architectural decision from the current conversation as a new ADR
  record at .ai/adrs/<id>/adr.md. Use when the user asks to save, record, or
  document a decision as an ADR, or mentions architecture decision records.
---

# Add ADR

Turn a decision made in the current conversation into a new ADR file.

## Workflow

1. Scan the conversation for an architectural decision: a technology/library/framework choice, an architecture or pattern choice, a convention, or a trade-off that was resolved.
2. **If no decision is found, do not create any file.** Reply that you found no decision in the conversation that qualifies as an ADR, and ask the user what exactly they want to record — then stop and wait for their answer.
3. If a decision is found, derive a short kebab-case `id` from its topic (e.g. `frontend-framework`, `auth-strategy`, `db-migrations`).
4. Check whether `.ai/adrs/<id>/adr.md` already exists. If it does, pick a more specific `id` instead of overwriting it.
5. Write the file to `.ai/adrs/<id>/adr.md` using the template below, creating directories as needed.
6. Report the created path and a one-line summary of the decision.

## Rules

- One decision per ADR. If the conversation contains several unrelated decisions, create one file per decision.
- Fill every section from what was actually discussed. Never invent reasons or consequences; if context or consequences were not discussed, state briefly what is known and ask the user to fill the gap.
- `created` is today's date in `YYYY-MM-DD`.
- Default `Status` is `Accepted` unless the conversation says the decision is proposed, rejected, or supersedes another ADR.
- Write the ADR body (including section headings) in the language the user speaks in the conversation.
- Keep it short — a reader should get the decision and its cost in under a minute.

## Template

```markdown
---
id: <kebab-case-id>
created: <YYYY-MM-DD>
---

# ADR: <short decision title>

## Status

Accepted

## Context

<Why this decision was needed: the problem, constraints, team/product context.>

## Decision

We choose **<option>** <short qualifier>.

Reasons:

- <reason 1>
- <reason 2>
- <reason 3>

## Consequences

**Positive**

- <benefit 1>
- <benefit 2>

**Negative / risks**

- <cost or risk 1>
- <cost or risk 2>
```

## Example

Conversation: the team compared Next.js and a separate React SPA + backend, and settled on Next.js because they already know React and want one codebase.

Result — `.ai/adrs/frontend-framework/adr.md`:

```markdown
---
id: frontend-framework
created: 2026-07-29
---

# ADR: Frontend framework choice

## Status

Accepted

## Context

We need a framework for the web app. The team has React experience. We want to avoid a separate backend at the start and keep frontend and API in one place.

## Decision

We choose **Next.js** (App Router) as the application framework.

Reasons:

- full-stack framework — frontend and backend (Route Handlers, Server Actions) in one project
- built on React, which the team already knows
- native SSR/SSG support and good DX in the React ecosystem

## Consequences

**Positive**

- single codebase for UI and server logic
- shorter onboarding thanks to React familiarity
- less glue code between a separate frontend and API

**Negative / risks**

- tighter coupling between frontend and backend
- some decisions (routing, caching, Server Components) are Next.js-specific
```

## No decision found

Do not guess and do not write a placeholder file. Respond in this shape:

> I did not find any decision in our conversation that qualifies as an ADR. What exactly would you like to record — which option was chosen, and what were the alternatives and reasons?
