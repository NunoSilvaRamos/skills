# Advanced Patterns (LOW Priority)

Advanced React patterns for edge cases and specialized scenarios. These are lower priority but important for specific use cases involving Effect Events, refs, and initialization logic.

**Impact:** Prevents subtle bugs in advanced React usage.

---

## Quick Reference

| Rule | Impact | When to Apply |
|------|--------|---------------|
| Do Not Put Effect Events in Dependency Arrays | LOW | Using useEffectEvent |
| Store Event Handlers in Refs | LOW | Stable subscriptions without re-subscribing |
| Initialize App Once, Not Per Mount | LOW-MEDIUM | App-wide initialization |
| useEffectEvent for Stable Callback Refs | LOW | Accessing latest values without dependencies |

---

## 1. Do Not Put Effect Events in Dependency Arrays

**Impact:** LOW (avoids unnecessary effect re-runs and lint errors)

Effect Event functions do not have a stable identity. Do not include the function returned by `useEffectEvent` in a `useEffect` dependency array.

### Incorrect (Effect Event added as a dependency):

```tsx
import { useEffect, useEffectEvent } from 'react'

function ChatRoom({ roomId, onConnected }: {
  roomId: string
  onConnected: () => void
}) {
  const handleConnected = useEffectEvent(onConnected)

  useEffect(() => {
    const connection = createConnection(roomId)
    connection.on('connected', handleConnected)
    connection.connect()

    return () => connection.disconnect()
  }, [roomId, handleConnected])  // ❌ handleConnected shouldn't be here
}
```

**Problem:** Including the Effect Event in dependencies makes the effect re-run every render.

### Correct (depend on reactive values, not the Effect Event):

```tsx
import { useEffect, useEffectEvent } from 'react'

function ChatRoom({ roomId, onConnected }: {
  roomId: string
  onConnected: () => void
}) {
  const handleConnected = useEffectEvent(onConnected)

  useEffect(() => {
    const connection = createConnection(roomId)
    connection.on('connected', handleConnected)
    connection.connect()

    return () => connection.disconnect()
  }, [roomId])  // ✅ Only roomId
}
```

**Reference:** [React useEffectEvent: Effect Event in deps](https://react.dev/reference/react/useEffectEvent#effect-event-in-deps)

---

## 2. Store Event Handlers in Refs

**Impact:** LOW (stable subscriptions)

Store callbacks in refs when used in effects that shouldn't re-subscribe on callback changes.

### Incorrect (re-subscribes on every render):

```tsx
function useWindowEvent(event: string, handler: (e) => void) {
  useEffect(() => {
    window.addEventListener(event, handler)
    return () => window.removeEventListener(event, handler)
  }, [event, handler])  // ❌ Re-subscribes when handler changes
}
```

**Problem:** Every time the parent component re-renders with a new handler function, the effect unsubscribes and resubscribes.

### Correct (stable subscription):

```tsx
function useWindowEvent(event: string, handler: (e) => void) {
  const handlerRef = useRef(handler)
  
  useEffect(() => {
    handlerRef.current = handler
  }, [handler])

  useEffect(() => {
    const listener = (e) => handlerRef.current(e)
    window.addEventListener(event, listener)
    return () => window.removeEventListener(event, listener)
  }, [event])  // ✅ Only re-subscribes when event changes
}
```

### Alternative: use `useEffectEvent` (React 19+):

```tsx
import { useEffectEvent } from 'react'

function useWindowEvent(event: string, handler: (e) => void) {
  const onEvent = useEffectEvent(handler)

  useEffect(() => {
    window.addEventListener(event, onEvent)
    return () => window.removeEventListener(event, onEvent)
  }, [event])
}
```

**Benefit:** `useEffectEvent` provides a cleaner API for the same pattern: it creates a stable function reference that always calls the latest version of the handler.

---

## 3. Initialize App Once, Not Per Mount

**Impact:** LOW-MEDIUM (avoids duplicate init in development)

Do not put app-wide initialization that must run once per app load inside `useEffect([])` of a component. Components can remount and effects will re-run.

### Incorrect (runs twice in dev, re-runs on remount):

```tsx
function App() {
  useEffect(() => {
    loadFromStorage()
    checkAuthToken()
    initAnalytics()
  }, [])

  return <Router />
}
```

**Problem:** 
- In React 18+ Strict Mode (development), effects run twice
- If the component unmounts and remounts, initialization runs again
- Can cause duplicate analytics events, multiple auth checks, etc.

### Correct (once per app load):

```tsx
let didInit = false

function App() {
  useEffect(() => {
    if (didInit) return
    didInit = true
    
    loadFromStorage()
    checkAuthToken()
    initAnalytics()
  }, [])

  return <Router />
}
```

### Alternative: Initialize at module level

```tsx
// app-init.ts
let didInit = false

export function initApp() {
  if (didInit) return
  didInit = true
  
  loadFromStorage()
  checkAuthToken()
  initAnalytics()
}

// main.tsx
import { initApp } from './app-init'

initApp()

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)
```

**Benefit:** Initialization happens once before React even starts, completely outside the component lifecycle.

**Reference:** [Initializing the application](https://react.dev/learn/you-might-not-need-an-effect#initializing-the-application)

---

## 4. useEffectEvent for Stable Callback Refs

**Impact:** LOW (prevents effect re-runs)

Access latest values in callbacks without adding them to dependency arrays. Prevents effect re-runs while avoiding stale closures.

### Incorrect (effect re-runs on every callback change):

```tsx
function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
  const [query, setQuery] = useState('')

  useEffect(() => {
    const timeout = setTimeout(() => onSearch(query), 300)
    return () => clearTimeout(timeout)
  }, [query, onSearch])  // ❌ Re-runs when onSearch changes
}
```

**Problem:** If the parent passes a new `onSearch` function on every render, the debounce effect resets constantly.

### Correct (using React's useEffectEvent):

```tsx
import { useEffectEvent } from 'react'

function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
  const [query, setQuery] = useState('')
  const onSearchEvent = useEffectEvent(onSearch)

  useEffect(() => {
    const timeout = setTimeout(() => onSearchEvent(query), 300)
    return () => clearTimeout(timeout)
  }, [query])  // ✅ Only re-runs when query changes
}
```

**Benefit:** The effect only re-runs when `query` changes, but always calls the latest `onSearch` function.

### When to use useEffectEvent:

- **Callbacks in effects** that shouldn't trigger re-runs
- **Event handlers in subscriptions** (WebSocket, intervals)
- **Accessing latest props/state** in long-running operations
- **Debounced/throttled callbacks** that need latest values

### When NOT to use useEffectEvent:

- **Regular event handlers** (onClick, onChange) - use normal functions
- **Memoized callbacks** - use useCallback instead
- **Synchronous operations** - just read the value directly

---

## Summary

Advanced patterns are about **handling edge cases correctly**:

1. **Don't put Effect Events in deps** - they're intentionally unstable
2. **Store handlers in refs** for stable subscriptions
3. **Initialize once** with module-level guards
4. **Use useEffectEvent** for latest values without dependencies

These patterns prevent subtle bugs in advanced React usage scenarios.
