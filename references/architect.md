# Architecture — Clean Architecture, Feature-First, SOLID, DDD

Read this when structuring a new project or feature, deciding where a file
lives, or auditing layer boundaries.

## 1. Layer model

Three layers, one dependency direction. **The domain layer is the center and
depends on nothing.**

```
presentation  ──depends on──▶  domain  ◀──depends on──  data
 (screens, hooks, stores)      (pure TS)          (axios, zod, MMKV, SQLite)
```

- **domain** — entities, value objects, repository *interfaces*, use cases,
  failures. Pure TypeScript. No `react`, no `react-native`, no `axios`, no
  `zod` schemas describing wire format, no navigation types.
  If you `import { View } from 'react-native'` here, it's wrong.
- **data** — repository *implementations*, remote/local data sources, DTO
  schemas, mappers. Depends on domain (implements its interfaces). Knows about
  the HTTP client, MMKV/AsyncStorage, SQLite, the native SDKs.
- **presentation** — screens, components, hooks, stores, query hooks,
  navigation. Depends on domain use cases. Never imports from `data/`.

**The Dependency Rule:** source code dependencies point inward only. Inner
layers know nothing about outer layers. The UI could be swapped from React
Native to anything and the domain would never notice.

A practical enforcement: an ESLint `no-restricted-imports` (or
`eslint-plugin-boundaries`) rule per layer, so a forbidden import fails CI
instead of failing review. See §7.

## 2. Feature-first folder structure

Organize by feature, not by technical type. A new engineer should read the
folder tree and understand *what the app does*, not just *what patterns it uses*.

```
src/
├── core/                       # shared, cross-feature
│   ├── error/                  # Failure types, Result helpers
│   ├── network/                # http client, interceptors
│   ├── storage/                # secure storage + MMKV wrappers
│   ├── theme/                  # tokens, provider, typed useTheme
│   ├── ui/                     # shared design-system components
│   ├── i18n/                   # translation setup
│   ├── navigation/             # root navigator, typed param lists
│   └── di/                     # composition root
├── features/
│   └── trips/
│       ├── domain/
│       │   ├── entities/          trip.ts
│       │   ├── repositories/      trip-repository.ts      (interface)
│       │   └── use-cases/         get-trips.ts
│       ├── data/
│       │   ├── dto/               trip.dto.ts             (zod schema + type)
│       │   ├── data-sources/      trip-remote-data-source.ts
│       │   ├── mappers/           trip.mapper.ts
│       │   └── repositories/      trip-repository.impl.ts
│       └── presentation/
│           ├── hooks/             use-trips.ts            (query/store binding)
│           ├── store/             trips.store.ts          (client state only)
│           ├── screens/           trips-screen.tsx
│           └── components/        trip-card.tsx
└── app.tsx
```

Rules:
- One feature never imports another feature's `data/` or `presentation/`. Shared
  needs go to `core/` or a shared feature exposing a domain interface.
- `core/` holds only genuinely cross-cutting code. Resist the urge to dump
  feature logic there — it becomes a god package.
- Barrel files (`index.ts`) at the **feature** boundary are fine and make the
  public surface explicit. Barrels inside a layer mostly cause cycles and slow
  Metro — skip them.

## 3. SOLID in React Native terms

**S — Single Responsibility.** A component has one reason to change. A
`TripsScreen` that also parses API JSON, formats dates, and owns pagination has
three. Split them: parsing → mapper, formatting → a pure helper, orchestration →
a hook or use case.

**O — Open/Closed.** Extend behavior without editing existing code. Add a new
`PaymentMethod` by adding a module implementing the interface, not by growing a
`switch` in the checkout screen. Use a lookup map or the Strategy pattern.

**L — Liskov Substitution.** A substitute must honor the same contract. A
`CachedTripRepository` implementing `TripRepository` returns the same failure
shapes — it must not start throwing where the interface promised a `Result`.

**I — Interface Segregation.** Prefer small, focused interfaces. A read-only
data source shouldn't be forced to implement `delete`/`update`. Split
`ReadableSource` and `WritableSource` when consumers differ.

**D — Dependency Inversion.** Depend on abstractions. The hook depends on a
`getTrips` use case, and the use case on a `TripRepository` interface — never on
`axios` or `TripRepositoryImpl` directly. Concrete classes are wired only in the
composition root.

```ts
// GOOD — presentation depends on the abstraction, injected once
export function useTrips(params: GetTripsParams) {
  const { getTrips } = useUseCases();            // typed DI context
  return useQuery({
    queryKey: tripKeys.list(params),
    queryFn: () => getTrips(params),
  });
}

// BAD — presentation reaches into data + transport
export function useTrips() {
  return useQuery({ queryFn: () => axios.get('/trips') }); // ❌ layer + framework leak
}
```

## 4. Domain-Driven Design essentials

- **Entities** have identity (an `id`) and equality by that identity.
- **Value objects** are immutable, equal by value, and enforce their own
  invariants at construction. Use them to make illegal states unrepresentable.
  In TypeScript, **branded types** give you this with zero runtime cost plus a
  smart constructor for the validation.

```ts
declare const brand: unique symbol;
type Brand<T, B> = T & { readonly [brand]: B };

export type PhoneNumber = Brand<string, 'PhoneNumber'>;

const E164 = /^\+[1-9]\d{7,14}$/;

export function createPhoneNumber(input: string): Result<ValidationFailure, PhoneNumber> {
  const normalized = input.replace(/\s+/g, '');
  if (!E164.test(normalized)) {
    return err({ kind: 'validation', message: 'Invalid phone number' });
  }
  return ok(normalized as PhoneNumber);
}
```

Prefer branded types over bare `string`/`number` for domain concepts (money,
IDs, coordinates, phone numbers). "Primitive obsession" is a real smell — it
scatters validation across the codebase, and `sendTo(userId, orderId)` compiles
happily when both are `string`.

- **Aggregates** — a cluster of entities/value objects with one root. Outside
  code talks only to the root, keeping invariants in one place.

## 5. Use cases

One use case = one business operation. It orchestrates repositories and encodes
a single intent. Keep them thin; they are *not* a dumping ground.

```ts
export type GetTrips = (params: GetTripsParams) => Promise<Result<Failure, Trip[]>>;

export const makeGetTrips =
  (repository: TripRepository): GetTrips =>
  (params) =>
    repository.getTrips(params);
```

A use case may be a plain function factory (above) or a class — pick one style
per repo. If a use case grows conditionals and coordinates five repositories,
that logic probably belongs in a domain service or should be split.

## 6. Common layer-boundary violations to reject

- A DTO type (`TripDto`) used in a component → map to the `Trip` entity first.
- A navigation object, `useTheme`, or any React hook imported into domain/data →
  never.
- A repository implementation letting an `AxiosError` escape → catch in the data
  layer, map to a typed `Failure`.
- Business rules living in `onPress` handlers → move into a use case.
- A feature importing another feature's internals → invert via a domain
  interface placed in `core/` or the owning feature's public API.
- A Zustand store holding server data that TanStack Query already owns → delete
  the duplicate (see `patterns.md` §8).

## 7. Package boundaries & cohesion

- **High cohesion:** things that change together live together (a feature's
  hooks, screen, and components).
- **Low coupling:** features communicate through domain contracts, not shared
  mutable singletons.
- Watch for **cyclic dependencies** between features — they signal a missing
  shared abstraction. Extract it to `core/` or a `shared` feature.
- Enforce the rules mechanically:

```jsonc
// .eslintrc — domain must stay pure
"no-restricted-imports": ["error", {
  "patterns": [
    { "group": ["react", "react-native", "@react-navigation/*", "axios"],
      "message": "domain/ must not depend on frameworks" }
  ]
}]
```

  Scope this per-directory with ESLint `overrides`, one block per layer.
- For large apps, consider splitting features into workspace packages (pnpm/yarn
  workspaces + Turborepo/Nx) so boundaries are enforced by the resolver, not by
  discipline. See `enterprise.md`.
