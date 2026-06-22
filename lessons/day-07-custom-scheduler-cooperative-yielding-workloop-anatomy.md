# Day 07 — Custom Scheduler Internals: Cooperative Yielding, the `workLoop`, and Building a Frame-Budget-Aware Task Queue

> **20-minute senior engineering digest** | 2026-06-22
> **Prerequisite fluency:** React Fiber reconciler internals, `MessageChannel` event loop mechanics, browser rendering pipeline, Lane model basics.

---

## ⚡ MIN 0–5: ARCHITECTURAL DEEP DIVE — React's `scheduler` Package and the Cooperative Yielding Contract

### The Core Illusion: Interruptible Rendering

React's concurrent mode is not magic — it's a disciplined application of **cooperative multitasking on a single-threaded runtime**. The mechanism lives entirely in the `scheduler` package (`packages/scheduler/src/forks/Scheduler.js` in the React source), and understanding its internals exposes exactly why `startTransition`, `useDeferredValue`, and Suspense boundaries behave the way they do under load.

#### The `workLoop` and the 5 ms Frame Budget

React's scheduler does not use `requestIdleCallback` in production (too coarse-grained, inconsistent across browsers). Instead it uses a `MessageChannel` trick:

```typescript
// Simplified from React's actual scheduler source
const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = performWorkUntilDeadline;

function schedulePerformWorkUntilDeadline() {
  port.postMessage(null); // posts a macrotask — runs AFTER paint
}
```

The `postMessage` call queues a **macrotask**, which means it runs after the browser has had a chance to paint, handle input, and run microtasks. This is intentionally chosen over `setTimeout(fn, 0)` because `setTimeout` has a 1 ms minimum clamp that compounds — `MessageChannel` does not.

#### Frame Budget Slicing

Every time the scheduler wakes up via that `onmessage` callback, it records the current time and grants itself a **5 ms work window** (the `frameYieldMs` constant — adjustable via `unstable_Profiling`):

```typescript
let startTime = getCurrentTime();
let deadline = startTime + frameYieldMs; // frameYieldMs = 5

function shouldYieldToHost(): boolean {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameYieldMs) return false;
  // Also yields if there are pending input events (navigator.scheduling.isInputPending)
  if (navigator.scheduling?.isInputPending?.()) return true;
  return timeElapsed >= continuousYieldMs; // 50ms hard cap
}
```

The `navigator.scheduling.isInputPending()` call is the **critical insight** most engineers miss. React proactively yields to the browser even before the 5 ms window expires if there are pending pointer or keyboard events in the queue. This is what makes React 18+ feel responsive under continuous typing — it's not just time-slicing, it's **input-priority yielding**.

#### Task Priority Levels and the Min-Heap Queue

The scheduler maintains a **binary min-heap** (`SchedulerMinHeap.js`) keyed by `expirationTime`. Priority levels map directly to timeout offsets:

| Priority | Timeout | Use Case |
|---|---|---|
| `ImmediatePriority` | -1 (sync) | `flushSync`, error boundaries |
| `UserBlockingPriority` | 250 ms | Discrete events (click, keydown) |
| `NormalPriority` | 5000 ms | Default render work |
| `LowPriority` | 10000 ms | `startTransition` transitions |
| `IdlePriority` | `MAX_INT` | Prefetching, analytics |

When you call `startTransition`, React schedules the update at `LowPriority`. A subsequent `UserBlockingPriority` task (a keystroke) will **preempt** the in-progress transition work — the scheduler pops its min-heap, sees a task with lower expiration time, and the `workLoop` yields mid-tree traversal. React checkpoints the interrupted work via the `wipRoot` pointer and resumes from the nearest committed boundary.

#### Why This Architecture Matters for Custom Schedulers

If you're building a custom micro-state manager or an off-thread computation pipeline (e.g., using `SharedArrayBuffer`-backed workers with `Atomics.waitAsync`), you **must** integrate with React's scheduler rather than fight it. Calling `setState` from a `worker.onmessage` handler flushes through the default event batching path — but if you need **priority-aware scheduling** of computed state, you reach for `scheduler.scheduleCallback(priority, callback)` directly.

---

## 🛠️ MIN 5–15: PERFORMANCE & CODE REVIEW PROFILE

### The Scenario: A "Real-Time" Dashboard with Custom Subscription and Layout Thrashing

This pattern appears in production dashboards constantly — a component subscribing to a high-frequency data source, with a derived layout computation that "seems fine" in dev but destroys frame rate in prod.

#### ❌ Flawed Implementation

```tsx
// BAD: LiveMetricsPanel.tsx — multiple architectural violations
import { useEffect, useState, useRef } from 'react';
import type { MetricSnapshot } from '@/types';

interface Props {
  source: EventEmitter<MetricSnapshot>;
  columnCount: number;
}

export function LiveMetricsPanel({ source, columnCount }: Props) {
  const [metrics, setMetrics] = useState<MetricSnapshot[]>([]);
  const containerRef = useRef<HTMLDivElement>(null);
  const [columnWidth, setColumnWidth] = useState(0);

  // FLAW 1: Unthrottled subscription — every 16ms event triggers a full re-render
  useEffect(() => {
    const handler = (snap: MetricSnapshot) => {
      setMetrics(prev => [...prev, snap]); // FLAW 2: unbounded array growth
    };
    source.on('data', handler);
    return () => source.off('data', handler);
  }, [source]); // FLAW 3: source identity not stable — re-subscribes on every parent render

  // FLAW 4: Forces synchronous layout read immediately after state update
  // — reads offsetWidth DURING the commit phase before paint,
  // then writes back via setState, causing a second layout/paint cycle
  useEffect(() => {
    if (containerRef.current) {
      const width = containerRef.current.offsetWidth; // FORCED REFLOW
      setColumnWidth(Math.floor(width / columnCount));
    }
  }); // FLAW 5: no dependency array — runs after EVERY render

  return (
    <div ref={containerRef} className="metrics-grid">
      {metrics.map((m, i) => (
        // FLAW 6: index as key on a growing array — React can't reconcile efficiently
        <MetricCard key={i} data={m} width={columnWidth} />
      ))}
    </div>
  );
}
```

#### Root-Cause Analysis

- **Flaw 1 + 2:** Unthrottled append to an unbounded array. At 60 Hz data, this queues 60 renders/sec, each allocating a new array. The GC pressure alone tanks the frame budget.
- **Flaw 3:** `source` is constructed inline (`new EventEmitter()` in parent render) or not stabilized — `useEffect`'s dependency re-fires on every parent render, causing subscribe/unsubscribe churn.
- **Flaw 4 + 5:** Reading `offsetWidth` is a **forced synchronous layout** (FSL). Doing this inside a `useEffect` with no deps means the sequence is: render → commit → DOM paint → effect reads layout → setState → second render → second commit → second paint. This is the classic **layout thrash** pattern — React's batching cannot save you because the FSL forces the browser to recalculate style before the read can return.
- **Flaw 6:** Index keys on a growing list cause React to diff every item from the insertion point forward on every update. O(n) reconciliation per append.

---

#### ✅ Optimized, Production-Hardened Refactor

```tsx
// GOOD: LiveMetricsPanel.tsx — corrected and hardened
import {
  useCallback,
  useEffect,
  useLayoutEffect,
  useRef,
  useSyncExternalStore,
  useState,
  startTransition,
} from 'react';
import type { MetricSnapshot } from '@/types';

interface Props {
  source: EventEmitter<MetricSnapshot>;
  columnCount: number;
  windowSize?: number; // sliding window cap — prevents unbounded growth
}

// ── Stable store outside component ──────────────────────────────────────────
// Moving the subscription store outside eliminates closure identity issues
// and makes it shareable if multiple panels consume the same source.
function createMetricsStore(source: EventEmitter<MetricSnapshot>, windowSize: number) {
  let snapshots: readonly MetricSnapshot[] = [];
  const listeners = new Set<() => void>();

  const handler = (snap: MetricSnapshot) => {
    // Sliding window: O(1) amortized via structural share + slice
    snapshots = snapshots.length >= windowSize
      ? [...snapshots.slice(-(windowSize - 1)), snap]
      : [...snapshots, snap];
    listeners.forEach(l => l());
  };

  source.on('data', handler);

  return {
    subscribe: (onStoreChange: () => void) => {
      listeners.add(onStoreChange);
      return () => {
        listeners.delete(onStoreChange);
        if (listeners.size === 0) source.off('data', handler); // cleanup when last subscriber leaves
      };
    },
    getSnapshot: () => snapshots,
    getServerSnapshot: () => [] as readonly MetricSnapshot[], // SSR-safe
  };
}

// ── Stable store reference tied to source identity ──────────────────────────
function useMetricsStore(source: EventEmitter<MetricSnapshot>, windowSize: number) {
  // useRef ensures one store per source instance, no re-creation on re-render
  const storeRef = useRef<ReturnType<typeof createMetricsStore> | null>(null);
  const prevSourceRef = useRef(source);

  if (!storeRef.current || prevSourceRef.current !== source) {
    // Tear down old store on source identity change (rare but correct)
    storeRef.current = createMetricsStore(source, windowSize);
    prevSourceRef.current = source;
  }

  return storeRef.current;
}

export function LiveMetricsPanel({ source, columnCount, windowSize = 200 }: Props) {
  const store = useMetricsStore(source, windowSize);

  // useSyncExternalStore: the ONLY correct way to subscribe to external mutable state.
  // It prevents the "tearing" problem where different components in the same render
  // see different snapshots of the same store during concurrent rendering.
  const metrics = useSyncExternalStore(
    store.subscribe,
    store.getSnapshot,
    store.getServerSnapshot,
  );

  // ── Column width via ResizeObserver (async, non-blocking) ────────────────
  // ResizeObserver fires asynchronously AFTER layout, in its own task.
  // This prevents forced synchronous layout reads entirely.
  const containerRef = useRef<HTMLDivElement>(null);
  const [columnWidth, setColumnWidth] = useState(0);

  useLayoutEffect(() => {
    const el = containerRef.current;
    if (!el) return;

    const ro = new ResizeObserver(entries => {
      const width = entries[0]?.contentRect.width ?? 0;
      // Wrap in startTransition: column reflow is low-priority vs. live data renders
      startTransition(() => setColumnWidth(Math.floor(width / columnCount)));
    });

    ro.observe(el);
    // Fire once synchronously for initial layout pass — avoids 0-width flash
    setColumnWidth(Math.floor(el.getBoundingClientRect().width / columnCount));

    return () => ro.disconnect();
  }, [columnCount]); // stable dep — only reconnect if column count changes

  return (
    <div ref={containerRef} className="metrics-grid">
      {metrics.map(m => (
        // Stable key from data identity — O(1) reconciliation for appends
        <MetricCard key={m.id} data={m} width={columnWidth} />
      ))}
    </div>
  );
}
```

#### Key Optimization Differentials

- **`useSyncExternalStore` vs `useEffect+useState`:** USES eliminates the "concurrent tearing" window where `useEffect` subscriptions can miss an update that fired between the render phase and the effect phase commit. The `subscribe` + `getSnapshot` contract guarantees snapshot consistency across concurrent renders.
- **`ResizeObserver` vs `useEffect` + `offsetWidth` read:** RO fires asynchronously after layout has settled — zero FSLs. The one synchronous `getBoundingClientRect()` on mount is intentional and bounded (runs once, not on every render).
- **Sliding window with structural share:** `snapshots.slice(-(windowSize-1))` allocates a new array, but the contained `MetricSnapshot` objects are **shared references** — no deep clone. GC pressure is O(windowSize) amortized rather than O(total events).
- **Store-outside-component pattern:** The store's `subscribe` and `getSnapshot` references are permanently stable (object methods on a `const`). `useSyncExternalStore` does reference-equality checks on `subscribe` — unstable references here cause a subscribe/teardown cycle on every render.
- **`startTransition` for column reflow:** Resize events triggering column width changes are lower priority than incoming live data. Wrapping in `startTransition` tells the scheduler to defer this behind any pending `UserBlockingPriority` work (e.g., the user scrolling or typing a filter).

---

## 🧠 MIN 15–20: SENIOR SYSTEM DIAGNOSTIC

### Scenario: The Phantom Double-Subscription Race

Your team ships a new feature: a `useRealtimeFlag` hook that subscribes to a WebSocket-backed feature-flag service. It's used in a high-traffic dashboard that renders 40+ flag-gated components. In production (React 18, Strict Mode **disabled** on prod), under concurrent rendering, you observe intermittent UI flickers where a component briefly shows the **wrong** flag state — a flag that was already `false` momentarily renders as `true` right before snapping back.

The hook implementation:

```tsx
// useRealtimeFlag.ts
import { useEffect, useReducer } from 'react';
import { flagSocket } from '@/services/flagSocket';

type FlagState = { value: boolean; version: number };

function reducer(state: FlagState, update: FlagState): FlagState {
  return update.version > state.version ? update : state;
}

export function useRealtimeFlag(flagKey: string): boolean {
  const [state, dispatch] = useReducer(reducer, {
    value: flagSocket.getCurrentValue(flagKey), // read current value synchronously
    version: flagSocket.getCurrentVersion(flagKey),
  });

  useEffect(() => {
    const unsubscribe = flagSocket.subscribe(flagKey, (update: FlagState) => {
      dispatch(update);
    });
    return unsubscribe;
  }, [flagKey]);

  return state.value;
}
```

#### The Diagnostic Question

The flicker is real, reproducible under CPU throttling (6x slowdown in DevTools), and correlates with React's concurrent rendering. The `flagSocket` service is correct — it never emits out-of-order versions.

**Which of the following is the root cause of the tearing artifact?**

**A.** The `useReducer` version-guard logic is inverted — it should be `>=` not `>`, causing missed updates at equal versions.

**B.** `useEffect` subscriptions do not fire until after the browser has painted, creating a **gap window** between the synchronous `flagSocket.getCurrentValue()` read during the render phase and the time the subscription becomes active — a concurrent render started during this gap reads a stale initial snapshot.

**C.** `flagSocket.subscribe` is not wrapped in `useCallback`, causing it to create a new subscription function reference on every render, leaking subscriptions.

**D.** The `useReducer` initial state reads `flagSocket.getCurrentValue` synchronously, but in concurrent mode React may **render the component multiple times** before committing — if `flagSocket`'s value changes between render invocations, the initial state captured in the first render attempt is stale when React commits the second attempt's output without re-running the initializer.

**E.** Both B and D — there are two distinct tearing windows, and the hook requires `useSyncExternalStore` to eliminate both.

---

<details>
<summary>

### 🔑 System Diagnostic Answer Key (click to reveal)

</summary>

#### Answer: **E — Both B and D**

This is a classic dual-path tearing failure, and it's the exact problem `useSyncExternalStore` was invented to solve.

**Tearing Window 1 (Option D — Render-Phase Stale Initial State):**

In concurrent mode, React can **speculatively render** a component multiple times before committing. The `useReducer` initializer (`flagSocket.getCurrentValue(flagKey)`) runs during the render phase. If React renders the component at `t=0` (flag=`false`, version=5), then a WebSocket message arrives changing the flag to `true` at `t=1`, and React then re-renders (due to scheduler interruption) at `t=2` — the initial state is **re-read** from `getCurrentValue` which now returns `true`. React commits `t=2`'s render. But because the `useEffect` subscription hasn't fired yet, the committed component shows `true` while the rest of the tree may still be showing values from the `t=0` snapshot. This is **intra-tree tearing**.

**Tearing Window 2 (Option B — Post-Paint Subscription Gap):**

`useEffect` fires **after the browser has painted**. Between commit and paint (and paint and effect fire), the component displays the initial snapshot value — but any `flagSocket` updates that arrive in this window are missed until the subscription is active. Under 6x CPU throttle, this window can span multiple frames, causing visible flickers.

**The correct fix:**

```tsx
import { useSyncExternalStore } from 'react';
import { flagSocket } from '@/services/flagSocket';

export function useRealtimeFlag(flagKey: string): boolean {
  return useSyncExternalStore(
    // subscribe: called synchronously during React's commit phase
    (onStoreChange) => flagSocket.subscribe(flagKey, onStoreChange),
    // getSnapshot: called during render to read current value
    () => flagSocket.getCurrentValue(flagKey),
    // getServerSnapshot: SSR-safe fallback
    () => flagSocket.getServerValue(flagKey),
  );
}
```

`useSyncExternalStore` closes both windows by:
1. **Snapshot consistency check:** After every render, React re-calls `getSnapshot()` and compares it to the snapshot from the render. If they differ (because the store mutated mid-render), React synchronously re-renders — preventing the committed output from ever reflecting a stale snapshot.
2. **Synchronous subscription registration:** The `subscribe` callback is wired up during the **commit phase** (before paint), not after — closing the post-paint gap.

**Option A** is wrong — `>` is correct; equal versions are idempotent no-ops.  
**Option C** is a real antipattern but not the root cause here — the subscription teardown/re-subscribe would cause a momentary gap but the `unsubscribe` return value from `useEffect` handles cleanup correctly.

</details>

---

*Next session target: `useOptimistic` internals — how React manages the optimistic-then-reconcile state fork inside the Fiber update queue, and why naive wrapping with `startTransition` causes double-renders.*
