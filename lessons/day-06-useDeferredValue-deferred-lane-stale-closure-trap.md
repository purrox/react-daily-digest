# Day 06 — `useDeferredValue`: Urgent/Deferred Render Passes, `DeferredLane` Assignment & the Stale-Closure Trap in Concurrent Trees

> **20-minute senior digest.** No syntax primers. No introductions. Straight to the internals.

---

## ⚡ MIN 0–5: ARCHITECTURAL DEEP DIVE — THE "WHY"

### `useDeferredValue` Is Not a Debounce — It's Two Renders on Two Lanes

`useDeferredValue(value)` spawns **two discrete render passes** scheduled on different priority lanes. The common mental model of "it debounces the value" is dangerously wrong — the timing is controlled by the **Fiber scheduler's work loop**, not wall-clock time.

---

### Pass 1: The Urgent Render (`InputContinuousLane` / `SyncLane`)

When a state update triggers (e.g., a text input fires `setState`), React schedules work on **`InputContinuousLane`** (bitmask `0b0000000000000000000000000000100`, lane 2). During `renderWithHooks`, the `updateDeferredValue` hook runs:

```typescript
// Simplified from packages/react-reconciler/src/ReactFiberHooks.js
function updateDeferredValueImpl<T>(
  hook: Hook,
  prevValue: T,
  value: T,
): T {
  if (is(value, prevValue)) {
    return prevValue; // fast-path: no change, no deferred lane needed
  }

  const shouldDeferValue = !includesOnlyNonUrgentLanes(renderLanes);

  if (shouldDeferValue) {
    // URGENT PASS: return the OLD value — schedule a deferred re-render
    const deferredLane = claimNextTransitionLane();
    currentlyRenderingFiber.lanes = mergeLanes(
      currentlyRenderingFiber.lanes,
      deferredLane,
    );
    scheduleUpdateOnFiber(currentlyRenderingFiber, deferredLane, NoTimestamp);
    return prevValue; // ← stale value returned to the component
  } else {
    // DEFERRED PASS: return the NEW value
    hook.memoizedState = value;
    return value;
  }
}
```

**`includesOnlyNonUrgentLanes(renderLanes)`** returns `false` when urgent lanes like `InputContinuousLane` are present. So during the urgent pass, the hook returns the **previous value**, schedules a deferred lane update via `scheduleUpdateOnFiber`, and exits. The committed tree shows the new urgent state (the input field updated) while deferred consumers see the previous value.

---

### Pass 2: The Deferred Render (`DeferredLane`, interruptible)

After the urgent commit, the scheduler yields and picks up the pending `DeferredLane` (bitmask `0b0000000000000010000000000000000`, lane 14 in React 19+). This render is **interruptible** — the work loop runs through `workLoopConcurrent` which checks `shouldYield()` at every fiber boundary:

```
performConcurrentWorkOnRoot
  → renderRootConcurrent          (DeferredLane set in renderLanes)
    → workLoopConcurrent          (calls shouldYield() per fiber)
      → beginWork (on fiber)
        → renderWithHooks
          → updateDeferredValueImpl
            → includesOnlyNonUrgentLanes(DeferredLane) === true
            → returns NEW value ✓
```

If a higher-priority event (another keystroke) arrives mid-deferred-render, `shouldYield()` returns `true`, the work loop breaks, and the incomplete deferred work is **thrown away** — the `workInProgress` tree is discarded via `prepareFreshStack`. The urgent event gets processed first, and a new deferred pass is re-queued. This is why `useDeferredValue` inherently provides "show last good result" UX: the deferred subtree never shows a partial/intermediate state.

---

### `claimNextTransitionLane()` vs. Dedicated `DeferredLane` (React 18 → 19 delta)

In React 18.x, `useDeferredValue` claimed from the `TransitionLane` pool (lanes 4–19), the same pool as `startTransition`. Under applications with many concurrent transitions, this caused **lane pool exhaustion** — once all 16 transition lanes were claimed, `claimNextTransitionLane()` wrapped around and overwrite still-pending transitions, forcing synchronous fallbacks.

React 19 introduced a **dedicated `DeferredLane`** separated from the transition pool. This prevents deferred value updates from consuming transition budget and eliminates the exhaustion failure mode.

---

### Interaction with Suspense: The "Stale-While-Revalidate" Primitive

When the deferred subtree suspends (throws a Promise), React catches it **on the deferred lane** and retains the committed (stale) content as the fallback. The Suspense boundary doesn't flash its fallback UI — the old content stays rendered until the deferred pass can complete without suspending. This makes `useDeferredValue` + Suspense the correct primitive for search-as-you-type with async data: stale results visible, no loading spinner flicker, new results appear atomically when ready.

---

## 🛠️ MIN 5–15: PERFORMANCE & CODE REVIEW PROFILE

### The Scenario

A search panel in a large analytics dashboard. Ships with no test failures, no linter warnings. Within two weeks of prod deployment: search results show stale data under fast typing, memory usage climbs ~4MB per minute of usage, CPU spikes on every keystroke even during empty queries, and occasionally the `onResultSelect` callback fires with data from a previous prop value.

---

### Flawed Implementation

```tsx
// SearchPanel.tsx — FLAWED PRODUCTION CODE
import { useState, useDeferredValue, useCallback, useEffect, useRef } from 'react';

interface SearchResult {
  id: string;
  title: string;
  score: number;
}

interface Props {
  dataSource: SearchResult[];
  onResultSelect: (id: string) => void;
}

export function SearchPanel({ dataSource, onResultSelect }: Props) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const [results, setResults] = useState<SearchResult[]>([]);
  const workerRef = useRef<Worker | null>(null);

  // BUG #1: Worker instantiated in render body — new Worker() on EVERY render.
  // Multiple concurrent workers accumulate. No .terminate() ever called.
  const worker = new Worker(new URL('./search.worker.ts', import.meta.url));
  workerRef.current = worker;

  // BUG #2: Empty dep array closes over the Worker instance from the FIRST render.
  // That worker is immediately replaced by BUG #1 on the next render.
  // The message handler is on an orphaned, unterminated worker.
  useEffect(() => {
    worker.onmessage = (e: MessageEvent<SearchResult[]>) => {
      setResults(e.data);
      if (e.data.length > 0) {
        // BUG #2b: `onResultSelect` is the version from mount — stale prop forever.
        onResultSelect(e.data[0].id);
      }
    };
  }, []);

  // BUG #3: Dep array is `[query]` — posts to worker on the URGENT render pass.
  // `deferredQuery` hasn't updated yet; worker receives the fresh query
  // before the deferred pass runs. Worker races the deferred render.
  // Also: `dataSource` missing from deps — stale data source reference.
  useEffect(() => {
    worker.postMessage({ query: deferredQuery, data: dataSource });
  }, [query]);

  // BUG #4: `useCallback` with empty deps closes over initial `onResultSelect`.
  // Even if parent re-renders with a new callback, this reference is frozen.
  const handleSelect = useCallback((id: string) => {
    onResultSelect(id);
  }, []);

  const isStale = query !== deferredQuery;

  return (
    <div style={{ opacity: isStale ? 0.6 : 1 }}>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search..."
      />
      {/* BUG #5: key={index} — on result reorder (score sort), React patches
          existing DOM nodes in-place instead of moving them. Breaks ARIA
          focus, animation state, and defeats reconciler identity. */}
      <ul>
        {results.map((r, i) => (
          <li key={i} onClick={() => handleSelect(r.id)}>
            {r.title} ({r.score})
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

### Optimized, Production-Hardened Refactor

```tsx
// SearchPanel.tsx — PRODUCTION REFACTOR
import {
  useState,
  useDeferredValue,
  useCallback,
  useEffect,
  useRef,
  memo,
  useId,
} from 'react';

interface SearchResult {
  id: string;
  title: string;
  score: number;
}

interface Props {
  dataSource: SearchResult[];
  onResultSelect: (id: string) => void;
}

// Fix #5 + perf: isolated memo boundary.
// Without this, the urgent-pass render re-renders the entire list on every
// keystroke — defeating the entire point of useDeferredValue.
// memo() causes Fiber's updateMemoComponent to bail out during the urgent
// pass (results/onSelect haven't changed yet), and re-render only on
// the deferred pass when results actually update.
const ResultList = memo(function ResultList({
  results,
  onSelect,
  listId,
}: {
  results: SearchResult[];
  onSelect: (id: string) => void;
  listId: string;
}) {
  return (
    <ul id={listId} role="listbox">
      {results.map((r) => (
        // Fix #5: stable id as key — preserves fiber identity through reorders
        <li key={r.id} role="option" onClick={() => onSelect(r.id)}>
          {r.title} ({r.score.toFixed(2)})
        </li>
      ))}
    </ul>
  );
});

export function SearchPanel({ dataSource, onResultSelect }: Props) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const [results, setResults] = useState<SearchResult[]>([]);

  // Fix #1: single Worker, created once, terminated on unmount.
  const workerRef = useRef<Worker | null>(null);
  useEffect(() => {
    const worker = new Worker(new URL('./search.worker.ts', import.meta.url));
    workerRef.current = worker;
    return () => {
      worker.terminate();
      workerRef.current = null;
    };
  }, []);

  // Fix #2: "event handler ref" pattern — the canonical pre-useEffectEvent backport.
  // A ref always holds the latest prop without causing effect re-subscriptions.
  const onResultSelectRef = useRef(onResultSelect);
  // No dep array: sync ref before every commit so async callbacks always see current prop.
  useEffect(() => {
    onResultSelectRef.current = onResultSelect;
  });

  // Fix #2 cont.: wire the message handler once against the stable workerRef.
  // Reads onResultSelectRef.current at call time — always fresh, never stale.
  useEffect(() => {
    const worker = workerRef.current;
    if (!worker) return;

    const handleMessage = (e: MessageEvent<SearchResult[]>) => {
      setResults(e.data);
      if (e.data.length > 0) {
        onResultSelectRef.current(e.data[0].id);
      }
    };

    worker.addEventListener('message', handleMessage);
    return () => worker.removeEventListener('message', handleMessage);
  }, []); // legitimately empty — workerRef.current is stable after first effect

  // Fix #3: gated on `deferredQuery` — only fires after the deferred lane commits.
  // Workers now receive queries in sync with what the UI is actually rendering.
  // dataSource included: stale data source reference was a silent correctness bug.
  useEffect(() => {
    const worker = workerRef.current;
    if (!worker) return;
    worker.postMessage({ query: deferredQuery, data: dataSource });
  }, [deferredQuery, dataSource]);

  // Fix #4: empty deps is now correct — reads through ref, not closure.
  // This callback is referentially stable for the component's entire lifetime,
  // which maximizes ResultList memo bail-out effectiveness.
  const handleSelect = useCallback((id: string) => {
    onResultSelectRef.current(id);
  }, []);

  const isStale = query !== deferredQuery;
  const listId = useId();

  return (
    <div
      style={{ opacity: isStale ? 0.6 : 1, transition: 'opacity 150ms ease' }}
      aria-busy={isStale}
    >
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
        aria-controls={listId}
        aria-label="Search"
        type="search"
      />
      <ResultList results={results} onSelect={handleSelect} listId={listId} />
    </div>
  );
}
```

---

### Key Optimization Differentials

- **Worker in render body (BUG #1)**: `new Worker()` called on every render — each keystroke spawned a new OS thread. No `.terminate()` = unbounded memory leak. Each orphaned worker still receives its one `onmessage` assignment and then idles permanently. Fix: `useEffect` with empty deps + cleanup teardown.
- **Stale closure across async boundary (BUG #2)**: Workers are async — by the time `.onmessage` fires, the React render that assigned it may be committed, updated, or unmounted. Closures don't survive through time; refs do. The "sync ref before effects" pattern (`useEffect` with no dep array) is the `useEffectEvent` RFC implemented today.
- **Urgent-pass worker spam (BUG #3)**: `[query]` as dep means the worker is posted on the `InputContinuousLane` render — before the deferred pass updates `deferredQuery`. Result: worker races the deferred render, and `setResults` fires before the deferred subtree re-renders, causing a one-frame stale display. `[deferredQuery]` ensures the worker only fires after the deferred lane commits, keeping worker output in sync with the rendered deferred value.
- **`useCallback` stale trap (BUG #4)**: Empty deps without a ref-backed escape freezes the closure. The ref pattern makes the empty dep array semantically correct — the callback reads from a mutable ref that is always current, so referential stability is legitimate, not a bug.
- **`key={index}` reconciler defeat (BUG #5)**: When search results reorder on a new query (common with score-ranked results), index keys force React to update existing fiber nodes in place. The reconciler sees "same key, different data" and patches props rather than unmounting/remounting. Consequence: ARIA `aria-selected` state on the wrong item, CSS transition on the wrong node, and potential focus loss. `key={r.id}` gives fibers stable identity across reorders.
- **Missing `memo` boundary**: `ResultList` without `memo` re-renders on every urgent pass (every keystroke), even though `results` hasn't changed yet. This nullifies `useDeferredValue`'s primary benefit — the expensive render runs on every lane, not just the deferred one. `memo` ensures `beginWork` reaches `updateMemoComponent`, finds equal props, and returns the bail-out path without entering the function body.

---

## 🧠 MIN 15–20: SENIOR SYSTEM DIAGNOSTIC

### Scenario: The Phantom Cross-User Data Leak

A team ships a Next.js 14 App Router dashboard using RSC streaming. QA reports that approximately 1-in-50 page loads shows **statistics from a different user's session** — PII exposure severity. No shared in-memory React state is apparent. The component tree:

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { UserStats } from '@/components/UserStats';     // Server Component
import { ActivityFeed } from '@/components/ActivityFeed'; // Server Component

export default async function DashboardPage() {
  return (
    <main>
      <Suspense fallback={<StatsShimmer />}>
        <UserStats />
      </Suspense>
      <Suspense fallback={<FeedShimmer />}>
        <ActivityFeed />
      </Suspense>
    </main>
  );
}
```

```tsx
// components/UserStats.tsx — Server Component
import { cache } from 'react';
import { getUser } from '@/lib/auth';
import { db } from '@/lib/db';

const getUserStats = cache(async () => {
  const user = await getUser(); // reads cookies/headers
  return db.stats.findUnique({ where: { userId: user.id } });
});

export async function UserStats() {
  const stats = await getUserStats();
  return <StatsDisplay data={stats} />;
}
```

```typescript
// lib/db.ts — a subtle singleton
import { PrismaClient } from '@prisma/client';

const db = (global as any).prisma ?? new PrismaClient({ log: ['query'] });

if (process.env.NODE_ENV !== 'production') {
  (global as any).prisma = db;
}

export { db };
```

---

### The Question

The 1-in-50 cross-user data leak is caused by which of the following?

**A.** React's `cache()` uses a global LRU memoization store keyed by function reference. Two concurrent requests call `getUserStats` within the same GC epoch before the cache key expires, and the second request receives the first request's memoized return value.

**B.** The `cache()` function is correctly scoped per RSC render pass, but Prisma's connection pool is returning a stale query result from a connection that wasn't flushed between requests — a connection-level prepared statement cache hit, not a React-level leak.

**C.** `cache()` stores its memoization map on an `AsyncLocalStorage` context, and under concurrent RSC streaming, two requests share the same `AsyncLocalStorage` scope when Node.js re-uses a worker thread between overlapping `async/await` continuations from different requests.

**D.** `DashboardPage` is being statically cached by Next.js's full-route cache (the default `force-cache` semantic), and the RSC payload written to the cache includes serialized user-specific data from the first request that populated it.

---

<details>
<summary><strong>System Diagnostic Answer Key — Click to Reveal</strong></summary>

### Answer: B

---

**Why A is wrong:**

React's `cache()` is **explicitly per-request scoped**. Internally it uses `AsyncLocalStorage` to bind a fresh `Map` to each incoming request via `run(new Map(), handler)`. There is no global LRU — the React team explicitly rejected that design to prevent cross-request leaks. The `cache()` store is created at the start of the RSC render pass and is garbage-collected after the response stream closes.

---

**Why C is wrong — the closest trap:**

`cache()` does use `AsyncLocalStorage`. However, Node.js **does not re-use** `AsyncLocalStorage` contexts across requests. Each request receives an independent context via `als.run(freshMap, ...)`. Within a single request, async continuations (across `await` boundaries) correctly inherit the same context via the async context propagation mechanism. The `AsyncLocalStorage` contract guarantees request isolation even under high concurrency on a single-threaded event loop.

---

**Why D is wrong:**

Next.js 14 opts a route into **dynamic rendering** whenever it detects dynamic data sources — specifically calls to `cookies()`, `headers()`, or `searchParams` within the render tree. `getUser()` reads from cookies/headers, which triggers this opt-out. The full-route cache is bypassed entirely. Even if it weren't, the RSC payload in the cache would contain the first user's rendered HTML/RSC tree, which would be visually wrong but would also fail React's hydration checksum — producing a visible console error, not a silent data leak.

---

**Why B is correct — the real culprit:**

Prisma's connection pool (via `pg` or `mysql2` adapters) implements **prepared statement caching** at the connection level. Under concurrent load:

1. Request A (User A) executes `db.stats.findUnique({ where: { userId: 'user-a' } })` on **Connection #7**
2. Connection #7 prepares the statement and caches the result or plan
3. Request A completes; Connection #7 returns to the pool
4. Request B (User B) is assigned **Connection #7** from the pool
5. Request B calls `getUser()` — returns User B's ID correctly
6. `db.stats.findUnique({ where: { userId: 'user-b' } })` executes, **but** if PgBouncer is running in `session` pooling mode (not `transaction` mode), connection-level session state — including `SET` variables, temp tables, or in some configurations advisory locks — persists across the pool checkout boundary

The concrete mechanism varies by infrastructure:

- **PgBouncer `session` mode**: session-level `SET` variables, `PREPARE`d statements, and `search_path` survive between pool checkouts. If the app sets `SET LOCAL app.user_id = ...` for row-level security and then a new request gets the same connection before the session resets, RLS policies evaluate against the previous user's context.
- **PlanetScale/Vitess query cache**: statement-level result cache keyed only on query text + params, where the params are bound at the driver level. A subtly non-unique parameter (e.g., integer coercion of UUIDs, or missing user isolation in multi-tenant queries) produces a cache hit across users.

```typescript
// The fix: enforce per-request connection context with RLS
// In middleware or a request-scoped db factory:
export async function getRequestScopedDb(userId: string) {
  // Forces this connection to evaluate RLS policies for the correct user,
  // even if connection is re-used from pool with stale session state.
  await db.$executeRaw`SELECT set_config('app.current_user_id', ${userId}, true)`;
  return db;
}

// In PostgreSQL RLS policy:
// CREATE POLICY user_isolation ON stats
//   USING (user_id = current_setting('app.current_user_id')::uuid);
```

Alternatively, switch PgBouncer to **transaction pooling mode** — session state is guaranteed clean at each transaction boundary, eliminating the leak entirely at the infrastructure layer without code changes.

---

**Architectural lesson:**

React's `cache()` and the RSC execution model are **correctly isolated**. The RSC architecture is not the failure surface here. Cross-request data leaks in RSC apps almost always originate at the **infrastructure layer**: connection pool session state, CDN cache keys missing `Vary: Cookie`, or Redis key collisions from insufficient namespace isolation. When diagnosing a data leak, exhaust the infrastructure stack before auditing React internals — the React layer is deterministic and well-specified; the database/network layer is stateful and environmental.

</details>

---

*Day 06 complete. Tomorrow: `react-reconciler` Host Config internals — building a custom renderer from `createInstance` to `commitUpdate`, and the fiber tag lifecycle for custom host environments.*
