# Investigate

Take an error message, log line, or bug description and trace it through the codebase to find the root cause and suggest fixes.

## Step 1: Extract search terms

The user provides an error message, stack trace, log line, error code, or description of unexpected behavior.

Parse the input for searchable identifiers:

- Error class names (e.g., `ValidationError`, `NEXT_NOT_FOUND`)
- Function or method names from stack traces
- File paths and line numbers
- HTTP status codes or error codes
- Specific string literals from error messages
- Variable or field names mentioned in the description

## Step 2: Search the codebase

For each search term, search in parallel:

1. Grep for exact matches of error messages or codes
2. Find where the error is thrown or the log line is emitted
3. Find where the throwing function is called from

Build a call chain from the trigger point back to the origin of the issue.

If no matches are found in the codebase, tell the user the error likely originates from a third-party dependency or external service. Suggest checking package docs or changelogs and stop.

## Step 3: Identify root cause

Trace the call chain and present:

- **Error origin**: The exact file and line where the error is thrown or logged
- **Call chain**: How execution reaches that point (list the key hops)
- **Root cause**: What actually goes wrong and why
- **Trigger conditions**: What input or state reproduces this

## Step 4: Suggest fixes

Propose 1-3 fixes ranked by confidence. For each fix:

1. What to change and where (specific files and lines)
2. Why this fixes the root cause
3. Any risks or side effects

Use AskUserQuestion with options:

- **"Apply fix"**: Apply the top-recommended fix
- **"Show code"**: Show the exact code changes without applying
- **"Done"**: User will handle it themselves
