---
description: Project-specific component folders, UI library, and translation patterns
globs: ["src/components/**", "src/**/*.tsx"]
---

# Component Organization

## Folder Conventions

| Folder      | Purpose                              | Example                |
| ----------- | ------------------------------------ | ---------------------- |
| `_ui/`      | Design system primitives             | Button, Input, Dialog  |
| `_forms/`   | Form components (react-hook-form)    | LoginForm, AddressForm |
| `_headers/` | Navigation and header components     | MainNav, MobileMenu    |
| Feature folders | Domain-specific components       | checkout/, cart/        |

## Available UI Components

Import from `@/components/_ui/`:
- `Button`, `Input`, `Label`, `Dialog`, `Alert`, `Collapsible`
- `InputOTP`, `InputOTPGroup`, `InputOTPSlot` (from `@/components/_ui/InputOtp`)
- `ErrorAlert` (from `@/components/_ui/ErrorAlert`)

## Translations

Always use `useTranslate()` hook for user-facing text:

```
const translate = useTranslate()
translate('page.title')
```

Access locale info: `const { locale, alpha2 } = translate`

## Analytics

Use `track` from `@/lib/hooks/mixpanel` for custom events.

## Page Structure

Pages live under `pages/` (Pages Router):
- `pages/[alpha3]/` - Dynamic country routes
- `pages/api/` - API routes
- `pages/reset/` - Application reset
- `pages/unsupported/` - Unsupported market
