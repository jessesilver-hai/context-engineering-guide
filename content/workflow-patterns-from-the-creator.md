---
title: "Workflow Patterns from the Creator"
description: "Actionable patterns from Boris Cherny (Head of Claude Code at Anthropic) and the Claude Code team -- extracted from his public threads, the team's internal practices, and Anthropic's official best practices documentation."
tags:
  - context-engineering
  - claude-code
---

# Workflow Patterns from the Creator of Claude Code

Actionable patterns from Boris Cherny (Head of Claude Code at Anthropic) and the Claude Code team — extracted from his public threads, the team's internal practices, and Anthropic's official best practices documentation. Every pattern here earned its place by demonstrating measurable impact on output quality or workflow speed.

---

## 1. Start Vanilla — Override Only What's Broken

Boris's own setup is "surprisingly vanilla." Claude Code works well out of the box. The instinct to heavily customize before using it is wrong — you don't know what's broken until you use it.

**The practice:** Start with minimal CLAUDE.md. Add rules only after observing a real mistake. Test whether the rule actually changes behavior. Remove rules that don't.

**Why this matters:** Every instruction in CLAUDE.md competes for attention. Speculative rules dilute the rules you actually need. The Claude Code team's own CLAUDE.md is built inductively from observed failures, not deductively from imagined ones.

**The test for every line:** "Would removing this cause Claude to make a mistake?" If no, cut it.

---

## 2. CLAUDE.md Is an Error Log, Not a Spec Sheet

The Claude Code team updates their CLAUDE.md multiple times per week. Entries are added "anytime we see Claude do something incorrectly." This is reactive accumulation from real failures, not proactive specification from anticipated ones.

**What belongs in CLAUDE.md:**
- Bash commands Claude can't guess (`npm run build`, custom scripts)
- Code style rules that differ from language defaults
- Repository-specific etiquette (branch naming, PR conventions)
- Architectural decisions specific to your project
- Developer environment quirks (required env vars, non-obvious paths)
- Common gotchas and non-obvious behaviors

**What does NOT belong:**
- Anything Claude can figure out by reading code
- Standard language conventions Claude already knows
- Detailed API documentation (link to docs instead)
- Long explanations or tutorials
- File-by-file descriptions of the codebase
- Self-evident practices like "write clean code"

**Hard size target:** Under 200 lines per CLAUDE.md file (Anthropic official guidance). Files beyond this cause Claude to ignore instructions — not selectively, but uniformly across all rules.

---

## 3. Verification Loops Are the Highest-Leverage Pattern

Boris calls this "probably the most important thing." Give Claude the ability to verify its own work. This alone improves output quality 2-3x.

**Implementation by domain:**
- **CLI/backend:** Run `npm test`, `npm run typecheck`, `npm run lint` after implementation
- **Test suites:** Require unit + integration + E2E; enforce coverage thresholds (>80%)
- **Browser/UI:** Manual flow validation via Chrome extension or screenshots
- **API:** Curl requests validating success paths, validation errors, and auth

**The principle:** If Claude can check its own work, it will iterate until it passes. Without verification, Claude stops when it *thinks* it's done. With verification, it stops when it *is* done.

**Stop hooks** enable autonomous verification loops — blocking Claude's exit if tests fail and re-feeding the same prompt for iteration. The "Ralph Wiggum" technique loops this 50+ times unattended.

---

## 4. Two-Phase Workflow: Plan, Then Execute

**Phase 1 (Interactive Planning):** Use plan mode (`Shift+Tab` twice) to iterate on the approach with Claude until the strategy is solid. Challenge assumptions, refine scope, agree on the plan.

**Phase 2 (Automated Execution):** Toggle auto-accept mode and let Claude execute the finalized plan without interruption.

**Why this matters:** Skipping planning leads to "the classic failure of 40 unintended changes." Planning costs minutes; rework costs hours.

---

## 5. Slash Commands for Inner-Loop Workflows

Commands checked into `.claude/commands/` automate workflows done many times per day. The key pattern: **pre-compute context with inline bash** so Claude starts with full situational awareness.

Examples:
- `/commit-push-pr` — git status + branch info + recent commits, then commit + push + PR
- `/test-and-fix` — run tests, read failures, fix, re-run
- `/review-changes` — diff + context, then targeted code review

Commands should be thin delegation layers (<30 lines). Accept args, pre-load context with `!bash`, delegate to Claude's reasoning.

---

## 6. Subagents for Specialized Roles

Treat subagents like specialized team members with isolated context windows and scoped tool access:

- **code-simplifier** — runs post-implementation to reduce complexity and extract repeated logic
- **verify-app** — end-to-end testing agent with detailed verification instructions
- **staff-reviewer** — skeptical architectural assessment (catches overengineering)

**Design principles:**
- Single responsibility per agent
- Explicit tool restrictions (don't give a read-only agent write access)
- Use `model: haiku` for exploration agents, `model: sonnet` for implementation agents
- Agents do NOT inherit skills from the parent — declare `skills:` in frontmatter explicitly

---

## 7. Hooks for Deterministic Enforcement

CLAUDE.md instructions are advisory — Claude tries to follow them but compliance is probabilistic. Hooks are deterministic — they run unconditionally on every matching event.

**The decision rule:** If you'd be upset when Claude ignores a rule, it belongs in a hook, not CLAUDE.md.

Boris uses a **PostToolUse hook** that auto-formats code after every Edit/Write operation:
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{ "type": "command", "command": "npx prettier --write \"$CLAUDE_FILE_PATH\"" }]
    }]
  }
}
```

This catches the last 10% of formatting issues that would fail CI. No instruction in CLAUDE.md can match a hook's reliability for this kind of enforcement.

---

## 8. Scoped Permissions, Never Bypassed

Never use `--dangerously-skip-permissions` in normal workflows. Instead, pre-allow safe commands via settings:

```json
{
  "permissions": {
    "allow": [
      "Bash(git status)", "Bash(git diff)", "Bash(git commit *)", "Bash(git push *)",
      "Bash(npm test)", "Bash(npm run build)", "Bash(npm run lint)"
    ]
  }
}
```

Reserve `--dangerously-skip-permissions` for isolated sandbox environments only. Pre-allowing specific commands encodes safety as the default while removing friction for routine operations.

---

## 9. Optimize Total Iteration Cost, Not Per-Token Speed

Boris uses Opus with thinking for everything — not because it's faster per token, but because it requires fewer iterations. A model that gets it right on the first try is faster than a model that processes quickly but needs 3 correction cycles.

**The formula:** Total time = (generation time per attempt) x (number of attempts needed). Opus's superior reasoning and tool selection typically produces fewer total attempts, making it faster in wall-clock time despite higher per-token latency.

This applies to model selection everywhere — main conversation, subagents, and skills.

---

## 10. Parallel Sessions for Throughput

Boris runs 10-15 concurrent Claude Code sessions: 5 in terminal (tabbed, numbered, with OS notifications), 5-10 in browser, plus mobile sessions. The bottleneck is attention allocation, not generation speed.

**The practice:** Distribute tasks across sessions, check in when notifications fire, context-switch only when value is ready. Treat Claude sessions like async workers — queue tasks, harvest results.

---

## Sources

- [Boris Cherny's setup thread](https://x.com/bcherny/status/2007179832300581177) (June 2025)
- [Boris Cherny's team tips thread](https://x.com/bcherny/status/2017742741636321619) (July 2025)
- [DEV Community breakdown](https://dev.to/sivarampg/how-the-creator-of-claude-code-uses-claude-code-a-complete-breakdown-4f07)
- [Configuration repo](https://github.com/0xquinto/bcherny-claude)
- [Anthropic: Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Anthropic: How Claude Remembers](https://code.claude.com/docs/en/memory)
