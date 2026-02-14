---
name: planning-features
description: Creates comprehensive implementation plans for new features and changes. Use when planning work with Claude Code, before implementing any non-trivial feature or change.
---

# Planning Features

Creating comprehensive implementation plans that ensure nothing is missed and all context is gathered before coding begins.

## When to Use This Skill

**ALWAYS use this skill when:**

- Planning new feature implementations
- Planning significant changes to existing functionality
- The user asks to "plan" or "create a plan" for any work
- Before starting any non-trivial implementation

**This skill is NOT for:**

- Quick bug fixes with obvious solutions
- Simple one-line changes
- Research-only tasks (use exploration instead)

---

## Contents

1. [Planning Process](#planning-process)
2. [Phase 1: Context Gathering](#phase-1-context-gathering)
3. [Phase 2: Requirements Discovery](#phase-2-requirements-discovery)
4. [Phase 3: Skill Review](#phase-3-skill-review)
5. [Phase 4: Implementation Plan](#phase-4-implementation-plan)
6. [Phase 5: Documentation Planning](#phase-5-documentation-planning)
7. [Plan Document Structure](#plan-document-structure)
8. [Plan Location](#plan-location)
9. [Post-Implementation](#post-implementation)
10. [Checklist](#checklist)

---

## Planning Process

Every plan follows these phases in order:

| Phase | Name                   | Purpose                                                  |
| ----- | ---------------------- | -------------------------------------------------------- |
| 1     | Context Gathering      | Read existing documentation to understand scope          |
| 2     | Requirements Discovery | Search codebase to find all affected areas               |
| 3     | Skill Review           | Identify relevant skills to follow during implementation |
| 4     | Implementation Plan    | Create detailed step-by-step implementation              |
| 5     | Documentation Planning | Plan what docs to create/update after completion         |

---

## Phase 1: Context Gathering

**Goal**: Understand the full scope by reading existing documentation first.

### Steps

1. **Read relevant architecture docs** in `docs/architecture/`
2. **Read related feature docs** in `docs/features/`
3. **Check for existing plans** in `docs/plans/` that may relate
4. **Read troubleshooting docs** in `docs/troubleshooting/` for known issues
5. **Check CLAUDE.md** for project-specific patterns and requirements

### What to Look For

- How existing systems work
- Architectural decisions and their rationale
- Related features that may be affected
- Known issues or constraints
- Established patterns to follow

### Document Findings

Record in the plan:

- Which docs were reviewed
- Key insights that affect the implementation
- Constraints or requirements discovered

---

## Phase 2: Requirements Discovery

**Goal**: Search the entire codebase to find all areas affected by the change.

### Steps

1. **Search for related code** - Find all files that touch the feature area
2. **Identify dependencies** - What does this feature depend on?
3. **Find dependents** - What depends on this feature?
4. **Check for tests** - Existing tests that may need updates
5. **Review API contracts** - External APIs or internal interfaces affected
6. **Check state management** - Store slices, React Query hooks involved
7. **Find UI components** - Components that will need changes

### Search Patterns

```bash
# Find related files
grep -r "featureName" src/
grep -r "relatedFunction" src/

# Find imports
grep -r "from.*featureModule" src/

# Find usages
grep -r "useFeatureHook" src/
```

### Document Findings

Record in the plan:

- All files that need modification
- All files that need creation
- Dependencies and their versions
- External services involved
- State management affected

---

## Phase 3: Skill Review

**Goal**: Identify which skills apply to this implementation and ensure they will be followed.

### Steps

1. **Review available skills** in `.claude/skills/`
2. **Identify relevant skills** based on what the implementation involves
3. **Note key patterns** from each relevant skill that must be followed
4. **Document skill requirements** in the plan

### Skill Selection Criteria

| If Implementation Involves... | Review Skill         |
| ----------------------------- | -------------------- |
| TypeScript code               | Check relevant skill |
| React components              | Check relevant skill |
| Data fetching                 | Check relevant skill |
| Forms                         | Check relevant skill |
| State management              | Check relevant skill |
| Error handling                | Check relevant skill |
| API routes                    | Check relevant skill |
| Tests                         | Check relevant skill |
| File creation                 | Check relevant skill |
| Logging                       | Check relevant skill |

### Document in Plan

For each relevant skill, note:

- Skill name
- Key patterns that apply to this implementation
- Anti-patterns to avoid

**CRITICAL**: During implementation, Claude MUST check and follow the identified skills before writing any code.

---

## Phase 4: Implementation Plan

**Goal**: Create a detailed, step-by-step implementation plan with all necessary details.

### Plan Characteristics

- **Comprehensive** - Include every step, no matter how small
- **Ordered** - Steps in logical execution order
- **Detailed** - Include file paths, function names, specific changes
- **Self-contained** - Someone could implement from the plan alone
- **Size doesn't matter** - The plan should be as long as needed

### Step Structure

Each step should include:

```markdown
## Step N: [Descriptive Title]

### Problem

What issue or gap does this step address?

### Changes Required

| File              | Change                |
| ----------------- | --------------------- |
| `path/to/file.ts` | Description of change |

### Implementation Details

Detailed explanation of what to do, including:

- Specific code patterns to follow
- Edge cases to handle
- Error handling requirements

### Acceptance Criteria

- [ ] Criterion 1
- [ ] Criterion 2
```

### Grouping Steps

Group related steps into phases:

```markdown
## Phase 1: Foundation

Steps 1-3...

## Phase 2: Core Implementation

Steps 4-7...

## Phase 3: Integration

Steps 8-10...
```

---

## Phase 5: Documentation Planning

**Goal**: Plan what documentation to create or update after implementation.

### Required Documentation

After every feature implementation:

1. **Feature docs** - Create under `docs/features/{feature-name}/`
2. **Acceptance criteria** - Testable criteria for the feature
3. **Architecture docs** - Update if architectural patterns change
4. **Existing docs** - Update any docs affected by the change

### Document in Plan

```markdown
## Post-Implementation Documentation

### New Documentation

| Document            | Location                                      | Purpose                   |
| ------------------- | --------------------------------------------- | ------------------------- |
| Feature doc         | `docs/features/{name}/{name}.md`              | Explain how feature works |
| Acceptance criteria | `docs/features/{name}/acceptance-criteria.md` | Testable criteria         |

### Documentation Updates

| Document                 | Update Required                |
| ------------------------ | ------------------------------ |
| `docs/architecture/X.md` | Add section on Y               |
| `docs/features/Z/Z.md`   | Update to reflect new behavior |
```

### Skill Reminder

**CRITICAL**: After implementation is complete, use the `documenting-systems` skill to:

1. Create proper feature documentation
2. Write acceptance criteria in Given/When/Then format
3. Update any existing documentation affected by the change

---

## Plan Document Structure

Every plan document follows this structure:

```markdown
# [Feature Name] Implementation Plan

## Overview

Brief description of what this plan implements and why.

## Context Gathered

### Documentation Reviewed

- List of docs read and key insights

### Codebase Analysis

- Files affected
- Dependencies identified
- State management involved

## Relevant Skills

List of skills that must be followed during implementation, with key patterns noted.

## Implementation Plan

### Phase 1: [Phase Name]

#### Step 1: [Step Title]

[Step details as described above]

### Phase 2: [Phase Name]

...

## Acceptance Criteria

Overall acceptance criteria for the complete implementation.

## Post-Implementation

### Documentation to Create

- Feature docs
- Acceptance criteria

### Documentation to Update

- Existing docs affected

### Reminder

After implementation, use the `documenting-systems` skill to create/update all documentation.
```

---

## Plan Location

**All plans MUST be saved under `docs/plans/`**

### Naming Convention

`{feature-name}-plan.md`

Examples:

- `checkout-payment-retry-plan.md`
- `user-authentication-flow-plan.md`
- `product-grid-infinite-scroll-plan.md`

### Plan Lifecycle

1. **Created** - When planning begins
2. **Updated** - As implementation reveals new requirements
3. **Archived or Deleted** - After implementation complete and docs created

---

## Post-Implementation

After the implementation is complete:

1. **Verify all acceptance criteria** are met
2. **Run tests and linting** - `npm run formatAndLint`
3. **Use `documenting-systems` skill** to create documentation:
   - Create feature folder: `docs/features/{feature-name}/`
   - Write main feature doc: `{feature-name}.md`
   - Write acceptance criteria: `acceptance-criteria.md`
4. **Update existing docs** affected by the change
5. **Delete or archive the plan** - The feature docs replace it

---

## Checklist

### Before Starting Plan

- [ ] User has requested planning or a non-trivial implementation
- [ ] Created plan file in `docs/plans/{feature-name}-plan.md`

### Phase 1: Context Gathering

- [ ] Read relevant docs in `docs/architecture/`
- [ ] Read relevant docs in `docs/features/`
- [ ] Checked `docs/plans/` for related plans
- [ ] Reviewed `CLAUDE.md` for project patterns
- [ ] Documented findings in plan

### Phase 2: Requirements Discovery

- [ ] Searched codebase for all affected files
- [ ] Identified dependencies
- [ ] Identified dependents
- [ ] Found existing tests
- [ ] Reviewed API contracts
- [ ] Checked state management impact
- [ ] Found UI components affected
- [ ] Documented all findings in plan

### Phase 3: Skill Review

- [ ] Reviewed available skills in `.claude/skills/`
- [ ] Identified all relevant skills for this implementation
- [ ] Noted key patterns from each skill
- [ ] Documented skills in plan

### Phase 4: Implementation Plan

- [ ] Created detailed step-by-step plan
- [ ] Each step has problem, changes, details, and criteria
- [ ] Steps are in logical order
- [ ] Plan is comprehensive (no missing steps)
- [ ] File paths and specifics are included

### Phase 5: Documentation Planning

- [ ] Planned feature docs to create
- [ ] Planned acceptance criteria to write
- [ ] Identified existing docs to update
- [ ] Added reminder to use `documenting-systems` skill

### Post-Implementation

- [ ] All acceptance criteria verified
- [ ] `npm run formatAndLint` passes
- [ ] `documenting-systems` skill used for documentation
- [ ] Feature docs created in `docs/features/{feature-name}/`
- [ ] Acceptance criteria written in Given/When/Then format
- [ ] Existing docs updated
- [ ] Plan archived or deleted
