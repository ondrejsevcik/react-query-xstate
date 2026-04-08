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

function toErrorMessage(error: unknown): string {
  return error instanceof Error ? error.message : String(error)
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
            error: ({ event }) => toErrorMessage(event.error),
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
import { useActorRef, useSelector } from '@xstate/react'
import { useQueryClient } from '@tanstack/react-query'
import { fromPromise } from 'xstate'
import { authFlowMachine } from '../machines/auth-flow'
import { fetchUserProfile } from '../api/users'

export function AuthFlow() {
  const queryClient = useQueryClient()

  const actorRef = useActorRef(
    authFlowMachine.provide({
      actors: {
        loadUserProfile: fromPromise(async ({ input }) => {
          // fetchQuery throws on network error even if stale cache exists,
          // ensuring onError fires so the machine can transition to an error state.
          // This also populates the React Query cache, so other components
          // using useQuery(['user', id]) get the data for free.
          return queryClient.fetchQuery({
            queryKey: ['user', input.userId],
            queryFn: () => fetchUserProfile(input.userId),
            staleTime: 5 * 60 * 1000,
          })
        }),
      },
    })
  )

  const step = useSelector(actorRef, (s) => s.value)
  const error = useSelector(actorRef, (s) => s.context.error)

  return (
    <>
      {step === 'unauthenticated' && <LoginForm onSuccess={(userId) => actorRef.send({ type: 'LOGIN_SUCCESS', userId })} />}
      {step === 'loadingProfile' && <FullPageSpinner />}
      {step === 'profileError' && (
        <ErrorScreen
          message={error}
          onRetry={() => actorRef.send({ type: 'RETRY' })}
          onLogout={() => actorRef.send({ type: 'LOGOUT' })}
        />
      )}
      {step === 'adminDashboard' && <AdminDashboard />}
      {step === 'onboarding' && <OnboardingWizard />}
      {step === 'home' && <HomePage />}
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

Both methods read the cache and respect `staleTime` — neither blindly fetches. The difference is error handling:

| Method | Cache behavior | On network error |
|--------|---------------|-----------------|
| `ensureQueryData` | Returns cached data if fresh, fetches if stale/missing | Returns stale cache if available; only throws if cache is empty **and** fetch fails |
| `fetchQuery` | Returns cached data if fresh, fetches if stale/missing | Always throws, even if stale cache exists |

**Inside invoke actors, prefer `fetchQuery`** — it guarantees that a failed fetch routes to `onError`, so your machine transitions to an error state. With `ensureQueryData`, a network failure might silently return stale data via `onDone`. **Outside invoke actors** (prefetching, cache warming), `ensureQueryData` is the right default.

## Trade-offs

**Gains:**
- Machine is fully testable without React or query client
- Machine controls the flow — explicit states, guarded transitions
- Query cache is populated as a side effect — other components benefit

**Costs:**
- You lose per-observer React Query features (`select`, `placeholderData`, `refetchInterval`)
- Components using `useQuery` alongside the machine need to coordinate
- `ensureQueryData` doesn't create a live subscription — background refetches won't update the machine
