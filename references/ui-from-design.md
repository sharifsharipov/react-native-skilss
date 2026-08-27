# Building UI from a design / screenshot

Read this **before writing the first component** whenever the input is a
screenshot, a Figma frame, or a described layout. It is the drawing procedure +
the token contract + the shape recipes + the reusable-component inventory.

Pairs with two files:

- **`ui-layout.md`** — flexbox, overflow, screen shapes, safe area, keyboard,
  responsive, screen states, accessibility. *How the layout survives real devices.*
- **`refactor.md` §8** — how to split what you build into files.

---

## Calibrate to the project — do this once, before anything else

**The API names in this file are illustrative placeholders, not a specification.**
Your project names the same concepts differently. The *rules* are universal —
"every color comes from a token, a missing token is a question" holds everywhere.
The *identifiers* are not.

So before drawing anything in a repo you haven't calibrated to, spend two minutes
finding the project's real equivalents:

| Concept | Find it by | Example in this file |
| --- | --- | --- |
| Styling approach | look at any existing component | `StyleSheet.create` + `useTheme()` |
| Color set | the theme folder / provider | `theme.colors.<name>` |
| Text styles | the same, for typography | `theme.typography.<role>` |
| Spacing / radius scale | grep an existing component for `padding`/`borderRadius` | `theme.spacing.md`, `theme.radius.lg` |
| Icons | grep for `react-native-svg` / an icon component | `<Icon name="chevron-right" />` |
| Images | local assets module vs `expo-image` / `FastImage` | `<AppImage source={…} />` |
| Shared component folder | where `Button`-style components live | `src/core/ui/` |
| Tap primitive | grep for the project's wrapper around `Pressable` | `<AppPressable>` |
| Localized strings | the i18n accessor | `t('trips.title')` |
| Loading / empty / error views | grep for skeleton/empty-state components | `<Skeleton />`, `<EmptyState />` |
| Safe area handling | `react-native-safe-area-context` usage | `useSafeAreaInsets()` |

Record what you find, then **read the rest of this file substituting your
project's names**. If the project genuinely has no theme layer, no spacing scale,
or no tap primitive, say so and ask whether to introduce one — do not silently
fall back to raw hex values and `padding: 16`, and do not invent a theme file
unasked.

`scaffold.md` §0 does the same thing for the data/domain layers; this is its UI
counterpart. Once calibrated, everything below applies as written.

> Maintaining a fork? Replace the example column with your own names, and this
> becomes a precise, project-specific contract instead of a general one.

---

## 0. Read the design before you draw it

Drawing UI is a **five-step pipeline**. Skipping steps 0–2 is why generated UI
looks approximately right and is structurally wrong.

### 0.1 Decompose — top-down, out loud

Before any code, write the component tree as a short outline in the response:

```
ProfileScreen (ScreenContainer)
├── AppHeader(title, right: notification icon)
└── ScrollView                              ← screen shape B (ui-layout §4)
    ├── ProfileHeader        avatar 64 + name + phone, row
    ├── gap 24
    ├── ProfileStatsRow      3 × flex:1 StatTile
    ├── gap 24
    └── ProfileMenuList      6 × ProfileMenuItem (icon + label + chevron)
```

Rules for the outline:
- Name every section the way the component will be named.
- Mark which sections repeat → those become one component with props.
- Mark which sections already exist in the shared UI folder (§4) → reuse, don't
  rebuild.
- Pick the screen shape from `ui-layout.md` §4 **here**, not after the layout
  breaks.

### 0.2 Measure, don't eyeball

From the screenshot, read off and write down:

| Read | Then |
| --- | --- |
| Outer screen padding | match to a spacing token |
| Vertical rhythm between sections | match to the spacing scale (designs are on a 4/8 grid — 13 px means you misread 12) |
| Corner radii | match to the radius scale |
| Font size + weight per text role | match to a typography token |
| Fill / border / text colors by **role** | match to a color token |
| Icon sizes | usually 16/20/24 — confirm against existing usage |
| Which elements stretch vs stay fixed | drives `flex: 1` vs a fixed `width` |

Match by **role**, never by hex. "Muted secondary label" → the muted token, even
if the token is two shades off the screenshot. A pixel-perfect literal that breaks
dark mode is worse than a one-shade-off token that doesn't.

Units: React Native has no `px`. Numbers are **density-independent points**, and
`1` is not a hairline on every device — use `StyleSheet.hairlineWidth` for
1-physical-pixel borders when the design shows one.

### 0.3 If the input is a Figma link, read it — don't guess from the PNG

The Figma MCP tools give exact values instead of eyeballed ones. When a
`figma.com` URL is provided:

1. `get_screenshot` — see the frame.
2. `get_variable_defs` — the **real** tokens (colors, spacing, radii, type). Map
   these to the project's tokens; a Figma variable name is not a theme key.
3. `get_design_context` / `get_metadata` — structure, auto-layout direction and
   constraints. Auto-layout maps almost 1:1 onto flexbox: direction →
   `flexDirection`, spacing → `gap`, "fill container" → `flex: 1`, "hug
   contents" → no flex, alignment → `justifyContent`/`alignItems`.
4. `download_assets` — export icons/images rather than recreating them as views.

If the Figma connector isn't authorized in this session, say so and fall back to
the screenshot procedure (§0.1–0.2) — don't silently invent values.

### 0.4 Batch every unknown into ONE question

Collect all missing tokens, ambiguous behaviors, and unexported assets from the
whole screen, then ask once, then build. Five interruptions mid-build is the
failure mode.

Also ask (once) about behavior the image can't show: what does tapping this do,
what shows while loading, what shows when the list is empty, is this text from the
API or i18n, is there a dark-theme frame, does this screen exist on both
platforms.

### 0.5 Then build → then verify

Build in the file structure of `refactor.md` §8, then run the verification loop
in §7 of this file. UI is not done when it compiles, and it is not done when it
looks right on one simulator.

---

## 1. The token contract — never invent a value

The rule is absolute: **every visual value resolves to a token, or it is a
question for the user.** The names below are the calibration example — substitute
the ones you found above.

| You need | You use *(example names — see Calibrate)* | Source |
| --- | --- | --- |
| Text style | `theme.typography.bodyMedium` | `core/theme/typography.ts` |
| Color | `theme.colors.textSecondary` | `core/theme/colors.ts` |
| Spacing / gap / padding | `theme.spacing.md` | `core/theme/spacing.ts` |
| Corner radius | `theme.radius.lg` | `core/theme/radius.ts` |
| Elevation / shadow | `theme.shadows.card` | `core/theme/shadows.ts` |
| Vector icon | `<Icon name="chevron-right" size={24} color={…} />` | `core/ui/icon.tsx` |
| Local raster | the typed assets module | `assets/index.ts` |
| Remote image | `<AppImage source={{ uri }} />` | `core/ui/app-image.tsx` |
| User-facing string | `t('feature.key')` | never a literal in JSX |

**Banned on sight while drawing UI** — these hold in every project, whatever the
theme layer is called:

- a raw hex (`'#F5F5F5'`) or a named CSS color (`'gray'`) in a stylesheet
- an inline `fontSize`/`fontWeight` pair instead of a typography token
- a bare number for padding/margin/gap instead of the spacing scale
- a bare number for `borderRadius` instead of the radius scale
- an asset path string instead of the typed asset reference
- a hardcoded user-facing string instead of an i18n key
- `Dimensions.get('window').width * 0.42` where the design means "two equal
  columns" (see `ui-layout.md` §6)
- `style={{ … }}` inline object literals in a re-rendering component or a list
  item — new object identity every render (`performance.md` §1)
- `shadowColor`/`elevation` written by hand per component instead of a shadow
  token — the two platforms need different properties and only the token knows
  both

### 1.1 When a token is missing — STOP and ask

Do not improvise. Ask the user, in one short question, and wait.

- **Missing text style** → ask for the exact size/weight/color/line-height/
  letter-spacing. Then add it to the typography scale for **both** themes. A
  style that exists in only one theme is a bug. Name it semantically
  (`bodySubheadline`), not by value (`text15Semibold`).
- **Missing color** → ask for the hex **and** the dark-theme counterpart. Never
  invent a hex, never fake it with `opacity`, never add a token unasked.
- **Missing shadow** → ask. A light-mode shadow on a dark surface looks like
  dirt, and iOS `shadow*` + Android `elevation` are different mechanisms.
- **Missing icon/asset** → ask whether it comes from Figma (they export it, as
  SVG where possible) or from the API (then it is a remote image, not an asset).
- **Missing i18n key** → add it to **every** locale catalog the project ships. A
  key present in one language is a visible `trips.title` string in another.
- **Missing font weight/family** → ask. Android maps `fontWeight` to real font
  files; a weight the family doesn't ship silently renders as regular.

One batched question at the start beats five interruptions mid-build (§0.4).

---

## 2. Recipe A — a pressable shape

Any tappable card, chip, icon button, or row. The project's tap primitive already
provides the press feedback, the disabled state, and the haptic — do not
hand-roll `TouchableOpacity` per component and do not use
`TouchableWithoutFeedback` on a real control (no feedback = not a control).

```tsx
export function ActionChip({ label, icon, onPress, disabled }: ActionChipProps) {
  const theme = useTheme();
  const styles = useStyles();

  return (
    <AppPressable
      onPress={onPress}
      disabled={disabled}
      accessibilityRole="button"
      accessibilityLabel={label}
      style={styles.container}
      hitSlop={8}
    >
      <Icon name={icon} size={20} color={theme.colors.iconPrimary} />
      <Text style={styles.label} numberOfLines={1}>
        {label}
      </Text>
    </AppPressable>
  );
}

const useStyles = makeStyles((theme) => ({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: theme.spacing.sm,
    minHeight: 44,                          // tap target floor, ui-layout §9
    paddingHorizontal: theme.spacing.md,
    borderRadius: theme.radius.lg,
    backgroundColor: theme.colors.surface,
    borderWidth: StyleSheet.hairlineWidth,
    borderColor: theme.colors.border,
  },
  label: {
    ...theme.typography.bodyMedium,
    color: theme.colors.textPrimary,
    flexShrink: 1,                          // long label ellipsises, not overflows
  },
}));
```

Rules for this recipe:

- **Android ripple must be clipped by the radius.** If the primitive uses
  `android_ripple`, the container needs `overflow: 'hidden'` (or the ripple
  bleeds past the corners) — the same defect as a mismatched ink radius.
- Padding lives **inside** the pressable, so the press surface covers the whole
  shape, not just the content box.
- Total tap area ≥ **44×44 pt (iOS) / 48×48 dp (Android)**; use `hitSlop` when
  the visual is smaller than the target.
- `disabled` is a real state — pair it with the design's disabled colors, don't
  just stop calling `onPress`.
- Every pressable carries `accessibilityRole="button"` and a label
  (`ui-layout.md` §9).
- Styles come from `StyleSheet.create` (here via a themed `makeStyles` helper),
  never an inline object.

## 3. Recipe B — a static shape

Same skeleton, no press layer:

```tsx
<View style={styles.badge}>
  <Icon name="tag" size={16} color={theme.colors.iconMuted} />
  <Text style={styles.badgeLabel}>{title}</Text>
</View>
```

```ts
badge: {
  flexDirection: 'row',
  alignItems: 'center',
  gap: theme.spacing.xs,
  paddingVertical: theme.spacing.xs,
  paddingHorizontal: theme.spacing.sm,
  borderRadius: theme.radius.full,
  backgroundColor: theme.colors.surfaceMuted,
},
```

A `View` with `flexDirection: 'row'` is the default container. Don't wrap a
single child in a `View` just to give it padding — put the padding on the child's
own style when it accepts one.

## 3.1 Recipe C — a list row (icon + text block + trailing)

The single most repeated shape in an app. Note `flex: 1` on the text block:
without it, a long name pushes the chevron off screen the moment the API returns
a real value.

```tsx
export function MenuItem({ icon, title, subtitle, onPress }: MenuItemProps) {
  const styles = useStyles();
  return (
    <AppPressable onPress={onPress} style={styles.row} accessibilityRole="button">
      <Icon name={icon} size={24} />
      <View style={styles.textBlock}>
        <Text style={styles.title} numberOfLines={1}>{title}</Text>
        {subtitle ? (
          <Text style={styles.subtitle} numberOfLines={2}>{subtitle}</Text>
        ) : null}
      </View>
      <Icon name="chevron-right" size={20} />
    </AppPressable>
  );
}
```

```ts
row:       { flexDirection: 'row', alignItems: 'center', gap: theme.spacing.sm,
             paddingVertical: theme.spacing.sm, minHeight: 48 },
textBlock: { flex: 1, gap: theme.spacing.xxs },            // ← the whole point
title:     { ...theme.typography.bodyMedium, color: theme.colors.textPrimary },
subtitle:  { ...theme.typography.caption, color: theme.colors.textSecondary },
```

Spreading a typography token and overriding **color only** is the acceptable
edit. Overriding its `fontSize` means you picked the wrong token — or you need a
new one (§1.1).

## 3.2 Recipe D — an icon

```tsx
<Icon name="filter" size={24} color={theme.colors.iconPrimary} />
```

- Tint through the color prop + a token — never ship two color variants of the
  same icon file, and never leave a themed icon untinted (it will be invisible in
  one of the two themes).
- Always give an explicit `size`; an unsized SVG takes its intrinsic viewBox size
  and silently wrecks a row.
- Multi-color/brand illustrations are the exception: no tint.
- Icons are **not** text. An emoji or an icon font glyph inside `<Text>` scales
  with `fontScale` and shifts the row height unpredictably.

## 3.3 Recipe E — an image

```tsx
<AppImage
  source={{ uri: user.avatarUrl }}
  style={styles.avatar}                     // explicit width + height, always
  contentFit="cover"
  placeholder={blurhash}
  transition={200}
  accessibilityIgnoresInvertColors
/>
```

- A remote image **always** has an explicit size and a placeholder. Without a
  size it renders 0×0 until it loads and the layout jumps.
- Use the project's image component (`expo-image` / `FastImage`) — it brings disk
  caching, progressive loading, and a proper failure state. Bare `<Image>` with a
  `uri` has no memory/disk cache policy worth the name.
- Decorative images get `accessibilityElementsHidden` / `importantForAccessibility="no"`;
  meaningful ones get an `accessibilityLabel`.

## 3.4 Recipe F — a shadow

```ts
card: {
  backgroundColor: theme.colors.surface,
  borderRadius: theme.radius.lg,
  ...theme.shadows.card,        // iOS shadowColor/Offset/Opacity/Radius + Android elevation
},
```

Shadows are the most common cross-platform break in generated UI: `shadow*`
properties do nothing on Android, `elevation` does nothing on iOS, and
`elevation` also changes the view's stacking order. Both live inside the token.
If there is no shadow token, that is a §1.1 question.

Also: a shadow on a view with `overflow: 'hidden'` is clipped away on Android.
Put the shadow on an outer wrapper and the clipping on the inner one.

---

## 4. Reuse before you build — the shared-component inventory

Check the project's shared UI folder **before** building anything. A second
header / button / text input is a defect, not a preference.

The inventory below is the calibration example (`src/core/ui/` is the
illustrative folder name used here). In a fresh repo, spend five minutes listing
your own equivalents once and paste them here — a fork with an accurate inventory
is worth far more than a generic instruction to "check for existing components",
because the assistant cannot reuse what it doesn't know exists.

### 4.1 The ones you will reach for constantly

| Component | File | Key props |
| --- | --- | --- |
| `ScreenContainer` | `screen-container.tsx` | `edges`, `scroll`, `padded`, `backgroundColor` |
| `AppHeader` | `app-header.tsx` | `title`, `left`, `right`, `showBack` (def. true), `centerTitle` |
| `AppButton` | `app-button.tsx` | `title`, `onPress`, `variant` (primary/secondary/ghost/danger), `size`, `loading`, `disabled`, `leftIcon`/`rightIcon`, `fullWidth` |
| `AppTextInput` | `app-text-input.tsx` | `label`, `placeholder`, `value`, `onChangeText`, `error`, `helperText`, `leftIcon`/`rightIcon`, `secureTextEntry`, `keyboardType`, `returnKeyType`, `onSubmitEditing` |
| `AppPressable` | `app-pressable.tsx` | `onPress`, `disabled`, `hitSlop`, `haptic`, `style`, `pressedOpacity` |

### 4.2 Everything else already in the shared folder

| Need | Component / helper |
| --- | --- |
| Icon | `Icon(name, size, color)` |
| Local/remote image | `AppImage(source, style, contentFit, placeholder)` |
| Avatar | `Avatar(uri, size, fallbackInitials)` |
| Badge / chip / tag | `Badge(label, tone)` |
| Card surface | `Card(children, padded, onPress?)` |
| Divider | `Divider(inset?)` |
| Bottom sheet | `useBottomSheet()` / `<AppBottomSheet>` |
| Modal / dialog | `showAlert({ title, message, actions })` |
| Toast / snackbar | `toast.success(...)`, `toast.error(...)` |
| Full-screen blocking loader | `<LoadingOverlay visible />` |
| List footer loader | `<ListFooterLoader loading />` |
| Skeleton / shimmer | `<Skeleton width height radius />` |
| Empty state | `<EmptyState icon title description action />` |
| Error state | `<ErrorState message onRetry />` |
| Pull to refresh | the list's `refreshControl` prop + `useRefresh()` |
| Keyboard-aware screen | `<KeyboardAvoidingScreen>` |
| Dismiss keyboard on tap | `<DismissKeyboard>` |
| Segmented control / tabs | `<SegmentedControl segments value onChange>` |
| Checkbox / switch row | `<SettingRow label value onChange>` |
| OTP input | `<OtpInput length value onChange>` |
| Rating | `<Rating value onChange size>` |

If the design needs something close to one of these but not identical, **extend
the existing component with a new optional prop** — do not fork a near-copy.

### 4.3 Drawing the four screen states

Never leave a screen with only its success state. Full rule + the render
skeleton: `ui-layout.md` §8.

| State | Reach for |
| --- | --- |
| Loading | `<Skeleton />` shaped like the real content — not a centered spinner on a full page, unless the design says so |
| Empty | `<EmptyState />` + i18n text (and a call to action where one exists) |
| Error | `<ErrorState onRetry={refetch} />`; transient → `toast.error(...)` |
| Inline/blocking action | `<AppButton loading />` or `<LoadingOverlay />` |

---

## 5. Structure rules while drawing

These are enforced, not advisory (full set: `refactor.md` §8):

1. **No `renderX()` functions returning JSX.** A render function is not a
   component: it has no identity, no memo boundary, no hooks of its own, and it
   re-runs with the whole parent. Extract a component.
2. **Every extracted component is exported and lives in its own file** under the
   feature's `components/` folder (or `core/ui/` if it is generic).
3. **File length: 300 lines is a hard cap.** Split well before that — as soon as
   a component has more than one visual section (~150 lines is the usual trigger).
4. **Styles via `StyleSheet.create`**, defined once at module level or through
   the themed `makeStyles` helper — never an inline object literal in JSX.
5. `React.memo` on list items and on components that re-render under a
   frequently-changing parent; not sprayed everywhere (`performance.md` §1).
6. The component body reads like a table of contents of child components.
7. Naming says what it shows: `OfferPriceCard`, not `ItemComponent`.
8. Components take **props**, not stores. A leaf may dispatch a store action, but
   it should not need a store or a query to know what to draw — pass the values
   in. That is what makes it renderable in a test and in Storybook.

---

## 6. Performance while drawing (the parts that apply to UI, not to lists)

- Repeating rows come from `FlashList`/`FlatList`, never `.map()` inside a
  `ScrollView` (`ui-layout.md` §4C).
- Scope re-renders: read stores with selectors, and keep frequently-changing
  state (a text input's value, an animation) in the smallest component that
  needs it.
- `useCallback` for handlers passed into memoized children or list items;
  `useMemo` for genuinely expensive derivations — not for every literal.
- Animations run on the UI thread: Reanimated worklets or
  `useNativeDriver: true`. A JS-driven animation drops frames the moment a
  fetch resolves.
- Remote images get an explicit size and the project's caching image component;
  a 4000 px photo decoded into a 40 px avatar is a memory defect.
- Effects that subscribe (listeners, timers, `AppState`, `NetInfo`,
  `Keyboard`) return a cleanup function. Always.

Deeper: `performance.md`.

---

## 7. Verify against the design — the loop that makes UI actually match

Compiling is not matching. After the build:

1. `tsc --noEmit` and the linter — zero new issues.
2. Run the app, navigate to the screen, screenshot it:
   ```bash
   xcrun simctl io booted screenshot shot-ios.png     # iOS simulator
   adb exec-out screencap -p > shot-android.png       # Android
   ```
3. **Put your screenshot next to the design and compare in this order:**
   overall structure → section spacing → alignment → radii → typography →
   colors → icon sizes. Structure errors first; nobody cares about a 2 pt radius
   on a screen whose sections are in the wrong order.
4. Write the delta list, fix, fast-refresh, screenshot again. Repeat until the
   list is empty or the remaining items are questions for the user.
5. Re-check **on the other platform**, in the **other theme**, at **`fontScale`
   1.3**, and on a **narrow screen** — see `ui-layout.md` §6. Shadows, fonts,
   keyboard behavior, and safe areas differ by platform; an iOS-only check is
   half a check.
6. Poke the four states (§4.3): loading, empty, error, success.

Then the `manual-test.md` protocol closes it out with a `Manual Test Result`
block — or, if no device is available, the honest
"code-complete, NOT device-tested" line plus the exact steps for the user.

---

## 8. Finish gates for a UI task

1. Every color / text style / radius / spacing / asset / string traced to a token
   or an i18n key — grep your own diff for `#`, `fontSize`, `padding:` with a
   bare number, `borderRadius:` with a bare number, and quoted user-facing
   strings in JSX.
2. Every tappable surface goes through the tap primitive (or a shared button) —
   press feedback + accessibility role — and is ≥ 44 pt / 48 dp.
3. Nothing duplicates a component that already exists in the shared UI folder.
4. No file over 300 lines; no JSX-returning `renderX` helpers; no inline style
   objects.
5. Layout checklist in `ui-layout.md` §11 passes — nothing clipped at 320 dp or
   `fontScale` 1.3, long strings truncated, correct screen shape.
6. All four screen states drawn.
7. **iOS and Android** both checked on a device/simulator, light **and** dark.
8. `tsc --noEmit` + lint clean.
9. New tokens → added for both themes; new strings → every locale catalog; new
   assets → committed and referenced through the typed module.
10. Design-comparison loop (§7) done, then `manual-test.md` — evidence, or an
    explicit "NOT device-tested" with steps.
