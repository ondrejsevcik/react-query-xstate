# Review Items

## Code-Level Bugs (fix first)

- [x] **1. Pattern A race condition in `onConfirm`** — moved `CONFIRM` to `onMutate` callback so mutation lifecycle is the sole bridge. (`patterns/pattern-a-flow-controller.md`)
- [x] **2. Pattern C `useQueryBridge` stale closure** — replaced with `useEffectEvent` (React 19+) so inline callbacks work without `useCallback`. (`patterns/pattern-c-reactive-bridge.md`)
- [x] **3. Pattern E navigation bridge timing** — `location.pathname` captured in subscription closure, torn down/recreated on every route change. Fix: use ref for pathname inside subscription. (`patterns/pattern-e-router-integration.md`)
- [x] **4. `ensureQueryData` vs `fetchQuery` table is incorrect** — both read cache and respect staleTime. Fix the table in golden rules. (`patterns/golden-rules.md`)
- [x] **5. Pattern A trade-offs typo** — fixed to "Pattern B (Orchestrator)". (`patterns/pattern-a-flow-controller.md`)

## Missing Guidance (add new sections)

- [x] **6. Suspense integration** — `useSuspenseQuery` doesn't accept `enabled`, breaking Pattern A's core mechanism. Add compatibility notes.
- [ ] **7. `useMachine` vs `useActorRef` + `useSelector` performance** — `useMachine` re-renders on every state change. Add perf guidance.
- [ ] **8. Error boundary placement** — no guidance on where error boundaries go relative to machine providers.
- [ ] **9. Replace `setTimeout` in tests with `waitFor`** — fragile in CI. Use XState's `waitFor` or subscription-based approach. (`patterns/pattern-b-orchestrator.md`, `patterns/pattern-d-mutation-orchestration.md`)
- [ ] **10. When NOT to use XState** — linear multi-step forms with no branching don't need a state machine.

## Minor Type Safety Nits

- [ ] **11. `event.error.message` assumes Error instance** — XState v5 types `error` as `unknown`. Needs type guard. (`patterns/pattern-b-orchestrator.md`)
- [ ] **12. `snapshot.value as string` cast** — hides bug for hierarchical/parallel states. (`patterns/pattern-e-router-integration.md`)
- [ ] **13. `AnyActorRef` in `useQueryBridge`** — loses event type safety. Use generic constrained to machine events. (`patterns/pattern-c-reactive-bridge.md`)
