---
title: "Agent Self-Optimization and Tool Generation"
description: "How to make agents faster over time by detecting slow operations, auto-generating scripts, and caching execution plans. Covers the research landscape (2025-2026), practical patterns for Claude Code, and the gap between what exists and what's needed."
tags:
  - context-engineering
  - claude-code
---

# Agent Self-Optimization and Automatic Tool Generation

How to make agents faster over time by detecting slow operations, auto-generating scripts, and caching execution plans. Covers the research landscape (2025-2026), practical patterns for Claude Code, and the gap between what exists and what's needed.

---

## 1. The Problem: Agents Are 90% Thinking, 10% Doing

Empirical benchmarking on HAI operator agents (March 2026) revealed a consistent pattern: agents spend 78-92% of their wall-clock time on LLM inference (planning, deciding the next step, formatting output), not on actual tool execution.

| Task | Raw Operation | Agent Time | LLM Thinking | Speedup After Scripting |
|---|---|---|---|---|
| Read a Google Sheet (998 rows) | 3.6s (curl) | 120s | 95s (78%) | 25x |
| Read + aggregate sheet by attempter | 5s (script) | 245s | 230s (94%) | 40x |
| BigQuery role counts | 1s (BQ query) | 25s | 15s (60%) | 5x |

The fix in every case was the same: **replace multi-turn LLM planning with a single pre-built script**. The agent calls one command instead of 8-12, eliminating the inference overhead entirely.

### Why Agents Don't Self-Optimize

Three missing capabilities prevent agents from fixing this themselves:

1. **No runtime performance awareness** — Agents don't have access to a clock or timer during execution. They can't detect that a task is taking too long.
2. **No cross-session learning for tools** — Even if an agent scripted a solution, that script wouldn't automatically be discoverable by future agent invocations. Claude Code's auto-memory is close but too unstructured for tool registration.
3. **No automatic tool registration** — Creating a script is half the problem. The script also needs to be referenced in the agent's `.md` definition, documented with usage examples, and placed in the right directory. This is a multi-file coordination problem.

---

## 2. Research Landscape (2025-2026)

### Agentic Plan Caching (APC) — NeurIPS 2025

**Paper:** Zhang et al., "Agentic Plan Caching: Test-Time Memory for Fast and Cost-Efficient LLM Agents" ([arXiv:2506.14852](https://arxiv.org/abs/2506.14852)) | [OpenReview](https://openreview.net/forum?id=n4V3MSqK77)

The closest existing work to agent self-optimization for repeated tasks. APC extracts structured "plan templates" from completed agent executions at test-time, stores them, and reuses them for semantically similar future tasks.

**How it works:**
1. Agent completes a task normally (expensive, full planning)
2. System extracts a plan template from the execution trace — the sequence of actions with dynamic parameters replaced by placeholders
3. Template is indexed by keyword extraction from the original request
4. On a new request, the system matches against cached plans
5. A lightweight model adapts the template to the new task's specific context

**Key insight:** In web/GUI navigation tasks, the same high-level query requires similar action sequences, but specifics differ (screen size, window position, data values). Conventional semantic caching fails because it doesn't separate core intent from dynamic context. APC does.

**Results:**
- 56% cost reduction
- 65% latency reduction
- Validated on web agent benchmarks (WebArena, VisualWebArena)

**Relevance to our problem:** APC is plan-level memoization. Our problem is operation-level scripting. APC caches "read sheet -> group by column -> filter -> sort" as a plan template. We script it as `read-sheet.sh --group-by X --where Y --sort Z`. Same principle, different granularity.

### Agentic Context Engineering (ACE) — ICLR 2025

**Paper:** Zhang et al., "Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models" ([arXiv:2510.04618](https://arxiv.org/abs/2510.04618)) | [OpenReview](https://openreview.net/forum?id=eC4ygDs02R) | [GitHub](https://github.com/ace-agent/ace)

ACE treats agent contexts (system prompts, instructions, memory) as evolving playbooks that improve through use. Three specialized components:

- **Generator**: Produces reasoning trajectories for new queries
- **Reflector**: Critiques traces to extract lessons — separated from curation to avoid monolithic rewriting
- **Curator**: Synthesizes lessons into compact delta entries, merged deterministically into existing context

**Key design principle:** Context is a collection of structured, itemized bullets — not a monolithic prompt. Incremental delta updates replace costly monolithic rewrites. A grow-and-refine mechanism balances steady context expansion with redundancy control.

**Results:** On the AppWorld leaderboard, ReAct + ACE matched the top-ranked system (IBM CUGA) despite using a smaller open-source model, and surpassed it by 8.4% on the harder test-challenge split with online adaptation.

**Why monolithic rewriting fails:** An LLM tasked with fully rewriting accumulated context tends to compress it into much shorter, less informative summaries, causing dramatic information loss ("context collapse").

**Relevance:** ACE's "evolving playbook" pattern is what we want for agent instructions — skills and agent `.md` files that automatically accumulate "use this script for X" entries based on observed performance.

### OpenSage: Self-Programming Agent Generation Engine — 2026

**Paper:** [arXiv:2602.16891](https://arxiv.org/html/2602.16891v1)

OpenSage integrates self-generated agent topology, dynamically-created tooling, hierarchical memory, and containerized execution into a unified framework. It automates the three components that current ADKs (Google ADK, OpenHands, Claude ADK, LangChain) still require humans to design:

1. **Agent topology** — which agents exist and how they communicate
2. **Tooling system** — what tools are available and when
3. **Memory system** — what state persists across steps

**Results:** Ranked #1 on CyberGym and Terminal-Bench 2.0 leaderboards, achieving a resolved rate >20% higher than OpenHands with the same backbone model.

**Relevance:** OpenSage demonstrates that auto-tool-creation is feasible and effective. The gap is that it's a research system, not integrated into Claude Code's skill/hook/agent architecture.

### Self-Evolving Agents — OpenAI Cookbook (2025)

**Source:** [OpenAI Cookbook — Self-Evolving Agents](https://developers.openai.com/cookbook/examples/partners/self_evolving_agents/autonomous_agent_retraining/)

A practical cookbook for building self-improving agent loops:

1. Agent executes a task
2. LLM-as-judge evaluates the execution quality
3. Meta-prompting generates improved instructions
4. Human reviews and approves changes
5. Agent instructions are updated for next run

**Relevance:** The feedback loop pattern is directly applicable. The missing piece for Claude Code is the automated detection trigger — knowing *when* to invoke the improvement loop.

### Meta Context Engineering via Agentic Skill Evolution — Peking University (2026)

"MCE is among the first to dynamically evolve [skills], bridging manual skill engineering and autonomous self-improvement."

**Relevance:** Validates that skill evolution is an active research direction. The gap between manual skill engineering (what we do now) and autonomous self-improvement (what we want) is being actively closed.

---

## 3. Practical Patterns for Claude Code

### Pattern A: PostToolUse Performance Logger

A hook that watches every Bash call in subagents and logs slow operations.

**Implementation:**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": { "toolName": "Bash" },
      "hooks": [{
        "type": "command",
        "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/perf-logger.sh"
      }]
    }]
  }
}
```

```bash
# perf-logger.sh — Log slow operations for future scripting
DURATION=$(echo "$TOOL_RESULT" | jq -r '.duration_ms // 0')
COMMAND=$(echo "$TOOL_INPUT" | jq -r '.command // ""')
AGENT_ID=$(echo "$TOOL_CONTEXT" | jq -r '.agentId // "main"')

if [ "$DURATION" -gt 30000 ]; then  # >30 seconds
  echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) | ${DURATION}ms | $AGENT_ID | $COMMAND" \
    >> "$HOME/.claude/slow-operations.log"
fi
```

**Limitation:** Claude Code's PostToolUse hooks don't currently receive `duration_ms` in the tool result. This would require either a wrapper script that times the command, or an enhancement to the hook system.

### Pattern B: Slow-Operation Detector (Pre-Scripting)

Instead of logging after the fact, detect *before* the agent starts that it's about to do something that should be scripted.

**Implementation:** A PreToolUse hook that pattern-matches common slow operations:

```bash
# pre-script-detector.sh
COMMAND=$(cat | jq -r '.tool_input.command // ""')

# Pattern: agent is about to do a multi-step sheet read + parse
if echo "$COMMAND" | grep -qE 'curl.*sheets.googleapis.com.*python3'; then
  echo '{"decision": "block", "reason": "Use read-sheet.sh instead of inline curl+python. Run: read-sheet.sh <id> <tab> [--group-by COL --agg COL:FN --where COL>=N --sort COL:desc]"}'
  exit 0
fi
```

**Advantage:** Prevents slow paths before they start. The agent gets redirected to the fast script immediately.

**Limitation:** Only works for patterns you've already identified and scripted. Doesn't discover new slow patterns automatically.

### Pattern C: Periodic Optimization Skill

A skill (e.g., `/optimize-agents`) that:

1. Reads `~/.claude/slow-operations.log`
2. Identifies repeated patterns (same command structure, different parameters)
3. Generates a parameterized script
4. Writes it to the appropriate plugin's `scripts/` directory
5. Updates the agent `.md` to reference the new script

**When to run:** Manually after a session where things felt slow, or automatically via a `/loop` that runs every few sessions.

**Advantage:** Closes the full feedback loop: detect -> script -> register -> use.

### Pattern D: Agent-Level `maxTurns` + Fallback

Set aggressive `maxTurns` limits on agents. If an agent hits its turn limit without completing, it returns what it has. The parent can then either:
- Retry with a more specific prompt
- Fall back to a simpler approach
- Flag the operation for scripting

```yaml
# Agent definition
maxTurns: 5  # Force the agent to be efficient
```

**Advantage:** Simple, no new infrastructure. Creates natural pressure toward scripted solutions.

### Pattern E: Script Discovery via Agent Memory

After manually creating a script, update the agent's `.md` file with a table of available scripts. This is the pattern we implemented for `read-sheet.sh` and `bq-*.sh`.

The key insight: **the bottleneck isn't detecting slow operations — it's someone building the script and updating the agent docs**. Patterns A-D automate detection. Pattern E is the manual-but-fast resolution step.

---

## 4. The Gap: What's Missing

| Capability | Research Status | Claude Code Status |
|---|---|---|
| Detect slow operations at runtime | Feasible (timer hooks) | Not built-in; needs custom hooks |
| Cache execution plans for reuse | APC (NeurIPS 2025) | Not available; would need custom implementation |
| Auto-generate scripts from repeated patterns | OpenSage demonstrates feasibility | Not available; would need a skill/agent to do this |
| Auto-register tools in agent definitions | OpenSage, ACE | Not available; agent `.md` files are static |
| Cross-session tool persistence | ACE's evolving playbook | Partially available via auto-memory; not structured enough for tool registration |
| Evolve agent instructions based on performance | ACE, Self-Evolving Agents | Partially available via `/revise-claude-md`; manual trigger |

**The most impactful missing piece:** A standard way for an agent to say "I just built a useful script — register it as a tool for future invocations." Currently this requires editing the agent's `.md` file, which is a human/main-context operation.

---

## 5. Recommended Implementation Path

### Phase 1: Manual Script-and-Register (Current)
- Build scripts as we discover slow patterns (`read-sheet.sh`, `bq-*.sh`)
- Update agent `.md` files with script tables
- Low effort, high impact, no infrastructure needed

### Phase 2: Slow-Operation Logger
- Add a PostToolUse hook that logs commands taking >30s
- Review the log periodically (or build a `/optimize-agents` skill)
- Medium effort, enables data-driven script creation

### Phase 3: Pre-Script Detector
- Add a PreToolUse hook that pattern-matches known slow operations
- Redirects agents to pre-built scripts before they start the slow path
- Medium effort, prevents regression

### Phase 4: Auto-Script Generator
- Build a skill that reads the slow-operations log and generates scripts
- Tests the generated scripts
- Updates agent `.md` files
- High effort, but closes the full feedback loop

### Phase 5: APC-Style Plan Caching
- Extract plan templates from successful agent executions
- Store and reuse for similar future tasks
- Highest effort, biggest payoff for repeated workflows

---

## 6. Sources

### Academic
- Zhang et al., "Agentic Plan Caching" ([arXiv:2506.14852](https://arxiv.org/abs/2506.14852), NeurIPS 2025)
- Zhang et al., "Agentic Context Engineering" ([arXiv:2510.04618](https://arxiv.org/abs/2510.04618), ICLR 2025)
- "OpenSage: Self-Programming Agent Generation Engine" ([arXiv:2602.16891](https://arxiv.org/html/2602.16891v1), 2026)
- "Meta Context Engineering via Agentic Skill Evolution" (Peking University, 2026)

### Industry
- [OpenAI Cookbook — Self-Evolving Agents](https://developers.openai.com/cookbook/examples/partners/self_evolving_agents/autonomous_agent_retraining/)
- [Claude Code Hooks Guide](https://www.eesel.ai/blog/hooks-in-claude-code)
- [Agentic Design Patterns 2026](https://www.sitepoint.com/the-definitive-guide-to-agentic-design-patterns-in-2026/)
- [Data Science Dojo — Agentic LLMs in 2025](https://datasciencedojo.com/blog/agentic-llm-in-2025/)
- [Agent Skills for Context Engineering (GitHub)](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering)

### Empirical (This Repo)
- HAI Operators agent benchmarking (March 2026): `read-sheet.sh` reduced sheet reads from 120s -> 5s (25x); `--group-by` + `--where` flags reduced aggregation from 245s -> 2.5s (100x); `bq-*.sh` scripts reduced BQ agent time from 120s -> 25s (5x)
