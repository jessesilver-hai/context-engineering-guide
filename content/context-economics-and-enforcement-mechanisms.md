---
title: "Context Economics and Enforcement Mechanisms"
description: "Every instruction, skill description, and MCP tool definition competes for the same finite attention budget. This document covers the research on how that budget works, how it degrades, and the architectural decisions that follow."
tags:
  - context-engineering
  - claude-code
---

# Context Economics and Enforcement Mechanisms

Every instruction, skill description, and MCP tool definition competes for the same finite attention budget. This document covers the research on how that budget works, how it degrades, and the architectural decisions that follow. The core principle: **every decision must pay rent** — if adding something doesn't measurably improve Claude's behavior, it's actively making everything else worse.

---

## 1. The Instruction Tax

Adding instructions to CLAUDE.md is not additive — it's multiplicatively dilutive. Every new rule reduces Claude's compliance with every existing rule.

### The Research

| Study                                           | Finding                                                                                                        | Implication                                                                |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| "When Instructions Multiply" (arxiv 2509.21051) | Claude 3.5 Sonnet code compliance: **0.96 at 1 instruction -> 0.01 at 6 instructions**                          | Stacking 6 active constraints on a code task drives compliance toward zero |
| "Curse of Instructions" (OpenReview)            | Claude 3.5 Sonnet: 44% accuracy at 10 simultaneous instructions                                                | Even frontier models can't reliably follow 10 rules at once                |
| IFScale (arxiv 2507.11538)                      | Claude follows a **linear decay** pattern — each instruction proportionally reduces compliance with all others | No free lunch: rule N+1 degrades rules 1 through N                         |
| "Paradoxical Interference" (arxiv 2601.22047)   | Self-evident constraints ("be accurate", "write clean code") **measurably degrade** task performance           | Instructions that produce no behavioral change still cost attention        |

### What This Means for Operators

- A CLAUDE.md with 40 rules is not 40 independent guardrails at full strength. It's 40 rules each operating at a fraction of their single-rule reliability.
- "Safety" rules that feel harmless ("follow best practices", "be concise") are actively harmful — they consume attention without changing behavior.
- The optimal CLAUDE.md is not "as comprehensive as possible." It's the **smallest file that closes actual gaps** in Claude's default behavior.

---

## 2. The Attention Budget

### CLAUDE.md

CLAUDE.md is loaded into every message as part of the cached prefix. It's always present, always processed. Anthropic's official guidance: target **under 200 lines per file**. Beyond this, Claude "ignores half of it because important rules get lost in the noise."

The system prompt (Claude Code's built-in instructions) already consumes ~50 instruction slots before your CLAUDE.md loads. The usable budget for your content is roughly **100-150 instructions** — not 200.

**The pruning test** (from Anthropic's own docs): For each line, ask "Would removing this cause Claude to make mistakes?" If not, cut it.

### Skill Descriptions

All installed skills share a ~16K character budget for their `name` + `description` fields (2% of context window, hardcoded fallback of 16,000 chars). This budget is cumulative and **silently truncating** — when it fills, skills at the end of the list are dropped with no warning to the agent.

At a mean description length of ~260 chars, a collection of 63 skills has only 42 visible. The other 21 are invisible — Claude cannot discover or invoke them.

**Budget levers:**
- `disable-model-invocation: true` removes a skill's description from the budget entirely (it only loads on explicit `/name` invocation). Use this for all side-effect skills.
- `user-invocable: false` keeps the description in the budget (still competes for space). Use this for auto-discoverable reference skills.
- Front-load trigger keywords in the first 50 characters of descriptions — Claude scans from the top when matching.

### MCP Tool Definitions

MCP tools compete for the same context. Before Tool Search, 50+ MCP tools consumed ~51K-77K tokens upfront. With Tool Search (`ENABLE_TOOL_SEARCH=auto`), that collapses to ~8.5K tokens. Tool Search auto-enables when MCP tool descriptions exceed 10% of context.

**Practical implication:** An operator with many MCP servers AND many skills can see both systems competing before the first user message is processed.

### Loaded Skills + Conversation History

Once a skill is invoked, its full SKILL.md body enters context and stays there for the remainder of the conversation turn. A 500-line SKILL.md (~6-10K tokens) loaded mid-conversation meaningfully reduces remaining history depth before compaction fires. Reference files in `reference/` cost zero tokens until Claude reads them — use this aggressively.

### Session Lifespan

Context is a diminishing resource over session length. After ~50 full exchanges in a 200K window, auto-compaction fires and summarizes history (lossy). Every file Claude reads, every command output, every tool response accumulates. Long sessions with many file reads deplete the usable window faster than raw token math suggests.

CLAUDE.md survives compaction — it's re-read from disk and re-injected fresh. Conversation-only instructions do not survive. Any rule that must persist across an entire session must be in CLAUDE.md.

### Subagent Multiplication

Each subagent starts with a fresh context window and loads CLAUDE.md, MCP servers, and skill descriptions from scratch. A 3-teammate team uses roughly 3-4x the tokens of a single sequential session. Every skill description that consumes budget in the main session also consumes budget identically in each spawned teammate — multiplied by team size.

---

## 3. The Enforcement Spectrum

Not all mechanisms are equal. Choose the right one based on how reliably the behavior must happen.

| Mechanism | Reliability | Cost | When to Use |
|---|---|---|---|
| **Hook** (deterministic script) | 100% — runs unconditionally on every matching event | Zero attention cost; runs outside the LLM | Anything that MUST happen with zero exceptions: formatting, linting, validation, blocking writes to protected files |
| **CLAUDE.md instruction** | ~70-90% for short files, degrades with length | Always-on attention cost every message | Behavioral guidance that applies broadly: tool routing, terminology, architectural decisions |
| **Skill** (on-demand) | ~80-95% when triggered, 0% when not | Zero cost until invoked; then competes with history | Domain knowledge relevant only to specific task types |
| **Conversation instruction** | ~60-80%, lost on compaction | One-time cost, doesn't persist | Session-specific overrides the user states in chat |

### The Decision Rule

**If you'd be upset when Claude ignores it -> hook.** Hooks are deterministic. Instructions are probabilistic.

Examples:
- "Always run ESLint after editing" -> **Hook** (PostToolUse on Edit|Write). CLAUDE.md compliance: ~80%. Hook compliance: 100%.
- "Use Notion MCP for project docs, Glean for Slack search" -> **CLAUDE.md**. Routing guidance that applies every session.
- "BigQuery schema for hai_dev tables" -> **Skill**. Only needed when writing SQL. Zero cost in non-SQL sessions.
- "For this task, output as CSV" -> **Conversation instruction**. One-time, doesn't need persistence.

### Common Misplacements

| What People Put | Where They Put It | Where It Should Go | Why |
|---|---|---|---|
| Code formatting rules | CLAUDE.md | PostToolUse hook running Prettier/Black | Linters are deterministic; instructions are probabilistic |
| "Write clean code" | CLAUDE.md | Nowhere (delete it) | Claude already does this; the instruction costs attention and changes nothing |
| Full API schema | CLAUDE.md | Skill reference file | Loaded every session whether needed or not; should be on-demand |
| "Always run tests" | CLAUDE.md | Stop hook that blocks exit if tests fail | Advisory instructions get ignored under time pressure; hooks don't |
| Domain terminology | Skill | CLAUDE.md | Needed every session for correct communication, not just when skill triggers |

---

## 4. The Lost-in-the-Middle Problem

LLMs don't treat all positions in the context window equally. Performance follows a U-shaped curve: strong attention at the start, strong at the end, weak in the middle.

**Implication for CLAUDE.md:** High-priority rules should be near the top or bottom. Mid-file rules are in the attention trough. If a rule keeps being ignored despite being in CLAUDE.md, try moving it to the top.

**Implication for skills:** SKILL.md content that matters for correct behavior should be near the top of the file, not buried in the middle of a 400-line document.

Source: Liu et al., "Lost in the Middle: How Language Models Use Long Contexts" (Stanford, 2023). >30% performance drop for mid-context information.

---

## 5. Anthropic's Official Include/Exclude Table

From the Claude Code best practices docs — the authoritative filter for what belongs in CLAUDE.md:

| Include (pays rent) | Exclude (doesn't pay rent) |
|---|---|
| Bash commands Claude can't guess | Anything Claude can figure out by reading code |
| Code style rules that differ from defaults | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners | Detailed API documentation (link or use skills) |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Developer environment quirks (required env vars) | File-by-file descriptions of the codebase |
| Common gotchas or non-obvious behaviors | Self-evident practices like "write clean code" |

---

## 6. Practical Sizing Guidelines

| Component | Target | Why |
|---|---|---|
| CLAUDE.md | <200 lines, ideally <100 | Uniform adherence degradation beyond this |
| Individual CLAUDE.md instruction | 1-2 lines | Bullets, not paragraphs. "X, not Y" format. |
| Skill description | 3-5 lines, trigger keywords in first 50 chars | Must be rich enough for routing but compete for 16K shared budget |
| SKILL.md body | <500 lines | Competes with conversation history once loaded |
| Reference files | Unlimited, but each read adds to context | Zero cost until accessed; use aggressively for large content |
| Total skills | Monitor at 30+; audit at 50+ | Silent truncation starts when descriptions exceed budget |
| MCP servers | 5-10 active; performance degrades beyond 20 | Tool definitions compete for context; use Tool Search above threshold |

---

## 7. The Canary Test

Embed a distinctive, easily verifiable rule in CLAUDE.md — a specific formatting convention, a specific phrase, a naming pattern. When Claude stops following it, your file has crossed the reliability threshold. This gives you an objective signal that it's time to prune.

---

## Sources

### Academic
- Liu et al., "Lost in the Middle" ([arxiv 2307.03172](https://arxiv.org/abs/2307.03172)) — U-shaped attention curve
- "When Instructions Multiply" ([arxiv 2509.21051](https://arxiv.org/abs/2509.21051)) — quantified instruction collapse
- "Curse of Instructions" ([OpenReview](https://openreview.net/forum?id=R6q67CDBCH)) — uniform degradation across instructions
- IFScale ([arxiv 2507.11538](https://arxiv.org/abs/2507.11538)) — linear decay pattern for Claude
- "Paradoxical Interference" ([arxiv 2601.22047](https://arxiv.org/html/2601.22047)) — self-evident constraints hurt performance

### Anthropic Official
- [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices) — include/exclude table, pruning test, over-specified anti-pattern
- [How Claude Remembers](https://code.claude.com/docs/en/memory) — 200-line target, compaction survival, scope hierarchy
- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — "context rot", smallest high-signal token set
- [Manage Costs Effectively](https://code.claude.com/docs/en/costs) — MCP overhead, agent team token multiplication, Tool Search
- [Extend Claude with Skills](https://code.claude.com/docs/en/skills) — 16K budget, silent truncation, invocation control

### Community
- [Skill budget research](https://gist.github.com/alexey-pelykh/faa3c304f731d6a962efc5fa2a43abe1) — empirical measurement of silent truncation
- [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) — 60-line production standard, ~50 built-in instruction overhead
- [MLOps Community: Prompt Bloat](https://mlops.community/the-impact-of-prompt-bloat-on-llm-output-quality/) — semantic noise penalty
