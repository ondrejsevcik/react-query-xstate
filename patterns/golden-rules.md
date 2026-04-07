# Golden Rules

## 1. Never duplicate server state

The query cache is the single source of truth for server data. Machine context holds only **user decisions** and **flow state** (selected IDs, form inputs, current step).

```tsx
// GOOD: machine holds the ID, React Query holds the data
context: { selectedProductId: 'prod-123' }
// + useQuery(['product', snapshot.context.selectedProductId])

// BAD: machine holds a copy of the product
context: { selectedProduct: { id: 'prod-123', name: 'Widget', price: 9.99 } }
```

## 2. Pick a direction for data flow

Each piece of server interaction should flow in ONE direction:

| Direction | Mechanism | Use when |
|-----------|-----------|----------|
| Machine → Query | `queryClient.ensureQueryData()` in `fromPromise` actor | Machine must control when/what to fetch |
| Query → Machine | `useEffect` bridge or mutation `onSuccess`/`onError` | Machine must react to data content |
| Independent | `enabled` flag derived from machine state | Machine controls flow, Query controls data |

**Never mix directions for the same data.** If the machine fetches user data via `ensureQueryData`, don't also have a `useQuery(['user'])` with a `useEffect` bridge pushing data back into the machine.

## 3. Use `machine.provide()` for testability

Always define actors as slots in `setup()` and inject implementations at the call site:

```tsx
// machine.ts — testable, framework-agnostic
setup({
  actors: {
    loadData: fromPromise<Data, Input>(async () => {
      throw new Error('must be provided')
    }),
  },
})

// component.tsx — wires in queryClient
useMachine(machine.provide({
  actors: {
    loadData: fromPromise(async ({ input }) =>
      queryClient.ensureQueryData({ ... })
    ),
  },
}))

// test.ts — wires in mock
createActor(machine.provide({
  actors: {
    loadData: fromPromise(async () => mockData),
  },
}))
```

## 4. Let mutations signal completion via events

When React Query `useMutation` handles a write, send events to the machine in callbacks:

```tsx
const mutation = useMutation({
  mutationFn: submitForm,
  onSuccess: () => send({ type: 'SUBMIT_SUCCESS' }),
  onError: () => send({ type: 'SUBMIT_ERROR' }),
})
```

Don't use `useEffect` to watch `mutation.isSuccess` — the callbacks are the clean path.

## 5. Disable queries when they're not needed

Use machine state to control the `enabled` flag:

```tsx
const { data } = useQuery({
  queryKey: ['shipping-rates', zip],
  queryFn: () => fetchRates(zip),
  enabled: snapshot.matches('shipping') && !!zip,
})
```

This prevents unnecessary fetches and respects the machine's flow control.

## 6. Cache invalidation belongs at the boundary

After a mutation completes (whether via XState actor or `useMutation`), invalidate relevant queries:

```tsx
// In a fromPromise actor (Pattern B/D):
await queryClient.invalidateQueries({ queryKey: ['orders'] })

// In useMutation callbacks (Pattern A):
onSettled: () => queryClient.invalidateQueries({ queryKey: ['orders'] })
```

## 7. Don't reimagine React Query's loading states in XState

If you find yourself writing this machine:

```tsx
states: {
  idle: {},
  loading: {},
  success: {},
  error: {},
  retrying: {},
}
```

...you're rebuilding React Query. Just use `useQuery`. Reserve XState for flows with **multiple meaningful states** and **transitions between them**.

## 8. Prefer `ensureQueryData` over `fetchQuery`

Both methods read the cache and respect `staleTime` — neither blindly fetches. The differences are subtler:

| Method | Behavior | Use when |
|--------|----------|----------|
| `ensureQueryData` | Returns cache if fresh. If stale/missing, fetches. Supports `revalidateIfStale` for background refresh | Default choice — "make sure this data exists" |
| `fetchQuery` | Returns cache if fresh. If stale/missing, fetches. Throws on error | You need to handle errors explicitly in the machine's `onError` |

`ensureQueryData` is correct 90% of the time — its "ensure" semantic matches what XState actors typically need: give me the data, whether from cache or network.

## 9. Test machines without React

The machine is a pure function of state + events. Test it with `createActor`:

```tsx
const actor = createActor(machine.provide({ actors: { ... } }))
actor.start()
actor.send({ type: 'SOME_EVENT' })
expect(actor.getSnapshot().value).toBe('expectedState')
```

No React Testing Library, no QueryClientProvider, no renders.

## 10. Machine definition vs actor instance

The machine (`createMachine(...)`) is a static blueprint — always defined at module level. The **actor** is a running instance. Where you create the actor determines its lifecycle:

| Scope | How | Lifecycle |
|-------|-----|-----------|
| Single component | `useMachine(machine)` | Created on mount, stopped on unmount |
| Shared across routes | `createActorContext(machine)` + `<Provider>` | Tied to Provider mount |
| App-global singleton | `createActor(machine).start()` at module scope | Lives forever (rare, hard to test) |

**Default to `useMachine`**. Use `createActorContext` when a flow spans multiple routes (Pattern E). Avoid module-level singletons unless you have a truly global background process (WebSocket manager, auth session) — they're hard to test and can't receive React-specific dependencies like `queryClient`.

## 11. Use `snapshot.can()` instead of reimplementing guards in UI

When components need to know if a transition is possible (to disable buttons, show tooltips), ask the machine — don't rewrite its guards in component code:

```tsx
// BAD: duplicates guard logic, will drift
const canGoForward = useSelector((snapshot) => {
  switch (snapshot.value) {
    case 'profile': return !!snapshot.context.profileData
    case 'workspace': return !!snapshot.context.workspaceData
    default: return false
  }
})

// GOOD: machine is the single authority
const canGoForward = useSelector((snapshot) =>
  snapshot.can({ type: 'NEXT' })
)
```

Co-locate selectors with the machine definition so they stay in sync:

```tsx
// machines/onboarding.ts
export const canGoBackSelector = (snapshot: SnapshotFrom<typeof onboardingMachine>) =>
  snapshot.can({ type: 'BACK' })
```

## 12. Start with Pattern A, escalate to Pattern B when needed

Pattern A (Flow Controller) is the simplest and cleanest. Only escalate to Pattern B (Orchestrator) when you need the machine to branch based on server data content. Only escalate to Pattern D when you have multi-step mutations with rollback needs.
