---
description: DataDog logging conventions and logger usage
globs: ["src/lib/logger/**", "src/lib/build/**", "src/**/*.ts"]
---

# Logging Conventions (DataDog)

## Logger Import

```
import { logger } from '@/lib/logger'
```

Tags are accessed via `logger.tags` (GROUP, PAGE, SOURCE, OPERATION, RETRY, ACTION, STATUS).

## Message Format

Always start with `[Context]` prefix in Title Case:

```
logger.info('[Product Page] Build completed', { ... })
logger.error('[Apple Pay: Session] Failed to create', { ... })
```

## Available Sources

`page-build`, `cosmic`, `jeeves`, `rosetta`, `hydra`, `client`

## Tag Order

`env` (auto) > `phase` (auto) > `group` > `page` > `source` > `operation` > `retry` > `action` > `status` (always last)

## Key Rules

- `market` param uses Alpha2 country codes (e.g., 'US', 'GB')
- `modeSlug` identifies site variant (e.g., 'ufeelgreat', null for default shop)
- High cardinality data (slugs, IDs, durations) goes in `metadata`, NOT tags
- Include `durationMs` in metadata for timed operations
- Pass error objects to `error` param (logger serializes them)
- Use `createRetryLogger` from `@/lib/build/create-retry-logger` for retry operations

See `writing-logs` skill for complete tag reference and detailed examples.
