# Explain System

Deep-dive into a system, module, or area of the codebase. Map dependencies, trace data flow, and identify entry points.

## Step 1: Locate the system

The user provides a path, module name, or keyword (e.g., `src/auth`, `payments`, `the billing system`).

If the input is ambiguous, search for matching directories and files. If multiple matches exist, use AskUserQuestion to let the user pick the right area.

If no matches are found, tell the user and stop.

## Step 2: Map the boundaries

Starting from the target path or matched files:

1. Identify all source files in the module
2. Find entry points — files imported by code outside the module, API routes, page components, CLI commands
3. Find external dependencies — what this module imports from outside itself (other internal modules, third-party packages)

Present a brief structural overview:

```
System: <name>
Entry points: <list>
Files: <count>
External deps: <key internal and external imports>
```

## Step 3: Trace data flow

For the primary entry points, trace how data flows through the system:

1. What triggers execution (HTTP request, user action, cron, event)
2. What transformations happen to the data
3. Where data is persisted or sent (DB, API, queue, file)
4. What gets returned or emitted

Present as a numbered flow for each key path.

## Step 4: Identify patterns and gotchas

Note architectural patterns in use (state management, error handling, caching, validation). Only mention patterns that are actually present.

Flag anything a new developer should know:

- Non-obvious side effects
- Implicit dependencies (env vars, feature flags, external services)
- Known workarounds or tech debt (TODOs, HACKs, workaround comments)
- Files that are surprisingly important or easy to break

## Step 5: Present explanation

Combine into a structured explanation with headers, short paragraphs, and code references (`file:line`). Aim for the level of detail that saves a new team member a full day of reading code.
