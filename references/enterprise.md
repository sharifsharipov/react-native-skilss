# Enterprise — CI/CD, Releases, OTA, Monorepo, Flavors, ADRs

Read this when setting up project infrastructure, scaling a codebase across a
team, or defining release/versioning process. These are the practices that keep a
large app shippable by many people over years.

## 1. CI/CD — the automated gate

Every PR must pass, before merge, in this order (fail fast on the cheap steps):

```yaml
# Conceptual pipeline
1. install        # cached node_modules / pnpm store
2. format-check   prettier --check .
3. lint           eslint . --max-warnings=0
4. typecheck      tsc --noEmit
5. test           jest --coverage        # + coverage threshold gate
6. build          eas build / gradle assembleRelease / xcodebuild, per flavor
7. e2e            maestro test / detox   # nightly if slow
8. (main only)    distribute (TestFlight / Play internal / EAS Update)
```

- **Gate on coverage** for business logic so it can't silently rot.
- Cache the package store, Gradle, and CocoaPods — a slow pipeline gets bypassed.
- Sign builds with secrets from the CI secret store, never from the repo.
- Protect `main`: required checks + review approval, no direct pushes.
- Native builds are the slow step: run them on `main` and release branches, and
  keep the JS gates on every PR so feedback stays fast.

## 2. Flavors / build environments

Separate `dev`, `staging`, `prod` with distinct bundle ids, display names, icons,
and config so they can coexist on one device and can't cross-talk.

- **Expo:** `app.config.ts` reading `process.env.APP_ENV`, plus EAS build
  profiles in `eas.json`.
- **Bare:** Android product flavors + iOS schemes/configurations, with a config
  module (`react-native-config`) per environment.

```jsonc
// eas.json (excerpt)
{
  "build": {
    "staging": { "env": { "APP_ENV": "staging" }, "distribution": "internal" },
    "production": { "env": { "APP_ENV": "production" }, "autoIncrement": true }
  }
}
```

- Config values come from the environment, kept out of git; secrets from the CI/
  EAS secret store (`security.md` §1).
- Never point a dev/staging build at production data, and never ship a debug
  affordance (a hidden env switcher, a cert bypass) in the production flavor.

## 3. OTA updates — power and hazard

`expo-updates` / CodePush can ship a JS fix in minutes. That makes it the most
abused tool in the React Native toolbox.

- **An OTA is a release.** It gets the same review, the same QA, and a changelog
  entry. "It's just JS" is how a bricked launch screen reaches every user at
  once.
- Ship to a **staged rollout channel** first, watch crash-free rate, then promote.
- **Runtime versions matter:** an OTA bundle that calls a native API not present
  in the installed binary crashes on launch. Bump the runtime version whenever
  native code changes, and let the update system refuse mismatched bundles.
- Keep a rollback path — the previous bundle, republishable in one command.
- Never OTA around a store review requirement; that gets apps removed.
- Sign updates and lock down who can publish (`security.md` §8).

## 4. Monorepo — enforcing boundaries by the resolver

When features grow, split them into workspace packages so layer boundaries are
enforced by the build system rather than by discipline. A `feature-trips` package
that doesn't depend on `feature-payments` *cannot* import it.

```
apps/
├── mobile/                 # thin shell: navigation, DI wiring, entry per flavor
packages/
├── core/                   # error, network, storage, theme, ui, i18n
├── feature-trips/
├── feature-payments/
└── config/                 # shared tsconfig/eslint/jest presets
```

- pnpm/yarn workspaces + **Turborepo** or **Nx** for task orchestration and
  caching.
- Metro needs explicit configuration for a monorepo (`watchFolders`,
  `nodeModulesPaths`) — budget time for it; this is the usual source of "works
  locally, fails in CI".
- The app package composes features; features never depend on each other
  directly — shared contracts live in `core`.

Don't start here for a small app (YAGNI) — migrate when a single `src/` becomes
painful to reason about or multiple squads step on each other.

## 5. Architecture Decision Records (ADRs)

Record significant, hard-to-reverse decisions so the *why* survives team
turnover. One short markdown file per decision in `docs/adr/`.

```markdown
# ADR-0007: TanStack Query for server state
Status: Accepted — 2026-01-15
Context: Server data was mirrored into Redux with hand-written invalidation;
         three staleness bugs shipped in two months.
Decision: TanStack Query owns all server state. Zustand keeps client state only.
Consequences: + caching/retry/dedupe for free, less boilerplate;
              − two state systems to teach; query keys need a convention.
Alternatives considered: RTK Query; SWR; keeping the hand-rolled layer.
```

Write an ADR for: state-management choice, error-handling strategy, DI approach,
navigation library, styling approach, Expo vs bare, New Architecture adoption,
monorepo split, and any choice you'd be annoyed to see silently reversed.

## 6. Versioning & releases

- **Semantic versioning** for the app and shared packages.
- `version` in `app.json`/`package.json` plus an auto-incremented native build
  number (`buildNumber` / `versionCode`) from CI — it must always increase, or
  the stores reject the upload.
- Maintain a **CHANGELOG** (Keep a Changelog format); the "Unreleased" section is
  updated per PR.
- Consider Conventional Commits (`feat:`, `fix:`, `refactor:`…) — it enables
  automated changelog + version bumping and makes history greppable.
- Tag releases, and **upload source maps** for each release so production crash
  traces are readable. A minified Hermes stack trace with no source map is an
  unactionable crash report.
- Know the store realities: iOS review latency, phased release, and the fact that
  a bad native build takes days to replace — which is exactly why §3's discipline
  matters.

## 7. Observability in production

- Crash reporting (Sentry / Crashlytics) wired from day one, with source maps
  uploaded in CI and PII scrubbed.
- Track **crash-free sessions** and **ANR rate** per release, not just crash
  counts. A release dashboard is how you notice a bad rollout in an hour instead
  of a week.
- Structured, leveled logging behind a logger interface — off or minimal in
  release. No raw `console.log` in shipped code.
- Performance monitoring: app start time, screen render time, and network
  latency per endpoint. Startup regressions are invisible without a number.
- Feature flags / remote config for risky rollouts and kill-switches.
- Analytics events defined centrally (a typed event map), not sprinkled as raw
  strings.

## 8. Dependency & native hygiene

- Pin versions; review updates deliberately, not reactively during an incident.
- Audit every new dependency: maintenance, popularity, bundle weight, license,
  **and whether it has native code**. A native dependency changes your build,
  your minimum OS versions, your Expo compatibility, and your upgrade cost.
- Prefer a thin adapter around volatile third-party SDKs (`patterns.md` §9) so a
  swap touches one file.
- **React Native upgrades are their own project**, not a chore squeezed into a
  feature PR: use the Upgrade Helper, do it on a dedicated branch, test both
  platforms end to end, and expect native config drift.
- Track the New Architecture status of your dependencies before committing to
  the migration.
- Run `npm audit` / Snyk in CI, and watch native SDK advisories too.

## 9. Definition of Done (team contract)

A change is "done" when:
```
[ ] prettier + eslint clean (zero warnings)
[ ] tsc --noEmit clean
[ ] Tests written & passing; business-logic coverage gate met
[ ] Self-review checklist (review.md) passed
[ ] Verified on iOS AND Android (manual-test.md), evidence attached
[ ] Docs/ADR/CHANGELOG updated if the change warrants it
[ ] Builds for all flavors in CI
[ ] Reviewed & approved by a peer
```

Infrastructure scales a team the same way clean architecture scales a codebase:
by making the right thing the easy thing and the wrong thing hard.
