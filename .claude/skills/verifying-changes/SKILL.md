---
name: verifying-changes
description: Self-verification loop for validating code changes through code quality checks and Chrome MCP browser verification. Use when implementing UI changes, fixing bugs, or completing features that need visual or functional verification.
---

# Verifying Changes

Build-Verify-Fix loop using code quality tools and Chrome MCP browser automation.

## Contents

- [Run Code Quality Checks First](#run-code-quality-checks-first)
- [Use Chrome MCP for Visual Verification](#use-chrome-mcp-for-visual-verification)
- [Verify by Change Type](#verify-by-change-type)
- [Check Console and Network for Silent Failures](#check-console-and-network-for-silent-failures)
- [Track Verification for Multi-Step Features](#track-verification-for-multi-step-features)
- [Avoid Claiming Without Observing](#avoid-claiming-without-observing)

---

## Run Code Quality Checks First

Run the project's lint/format command before any other verification. Nothing else matters if the code doesn't pass static checks.

```bash
npm run formatAndLint
```

If this fails: fix the issue, re-run. Do not proceed to browser verification until this passes.

Why: Catching type errors and lint violations early prevents wasting time debugging visual issues caused by broken code.

---

## Use Chrome MCP for Visual Verification

**Startup sequence**: `tabs_context_mcp` → `tabs_create_mcp` → navigate to the relevant page on the dev server.

**After every interaction** (click, form input, navigation), re-read the page to confirm the DOM updated as expected.

**Tools to use**:

| Tool                 | Purpose                                      |
| -------------------- | -------------------------------------------- |
| `screenshot`         | Capture visual state                         |
| `read_page` / `find` | Confirm elements exist with expected content |
| `left_click`         | Test clickable elements                      |
| `form_input`         | Test form fields accept input                |
| `get_page_text`      | Verify rendered text content                 |

Why: Screenshots and DOM reads are the only way to confirm what the user will actually see. Code review alone misses rendering bugs, z-index issues, and styling regressions.

---

## Verify by Change Type

Match verification depth to the type of change:

| Change type        | Minimum verification                                  |
| ------------------ | ----------------------------------------------------- |
| Logic-only (no UI) | Code quality checks only                              |
| UI change          | Code quality + Chrome MCP visual check                |
| Form/interactive   | Code quality + Chrome MCP interaction + console check |
| API route          | Code quality + test the endpoint + console check      |
| Full feature       | All verification steps                                |

**What to verify per change type**:

| Change type       | What to verify                                                        |
| ----------------- | --------------------------------------------------------------------- |
| UI component      | Element renders, correct text/styling, responsive states              |
| Form              | Fields accept input, validation fires, submission works               |
| Navigation        | Links navigate correctly, URL updates, back button works              |
| Data display      | API data renders, loading states show, empty states handled           |
| Interactive state | Clicks/hovers trigger expected behavior, state updates reflect in DOM |

---

## Check Console and Network for Silent Failures

After visual verification, check for runtime errors and failed requests.

**Console**: `read_console_messages` with `pattern: "error|Error|ERR|fail"`

Ignore HMR/DevTools noise. Focus on:

- Unhandled promise rejections
- Failed API calls
- Render errors
- Hydration mismatches

**Network**: `read_network_requests` with `urlPattern: "/api/"`

Verify:

- No 4xx/5xx responses
- No missing expected calls
- No unexpected duplicate requests

On finding errors: fix the root cause, re-run verification from the code quality step.

Why: Many bugs are invisible in the UI — a component can render correctly while silently failing to persist data or triggering console errors that degrade performance.

---

## Track Verification for Multi-Step Features

For features with multiple acceptance criteria, use task tracking (`TaskCreate`/`TaskUpdate`) with one task per criterion.

**Criteria must be concrete and testable**:

- "Cart icon shows updated count after adding item"
- "Form shows validation error when email is empty"
- "Page redirects to /confirmation after successful checkout"

**Not vague**: ~~"Cart works correctly"~~ ~~"Form validation is good"~~

Mark `completed` only after all verification steps pass for that criterion.

Why: Vague criteria lead to "it looks fine" verification. Concrete criteria force you to actually test each behavior.

---

## Avoid Claiming Without Observing

Never say "this should work." Run it, verify it, report what was observed.

**Build-Verify-Fix loop**:

1. Implement the change
2. Run code quality checks — must pass
3. Verify in browser (if applicable) — must match expectations
4. Check console/network — must be clean
5. If any step fails: fix the root cause, re-run from that step

**After 3 failed attempts on the same step**, stop and report to the user. Do not keep retrying the same approach.

**If verification is not possible** (dev server down, Chrome MCP disconnected), tell the user what to verify manually instead of skipping verification.

---

## Checklist

- [ ] Code quality checks pass before browser verification
- [ ] Chrome MCP used for visual verification (not just code review)
- [ ] Page re-read after every interaction to confirm DOM updates
- [ ] Console checked for runtime errors after visual verification
- [ ] Network checked for failed/missing API requests
- [ ] Multi-step features tracked with concrete, testable criteria
- [ ] No "this should work" claims — only observed results reported
