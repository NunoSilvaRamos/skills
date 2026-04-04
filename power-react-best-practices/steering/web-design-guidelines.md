# Web Design Guidelines (HIGH Priority)

Comprehensive UI/UX best practices for building accessible, performant, and user-friendly web interfaces. These rules cover accessibility, forms, animation, typography, performance, navigation, touch interaction, dark mode, and internationalization.

**Impact:** Ensures WCAG compliance, prevents common UX pitfalls, improves perceived performance, and creates production-ready interfaces.

---

## Quick Reference

| Category | Priority | Key Rules |
|----------|----------|-----------|
| Accessibility | CRITICAL | aria-label, semantic HTML, keyboard handlers, focus states |
| Forms | HIGH | autocomplete, labels, inline errors, no paste blocking |
| Animation | HIGH | prefers-reduced-motion, transform/opacity only, interruptible |
| Performance | HIGH | Virtualize lists, no layout reads in render, preconnect |
| Navigation | MEDIUM | URL reflects state, deep-link everything, confirm destructive actions |
| Touch | MEDIUM | touch-action, tap-highlight, overscroll-behavior |
| Typography | LOW | Ellipsis, curly quotes, tabular-nums, text-wrap |

---

## 1. Accessibility (CRITICAL)

### 1.1 Icon-Only Buttons Need aria-label

**Impact:** CRITICAL (screen readers can't announce button purpose)

#### Incorrect:

```tsx
<button onClick={handleDelete}>
  <TrashIcon />
</button>
```

#### Correct:

```tsx
<button onClick={handleDelete} aria-label="Delete item">
  <TrashIcon />
</button>
```

### 1.2 Form Controls Need Labels

**Impact:** CRITICAL (screen readers can't identify inputs)

#### Incorrect:

```tsx
<input type="email" placeholder="Email" />
```

#### Correct (visible label):

```tsx
<label htmlFor="email">Email</label>
<input id="email" type="email" />
```

#### Correct (aria-label when visual label not desired):

```tsx
<input type="email" aria-label="Email address" placeholder="you@example.com" />
```

### 1.3 Use Semantic HTML Before ARIA

**Impact:** HIGH (better browser support, clearer intent)

#### Incorrect:

```tsx
<div onClick={handleClick} role="button" tabIndex={0}>
  Submit
</div>
```

#### Correct:

```tsx
<button onClick={handleClick}>
  Submit
</button>
```

**Semantic elements to prefer:**
- `<button>` for actions
- `<a>` or `<Link>` for navigation
- `<label>` for form labels
- `<table>` for tabular data
- `<h1>`–`<h6>` for headings (hierarchical)

### 1.4 Interactive Elements Need Keyboard Handlers

**Impact:** HIGH (keyboard users can't interact)

#### Incorrect:

```tsx
<div onClick={handleClick}>
  Click me
</div>
```

#### Correct:

```tsx
<button onClick={handleClick}>
  Click me
</button>

// Or if you must use div:
<div
  onClick={handleClick}
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault()
      handleClick()
    }
  }}
  role="button"
  tabIndex={0}
>
  Click me
</div>
```

### 1.5 Images Need alt Text

**Impact:** HIGH (screen readers can't describe images)

#### Incorrect:

```tsx
<img src="/logo.png" />
```

#### Correct (meaningful image):

```tsx
<img src="/logo.png" alt="Company logo" />
```

#### Correct (decorative image):

```tsx
<img src="/decoration.png" alt="" />
```

### 1.6 Decorative Icons Need aria-hidden

**Impact:** MEDIUM (reduces screen reader noise)

#### Incorrect:

```tsx
<button>
  <CheckIcon />
  Save
</button>
```

#### Correct:

```tsx
<button>
  <CheckIcon aria-hidden="true" />
  Save
</button>
```

### 1.7 Async Updates Need aria-live

**Impact:** HIGH (screen readers miss dynamic content)

#### Incorrect:

```tsx
{showToast && <div>Saved successfully</div>}
```

#### Correct:

```tsx
{showToast && <div aria-live="polite">Saved successfully</div>}
```

---

## 2. Focus States (CRITICAL)

### 2.1 Interactive Elements Need Visible Focus

**Impact:** CRITICAL (keyboard users can't see focus)

#### Incorrect:

```css
button {
  outline: none;
}
```

#### Correct (Tailwind):

```tsx
<button className="focus-visible:ring-2 focus-visible:ring-blue-500">
  Submit
</button>
```

#### Correct (CSS):

```css
button:focus-visible {
  outline: 2px solid blue;
  outline-offset: 2px;
}
```

### 2.2 Use :focus-visible Over :focus

**Impact:** MEDIUM (avoids focus ring on mouse click)

#### Incorrect:

```css
button:focus {
  outline: 2px solid blue;
}
```

#### Correct:

```css
button:focus-visible {
  outline: 2px solid blue;
}
```

**Benefit:** Focus ring only appears for keyboard navigation, not mouse clicks.

---

## 3. Forms (HIGH)

### 3.1 Inputs Need autocomplete and name

**Impact:** HIGH (password managers and autofill won't work)

#### Incorrect:

```tsx
<input type="email" />
```

#### Correct:

```tsx
<input
  type="email"
  name="email"
  autocomplete="email"
/>
```

**Common autocomplete values:**
- `email`, `username`, `current-password`, `new-password`
- `given-name`, `family-name`, `name`
- `tel`, `street-address`, `postal-code`, `country`
- `cc-number`, `cc-exp`, `cc-csc`

### 3.2 Use Correct Input Types and inputmode

**Impact:** MEDIUM (mobile keyboards show wrong layout)

#### Incorrect:

```tsx
<input type="text" placeholder="Phone" />
```

#### Correct:

```tsx
<input
  type="tel"
  inputmode="tel"
  autocomplete="tel"
  placeholder="(555) 123-4567"
/>
```

**Input types:** `email`, `tel`, `url`, `number`, `date`, `time`, `search`

**inputmode values:** `numeric`, `tel`, `email`, `url`, `decimal`

### 3.3 Never Block Paste

**Impact:** HIGH (breaks password managers, frustrates users)

#### Incorrect:

```tsx
<input
  type="password"
  onPaste={(e) => e.preventDefault()}
/>
```

#### Correct:

```tsx
<input type="password" />
```

### 3.4 Submit Button Stays Enabled Until Request Starts

**Impact:** MEDIUM (prevents accidental double-submit)

#### Incorrect:

```tsx
<button disabled={!isValid}>
  Submit
</button>
```

#### Correct:

```tsx
<button disabled={isSubmitting}>
  {isSubmitting ? 'Submitting…' : 'Submit'}
</button>
```

### 3.5 Errors Inline Next to Fields

**Impact:** HIGH (users can't see what's wrong)

#### Incorrect:

```tsx
{error && <div>Form has errors</div>}
<input type="email" />
```

#### Correct:

```tsx
<input
  type="email"
  aria-invalid={!!emailError}
  aria-describedby={emailError ? 'email-error' : undefined}
/>
{emailError && (
  <div id="email-error" role="alert">
    {emailError}
  </div>
)}
```

### 3.6 Placeholders End with … and Show Example Pattern

**Impact:** LOW (clearer expectations)

#### Incorrect:

```tsx
<input placeholder="Email" />
```

#### Correct:

```tsx
<input placeholder="you@example.com…" />
```

### 3.7 Warn Before Navigation with Unsaved Changes

**Impact:** HIGH (prevents data loss)

#### Correct:

```tsx
useEffect(() => {
  if (hasUnsavedChanges) {
    const handler = (e: BeforeUnloadEvent) => {
      e.preventDefault()
      e.returnValue = ''
    }
    window.addEventListener('beforeunload', handler)
    return () => window.removeEventListener('beforeunload', handler)
  }
}, [hasUnsavedChanges])
```

---

## 4. Animation (HIGH)

### 4.1 Honor prefers-reduced-motion

**Impact:** CRITICAL (accessibility requirement)

#### Incorrect:

```css
.fade-in {
  animation: fadeIn 0.3s ease-out;
}
```

#### Correct:

```css
.fade-in {
  animation: fadeIn 0.3s ease-out;
}

@media (prefers-reduced-motion: reduce) {
  .fade-in {
    animation: none;
  }
}
```

### 4.2 Animate transform/opacity Only

**Impact:** HIGH (GPU-accelerated, no layout thrashing)

#### Incorrect:

```css
.slide {
  transition: left 0.3s;
}
```

#### Correct:

```css
.slide {
  transition: transform 0.3s;
}
```

**GPU-accelerated properties:** `transform`, `opacity`

**Avoid animating:** `width`, `height`, `top`, `left`, `margin`, `padding`

### 4.3 Never transition: all

**Impact:** MEDIUM (unpredictable, can animate unintended properties)

#### Incorrect:

```css
button {
  transition: all 0.2s;
}
```

#### Correct:

```css
button {
  transition: background-color 0.2s, transform 0.2s;
}
```

### 4.4 Animations Must Be Interruptible

**Impact:** MEDIUM (responsive to user input)

Animations should respond to user input mid-animation. Use CSS transitions or animation libraries that support interruption.

---

## 5. Typography (LOW)

### 5.1 Use … Not ...

**Impact:** LOW (typographically correct)

#### Incorrect:

```tsx
<button disabled={isLoading}>
  {isLoading ? 'Loading...' : 'Submit'}
</button>
```

#### Correct:

```tsx
<button disabled={isLoading}>
  {isLoading ? 'Loading…' : 'Submit'}
</button>
```

### 5.2 Use Curly Quotes

**Impact:** LOW (typographically correct)

#### Incorrect:

```tsx
<p>He said "hello"</p>
```

#### Correct:

```tsx
<p>He said "hello"</p>
```

### 5.3 Non-Breaking Spaces

**Impact:** LOW (prevents awkward line breaks)

#### Incorrect:

```tsx
<p>10 MB</p>
<p>⌘ K</p>
```

#### Correct:

```tsx
<p>10&nbsp;MB</p>
<p>⌘&nbsp;K</p>
```

### 5.4 Use tabular-nums for Number Columns

**Impact:** MEDIUM (numbers align properly)

#### Incorrect:

```tsx
<td>1,234</td>
<td>56</td>
<td>7,890</td>
```

#### Correct:

```css
td {
  font-variant-numeric: tabular-nums;
}
```

### 5.5 Use text-wrap: balance on Headings

**Impact:** LOW (prevents widows)

#### Correct:

```css
h1, h2, h3 {
  text-wrap: balance;
}
```

---

## 6. Content Handling (MEDIUM)

### 6.1 Text Containers Handle Long Content

**Impact:** HIGH (prevents layout breaks)

#### Incorrect:

```tsx
<div className="w-64">
  <p>{veryLongTextWithNoSpaces}</p>
</div>
```

#### Correct (truncate):

```tsx
<div className="w-64">
  <p className="truncate">{veryLongTextWithNoSpaces}</p>
</div>
```

#### Correct (line clamp):

```tsx
<div className="w-64">
  <p className="line-clamp-3">{longText}</p>
</div>
```

#### Correct (break words):

```tsx
<div className="w-64">
  <p className="break-words">{veryLongTextWithNoSpaces}</p>
</div>
```

### 6.2 Flex Children Need min-w-0 for Truncation

**Impact:** MEDIUM (text won't truncate in flex containers)

#### Incorrect:

```tsx
<div className="flex">
  <p className="truncate">{longText}</p>
</div>
```

#### Correct:

```tsx
<div className="flex">
  <p className="min-w-0 truncate">{longText}</p>
</div>
```

### 6.3 Handle Empty States

**Impact:** HIGH (prevents broken UI)

#### Incorrect:

```tsx
<ul>
  {items.map(item => <li key={item.id}>{item.name}</li>)}
</ul>
```

#### Correct:

```tsx
{items.length > 0 ? (
  <ul>
    {items.map(item => <li key={item.id}>{item.name}</li>)}
  </ul>
) : (
  <p>No items found</p>
)}
```

---

## 7. Images (MEDIUM)

### 7.1 Images Need Explicit width and height

**Impact:** HIGH (prevents Cumulative Layout Shift)

#### Incorrect:

```tsx
<img src="/hero.jpg" alt="Hero" />
```

#### Correct:

```tsx
<img src="/hero.jpg" alt="Hero" width={1200} height={600} />
```

### 7.2 Below-Fold Images: loading="lazy"

**Impact:** MEDIUM (faster initial page load)

#### Correct:

```tsx
<img src="/below-fold.jpg" alt="Content" loading="lazy" width={800} height={400} />
```

### 7.3 Above-Fold Critical Images: priority

**Impact:** MEDIUM (faster LCP)

#### Correct (Next.js):

```tsx
<Image src="/hero.jpg" alt="Hero" priority width={1200} height={600} />
```

#### Correct (HTML):

```tsx
<img src="/hero.jpg" alt="Hero" fetchpriority="high" width={1200} height={600} />
```

---

## 8. Performance (HIGH)

### 8.1 Virtualize Large Lists

**Impact:** CRITICAL (prevents DOM bloat)

#### Incorrect:

```tsx
<ul>
  {items.map(item => <li key={item.id}>{item.name}</li>)}
</ul>
// 1000+ items = 1000+ DOM nodes
```

#### Correct:

```tsx
import { VirtualList } from 'virtua'

<VirtualList data={items}>
  {(item) => <li key={item.id}>{item.name}</li>}
</VirtualList>
```

### 8.2 No Layout Reads in Render

**Impact:** HIGH (causes forced reflow)

#### Incorrect:

```tsx
function MyComponent() {
  const ref = useRef<HTMLDivElement>(null)
  const width = ref.current?.offsetWidth // Layout read in render
  
  return <div ref={ref}>Width: {width}</div>
}
```

#### Correct:

```tsx
function MyComponent() {
  const ref = useRef<HTMLDivElement>(null)
  const [width, setWidth] = useState(0)
  
  useEffect(() => {
    if (ref.current) {
      setWidth(ref.current.offsetWidth)
    }
  }, [])
  
  return <div ref={ref}>Width: {width}</div>
}
```

### 8.3 Preconnect to CDN/Asset Domains

**Impact:** MEDIUM (faster resource loading)

#### Correct:

```html
<link rel="preconnect" href="https://cdn.example.com" />
<link rel="dns-prefetch" href="https://cdn.example.com" />
```

### 8.4 Preload Critical Fonts

**Impact:** MEDIUM (prevents FOIT/FOUT)

#### Correct:

```html
<link
  rel="preload"
  href="/fonts/inter.woff2"
  as="font"
  type="font/woff2"
  crossorigin="anonymous"
/>
```

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter.woff2') format('woff2');
  font-display: swap;
}
```

---

## 9. Navigation & State (MEDIUM)

### 9.1 URL Reflects State

**Impact:** HIGH (enables deep-linking, back button works)

#### Incorrect:

```tsx
const [tab, setTab] = useState('overview')
```

#### Correct:

```tsx
const searchParams = useSearchParams()
const tab = searchParams.get('tab') ?? 'overview'

const setTab = (newTab: string) => {
  const params = new URLSearchParams(searchParams)
  params.set('tab', newTab)
  router.push(`?${params.toString()}`)
}
```

### 9.2 Links Use <a> or <Link>

**Impact:** HIGH (Cmd/Ctrl+click, middle-click support)

#### Incorrect:

```tsx
<div onClick={() => router.push('/about')}>
  About
</div>
```

#### Correct:

```tsx
<Link href="/about">
  About
</Link>
```

### 9.3 Destructive Actions Need Confirmation

**Impact:** HIGH (prevents accidental data loss)

#### Incorrect:

```tsx
<button onClick={handleDelete}>
  Delete
</button>
```

#### Correct:

```tsx
<button onClick={() => setShowConfirm(true)}>
  Delete
</button>

{showConfirm && (
  <Dialog>
    <p>Are you sure you want to delete this item?</p>
    <button onClick={handleDelete}>Delete</button>
    <button onClick={() => setShowConfirm(false)}>Cancel</button>
  </Dialog>
)}
```

---

## 10. Touch & Interaction (MEDIUM)

### 10.1 Use touch-action: manipulation

**Impact:** MEDIUM (prevents double-tap zoom delay)

#### Correct:

```css
button, a {
  touch-action: manipulation;
}
```

### 10.2 Set -webkit-tap-highlight-color

**Impact:** LOW (consistent tap feedback)

#### Correct:

```css
button, a {
  -webkit-tap-highlight-color: rgba(0, 0, 0, 0.1);
}
```

### 10.3 Use overscroll-behavior: contain in Modals

**Impact:** MEDIUM (prevents background scroll)

#### Correct:

```css
.modal {
  overscroll-behavior: contain;
}
```

---

## 11. Dark Mode & Theming (LOW)

### 11.1 Set color-scheme: dark

**Impact:** MEDIUM (fixes scrollbar, inputs in dark mode)

#### Correct:

```css
:root {
  color-scheme: light dark;
}

html[data-theme="dark"] {
  color-scheme: dark;
}
```

### 11.2 Set theme-color Meta Tag

**Impact:** LOW (matches browser chrome to page)

#### Correct:

```html
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)" />
<meta name="theme-color" content="#000000" media="(prefers-color-scheme: dark)" />
```

---

## 12. Locale & i18n (MEDIUM)

### 12.1 Use Intl.DateTimeFormat for Dates

**Impact:** HIGH (respects user locale)

#### Incorrect:

```tsx
const formatted = `${date.getMonth()}/${date.getDate()}/${date.getFullYear()}`
```

#### Correct:

```tsx
const formatted = new Intl.DateTimeFormat('en-US', {
  dateStyle: 'medium'
}).format(date)
```

### 12.2 Use Intl.NumberFormat for Numbers

**Impact:** HIGH (respects user locale)

#### Incorrect:

```tsx
const formatted = `$${amount.toFixed(2)}`
```

#### Correct:

```tsx
const formatted = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD'
}).format(amount)
```

---

## Anti-Patterns (Flag These)

- `user-scalable=no` or `maximum-scale=1` disabling zoom
- `onPaste` with `preventDefault`
- `transition: all`
- `outline-none` without focus-visible replacement
- Inline `onClick` navigation without `<a>`
- `<div>` or `<span>` with click handlers (should be `<button>`)
- Images without dimensions
- Large arrays `.map()` without virtualization
- Form inputs without labels
- Icon buttons without `aria-label`
- Hardcoded date/number formats (use `Intl.*`)
- `autoFocus` without clear justification

---

## Summary

Web design guidelines ensure:

1. **Accessibility** - Screen readers, keyboard navigation, semantic HTML
2. **Forms** - Autocomplete, labels, inline errors, no paste blocking
3. **Animation** - Reduced motion, GPU-accelerated, interruptible
4. **Performance** - Virtualization, no layout reads, preconnect
5. **Navigation** - URL state, deep-linking, confirmation dialogs
6. **Touch** - Manipulation, tap-highlight, overscroll-behavior
7. **Typography** - Ellipsis, curly quotes, tabular-nums
8. **i18n** - Intl.* formatters, locale detection

These patterns create production-ready, accessible, and user-friendly web interfaces.
