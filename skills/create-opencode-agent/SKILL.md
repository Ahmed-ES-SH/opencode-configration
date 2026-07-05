---
name: opencode-agents
description: >
  Generate production-ready OpenCode agent `.md` files and `opencode.json` agent
  configs. Use this skill whenever the user asks to create, build, configure, scaffold,
  or design an OpenCode agent — even for casual phrases like "make me an agent that
  reviews code", "I want a read-only agent", "create an opencode agent for X",
  "how do I add a custom agent to opencode", "I want an agent that can't edit files",
  or any mention of `.opencode/agents/`, `~/.config/opencode/agents/`, or the
  `agent` key in `opencode.json`. Always use this skill for any OpenCode agent
  creation or configuration task, regardless of how casually the request is phrased.
---
 
# OpenCode Agents Skill
 
Generate valid, production-ready OpenCode agent `.md` files and JSON configs.
 
## What is an OpenCode Agent?
 
Agents are specialized AI assistants with custom prompts, models, and tool permissions.
They can be defined as Markdown files or in `opencode.json`.
 
### Two Types
 
| Type | How Used | Built-in examples |
|---|---|---|
| `primary` | User-facing; cycle with **Tab** or `switch_agent` keybind | `build`, `plan` |
| `subagent` | Invoked by primary agents, or manually via `@name` | `general`, `explore`, `scout` |
 
**Mode `all`** (default if omitted) means the agent works as both.
 
---
 
## Skill Workflow
 
### Step 1 — Extract intent
 
Before asking questions, infer from the conversation:
- What specialized task should the agent perform?
- Should it be a primary agent (user switches to it) or a subagent (invoked programmatically / `@name`)?
- Does it need to modify files, run bash, or be strictly read-only?
- Is there a preferred model or temperature?
Ask at most 2 clarifying questions for what cannot be inferred.
 
### Step 2 — Choose the right mode and permissions
 
Use this decision matrix:
 
| Intent | Recommended mode | Key permissions |
|---|---|---|
| Full development work | `primary` | `edit: allow`, `bash: allow` |
| Analysis / planning only | `primary` | `edit: ask`, `bash: ask` |
| Code review (read-only) | `subagent` | `edit: deny`, `bash: deny` |
| Codebase exploration | `subagent` | `edit: deny`, `bash: deny` |
| Docs writing | `subagent` | `bash: deny` |
| Security audit | `subagent` | `edit: deny` |
| Orchestrator (spawns others) | `primary` | `task: { "*": "allow" }` |
| Internal/hidden helper | `subagent` + `hidden: true` | per use case |
 
### Step 3 — Build the agent file (Markdown format)
 
```
---
description: <what this agent does and when to use it — REQUIRED>
mode: <primary | subagent | all>
model: <optional: provider/model-id>
temperature: <optional: 0.0-1.0>
steps: <optional: max agentic iterations>
color: <optional: hex #RRGGBB or theme key>
top_p: <optional: 0.0-1.0>
hidden: <optional: true | false — subagents only>
permission:
  edit: <allow | ask | deny>
  bash: <allow | ask | deny | pattern map>
  read: <allow | ask | deny>
  webfetch: <allow | ask | deny>
  websearch: <allow | ask | deny>
  task: <allow | ask | deny | pattern map>
---
 
<system prompt body — specialized instructions for this agent>
```
 
### Step 4 — Output
 
**Always** save the agent as a `.md` file and present it with `present_files`.
- Filename = agent name (e.g., `code-reviewer.md` → invoked as `@code-reviewer`)
- Save to `/mnt/user-data/outputs/<agent-name>.md`
- Tell the user where to place it:
  - **Per-project**: `.opencode/agents/<name>.md`
  - **Global**: `~/.config/opencode/agents/<name>.md`
---
 
## All Configuration Options
 
### `description` (required)
Brief description shown in the TUI and used by the model to decide when to invoke this agent.
Keep it specific. Bad: "Reviews code." Good: "Read-only agent that reviews code for security, performance, and maintainability issues."
 
### `mode`
- `primary` — user-selectable via Tab; handles main conversation
- `subagent` — invoked automatically or via `@name`; isolated session
- `all` — works as both (default when omitted)
### `model`
Override the model for this agent. Format: `provider/model-id`.
Examples: `anthropic/claude-sonnet-4-20250514`, `anthropic/claude-haiku-4-20250514`, `openai/gpt-5`.
- If omitted: primary agents use the globally configured model; subagents inherit the invoking primary agent's model.
### `temperature`
Controls randomness (0.0–1.0):
- `0.0–0.2` — deterministic; best for code analysis, security review, planning
- `0.3–0.5` — balanced; good for general development
- `0.6–1.0` — creative; best for brainstorming, docs drafting
Default: 0 for most models, 0.55 for Qwen models.
 
### `steps`
Max agentic iterations before forcing a text-only response. Omit for unlimited.
When the limit is hit, the agent summarizes its work and lists remaining tasks.
 
### `top_p`
Alternative to temperature for response diversity. Range: 0.0–1.0.
 
### `color`
Agent color in the TUI. Use hex (`#FF5733`) or a theme key: `primary`, `secondary`, `accent`, `success`, `warning`, `error`, `info`.
 
### `hidden`
`true` — hides the agent from `@` autocomplete. Only applies to `mode: subagent`.
The model can still invoke it via the Task tool if permissions allow.
 
### `disable`
`true` — disables the agent entirely.
 
### `permission`
Fine-grained tool access control. Values: `"allow"`, `"ask"`, `"deny"`.
 
| Permission key | Tools it controls |
|---|---|
| `read` | `read` |
| `edit` | `write`, `edit`, `apply_patch` |
| `glob` | `glob` |
| `grep` | `grep` |
| `list` | `list` |
| `bash` | `bash` |
| `task` | `task` (subagent invocation) |
| `external_directory` | Files outside project worktree |
| `todowrite` | `todowrite`, `todoread` |
| `webfetch` | `webfetch` |
| `websearch` | `websearch` |
| `lsp` | `lsp` |
| `skill` | `skill` |
| `question` | `question` |
| `doom_loop` | Recovery prompts when stuck |
 
**Bash permission patterns** (last matching rule wins — put `*` first):
```yaml
permission:
  bash:
    "*": ask          # ask for everything by default
    "git status": allow
    "git log*": allow
    "grep *": allow
```
 
**Task permission patterns** (for orchestrators):
```yaml
permission:
  task:
    "*": deny
    "code-reviewer": allow
    "docs-*": allow
```
 
---
 
## JSON Config Format (opencode.json)
 
For teams or when you want everything in one place:
 
```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "code-reviewer": {
      "description": "Read-only code review for quality and security",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.1,
      "prompt": "{file:./prompts/code-review.txt}",
      "permission": {
        "edit": "deny",
        "bash": "deny"
      }
    }
  }
}
```
 
The `prompt` field references an external file: `{file:./path/to/file.txt}` (relative to the config file location).
 
---
 
## Built-in Agents Reference
 
Don't recreate these — override them via config if needed.
 
| Name | Mode | Default permissions | When to use |
|---|---|---|---|
| `build` | primary | all `allow` | Standard development |
| `plan` | primary | `edit: ask`, `bash: ask` | Analysis without changes |
| `general` | subagent | full (except todo) | Parallel multi-step tasks |
| `explore` | subagent | read-only | Fast codebase exploration |
| `scout` | subagent | read-only | External docs / dependency research |
| `compaction` | primary, hidden | — | Auto context compaction |
| `title` | primary, hidden | — | Auto session title generation |
| `summary` | primary, hidden | — | Auto session summaries |
 
To override a built-in (e.g., change `plan`'s model):
```json
{
  "agent": {
    "plan": {
      "model": "anthropic/claude-haiku-4-20250514",
      "temperature": 0.1
    }
  }
}
```
 
---
 
## Quality Rules
 
- **`description` is the agent's identity** — make it precise; the model reads it to decide when to invoke the agent.
- **Match permissions to the agent's job** — read-only agents must have `edit: deny`; don't leave it ambiguous.
- **Use `steps` for cost-sensitive agents** — especially for subagents that run in parallel.
- **Use `hidden: true`** for internal orchestration helpers that shouldn't be user-selectable.
- **Prefer `subagent` for specialization** — keep primary agents general, push expertise into subagents.
- **Use `{file:./prompts/agent.txt}`** for long system prompts in JSON configs to keep the config readable.
- **Agent name = filename** — use `kebab-case`, no spaces, all lowercase.
- **Temperature 0.1** for analysis/review agents; **0.3** for build agents; **0.7+** for brainstorming agents.
---
 
## Common Agent Templates
 
### Code Reviewer (subagent, read-only)
```md
---
description: Reviews code for quality, security, and performance without making changes
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.1
permission:
  edit: deny
  bash:
    "*": deny
    "git diff": allow
    "git log*": allow
---
 
You are a senior code reviewer. Analyze code for:
- Security vulnerabilities (injection, auth flaws, data exposure)
- Performance issues and scalability concerns
- Code quality, maintainability, and clean code principles
- Missing error handling and edge cases
- API design consistency
 
Provide specific, actionable feedback with code examples. Never modify files directly.
```
 
### Security Auditor (subagent, read-only)
```md
---
description: Performs security audits, identifies vulnerabilities, and suggests fixes
mode: subagent
temperature: 0.1
permission:
  edit: deny
  bash: deny
---
 
You are a security expert. Audit code for:
- Input validation vulnerabilities (SQL injection, XSS, CSRF)
- Authentication and authorization flaws
- Sensitive data exposure
- Dependency vulnerabilities
- Misconfigured permissions and environment variables
 
Report findings by severity (Critical / High / Medium / Low) with remediation steps.
```
 
### Documentation Writer (subagent)
```md
---
description: Writes and maintains project documentation, READMEs, and API docs
mode: subagent
permission:
  bash: deny
---
 
You are a technical writer. Create clear, comprehensive documentation:
- README files with setup, usage, and API reference
- Inline code comments and JSDoc/PHPDoc blocks
- Architecture decision records (ADRs)
- API endpoint documentation
 
Use plain language. Always include code examples.
```
 
### Debug Agent (subagent, read + bash)
```md
---
description: Investigates bugs, traces errors, and proposes fixes without applying them
mode: subagent
temperature: 0.2
steps: 20
permission:
  edit: deny
  bash:
    "*": ask
    "cat *": allow
    "grep *": allow
    "git log*": allow
    "git diff*": allow
---
 
You are a debugging specialist. Investigate issues by:
- Reading error logs and stack traces
- Tracing the execution path
- Identifying root cause vs symptoms
- Proposing concrete fixes with reasoning
 
Never apply changes directly — output a clear diagnosis and recommended fix.
```
 
### Orchestrator (primary, manages subagents)
```md
---
description: Orchestrates parallel subagent workflows for complex multi-step tasks
mode: primary
model: anthropic/claude-sonnet-4-20250514
temperature: 0.3
permission:
  edit: ask
  bash: ask
  task:
    "*": allow
---
 
You are an orchestration agent. Break complex tasks into parallel subtasks:
1. Analyze the request and identify independent units of work
2. Spawn subagents (@general, @explore, @scout or custom agents) for specialized tasks
3. Collect and synthesize results
4. Present a consolidated summary
 
Prefer parallel execution over sequential when tasks are independent.
```
 
---
 
## Response Format
 
When generating an agent file for the user:
 
1. **Brief explanation** — what the agent does, its mode, and key permissions
2. **Usage example** — how to invoke it: Tab (primary) or `@agent-name` (subagent)
3. **Key decisions** — explain the permission choices and model/temperature selections
4. **Placement instructions** — global vs per-project path
5. **Present the `.md` file** using `present_files`
---
 
## Interactive Creation
 
Remind the user they can also use the interactive CLI to create agents:
 
```bash
opencode agent create
```
 
This prompts for location, description, generates a system prompt, and lets the user select permissions interactively.
 