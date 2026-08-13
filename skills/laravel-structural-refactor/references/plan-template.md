# Plan file template

Save Phase 1 output to `plans/<domain-slug>-refactor-plan.md` using this
structure. The move table is the part the user actually needs to review
and approve — keep it complete and skimmable.

```markdown
# Structural refactor plan: <Domain>

Generated: <date>
Status: PROPOSED — not yet applied

## Scope

<One or two sentences: what domain, what triggered this, which
controllers/models/services are in scope.>

## Move table

| Current path | New path | New namespace |
|---|---|---|
| app/Http/Controllers/ArticleController.php | app/Modules/Article/Controllers/ArticleController.php | App\Modules\Article\Controllers |
| app/Models/Article.php | app/Modules/Article/Models/Article.php | App\Modules\Article\Models |
| app/Http/Requests/StoreArticleRequest.php | app/Modules/Article/Requests/StoreArticleRequest.php | App\Modules\Article\Requests |
| app/Http/Resources/ArticleResource.php | app/Modules/Article/Resources/ArticleResource.php | App\Modules\Article\Resources |
| (extracted from) routes/api.php | app/Modules/Article/routes/api.php | n/a — route file, no namespace |

## Routes wiring

<List the route definitions being extracted (by URI + method, or by
group), confirm relative order is preserved for anything that could
overlap with another domain's routes, and state the `require` line
being added to the root `routes/api.php`, e.g.:>

```php
require __DIR__.'/../app/Modules/Article/routes/api.php';
```

## References to update per file

<For each moved file, list what points at it today — imports, route
files, service-provider bindings, tests, factories, OpenAPI annotations
— so the apply phase doesn't have to rediscover this.>

- `ArticleController` — referenced in `routes/api.php`, no bindings.
- `Article` model — referenced in `ArticleFactory`,
  `ArticleController`, `ArticleObserver` (if any).

## Staying global (not moving)

<Classes considered but excluded, with the one-line reason — e.g. "used
by both Article and Organization domains."-->

## Uncertain / needs a decision

<Anything ambiguous — a shared trait, a facade-resolved class, a
controller that straddles two domains.>

## Pre-existing issues noticed (not fixed)

<Report only. Bugs or code smells spotted during inspection that are out
of scope for this structural pass.>
```

## Notes on filling it in

- The move table should cover every file identified in Phase 1's
  inspection step, not just the "obvious" controller/model pair —
  requests, resources, DTOs, events, listeners, jobs, mail, policies,
  and factories that belong to the domain all get a row.
- Keep the new-path convention consistent:
  `app/Modules/<Domain>/<Type>/<ClassName>.php`, where `<Type>` mirrors
  the original technical folder name (`Controllers`, `Models`,
  `Requests`, `Resources`, `Services`, `DTOs`, `Events`, `Listeners`,
  `Jobs`, `Mail`, `Policies`, `routes` (lowercase, matching Laravel's
  own top-level `routes/` convention).
- If a file's home is genuinely unclear, put it in "Uncertain" rather
  than picking a domain arbitrarily — the whole point of the gate is to
  catch this before anything moves.
