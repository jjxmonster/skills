---
name: blast-radius
description: Blast-radius check for a code change in a JavaScript/TypeScript repo (single app or monorepo). Finds what the change affects outside its own diff — broken consumers, changed contracts, and coupling that a grep for the symbol name misses (string keys, changed semantics, persisted data, external consumers). Use whenever the user asks what a change breaks, affects, touches or impacts, "did I break anything", "what depends on this", "is this safe to merge", or wants a regression check before a PR. Also use proactively before declaring a change finished when the diff touched an exported signature, a return type, a default, a string used as a key, a schema, or anything serialised or persisted.
---

# Change Impact

Find what a change affects **outside the diff**. Type checkers and tests cover most in-repo breakage; this skill covers what they miss: coupling through strings, network and storage boundaries, and unchanged signatures whose meaning changed.

The reader wrote the diff minutes ago. Do not describe the change. Report what follows from it elsewhere.

Two rules govern everything below:
- Report only what is reachable from the change. An empty report is a valid result; say so plainly.
- Never report clean when you could not look. Coupling the repo cannot see is a `BLIND` finding, not silence.

## Step 0 — Establish the baseline

Check whether type-check, tests and lint have been run on the analysed tree (look for recent output, CI status, or run them if cheap). Record the result in one line of the report. If TypeScript passes, channel A below is covered except for `.js` files, `any`, dynamic imports and barrel re-exports — spend the effort on channels B–E instead. Do not report anything `tsc` or an existing test already catches.

## Step 1 — Resolve scope

State the scope in the report.

| Argument | Scope |
|----------|-------|
| *(none)* | `main...HEAD` plus uncommitted changes |
| single commit (`3fa3fa`, `HEAD~3`) | `<rev>^!` |
| contains dots (`a..b`, `a...b`) | passed to git unchanged |
| branch name | `<branch>...HEAD` |
| `--staged` / `--dirty` | index only / working tree only |

- Uncommitted changes are included only in default mode; an explicit revision is a question about history.
- `main` missing → try `master`, `trunk`, then `origin/HEAD`. If none resolve, ask; never guess.
- On `main` with a clean tree → report `nothing to analyse`. This is not `no impact`.
- Merge commit → diff against the first parent.
- Use `git diff -M` so moved files do not read as removed exports.
- For an explicit historical revision, search and read the analysed tree, not the disk: `git grep <pattern> <rev>` and `git show <rev>:<path>`. Line numbers in the report refer to `<rev>`.

## Step 2 — Extract contracts, not hunks

Start with `git diff --stat`; read hunks per file. From them extract only **contracts that cross a file boundary**:
- exported symbol: name, path, signature
- return type — especially becoming nullable, `undefined`, `Promise`, or a union
- whether it throws, and what
- a default parameter or default config value
- a string literal used as a key (channel B)
- the shape of anything serialised, persisted, or sent over a boundary
- unit, format, timezone, ordering, limit
- side effects: argument mutation, sync → async

Hunks that change none of these are internal: drop them, search nothing. Expect to discard most hunks; discarding none means you are analysing lines, not contracts.

Diff patterns that signal a **semantic** change behind an unchanged name (channel C): edited `return` / `throw` statements, changed default values, `.sort` / `.slice` / `.reverse` / comparators, arithmetic on money or durations, `Date`/timezone handling, `async` added or removed, `Object.assign` or spread replaced by in-place mutation.

Cap at **5 contracts**, prioritising channels B, C, D over A. List the ones you skipped:

```
+ 4 more contracts not analysed: <names>
```

## Step 3 — Pick the channel, derive the search key

Choose the channel first; it decides what to grep for.

**A. Name is the key** — exported functions, components, types, props. Search the symbol name **and** the import path. Watch barrel re-exports under another name and renamed default imports. In a monorepo, search `@scope/package` plus subpath, not the relative path.

**B. A string is the key** — config and env names, feature flags, i18n keys, routes and params, query params, `localStorage`/cookie/cache keys, analytics events, `data-testid`, CMS/schema field names, dispatch maps (`registry[item.type]`). Search the **string value**, not the identifier holding it. A key assembled at runtime (`prefix.${name}`) cannot be found by grep → `BLIND`, never `none`.

**C. Name unchanged, meaning changed** — nullable return, stopped throwing, changed default, order, limit, unit, timezone, sync → async, new mutation. Nothing to grep for except the unchanged name; zero hits is the most dangerous result here. Find call sites, then read them against one question (Step 4).

**D. Beyond the repo** — always `BLIND`, even when every in-repo consumer was updated:
- data that outlives the deploy: rows and JSON columns, cached shapes, in-flight queue messages, `localStorage` in users' browsers, CDN responses, externally authored content
- consumers over a boundary: an API response shape read by another service, a mobile app or a separate frontend; a package published to npm

**E. Build and configuration** — env vars, `tsconfig` paths, `package.json` `exports`, bundler aliases, design tokens, workspace deps. Generated code (GraphQL codegen, ORM clients): if the schema changed and the artefact was not regenerated, the mismatch is real even if types pass.

**F. Order and lifecycle** — middleware order, provider nesting, import-time side effects. Only when the diff touches a file that exists purely to compose things.

## Step 4 — Read call sites against one question

Form one question per contract from the kind of change and ask it unchanged at every site:
- became nullable → dereferenced without a guard?
- stopped throwing → a `try`/`catch` or `.catch()` relies on the throw?
- unit changed → multiplied, formatted, or compared to a literal?
- order changed → takes `[0]`, `.find()`, or assumes the first element?
- became async → used without `await`?

Then decide. An unguarded dereference is `BROKEN`, not "worth a look". A site already handling the new behaviour drops out entirely. If everything lands in `CHECK`, you collected instead of deciding.

Limits:
- One hop. If a site wraps the value and passes it on, note the propagation and stop.
- More than ~15 call sites → read a representative sample and say so in the footer.
- Roughly 3 searches per contract.
- Stories, scripts, fixtures and e2e tests count as consumers only when they are the sole consumer or would fail loudly; otherwise omit them.

## Output

```
Scope     main...HEAD, 4 commits, +2 uncommitted files
Baseline  tsc clean, 212 tests pass
Impact    3 things outside your diff depend on what changed.

BROKEN  getProduct() now returns null on 404
        src/routes/product.tsx:52 does `product.title` with no guard.
        Read 6 call sites; the other 5 already handle null.

BROKEN  config key `api.timeout` → `api.timeoutMs`
        src/lib/http.ts:14 still reads the old key. Rename it.
        + 2 more under src/services

CHECK   getCart() no longer throws on empty
        src/routes/api.cart.ts:60 branches on the throw. Confirm the empty path.

BLIND   `hero.headline` is authored outside this repo
        Published entries still carry the old key; the repo can't check them.
        Migrate the content or accept both keys for one release.

Scanned 3 contracts, read 14 call sites, 2 sampled.
Searched: getProduct, `api.timeout`, `api.timeoutMs`, getCart, `hero.headline`
```

Nothing to report:

```
Scope     main...HEAD, 2 commits
Baseline  tsc clean, tests not run
Impact    none. Nothing outside your diff reads what changed.

Scanned 2 contracts, read 9 call sites.
Searched: formatPrice, `checkout.currency`
```

### Labels

- `BROKEN` — a specific location that does not work after the change, and you can name what fails.
- `CHECK` — genuinely coupled, but whether it breaks depends on intent you do not know. Not a parking space for unread sites.
- `BLIND` — the contract left what the repo can see. No location, by design.

### Rules

- One finding per changed contract, not per location. Header: the contract change, short, with an arrow where it fits.
- Effect: two lines maximum — what happens, then what to do.
- Locate the **affected** site, not the changed one. Fan-out: one representative location, then `+ N more` with a directory.
- The `Searched:` line lists the literal patterns you grepped. It is mandatory: it is what lets the reader see what was not searched, and what makes `none` trustworthy.
- The verdict describes reach, never approval. Do not write "safe" or "approved".

## Never report

Formatting, comments, local renames, added tests, dependency bumps that change no call site, anything `tsc` or an existing test already catches, and refactors contained in one file with the exported surface unchanged.
