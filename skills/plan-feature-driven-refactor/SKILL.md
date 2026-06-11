---
name: plan-feature-driven-refactor
description: Produces a complete, executable plan for restructuring a codebase from a monolithic layer-by-type layout (flat components/, hooks/, services/, utils/...) into feature-driven architecture (features/<feature>/ + shared/). Use when the user wants to refactor, restructure, or migrate an app from monolith to feature-based/feature-driven/vertical-slice architecture, or asks to "analyze the structure for refactoring". The output is a written plan; execution only starts after the user approves it.
---

# Plan a Monolith → Feature-Driven Refactor

Produce a migration plan that is safe to execute mechanically: every file accounted for, every step independently verifiable, zero logic changes. The plan is the deliverable — do not move files until the user approves it.

**Core principle: move-only.** A structure refactor changes directory layout and import specifiers, nothing else. Splitting large files, fixing pre-existing type errors, and deleting dead code are separate follow-up tasks. This constraint is what makes a no-test-suite migration safe: the type checker becomes a complete verification gate.

## Phase 1 — Understand the current architecture (read-only, parallel subagents)

Start by discovering what actually exists — assume nothing about the project's stack, layer dirs, naming, or conventions; everything below is illustrative, not a checklist of expected dirs. Launch 2 explore agents in parallel and have them report:

**Agent A — structural inventory:**
- Full directory tree with file counts per layer dir (`components/`, `hooks/`, `services/`, `queries/`, `utils/`, `constants/`, `lib/`, `store/`...)
- Routing/entry structure (router dirs, screens, pages) — these usually cannot move
- Every file with a one-line description and a guessed domain
- Distinct features/domains visible in the app
- Path aliases (`tsconfig.json` paths), existing barrels, platform-suffix file pairs (`.web.ts`, `.ios.tsx`, `.native.ts`)
- State management and where backend/API calls live

**Agent B — coupling analysis:**
- Which modules are imported by multiple unrelated screens (shared) vs exactly one (feature-local) — with concrete paths
- Import style consistency (alias vs relative)
- Infrastructure dependencies: what do the API client / store / root layout import? (This decides what must live in `shared/`.)
- Tests, lint, format gates and how they're configured

Then **verify the three load-bearing facts yourself** (don't trust the report):

1. Read the API-client/infrastructure file to confirm its imports.
2. Read the alias config to confirm the new top-level dirs are already covered (alias `@/* → ./*` usually means zero config changes).
3. Find **every config that enumerates source paths** — these break silently, the type checker never sees them: `tailwind.config.*` `content` globs (NativeWind/Tailwind), metro/babel `module-resolver` aliases, jest/vitest `roots`/`testMatch`/`moduleNameMapper`, lint ignore patterns, storybook globs, CI path filters. Record which need new `features/**` and `shared/**` entries. Also note the source dirs list (`$SRC_DIRS`) for the rewrite/verification greps.

## Phase 2 — Lock the four decisions with the user

Ask only after Phase 1, so every option is grounded in findings — name the actual domains discovered, with file counts, instead of hypotheticals (use AskUserQuestion or equivalent). Each answer changes the plan's shape:

1. **Strategy**: big-bang (one PR) vs incremental (one commit/PR per feature). Incremental means two conventions co-exist temporarily; big-bang means one large review.
2. **Large-domain granularity**: for each domain with many files (50+) found in Phase 1 — one feature with subfolders, or split into several per-domain features? Splitting gives smaller cohesive modules but duplicate file names across features.
3. **Barrel files**: per-feature `index.ts` public API vs direct full-path imports. For Metro/React Native, recommend **no barrels** (eager loading hurts startup). For tree-shaking bundlers it's a style choice.
4. **Scope**: move-only (recommended) vs move + split large files. Default hard to move-only.

## Phase 3 — Design the target structure

```
src-root/
├── app/ (or pages/, routes/)   # UNCHANGED — framework-owned routing; thin wrappers
├── shared/                     # cross-cutting; MUST NEVER import from features/
│   ├── lib/                    # API client, query client, app singletons
│   ├── components/ (+ ui/)     # design-system primitives used across features
│   ├── hooks/  constants/  utils/  types/
│   └── <infra-state>/          # e.g. auth session store — see ownership rules
└── features/<feature>/
    ├── components/             # UI (+ co-located alert/dialog helpers)
    ├── hooks/                  # screen/component logic
    ├── api/                    # query factories + mutation hooks + upload/side-effect services
    ├── utils/                  # pure helpers owned by the feature
    ├── types/                  # domain types owned by the feature
    └── constants/              # only create subfolders that will contain files
```

A flat legacy `types/` dir splits by the same importer rule as everything else: types used by one feature → that feature's `types/`; types imported by `shared/` code or by many features → `shared/types/`. Test files (`*.test.ts`, `__tests__/`) are co-located — they move with the module they test, in the same commit.

### Ownership rules for ambiguous files

- **Infrastructure trumps feature naming.** If `shared/lib/api-client.ts` imports the auth store (token injection, 401 handling), the *store* is infrastructure → `shared/`. The auth *feature* is only the sign-in/up/reset UI. Generalize: anything imported by `shared/` code must itself live in `shared/` — never let the dependency arrow point from `shared/` into `features/`.
- **A module used by exactly one feature belongs to that feature**, even if it currently sits in a generic dir. Verify by grepping importers, not by name.
- **A module used by 2–3 sibling features**: either give it to the primary consumer (document the cross-feature import) or extract a small "dependency feature" (e.g. `product-catalog` consumed by `cart`, `orders`, `search`) so the arrows stay one-directional. Prefer the dependency feature when 3+ features consume it.
- **A component used by both a "user" and an "admin" variant of a flow** (e.g. a selection grid used by both the user-facing and admin create screens): owned by the feature whose data it renders; the other imports cross-feature.
- **Aggregation endpoints** (a dashboard/profile query imported by several features): leave ownership with the aggregate feature, note the cross-feature imports, and flag "extract a narrower query" as a follow-up — do NOT restructure data fetching during a move-only refactor.
- **Template/dead code** (starter-kit leftovers with zero importers): move it to `shared/`, flag it, delete in a separate follow-up PR. Never delete during the move.
- **Route files never move** — they stay where the framework requires and only get their import specifiers rewritten. Routes may import from any feature.

### Conventions to write into the plan

- No barrels (if so decided); imports always full path via alias: `@/features/orders/components/order-card`.
- Cross-feature imports allowed but rare, full-path, acyclic at module level.
- `api/` merges the old query-options/mutations/services concepts; keep original file names verbatim.
- Duplicate file names across features (`orders-list.tsx` in `orders` and `admin-orders`) are acceptable; distinct old paths keep rewrites unambiguous.

## Phase 4 — File inventory

Map every existing file to a destination, pattern-level where a directory moves wholesale:

- `components/<domain>/*  →  features/<domain>/components/`  (wholesale)
- individually named hooks/queries/mutations/utils/constants per feature
- explicit list of what lands in `shared/`
- explicit "does NOT move" list: routes, env, config files, assets

**Accounting check:** after all steps, every legacy layer dir must be empty and deleted. If a listed file doesn't exist at execution time, the inventory was wrong — verify and continue, don't invent a placeholder.

## Phase 5 — Migration order

1. **Step 1: infrastructure** (`shared/lib` + the infra store) — smallest, most load-bearing; establishes the dependency-direction rule. This step also updates every path-enumerating config found in Phase 1 (tailwind `content`, jest roots/moduleNameMapper, metro/babel resolver...) to cover `features/**` and `shared/**` — up front, not in cleanup.
2. **Step 2: the rest of `shared/`** (theme, primitives, generic hooks/utils) — the largest diff (touches nearly every file) but purely mechanical; doing it early prevents rewriting the same import lines twice in later steps.
3. **Then features, smallest/least-coupled → largest**, dependencies before dependents: self-contained clusters first (typically things like settings, notifications, onboarding, billing), then core domains, with each dependency feature moved BEFORE the features that import it, and the heaviest most-coupled domains last.
4. **Final step: cleanup** — delete empty legacy dirs, update docs/rules/skills, full verification.

Each step must leave the repo compiling and lint-clean, in its own commit.

## Phase 6 — Mechanics (embed verbatim in the plan)

### Baseline first

```bash
git checkout -b refactor/feature-driven
npx tsc --noEmit   # or the project's typecheck command
```

If the baseline is NOT green, do not fix the errors (logic change). Record them and gate on "no new errors":

```bash
npx tsc --noEmit --pretty false 2>&1 | sed 's|^[^(]*/\([^/(]*(\)|\1|' | sort > /tmp/tsc-baseline.txt   # strip dirs, keep filename(line,col)
# after each step:
npx tsc --noEmit --pretty false 2>&1 | sed 's|^[^(]*/\([^/(]*(\)|\1|' | sort | diff /tmp/tsc-baseline.txt -
```

Path-normalize because moved files change error paths, and an error *message* that quotes a module specifier (`Module '"@/hooks/x"' declares...`) will legitimately change when that module moves — inspect such diffs manually, then refresh the baseline. Caveat: stripping dirs makes duplicate file names across features (allowed by convention) collide in the diff — when a diff line involves a name that exists in several features, check the full un-normalized output before trusting it.

### Per step

1. **`git mv`** (preserves history; rename detection survives the small import edits inside moved files). `git mv` does NOT create missing parent directories — always `mkdir -p` the destination parent first:
   ```bash
   mkdir -p features/orders && git mv components/orders features/orders/components   # wholesale dir
   mkdir -p features/orders/hooks && git mv hooks/use-orders.ts features/orders/hooks/
   ```
2. **Rewrite imports — anchored to avoid prefix collisions** (`use-order` vs `use-order-detail`, `orders` vs `order-detail`):
   - directory-wide patterns end with `/` and are replaced as-is
   - single-module patterns are anchored with the closing quote
   - grep over the source dirs identified in Phase 1 (`$SRC_DIRS` below) — do NOT assume `src/` exists; in Expo/Next apps source dirs often sit at the project root (`app/ components/ features/ shared/ ...`)
   ```bash
   grep -rl '@/components/orders/' $SRC_DIRS | xargs sed -i '' 's|@/components/orders/|@/features/orders/components/|g'
   grep -rl '@/hooks/use-orders"' $SRC_DIRS | xargs sed -i '' 's|@/hooks/use-orders"|@/features/orders/hooks/use-orders"|g'
   ```
   (macOS sed needs `-i ''`. If you script this with a generic pair-list helper, do NOT also put the quote in the pattern — it gets double-appended and silently matches nothing.)
3. **Verify**: typecheck diff vs baseline + lint. The type checker is the real gate; it catches every broken specifier including relative imports that now point across feature boundaries.
4. **Fix surfaced relative imports** by converting them to alias paths pointing at the file's *current* location — a later step's directory-wide rewrite will pick them up automatically.
5. **Commit**: `refactor(<scope>): move <feature> into features/<feature>` — one commit per step, every intermediate state compiles.

### Hard-won pitfalls (include in the plan's risk section)

- **Platform-suffix pairs** (`use-color-scheme.ts` + `.web.ts`, `icon-symbol.tsx` + `.ios.tsx`) must move in the same commit. The type checker will NOT catch a split pair — the bundler silently resolves the wrong module.
- **Tailwind/NativeWind `content` globs** typically hardcode the old dirs (`"./components/**"`, `"./constants/**"`). Files moved to `features/`/`shared/` silently stop generating classes — typecheck, lint AND the bundler export all pass; only the manual styling smoke test catches it. Add `"./features/**/*.{js,jsx,ts,tsx}"` and `"./shared/**/*.{js,jsx,ts,tsx}"` in migration step 1, before any file lands there.
- **Relative asset `require()` / dynamic import strings** (`require("../../assets/x.png")`) are not resolved by tsc (wildcard `*.png` declarations) — they break only at runtime. Wholesale dir moves keep internal relative paths intact, but paths escaping the moved dir change depth. Before committing a step, grep the moved files for `require("..` and `import("` with relative strings.
- **Empty leftover directories can break tooling**: some linters (`expo lint`) enumerate top-level dirs and crash on a dir with no matching files. Delete legacy dirs as soon as they're empty.
- **Inventory drift**: exploration reports occasionally list files that don't exist (already deleted) — `git mv` failing with "bad source" on one file shouldn't abort the step; verify state and continue.
- **Formatting**: path rewrites via sed don't reformat, so the formatter normally isn't needed. If Phase 1 surfaced formatter quirks (e.g. non-default flags required to preserve the project's indentation style), note them in the plan.
- **Pre-commit hooks** may run repo-wide formatters — expect them, don't fight them.

### Final verification

```bash
npx tsc --noEmit --pretty false                    # matches baseline
npm run lint                                       # green
grep -rn '@/components/\|@/hooks/\|@/services/\|@/utils/\|@/constants/\|@/lib/' $SRC_DIRS  # zero hits — build the pattern from the legacy layer dirs found in Phase 1
# the project's full bundler/build pass, e.g.:
#   npx expo export --platform ios | next build | vite build | npm run build
# — proves module resolution (incl. platform pairs) without launching anything
```

Plus a manual smoke test of every main screen before merge — bundling proves resolution, not runtime behavior.

## Phase 7 — Update the knowledge layer

Grep for hardcoded old paths OUTSIDE the source tree and update them in the same migration:

- `CLAUDE.md` / `AGENTS.md` (project root AND app-level) — rewrite the architecture section with the new layout, import rules, and a feature list with ownership notes for the ambiguous cases
- editor rules (`.cursor/rules/*.mdc`, `.claude/rules/`) — keep duplicate copies in sync
- agent skills / scripts / CI jobs that hardcode file paths (e.g. a release-notes script pointing at a moved `constants/` file)

## Plan document template

```markdown
# Refactor <app>: monolith → feature-driven architecture

## Context            — why; current file counts per layer dir; the four locked decisions
## Verified facts     — alias coverage, path-enumerating configs (tailwind content, jest, metro/babel), infra dependencies, test/lint/format gates, baseline state
## Target structure   — tree for features/ and shared/ + conventions + dependency rules
## Feature inventory  — per feature: wholesale dir moves + named files, ambiguous cases resolved with rationale
## Does NOT move      — routes, config, assets; legacy dirs that must end empty
## Migration order    — numbered steps with rationale (infra → shared → small → large → cleanup)
## Step mechanics     — git mv + anchored rewrite + verify + commit, with exact commands
## Risks              — platform pairs, sed collisions, tailwind content globs, relative asset requires, large step-2 diff, no-test caveat
## Docs update        — CLAUDE.md, rules, skills with hardcoded paths
## Final verification — typecheck, lint, legacy-import grep, bundler export, manual smoke list
```

## Anti-patterns

- Fixing type errors, renaming exports, or "improving" code mid-move — breaks the move-only guarantee that makes verification trivial.
- `mv` instead of `git mv`, or moving + rewriting in separate commits — kills rename detection and `git log --follow`.
- Un-anchored search/replace (`s|@/hooks/use-order|...|`) — corrupts `use-order-detail` imports.
- Creating empty subfolders "for consistency".
- Deleting dead code during the move (separate PR).
- A `shared/` module importing from `features/` — the one inviolable rule.

## Post-migration hardening (follow-up, separate PR)

Prose rules erode; recommend enforcing them in lint once the move lands:

- ESLint `import/no-restricted-paths` (zones: `shared` cannot import `features/*`; optionally `features/*` cannot import other features except documented dependency features) or `eslint-plugin-boundaries`.
- `dependency-cruiser` in CI for cycle detection across features.

List this in the plan as a named follow-up — never bundle it into the move itself.
