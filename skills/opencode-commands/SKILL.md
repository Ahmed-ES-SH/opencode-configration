---
name: opencode-commands
description: >
  Generate production-ready OpenCode custom command `.md` files. Use this skill
  whenever the user asks to create, generate, build, write, scaffold, or add an
  OpenCode command — even if they just say things like "make me a command for X",
  "I want a /slash command that does Y", "create an opencode command", "help me
  automate X in opencode", or reference `.opencode/commands/`. Always use this
  skill for any OpenCode command creation task, regardless of how casually the
  request is phrased.
---

# OpenCode Commands Skill

Generate valid, production-ready OpenCode custom command `.md` files.

## What is an OpenCode Command?

Custom commands are prompt templates invoked in the OpenCode TUI via `/command-name`.
They are defined as Markdown files placed in:

- **Per-project**: `.opencode/commands/<name>.md`
- **Global**: `~/.config/opencode/commands/<name>.md`

Alternatively defined in `opencode.jsonc` under the `command` key.

---

## Skill Workflow

### Step 1 — Extract intent from the request

Before asking any questions, try to infer from the conversation:
- What is the command supposed to do?
- What arguments (if any) does it need?
- Is there a preferred agent or model?
- Should it run in a subtask (isolated context)?

Only ask clarifying questions for what you **cannot** infer. Keep it to at most 2 questions.

### Step 2 — Build the command file

Use the structure below. Fill every field you have data for; omit optional fields you don't.

```
---
description: <short description shown in TUI>
agent: <optional: e.g. build, plan, code-review>
model: <optional: e.g. anthropic/claude-sonnet-4-20250514>
subtask: <optional: true | false>
---

<prompt body — the template sent to the LLM>
```

### Step 3 — Apply prompt features

Use these placeholders when the command needs dynamic input:

| Syntax | Meaning |
|---|---|
| `$ARGUMENTS` | All arguments passed to the command as a single string |
| `$1`, `$2`, `$3` | Individual positional arguments |
| `` !`shell command` `` | Inject live shell output into the prompt |
| `@path/to/file` | Include file content in the prompt |

### Step 4 — Output

**Always** save the command as a `.md` file and present it using `present_files`.
- Filename = the command name (e.g., `review-pr.md` → invoked as `/review-pr`)
- Save to `/mnt/user-data/outputs/<command-name>.md`
- After presenting the file, tell the user where to place it:
  - Per-project: `.opencode/commands/<name>.md`
  - Global: `~/.config/opencode/commands/<name>.md`

---

## Frontmatter Reference

| Field | Required | Type | Notes |
|---|---|---|---|
| `description` | Recommended | string | Shown in TUI autocomplete |
| `agent` | Optional | string | Which OpenCode agent runs this command |
| `model` | Optional | string | Override the default model |
| `subtask` | Optional | boolean | `true` = isolated subagent context (won't pollute main chat) |

### Known agents
- `build` — for build/test/run tasks
- `plan` — for planning/architecture tasks
- Default (omit field) — uses current active agent

---

## Quality Rules

- **Keep the prompt body focused** — one clear job per command.
- **Use `$ARGUMENTS` for flexible commands**, positional args (`$1`, `$2`) for structured ones.
- **Use `subtask: true`** when the command should not affect the main conversation context (e.g. analysis, reporting).
- **Use `` !`command` ``** to inject live context (git log, test results, env info).
- **Use `@file`** to pull in static references (config files, schemas, spec docs).
- **Command name = filename** — use `kebab-case`, no spaces.
- Match the user's stack — if they work in Next.js + Laravel, phrase the prompt accordingly.

---

## Example Commands

### Basic — no arguments
```md
---
description: Review recent git changes and suggest improvements
agent: plan
---

Recent git commits:
!`git log --oneline -10`

Review these changes. Flag any issues with:
- Code quality and consistency
- Security concerns
- Missing tests or docs
Suggest concrete improvements.
```

### With arguments — `/component Button`
```md
---
description: Scaffold a new React component
agent: build
---

Create a new React component named $ARGUMENTS.
- Use TypeScript with proper prop types
- Follow the project's existing component structure in @src/components
- Include a default export and named export
- Add JSDoc comments for all props
```

### Multi-argument — `/create-api-route users GET`
```md
---
description: Generate a Next.js App Router API route
---

Create a Next.js App Router API route handler.
- Route name: $1
- HTTP method: $2
- Follow the conventions in @src/app/api
- Use proper TypeScript types
- Include error handling and validation
- Return consistent JSON responses
```

### Subtask — runs isolated, doesn't pollute context
```md
---
description: Analyze test coverage and report gaps
subtask: true
model: anthropic/claude-sonnet-4-20250514
---

Here are the current test results:
!`npm test -- --coverage 2>&1 | tail -40`

Analyze the coverage report. List:
1. Files with < 80% coverage
2. Critical paths that are untested
3. Suggested test cases to add
Output as a concise markdown report.
```

### File reference
```md
---
description: Audit a Laravel controller for security issues
agent: plan
---

Audit the controller at @$1 for:
- Mass assignment vulnerabilities
- Missing authorization checks
- SQL injection risks
- Missing input validation
- Exposed sensitive data in responses

Suggest fixes with code examples.
```

---

## Response Format (when generating a command for the user)

Structure your response as:

1. **Brief explanation** — what the command does and how it works
2. **Command invocation example** — show how to run it: `/command-name [args]`
3. **Prompt features used** — explain any `$ARGUMENTS`, shell injections, or file refs
4. **Placement instructions** — where to put the file
5. **Present the `.md` file** using `present_files`
