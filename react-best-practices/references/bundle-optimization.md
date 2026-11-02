# Bundle Optimization (CRITICAL Priority)

Bundle size directly impacts Time to Interactive (TTI) and Largest Contentful Paint (LCP). Every kilobyte of JavaScript must be downloaded, parsed, and executed before your app becomes interactive. This section covers strategies to minimize bundle size and defer non-critical code.

**Impact:** 15-70% faster load times, 200-800ms savings on cold starts.

---

## Quick Reference

| Rule | Impact | When to Apply |
|------|--------|---------------|
| Avoid Barrel File Imports | CRITICAL | Icon libraries, component libraries, utility libraries |
| Dynamic Imports for Heavy Components | CRITICAL | Large components not needed on initial render |
| Defer Non-Critical Third-Party Libraries | MEDIUM | Analytics, logging, error tracking |
| Conditional Module Loading | HIGH | Features activated by user action or flags |
| Preload Based on User Intent | MEDIUM | Heavy bundles that will likely be needed soon |

---

## 1. Avoid Barrel File Imports

**Impact:** CRITICAL (200-800ms import cost, slow builds)

Import directly from source files instead of barrel files to avoid loading thousands of unused modules. **Barrel files** are entry points that re-export multiple modules (e.g., `index.js` that does `export * from './module'`).

Popular icon and component libraries can have **up to 10,000 re-exports** in their entry file. For many React packages, **it takes 200-800ms just to import them**, affecting both development speed and production cold starts.

### Why tree-shaking doesn't help

When a library is marked as external (not bundled), the bundler can't optimize it. If you bundle it to enable tree-shaking, builds become substantially slower analyzing the entire module graph.

### Incorrect (imports entire library):

```tsx
import { Check, X, Menu } from 'lucide-react'
// Loads 1,583 modules, takes ~2.8s extra in dev
// Runtime cost: 200-800ms on every cold start

import { Button, TextField } from '@mui/material'
// Loads 2,225 modules, takes ~4.2s extra in dev
```

**Problem:** The entire library is loaded into memory even though you only use 3 icons or 2 components.

### Correct - Next.js 13.5+ (recommended):

```js
// next.config.js - automatically optimizes barrel imports at build time
module.exports = {
  experimental: {
    optimizePackageImports: ['lucide-react', '@mui/material']
  }
}
```

```tsx
// Keep the standard imports - Next.js transforms them to direct imports
import { Check, X, Menu } from 'lucide-react'
// Full TypeScript support, no manual path wrangling
```

**Benefit:** This is the recommended approach because it preserves TypeScript type safety and editor autocompletion while still eliminating the barrel import cost.

### Correct - Direct imports (non-Next.js projects):

```tsx
import Button from '@mui/material/Button'
import TextField from '@mui/material/TextField'
// Loads only what you use
```

**Benefit:** Only the specific modules you need are loaded.

### TypeScript Warning

Some libraries (notably `lucide-react`) don't ship `.d.ts` files for their deep import paths. Importing from `lucide-react/dist/esm/icons/check` resolves to an implicit `any` type, causing errors under `strict` or `noImplicitAny`. Prefer `optimizePackageImports` when available, or verify the library exports types for its subpaths before using direct imports.

### Performance Gains

These optimizations provide:
- **15-70% faster dev boot**
- **28% faster builds**
- **40% faster cold starts**
- **Significantly faster HMR**

### Libraries Commonly Affected

- `lucide-react`
- `@mui/material`
- `@mui/icons-material`
- `@tabler/icons-react`
- `react-icons`
- `@headlessui/react`
- `@radix-ui/react-*`
- `lodash`
- `ramda`
- `date-fns`
- `rxjs`
- `react-use`

**Reference:** [How we optimized package imports in Next.js](https://vercel.com/blog/how-we-optimized-package-imports-in-next-js)

---

## 2. Dynamic Imports for Heavy Components

**Impact:** CRITICAL (directly affects TTI and LCP)

Use `next/dynamic` (or `React.lazy`) to lazy-load large components not needed on initial render.

### Incorrect (Monaco bundles with main chunk ~300KB):

```tsx
import { MonacoEditor } from './monaco-editor'

function CodePanel({ code }: { code: string }) {
  return <MonacoEditor value={code} />
}
```

**Problem:** Monaco Editor (~300KB) is included in the initial bundle even if the user never opens the code panel.

### Correct (Monaco loads on demand):

```tsx
import dynamic from 'next/dynamic'

const MonacoEditor = dynamic(
  () => import('./monaco-editor').then(m => m.MonacoEditor),
  { ssr: false }
)

function CodePanel({ code }: { code: string }) {
  return <MonacoEditor value={code} />
}
```

**Benefit:** Monaco Editor is split into a separate chunk and only loaded when the component renders.

### When to Use Dynamic Imports

- **Heavy third-party libraries** (charts, editors, maps)
- **Components behind tabs or modals**
- **Admin-only features**
- **Components that depend on browser APIs** (use `ssr: false`)

### React.lazy Alternative (non-Next.js):

```tsx
import { lazy, Suspense } from 'react'

const MonacoEditor = lazy(() => 
  import('./monaco-editor').then(m => ({ default: m.MonacoEditor }))
)

function CodePanel({ code }: { code: string }) {
  return (
    <Suspense fallback={<Skeleton />}>
      <MonacoEditor value={code} />
    </Suspense>
  )
}
```

---

## 3. Defer Non-Critical Third-Party Libraries

**Impact:** MEDIUM (loads after hydration)

Analytics, logging, and error tracking don't block user interaction. Load them after hydration.

### Incorrect (blocks initial bundle):

```tsx
import { Analytics } from '@vercel/analytics/react'

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  )
}
```

**Problem:** Analytics code is included in the initial bundle, increasing TTI.

### Correct (loads after hydration):

```tsx
import dynamic from 'next/dynamic'

const Analytics = dynamic(
  () => import('@vercel/analytics/react').then(m => m.Analytics),
  { ssr: false }
)

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  )
}
```

**Benefit:** Analytics loads after the page is interactive, prioritizing user experience.

### What to Defer

- Analytics (Google Analytics, Vercel Analytics, Plausible)
- Error tracking (Sentry, Bugsnag)
- Chat widgets (Intercom, Zendesk)
- A/B testing libraries
- Marketing pixels
- Social media widgets

### What NOT to Defer

- Authentication libraries
- Critical UI components
- Data fetching utilities
- State management

---

## 4. Conditional Module Loading

**Impact:** HIGH (loads large data only when needed)

Load large data or modules only when a feature is activated.

### Example (lazy-load animation frames):

```tsx
function AnimationPlayer({ 
  enabled, 
  setEnabled 
}: { 
  enabled: boolean
  setEnabled: React.Dispatch<React.SetStateAction<boolean>> 
}) {
  const [frames, setFrames] = useState<Frame[] | null>(null)

  useEffect(() => {
    if (enabled && !frames && typeof window !== 'undefined') {
      import('./animation-frames.js')
        .then(mod => setFrames(mod.frames))
        .catch(() => setEnabled(false))
    }
  }, [enabled, frames, setEnabled])

  if (!frames) return <Skeleton />
  return <Canvas frames={frames} />
}
```

**Benefit:** The `animation-frames.js` module (potentially large) only loads when the user enables the animation.

### The `typeof window !== 'undefined'` Check

This check prevents bundling this module for SSR, optimizing:
- Server bundle size
- Build speed
- Server memory usage

### Use Cases

- **Large JSON data** (maps, translations, configuration)
- **Feature-flagged code**
- **Admin-only modules**
- **Experimental features**
- **Heavy computations** (WebAssembly modules)

---

## 5. Preload Based on User Intent

**Impact:** MEDIUM (reduces perceived latency)

Preload heavy bundles before they're needed to reduce perceived latency.

### Example (preload on hover/focus):

```tsx
function EditorButton({ onClick }: { onClick: () => void }) {
  const preload = () => {
    if (typeof window !== 'undefined') {
      void import('./monaco-editor')
    }
  }

  return (
    <button
      onMouseEnter={preload}
      onFocus={preload}
      onClick={onClick}
    >
      Open Editor
    </button>
  )
}
```

**Benefit:** By the time the user clicks, the editor is likely already loaded.

### Example (preload when feature flag is enabled):

```tsx
function FlagsProvider({ children, flags }: Props) {
  useEffect(() => {
    if (flags.editorEnabled && typeof window !== 'undefined') {
      void import('./monaco-editor').then(mod => mod.init())
    }
  }, [flags.editorEnabled])

  return (
    <FlagsContext.Provider value={flags}>
      {children}
    </FlagsContext.Provider>
  )
}
```

**Benefit:** The editor is preloaded as soon as the feature is enabled, before the user navigates to it.

### Preload Triggers

- **Hover/Focus** - User is likely to click
- **Viewport intersection** - Component is about to scroll into view
- **Route prefetch** - User is likely to navigate
- **Feature flags** - Feature is enabled for this user
- **Idle time** - Browser has spare cycles

### The `typeof window !== 'undefined'` Check

This check prevents bundling preloaded modules for SSR, optimizing server bundle size and build speed.

---

## Summary

Bundle optimization is about **loading only what you need, when you need it**:

1. **Avoid barrel imports** - Use `optimizePackageImports` or direct imports
2. **Dynamic imports** - Split heavy components into separate chunks
3. **Defer third-party** - Load analytics and tracking after hydration
4. **Conditional loading** - Load modules only when features are activated
5. **Preload on intent** - Start loading before the user needs it

These patterns compound: a well-optimized app uses all five strategies together.

### Measuring Impact

Use these tools to measure bundle size improvements:

- **Next.js Bundle Analyzer** - `@next/bundle-analyzer`
- **Webpack Bundle Analyzer** - `webpack-bundle-analyzer`
- **Chrome DevTools** - Coverage tab shows unused code
- **Lighthouse** - Measures TTI and LCP
- **WebPageTest** - Real-world performance testing

### Target Metrics

- **Initial bundle** - < 200KB (gzipped)
- **TTI** - < 3.8s on 3G
- **LCP** - < 2.5s
- **Unused code** - < 20% of bundle
