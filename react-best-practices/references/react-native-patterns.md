# React Native Patterns (CRITICAL Priority)

Comprehensive performance optimization guide for React Native applications. Contains rules across multiple categories, prioritized by impact from critical (core rendering, list performance) to incremental (fonts, imports).

**Impact:** 2-10× performance improvement in mobile applications, native-quality UX, and crash prevention.

---

## Quick Reference

| Category | Priority | Key Rules |
|----------|----------|-----------|
| Core Rendering | CRITICAL | Never use && with falsy values, wrap strings in Text |
| List Performance | HIGH | Use FlashList/LegendList, avoid inline objects, pass primitives |
| Animation | HIGH | Animate transform/opacity only, use GestureDetector |
| Navigation | HIGH | Use native navigators (native-stack, native tabs) |
| React State | MEDIUM | Minimize state, use fallback pattern, dispatch updaters |
| UI Patterns | MEDIUM | expo-image, Galeria, native menus, Pressable |
| Monorepo | LOW | Install native deps in app, single versions |

---

## 1. Core Rendering (CRITICAL)

### 1.1 Never Use && with Potentially Falsy Values

**Impact:** CRITICAL (prevents production crash)

Never use `{value && <Component />}` when `value` could be an empty string or `0`. These are falsy but JSX-renderable—React Native will try to render them as text outside a `<Text>` component, causing a hard crash in production.

#### Incorrect (crashes if count is 0 or name is ""):

```tsx
function Profile({ name, count }: { name: string; count: number }) {
  return (
    <View>
      {name && <Text>{name}</Text>}
      {count && <Text>{count} items</Text>}
    </View>
  )
}
// If name="" or count=0, renders the falsy value → crash
```

#### Correct (ternary with null):

```tsx
function Profile({ name, count }: { name: string; count: number }) {
  return (
    <View>
      {name ? <Text>{name}</Text> : null}
      {count ? <Text>{count} items</Text> : null}
    </View>
  )
}
```

#### Best (early return):

```tsx
function Profile({ name, count }: { name: string; count: number }) {
  if (!name) return null

  return (
    <View>
      <Text>{name}</Text>
      {count > 0 ? <Text>{count} items</Text> : null}
    </View>
  )
}
```

**Lint rule:** Enable `react/jsx-no-leaked-render` from eslint-plugin-react to catch this automatically.

### 1.2 Wrap Strings in Text Components

**Impact:** CRITICAL (prevents runtime crash)

Strings must be rendered inside `<Text>`. React Native crashes if a string is a direct child of `<View>`.

#### Incorrect (crashes):

```tsx
import { View } from 'react-native'

function Greeting({ name }: { name: string }) {
  return <View>Hello, {name}!</View>
}
// Error: Text strings must be rendered within a <Text> component.
```

#### Correct:

```tsx
import { View, Text } from 'react-native'

function Greeting({ name }: { name: string }) {
  return (
    <View>
      <Text>Hello, {name}!</Text>
    </View>
  )
}
```

---

## 2. List Performance (HIGH)

### 2.1 Use a List Virtualizer for Any List

**Impact:** HIGH (reduced memory, faster mounts)

Use a list virtualizer like LegendList or FlashList instead of ScrollView with mapped children—even for short lists. Virtualizers only render visible items, reducing memory usage and mount time.

#### Incorrect (ScrollView renders all items at once):

```tsx
function Feed({ items }: { items: Item[] }) {
  return (
    <ScrollView>
      {items.map((item) => (
        <ItemCard key={item.id} item={item} />
      ))}
    </ScrollView>
  )
}
// 50 items = 50 components mounted, even if only 10 visible
```

#### Correct (virtualizer renders only visible items):

```tsx
import { LegendList } from '@legendapp/list'

function Feed({ items }: { items: Item[] }) {
  return (
    <LegendList
      data={items}
      renderItem={({ item }) => <ItemCard item={item} />}
      keyExtractor={(item) => item.id}
      estimatedItemSize={80}
    />
  )
}
// Only ~10-15 visible items mounted at a time
```

### 2.2 Avoid Inline Objects in renderItem

**Impact:** HIGH (prevents unnecessary re-renders of memoized list items)

Don't create new objects inside `renderItem` to pass as props. Inline objects create new references on every render, breaking memoization. Pass primitive values directly from `item` instead.

#### Incorrect (inline object breaks memoization):

```tsx
function UserList({ users }: { users: User[] }) {
  return (
    <LegendList
      data={users}
      renderItem={({ item }) => (
        <UserRow
          // Bad: new object on every render
          user={{ id: item.id, name: item.name, avatar: item.avatar }}
        />
      )}
    />
  )
}
```

#### Correct (pass item directly or primitives):

```tsx
function UserList({ users }: { users: User[] }) {
  return (
    <LegendList
      data={users}
      renderItem={({ item }) => (
        // Good: pass the item directly
        <UserRow user={item} />
      )}
    />
  )
}
```

#### Correct (pass primitives, derive inside child):

```tsx
renderItem={({ item }) => (
  <UserRow
    id={item.id}
    name={item.name}
    isActive={item.isActive}
  />
)}

const UserRow = memo(function UserRow({ id, name, isActive }: Props) {
  // Good: derive style inside memoized component
  const backgroundColor = isActive ? 'green' : 'gray'
  return <View style={[styles.row, { backgroundColor }]}>{/* ... */}</View>
})
```

### 2.3 Optimize List Performance with Stable Object References

**Impact:** CRITICAL (virtualization relies on reference stability)

Don't map or filter data before passing to virtualized lists. Virtualization relies on object reference stability to know what changed—new references cause full re-renders of all visible items.

#### Incorrect (creates new object references on every keystroke):

```tsx
function DomainSearch() {
  const { keyword, setKeyword } = useKeywordZustandState()
  const { data: tlds } = useTlds()

  // Bad: creates new objects on every render
  const domains = tlds.map((tld) => ({
    domain: `${keyword}.${tld.name}`,
    tld: tld.name,
    price: tld.price,
  }))

  return (
    <>
      <TextInput value={keyword} onChangeText={setKeyword} />
      <LegendList
        data={domains}
        renderItem={({ item }) => <DomainItem item={item} keyword={keyword} />}
      />
    </>
  )
}
```

#### Correct (stable references, transform inside items):

```tsx
const renderItem = ({ item }) => <DomainItem tld={item} />

function DomainSearch() {
  const { data: tlds } = useTlds()

  return (
    <LegendList
      // Good: stable references
      data={tlds}
      renderItem={renderItem}
    />
  )
}

function DomainItem({ tld }: { tld: Tld }) {
  // Good: transform within items
  const domain = useKeywordZustandState((s) => s.keyword + '.' + tld.name)
  return <Text>{domain}</Text>
}
```

### 2.4 Keep List Items Lightweight

**Impact:** HIGH (reduces render time for visible items during scroll)

List items should be as inexpensive as possible to render. Minimize hooks, avoid queries, and limit React Context access. Virtualized lists render many items during scroll—expensive items cause jank.

#### Incorrect (heavy list item):

```tsx
function ProductRow({ id }: { id: string }) {
  // Bad: query inside list item
  const { data: product } = useQuery(['product', id], () => fetchProduct(id))
  // Bad: multiple context accesses
  const theme = useContext(ThemeContext)
  const user = useContext(UserContext)
  const cart = useContext(CartContext)

  return <View>{/* ... */}</View>
}
```

#### Correct (lightweight list item):

```tsx
function ProductRow({ name, price, imageUrl }: Props) {
  // Good: receives only primitives, minimal hooks
  return (
    <View>
      <Image source={{ uri: imageUrl }} />
      <Text>{name}</Text>
      <Text>{price}</Text>
    </View>
  )
}

// Parent fetches all data once
function ProductList() {
  const { data: products } = useQuery(['products'], fetchProducts)

  return (
    <LegendList
      data={products}
      renderItem={({ item }) => (
        <ProductRow name={item.name} price={item.price} imageUrl={item.image} />
      )}
    />
  )
}
```

### 2.5 Use Compressed Images in Lists

**Impact:** HIGH (faster load times, less memory)

Always load compressed, appropriately-sized images in lists. Full-resolution images consume excessive memory and cause scroll jank. Request thumbnails from your server or use an image CDN with resize parameters.

#### Incorrect (full-resolution images):

```tsx
function ProductItem({ product }: { product: Product }) {
  return (
    <View>
      {/* 4000x3000 image loaded for a 100x100 thumbnail */}
      <Image
        source={{ uri: product.imageUrl }}
        style={{ width: 100, height: 100 }}
      />
      <Text>{product.name}</Text>
    </View>
  )
}
```

#### Correct (request appropriately-sized image):

```tsx
function ProductItem({ product }: { product: Product }) {
  // Request a 200x200 image (2x for retina)
  const thumbnailUrl = `${product.imageUrl}?w=200&h=200&fit=cover`

  return (
    <View>
      <Image
        source={{ uri: thumbnailUrl }}
        style={{ width: 100, height: 100 }}
        contentFit='cover'
      />
      <Text>{product.name}</Text>
    </View>
  )
}
```

---

## 3. Animation (HIGH)

### 3.1 Animate Transform and Opacity Instead of Layout Properties

**Impact:** HIGH (GPU-accelerated animations, no layout recalculation)

Avoid animating `width`, `height`, `top`, `left`, `margin`, or `padding`. These trigger layout recalculation on every frame. Instead, use `transform` (scale, translate) and `opacity` which run on the GPU without triggering layout.

#### Incorrect (animates height, triggers layout every frame):

```tsx
import Animated, { useAnimatedStyle, withTiming } from 'react-native-reanimated'

function CollapsiblePanel({ expanded }: { expanded: boolean }) {
  const animatedStyle = useAnimatedStyle(() => ({
    height: withTiming(expanded ? 200 : 0), // triggers layout on every frame
    overflow: 'hidden',
  }))

  return <Animated.View style={animatedStyle}>{children}</Animated.View>
}
```

#### Correct (animates scaleY, GPU-accelerated):

```tsx
import Animated, { useAnimatedStyle, withTiming } from 'react-native-reanimated'

function CollapsiblePanel({ expanded }: { expanded: boolean }) {
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { scaleY: withTiming(expanded ? 1 : 0) },
    ],
    opacity: withTiming(expanded ? 1 : 0),
  }))

  return (
    <Animated.View style={[{ height: 200, transformOrigin: 'top' }, animatedStyle]}>
      {children}
    </Animated.View>
  )
}
```

**GPU-accelerated properties:** `transform` (translate, scale, rotate), `opacity`. Everything else triggers layout.

### 3.2 Use GestureDetector for Animated Press States

**Impact:** MEDIUM (UI thread animations, smoother press feedback)

For animated press states (scale, opacity on press), use `GestureDetector` with `Gesture.Tap()` and shared values instead of Pressable's `onPressIn`/`onPressOut`. Gesture callbacks run on the UI thread as worklets—no JS thread round-trip for press animations.

#### Incorrect (Pressable with JS thread callbacks):

```tsx
import { Pressable } from 'react-native'
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
} from 'react-native-reanimated'

function AnimatedButton({ onPress }: { onPress: () => void }) {
  const scale = useSharedValue(1)

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }))

  return (
    <Pressable
      onPress={onPress}
      onPressIn={() => (scale.value = withTiming(0.95))}
      onPressOut={() => (scale.value = withTiming(1))}
    >
      <Animated.View style={animatedStyle}>
        <Text>Press me</Text>
      </Animated.View>
    </Pressable>
  )
}
```

#### Correct (GestureDetector with UI thread worklets):

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler'
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  interpolate,
  runOnJS,
} from 'react-native-reanimated'

function AnimatedButton({ onPress }: { onPress: () => void }) {
  // Store the press STATE (0 = not pressed, 1 = pressed)
  const pressed = useSharedValue(0)

  const tap = Gesture.Tap()
    .onBegin(() => {
      pressed.set(withTiming(1))
    })
    .onFinalize(() => {
      pressed.set(withTiming(0))
    })
    .onEnd(() => {
      runOnJS(onPress)()
    })

  // Derive visual values from the state
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { scale: interpolate(withTiming(pressed.get()), [0, 1], [1, 0.95]) },
    ],
  }))

  return (
    <GestureDetector gesture={tap}>
      <Animated.View style={animatedStyle}>
        <Text>Press me</Text>
      </Animated.View>
    </GestureDetector>
  )
}
```

**Benefit:** Store the press state (0 or 1), then derive the scale via `interpolate`. Use `runOnJS` to call JS functions from worklets. Use `.set()` and `.get()` for React Compiler compatibility.

---

## 4. Navigation (HIGH)

### 4.1 Use Native Navigators for Navigation

**Impact:** HIGH (native performance, platform-appropriate UI)

Always use native navigators instead of JS-based ones. Native navigators use platform APIs (UINavigationController on iOS, Fragment on Android) for better performance and native behavior.

**For stacks:** Use `@react-navigation/native-stack` or expo-router's default stack. Avoid `@react-navigation/stack`.

**For tabs:** Use `react-native-bottom-tabs` (native) or expo-router's native tabs. Avoid `@react-navigation/bottom-tabs` when native feel matters.

#### Incorrect (JS stack navigator):

```tsx
import { createStackNavigator } from '@react-navigation/stack'

const Stack = createStackNavigator()

function App() {
  return (
    <Stack.Navigator>
      <Stack.Screen name='Home' component={HomeScreen} />
      <Stack.Screen name='Details' component={DetailsScreen} />
    </Stack.Navigator>
  )
}
```

#### Correct (native stack with react-navigation):

```tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack'

const Stack = createNativeStackNavigator()

function App() {
  return (
    <Stack.Navigator>
      <Stack.Screen name='Home' component={HomeScreen} />
      <Stack.Screen name='Details' component={DetailsScreen} />
    </Stack.Navigator>
  )
}
```

#### Correct (expo-router uses native stack by default):

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router'

export default function Layout() {
  return <Stack />
}
```

---

## 5. React State (MEDIUM)

### 5.1 Minimize State Variables and Derive Values

**Impact:** MEDIUM (fewer re-renders, less state drift)

Use the fewest state variables possible. If a value can be computed from existing state or props, derive it during render instead of storing it in state. Redundant state causes unnecessary re-renders and can drift out of sync.

#### Incorrect (redundant state):

```tsx
function Cart({ items }: { items: Item[] }) {
  const [total, setTotal] = useState(0)
  const [itemCount, setItemCount] = useState(0)

  useEffect(() => {
    setTotal(items.reduce((sum, item) => sum + item.price, 0))
    setItemCount(items.length)
  }, [items])

  return (
    <View>
      <Text>{itemCount} items</Text>
      <Text>Total: ${total}</Text>
    </View>
  )
}
```

#### Correct (derived values):

```tsx
function Cart({ items }: { items: Item[] }) {
  const total = items.reduce((sum, item) => sum + item.price, 0)
  const itemCount = items.length

  return (
    <View>
      <Text>{itemCount} items</Text>
      <Text>Total: ${total}</Text>
    </View>
  )
}
```

**Benefit:** State should be the minimal source of truth. Everything else is derived.

### 5.2 Use Fallback State Instead of initialState

**Impact:** MEDIUM (reactive fallbacks without syncing)

Use `undefined` as initial state and nullish coalescing (`??`) to fall back to parent or server values. State represents user intent only—`undefined` means "user hasn't chosen yet." This enables reactive fallbacks that update when the source changes.

#### Incorrect (syncs state, loses reactivity):

```tsx
type Props = { fallbackEnabled: boolean }

function Toggle({ fallbackEnabled }: Props) {
  const [enabled, setEnabled] = useState(defaultEnabled)
  // If fallbackEnabled changes, state is stale

  return <Switch value={enabled} onValueChange={setEnabled} />
}
```

#### Correct (state is user intent, reactive fallback):

```tsx
type Props = { fallbackEnabled: boolean }

function Toggle({ fallbackEnabled }: Props) {
  const [_enabled, setEnabled] = useState<boolean | undefined>(undefined)
  const enabled = _enabled ?? defaultEnabled
  // undefined = user hasn't touched it, falls back to prop
  // If defaultEnabled changes, component reflects it

  return <Switch value={enabled} onValueChange={setEnabled} />
}
```

---

## 6. UI Patterns (MEDIUM)

### 6.1 Use expo-image for Optimized Images

**Impact:** HIGH (memory efficiency, caching, blurhash placeholders)

Use `expo-image` instead of React Native's `Image`. It provides memory-efficient caching, blurhash placeholders, progressive loading, and better performance for lists.

#### Incorrect (React Native Image):

```tsx
import { Image } from 'react-native'

function Avatar({ url }: { url: string }) {
  return <Image source={{ uri: url }} style={styles.avatar} />
}
```

#### Correct (expo-image):

```tsx
import { Image } from 'expo-image'

function Avatar({ url }: { url: string }) {
  return <Image source={{ uri: url }} style={styles.avatar} />
}
```

#### With blurhash placeholder:

```tsx
<Image
  source={{ uri: url }}
  placeholder={{ blurhash: 'LGF5]+Yk^6#M@-5c,1J5@[or[Q6.' }}
  contentFit="cover"
  transition={200}
  style={styles.image}
/>
```

### 6.2 Use Pressable Instead of Touchable Components

**Impact:** LOW (modern API, more flexible)

Never use `TouchableOpacity` or `TouchableHighlight`. Use `Pressable` from `react-native` or `react-native-gesture-handler` instead.

#### Incorrect (legacy Touchable components):

```tsx
import { TouchableOpacity } from 'react-native'

function MyButton({ onPress }: { onPress: () => void }) {
  return (
    <TouchableOpacity onPress={onPress} activeOpacity={0.7}>
      <Text>Press me</Text>
    </TouchableOpacity>
  )
}
```

#### Correct (Pressable):

```tsx
import { Pressable } from 'react-native'

function MyButton({ onPress }: { onPress: () => void }) {
  return (
    <Pressable onPress={onPress}>
      <Text>Press me</Text>
    </Pressable>
  )
}
```

### 6.3 Modern React Native Styling Patterns

**Impact:** MEDIUM (consistent design, smoother borders, cleaner layouts)

Follow these styling patterns for cleaner, more consistent React Native code.

**Always use `borderCurve: 'continuous'` with `borderRadius`:**

```tsx
<View style={{ borderRadius: 12, borderCurve: 'continuous' }} />
```

**Use `gap` instead of margin for spacing between elements:**

```tsx
// Incorrect – margin on children
<View>
  <Text style={{ marginBottom: 8 }}>Title</Text>
  <Text style={{ marginBottom: 8 }}>Subtitle</Text>
</View>

// Correct – gap on parent
<View style={{ gap: 8 }}>
  <Text>Title</Text>
  <Text>Subtitle</Text>
</View>
```

**Use CSS `boxShadow` string syntax for shadows:**

```tsx
// Incorrect – legacy shadow objects
{ shadowColor: '#000', shadowOffset: { width: 0, height: 2 }, shadowOpacity: 0.1 }

// Correct – CSS box-shadow syntax
{ boxShadow: '0 2px 8px rgba(0, 0, 0, 0.1)' }
```

---

## 7. Monorepo (LOW)

### 7.1 Install Native Dependencies in App Directory

**Impact:** CRITICAL (required for autolinking to work)

In a monorepo, packages with native code must be installed in the native app's directory directly. Autolinking only scans the app's `node_modules`—it won't find native dependencies installed in other packages.

#### Incorrect (native dep in shared package only):

```typescript
packages/
  ui/
    package.json  # has react-native-reanimated
  app/
    package.json  # missing react-native-reanimated
```

#### Correct (native dep in app directory):

```json
// packages/app/package.json
{
  "dependencies": {
    "react-native-reanimated": "3.16.1"
  }
}
```

**Benefit:** Even if the shared package uses the native dependency, the app must also list it for autolinking to detect and link the native code.

---

## Summary

React Native optimization is about:

1. **Preventing crashes** - Never use && with falsy values, wrap strings in Text
2. **List performance** - Use virtualizers, stable references, lightweight items
3. **GPU animations** - Animate transform/opacity only, use GestureDetector
4. **Native APIs** - Use native navigators, expo-image, native menus
5. **Minimal state** - Derive values, use fallback pattern, dispatch updaters
6. **Modern patterns** - Pressable, gap, boxShadow, borderCurve

These patterns compound: applying multiple rules to the same code path can yield 10× or greater improvements in mobile app performance.
