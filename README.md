# React Query + XState: Integration Patterns

A practical guide for combining TanStack Query v5 and XState v5 in React applications.

## The Core Principle

> **React Query owns server state. XState owns application flow. Neither should do the other's job.**

| Concern | Owner | Why |
|---------|-------|-----|
| Server data (fetching, caching, syncing) | React Query | Built-in cache, deduplication, background refetch, stale-time |
| UI flow (wizard steps, multi-step forms) | XState | Explicit states, guarded transitions, impossible states are impossible |
| Ephemeral UI state (selected tab, open modal) | React `useState` or XState | Either works; XState if it's part of a flow |
| Form state before submission | XState context or React Hook Form | XState if the form is part of a multi-step flow |

## When You DON'T Need Both

**React Query alone** is sufficient for:
- Simple CRUD (list, detail, create, edit, delete)
- Data fetching with loading/error states
- Optimistic updates
- Pagination, infinite scroll
- Polling / refetch intervals

**XState alone** is sufficient for:
- Complex UI flows with no server interaction
- State machines for animations, drag-and-drop
- Local-only business logic (calculators, games)

**Use both** when you have:
- Multi-step flows that involve server data (checkout, onboarding)
- Complex error recovery (retry strategies, fallback paths)
- Flows where an API response determines the next step
- Orchestrated mutations (save A, then B, then invalidate C)

## Patterns

See the `patterns/` directory for detailed examples of each integration pattern:

1. **[Pattern A: Machine as Flow Controller](patterns/pattern-a-flow-controller.md)** — XState manages flow only, React Query runs alongside
2. **[Pattern B: Machine as Orchestrator](patterns/pattern-b-orchestrator.md)** — XState invokes queries via `queryClient` imperative API
3. **[Pattern C: Reactive Bridge](patterns/pattern-c-reactive-bridge.md)** — `useQuery` results synced back to machine via events
4. **[Pattern D: Mutation Orchestration](patterns/pattern-d-mutation-orchestration.md)** — XState coordinates multi-step mutations

## Decision Guide

```
Do you have a multi-step flow or complex state transitions?
  NO  → Use React Query alone
  YES → Does the flow need server data to determine next steps?
    NO  → Pattern A (Flow Controller) — cleanest separation
    YES → Does the machine need to control WHEN fetching happens?
      NO  → Pattern C (Reactive Bridge) — React Query stays declarative  
      YES → Pattern B (Orchestrator) — machine drives everything
              ↳ Multiple sequential mutations? → Pattern D
```

## Anti-Patterns

### 1. Double State (storing server data in machine context AND query cache)
```tsx
// BAD: data lives in two places that can diverge
assign({ todos: ({ event }) => event.output })  // machine context
// AND useQuery(['todos']) returns the same data from cache
```

### 2. useEffect Ping-Pong
```tsx
// BAD: creates render loops and timing issues
useEffect(() => {
  if (data) send({ type: 'DATA_LOADED', data })
}, [data])  // query result → machine → state change → re-render → repeat?
```

### 3. Reimplementing React Query in XState
```tsx
// BAD: you're building a worse cache
states: {
  idle: {},
  loading: {},     // React Query already handles this
  success: {},     // React Query already handles this  
  error: {},       // React Query already handles this
  retrying: {},    // React Query already handles this
}
```

## Golden Rules

See **[Golden Rules](patterns/golden-rules.md)** for the 10 principles that keep this integration clean. The top 3:

1. **Never duplicate server state** — query cache is the single source of truth for server data
2. **Pick one data-flow direction** per interaction — don't mix Machine→Query and Query→Machine for the same data  
3. **Use `machine.provide()` for testability** — machines stay framework-agnostic

## Testing

Machines are testable in isolation — no React, no query client needed:

```ts
import { createActor } from 'xstate'

const actor = createActor(
  checkoutMachine.provide({
    actors: {
      submitOrder: fromPromise(async () => ({ orderId: '123' })),
    },
  })
)
actor.start()
actor.send({ type: 'SELECT_PRODUCT', id: 'prod-1' })
expect(actor.getSnapshot().value).toBe('shipping')
```

For integration tests, always disable React Query retries:
```ts
const queryClient = new QueryClient({
  defaultOptions: { queries: { retry: false } },
})
```
