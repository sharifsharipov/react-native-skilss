# Refactoring — KISS, DRY, YAGNI, Extraction

Read this when cleaning up existing code, breaking down a large component/module,
or removing duplication. Refactoring changes structure, **not behavior** — so it
is only safe with tests (see `testing.md`). Refactor in small, verifiable steps.

§1–7 are general refactoring. **§8 is the component-composition rulebook** — read
it whenever you split a screen, fill a feature's `components/` folder, or review
UI code. For creating a feature from scratch, see `scaffold.md`. For building UI
from a screenshot/Figma (drawing pipeline, tokens, shape recipes, reusable-
component inventory), read `ui-from-design.md` first, and `ui-layout.md` for
flexbox, clipping, screen shapes, screen states, and accessibility — §8 only
covers how to structure what you build.

## 1. The three principles that drive most refactors

- **KISS — Keep It Simple.** The simplest solution that fully solves the problem
  wins. Clever one-liners that need a comment to explain are usually a net loss.
  Readability over cleverness (the Golden Rule).
- **DRY — Don't Repeat Yourself.** Duplicated *knowledge* is the enemy, not
  duplicated *lines*. Extract when the same rule appears in multiple places, so a
  change happens once. But don't over-abstract two things that merely look
  similar today (AHA: "Avoid Hasty Abstractions"). Two components that share a
  shape but not a reason to change should stay two components.
- **YAGNI — You Aren't Gonna Need It.** Don't build for imagined future
  requirements. Delete speculative props, unused variants, and "just in case"
  callbacks. The best code is the code you didn't write.

## 2. Component extraction — break down huge screens

A component whose JSX you must scroll to read, or that nests more than ~4 levels
of layout `View`s, is a refactor target. Extract cohesive subtrees into their own
**components** (not helper functions returning JSX).

```tsx
// BEFORE — one 250-line screen
export function TripsScreen() {
  return (
    <View style={styles.root}>
      {/* header 40 lines */}
      {/* list 120 lines */}
      {/* footer 60 lines */}
    </View>
  );
}

// AFTER — cohesive, memoizable, independently testable pieces
// each one an EXPORTED component in its own file under components/
export function TripsScreen() {
  return (
    <ScreenContainer>
      <TripsHeader />
      <TripsList />
      <TripsFooter />
    </ScreenContainer>
  );
}
```

**Prefer extracting a component over a `renderX()` helper.** A component gets its
own identity in the tree, its own memo boundary, and its own hooks; it can be
tested and previewed alone. A render helper re-runs with the parent every time,
cannot be memoized, and cannot own state.

Extracted components are **exported, one per file** — not a stack of five
components in the screen file. Full rules: §8.

## 3. God components, god hooks & long functions

Symptoms: a component with 8+ hooks and mixed concerns, a function over ~40
lines, a `useEffect` doing four unrelated things, or a hook named
`useScreenLogic` doing everything.

Refactor moves:
- **Extract a hook** — pull a cohesive cluster of state + effects into
  `useTripFilters()`, `usePagination()`. Named hooks are the React equivalent of
  Extract Class.
- **Split an effect per concern.** One `useEffect` per subscription/sync reason,
  each with its own cleanup and its own honest dependency array.
- **Move business logic out of the UI** — a use case or a hook owns it; the
  component renders state and calls handlers.
- **Replace a growing conditional with a lookup** — a `switch` on a `type` that
  keeps growing becomes a `Record<Kind, Component>` map (Open/Closed).
- **Delete derived state.** `const [total, setTotal] = useState(0)` plus an
  effect that recomputes it is a bug waiting to happen; compute it during render.

## 4. Magic numbers & strings

Named constants make intent explicit and changes safe. Route names, query keys,
storage keys, durations, and sizes should never be scattered literals.

```ts
// BEFORE
if (status === 3) { … }
setTimeout(fn, 300);
navigation.navigate('TripDetails' as never);
storage.set('tkn', token);

// AFTER
if (trip.status === 'delivered') { … }
setTimeout(fn, DURATIONS.debounce);
navigation.navigate('TripDetails', { tripId });   // typed param list
secureStorage.setRefreshToken(token);             // typed accessor
```

Group constants meaningfully (`DURATIONS`, `STORAGE_KEYS`, `tripKeys`, theme
tokens) rather than one giant `constants.ts` bucket.

## 5. Common React Native smells & their fixes

| Smell | Fix |
| --- | --- |
| `renderItem` defined inline in JSX | Hoist to a `useCallback`, or extract a memoized item component |
| `style={{ … }}` inline object | `StyleSheet.create` / themed `makeStyles` |
| `useEffect` + `useState` fetching | The project's query layer (dedupe, cancel, retry, cache) |
| `useEffect` syncing derived state | Compute during render |
| `useState` that is never read, only written | Delete it or use a ref |
| Deeply nested `View` tree | Extract components; use `gap` instead of spacer views |
| Prop drilling 4 levels deep | Context for genuinely global concerns, composition (`children`) otherwise |
| `TouchableWithoutFeedback` on a control | `Pressable` / the project's tap primitive |
| `.map()` over a long array in a `ScrollView` | `FlashList`/`FlatList` |
| `console.log` in shipped code | A logger gated by `__DEV__` |
| `any` / `as unknown as X` | A real type, or a zod schema at the boundary |
| Business logic inside `onPress` | Call a use case / hook action |
| Global mutable singleton passing data | DI container + props/context |
| `catch (e) {}` | Map to a typed failure, or let it reach the boundary that handles it |
| A component reading a store to know what to draw | Pass props; keep the store read at the screen level |

## 6. Refactoring safely — the loop

1. Ensure there's a test covering the current behavior. If not, write a
   **characterization test** first (assert what it does now, even if ugly).
2. Make **one** small structural change.
3. Run `tsc --noEmit`, lint, and the tests. Green? Commit. Red? Revert or fix
   immediately.
4. Repeat. Never mix a refactor and a behavior change in the same commit — it
   makes review and bisecting impossible.

TypeScript makes step 2 cheaper than in most stacks: a rename or a signature
change is compiler-verified. Lean on that, but remember the compiler does not
check what the user sees — a pure UI refactor still needs a screenshot diff
(`ui-from-design.md` §7).

## 7. When NOT to refactor

- No tests and no time to add them for a risky area → add tests first, or leave
  it.
- "It's ugly but isolated and never changes" → low value; spend effort where
  churn is high.
- Refactoring to introduce an abstraction you *might* need → YAGNI; wait for the
  second real use case (Rule of Three) before abstracting.
- Adding `memo`/`useMemo`/`useCallback` everywhere "for performance" without a
  measurement → this is a cost (allocation + complexity), not a free win. See
  `performance.md` §1.

---

## 8. Component composition — the UI rulebook

Applies to every component you write or touch: filling a scaffold's empty
`components/` folder, splitting an existing screen, or reviewing UI in a PR. Goal
is readability and maintainability; on a pure refactor, **behavior and UI stay
pixel-for-pixel and logic-for-logic identical.**

### 8.1 Rules

1. **Never build UI with helper functions.** No `renderHeader()`. Extract a real
   component (`TripsHeader`). A JSX-returning function is never an acceptable
   substitute, no matter how small — it has no identity in the tree, so React
   remounts its state on every structural change, and it can never be memoized.
2. **Extracted components are exported and live in their own file.** No five
   components stacked in one screen file. One file per component, named after
   what it shows.
3. **Split large components. 300 lines per file is a hard cap.** Split well
   before that — several UI sections, multiple render helpers, or simply hard to
   read (~150 lines is the usual trigger) → split (`HomeScreen` →
   `HomeHeader`, `HomeBanner`, `HomeStats`, `HomeBody`).
4. **Extract repeated UI.** Any block appearing more than once becomes its own
   component (a repeated bordered box → `InfoCard`).
5. **Styles come from `StyleSheet.create`** (or the project's styling library),
   defined once per file at module level or via a themed `makeStyles` hook.
   Inline style objects allocate a new object every render and defeat
   `memo` on the child. The exception is a genuinely dynamic value, and then it
   is composed with an array: `style={[styles.bar, { width: progress }]}`.
6. **Function components only.** No class components in new code; no
   `getDerivedStateFromProps` thinking. Local state → `useState`; a value that
   must persist without re-rendering → `useRef`; expensive derivation →
   `useMemo` with an honest dependency array.
7. **One component = one responsibility.** A single component does not own
   header + content + stats + settings.
8. **Hooks are for logic, components for rendering.** A component body that
   contains 30 lines of computation before its `return` is really a hook plus a
   view — split them.
9. **The component body reads like a table of contents:**

   ```tsx
   export function HomeScreen() {
     return (
       <ScreenContainer>
         <HomeHeader />
         <HomeBody />
         <HomeBottomBar />
       </ScreenContainer>
     );
   }
   ```
10. **Naming describes responsibility** — `LoginButton`, `ProfileAvatar`,
    `UserInfoCard`, `PaymentSummary`. Never `Component1`, `ItemView`,
    `CustomView`, `MyContainer`.
11. **Design tokens, never hardcoded values.** Colors, radii, spacing, and text
    styles come from the project's theme layer — never a raw hex, never an inline
    `fontSize`/`fontWeight` pair. A literal in a component is a bug. See §8.5 for
    what to do when the token you need doesn't exist.
12. **Interactive elements get press feedback and a role.** Use the project's tap
    primitive or a shared button; `Pressable` with a pressed style if the project
    has none. A `View` with an `onTouchEnd`, or a `TouchableWithoutFeedback` on a
    real control, is a defect. On Android, ripple respects the corner radius only
    with `overflow: 'hidden'`.
13. **`memo` where it pays** — list items, and children of a component that
    re-renders often. Combined with stable props (`useCallback` handlers,
    hoisted styles); `memo` on a component receiving a fresh inline object every
    render does nothing but cost a comparison.
14. **`key` from stable data.** `key={item.id}`, never the array index for lists
    that can reorder, filter, or paginate — index keys corrupt input state and
    animations.
15. **Don't prop-drill more than ~2 levels.** Use composition (`children`,
    render props) or a context for genuinely global concerns. A context whose
    value changes every render is a re-render broadcast — memoize the value.
16. **Components take props, not stores.** A leaf may call a store action, but it
    should not need a store or a query to know what to draw. That is what makes
    it testable and previewable.
17. **Layout is not improvised.** Pick a screen shape and a flex model from
    `ui-layout.md` (§2–4); any `Text` fed by the API or i18n inside a row gets a
    `flex: 1` container + `numberOfLines`. Clipped content is a defect, and
    shrinking the design's paddings or font sizes is not the fix.
18. **A screen is not one state.** Loading, empty, error, and success are all
    drawn, each as its own component file (`ui-layout.md` §8).
19. **Every effect cleans up.** Subscriptions, timers, listeners, animations, and
    in-flight requests are torn down in the effect's return. A `setState` after
    unmount is the visible symptom of a missing cleanup, not the bug itself.

### 8.2 File structure

```
features/<feature>/presentation/
  screens/
    <name>-screen.tsx
  hooks/
    use-<name>.ts
  components/
    <name>-header.tsx
    <name>-list.tsx
    <name>-card.tsx
    <name>-skeleton.tsx
    <name>-empty-state.tsx
```

One file per extracted component, named after what it shows, not after its type.
Co-locate a component's styles and its `Props` type in the same file; export the
`Props` type when another module composes it.

### 8.3 Do NOT change during a refactor

Business logic · API behavior · state flow · navigation · UI appearance (pixels,
spacing, colors, text) · animations.

Only the structure improves. A refactor is invisible to the end user.

### 8.4 Per-file checklist

- [ ] No JSX-returning helper functions (`renderX`) left
- [ ] Each became a real component, exported, in its own file
- [ ] Repeated UI extracted into a shared component
- [ ] No file over 300 lines (hard cap); split already at ~150 / multiple sections
- [ ] `StyleSheet.create` / themed styles only — no inline style objects
- [ ] Theme tokens only — no hardcoded colors, spacing, radii, or type
- [ ] No hardcoded user-facing strings — i18n keys only
- [ ] Tappable surfaces have press feedback, a role, and a ≥44 pt / 48 dp target
- [ ] `memo` + stable props on list items; `key` from stable ids
- [ ] Every effect has a cleanup; no `setState` on an unmounted screen
- [ ] Props typed; no `any`, no unjustified assertions
- [ ] Names describe responsibility
- [ ] The component body is short and readable
- [ ] Behavior and UI preserved exactly
- [ ] Nothing clipped at 320 dp / `fontScale` 1.3; dynamic text truncated
- [ ] Loading / empty / error / success present (new UI, not pure refactors)
- [ ] `tsc --noEmit` + lint — zero new issues

### 8.5 Building UI from a screenshot or design

When the user sends a screenshot/mockup and asks for the UI, the design is the
**target**, not the **source of values**. Values come from the theme layer.

Summary below; the full contract — the decompose/measure/ask pipeline, the token
table, the shape recipes, asset rules, the shared component inventory, and the
design-comparison loop — is in **`ui-from-design.md`**, with layout/clipping/
responsive/a11y in **`ui-layout.md`**. Read those before drawing, not after.

1. **Open the theme files before writing the component.** Find the project's
   color set, typography scale, spacing/radius scales, and the hook that exposes
   them. Read the available names — you cannot pick the right token from memory.
2. **Text style → always a typography token.** Spread it and override **color
   only** when the design differs solely by color. An overridden `fontSize` means
   you picked the wrong token, or you need a new one.
3. **Color → the token if it exists.** Match by **role** (surface, border, muted
   text, danger), not by eyeballing the hex.
4. **Color missing from the token set → STOP and ask the user.** Do not invent a
   hex. Do not fake it with `opacity` on a nearby token. Do not add a token on
   your own initiative. Ask: "the design uses this swatch; there's no matching
   token — should I add one, and under what name, and what is its dark-theme
   value?" Then continue once answered.
5. **Never sample a color out of the screenshot and paste it as a literal.** That
   is exactly the bug the theme layer exists to prevent, and it silently breaks
   the other theme.
6. Spacing, radii, shadows, and icon sizes follow the same order: existing token
   → ask.
