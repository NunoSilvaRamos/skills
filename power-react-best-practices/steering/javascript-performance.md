# JavaScript Performance (LOW-MEDIUM Priority)

JavaScript micro-optimizations focus on reducing computational overhead in hot paths. While individually small, these optimizations compound in data-heavy applications with frequent operations.

**Impact:** Reduced CPU usage, smoother interactions, better battery life.

---

## Quick Reference

| Rule | Impact | When to Apply |
|------|--------|---------------|
| Avoid Layout Thrashing | MEDIUM | DOM manipulation with style reads |
| Build Index Maps for Repeated Lookups | LOW-MEDIUM | Multiple .find() calls |
| Cache Property Access in Loops | LOW-MEDIUM | Nested property access in loops |
| Cache Repeated Function Calls | MEDIUM | Same function called with same inputs |
| Cache Storage API Calls | LOW-MEDIUM | localStorage/sessionStorage reads |
| Combine Multiple Array Iterations | LOW-MEDIUM | Multiple .filter() or .map() calls |
| Early Length Check for Array Comparisons | MEDIUM-HIGH | Expensive array comparisons |
| Early Return from Functions | LOW-MEDIUM | Validation or search functions |
| Hoist RegExp Creation | LOW-MEDIUM | RegExp in render or loops |
| Use Loop for Min/Max Instead of Sort | LOW | Finding min/max values |
| Use Set/Map for O(1) Lookups | LOW-MEDIUM | Repeated membership checks |
| Use toSorted() for Immutability | MEDIUM-HIGH | Sorting React state/props |
| Use flatMap to Map and Filter | LOW-MEDIUM | Chained .map().filter() |
| Defer Non-Critical Work | MEDIUM | Analytics, logging, prefetching |

---

## 1. Avoid Layout Thrashing

**Impact:** MEDIUM (prevents forced synchronous layouts)

Avoid interleaving style writes with layout reads.

### Incorrect (interleaved reads and writes force reflows):

```typescript
function layoutThrashing(element: HTMLElement) {
  element.style.width = '100px'
  const width = element.offsetWidth  // Forces reflow
  element.style.height = '200px'
  const height = element.offsetHeight  // Forces another reflow
}
```

### Correct (batch writes, then read once):

```typescript
function updateElementStyles(element: HTMLElement) {
  // Batch all writes together
  element.style.width = '100px'
  element.style.height = '200px'
  element.style.backgroundColor = 'blue'
  
  // Read after all writes are done (single reflow)
  const { width, height } = element.getBoundingClientRect()
}
```

### Better: use CSS classes

```tsx
function Box({ isHighlighted }: { isHighlighted: boolean }) {
  return (
    <div className={isHighlighted ? 'highlighted-box' : ''}>
      Content
    </div>
  )
}
```

---

## 2. Build Index Maps for Repeated Lookups

**Impact:** LOW-MEDIUM (O(n) → O(1) per lookup)

Multiple `.find()` calls by the same key should use a Map.

### Incorrect (O(n) per lookup):

```typescript
function processOrders(orders: Order[], users: User[]) {
  return orders.map(order => ({
    ...order,
    user: users.find(u => u.id === order.userId)
  }))
}
```

### Correct (O(1) per lookup):

```typescript
function processOrders(orders: Order[], users: User[]) {
  const userById = new Map(users.map(u => [u.id, u]))

  return orders.map(order => ({
    ...order,
    user: userById.get(order.userId)
  }))
}
```

For 1000 orders × 1000 users: 1M ops → 2K ops.

---

## 3. Cache Property Access in Loops

**Impact:** LOW-MEDIUM (reduces lookups)

Cache object property lookups in hot paths.

### Incorrect (3 lookups × N iterations):

```typescript
for (let i = 0; i < arr.length; i++) {
  process(obj.config.settings.value)
}
```

### Correct (1 lookup total):

```typescript
const value = obj.config.settings.value
const len = arr.length
for (let i = 0; i < len; i++) {
  process(value)
}
```

---

## 4. Cache Repeated Function Calls

**Impact:** MEDIUM (avoid redundant computation)

Use a module-level Map to cache function results.

### Incorrect (redundant computation):

```typescript
function ProjectList({ projects }: { projects: Project[] }) {
  return (
    <div>
      {projects.map(project => {
        const slug = slugify(project.name)  // Called 100+ times
        return <ProjectCard key={project.id} slug={slug} />
      })}
    </div>
  )
}
```

### Correct (cached results):

```typescript
const slugifyCache = new Map<string, string>()

function cachedSlugify(text: string): string {
  if (slugifyCache.has(text)) {
    return slugifyCache.get(text)!
  }
  const result = slugify(text)
  slugifyCache.set(text, result)
  return result
}

function ProjectList({ projects }: { projects: Project[] }) {
  return (
    <div>
      {projects.map(project => {
        const slug = cachedSlugify(project.name)
        return <ProjectCard key={project.id} slug={slug} />
      })}
    </div>
  )
}
```

---

## 5. Cache Storage API Calls

**Impact:** LOW-MEDIUM (reduces expensive I/O)

`localStorage`, `sessionStorage`, and `document.cookie` are synchronous and expensive.

### Incorrect (reads storage on every call):

```typescript
function getTheme() {
  return localStorage.getItem('theme') ?? 'light'
}
```

### Correct (Map cache):

```typescript
const storageCache = new Map<string, string | null>()

function getLocalStorage(key: string) {
  if (!storageCache.has(key)) {
    storageCache.set(key, localStorage.getItem(key))
  }
  return storageCache.get(key)
}

function setLocalStorage(key: string, value: string) {
  localStorage.setItem(key, value)
  storageCache.set(key, value)
}
```

---

## 6. Combine Multiple Array Iterations

**Impact:** LOW-MEDIUM (reduces iterations)

Multiple `.filter()` or `.map()` calls iterate the array multiple times.

### Incorrect (3 iterations):

```typescript
const admins = users.filter(u => u.isAdmin)
const testers = users.filter(u => u.isTester)
const inactive = users.filter(u => !u.isActive)
```

### Correct (1 iteration):

```typescript
const admins: User[] = []
const testers: User[] = []
const inactive: User[] = []

for (const user of users) {
  if (user.isAdmin) admins.push(user)
  if (user.isTester) testers.push(user)
  if (!user.isActive) inactive.push(user)
}
```

---

## 7. Early Length Check for Array Comparisons

**Impact:** MEDIUM-HIGH (avoids expensive operations)

Check lengths first before expensive comparisons.

### Incorrect (always runs expensive comparison):

```typescript
function hasChanges(current: string[], original: string[]) {
  return current.sort().join() !== original.sort().join()
}
```

### Correct (O(1) length check first):

```typescript
function hasChanges(current: string[], original: string[]) {
  if (current.length !== original.length) {
    return true
  }
  const currentSorted = current.toSorted()
  const originalSorted = original.toSorted()
  for (let i = 0; i < currentSorted.length; i++) {
    if (currentSorted[i] !== originalSorted[i]) {
      return true
    }
  }
  return false
}
```

---

## 8. Early Return from Functions

**Impact:** LOW-MEDIUM (avoids unnecessary computation)

Return early when result is determined.

### Incorrect (processes all items):

```typescript
function validateUsers(users: User[]) {
  let hasError = false
  let errorMessage = ''
  
  for (const user of users) {
    if (!user.email) {
      hasError = true
      errorMessage = 'Email required'
    }
  }
  
  return hasError ? { valid: false, error: errorMessage } : { valid: true }
}
```

### Correct (returns immediately):

```typescript
function validateUsers(users: User[]) {
  for (const user of users) {
    if (!user.email) {
      return { valid: false, error: 'Email required' }
    }
  }
  return { valid: true }
}
```

---

## 9. Hoist RegExp Creation

**Impact:** LOW-MEDIUM (avoids recreation)

Don't create RegExp inside render.

### Incorrect (new RegExp every render):

```tsx
function Highlighter({ text, query }: Props) {
  const regex = new RegExp(`(${query})`, 'gi')
  const parts = text.split(regex)
  return <>{parts.map((part, i) => ...)}</>
}
```

### Correct (memoize):

```tsx
function Highlighter({ text, query }: Props) {
  const regex = useMemo(
    () => new RegExp(`(${escapeRegex(query)})`, 'gi'),
    [query]
  )
  const parts = text.split(regex)
  return <>{parts.map((part, i) => ...)}</>
}
```

---

## 10. Use Loop for Min/Max Instead of Sort

**Impact:** LOW (O(n) instead of O(n log n))

Finding min/max only requires a single pass.

### Incorrect (O(n log n)):

```typescript
function getLatestProject(projects: Project[]) {
  const sorted = [...projects].sort((a, b) => b.updatedAt - a.updatedAt)
  return sorted[0]
}
```

### Correct (O(n)):

```typescript
function getLatestProject(projects: Project[]) {
  if (projects.length === 0) return null
  
  let latest = projects[0]
  for (let i = 1; i < projects.length; i++) {
    if (projects[i].updatedAt > latest.updatedAt) {
      latest = projects[i]
    }
  }
  return latest
}
```

---

## 11. Use Set/Map for O(1) Lookups

**Impact:** LOW-MEDIUM (O(n) → O(1))

Convert arrays to Set/Map for repeated membership checks.

### Incorrect (O(n) per check):

```typescript
const allowedIds = ['a', 'b', 'c']
items.filter(item => allowedIds.includes(item.id))
```

### Correct (O(1) per check):

```typescript
const allowedIds = new Set(['a', 'b', 'c'])
items.filter(item => allowedIds.has(item.id))
```

---

## 12. Use toSorted() for Immutability

**Impact:** MEDIUM-HIGH (prevents mutation bugs)

`.sort()` mutates the array, which breaks React's immutability model.

### Incorrect (mutates original):

```typescript
const sorted = useMemo(
  () => users.sort((a, b) => a.name.localeCompare(b.name)),
  [users]
)
```

### Correct (creates new array):

```typescript
const sorted = useMemo(
  () => users.toSorted((a, b) => a.name.localeCompare(b.name)),
  [users]
)
```

---

## 13. Use flatMap to Map and Filter

**Impact:** LOW-MEDIUM (eliminates intermediate array)

Chaining `.map().filter(Boolean)` creates an intermediate array.

### Incorrect (2 iterations):

```typescript
const userNames = users
  .map(user => user.isActive ? user.name : null)
  .filter(Boolean)
```

### Correct (1 iteration):

```typescript
const userNames = users.flatMap(user =>
  user.isActive ? [user.name] : []
)
```

---

## 14. Defer Non-Critical Work

**Impact:** MEDIUM (keeps UI responsive)

Use `requestIdleCallback()` to schedule non-critical work during browser idle periods.

### Incorrect (blocks main thread):

```typescript
function handleSearch(query: string) {
  const results = searchItems(query)
  setResults(results)
  
  analytics.track('search', { query })
  saveToRecentSearches(query)
}
```

### Correct (defers to idle time):

```typescript
function handleSearch(query: string) {
  const results = searchItems(query)
  setResults(results)
  
  requestIdleCallback(() => {
    analytics.track('search', { query })
  })
  
  requestIdleCallback(() => {
    saveToRecentSearches(query)
  })
}
```

---

## Summary

JavaScript performance is about **reducing computational overhead**:

1. **Batch DOM operations** to avoid layout thrashing
2. **Build index maps** for repeated lookups
3. **Cache property access** in loops
4. **Cache function results** with Map
5. **Cache storage reads** in memory
6. **Combine iterations** into single loops
7. **Check lengths first** before expensive comparisons
8. **Return early** when result is determined
9. **Hoist RegExp** creation outside render
10. **Use loops** for min/max instead of sort
11. **Use Set/Map** for O(1) lookups
12. **Use toSorted()** for immutability
13. **Use flatMap** to combine map and filter
14. **Defer non-critical work** to idle time

These micro-optimizations compound in data-heavy applications.
