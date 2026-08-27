# Feature Scaffolding — Clean Architecture, end to end

Read this when the task is **"create a feature"**, "add an endpoint/use case",
"wire a new store or query", or "how should this new piece be structured". It is
the concrete build order and file layout that `architect.md` describes abstractly
and `patterns.md` justifies pattern-by-pattern.

Pairs with `refactor.md` (§8 Component composition) for the UI half — this file
deliberately stops **before** custom UI.

---

## 0. Calibrate to the repository FIRST

This skill is repo-agnostic; every project names things slightly differently.
Before writing a single file, open the **canonical reference feature** — the most
complete, already-working feature in the codebase (usually `auth` or the oldest
shipped flow) — and record its actual conventions:

| Thing to confirm | Typical variants seen in the wild |
| --- | --- |
| `Result` implementation | project-local `core/error/result.ts` vs `neverthrow` vs plain `throw` + error boundary; **never** add a second one |
| Failure type | discriminated union on `kind` vs `Error` subclasses vs raw strings |
| File naming | `trip-card.tsx` (kebab) vs `TripCard.tsx` (Pascal) vs `trip_card.tsx` |
| Folder plurality | `use-cases/` vs `usecase/`, `components/` vs `ui/` |
| DTO validation | zod / valibot at the boundary vs an unchecked `as` cast |
| Mapping style | `mappers/trip.mapper.ts` vs `Trip.fromDto()` static vs inline in the repo |
| Server state | TanStack Query vs RTK Query vs SWR vs hand-rolled `useEffect` fetching |
| Client state | Zustand vs Redux Toolkit vs Context + reducer vs MobX |
| Query keys | a `tripKeys` factory vs inline array literals |
| DI | composition root + context vs module singletons vs `tsyringe`/InversifyJS |
| HTTP client | `axios` instance vs `ky` vs bare `fetch` wrapper — and where interceptors live |
| Navigation | React Navigation typed param list vs Expo Router file routes |
| i18n | `react-i18next` vs `i18n-js` vs `lingui`; where the catalogs live |
| Styling | `StyleSheet.create` + theme context vs NativeWind vs Unistyles vs Restyle |
| Codegen / generated files | API client generation, `react-native-svg-transformer`, typed routes |

**Match the reference feature. Do not import conventions from another repo.**
If the reference feature itself violates the standard, say so and propose the
fix — but do not silently introduce a third style.

Never hand-edit generated files: generated API clients, `*.d.ts` produced by a
codegen step, `ios/Pods`, `android/build`, or Expo's `.expo/types`.

---

## 1. Hard boundary — scaffold stops before custom UI

Scaffold everything **up to and including an empty screen skeleton**, then STOP.

- Do NOT design components, layouts, colors, charts, or cards on a fresh
  scaffold.
- The screen skeleton is a function component returning a `SafeAreaView`/screen
  container with at most a header. Nothing more.
- Do NOT copy UI patterns from unrelated screens or mock-data files — they are
  not necessarily representative.
- An empty `components/` folder is created next to the screen. It gets filled
  later, under `refactor.md` §8 rules.

Exception: if the user explicitly asks for the UI in the same request, build it
— but build it in **UI mode**: `ui-from-design.md` (pipeline + tokens + recipes),
`ui-layout.md` (screen shape, overflow, states, a11y), `refactor.md` §8 (file
structure). Never ad hoc.

---

## 2. Dependency rules (enforce strictly)

Allowed import direction — violations are architecture bugs, not style nits:

| Layer | May import | Must NOT import |
| --- | --- | --- |
| `domain/entities` | core types, other entities | data, presentation, react, react-native, axios, zod-of-the-wire |
| `domain/repositories` | core, own entities + params types | data, presentation |
| `domain/use-cases` | core, own entities + repository interface | data, presentation |
| `data/dto` | zod, core types | domain, presentation |
| `data/data-sources` | core, own DTOs, http client, storage, native SDKs | domain entities, presentation |
| `data/mappers` | own DTOs + domain entities | presentation |
| `data/repositories` | own data sources + mappers + domain interface | presentation, react |
| `presentation/hooks` | core, domain (use cases, entities), query/store libs | data layer, http client |
| `presentation/screens` | core, own hooks + components, navigation | data layer, use cases invoked directly from JSX |
| `presentation/components` | core theme + ui, own props | data, stores of other features, navigation objects |

Key consequences:

- Hooks talk ONLY to use cases (or the repository interface in a small app).
  Never import the HTTP client into a hook.
- UI reads ONLY entities. DTOs never cross into presentation.
- **Data source is transport + parsing only.** It does not decide business-level
  failures.
- **Repository owns DTO→entity mapping and failure classification** — that
  decision lives here, not in the data source and not in the hook.
- Offline / connectivity checks belong in the repository, before touching the
  data source.
- A **component** never reads navigation or a global store to know what to draw;
  it takes props (`refactor.md` §8.1).

---

## 3. Planning order (domain-first)

Each step fixes the contract for the next. Keep this order even when only some
layers are needed:

1. **Entity + branded ids** — what the UI ultimately needs. No JSON, no API
   field names, no optionals the UI would have to re-check.
2. **Repository interface** (domain) — operations in business terms, returning
   `Result<Failure, Entity>`.
3. **Use case(s) + params types** — one per operation.
4. **DTO schemas** — mirror the actual API payload exactly (request + response),
   as zod schemas with inferred types.
5. **Mapper** — params→request, response→entity.
6. **Data source** — transport + schema parsing only.
7. **Repository implementation** — mapping + failure classification +
   connectivity.
8. **Query keys + hooks** (server state) and/or **store** (client state). One
   hook per screen need; events mirror user intents.
9. **Screen skeleton + route registration + DI wiring.** STOP.
10. **Tests** — see `testing.md`; a scaffold is not done because it type-checks.

---

## 4. Decision guide

- **No backend yet?** Create the data-source interface + a stub implementation
  that rejects with a `not_implemented` failure. Contract first, transport later.
- **Operation returns nothing?** `Result<Failure, void>` — return `ok(undefined)`.
  Do not return `true` as a stand-in for success.
- **Use case without input?** Take no parameter; do not invent a `NoParams` type
  in TypeScript, where an optional parameter is native.
- **Local persistence (tokens, flags, small prefs)?** Do NOT create a new storage
  instance. Add a typed getter/setter to the existing storage wrapper and inject
  it. **Tokens and credentials go to Keychain/Keystore, never MMKV or
  AsyncStorage** (`security.md` §2).
- **Offline support needed?** A local data source next to the remote one, same
  shape, `cache` failures on error; the repository decides the order.
- **New error case?** Add a variant to the `Failure` union with its i18n key.
  Do not sprinkle raw message strings through layers.
- **Several operations in one feature?** One use case per operation, ONE
  repository interface holding all methods, one hook per screen concern. Never
  two competing stores for one screen.
- **Mutation?** `useMutation` + explicit `invalidateQueries` on the affected
  keys, or an optimistic update with a rollback in `onError`. A mutation that
  leaves stale data on screen is an unfinished mutation.
- **Extending an existing feature?** Same order, additive only: repository
  interface method → implementation → use case → hook → screen wiring.

---

## 5. Anti-patterns — reject on sight

- A second `Result`/error library when the project already has one.
- `throw` crossing a layer boundary the project doesn't expect it to cross.
- A hook or component importing `axios`, a data source, or a DTO.
- `useEffect` + `useState` hand-rolled fetching when the project has a query
  library — no dedupe, no cancellation, no retry, and a race on fast navigation.
- An entity carrying `created_at` (wire naming) or `fromJson` logic.
- Storing server data in Zustand alongside Query (`patterns.md` §7).
- `as TripDto` on an unvalidated response.
- Manual `Dimensions.get('window')` at module scope (frozen on rotation/fold).
- Building custom UI beyond the skeleton on a fresh scaffold.
- Hardcoded user-facing strings — add the key to **every** locale catalog the
  project ships.

---

## 6. Layer templates

Shapes, not gospel — bend the names to the reference feature (§0).

### 6.1 Entity (domain, pure)

```ts
// features/trips/domain/entities/trip.ts
export type TripId = Brand<string, 'TripId'>;

export type TripStatus = 'draft' | 'active' | 'delivered' | 'cancelled';

export interface Trip {
  readonly id: TripId;
  readonly origin: string;
  readonly destination: string;
  readonly status: TripStatus;
  readonly price: Money;
}
```

### 6.2 Repository interface (domain)

```ts
// features/trips/domain/repositories/trip-repository.ts
export interface GetTripsParams {
  readonly status?: TripStatus;
  readonly page: number;
}

export interface TripRepository {
  getTrips(params: GetTripsParams): Promise<Result<Failure, Trip[]>>;
  getTripById(id: TripId): Promise<Result<Failure, Trip>>;
}
```

### 6.3 Use case (domain)

```ts
// features/trips/domain/use-cases/get-trips.ts
export const makeGetTrips =
  (repository: TripRepository) =>
  (params: GetTripsParams): Promise<Result<Failure, Trip[]>> =>
    repository.getTrips(params);

export type GetTrips = ReturnType<typeof makeGetTrips>;
```

### 6.4 DTO (data) — schema first, type inferred

```ts
// features/trips/data/dto/trip.dto.ts
export const tripDtoSchema = z.object({
  id: z.string(),
  origin_city: z.string(),
  destination_city: z.string(),
  status_code: z.number().int(),
  price_minor: z.number().int(),
  currency: z.string().length(3),
});

export type TripDto = z.infer<typeof tripDtoSchema>;
```

### 6.5 Mapper (data)

```ts
// features/trips/data/mappers/trip.mapper.ts
const STATUS_BY_CODE: Record<number, TripStatus> = {
  0: 'draft', 1: 'active', 2: 'delivered', 3: 'cancelled',
};

export function toDomain(dto: TripDto): Trip {
  return {
    id: dto.id as TripId,
    origin: dto.origin_city,
    destination: dto.destination_city,
    status: STATUS_BY_CODE[dto.status_code] ?? 'draft',
    price: money(dto.price_minor, dto.currency),
  };
}
```

### 6.6 Data source (data) — transport + parse only

```ts
// features/trips/data/data-sources/trip-remote-data-source.ts
export interface TripRemoteDataSource {
  getTrips(params: GetTripsParams): Promise<TripDto[]>;
}

export function makeTripRemoteDataSource(http: HttpClient): TripRemoteDataSource {
  return {
    async getTrips(params) {
      const response = await http.get('/trips', {
        params: { status: params.status, page: params.page },
      });
      return z.array(tripDtoSchema).parse(response.data);   // contract enforced here
    },
  };
}
```

### 6.7 Repository implementation (data)

```ts
// features/trips/data/repositories/trip-repository.impl.ts
export function makeTripRepository(deps: {
  remote: TripRemoteDataSource;
  network: NetworkInfo;
}): TripRepository {
  return {
    async getTrips(params) {
      if (!(await deps.network.isConnected())) {
        return err({ kind: 'network', message: 'no_internet' });
      }
      try {
        const dtos = await deps.remote.getTrips(params);
        return ok(dtos.map(toDomain));
      } catch (e) {
        return err(toFailure(e));         // ONE place classifies errors
      }
    },
    /* getTripById… */
  };
}
```

### 6.8 Query keys + hook (presentation)

```ts
// features/trips/presentation/hooks/use-trips.ts
export const tripKeys = {
  all: ['trips'] as const,
  list: (params: GetTripsParams) => [...tripKeys.all, 'list', params] as const,
};

export function useTrips(params: GetTripsParams) {
  const { getTrips } = useUseCases();
  return useQuery({
    queryKey: tripKeys.list(params),
    queryFn: async () => {
      const result = await getTrips(params);
      if (!result.ok) throw result.error;
      return result.value;
    },
  });
}
```

Hook rules:

- ONE hook per screen concern; the screen consumes hooks, never raw use cases.
- Derived values (`isEmpty`, `total`) are computed in the hook or during render,
  never stored in state.
- A mutation hook invalidates or updates the exact keys it affects — no blanket
  `invalidateQueries()` with no key.
- Query/store state maps to the four screen states (`ui-layout.md` §8); the
  screen must not invent a fifth.

### 6.9 Screen skeleton (STOP here on a fresh scaffold)

```tsx
// features/trips/presentation/screens/trips-screen.tsx
export function TripsScreen() {
  return (
    <ScreenContainer>
      <AppHeader title={t('trips.title')} />
    </ScreenContainer>
  );
}
```

Effects, listeners, and helper logic go in a feature hook, not inline in the
screen body. New i18n keys go into **every** locale catalog the project ships.

### 6.10 Route registration — typed, never stringly

```ts
// core/navigation/types.ts
export type RootStackParamList = {
  Trips: undefined;
  TripDetails: { tripId: TripId };
};

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

```tsx
<Stack.Screen name="TripDetails" component={TripDetailsScreen} />
```

Routing details that are easy to get wrong:

- **Never type a route name as a bare string at a call site.** The param list is
  the contract; `navigation.navigate('TripDetails', { tripId })` must fail to
  compile if the params are wrong.
- **Pass ids, not entities.** Route params end up in deep-link URLs and state
  persistence; a whole object there is unserializable weight. Pass `tripId` and
  read the entity from the query cache.
- **Returning a result** from a screen: a callback param is not serializable —
  use a store value, a query invalidation, or `navigate` back with params.
- **Auth/onboarding guards** live in one place — the navigator's conditional
  tree — not in each screen's `useEffect`.
- **Deep links** validate their parameters before acting (`security.md` §6); a
  link must not drive privileged navigation with an unchecked id.
- **Effects tied to focus** use `useFocusEffect`, not `useEffect` — a screen in
  the stack is mounted but not visible.

### 6.11 DI wiring

Register the new repository + use cases in the composition root
(`patterns.md` §6). No module-level singletons, no importing the container from
a component — `useUseCases()` only.

---

## 7. Directory layout (per feature)

```
src/features/<feature>/
  data/
    data-sources/
      <feature>-remote-data-source.ts
      <feature>-local-data-source.ts        # only if caching is real
    dto/
      <name>.dto.ts
    mappers/
      <name>.mapper.ts
    repositories/
      <feature>-repository.impl.ts
  domain/
    entities/
      <name>.ts
    repositories/
      <feature>-repository.ts               # interface
    use-cases/
      <name>.ts
  presentation/
    hooks/
      use-<name>.ts
    store/
      <feature>.store.ts                    # client state only, if needed
    screens/
      <name>-screen.tsx
    components/                             # created empty — filled per refactor.md §8
  index.ts                                  # the feature's public surface
```

File naming and folder plurality follow the reference feature (§0).

---

## 8. Finish checklist

1. Dependency table (§2) holds — no forbidden imports (run the lint boundary
   rule, don't eyeball it).
2. `tsc --noEmit` — zero errors, no new `any`, no new `@ts-expect-error`.
3. ESLint — zero new warnings, `react-hooks/exhaustive-deps` honest.
4. Locale catalogs updated in **every** language, if strings were added.
5. Tests exist for the use case + repository + hook at minimum
   (`testing.md`). A scaffold with no tests is not finished, it is started.
6. If any runtime behavior changed, the `Manual Test Result` block from
   `manual-test.md` is filled in — or the response says plainly
   "code-complete, NOT device-tested" with exact steps.
7. Report to the user: files created, which use cases/hooks exist, and that the
   screen body + `components/` are intentionally left for them to draw.
