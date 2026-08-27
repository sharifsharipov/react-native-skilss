# Layout & Flexbox — how not to break the screen

Read this **together with `ui-from-design.md`** whenever you draw UI. That file
answers *what values and components to reach for*; this one answers *how the
layout survives real devices* — long text, small screens, large system fonts, a
keyboard, a notch, a gesture bar, dark mode, and an empty API response.

Most AI-drawn React Native UI fails here, not in the tokens. A screen that looks
right on an iPhone 15 Pro simulator and clips its content on a 320 dp Android
phone at `fontScale` 1.3 is a defect, not a near miss.

> **On the names in the code samples.** `theme.spacing.md`, `theme.colors.*`,
> `theme.typography.*`, `AppHeader`, `AppButton`, `Skeleton` are illustrative
> token and component names — substitute your project's equivalents (see the
> *Calibrate* section of `ui-from-design.md`). Every layout rule here is
> framework-level and applies unchanged.

---

## 1. The layout model — what differs from the web

React Native uses flexbox, with defaults that trip up anyone coming from CSS:

| | React Native | CSS |
| --- | --- | --- |
| `display` | always flex | `block` |
| `flexDirection` default | `column` | `row` |
| `position` default | `relative` | `static` |
| `flex` shorthand | a single number | three values |
| Units | density-independent points, unitless numbers | px/em/rem/% |
| Box model | `width` never includes padding (`border-box`-ish, always) | configurable |
| Percentages | work for width/height/margins/padding | everywhere |
| `overflow` | `visible` by default; **Android clips children outside a parent's bounds anyway** | `visible` |
| Text | only renders inside `<Text>`; styles do **not** cascade to children except within `<Text>` | cascades |

Consequences you must hold in your head while drawing:

- **A child cannot escape its parent on Android.** Absolutely positioned badges,
  overlapping avatars, and shadows drawn outside a container work on iOS and are
  clipped on Android. Give the parent room, or accept the clip deliberately.
- **`flex: 1` means "take the remaining space of the parent's main axis"** — it
  does nothing if the parent has no bounded size in that axis.
- **A `ScrollView` gives its content unbounded height** in the scroll axis.
  `flex: 1` on a direct child of the content container is meaningless; use
  `contentContainerStyle={{ flexGrow: 1 }}` when the content must fill the
  viewport but still scroll when it can't.
- **Text does not shrink on its own.** A long string in a row pushes siblings out
  unless the text container has `flex: 1`/`flexShrink: 1` and the `<Text>` has
  `numberOfLines`.
- **`gap` works** (RN 0.71+) and is preferable to margin chains. On older
  versions use a spacer component, not alternating margins.

When unsure what size a view is actually getting, don't guess — see §10.

## 2. Choosing the layout

| The design shows | Use | Notes |
| --- | --- | --- |
| Items side by side | `flexDirection: 'row'` | Long text child → `flex: 1` + `numberOfLines` |
| Items stacked vertically | default column | `gap` for rhythm, not margins on each child |
| One item fills the leftover space | `flex: 1` | Ratio split → `flex: 2` / `flex: 1` |
| Item takes at most what it needs | `flexShrink: 1` | Pairs with `numberOfLines` on text |
| Push apart / fixed empty space | `justifyContent: 'space-between'` or a `<View style={{ flex: 1 }} />` spacer | Fixed gap → the spacing token |
| Overlap (badge, avatar ring, gradient scrim) | `position: 'absolute'` inside a `relative` parent | Parent needs bounded size; watch Android clipping |
| Chips that wrap to the next line | `flexWrap: 'wrap'` + `gap` | Never a fixed-width row that can overflow |
| Equal-height siblings | `alignItems: 'stretch'` (the default) | Don't measure and set heights by hand |
| Fixed aspect box (video, map, banner) | `aspectRatio: 16 / 9` | Better than magic heights |
| Centering one child | `justifyContent: 'center', alignItems: 'center'` | Not two spacers |
| Size relative to parent | percentage width or `onLayout`/`useWindowDimensions` | Not `Dimensions.get()` at module scope |
| Scrollable list | `FlashList` / `FlatList` | Never `.map()` in a `ScrollView` for N items |
| Grid | `FlashList` with `numColumns` | Or `flexWrap` for a handful of fixed items |
| A box that only has a shape | `View` + a style | See `ui-from-design.md` Recipe A/B |

## 3. The layout failures and their fixes

These are the actual runtime symptoms. Learn the fix, not the symptom.

| Symptom | Cause | Fix |
| --- | --- | --- |
| Text pushes the trailing icon off screen | Text container has no flex | `flex: 1` on the text block + `numberOfLines` + `ellipsizeMode` |
| Text truncates too early / mid-word on Android | Nested text without flex, or `numberOfLines` on the wrong node | Put `numberOfLines` on the leaf `<Text>`, `flexShrink: 1` on its container |
| Content cut off at the bottom, no scroll | Column taller than the screen, no `ScrollView` | Make the screen scrollable (§4) — do **not** shrink paddings to make it fit |
| `ScrollView` doesn't scroll | Its parent has no bounded height, or `flex: 1` is missing on the `ScrollView` | `flex: 1` on the ScrollView, bounded parent |
| Nested scroll fights / inner list won't scroll | A `FlatList` inside a `ScrollView` on the same axis | One scroller: `FlatList` with `ListHeaderComponent`/`ListFooterComponent` |
| `VirtualizedLists should never be nested…` warning | Same as above | Same fix — the warning means virtualization is off and memory is unbounded |
| Absolutely positioned child invisible on Android only | Child drawn outside the parent's bounds | Move it inside the bounds, or give the parent padding/size |
| Shadow invisible on Android | `shadow*` only, or `overflow: 'hidden'` on the shadowed view | Use the shadow token (`elevation` too); shadow on the outer wrapper |
| Bottom button hidden behind the keyboard | No inset handling | §5 |
| Bottom content under the home indicator / nav bar | No safe-area padding | §5 |
| Layout jumps when an image loads | Remote image with no explicit size | Always size remote images |
| Row height changes per item with the same content | Emoji/icon font inside `<Text>` scaling with `fontScale` | Use a real icon component |

**Text is the usual culprit.** Any `Text` whose content comes from the API or
from i18n gets a deliberate truncation policy:

```tsx
<View style={{ flex: 1 }}>
  <Text numberOfLines={1} ellipsizeMode="tail" style={styles.title}>
    {user.fullName}
  </Text>
</View>
```

Never "fix" clipping by hardcoding a smaller font size, shrinking the design's
padding, or disabling font scaling. Fix the flex model.

## 4. The five canonical screen shapes

Pick one deliberately before writing the screen body. Almost every screen is one
of these.

**A. Short static content, may not fit on small phones**

```tsx
<ScreenContainer edges={['top']}>
  <AppHeader title={t('profile.title')} />
  <ScrollView contentContainerStyle={styles.content}>
    {/* sections */}
  </ScrollView>
</ScreenContainer>
```

**B. Content that must fill the screen but still scroll when it can't**

```tsx
<ScrollView
  style={{ flex: 1 }}
  contentContainerStyle={{ flexGrow: 1, justifyContent: 'space-between', padding: theme.spacing.md }}
>
  <HeaderSection />
  <FooterActions />
</ScrollView>
```

`flexGrow: 1` on the **content container** is the whole trick — `flex: 1` there
breaks scrolling instead of enabling it.

**C. A list (the default for any repeating row)**

```tsx
<FlashList
  data={items}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <OfferCard offer={item} />}
  ItemSeparatorComponent={Separator}
  contentContainerStyle={{ padding: theme.spacing.md }}
  ListEmptyComponent={<EmptyState … />}
  refreshControl={<RefreshControl refreshing={isRefetching} onRefresh={refetch} />}
  onEndReached={fetchNextPage}
  onEndReachedThreshold={0.5}
/>
```

**D. Header + list scrolling as one surface → one list, never nested scrollers**

```tsx
<FlashList
  data={items}
  ListHeaderComponent={<ProfileHeader />}
  renderItem={({ item }) => <OfferCard offer={item} />}
  …
/>
```

A `ScrollView` wrapping a `FlatList` is always wrong. `ListHeaderComponent` is
the answer; for genuinely complex composed scroll surfaces, use one list with
section data or a Reanimated collapsible header — still one scroller.

**E. Form / flow with a pinned bottom action**

```tsx
<KeyboardAvoidingView
  style={{ flex: 1 }}
  behavior={Platform.OS === 'ios' ? 'padding' : undefined}
  keyboardVerticalOffset={headerHeight}
>
  <ScrollView keyboardShouldPersistTaps="handled" contentContainerStyle={styles.form}>
    {/* fields */}
  </ScrollView>
  <View style={[styles.footer, { paddingBottom: insets.bottom + theme.spacing.md }]}>
    <AppButton title={t('common.next')} onPress={onNext} />
  </View>
</KeyboardAvoidingView>
```

Notes that matter: `keyboardShouldPersistTaps="handled"` or the first tap on a
button only dismisses the keyboard; the footer's bottom padding comes from the
safe-area inset, not a magic 34.

## 5. Safe areas, notches, and the keyboard

- Use `react-native-safe-area-context` (`useSafeAreaInsets` /
  `SafeAreaView` from that package), **not** React Native's own `SafeAreaView` —
  the built-in one is iOS-only and does nothing for Android's gesture bar or
  cutouts.
- Apply insets **once**, at the screen container, and pass `edges` explicitly.
  A screen under a navigation header does not need `top` again — double insets
  are the phantom-padding bug.
- A list's bottom inset belongs in `contentContainerStyle`'s padding, not in a
  wrapper `View` — otherwise the last row sits under the gesture bar while
  scrolling, or the scroll indicator floats.
- **Keyboard:** `KeyboardAvoidingView` with `behavior="padding"` on iOS and
  usually nothing (or `height`) on Android, where `windowSoftInputMode`
  handles it — verify what the project's `AndroidManifest.xml` sets. Test with a
  real keyboard, both platforms; this is the single most platform-divergent
  layout area.
- Dismiss the keyboard on background tap with the project's wrapper, not a bare
  `TouchableWithoutFeedback` around the whole screen (it swallows a11y).
- After an `await`, don't touch state on an unmounted screen — guard with the
  effect's cleanup or an `isMounted`/`AbortController` pattern.

## 6. Responsive & scalable — the four things that actually break

1. **Narrow phones (320–360 dp).** A row of three fixed-width cards will clip.
   Use `flex: 1` with `gap`, or `flexWrap`. Never hardcode a width the design
   measured on a 430 dp frame.
2. **System font scale.** `fontScale` can be 1.3–2.0 (much higher with iOS
   accessibility sizes). Any fixed-height box containing text must either grow
   or truncate. Test one screen at `fontScale` 1.3 before calling UI done.
   `allowFontScaling={false}` is a last resort for a few numeric badges — never
   a screen-wide fix.
3. **Long translations.** A label that is 6 characters in English can be 18 in
   German or Russian. Every static label gets the same truncation policy as API
   text.
4. **Orientation, tablets, and foldables.** `Dimensions.get('window')` read at
   module scope is frozen at startup and wrong after a rotation or an unfold.
   Read `useWindowDimensions()` inside the component instead.

Sizing guidance:

- `useWindowDimensions()` for **screen-level** decisions; `onLayout` for
  **component-level** ones (a card doesn't care about the screen, it cares about
  its slot).
- `width * 0.42` is a smell — it hardcodes a ratio the designer expressed as
  "two equal columns with a 12 gap". Model it as `flex: 1` + `gap`.
- Breakpoints only when the design actually has a tablet layout. Don't invent
  responsive behavior nobody asked for (YAGNI).
- `PixelRatio.getFontScale()` and `StyleSheet.hairlineWidth` exist for the two
  cases where device density genuinely matters. Nothing else needs manual
  density math.

## 7. Platform differences you must account for, not discover

| Area | iOS | Android |
| --- | --- | --- |
| Shadow | `shadowColor/Offset/Opacity/Radius` | `elevation` (also affects z-order) |
| Ripple / press feedback | opacity or highlight | `android_ripple`, needs `overflow: 'hidden'` to respect radius |
| Keyboard | `KeyboardAvoidingView behavior="padding"` | `windowSoftInputMode`, often no KAV |
| Back gesture/button | edge swipe | hardware/gesture back → handle it, or the user leaves the flow |
| Fonts | family + weight | weight maps to a shipped font file; missing weight silently falls back |
| Status bar | `barStyle` | `barStyle` + `backgroundColor` + translucency |
| Text vertical centering | tight | extra line padding; `includeFontPadding: false` when it matters |
| `overflow: 'visible'` | works | children clipped |
| Modals / sheets | native feel differs | back button must close them |

Use `Platform.select` for genuine divergence, in the theme or a shared component
— not scattered `Platform.OS === 'ios' ?` checks inside feature JSX.

## 8. Every screen has four states

A screen that only draws the happy path is half-drawn. Before the UI task is
done, all four exist and are reachable from the hook's state:

| State | What to draw |
| --- | --- |
| Loading | Skeletons shaped like the real content — not a centered spinner on a full page, unless the design says so |
| Empty | The design's empty view with the i18n message and, where one exists, a call to action |
| Error | Error view + a retry that re-runs the query; transient errors → a toast |
| Success | The real content |

```tsx
export function TripsScreen() {
  const { data, isPending, isError, refetch } = useTrips({ page: 1 });

  if (isPending) return <TripsSkeleton />;
  if (isError)   return <ErrorState onRetry={refetch} />;
  if (data.length === 0) return <TripsEmptyState />;
  return <TripsList trips={data} />;
}
```

Each of these is its own component file under `components/` — same rule as any
other extracted component. Note the ordering: an empty check before the error
check would hide a failure behind an "empty" screen.

## 9. Accessibility — the minimum bar, not an extra

- **Tap targets ≥ 44×44 pt (iOS) / 48×48 dp (Android).** A 20 pt icon needs
  padding or `hitSlop`. `hitSlop` grows the touch area without moving pixels.
- **Every control is announced:** `accessibilityRole` (`button`, `link`,
  `header`, `switch`, `image`) + `accessibilityLabel` when the visible text
  isn't enough, + `accessibilityState={{ disabled, selected, checked }}`.
- **Group what reads as one thing:** `accessible={true}` on a card so a screen
  reader announces it once, not five fragments.
- **Decorative images are hidden:** `accessibilityElementsHidden` (iOS) +
  `importantForAccessibility="no-hide-descendants"` (Android).
- **Don't encode meaning in color alone** — a red border needs error text too.
- **Respect reduce-motion** (`AccessibilityInfo.isReduceMotionEnabled`) for
  non-essential animation.
- **Contrast** comes from the token pair; if the design puts muted text on a
  colored surface, that's a question for the designer, not a value to eyeball.
- Verify with VoiceOver (iOS) and TalkBack (Android) on the screens you touched
  — not by reading the props back to yourself.

## 10. Debugging a layout instead of guessing

- **React DevTools / the element inspector** shows the computed layout box of
  the selected view. Use it before changing numbers at random.
- A temporary `borderWidth: 1, borderColor: 'red'` on the three suspect views
  answers "who is actually taking the space" in one reload. Remove it after.
- `onLayout={(e) => console.log(e.nativeEvent.layout)}` prints the real measured
  box — the fastest way to prove a view is 0-height.
- Reproduce at the failing configuration: small screen, `fontScale`, long string,
  keyboard open, the other platform — not on the one device where it looked fine.

## 11. Layout checklist (run before calling UI done)

- [ ] Nothing clipped at 320 dp width or at `fontScale` 1.3
- [ ] Every API/i18n `Text` in a row has a `flex: 1` container + `numberOfLines`
- [ ] Screen shape chosen from §4 — no nested same-axis scrollers, no
      `VirtualizedLists should never be nested` warning
- [ ] Lists use `FlashList`/`FlatList` with a stable `keyExtractor`, never `.map()`
- [ ] Safe-area insets applied once, with explicit `edges`; list bottom padding
      from the inset
- [ ] Keyboard verified on **both** platforms; `keyboardShouldPersistTaps` set
      where the form has buttons
- [ ] Android back gesture/button handled on modals and multi-step flows
- [ ] Loading / empty / error / success all drawn and reachable
- [ ] Tap targets ≥ 44 pt / 48 dp; roles and labels on every control
- [ ] Dark theme checked, not assumed
- [ ] Shadows verified on Android (elevation), not only on iOS
- [ ] No `Dimensions.get()` at module scope; no `width * 0.xx` standing in for a
      flex ratio
