---
description: Project-specific test structure, commands, and conventions
globs: ["tests/**", "**/*.test.ts", "**/*.spec.ts"]
---

# Testing Conventions

## Folder Structure

```
tests/
├── _tests/endToEnd/    # Main test suites
├── helpers/            # Test utilities (auth, navigation)
├── mocks/              # Mock data (users, cards, addresses)
├── poms/               # Page Object Models
├── results/            # Test outputs (gitignored)
└── test.config.ts      # Global test setup
```

## Commands

| Command                | Environment |
| ---------------------- | ----------- |
| `npm run test:e2e`     | Local       |
| `npm run test:e2e:stg` | Staging     |
| `npm run test:e2e:prod`| Production  |

## Test Tags

- `@smoke` - Critical path tests (run on every PR)
- `@regression` - Full test suite (run nightly)

## Mock Data Location

- Users: `tests/mocks/users.ts`
- Cards: `tests/mocks/cards.ts`
- Addresses: `tests/mocks/addresses.ts`

## Helper Utilities

- `tests/helpers/auth.ts` - `loginAs()`, `logout()`
- Common flows should be in `tests/helpers/`

## POM Convention

One POM per page/major component in `tests/poms/`. POMs encapsulate selectors and actions.
