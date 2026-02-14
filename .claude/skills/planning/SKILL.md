---
name: planning
description: Creates implementation plans for features and changes. Use when planning any non-trivial work before implementation begins.
---

# Planning

Creating implementation plans that ensure nothing is missed and all context is gathered before coding begins.

## When to Plan

**ALWAYS plan when:**

- Implementing new features
- Making significant changes to existing functionality
- The user asks to "plan" or "think through" any work
- Before starting any non-trivial implementation

**Skip planning for:**

- Quick bug fixes with obvious solutions
- Simple one-line changes
- Research-only tasks

---

## Planning Process

| Phase | Name              | Purpose                                          | Tools                              |
| ----- | ----------------- | ------------------------------------------------ | ---------------------------------- |
| 1     | Enter Plan Mode   | Signal intent to plan, explore before committing  | `EnterPlanMode`                    |
| 2     | Context Gathering | Read docs and search codebase for affected areas  | `Explore` agents, `Read`, `Grep`   |
| 3     | Plan Creation     | Write detailed step-by-step implementation plan   | `Write` to `docs/plans/`          |
| 4     | Review            | Present plan for user approval                    | `ExitPlanMode`                     |

---

## Phase 1: Enter Plan Mode

**Start every plan by entering plan mode.** This signals to the user that you're exploring and designing — not writing code yet.

Plan mode gives you access to read and search tools without making changes. Use this phase to understand the problem space before proposing a solution.

---

## Phase 2: Context Gathering

**Goal**: Understand the full scope before writing the plan.

### Read Existing Documentation

1. Check `docs/architecture/` for related system docs
2. Check `docs/features/` for related feature docs
3. Check `docs/plans/` for existing or related plans
4. Read `CLAUDE.md` for project patterns

### Search the Codebase

Use `Explore` agents to search multiple areas in parallel. Launch independent searches simultaneously rather than sequentially:

- **Related code** — files that touch the feature area
- **Dependencies** — what this feature depends on
- **Dependents** — what depends on code you'll change
- **Tests** — existing tests that may need updates
- **State management** — store slices, React Query hooks involved
- **UI components** — components that will need changes

### Identify Applicable Skills

Scan `.claude/skills/` and note which skills apply. For each relevant skill, note the key patterns and anti-patterns that affect this implementation.

### Record Findings

Document in the plan:
- Which docs were reviewed and key insights
- All files that need modification or creation
- External services involved
- Which skills must be followed

---

## Phase 3: Plan Creation

**Goal**: Write a detailed, step-by-step plan that someone could implement from alone.

### Save Location

All plans go in `docs/plans/{feature-name}-plan.md`

Examples: `checkout-payment-retry-plan.md`, `user-authentication-flow-plan.md`

### Plan Document Structure

```markdown
# [Feature Name] Implementation Plan

## Overview

What this plan implements and why. 1-2 sentences.

## Context Gathered

### Documentation Reviewed

- List of docs read and key insights

### Codebase Analysis

- Files affected, dependencies, state management involved

### Applicable Skills

- List of skills that must be followed, with key patterns noted

## Implementation Plan

### Phase 1: [Phase Name]

#### Step 1: [Step Title]

**Problem**: What issue or gap does this step address?

**Changes**:

| File              | Change                |
| ----------------- | --------------------- |
| `path/to/file.ts` | Description of change |

**Details**: Specific patterns to follow, edge cases, error handling.

#### Step 2: [Step Title]

...

### Phase 2: [Phase Name]

...

## Post-Implementation

### Documentation to Create

| Document    | Location                         |
| ----------- | -------------------------------- |
| Feature doc | `docs/features/{name}/{name}.md` |

### Documentation to Update

| Document                  | Update Required                |
| ------------------------- | ------------------------------ |
| `docs/architecture/X.md`  | Add section on Y               |

### Reminder

After implementation, use the `writing-design-docs` skill to create feature documentation with Purpose/Behavior/Verify structure.
```

### Plan Characteristics

- **Comprehensive** — include every step, no matter how small
- **Ordered** — steps in logical execution order
- **Detailed** — file paths, function names, specific changes
- **Self-contained** — implementable from the plan alone

---

## Phase 4: Review

**Exit plan mode to present the plan for user approval.** The user reads the plan, asks questions, requests changes, or approves.

Nothing gets implemented until the user explicitly approves.

---

## Post-Implementation

After implementation is complete:

1. **Verify** — all acceptance criteria met
2. **Lint** — `npm run formatAndLint` passes
3. **Document** — use the `writing-design-docs` skill to create feature docs at `docs/features/{feature-name}/{feature-name}.md` with Purpose/Behavior/Verify
4. **Update** — any existing docs affected by the change
5. **Clean up** — delete or archive the plan from `docs/plans/`

---

## Using Agents Effectively

### Parallel Exploration

During context gathering, launch multiple `Explore` agents simultaneously for independent searches:

```
Agent 1: "Find all files related to payment processing"
Agent 2: "Find all React Query hooks in the checkout flow"
Agent 3: "Search for error handling patterns in the cart"
```

### Parallel Implementation

During implementation, use `Task` agents for independent work streams:

```
Agent 1: Create type definitions
Agent 2: Scaffold component structure
Agent 3: Write test fixtures
```

Only parallelize truly independent work. If step B depends on step A's output, run them sequentially.
