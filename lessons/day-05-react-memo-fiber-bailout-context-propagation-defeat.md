# Day 05 — `React.memo` Internals: Fiber Bail-out in `beginWork`, the `REACT_MEMO_TYPE` Chain & Context Propagation Silently Defeating Memoization

**Date:** 2026-06-21 | **Read Time:** ~20 min | **Focus:** Fiber architecture, `SimpleMemoComponent` vs `MemoComponent`, context `propagateContextChange`, concurrent render tearing

---

## ⚡ MIN 0–5: ARCHITECTURAL DEEP DIVE — The `beginWork` Bail-out Path & Why `memo()` Is Not a Shield

### The `SimpleMemoComponent` vs. `MemoComponent` Fork

When `performUnitOfWork` processes a fiber, it dispatches to `beginWork(current, workInProgress, renderLanes)`. For `React.memo`-wrapped components, the fiber's `tag` resolves to one of two values — and the distinction is non-trivial:

- **`SimpleMemoComponent` (tag = 15):** Assigned when `React.memo(Fn)` is called with *no* custom `areEqual` argument and the inner component carries no `defaultProps`. The bail-out fast-path calls `shallowEqual(prevProps, nextProps)` directly inside `updateSimpleMemoComponent`. This path is cheaper — it avoids an additional fiber traversal to unwrap the memo shell.
- **`MemoComponent` (tag = 14):** Assigned when a custom `areEqual` is present or `defaultProps` exist. `updateMemoComponent` calls `areEqual(prevProps, nextProps)` (note the inverted return convention — `true` means "equal, skip"). On success it falls through to the same cloned-subtree path as `SimpleMemoComponent`.

`shallowEqual` walks own enumerable keys of both prop objects and calls `Object.is` per key — **one level deep only**. Nested objects, arrays, or functions that are reconstructed inline always fail equality even with identical runtime values.

### The `includeSomeLanes` Gate That Runs Before `shallowEqual`

Documentation never surfaces this: **memo bail-out is gated by lane intersection before shallow equality even executes.**

Inside `beginWork`:

```
if (includesSomeLane(workInProgress.lanes, renderLanes)) {
  // Fiber has pending work — skip bail-out entirely, re-render unconditionally
}
```

If the `memo`-wrapped fiber holds pending update lanes that overlap the current render's `renderLanes` bitmask (e.g., a `useState` setter was called inside it from an unrelated concurrent update), the bail-out is bypassed regardless of whether props are identical. The component re-renders. This is the mechanism behind the surprising behavior where a `memo` component re-renders even when its parent passes the exact same props reference.

### Context Propagation: The `didReceiveUpdate` Backdoor

`React.createContext` maintains `_currentValue` on the context object. When a `Provider` renders with a new value, React evaluates `Object.is(oldValue, newValue)`. On inequality, it calls `propagateContextChange(workInProgress, context, renderLanes)`. This function:

1. Traverses descendant fibers depth-first via the `child` / `sibling` fiber tree pointers.
2. For every fiber whose `contextDependencies` linked list contains a node matching this context, it calls `scheduleContextWorkOnParentPath` to bubble lane work up to the root, **and** calls `markWorkInProgressReceivedUpdate()`.

`markWorkInProgressReceivedUpdate()` sets the global `didReceiveUpdate = true`.

In `beginWork`, after the `includeSomeLanes` check, there is a second bail-out gate:

```
if (!didReceiveUpdate) {
  // Safe to bail out — return existing fiber subtree
  return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
}
```

Once `propagateContextChange` has set `didReceiveUpdate = true` on a fiber, **this second gate is also bypassed**. The fiber re-renders unconditionally. The `tag` may still be `SimpleMemoComponent`. The `shallowEqual` check may have passed. None of it matters — the global flag was already set by the context propagation walk before `beginWork` ran the equality checks.

This is the precise architectural reason `React.memo` cannot protect against context-driven re-renders: the invalidation happens at the fiber-traversal level during the Provider's render, before the memoized component's `beginWork` gets a turn in the work loop. By the time `SimpleMemoComponent`'s bail-out logic runs, it's already been told to re-render.

---

## 🛠️ MIN 5–15: PERFORMANCE & CODE REVIEW PROFILE

### The Bug: A "Memoized" List That Re-renders on Every Keystroke

This pattern is endemic in production React codebases. It compiles, it passes TypeScript, it looks optimized — and it silently degrades on every parent state change.

```tsx
// ❌ FLAWED — appears memoized, silently re-renders on every keystroke

import React, { memo, useCallback, useContext, useState } from 'react';

const ThemeContext = React.createContext<{ theme: string; locale: string }>({
  theme: 'light',
  locale: 'en',
});

interface ListItemProps {
  id: string;
  label: string;
  onSelect: (id: string) => void;
}

const ListItem = memo(({ id, label, onSelect }: ListItemProps) => {
  console.log(`Rendering: ${id}`); // fires on EVERY parent keystroke
  const { theme } = useContext(ThemeContext);

  return (
    <div className={`item item--${theme}`} onClick={() => onSelect(id)}>
      {label}
    </div>
  );
});

export function SearchableList() {
  const [query, setQuery] = useState('');
  const [selected, setSelected] = useState<string | null>(null);

  // ❌ Flaw 1: New object literal on every render → breaks context identity
  const themeCtx = { theme: 'dark', locale: 'en' };

  const handleSelect = useCallback((id: string) => {
    setSelected(id);
  }, []); // stable ref — NOT the problem

  // ❌ Flaw 2: Inline array → new identity → shallowEqual fails on item objects
  const items = [
    { id: 'a', label: 'Alpha' },
    { id: 'b', label: 'Beta' },
    { id: 'c', label: 'Gamma' },
  ];

  return (
    <ThemeContext.Provider value={themeCtx}>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {items.map(item => (
        <ListItem key={item.id} {...item} onSelect={handleSelect} />
      ))}
    </ThemeContext.Provider>
  );
}
```

### Flaw Taxonomy

**Flaw 1 — Context object identity destroyed every render.**
`const themeCtx = { theme: 'dark', locale: 'en' }` is a new object literal on every `SearchableList` render. `ThemeContext.Provider` tests `Object.is(prevValue, nextValue)` → `false` on every keystroke → `propagateContextChange` fires → all `ListItem` fibers receive `didReceiveUpdate = true` via the traversal walk → `memo` bail-out never executes. This is the primary flaw.

**Flaw 2 — `items` array and spread objects reconstructed inline.**
Even if context were stable, spreading `{...item}` produces new prop object identities every render. `shallowEqual` walks keys — `id` and `label` are primitives (pass), `onSelect` is stable (pass) — but the spread produces a new top-level props object. Actually `shallowEqual` receives the top-level props and compares values, not the container — so this particular spread is harmless *at the props level* since the values are primitives. The real problem is the `items` array causing the `map` to re-run and call `React.createElement` for all three items on each keystroke, creating new element descriptors. Reconciliation cost is O(n) even when no DOM mutations occur.

**Flaw 3 — `locale` in context shape consumed by components that never use it.**
`ListItem` calls `useContext(ThemeContext)` and destructures only `theme`. It still registers a `contextDependencies` entry for the *entire* `ThemeContext`. Any `locale` change (irrelevant to this component) triggers `propagateContextChange` into all `ListItem` fibers. Context shape should be split by update frequency.

**Flaw 4 — No `displayName` on memo wrapper.**
React DevTools Profiler and the flame graph display `Anonymous` instead of `ListItem`, making performance triage via Profiler recordings impossible at scale.

---

### The Refactor: Context Splitting + Memoized Provider Value + Stable Primitives

```tsx
// ✅ PRODUCTION-HARDENED REFACTOR

import React, {
  createContext,
  memo,
  useCallback,
  useContext,
  useMemo,
  useState,
  type ReactNode,
} from 'react';

// ── 1. Context split by update cadence ──────────────────────────────────────
// High-churn context (locale changes on user action) is isolated from
// low-churn context (theme changes rarely). Subscribers pay propagation cost
// only for the context they actually depend on.

interface ThemeContextValue { theme: 'light' | 'dark' }
interface LocaleContextValue { locale: string }

const ThemeContext = createContext<ThemeContextValue>({ theme: 'light' });
const LocaleContext = createContext<LocaleContextValue>({ locale: 'en' });

// ── 2. Module-scope constant — zero allocation per render ────────────────────
const ITEMS = [
  { id: 'a', label: 'Alpha' },
  { id: 'b', label: 'Beta' },
  { id: 'c', label: 'Gamma' },
] as const;

type ItemId = (typeof ITEMS)[number]['id'];

// ── 3. ListItem: consumes ONLY ThemeContext, derives no dynamic objects ──────
interface ListItemProps {
  id: string;
  label: string;
  onSelect: (id: string) => void;
  isSelected: boolean; // primitive boolean — shallowEqual resolves in O(1)
}

const ListItem = memo(
  ({ id, label, onSelect, isSelected }: ListItemProps) => {
    const { theme } = useContext(ThemeContext);
    return (
      <div
        className={`item item--${theme}${isSelected ? ' item--selected' : ''}`}
        onClick={() => onSelect(id)}
      >
        {label}
      </div>
    );
  },
  // Explicit equality contract — insulates against accidental prop additions
  // that carry non-primitive values later in the component's lifecycle.
  (prev, next) =>
    prev.id === next.id &&
    prev.label === next.label &&
    prev.onSelect === next.onSelect &&
    prev.isSelected === next.isSelected,
);
ListItem.displayName = 'ListItem';

// ── 4. Providers: useMemo stabilizes context value identity ─────────────────
function AppProviders({ children }: { children: ReactNode }) {
  // In production these deps come from store/config slices.
  // useMemo ensures the object reference is stable across re-renders
  // of AppProviders itself, so Object.is(prev, next) === true in the Provider.
  const themeValue = useMemo<ThemeContextValue>(() => ({ theme: 'dark' }), []);
  const localeValue = useMemo<LocaleContextValue>(() => ({ locale: 'en' }), []);

  return (
    <ThemeContext.Provider value={themeValue}>
      <LocaleContext.Provider value={localeValue}>
        {children}
      </LocaleContext.Provider>
    </ThemeContext.Provider>
  );
}

// ── 5. SearchableList: re-renders only on its own state changes ──────────────
export function SearchableList() {
  const [query, setQuery] = useState('');
  const [selected, setSelected] = useState<ItemId | null>(null);

  const handleSelect = useCallback((id: string) => {
    setSelected(id as ItemId);
  }, []);

  // useMemo: filtered array recalculated only when query changes
  const filtered = useMemo(
    () => ITEMS.filter(i => i.label.toLowerCase().includes(query.toLowerCase())),
    [query],
  );

  return (
    <AppProviders>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {filtered.map(item => (
        <ListItem
          key={item.id}
          id={item.id}
          label={item.label}
          onSelect={handleSelect}
          isSelected={selected === item.id}
        />
      ))}
    </AppProviders>
  );
}
```

### Key Optimization Differentials

- **Context split by update frequency** — `ThemeContext` and `LocaleContext` are decoupled. A `locale` mutation no longer fires `propagateContextChange` into fibers whose `contextDependencies` list only contains `ThemeContext`. Each `useContext` call appends to a distinct dependency chain.
- **`useMemo` on provider values** — `themeValue` and `localeValue` maintain object identity across parent re-renders. `Object.is(prev, next)` returns `true` in the Provider → `propagateContextChange` is never invoked → `didReceiveUpdate` stays `false` in all `ListItem` fibers → `memo` bail-out executes cleanly.
- **`ITEMS` hoisted to module scope** — zero allocation; `filtered` via `useMemo` produces a new array only when `query` changes, not every render cycle. `React.createElement` is not called for unchanged items.
- **`memo` with custom `areEqual`** — overrides `shallowEqual` with an explicit primitive-key contract; future prop additions that carry non-primitive values (e.g., an `onClick` object config) won't silently break memoization without a conscious equality update.
- **`isSelected` as a primitive boolean** — avoids threading `selected: string | null` into each `ListItem` and calling string comparison inside `shallowEqual`. The derived boolean comparison is `Object.is(false, true)` — a single JIT-inlined branch.
- **`displayName`** — the fiber label is `ListItem` in React DevTools Profiler flame graphs, enabling accurate render attribution during performance profiling sessions. Without it, `memo(Component)` appears as `Anonymous` and collapses multiple components into one unreadable entry.

---

## 🧠 MIN 15–20: SENIOR SYSTEM DIAGNOSTIC

### Scenario: "Stable" Subscription Producing Phantom Stale Reads Under Concurrent Mode

Your team ships a custom micro-state manager wrapping `useSyncExternalStore`. A senior engineer flags the implementation below as correct. After 3 weeks in production, you receive reports of occasional phantom stale reads — the UI displays a value that was already superseded by a subsequent store update, but *only* under heavy user interaction: rapid tab-switching, multiple Suspense boundaries resolving simultaneously, React DevTools' "Highlight updates" showing unexpected green flashes on committed fibers.

Study the store contract below and identify the structural flaw:

```tsx
// store.ts
type Listener = () => void;

class DataStore<T> {
  private state: T;
  private listeners = new Set<Listener>();

  constructor(initial: T) {
    this.state = initial;
  }

  getSnapshot(): T {
    return this.state; // returns the mutable internal reference directly
  }

  setState(updater: (prev: T) => T): void {
    this.state = updater(this.state);
    this.listeners.forEach(l => l());
  }

  subscribe(listener: Listener): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }
}

export const userStore = new DataStore({ name: 'Alice', role: 'admin' });

// UserBadge.tsx
import { useSyncExternalStore } from 'react';
import { userStore } from './store';

export function UserBadge() {
  const user = useSyncExternalStore(
    userStore.subscribe,      // Line A
    userStore.getSnapshot,    // Line B
    userStore.getSnapshot,    // Line C — server snapshot
  );

  return <span>{user.name} ({user.role})</span>;
}
```

**The Question (multi-part):**

**A.** Identify the exact line(s) responsible and name the specific V8/React mechanism that surfaces the failure under concurrent rendering.

**B.** The senior engineer argues: *"getSnapshot is called synchronously during render, and useSyncExternalStore guarantees consistency by comparing snapshots between getSnapshot calls — so there can't be a tear."* Where precisely is this reasoning correct, and where does it break down?

**C.** Rewrite the minimum number of lines to make the store safe for React 18 Concurrent Mode and Strict Mode double-invocation.

---

<details>
<summary><strong>System Diagnostic Answer Key</strong> — expand after forming your diagnosis</summary>

### Answer A — Unbound Method References + Snapshot Identity Contract Violation

**Lines A, B, and C** all pass **unbound class method references** as bare function values.

```ts
userStore.subscribe    // `this` is unbound when React calls it
userStore.getSnapshot  // `this` is unbound when React calls it
```

In JavaScript strict mode (which React's Concurrent Mode render functions run in), calling an unbound method causes `this` to be `undefined`. This particular `getSnapshot` implementation happens to work *incidentally* because V8 can often resolve `this.state` via the instance pointer even when `this` is not explicitly bound — but the behavior is undefined and breaks under certain engine optimization tiers.

The **primary** race condition is more subtle: `getSnapshot` returns `this.state`, which is the **same mutable reference** that `setState` replaces. The `useSyncExternalStore` contract requires that `getSnapshot` return a **snapshot** — a value that is:

1. **Referentially stable** across multiple synchronous calls during the same render (same render invocation should return the same reference)
2. **Not mutated after return** (the returned object must remain unchanged)

`setState` does `this.state = updater(this.state)`. This *replaces* the `state` reference with a new object — which satisfies condition 2 for the new value. But here's the concurrent failure path:

1. React begins rendering `UserBadge` in a deferred concurrent lane.
2. `getSnapshot()` is called — returns reference `A` (e.g., `{ name: 'Alice', role: 'admin' }`).
3. A Suspense boundary triggers; React suspends the render and parks the work-in-progress fiber.
4. An external event fires `userStore.setState(...)` — `this.state` is now reference `B` (`{ name: 'Bob', role: 'user' }`). Listeners are notified.
5. React resumes the deferred render. `useSyncExternalStore` calls `getSnapshot()` again to check for consistency — now gets reference `B`.
6. `Object.is(A, B)` → `false`. React detects a "tear" and **forces a synchronous re-render** to flush the stale concurrent work and re-render with `B`.
7. The synchronous re-render is correct — but the cost is that React has abandoned the concurrent render in flight, returned to synchronous mode, and blocked the main thread. Under heavy interaction, this cascade repeats.

The "phantom stale read" is the fiber output from step 2's render that committed before step 6 ran — possible when the Suspense resolution sequence commits the shell before the re-render from step 6 can replace it, producing a visible intermediate state.

### Answer B — Where the Reasoning Breaks Down

The senior engineer is **correct** that `useSyncExternalStore` detects tears by comparing snapshots. This is the mechanism from [React RFC #147](https://github.com/reactwg/react-18/discussions/86) — React calls `getSnapshot` during render, stores the return value, then calls it again in `commitRoot` via `checkIfSnapshotChanged`. If the values differ, React schedules a re-render.

The reasoning **fails** because:

1. The guarantee is only valid within a **single synchronous render pass**. Concurrent renders are interruptible. Between a suspended render's first `getSnapshot()` call and the commit-phase `checkIfSnapshotChanged()` call, the store can mutate. The re-render from the consistency check is synchronous but occurs *after* the visual commit — which is the window in which the stale state becomes visible.
2. Returning `this.state` (a mutable class field) means there is no stable snapshot contract. Even if values are equivalent, if `setState` has run between render and commit, `Object.is(snapshot, currentValue)` returns `false` even though no logical change occurred to the *values the component cares about*. This triggers spurious synchronous re-renders that defeat the purpose of the concurrent scheduler.

### Answer C — Minimum-Diff Fix

```ts
// store.ts — bind in constructor + guard setState against no-op mutations

class DataStore<T> {
  private state: T;
  private listeners = new Set<Listener>();

  constructor(initial: T) {
    this.state = initial;
    // Bind so React can call these as bare function references safely
    this.subscribe = this.subscribe.bind(this);
    this.getSnapshot = this.getSnapshot.bind(this);
  }

  getSnapshot(): T {
    // Contract: returns this.state which is replaced (not mutated) by setState.
    // Identity changes only on real logical state changes — safe for Object.is.
    return this.state;
  }

  setState(updater: (prev: T) => T): void {
    const next = updater(this.state);
    // Guard: skip listener notification if the updater returned the same reference.
    // Prevents spurious snapshot invalidations on no-op updates.
    if (!Object.is(this.state, next)) {
      this.state = next;
      this.listeners.forEach(l => l());
    }
  }

  subscribe(listener: Listener): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }
}
```

```tsx
// UserBadge.tsx — stable subscribe/getSnapshot references via useCallback
import { useCallback } from 'react';
import { useSyncExternalStore } from 'react';
import { userStore } from './store';

export function UserBadge() {
  // Alternatively: bind in constructor (done above) and pass directly.
  // useCallback here is belt-and-suspenders if the store constructor binding
  // is ever removed — and it makes the dependency contract explicit.
  const subscribe = useCallback(
    (l: Listener) => userStore.subscribe(l),
    [], // userStore is module-scope singleton — stable across all renders
  );
  const getSnapshot = useCallback(() => userStore.getSnapshot(), []);

  const user = useSyncExternalStore(subscribe, getSnapshot, getSnapshot);

  return <span>{user.name} ({user.role})</span>;
}
```

**Contracts enforced by this fix:**
- `subscribe` and `getSnapshot` are stable references — React's internal equality check on the subscribe function between renders won't force a re-subscription.
- `setState` is guarded by `Object.is` — no-op updates don't fire listeners, so `checkIfSnapshotChanged` always returns `false` when there's no real change, eliminating spurious synchronous re-renders.
- Constructor binding satisfies the `useSyncExternalStore` API contract that `subscribe` must always unsubscribe via the returned cleanup function — this was already correct, but binding ensures `this.listeners` is resolved correctly in all engine optimization tiers.

</details>

---

*Next session target: `useTransition` + `useDeferredValue` cooperative scheduling — priority inheritance across Scheduler lanes and why `useDeferredValue` can produce two simultaneous in-flight renders of the same subtree in the Fiber work loop.*
