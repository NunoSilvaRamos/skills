# Client Optimization (MEDIUM-HIGH Priority)

Client-side performance focuses on efficient data fetching, event handling, and storage management. These patterns reduce redundant network requests, improve scroll performance, and optimize browser storage usage.

**Impact:** Reduced network traffic, smoother scrolling, better offline experience.

---

## Quick Reference

| Rule | Impact | When to Apply |
|------|--------|---------------|
| Use SWR for Automatic Deduplication | MEDIUM-HIGH | Client-side data fetching across components |
| Deduplicate Global Event Listeners | LOW | Multiple components listening to same global events |
| Use Passive Event Listeners for Scrolling | MEDIUM | Touch and wheel event listeners |
| Version and Minimize localStorage Data | MEDIUM | Storing user preferences or cached data |

---

## 1. Use SWR for Automatic Deduplication

**Impact:** MEDIUM-HIGH (automatic deduplication)

SWR enables request deduplication, caching, and revalidation across component instances.

### Incorrect (no deduplication, each instance fetches):

```tsx
function UserList() {
  const [users, setUsers] = useState([])
  useEffect(() => {
    fetch('/api/users')
      .then(r => r.json())
      .then(setUsers)
  }, [])
}
```

**Problem:** If you render `<UserList />` multiple times, each instance makes its own fetch request.

### Correct (multiple instances share one request):

```tsx
import useSWR from 'swr'

function UserList() {
  const { data: users } = useSWR('/api/users', fetcher)
}
```

**Benefit:** All `<UserList />` instances share a single request. SWR automatically:
- Deduplicates concurrent requests
- Caches responses
- Revalidates on focus/reconnect
- Provides loading and error states

### For immutable data:

```tsx
import { useImmutableSWR } from '@/lib/swr'

function StaticContent() {
  const { data } = useImmutableSWR('/api/config', fetcher)
}
```

**Benefit:** Data is fetched once and never revalidated (perfect for static config).

### For mutations:

```tsx
import { useSWRMutation } from 'swr/mutation'

function UpdateButton() {
  const { trigger } = useSWRMutation('/api/user', updateUser)
  return <button onClick={() => trigger()}>Update</button>
}
```

**Benefit:** Optimistic updates, automatic revalidation, error handling.

### SWR Configuration Example:

```tsx
import { SWRConfig } from 'swr'

function App({ children }) {
  return (
    <SWRConfig
      value={{
        fetcher: (url) => fetch(url).then(r => r.json()),
        revalidateOnFocus: true,
        revalidateOnReconnect: true,
        dedupingInterval: 2000, // Dedupe requests within 2s
      }}
    >
      {children}
    </SWRConfig>
  )
}
```

### When to Use SWR:

- **Multiple components** need the same data
- **Real-time updates** are important (revalidate on focus)
- **Offline support** is needed (cache + revalidation)
- **Optimistic updates** for better UX

### When NOT to Use SWR:

- **One-time fetches** that never change
- **Server Components** (use native fetch with caching)
- **Form submissions** (use useSWRMutation or Server Actions)

**Reference:** [https://swr.vercel.app](https://swr.vercel.app)

---

## 2. Deduplicate Global Event Listeners

**Impact:** LOW (single listener for N components)

Use `useSWRSubscription()` to share global event listeners across component instances.

### Incorrect (N instances = N listeners):

```tsx
function useKeyboardShortcut(key: string, callback: () => void) {
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      if (e.metaKey && e.key === key) {
        callback()
      }
    }
    window.addEventListener('keydown', handler)
    return () => window.removeEventListener('keydown', handler)
  }, [key, callback])
}
```

**Problem:** When using the `useKeyboardShortcut` hook multiple times, each instance will register a new listener. 10 components = 10 event listeners.

### Correct (N instances = 1 listener):

```tsx
import useSWRSubscription from 'swr/subscription'

// Module-level Map to track callbacks per key
const keyCallbacks = new Map<string, Set<() => void>>()

function useKeyboardShortcut(key: string, callback: () => void) {
  // Register this callback in the Map
  useEffect(() => {
    if (!keyCallbacks.has(key)) {
      keyCallbacks.set(key, new Set())
    }
    keyCallbacks.get(key)!.add(callback)

    return () => {
      const set = keyCallbacks.get(key)
      if (set) {
        set.delete(callback)
        if (set.size === 0) {
          keyCallbacks.delete(key)
        }
      }
    }
  }, [key, callback])

  useSWRSubscription('global-keydown', () => {
    const handler = (e: KeyboardEvent) => {
      if (e.metaKey && keyCallbacks.has(e.key)) {
        keyCallbacks.get(e.key)!.forEach(cb => cb())
      }
    }
    window.addEventListener('keydown', handler)
    return () => window.removeEventListener('keydown', handler)
  })
}

function Profile() {
  // Multiple shortcuts will share the same listener
  useKeyboardShortcut('p', () => { /* open profile */ }) 
  useKeyboardShortcut('k', () => { /* open command palette */ })
  // ...
}
```

**Benefit:** Only one `keydown` listener is registered, regardless of how many components use the hook.

### Other Use Cases:

- **Window resize** - Share one listener across multiple responsive components
- **Online/offline** - Share network status across components
- **Visibility change** - Pause/resume features when tab is hidden
- **Media queries** - Share breakpoint detection

### Pattern:

1. Use a module-level Map/Set to track callbacks
2. Use `useSWRSubscription` to register the global listener once
3. Dispatch to registered callbacks when event fires

---

## 3. Use Passive Event Listeners for Scrolling Performance

**Impact:** MEDIUM (eliminates scroll delay caused by event listeners)

Add `{ passive: true }` to touch and wheel event listeners to enable immediate scrolling. Browsers normally wait for listeners to finish to check if `preventDefault()` is called, causing scroll delay.

### Incorrect:

```typescript
useEffect(() => {
  const handleTouch = (e: TouchEvent) => console.log(e.touches[0].clientX)
  const handleWheel = (e: WheelEvent) => console.log(e.deltaY)
  
  document.addEventListener('touchstart', handleTouch)
  document.addEventListener('wheel', handleWheel)
  
  return () => {
    document.removeEventListener('touchstart', handleTouch)
    document.removeEventListener('wheel', handleWheel)
  }
}, [])
```

**Problem:** Browser waits for `handleTouch` and `handleWheel` to complete before scrolling, causing janky scrolling.

### Correct:

```typescript
useEffect(() => {
  const handleTouch = (e: TouchEvent) => console.log(e.touches[0].clientX)
  const handleWheel = (e: WheelEvent) => console.log(e.deltaY)
  
  document.addEventListener('touchstart', handleTouch, { passive: true })
  document.addEventListener('wheel', handleWheel, { passive: true })
  
  return () => {
    document.removeEventListener('touchstart', handleTouch)
    document.removeEventListener('wheel', handleWheel)
  }
}, [])
```

**Benefit:** Browser can scroll immediately without waiting for the listener.

### When to Use Passive:

- **Tracking/analytics** - Logging scroll position
- **Lazy loading** - Detecting when elements enter viewport
- **Scroll animations** - Parallax effects (read-only)
- **Any listener that doesn't call `preventDefault()`**

### When NOT to Use Passive:

- **Custom swipe gestures** - Need to prevent default scroll
- **Custom zoom controls** - Need to prevent default pinch-zoom
- **Pull-to-refresh** - Need to prevent default overscroll
- **Any listener that needs `preventDefault()`**

### Events That Benefit from Passive:

- `touchstart`
- `touchmove`
- `touchend`
- `wheel`
- `mousewheel`

### Performance Impact:

Without passive listeners, scroll can be delayed by 100-500ms on mobile devices, causing visible jank. Passive listeners eliminate this delay entirely.

---

## 4. Version and Minimize localStorage Data

**Impact:** MEDIUM (prevents schema conflicts, reduces storage size)

Add version prefix to keys and store only needed fields. Prevents schema conflicts and accidental storage of sensitive data.

### Incorrect:

```typescript
// No version, stores everything, no error handling
localStorage.setItem('userConfig', JSON.stringify(fullUserObject))
const data = localStorage.getItem('userConfig')
```

**Problems:**
- No versioning - schema changes break old data
- Stores entire object - wastes space, may include sensitive data
- No error handling - throws in incognito mode

### Correct:

```typescript
const VERSION = 'v2'

function saveConfig(config: { theme: string; language: string }) {
  try {
    localStorage.setItem(`userConfig:${VERSION}`, JSON.stringify(config))
  } catch {
    // Throws in incognito/private browsing, quota exceeded, or disabled
  }
}

function loadConfig() {
  try {
    const data = localStorage.getItem(`userConfig:${VERSION}`)
    return data ? JSON.parse(data) : null
  } catch {
    return null
  }
}

// Migration from v1 to v2
function migrate() {
  try {
    const v1 = localStorage.getItem('userConfig:v1')
    if (v1) {
      const old = JSON.parse(v1)
      saveConfig({ theme: old.darkMode ? 'dark' : 'light', language: old.lang })
      localStorage.removeItem('userConfig:v1')
    }
  } catch {}
}
```

**Benefits:**
- Versioning allows schema evolution
- Migration path for old data
- Error handling prevents crashes

### Store minimal fields from server responses:

```typescript
// User object has 20+ fields, only store what UI needs
function cachePrefs(user: FullUser) {
  try {
    localStorage.setItem('prefs:v1', JSON.stringify({
      theme: user.preferences.theme,
      notifications: user.preferences.notifications
    }))
  } catch {}
}
```

**Benefit:** Reduced storage size, prevents storing tokens/PII/internal flags.

### Always wrap in try-catch

`getItem()` and `setItem()` throw in:
- **Incognito/private browsing** (Safari, Firefox)
- **Quota exceeded** (5-10MB limit)
- **localStorage disabled** (user settings, enterprise policy)

### Best Practices:

1. **Version all keys** - `key:v1`, `key:v2`
2. **Store minimal data** - Only what the UI needs
3. **Never store sensitive data** - Tokens, passwords, PII
4. **Always try-catch** - Handle storage failures gracefully
5. **Provide migration path** - Convert old versions to new
6. **Set expiration** - Add timestamp, clean up old data

### Example with Expiration:

```typescript
interface CachedData<T> {
  version: string
  data: T
  timestamp: number
}

function saveWithExpiry<T>(key: string, data: T, ttlMs: number) {
  try {
    const cached: CachedData<T> = {
      version: 'v1',
      data,
      timestamp: Date.now()
    }
    localStorage.setItem(key, JSON.stringify(cached))
  } catch {}
}

function loadWithExpiry<T>(key: string, ttlMs: number): T | null {
  try {
    const item = localStorage.getItem(key)
    if (!item) return null
    
    const cached: CachedData<T> = JSON.parse(item)
    const age = Date.now() - cached.timestamp
    
    if (age > ttlMs) {
      localStorage.removeItem(key)
      return null
    }
    
    return cached.data
  } catch {
    return null
  }
}
```

---

## Summary

Client-side optimization is about **reducing redundancy and improving responsiveness**:

1. **Use SWR** for automatic request deduplication and caching
2. **Deduplicate event listeners** with useSWRSubscription
3. **Use passive listeners** for smooth scrolling
4. **Version localStorage** and store minimal data

These patterns create a faster, more efficient client-side experience with less network traffic and smoother interactions.
