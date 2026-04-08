# Review TODO

Staff engineer review findings, to be triaged one by one.

## Architectural Concerns

- [x] **`useMachine` vs `useActorRef` contradiction** — Rule 12 says always use `useActorRef` + `useSelector`, but every pattern example uses `useMachine`. Either update examples or add a note explaining the simplification.
- [x] **Missing React 19 / concurrent features guidance** — `useEffectEvent` (Pattern C, E) is React 19+ only. No mention of this. Also no guidance on Suspense + synchronous `send()` interaction in concurrent mode.
- [ ] **Machine lifecycle vs component lifecycle** — No guidance on what happens when a component unmounts during an active `invoke` (e.g., charge already hit payment processor but `FulfillOrder` unmounted).

## Pattern A (Flow Controller)

- [ ] **Remove empty `always` no-op block** — The empty `always` with a comment explaining it does nothing is noise. Remove or move comment outside.
- [ ] **Document that unhandled events are load-bearing** — `onMutate: () => send({ type: 'CONFIRM' })` relies on XState silently dropping events with no handler (e.g., double-click sending CONFIRM while in `submitting`). Worth calling out for XState newcomers.

## Pattern B (Orchestrator)

- [ ] **`ensureQueryData` vs `fetchQuery` table is misleading** — Descriptions are nearly identical. The real difference: `fetchQuery` throws on error (maps to XState `onError`), `ensureQueryData` swallows errors for stale data. For XState invoke actors, `fetchQuery` may be the better default.

## Pattern C (Reactive Bridge)

- [ ] **Initial-mount-with-stale-data correctness gap** — `useEffect` bridge fires on mount with cached data. If query cache has stale data from a previous mount, machine processes a potentially stale `ORDER_UPDATE` immediately.
- [ ] **`useQueryBridge` eslint-disable note** — `useEffectEvent` intentionally omitted from deps, but `react-hooks/exhaustive-deps` will warn. Worth noting for teams.

## Pattern D (Mutation Orchestration)

- [ ] **Show retry-from-failed-step** — Currently RETRY always restarts from `reservingInventory`. The trade-offs section promises "trivial to add" but doesn't show it. This is a key selling point.
- [ ] **`event.error.message` assumes Error shape** — `event.error` is typed as `unknown` in XState v5 `onError`. Pattern B handles this with `toErrorMessage()`, Pattern D doesn't.

## Pattern E (Router Integration)

- [ ] **`machineNavigatedRef` loop-breaker is fragile** — Mutable ref to break event loops is a maintenance hazard. Consider documenting the assumption or suggesting alternatives.
- [ ] **`SYNC_TO_STEP` + `getSnapshot()` race assumption** — Code assumes `getSnapshot()` after `send()` reflects the new state. True for sync events in XState v5 but not a documented contract.

## Golden Rules / General

- [ ] **Add XState v5 + TanStack Query v5 version requirement** — The guide assumes v5 of both libraries (`setup()`, `fromPromise`, etc.) but never states this.
- [ ] **Non-null assertions in examples** — Several examples use `!` assertions (`context.selectedProductId!`). Show type-safe alternatives (discriminated unions, context narrowing via `snapshot.matches()`).
- [ ] **No mention of DevTools** — `@xstate/inspect` and TanStack Query DevTools are both excellent. A section on how they complement each other during debugging would add value.
- [ ] **Missing visual data flow diagram** — The decision tree is textual. A diagram showing what happens on a user action (machine transition → query enabled/disabled → cache hit/miss) would make the mental model concrete.
- [ ] **Performance at scale** — No guidance on what happens with many queries gated by `snapshot.matches()`. Connect this to Rule 12 (`useSelector` prevents cascading re-renders).
