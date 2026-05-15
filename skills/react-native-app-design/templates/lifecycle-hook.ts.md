# Template — Lifecycle hook

Path: `src/hooks/use<Name>.ts` (shared) or `src/hooks/<feature>/use<Name>.ts`

A non-form custom hook: wires a native subscription, polling, OTA updates, or
derived device data. Read `lifecycle-hooks.md` for the full set of patterns.

```typescript
// use<Name>.ts
// <One line: what this hook subscribes to / manages and why.>

// ---- Imports ----
import { useEffect, useRef, useState, useCallback } from 'react';
import { useAppDispatch, useAppSelector } from '../store/hooks';

// ---- Types ----
/** Value returned by use<Name>. */
interface Use<Name>Result {
    isActive: boolean;
}

// ---- Hook ----
/**
 * use<Name> — <what it does>.
 * <If it manages a subscription or side effect, explain the lifecycle and why.>
 * @returns <what the consumer gets back>.
 */
export const use<Name> = (): Use<Name>Result => {
    const dispatch = useAppDispatch();
    const [isActive, setIsActive] = useState(false);

    // Native subscriptions / timers are held in refs so the cleanup function
    // can tear down the exact same instances.
    const subscriptionRef = useRef<{ remove: () => void } | undefined>(undefined);

    // Ref guard: blocks a duplicate in-flight async operation.
    const isRunning = useRef(false);

    useEffect(() => {
        // --- setup ---
        // …subscribe / start the timer / begin the async work…

        // --- cleanup ---
        // Runs on unmount (and before re-running the effect): tear everything
        // down so nothing fires against an unmounted component.
        return () => {
            subscriptionRef.current?.remove();
        };
    }, [dispatch]);

    return { isActive };
};
```

If the hook returns a single derived value, return that value directly instead
of wrapping it in an object:

```typescript
/** Returns the current user's timezone, or undefined if unknown. */
export const useUserTimezone = (): string | undefined => {
    const userId = useAppSelector((s) => s.auth.user?.id);
    const { data } = useGetProfileQuery(userId ?? skipToken);
    return data?.timezone;
};
```

Checklist:
- Returns an object with named fields, OR a single value directly.
- Native subscriptions held in `useRef`, removed in the effect cleanup.
- Every `useEffect` has a correct, exhaustive dependency array.
- Callbacks in deps or passed to children are wrapped in `useCallback`.
- A `useRef<boolean>` guard prevents duplicate async runs where relevant.
- Errors caught and surfaced/logged, never silently swallowed (R56).
- Doc comment on the hook explaining the lifecycle and the why.
