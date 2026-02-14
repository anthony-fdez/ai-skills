---
name: writing-forms
description: Enforces form patterns using react-hook-form and Zod validation. Use when creating forms, handling validation, or managing form state.
---

# Writing Forms

Patterns for implementing forms with react-hook-form and Zod validation.

## Standard Form Pattern

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import * as z from 'zod'

// 1. Define schema
const formSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
})

// 2. Infer type from schema
type FormData = z.infer<typeof formSchema>

// 3. Create form component
const LoginForm = () => {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    setError,
    reset,
  } = useForm<FormData>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      email: '',
      password: '',
    },
  })

  const onSubmit = async (data: FormData) => {
    const response = await loginUser(data)

    if (response.error) {
      setError('root', { message: response.message })
      return
    }

    reset()
    router.push('/dashboard')
  }

  return <form onSubmit={handleSubmit(onSubmit)}>{/* Form fields */}</form>
}
```

## Field Component Pattern

```tsx
// ✅ Good: Complete field with label, input, and error
<div className="space-y-2">
  <Label htmlFor="email">Email</Label>
  <Input
    id="email"
    type="email"
    data-test="email-input"
    {...register('email')}
    aria-invalid={!!errors.email}
    aria-describedby={errors.email ? 'email-error' : undefined}
  />
  {errors.email && (
    <p id="email-error" className="text-sm text-destructive">
      {translate(errors.email.message)}
    </p>
  )}
</div>
```

## Validation Patterns

### Common Validators

```typescript
// Email
z.string().email('Invalid email')

// Phone (E.164 format)
z.string().regex(/^\+?[1-9]\d{1,14}$/, 'Invalid phone number')

// Required field
z.string().min(1, 'This field is required')

// Optional field
z.string().optional()

// Optional but validated when present
z.string()
  .optional()
  .refine(
    (val) => !val || val.length >= 3,
    'Must be at least 3 characters if provided',
  )

// Password with requirements
z.string()
  .min(8, 'Password must be at least 8 characters')
  .regex(/[A-Z]/, 'Password must contain an uppercase letter')
  .regex(/[0-9]/, 'Password must contain a number')

// Confirm password
z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],
})

// Custom validation
z.string().refine((val) => validateCustomLogic(val), 'Custom error message')
```

## Error Handling

### Field-Level Errors

```tsx
{
  errors.fieldName && (
    <p className="text-sm text-destructive">
      {translate(errors.fieldName.message)}
    </p>
  )
}
```

### Form-Level Errors

```tsx
// Set form-level error from API response
setError('root', {
  message: response.message,
})

// Display form-level error
{
  errors.root && (
    <Alert variant="destructive">
      <AlertDescription>{errors.root.message}</AlertDescription>
    </Alert>
  )
}
```

### API Error Handling

```tsx
const onSubmit = async (data: FormData) => {
  try {
    const response = await apiCall(data)

    if (response.error) {
      setError('root', { message: response.message })
      return
    }

    router.push('/success')
  } catch (error) {
    setError('root', { message: 'Something went wrong. Please try again.' })
  }
}
```

## Loading States

```tsx
<Button type="submit" disabled={isSubmitting}>
  {isSubmitting ? (
    <>
      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
      Processing...
    </>
  ) : (
    'Submit'
  )}
</Button>
```

## Multi-Step Forms

```tsx
const MultiStepForm = () => {
  const [step, setStep] = useState(1)

  const {
    register,
    handleSubmit,
    trigger,
    formState: { errors },
  } = useForm<FormData>({
    resolver: zodResolver(formSchema),
  })

  const nextStep = async () => {
    // Validate only current step fields
    const fieldsToValidate =
      step === 1 ? ['email', 'name'] : ['address', 'city']

    const isValid = await trigger(fieldsToValidate)
    if (isValid) setStep(step + 1)
  }

  const prevStep = () => setStep(step - 1)

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {step === 1 && <Step1Fields register={register} errors={errors} />}
      {step === 2 && <Step2Fields register={register} errors={errors} />}
      {step === 3 && <Step3Fields register={register} errors={errors} />}

      <div className="flex gap-4">
        {step > 1 && (
          <Button type="button" variant="outline" onClick={prevStep}>
            Back
          </Button>
        )}
        {step < 3 ? (
          <Button type="button" onClick={nextStep}>
            Next
          </Button>
        ) : (
          <Button type="submit">Submit</Button>
        )}
      </div>
    </form>
  )
}
```

## Special Input Components

### OTP Input

```tsx
import {
  InputOTP,
  InputOTPGroup,
  InputOTPSlot,
} from '@/components/_ui/InputOtp'
;<InputOTP maxLength={6} value={otp} onChange={setOtp} data-test="otp-input">
  <InputOTPGroup>
    {[...Array(6)].map((_, i) => (
      <InputOTPSlot key={i} index={i} />
    ))}
  </InputOTPGroup>
</InputOTP>
```

### Phone Input

```tsx
import PhoneInput from 'react-phone-number-input'
;<PhoneInput
  international
  defaultCountry="US"
  value={phone}
  onChange={setPhone}
  data-test="phone-input"
/>
```

## Controlled vs Uncontrolled

### Prefer Uncontrolled (register)

```tsx
// ✅ Good: Uncontrolled with register (better performance)
<Input {...register('email')} />
```

### Use Controller for Custom Components

```tsx
// When component doesn't support ref forwarding
import { Controller } from 'react-hook-form'
;<Controller
  name="phone"
  control={control}
  render={({ field }) => (
    <PhoneInput {...field} international defaultCountry="US" />
  )}
/>
```

## Form Reset

```tsx
// Reset to default values
reset()

// Reset to specific values
reset({
  email: 'new@example.com',
  password: '',
})

// Reset after successful submission
const onSubmit = async (data: FormData) => {
  const result = await submitForm(data)
  if (result.success) {
    reset()
    setIsSuccess(true)
  }
}
```

## Accessibility

### Required Patterns

```tsx
// Proper label association
<Label htmlFor="email">Email</Label>
<Input id="email" {...register('email')} />

// Error message association
<Input
  id="email"
  aria-invalid={!!errors.email}
  aria-describedby={errors.email ? 'email-error' : undefined}
/>
{errors.email && (
  <p id="email-error" role="alert">{errors.email.message}</p>
)}

// Loading announcement
<Button type="submit" aria-busy={isSubmitting}>
  {isSubmitting ? 'Submitting...' : 'Submit'}
</Button>
```

### Focus Management in Multi-Step

```tsx
const stepRef = useRef<HTMLDivElement>(null)

useEffect(() => {
  // Focus first input when step changes
  const firstInput = stepRef.current?.querySelector('input')
  firstInput?.focus()
}, [step])

return <div ref={stepRef}>{step === 1 && <Step1 />}</div>
```

## Testing Forms

```tsx
// Add data-test attributes
<Input data-test="email-input" {...register('email')} />
<Button data-test="submit-button" type="submit">Submit</Button>

// Playwright test
await page.fill('[data-test="email-input"]', 'test@example.com')
await page.fill('[data-test="password-input"]', 'password123')
await page.click('[data-test="submit-button"]')
await expect(page).toHaveURL('/dashboard')
```

## Checklist

- [ ] Form uses react-hook-form with zodResolver
- [ ] Schema defined with Zod, type inferred with z.infer
- [ ] Default values provided to useForm
- [ ] Fields have proper labels with htmlFor
- [ ] Error messages displayed for each field
- [ ] Form-level errors handled with setError('root', ...)
- [ ] Loading state shown during submission (isSubmitting)
- [ ] Submit button disabled while submitting
- [ ] data-test attributes added for E2E testing
- [ ] aria-invalid set on fields with errors
- [ ] aria-describedby links inputs to error messages
