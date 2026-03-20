---
title: "Context Management for Monolithic Workflows"
description: "How to manage context in complex, multi-step operational workflows that consume excessive context and degrade model performance. Covers state-file patterns, progressive summarization, observation masking, and recursive decomposition."
tags:
  - context-engineering
  - claude-code
---

# Context Management for Monolithic Workflows

**Problem**: Monolithic skills/commands that walk users through complex, multi-step operational workflows consume excessive context and degrade model performance as the conversation grows.

**Core tension**: The workflow *is* sequential — you can't just chop a 9-phase QC pipeline into 9 independent skills because phases depend on each other. But carrying full conversation history through all phases causes ~39% performance degradation in multi-turn workflows (Laban et al., 2025). Every token added to the context window competes for the model's attention — the agent doesn't forget because it ran out of space, it forgets because signal gets drowned by accumulation.

---

## The Fix: State Files, Not Context

The key insight from context engineering research (Anthropic, Manus, LangChain, Google, FlowHunt): **externalize intermediate state to structured files, not the conversation history.**

Instead of one long conversation accumulating every tool result, each phase:
1. Reads a structured state file (what happened so far)
2. Does its work in a scoped context (subagent or fresh prompt)
3. Writes its results back to the state file
4. Returns only a summary to the parent

This is the pattern Manus calls "file system as external memory" — the file system is unlimited, persistent, and directly manipulable by the agent. Rather than irreversible context truncation, this enables "recoverable compression": web page content can be dropped if the URL is preserved, document contents can be omitted if file paths remain accessible, and all compression maintains the ability to restore information when needed.

---

## Pattern 1: Phase-per-Subagent with Structured Handoff

Each phase runs in its own Agent call with only:
- The structured state file
- Phase-specific instructions

The parent orchestrator reads the state file between phases and decides what's next. Context stays small per phase.

**Example**: GSD already does this — `PLAN.md`, `VERIFICATION.md`, checkpoint files are all structured state handoff.

**Claude Code specifics**: Each subagent (via the Task tool) gets its own isolated 200K context window. Intermediate tool calls and results stay inside the subagent; only its final message returns to the parent. This means a subagent that processes 150K tokens of logs returns only a 500-token summary to the orchestrator — a 300:1 compression ratio on the parent's context.

**Best practices for subagents**:
- Give each subagent one clear goal, input, output, and handoff rule
- Grant only the minimum set of tools/permissions required for the subagent's role
- Use Haiku for exploration subagents and Sonnet/Opus for implementation — reduces costs 40-50%
- Even without custom subagents, Claude can spawn the built-in general-purpose subagent when Task is in allowedTools

## Pattern 2: Progressive Summarization at Phase Boundaries

After each phase completes, summarize its output into 3-5 lines and discard the raw tool results. This is the "sliding window + prioritization" approach — keep recent detail, compress the past.

**When to use**: When phases are tightly coupled and subagent isolation is overkill, but you still need to prevent context bloat.

**Implementation detail (Factory.ai findings)**: Factory.ai tested three summarization approaches on 36,000+ production messages and found that structured summarization (incrementally maintaining context around intent, changes made, decisions taken, and next steps) scored 3.70/5.0 overall vs. 3.44 for Anthropic's built-in compression and 3.35 for OpenAI's. The key insight: naive optimization for compression ratio (minimizing tokens per request) often *increases* total tokens per task because agents lose critical context and must re-fetch files, re-read documentation, and re-explore previously rejected approaches. Optimize at the **task level**, not the request level.

**Manus's approach**: Constantly rewriting the todo list pushes global objectives into the end of the context, exploiting the recency bias to avoid "lost-in-the-middle" issues and reducing goal misalignment. This is a form of "attention management through recitation."

## Pattern 3: Scoped Context Injection (Progressive Disclosure)

Instead of one giant skill prompt that covers all phases, load only the current phase's instructions. Claude Code's skill descriptions are part of the always-on context budget, so you should inject phase-specific guidance, not the whole workflow.

**Implementation**: Structure the skill folder with per-phase instruction files. The orchestrator loads only the relevant phase file at each step.

**The instruction budget problem**: Research shows frontier LLMs reliably follow approximately 150-200 instructions before quality degrades (Jaroslawicz et al., 2025). Claude Code's own system prompt already consumes a significant chunk of that budget before your CLAUDE.md is even read. Models exhibit distinct degradation patterns: *threshold decay* (near-perfect then cliff — reasoning models like o3, Gemini 2.5 Pro), *linear decay* (GPT-4.1, Claude Sonnet 4), and *exponential decay* (GPT-4o, Llama 4 Scout). At 500 instructions, even the best frontier models only achieve 68% accuracy.

**Implication**: CLAUDE.md files and skill descriptions should be ruthlessly pruned. Every instruction must earn its place.

## Pattern 4: DAG Decomposition

Model the workflow as a directed graph:
- Phases that don't depend on each other run in parallel agents
- Phases that do depend run sequentially but with explicit state handoff, not implicit context accumulation

**When to use**: When your workflow has independent branches (e.g., "gather data from Slack" and "pull metrics from BQ" can run in parallel before "analyze results").

**Claude Code implementation**: Multiple subagents can run concurrently, dramatically speeding up complex workflows. Each parallel branch gets its own isolated context window and only sends relevant results back to the orchestrator.

## Pattern 5: Observation Masking (The Complexity Trap)

Instead of summarizing the full conversation history, selectively mask environment observations (tool outputs) while preserving the action and reasoning history in full. This makes sense because a typical software engineering agent's turn heavily skews toward observation — file contents, command outputs, API responses — which can be dropped without losing the agent's reasoning chain.

**Research backing**: JetBrains' "The Complexity Trap" (NeurIPS 2025 DL4Code Workshop) found that simple observation masking halves cost relative to the raw agent while matching, and sometimes slightly exceeding, the solve rate of LLM summarization. A hybrid approach (masking + selective summarization) further reduces costs by 7-11%.

**When to use**: When you need cost reduction without the latency or risk of LLM summarization. Especially effective for software engineering agents where tool outputs are verbose but the reasoning chain is the critical thread.

## Pattern 6: Hierarchical Action Space

Reduce context confusion by tiering the available tools:
- **Level 1 (Atomic)**: ~20 core tools (file_write, browser_navigate, bash, search). Stable and cache-friendly.
- **Level 2 (Compound)**: Higher-level operations composed from atomic tools, loaded only when relevant.

This is how Manus manages tool proliferation — keeping the base action space small preserves KV-cache hits and reduces decision complexity.

## Pattern 7: Recursive Self-Invocation (RLMs)

For workflows that genuinely exceed any single context window, Recursive Language Models (Zhang et al., 2025) allow the LLM to programmatically decompose tasks and recursively call itself over subsets of the input.

**How it works**: The prompt is loaded into an external environment (e.g., Python REPL). The LLM writes code to peek at only necessary parts and delegates subtasks to new instances of itself through recursive self-invocation.

**Results**: RLMs process inputs up to two orders of magnitude beyond model context windows without performance degradation, even at 10M+ tokens. They dramatically outperform vanilla frontier LLMs and common long-context scaffolds across diverse tasks with comparable cost.

**When to use**: When the input data itself is massive (entire codebases, years of logs, large document collections) — not just when the conversation gets long.

## Pattern 8: Agentic Context Engineering (ACE)

Rather than treating context as a static prompt, ACE (Zhang et al., 2025) maintains a persistent, structured "playbook" that evolves through iterative refinement.

**Architecture**: Three specialized components:
- **Generator**: Produces reasoning trajectories for new queries
- **Reflector**: Critiques traces to extract lessons (separated from curation to avoid monolithic rewriting)
- **Curator**: Synthesizes lessons into compact delta entries, merged deterministically into existing context by lightweight, non-LLM logic

**Key design principle**: Context is represented as a collection of structured, itemized bullets — not a single monolithic prompt. Incremental delta updates replace costly monolithic rewrites with localized edits. A grow-and-refine mechanism balances steady context expansion with redundancy control.

**Results**: On the AppWorld leaderboard, ReAct + ACE matched the top-ranked system (IBM CUGA) despite using a smaller open-source model, and surpassed it by 8.4% on the harder test-challenge split with online adaptation.

**Why monolithic rewriting fails**: An LLM tasked with fully rewriting accumulated context tends to compress it into much shorter, less informative summaries, causing dramatic information loss ("context collapse").

---

## Architecture Summary

```
User runs /my-workflow
    │
    ▼
Orchestrator (owns workflow graph + state transitions)
    │
    ├── Phase 1 (subagent) ──▶ writes state.json
    │
    ├── Orchestrator reads state.json, summarizes Phase 1
    │
    ├── Phase 2 (subagent) ──▶ reads state.json, writes updated state.json
    │
    ├── Orchestrator reads state.json, summarizes Phase 2
    │
    └── ... continues until complete
```

Each subagent receives:
- Structured state file (not full history)
- Phase-specific instructions (not the whole skill prompt)
- Just enough prior context to do its job

The workflow is still one command from the user's perspective. The context management is invisible.

---

## Claude Code Specifics

### Context Window Mechanics
- Main sessions: 200K context window
- Subagents (Task tool): Each gets an isolated 200K context window
- Auto-compact triggers at ~83.5% capacity (~33K-45K token buffer remaining)
- Compaction process: clears older tool outputs first, then summarizes the conversation; requests and key code snippets are preserved but detailed instructions from early in the conversation may be lost
- Manual `/compact` is recommended between tasks to avoid unrelated work accumulating
- Since v2.0.64, compacting is instant — no blocking

### Skills and Context Budget
- Skill descriptions are always-on context — they consume tokens even when not invoked
- Skills use progressive disclosure: Claude loads information only as needed based on description matching
- CLAUDE.md is always loaded — keep it lean and high-signal
- Tool responses are restricted to 25,000 tokens by default

### KV-Cache Optimization
- Production agents exhibit input/output ratios of 100:1 or higher
- KV-cache hit rate is the single most important metric for production agents
- Cached token cost is 10x lower than uncached ($0.30 vs $3.00 per million tokens)
- Keep prompt prefixes stable — even a single-token difference invalidates the cache from that token onward
- Place constant/rarely-changing information at the beginning of prompts; dynamic content at the end

### Hooks for Context Control
- PreToolUse hooks can validate operations before they execute, useful for intercepting and filtering tool outputs that would bloat context
- Hooks can enforce context hygiene automatically (e.g., truncating verbose outputs, filtering irrelevant file contents)

---

## Key Research Findings

### Context Rot (Chroma, July 2025)
Context rot is the degradation in LLM performance that occurs as input context length increases — models produce less accurate, less reliable outputs when processing longer inputs, even when the context window isn't full. Chroma evaluated 18 state-of-the-art models (GPT-4.1, Claude 4, Gemini 2.5, Qwen3) and found that model reliability decreases significantly with longer inputs, even on simple tasks like retrieval and text replication.

Three mechanisms drive context rot:
1. **Lost-in-the-middle**: Models attend strongly to tokens at the beginning and end but poorly to the middle
2. **Quadratic attention scaling**: More tokens exponentially increase pairwise relationships the model must track
3. **Semantic interference**: Semantically similar distractors interfere with identifying relevant information

As needle-question similarity decreases, performance degrades more significantly with increasing input length. Different model families exhibit unique behaviors — OpenAI's GPT series often shows erratic and inconsistent outputs, while others degrade more predictably.

### Multi-Turn Degradation (Laban et al., May 2025)
"LLMs Get Lost In Multi-Turn Conversation" (arXiv:2505.06120) — a Microsoft/Salesforce study analyzing 200,000+ simulated conversations across six generation tasks (Code, Database, Actions, Data-to-text, Math, Summary). Key findings:
- **39% average performance drop** from single-turn to multi-turn across all top open- and closed-weight LLMs
- Degradation decomposes into a minor loss in aptitude and a **significant increase in unreliability**
- LLMs make assumptions in early turns and prematurely attempt final solutions, then overly rely on those attempts
- **When LLMs take a wrong turn, they do not recover** — errors compound rather than self-correct
- OpenAI's o3 fell from 98.1% to 64.1% accuracy in multi-turn settings

A follow-up study (arXiv:2602.07338, Feb 2026) identified intent mismatch as a key causal mechanism.

### Lost-in-the-Middle (Liu et al., 2023; TACL 2024)
The foundational paper showing that LLMs miss information buried in the middle of long contexts, even when it's technically "in context." Performance is highest when relevant information occurs at the beginning or end of the input, and significantly degrades in the middle — even for explicitly long-context models. This means phase 3 results may be invisible to phase 7 even if they're all in the same conversation.

### Instruction-Following Degradation (Jaroslawicz et al., July 2025)
"How Many Instructions Can LLMs Follow at Once?" (arXiv:2507.11538) — evaluated with IFScale benchmark using up to 500 keyword-inclusion instructions:
- Primacy effects peak around 150-200 instructions, then level off
- Three degradation patterns: threshold decay (reasoning models), linear decay (Claude Sonnet 4, GPT-4.1), exponential decay (GPT-4o, Llama 4 Scout)
- At 500 instructions, even the best frontier models only achieve 68% accuracy
- At 300+ instructions, models shift from selective instruction satisfaction to uniform failure

### The Complexity Trap (Lindenbauer et al., NeurIPS 2025)
"Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management" — JetBrains Research found that simple observation masking (dropping old tool outputs while keeping reasoning history) halves cost while matching or slightly exceeding the solve rate of LLM summarization on SWE-bench Verified. A hybrid approach further reduces costs by 7-11%.

### ACON: Optimizing Context Compression (Kang et al., October 2025)
arXiv:2510.00615 — a unified framework for compressing both environment observations and interaction histories:
- Reduces memory usage by **26-54% (peak tokens)** while preserving task performance
- Preserves over 95% of accuracy when distilled into smaller compressors
- Enhances smaller LMs as long-horizon agents with up to 46% performance improvement
- Validated on three multi-step agent benchmarks requiring 15+ interaction steps
- Uses gradient-free optimization of natural language compression guidelines

### Factory.ai Production Findings (2025)
Tested compression strategies on 36,000+ production messages across debugging, PR review, feature implementation, CI troubleshooting:
- Structured summarization scored **3.70/5.0** vs. Anthropic (3.44) and OpenAI (3.35)
- Key principle: optimize at the **task level** (total tokens to complete the task), not request level (tokens per API call)
- Naive compression ratio optimization often increases total tokens because agents re-fetch and re-explore

### Recursive Language Models (Zhang et al., December 2025)
arXiv:2512.24601 — treat the prompt as an object in an external environment:
- Process inputs up to **two orders of magnitude beyond** model context windows
- No performance degradation at 10M+ tokens
- Outperform vanilla frontier LLMs and common long-context scaffolds
- Approach: LLM writes code to programmatically construct sub-tasks on which it invokes itself recursively

### Token Budget ROI
~65% of enterprise AI failures in 2025 were attributed to context drift or memory loss during multi-step reasoning. Smart compression can reduce context by 68% while retaining 91% of critical information.

---

## Vendor-Specific Guidance

### Anthropic (Claude / Claude Code)
- **Effective Context Engineering for AI Agents** (Sept 2025): System prompts should use clear, direct language at the right altitude. Tool descriptions should be written as if describing to a new hire. Implement pagination, filtering, and truncation with sensible defaults. Tool responses capped at 25K tokens.
- **Effective Harnesses for Long-Running Agents** (Nov 2025): Two-agent harness — an initializer agent sets up the environment (log file + initial commit), then a coding agent works in subsequent sessions making incremental progress. Best approach: commit progress to git with descriptive messages and write summaries to a progress file for cross-session continuity.
- **Agent Skills** (2025): Progressive disclosure is the core design principle. Skills are organized folders of instructions, scripts, and resources that agents discover and load dynamically based on description matching.
- **Writing Effective Tools for Agents** (2025): Prompt-engineering tool descriptions is one of the most effective methods for improving tool performance. Descriptions are loaded into agents' context and steer tool-calling behavior.

### OpenAI (Codex / Agents SDK)
- **Codex**: Operates in isolated cloud sandboxes with ~192K context window (experimental 1M with GPT-5.4). Uses AGENTS.md files for persistent instructions and Skills for reusable workflow bundles.
- **Agents SDK**: Provides agents with clear instructions and built-in tools, handoffs for control transfer between agents, guardrails for input/output validation, and tracing for observability.
- **AgentKit**: Exposes runtime events and guardrails signals for context management.

### Google (Gemini)
- **Context caching**: Upload reusable context once, then reference in subsequent calls to reduce tokenization cost.
- **Gemini Interactions API** (beta, 2026): Server-side context handling for long-running interactions managed centrally, with background execution for independent task continuation.
- **Gemini 2.5/3**: 1M+ token context windows with context caching support. Deep Research agent handles large context dumps.

### Manus
- **Context Engineering Lessons** (rebuilt framework 4 times):
  - KV-cache hit rate is the #1 production metric (100:1 input/output ratio)
  - File system as external memory — unlimited, persistent, agent-manipulable
  - Attention management through recitation (rewriting todo lists)
  - Preserve failure information — seeing its own mistakes helps the agent avoid repeating them
  - Multi-agent context isolation — sub-agents isolate context, not just parallelize work
  - Hierarchical action space — ~20 core tools at Level 1, compound operations loaded on demand

---

## Framework Patterns

### LangGraph (Graph-Based Workflows)
- Workflows as stateful graphs: nodes (functions/agents), edges (transitions), cycles (controlled loops for reflection/retry)
- Explicit state management with reducer logic to merge concurrent updates
- Superior for workflows requiring conditional branching, cyclical patterns, or precise state management
- State persistence built in — each agent reads from and writes to a central state object

### CrewAI (Role-Based Teams)
- Models multi-agent collaboration as a "crew" of role-playing agents
- Each agent has a role, backstory, and goal; agents can delegate to each other
- Shared crew context across all agents
- Best for linear or parallel task execution with role-based specialization

### AutoGen (Conversational Multi-Agent)
- Structured conversations where specialized agents exchange messages
- Conversational model — agents coordinate through message passing
- Good for tasks that naturally decompose into dialogue-like exchanges

### MemGPT / Letta (Virtual Context Management)
- Inspired by OS virtual memory paging — main context (RAM) and external context (disk)
- Three-tier architecture: Core Memory (always-accessible compressed representation), Recall Memory (searchable session database), Archival Memory (long-term vector storage)
- LLM manages its own memory autonomously via tool calls — decides what to store, summarize, or forget
- Memory operations exposed as tools: store, retrieve, update, summarize, discard

### Selection Guidance (2026)
- **LangGraph**: Complex workflows with cycles, conditional branching, parallel fan-out, precise state management
- **CrewAI**: Linear/parallel task execution with role-based agents, simpler setup
- **AutoGen**: Conversational coordination patterns, research prototyping
- **MemGPT/Letta**: Perpetual conversations, long-term memory persistence across sessions

---

## The "Context Stuffing" Anti-Pattern

Context stuffing — shoving everything into the prompt — is the single largest source of wasted compute, hallucinated outputs, and unreliable agent behavior in production systems. With ever-larger context windows, the temptation is to "shove it all in," but:

- **Token bloat != signal**: More text means more room for distractors. LLMs hedge; answers go fuzzy.
- **Lost-in-the-middle**: Models recall the beginning and end far more reliably than anything buried in the middle.
- **Latency explosion**: 4x the tokens often means ~2-3x response delay. Your p95 becomes your p50.
- **Cache invalidation**: Changing tokens early in the prompt invalidates the entire KV-cache from that point forward.

### Alternatives to Context Stuffing

1. **Just-In-Time (JIT) Context Retrieval**: Carry lightweight identifiers (file paths, database indices, API endpoint references). When execution reaches a step that needs specific information, retrieve exactly that chunk, use it, and let it be compacted or dropped from subsequent turns.

2. **Context Deduplication and Differencing**: Identical stack traces, repeated log lines, unchanged diff sections don't need to appear multiple times. Identify duplicates, collapse them, highlight only what's new.

3. **Relevance Ranking and Selective Curation**: Tier all potential information sources by relevance. Build templates that fit within a sensible token budget — not the maximum window size.

4. **Multi-Agent Context Isolation**: Specialized sub-agents handle specific tasks with their own tools, instructions, and context windows. Anthropic's research shows many agents with isolated contexts outperform single-agent implementations.

5. **Observation Masking**: Target environment observations only, preserving action and reasoning history in full. Agent turns heavily skew toward observation (tool outputs), which can be dropped without losing the reasoning thread.

6. **Recursive Decomposition (RLMs)**: For truly massive inputs, let the LLM programmatically construct sub-tasks and recursively invoke itself over subsets, treating the prompt as an external environment object.

---

## The Mantra

> "Every token must earn its place." — FlowHunt Context Engineering Guide

The prompt is not a container for everything the model might need. It is a strict, scoped runtime environment.

---

## Case Studies

### 11x Alice: Monolithic SDR to Multi-Agent System (2025)
11x rebuilt their AI Sales Development Representative (Alice) from a monolithic campaign creation tool to a hierarchical multi-agent system using LangGraph. Key lessons:
- **Task decomposition**: Campaign creation as a monolithic task was intractable; decomposing into smaller tasks ("write an email") made it tractable
- **Tools over skills**: Rather than making agents "smart" through extensive prompting, provide tools and explain usage — minimizes token usage and improves reliability
- **Results**: 2M leads sourced, 3M messages sent, 21K replies generated. 2% reply rate comparable to human SDRs.
- **Architecture evolution**: Iterated through React, workflow-based, and multi-agent architectures before settling on hierarchical multi-agent

### Anthropic's Two-Agent Harness for Long-Running Tasks (Nov 2025)
Anthropic demonstrated a production pattern for multi-session projects:
- **Initializer agent**: Sets up environment, creates log file and initial commit
- **Coding agent**: Works in subsequent sessions, making incremental progress
- **State persistence**: Git commits with descriptive messages + progress file summaries
- **Recovery**: Git enables reverting bad changes and recovering working states

### Manus: Four Framework Rebuilds (2025)
Manus rebuilt their agent framework four times, each after discovering a better context management approach:
- Converged on: file system as memory, KV-cache optimization, attention management through recitation, hierarchical action spaces, and preserving failure information in context

---

## Sources

### Anthropic Official
- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (Sept 2025)
- [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) (Nov 2025)
- [Equipping Agents with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) (2025)
- [Writing Effective Tools for AI Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) (2025)
- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills)
- [Claude Code Subagents Documentation](https://code.claude.com/docs/en/sub-agents)
- [Compaction Documentation](https://platform.claude.com/docs/en/build-with-claude/compaction)

### Research Papers
- [LLMs Get Lost In Multi-Turn Conversation](https://arxiv.org/abs/2505.06120) — Laban et al., May 2025
- [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172) — Liu et al., 2023 (TACL 2024)
- [Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://research.trychroma.com/context-rot) — Chroma, July 2025
- [The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization](https://arxiv.org/abs/2508.21433) — Lindenbauer et al., NeurIPS 2025
- [ACON: Optimizing Context Compression for Long-horizon LLM Agents](https://arxiv.org/abs/2510.00615) — Kang et al., Oct 2025
- [Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models](https://arxiv.org/abs/2510.04618) — Zhang et al., Oct 2025
- [Recursive Language Models](https://arxiv.org/abs/2512.24601) — Zhang et al., Dec 2025
- [How Many Instructions Can LLMs Follow at Once?](https://arxiv.org/abs/2507.11538) — Jaroslawicz et al., July 2025
- [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) — Packer et al., 2023

### Industry Sources
- [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) (July 2025)
- [Compressing Context — Factory.ai](https://factory.ai/news/compressing-context) (2025)
- [Evaluating Context Compression — Factory.ai](https://factory.ai/news/evaluating-compression) (2025)
- [Cutting Through the Noise: Smarter Context Management — JetBrains Research](https://blog.jetbrains.com/research/2025/12/efficient-context-management/) (Dec 2025)
- [Ballooning Context in the MCP Era — CodeRabbit](https://www.coderabbit.ai/blog/handling-ballooning-context-in-the-mcp-era-context-engineering-on-steroids) (2025)
- [My LLM Coding Workflow Going Into 2026 — Addy Osmani](https://addyosmani.com/blog/ai-coding-workflow/) (2026)
- [Context Engineering: The Definitive 2025 Guide — FlowHunt](https://www.flowhunt.io/blog/context-engineering/)
- [11x: Rebuilding an AI SDR Agent with Multi-Agent Architecture — ZenML](https://www.zenml.io/llmops-database/rebuilding-an-ai-sdr-agent-with-multi-agent-architecture-for-enterprise-sales-automation) (2025)
- [Context Engineering Best Practices — Kubiya](https://www.kubiya.ai/blog/context-engineering-best-practices)
- [The LLM Context Problem in 2026 — LogRocket](https://blog.logrocket.com/llm-context-problem/)
- [Context Engineering for Agents — LangChain](https://blog.langchain.com/context-engineering-for-agents/)
- [Google: Architecting Efficient Context-Aware Multi-Agent Framework](https://developers.googleblog.com/architecting-efficient-context-aware-multi-agent-framework-for-production/)
- [Context Window Management Strategies — GetMaxim](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots/)
- [The 2026 Guide to Agentic Workflow Architectures — Stack AI](https://www.stackai.com/blog/the-2026-guide-to-agentic-workflow-architectures/)
- [LLM Context Window Limitations — Atlan](https://atlan.com/know/llm-context-window-limitations/)
- [Context Window Overflow in 2026 — Redis](https://redis.io/blog/context-window-overflow/)
- [Long Context — Gemini API Documentation](https://ai.google.dev/gemini-api/docs/long-context)
- [Agent Memory: How to Build Agents that Learn and Remember — Letta](https://www.letta.com/blog/agent-memory)
