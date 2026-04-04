# Re-render Optimization (MEDIUM Priority)

Unnecessary re-renders are a common source of performance issues in React applications. This section covers patterns to minimize re-renders through proper state management, memoization, and component architecture.

**Impact:** Smoother UI, reduced CPU usage, better battery life on mobile.

**Note:** If your project uses React Compiler (React 19+), many of these manual optimizations are handled automatically. However, understanding these patterns is still valuable for debugging and architectural decisions.

---

## Quick Reference

| Rule | Impact | When to Apply |
|------|--------|---------------|
| Defer State Reads to Usage Point | MEDIUM | State only used in callbacks |
| Extract to Memoized Components | MEDIUM | Expensive computations with early returns |
| Extract Default Non-primitive Values | MEDIUM | Memoized components with default props |
| Narrow Effect Dependencies | LOW | Effects with object dependencies |
| Subscribe to Derived State | MEDIUM | Continuous values (width, scroll) |
| Calculate Derived State During Rendering | MEDIUM | Values computed from props/state |
| Use Functional setState Updates | MEDIUM | State updates based on current state |
| Use Lazy State Initialization | MEDIUM | Expensive initial state computation |
| Don't Wrap Simple Expressions in useMemo | LOW-MEDIUM | Primitive results from simple expressions |
| Split Combined Hook Computations | MEDIUM | Hooks with independent dependencies |
| Put Interaction Logic in Event Handlers | MEDIUM | Side effects from user actions |
| Use Transitions for Non-Urgent Updates | MEDIUM | Frequent, non-critical updates |
| Use useDeferredValue for Expensive Renders | MEDIUM | Input triggering heavy computation |
| Use useRef for Transient Values | MEDIUM | Frequently changing non-UI values |
| Don't Define Components Inside Components | HIGH | Always - this is a critical bug |

---

## 1. Defer State Reads to Usage Point

**Impact:** MEDIUM (avoids unnecessary subscriptions)

Don't subscribe to dynamic state (searchParams, localStorage) if you only read it inside callbacks.

### Incorrect (subscribes to all searchParams changes):

```tsx
function ShareButton({ chatId }: { chatId: string }) {
  const searchParams = useSearchParams()

  const handleShare = () => {
    const ref = searchParams.get('ref')
    shareChat(chatId, { ref })
  }

  return <button onClick={handleShare}>Share</button>
}
```

**Problem:** Component re-renders every time any search param changes, even though it only reads `ref` on click.

### Correct (reads on demand, no subscription):

```tsx
function ShareButton({ chatId }: { chatId: string }) {
  const handleShare = () => {
    const params = new URLSearchParams(window.location.search)
    const ref = params.get('ref')
    shareChat(chatId, { ref })
  }

  return <button onClick={handleShare}>Share</button>
}
```

**Benefit:** Component only re-renders when its props change, not on every URL change.

---

## 2. Extract to Memoized Components

**Impact:** MEDIUM (enables early returns)

Extract expensive work into memoized components to enable early returns before computation.

### Incorrect (computes avatar even when loading):

```tsx
function Profile({ user, loading }: Props) {
  const avatar = useMemo(() => {
    const id = computeAvatarId(user)
    return <Avatar id={id} />
  }, [user])

  if (loading) return <Skeleton />
  return <div>{avatar}</div>
}
```

**Problem:** `computeAvatarId` runs even when `loading` is true.

### Correct (skips computation when loading):

```tsx
const UserAvatar = memo(function UserAvatar({ user }: { user: User }) {
  const id = useMemo(() => computeAvatarId(user), [user])
  return <Avatar id={id} />
})

function Profile({ user, loading }: Props) {
  if (loading) return <Skeleton />
  return (
    <div>
      <UserAvatar user={user} />
    </div>
  )
}
```

**Benefit:** When `loading` is true, `UserAvatar` never renders and `computeAvatarId` never runs.

---

## 3. Extract Default Non-primitive Values

**Impact:** MEDIUM (restores memoization)

When memoized components have default values for non-primitive optional parameters, extract them to constants.

### Incorrect (onClick has different values on every rerender):

```tsx
const UserAvatar = memo(function UserAvatar({ 
  onClick = () => {} 
}: { 
  onClick?: () => void 
}) {
  // ...
})

// Used without optional onClick
<UserAvatar />
```

**Problem:** `() => {}` creates a new function on every render, breaking memoization.

### Correct (stable default value):

```tsx
const NOOP = () => {}

const UserAvatar = memo(function UserAvatar({ 
  onClick = NOOP 
}: { 
  onClick?: () => void 
}) {
  // ...
})

// Used without optional onClick
<UserAvatar />
```

**Benefit:** Memoization works correctly because `NOOP` is always the same reference.

---

## 4. Narrow Effect Dependencies

**Impact:** LOW (minimizes effect re-runs)

Specify primitive dependencies instead of objects to minimize effect re-runs.

### Incorrect (re-runs on any user field change):

```tsx
useEffect(() => {
  console.log(user.id)
}, [user])
```

### Correct (re-runs only when id changes):

```tsx
useEffect(() => {
  console.log(user.id)
}, [user.id])
```

### For derived state, compute outside effect:

```tsx
// Incorrect: runs on width=767, 766, 765...
useEffect(() => {
  if (width < 768) {
    enableMobileMode()
  }
}, [width])

// Correct: runs only on boolean transition
const isMobile = width < 768
useEffect(() => {
  if (isMobile) {
    enableMobileMode()
  }
}, [isMobile])
```

---

## 5. Subscribe to Derived State

**Impact:** MEDIUM (reduces re-render frequency)

Subscribe to derived boolean state instead of continuous values to reduce re-render frequency.

### Incorrect (re-renders on every pixel change):

```tsx
function Sidebar() {
  const width = useWindowWidth()  // updates continuously
  const isMobile = width < 768
  return <nav className={isMobile ? 'mobile' : 'desktop'} />
}
```

**Problem:** Component re-renders on every pixel change (767, 766, 765...).

### Correct (re-renders only when boolean changes):

```tsx
function Sidebar() {
  const isMobile = useMediaQuery('(max-width: 767px)')
  return <nav className={isMobile ? 'mobile' : 'desktop'} />
}
```

**Benefit:** Component only re-renders when crossing the 768px threshold.

---

## 6. Calculate Derived State During Rendering

**Impact:** MEDIUM (avoids redundant renders and state drift)

If a value can be computed from current props/state, derive it during render instead of storing it in state.

### Incorrect (redundant state and effect):

```tsx
function Form() {
  const [firstName, setFirstName] = useState('First')
  const [lastName, setLastName] = useState('Last')
  const [fullName, setFullName] = useState('')

  useEffect(() => {
    setFullName(firstName + ' ' + lastName)
  }, [firstName, lastName])

  return <p>{fullName}</p>
}
```

**Problem:** Extra state, extra effect, extra render cycle.

### Correct (derive during render):

```tsx
function Form() {
  const [firstName, setFirstName] = useState('First')
  const [lastName, setLastName] = useState('Last')
  const fullName = firstName + ' ' + lastName

  return <p>{fullName}</p>
}
```

**Benefit:** Simpler code, fewer renders, no state drift.

**Reference:** [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)

---

## 7. Use Functional setState Updates

**Impact:** MEDIUM (prevents stale closures and unnecessary callback recreations)

When updating state based on the current state value, use the functional update form.

### Incorrect (requires state as dependency):

```tsx
function TodoList() {
  const [items, setItems] = useState(initialItems)
  
  // Callback must depend on items, recreated on every items change
  const addItems = useCallback((newItems: Item[]) => {
    setItems([...items, ...newItems])
  }, [items])  // ❌ items dependency causes recreations
  
  // Risk of stale closure if dependency is forgotten
  const removeItem = useCallback((id: string) => {
    setItems(items.filter(item => item.id !== id))
  }, [])  // ❌ Missing items dependency - will use stale items!
  
  return <ItemsEditor items={items} onAdd={addItems} onRemove={removeItem} />
}
```

### Correct (stable callbacks, no stale closures):

```tsx
function TodoList() {
  const [items, setItems] = useState(initialItems)
  
  // Stable callback, never recreated
  const addItems = useCallback((newItems: Item[]) => {
    setItems(curr => [...curr, ...newItems])
  }, [])  // ✅ No dependencies needed
  
  // Always uses latest state, no stale closure risk
  const removeItem = useCallback((id: string) => {
    setItems(curr => curr.filter(item => item.id !== id))
  }, [])  // ✅ Safe and stable
  
  return <ItemsEditor items={items} onAdd={addItems} onRemove={removeItem} />
}
```

**Benefits:**
1. Stable callback references
2. No stale closures
3. Fewer dependencies
4. Prevents bugs

---

## 8. Use Lazy State Initialization

**Impact:** MEDIUM (avoids wasted computation on every render)

Pass a function to `useState` for expensive initial values.

### Incorrect (runs on every render):

```tsx
function FilteredList({ items }: { items: Item[] }) {
  // buildSearchIndex() runs on EVERY render, even after initialization
  const [searchIndex, setSearchIndex] = useState(buildSearchIndex(items))
  const [query, setQuery] = useState('')
  
  return <SearchResults index={searchIndex} query={query} />
}
```

### Correct (runs only once):

```tsx
function FilteredList({ items }: { items: Item[] }) {
  // buildSearchIndex() runs ONLY on initial render
  const [searchIndex, setSearchIndex] = useState(() => buildSearchIndex(items))
  const [query, setQuery] = useState('')
  
  return <SearchResults index={searchIndex} query={query} />
}
```

**Use lazy initialization for:**
- localStorage/sessionStorage reads
- Building data structures (indexes, maps)
- Reading from the DOM
- Heavy transformations

---

## 9. Don't Wrap Simple Expressions in useMemo

**Impact:** LOW-MEDIUM (useMemo overhead > expression cost)

When an expression is simple and returns a primitive, don't wrap it in `useMemo`.

### Incorrect:

```tsx
function Header({ user, notifications }: Props) {
  const isLoading = useMemo(() => {
    return user.isLoading || notifications.isLoading
  }, [user.isLoading, notifications.isLoading])

  if (isLoading) return <Skeleton />
  // ...
}
```

### Correct:

```tsx
function Header({ user, notifications }: Props) {
  const isLoading = user.isLoading || notifications.isLoading

  if (isLoading) return <Skeleton />
  // ...
}
```

**Benefit:** Simpler code, no memoization overhead.

---

## 10. Split Combined Hook Computations

**Impact:** MEDIUM (avoids recomputing independent steps)

When a hook contains multiple independent tasks with different dependencies, split them.

### Incorrect (changing sortOrder recomputes filtering):

```tsx
const sortedProducts = useMemo(() => {
  const filtered = products.filter((p) => p.category === category)
  const sorted = filtered.toSorted((a, b) =>
    sortOrder === "asc" ? a.price - b.price : b.price - a.price
  )
  return sorted
}, [products, category, sortOrder])
```

### Correct (filtering only recomputes when products or category change):

```tsx
const filteredProducts = useMemo(
  () => products.filter((p) => p.category === category),
  [products, category]
)

const sortedProducts = useMemo(
  () =>
    filteredProducts.toSorted((a, b) =>
      sortOrder === "asc" ? a.price - b.price : b.price - a.price
    ),
  [filteredProducts, sortOrder]
)
```

---

## 11. Put Interaction Logic in Event Handlers

**Impact:** MEDIUM (avoids effect re-runs and duplicate side effects)

If a side effect is triggered by a specific user action, run it in that event handler.

### Incorrect (event modeled as state + effect):

```tsx
function Form() {
  const [submitted, setSubmitted] = useState(false)
  const theme = useContext(ThemeContext)

  useEffect(() => {
    if (submitted) {
      post('/api/register')
      showToast('Registered', theme)
    }
  }, [submitted, theme])

  return <button onClick={() => setSubmitted(true)}>Submit</button>
}
```

**Problem:** Effect runs when `theme` changes, potentially submitting twice.

### Correct (do it in the handler):

```tsx
function Form() {
  const theme = useContext(ThemeContext)

  function handleSubmit() {
    post('/api/register')
    showToast('Registered', theme)
  }

  return <button onClick={handleSubmit}>Submit</button>
}
```

---

## 12. Use Transitions for Non-Urgent Updates

**Impact:** MEDIUM (maintains UI responsiveness)

Mark frequent, non-urgent state updates as transitions.

### Incorrect (blocks UI on every scroll):

```tsx
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0)
  useEffect(() => {
    const handler = () => setScrollY(window.scrollY)
    window.addEventListener('scroll', handler, { passive: true })
    return () => window.removeEventListener('scroll', handler)
  }, [])
}
```

### Correct (non-blocking updates):

```tsx
import { startTransition } from 'react'

function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0)
  useEffect(() => {
    const handler = () => {
      startTransition(() => setScrollY(window.scrollY))
    }
    window.addEventListener('scroll', handler, { passive: true })
    return () => window.removeEventListener('scroll', handler)
  }, [])
}
```

---

## 13. Use useDeferredValue for Expensive Renders

**Impact:** MEDIUM (keeps input responsive during heavy computation)

When user input triggers expensive computations, use `useDeferredValue`.

### Incorrect (input feels laggy while filtering):

```tsx
function Search({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('')
  const filtered = items.filter(item => fuzzyMatch(item, query))

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ResultsList results={filtered} />
    </>
  )
}
```

### Correct (input stays snappy, results render when ready):

```tsx
function Search({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('')
  const deferredQuery = useDeferredValue(query)
  const filtered = useMemo(
    () => items.filter(item => fuzzyMatch(item, deferredQuery)),
    [items, deferredQuery]
  )
  const isStale = query !== deferredQuery

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <div style={{ opacity: isStale ? 0.7 : 1 }}>
        <ResultsList results={filtered} />
      </div>
    </>
  )
}
```

---

## 14. Use useRef for Transient Values

**Impact:** MEDIUM (avoids unnecessary re-renders on frequent updates)

When a value changes frequently and you don't want a re-render on every update, use `useRef`.

### Incorrect (renders every update):

```tsx
function Tracker() {
  const [lastX, setLastX] = useState(0)

  useEffect(() => {
    const onMove = (e: MouseEvent) => setLastX(e.clientX)
    window.addEventListener('mousemove', onMove)
    return () => window.removeEventListener('mousemove', onMove)
  }, [])

  return (
    <div
      style={{
        position: 'fixed',
        top: 0,
        left: lastX,
        width: 8,
        height: 8,
        background: 'black',
      }}
    />
  )
}
```

### Correct (no re-render for tracking):

```tsx
function Tracker() {
  const lastXRef = useRef(0)
  const dotRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const onMove = (e: MouseEvent) => {
      lastXRef.current = e.clientX
      const node = dotRef.current
      if (node) {
        node.style.transform = `translateX(${e.clientX}px)`
      }
    }
    window.addEventListener('mousemove', onMove)
    return () => window.removeEventListener('mousemove', onMove)
  }, [])

  return (
    <div
      ref={dotRef}
      style={{
        position: 'fixed',
        top: 0,
        left: 0,
        width: 8,
        height: 8,
        background: 'black',
        transform: 'translateX(0px)',
      }}
    />
  )
}
```

---

## 15. Don't Define Components Inside Components

**Impact:** HIGH (prevents remount on every render)

Defining a component inside another component creates a new component type on every render.

### Incorrect (remounts on every render):

```tsx
function UserProfile({ user, theme }) {
  // Defined inside to access `theme` - BAD
  const Avatar = () => (
    <img
      src={user.avatarUrl}
      className={theme === 'dark' ? 'avatar-dark' : 'avatar-light'}
    />
  )

  return (
    <div>
      <Avatar />
    </div>
  )
}
```

**Problem:** Every render creates a new `Avatar` component type. React unmounts and remounts it, losing state and recreating DOM.

### Correct (pass props instead):

```tsx
function Avatar({ src, theme }: { src: string; theme: string }) {
  return (
    <img
      src={src}
      className={theme === 'dark' ? 'avatar-dark' : 'avatar-light'}
    />
  )
}

function UserProfile({ user, theme }) {
  return (
    <div>
      <Avatar src={user.avatarUrl} theme={theme} />
    </div>
  )
}
```

**Symptoms of this bug:**
- Input fields lose focus on every keystroke
- Animations restart unexpectedly
- useEffect cleanup/setup runs on every parent render
- Scroll position resets

---

## Summary

Re-render optimization is about **minimizing unnecessary work**:

1. **Defer state reads** to callbacks when possible
2. **Extract memoized components** for expensive work
3. **Use stable references** for default props
4. **Narrow dependencies** to primitives
5. **Subscribe to derived state** instead of continuous values
6. **Derive during render** instead of storing in state
7. **Use functional setState** for stable callbacks
8. **Lazy initialize** expensive state
9. **Don't memo simple expressions**
10. **Split independent computations**
11. **Put interaction logic in handlers**
12. **Use transitions** for non-urgent updates
13. **Use useDeferredValue** for expensive renders
14. **Use useRef** for transient values
15. **Never define components inside components**

These patterns create smoother, more responsive UIs with less CPU usage.
