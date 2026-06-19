# Day 02 — `startTransition` & Scheduler Lane Bitmask Internals: Interruptible Rendering, Priority Inversion, and the Hidden Cost of Concurrent Opt-Ins

> **Date:** 2026-06-19 | **Read time:** 20 minutes | **Difficulty:** Staff / Principal

---

## ⚡ MIN 0–5: ARCHITECTURAL DEEP DIVE — THE LANE MODEL & TRANSITION SCHEDULING

### The Bitmask Under the Hood

React's Concurrent Mode doesn't operate on a simple priority queue. It operates on a **Lane model** — a 31-bit bitmask (constrained by JavaScript's 32-bit signed integer bitwise ops) where each bit or contiguous group of bits represents a priority class.

```typescript
// Simplified from packages/react-reconciler/src/ReactFiberLane.js
export type Lanes = number;
export type Lane  = number;

export const NoLanes: Lanes                  = /*                      */ 0b0000000000000000000000000000000;
export const SyncLane: Lane                  = /*                      */ 0b0000000000000000000000000000001;
export const InputContinuousLane: Lane       = /*                      */ 0b0000000000000000000000000000100;
export const DefaultLane: Lane               = /*                      */ 0b0000000000000000000000000010000;
export const TransitionLane1: Lane           = /*                      */ 0b0000000000000000000000001000000;
// ...TransitionLane2–16 occupy bits 7-22
export const IdleLane: Lane                  = /*                      */ 0b0010000000000000000000000000000;
export const OffscreenLane: Lane             = /*                      */ 0b0100000000000000000000000000000;
```

When you call `startTransition(fn)`, React internally:
1. Sets `ReactCurrentBatchConfig.transition` to a non-null transition object.
2. Inside `fn`, any `setState` calls that fire will have their lanes tagged via `requestTransitionLane()`, which picks the **lowest-numbered available `TransitionLane`** from a round-robin pool of 16.
3. After `fn` returns, `ReactCurrentBatchConfig.transition` is reset.

The key invariant: **`SyncLane` has the lowest bit index (highest priority). `TransitionLane` bits are two orders of magnitude higher in index (lower priority).** Fiber's scheduler always processes the lowest-lane-index pending work first.

### The Scheduler's Render Loop — Where Yielding Actually Happens

The browser has one main thread. React's time-slicer doesn't use `Worker` threads — it uses **cooperative yielding via the message channel loop**:

```
performConcurrentWorkOnRoot
  └── renderRootConcurrent
        └── workLoopConcurrent
              └── while (workInProgress !== null && !shouldYield()) {
                    performUnitOfWork(workInProgress);
                  }
```

`shouldYield()` delegates to `Scheduler.shouldYieldToHost()`, which checks:

```typescript
// packages/scheduler/src/forks/Scheduler.js
function shouldYieldToHost(): boolean {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {        // frameInterval = 5ms
    return false;
  }
  if (enableIsInputPending) {
    if (needsPaint) return true;
    if (timeElapsed < continuousInputInterval) {  // 50ms
      return isInputPending?.(['mousemove', 'pointermove']);
    }
    // ...
  }
  return true;
}
```

**The 5ms slice is not a frame budget** — it's a minimum yield interval. On a 120Hz display (8.33ms frames) the scheduler is deliberately more aggressive about yielding than the frame cadence. The implication: a transition rendering a deeply nested tree will be interrupted and resumed across many message-channel ticks.

### What "Interruption" Means for WIP Tree State

React maintains two fiber trees:
- **`current`** — the committed, stable tree backing the visible DOM.
- **`workInProgress`** — the in-flight draft being reconciled.

When the scheduler yields mid-reconcile, the `workInProgress` tree is left **partially constructed** in memory. If a higher-priority update (e.g., `SyncLane` keystroke) arrives:

1. React **discards** the partial WIP tree (sets it to `null` conceptually by abandoning it at `performConcurrentWorkOnRoot`).
2. Processes the sync update from scratch on a fresh WIP tree derived from `current`.
3. Commits the sync result.
4. Then **restarts** the transition from scratch — not from the checkpoint.

**There are no checkpoints. Transitions are restartable, not resumable.** This means expensive render work in transition paths is inherently speculative and may be thrown away. Heavy components inside `<Suspense>` boundaries that suspend during a transition are the primary retry surface; `use(promise)` in RSC contexts maps onto the same mechanism.

---

## 🛠️ MIN 5–15: PERFORMANCE & CODE REVIEW PROFILE

### The Bug: Three Nested Landmines in a "Concurrent-Safe" Search Component

The following looks like a well-intentioned transition-aware search with loading UI. It has **three distinct architectural defects** that will bite in production.

```tsx
// ❌ FLAWED — Do not ship
import { useState, startTransition, useTransition } from 'react';

interface SearchResult { id: string; title: string; score: number }

function SearchPage() {
  const [query, setQuery]   = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isPending, startT]   = useTransition();

  // Defect 1: dual-state sync over a concurrent boundary
  const handleInput = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value);          // SyncLane — commits immediately
    startT(() => {
      fetch(`/api/search?q=${value}`)    // Defect 2: fetch inside transition callback
        .then(r => r.json())
        .then((data: SearchResult[]) => {
          setResults(data);              // Defect 3: state update outside React context
        });
    });
  };

  return (
    <div>
      <input value={query} onChange={handleInput} />
      {isPending && <span>Loading…</span>}
      <ul>
        {results.map(r => <li key={r.id}>{r.title} ({r.score})</li>)}
      </ul>
    </div>
  );
}
```

#### Defect Dissection

**Defect 1 — Tearing Between `query` and `results`:**
`setQuery` fires in `SyncLane`. The input commits its new value instantly. But `setResults` fires asynchronously inside a `.then()` that is **not inside any React batch context**. The moment `setResults` fires, it creates a new `DefaultLane` update (not a `TransitionLane` update), because the `ReactCurrentBatchConfig.transition` reference has long since been reset by the time the Promise resolves. `isPending` will flip to `false` on the transition commit even though results haven't arrived — leaving a window where `query` shows "react fiber" but `results` is stale from the previous search.

**Defect 2 — Side Effect Inside `startTransition` Callback:**
The `startTransition` callback is **synchronous**. React executes it immediately and only inspects which state updates were called inside it. `fetch()` is a side effect that fires its network request right now regardless of scheduler priority. If the scheduler discards and restarts this transition (because a new keystroke arrived), the `fetch` fires again — no cleanup, no AbortController, a cascade of in-flight requests, and the last-to-resolve wins (classic race condition).

**Defect 3 — `setResults` Is Unguarded Against Component Unmount:**
The `.then()` closure captures `setResults`. If the component unmounts (e.g., navigating away) before the fetch resolves, calling `setResults` on a dead component won't throw in React 18+ (the "can't setState on unmounted component" warning was removed), but it will schedule a reconcile on a root that has already been torn down — wasted CPU and a potential memory leak if the closure holds references to large data.

---

### The Refactor: Production-Hardened Concurrent Search

```tsx
// ✅ PRODUCTION-HARDENED
import {
  useState,
  useTransition,
  useDeferredValue,
  useEffect,
  useRef,
  useCallback,
} from 'react';

interface SearchResult { id: string; title: string; score: number }

// ── Isolated fetch logic with abort support ──────────────────────────────────
async function fetchResults(
  query: string,
  signal: AbortSignal,
): Promise<SearchResult[]> {
  const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, { signal });
  if (!res.ok) throw new Error(`Search failed: ${res.status}`);
  return res.json() as Promise<SearchResult[]>;
}

// ── Component ────────────────────────────────────────────────────────────────
export function SearchPage() {
  const [query,   setQuery]   = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [error,   setError]   = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  // deferredQuery lags behind query by one scheduler cycle — used to
  // debounce the expensive results list render without blocking the input.
  const deferredQuery = useDeferredValue(query);

  // AbortController ref persists across renders; new one per search.
  const abortRef = useRef<AbortController | null>(null);

  const runSearch = useCallback((value: string) => {
    // Cancel any in-flight request before issuing a new one.
    abortRef.current?.abort();
    const controller = new AbortController();
    abortRef.current = controller;

    // ✅ All async work lives OUTSIDE startTransition.
    // startTransition only wraps the state updates that should be
    // deprioritized — i.e., the results commit, not the fetch itself.
    fetchResults(value, controller.signal)
      .then((data) => {
        // ✅ Re-enter React's concurrent scheduler as a transition,
        // so the DOM commit for results is interruptible.
        startTransition(() => {
          setResults(data);
          setError(null);
        });
      })
      .catch((err: unknown) => {
        if ((err as Error).name === 'AbortError') return; // intentional cancel
        startTransition(() => setError((err as Error).message));
      });
  }, [startTransition]);

  const handleChange = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => {
      const value = e.target.value;
      setQuery(value); // SyncLane: input must never lag
      if (value.trim().length > 1) runSearch(value);
      else startTransition(() => setResults([]));
    },
    [runSearch, startTransition],
  );

  // Abort on unmount — prevents setState on dead root.
  useEffect(() => () => { abortRef.current?.abort(); }, []);

  // Stale-UI indicator: when deferredQuery hasn't caught up to query yet,
  // the results list is showing data for the previous query.
  const isStale = deferredQuery !== query;

  return (
    <div>
      <input
        value={query}
        onChange={handleChange}
        aria-busy={isPending}
      />
      {isPending && <span role="status">Fetching…</span>}
      {error && <p role="alert">{error}</p>}
      <ul style={{ opacity: isStale ? 0.5 : 1 }}>
        {results.map((r) => (
          <li key={r.id}>
            {r.title} <small>({r.score.toFixed(2)})</small>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

#### Optimization Differentials

- **`fetch` outside `startTransition`** — The network request fires eagerly in `SyncLane` time. Only the state-update-driven DOM commit is deferred. This is the correct inversion: I/O is never interruptible by React anyway; only the render is.
- **`AbortController` per keystroke** — Each `handleChange` cancels its predecessor. At high typing velocity (>5 WPM on a slow API) this reduces in-flight requests from N to 1. The `AbortError` guard prevents spurious `setError` calls.
- **`useDeferredValue(query)` drives stale-UI opacity** — `deferredQuery` lags one concurrent scheduler cycle behind `query`. The `isStale` boolean lets you visually telegraph "these results are from a previous query" without blocking the input or flipping a separate `isStale` state.
- **`startTransition` wraps only `setResults`/`setError`** — This ensures the results commit is tagged as `TransitionLane`, leaving the input in `SyncLane`. The scheduler will always prioritize new keystrokes over committing search results, eliminating input jank.
- **`useEffect` abort-on-unmount** — Zero-cost guard: the `AbortController` ref is already live; the cleanup is a single method call. No `isMounted` flag, no WeakRef gymnastics.

---

## 🧠 MIN 15–20: SENIOR SYSTEM DIAGNOSTIC

### Scenario: The Phantom `isPending`

Your team ships a real-time collaborative editor. The editor debounces user keystrokes and flushes document state through a `startTransition`. A QA engineer reports: *"After the latest deploy, the saving spinner sometimes shows for 4–6 seconds even when the document hasn't changed."*

You pull the relevant component:

```tsx
// packages/editor/src/AutoSaveProvider.tsx
import { createContext, useContext, useState, useTransition, useEffect } from 'react';

const SaveCtx = createContext({ isSaving: false });

export function AutoSaveProvider({ doc, onSave }: {
  doc: DocumentState;
  onSave: (d: DocumentState) => Promise<void>;
  children: React.ReactNode;
}) {
  const [savedDoc, setSavedDoc]   = useState(doc);
  const [isPending, startTrans]    = useTransition();

  useEffect(() => {
    // debounce: save 800ms after doc changes
    const timer = setTimeout(() => {
      startTrans(async () => {            // ← line 17
        await onSave(doc);
        setSavedDoc(doc);
      });
    }, 800);
    return () => clearTimeout(timer);
  }, [doc, onSave, startTrans]);

  return (
    <SaveCtx.Provider value={{ isSaving: isPending }}>
      {children}
    </SaveCtx.Provider>
  );
}

export const useSaveStatus = () => useContext(SaveCtx);
```

**The diagnostic questions:**

**Q1.** Line 17 passes an `async` function to `startTransition`. What does React do with the Promise returned by that async function, and what is the observable consequence for `isPending`?

**A)** React awaits the Promise and keeps `isPending: true` until it resolves.
**B)** React ignores the Promise return value; `isPending` flips to `false` as soon as the synchronous portion of the callback completes, before `await onSave(doc)` resolves.
**C)** React throws a runtime error: "Transition callback must be synchronous."
**D)** React defers the Promise via the microtask queue and batches its resolution with the next scheduler flush.

**Q2.** The phantom spinner lasts 4–6 seconds. Identify the secondary defect that explains the _duration_ of the bug, not merely its existence.

**Q3.** Provide the minimal correct refactor — no third-party libraries.

---

<details>
<summary><strong>▶ System Diagnostic Answer Key (expand after analysis)</strong></summary>

### A1: Answer is **B**

`startTransition` accepts a **synchronous** callback. React invokes it immediately and collects any `setState` calls made synchronously within it. If you pass an `async` function, React calls it, receives a `Promise` back, and **discards the Promise entirely** — it does not await it, it does not attach any `.then()`, it has no mechanism to do so.

The consequence: React sees _no synchronous state updates_ inside the callback (because `setSavedDoc(doc)` is behind an `await`). The transition is registered but completes trivially — React has nothing to batch — and `isPending` immediately flips back to `false`. The `await onSave(doc)` and `setSavedDoc(doc)` execute later in the microtask queue as a **non-transition, non-batched** `DefaultLane` update.

This is a documented foot-gun. The React team has noted that async support for `startTransition` is staged: React 19's `use client` actions introduce first-class async transition semantics via the `useActionState` / `<form action>` pathway, but `startTransition` itself remains synchronous in its lane-tagging logic.

### A2: The Duration Defect — `onSave` Identity Instability

The `useEffect` dependency array includes `onSave`. If the parent component renders `onSave` as an inline arrow function (common with `useCallback`-less patterns), every parent render creates a new `onSave` reference. This causes `useEffect` to re-run, resetting the 800ms debounce timer on **every parent render** — including the renders triggered by `setSavedDoc` (which is called after the async save resolves and causes a `DefaultLane` commit, which re-renders the parent, which produces a new `onSave` reference...).

The 4–6 second window is the time for this render-timer-reset feedback loop to finally quiesce (when the parent stabilizes). The phantom spinner is not caused by `isPending` being `true` (it's actually `false` the whole time due to A1) — it's caused by a **separate `isSaving` boolean** that the team added as a local workaround (`useState(false)` set to `true` before `startTrans`, set to `false` in the `.then()`), which they forgot to remove, and which correctly stays `true` until the `onSave` Promise resolves — but the debounce reset means `onSave` keeps re-triggering every ~800ms.

### A3: Minimal Correct Refactor

```tsx
import {
  createContext, useContext, useState,
  useEffect, useRef, useCallback,
} from 'react';

const SaveCtx = createContext({ isSaving: false });

export function AutoSaveProvider({ doc, onSave, children }: {
  doc: DocumentState;
  onSave: (d: DocumentState) => Promise<void>;
  children: React.ReactNode;
}) {
  const [isSaving, setIsSaving] = useState(false);
  // Stable ref — never triggers effect re-runs
  const onSaveRef = useRef(onSave);
  useEffect(() => { onSaveRef.current = onSave; }, [onSave]);

  const stableDoc = useRef(doc);
  useEffect(() => { stableDoc.current = doc; }, [doc]);

  useEffect(() => {
    const timer = setTimeout(async () => {
      setIsSaving(true);
      try {
        await onSaveRef.current(stableDoc.current);
      } finally {
        setIsSaving(false);
      }
    }, 800);
    return () => clearTimeout(timer);
  }, [doc]); // Only doc changes reset the debounce; onSave identity is irrelevant

  return (
    <SaveCtx.Provider value={{ isSaving }}>
      {children}
    </SaveCtx.Provider>
  );
}

export const useSaveStatus = () => useContext(SaveCtx);
```

**Key fixes:**
- `onSaveRef` pattern decouples the effect dependency from `onSave` identity churn — the effect only re-fires when `doc` changes, which is the actual intent.
- `isSaving` is now a plain `boolean` state toggled around the `await` — correctly models "network I/O in flight" rather than "React render pending," which are two different things.
- `startTransition` is removed entirely: committing `isSaving` must be synchronous and immediate (SyncLane) so the spinner appears without scheduler delay.

</details>

---

*Day 03 preview: **React Server Components — RSC Payload Wire Format, `__webpack_require__` Client Manifest, and Selective Hydration Priority under `<Suspense>` Boundaries.***
