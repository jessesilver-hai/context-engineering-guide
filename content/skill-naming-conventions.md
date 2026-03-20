---
title: "Skill Naming Conventions"
description: "Comprehensive guide to naming skills (slash commands / SKILL.md files) in Claude Code -- compiled from Anthropic's official spec, platform docs, engineering blog, community repos, and practitioner guides."
tags:
  - context-engineering
  - claude-code
---

# Skill Naming Conventions for Claude Code

Comprehensive guide to naming skills (slash commands / SKILL.md files) — compiled from Anthropic's official spec, platform docs, engineering blog, `anthropics/skills` repo, community repos, and practitioner guides.

---

## 1. Hard Constraints (Enforced by Spec)

The `name` field in SKILL.md frontmatter has strict validation rules from the [Agent Skills Specification](https://agentskills.io/specification):

| Rule | Detail |
|---|---|
| **Length** | 1-64 characters |
| **Allowed chars** | Lowercase `a-z`, digits `0-9`, hyphens `-` only |
| **No leading/trailing hyphens** | `-pdf` and `pdf-` are invalid |
| **No consecutive hyphens** | `pdf--processing` is invalid |
| **No uppercase** | `PDF-Processing` is invalid |
| **No underscores or spaces** | Hyphens only as separators |
| **No reserved words** | Cannot contain `anthropic` or `claude` |
| **No XML tags** | Cannot contain XML syntax |
| **Must match directory** | `name` field should match the parent directory name |

In Claude Code specifically, `name` is optional — the directory name is used if omitted. But the open standard requires it, and best practice is to always include it.

**Valid:** `pdf-processing`, `fix-issue`, `deploy`, `heteronym-qc-score`
**Invalid:** `PDF-Processing`, `claude-tools`, `-pdf`, `my_skill`, `pdf--proc`

Sources: [Agent Skills Spec](https://agentskills.io/specification) | [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

---

## 2. The Name vs. Description Split

**This is the single most important insight:** The name is for humans. The description is for Claude.

### How Claude selects skills at runtime

1. At startup, Claude loads `name` + `description` of all skills into its system prompt
2. Claude uses **pure LLM reasoning** on the description to decide which skill matches the task
3. No keyword matching, no embeddings — just language understanding against the description text

### What this means

- **Spend 20% of effort on the name** (human typing, menu scanning, permission rules)
- **Spend 80% on the description** (Claude's routing mechanism, trigger contexts, semantic matching)
- A great name with a vague description = low activation rate (~20%)
- A decent name with a rich description = high activation rate (50%+)

### Description best practices (from Anthropic's own [skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md))

- Be "pushy" — explicitly list trigger contexts
- Include **what** the skill does AND **when** to use it
- Write in third person: "Processes Excel files" not "I can help you"
- Use specific terms users would naturally say

**Good:** `"Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction."`

**Bad:** `"Helps with documents"`

### Budget constraint

All skill descriptions share a character budget: 2% of context window (~16,000 chars fallback). Override with `SLASH_COMMAND_TOOL_CHAR_BUDGET` env var. Keep descriptions information-dense but not bloated.

Sources: [Skills docs](https://code.claude.com/docs/en/skills) | [Agent Skills Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)

---

## 3. Three Naming Forms

Anthropic officially recommends gerund form but acknowledges alternatives. Community practice is mixed — **consistency within your collection matters more than which form you pick.**

### Gerund (verb + -ing) — Officially Recommended

Describes the skill as an ongoing activity/capability.

| Example | Why it works |
|---|---|
| `processing-pdfs` | Reads as "the activity of processing PDFs" |
| `analyzing-spreadsheets` | Natural noun phrase |
| `writing-documentation` | Describes capability clearly |
| `systematic-debugging` | Adjective + gerund for specificity |

**Best for:** Skills that represent a general capability Claude can apply. Background/reference skills.

### Imperative (verb + object) — "Acceptable Alternative"

Reads as a command — "do this thing."

| Example | Why it works |
|---|---|
| `deploy` | Direct, memorable, fast to type |
| `fix-issue` | Clear action |
| `commit` | Reads naturally after `/` |
| `pr-summary` | Action + object |

**Best for:** User-invoked task skills (`disable-model-invocation: true`). Skills that feel like commands.

### Noun Phrase — "Acceptable Alternative"

Anthropic's own `anthropics/skills` repo actually favors this pattern.

| Example | Why it works |
|---|---|
| `pdf` | Minimal, direct |
| `skill-creator` | Compound noun |
| `api-conventions` | Domain descriptor |
| `code-review` | Natural compound |

**Best for:** Domain/reference skills. Skills named after what they know rather than what they do.

### The Anthropic Contradiction

Official docs recommend gerund. Anthropic's own published skill repo uses mostly simple/compound nouns (`pdf`, `docx`, `xlsx`, `skill-creator`). Community repos split roughly 50/50 between gerund and imperative.

**Takeaway:** Pick one form, use it consistently, don't stress about which one.

Sources: [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) | [anthropics/skills repo](https://github.com/anthropics/skills)

---

## 4. Naming by Skill Type

### Reference/Background Skills

Claude auto-invokes these to add context. Users never type the name.

**Pattern:** Noun phrases describing the domain.
**Frontmatter:** `user-invocable: false`

| Name | Description Pattern |
|---|---|
| `api-conventions` | "API design patterns for this codebase" |
| `legacy-system-context` | "Explains how the old system works" |
| `coding-standards` | "Team coding standards and patterns" |

Since users don't type these, optimize the name for Claude's understanding. The description handles routing.

### Task/Workflow Skills

User controls timing. Side effects involved.

**Pattern:** Imperative verb + object. Short, memorable.
**Frontmatter:** `disable-model-invocation: true`

| Name | Description Pattern |
|---|---|
| `deploy` | "Deploy the application to production" |
| `fix-issue` | "Fix a GitHub issue by number" |
| `commit` | "Generate commit messages from staged changes" |
| `slack-check` | "Review Slack inbox for action items" |

These are commands. Name them like commands.

### Pipeline/Composite Skills

Multi-step processes, often with sub-skills.

**Pattern:** Shared prefix + step suffix.

| Name | Role |
|---|---|
| `heteronym-qc` | Parent pipeline |
| `heteronym-qc-score` | Child: scoring step |
| `heteronym-qc-eval` | Child: evaluation step |
| `ssot` | Parent pipeline |
| `ssot-update` | Child: update step |

The parent has the base name. Children append a suffix describing their role.

---

## 5. Namespace and Collision Avoidance

### Priority hierarchy

| Level | Path | Priority |
|---|---|---|
| Enterprise | Managed settings | Highest |
| Personal | `~/.claude/skills/<name>/SKILL.md` | High |
| Project | `.claude/skills/<name>/SKILL.md` | Medium |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Namespaced separately |

When skills share a name across levels, higher priority wins. Plugin skills are automatically namespaced as `plugin-name:skill-name`.

### Prefix strategies to avoid collisions

| Prefix | Use Case | Example |
|---|---|---|
| `my-` | Personal global skills | `my-review` |
| `team-` | Project-shared skills | `team-deploy` |
| `{project}-` | Project-scoped in multi-project env | `argus-qc`, `heteronym-qc` |
| `{domain}-` | Domain-scoped within a project | `data-validate`, `api-scaffold` |

### Subfolder namespacing

```
.claude/commands/team/review.md    --> /team:review
~/.claude/commands/perso/review.md --> /perso:review
```

The colon separator is auto-generated from folder structure. Monorepos get automatic skill discovery from nested `.claude/skills/` directories.

### Built-in command conflicts

**Never name a skill after a built-in:** `init`, `compact`, `review`, `memory`, `help`, `clear`, `permissions`. Use suffixed alternatives: `compact-plus`, `init-custom`.

Sources: [Skills docs](https://code.claude.com/docs/en/skills) | [SFEIR Institute: Common Mistakes](https://institute.sfeir.com/en/claude-code/claude-code-custom-commands-and-skills/errors/)

---

## 6. Composable Skill Naming

Skills can call other skills as subskills via the Skill tool (only restriction: can't re-invoke the currently running skill).

### Parent/child naming patterns

**Pattern 1: Prefix + role suffix** (most common, your existing pattern)
```
heteronym-qc           # parent
heteronym-qc-score     # child
heteronym-qc-eval      # child
```

**Pattern 2: Domain prefix + action suffix**
```
data-pipeline          # parent
data-validate          # child
data-transform         # child
data-export            # child
```

**Pattern 3: Verb + step suffix**
```
deploy                 # parent
deploy-verify          # child
deploy-rollback        # child
```

### Visibility for composable skills

| Child skill should be... | Frontmatter |
|---|---|
| Only callable by parent | `disable-model-invocation: true` + `user-invocable: false` |
| Callable by parent or user, but not Claude | `disable-model-invocation: true` |
| Callable by anyone | (defaults) |

### The three-tier architecture pattern

From [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice):

| Tier | Location | Naming | Purpose |
|---|---|---|---|
| Commands | `.claude/commands/` or skills with `disable-model-invocation: true` | Action-oriented: `/build-feature` | User entry points |
| Agents | `.claude/agents/` | Role-oriented: `frontend-developer` | Workflow executors |
| Skills | `.claude/skills/` with `user-invocable: false` | Domain-oriented: `api-conventions` | Knowledge/context |

---

## 7. Suffix Reference

Common suffixes seen across community repos and Anthropic examples:

| Suffix | Meaning | Example |
|---|---|---|
| `-eval` | Evaluation/verification | `heteronym-qc-eval` |
| `-score` | Scoring step | `heteronym-qc-score` |
| `-update` | Update/sync operation | `ssot-update` |
| `-check` | Validation/audit | `slack-check`, `deps-check` |
| `-audit` | Comprehensive audit | `deps-audit` |
| `-summary` | Summarization | `pr-summary` |
| `-plus` / `-custom` | Extended built-in | `compact-plus`, `init-custom` |
| `-trace` | Debugging/tracing | `debug-trace` |

---

## 8. Anti-Patterns

| Anti-Pattern | Why it fails | Fix |
|---|---|---|
| `helper`, `utils`, `tools` | Tells Claude nothing about when to use it | Name the specific capability |
| `claude-tools`, `anthropic-helper` | Fails validation (reserved words) | Remove reserved words |
| `init`, `compact`, `review` | Conflicts with built-in commands | Suffix: `init-custom`, `review-pr` |
| Mixing gerund + imperative + noun randomly | Inconsistent, harder to scan in menu | Pick one form, use it everywhere |
| Name > 30 chars for user-invoked skills | Painful to type as `/very-long-skill-name-here` | Shorten. Save detail for description. |
| Great name, vague description | Claude can't route to the skill | Invest in the description |
| `my-tool-v2`, `test-skill-final` | Version/status in the name | Use versioning in the directory or docs, not the name |

---

## 9. Decision Checklist

When naming a new skill, answer these:

1. **Who invokes it?** User-only = imperative verb. Claude-only = noun phrase. Both = either.
2. **Is it part of a group?** Yes = shared prefix (`heteronym-qc-*`, `data-*`).
3. **Could it collide?** Check built-ins. Check existing personal + project skills. Add prefix if needed.
4. **Is it typeable?** Under 30 chars for user-invoked skills. Aim for under 20.
5. **Is the description doing the heavy lifting?** The description should be 3-5x more detailed than the name. Include explicit trigger contexts.
6. **Does it match your collection's form?** Gerund, imperative, or noun — whatever your other skills use.

---

## Sources

- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills)
- [Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Agent Skills Overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Agent Skills Specification](https://agentskills.io/specification)
- [Anthropic Engineering Blog](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills)
- [Anthropic skills repo](https://github.com/anthropics/skills)
- [Anthropic skill-creator SKILL.md](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
- [Agent Skills Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)
- [wshobson/commands (57 commands)](https://github.com/wshobson/commands)
- [awesome-claude-skills](https://github.com/karanb192/awesome-claude-skills)
- [SFEIR Institute: Common Mistakes](https://institute.sfeir.com/en/claude-code/claude-code-custom-commands-and-skills/errors/)
- [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
- [The Autonomy Gap (Anthropic Research)](https://www.shashi.co/2026/02/the-autonomy-gap-what-anthropic-learned.html)
