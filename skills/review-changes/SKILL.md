---
name: review-changes
description: Reviews uncommitted git changes against project rule files (rules/ directory) and built-in React best practices, then conditionally spawns a deslop subagent. Use when the user wants a code review of working-tree changes or mentions "/review-changes".
---

# Review Changes

Review uncommitted git changes against your project's own rule files and a built-in React best-practices checklist. Deliver a structured report, then conditionally spawn a deslop subagent when slop-like patterns are found.

This skill is project-agnostic. Drop your project's convention files into the `rules/` directory next to this file to teach it your codebase — see `rules/README.md`.

## Workflow

Follow these steps in order. Do not edit code during the review itself — edits happen only via the conditional deslop subagent or explicit user follow-up.

### 1. Gather changes

Run from the repository root:

```bash
git status --short
git diff HEAD
```

- If both outputs are empty, stop and report: **No changes to review.**
- Read each touched file in full so the review covers context, not just diff hunks.

### 2. Load project rules

- Read every `.md` and `.mdc` file in the `rules/` directory next to this SKILL.md (skip `README.md`).
- Also read `CLAUDE.md` and `AGENTS.md` at the repository root, if they exist.
- Each rule file becomes one findings group in the report. Every finding must cite the rule file and the specific rule violated.
- If `rules/` contains no rule files, note in the report: **No project rules configured — reviewing against built-in React checklist only.** Then continue.

### 3. Review against the built-in React checklist

Check every changed file against this checklist. It applies to both React DOM and React Native code; skip categories that do not apply to the change.

| Priority | Category | What to check |
|----------|----------|---------------|
| **HIGH** | Components & hooks | Functional components with hooks. Non-trivial `useState`/`useEffect` clusters, data wiring, and multi-concern handlers extracted to custom hooks. Named exports for reusable code (default exports OK where the framework requires them, e.g. file-based routing). File structure and naming consistent with neighboring files. |
| **HIGH** | State & effects | Derived state computed during render, not in effects. Narrow effect dependency arrays. Event handlers over effect-driven side effects. Lazy `useState` initialization for expensive values. Functional `setState` when the next value depends on the previous one. |
| **HIGH** | Data fetching | Use the project's query layer (query options factories, typed clients) where one exists — no raw `fetch` or inline query configs in components. Loading and error states for data-fetching components. No request waterfalls: independent operations run in parallel. |
| **MEDIUM** | Rendering | Explicit `?:` over `&&` for conditionals. Stable, meaningful list keys. Hoist static JSX and values out of the component body. |
| **MEDIUM** | Clean code | Guard clauses and early returns. Happy path last. No unnecessary `else`. No dead code. User-visible errors surfaced — no silent failures for flows the user expects to succeed. |
| **MEDIUM** | Accessibility | Semantic elements and roles. Labels for interactive elements. Touch targets and focus handling appropriate to the platform. |

Tag each finding with impact: **CRITICAL**, **HIGH**, **MEDIUM**, or **LOW**.

### 4. Produce the report

Deliver a concise Markdown report with these sections:

#### Files reviewed

Bullet list of changed files.

#### Project rule findings

Grouped by rule file (one group per file in `rules/`, plus `CLAUDE.md`/`AGENTS.md` if present). Each item must cite file path, line range, and the specific rule violated. If none, write "None."

#### React best-practice findings

Grouped by checklist category (Components & hooks, State & effects, Data fetching, Rendering, Clean code, Accessibility). Each item tagged with impact level. If none, write "None."

#### Suggested fixes

Minimal, focused recommendations that preserve behavior.

#### Slop signals detected

Short bullets if any of these are present in the diff:

- Extra comments unnecessary or inconsistent with local style
- Defensive checks or try/catch blocks abnormal for trusted code paths
- Casts to `any` used only to bypass type issues
- Deeply nested code fixable with early returns
- Other patterns inconsistent with the file and surrounding codebase

If none, write "None."

### 5. Conditional deslop subagent

After delivering the report, evaluate **Slop signals detected**.

**If at least one slop signal is present**, spawn a subagent:

```
Task tool:
  subagent_type: generalPurpose
  description: Remove AI code slop
  run_in_background: false
  prompt: |
    Remove AI-generated slop from the working tree.

    Focus areas:
    - Extra comments that are unnecessary or inconsistent with local style
    - Defensive checks or try/catch blocks that are abnormal for trusted code paths
    - Casts to `any` used only to bypass type issues
    - Deeply nested code that should be simplified with early returns
    - Other patterns inconsistent with the file and surrounding codebase

    Guardrails:
    - Restrict edits to the files and patterns flagged in the review below.
    - Keep behavior unchanged unless fixing a clear bug.
    - Prefer minimal, focused edits over broad rewrites.

    Slop signals to address:
    [paste slop signals from report]

    Affected files:
    [paste file paths from report]

    Return a 1-3 sentence summary of what was cleaned up.
```

**If no slop signals are present**, state: **No slop signals detected — skipping deslop.** Do not spawn the subagent.

## Guardrails

- Cite file paths and line ranges for every finding.
- Do not flag added or removed `useMemo`/`useCallback`/`React.memo` without evidence — first check whether the project has React Compiler configured (e.g. in `babel.config.js`, `next.config.ts`, or app config). If it does, manual memoization is usually noise; if it does not, removals may regress performance.
- Do not speculate stale closures or performance regressions — show the stale read path, a repro, or profiler evidence.
- Do not treat component tree depth or count as primary performance evidence.
- Check library versions in the project's lockfile or `package.json` before API-specific suggestions.
- Keep the final report concise and skim-friendly.
- The parent agent does not edit code itself during review; edits happen only via the deslop subagent or explicit user follow-up.
