# Pattern C: Reactive Bridge

**`useQuery` runs declaratively, and its results are sent to the machine as events. The machine reacts to data changes without owning the fetch lifecycle.**

## When to Use

- You need React Query's full per-observer features AND the machine needs to react to data content
- Real-time data (polling, WebSocket-backed queries) where the machine must respond to updates
- Transition between patterns: this is a stepping stone from Pattern B toward Pattern A

## Warning: Use Sparingly

This pattern introduces a `useEffect` bridge — React's classic "sync external state" escape hatch. It works, but it creates render timing complexity. Prefer Pattern A or B when possible.

## Example: Real-Time Order Tracking

An order status is polled from the server. The machine must transition based on the current status to show different UIs and trigger notifications.

### Machine Definition

```tsx
// machines/order-tracking.ts
import { setup, assign } from 'xstate'

type OrderStatus = 'processing' | 'shipped' | 'out_for_delivery' | 'delivered' | 'cancelled'

type OrderData = {
  id: string
  status: OrderStatus
  estimatedDelivery: string | null
  trackingNumber: string | null
}

export const orderTrackingMachine = setup({
  types: {
    context: {} as {
      orderId: string
      // Note: this looks like duplicating server state (see Golden Rule 1), but it serves
      // a different purpose — it's the machine's memory of which transitions already fired,
      // not a cache of server data. The query cache remains the source of truth for display.
      lastKnownStatus: OrderStatus | null
    },
    events: {} as
      | { type: 'ORDER_UPDATE'; data: OrderData }
      | { type: 'FETCH_ERROR'; error: string }
      | { type: 'DISMISS' },
  },
  actions: {
    notifyShipped: ({ event }) => {
      // Side effect: show a toast, send analytics, etc.
      if (event.type === 'ORDER_UPDATE') {
        console.log('Order shipped!', event.data.trackingNumber)
      }
    },
    notifyDelivered: () => {
      console.log('Order delivered!')
    },
  },
}).createMachine({
  id: 'orderTracking',
  initial: 'watching',
  context: ({ input }: { input: { orderId: string } }) => ({
    orderId: input.orderId,
    lastKnownStatus: null,
  }),
  states: {
    watching: {
      on: {
        ORDER_UPDATE: [
          {
            guard: ({ event }) => event.data.status === 'shipped',
            target: 'shipped',
            actions: [
              assign({ lastKnownStatus: 'shipped' }),
              'notifyShipped',
            ],
          },
          {
            guard: ({ event }) => event.data.status === 'out_for_delivery',
            target: 'outForDelivery',
            actions: assign({ lastKnownStatus: 'out_for_delivery' }),
          },
          {
            guard: ({ event }) => event.data.status === 'delivered',
            target: 'delivered',
            actions: ['notifyDelivered', assign({ lastKnownStatus: 'delivered' })],
          },
          {
            guard: ({ event }) => event.data.status === 'cancelled',
            target: 'cancelled',
            actions: assign({ lastKnownStatus: 'cancelled' }),
          },
          {
            // Default: stay in watching, update status
            actions: assign({ lastKnownStatus: ({ event }) => event.data.status }),
          },
        ],
        FETCH_ERROR: 'error',
      },
    },

    shipped: {
      on: {
        ORDER_UPDATE: [
          {
            guard: ({ event }) => event.data.status === 'out_for_delivery',
            target: 'outForDelivery',
          },
          {
            guard: ({ event }) => event.data.status === 'delivered',
            target: 'delivered',
            actions: 'notifyDelivered',
          },
        ],
      },
    },

    outForDelivery: {
      on: {
        ORDER_UPDATE: {
          guard: ({ event }) => event.data.status === 'delivered',
          target: 'delivered',
          actions: 'notifyDelivered',
        },
      },
    },

    delivered: {
      type: 'final',
    },

    cancelled: {
      type: 'final',
    },

    error: {
      on: {
        ORDER_UPDATE: 'watching', // recovered
      },
    },
  },
})
```

### React Component (the bridge)

```tsx
// components/OrderTracker.tsx
import { useActorRef, useSelector } from '@xstate/react'
import { useQuery } from '@tanstack/react-query'
import { useEffect } from 'react'
import { orderTrackingMachine } from '../machines/order-tracking'

export function OrderTracker({ orderId }: { orderId: string }) {
  const actorRef = useActorRef(orderTrackingMachine, {
    input: { orderId },
  })

  const step = useSelector(actorRef, (s) => s.value)

  // React Query polls for updates
  const { data, error } = useQuery({
    queryKey: ['order', orderId],
    queryFn: () => fetchOrderStatus(orderId),
    refetchInterval: 30_000, // poll every 30s
    // Stop polling when order is in a final state
    enabled: step !== 'delivered' && step !== 'cancelled',
  })

  // Bridge: send data updates to machine
  // React Query's structural sharing keeps `data` reference stable
  // when the response hasn't changed, so this only fires on real updates
  useEffect(() => {
    if (data) {
      actorRef.send({ type: 'ORDER_UPDATE', data })
    }
  }, [data, actorRef])

  useEffect(() => {
    if (error) {
      actorRef.send({ type: 'FETCH_ERROR', error: error.message })
    }
  }, [error, actorRef])

  // Render based on machine state
  return (
    <div>
      <OrderStatusTimeline currentState={step} />
      {step === 'shipped' && <TrackingMap orderId={orderId} />}
      {step === 'outForDelivery' && <LiveTrackingMap orderId={orderId} />}
      {step === 'delivered' && <DeliveryConfirmation />}
      {step === 'error' && <p>Having trouble loading updates...</p>}
    </div>
  )
}
```

## Making the Bridge Safe

### Why the bridge is simpler than it looks

React Query's **structural sharing** (`structuralSharing: true`, the default) keeps the `data` reference stable when the refetched response is deeply equal. This means `data` only gets a new reference when the data actually changed — so a simple `[data, send]` dependency array is all you need.

```tsx
// This is sufficient — no ref, no dataUpdatedAt needed
useEffect(() => {
  if (data) {
    actorRef.send({ type: 'ORDER_UPDATE', data })
  }
}, [data, actorRef])
```

### What about React Strict Mode?

Strict mode double-invokes effects, so the machine may receive the same event twice on mount. This is harmless — the machine handles it as a no-op since the guards check for a *different* status (e.g., a machine already in `shipped` ignores another `ORDER_UPDATE` with `status: 'shipped'`).

### Avoiding render loops

If sending an event causes a state change that changes `enabled`, which triggers a fetch, which triggers `useEffect`, which sends an event...

**Solution:** Design the machine so that `ORDER_UPDATE` with the same status is a no-op (no transition). The guards above already do this — each transition requires a *different* status.

## Custom Hook to Encapsulate the Bridge

> **Note:** `useEffectEvent` requires React 19+.

```tsx
// hooks/useQueryBridge.ts
import { useEffect, useEffectEvent } from 'react'
import { type ActorRef, type EventObject } from 'xstate'

/**
 * Bridges a React Query result to an XState machine by sending events
 * when the query data changes.
 *
 * Uses useEffectEvent (React 19+) so the callbacks always read the latest
 * closure values without needing to be in the effect dependency array.
 * Consumers can pass inline arrows freely — no useCallback required.
 */
export function useQueryBridge<TData, TEvent extends EventObject>(
  actorRef: ActorRef<any, TEvent>,
  query: { data: TData | undefined; error: Error | null },
  options: {
    onData: (data: TData) => TEvent
    onError?: (error: Error) => TEvent
  }
) {
  const handleData = useEffectEvent((data: TData) => {
    actorRef.send(options.onData(data))
  })

  const handleError = useEffectEvent((error: Error) => {
    if (options.onError) {
      actorRef.send(options.onError(error))
    }
  })

  useEffect(() => {
    if (query.data) handleData(query.data)
  }, [query.data])

  useEffect(() => {
    if (query.error) handleError(query.error)
  }, [query.error])
}

// Usage — inline arrows are fine, no useCallback needed:
const actorRef = useActorRef(orderTrackingMachine, { input: { orderId } })
const query = useQuery({ queryKey: ['order', orderId], queryFn: ... })

useQueryBridge(actorRef, query, {
  onData: (data) => ({ type: 'ORDER_UPDATE', data }),
  onError: (error) => ({ type: 'FETCH_ERROR', error: error.message }),
})
```

## Trade-offs

**Gains:**
- Full React Query features: polling, background refetch, `select`, deduplication
- Machine reacts to data content (can branch based on API response)
- React Query manages the fetch lifecycle — retries, caching, stale-time all work

**Costs:**
- `useEffect` bridge adds complexity and render-timing considerations
- Must guard against duplicate sends and render loops
- Harder to test — needs both React tree and query client
- Machine doesn't control *when* fetching happens (React Query does)
