# View Transitions (MEDIUM Priority)

Guide for implementing smooth, native-feeling animations using React's View Transition API. Use this for page transitions, route changes, shared element animations, component enter/exit, list reorder, and directional navigation animations.

**Impact:** Native-quality animations without third-party libraries, improved perceived performance, better UX.

---

## Quick Reference

| Pattern | Priority | When to Apply |
|---------|----------|---------------|
| Shared Element Transitions | HIGH | List → detail navigation, image galleries |
| Suspense Reveal | HIGH | Data loading states |
| List Identity | MEDIUM | Reorderable lists, filtered lists |
| State Change (enter/exit) | MEDIUM | Modals, toasts, conditional content |
| Route Change | LOW | Page-to-page navigation |

---

## Core Concepts

### The `<ViewTransition>` Component

```jsx
import { ViewTransition } from 'react';

<ViewTransition>
  <Component />
</ViewTransition>
```

React auto-assigns a unique `view-transition-name` and calls `document.startViewTransition` behind the scenes. Never call `startViewTransition` yourself.

### Animation Triggers

| Trigger | When it fires |
|---------|--------------|
| **enter** | `<ViewTransition>` first inserted during a Transition |
| **exit** | `<ViewTransition>` first removed during a Transition |
| **update** | DOM mutations inside a `<ViewTransition>` |
| **share** | Named VT unmounts and another with same `name` mounts in the same Transition |

Only `startTransition`, `useDeferredValue`, or `Suspense` activate VTs. Regular `setState` does not animate.

### Critical Placement Rule

`<ViewTransition>` only activates enter/exit if it appears **before any DOM nodes**:

```jsx
// Works
<ViewTransition enter="auto" exit="auto">
  <div>Content</div>
</ViewTransition>

// Broken — div wraps the VT, suppressing enter/exit
<div>
  <ViewTransition enter="auto" exit="auto">
    <div>Content</div>
  </ViewTransition>
</div>
```

---

## When to Animate

Every `<ViewTransition>` should communicate a spatial relationship or continuity. If you can't articulate what it communicates, don't add it.

Implement **all** applicable patterns from this list, in this order:

| Priority | Pattern | What it communicates |
|----------|---------|---------------------|
| 1 | **Shared element** (`name`) | "Same thing — going deeper" |
| 2 | **Suspense reveal** | "Data loaded" |
| 3 | **List identity** (per-item `key`) | "Same items, new arrangement" |
| 4 | **State change** (`enter`/`exit`) | "Something appeared/disappeared" |
| 5 | **Route change** (layout-level) | "Going to a new place" |

### Choosing Animation Style

| Context | Animation | Why |
|---------|-----------|-----|
| Hierarchical navigation (list → detail) | Type-keyed `nav-forward` / `nav-back` | Communicates spatial depth |
| Lateral navigation (tab-to-tab) | Bare `<ViewTransition>` (fade) or `default="none"` | No depth to communicate |
| Suspense reveal | `enter`/`exit` string props | Content arriving |
| Revalidation / background refresh | `default="none"` | Silent — no animation needed |

Reserve directional slides for hierarchical navigation (list → detail) and ordered sequences (prev/next photo, carousel, paginated results).

---

## Availability

- **Next.js:** Do **not** install `react@canary` — the App Router already bundles React canary internally. `ViewTransition` works out of the box.
- **Without Next.js:** Install `react@canary react-dom@canary` (`ViewTransition` is not in stable React).
- Browser support: Chromium 111+, Firefox 144+, Safari 18.2+. Graceful degradation on unsupported browsers.

---

## 1. Shared Element Transitions (HIGH Priority)

**Impact:** HIGH (communicates continuity, "same thing going deeper")

Same `name` on two VTs — one unmounting, one mounting — creates a shared element morph.

### Basic Shared Element:

```jsx
<ViewTransition name="hero-image">
  <img src="/thumb.jpg" onClick={() => startTransition(() => onSelect())} />
</ViewTransition>

// On the other view — same name
<ViewTransition name="hero-image">
  <img src="/full.jpg" />
</ViewTransition>
```

### Important Rules:

- Only one VT with a given `name` can be mounted at a time — use unique names (`photo-${id}`)
- Watch for reusable components: if a component with a named VT is rendered in both a modal/popover *and* a page, both mount simultaneously and break the morph
- `share` takes precedence over `enter`/`exit`
- Never use a fade-out exit on pages with shared morphs — use a directional slide instead

### Composing Shared Elements with List Identity:

Shared elements and list identity are independent concerns. When a list item contains a shared element (e.g., an image that morphs into a detail view), use two nested `<ViewTransition>` boundaries:

```jsx
{items.map(item => (
  <ViewTransition key={item.id}>                                      {/* list identity */}
    <Link href={`/items/${item.id}`}>
      <ViewTransition name={`item-image-${item.id}`} share="morph">   {/* shared element */}
        <Image src={item.image} />
      </ViewTransition>
      <p>{item.name}</p>
    </Link>
  </ViewTransition>
))}
```

The outer VT handles list reorder/enter animations. The inner VT handles the cross-route shared element morph.

---

## 2. Suspense Reveal (HIGH Priority)

**Impact:** HIGH (communicates "data loaded")

Instead of awaiting data in async components before returning JSX, use Suspense boundaries to show the wrapper UI faster while data loads.

### Simple Cross-Fade:

```jsx
<ViewTransition>
  <Suspense fallback={<Skeleton />}><Content /></Suspense>
</ViewTransition>
```

### Directional Reveal:

```jsx
<Suspense fallback={<ViewTransition exit="slide-down"><Skeleton /></ViewTransition>}>
  <ViewTransition enter="slide-up" default="none"><Content /></ViewTransition>
</Suspense>
```

### Share Promise Across Components:

```tsx
function Page() {
  // Start fetch immediately, but don't await
  const dataPromise = fetchData()
  
  return (
    <div>
      <div>Sidebar</div>
      <div>Header</div>
      <Suspense fallback={<Skeleton />}>
        <DataDisplay dataPromise={dataPromise} />
        <DataSummary dataPromise={dataPromise} />
      </Suspense>
      <div>Footer</div>
    </div>
  )
}

function DataDisplay({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise) // Unwraps the promise
  return <div>{data.content}</div>
}
```

**Benefit:** Both components share the same promise, so only one fetch occurs. Layout renders immediately while both components wait together.

---

## 3. List Identity (MEDIUM Priority)

**Impact:** MEDIUM (communicates "same items, new arrangement")

Use `key` on `<ViewTransition>` to animate list reorder, filter, or sort.

### List Reorder:

```jsx
{items.map(item => (
  <ViewTransition key={item.id}><ItemCard item={item} /></ViewTransition>
))}
```

Trigger inside `startTransition`. Avoid wrapper `<div>`s between list and VT.

---

## 4. State Change (enter/exit) (MEDIUM Priority)

**Impact:** MEDIUM (communicates "something appeared/disappeared")

### Enter/Exit:

```jsx
{show && (
  <ViewTransition enter="fade-in" exit="fade-out"><Panel /></ViewTransition>
)}
```

### Force Re-Enter with `key`:

```jsx
<ViewTransition key={searchParams.toString()} enter="slide-up" default="none">
  <ResultsGrid />
</ViewTransition>
```

**Caution:** If wrapping `<Suspense>`, changing `key` remounts the boundary and refetches.

---

## 5. Transition Types (Directional Navigation)

**Impact:** HIGH (communicates spatial depth)

Tag transitions with `addTransitionType` so VTs can pick different animations based on context.

### Basic Usage:

```jsx
startTransition(() => {
  addTransitionType('nav-forward');
  router.push('/detail/1');
});
```

### Type-Keyed Animations:

```jsx
<ViewTransition
  enter={{ 'nav-forward': 'slide-from-right', 'nav-back': 'slide-from-left', default: 'none' }}
  exit={{ 'nav-forward': 'slide-to-left', 'nav-back': 'slide-to-right', default: 'none' }}
  share={{ 'nav-forward': 'morph-forward', 'nav-back': 'morph-back', default: 'morph' }}
  default="none"
>
  <Page />
</ViewTransition>
```

### Asymmetric Animations:

`enter` and `exit` don't have to be symmetric. For example, fade in but slide out directionally:

```jsx
<ViewTransition
  enter={{ 'nav-forward': 'fade-in', 'nav-back': 'fade-in', default: 'none' }}
  exit={{ 'nav-forward': 'nav-forward', 'nav-back': 'nav-back', default: 'none' }}
  default="none"
>
```

### router.back() and Browser Back Button:

`router.back()` and the browser's back/forward buttons do **not** trigger view transitions (`popstate` is synchronous, incompatible with `startViewTransition`). Use `router.push()` with an explicit URL instead.

---

## Styling with View Transition Classes

### Props:

Values: `"auto"` (browser cross-fade), `"none"` (disabled), `"class-name"` (custom CSS), or `{ [type]: value }` for type-specific animations.

```jsx
<ViewTransition default="none" enter="slide-in" exit="slide-out" share="morph" />
```

If `default` is `"none"`, all triggers are off unless explicitly listed.

### CSS Pseudo-Elements:

- `::view-transition-old(.class)` — outgoing snapshot
- `::view-transition-new(.class)` — incoming snapshot
- `::view-transition-group(.class)` — container
- `::view-transition-image-pair(.class)` — old + new pair

### Example CSS Recipes:

```css
/* Slide from right */
@keyframes slide-from-right {
  from { transform: translateX(100%); }
}

::view-transition-new(.slide-from-right) {
  animation: slide-from-right 0.3s ease-out;
}

/* Fade in */
@keyframes fade-in {
  from { opacity: 0; }
}

::view-transition-new(.fade-in) {
  animation: fade-in 0.2s ease-out;
}

/* Morph (shared element) */
::view-transition-old(.morph),
::view-transition-new(.morph) {
  animation-duration: 0.3s;
  animation-timing-function: ease-in-out;
}
```

---

## Use `default="none"` Liberally

Without it, every VT fires the browser cross-fade on **every** transition — Suspense resolves, `useDeferredValue` updates, background revalidations. Always use `default="none"` and explicitly enable only desired triggers.

### Two Patterns Coexist:

**Pattern A — Directional slides:** Type-keyed VT on each page, fires during navigation.
**Pattern B — Suspense reveals:** Simple string props, fires when data loads (no type).

They coexist because they fire at different moments. `default="none"` on both prevents cross-interference. Always pair `enter` with `exit`. Place directional VTs in page components, not layouts.

---

## How Multiple VTs Interact

Every VT matching the trigger fires simultaneously in a single `document.startViewTransition`. VTs in **different** transitions (navigation vs later Suspense resolve) don't compete.

### Nested VT Limitation:

When a parent VT exits, nested VTs inside it do **not** fire their own enter/exit — only the outermost VT animates. Per-item staggered animations during page navigation are not possible today.

---

## Next.js Integration

For Next.js setup (`experimental.viewTransition` flag, `transitionTypes` prop on `next/link`, App Router patterns, Server Components), refer to the official Next.js documentation.

---

## Accessibility

Always add reduced motion CSS to your global stylesheet:

```css
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation: none !important;
  }
}
```

---

## Summary

View Transitions enable native-quality animations by:

1. **Shared elements** - Morph between list and detail views
2. **Suspense reveals** - Animate data loading states
3. **List identity** - Animate reorder, filter, sort
4. **Enter/exit** - Animate conditional content
5. **Directional navigation** - Communicate spatial depth with type-keyed animations
6. **default="none"** - Prevent unwanted animations, enable only what you need

These patterns create a cohesive, native-feeling experience without third-party animation libraries.
