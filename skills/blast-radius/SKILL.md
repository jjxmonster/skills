---
name: blast-radius
description: Analyse what a code change affects outside its own diff — broken consumers, changed contracts, and coupling that the type checker, linters and greps for symbol names will miss. Use this whenever the user asks what their change breaks, affects, touches, or impacts; when they ask "did I break anything", "what depends on this", "is this safe to merge", or ask for a blast-radius or regression check before a PR. Also use it proactively before declaring a non-trivial code change finished, and after any rename, signature change, return-type change, API or schema edit. Applies to JavaScript and TypeScript repositories, single-app and monorepo. Not a code-quality review — it reports reach, not style.
---

# Change Impact

Find what a change affects **outside the diff**.

The type checker, tests and linters already cover most in-repo breakage. This skill exists for what they miss: coupling that travels through strings, network boundaries, persisted data, clients that are not redeployed, and unchanged signatures whose meaning changed. That is where an agent that finishes on green tests ships a regression.

The report goes to a developer who wrote the diff minutes ago. Do not describe what they changed. Report what follows from it elsewhere.

## Core commitment

Report only what is actually reachable from the change. An empty report is a valid and useful result — say so plainly instead of manufacturing findings. A report of five vague "might be affected" items is worse than nothing, because after a week nobody reads it.

Equally: never report clean when you could not look. If the coupling runs through a channel this repo cannot see, that is a finding (`BLIND`), not silence.

## Step 0 — Delegate what the tools already cover

Before searching anything, check what the repo can verify mechanically:

- If the project has TypeScript, run `tsc --noEmit` (or the project's `typecheck` script) on the analysed tree. Note whether `strict` / `strictNullChecks` is on.
- If there is a fast test command, run it.

Anything these catch is **not a finding for this skill** — the developer will see it in CI. In particular, with `strictNullChecks` on, a return type becoming nullable and being dereferenced in a `.ts` caller is `tsc`'s job, not yours. Report it only where the type checker's reach ends: `.js` callers, values that pass through `any`, `JSON.parse`, `as` casts, or a package that is not part of the same type-check run.

If you cannot run the type checker (no toolchain, remote tree), say so in the footer and treat channel A as unverified rather than assuming green.

## Step 1 — Resolve scope

The result depends entirely on what "the change" means, so resolve it explicitly and state it in the report.

| Argument | Scope |
|---|---|
| *(none)* | `main...HEAD` plus uncommitted changes |
| `3fa3fa`, `HEAD~3`, `HEAD` | that single commit (`<rev>^!`) |
| `abc..def`, `abc...def` | pass to git unchanged |
| a branch name | `<branch>...HEAD` |
| `--staged` | index only |
| `--dirty` | working tree only |

Rule: an argument that resolves to a single commit means that commit; one that resolves to a branch means `...HEAD`; anything containing dots goes to git as given.

Uncommitted changes are included **only** in default mode. An explicit revision is a question about history, not about the working tree.

Edge cases:
- `main` missing → try `master`, then `trunk`, then the default branch from `origin/HEAD`. If none resolve, stop and ask for an explicit scope. Never guess silently.
- On `main` with a clean tree → report `nothing to analyse`. This is not the same as `no impact`, and conflating them is exactly the false reassurance this skill exists to prevent.
- A merge commit → diff against its first parent, otherwise the whole merged branch reads as one commit's changes.
- Unknown revision or shallow clone → fail with a clear message.

**Search the analysed tree, not the working tree.** For an explicit historical revision, run searches as `git grep <pattern> <rev>` where `<rev>` is the end of the analysed range. Reading files from disk instead will report fixes that came later as breakage, or miss breakage that was real at the time. In default mode `<rev>` is the working tree and normal file reads are fine.

## Step 2 — Extract contracts, not changed lines

This step is what keeps the report small. Do not analyse hunks. From the diff, extract only the **contracts that crossed a file boundary** — the things another file, service, store or client could depend on.

A change is a contract change when it alters any of:
- an exported symbol's name, path, or signature
- a return type, especially becoming nullable, `undefined`, `Promise`, or a union
- whether something throws, and what it throws
- a default parameter or default config value
- a string literal that acts as a key (see channels below)
- the shape of anything serialised, persisted, or sent over a boundary
- a unit, format, timezone, ordering, or limit
- a side effect: mutation of an argument, sync becoming async

If a hunk changes none of these, it is internal. Drop it and search nothing. Most diffs are mostly internal — expect to discard the majority of hunks here. A run where you discarded nothing means you are analysing lines rather than contracts.

**Worked example** — a diff with 9 hunks:

```
src/lib/pricing.ts    reorder imports                         internal, drop
src/lib/pricing.ts    rename local `tmp` → `subtotal`         internal, drop
src/lib/pricing.ts    formatPrice() now takes cents, not EUR  CONTRACT: unit changed
src/lib/pricing.ts    add JSDoc                               internal, drop
src/lib/cart.ts       getCart() returns [] instead of throwing CONTRACT: stopped throwing
src/lib/cart.ts       extract helper `mergeLines()` (unexported) internal, drop
src/config.ts         `api.timeout` → `api.timeoutMs`         CONTRACT: string key changed
src/config.ts         reformat object                         internal, drop
src/lib/cart.test.ts  add tests                               internal, drop
```

9 hunks → 3 contracts. Everything below runs against those three only.

Cap at **5 contracts**. Past that, report the overflow as a signal in itself:

```
+ 4 more contracts changed — this diff is too broad to analyse
```

## Step 3 — Pick the channel, derive the search key

Grepping the symbol name is the default instinct and it is right for exactly one channel. For everything else the search key is something else entirely. Choose the channel first, then search.

### A. Name is the key

Exported functions, components, types, props. `git grep` and find-references work — and so does `tsc`, which is why Step 0 usually clears most of this channel.

What `tsc` does not clear: barrel files re-exporting under another name, default exports renamed at the import site, `.js` consumers, dynamic `import()` with a string path, and packages outside the current type-check run. Search key: the symbol name **and** the import path.

### B. A string is the key

The highest-value channel, and the one nothing else covers.

Config and env var names, feature flag keys, i18n keys, route paths and params, query params, `localStorage` / cookie / cache keys, analytics event names, `data-testid`, CMS or schema field names, and dynamic dispatch maps (`registry[item.type]`).

Search key: **the string value**, not the identifier holding it.

If the key is assembled at runtime (`prefix.${name}`), grep cannot find consumers by construction. Search for the prefix and for the construction site to bound the problem, then report `BLIND`, never `none`. Recognising that the tool does not reach is part of the job.

### C. The name is unchanged, the meaning changed

Nothing to grep for, and the type checker sees no difference, which is why zero hits here is the most dangerous result in this skill.

**How to detect it from the diff:** for every function, method or exported value whose body changed but whose signature did not, ask one question — *could the same input now produce a different output, throw differently, or have a different side effect than before this diff?* If yes, it is a channel-C contract. Typical answers: return type became nullable; a function stopped throwing and now returns `null` or `[]`; a default changed; sort order, pagination limit, unit (cents vs whole units), timezone; sync became async; a function now mutates its argument.

Method: find call sites of the **unchanged** name, then read each one against a specific question (Step 4).

### D. Data and clients that outlive the deploy

Rows and JSON columns already written, cache keys and cached shapes, in-flight queue messages, `localStorage` in users' browsers, CDN responses, content authored in an external system.

Also **clients you do not deploy**: mobile app versions already installed, a published npm package's consumers, webhook receivers, partners integrating against an OpenAPI contract. The data may be fine; the old code is still out in the world.

Rule: a shape change to anything in this channel is `BLIND` by default, even when every in-repo consumer was updated — old data and old clients still carry the old shape and the repo has no way to check them.

Downgrade or drop the finding only when the diff itself shows the handling: a migration or backfill for the persisted data, a versioned key with a fallback, a reader that tolerates both shapes (a parser with defaults, an `oneOf`), or an API version that keeps the old contract alive. If the handling exists outside the diff (a migration run last week), that is the developer's knowledge, not yours — keep the `BLIND` and say what would clear it.

### E. Build and configuration

Env vars, `tsconfig` paths, `package.json` `exports`, bundler aliases, design tokens, workspace dependencies.

Also generated code — GraphQL codegen, ORM clients, generated types. Ask whether the artefact was regenerated; if not, the mismatch is real despite types passing.

In a monorepo, a change inside a package is a public contract change toward every package importing it. Search key: `@scope/package-name` plus subpath, not a relative path.

### F. Order and lifecycle

Middleware order, provider nesting, import-time side effects, registration order. Do not hunt for this proactively — only when the diff touches a file that exists purely to compose things.

## Step 4 — Read call sites against one question

Reading is for **deciding**, not collecting. Form the question once per contract from the kind of change, then ask it unchanged at every call site:
- became nullable (and `tsc` does not cover the caller) → is the result dereferenced without a guard?
- stopped throwing → is there a `try`/`catch` or `.catch()` relying on the throw?
- unit changed → is the value multiplied, formatted, or compared to a literal?
- order changed → does anything take `[0]`, `.find()`, or assume the first element?
- became async → is the result used without `await`?
- key renamed → does the site still read the old string?

Then resolve, and resolve properly: an unguarded dereference is `BROKEN`, not "worth a look". A site already handling the new behaviour drops out of the report entirely. If you finish reading and everything lands in `CHECK`, you collected instead of deciding.

Stopping rules, because depth is what explodes:
- **One hop.** If a call site wraps the value and passes it on, note the propagation and stop. Do not walk the call graph.
- More than ~15 call sites for one contract → read a representative sample and say in the footer that it was a sample. A silent truncation turns "no findings" into a lie.
- Search budget is per run, not per contract: roughly **15 searches total**. Channel A typically needs 3–4 (name, import path, barrel, renamed default); channel B often needs 1; channel C needs one search and then reading. Spend accordingly, and say in the footer if you ran out.

## Output

One scope line, one impact line, then findings grouped by cause. Always end with the footer.

The unit of a finding is **the changed contract**, not the location. One cause that breaks twelve places is one finding, not twelve — otherwise a single rename floods the report and the developer has to reassemble the cause themselves.

The `Impact` line counts **every finding including `BLIND`** — a `BLIND` is something outside the diff that depends on what changed; the repo just cannot show it. The footer's "scanned N contracts" counts every contract extracted in Step 2, including those that produced no finding.

```
Scope   main...HEAD, 4 commits, +2 uncommitted files
Impact  4 things outside your diff depend on what changed.

BROKEN  formatPrice() now takes cents, not EUR
        src/routes/checkout.tsx:88 still passes `order.total` (EUR). Prices render ×100.
        Read 7 call sites; the other 6 were updated in this diff.

BROKEN  config key `api.timeout` → `api.timeoutMs`
        src/lib/http.ts:14 still reads the old key. Rename it.
        + 2 more under src/services

CHECK   getCart() returns [] instead of throwing on empty
        src/routes/api.cart.ts:60 has a catch that redirects to /shop. Confirm the empty path still redirects.

BLIND   `hero.headline` is authored outside this repo
        Published CMS entries still carry the old key and the repo can't check them.
        Migrate the content or accept both keys for one release.

Scanned 4 contracts, read 19 call sites, 1 contract sampled. tsc: clean (strict).
```

Nothing to report:

```
Scope   main...HEAD, 2 commits
Impact  none. Nothing outside your diff reads what changed.

Scanned 2 contracts, read 9 call sites. tsc: clean (strict).
```

### Labels

- `BROKEN` — a specific location that does not work after this change, and you can name what fails there.
- `CHECK` — the location is coupled to the change and behaves differently now, but whether the new behaviour is *wanted* there depends on intent only the developer has. Nothing crashes; something changed.
- `CHECK` is not a parking space for sites you did not read.
- `BLIND` — the contract left what the repo can see. There is no location to point at; the absence of one is the finding.

### Rules

- Cause header: the contract change itself, short, with an arrow where it fits. Meant to be recognised, not read.
- Effect: two lines maximum — what happens, then what to do about it. The action may be "confirm X still works"; not every impact is a bug.
- Locate the **affected** site, not the changed one. Pointing outside the diff is the entire purpose.
- Fan-out: one representative location, then `+ N more` with a directory.
- `BLIND` has no location, by design.
- The verdict describes reach, never approval. This skill can miss things; "approved" would give false confidence exactly where it costs most.
- The footer is not decoration — it tells the developer what was *not* reported and why, which is what makes `none` trustworthy. Always include: contracts scanned, call sites read, whether any contract was sampled, whether `tsc` ran and with what strictness, and whether the search budget ran out.

## Never report

Formatting, comments, local variable renames, added tests, dependency bumps that change no call site, refactors contained entirely within one file that leave the exported surface unchanged, and anything `tsc` or the test run already flagged.

Asked to find impact, a model will always find some. This list is what keeps the report worth reading.