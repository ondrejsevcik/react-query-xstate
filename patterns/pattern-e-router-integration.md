# Pattern E: Multi-Page Flows with React Router

**XState manages a flow that spans multiple routes. A bridge component keeps machine state and URL in sync.**

## When to Use

- Onboarding wizards, checkout flows, or any multi-step process where each step has its own URL
- You want browser back/forward to work (or explicitly want to disable it)
- Deep linking matters (user bookmarks or shares `/onboarding/step-3`)

## The Core Problem

Machine state and URL can get out of sync in three ways:

| Scenario | What happens |
|----------|-------------|
| Machine transitions | URL doesn't update unless you call `navigate()` |
| Browser back button | URL changes but machine stays in previous state |
| Page refresh / deep link | Machine reinitializes to `initial`, but URL says step 3 |

## Architecture: Where to Place the Machine

The machine must live **above the router** so it survives route transitions.

```
<FlowProvider>              ← machine lives here
  <RouterProvider>
    <Layout>
      <NavigationBridge />   ← syncs machine ↔ URL
      <Outlet />             ← route components read machine state
    </Layout>
  </RouterProvider>
</FlowProvider>
```

### Setup with `createActorContext`

```tsx
// context/onboarding-flow.ts
import { createActorContext } from '@xstate/react'
import { onboardingMachine } from '../machines/onboarding'

export const OnboardingFlow = createActorContext(onboardingMachine)
```

```tsx
// main.tsx
import { OnboardingFlow } from './context/onboarding-flow'

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <OnboardingFlow.Provider>
        <RouterProvider router={router} />
      </OnboardingFlow.Provider>
    </QueryClientProvider>
  )
}
```

## Example: Onboarding Flow

### Machine Definition

```tsx
// machines/onboarding.ts
import { setup, assign, fromPromise } from 'xstate'

type OnboardingContext = {
  userId: string | null
  profileData: { name: string; role: string } | null
  workspaceData: { name: string; plan: string } | null
}

export const onboardingMachine = setup({
  types: {
    context: {} as OnboardingContext,
    events: {} as
      | { type: 'START'; userId: string }
      | { type: 'SUBMIT_PROFILE'; name: string; role: string }
      | { type: 'SUBMIT_WORKSPACE'; name: string; plan: string }
      | { type: 'CONFIRM' }
      | { type: 'BACK' }
      // Sync events for router → machine
      | { type: 'SYNC_TO_STEP'; step: string },
  },
  actors: {
    completeOnboarding: fromPromise<void, OnboardingContext>(async () => {
      throw new Error('must be provided')
    }),
  },
}).createMachine({
  id: 'onboarding',
  initial: 'profile',
  context: {
    userId: null,
    profileData: null,
    workspaceData: null,
  },
  states: {
    profile: {
      on: {
        SUBMIT_PROFILE: {
          target: 'workspace',
          actions: assign({
            profileData: ({ event }) => ({ name: event.name, role: event.role }),
          }),
        },
      },
    },

    workspace: {
      on: {
        SUBMIT_WORKSPACE: {
          target: 'review',
          actions: assign({
            workspaceData: ({ event }) => ({ name: event.name, plan: event.plan }),
          }),
        },
        BACK: 'profile',
      },
    },

    review: {
      on: {
        CONFIRM: 'submitting',
        BACK: 'workspace',
      },
    },

    submitting: {
      invoke: {
        src: 'completeOnboarding',
        input: ({ context }) => context,
        onDone: 'complete',
        onError: 'review', // go back to review on failure
      },
    },

    complete: {
      type: 'final',
    },
  },
})
```

### The Navigation Bridge

This is the critical piece — it syncs machine state with the URL in both directions.

```tsx
// components/OnboardingBridge.tsx
import { useEffect, useRef } from 'react'
import { useNavigate, useLocation } from 'react-router'
import { OnboardingFlow } from '../context/onboarding-flow'

// Machine state → URL path
const STATE_TO_PATH: Record<string, string> = {
  profile: '/onboarding/profile',
  workspace: '/onboarding/workspace',
  review: '/onboarding/review',
  submitting: '/onboarding/review', // same URL, just show a spinner
  complete: '/dashboard',
}

// URL path → machine event
const PATH_TO_EVENT: Record<string, string> = {
  '/onboarding/profile': 'profile',
  '/onboarding/workspace': 'workspace',
  '/onboarding/review': 'review',
}

export function OnboardingBridge() {
  const navigate = useNavigate()
  const location = useLocation()
  const actorRef = OnboardingFlow.useActorRef()
  const machineNavigatedRef = useRef(false)

  // Machine → URL: when machine transitions, update the URL
  useEffect(() => {
    const subscription = actorRef.subscribe((snapshot) => {
      const targetPath = STATE_TO_PATH[snapshot.value as string]
      if (targetPath && targetPath !== location.pathname) {
        machineNavigatedRef.current = true
        navigate(targetPath)
      }
    })
    return () => subscription.unsubscribe()
  }, [actorRef, navigate, location.pathname])

  // URL → Machine: when URL changes (back button, deep link), sync machine
  useEffect(() => {
    // Skip if this URL change was caused by the machine
    if (machineNavigatedRef.current) {
      machineNavigatedRef.current = false
      return
    }

    const targetStep = PATH_TO_EVENT[location.pathname]
    if (targetStep) {
      const currentState = actorRef.getSnapshot().value
      if (currentState !== targetStep) {
        actorRef.send({ type: 'SYNC_TO_STEP', step: targetStep })
      }
    }
  }, [location.pathname, actorRef])

  return null
}
```

### Handling `SYNC_TO_STEP` and Conditional Navigation

The machine needs to handle external navigation attempts (browser back/forward, deep links) and decide whether to allow them. This is where guards make navigation rules explicit.

#### The bridge must handle rejected navigations

When the machine rejects a `SYNC_TO_STEP` (no matching guard), the URL is now wrong — it shows a page the machine didn't transition to. The bridge must correct this:

```tsx
// components/OnboardingBridge.tsx (updated)
export function OnboardingBridge() {
  const navigate = useNavigate()
  const location = useLocation()
  const actorRef = OnboardingFlow.useActorRef()
  const machineNavigatedRef = useRef(false)

  // Machine → URL
  useEffect(() => {
    const subscription = actorRef.subscribe((snapshot) => {
      const targetPath = STATE_TO_PATH[snapshot.value as string]
      if (targetPath && targetPath !== location.pathname) {
        machineNavigatedRef.current = true
        navigate(targetPath)
      }
    })
    return () => subscription.unsubscribe()
  }, [actorRef, navigate, location.pathname])

  // URL → Machine (with rejection handling)
  useEffect(() => {
    if (machineNavigatedRef.current) {
      machineNavigatedRef.current = false
      return
    }

    const targetStep = PATH_TO_EVENT[location.pathname]
    if (!targetStep) return

    const currentState = actorRef.getSnapshot().value
    if (currentState === targetStep) return

    // Attempt the sync
    actorRef.send({ type: 'SYNC_TO_STEP', step: targetStep })

    // Check if the machine actually transitioned
    const newState = actorRef.getSnapshot().value
    if (newState !== targetStep) {
      // Machine rejected the navigation — correct the URL back
      const correctPath = STATE_TO_PATH[newState as string]
      if (correctPath) {
        machineNavigatedRef.current = true
        navigate(correctPath, { replace: true })
      }
    }
  }, [location.pathname, actorRef, navigate])

  return null
}
```

Now the machine is the authority. If a user hits browser-back to a step the machine won't allow, the URL snaps back to where the machine actually is.

#### Conditional guards in the machine

Here's where the real power is. Guards can check context to decide what navigation is allowed:

```tsx
export const onboardingMachine = setup({
  types: {
    context: {} as OnboardingContext,
    events: {} as
      | { type: 'SUBMIT_PROFILE'; name: string; role: string }
      | { type: 'SUBMIT_WORKSPACE'; name: string; plan: string }
      | { type: 'CONFIRM' }
      | { type: 'BACK' }
      | { type: 'SYNC_TO_STEP'; step: string },
  },
  guards: {
    // Can only go back to profile if it hasn't been locked by admin review
    profileEditable: ({ context }) => !context.profileLocked,

    // Can only go back to workspace if payment hasn't been initiated
    workspaceEditable: ({ context }) => !context.paymentInitiated,

    // Can skip workspace entirely if user picked a role that doesn't need one
    workspaceRequired: ({ context }) => context.profileData?.role !== 'viewer',

    // Only allow forward navigation to review if all data is present
    canReview: ({ context }) =>
      !!context.profileData && !!context.workspaceData,

    // Target step matches
    toProfile: ({ event }) => (event as any).step === 'profile',
    toWorkspace: ({ event }) => (event as any).step === 'workspace',
    toReview: ({ event }) => (event as any).step === 'review',
  },
}).createMachine({
  id: 'onboarding',
  initial: 'profile',
  context: {
    userId: null,
    profileData: null,
    workspaceData: null,
    profileLocked: false,
    paymentInitiated: false,
  },
  states: {
    profile: {
      on: {
        SUBMIT_PROFILE: [
          {
            // Viewers skip workspace, go straight to review
            guard: { not: 'workspaceRequired' },
            target: 'review',
            actions: assign({
              profileData: ({ event }) => ({ name: event.name, role: event.role }),
            }),
          },
          {
            target: 'workspace',
            actions: assign({
              profileData: ({ event }) => ({ name: event.name, role: event.role }),
            }),
          },
        ],
        // Can't go back from step 1 — no SYNC_TO_STEP handler needed
      },
    },

    workspace: {
      on: {
        SUBMIT_WORKSPACE: {
          target: 'review',
          actions: assign({
            workspaceData: ({ event }) => ({ name: event.name, plan: event.plan }),
          }),
        },
        BACK: {
          guard: 'profileEditable',
          target: 'profile',
        },
        SYNC_TO_STEP: [
          {
            guard: { and: ['toProfile', 'profileEditable'] },
            target: 'profile',
          },
          // Forward to review only if data is complete
          {
            guard: { and: ['toReview', 'canReview'] },
            target: 'review',
          },
        ],
      },
    },

    review: {
      on: {
        CONFIRM: 'submitting',
        BACK: {
          guard: 'workspaceEditable',
          target: 'workspace',
        },
        SYNC_TO_STEP: [
          {
            guard: { and: ['toProfile', 'profileEditable'] },
            target: 'profile',
          },
          {
            guard: { and: ['toWorkspace', 'workspaceEditable'] },
            target: 'workspace',
          },
        ],
      },
    },

    submitting: {
      // No BACK, no SYNC_TO_STEP — can't navigate away while submitting
      invoke: {
        src: 'completeOnboarding',
        input: ({ context }) => context,
        onDone: 'complete',
        onError: 'review',
      },
    },

    complete: {
      type: 'final',
    },
  },
})
```

This gives you fine-grained control:

| Scenario | What the machine does |
|----------|----------------------|
| Back from workspace → profile | Allowed only if `profileEditable` (profile not locked) |
| Back from review → workspace | Allowed only if `workspaceEditable` (payment not started) |
| Browser back to profile from review | Allowed only if *both* `toProfile` and `profileEditable` pass |
| Forward skip to review from workspace | Allowed only if all required data is present (`canReview`) |
| Viewer role submits profile | Skips workspace entirely, goes to review |
| Any navigation during submitting | Blocked — no `SYNC_TO_STEP` handler in that state |
| Browser back during submitting | URL snaps back to `/onboarding/review` (bridge corrects it) |

#### Exposing navigation availability to the UI

Components often need to know whether back/forward is currently allowed (to disable buttons, show tooltips, etc.). The key insight: **`snapshot.can()` checks if a transition would succeed without firing it**. This means your guards serve double duty — they control transitions AND drive UI state.

##### Don't: reimplement guard logic in the component

```tsx
// BAD: duplicates what the machine already knows
const canGoForward = OnboardingFlow.useSelector((snapshot) => {
  switch (snapshot.value) {
    case 'profile':
      return !!snapshot.context.profileData
    case 'workspace':
      return !!snapshot.context.workspaceData
    case 'review':
      return true
    default:
      return false
  }
})
```

This switch statement reimplements the machine's guards. When you add a new condition (e.g., "workspace also requires a billing address"), you'd have to update both the machine AND this selector — they'll inevitably drift apart.

##### Do: use `snapshot.can()` to ask the machine directly

```tsx
// GOOD: machine is the single authority
const canGoForward = OnboardingFlow.useSelector((snapshot) =>
  snapshot.can({ type: 'NEXT' })
)
```

`snapshot.can()` runs the actual guards defined in the machine without causing a transition. One source of truth, zero duplication.

##### Co-locate selectors with the machine definition

For selectors used across multiple components, define them alongside the machine — not in the components that consume them:

```tsx
// machines/onboarding.ts — selectors live with the machine
import { type SnapshotFrom } from 'xstate'

export const onboardingMachine = setup({ ... }).createMachine({ ... })

// Selectors — co-located so they stay in sync with the machine
export const canGoBackSelector = (snapshot: SnapshotFrom<typeof onboardingMachine>) =>
  snapshot.can({ type: 'BACK' })

export const canGoForwardSelector = (snapshot: SnapshotFrom<typeof onboardingMachine>) =>
  snapshot.can({ type: 'NEXT' })

export const currentStepSelector = (snapshot: SnapshotFrom<typeof onboardingMachine>) =>
  snapshot.value as string
```

This pattern (borrowed from Redux) keeps "how to read machine state" next to "how machine state works." When you change a guard, the selector stays in sync automatically because it delegates to the machine.

##### The `useFlowNavigation` hook

```tsx
// hooks/useFlowNavigation.ts
import { OnboardingFlow } from '../context/onboarding-flow'
import { canGoBackSelector, canGoForwardSelector } from '../machines/onboarding'

export function useFlowNavigation() {
  const actorRef = OnboardingFlow.useActorRef()

  const canGoBack = OnboardingFlow.useSelector(canGoBackSelector)
  const canGoForward = OnboardingFlow.useSelector(canGoForwardSelector)

  return {
    canGoBack,
    canGoForward,
    goBack: () => actorRef.send({ type: 'BACK' }),
  }
}
```

```tsx
// In a step component:
function WorkspaceStep() {
  const { canGoBack, goBack } = useFlowNavigation()

  return (
    <form ...>
      {/* ... */}
      <button
        type="button"
        onClick={goBack}
        disabled={!canGoBack}
        title={canGoBack ? 'Go back' : 'Profile has been locked and cannot be edited'}
      >
        Back
      </button>
      <button type="submit">Next</button>
    </form>
  )
}
```

#### Option: Disable back button entirely (use `replace: true`)

If back/forward should NOT work in your flow at all, skip the guards and use `replace` for all navigations:

```tsx
// In the bridge:
navigate(targetPath, { replace: true })
```

This replaces the history entry so back button goes to wherever the user was *before* the flow started, not to the previous step.

### Route Setup

```tsx
// router.tsx
import { createBrowserRouter } from 'react-router'

export const router = createBrowserRouter([
  {
    path: '/onboarding',
    Component: OnboardingLayout,
    children: [
      { path: 'profile', Component: ProfileStep },
      { path: 'workspace', Component: WorkspaceStep },
      { path: 'review', Component: ReviewStep },
    ],
  },
  {
    path: '/dashboard',
    Component: Dashboard,
  },
])
```

### Route Components (read machine state, send events)

```tsx
// pages/ProfileStep.tsx
import { OnboardingFlow } from '../context/onboarding-flow'

export function ProfileStep() {
  const actorRef = OnboardingFlow.useActorRef()
  const profileData = OnboardingFlow.useSelector(s => s.context.profileData)

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault()
        const data = new FormData(e.currentTarget)
        actorRef.send({
          type: 'SUBMIT_PROFILE',
          name: data.get('name') as string,
          role: data.get('role') as string,
        })
      }}
    >
      <input name="name" defaultValue={profileData?.name ?? ''} />
      <input name="role" defaultValue={profileData?.role ?? ''} />
      <button type="submit">Next</button>
    </form>
  )
}
```

```tsx
// pages/WorkspaceStep.tsx
export function WorkspaceStep() {
  const actorRef = OnboardingFlow.useActorRef()

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault()
        const data = new FormData(e.currentTarget)
        actorRef.send({
          type: 'SUBMIT_WORKSPACE',
          name: data.get('workspace') as string,
          plan: data.get('plan') as string,
        })
      }}
    >
      <input name="workspace" />
      <select name="plan">
        <option value="free">Free</option>
        <option value="pro">Pro</option>
      </select>
      <button type="button" onClick={() => actorRef.send({ type: 'BACK' })}>
        Back
      </button>
      <button type="submit">Next</button>
    </form>
  )
}
```

## Deep Linking: User Lands on `/onboarding/workspace` Directly

Three strategies, pick one:

### Strategy 1: Redirect to start (simplest)

Use a React Router loader to always redirect to step 1:

```tsx
{
  path: '/onboarding',
  loader: () => redirect('/onboarding/profile'),
  children: [...]
}
```

The machine starts at `profile`, URL starts at `/onboarding/profile`. Always in sync.

### Strategy 2: Validate prerequisites in loader

```tsx
{
  path: '/onboarding/workspace',
  loader: async () => {
    const progress = await getOnboardingProgress()
    if (!progress.profileComplete) {
      return redirect('/onboarding/profile')
    }
    return null
  },
  Component: WorkspaceStep,
}
```

### Strategy 3: Persist and rehydrate machine state

```tsx
// Persist on every transition
actorRef.subscribe((snapshot) => {
  sessionStorage.setItem(
    'onboarding-state',
    JSON.stringify(actorRef.getPersistedSnapshot())
  )
})

// Rehydrate on mount
const saved = sessionStorage.getItem('onboarding-state')

<OnboardingFlow.Provider
  options={{
    snapshot: saved ? JSON.parse(saved) : undefined,
  }}
>
```

This is the most robust but adds persistence complexity. Use it when form data must survive refresh.

## Combining with React Query (Pattern A + E)

The onboarding flow often needs to fetch data (e.g., available plans) and submit at the end. Combine the Flow Controller pattern with the router integration:

```tsx
// pages/WorkspaceStep.tsx — React Query runs alongside
export function WorkspaceStep() {
  const actorRef = OnboardingFlow.useActorRef()
  const currentState = OnboardingFlow.useSelector(s => s.value)

  // React Query fetches plans when this step is active
  const { data: plans } = useQuery({
    queryKey: ['plans'],
    queryFn: fetchPlans,
    enabled: currentState === 'workspace',
  })

  return (
    <form onSubmit={...}>
      <select name="plan">
        {plans?.map(p => <option key={p.id} value={p.id}>{p.name}</option>)}
      </select>
      ...
    </form>
  )
}
```

```tsx
// OnboardingLayout.tsx — inject queryClient for the final submission
export function OnboardingLayout() {
  const queryClient = useQueryClient()

  return (
    <OnboardingFlow.Provider
      logic={onboardingMachine.provide({
        actors: {
          completeOnboarding: fromPromise(async ({ input }) => {
            await api.completeOnboarding(input)
            await queryClient.invalidateQueries({ queryKey: ['user'] })
          }),
        },
      })}
    >
      <OnboardingBridge />
      <Outlet />
    </OnboardingFlow.Provider>
  )
}
```

## Where Guards Should Live

| Guard type | Where |
|-----------|-------|
| "Is user authenticated?" | Router loader — `redirect('/login')` |
| "Has user completed prior steps?" | Router loader (reads persisted progress from API/session) |
| "Can user go back from this step?" | Machine guard |
| "Is the form data valid to proceed?" | Machine guard or form validation |
| "Has the flow expired/timed out?" | Machine (use delayed transitions) |

**Rule of thumb:** If the guard needs data from the server or determines whether to *render* the page at all, put it in the router loader. If the guard is about *flow logic within the session*, put it in the machine.

## Trade-offs

**Gains:**
- Each step has a URL — shareable, bookmarkable, refreshable
- Machine keeps flow logic centralized, not scattered across route components
- Back/forward behavior is explicitly controlled
- Works with all four query patterns (A through D)

**Costs:**
- The navigation bridge adds coordination complexity
- Two-way sync requires the loop-breaker ref pattern
- Deep linking requires either redirect-to-start or state persistence
- Machine must handle `SYNC_TO_STEP` events from the router
