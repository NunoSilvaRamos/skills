# Async Optimization (CRITICAL Priority)

Eliminating request waterfalls is the highest-impact performance optimization you can make. Sequential async operations create cascading delays where each request waits for the previous one to complete. This section covers patterns to maximize parallelism and minimize blocking.

**Impact:** 2-10× performance improvement in data-heavy applications.

---

## Quick Reference

| Rule | Impact | When to Apply |
|------|--------|---------------|
| Promise.all() for Independent Operations | CRITICAL | Multiple async operations with no dependencies |
| Defer Await Until Needed | HIGH | Conditional branches where not all paths need the data |
| Check Cheap Conditions Before Async Flags | HIGH | Feature flags combined with synchronous guards |
| Dependency-Based Parallelization | CRITICAL | Operations with partial dependencies |
| Prevent Waterfall Chains in API Routes | CRITICAL | API routes and Server Actions |
| Strategic Suspense Boundaries | HIGH | React Server Components with slow data fetching |

---

## 1. Promise.all() for Independent Operations

**Impact:** CRITICAL (2-10× improvement)

When async operations have no interdependencies, execute them concurrently using `Promise.all()`.

### Incorrect (sequential execution, 3 round trips):

```typescript
const user = await fetchUser()
const posts = await fetchPosts()
const comments = await fetchComments()
```

**Problem:** Each fetch waits for the previous one. If each takes 100ms, total time is 300ms.

### Correct (parallel execution, 1 round trip):

```typescript
const [user, posts, comments] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments()
])
```

**Benefit:** All fetches start simultaneously. Total time is ~100ms (the slowest operation).

---

## 2. Defer Await Until Needed

**Impact:** HIGH (avoids blocking unused code paths)

Move `await` operations into the branches where they're actually used to avoid blocking code paths that don't need them.

### Incorrect (blocks both branches):

```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  const userData = await fetchUserData(userId)
  
  if (skipProcessing) {
    // Returns immediately but still waited for userData
    return { skipped: true }
  }
  
  // Only this branch uses userData
  return processUserData(userData)
}
```

**Problem:** The skip path still pays the cost of fetching userData.

### Correct (only blocks when needed):

```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  if (skipProcessing) {
    // Returns immediately without waiting
    return { skipped: true }
  }
  
  // Fetch only when needed
  const userData = await fetchUserData(userId)
  return processUserData(userData)
}
```

**Benefit:** Skip path returns instantly without any async work.

### Another Example (early return optimization):

```typescript
// Incorrect: always fetches permissions
async function updateResource(resourceId: string, userId: string) {
  const permissions = await fetchPermissions(userId)
  const resource = await getResource(resourceId)
  
  if (!resource) {
    return { error: 'Not found' }
  }
  
  if (!permissions.canEdit) {
    return { error: 'Forbidden' }
  }
  
  return await updateResourceData(resource, permissions)
}

// Correct: fetches only when needed
async function updateResource(resourceId: string, userId: string) {
  const resource = await getResource(resourceId)
  
  if (!resource) {
    return { error: 'Not found' }
  }
  
  const permissions = await fetchPermissions(userId)
  
  if (!permissions.canEdit) {
    return { error: 'Forbidden' }
  }
  
  return await updateResourceData(resource, permissions)
}
```

**Benefit:** When resource doesn't exist (common case), we avoid the permissions fetch entirely.

This optimization is especially valuable when:
- The skipped branch is frequently taken
- The deferred operation is expensive
- You have multiple early-return conditions

---

## 3. Check Cheap Conditions Before Async Flags

**Impact:** HIGH (avoids unnecessary async work)

When a branch uses `await` for a flag or remote value and also requires a **cheap synchronous** condition (local props, request metadata, already-loaded state), evaluate the cheap condition **first**.

This is a specialization of "Defer Await Until Needed" for `flag && cheapCondition` style checks.

### Incorrect:

```typescript
const someFlag = await getFlag()

if (someFlag && someCondition) {
  // ...
}
```

**Problem:** You pay for the async flag fetch even when `someCondition` is false.

### Correct:

```typescript
if (someCondition) {
  const someFlag = await getFlag()
  if (someFlag) {
    // ...
  }
}
```

**Benefit:** When `someCondition` is false, we skip the network call entirely.

This matters when `getFlag` hits:
- The network
- A feature-flag service
- `React.cache` / database work

**When NOT to apply:**
- Keep the original order if `someCondition` is expensive
- Keep the original order if `someCondition` depends on the flag
- Keep the original order if you must run side effects in a fixed order

---

## 4. Dependency-Based Parallelization

**Impact:** CRITICAL (2-10× improvement)

For operations with partial dependencies, use `better-all` to maximize parallelism. It automatically starts each task at the earliest possible moment.

### Incorrect (profile waits for config unnecessarily):

```typescript
const [user, config] = await Promise.all([
  fetchUser(),
  fetchConfig()
])
const profile = await fetchProfile(user.id)
```

**Problem:** `profile` depends on `user`, but `config` is independent. Yet `profile` waits for both to complete.

### Correct (config and profile run in parallel):

```typescript
import { all } from 'better-all'

const { user, config, profile } = await all({
  async user() { return fetchUser() },
  async config() { return fetchConfig() },
  async profile() {
    return fetchProfile((await this.$.user).id)
  }
})
```

**Benefit:** `config` and `profile` start as soon as possible. `profile` waits only for `user`, not `config`.

### Alternative without extra dependencies:

```typescript
const userPromise = fetchUser()
const profilePromise = userPromise.then(user => fetchProfile(user.id))

const [user, config, profile] = await Promise.all([
  userPromise,
  fetchConfig(),
  profilePromise
])
```

**When to use:**
- Complex dependency chains (A → B → C, A → D)
- Multiple operations where some depend on others
- When you want to maximize parallelism automatically

**Reference:** [https://github.com/shuding/better-all](https://github.com/shuding/better-all)

---

## 5. Prevent Waterfall Chains in API Routes

**Impact:** CRITICAL (2-10× improvement)

In API routes and Server Actions, start independent operations immediately, even if you don't await them yet.

### Incorrect (config waits for auth, data waits for both):

```typescript
export async function GET(request: Request) {
  const session = await auth()
  const config = await fetchConfig()
  const data = await fetchData(session.user.id)
  return Response.json({ data, config })
}
```

**Problem:** Three sequential round trips. If each takes 50ms, total is 150ms.

### Correct (auth and config start immediately):

```typescript
export async function GET(request: Request) {
  const sessionPromise = auth()
  const configPromise = fetchConfig()
  const session = await sessionPromise
  const [config, data] = await Promise.all([
    configPromise,
    fetchData(session.user.id)
  ])
  return Response.json({ data, config })
}
```

**Benefit:** `auth` and `config` run in parallel. `data` starts as soon as `session` resolves. Total time is ~100ms (two parallel rounds).

**Pattern:**
1. Start all independent operations immediately (don't await)
2. Await only when you need the result
3. Use `Promise.all()` for the final wait

For operations with more complex dependency chains, use `better-all` to automatically maximize parallelism.

---

## 6. Strategic Suspense Boundaries

**Impact:** HIGH (faster initial paint)

Instead of awaiting data in async components before returning JSX, use Suspense boundaries to show the wrapper UI faster while data loads.

### Incorrect (wrapper blocked by data fetching):

```tsx
async function Page() {
  const data = await fetchData() // Blocks entire page
  
  return (
    <div>
      <div>Sidebar</div>
      <div>Header</div>
      <div>
        <DataDisplay data={data} />
      </div>
      <div>Footer</div>
    </div>
  )
}
```

**Problem:** The entire layout waits for data even though only the middle section needs it.

### Correct (wrapper shows immediately, data streams in):

```tsx
function Page() {
  return (
    <div>
      <div>Sidebar</div>
      <div>Header</div>
      <div>
        <Suspense fallback={<Skeleton />}>
          <DataDisplay />
        </Suspense>
      </div>
      <div>Footer</div>
    </div>
  )
}

async function DataDisplay() {
  const data = await fetchData() // Only blocks this component
  return <div>{data.content}</div>
}
```

**Benefit:** Sidebar, Header, and Footer render immediately. Only DataDisplay waits for data.

### Alternative (share promise across components):

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

function DataSummary({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise) // Reuses the same promise
  return <div>{data.summary}</div>
}
```

**Benefit:** Both components share the same promise, so only one fetch occurs. Layout renders immediately while both components wait together.

### When NOT to use this pattern:

- **Critical data needed for layout decisions** (affects positioning)
- **SEO-critical content above the fold**
- **Small, fast queries** where suspense overhead isn't worth it
- **When you want to avoid layout shift** (loading → content jump)

**Trade-off:** Faster initial paint vs potential layout shift. Choose based on your UX priorities.

---

## Summary

Async optimization is about **starting work as early as possible** and **waiting as late as possible**:

1. **Start independent operations in parallel** with `Promise.all()`
2. **Defer awaits** to the branches that need them
3. **Check cheap conditions first** before expensive async calls
4. **Use better-all** for complex dependency chains
5. **Start promises early** in API routes, await late
6. **Use Suspense** to stream UI while data loads

These patterns compound: applying multiple rules to the same code path can yield 10× or greater improvements.
