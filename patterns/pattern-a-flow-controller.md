# Pattern A: Machine as Flow Controller

**XState manages only the UI flow (which step, what's selected, navigation guards). React Query runs alongside with `useQuery`/`useMutation` hooks, driven by machine state via the `enabled` flag.**

## When to Use

- Multi-step flows (checkout, onboarding, wizards)
- Flow logic is independent of server data shape
- You want full React Query features (background refetch, `select`, `staleTime` per query)
- The machine doesn't need to "know" about the server data — just which step the user is on

## The Cleanest Pattern

This is the recommended starting point. It keeps the separation crisp: XState never touches server data, React Query never manages flow state.

## Example: Checkout Flow

### Machine Definition (pure flow, no server data)

```tsx
// machines/checkout.ts
import { setup, assign } from 'xstate'

type Address = {
  street: string
  city: string
  zip: string
}

export const checkoutMachine = setup({
  types: {
    context: {} as {
      selectedProductId: string | null
      quantity: number
      shippingAddress: Address | null
      paymentMethodId: string | null
    },
    events: {} as
      | { type: 'SELECT_PRODUCT'; productId: string }
      | { type: 'SET_QUANTITY'; quantity: number }
      | { type: 'SUBMIT_SHIPPING'; address: Address }
      | { type: 'SELECT_PAYMENT'; methodId: string }
      | { type: 'CONFIRM' }
      | { type: 'ORDER_PLACED' }
      | { type: 'ORDER_FAILED' }
      | { type: 'BACK' }
      | { type: 'RESET' },
  },
  guards: {
    hasProduct: ({ context }) => context.selectedProductId !== null,
    hasShipping: ({ context }) => context.shippingAddress !== null,
    hasPayment: ({ context }) => context.paymentMethodId !== null,
  },
}).createMachine({
  id: 'checkout',
  initial: 'selectingProduct',
  context: {
    selectedProductId: null,
    quantity: 1,
    shippingAddress: null,
    paymentMethodId: null,
  },
  states: {
    selectingProduct: {
      on: {
        SELECT_PRODUCT: {
          actions: assign({ selectedProductId: ({ event }) => event.productId }),
        },
        SET_QUANTITY: {
          actions: assign({ quantity: ({ event }) => event.quantity }),
        },
        SUBMIT_SHIPPING: {
          // Can jump to shipping only if product is selected
          guard: 'hasProduct',
          target: 'shipping',
          actions: assign({ shippingAddress: ({ event }) => event.address }),
        },
      },
    },

    shipping: {
      on: {
        SUBMIT_SHIPPING: {
          target: 'payment',
          actions: assign({ shippingAddress: ({ event }) => event.address }),
        },
        BACK: 'selectingProduct',
      },
    },

    payment: {
      on: {
        SELECT_PAYMENT: {
          target: 'reviewing',
          actions: assign({ paymentMethodId: ({ event }) => event.methodId }),
        },
        BACK: 'shipping',
      },
    },

    reviewing: {
      on: {
        CONFIRM: 'submitting',
        BACK: 'payment',
      },
    },

    submitting: {
      // NOTE: No invoke here! The mutation is handled by React Query.
      // The machine just represents "we are in the submitting state"
      on: {
        ORDER_PLACED: 'complete',
        ORDER_FAILED: 'reviewing', // go back to review to retry
      },
    },

    complete: {
      on: {
        RESET: {
          target: 'selectingProduct',
          actions: assign({
            selectedProductId: null,
            quantity: 1,
            shippingAddress: null,
            paymentMethodId: null,
          }),
        },
      },
    },
  },
})
```

### React Component (React Query runs alongside)

```tsx
// components/Checkout.tsx
import { useActorRef, useSelector } from '@xstate/react'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { checkoutMachine } from '../machines/checkout'

export function Checkout() {
  const actorRef = useActorRef(checkoutMachine)
  const queryClient = useQueryClient()

  // --- Select only what's needed ---
  const step = useSelector(actorRef, (s) => s.value)
  const selectedProductId = useSelector(actorRef, (s) => s.context.selectedProductId)
  const quantity = useSelector(actorRef, (s) => s.context.quantity)
  const shippingAddress = useSelector(actorRef, (s) => s.context.shippingAddress)
  const paymentMethodId = useSelector(actorRef, (s) => s.context.paymentMethodId)

  // --- React Query handles all server data ---

  // Products list — always available
  const { data: products } = useQuery({
    queryKey: ['products'],
    queryFn: fetchProducts,
  })

  // Selected product details — skipToken disables the query when there's no ID
  const { data: product } = useQuery({
    queryKey: ['product', selectedProductId],
    queryFn: selectedProductId
      ? () => fetchProduct(selectedProductId)
      : skipToken,
  })

  // Shipping rates — skipToken disables the query until we have a zip
  const { data: shippingRates } = useQuery({
    queryKey: ['shipping-rates', shippingAddress?.zip],
    queryFn: step === 'shipping' && shippingAddress?.zip
      ? () => fetchShippingRates(shippingAddress.zip)
      : skipToken,
  })

  // Payment methods — prefetch when user reaches payment step
  const { data: paymentMethods } = useQuery({
    queryKey: ['payment-methods'],
    queryFn: fetchPaymentMethods,
    enabled: step === 'payment' || step === 'reviewing',
  })

  // Place order mutation — callbacks are the sole bridge to the machine.
  // onMutate fires synchronously before the request, so the machine
  // transitions to 'submitting' at exactly the right moment.
  const placeOrder = useMutation({
    mutationFn: (order: OrderPayload) => submitOrder(order),
    onMutate: () => actorRef.send({ type: 'CONFIRM' }),
    onSuccess: () => {
      actorRef.send({ type: 'ORDER_PLACED' })
      queryClient.invalidateQueries({ queryKey: ['orders'] })
    },
    onError: () => {
      actorRef.send({ type: 'ORDER_FAILED' })
    },
  })

  // --- Machine state determines which UI to show ---

  if (step === 'selectingProduct') {
    return (
      <ProductSelector
        products={products}
        selectedId={selectedProductId}
        quantity={quantity}
        onSelect={(productId) => actorRef.send({ type: 'SELECT_PRODUCT', productId })}
        onQuantityChange={(quantity) => actorRef.send({ type: 'SET_QUANTITY', quantity })}
        onNext={(address) => actorRef.send({ type: 'SUBMIT_SHIPPING', address })}
      />
    )
  }

  if (step === 'shipping') {
    return (
      <ShippingForm
        rates={shippingRates}
        onSubmit={(address) => actorRef.send({ type: 'SUBMIT_SHIPPING', address })}
        onBack={() => actorRef.send({ type: 'BACK' })}
      />
    )
  }

  if (step === 'payment') {
    return (
      <PaymentSelector
        methods={paymentMethods}
        onSelect={(methodId) => actorRef.send({ type: 'SELECT_PAYMENT', methodId })}
        onBack={() => actorRef.send({ type: 'BACK' })}
      />
    )
  }

  if (step === 'reviewing') {
    return (
      <OrderReview
        product={product}
        quantity={quantity}
        address={shippingAddress}
        onConfirm={() =>
          placeOrder.mutate({
            productId: selectedProductId!,
            quantity: quantity,
            shippingAddress: shippingAddress!,
            paymentMethodId: paymentMethodId!,
          })
        }
        onBack={() => actorRef.send({ type: 'BACK' })}
      />
    )
  }

  if (step === 'submitting') {
    return <SubmittingOrder />
  }

  if (step === 'complete') {
    return (
      <OrderComplete
        onNewOrder={() => actorRef.send({ type: 'RESET' })}
      />
    )
  }

  return null
}
```

## Key Observations

### 1. No data in machine context
The machine context only holds **user decisions**: which product they picked, what address they entered. The actual product data, shipping rates, and payment methods live in the React Query cache.

### 2. `enabled` flag derives from machine state
```tsx
enabled: step === 'shipping' && !!shippingAddress?.zip
```
This is the bridge — machine state controls when queries run, but React Query manages the fetching lifecycle.

### 3. Mutation callbacks are the sole bridge to the machine
```tsx
onMutate: () => actorRef.send({ type: 'CONFIRM' })      // → submitting (sync, before request)
onSuccess: () => actorRef.send({ type: 'ORDER_PLACED' }) // → complete
onError: () => actorRef.send({ type: 'ORDER_FAILED' })   // → reviewing (retry)
```
The mutation's lifecycle callbacks drive all machine transitions. `onMutate` fires synchronously before the request, so the machine enters `submitting` at exactly the right moment — no dual-dispatch, no double-click risk. The component just calls `mutate()`; the mutation owns the bridge.

### 4. The machine's `submitting` state has no `invoke`
The machine doesn't own the mutation. It just represents the semantic state "we are submitting." React Query's `useMutation` does the actual work. This avoids double-tracking the loading state.

## Suspense Compatibility

`useSuspenseQuery` does not accept `enabled` — it always fetches when the component mounts. This breaks Pattern A's core mechanism of deriving `enabled` from machine state.

**The fix:** Pattern A's step-based conditional rendering already solves this. The machine controls *which component mounts*, so each step component can use `useSuspenseQuery` freely — it only fetches when the machine is in that state.

### Before (non-Suspense, single component with `enabled`)

```tsx
// All queries in one component, gated by `enabled`
const { data: shippingRates } = useQuery({
  queryKey: ['shipping-rates', zip],
  queryFn: () => fetchShippingRates(zip),
  enabled: snapshot.matches('shipping') && !!zip,
})
```

### After (Suspense, query lives in step component)

```tsx
// Checkout.tsx — machine controls which step mounts
if (snapshot.matches('shipping')) {
  return (
    <Suspense fallback={<ShippingFormSkeleton />}>
      <ShippingStep
        zip={context.shippingAddress?.zip ?? ''}
        onSubmit={(address) => send({ type: 'SUBMIT_SHIPPING', address })}
        onBack={() => send({ type: 'BACK' })}
      />
    </Suspense>
  )
}

// ShippingStep.tsx — always fetches, no `enabled` needed
function ShippingStep({ zip, onSubmit, onBack }) {
  const { data: shippingRates } = useSuspenseQuery({
    queryKey: ['shipping-rates', zip],
    queryFn: () => fetchShippingRates(zip),
  })

  // `shippingRates` is always defined here — no loading check needed
  return <ShippingForm rates={shippingRates} onSubmit={onSubmit} onBack={onBack} />
}
```

The machine's conditional rendering (`snapshot.matches('shipping')`) replaces `enabled` — the component only mounts when the machine is in the right state, so the query only runs at the right time. Each `<Suspense>` boundary keeps other steps visible while one step loads.

## Trade-offs

**Gains:**
- Cleanest separation of concerns
- Full React Query features available (background refetch, `select`, `staleTime`)
- Machine is tiny and testable — only flow logic
- No double state problem — server data lives only in the cache

**Costs:**
- The machine can't react to the _content_ of server data (e.g., "if the API returns X, go to state Y")
- For that, you need Pattern B (Orchestrator) where the machine invokes the query
- Mutation callbacks (`onSuccess`/`onError`) are the sole bridge — if you have many mutations, the bridge code adds up
