# Performance — Re-renders, Lists, Memory, Animations, Images, Startup

Read this when building UI that renders lists, animates, holds subscriptions,
loads images, or shows jank / high memory. Optimize by measurement, not by
superstition — but these defaults prevent the common regressions.

React Native has two threads that matter for the user: the **JS thread** (your
code, effects, state updates) and the **UI/main thread** (native layout and
drawing). Jank has different fixes depending on which one is blocked, so name the
thread before you optimize.

## 1. Minimize re-renders

The single biggest source of jank on the JS thread is re-rendering too much, too
often.

- **Read stores with selectors.** `useStore()` re-renders on every change in the
  whole store; `useStore((s) => s.user.name)` re-renders on that slice only.
  Selectors that build a new object/array need a shallow comparator
  (`useShallow` in Zustand, `shallowEqual` in Redux).
- **Keep fast-changing state low in the tree.** A `TextInput`'s value held in a
  screen-level state re-renders the whole screen on every keystroke. Move it into
  the input's own component, or use an uncontrolled input + a ref.
- **`React.memo` where it pays** — list items and children under a
  frequently-rendering parent. It only works with **stable props**: hoisted
  styles, `useCallback` handlers, and no inline object/array literals.
- **`useCallback` / `useMemo` are not free.** They allocate and add a dependency
  array to keep honest. Use them for: props to memoized children, dependency
  array inputs, and genuinely expensive computations. Not for every arrow
  function in a leaf component.
- **Memoize context values.** A provider whose `value={{ a, b }}` is a fresh
  object every render re-renders every consumer — the exact opposite of what the
  context was for. Split a context that mixes rarely-changing config with
  fast-changing state into two.
- **Derive, don't store.** A `useState` + `useEffect` pair that recomputes a
  value from props is two extra renders and a stale-value bug; compute it during
  render.

Find them, don't guess: React DevTools Profiler with "Highlight updates", or the
`why-did-you-render` hook in dev builds.

## 2. Lists — the highest-leverage area in most apps

- **Always virtualize.** `FlashList` (recommended) or `FlatList`. A `.map()`
  inside a `ScrollView` mounts every row up front — 500 rows means 500 mounted
  subtrees and an unbounded memory graph.
- **Never nest a `FlatList` inside a `ScrollView`** on the same axis. It disables
  virtualization entirely (that is what the warning means). Use
  `ListHeaderComponent`/`ListFooterComponent`.
- **`keyExtractor` from a stable id**, never the array index.
- **Memoize the row.** `renderItem` should render an already-`memo`'d component
  and be itself hoisted or `useCallback`'d — otherwise every row re-renders on
  every parent render.
- **Give the list a size hint.** `estimatedItemSize` (FlashList) or
  `getItemLayout` (FlatList, fixed heights) removes measurement work and makes
  `scrollToIndex` instant.
- **Keep rows cheap.** No inline styles, no `.map()` inside a row, no shadow on
  every row on Android (each elevation is a layer), no unsized remote images.
- **Pagination:** fetch pages on demand via `onEndReached` +
  `onEndReachedThreshold`, keep the cursor in the query layer, and guard against
  duplicate triggers while a page is in flight. Don't fetch 5,000 rows to show
  20.
- FlatList tuning knobs worth knowing when rows are heavy: `initialNumToRender`,
  `maxToRenderPerBatch`, `windowSize`, `removeClippedSubviews` (Android). Change
  one at a time and measure; these trade blank space for CPU.

## 3. Memory leaks & cleanup — the top mobile bug class

Every subscription, listener, timer, and animation you create, you must tear
down. Leaks manifest as growing memory, "state update on an unmounted component",
and ghost callbacks firing on dead screens.

```tsx
useEffect(() => {
  const sub = locationService.subscribe(onPosition);
  const appStateSub = AppState.addEventListener('change', onAppState);
  const timer = setInterval(tick, 1000);

  return () => {
    sub.unsubscribe();
    appStateSub.remove();
    clearInterval(timer);
  };
}, [onPosition, onAppState]);
```

Things that MUST be cleaned up: event listeners (`AppState`, `Keyboard`,
`Dimensions`, `NetInfo`, `Linking`), native module subscriptions, `setInterval`/
`setTimeout`, WebSocket connections, `Animated` loops, Reanimated shared-value
reactions, in-flight requests (`AbortController`), camera/audio/location
sessions, and any listener you registered on a singleton.

Also: an `AppState` change to `background` should stop location/sensor streams
and pause polling. A screen that keeps a 1 Hz timer running in the background is
a battery complaint waiting to happen.

## 4. Animations — get off the JS thread

- **Reanimated worklets** run on the UI thread and keep animating even while JS
  is busy. This is the default choice for gestures and continuous animation.
- If using the `Animated` API, `useNativeDriver: true` — but note it only
  supports transform and opacity. Animating `width`/`height`/`backgroundColor`
  without it means a JS-thread frame per tick.
- Gesture handling: `react-native-gesture-handler`, not `PanResponder`.
- `LayoutAnimation` is cheap for simple list insert/remove; it is also
  all-or-nothing and Android needs the flag enabled.
- Never animate through `setState` at 60 fps. A state update per frame is a
  re-render per frame.
- Respect reduce-motion (`ui-layout.md` §9) for non-essential motion.

## 5. Async & the JS thread

- Never do heavy CPU work synchronously on the JS thread (parsing a huge JSON
  payload, crypto, image processing). Options: ask the backend for a smaller
  payload (best), move work to a native module or a Reanimated worklet, or chunk
  it with `InteractionManager.runAfterInteractions` so it doesn't compete with a
  transition.
- Don't `await` sequentially when calls are independent — `Promise.all`.
- Debounce high-frequency inputs (search fields, scroll-driven fetches) so you
  don't fire a request per keystroke.
- Let the query layer own async lifecycles: dedupe, cancellation on unmount,
  retry, and background refetch are already solved there. Hand-rolled
  `useEffect` fetching re-fetches on every render loop mistake and races on fast
  navigation.

## 6. Images

- **Always give a remote image an explicit width and height.** Without it the
  layout jumps when it loads, and the decoder has no target size.
- Use the project's caching image component (`expo-image` / `FastImage`): disk +
  memory cache, placeholders, priority, and a real error state.
- Ask the backend for correctly-sized images (a CDN resize parameter) rather than
  downloading a 4000 px original for a 40 px avatar. This is the biggest single
  image-memory win.
- Ship raster assets at `@2x`/`@3x`; prefer SVG (via `react-native-svg`) for
  icons — one file, any size, tintable.
- Prefetch above-the-fold images before navigating (`Image.prefetch` / the
  library's equivalent) to avoid first-frame pop.

## 7. Startup time

- Hermes on, and check that it actually is (`global.HermesInternal`).
- Enable the **New Architecture** where the project's dependencies support it —
  and treat the migration as its own task with its own testing, never as a
  side effect of a feature PR.
- Lazy-load heavy screens and libraries: `React.lazy` + `Suspense` for rarely
  visited screens, dynamic `import()` for a large SDK used in one flow.
- Don't do work at module scope. A module-level `Dimensions.get()`, a store
  hydration, or an analytics init runs during the bundle's first evaluation and
  lands directly on time-to-interactive.
- Keep the splash screen up until the first real frame is ready, then hide it
  explicitly — a flash of an empty screen reads as a crash.
- Watch bundle size: `npx react-native-bundle-visualizer` (or the Expo Atlas
  tool). A single misused date/locale library can dominate it.

## 8. Measure, don't guess

- **React DevTools Profiler** — which components re-render, and how often.
- **Flipper / the new dev tools + Hermes sampling profiler** — where JS time
  goes.
- **Xcode Instruments (Time Profiler, Allocations)** and **Android Studio
  Profiler / Perfetto** — for main-thread and native memory questions.
- **Always profile a release/production build.** Dev builds run unminified with
  the bridge in debug mode and inspector hooks attached; their numbers are
  fiction.
- Test on the **oldest device the app supports**, not the newest. Jank that is
  invisible on a Pro is a one-star review on a mid-range Android.

Optimize the thing the profiler points at. Premature micro-optimization that
hurts readability violates the Golden Rule.
