# Rendering Performance (MEDIUM Priority)

Rendering performance focuses on optimizing how React and the browser paint content to the screen. These patterns cover DOM manipulation, hydration, resource loading, and CSS optimizations.

**Impact:** Faster initial paint, smoother animations, better perceived performance.

---

## Quick Reference

| Rule | Impact | When to Apply |
|------|--------|---------------|
| Animate SVG Wrapper Instead of SVG Element | LOW | CSS animations on SVG |
| CSS content-visibility for Long Lists | HIGH | Lists with 100+ items |
| Hoist Static JSX Elements | LOW | Static JSX recreated on every render |
| Optimize SVG Precision | LOW | Large or numerous SVG files |
| Prevent Hydration Mismatch Without Flickering | MEDIUM | Client-only data (localStorage, theme) |
| Suppress Expected Hydration Mismatches | LOW-MEDIUM | Known server/client differences |
| Use Activity Component for Show/Hide | MEDIUM | Expensive components that toggle visibility |
| Use Explicit Conditional Rendering | LOW | Conditions that can be 0 or NaN |
| Use useTransition Over Manual Loading States | LOW | Loading states for async operations |
| Use React DOM Resource Hints | HIGH | Critical resources (fonts, styles, APIs) |
| Use defer or async on Script Tags | HIGH | Third-party scripts |

---

## 1. Animate SVG Wrapper Instead of SVG Element

**Impact:** LOW (enables hardware acceleration)

Many browsers don't have hardware acceleration for CSS3 animations on SVG elements. Wrap SVG in a `<div>` and animate the wrapper instead.

### Incorrect (animating SVG directly - no hardware acceleration):

```tsx
function LoadingSpinner() {
  return (
    <svg 
      className="animate-spin"
      width="24" 
      height="24" 
      viewBox="0 0 24 24"
    >
      <circle cx="12" cy="12" r="10" stroke="currentColor" />
    </svg>
  )
}
```

### Correct (animating wrapper div - hardware accelerated):

```tsx
function LoadingSpinner() {
  return (
    <div className="animate-spin">
      <svg 
        width="24" 
        height="24" 
        viewBox="0 0 24 24"
      >
        <circle cx="12" cy="12" r="10" stroke="currentColor" />
      </svg>
    </div>
  )
}
```

This applies to all CSS transforms and transitions (`transform`, `opacity`, `translate`, `scale`, `rotate`).

---

## 2. CSS content-visibility for Long Lists

**Impact:** HIGH (faster initial render)

Apply `content-visibility: auto` to defer off-screen rendering.

### CSS:

```css
.message-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 80px;
}
```

### Example:

```tsx
function MessageList({ messages }: { messages: Message[] }) {
  return (
    <div className="overflow-y-auto h-screen">
      {messages.map(msg => (
        <div key={msg.id} className="message-item">
          <Avatar user={msg.author} />
          <div>{msg.content}</div>
        </div>
      ))}
    </div>
  )
}
```

For 1000 messages, browser skips layout/paint for ~990 off-screen items (10× faster initial render).

---

## 3. Hoist Static JSX Elements

**Impact:** LOW (avoids re-creation)

Extract static JSX outside components to avoid re-creation.

### Incorrect (recreates element every render):

```tsx
function Container() {
  return (
    <div>
      {loading && <div className="animate-pulse h-20 bg-gray-200" />}
    </div>
  )
}
```

### Correct (reuses same element):

```tsx
const loadingSkeleton = (
  <div className="animate-pulse h-20 bg-gray-200" />
)

function Container() {
  return (
    <div>
      {loading && loadingSkeleton}
    </div>
  )
}
```

**Note:** React Compiler automatically hoists static JSX.

---

## 4. Optimize SVG Precision

**Impact:** LOW (reduces file size)

Reduce SVG coordinate precision to decrease file size.

### Incorrect (excessive precision):

```svg
<path d="M 10.293847 20.847362 L 30.938472 40.192837" />
```

### Correct (1 decimal place):

```svg
<path d="M 10.3 20.8 L 30.9 40.2" />
```

### Automate with SVGO:

```bash
npx svgo --precision=1 --multipass icon.svg
```

---

## 5. Prevent Hydration Mismatch Without Flickering

**Impact:** MEDIUM (avoids visual flicker and hydration errors)

When rendering content that depends on client-side storage, inject a synchronous script that updates the DOM before React hydrates.

### Incorrect (breaks SSR):

```tsx
function ThemeWrapper({ children }: { children: ReactNode }) {
  // localStorage is not available on server - throws error
  const theme = localStorage.getItem('theme') || 'light'
  
  return (
    <div className={theme}>
      {children}
    </div>
  )
}
```

### Incorrect (visual flickering):

```tsx
function ThemeWrapper({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState('light')
  
  useEffect(() => {
    // Runs after hydration - causes visible flash
    const stored = localStorage.getItem('theme')
    if (stored) {
      setTheme(stored)
    }
  }, [])
  
  return (
    <div className={theme}>
      {children}
    </div>
  )
}
```

### Correct (no flicker, no hydration mismatch):

```tsx
function ThemeWrapper({ children }: { children: ReactNode }) {
  return (
    <>
      <div id="theme-wrapper">
        {children}
      </div>
      <script
        dangerouslySetInnerHTML={{
          __html: `
            (function() {
              try {
                var theme = localStorage.getItem('theme') || 'light';
                var el = document.getElementById('theme-wrapper');
                if (el) el.className = theme;
              } catch (e) {}
            })();
          `,
        }}
      />
    </>
  )
}
```

The inline script executes synchronously before showing the element.

---

## 6. Suppress Expected Hydration Mismatches

**Impact:** LOW-MEDIUM (avoids noisy warnings)

For *expected* mismatches (random IDs, dates, locale formatting), use `suppressHydrationWarning`.

### Incorrect (known mismatch warnings):

```tsx
function Timestamp() {
  return <span>{new Date().toLocaleString()}</span>
}
```

### Correct (suppress expected mismatch only):

```tsx
function Timestamp() {
  return (
    <span suppressHydrationWarning>
      {new Date().toLocaleString()}
    </span>
  )
}
```

---

## 7. Use Activity Component for Show/Hide

**Impact:** MEDIUM (preserves state/DOM)

Use React's `<Activity>` to preserve state/DOM for expensive components that frequently toggle visibility.

### Usage:

```tsx
import { Activity } from 'react'

function Dropdown({ isOpen }: Props) {
  return (
    <Activity mode={isOpen ? 'visible' : 'hidden'}>
      <ExpensiveMenu />
    </Activity>
  )
}
```

Avoids expensive re-renders and state loss.

---

## 8. Use Explicit Conditional Rendering

**Impact:** LOW (prevents rendering 0 or NaN)

Use explicit ternary operators instead of `&&` when the condition can be falsy values that render.

### Incorrect (renders "0" when count is 0):

```tsx
function Badge({ count }: { count: number }) {
  return (
    <div>
      {count && <span className="badge">{count}</span>}
    </div>
  )
}
```

### Correct (renders nothing when count is 0):

```tsx
function Badge({ count }: { count: number }) {
  return (
    <div>
      {count > 0 ? <span className="badge">{count}</span> : null}
    </div>
  )
}
```

---

## 9. Use useTransition Over Manual Loading States

**Impact:** LOW (reduces re-renders and improves code clarity)

Use `useTransition` instead of manual `useState` for loading states.

### Incorrect (manual loading state):

```tsx
function SearchResults() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])
  const [isLoading, setIsLoading] = useState(false)

  const handleSearch = async (value: string) => {
    setIsLoading(true)
    setQuery(value)
    const data = await fetchResults(value)
    setResults(data)
    setIsLoading(false)
  }

  return (
    <>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isLoading && <Spinner />}
      <ResultsList results={results} />
    </>
  )
}
```

### Correct (useTransition with built-in pending state):

```tsx
import { useTransition, useState } from 'react'

function SearchResults() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])
  const [isPending, startTransition] = useTransition()

  const handleSearch = (value: string) => {
    setQuery(value)
    
    startTransition(async () => {
      const data = await fetchResults(value)
      setResults(data)
    })
  }

  return (
    <>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  )
}
```

---

## 10. Use React DOM Resource Hints

**Impact:** HIGH (reduces load time for critical resources)

React DOM provides APIs to hint the browser about resources it will need.

### Example (preconnect to third-party APIs):

```tsx
import { preconnect, prefetchDNS } from 'react-dom'

export default function App() {
  prefetchDNS('https://analytics.example.com')
  preconnect('https://api.example.com')

  return <main>{/* content */}</main>
}
```

### Example (preload critical fonts and styles):

```tsx
import { preload, preinit } from 'react-dom'

export default function RootLayout({ children }) {
  preload('/fonts/inter.woff2', { 
    as: 'font', 
    type: 'font/woff2', 
    crossOrigin: 'anonymous' 
  })

  preinit('/styles/critical.css', { as: 'style' })

  return (
    <html>
      <body>{children}</body>
    </html>
  )
}
```

### When to use each:

| API | Use case |
|-----|----------|
| `prefetchDNS` | Third-party domains you'll connect to later |
| `preconnect` | APIs or CDNs you'll fetch from immediately |
| `preload` | Critical resources needed for current page |
| `preloadModule` | JS modules for likely next navigation |
| `preinit` | Stylesheets/scripts that must execute early |
| `preinitModule` | ES modules that must execute early |

---

## 11. Use defer or async on Script Tags

**Impact:** HIGH (eliminates render-blocking)

Script tags without `defer` or `async` block HTML parsing.

### Incorrect (blocks rendering):

```tsx
export default function Document() {
  return (
    <html>
      <head>
        <script src="https://example.com/analytics.js" />
        <script src="/scripts/utils.js" />
      </head>
      <body>{/* content */}</body>
    </html>
  )
}
```

### Correct (non-blocking):

```tsx
export default function Document() {
  return (
    <html>
      <head>
        <script src="https://example.com/analytics.js" async />
        <script src="/scripts/utils.js" defer />
      </head>
      <body>{/* content */}</body>
    </html>
  )
}
```

### In Next.js:

```tsx
import Script from 'next/script'

export default function Page() {
  return (
    <>
      <Script src="https://example.com/analytics.js" strategy="afterInteractive" />
      <Script src="/scripts/utils.js" strategy="beforeInteractive" />
    </>
  )
}
```

---

## Summary

Rendering performance is about **optimizing the paint pipeline**:

1. **Animate wrappers** for SVG hardware acceleration
2. **Use content-visibility** for long lists
3. **Hoist static JSX** to avoid recreation
4. **Optimize SVG precision** for smaller files
5. **Prevent hydration flicker** with inline scripts
6. **Suppress expected mismatches** to reduce noise
7. **Use Activity** to preserve expensive component state
8. **Use explicit conditionals** to avoid rendering falsy values
9. **Use useTransition** for cleaner loading states
10. **Use resource hints** to preload critical assets
11. **Use defer/async** on scripts to avoid blocking

These patterns create faster initial paints and smoother user experiences.
