# Laravel CI

## Core CI job

```yaml
name: CI

on:
  push:
    branches: ['**']
  pull_request:
    branches: [main, develop]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8
        env:
          MYSQL_ALLOW_EMPTY_PASSWORD: true
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v6

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, dom, curl, libxml, pdo, pdo_mysql, bcmath, intl, gd, zip
          coverage: none
          tools: composer:v2

      - name: Cache Composer dependencies
        uses: actions/cache@v4
        with:
          path: vendor
          key: composer-${{ runner.os }}-${{ hashFiles('**/composer.lock') }}
          restore-keys: composer-${{ runner.os }}-

      - name: Install Composer dependencies
        run: composer install --prefer-dist --no-interaction --no-progress

      - name: Copy environment file
        run: cp .env.testing .env || cp .env.example .env

      - name: Generate app key
        run: php artisan key:generate

      - name: Run migrations
        run: php artisan migrate --force
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: ''

      - name: Run Pint (code style)
        run: vendor/bin/pint --test
        # Swap for `vendor/bin/php-cs-fixer fix --dry-run --diff` if the
        # project uses PHP-CS-Fixer instead of Pint.

      - name: Run tests
        run: php artisan test
        # Use `vendor/bin/pest` directly if the project is Pest-only and
        # doesn't go through `artisan test`.
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: ''

  frontend-assets:
    runs-on: ubuntu-latest
    if: false # flip to true (remove this line) if the project builds
              # frontend assets via Vite/Mix inside the Laravel repo itself
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npm run build
```

## Notes specific to Laravel

- **SQLite in-memory is a faster alternative** to the MySQL service container when the project doesn't rely on MySQL-specific SQL (`json_table`, certain collation behavior, etc.) — set `DB_CONNECTION=sqlite` and `DB_DATABASE=:memory:` and skip the `services:` block entirely. This is worth suggesting if CI speed matters more than perfect environment parity; recommend the MySQL container instead when the project's queries or migrations depend on MySQL-specific behavior.
- **Pint vs PHP-CS-Fixer vs Larastan/PHPStan**: check `composer.json`'s `require-dev` to see which tools are actually installed before assuming Pint — add a `larastan` step (`vendor/bin/phpstan analyse`) if static analysis is part of the project's standards.
- **Pest vs PHPUnit**: `php artisan test` works for both since Pest is built on PHPUnit under the hood; only bypass it if the project has custom Artisan test command wiring.
- **`--force` on migrate**: Laravel blocks destructive commands in production by default; `--force` is required even in CI since `APP_ENV` isn't `local` there.
- **Separate frontend-assets job**: only needed if Vite/Laravel Mix lives inside the same repo (a classic Laravel + Blade + Vite setup). Skip it entirely for an API-only Laravel backend paired with a separate Next.js frontend — which is the more common shape for Ahmed's stack.

## Matching Ahmed's typical setup (Laravel or NestJS API + Next.js frontend)

When Laravel is the backend API for a separate Next.js frontend, there's no `frontend-assets` job needed — leave that job's `if: false` in place (or delete it) and treat this as a pure API CI pipeline. Extensions listed above cover the common case (auth, MySQL, image handling via `gd`); trim or extend based on what the project's `composer.json` actually requires (e.g. add `redis` if the queue driver needs it, drop `gd` if there's no image processing).

For the deploy side:
- `references/deploy-vps.md` for the classic Laravel deploy — rsync/SSH + `artisan migrate`, `queue:restart`, `config:cache`
- `references/deploy-docker.md` if containerizing PHP-FPM + Nginx
- `references/deploy-platforms.md` if using a managed platform like Render for the API
