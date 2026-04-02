# Pattern D: Mutation Orchestration

**XState coordinates multi-step mutations where order matters, rollback is needed, or intermediate states must be tracked. React Query handles cache invalidation after the machine completes.**

## When to Use

- Sequential mutations where step 2 depends on step 1's result
- Operations that need rollback if a middle step fails
- Complex optimistic update sequences
- File upload flows with progress tracking, retry, and confirmation

## Example: Multi-Step Order Fulfillment

Fulfilling an order requires: reserve inventory → charge payment → create shipment. If any step fails, previous steps must be rolled back.

### Machine Definition

```tsx
// machines/fulfill-order.ts
import { setup, assign, fromPromise } from 'xstate'

type FulfillmentContext = {
  orderId: string
  reservationId: string | null
  chargeId: string | null
  shipmentId: string | null
  error: { step: string; message: string } | null
}

export const fulfillOrderMachine = setup({
  types: {
    context: {} as FulfillmentContext,
    events: {} as
      | { type: 'FULFILL'; orderId: string }
      | { type: 'RETRY' }
      | { type: 'CANCEL' },
    input: {} as { orderId: string },
  },
  actors: {
    // All slots — injected via machine.provide()
    reserveInventory: fromPromise<{ reservationId: string }, { orderId: string }>(
      async () => { throw new Error('not provided') }
    ),
    chargePayment: fromPromise<{ chargeId: string }, { orderId: string }>(
      async () => { throw new Error('not provided') }
    ),
    createShipment: fromPromise<{ shipmentId: string }, { orderId: string; reservationId: string }>(
      async () => { throw new Error('not provided') }
    ),
    // Rollback actors
    releaseInventory: fromPromise<void, { reservationId: string }>(
      async () => { throw new Error('not provided') }
    ),
    refundPayment: fromPromise<void, { chargeId: string }>(
      async () => { throw new Error('not provided') }
    ),
    invalidateCache: fromPromise<void, { orderId: string }>(
      async () => { throw new Error('not provided') }
    ),
  },
}).createMachine({
  id: 'fulfillOrder',
  initial: 'idle',
  context: ({ input }) => ({
    orderId: input.orderId,
    reservationId: null,
    chargeId: null,
    shipmentId: null,
    error: null,
  }),
  states: {
    idle: {
      on: {
        FULFILL: 'reservingInventory',
      },
    },

    reservingInventory: {
      invoke: {
        src: 'reserveInventory',
        input: ({ context }) => ({ orderId: context.orderId }),
        onDone: {
          target: 'chargingPayment',
          actions: assign({
            reservationId: ({ event }) => event.output.reservationId,
          }),
        },
        onError: {
          target: 'failed',
          actions: assign({
            error: ({ event }) => ({
              step: 'reserveInventory',
              message: (event.error as Error).message,
            }),
          }),
        },
      },
    },

    chargingPayment: {
      invoke: {
        src: 'chargePayment',
        input: ({ context }) => ({ orderId: context.orderId }),
        onDone: {
          target: 'creatingShipment',
          actions: assign({
            chargeId: ({ event }) => event.output.chargeId,
          }),
        },
        onError: {
          // Payment failed → need to release inventory
          target: 'rollingBackInventory',
          actions: assign({
            error: ({ event }) => ({
              step: 'chargePayment',
              message: (event.error as Error).message,
            }),
          }),
        },
      },
    },

    creatingShipment: {
      invoke: {
        src: 'createShipment',
        input: ({ context }) => ({
          orderId: context.orderId,
          reservationId: context.reservationId!,
        }),
        onDone: {
          target: 'invalidatingCache',
          actions: assign({
            shipmentId: ({ event }) => event.output.shipmentId,
          }),
        },
        onError: {
          // Shipment failed → need to refund AND release inventory
          target: 'rollingBackPayment',
          actions: assign({
            error: ({ event }) => ({
              step: 'createShipment',
              message: (event.error as Error).message,
            }),
          }),
        },
      },
    },

    // --- Success path ---

    invalidatingCache: {
      invoke: {
        src: 'invalidateCache',
        input: ({ context }) => ({ orderId: context.orderId }),
        onDone: 'complete',
        onError: 'complete', // cache invalidation failure is non-critical
      },
    },

    complete: {
      type: 'final',
    },

    // --- Rollback path ---

    rollingBackPayment: {
      invoke: {
        src: 'refundPayment',
        input: ({ context }) => ({ chargeId: context.chargeId! }),
        onDone: 'rollingBackInventory',
        onError: {
          // Rollback failed — critical error, needs manual intervention
          target: 'rollbackFailed',
        },
      },
    },

    rollingBackInventory: {
      invoke: {
        src: 'releaseInventory',
        input: ({ context }) => ({ reservationId: context.reservationId! }),
        onDone: 'failed',
        onError: 'rollbackFailed',
      },
    },

    failed: {
      on: {
        RETRY: 'reservingInventory',
        CANCEL: 'cancelled',
      },
    },

    rollbackFailed: {
      // Manual intervention needed — show error to admin
      type: 'final',
    },

    cancelled: {
      type: 'final',
    },
  },
})
```

### React Component

```tsx
// components/FulfillOrder.tsx
import { useMachine } from '@xstate/react'
import { useQueryClient } from '@tanstack/react-query'
import { fromPromise } from 'xstate'
import { fulfillOrderMachine } from '../machines/fulfill-order'
import * as api from '../api/orders'

export function FulfillOrder({ orderId }: { orderId: string }) {
  const queryClient = useQueryClient()

  const [snapshot, send] = useMachine(
    fulfillOrderMachine.provide({
      actors: {
        reserveInventory: fromPromise(async ({ input }) =>
          api.reserveInventory(input.orderId)
        ),
        chargePayment: fromPromise(async ({ input }) =>
          api.chargePayment(input.orderId)
        ),
        createShipment: fromPromise(async ({ input }) =>
          api.createShipment(input.orderId, input.reservationId)
        ),
        releaseInventory: fromPromise(async ({ input }) =>
          api.releaseInventory(input.reservationId)
        ),
        refundPayment: fromPromise(async ({ input }) =>
          api.refundPayment(input.chargeId)
        ),
        invalidateCache: fromPromise(async ({ input }) => {
          await queryClient.invalidateQueries({ queryKey: ['order', input.orderId] })
          await queryClient.invalidateQueries({ queryKey: ['orders'] })
          await queryClient.invalidateQueries({ queryKey: ['inventory'] })
        }),
      },
    }),
    { input: { orderId } }
  )

  const { value, context } = snapshot

  return (
    <div>
      <h3>Fulfilling Order {orderId}</h3>

      {/* Progress indicator */}
      <FulfillmentProgress
        steps={[
          { label: 'Reserve Inventory', status: stepStatus(value, 'reservingInventory') },
          { label: 'Charge Payment', status: stepStatus(value, 'chargingPayment') },
          { label: 'Create Shipment', status: stepStatus(value, 'creatingShipment') },
        ]}
      />

      {snapshot.matches('idle') && (
        <button onClick={() => send({ type: 'FULFILL', orderId })}>
          Start Fulfillment
        </button>
      )}

      {snapshot.matches('failed') && (
        <div>
          <p>Failed at: {context.error?.step}</p>
          <p>{context.error?.message}</p>
          <button onClick={() => send({ type: 'RETRY' })}>Retry</button>
          <button onClick={() => send({ type: 'CANCEL' })}>Cancel</button>
        </div>
      )}

      {snapshot.matches('rollbackFailed') && (
        <div>
          <p>CRITICAL: Rollback failed. Manual intervention required.</p>
          <p>Reservation: {context.reservationId}</p>
          <p>Charge: {context.chargeId}</p>
        </div>
      )}

      {snapshot.matches('complete') && (
        <div>
          <p>Order fulfilled successfully!</p>
          <p>Shipment: {context.shipmentId}</p>
        </div>
      )}
    </div>
  )
}

function stepStatus(currentState: string, stepState: string) {
  const order = ['reservingInventory', 'chargingPayment', 'creatingShipment']
  const currentIdx = order.indexOf(currentState as string)
  const stepIdx = order.indexOf(stepState)

  if (currentState === 'complete' || currentState === 'invalidatingCache') return 'done'
  if (currentIdx === stepIdx) return 'active'
  if (currentIdx > stepIdx) return 'done'
  return 'pending'
}
```

### Testing

```tsx
// machines/__tests__/fulfill-order.test.ts
import { createActor, fromPromise } from 'xstate'
import { fulfillOrderMachine } from '../fulfill-order'

function createTestActor(overrides: Record<string, () => Promise<any>> = {}) {
  return createActor(
    fulfillOrderMachine.provide({
      actors: {
        reserveInventory: fromPromise(async () => ({ reservationId: 'res-1' })),
        chargePayment: fromPromise(async () => ({ chargeId: 'chg-1' })),
        createShipment: fromPromise(async () => ({ shipmentId: 'ship-1' })),
        releaseInventory: fromPromise(async () => {}),
        refundPayment: fromPromise(async () => {}),
        invalidateCache: fromPromise(async () => {}),
        ...Object.fromEntries(
          Object.entries(overrides).map(([k, v]) => [k, fromPromise(v)])
        ),
      },
    }),
    { input: { orderId: 'order-1' } }
  )
}

test('happy path: reserve → charge → ship → complete', async () => {
  const actor = createTestActor()
  actor.start()
  actor.send({ type: 'FULFILL', orderId: 'order-1' })

  // Wait for all async steps
  await new Promise((r) => setTimeout(r, 50))

  expect(actor.getSnapshot().value).toBe('complete')
  expect(actor.getSnapshot().context.shipmentId).toBe('ship-1')
})

test('payment failure triggers inventory rollback', async () => {
  const actor = createTestActor({
    chargePayment: async () => { throw new Error('Card declined') },
  })
  actor.start()
  actor.send({ type: 'FULFILL', orderId: 'order-1' })

  await new Promise((r) => setTimeout(r, 50))

  expect(actor.getSnapshot().value).toBe('failed')
  expect(actor.getSnapshot().context.error?.step).toBe('chargePayment')
  // Inventory was released (rollingBackInventory completed)
  expect(actor.getSnapshot().context.reservationId).toBe('res-1')
})

test('shipment failure triggers payment refund then inventory release', async () => {
  const rollbackOrder: string[] = []

  const actor = createTestActor({
    createShipment: async () => { throw new Error('No carriers available') },
    refundPayment: async () => { rollbackOrder.push('refund') },
    releaseInventory: async () => { rollbackOrder.push('release') },
  })
  actor.start()
  actor.send({ type: 'FULFILL', orderId: 'order-1' })

  await new Promise((r) => setTimeout(r, 50))

  expect(actor.getSnapshot().value).toBe('failed')
  expect(rollbackOrder).toEqual(['refund', 'release'])
})
```

## Why This Pattern Shines

This is where XState truly adds value over React Query alone. Try implementing this with just `useMutation`:

```tsx
// Without XState — painful and error-prone
const fulfillOrder = useMutation({
  mutationFn: async (orderId) => {
    const reservation = await reserveInventory(orderId)
    try {
      const charge = await chargePayment(orderId)
      try {
        const shipment = await createShipment(orderId, reservation.id)
        return shipment
      } catch (e) {
        await refundPayment(charge.chargeId)  // what if THIS fails?
        await releaseInventory(reservation.id) // and THIS?
        throw e
      }
    } catch (e) {
      await releaseInventory(reservation.id) // duplicated rollback logic
      throw e
    }
  },
})
// No intermediate progress, no clear state, no retry from specific step,
// no handling of rollback failures, no testability of the flow
```

The machine makes every state explicit, every transition visible, every failure mode testable.

## Trade-offs

**Gains:**
- Every step and failure mode is an explicit, named state
- Rollback logic is structural, not buried in nested try/catch
- Progress tracking is built in (just check `snapshot.value`)
- Each step is independently testable via actor injection
- Retry can resume from the failed step (not shown above, but trivial to add)

**Costs:**
- More upfront code than a simple `useMutation`
- Overkill for single-step mutations (just use `useMutation` directly)
- Cache invalidation is a separate step — must remember to include it
