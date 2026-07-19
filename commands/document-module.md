---
description: Analyze a module's files and generate one organized .md doc with tree structure, full code per file, and related-module context
agent: build
subtask: true
---

Module path: $1

## Step 1 — Map the module structure

Tree structure of the module:
!`tree "$1" -I 'node_modules|dist|.git' 2>/dev/null || find "$1" -type f -not -path '*/node_modules/*' -not -path '*/dist/*' -not -path '*/.git/*' | sort`

## Step 2 — Read every file in the module

Read the full content of every file listed in the tree above (skip lockfiles, build artifacts, and generated files like `*.map`, `*.lock`, `dist/**`).

## Step 3 — Detect relations to other modules

For each file, check its imports/`use`/`require` statements. If a file imports from outside `$1` (e.g. a shared service, another Nest module, a Laravel shared trait/interface, a shared DTO/type, a shared model), locate that file in the project and read it too — it's "related context," not part of the module itself.

Relation signals to look for:
- NestJS: `imports: [...]` in the `@Module()` decorator, injected providers from other modules, shared DTOs/entities/interfaces, guards/pipes/decorators imported from `common/` or `shared/`
- Laravel: other models referenced via relationships (`belongsTo`, `hasMany`, etc.), traits, form requests, policies, service classes injected via constructor
- Next.js: shared components, hooks, types, or API clients imported from outside the module's folder

## Step 4 — Generate the single output .md file

Produce **one** markdown file named `$1`-module.md (derive a clean kebab-case name from the module path, e.g. `users` module → `users-module.md`). Structure it exactly like this:

```
# Module: <module name>

## 📁 Tree Structure
<the tree from Step 1, in a code block>

## 📄 Module Files

--------------------------------------------------------------------------------
### 📄 <relative/path/to/file-1.ts>
--------------------------------------------------------------------------------

```<language>
<full file content>
```

--------------------------------------------------------------------------------
### 📄 <relative/path/to/file-2.ts>
--------------------------------------------------------------------------------

```<language>
<full file content>
```

(... repeat for every file in the module, in the same order as the tree ...)

## 🔗 Related Files (outside the module)

Only include this section if Step 3 found relations. For each related file:

--------------------------------------------------------------------------------
### 🔗 <relative/path/to/related-file.ts> — related to: <which module file(s) use it, and why>
--------------------------------------------------------------------------------

```<language>
<full file content>
```
```

Formatting rules — follow exactly:
- Every file section (module file or related file) MUST be separated by a full-width `-` divider line above and below its header, as shown.
- The header line MUST contain the file's relative path (not just the filename) so it's unambiguous.
- Use the correct language tag on each code fence (`ts`, `tsx`, `php`, `json`, etc.) based on the file extension.
- Keep module files and related files in two clearly separate sections — never interleave them.
- Do not summarize or truncate file content — reproduce it in full, exactly as it is in the repo.
- Do not add commentary/analysis between files unless it's the one-line "related to" note on related files.

Save the result to `docs/modules/<module-name>-module.md` (create the `docs/modules/` directory if it doesn't exist).
