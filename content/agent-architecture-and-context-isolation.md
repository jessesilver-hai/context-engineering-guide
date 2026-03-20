---
title: "Agent Architecture and Context Isolation"
description: "How agent decomposition solves context management problems in Claude Code and the broader LLM agent ecosystem, covering subagent architecture, agent teams, multi-agent patterns, and state handoff between agents."
tags:
  - context-engineering
  - claude-code
---

# Agent Architecture and Context Isolation

How agent decomposition solves context management problems in Claude Code and the broader LLM agent ecosystem. Companion to [[context-management-for-monolithic-workflows|Context Management for Monolithic Workflows]], which covers state-file patterns for sequential pipelines.

---

## Table of Contents

1. [Agent Files in Claude Code](#1-agent-files-in-claude-code)
2. [Why Agent Isolation Matters](#2-why-agent-isolation-matters)
3. [Claude Code Subagent Architecture](#3-claude-code-subagent-architecture)
4. [Claude Code Agent Teams](#4-claude-code-agent-teams)
5. [The Claude Agent SDK](#5-the-claude-agent-sdk)
6. [Multi-Agent Patterns](#6-multi-agent-patterns)
7. [State Handoff Between Agents](#7-state-handoff-between-agents)
8. [Framework Comparison](#8-framework-comparison)
9. [When to Use Agents vs. Main Context](#9-when-to-use-agents-vs-main-context)
10. [Best Practices for Agent Boundaries](#10-best-practices-for-agent-boundaries)
11. [Sources](#11-sources)

---

## 1. Agent Files in Claude Code

Claude Code uses three types of instruction files that shape agent behavior:

### CLAUDE.md

The foundational configuration file. CLAUDE.md is injected as a system reminder into every conversation. Key properties:

- Lives at project root, user home (`~/.claude/CLAUDE.md`), or project-specific (`~/.claude/projects/`)
- Injected with a system reminder that "this context may or may not be relevant" -- Claude will ignore instructions it deems irrelevant to the current task
- Should be kept to ~30 lines of universally-applicable instructions
- Covers: tech stack, project structure, coding conventions, and behavioral rules
- The *only* file that automatically enters every conversation context

### AGENTS.md

An emerging cross-tool standard (mid-2025) maintained by the Agentic AI Foundation under the Linux Foundation. Supported by Claude Code, Cursor, GitHub Copilot, Gemini CLI, Windsurf, Aider, Zed, Warp, RooCode, and others.

- Standard Markdown, no special schema or YAML frontmatter required
- Drop it anywhere in your project directory tree
- The closest AGENTS.md to the file being edited takes precedence (directory-scoped)
- Complementary to CLAUDE.md -- AGENTS.md provides directory-level instructions while CLAUDE.md provides global instructions

### Custom Subagent Files (.agent.md)

Subagent definition files that create specialized agents invocable from the main conversation. These are the core building block for agent decomposition in Claude Code.

**File location:**
- Project subagents: `.claude/agents/*.agent.md` (checked into version control)
- User subagents: `~/.claude/agents/*.agent.md` (personal, available in all projects)

**File structure:** YAML frontmatter for configuration, followed by the system prompt in Markdown.

```yaml
---
name: code-reviewer
description: "Reviews code changes for security vulnerabilities and style violations."
tools: Read, Grep, Glob, Bash
model: sonnet
permissionMode: default
maxTurns: 15
---

You are a code reviewer specializing in security and style.

When reviewing code:
1. Check for injection vulnerabilities
2. Verify input validation
3. Flag hardcoded secrets
4. Assess adherence to project style guide

Return a structured report with severity levels.
```

**Frontmatter fields:**

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Identifier used to invoke the subagent |
| `description` | Yes | When the agent should be invoked (Claude matches tasks to this) |
| `tools` | No | Explicit allowlist. Omit to inherit parent tools |
| `disallowedTools` | No | Denylist evaluated after `tools` |
| `model` | No | `sonnet`, `opus`, `haiku`, or `inherit` |
| `permissionMode` | No | Controls approval prompts. `dontAsk` denies unlisted tools silently |
| `mcpServers` | No | MCP servers available to this agent |
| `hooks` | No | Lifecycle hooks |
| `maxTurns` | No | Maximum inference turns before forced return |
| `skills` | No | Skills available to this agent |
| `memory` | No | Memory configuration |

**Tool scoping with nested subagents:**
```yaml
tools: Read, Write, Task(rollback)
```
This agent can read, write, and spawn the `rollback` subagent -- but nothing else. The `Task(agent_type)` syntax restricts which subagents an agent can itself spawn.

**Critical constraint:** Subagents cannot spawn their own subagents. Never include `Task` in a subagent's tools without specifying which agent type. The architecture is strictly two-level: parent -> subagent, not parent -> subagent -> sub-subagent.

---

## 2. Why Agent Isolation Matters

### The Context Rot Problem

Context rot is the performance degradation that happens when LLMs process increasingly long input contexts -- even when the context window is not full. A landmark 2025 study analyzing 18 leading LLMs (GPT-4.1, Claude 4, Gemini 2.5, Qwen 3) quantified this phenomenon across models.

Three mechanisms drive context rot:

1. **Lost in the middle**: LLM performance drops by more than 30% when relevant information sits in the middle of the context rather than at the beginning or end
2. **Attention dilution**: As the context window grows, important constraints get buried and the model's tool choices start to drift
3. **Distractor interference**: Semantically similar but irrelevant content interferes with the model's ability to identify what actually matters

Coding agents (and operational agents) are especially vulnerable because they accumulate context during multi-step tasks. Each file read, grep result, API response, and exploration dead-end stays in the context window. The problem is not the context window size -- it is that more tokens means worse performance on complex tasks.

### How Agent Isolation Fixes This

```
WITHOUT ISOLATION (monolithic):
┌─────────────────────────────────────────────────────────┐
│ Context Window (filling up)                              │
│ ┌───────┐┌────────┐┌──────────┐┌─────────┐┌──────────┐ │
│ │Task 1 ││Task 2  ││Task 3    ││Dead ends││Task 4    │ │
│ │results││results ││results   ││& noise  ││results   │ │
│ └───────┘└────────┘└──────────┘└─────────┘└──────────┘ │
│ ⚠ Attention diluted across everything                    │
└─────────────────────────────────────────────────────────┘

WITH ISOLATION (subagents):
┌─────────────────────────────────────────┐
│ Parent Context (stays small)             │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│ │Summary 1 │ │Summary 2 │ │Summary 3 │ │
│ └──────────┘ └──────────┘ └──────────┘ │
│ ✓ Only distilled results in main context │
└─────────────────────────────────────────┘
     ↑              ↑              ↑
┌────┴────┐  ┌──────┴──────┐  ┌───┴─────┐
│Subagent │  │ Subagent    │  │Subagent │
│(fresh   │  │ (fresh      │  │(fresh   │
│context) │  │ context)    │  │context) │
│Tool call│  │Tool call    │  │Tool call│
│Tool call│  │Tool call    │  │Tool call│
│Tool call│  │Tool call    │  │Tool call│
│...      │  │...          │  │...      │
│RETURN:  │  │RETURN:      │  │RETURN:  │
│summary  │  │summary      │  │summary  │
└─────────┘  └─────────────┘  └─────────┘
```

Each subagent gets its own fresh context window. All intermediate tool calls, dead ends, and raw data stay inside the subagent. Only the final summary returns to the parent. This is the single most effective mitigation for context rot in agent workflows.

**Quantified benefit:** Anthropic's multi-agent research system with Claude Opus 4 as lead and Claude Sonnet 4 subagents outperformed single-agent Claude Opus 4 by 90.2% on internal research evaluations. Token usage explains ~80% of the variance -- multi-agent systems work mainly because they help spend enough tokens to solve the problem while keeping each agent's context clean.

**Cost trade-off:** Multi-agent systems consume approximately 15x more tokens than standard single-agent interactions. The pattern is best suited for tasks where outcome quality justifies the expense.

---

## 3. Claude Code Subagent Architecture

### Context Isolation Model

A subagent's context window starts completely fresh -- no parent conversation history. The only channel from parent to subagent is the **Task prompt string**. Any file paths, error messages, prior decisions, or relevant context the subagent needs must be included explicitly in that prompt.

```
Parent Agent
│
├─ Constructs task prompt (the ONLY input to subagent)
│  "Review the file at /src/auth.ts for SQL injection.
│   The project uses Express + Prisma. Check query params
│   in lines 45-80 specifically."
│
└─ Spawns Subagent ──► Fresh context window
                        ├─ System prompt (from .agent.md)
                        ├─ Task prompt (from parent)
                        ├─ Tool call: Read /src/auth.ts
                        ├─ Tool result: [file contents]
                        ├─ Tool call: Grep "req.query"
                        ├─ Tool result: [matches]
                        ├─ ... (all stays INSIDE subagent)
                        └─ RETURN: "Found 2 potential injection
                           points at lines 52 and 71..."
                           ▲
                           │
Parent receives ONLY this final message
```

**Key properties:**
- Intermediate tool calls and results stay inside the subagent; only the final message returns to the parent
- Subagents don't inherit skills from the parent conversation -- tools must be listed explicitly in the `.agent.md` file
- Multiple subagents can run concurrently (parallel execution)
- The architecture is strictly two-level: subagents cannot spawn their own subagents

### The "All or Nothing" Limitation

As of 2025, context isolation is binary. When a task is delegated, the subagent starts with a completely clean slate. There is no mechanism to selectively pass a subset of the parent's context. This forces subagents to either:

1. Perform redundant context-gathering steps the parent already completed
2. Receive all needed context packed into the task prompt string (which has its own size limits)

A scoped context passing feature (partial context inheritance) has been proposed but is not yet implemented. The workaround is to write relevant context to a file and point the subagent at it.

### Model Routing

Each subagent can specify its own model, enabling cost-quality optimization:

```yaml
# Expensive, high-quality analysis
model: opus

# Good balance for most tasks
model: sonnet

# Fast, cheap for simple lookups
model: haiku

# Use whatever the parent is using
model: inherit
```

This allows an architecture where the orchestrator runs on Opus (for complex reasoning about task decomposition) while workers run on Sonnet or Haiku (for straightforward execution).

---

## 4. Claude Code Agent Teams

Agent Teams is an experimental feature (disabled by default) that goes beyond the parent-subagent model to enable peer-to-peer communication between multiple Claude Code sessions.

### How They Differ from Subagents

| Property | Subagents | Agent Teams |
|---|---|---|
| Communication | One-way: subagent -> parent | Multi-directional: any teammate -> any teammate |
| Coordination | Parent acts as intermediary | Shared task list + direct messaging |
| Context | Fresh per subagent | Fresh per teammate, shared via inbox files |
| Discovery | Teammates can't see each other | Teammates aware of each other via config |
| Spawning | Implicit (parent delegates) | Explicit team creation with team lead |

### Architecture

When you create a team, Claude Code creates:
- Directory trees for team configuration
- Inbox files for each agent (created lazily on first message)
- A shared task list
- A team lead role that coordinates work assignment

```
Team Lead (orchestrator session)
├── Assigns tasks from shared task list
├── Reads teammate inboxes for status
├── Synthesizes results
│
├── Teammate A (own session, own context)
│   ├── Reads inbox for assignments
│   ├── Works independently
│   ├── Writes results to own inbox
│   └── Can message Teammate B directly
│
├── Teammate B (own session, own context)
│   ├── Reads inbox for assignments
│   ├── Works independently
│   ├── Writes results to own inbox
│   └── Can message Teammate A directly
│
└── Teammate C (own session, own context)
    └── ...
```

### Best Use Cases

- **Research and review**: Multiple teammates investigate different aspects simultaneously
- **New modules or features**: Each teammate owns a separate component
- **Debugging with competing hypotheses**: Teammates test different theories in parallel
- **Cross-layer coordination**: Changes spanning frontend, backend, and tests

### Enabling Agent Teams

```json
// settings.json or environment variable
{
  "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": true
}
```

---

## 5. The Claude Agent SDK

The Claude Agent SDK (formerly Claude Code SDK, renamed September 2025) enables programmatic creation of AI agents with Claude's capabilities.

### Session Management

A **session** is the conversation history the SDK accumulates while an agent works. It contains prompts, tool calls, tool results, and responses. The SDK writes sessions to disk (`~/.claude/projects/`) automatically.

Three operations for working with sessions:

| Operation | Description | Use Case |
|---|---|---|
| **Continue** | Finds most recent session in current directory | Single-conversation apps |
| **Resume** | Takes a specific session ID | Multi-user apps, returning to specific sessions |
| **Fork** | Creates new session from copy of original's history | Branching exploration, A/B testing approaches |

### Context Compaction

Compaction is the SDK's automatic context management. When token usage exceeds a threshold (~95% capacity), the SDK:

1. Injects a summary prompt as a user turn
2. Claude generates a structured summary (wrapped in summary tags)
3. The SDK replaces the entire message history with that summary
4. The agent continues with a compressed context

In Claude Code, the `/compact` command triggers this manually. Auto-compact triggers at ~95% capacity (25% remaining).

### File Checkpointing

The SDK tracks file modifications made through Write, Edit, and NotebookEdit tools during a session, allowing you to rewind files to any previous state. This enables safe exploration -- an agent can try an approach, and if it fails, revert to a checkpoint.

### Long-Running Agent Pattern

Anthropic's engineering team proposed a two-agent pattern for long-running work:

1. **Initializer Agent**: Sets up the environment, logs what has been done, tracks which files have been added
2. **Coding Agent**: Makes incremental progress in each session and leaves structured artifacts for the next session

The combination of memory files, context compaction, and file checkpointing enables workflows that would otherwise exceed context limits. Tasks persist across context compactions, so even in long sessions with memory resets, the task state survives.

---

## 6. Multi-Agent Patterns

### Pattern 1: Orchestrator-Worker (Hub and Spoke)

The most common pattern. A lead agent decomposes queries, delegates to specialist workers, and synthesizes results.

```
         ┌──────────────┐
         │ Orchestrator  │
         │ (lead agent)  │
         └───┬──┬──┬─────┘
             │  │  │
    ┌────────┘  │  └────────┐
    ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐
│Worker A│ │Worker B│ │Worker C│
│(search)│ │(analyze│ │(format)│
└────────┘ └────────┘ └────────┘
```

**When to use:** Open-ended research, code review with multiple concerns, multi-source data gathering.

**Anthropic's research system** uses this pattern: Claude Opus 4 as lead, Claude Sonnet 4 as workers. The lead decomposes a research query, spawns subagents to explore different aspects simultaneously, then compiles a final answer.

### Pattern 2: Sequential Pipeline

Agents process in a fixed linear order. Each agent's output becomes the next agent's input.

```
┌────────┐    ┌───────────┐    ┌────────────┐    ┌───────────┐
│ Parser │───►│ Extractor │───►│ Summarizer │───►│ Formatter │
│ Agent  │    │ Agent     │    │ Agent      │    │ Agent     │
└────────┘    └───────────┘    └────────────┘    └───────────┘
```

**When to use:** Well-defined transformation pipelines, document processing, data enrichment workflows.

**Google ADK** implements this as `SequentialAgent` -- a classic assembly line where Agent A finishes a task and hands the baton directly to Agent B.

### Pattern 3: Parallel Fan-Out / Fan-In

Multiple agents work on the same input simultaneously, results are aggregated.

```
                 ┌───────────┐
            ┌───►│ Security  │───┐
            │    │ Auditor   │   │
┌────────┐  │    └───────────┘   │    ┌────────────┐
│ Input  │──┤    ┌───────────┐   ├───►│ Aggregator │
│        │  ├───►│ Style     │───┤    │ Agent      │
└────────┘  │    │ Enforcer  │   │    └────────────┘
            │    └───────────┘   │
            │    ┌───────────┐   │
            └───►│Performance│───┘
                 │ Analyst   │
                 └───────────┘
```

**When to use:** Multiple independent analyses of the same artifact, code review from different angles, multi-source data collection.

**Microsoft Azure** documents this as the Fan-out/Fan-in pattern. The critical component is the aggregation function -- custom logic to reconcile multi-branch returns via voting, weighted consolidation, or priority-based selection.

### Pattern 4: Hierarchical (Manager-Worker Trees)

Multi-level delegation where managers coordinate sub-managers who coordinate workers.

```
            ┌──────────┐
            │ Director │
            └────┬─────┘
         ┌───────┼───────┐
         ▼       ▼       ▼
    ┌────────┐┌────────┐┌────────┐
    │Manager ││Manager ││Manager │
    │Frontend││Backend ││Testing │
    └──┬──┬──┘└──┬──┬──┘└──┬──┬──┘
       │  │      │  │      │  │
       ▼  ▼      ▼  ▼      ▼  ▼
      W1  W2    W3  W4    W5  W6
```

**When to use:** Large-scale projects with natural organizational boundaries. Rare in practice due to the constraint that Claude Code subagents cannot spawn sub-subagents (strictly two-level). Agent Teams partially addresses this with peer-to-peer communication.

### Pattern 5: Handoff Chain (Routing/Transfer)

An active conversation transfers from one specialist agent to another, carrying full conversation history.

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Triage   │───►│ Billing  │───►│ Escalate │
│ Agent    │    │ Agent    │    │ Agent    │
└──────────┘    └──────────┘    └──────────┘
     Chat history transfers with each handoff
```

**When to use:** Customer support flows, multi-domain queries where only one specialist is active at a time.

**OpenAI Swarm** pioneered this pattern with simple `transfer_to_XXX` functions. The system prompt changes on handoff but chat history persists. Now superseded by the OpenAI Agents SDK.

### Pattern 6: Loop (Iterative Refinement)

An agent runs repeatedly until a quality threshold is met or max iterations are reached.

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Generate │───►│ Evaluate │───►│ Refine   │──┐
│ Agent    │    │ Agent    │    │ Agent    │  │
└──────────┘    └──────────┘    └──────────┘  │
                     ▲                         │
                     └─────────────────────────┘
                     (loop until quality >= threshold
                      or max_iterations reached)
```

**When to use:** Content generation with quality gates, code generation with test validation, iterative debugging.

**Google ADK** implements this as `LoopAgent` with `max_iterations` and an early-exit signal (`escalate=True`).

---

## 7. State Handoff Between Agents

### The Fundamental Problem

Agents operate in isolated contexts. State must cross the boundary somehow. There are four approaches, each with different trade-offs:

### Approach 1: Task Prompt (In-Band)

Pack everything the subagent needs into the task prompt string.

```
Pros: Simple, no external dependencies
Cons: Limited by prompt size, no structured data
Best for: Small, self-contained tasks
```

### Approach 2: Filesystem (Out-of-Band)

Write state to a structured file. Point the subagent at it.

```
Parent writes: /tmp/state/phase-2-input.json
Subagent reads: /tmp/state/phase-2-input.json
Subagent writes: /tmp/state/phase-2-output.json
Parent reads: /tmp/state/phase-2-output.json

Pros: Unlimited size, structured data, persistent
Cons: Requires file I/O tools, coordination overhead
Best for: Complex multi-phase workflows
```

This is the pattern recommended in the companion doc [[context-management-for-monolithic-workflows|Context Management for Monolithic Workflows]].

### Approach 3: Memory Files (Persistent)

Use CLAUDE.md, memory files, or the SDK's memory tool to persist state across sessions.

```
Pros: Survives context compaction, available across sessions
Cons: Unstructured, can accumulate stale data
Best for: Long-running projects, cross-session continuity
```

### Approach 4: Session Fork (SDK)

Fork a session to create a new branch with full history, then let it diverge.

```
Pros: Full context inheritance, clean branching
Cons: Copies entire history (expensive), SDK-only
Best for: Exploring alternative approaches, A/B testing
```

### Checkpoint and Serialization Patterns

**LangGraph** offers the most mature checkpoint system:
- State persisted to a database using a checkpointer after each node execution
- `thread_id` provides isolation between user threads
- Time Travel feature enables replaying from any checkpoint for debugging
- TypedDict or Pydantic models for structured state

**Letta** introduced the Agent File (`.af`) format: an open file format for serializing stateful agents with persistent memory and behavior.

**Multi-agent memory protocols** (from recent research): The simplest mechanism for shared state is serialized turns -- agents act one at a time in a fixed order, each reading the latest memory and then writing its updates. More sophisticated approaches use explicit read/write locks or event-driven updates.

---

## 8. Framework Comparison

### Context Isolation and State Management

| Framework | Context Model | State Persistence | Agent Communication | Parallel Execution |
|---|---|---|---|---|
| **Claude Code Subagents** | Complete isolation (fresh context per subagent) | Filesystem + session files | One-way (subagent -> parent) | Yes (concurrent subagents) |
| **Claude Code Agent Teams** | Complete isolation (fresh context per teammate) | Shared inbox files + task list | Multi-directional (peer-to-peer) | Yes (independent sessions) |
| **Claude Agent SDK** | Session-based with compaction | Disk-persisted sessions, fork/resume | Via session management | Via programmatic orchestration |
| **LangGraph** | Graph state (shared typed state object) | Checkpointer (DB-backed) | Shared state + edges | Yes (independent nodes execute in parallel) |
| **CrewAI** | Per-role isolation with shared crew store | SQLite-backed crew store | Role-based delegation + shared memory | Sequential or parallel crews |
| **AutoGen** | Centralized transcript (shared chat) | Aggressive pruning + external stores | Group chat with turn-taking | Limited (chat-based coordination) |
| **OpenAI Agents SDK** | Handoff-based (system prompt changes, chat history persists) | Stateless between calls | Transfer functions | Via orchestration layer |
| **Google ADK** | Shared session state across agents | Session state dict (shared keys) | Sequential, Parallel, or Loop agents | Yes (ParallelAgent primitive) |

### Error Recovery

| Framework | Recovery Approach |
|---|---|
| **LangGraph** | Most deterministic: checkpoint-based replay from any node |
| **CrewAI** | Pragmatic: per-role isolation limits blast radius |
| **AutoGen** | Flexible but chat-heavy: repair through conversation |
| **OpenAI Agents SDK** | Lightweight: works until you need complex compensation |
| **Claude Code** | File checkpointing for reverting changes; session resume for continuing |
| **Google ADK** | Loop agents with exit conditions; session state rollback |

### Cost-Quality Trade-offs

| Framework | Model Routing | Token Efficiency |
|---|---|---|
| **Claude Code Subagents** | Per-subagent model selection (opus/sonnet/haiku) | High (fresh context per subagent = no waste) |
| **LangGraph** | Per-node model selection | Medium (shared state can accumulate) |
| **CrewAI** | Per-agent model selection | Medium (shared crew store) |
| **AutoGen** | Per-agent model selection | Lower (centralized transcript grows) |
| **Google ADK** | Per-agent model selection | High (parallel agents share session state efficiently) |

---

## 9. When to Use Agents vs. Main Context

### Keep It in Main Context When:

- **The task is linear and short** (< 10 tool calls, < 30K tokens of context)
- **Every step depends on the previous step's full output** -- the compression loss from subagent summarization would hurt quality
- **You need real-time user interaction** at each step
- **The overhead of agent decomposition exceeds the task complexity** -- don't use a multi-agent system for a simple file edit
- **Context coherence matters more than context freshness** -- some tasks need the model to "see" the full trajectory of reasoning

### Use Subagents When:

- **Independent subtasks exist** -- research, review, and formatting can happen in parallel without dependency
- **Context is accumulating waste** -- grep results, file reads, and dead-end explorations are polluting the main context
- **You need specialist focus** -- a security review should not be distracted by style guide contents
- **The main context is approaching capacity** -- subagents get fresh windows
- **Multiple data sources need harvesting** -- Slack, BigQuery, Sheets, Notion can all be queried in parallel subagents

### Use Agent Teams When:

- **Peer coordination is needed** -- agents need to share discoveries mid-task, not just report back to a parent
- **Cross-cutting concerns span the work** -- frontend, backend, and test changes need awareness of each other
- **Competing hypotheses should be explored** -- different agents can test different theories and compare notes
- **The task naturally decomposes into semi-autonomous workstreams** with occasional synchronization

### Decision Flowchart

```
Is the task < 10 tool calls and < 30K tokens?
├── Yes → Main context. No agents needed.
└── No
    ├── Are subtasks independent?
    │   ├── Yes → Do subtasks need to communicate with each other?
    │   │   ├── Yes → Agent Teams
    │   │   └── No → Parallel Subagents
    │   └── No (sequential dependencies)
    │       ├── Is intermediate state too large for prompt?
    │       │   ├── Yes → Sequential subagents with filesystem state handoff
    │       │   └── No → Sequential subagents with task prompt handoff
    │       └── Will total context exceed window?
    │           ├── Yes → Subagents (mandatory for context management)
    │           └── No → Consider main context with /compact as fallback
    └── Is the work expected to span multiple sessions?
        └── Yes → Agent SDK sessions with resume/fork
```

---

## 10. Best Practices for Agent Boundaries

### 1. Start Simple, Add Agents When Needed

Do not build a nested multi-agent system on day one. Start with a single agent. Add subagents when you hit a specific problem: context rot, task independence, or the need for parallel execution. Measure whether agents actually improve outcomes.

### 2. Define Clear Contracts

Each subagent needs:
- **Objective**: What it should accomplish (not how)
- **Output format**: Structured response the parent can parse
- **Tool guidance**: Which tools and sources to use
- **Boundaries**: What is out of scope (prevents scope creep and wasted tokens)

Without detailed task descriptions, agents duplicate work, leave gaps, or fail to find necessary information.

### 3. Embed Effort Scaling Rules

Agents struggle to judge appropriate effort for different tasks. Embed scaling rules in prompts:

```markdown
## Effort Scaling
- Simple lookup: 1-3 tool calls, return immediately
- Standard analysis: 5-10 tool calls, check 2-3 sources
- Deep research: 15-25 tool calls, cross-reference multiple sources
- Exhaustive investigation: up to maxTurns, leave no stone unturned
```

### 4. Minimize Shared Context

Treat shared context as an expensive dependency. Only share the full memory/context history when the sub-agent must understand the entire trajectory of the problem to function. For most tasks, a focused task prompt is sufficient and produces better results than a context dump.

Forking context breaks the prompt cache -- each subagent with unique context starts a new cache line.

### 5. Use Model Routing Deliberately

```
Opus  → Orchestration, complex reasoning, ambiguous task decomposition
Sonnet → Execution, tool use, standard analysis
Haiku  → Simple lookups, formatting, single-field extraction
```

An orchestrator on Opus decomposing work for Sonnet workers is the most cost-effective pattern for complex tasks.

### 6. Design for Observability

Agent systems are harder to debug than single-agent systems. Design for it:
- Have each agent return structured summaries (not just prose)
- Include agent name/role in outputs
- Log which agents were spawned and what they were asked
- Use filesystem state files as an audit trail

### 7. Handle Failures Gracefully

Subagents can fail (bad tool calls, context exhaustion, hallucinated results). The parent should:
- Validate subagent outputs before synthesizing
- Have fallback strategies (retry with different prompt, escalate to user)
- Set `maxTurns` to prevent runaway agents
- Use file checkpointing to revert changes from failed agents

---

## 11. Sources

### Anthropic Official

- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Building Effective AI Agents](https://www.anthropic.com/research/building-effective-agents)
- [Work with sessions - Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/sessions)
- [Subagents in the SDK - Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/subagents)
- [Compaction - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/compaction)
- [Rewind file changes with checkpointing - Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/file-checkpointing)
- [How Claude Code works - Claude Code Docs](https://code.claude.com/docs/en/how-claude-code-works)

### Context Engineering and Context Rot

- [Context Engineering - LangChain Blog](https://blog.langchain.com/context-engineering-for-agents/)
- [Context Rot: How Increasing Input Tokens Impacts LLM Performance - Chroma Research](https://research.trychroma.com/context-rot)
- [Context rot: the emerging challenge - Understanding AI](https://www.understandingai.org/p/context-rot-the-emerging-challenge)
- [Context rot explained & how to prevent it - Redis](https://redis.io/blog/context-rot/)
- [The Context Window Problem: Scaling Agents Beyond Token Limits - Factory.ai](https://factory.ai/news/context-window-problem)
- [Context Management with Subagents in Claude Code - RichSnapp](https://www.richsnapp.com/article/2025/10-05-context-management-with-subagents-in-claude-code)
- [Context Engineering for AI Agents: Part 2 - Phil Schmid](https://www.philschmid.de/context-engineering-part-2)

### Framework Documentation and Comparisons

- [Developer's guide to multi-agent patterns in ADK - Google Developers Blog](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)
- [AI Agent Orchestration Patterns - Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [AutoGen vs. CrewAI vs. LangGraph vs. OpenAI AI Agents Framework - Galileo](https://galileo.ai/blog/autogen-vs-crewai-vs-langgraph-vs-openai-agents-framework)
- [CrewAI vs LangGraph vs AutoGen - DataCamp](https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen)
- [LangGraph vs AutoGen vs CrewAI - Latenode](https://latenode.com/blog/platform-comparisons-alternatives/automation-platform-comparisons/langgraph-vs-autogen-vs-crewai-complete-ai-agent-framework-comparison-architecture-analysis-2025)
- [Orchestrating Agents: Routines and Handoffs - OpenAI Cookbook](https://cookbook.openai.com/examples/orchestrating_agents)
- [OpenAI Swarm - GitHub](https://github.com/openai/swarm)
- [LangGraph: Agent Orchestration Framework](https://www.langchain.com/langgraph)

### Practical Guides and Community

- [Best practices for Claude Code subagents - PubNub](https://www.pubnub.com/blog/best-practices-for-claude-code-sub-agents/)
- [Claude Agent SDK Best Practices - Skywork](https://skywork.ai/blog/claude-agent-sdk-best-practices-ai-agents-2025/)
- [Claude Agent SDK: Subagents, Sessions and Why It's Worth It](https://www.ksred.com/the-claude-agent-sdk-what-it-is-and-why-its-worth-understanding/)
- [Awesome Claude Code Subagents - VoltAgent/GitHub](https://github.com/VoltAgent/awesome-claude-code-subagents)
- [Claude Code Sub-Agent Collective - GitHub](https://github.com/vanzan01/claude-code-sub-agent-collective)
- [From Tasks to Swarms: Agent Teams in Claude Code - alexop.dev](https://alexop.dev/posts/from-tasks-to-swarms-agent-teams-in-claude-code/)
- [Claude Code Agent Teams: How They Work Under the Hood](https://www.claudecodecamp.com/p/claude-code-agent-teams-how-they-work-under-the-hood)
- [Writing a good CLAUDE.md - HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [AGENTS.md becomes the convention](https://pnote.eu/notes/agents-md/)
- [Feature Request: Scoped Context Passing for Subagents - GitHub Issue #4908](https://github.com/anthropics/claude-code/issues/4908)
- [Feature Request: Support AGENTS.md - GitHub Issue #6235](https://github.com/anthropics/claude-code/issues/6235)

### Research and Deep Dives

- [Best practices for building AI multi agent systems - Vellum](https://www.vellum.ai/blog/multi-agent-systems-building-with-context-engineering)
- [A Practical Guide for Designing, Developing, and Deploying Production-Grade Agentic AI Workflows - arXiv](https://arxiv.org/html/2512.08769v1)
- [Memory Blocks: The Key to Agentic Context Management - Letta](https://www.letta.com/blog/memory-blocks)
- [Stateful Agents: The Missing Link in LLM Intelligence - Letta](https://www.letta.com/blog/stateful-agents)
- [Checkpoint/Restore Systems: Applications in AI Agents - Eunomia](https://eunomia.dev/blog/2025/05/11/checkpointrestore-systems-evolution-techniques-and-applications-in-ai-agents/)
- [Debugging Non-Deterministic LLM Agents: Checkpoint-Based State Replay with LangGraph](https://dev.to/sreeni5018/debugging-non-deterministic-llm-agents-implementing-checkpoint-based-state-replay-with-langgraph-5171)
- [How Anthropic Built a Multi-Agent Research System - ByteByteGo](https://blog.bytebytego.com/p/how-anthropic-built-a-multi-agent)
- [A Practical Guide to Building Agents - OpenAI (PDF)](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)
