# TypeScript / Next.js Coding Rules

## TypeScript

- Always enable `strict: true` in tsconfig. Never disable `strictNullChecks`.
- Never use `any`. Use `unknown` + type narrowing, or define a proper interface.
- Never use non-null assertion `!` without a comment explaining why it's safe.
- Always define return types for exported or non-trivial functions.
- Use `type` for unions/intersections; use `interface` for extendable object shapes.
- Use `satisfies` to validate object literals against a type without widening.
- Never cast API responses with `as SomeType` — always parse with Zod or equivalent.
- Prefer `const` over `let`; never use `var`.
- Prefer named exports over default exports (except Next.js pages/layouts).
- Avoid barrel files (`index.ts` re-exports) in large codebases — they hurt tree-shaking.

## Naming

| Construct        | Convention           | Example                |
|------------------|----------------------|------------------------|
| Component        | PascalCase           | `UserProfileCard`      |
| Hook             | camelCase + `use`    | `useAuthSession`       |
| Utility fn       | camelCase            | `formatCurrency`       |
| Constant         | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT`      |
| Type / Interface | PascalCase           | `UserProfile`          |
| Enum             | PascalCase members   | `Direction.Up`         |
| File (component) | PascalCase           | `UserProfileCard.tsx`  |
| File (util/hook) | camelCase            | `useAuthSession.ts`    |

## Functions & Logic

- Prefer pure functions — no side effects unless necessary.
- Keep functions small and single-purpose (< 40 lines as a guide).
- Use early returns to reduce nesting instead of deeply nested if/else.
- Avoid callback hell — use async/await over raw `.then()` chains.
- Never swallow errors silently (`catch (e) {}`). Always log or rethrow.

## React Components

- One component per file.
- Define a `Props` type above every component; destructure inline.
- Avoid inline object/array literals in JSX props — they cause unnecessary re-renders.
- Never mutate state directly — always return a new reference.
- Avoid `useEffect` unless truly needed for syncing with an external system.
- Split large components early — if it's > 150 lines, consider breaking it up.

```tsx
// ✅ Good
type Props = { userId: string; onSuccess: (user: User) => void };

export function UserCard({ userId, onSuccess }: Props) {
  if (!userId) return null;
  // ...
}

// ❌ Bad
export default function(props: any) { ... }


Next.js
	∙	Default to Server Components. Add "use client" only when you need browser APIs, event handlers, or hooks.
	∙	Never useEffect for initial data fetching — fetch on the server instead.
	∙	Always add <Suspense> boundaries around async Server Components.
	∙	Always add error.tsx to every route segment.
	∙	Use next/image instead of <img> — always provide width, height, and alt.
	∙	Use next/link instead of <a> for internal navigation.
	∙	Never store secrets in NEXT_PUBLIC_ env vars — those are exposed to the client.
	∙	Colocate route-specific components inside the app/ segment, not in components/.
Data Fetching & Validation
	∙	Always validate external data at the boundary (API response, form input, URL params) with Zod.
	∙	Never trust JSON.parse() output without schema validation.
	∙	Handle all three states for async data: loading, error, success.

// ✅ Good
const result = UserSchema.safeParse(apiResponse);
if (!result.success) throw new Error('Invalid user data');

// ❌ Bad
const user = apiResponse as User;


Styling
	∙	Use Tailwind utility classes as the default styling approach.
	∙	Avoid inline style={{}} except for truly dynamic values (e.g. CSS variables).
	∙	Never mix Tailwind with global CSS on the same element.
	∙	Use cn() (clsx + tailwind-merge) to conditionally combine class names.
Testing
	∙	Test behavior, not implementation. Never test internal state directly.
	∙	Testing Library query priority: getByRole > getByLabelText > getByText > getByTestId.
	∙	Every bug fix must include a regression test.
	∙	Keep unit tests next to source: Button.tsx → Button.test.tsx.
	∙	Use Vitest for unit/component tests, Playwright for e2e.



# Testing Rules — TypeScript / Next.js

## Core Philosophy

- Test **behavior**, not implementation. Users don't care about internal state.
- Tests are first-class code — apply the same quality standards as production code.
- A test that never fails is worthless. A test that always fails is noise. Aim for tests that catch real regressions.
- Every bug fix **must** ship with a regression test.

## Stack

- **Unit / Component** → Vitest + Testing Library
- **E2E** → Playwright
- **API / Server** → Vitest + supertest (or Next.js route handlers with MSW)
- **Mocking** → MSW (network), `vi.mock()` (modules), `vi.fn()` (functions)

## What to Test

| Layer | What to cover |
|---|---|
| `src/lib/` utils | All exported functions, edge cases, error paths |
| Custom hooks | State transitions, side effects, error states |
| Components | User interactions, conditional rendering, accessibility |
| API routes | Happy path, validation errors, auth errors |
| E2E | Critical user flows only (auth, checkout, key forms) |

**Don't test:**
- Third-party libraries (trust them to test themselves)
- Implementation details (internal state, private methods)
- Styling (use visual regression tools like Percy if needed)
- Simple pass-through wrappers

## File Structure



Unit / component tests — colocate with source
src/components/UserCard/
├── UserCard.tsx
└── UserCard.test.tsx
src/lib/
├── formatCurrency.ts
└── formatCurrency.test.ts
E2E tests — separate directory
e2e/
├── auth.spec.ts
└── checkout.spec.ts


## Naming

```ts
// Pattern: describe WHAT it does, not HOW
describe('UserCard', () => {
  it('renders the user name', () => { … })
  it('calls onSuccess after form submission', () => { … })
  it('shows an error message when userId is invalid', () => { … })
  it('disables the submit button while loading', () => { … })
})

// ❌ Bad names
it('test 1', …)
it('works correctly', …)
it('should render', …)


Query Priority (Testing Library)
Always use the most semantic query available:

1. getByRole          — best, mirrors how assistive tech sees the UI
2. getByLabelText     — for form inputs
3. getByPlaceholderText
4. getByText          — for non-interactive visible text
5. getByDisplayValue  — for current value of input/select
6. getByAltText       — for images
7. getByTitle
8. getByTestId        — last resort only, add data-testid sparingly


// ✅ Good
screen.getByRole('button', { name: /submit/i })
screen.getByLabelText('Email address')

// ❌ Bad
screen.getByTestId('submit-btn')
container.querySelector('.submit-button')


Async

// ✅ Always await async assertions
await screen.findByText('Welcome back')
await waitFor(() => expect(mockFn).toHaveBeenCalledTimes(1))

// ❌ Never assert on unresolved async state
expect(screen.getByText('Welcome back')).toBeInTheDocument() // may be flaky


Mocking

// ✅ Mock at the network layer with MSW — not inside components
// handlers.ts
http.get('/api/user/:id', () => HttpResponse.json({ id: '1', name: 'Alice' }))

// ✅ Mock modules when MSW isn't enough
vi.mock('@/lib/analytics', () => ({ track: vi.fn() }))

// ❌ Never mock what you're testing
// ❌ Never mock internal implementation details


Component Tests

// ✅ Good — renders, interacts, asserts on outcome
it('submits the form with user input', async () => {
  const onSubmit = vi.fn()
  render(<LoginForm onSubmit={onSubmit} />)

  await userEvent.type(screen.getByLabelText('Email'), 'alice@example.com')
  await userEvent.type(screen.getByLabelText('Password'), 'secret123')
  await userEvent.click(screen.getByRole('button', { name: /log in/i }))

  expect(onSubmit).toHaveBeenCalledWith({
    email: 'alice@example.com',
    password: 'secret123',
  })
})

// ❌ Bad — tests implementation, not behavior
it('sets isLoading to true on submit', () => {
  const { result } = renderHook(() => useLoginForm())
  act(() => result.current.submit())
  expect(result.current.isLoading).toBe(true)
})


Hook Tests

// ✅ Use renderHook + act for state transitions
it('increments count on each call', () => {
  const { result } = renderHook(() => useCounter())
  act(() => result.current.increment())
  expect(result.current.count).toBe(1)
})


E2E (Playwright)

// ✅ Good — full user flow, real assertions
test('user can log in and see dashboard', async ({ page }) => {
  await page.goto('/login')
  await page.getByLabel('Email').fill('alice@example.com')
  await page.getByLabel('Password').fill('secret123')
  await page.getByRole('button', { name: /log in/i }).click()
  await expect(page).toHaveURL('/dashboard')
  await expect(page.getByRole('heading', { name: 'Welcome, Alice' })).toBeVisible()
})

// ✅ Use page object model for flows used across multiple tests
// e2e/pages/LoginPage.ts
export class LoginPage {
  async login(email: string, password: string) { … }
}


Coverage
	∙	Aim for meaningful coverage, not 100% line coverage.
	∙	Focus on: critical paths, error boundaries, edge cases.
	∙	Ignore: generated files, config files, type-only files.
	∙	Never add tests purely to hit a coverage number.
Anti-patterns

// ❌ Snapshot testing large trees — brittle, hard to review
expect(container).toMatchSnapshot()

// ❌ Testing third-party behavior
expect(router.push).toHaveBeenCalled() // test your code called it, not that it works

// ❌ Relying on test order
// Tests must be fully independent and runnable in isolation

// ❌ Magic sleeps
await new Promise(r => setTimeout(r, 500)) // use waitFor() instead

// ❌ Asserting on exact classnames
expect(el).toHaveClass('text-red-500') // test the behavior that class produces
