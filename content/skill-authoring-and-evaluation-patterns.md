---
title: "Skill Authoring and Evaluation Patterns"
description: "How to write effective skills and iteratively improve them -- extracted from Anthropic's official skill-creator skill (the meta-skill for building skills), supplemented by patterns from the Agent Skills Specification and platform docs."
tags:
  - context-engineering
  - claude-code
---

# Skill Authoring and Evaluation Patterns

How to write effective skills and iteratively improve them — extracted from Anthropic's official `skill-creator` skill (the meta-skill for building skills), supplemented by patterns from the Agent Skills Specification and platform docs.

---

## 1. Skill Anatomy

Every skill lives in a directory with this structure:

```
skill-name/
├── SKILL.md              # Required. YAML frontmatter + markdown instructions.
└── (optional resources)
    ├── scripts/          # Executable code for deterministic/repetitive tasks
    ├── references/       # Docs loaded into context as needed
    └── assets/           # Files used in output (templates, icons, fonts)
```

### The Three-Level Loading System

Skills use progressive disclosure — not everything loads at once:

| Level | What Loads | When | Budget |
|---|---|---|---|
| **Metadata** | `name` + `description` from frontmatter | Always in context at session start | ~100 words per skill, shared ~16K char budget across all skills |
| **SKILL.md body** | Full markdown instructions | When a task matches the description | <500 lines ideal |
| **Bundled resources** | Scripts, references, assets | Only when explicitly read/executed from within the skill | Unlimited (scripts execute without loading into context) |

This means: **descriptions must be good enough to trigger loading, SKILL.md must be concise enough to fit in context, and heavy content belongs in reference files.**

---

## 2. Writing Effective SKILL.md Files

### Frontmatter Fields

```yaml
---
name: skill-name                    # Must match directory name. Kebab-case only.
description: >
  Rich multi-line description...    # PRIMARY triggering mechanism. 3-5 lines.
user-invocable: false               # true = slash command, false = auto-triggered
argument-hint: "[args]"             # Shown in / menu (commands only)
disable-model-invocation: true      # true = user controls timing (commands only)
compatibility:                      # Optional, rarely needed
  requires_tools:
    - Bash
---
```

### Body Structure Principles

- **Keep SKILL.md under 500 lines.** If approaching this limit, add hierarchy — extract sections into `references/*.md` with clear pointers about when to read them.
- **Use imperative form.** "Read the file" not "You should read the file."
- **Explain the why, not just the what.** LLMs respond better to understanding reasoning than to rigid directives. Instead of "ALWAYS use format X", explain why format X matters — the model will follow more reliably and generalize better.
- **Include examples.** Claude performs notably better with concrete input/output pairs than abstract rules. Format them clearly:
  ```markdown
  **Example 1:**
  Input: Added user authentication with JWT tokens
  Output: feat(auth): implement JWT-based authentication
  ```
- **Define output formats explicitly** when the skill produces structured output:
  ```markdown
  ## Report structure
  ALWAYS use this exact template:
  # [Title]
  ## Executive summary
  ## Key findings
  ## Recommendations
  ```
- **Organize by variant** when a skill supports multiple domains/frameworks. Claude reads only the relevant reference file:
  ```
  cloud-deploy/
  ├── SKILL.md (workflow + selection logic)
  └── references/
      ├── aws.md
      ├── gcp.md
      └── azure.md
  ```
- **For large reference files (>300 lines),** include a table of contents at the top so Claude can navigate efficiently.

### Yellow Flags in Skill Writing

These patterns suggest the skill could be improved:

| Pattern | Problem | Better Approach |
|---|---|---|
| Heavy use of ALWAYS/NEVER/MUST in caps | Rigid, brittle, doesn't generalize | Explain the reasoning — the model will follow more reliably |
| Super-narrow examples only | Overfits to specific cases | Use theory of mind; make instructions general enough to transfer |
| No examples at all | Model has to guess the output format | Include 2-3 concrete input/output pairs |
| >500 lines in SKILL.md | Too much context loaded at once | Extract to reference files with routing index |
| Redundant instructions across sections | Wastes token budget | State once where first encountered, reference later |

### Script Bundling Signals

If during testing you notice the model independently writing the same helper script across multiple test cases (e.g., all test runs create a `build_chart.py`), that's a strong signal the skill should bundle that script. Write it once in `scripts/`, and tell the skill to use it. This saves every future invocation from reinventing the wheel.

---

## 3. Description Writing (The Most Important Field)

The `description` field is the primary mechanism that determines whether Claude invokes a skill. Claude sees every installed skill's description at session start and uses LLM reasoning on the description text to decide which skills match a given task.

### Current Triggering Behavior

Claude tends to **under-trigger** skills — it won't use them even when they'd be useful. To combat this, descriptions should be slightly "pushy": explicitly list when to use the skill, what task types it covers, and what domain terms should trigger it.

### Description Anatomy

A good description has three components:

1. **What it does** (1 sentence): Core capability
2. **When to use it** (2-3 sentences): Explicit trigger contexts, task types, user phrasings
3. **Domain terms** (embedded naturally): Keywords the model matches against

**Bad:** `"How to build a simple fast dashboard to display internal data."`

**Good:** `"How to build a simple fast dashboard to display internal data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"`

### Description vs. Body

All "when to use" information goes in the description, not the body. The description is always in context; the body only loads after triggering. If triggering logic is buried in the body, it never gets read in time to make the triggering decision.

### How Triggering Actually Works

Skills appear in Claude's `available_skills` list with their `name` + `description`. Claude decides whether to consult a skill based on that description. Important nuance: **Claude only consults skills for tasks it can't easily handle on its own.** Simple, one-step queries may not trigger a skill even if the description matches perfectly, because Claude can handle them directly with basic tools. Complex, multi-step, or specialized queries reliably trigger skills when the description matches.

---

## 4. User-Invocable vs. Auto-Triggered

| Type | Frontmatter | When to Use | Examples |
|---|---|---|---|
| **Auto-triggered** (`user-invocable: false`) | Description handles routing | Reference knowledge Claude consults automatically | Playbooks, schemas, guides, domain knowledge |
| **Slash command** (`user-invocable: true`) | User types `/skill-name` | Action workflows with side effects where user controls timing | Deploy, offboard, create ramp plan, run pipeline |

For slash commands, also set `disable-model-invocation: true` so Claude won't invoke the command autonomously — the user decides when to run it.

---

## 5. Iterative Skill Development

Skills improve through test-evaluate-revise cycles. Iteration count is a strong predictor of skill quality.

### The Core Loop

1. **Draft** the skill based on intent and requirements
2. **Write 2-3 realistic test prompts** — the kind of thing a real user would actually say
3. **Run** Claude with the skill on each test prompt (and without the skill as baseline)
4. **Evaluate** outputs both qualitatively (human review) and quantitatively (assertions)
5. **Revise** the skill based on feedback
6. **Repeat** until outputs consistently meet quality bar
7. **Expand** the test set and try again at larger scale

### Writing Good Test Prompts

Test prompts should be realistic — detailed, specific, with personal context. Not abstract requests.

**Bad:** `"Format this data"`, `"Extract text from PDF"`, `"Create a chart"`

**Good:** `"ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think"`

Mix different lengths, levels of formality, and include edge cases. Focus on cases where the skill's value is ambiguous, not obvious wins.

### Revision Principles

When improving a skill based on test results:

1. **Generalize from feedback.** The skill will be used across many different prompts. If a fix only works for the specific test case, it's overfitting. Look for the underlying pattern and address that.
2. **Keep the prompt lean.** Read the transcripts, not just the final outputs. If the model is wasting time on unproductive steps, remove the instructions causing that behavior.
3. **Explain the why.** Instead of adding rigid constraints, explain the reasoning so the model understands and generalizes. "NEVER use format X" is less effective than "Format Y is preferred because [reason] — it handles edge cases like [example] that format X misses."
4. **Look for repeated work across test cases.** If all test runs independently create similar helper scripts, bundle that script into the skill.

---

## 6. Quantitative Evaluation (Assertions)

For skills with objectively verifiable outputs (file transforms, data extraction, code generation, fixed workflow steps), write assertions that can be checked programmatically.

### Good Assertion Qualities

- **Objectively verifiable** — can be checked by code, not human judgment
- **Descriptive names** — someone glancing at results immediately understands what each checks
- **Discriminating** — the assertion should pass with the skill and fail (or pass less often) without it. If it always passes regardless, it's not testing the skill's value.

### When NOT to Use Assertions

Skills with subjective outputs (writing style, design quality, creative work) are better evaluated qualitatively through human review. Don't force assertions onto things that need human judgment.

---

## 7. Description Optimization (Advanced)

After a skill is stable, its description can be systematically optimized for triggering accuracy.

### Process

1. **Create trigger eval queries** — 20 queries split between should-trigger (8-10) and should-not-trigger (8-10)
2. **Focus on edge cases.** Should-trigger queries should include uncommon use cases and cases where the skill competes with others. Should-not-trigger queries should be near-misses — queries that share keywords but actually need something different.
3. **Run optimization** — evaluate the current description against queries, propose improvements based on failures, re-evaluate, iterate
4. **Use held-out test set** — split 60% train / 40% test to avoid overfitting the description to the eval set

### Eval Query Quality

Valuable negative test cases are near-misses, not obviously irrelevant queries. "Write a fibonacci function" as a negative test for a PDF skill is too easy. The negative cases should be genuinely tricky — adjacent domains, ambiguous phrasing where keyword matching would trigger but shouldn't.

---

## 8. Command vs. Skill Decision Tree

```
Does the user explicitly invoke it?
├── YES → It's a command (user-invocable: true)
│   ├── Does it have side effects? → disable-model-invocation: true
│   └── Is it pure information? → Consider making it auto-triggered instead
└── NO → It's a background skill (user-invocable: false)
    ├── Is it reference knowledge? → SKILL.md + reference files
    ├── Is it a workflow with judgment calls? → SKILL.md with decision frameworks
    └── Is it deterministic steps? → Consider a script instead
```

---

## 9. Common Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|---|---|---|
| One-line description | Claude can't decide when to trigger | Write 3-5 lines with trigger contexts and domain terms |
| Everything in SKILL.md body | Exceeds 500 lines, loads too much context | Extract to reference files with routing index |
| No examples in skill | Model guesses output format, often wrong | Add 2-3 concrete input/output examples |
| Overfitting to test cases | Skill works for 3 examples, fails on real usage | Generalize from feedback patterns, not specific fixes |
| Rigid MUST/NEVER without reasoning | Brittle, doesn't transfer to novel cases | Explain the why — model follows reasoning better than rules |
| Description duplicates body content | Wastes the always-in-context budget | Description = when to trigger. Body = how to execute. |
| Excluding large reference files | Loses useful context | Use `[GREP-ONLY]` headers instead of exclusion |
| Creating parallel files for one topic | Confuses routing, wastes context | Merge into single file organized by subtopic |

---

## Sources

- Anthropic's official `skill-creator` skill (Claude Code plugin, Feb 2026)
- [Agent Skills Specification](https://agentskills.io/specification)
- [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- Anthropic engineering blog: "Building effective skills for Claude Code" (2026)
