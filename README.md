# React & Next.js Best Practices Power

Comprehensive performance optimization guidelines for React, Next.js, and React Native applications from Vercel Engineering.

## What's Included

- **69 React/Next.js performance rules** across 8 categories (async, bundle, server, client, rerender, rendering, JavaScript, advanced)
- **React composition patterns** for scalable component architecture
- **React Native and Expo best practices** for mobile performance
- **View Transitions API guide** for smooth, native-feeling animations
- **100+ web design and accessibility guidelines** covering UX, a11y, performance, and best practices

## Installation

### Via Kiro Powers Panel (Recommended)

1. Open Kiro → Powers panel
2. Click "Add power from GitHub"
3. Enter: `https://github.com/[your-username]/power-react-best-practices`
4. Click Install

### Via Local Path (Development)

1. Clone this repository
2. Open Kiro → Powers panel
3. Click "Add power from Local Path"
4. Select the cloned directory

## Usage

The power activates automatically when you mention relevant keywords in your conversation with Kiro:

**Performance Optimization:**
- "Optimize this React component"
- "Reduce bundle size"
- "Fix re-render issues"
- "Improve Next.js performance"
- "Speed up data fetching"

**Architecture & Patterns:**
- "Refactor this component"
- "Review component architecture"
- "Implement compound components"
- "Add composition patterns"

**Mobile Development:**
- "Optimize React Native performance"
- "Improve list rendering"
- "Add animations with Reanimated"

**UI & Accessibility:**
- "Review accessibility"
- "Audit UI design"
- "Check WCAG compliance"
- "Improve UX"

**Animations:**
- "Add page transitions"
- "Implement view transitions"
- "Create shared element animations"

## What You'll Learn

### Async Optimization (CRITICAL)
- Eliminate request waterfalls with Promise.all()
- Defer await to branches where values are used
- Parallelize independent operations
- Use Suspense boundaries for streaming

### Bundle Optimization (CRITICAL)
- Avoid barrel file imports (200-800ms savings)
- Use dynamic imports for code splitting
- Defer third-party scripts
- Preload on hover for perceived speed

### Server Performance (HIGH)
- Use React.cache() for deduplication
- Implement LRU caching for cross-request data
- Parallelize server-side fetches
- Use after() for non-blocking operations

### Re-render Optimization (MEDIUM)
- Extract expensive work to memoized components
- Use primitive dependencies in effects
- Implement useTransition for non-urgent updates
- Avoid inline component definitions

### Composition Patterns (HIGH)
- Avoid boolean prop proliferation
- Build compound components with context
- Lift state to provider components
- Use children over render props

### React Native (CRITICAL for mobile)
- Use FlashList for large lists
- Animate only transform and opacity
- Use expo-image for all images
- Implement native navigators

### View Transitions (MEDIUM)
- Implement shared element transitions
- Add directional navigation animations
- Use Suspense reveal patterns
- Handle list reordering with animations

### Web Design & Accessibility (HIGH)
- Implement proper ARIA labels
- Ensure keyboard navigation
- Use semantic HTML
- Support prefers-reduced-motion
- Optimize for touch interactions

## Priority Levels

Rules are organized by impact:

- **CRITICAL** - 2-10× performance improvements
- **HIGH** - Significant gains
- **MEDIUM** - Moderate improvements
- **LOW** - Incremental optimizations

## Structure

```
power-react-best-practices/
├── POWER.md                           # Power metadata and onboarding
├── README.md                          # This file
└── steering/                          # Detailed guidelines
    ├── async-optimization.md          # Async patterns (CRITICAL)
    ├── bundle-optimization.md         # Bundle size (CRITICAL)
    ├── server-performance.md          # Server-side (HIGH)
    ├── client-optimization.md         # Client-side (MEDIUM-HIGH)
    ├── rerender-optimization.md       # Re-renders (MEDIUM)
    ├── rendering-performance.md       # Rendering (MEDIUM)
    ├── javascript-performance.md      # JS micro-optimizations (LOW-MEDIUM)
    ├── advanced-patterns.md           # Advanced React (LOW)
    ├── composition-patterns.md        # Component architecture (HIGH)
    ├── react-native-patterns.md       # Mobile (CRITICAL for RN)
    ├── view-transitions.md            # Animations (MEDIUM)
    └── web-design-guidelines.md       # UI/UX/A11y (HIGH)
```

## Publisher

**Nuno Ramos**

## Credits

Original content from [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills)

Created by [@shuding](https://x.com/shuding) and the Vercel Engineering team.

## License

MIT

## Contributing

Found an issue or want to suggest improvements? Please open an issue or pull request in the repository.

## Resources

- [React Documentation](https://react.dev)
- [Next.js Documentation](https://nextjs.org)
- [React Native Documentation](https://reactnative.dev)
- [Vercel Blog](https://vercel.com/blog)
- [Web Content Accessibility Guidelines (WCAG)](https://www.w3.org/WAI/WCAG21/quickref/)
