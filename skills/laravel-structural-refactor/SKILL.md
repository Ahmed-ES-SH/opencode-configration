---
name: laravel-structural-refactor
description: 'Plan and execute structural-only reorganization of a Laravel app into domain/feature modules (app/Modules/DomainName/...) — moving controllers, models, requests, resources, services, DTOs, events, listeners, jobs, mail, policies, factories, and tests into cohesive domain folders while preserving all existing behavior. Use whenever the user asks to restructure, reorganize, split by domain/feature, move code into modules, clean up app/Http/Controllers, or "refactor" a Laravel project meaning the folder/namespace layout, not the code inside files — even if they just say "refactor this" or reference AGENTS.md, app/Modules, or a domain name like "move Article stuff into its own module." Runs as two gated phases: a read-only PLAN saved to markdown and requiring approval, then APPLY, which performs the moves and hands off to clean-code-guard to review the diff. Do NOT use for code-quality fixes (N+1s, overengineered layers, naming) with no reorganization requested — that is a separate concern.'
---

# Laravel Structural Refactor

Reorganize a Laravel codebase from technical-type folders (`app/Models`,
`app/Http/Controllers`, `app/Services`, ...) into domain/feature modules
(`app/Modules/<Domain>/...`) — **without changing what the application
does.** A structural refactor that quietly changes a query, a validation
rule, or an API response isn't a successful refactor, it's a regression
wearing a disguise. Every instruction below exists to protect that
boundary.

## The one rule everything else follows

> **Move code, don't rewrite it.** Change a namespace or an import only
> because the file moved — never because you saw a way to improve it.

If you're ever unsure whether a change is structural or behavioral,
treat it as behavioral: leave it alone and report it instead of guessing.
See `references/hard-rules.md` for the full allowed/forbidden list — read
it before Phase 2 on any real codebase, since it covers cases (service
provider bindings, factories, OpenAPI annotations, queued jobs) that are
easy to miss.

## Two-phase gate — never skip the gate

This workflow is deliberately split into two phases with a hard stop
between them. A file-moving, namespace-rewriting change across a real
codebase is exactly the kind of thing that should not run unattended:
the plan is cheap to get wrong and expensive to unwind once files have
moved, so the person doing the asking should see the move table before
anything happens.

1. **Plan** (read-only) — inspect the codebase, propose a module
   structure, save it to a markdown file, and stop. Do not move, edit,
   or create any application files in this phase.
2. **Apply** — only after the user approves the plan (or hands you an
   already-written plan file and asks you to execute it directly) —
   perform the moves, verify, and hand off to `clean-code-guard` for a
   review pass.

**Skipping straight to Apply is only correct when the user gives you an
existing plan file (or an explicit, already-approved move table) and
asks you to implement it.** If they say "refactor the Article
controllers into a module" with no plan yet, you start at Phase 1 every
time, even if they sound like they're in a hurry — the gate is what
makes the eventual `git diff` reviewable in isolation instead of a
surprise.

---

## Phase 1 — Plan

### Step 0: Pre-flight

Run `git status` on the repo. If there are uncommitted changes already
sitting in the area you're about to touch, say so and ask the user to
commit or stash first — your eventual diff needs to be reviewable on its
own, not mixed in with unrelated in-progress work.

### Step 1: Inspect

Find every file related to the target domain (e.g. "Article",
"Payment") in the *actual current repository* — don't assume any example
structure matches this project. Look in (at minimum):

```
app/Models/            app/Events/          routes/
app/Http/Controllers/  app/Listeners/       tests/
app/Http/Requests/     app/Jobs/            database/
app/Http/Resources/    app/Mail/            composer.json
app/Http/Services/     app/Policies/
app/DTOs/              database/factories/
```

Also check model relationships, controller/service dependencies, route
references, event↔listener wiring, job references, and service-provider
bindings — these are what tell you which classes travel together.

### Step 2: Map

Decide the domain boundary: which classes clearly belong to the target
domain, and which are shared infrastructure that must stay put. Use this
test — it resolves most edge cases:

- **Belongs in the module** if it's specific to one business domain
  (e.g. `ArticleController`, `ArticleResource`, `StoreArticleRequest`,
  `ArticlePublished` event).
- **Stays global** if it's shared infrastructure used across domains
  (middleware, providers, helpers, console commands, framework
  integrations).

When a class is used by more than one domain, don't split it or
duplicate it to force a fit — flag it as shared/global instead of
guessing which module "owns" it.

### Step 3: Propose

Lay out the exact target structure: every file's current path → new
path → new namespace, following the `app/Modules/<Domain>/<Type>/...`
shape (see `references/plan-template.md` for the full layout convention
and a worked example). Every module you propose gets its own
`app/Modules/<Domain>/routes/api.php` — see "Module routes" below for
what goes in it and how it gets wired into the app.

### Step 4: Validate

For every file you propose to move, list what references it — imports,
route files, service-provider bindings, tests, factories, OpenAPI
annotations — and confirm the move won't break any of them. If a
reference is ambiguous (e.g. a facade or dynamically-resolved class),
flag it rather than guessing.

### Module routes

Endpoints move with their domain too — a module isn't complete if its
controllers live in `app/Modules/Article/` but its routes are still
buried in one monolithic `routes/api.php` alongside forty other
domains. Each module gets:

```
app/Modules/<Domain>/routes/api.php
```

containing exactly the routes that belong to that domain, moved
verbatim from the root `routes/api.php` — same URI, HTTP verb, route
name, middleware, and controller reference. Nothing about how a route
resolves should change, only which file it's declared in.

**How the root file finds them:** the root `routes/api.php` becomes a
thin aggregator. The default, recommended way is an explicit `require`
per module:

```php
// routes/api.php
require __DIR__.'/../app/Modules/Article/routes/api.php';
require __DIR__.'/../app/Modules/Organization/routes/api.php';
```

This is preferred over auto-discovery because it's visible at a glance
which modules are wired in, matches how Laravel developers already read
a routes file, and doesn't depend on every module following a naming
convention perfectly. Nothing in `bootstrap/app.php` (or
`RouteServiceProvider` on older Laravel versions) needs to change —
the root `routes/api.php` is still the single entry point the framework
already loads, it just delegates outward now.

If the project has many modules and the user explicitly wants
zero-touch registration (no edit to the root file every time a module
is added), a glob-based loader is a reasonable alternative — mention it
as an option rather than defaulting to it:

```php
foreach (glob(app_path('Modules/*/routes/api.php')) as $file) {
    require $file;
}
```

**Order matters.** Laravel matches routes in registration order, so if
two domains define overlapping URI patterns (e.g. both touch
`/api/items/{id}` in a way where order decides which wins), moving them
into separate files and reordering the `require` lines can silently
change which route handles a request even though every route itself is
untouched. Preserve the original relative order between routes you
move whenever they could plausibly overlap. If you spot a real overlap,
flag it in the plan under "Uncertain / needs a decision" rather than
guessing at safe ordering.

Record the routes-file move as its own row in the plan's move table
(see `references/plan-template.md`), and note in "References to update
per file" that the root `routes/api.php` needs the corresponding
`require` line added.

### Save the plan and stop

Write the plan to `plans/<domain-slug>-refactor-plan.md` (create the
`plans/` folder at the repo root if it doesn't exist yet) using the
template in `references/plan-template.md`. Then stop — do not proceed to
Phase 2 in the same turn. End your message with exactly this line:

> **Plan saved to `plans/<domain-slug>-refactor-plan.md`. Review the
> move table above — if it looks right, tell me to go ahead (or run this
> skill again with "implement the plan") and I'll apply it.**

Also report, outside the move table: anything you're uncertain belongs
in the module, and any pre-existing bugs or code-quality issues you
noticed along the way — report only, do not fix them here. Bug fixes and
code-quality cleanup are a different task from structural reorganization
and mixing them in defeats the purpose of a reviewable diff.

If the target doesn't map cleanly onto a single domain, say so and ask
which specific controllers/models/services to include rather than
guessing at the boundary.

---

## Phase 2 — Apply

Enter this phase only when the user has approved a plan you wrote, or
handed you a plan file directly and asked you to execute it. Re-read the
plan file first — don't rely on memory of a plan from earlier in a long
conversation, since the approved version is the source of truth.

### Move, file by file

For each file in the plan's move table:

1. Move the file to its new path.
2. Update its namespace to match the new path.
3. Search the codebase for references to its old namespace/class name.
4. Update imports, route references, and service-provider bindings that
   point at it.
5. Update tests/factories that reference it, if any.

Keep the diff to exactly this — namespace and import lines changed by
the move. Don't reformat the file, reorder imports beyond what moved,
or touch anything else in it. A large, hard-to-review diff defeats the
purpose of doing this as a gated, inspectable change.

### Move the module's routes

Before touching anything, capture a baseline: `php artisan route:list
--json > /tmp/routes-before.json` (or plain `route:list` output if
`--json` isn't available). This is what you'll diff against once the
split is done — routes are the one thing where "looks right" isn't
good enough, since a silently reordered or dropped route fails at
request time, not at review time.

Then, for the routes in scope:

1. Create `app/Modules/<Domain>/routes/api.php` and move the domain's
   route definitions into it verbatim — same URI, verb, name,
   middleware, controller reference, and relative order.
2. Remove those same route definitions from the root `routes/api.php`
   and replace them with a `require` line pointing at the new file (see
   "Module routes" in Phase 1 for the pattern).
3. Leave routes belonging to other domains exactly where they are —
   don't restructure the whole file in one pass just because you're in
   there.

### Verify

Run, in order, whatever of these apply to the project:

```bash
composer dump-autoload
php artisan route:clear
php artisan optimize:clear
php artisan route:list --json > /tmp/routes-after.json
php artisan test
```

Diff `/tmp/routes-before.json` against `/tmp/routes-after.json` (same
URIs, methods, names, middleware, action classes — order of unrelated
routes may shift, but nothing should appear, disappear, or change
target). If the routes moved cleanly, this diff should be empty or show
only the ones you intentionally touched pointing at the same
controller action as before. Report what actually ran and passed —
don't claim tests or route checks passed if you weren't able to run
them.

### Review the diff yourself before reporting

```bash
git diff
git status
```

The diff should show file moves, namespace changes, and reference
updates — nothing else. If you spot a business-logic change you didn't
intend (even a one-line one, e.g. a query got reformatted differently),
revert that hunk before reporting completion.

### Hand off to clean-code-guard

Once the move is verified and the diff looks structural-only, **use the
`clean-code-guard` skill to review the changes you just made** before
telling the user you're done. This is a required step, not optional
follow-up — a second pass with different instructions catches things
your own diff review misses, and the whole point of gating this workflow
is to end with a diff someone can trust. Give it the diff or the list of
changed files as its scope.

### Report to the user

Summarize: which files moved, verification results, the clean-code-guard
review outcome, and anything flagged as out of scope (bugs noticed but
not fixed, ambiguous references left alone). Recommend committing this
diff before starting any further refactor on the same files, so a
structural change and a future code-quality change never land in the
same uncommitted diff.

---

## Hard rules (read `references/hard-rules.md` for the full list)

**Allowed:** move files, create domain directories, update namespaces
and imports required by the move, update route/binding references
required by the move, run autoload/tests.

**Forbidden:** rewriting business logic, optimizing queries, changing
validation/auth/response formats, renaming domain concepts or DB
tables/columns/routes, introducing new architectural layers
(repositories, interfaces, CQRS, base classes) "while you're in there,"
fixing bugs you notice, opportunistic formatting or cleanup.

If a request is actually about fixing code quality *inside* files
(N+1s, overengineered layers, naming) rather than *where files live*,
that's a different task — say so and ask whether they want the
structural move, the quality pass, or both as separate reviewable
changes.
