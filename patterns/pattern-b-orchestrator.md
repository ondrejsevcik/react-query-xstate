# Pattern B: Machine as Orchestrator

**XState drives fetching via `queryClient` imperative API. The machine decides WHEN to fetch; React Query provides caching and deduplication.**

## When to Use

- The machine must control exactly when a fetch happens (not just "when mounted")
- An API response determines which state the machine transitions to
- You need explicit retry/error-recovery flows beyond React Query's built-in retries

## The Key Mechanism: `machine.provide()`

The machine declares actor slots. At the React call site, `machine.provide()` injects implementations that use `queryClient`. This keeps the machine framework-agnostic and testable.

## Example: User Profile Loading with Role-Based Routing

The API response determines what the user sees next — admin goes to dashboard, regular user goes to onboarding if incomplete, otherwise to home.

### Machine Definition (framework-agnostic)

```tsx
// machines/auth-flow.ts
import { setup, assign, fromPromise } from 'xstate'

type User = {
  id: string
  name: string
  role: 'admin' | 'user'
  onboardingComplete: boolean
}

export const authFlowMachine = setup({
  types: {
    context: {} as {
      userId: string | null
      error: string | null
    },
    events: {} as
      | { type: 'LOGIN_SUCCESS'; userId: string }
      | { type: 'LOGOUT' }
      | { type: 'RETRY' },
  },
  actors: {
    // Slot: will be injected with queryClient-backed implementation
    loadUserProfile: fromPromise<User, { userId: string }>(async () => {
      throw new Error('loadUserProfile must be provided')
    }),
  },
}).createMachine({
  id: 'authFlow',
  initial: 'unauthenticated',
  context: {
    userId: null,
    error: null,
  },
  states: {
    unauthenticated: {
      on: {
        LOGIN_SUCCESS: {
          target: 'loadingProfile',
          actions: assign({ userId: ({ event }) => event.userId }),
        },
      },
    },

    loadingProfile: {
      invoke: {
        src: 'loadUserProfile',
        input: ({ context }) => ({ userId: context.userId! }),
        onDone: [
          // Inline guards on onDone — XState types event.output as User
          {
            guard: ({ event }) => event.output.role === 'admin',
            target: 'adminDashboard',
          },
          {
            guard: ({ event }) =>
              event.output.role === 'user' && !event.output.onboardingComplete,
            target: 'onboarding',
          },
          { target: 'home' },
        ],
        onError: {
          target: 'profileError',
          actions: assign({
            error: ({ event }) => event.error.message,
          }),
        },
      },
    },

    profileError: {
      on: {
        RETRY: 'loadingProfile',
        LOGOUT: 'unauthenticated',
      },
    },

    adminDashboard: {
      on: { LOGOUT: 'unauthenticated' },
    },

    onboarding: {
      on: { LOGOUT: 'unauthenticated' },
    },

    home: {
      on: { LOGOUT: 'unauthenticated' },
    },
  },
})
```

### React Component (wires machine to queryClient)

```tsx
// components/AuthFlow.tsx
import { useMachine } from '@xstate/react'
import { useQueryClient } from '@tanstack/react-query'
import { fromPromise } from 'xstate'
import { authFlowMachine } from '../machines/auth-flow'
import { fetchUserProfile } from '../api/users'

export function AuthFlow() {
  const queryClient = useQueryClient()

  const [snapshot, send] = useMachine(
    authFlowMachine.provide({
      actors: {
        loadUserProfile: fromPromise(async ({ input }) => {
          // ensureQueryData = return cached if fresh, fetch if stale
          // This populates the React Query cache, so other components
          // using useQuery(['user', id]) get the data for free
          return queryClient.ensureQueryData({
            queryKey: ['user', input.userId],
            queryFn: () => fetchUserProfile(input.userId),
            staleTime: 5 * 60 * 1000,
          })
        }),
      },
    })
  )

  return (
    <>
      {snapshot.matches('unauthenticated') && <LoginForm onSuccess={(userId) => send({ type: 'LOGIN_SUCCESS', userId })} />}
      {snapshot.matches('loadingProfile') && <FullPageSpinner />}
      {snapshot.matches('profileError') && (
        <ErrorScreen
          message={snapshot.context.error}
          onRetry={() => send({ type: 'RETRY' })}
          onLogout={() => send({ type: 'LOGOUT' })}
        />
      )}
      {snapshot.matches('adminDashboard') && <AdminDashboard />}
      {snapshot.matches('onboarding') && <OnboardingWizard />}
      {snapshot.matches('home') && <HomePage />}
    </>
  )
}
```

### Testing (no React needed)

```tsx
// machines/__tests__/auth-flow.test.ts
import { createActor, fromPromise, waitFor } from 'xstate'
import { authFlowMachine } from '../auth-flow'

function createTestActor(userOverrides: Partial<User> = {}) {
  return createActor(
    authFlowMachine.provide({
      actors: {
        loadUserProfile: fromPromise(async () => ({
          id: '1',
          name: 'Test',
          role: 'user' as const,
          onboardingComplete: true,
          ...userOverrides,
        })),
      },
    })
  )
}

test('admin goes to admin dashboard', async () => {
  const actor = createTestActor({ role: 'admin' })
  actor.start()
  actor.send({ type: 'LOGIN_SUCCESS', userId: '1' })
  const snapshot = await waitFor(actor, (s) => s.matches('adminDashboard'))
  expect(snapshot.value).toBe('adminDashboard')
})

test('new user goes to onboarding', async () => {
  const actor = createTestActor({ role: 'user', onboardingComplete: false })
  actor.start()
  actor.send({ type: 'LOGIN_SUCCESS', userId: '1' })
  const snapshot = await waitFor(actor, (s) => s.matches('onboarding'))
  expect(snapshot.value).toBe('onboarding')
})

test('error leads to retry flow', async () => {
  const actor = createActor(
    authFlowMachine.provide({
      actors: {
        loadUserProfile: fromPromise(async () => {
          throw new Error('Network error')
        }),
      },
    })
  )
  actor.start()
  actor.send({ type: 'LOGIN_SUCCESS', userId: '1' })
  const snapshot = await waitFor(actor, (s) => s.matches('profileError'))
  expect(snapshot.value).toBe('profileError')
  expect(snapshot.context.error).toBe('Network error')
})
```

## `ensureQueryData` vs `fetchQuery`

| Method | Behavior |
|--------|----------|
| `ensureQueryData` | Returns cached data if fresh (respects `staleTime`), fetches only if stale or missing |
| `fetchQuery` | Always fetches, even if cache has fresh data |

**Prefer `ensureQueryData`** — it respects the cache, which is the whole point of using React Query. Use `fetchQuery` only when you explicitly need fresh data regardless of cache state.

## Trade-offs

**Gains:**
- Machine is fully testable without React or query client
- Machine controls the flow — explicit states, guarded transitions
- Query cache is populated as a side effect — other components benefit

**Costs:**
- You lose per-observer React Query features (`select`, `placeholderData`, `refetchInterval`)
- Components using `useQuery` alongside the machine need to coordinate
- `ensureQueryData` doesn't create a live subscription — background refetches won't update the machine
