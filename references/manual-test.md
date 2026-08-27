# Manual Self-Test — Run What You Wrote

Read this after producing or changing any code that affects runtime behavior.
Unit tests (see `testing.md`) prove the logic; this file is about proving the
*actual app* — the assistant launches its own change, pokes it like a QA
engineer, and reports evidence. "It compiles" and "the tests pass" are not the
same claim as "I watched it work".

## The Honesty Rule

> **Never claim a change works unless you ran it and saw it work.**
>
> If no device/simulator is available, or the flow needs a second party (a real
> push, an SMS code, a payment sandbox), say exactly that: *"code-complete, not
> device-tested — verify X on a device"*. A false "tested ✓" is worse than an
> honest "untested" because it cancels the user's own verification.

React Native adds a second axis to this rule: **a change verified on one platform
is verified on one platform.** Shadows, keyboard behavior, safe areas, fonts,
back navigation, and permissions all diverge. Say which platform you tested.

## 1. Gate order — cheapest signal first

Run these in order; stop and fix at the first failure. Don't boot a simulator to
discover a type error.

```bash
npx prettier --check .            # or the project's format script
npx eslint . --max-warnings=0
npx tsc --noEmit
npx jest --silent                 # unit/component suite still green
```

Only when all four are green does manual runtime testing begin.

## 2. Write the test plan BEFORE launching

Three lines minimum, derived from the change itself — not a generic smoke test.
For every change list:

```
Manual Test Plan
  Feature/fix  : <what changed, one line>
  Preconditions: <login state, seed data, locale, feature flags, platform>
  Steps        : 1) … 2) … 3) …
  Expect       : <observable outcome — text, navigation, log line, no crash>
  Edge pokes   : <the 2-3 unhappy paths this change could break>
```

Edge pokes to always consider: loading/empty/error states, airplane mode, slow
network, rapid double-tap, back gesture mid-operation, locale switch, dark mode,
`fontScale` 1.3, small screen, app backgrounded mid-flow, and **the other
platform**.

## 3. Launch and drive

```bash
# Expo
npx expo start                      # then i / a, or a dev-client build
# Bare React Native
npx react-native run-ios --simulator="iPhone 15"
npx react-native run-android
```

- Prefer a **running session + Fast Refresh** while iterating; do one **cold
  start** at the end, because init paths (storage hydration, navigation
  restoration, deep links, permissions) only execute there.
- If you changed native code, a config plugin, or a dependency with native
  modules, Fast Refresh proves nothing — **rebuild**, and say that you did.
- Drive the UI yourself where possible. On Android, `adb shell input
  tap/text/swipe/keyevent` keeps the loop scriptable; on iOS,
  `xcrun simctl` covers launch, deep links, and permissions.
- For flows too complex to drive by hand, write a throwaway Maestro flow, run it
  once, then delete it — it is a probe, not a suite member.
- Simulate the states you can: airplane mode, a throttled network profile,
  `xcrun simctl ui booted appearance dark`, and a large `fontScale` in system
  settings.

## 4. Collect evidence — don't just watch

A manual test that leaves no artifact is a claim, not a result. Capture at
least one of:

```bash
# Screenshots
xcrun simctl io booted screenshot shot-ios.png
adb exec-out screencap -p > shot-android.png

# Logs, filtered to the change
npx react-native log-ios | grep -E "MYTAG|Error"
adb logcat -s ReactNativeJS:V | grep -E "MYTAG|Error"
```

- Add temporary `console.log('MYTAG: …')` markers at the decision points of your
  change, assert their order in the log output, then **remove them before
  finishing** — a leftover marker is a review defect.
- Read the log for *unexpected* lines too: a feature can "work" while spamming
  key warnings, `act` warnings, or an unhandled promise rejection from a listener
  you forgot to clean up.
- A UI change without a screenshot is not verified. Take one per platform.

## 5. Report — evidence in, adjectives out

End the response with the executed plan, not a vibe:

```
Manual Test Result
  ✓ Steps 1-3 pass on iPhone 15 simulator (iOS 17.4) and Pixel 6 emulator (API 34)
  ✓ Log shows MYTAG:start → MYTAG:done, no unhandled rejections
  ✓ Error path: airplane mode → error state + retry works
  ✓ Dark mode + fontScale 1.3 checked, no clipping
  ✗ NOT tested: push-tap deep link (needs a real FCM push) — verify on device
```

Every `✗ NOT tested` must name what remains and how the user can verify it.
Failed steps are reported with the exact output, then fixed — never silently
re-run until green without saying what changed.

## 6. When runtime testing is impossible

No simulator, a native module needing real hardware (camera, NFC, biometrics), a
flow needing a second user or a backend event:

1. Still run all of §1 (static gates + unit tests always work).
2. Cover the logic with a component test that simulates the runtime path as
   closely as possible.
3. Declare the gap explicitly using the Honesty Rule wording, and hand the user
   a copy-pasteable test plan (§2) so *they* can run it in one minute.

## What NOT to do

- Don't substitute a green `tsc` for a runtime check on UI changes.
- Don't call a change cross-platform-verified after testing one platform.
- Don't test only the happy path — the plan's "edge pokes" line exists because
  regressions live there.
- Don't leave probe files, debug markers, or commented-out mocks in the diff.
- Don't run the full manual loop for a comment typo or a pure refactor with green
  tests — scale the ceremony to the blast radius of the change.
