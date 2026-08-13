# Hard rules for structural-only refactoring

Read this in full before starting Phase 2 (Apply) on a real codebase —
these are the specific ways a "just move the file" task quietly turns
into a rewrite if you're not watching for them.

## Allowed

- Move files.
- Create domain directories (`app/Modules/<Domain>/...`).
- Update namespaces when required by the move.
- Update `use` statements when required by the move.
- Update route imports/references when required by the move.
- Move a domain's route definitions out of the root `routes/api.php`
  into `app/Modules/<Domain>/routes/api.php`, and wire the root file to
  `require` it — as long as URI, HTTP verb, route name, middleware, and
  controller action are identical before and after, and relative order
  between routes that could plausibly overlap is preserved.
- Update service-provider bindings, tests, and factories that reference
  a moved class.
- Update Composer autoload configuration only if strictly necessary.
- Run `composer dump-autoload`, `php artisan optimize:clear`,
  `php artisan route:list`, `php artisan test`.
- Run `git diff` / `git status` to inspect what changed.

## Forbidden

Anything that changes *behavior*, however small:

- Rewriting business logic, refactoring algorithms.
- Changing database queries, optimizing Eloquent queries, changing
  relationships or model behavior.
- Changing validation rules, request authorization logic.
- Changing API response formats or Resource serialization.
- Changing authentication or authorization logic.
- Changing middleware, policy, event, listener, job, or mail behavior.
- Changing queue behavior, transactions, or exception handling.
- Changing route URLs, HTTP methods, route names, or route middleware —
  including accidentally changing which route wins by reordering
  overlapping patterns when splitting routes into per-module files.
- Changing database schema, migrations, seeders.
- Adding or removing features.
- Replacing, upgrading, or downgrading packages.
- Introducing a modular-architecture package.
- Introducing repositories, interfaces, CQRS, event sourcing, hexagonal
  layers, generic base classes/services/repositories that didn't
  already exist — "cleaning up while I'm in here" is exactly the trap
  this rule exists to name.
- Renaming business concepts (e.g. `Article`, `Organization`, `Payment`,
  `User`, `Wallet`) or any existing domain class name.
- Renaming database tables, database columns, API fields, route names,
  or endpoints.
- Fixing bugs you notice along the way — report them separately instead:
  `Found existing issue: X. No change made — bug fixing is outside the
  scope of this refactor.`
- Opportunistic cleanup: don't split a large method, don't simplify code
  that "could be" simpler, don't reformat a whole file because one line
  in it changed namespace.

## Domain discovery — what to inspect

Before proposing a module boundary, check:

```
app/Models/            app/Jobs/           tests/
app/Http/Controllers/  app/Mail/           database/
app/Http/Requests/     app/Policies/       composer.json
app/Http/Resources/    routes/
app/Http/Services/     database/factories/
app/DTOs/
app/Events/
app/Listeners/
```

And also inspect: model relationships, controller/service dependencies,
route references, event↔listener wiring, job references, policy
references, OpenAPI annotations, tests, factories, and service-provider
bindings. The goal is to find which classes travel together — a
controller that's the only caller of a service is one domain; a service
called from three unrelated controllers is probably shared
infrastructure, not part of any one module.

## Global vs. module-specific — the test

A class moves into a module when it clearly belongs to one business
domain (a controller, its requests, its resources, its domain-specific
service, its events). A class stays global when it's shared
infrastructure used across domains: middleware, service providers,
helpers, console commands, or any class more than one unrelated domain
depends on. When genuinely unsure, leave it global and note the
uncertainty in the plan rather than forcing a boundary.

## Namespace migration — the only kind of change a move causes

Before:

```php
namespace App\Http\Controllers;

use App\Models\Article;
use App\Http\Requests\StoreArticleRequest;
```

After:

```php
namespace App\Modules\Article\Controllers;

use App\Modules\Article\Models\Article;
use App\Modules\Article\Requests\StoreArticleRequest;
```

Only the namespace and `use` lines change. The body of the class —
every method, every line of logic — stays byte-for-byte identical
unless a line literally cannot compile without the updated import.

## Definition of success

The refactor succeeded when: the application behaves exactly as before,
related domain code now lives together, a developer can find a feature
without searching the whole `app/` tree, namespaces match the new
structure, Laravel boots, routes work, and tests pass — and none of that
required changing what any single line of business logic does.

When in doubt whether something is structural or behavioral: don't
change it. Report it instead and let the user decide.
