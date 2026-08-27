# Best Practices — TypeScript, React & React Native Idioms

Read this for language-level and idiom-level guidance: typing, naming, hooks
discipline, immutability, lints, and the day-to-day "prefer/avoid" of writing
good React Native.

## 1. Enable strict typing and lints — let the machine enforce the boring rules

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,      // arr[i] is T | undefined — it really is
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

```jsonc
// eslint — the rules that catch real defects
{
  "extends": ["@react-native", "plugin:@typescript-eslint/recommended-type-checked"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "error",       // error, not warn
    "@typescript-eslint/no-floating-promises": "error",
    "@typescript-eslint/no-misused-promises": "error",
    "@typescript-eslint/no-explicit-any": "error",
    "react-native/no-inline-styles": "error",
    "react-native/no-raw-text": "error",
    "no-restricted-imports": ["error", { /* layer boundaries — architect.md §7 */ }]
  }
}
```

A clean `tsc --noEmit` and a clean lint run are part of "done". Don't
`// eslint-disable` or `@ts-expect-error` without a one-line reason on the same
line.

## 2. Naming

- **Types, interfaces, components:** `PascalCase` — `TripRepository`, `TripCard`.
- **Variables, functions, hooks:** `camelCase`, hooks always `useX`.
- **Files:** follow the repo (`kebab-case.ts` is the common default; component
  files may be `PascalCase.tsx` — pick one and hold it).
- **Constants:** `SCREAMING_SNAKE_CASE` for true module constants
  (`MAX_RETRIES`), `camelCase` for token objects (`tripKeys`).
- Name booleans as assertions: `isEnabled`, `hasError`, `canSubmit`.
- Name things by *what they mean*, not their type: `elapsedMs`, not `numberVal`.
- Handlers: `onX` for the prop, `handleX` for the implementation.
- Don't abbreviate unless the abbreviation is more common than the full word
  (`id`, `url`, `db` OK; `usrRepo` not OK).

## 3. Types — make illegal states unrepresentable

- **No `any`.** Use `unknown` at boundaries and narrow it. `as` is an assertion,
  not a check — the only safe cast is one you just proved.
- **Discriminated unions over optional-field soup.** A type with
  `isLoading`, `error`, and `data` all optional permits "loading AND error";
  a union does not:

```ts
type ScreenState<T> =
  | { status: 'loading' }
  | { status: 'error'; error: Failure }
  | { status: 'empty' }
  | { status: 'ready'; data: T };
```

- **Exhaustive switches.** A `default: assertNever(state)` turns a new variant
  into a compile error instead of a blank screen.
- **`type` for data shapes, `interface` for contracts you implement.** Consistency
  matters more than the choice.
- **Infer types from schemas** (`z.infer`) rather than hand-writing a type next
  to a schema that will drift from it.
- **Branded types** for domain identifiers (`architect.md` §4) so `TripId` and
  `UserId` can't be swapped.
- `satisfies` to check an object against a type without widening it.
- Type the props explicitly; don't rely on `React.FC` (it adds implicit
  `children` you may not want).

## 4. Immutability by default

- `const` unless reassignment is genuinely needed. `readonly` on props and
  entity fields; `readonly T[]` for arrays you don't own.
- Never mutate props, state, or a store's state object — always produce a new
  value. (Immer, via RTK or Zustand's middleware, lets you write mutable-looking
  code that produces a new value; that is not an exception to the rule.)
- Don't mutate an array you were handed (`sort`, `reverse`, `splice` mutate) —
  copy first: `[...items].sort(…)`.
- `as const` for literal tuples and lookup tables so they get precise types.

## 5. Hooks discipline

- **Rules of hooks are not stylistic**: top level only, same order every render.
- **`useEffect` is for synchronizing with something outside React** — a
  subscription, a native module, a timer, an imperative API. It is **not** for:
  transforming props into state, reacting to a user event (do it in the handler),
  or fetching (the query layer owns that).
- **The dependency array must be honest.** Silencing `exhaustive-deps` hides a
  stale closure that will bite months later. If the array is awkward, the effect
  is doing too much — split it, or move the value into a ref deliberately.
- **Every effect that subscribes returns a cleanup.** Always.
- **`useRef` for values that must persist without re-rendering** (timers,
  animation values, "did I already do this"), not as a state loophole.
- **Custom hooks are the unit of logic reuse.** A hook that returns a tuple of
  ten things is a god hook — split by concern.
- `useFocusEffect` (React Navigation) for anything that must run when the screen
  becomes visible; a stacked screen stays mounted.

## 6. Async idioms

- `async`/`await` over `.then()` chains.
- Never leave a floating promise — `await` it, `void` it deliberately, or attach
  a `.catch`. An unhandled rejection in React Native is a silent failure.
- Cancel in-flight work on unmount (`AbortController` / the query layer).
- Don't `setState` after unmount; the effect's cleanup is the fix, not an
  `isMounted` flag (which just hides the leak).
- `Promise.all` for independent calls; `Promise.allSettled` when partial success
  is acceptable — and then actually handle the rejected entries.

## 7. React Native UI idioms

- Function components only. Local state via `useState`; derived values computed
  during render.
- Extract components, not `renderX()` helpers (see `refactor.md` §8).
- Theme tokens instead of hardcoded colors/sizes; `StyleSheet.create` instead of
  inline objects.
- Localize user-facing strings from day one — retrofitting i18n is painful, and a
  hardcoded string is a defect, not a TODO.
- Any `Text` fed by the API or i18n gets an explicit truncation policy
  (`numberOfLines` + a `flex: 1` container) — see `ui-layout.md` §3.
- Respect the system font scale: never lock a text box to a fixed height that
  can't grow, and don't disable font scaling app-wide to "fix" a layout.
- Interactive elements carry `accessibilityRole` and a label; decorative images
  are hidden from the accessibility tree. Minimum tap target 44 pt / 48 dp.
- `Platform.select` for genuine platform divergence, centralized in the theme or
  a shared component — not `Platform.OS === 'ios' ?` scattered through feature
  JSX.
- Keys from stable ids, never the array index.

## 8. Prefer / Avoid quick reference

**Prefer:** `strict` TS · discriminated unions · `unknown` + narrowing · branded
ids · `readonly` · `const` · function components · derived state · selector-scoped
store reads · `Pressable` · `FlashList` · `StyleSheet.create` · typed navigation
params · zod at the boundary · early returns.

**Avoid:** `any` · `as` casts to silence the compiler · `@ts-ignore` ·
`useEffect` for derivation or fetching · dishonest dependency arrays · inline
styles · index keys · `.map()` in a `ScrollView` · module-scope
`Dimensions.get()` · global mutable singletons · `console.log` in shipped code ·
swallowing errors (`catch {}`) · class components in new code.

## 9. Documentation

- TSDoc (`/** … */`) on public APIs where the intent isn't obvious from the
  name. Explain *why*, not what the code literally does.
- Types are documentation — a precise type removes the need for half the
  comments people write.
- Keep comments truthful; a stale comment is worse than none.
- A short `README` per non-trivial feature/package (what it does, how to run its
  tests) pays off in onboarding.
