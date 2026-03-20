---
title: "Skill and Memory Architecture"
description: "Best practices for structuring skills, memory, and context in Claude Code agents -- compiled from Anthropic's official docs, engineering blog, community patterns, and the Feb 2026 autonomy research."
tags:
  - context-engineering
  - claude-code
---

# Skill and Memory Architecture for Claude Code Agents

Best practices for structuring skills, memory, and context — compiled from Anthropic's official docs, engineering blog, community patterns, and the Feb 2026 autonomy research.

---

## 1. Memory Architecture: What Goes Where

Claude Code has a clear hierarchy. The key insight: **reliability of loading determines what should go where.**

| Layer | What belongs here | Why |
|---|---|---|
| **CLAUDE.md** (project root) | Team-shared instructions, architecture, coding standards | Auto-loaded every session, high priority, version-controlled |
| **`.claude/rules/*.md`** | Modular topic-specific rules (code style, testing, API patterns) | Auto-loaded, can be path-scoped with `paths:` frontmatter |
| **`~/.claude/CLAUDE.md`** (user) | Personal preferences across all projects | Auto-loaded, personal, not shared |
| **`CLAUDE.local.md`** | Personal project-specific prefs (sandbox URLs, test data) | Auto-loaded, gitignored automatically |
| **Auto memory** (`~/.claude/projects/<project>/memory/`) | Claude's learnings: patterns, debugging insights, architecture notes | First 200 lines of MEMORY.md auto-loaded; topic files loaded on-demand |
| **Skill files** (`SKILL.md` + supporting files) | Workflow instructions, reference material, scripts | Descriptions always in context; full content loaded only when skill triggers |

### Design Principles

- **CLAUDE.md = always-loaded instructions.** Keep it specific, structured as bullets under headings. Every line competes for the ~150-200 effective instruction slots Claude can reliably follow.
- **Auto memory = Claude's own notes.** MEMORY.md is an index; detailed notes go in topic files (`debugging.md`, `api-conventions.md`). Only 200 lines of MEMORY.md load at startup — everything else is on-demand.
- **Skills = contextual toolkits.** Only the description loads at startup (for discovery). Full content loads when relevant. Reference files load only when needed within a skill. This is progressive disclosure.
- **External memory tools (mem0, etc.) = preference/correction store.** Good for learning from user signals (corrections, approvals, feedback) that aren't structural enough for CLAUDE.md. Requires search to retrieve — not guaranteed to surface.

### The Core Tradeoff

CLAUDE.md files bypass the retrieval problem — they're always present. Memory tools and on-demand files require Claude to decide to look, and that decision can fail. **Put anything Claude must always know in CLAUDE.md. Put everything else in files Claude can reach when it needs them.**

Sources: [Manage Claude's memory](https://code.claude.com/docs/en/memory) | [How Claude's Memory Actually Works](https://rajiv.com/blog/2025/12/12/how-claude-memory-actually-works-and-why-claude-md-matters/)

---

## 2. Skill Structure: How Much to Decompose

### Granularity Spectrum

Skills sit on a spectrum from **reference context** to **task workflows**:

| Type | Example | Structure |
|---|---|---|
| **Reference** (background knowledge) | API conventions, code style guide | Short SKILL.md with patterns/rules. `user-invocable: false` if not a command. |
| **Lightweight task** | Generate commit message, explain code | Single SKILL.md, <100 lines, high degrees of freedom |
| **Structured workflow** | QC pipeline, deployment, data delivery | SKILL.md as orchestrator + supporting files (scripts, references, templates). Checklist pattern. |
| **Complex pipeline** | Multi-phase assessment, SSOT build | SKILL.md as table of contents pointing to phase-specific files. Consider `context: fork` for isolation. |

### Decomposition Rules of Thumb

1. **One skill = one coherent workflow.** If you can't describe what the skill does in one sentence, it's too broad.
2. **SKILL.md under 500 lines.** Beyond this, split into supporting files. Claude partially reads deeply nested references — keep everything one level deep from SKILL.md.
3. **Scripts for deterministic operations.** Anything that must be exact (validation, data transforms, API calls) should be a script Claude executes, not instructions Claude interprets.
4. **Skills CAN call other skills** as subskills using the Skill tool. Use this for composability rather than inlining reusable patterns.
5. **Use `context: fork` for isolated execution.** When a skill's task doesn't need conversation history (research, analysis, generation), fork it to a subagent. This protects context and enables parallelism.
6. **Don't over-decompose.** Three lines of inline instruction is better than a premature skill. Skills have overhead (discovery, loading, context consumption). Only create a skill when the pattern repeats or is complex enough to warrant structure.

### Practical Architecture

```
my-skill/
SKILL.md              # Core instructions (overview + navigation)
scripts/              # Deterministic operations Claude executes
  validate.py
  transform.sh
reference/            # Domain-specific context loaded on-demand
  schema.md
  api-docs.md
templates/            # Output templates Claude fills in
  report-template.md
examples/             # Input/output pairs for calibration
  good-example.md
```

SKILL.md serves as a table of contents. It tells Claude what each file is and when to load it. Claude reads only what the current task requires.

Sources: [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) | [Extend Claude with skills](https://code.claude.com/docs/en/skills) | [Agent Skills Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)

---

## 3. Autonomy: Ask User vs. Make Decisions

### The Anthropic Research (Feb 2026)

Key findings from Anthropic's [autonomy measurement research](https://www.shashi.co/2026/02/the-autonomy-gap-what-anthropic-learned.html):

- **Claude asks clarifying questions 2x more often than users interrupt it.** The model defaults to checking when uncertain — and this is correct behavior.
- **Experienced users grant MORE autonomy AND interrupt MORE.** This is "pattern-matching oversight" — trust the agent for routine work, intervene when deviations appear likely.
- **Auto-approval grows with experience:** ~20% of sessions at first, >40% by 750 sessions. Trust is earned gradually.
- **The bottleneck is psychological, not technical.** Model capability exceeds human willingness to delegate. Build transparency to close this gap.

### Design Framework: When to Ask vs. Act

| Situation | Approach | Why |
|---|---|---|
| **Fragile, irreversible operations** (deploy, delete, send message) | `disable-model-invocation: true` — user must invoke explicitly | Side effects can't be undone. User controls timing. |
| **Multiple valid approaches** (architecture, design, algorithm choice) | Present options with tradeoffs, let user choose | Context-dependent decisions where user has domain knowledge the model may lack |
| **Ambiguous requirements** (vague instructions, missing context) | Ask for clarification before proceeding | Garbage in, garbage out. A question is cheaper than rework. |
| **Well-defined, reversible operations** (file edits, test runs, analysis) | Act autonomously | Low risk, easy to undo, user can review results |
| **Established patterns** (style guide compliance, convention following) | Act autonomously per the documented pattern | The pattern IS the decision. No ambiguity. |
| **Background knowledge application** (API conventions, codebase context) | `user-invocable: false` — Claude loads automatically | Users shouldn't invoke reference context as a command |

### Practical Autonomy Patterns

**1. Degrees of Freedom (from Anthropic's official best practices)**

Match specificity to task fragility:
- **High freedom** (text instructions, Claude chooses approach): Code review, analysis, research
- **Medium freedom** (pseudocode/templates with parameters): Report generation, data processing
- **Low freedom** (exact scripts, no modifications): Database migrations, deployments, data delivery

**2. Feedback Loops Over Gates**

Instead of asking before every action, build validation into the workflow:
```
Do work -> Validate output -> If errors, fix and re-validate -> Only present final result
```
This is more autonomous AND higher quality than asking permission at each step.

**3. Progressive Trust**

New skills should start conservative (more user confirmation) and graduate to autonomous as the workflow proves reliable. Use `disable-model-invocation: true` initially, then remove it once the skill is battle-tested.

**4. Transparency Over Permission**

The 2026 trend: build skills that surface what they're doing and why, rather than constantly asking. Observable agents earn trust faster than cautious ones. Show intermediate results, explain reasoning, flag uncertainty — but keep moving.

Sources: [The Autonomy Gap](https://www.shashi.co/2026/02/the-autonomy-gap-what-anthropic-learned.html) | [Agent design patterns](https://rlancemartin.github.io/2026/01/09/agent_design/) | [Equipping agents for the real world](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills)

---

## 4. Context Management: Keeping the Window Clean

### The Context Window is a Public Good

Every token in the context window competes with conversation history, other skills, and the system prompt. The hierarchy of context cost:

| Tier | Cost | When it loads |
|---|---|---|
| Skill descriptions | Low (always loaded) | Session start |
| CLAUDE.md / rules | Medium (always loaded) | Session start |
| SKILL.md body | Medium (loaded on trigger) | When skill is relevant |
| Supporting files | Low (loaded on-demand) | When Claude reads them |
| Script output | Variable | When Claude executes |

### Strategies

1. **Progressive disclosure everywhere.** SKILL.md is a table of contents. Supporting files are chapters. Scripts are appendices. Claude reads only what the current task needs.
2. **Offload to the filesystem.** Move older tool results, large datasets, and intermediate state to files. Claude can read them back when needed without consuming persistent context.
3. **Use sub-agents for isolation.** Fork context-heavy tasks to sub-agents (`context: fork`). Their work happens in a separate context window, and only the summary returns to the main conversation.
4. **Start fresh for each major task.** Use `/clear` between unrelated tasks to prevent context drift. The CLAUDE.md and auto memory reload automatically.
5. **Don't explain what Claude already knows.** A common anti-pattern in skill authoring. Claude understands programming concepts, common libraries, and standard patterns. Only add context Claude doesn't have — your specific schemas, your conventions, your edge cases.
6. **Structure reference files with TOCs.** For files >100 lines, add a table of contents at the top so Claude can see scope even when previewing with partial reads.

Sources: [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) | [Agent design patterns](https://rlancemartin.github.io/2026/01/09/agent_design/) | [Optimizing Agentic Coding](https://aimultiple.com/agentic-coding)

---

## 5. Development Workflow: How to Build Good Skills

Anthropic's recommended cycle:

1. **Complete a task manually with Claude.** Notice what context you repeatedly provide.
2. **Identify the reusable pattern.** What knowledge would make this task self-sufficient?
3. **Build evaluations first.** Define 3+ test scenarios that cover the gaps. Measure baseline without the skill.
4. **Write minimal instructions.** Only what's needed to pass the evaluations. No speculative docs.
5. **Test with a fresh Claude instance.** The "Claude A / Claude B" pattern — Claude A helps you write the skill, Claude B tests it cold.
6. **Observe how Claude navigates the skill.** Watch for: files it never reads (unnecessary), files it reads repeatedly (should be more prominent), unexpected exploration paths (structure isn't intuitive).
7. **Iterate based on real usage, not assumptions.** Each failure reveals a gap. Fix the gap, re-test, repeat.

### The Development Lifecycle

The recommended progression from Anthropic's guide and the skill-creator skill:

```
1. Conversation  — do it manually with Claude, notice what you repeat
2. Skill         — capture the reusable pattern as an auto-triggered skill
3. Command       — only if the workflow needs all three:
                   side effects + user-controlled timing + structured arguments
```

The command is a **conditional endpoint**, not a waypoint. Most workflows should stop at step 2. A diagnostic decision tree, a metrics analysis framework, a quality management playbook — these are skills (reference knowledge Claude consults), not commands (buttons the user pushes).

**The anti-pattern is not writing monolithic commands.** It's confusing "I do this repeatedly" with "this needs to be a command." Repetition signals a skill. A command is only warranted when the user must control *when* and *with what parameters* the workflow runs, and the workflow has side effects.

**From the Anthropic PDF (p.15):** "We've found that effective skill creators iterate on a single challenging task until Claude succeeds, then extract the winning approach into a skill." — conversation -> skill, no command in the middle.

### Anti-Patterns

- **Verbose explanations of common concepts.** Claude knows what PDFs are. Don't explain.
- **Multiple tool/library options without a default.** Pick one, explain the escape hatch.
- **Deeply nested references.** Claude partially reads files referenced from referenced files. Keep everything one level from SKILL.md.
- **Time-sensitive information.** Use "current method" / "old patterns" structure instead.
- **Inconsistent terminology.** Pick one term per concept and use it everywhere.
- **Monolithic SKILL.md.** >500 lines = split into files. >5000 words = definitely split.
- **Defaulting to commands for repeated workflows.** Repetition signals a skill, not a command. Commands are only for workflows with side effects, user-controlled timing, and structured arguments — all three.

Sources: [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) | [Agent Skills Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/) | [Anthropic: The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)

---

## 6. When Is a Command Actually Necessary?

Most workflows structured as monolithic slash commands can be decomposed into lighter-weight primitives. The question is when decomposition helps vs. when it adds indirection.

### The Primitives (Lightest to Heaviest)

| Primitive | Context Cost | Reliability | User Action | Fires When |
|---|---|---|---|---|
| **CLAUDE.md rule** | Always loaded (~1 instruction slot) | ~70-90%, degrades with file length | None | Every message |
| **Hook** | Zero (runs outside LLM) | 100% deterministic | None | Matching event |
| **Auto-triggered skill** (`user-invocable: false`) | Description always loaded; body on-demand | ~80-95% when triggered | None | Claude matches task to description |
| **Subagent** (`.agent.md`) | Fresh context per invocation | High (isolated, focused) | None (Claude delegates) | Task matches description |
| **Nested skill call** (skill invokes skill via Skill tool) | Additive — both bodies in context | High but compounds context | None | Parent skill invokes |
| **Slash command** (`disable-model-invocation: true`) | Zero until invoked; full body after | 100% triggering (user controls) | User types `/command` | Explicit invocation |

### A Command Is Necessary When ALL Three Are True

1. **Side effects** — the workflow writes data, sends messages, or creates resources
2. **User-controlled timing** — not "whenever it seems relevant" but "right now, for this project"
3. **Structured arguments** — the workflow needs parameters (project name, date range, etc.)

### A Command Is Unnecessary When ANY Are True

- Claude could figure out when to do it from conversational context (auto-trigger)
- It's pure read/analysis with no side effects
- The knowledge is reference material Claude should consult silently
- The same instructions would work phrased as natural language

### The Monolithic Command Anti-Pattern

A command that embeds workflow logic, SQL patterns, table formatting, data freshness checks, and output templates directly in the `.md` file. This loads the entire instruction set into context and keeps it there for the rest of the conversation.

**The fix:** Commands should be **thin delegation layers** (<30 lines). The command parses arguments and delegates to an agent. The agent orchestrates the workflow in an isolated context. Skills provide reference knowledge the agent consults on-demand.

```
Command (thin entry point, <30 lines)
  -> parse arguments
  -> delegate to agent with task prompt

Agent (workflow logic, isolated 200K context, model routing)
  -> consult skill for domain knowledge
  -> execute tools
  -> return structured results

Skill (reference knowledge, loaded on-demand)
  -> schema definitions, decision trees, conventions
```

**Why this is better:**
- The agent gets a fresh 200K context window — no pollution from prior conversation
- All intermediate queries, dead ends, and raw data stay inside the agent (300:1 compression)
- The agent can be model-routed (Opus for complex reasoning, Sonnet for data pulls)
- The command file stays maintainable and readable

### Five Alternatives to Monolithic Commands

**1. Auto-triggered skill with agent delegation.** User says "how is Argus doing?" -> Claude auto-loads the playbook skill -> consults the database skill -> spawns a subagent -> returns results. Works when intent is unambiguous from natural language. Doesn't work when you need specific parameters or user-controlled timing.

**2. Reference skills with conversational guidance.** An auto-triggered skill provides the decision framework, and Claude walks the user through it conversationally — asking questions, pulling data, adjusting based on answers. The skill teaches Claude the methodology; Claude applies it through conversation. Fundamentally different from a command that prescribes every step.

**3. Composable skill chains (nested skills).** Claude loads skill A -> which invokes skill B for data -> which invokes skill C for formatting. Works well for the `playbook` -> `database` -> `drive` pattern. Trade-off: nested skills are additive in context — 3 loaded skills = 1500+ lines of instructions competing for attention.

**4. Agent + skill composition (recommended for complex workflows).** Thin command -> agent -> skills. The agent runs in isolation, consults skills as reference, and returns only the final result. This is the recommended pattern for any workflow exceeding 30 lines of instructions.

**5. Context files for universal behavior.** Tool routing rules, source-of-truth hierarchies, write policies — these apply to every interaction. They belong in CLAUDE.md, not repeated in every command or skill.

### When Each Primitive Wins

| User Need | Best Primitive | Why |
|---|---|---|
| "Always look up project IDs dynamically" | CLAUDE.md rule | Universal, must always apply, 1 line |
| "Format code after every edit" | Hook | Deterministic, can't be forgotten |
| "Know the project lifecycle phases" | Auto-triggered skill | Reference knowledge, load when relevant |
| "Know the BQ schema" | Auto-triggered skill | Domain knowledge, zero cost until needed |
| "Query BQ for daily metrics" | Subagent | Isolate context, parallel execution |
| "Run a full diagnostic with recommendations" | Command -> Agent -> Skills | User controls timing, needs arguments, recommendations affect forecasts |
| "Pull fresh actuals and present them" | Command | Side effects (writes to filesystem), user controls timing |

### The Decision Rule

> If you're writing more than 30 lines in a command file, the logic belongs in an agent or skill, not the command.
>
> If the workflow doesn't have side effects and doesn't need structured arguments, it probably doesn't need to be a command at all.
>
> Commands are buttons. Agents are workers. Skills are manuals. Context files are rules. Use the lightest primitive that achieves reliable behavior.

Sources: [Anthropic: The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf) | [Anthropic: Equipping Agents with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) | [Anthropic: Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

---

## 7. Summary Decision Matrix

| Question | Answer |
|---|---|
| **Should this be in CLAUDE.md or a skill?** | If it applies to every task in this project -> CLAUDE.md. If it's a specific workflow triggered by context -> skill. |
| **Should this be a skill or a command?** | Default to skill. A command is only warranted when the workflow has side effects AND the user must control timing AND it needs structured arguments — all three. If any condition is missing, it's a skill. |
| **Should this be one skill or multiple?** | If the steps always run together -> one skill. If they're independently useful -> separate skills that can call each other. |
| **Should this ask the user or act?** | If irreversible/high-stakes -> ask. If ambiguous -> ask. If well-defined and reversible -> act. If established pattern -> act. |
| **Should this use `context: fork`?** | If it doesn't need conversation history AND would benefit from isolation -> yes. If it's applying guidelines to current work -> no (run inline). |
| **Should this use scripts or instructions?** | If the operation must be exact -> script. If Claude needs judgment -> instructions. If both -> instructions that call scripts at key steps. |
| **How much should I decompose?** | Minimum viable decomposition. Only split when a sub-workflow is independently reusable OR the combined SKILL.md exceeds 500 lines. |
| **I keep repeating this workflow — command?** | No. Repetition signals a skill, not a command. Capture the pattern as an auto-triggered skill first. Only promote to a command if side effects + timing + arguments all apply. |
