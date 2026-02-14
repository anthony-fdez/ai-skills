---
name: writing-skills
description: Writes Claude Code skills for AI-assisted development patterns. Use when creating or updating .claude/skills/*/SKILL.md files.
---

# Writing Skills

Patterns for creating effective Claude Code skills that serve as both AI guidance and code review checklists.

## Contents

- [Use Frontmatter with Trigger Phrase](#use-frontmatter-with-trigger-phrase)
- [Use Table of Contents as Code Review Checklist](#use-table-of-contents-as-code-review-checklist)
- [Name Patterns with Verb Phrases](#name-patterns-with-verb-phrases)
- [Explain Why After Each Pattern](#explain-why-after-each-pattern)
- [Scope Patterns to This Codebase](#scope-patterns-to-this-codebase)
- [Follow Skill File Conventions](#follow-skill-file-conventions)

---

## Use Frontmatter with Trigger Phrase

Every skill needs YAML frontmatter with `name` and `description`. The description must include a "Use when..." trigger phrase so Claude knows when to apply the skill automatically.

```yaml
---
name: skill-name
description: Brief description. Use when [trigger condition].
---
```

- `name` — kebab-case, matches the folder name
- `description` — one sentence of purpose + one "Use when..." sentence

Why: The trigger phrase is how Claude decides whether to load a skill for the current task. Without it, the skill may never activate.

---

## Use Table of Contents as Code Review Checklist

The `## Contents` section serves two purposes: navigation AND a checklist. Every item must be a pattern or anti-pattern named as something verifiable:

**Good — actionable, verifiable:**

- "Create Dedicated Query Hooks"
- "Validate Input with Zod at API Boundaries"
- "Avoid Syncing React Query Data to Zustand"

**Bad — categorical, not enforceable:**

- ~~"Query Patterns"~~
- ~~"Architecture Overview"~~
- ~~"Error Handling"~~

Why: When reviewing code, scan the TOC to check whether each pattern was followed. Category headers don't tell you what to check.

---

## Name Patterns with Verb Phrases

Each `##` heading is an acceptance criterion for code. Name them with imperative verb phrases: "Use X", "Create Y", "Avoid Z", "Keep X as Y".

| Verb    | When to use                         | Example                                |
| ------- | ----------------------------------- | -------------------------------------- |
| Use     | Prescribing a specific tool/pattern | "Use select for Transformations"       |
| Create  | Something should be built           | "Create Dedicated Query Hooks"         |
| Keep    | Preserving a quality                | "Keep Mutation Hooks as Thin Wrappers" |
| Avoid   | Anti-pattern                        | "Avoid Wrapping Mutations in useCallback" |
| Derive  | Computed values                     | "Derive State from Mutations"          |

Why: Verb phrases make patterns testable. "Use X" can be checked — "X Overview" cannot.

---

## Explain Why After Each Pattern

Add a `Why:` line after patterns where the reasoning isn't obvious. This helps both AI and developers understand the tradeoff, not just the rule.

```markdown
## Keep Query Functions Pure

[explanation and examples]

Why: Side effects in queryFn run on every refetch, causing duplicated store updates and unpredictable behavior.
```

**Skip "Why:" when the reason is self-evident** (e.g., "Use Consistent Naming" doesn't need justification).

Why: Rules without reasoning become cargo cult. When developers understand the tradeoff, they can make correct judgment calls in edge cases.

---

## Scope Patterns to This Codebase

Skills should document patterns **specific to this codebase**, not general programming knowledge. Claude already knows general TypeScript, React, and Next.js patterns.

**Worth documenting:**

- How we use our logger (`@/lib/logger`) with DataDog tags
- Our Zustand store slice conventions
- Our API route error response pattern (`createErrorResponse`)
- Service-specific patterns (Hydra, Cosmic, Jeeves)

**Not worth documenting:**

- How `useState` works
- What TypeScript generics are
- Standard React Query setup

**One skill per domain** — don't create overlapping skills. Check existing skills in `.claude/skills/` before creating a new one.

Why: Generic patterns bloat skills without adding value. Codebase-specific patterns are what Claude can't infer from general training.

---

## Follow Skill File Conventions

```
.claude/skills/{skill-name}/SKILL.md
```

- **Folder name**: kebab-case, matches frontmatter `name`
- **One skill per domain**: e.g., `writing-react-query`, `securing-code`
- **Structure**: Frontmatter → Title → One-line purpose → Contents → Patterns (with `---` separators)
- **Reference framework in title when framework-specific**: e.g., a pattern that only applies to Next.js should say "(Next.js)" in the heading

**Skill size targets:**

- Aim for 100–300 lines
- If a skill exceeds 400 lines, consider splitting into two domains
- If a skill is under 50 lines, it may not need to be a standalone skill — consider merging into a related one
