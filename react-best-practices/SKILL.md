---
name: react-best-practices
description: Performance and best-practice guidelines for React, Next.js, and React Native (Expo) code, from Vercel Engineering. Use when optimizing React components, reducing bundle size, fixing unnecessary re-renders, improving async data fetching or request waterfalls, working with Server Components, Server Actions, or SSR, designing component composition or APIs, building React Native or Expo screens and lists, adding view transitions or animations, or reviewing UI for accessibility and UX.
---

# React & Next.js Best Practices

Performance and best-practice guidance for React, Next.js, and React Native applications, adapted from Vercel Engineering's agent-skills guidelines.

## Step 1: Check the project type

Before applying these rules, confirm the code is React-based. Look for `react`, `next`, `react-native`, or `expo` in `package.json`, or JSX/TSX files.

Then focus on what fits the project:

- **Next.js App Router**: server-performance, async-optimization, bundle-optimization
- **React SPA**: bundle-optimization, client-optimization, rerender-optimization
- **React Native / Expo**: react-native-patterns
- **UI or design review**: web-design-guidelines
- **Animation work**: view-transitions

## Step 2: Load only the relevant reference

Don't load every file. Open just the reference that matches the task. The files are in `references/`.

| Task | Reference |
|---|---|
| Async operations, data fetching, waterfalls, parallelizing requests, Suspense streaming | `references/async-optimization.md` |
| Bundle size, load time, tree shaking, code splitting, dynamic imports, lazy loading | `references/bundle-optimization.md` |
| Server Components, Server Actions, SSR, server-side caching | `references/server-performance.md` |
| Client-side fetching and caching, SWR, event listeners, localStorage | `references/client-optimization.md` |
| Unnecessary re-renders, memo, useMemo, useCallback, slow components | `references/rerender-optimization.md` |
| Rendering, hydration, DOM and SVG performance, content-visibility, resource hints | `references/rendering-performance.md` |
| JavaScript micro-optimizations (loops, map/filter, Set, caching) | `references/javascript-performance.md` |
| Advanced React: useEffectEvent, refs, initialization | `references/advanced-patterns.md` |
| Component architecture, compound components, boolean props, API design | `references/composition-patterns.md` |
| React Native and Expo: lists, FlashList, animations, navigation, expo-image | `references/react-native-patterns.md` |
| Page transitions, shared element transitions, the View Transitions API | `references/view-transitions.md` |
| Accessibility, ARIA, keyboard navigation, UX and design review, WCAG | `references/web-design-guidelines.md` |

## Priority levels

Rules are ranked by expected impact. Address higher levels first.

- **CRITICAL**: highest priority, large gains (2-10x). Usually waterfalls and bundle size.
- **HIGH**: significant improvements
- **MEDIUM-HIGH**: moderate-to-high gains
- **MEDIUM**: moderate improvements
- **LOW-MEDIUM**: small-to-moderate gains
- **LOW**: incremental improvements and advanced patterns

## Working principles

1. **Start with CRITICAL rules.** Eliminate waterfalls and reduce bundle size before micro-optimizing.
2. **Measure before and after.** Use profiling to confirm an optimization helped.
3. **Apply in context.** Not every rule fits every situation.
4. **Check for React Compiler.** On React 19+ with the compiler enabled, many manual memoization rules are unnecessary.
5. **Test the change.** Performance optimizations can introduce bugs.

## Credits and license

Original content from [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills), created by [@shuding](https://x.com/shuding) and the Vercel Engineering team. Licensed under MIT.
