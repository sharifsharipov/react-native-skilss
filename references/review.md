# Code Review — PR Review & AI Self-Review

Read this when reviewing a PR / auditing existing code, and use the self-review
protocol at the end of *every* code-producing response. This is where the Staff
Engineer mindset actually shows up: catch problems before they merge.

## 1. Review order — cheapest signal first

Don't start at line 1. Triage in this order so you spend attention where it
matters:

1. **Does it belong?** Right layer, right feature, right level of abstraction?
   A file in the wrong layer (see `architect.md`) is a bigger problem than a
   naming nit.
2. **Correctness & edge cases.** Loading/empty/error states, boundary values,
   race conditions on navigation and async, the unhappy path.
3. **Architecture & coupling.** SOLID, layer boundaries, dependency direction,
   testability.
4. **Performance.** Re-render scope, list virtualization, cleanup, images.
5. **Security.** Secrets, tokens, storage, transport, deep links
   (see `security.md`).
6. **Cross-platform.** Does it work on the *other* platform — shadows, keyboard,
   back button, fonts, safe areas?
7. **Tests.** Do they exist, do they test behavior (not implementation), do they
   cover the failure paths?
8. **Style & readability.** Naming, magic values, docs. Real, but lowest
   priority — and largely automatable via lints.

## 2. Full review checklist

**Architecture & design**
```
[ ] Layer boundaries respected (domain pure; UI doesn't import data/)
[ ] SOLID — esp. single responsibility & dependency inversion
[ ] No cyclic or cross-feature coupling
[ ] Abstractions justified (not speculative — YAGNI)
[ ] Business logic out of components (in use cases / hooks)
[ ] Server state in the query layer; client state in the store; no duplication
```

**Correctness**
```
[ ] Loading / empty / error / success all handled
[ ] Failures typed and mapped (no exceptions crossing layers)
[ ] No `any`, no assertion used to silence a real type error
[ ] API responses validated at the boundary, not asserted
[ ] Edge cases: empty lists, pagination end, timeouts, retries, offline
[ ] No state updates after unmount; effects cleaned up
[ ] Navigation params typed; back/gesture navigation handled
```

**Performance**
```
[ ] Re-renders scoped (selectors, memo with stable props, state pushed down)
[ ] Lists virtualized, memoized rows, stable keys, no nested same-axis scrollers
[ ] All listeners/timers/subscriptions/animations torn down
[ ] Heavy work off the JS thread; animations on the UI thread
[ ] Remote images sized and cached
```

**UI & layout** (see `ui-layout.md`, `ui-from-design.md`)
```
[ ] Values from tokens/i18n — no raw hex, inline font sizes, bare spacing numbers,
    or hardcoded strings
[ ] Nothing clipped at 320 dp or fontScale 1.3; API/i18n text truncates
[ ] Loading / empty / error / success states all drawn and reachable
[ ] Tappable surfaces have press feedback, a role, and a ≥44 pt / 48 dp target
[ ] Decorative images hidden from a11y; controls labelled
[ ] Dark theme verified, not assumed
[ ] Verified on BOTH platforms (shadows, keyboard, safe area, fonts)
[ ] Nothing duplicates an existing shared component
```

**Security**
```
[ ] No hardcoded secrets/keys/tokens (the JS bundle is readable)
[ ] Tokens in Keychain/Keystore; cleared on logout along with caches
[ ] 401/refresh single-flight; no token logging
[ ] HTTPS enforced; verification never disabled; pinning where warranted
[ ] Deep-link params validated; WebView locked down
```

**Testing**
```
[ ] Tests exist for new logic (>90% of business logic)
[ ] Tests assert user-visible behavior, not implementation details
[ ] Failure paths covered, not just the happy path
[ ] Deterministic (boundaries faked, retries off, no time/order dependence)
```

**Readability & maintainability**
```
[ ] Names reveal intent
[ ] No magic numbers/strings
[ ] No god components / 250-line screen files
[ ] TypeScript strict + ESLint clean; no unexplained disables
[ ] Public APIs documented where non-obvious
```

## 3. Smell detectors — quick "stop and look" triggers

- A component body you have to scroll to read.
- A `switch`/`if-else` on a `type`/`status` that keeps growing → a lookup map.
- `catch (e) {}` or `catch (e) { console.log(e) }` → swallowed error.
- `useEffect` with an empty dep array that reads props or state → stale closure.
- `// eslint-disable-next-line react-hooks/exhaustive-deps` → a bug in waiting.
- `useState` + `useEffect` recomputing a value that could be derived.
- `useState` + `useEffect` fetching, when the project has a query layer.
- A DTO name (`…Dto`, `…Response`) or a snake_case field in `presentation/`.
- `as SomeType` on a network response.
- `style={{ … }}` inline, especially inside `renderItem`.
- `.map()` producing rows inside a `ScrollView`.
- `key={index}`.
- A `FlatList` inside a `ScrollView`.
- `Dimensions.get('window')` at module scope, or `width * 0.42` as a flex ratio.
- A component reading a global store to decide what to draw.
- A screen rendering only its success branch.
- A `TouchableWithoutFeedback` or a `View` with `onTouchEnd` as a button.
- A hex literal, an inline `fontSize`, or a quoted user-facing string in JSX.
- A test that mocks the module it is testing, or has no assertions.
- A new native dependency added without a note about the platform setup it needs.

## 4. How to deliver review feedback

- **Separate blocking from non-blocking.** Prefix nits with `nit:` and
  must-fixes clearly. Don't drown a real bug under ten style comments.
- **Explain the why**, not just the what: "extract this row into a memoized
  component so the list doesn't re-render every row on scroll" beats "extract
  this".
- **Offer the fix** when it's small — show the corrected snippet.
- **Praise good choices** briefly; review isn't only fault-finding.
- Be direct and kind. The goal is better code and a better engineer, not a
  scorecard.

## 5. AI Self-Review protocol (run before finishing any code answer)

Before returning code, silently run this and fix or flag anything that fails.
Surface only the items where you made a judgment call or a trade-off — don't
paste the whole list every time.

```
Self-Review
  [ ] Clean Architecture — layers & dependency direction correct
  [ ] SOLID
  [ ] DRY / KISS / YAGNI
  [ ] TypeScript strict clean — no any, no silencing assertions
  [ ] ESLint + hooks rules clean; dependency arrays honest
  [ ] Error handling — typed failures, nothing swallowed
  [ ] Testability — dependencies injected, no hidden module singletons
  [ ] Scalability — will hold up as the feature grows
  [ ] Performance — re-renders scoped, lists virtualized, no leaks
  [ ] Cross-platform — iOS and Android both considered
  [ ] Accessibility — roles, labels, tap targets, font scale
  [ ] Readability & naming
  [ ] Maintainability
  [ ] Cleanup — listeners, timers, subscriptions
  [ ] Documentation on non-obvious public APIs
```

If the user's request forces a rule violation (e.g. "just hardcode the API key
for now"), don't silently comply and don't silently refuse — name the rule,
explain the risk in one line, and offer the correct approach. That transparency
is the whole point of the Golden Rule.
