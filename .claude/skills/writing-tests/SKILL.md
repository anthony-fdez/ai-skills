---
name: writing-tests
description: Enforces Playwright testing patterns including selectors, POMs, and best practices. Use when writing E2E tests, creating page objects, or debugging test failures.
---

# Writing Tests (Playwright)

Patterns for Playwright E2E testing including test structure, selectors, and page object models.

## Test Structure

```
tests/
├── _tests/endToEnd/    # Main test suites
├── helpers/            # Test utilities
├── mocks/              # Mock data
├── poms/               # Page Object Models
├── results/            # Test outputs (gitignored)
└── test.config.ts      # Global test setup
```

## Standard Test Pattern (AAA)

```typescript
import { test, expect } from '@playwright/test'

test.describe('Checkout Flow', () => {
  test('should complete purchase successfully', async ({ page }) => {
    // Arrange
    await page.goto('/checkout')
    const checkoutPage = new CheckoutPage(page)

    // Act
    await checkoutPage.fillEmail('test@example.com')
    await checkoutPage.fillPayment(testCards.visa)
    await checkoutPage.submit()

    // Assert
    await expect(page.getByTestId('success-message')).toBeVisible()
    await expect(page).toHaveURL(/\/confirmation/)
  })
})
```

## Selector Strategy (Priority Order)

### 1. data-test Attributes (Preferred)

```typescript
// ✅ Best: Explicit test IDs
await page.getByTestId('email-input').fill('test@example.com')
await page.getByTestId('submit-button').click()

// In component:
<Input data-test="email-input" />
<Button data-test="submit-button">Submit</Button>
```

### 2. Role-Based Selectors

```typescript
// ✅ Good: Semantic and accessible
await page.getByRole('button', { name: 'Submit' }).click()
await page.getByRole('heading', { name: 'Checkout' }).isVisible()
await page.getByRole('textbox', { name: 'Email' }).fill('test@example.com')
```

### 3. Text Content

```typescript
// ✅ OK: When unique and stable
await page.getByText('Sign In').click()
await page.getByLabel('Email address').fill('test@example.com')
```

### 4. CSS Selectors (Last Resort)

```typescript
// ❌ Avoid: Brittle, tied to implementation
await page.locator('.submit-btn').click()
await page.locator('#email').fill('test@example.com')
```

## Page Object Models (POMs)

### Structure

```typescript
// poms/CheckoutPage.ts
import type { Page, Locator } from '@playwright/test'

export class CheckoutPage {
  readonly page: Page
  readonly emailInput: Locator
  readonly submitButton: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByTestId('checkout-email')
    this.submitButton = page.getByTestId('checkout-submit')
  }

  // Actions
  async fillEmail(email: string) {
    await this.emailInput.fill(email)
  }

  async fillShipping(address: ShippingAddress) {
    await this.page.getByTestId('address-line1').fill(address.line1)
    await this.page.getByTestId('address-city').fill(address.city)
    await this.page.getByTestId('address-zip').fill(address.zip)
  }

  async submit() {
    await this.submitButton.click()
  }

  // Assertions
  async expectSuccess() {
    await expect(this.page.getByTestId('success-message')).toBeVisible()
  }

  async expectError(message: string) {
    await expect(this.page.getByTestId('error-message')).toContainText(message)
  }
}
```

### Usage

```typescript
test('checkout flow', async ({ page }) => {
  const checkout = new CheckoutPage(page)

  await page.goto('/checkout')
  await checkout.fillEmail('test@example.com')
  await checkout.fillShipping(testAddresses.us)
  await checkout.submit()
  await checkout.expectSuccess()
})
```

## Test Configuration

### Environment Commands

```bash
# Local
npm run test:e2e

# Staging
npm run test:e2e:stg

# Production
npm run test:e2e:prod
```

### Test Tags

```typescript
// @smoke - Critical path tests (run on every PR)
test.describe('@smoke Checkout', () => {
  test('should complete purchase', async ({ page }) => {})
})

// @regression - Full test suite (run nightly)
test.describe('@regression Product Search', () => {
  test('should filter by category', async ({ page }) => {})
})

// Skip WIP tests
test.describe.skip('Feature in progress', () => {})
```

## Mock Data

### Location: tests/mocks/

```typescript
// mocks/users.ts
export const testUsers = {
  standard: {
    email: 'test@example.com',
    password: 'Test123!',
  },
  premium: {
    email: 'premium@example.com',
    password: 'Premium123!',
  },
}

// mocks/cards.ts
export const testCards = {
  visa: {
    number: '4111111111111111',
    expiry: '12/25',
    cvv: '123',
  },
  declined: {
    number: '4000000000000002',
    expiry: '12/25',
    cvv: '123',
  },
}
```

### Usage

```typescript
import { testUsers } from '@/tests/mocks/users'
import { testCards } from '@/tests/mocks/cards'

test('payment flow', async ({ page }) => {
  await loginAs(page, testUsers.standard)
  await fillPayment(page, testCards.visa)
})
```

## Helper Functions

### tests/helpers/auth.ts

```typescript
export const loginAs = async (page: Page, user: TestUser) => {
  await page.goto('/login')
  await page.getByTestId('email-input').fill(user.email)
  await page.getByTestId('password-input').fill(user.password)
  await page.getByTestId('login-button').click()
  await expect(page).toHaveURL(/\/dashboard/)
}

export const logout = async (page: Page) => {
  await page.getByTestId('user-menu').click()
  await page.getByTestId('logout-button').click()
  await expect(page).toHaveURL(/\/login/)
}
```

## Best Practices

### Independent Tests

```typescript
// ✅ Good: Each test is self-contained
test('should add to cart', async ({ page }) => {
  await page.goto('/products/test-product')
  await page.getByTestId('add-to-cart').click()
  await expect(page.getByTestId('cart-count')).toHaveText('1')
})

// ❌ Bad: Test depends on previous test state
test('should checkout', async ({ page }) => {
  // Assumes cart already has items from previous test
  await page.goto('/checkout')
})
```

### Use Auto-Retry Assertions

```typescript
// ✅ Good: expect() auto-retries until timeout
await expect(page.getByTestId('loading')).not.toBeVisible()
await expect(page.getByTestId('success')).toBeVisible()

// ❌ Bad: Arbitrary waits
await page.waitForTimeout(3000) // Never do this
```

### Parallel Execution

```typescript
// Tests should support parallel runs
// Avoid shared state, use unique test data

test('user A checkout', async ({ page }) => {
  const uniqueEmail = `test-${Date.now()}@example.com`
  // ...
})
```

## Debugging

### Headed Mode

```bash
npx playwright test --headed
```

### Debug Mode (Step Through)

```bash
npx playwright test --debug
```

### Trace Viewer

```bash
# Run with trace
npx playwright test --trace on

# View trace
npx playwright show-trace trace.zip
```

### Slow Down Execution

```typescript
test.use({ launchOptions: { slowMo: 500 } })

test('debug this test', async ({ page }) => {
  // Runs slowly for observation
})
```

### Screenshots on Failure

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
  },
})
```

## Adding Test IDs to Components

```tsx
// Always add data-test to interactive elements
<Input data-test="email-input" {...register('email')} />
<Button data-test="submit-button" type="submit">Submit</Button>

// Add to containers for assertions
<div data-test="success-message">Order confirmed!</div>
<div data-test="error-message">{error.message}</div>

// Add to dynamic content
<div data-test={`product-card-${product.id}`}>
  {product.name}
</div>
```

## Checklist

- [ ] Test uses AAA pattern (Arrange, Act, Assert)
- [ ] Selectors use data-test attributes (not CSS classes)
- [ ] Page Object Model created for reusable page interactions
- [ ] Test is independent (doesn't rely on other tests)
- [ ] Mock data used for test users, cards, addresses
- [ ] Assertions use expect() with auto-retry (no waitForTimeout)
- [ ] Test tagged appropriately (@smoke or @regression)
- [ ] data-test attributes added to new UI elements
- [ ] Helper functions used for common flows (login, navigation)
