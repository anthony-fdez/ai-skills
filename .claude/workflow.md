# Development Workflow with Claude Code

How to ship features and fixes using Claude Code, from start to PR.

The full loop: **Prompt → Plan → Review Plan → Implement → Review Code → Document → Verify → PR**

---

## 1. Write a Detailed Prompt

The quality of your prompt determines the quality of everything that follows. Be specific about the problem — Claude can't read your mind, but it can read your codebase.

**Include as much context as you can:**

- What the problem is or what the feature should do
- Why it matters (business context, user impact)
- Edge cases you already know about
- Constraints (performance, compatibility, scope)
- Relevant file paths, error messages, or reproduction steps

**End your prompt with:** "Use the planning skill to create a comprehensive plan."

This tells Claude to enter plan mode and explore the codebase before writing any code. It will search for affected files, read existing documentation, and identify which coding patterns (skills) apply.

---

## 2. Review the Plan

Claude produces a plan at `docs/plans/{feature-name}-plan.md` with every file it will touch, the specific changes it will make, and the order it will make them in.

**This is the most important review in the entire workflow.** Catching a wrong approach here costs you one sentence of feedback. Catching it after implementation costs you a revert.

- Read the plan thoroughly
- Push back on anything that feels off
- Ask "what about X?" to surface edge cases Claude missed
- Request changes until the plan covers everything
- Approve only when you're satisfied — nothing gets implemented until you explicitly say so

Iterate as many times as needed. Plans are cheap, rewrites are expensive.

---

## 3. Implement

Tell Claude to implement the approved plan. It will follow the plan step by step, using the coding patterns defined in the skills (TypeScript, React, React Query, etc.) and running `npm run formatAndLint` after changes.

**Your job during implementation:** Watch the progress. If something looks off, interrupt and redirect. You can pause at any time — Claude won't keep going if you tell it to stop.

---

## 4. Review the Code

Before moving on, review the implementation carefully. This is your standard code review, but with extra attention to:

- Were all patterns from the relevant skills followed?
- Does the code match the approved plan?
- Are edge cases and error handling covered?
- Were any unnecessary files created or modified?
- Does the code look like something you'd write yourself?

Request fixes for anything that doesn't meet your standards. Don't move to documentation until the code is right.

---

## 5. Document

Once the code is solid, run the `/writing-design-docs` command. Claude will create a feature doc at `docs/features/{feature-name}/{feature-name}.md` using the Purpose/Behavior/Verify structure:

- **Purpose** — one sentence explaining what it does and why
- **Behavior** — numbered trigger/outcome scenarios describing how it works
- **Verify** — steps to confirm each behavior works correctly

Review the doc and make sure the behaviors match what was actually built. Documentation is written after implementation so it reflects reality, not intentions.

---

## 6. Verify the Branch

Run the `/verify-branch` command. This is the safety net — it diffs your branch against main, finds every design doc affected by your changes, and flags anything that's stale:

- Behaviors that changed but weren't updated in the docs
- New behaviors that aren't captured anywhere
- Verify steps that reference moved routes or changed API shapes

This catches the documentation you forgot to update. Approve or reject the suggested changes, and optionally let Claude run verification steps against the actual app (preview or local).

---

## 7. Create the PR

Run the `/create-pr` command. Claude analyzes all commits on the branch, generates a PR title, description, technical details, and a testing checklist. It presents everything for your review before pushing anything.

Confirm when it looks good, and Claude pushes the branch and creates the PR on GitHub.

---

## Quick Reference

| Step             | What you do                        | What Claude does                     | Command / Skill        |
| ---------------- | ---------------------------------- | ------------------------------------ | ---------------------- |
| 1. Prompt        | Write detailed problem description | —                                    | —                      |
| 2. Review Plan   | Read, question, approve            | Explore codebase, produce plan       | `planning` skill       |
| 3. Implement     | Monitor, redirect                  | Write code following plan and skills | —                      |
| 4. Review Code   | Inspect changes, request fixes     | Fix issues                           | —                      |
| 5. Document      | Review doc                         | Write Purpose/Behavior/Verify        | `/writing-design-docs` |
| 6. Verify Branch | Approve doc updates                | Diff branch, flag stale docs         | `/verify-branch`       |
| 7. Create PR     | Review and confirm                 | Generate description, push           | `/create-pr`           |
