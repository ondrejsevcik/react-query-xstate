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
      // Allow proceeding when product is selected
      always: {
        // This is intentionally NOT an always transition —
        // we let the user explicitly proceed
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
import { useMachine } from '@xstate/react'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { checkoutMachine } from '../machines/checkout'

export function Checkout() {
  const [snapshot, send] = useMachine(checkoutMachine)
  const queryClient = useQueryClient()
  const { context } = snapshot

  // --- React Query handles all server data ---

  // Products list — always available
  const { data: products } = useQuery({
    queryKey: ['products'],
    queryFn: fetchProducts,
  })

  // Selected product details — enabled when a product is selected
  const { data: product } = useQuery({
    queryKey: ['product', context.selectedProductId],
    queryFn: () => fetchProduct(context.selectedProductId!),
    enabled: !!context.selectedProductId,
  })

  // Shipping rates — enabled when we're on the shipping step
  const { data: shippingRates } = useQuery({
    queryKey: ['shipping-rates', context.shippingAddress?.zip],
    queryFn: () => fetchShippingRates(context.shippingAddress!.zip),
    enabled: snapshot.matches('shipping') && !!context.shippingAddress?.zip,
  })

  // Payment methods — prefetch when user reaches payment step
  const { data: paymentMethods } = useQuery({
    queryKey: ['payment-methods'],
    queryFn: fetchPaymentMethods,
    enabled: snapshot.matches('payment') || snapshot.matches('reviewing'),
  })

  // Place order mutation — callbacks are the sole bridge to the machine.
  // onMutate fires synchronously before the request, so the machine
  // transitions to 'submitting' at exactly the right moment.
  const placeOrder = useMutation({
    mutationFn: (order: OrderPayload) => submitOrder(order),
    onMutate: () => send({ type: 'CONFIRM' }),
    onSuccess: () => {
      send({ type: 'ORDER_PLACED' })
      queryClient.invalidateQueries({ queryKey: ['orders'] })
    },
    onError: () => {
      send({ type: 'ORDER_FAILED' })
    },
  })

  // --- Machine state determines which UI to show ---

  if (snapshot.matches('selectingProduct')) {
    return (
      <ProductSelector
        products={products}
        selectedId={context.selectedProductId}
        quantity={context.quantity}
        onSelect={(productId) => send({ type: 'SELECT_PRODUCT', productId })}
        onQuantityChange={(quantity) => send({ type: 'SET_QUANTITY', quantity })}
        onNext={(address) => send({ type: 'SUBMIT_SHIPPING', address })}
      />
    )
  }

  if (snapshot.matches('shipping')) {
    return (
      <ShippingForm
        rates={shippingRates}
        onSubmit={(address) => send({ type: 'SUBMIT_SHIPPING', address })}
        onBack={() => send({ type: 'BACK' })}
      />
    )
  }

  if (snapshot.matches('payment')) {
    return (
      <PaymentSelector
        methods={paymentMethods}
        onSelect={(methodId) => send({ type: 'SELECT_PAYMENT', methodId })}
        onBack={() => send({ type: 'BACK' })}
      />
    )
  }

  if (snapshot.matches('reviewing')) {
    return (
      <OrderReview
        product={product}
        quantity={context.quantity}
        address={context.shippingAddress}
        onConfirm={() =>
          placeOrder.mutate({
            productId: context.selectedProductId!,
            quantity: context.quantity,
            shippingAddress: context.shippingAddress!,
            paymentMethodId: context.paymentMethodId!,
          })
        }
        onBack={() => send({ type: 'BACK' })}
      />
    )
  }

  if (snapshot.matches('submitting')) {
    return <SubmittingOrder />
  }

  if (snapshot.matches('complete')) {
    return (
      <OrderComplete
        onNewOrder={() => send({ type: 'RESET' })}
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
enabled: snapshot.matches('shipping') && !!context.shippingAddress?.zip
```
This is the bridge — machine state controls when queries run, but React Query manages the fetching lifecycle.

### 3. Mutation callbacks are the sole bridge to the machine
```tsx
onMutate: () => send({ type: 'CONFIRM' })      // → submitting (sync, before request)
onSuccess: () => send({ type: 'ORDER_PLACED' }) // → complete
onError: () => send({ type: 'ORDER_FAILED' })   // → reviewing (retry)
```
The mutation's lifecycle callbacks drive all machine transitions. `onMutate` fires synchronously before the request, so the machine enters `submitting` at exactly the right moment — no dual-dispatch, no double-click risk. The component just calls `mutate()`; the mutation owns the bridge.

### 4. The machine's `submitting` state has no `invoke`
The machine doesn't own the mutation. It just represents the semantic state "we are submitting." React Query's `useMutation` does the actual work. This avoids double-tracking the loading state.

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
