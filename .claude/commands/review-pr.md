# Review PR

Review a pull request: summarize changes, flag potential issues, and check test coverage gaps.

## Step 1: Find the PR

The user may provide a PR number or URL. If not, check the current branch:

```bash
gh pr view --json number,url,title,baseRefName,headRefName
```

If no PR exists for the current branch, tell the user and stop.

## Step 2: Gather context

Run in parallel:

```bash
gh pr view <number> --json title,body,baseRefName,headRefName,files,additions,deletions
gh pr diff <number>
```

## Step 3: Analyze and generate review

Produce a structured review with these sections:

### Summary

3-5 sentences on what the PR does and why. Group changes by area (e.g., "API changes", "UI updates", "config").

### Issues

Scan the diff for:

- **Security**: hardcoded secrets, injection vectors, unvalidated input
- **Performance**: N+1 queries, missing indexes, unbounded loops, large payloads without pagination
- **Breaking changes**: changed API contracts, renamed exports, modified DB schemas without migrations
- **Error handling**: missing try/catch on async ops, swallowed errors, missing error states in UI
- **Race conditions**: shared mutable state, missing locks, unguarded concurrent access
- **Code smells**: duplicated logic, functions >50 lines, deeply nested conditionals

For each issue, reference the specific file and line range. Assign severity: critical, warning, or info.

If no issues are found, say so. Do not manufacture problems.

### Test coverage

Compare changed files against test files:

1. For each changed source file, look for a corresponding test file (`foo.test.ts`, `foo.spec.ts`)
2. If a test file exists, check whether the diff touches logic that existing tests cover
3. Flag files with changed logic but no test updates

Present as a table:

| File | Has Tests | Tests Updated | Gap |
|------|-----------|---------------|-----|

### Verdict

One of: "Looks good", "Minor issues", "Needs changes"

## Step 4: Confirm action

Use AskUserQuestion with options:

- **"Post as comment"**: Post the review as a PR comment via `gh pr comment <number> --body`
- **"Done"**: Show the review without posting
