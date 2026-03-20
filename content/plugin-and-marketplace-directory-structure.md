---
title: "Plugin and Marketplace Directory Structure"
description: "How to rationally organize a Claude Code plugin marketplace -- compiled from Anthropic's official spec, community repos, and practitioner patterns. Covers the canonical plugin layout, three plugin archetypes, and the agent-skill-command split."
tags:
  - context-engineering
  - claude-code
---

# Plugin and Marketplace Directory Structure

How to rationally organize a Claude Code plugin marketplace — compiled from Anthropic's official spec, the Handshake internal marketplace (critiqued), community repos, and practitioner patterns.

---

## 1. The Canonical Plugin Layout

Every plugin follows this structure. Components live at the plugin root, never inside `.claude-plugin/`.

```
my-plugin/
.claude-plugin/
  plugin.json              # Metadata only. Nothing else goes here.
agents/                    # Subagent definitions (.md files)
  explorer.md
skills/                    # Reference knowledge (SKILL.md + supporting files)
  my-domain/
    SKILL.md
    reference.md
    scripts/
commands/                  # User-invocable slash commands (.md files)
  init.md
  do-thing.md
hooks/                     # Event-driven automation
  hooks.json
scripts/                   # Executable utilities (called by hooks/skills/agents)
  validate.sh
.mcp.json                  # MCP server connections (remote or local)
.lsp.json                  # Language server configs (optional)
settings.json              # Default settings when plugin enabled (optional)
README.md                  # Documentation
```

### What Goes Where: The Decision Framework

| Component | Purpose | Who invokes | Loaded when |
|---|---|---|---|
| **`plugin.json`** | Identity, metadata, versioning | System | Plugin discovery |
| **`agents/`** | Isolated subagents with their own context, model, tools | Claude delegates or user requests | Task matches agent description |
| **`skills/`** | Reference knowledge, query syntax, domain context | Claude auto-loads when relevant | Description matches task context |
| **`commands/`** | User-facing entry points (slash commands) | User types `/command-name` | User invokes explicitly |
| **`hooks/`** | Event-driven automation (session start, tool use, etc.) | System triggers on events | Automatically on matching events |
| **`scripts/`** | Deterministic operations called by other components | Hooks, agents, skills execute them | When referenced component runs |
| **`.mcp.json`** | External service connections | System + agents | Plugin enabled |
| **`.lsp.json`** | Code intelligence for specific languages | System | Plugin enabled |
| **`settings.json`** | Default config (currently only `agent` key supported) | System | Plugin enabled |

Sources: [Plugins reference](https://code.claude.com/docs/en/plugins-reference) | [Create plugins](https://code.claude.com/docs/en/plugins)

---

## 2. plugin.json: What Belongs Here

The manifest defines identity and metadata. It does NOT contain instructions, prompts, or workflow logic.

### Minimal (most plugins)

```json
{
  "name": "my-plugin",
  "description": "What this plugin does in one sentence",
  "version": "1.0.0"
}
```

### Full Schema

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "Brief plugin description",
  "author": {
    "name": "Author Name",
    "email": "author@example.com",
    "url": "https://github.com/author"
  },
  "homepage": "https://docs.example.com/plugin",
  "repository": "https://github.com/org/plugin",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"],
  "commands": ["./custom/commands/special.md"],
  "agents": "./custom/agents/",
  "skills": "./custom/skills/",
  "hooks": "./config/hooks.json",
  "mcpServers": "./mcp-config.json",
  "lspServers": "./.lsp.json",
  "outputStyles": "./styles/"
}
```

### Key Rules

- **`name`** is the only required field. Kebab-case, no spaces. Becomes the namespace prefix: `/my-plugin:command-name`.
- **Component path fields** (`commands`, `agents`, `skills`, etc.) supplement default directories — they don't replace them. If `commands/` exists, it's always discovered. Custom paths add more.
- **All paths must be relative** and start with `./`.
- **Version** follows semver. If also set in marketplace.json, plugin.json takes priority. Set it in only one place.
- **Keywords** are discovery tags — use them for searchability in marketplace UIs.

### What keywords Should Cover

Keywords serve marketplace discovery. Good keywords describe:
- **Domain**: `bigquery`, `datadog`, `confluence`, `linear`
- **Function**: `monitoring`, `documentation`, `authentication`, `deployment`
- **Integration type**: `mcp`, `api`, `database`, `observability`

Don't duplicate what the name already says. Don't use generic terms like `tool` or `plugin`.

Sources: [Plugin manifest schema](https://code.claude.com/docs/en/plugins-reference#plugin-manifest-schema)

---

## 3. The Three Plugin Archetypes

Studying the Handshake marketplace and community repos reveals three distinct plugin patterns. Most plugins fit one of these.

### Archetype 1: MCP Subagent Plugin (Most Common)

**Pattern:** Connect to an external service via MCP, provide an agent to work with it, expose commands as entry points.

```
datadog/
.claude-plugin/plugin.json
.mcp.json                    # MCP connection to Datadog API
agents/
  explorer.md                # Isolated subagent with Sonnet model
commands/
  init.md                    # Interactive setup wizard
  dd-logs.md                 # Thin delegation: "ask explorer to query logs"
  dd-dashboard.md
  dd-incidents.md
skills/
  datadog/
    SKILL.md                 # Query syntax reference (loaded by agent on-demand)
```

**The delegation chain:** User types `/dd-logs` -> Command delegates to `datadog:explorer` agent -> Agent discovers MCP tools via `ToolSearch` -> Agent consults `skills/datadog/SKILL.md` for query syntax -> Agent queries Datadog -> Returns formatted results.

**When to use:** Any integration with an external service that has an MCP server or API.

### Archetype 2: Documentation-as-Skill Plugin

**Pattern:** Package domain knowledge as a skill tree that Claude consults automatically.

```
promotions-docs/
.claude-plugin/plugin.json
commands/
  promotions-docs.md          # Manual entry point to browse docs
skills/
  promotions-docs/
    SKILL.md                  # Entry point / index (user-invocable: false)
    system-architecture.md
    glossary.md
    channels/
      feed.md
      digest.md
    events/
      job-impression.md
      job-open.md
```

**No agent, no MCP.** The value is reference knowledge that Claude auto-loads when topics match. SKILL.md is the table of contents.

**When to use:** Internal product documentation, API references, architecture guides, onboarding knowledge.

### Archetype 3: Hook-Only Plugin

**Pattern:** Event-driven automation with no user-facing commands.

```
gcauth-guard/
.claude-plugin/plugin.json
hooks/
  hooks.json                  # SessionStart hook
scripts/
  gcauth-guard.sh             # Auth verification script
skills/
  gcauth-guard/
    SKILL.md                  # Troubleshooting reference (user-invocable: false)
```

**When to use:** Environment validation, auto-formatting, security checks, CI/CD gates.

Sources: Handshake `claude-code-marketplace` analysis

---

## 4. The Agent-Skill-Command Split (Critical Design Decision)

A key structural question: what goes in an agent vs. a skill vs. a command?

### The Principle

| Layer | Contains | Analogy |
|---|---|---|
| **Commands** | Entry points + delegation instructions | The button on the dashboard |
| **Agents** | Workflow logic, tool orchestration, output formatting | The operator who does the work |
| **Skills** | Reference knowledge, syntax, conventions | The manual the operator consults |

### Commands: Thin Delegation Layers

Commands should be **minimal**. Their job is to:
1. Accept user input via `$ARGUMENTS`
2. Delegate to the right agent with a task prompt
3. Handle input-dependent branching (e.g., "if this looks like an ID, fetch it; if it looks like a name, search for it")

**Good command (from Handshake `dd-logs.md`):**
```markdown
---
name: dd-logs
description: Search and analyze Datadog logs
argument-hint: [service] [query]
disable-model-invocation: true
---
Delegate this to the `datadog:explorer` agent with context:
- Service: {{ service | default: "not specified" }}
- Query: {{ query | default: "not specified" }}
Investigate logs matching the criteria...
```

**Anti-pattern:** Putting workflow logic, output formatting, or reference material directly in a command. Commands should be <30 lines.

### Agents: Isolated Workflow Executors

Agents contain:
- **System prompt** defining role, expertise, workflow patterns
- **Tool restrictions** (what the agent can/can't do)
- **Model selection** (use `sonnet` for most subagent work — cost-effective)
- **MCP tool discovery instructions** (`ToolSearch` with service filters)
- **Output formatting rules** (tables, not raw JSON)
- **Error handling guidance** (OAuth re-auth, retry patterns)

### Skills: Reference Material

Skills contain domain knowledge the agent consults on-demand:
- Query syntax references
- API documentation
- State machine definitions
- Convention guides
- Troubleshooting playbooks

Use `user-invocable: false` for skills that are pure reference (users shouldn't invoke them as commands).

### Critique of the Handshake Pattern

**What Handshake does well:**
- Clean three-layer separation (commands delegate to agents, agents consult skills)
- Consistent structure across all MCP plugins (atlassian, datadog, linear are structurally identical)
- `init.md` wizards that persist config to `~/.claude/CLAUDE.md` — good pattern for user-specific setup
- Agents use `ToolSearch` with service filters rather than hardcoding tool names — robust against MCP API changes
- All agents use `model: sonnet` — cost-effective for subagent work

**What Handshake gets wrong or could improve:**

1. **Config persistence to `~/.claude/CLAUDE.md` is fragile.** All three MCP plugins write their config as markdown sections into the same shared file. This creates merge conflicts if multiple init wizards run, and pollutes the global CLAUDE.md with plugin-specific data. **Better:** Write to `~/.claude/plugins/<plugin-name>/config.md` or use a plugin-scoped settings file. Or at minimum, use `CLAUDE.local.md` so it doesn't get version-controlled.

2. **marketplace.json duplicates plugin.json metadata.** Every plugin entry in marketplace.json repeats name, version, description, author, and keywords from the plugin's own plugin.json. This creates a sync burden — change one, forget the other. **Better:** Set version in only one place. For relative-path plugins in a monorepo marketplace, set version in marketplace.json only and omit from plugin.json (per Anthropic's guidance). Let plugin.json own name and description; let marketplace.json add only marketplace-specific fields (source, category, tags).

3. **No CODEOWNERS-level granularity for skills/commands within a plugin.** CODEOWNERS operates at the plugin directory level, not per-component. If two teams need to own different commands within the same plugin, the plugin should be split. **Better:** One plugin per owning team. If ownership is shared, use CODEOWNERS at the file level within the plugin.

4. **Agent descriptions lack `<example>` blocks in some plugins.** The Datadog agent has rich example routing blocks; others don't. Inconsistent discoverability. **Better:** Every agent should include 2-3 `<example>` blocks showing user/assistant pairs with `<commentary>` explaining routing logic.

5. **No `settings.json` usage.** None of the plugins use `settings.json` to set a default agent. The `datadog:explorer` agent could be the default when the plugin is enabled, activated by `{"agent": "explorer"}` in settings.json. **Opportunity, not a bug** — but worth considering for plugins where the agent should be proactively invoked.

6. **No persistent agent memory.** None of the agents use the `memory` frontmatter field. The datadog explorer, for instance, could learn common query patterns, frequently investigated services, and recurring incident types across sessions. **Better:** Add `memory: user` to agents that benefit from cross-session learning.

7. **Command naming inconsistency.** Datadog uses `dd-` prefix (`dd-logs`, `dd-dashboard`), but Atlassian uses `confluence-` prefix instead of `at-` or `atlassian-`. Linear uses `linear-` prefix. **Minor issue** — the namespace prefix from plugin.json handles collision avoidance anyway, but within-plugin consistency could be tighter.

---

## 5. Marketplace-Level Organization

### marketplace.json Structure

```json
{
  "name": "company-marketplace",
  "owner": {
    "name": "Platform Team",
    "email": "platform@company.com"
  },
  "metadata": {
    "description": "Internal Claude Code plugins",
    "version": "1.0.0",
    "pluginRoot": "./plugins"
  },
  "plugins": [
    {
      "name": "datadog",
      "source": "./plugins/datadog",
      "description": "Datadog observability integration",
      "version": "1.1.0",
      "keywords": ["monitoring", "observability", "logs", "metrics"],
      "category": "observability"
    }
  ]
}
```

### Key Design Decisions

**Organize plugins by external service/domain, not by function type.**

Good:
```
plugins/
  datadog/        # Everything for Datadog
  linear/         # Everything for Linear
  confluence/     # Everything for Confluence
```

Bad:
```
plugins/
  mcp-connections/    # All MCP configs together
  all-agents/         # All agents together
  all-commands/       # All commands together
```

Each plugin should be self-contained. A user installing `datadog` gets everything they need for Datadog — the MCP connection, the agent, the commands, the reference skill — in one install.

**Use `metadata.pluginRoot`** to avoid repeating `./plugins/` in every source path:
```json
{
  "metadata": { "pluginRoot": "./plugins" },
  "plugins": [
    { "name": "datadog", "source": "datadog" }
  ]
}
```

**Use categories and tags for marketplace-level organization:**
- `category`: Broad grouping (`observability`, `project-management`, `documentation`, `security`, `devtools`)
- `tags`: Specific terms for search (`logs`, `metrics`, `incidents`, `jira`, `confluence`)
- `keywords` (in plugin.json): Plugin-level discovery terms

**Version management rules:**
- Set version in ONE place. For monorepo marketplaces with relative paths, set in marketplace.json.
- For external-source plugins (GitHub, npm), set in plugin.json.
- Never set in both — plugin.json silently wins, causing confusion.

Sources: [Plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) | [Plugins reference](https://code.claude.com/docs/en/plugins-reference)

---

## 6. The Init Wizard Pattern

Every MCP subagent plugin should have an `init.md` command that runs a guided setup wizard.

### What the Init Wizard Does

1. **Verify authentication** (OAuth, API key, SSO)
2. **Discover user context** (which teams, projects, spaces they care about)
3. **Write configuration** to a persistent location
4. **Confirm setup** with a summary

### Where to Persist Config

| Approach | Pros | Cons |
|---|---|---|
| `~/.claude/CLAUDE.md` (Handshake's approach) | Always loaded, agents see it automatically | Shared file, collision risk, pollutes global context |
| `~/.claude/plugins/<name>/config.md` | Isolated, no collisions | Must be explicitly referenced by agent |
| Plugin-scoped `settings.json` | Structured, machine-readable | Limited to supported keys (currently only `agent`) |
| Environment variables | Standard, works with CI/CD | No persistence across sessions without shell config |

**Recommended:** Write to `~/.claude/CLAUDE.md` as a clearly demarcated section (e.g., `## Datadog Configuration`) for simplicity, but be aware of the collision risk. If your marketplace has 5+ plugins all writing to the same file, consider a plugin-scoped approach.

### Init Command Pattern

```markdown
---
name: init
description: Set up the plugin with your credentials and preferences
disable-model-invocation: true
---

Run an interactive setup wizard:

1. **Authentication**: Check if [service] is accessible
2. **Context**: Ask the user for their commonly used [resources]
3. **Write config**: Save to `~/.claude/CLAUDE.md` under `## [Service] Configuration`
4. **Confirm**: Show a summary of what was configured

If the user already has a `## [Service] Configuration` section, update it in-place.
```

---

## 7. MCP Configuration (.mcp.json)

### When to Use

- Plugin integrates with an external service that has an MCP server
- The service provides tools Claude needs (query, create, update operations)

### Structure

```json
{
  "mcpServers": {
    "service-name": {
      "type": "http",
      "url": "https://mcp.service.com/v1/mcp"
    }
  }
}
```

Or for local process-based servers:

```json
{
  "mcpServers": {
    "service-name": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/my-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
      "env": {
        "API_KEY": "${SERVICE_API_KEY}"
      }
    }
  }
}
```

### Key Rules

- **Always use `${CLAUDE_PLUGIN_ROOT}`** for paths within the plugin directory. Plugins are cached to `~/.claude/plugins/cache/` — absolute paths break after installation.
- **Server names become tool prefixes.** Name them after the service, not the function.
- **Remote HTTP MCP servers** (`type: http`) are simpler and preferred when available.
- **Agents should use `ToolSearch`** to discover MCP tools dynamically rather than hardcoding tool names. MCP APIs change; hardcoded names break.

---

## 8. Agent Design Within Plugins

### Frontmatter Template

```markdown
---
name: explorer
description: "[Service] expert that can query, analyze, and manage [domain]. Use proactively when the user mentions [trigger terms]."
model: sonnet
---

You are a [Service] expert. Your role is to help users [core capability].

## Tool Discovery (CRITICAL)
Before any operation, discover available tools:
Use `ToolSearch` with query "+[service]" to find all available [service] tools.

## Configuration
Read `~/.claude/CLAUDE.md` for user-specific [service] configuration.

## Workflow Patterns

### [Pattern 1]: [Name]
1. Step
2. Step
3. Step

### [Pattern 2]: [Name]
...

## Output Formatting
- Present results in clean markdown tables
- Never dump raw JSON to the user
- Include relevant links

## Error Handling
- If authentication fails: [specific guidance]
- If rate limited: [specific guidance]
```

### Agent Naming

| Pattern | Example | Use when |
|---|---|---|
| `explorer` | `datadog:explorer` | General-purpose agent for an external service |
| `workspace` | `linear:workspace` | Agent that manages a specific workspace/environment |
| Role-based | `security-reviewer` | Agent with a specialized function |

Plugin namespacing (`plugin-name:agent-name`) prevents collisions, so agent names can be generic within their plugin.

### Model Selection

- **`sonnet`** for most subagent work — fast, cost-effective, capable enough for tool orchestration
- **`opus`** for complex reasoning tasks — analysis, planning, multi-step debugging
- **`haiku`** for simple, high-volume operations — formatting, search, quick lookups
- **`inherit`** when the agent should match the user's current model choice

Sources: [Subagents](https://code.claude.com/docs/en/sub-agents)

---

## 9. Complete Example: Designing a New Plugin

Suppose you need a BigQuery plugin. Here's how to think through the structure:

### Step 1: Identify the archetype

BigQuery has an MCP server? Check. Need to run queries, explore schemas, manage datasets? Yes. -> **MCP Subagent Plugin** archetype.

### Step 2: Design the directory

```
bigquery/
.claude-plugin/
  plugin.json
.mcp.json                      # MCP connection to BigQuery
agents/
  explorer.md                  # Query + schema exploration agent
commands/
  init.md                      # Setup wizard (project, datasets, auth)
  bq-query.md                  # Run a query
  bq-schema.md                 # Explore schema
  bq-export.md                 # Export results to CSV
skills/
  bigquery/
    SKILL.md                   # Entry point + index
    reference/
      sql-patterns.md          # Common query patterns
      schema-map.md            # Key tables and relationships
      cost-optimization.md     # Query cost guidance
    scripts/
      validate-query.py        # SQL validation before execution
README.md
```

### Step 3: Define plugin.json

```json
{
  "name": "bigquery",
  "description": "Query, explore, and manage BigQuery datasets with guided workflows",
  "version": "1.0.0",
  "author": { "name": "Data Platform Team" },
  "keywords": ["bigquery", "sql", "database", "analytics", "gcp"]
}
```

### Step 4: Wire the delegation chain

- `/bq-query "SELECT * FROM ..."` -> delegates to `bigquery:explorer` with task "run this query"
- Agent discovers BQ MCP tools via `ToolSearch +bigquery`
- Agent consults `skills/bigquery/reference/sql-patterns.md` for syntax guidance
- Agent validates query using `scripts/validate-query.py` before execution
- Agent formats results as markdown table and returns

---

## 10. Summary: Decision Checklist for New Plugins

| Question | Answer |
|---|---|
| **What archetype?** | MCP subagent (external service), Documentation-as-skill (reference knowledge), or Hook-only (event automation)? |
| **One plugin or multiple?** | One plugin per external service/domain. Don't combine unrelated services. |
| **What goes in the agent vs. skill?** | Agent = workflow logic + tool orchestration. Skill = reference knowledge the agent consults. |
| **What goes in commands?** | Thin delegation layers. <30 lines. Accept args, delegate to agent, done. |
| **Does it need an init wizard?** | Yes if it requires user-specific config (auth, project IDs, preferences). |
| **Where to persist config?** | `~/.claude/CLAUDE.md` for simplicity. Plugin-scoped file for isolation. |
| **Which model for the agent?** | `sonnet` by default. `opus` for complex reasoning. `haiku` for simple lookups. |
| **Should the skill be user-invocable?** | No (`user-invocable: false`) if it's pure reference. Yes if users might want to browse it directly. |
| **Does the agent need persistent memory?** | Yes (`memory: user`) if it benefits from learning across sessions (common queries, user patterns). |
| **How to handle MCP tool discovery?** | `ToolSearch` with service filter, never hardcode tool names. |

---

## Sources

- [Create plugins](https://code.claude.com/docs/en/plugins)
- [Plugins reference](https://code.claude.com/docs/en/plugins-reference)
- [Plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
- [Subagents](https://code.claude.com/docs/en/sub-agents)
- [Skills](https://code.claude.com/docs/en/skills)
- [Hooks](https://code.claude.com/docs/en/hooks)
- [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- Handshake `claude-code-marketplace` (internal, analyzed Feb 2026)
- [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
- [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)
