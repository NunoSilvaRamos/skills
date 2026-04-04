name: "react-best-practices"
displayName: "React & Next.js Best Practices"
description: "Comprehensive React, Next.js, and React Native performance optimization guidelines from Vercel Engineering. Covers async patterns, bundle optimization, server/client performance, re-render optimization, composition patterns, animations, and web design best practices."
keywords: [
  "react", "nextjs", "next.js", "next", "performance", "optimization", "optimize",
  "bundle", "bundle size", "tree shaking", "code splitting", "dynamic import",
  "async", "promises", "await", "parallel", "waterfalls", "loading", "fetching",
  "server components", "rsc", "server actions", "ssr", "hydration",
  "rerender", "re-render", "memo", "usememo", "usecallback", "react compiler",
  "hooks", "useeffect", "usestate", "useref", "context", "suspense",
  "composition", "compound components", "render props", "patterns",
  "react native", "expo", "mobile", "ios", "android", "flashlist",
  "animations", "transitions", "view transitions", "reanimated", "gestures",
  "accessibility", "a11y", "aria", "wcag", "ui", "ux", "design", "web design",
  "typescript", "javascript", "jsx", "tsx", "component", "components"
]
---

# React & Next.js Best Practices Power

Comprehensive performance optimization and best practices for React, Next.js, and React Native applications from Vercel Engineering.

## What's Included

This Power packages 100+ rules and patterns across multiple categories:

- **69 React/Next.js performance rules** across 8 priority levels (async, bundle, server, client, rerender, rendering, JavaScript, advanced)
- **React composition patterns** for scalable component architecture
- **React Native and Expo best practices** for mobile performance
- **View Transitions API guide** for smooth, native-feeling animations
- **100+ web design and accessibility guidelines** covering UX, a11y, performance, and best practices

## Onboarding

### Step 1: Validate Environment

Before applying these guidelines, Kiro will check if this is a React-based project by looking for:
- `package.json` with `react`, `next`, `react-native`, or `expo` dependencies
- TypeScript or JavaScript files with JSX/TSX syntax
- React component patterns in the codebase

### Step 2: Identify Project Type

Kiro will determine which steering files are most relevant based on your project:

- **Next.js App Router** → Focus on server-performance, async-optimization, bundle-optimization
- **React SPA** → Focus on bundle-optimization, client-optimization, rerender-optimization
- **React Native/Expo** → Focus on react-native-patterns
- **UI/Design Review** → Focus on web-design-guidelines
- **Animation Work** → Focus on view-transitions

### Step 3: Apply Contextually

These guidelines are applied contextually based on what you're working on. Kiro will load the relevant steering file when you mention related keywords or work on specific features.

## When to Load Steering Files

Kiro automatically loads the appropriate steering file based on your work context:

- **Optimizing async operations or data fetching** → `async-optimization.md`
  - Keywords: async, await, promises, parallel, waterfalls, fetching, loading
  
- **Reducing bundle size or improving load times** → `bundle-optimization.md`
  - Keywords: bundle, bundle size, tree shaking, code splitting, dynamic import, lazy load
  
- **Working with Server Components, Server Actions, or SSR** → `server-performance.md`
  - Keywords: server components, rsc, server actions, ssr, server-side, caching
  
- **Implementing client-side data fetching or caching** → `client-optimization.md`
  - Keywords: client-side, swr, fetch, cache, event listeners, localstorage
  
- **Fixing unnecessary re-renders or performance issues** → `rerender-optimization.md`
  - Keywords: rerender, re-render, memo, usememo, usecallback, performance, slow
  
- **Optimizing rendering, hydration, or DOM performance** → `rendering-performance.md`
  - Keywords: rendering, hydration, dom, svg, content-visibility, resource hints
  
- **Micro-optimizing JavaScript code** → `javascript-performance.md`
  - Keywords: javascript performance, loops, map, filter, set, cache
  
- **Using advanced React patterns** → `advanced-patterns.md`
  - Keywords: useeffectevent, refs, advanced patterns, initialization
  
- **Refactoring components with many props or building component libraries** → `composition-patterns.md`
  - Keywords: composition, compound components, boolean props, component library, api design
  
- **Building React Native or Expo apps** → `react-native-patterns.md`
  - Keywords: react native, expo, mobile, ios, android, flashlist, reanimated
  
- **Implementing page transitions or animations** → `view-transitions.md`
  - Keywords: animations, transitions, view transitions, page transitions, shared element
  
- **Reviewing UI for accessibility, UX, or design best practices** → `web-design-guidelines.md`
  - Keywords: accessibility, a11y, aria, wcag, ui review, ux, design, audit

## Priority Levels

Rules are organized by impact to help you focus on what matters most:

- **CRITICAL** - Highest priority, major performance gains (2-10× improvements)
- **HIGH** - Significant performance improvements
- **MEDIUM-HIGH** - Moderate-high gains
- **MEDIUM** - Moderate performance improvements
- **LOW-MEDIUM** - Low-medium gains
- **LOW** - Incremental improvements, advanced patterns

## Best Practices

When applying these guidelines:

1. **Start with CRITICAL rules** - Focus on eliminating waterfalls and reducing bundle size first
2. **Measure before and after** - Use performance profiling to validate improvements
3. **Apply contextually** - Not every rule applies to every situation
4. **Consider React Compiler** - If using React 19+ with React Compiler, many manual memoization rules are unnecessary
5. **Test thoroughly** - Performance optimizations can introduce bugs if not tested properly

## Credits

Original content from [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills)

Created by [@shuding](https://x.com/shuding) and the Vercel Engineering team.

## License

MIT
