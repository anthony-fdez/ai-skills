# Verify Branch

Review the current branch's changes against design docs. Find affected features, suggest doc updates, and verify behavior.

Do not make changes without asking. Present findings and ask for approval at each step.

## Step 1: Understand the branch

Run in parallel:

```bash
git diff main...HEAD --stat
git log main..HEAD --oneline
```

If there are no commits above main, tell the user and stop.

Summarize what the branch does in 2-3 sentences.

## Step 2: Find affected design docs

Search for design docs in common locations:

- `docs/systems/*.md`
- `docs/features/**/*.md`
- `docs/*.md`
- `apps/*/docs/*.md`
- Any `*.md` files containing sections like `## Behavior`, `## Purpose`, or `## Verify`

For each doc, check if the branch's changes touch code related to its features. Present a table:

| Design Doc | Affected Features | Why                        |
| ---------- | ----------------- | -------------------------- |
| doc path   | feature names     | which changed files relate |

If no design docs are affected, tell the user and stop.

## Step 3: Suggest doc updates

For each affected feature, read its Purpose/Behavior/Verify sections and compare against the code changes.

Flag if the branch:

- **Changes existing behavior** — a behavior statement is no longer accurate
- **Adds new behavior** — not captured in the doc
- **Changes verify steps** — routes moved, API shape changed, etc.

Present suggested changes and use AskUserQuestion:

- **"Update docs"**: Apply the suggested doc changes
- **"Skip"**: Move on without updating

## Step 4: Build verification plan

For each affected feature, extract its Verify steps from the design doc. Build a plan:

| Feature      | Verify Step        | Method          | Why    |
| ------------ | ------------------ | --------------- | ------ |
| feature name | verify description | Preview / Local | reason |

**Method rules:**

- DB migration changes (migration files, schema changes) -> Local
- Check for a preview URL: run `gh pr view --json url` then look for preview deployment URLs in PR comments or checks
- Preview URL exists -> Preview
- No preview URL -> Local

Present the plan and use AskUserQuestion:

- **"Run verification"**: Execute the plan
- **"Done"**: User will verify manually

## Step 5: Run verification

Execute each verify step from the plan.

**For Preview verification:**

- Navigate to the preview URL in the browser
- Follow the verify steps from the design doc

**For Local verification:**

- Check if the dev server is running, start if needed
- Run migrations if applicable
- Follow the verify steps from the design doc
- For API verifications, use `curl` or the browser

**Capture evidence for every step:**

- **Page verifications**: Take a screenshot
- **API verifications**: Capture request and response (method, URL, status code, body)
- **DB verifications**: Capture query and result rows

## Step 6: Report

Present a summary with evidence:

| Feature      | Verify Step        | Result    | Evidence                             |
| ------------ | ------------------ | --------- | ------------------------------------ |
| feature name | verify description | Pass/Fail | screenshot / response / query result |

If any steps fail, use AskUserQuestion:

- **"Investigate"**: Dig into the failure
- **"Done"**: User will handle it
