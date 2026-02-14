# Create Pull Request

Create a GitHub pull request from the current branch with minimal friction.

## Steps

### 1. Ask for target branch

Use AskUserQuestion to ask the user which branch to target the PR against. Suggest likely candidates based on common branch names (`main`, `master`, `develop`, `release/*`), but always ask — never assume.

### 2. Gather context

Run these commands in parallel to understand the changes (use the target branch from step 1):

```bash
git branch --show-current
git log <target_branch>..HEAD --oneline
git diff <target_branch>...HEAD --stat
git diff <target_branch>...HEAD
```

### 3. Generate PR title and description

**Title** MUST follow this exact pattern:

```
<target_branch> <- <current_branch> (<very brief description>)
```

Examples:

- `main <- feat/add-auth (add user authentication)`
- `develop <- fix/cart-total (fix cart total calculation)`

The description in parentheses should be 3-7 words.

**Body** is a concise GitHub PR description in markdown:

1. **Description section** (< 100 words, or < 120 words if explaining complex changes): Brief explanation of what was implemented/changed and why
2. **Technical Details** (if applicable): Explanation of complicated code changes or architectural decisions
3. **Note section** (if applicable): Any dependencies, requirements, or follow-up tasks needed
4. **Testing Checklist**: 5-7 actionable test items using `- [ ]` checkbox format

Requirements:

- Keep description under 100 words (can extend to 120 words if explaining complex technical changes)
- Only add Technical Details if code changes involve complex patterns, new architecture, or non-obvious implementations
- Make checklist items specific and actionable
- Include both happy path and error/fallback scenario tests
- If external dependencies exist (API keys, translations, etc.), mention them in a **Note**

### 4. Present for review and confirm

Show the full PR title and body to the user, then immediately use AskUserQuestion with these options:

- **"Create PR"**: Proceed to create the PR as shown
- **"Edit"**: User wants to provide changes before creating

If the user selects "Create PR", proceed to step 4 immediately.
If the user selects "Edit" or provides custom feedback, apply their changes and present again.

### 5. Create the PR

Push the branch and create the PR:

```bash
git push -u origin HEAD
gh pr create --base <target_branch> --title "<title>" --body "<body>"
```

Return the PR URL to the user.
