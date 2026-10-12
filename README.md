# pi-subagents

Delegate tasks to specialized subagents in [pi](https://pi.dev), each running in its own `pi` process with an isolated context window.

Packaged as a [pi package](https://pi.dev/packages): the `subagent` tool and workflow prompt templates install automatically; example agents are one command away.

## Features

- **Isolated context**: each subagent runs in a separate `pi` process
- **Streaming output**: see tool calls and progress live (parallel tasks stream simultaneously)
- **Usage tracking**: turns, tokens, cost, and context usage per agent
- **Abort support**: Ctrl+C propagates to kill subagent processes

## Install

```bash
pi install git:github.com/jaxkodex/pi-subagents
```

Project-local instead (written to `.pi/settings.json`, loaded after project trust):

```bash
pi install -l git:github.com/jaxkodex/pi-subagents
```

Try without installing:

```bash
pi -e git:github.com/jaxkodex/pi-subagents
```

This auto-loads the `subagent` tool and the `/implement`, `/scout-and-plan`, and `/implement-and-review` prompt templates.

## Install example agents

Packages cannot distribute agents (pi discovers them directly from your agent directories), so copy the ones you want:

```bash
git clone git@github.com:jaxkodex/pi-subagents.git
mkdir -p ~/.pi/agent/agents
cp pi-subagents/agents/*.md ~/.pi/agent/agents/
```

| Agent | Purpose | Model | Tools |
|-------|---------|-------|-------|
| `scout` | Fast codebase recon, returns compressed context | Haiku | read, grep, find, ls, bash |
| `planner` | Implementation plans | Sonnet | read, grep, find, ls |
| `reviewer` | Code review | Sonnet | read, grep, find, ls, bash |
| `worker` | General-purpose | Sonnet | all default |

## Usage

### Single agent

```
Use scout to find all authentication code
```

### Parallel execution

```
Run 2 scouts in parallel: one to find models, one to find providers
```

### Chained workflow

```
Use a chain: first have scout find the read tool, then have planner suggest improvements
```

### Workflow prompts

```
/implement add Redis caching to the session store
/scout-and-plan refactor auth to support OAuth
/implement-and-review add input validation to API endpoints
```

| Prompt | Flow |
|--------|------|
| `/implement <query>` | scout → planner → worker |
| `/scout-and-plan <query>` | scout → planner |
| `/implement-and-review <query>` | worker → reviewer → worker |

## Tool modes

| Mode | Parameter | Description |
|------|-----------|-------------|
| Single | `{ agent, task }` | One agent, one task |
| Parallel | `{ tasks: [...] }` | Up to 8 tasks, 4 concurrent |
| Chain | `{ chain: [...] }` | Sequential with `{previous}` placeholder |

## Writing your own agents

Agents are markdown files with YAML frontmatter:

```markdown
---
name: my-agent
description: What this agent does
tools: read, grep, find, ls
model: claude-haiku-4-5
fallback_models: [provider/other-model]
thinking: high
---

System prompt for the agent goes here.
```

- `tools` — restricts the subagent's toolset (comma string or YAML list); omit for all default tools
- `model` — omitted means the subagent inherits the dispatching session's model and thinking level
- `fallback_models` — tried in order when `model` is rate-limited or unavailable
- `thinking` — pi thinking level (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`) the subagent starts with. Invalid values are ignored. It takes precedence over the dispatching session's level, and applies even when `model` is pinned (otherwise a pinned agent keeps its own startup level from settings). Pi clamps the level to what the selected model supports, so `thinking: max` is safe to set on an agent whose model only goes to `high`.

Locations:

- `~/.pi/agent/agents/*.md` — user-level, always loaded
- `.pi/agents/*.md` — project-level, loaded with `agentScope: "project"` or `"both"`; overrides user agents with the same name

## Security model

The tool spawns a `pi` subprocess with a delegated system prompt. Project-local agents (`.pi/agents/*.md`) are repo-controlled prompts that can run bash — only enable `agentScope: "both"`/`"project"` for repositories you trust. Interactively, pi prompts before running project-local agents in untrusted projects (`confirmProjectAgents: false` disables that).

## Layout

```
pi-subagents/
├── package.json
├── extensions/subagent/
│   ├── index.ts             # The subagent tool
│   └── agents.ts            # Agent discovery
├── agents/                  # Example agents (copy into ~/.pi/agent/agents/)
└── prompts/                 # Workflow prompt templates (auto-loaded)
```

## Migrating from the symlinked example

If you previously symlinked pi's bundled example:

```bash
rm -rf ~/.pi/agent/extensions/subagent
pi install git:github.com/jaxkodex/pi-subagents
```

Keep project-specific agents (e.g. a planner that knows your `repos.yml`) in each repo's `.pi/agents/` — the package is the mechanism, the repo is the persona.

## Limitations

- Collapsed view shows the last 10 items (expand with Ctrl+O)
- Parallel output capped at 50 KB per task in the model-visible result
- Agents are rediscovered on every invocation (edit them mid-session freely)
