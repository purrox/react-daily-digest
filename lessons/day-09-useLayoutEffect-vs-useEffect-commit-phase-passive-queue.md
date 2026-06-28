# Day 09 — `useLayoutEffect` vs `useEffect`: Commit-Phase Sequencing, Passive-Effects Queue Flushing & the Browser Paint Race

> **20-Minute Senior Engineering Digest** · 2026-06-28
> **Prerequisite fluency:** React Fiber commit phases, browser rendering pipeline, scheduler microtask queue.

---

## ⚡ MIN 0–5: ARCHITECTURAL DEEP DIVE — THE "WHY"

### Commit Phase Anatomy: Three Distinct Sub-Phases

The Fiber reconciler's **commit phase** is synchronous and uninterruptible. After `completeWork` builds the finished work tree, `commitRoot` runs three sequential sub-phases, each walking the effect list:

```
commitRoot(root)
  └─ commitBeforeMutationEffects()   // getSnapshotBeforeUpdate, focusedInstanceHandle
  └─ commitMutationEffects()         // DOM insertions/deletions, ref detachments
  └─ commitLayoutEffects()           // useLayoutEffect callbacks, ref attachments
  └─ scheduleCallback(flushPassiveEffects)  // useEffect → scheduled asynchronously
```

**`useLayoutEffect` fires inside `commitLayoutEffects`** — synchronously, before the browser has any opportunity to paint. React walks the effect list (`flags & HookLayout`), runs **cleanup** of the previous effect first, then runs the **setup**. This is identical in timing to class-component `componentDidMount` / `componentDidUpdate`.

**`useEffect` fires inside `flushPassiveEffects`** — scheduled via the Scheduler at `NormalPriority` after React yields control back to the browser. The browser may paint one or more frames before passive effects flush. In React 18, this scheduling uses `MessageChannel` (a macrotask) rather than `requestAnimationFrame`, meaning:

```
commitRoot → browser paint opportunity → MessageChannel macrotask → flushPassiveEffects
```

The practical consequence is a **race window** between the DOM mutation committed by React and the passive effect that reads or reacts to it. Any DOM measurement or subscription setup in `useEffect` will observe stale or visually-already-painted state.

### The Flag Bitmask Distinction

Inside `ReactFiberFlags.js`, these are distinct bits on the fiber's `flags` and `subtreeFlags` fields:

```
// Simplified from React source
export const Passive    = 0b00000000000000010000000000; // useEffect
export const Layout     = 0b00000000000000000001000000; // useLayoutEffect
export const Insertion  = 0b00000000000000100000000000; // useInsertionEffect
```

During `completeWork`, React unions these flags up the tree via `subtreeFlags`. The commit-phase walkers check `subtreeFlags` first to skip entire subtrees with no matching effects — a critical short-circuit for large trees. If your component renders a large subtree but only one deep node has a `Layout` flag, the walker still descends. This is why **colocating effects near the DOM they mutate** matters structurally, not just organizationally.

### Passive Effects in Concurrent Mode: StrictMode Double-Fire

React 18 `StrictMode` in development mounts → unmounts → re-mounts every component to surface missing cleanup. This fires:

1. Setup (mount)
2. Cleanup (simulated unmount)
3. Setup again (remount)

This applies to **both** `useEffect` and `useLayoutEffect`. The dev-only double-invoke exposes subscriptions, timers, or event listeners that are not idempotently cleaned up. Production does not double-invoke. A component that works in production but fails in StrictMode has a latent bug that will surface whenever Fast Refresh remounts it.

### `useInsertionEffect`: The CSS-in-JS Escape Hatch

Fires **before** `commitMutationEffects`, giving CSS-in-JS libraries a guaranteed synchronous slot to inject `<style>` nodes before any DOM mutation or layout calculation occurs. Using `useLayoutEffect` for style injection is wrong because it runs *after* mutation — the browser already has an unstyled DOM snapshot at that point. `useInsertionEffect` has **no access to refs** (they are not yet attached) and cannot schedule state updates.

---

## 🛠️ MIN 5–15: PERFORMANCE & CODE REVIEW PROFILE

### The Flawed Code — Four Distinct Bugs

```tsx
// ❌ PRODUCTION BUG CLUSTER — tooltip anchor measurement
import { useEffect, useRef, useState } from 'react';

interface TooltipProps {
  anchorId: string;
  content: string;
}

export function Tooltip({ anchorId, content }: TooltipProps) {
  const tooltipRef = useRef<HTMLDivElement>(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  // BUG 1: useEffect for DOM measurement — causes visible flash
  useEffect(() => {
    const anchor = document.getElementById(anchorId);
    if (!anchor || !tooltipRef.current) return;

    const anchorRect = anchor.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect(); // BUG 2: layout thrash

    setPosition({
      top: anchorRect.bottom + window.scrollY,
      left: anchorRect.left + window.scrollX - tooltipRect.width / 2,
    });
  }, [anchorId]); // BUG 3: stale closure — content changes don't re-measure

  // BUG 4: ResizeObserver leak — no cleanup
  useEffect(() => {
    const anchor = document.getElementById(anchorId);
    if (!anchor) return;

    const observer = new ResizeObserver(() => {
      const rect = anchor.getBoundingClientRect();
      setPosition(prev => ({ ...prev, top: rect.bottom + window.scrollY }));
    });
    observer.observe(anchor);
    // missing return () => observer.disconnect()
  }, [anchorId]);

  return (
    <div
      ref={tooltipRef}
      style={{ position: 'fixed', top: position.top, left: position.left }}
    >
      {content}
    </div>
  );
}
```

### Bug Breakdown

| # | Location | Root Cause | Symptom |
|---|----------|-----------|---------|
| 1 | `useEffect` for measurement | Runs post-paint → tooltip renders at `{0,0}` first, then jumps | **FOUC / coordinate flash** |
| 2 | `getBoundingClientRect` after `setPosition` | Forces synchronous layout recalculation mid-effect | **Layout thrash, jank** |
| 3 | `[anchorId]` dep array only | `content` prop changes resize tooltip width — position stale | **Misaligned tooltip** |
| 4 | No `ResizeObserver.disconnect()` | Observer holds strong ref to anchor + closure over setState | **Memory leak, stale setState on unmounted component** |

### Optimized Production Refactor

```tsx
// ✅ PRODUCTION-HARDENED — zero flash, no leaks, structurally correct
import {
  useLayoutEffect,
  useRef,
  useState,
  useCallback,
  useId,
  type RefObject,
} from 'react';

interface TooltipPosition {
  top: number;
  left: number;
}

function useAnchorPosition(
  anchorId: string,
  tooltipRef: RefObject<HTMLDivElement | null>,
): TooltipPosition {
  const [position, setPosition] = useState<TooltipPosition>({ top: 0, left: 0 });

  const measure = useCallback(() => {
    const anchor = document.getElementById(anchorId);
    const tooltip = tooltipRef.current;
    if (!anchor || !tooltip) return;

    // Read phase — batch all getBCR calls together to avoid interleaved read/write
    const anchorRect = anchor.getBoundingClientRect();
    const tooltipWidth = tooltip.offsetWidth; // offsetWidth avoids forced reflow if layout is clean

    setPosition({
      top: anchorRect.bottom + window.scrollY,
      // Center tooltip; clamp to viewport to prevent off-screen bleed
      left: Math.max(
        8,
        Math.min(
          anchorRect.left + window.scrollX - tooltipWidth / 2,
          window.innerWidth - tooltipWidth - 8,
        ),
      ),
    });
  }, [anchorId, tooltipRef]);

  // useLayoutEffect: fires synchronously post-mutation, pre-paint
  // Guarantees tooltip is positioned before the browser composites the frame
  useLayoutEffect(() => {
    measure();

    const anchor = document.getElementById(anchorId);
    if (!anchor) return;

    const observer = new ResizeObserver(measure);
    observer.observe(anchor);
    // Observe window resize — use passive to avoid blocking scroll
    window.addEventListener('resize', measure, { passive: true });

    return () => {
      observer.disconnect();
      window.removeEventListener('resize', measure);
    };
  }, [anchorId, measure]); // measure is stable — useCallback deps correctly declared

  return position;
}

interface TooltipProps {
  anchorId: string;
  content: string;
}

export function Tooltip({ anchorId, content }: TooltipProps) {
  const tooltipRef = useRef<HTMLDivElement>(null);
  const position = useAnchorPosition(anchorId, tooltipRef);

  return (
    <div
      ref={tooltipRef}
      role="tooltip"
      style={{
        position: 'fixed',
        top: position.top,
        left: position.left,
        // Prevent tooltip from being a layout participant before positioned
        visibility: position.top === 0 && position.left === 0 ? 'hidden' : 'visible',
      }}
    >
      {content}
    </div>
  );
}
```

### Optimization Differentials

- **`useLayoutEffect` instead of `useEffect`** — measurement happens inside the commit phase synchronously. The browser never paints the `{0,0}` initial position. Zero frames of FOUC.
- **Read-batch before write** — both `getBoundingClientRect()` and `offsetWidth` are called before any `setPosition` write. Avoids forced synchronous layout recalculation (layout thrash) where a read after a write forces the browser to flush pending style calculations.
- **`useCallback` for `measure`** — stabilizes the function reference so `useLayoutEffect`'s dep array doesn't re-run the effect on every render while still capturing the latest `anchorId`. The `tooltipRef` is a stable `RefObject` (same object across renders) so it's safe as a dep.
- **`ResizeObserver.disconnect()` in cleanup** — prevents observer from firing on unmounted component. Eliminates memory leak and `setState` on unmounted component warning.
- **`{ passive: true }` on resize listener** — tells the browser this handler will never call `preventDefault()`, allowing the browser to defer dispatching it non-blocking. Matters for scroll-linked animations.
- **Viewport clamping** — prevents tooltip from rendering off-screen; pure math, zero extra DOM reads.
- **`visibility: hidden` guard** — hides the tooltip for exactly zero painted frames (it's set synchronously in the same layout-effect pass), but prevents a technically correct yet invisible artifact if the initial measure fails (e.g., anchor not yet in DOM).

---

## 🧠 MIN 15–20: SENIOR SYSTEM DIAGNOSTIC

### The Scenario

A team migrates a large dashboard from React 17 to React 18 with concurrent features enabled. The dashboard has a `<ChartPanel>` that uses a custom `useResizeSync` hook:

```tsx
// useResizeSync.ts — React 17 era, class-component migration artifact
import { useLayoutEffect, useRef } from 'react';

export function useResizeSync(onResize: (width: number, height: number) => void) {
  const containerRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    const el = containerRef.current!;
    // Class-component pattern: manual size check on mount
    onResize(el.offsetWidth, el.offsetHeight);

    const observer = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect;
      onResize(width, height);
    });
    observer.observe(el);

    return () => observer.disconnect();
  }); // ← no dependency array

  return containerRef;
}
```

```tsx
// ChartPanel.tsx
import { useState, startTransition } from 'react';
import { useResizeSync } from './useResizeSync';

export function ChartPanel({ dataSource }: { dataSource: DataSource }) {
  const [dims, setDims] = useState({ w: 0, h: 0 });

  const containerRef = useResizeSync((w, h) => {
    startTransition(() => {
      setDims({ w, h });
    });
  });

  return (
    <div ref={containerRef} style={{ width: '100%', height: '100%' }}>
      <ExpensiveChart width={dims.w} height={dims.h} data={dataSource} />
    </div>
  );
}
```

In React 17, this worked perfectly. After upgrading to React 18 with `<React.StrictMode>` in development, the team observes:

1. `ExpensiveChart` renders with `{ w: 0, h: 0 }` on every Fast Refresh
2. `onResize` is called with valid dimensions, but the chart briefly shows the zero-dimension render
3. Under low CPU conditions, the chart never fully renders after a Fast Refresh

**Identify the three distinct failure modes and their root causes, then explain the correct fix without changing the `useResizeSync` API signature.**

```tsx
// Proposed "fix" from junior engineer — evaluate whether this resolves all three issues:
export function ChartPanel({ dataSource }: { dataSource: DataSource }) {
  const [dims, setDims] = useState({ w: 0, h: 0 });

  const handleResize = useCallback((w: number, h: number) => {
    setDims({ w, h }); // removed startTransition
  }, []);

  const containerRef = useResizeSync(handleResize);

  return (
    <div ref={containerRef} style={{ width: '100%', height: '100%' }}>
      <ExpensiveChart width={dims.w} height={dims.h} data={dataSource} />
    </div>
  );
}
```

---

<details>
<summary><strong>⚙️ System Diagnostic Answer Key — click to expand</strong></summary>

### Failure Mode 1: `useLayoutEffect` with no dep array re-runs every render

`useLayoutEffect(() => { ... })` with **no dependency array** runs after every single commit. In React 17, this was survivable because renders were synchronous and the ResizeObserver would quickly re-sync. In React 18's concurrent mode, an interleaved render can trigger the layout effect multiple times in rapid succession — each run calls `observer.disconnect()` then immediately `observer.observe()`. The ResizeObserver fires its callback asynchronously, but the synchronous layout effect has already re-registered. Result: stale observer callbacks queued before disconnect fire after re-registration, producing double-fire of `onResize`.

**Fix:** Add `[]` dep array to `useLayoutEffect` (only run on mount/unmount). The `onResize` ref-stabilization pattern should be used to keep the callback fresh without re-registering the observer:

```tsx
export function useResizeSync(onResize: (w: number, h: number) => void) {
  const containerRef = useRef<HTMLDivElement>(null);
  const onResizeRef = useRef(onResize);
  useLayoutEffect(() => { onResizeRef.current = onResize; }); // sync every render, no observer re-registration

  useLayoutEffect(() => {
    const el = containerRef.current!;
    onResizeRef.current(el.offsetWidth, el.offsetHeight);
    const observer = new ResizeObserver(([entry]) => {
      onResizeRef.current(entry.contentRect.width, entry.contentRect.height);
    });
    observer.observe(el);
    return () => observer.disconnect();
  }, []); // ← mount-only

  return containerRef;
}
```

### Failure Mode 2: `startTransition` wrapping a layout-derived state update causes a deferred re-render gap

`startTransition` marks the resulting state update as `TransitionLane` — low priority, interruptible, and deferred. React may not commit the transition update before the browser paints. Since `dims` starts at `{ w: 0, h: 0 }` and the ResizeObserver callback fires asynchronously (always, even from inside `useLayoutEffect`), the chart always renders once at zero dimensions before the transition resolves. Under low CPU (Failure Mode 3), the scheduler may never pick up the transition update if higher-priority work (user events, urgent updates) keeps pre-empting it.

`startTransition` is appropriate for deferring *expensive renders caused by user interaction*, not for applying *accurate layout data derived from the DOM*. DOM measurements should always drive synchronous state updates.

**Fix:** Remove `startTransition`. Use direct `setDims`:

```tsx
const handleResize = useCallback((w: number, h: number) => {
  setDims({ w, h }); // synchronous urgent update — layout data is always urgent
}, []);
```

### Failure Mode 3: StrictMode double-mount zeroing dims

React 18 StrictMode mounts → unmounts → remounts. On the simulated unmount, the observer is disconnected. On remount, the observer fires again. But `dims` is reset to `{ w: 0, h: 0 }` on the simulated unmount because the component unmounts and remounts as a fresh instance. If `startTransition` is still wrapping the update, the deferred lane update from the first mount is discarded (deferred work from an unmounted tree is abandoned). The remounted instance starts fresh at zero, and the ResizeObserver fires asynchronously, so the chart renders at zero then jumps — every Fast Refresh.

StrictMode does NOT reset `useState` on remount in production. This is a dev-only simulation. The fix for Failure Mode 2 (remove `startTransition`) resolves Failure Mode 3 as well: without the deferred lane, the update is urgent and commits synchronously before the browser paints.

### Does the Junior's Fix Resolve All Three?

**Partially.** Removing `startTransition` (Failure Mode 2 + 3 fixed via `useCallback` + `setDims`) resolves two of the three. But `useLayoutEffect` still has no dep array (Failure Mode 1 unfixed) — the junior's fix leaves the ResizeObserver re-registration on every render, which will still produce double-fire under concurrent scheduling pressure. The correct complete fix requires BOTH the `onResizeRef` ref-forwarding pattern in `useResizeSync` AND removal of `startTransition` in `ChartPanel`.

</details>

---

*Day 09 complete. Next: React's event delegation system — synthetic event pooling removal in React 17, the `nativeEvent` reference lifecycle, and why `event.persist()` is a historical artifact.*
