# Design Patterns — Repository, UseCase, Result, zod, DI, Mapper, State

Read this when implementing data flow, error handling, DI wiring, or choosing a
pattern for a piece of logic.

## 1. Result type — error handling without exceptions

Do not throw across layers. The domain boundary returns a `Result` discriminated
union (or `neverthrow`'s `Result` if the project uses it). This makes failure
part of the type signature, so callers cannot forget to handle it.

```ts
// core/error/result.ts
export type Result<E, T> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

export const ok  = <T>(value: T): Result<never, T> => ({ ok: true, value });
export const err = <E>(error: E): Result<E, never> => ({ ok: false, error });
```

```ts
// core/error/failure.ts — typed, exhaustive, serializable
export type Failure =
  | { kind: 'server';     message: string; status?: number }
  | { kind: 'network';    message: string }
  | { kind: 'cache';      message: string }
  | { kind: 'validation'; message: string; field?: string }
  | { kind: 'unauthorized'; message: string }
  | { kind: 'unknown';    message: string };
```

A `kind` discriminant beats an `Error` subclass hierarchy in React Native:
`instanceof` survives neither serialization across the bridge nor some Hermes/
transpile setups, and a union gives you exhaustive `switch` checking for free.

Consuming a `Result` — exhaustively, no unhandled path:

```ts
const result = await getTrips(params);
if (!result.ok) {
  setError(toMessage(result.error));   // maps Failure.kind → i18n key
  return;
}
setTrips(result.value);
```

Map exceptions to failures **only in the data layer**:

```ts
async function getTrips(params: GetTripsParams): Promise<Result<Failure, Trip[]>> {
  try {
    const dtos = await remote.fetchTrips(params);
    return ok(dtos.map(toDomain));
  } catch (e) {
    return err(toFailure(e));          // one place classifies transport errors
  }
}
```

## 2. Repository pattern

The repository is the seam between domain and data. Domain defines the
*interface*; data provides the *implementation*. This lets you swap remote for
cache, or fake it in tests, with zero UI changes.

```ts
// domain/repositories/trip-repository.ts  (pure abstraction)
export interface TripRepository {
  getTrips(params: GetTripsParams): Promise<Result<Failure, Trip[]>>;
  getTripById(id: TripId): Promise<Result<Failure, Trip>>;
}
```

```ts
// data/repositories/trip-repository.impl.ts
export function makeTripRepository(deps: {
  remote: TripRemoteDataSource;
  local: TripLocalDataSource;
  network: NetworkInfo;
}): TripRepository {
  return {
    async getTrips(params) {
      if (!(await deps.network.isConnected())) {
        const cached = await deps.local.getCachedTrips();
        return cached.length > 0
          ? ok(cached.map(toDomain))
          : err({ kind: 'network', message: 'offline_no_cache' });
      }
      try {
        const dtos = await deps.remote.getTrips(params);
        await deps.local.cacheTrips(dtos);        // offline-first write-through
        return ok(dtos.map(toDomain));
      } catch (e) {
        return err(toFailure(e));
      }
    },
    /* getTripById… */
  };
}
```

Factory functions (above) and classes are both fine. Pick the one the repo
already uses and hold it — a codebase with both styles costs more than either.

## 3. Mapper pattern — DTO ⇄ Entity

DTOs describe the wire format and live in `data/`. Entities live in `domain/`. A
mapper translates between them so a JSON shape change never ripples into the UI.

```ts
// data/dto/trip.dto.ts — the schema IS the parser and the type
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

```ts
// data/mappers/trip.mapper.ts
export function toDomain(dto: TripDto): Trip {
  return {
    id: dto.id as TripId,
    origin: dto.origin_city,
    destination: dto.destination_city,
    status: tripStatusFromCode(dto.status_code),   // number → union here, not in UI
    price: money(dto.price_minor, dto.currency),
  };
}
```

Rules:
- snake_case ends at the mapper. It must never reach a component.
- Nullable/optional wire fields are resolved here into a total domain type —
  the UI should not have to ask "is `price` maybe undefined?".
- A DTO that maps to a union of domain types switches exhaustively on its
  discriminator **inside the mapper**.

## 4. Validate at the boundary with zod

Never trust the server response's type. `await http.get<TripDto[]>()` is a lie —
the generic is an assertion, not a check. Parse it:

```ts
const response = await http.get('/trips', { params });
const parsed = z.array(tripDtoSchema).safeParse(response.data);
if (!parsed.success) {
  throw new ContractError('GET /trips', parsed.error);  // caught in the repo → Failure
}
return parsed.data;
```

This turns "the app crashes three screens later on `undefined.toFixed`" into one
named failure at the boundary, with the offending field in the message. Parse
once, at the data source; downstream code works with the validated type.

## 5. UseCase pattern

One business operation per use case, callable as a function.

```ts
export type UseCase<Params, T> = (params: Params) => Promise<Result<Failure, T>>;

export const makeGetTrips =
  (repository: TripRepository): UseCase<GetTripsParams, Trip[]> =>
  (params) =>
    repository.getTrips(params);
```

Use cases earn their keep when they compose (validate → call → record analytics)
or when several screens share one operation. For a straight pass-through in a
small app, calling the repository from the query hook is acceptable — say so
rather than generating ceremony (YAGNI).

## 6. Dependency Injection — composition root + typed context

No service locator, no importing a concrete data source from a component.
Everything concrete is constructed **once**, at the composition root.

```ts
// core/di/container.ts
export function createContainer(http: HttpClient, storage: Storage) {
  const tripRemote = makeTripRemoteDataSource(http);
  const tripLocal  = makeTripLocalDataSource(storage);
  const tripRepo   = makeTripRepository({ remote: tripRemote, local: tripLocal, network });

  return {
    useCases: {
      getTrips: makeGetTrips(tripRepo),
      getTripById: makeGetTripById(tripRepo),
    },
  } as const;
}
export type Container = ReturnType<typeof createContainer>;
```

```tsx
// core/di/container-context.tsx
const ContainerContext = createContext<Container | null>(null);

export function ContainerProvider({ container, children }: Props) {
  return <ContainerContext.Provider value={container}>{children}</ContainerContext.Provider>;
}

export function useUseCases() {
  const container = useContext(ContainerContext);
  if (!container) throw new Error('ContainerProvider is missing');
  return container.useCases;
}
```

Why this and not a module-level singleton: tests swap the whole container in one
line, a second environment (Storybook, E2E with a fake backend) is a different
container, and nothing in the tree can accidentally reach into `data/`.

## 7. Server state vs client state — the split that prevents most state bugs

| The data is… | It belongs in |
| --- | --- |
| Fetched from the API, cacheable, shared across screens | **TanStack Query** |
| Auth session, feature flags, theme choice, cart draft | **Zustand / Redux** |
| Form input, an open modal, a selected tab | **local `useState`** |
| Derived from any of the above | **computed during render — not stored** |

The single most common React Native state defect is copying server data into a
global store, then hand-writing the invalidation. Query already owns fetching,
caching, deduplication, retries, background refetch, and invalidation.

```ts
export const tripKeys = {
  all: ['trips'] as const,
  list: (params: GetTripsParams) => [...tripKeys.all, 'list', params] as const,
  detail: (id: TripId) => [...tripKeys.all, 'detail', id] as const,
};

export function useTrips(params: GetTripsParams) {
  const { getTrips } = useUseCases();
  return useQuery({
    queryKey: tripKeys.list(params),
    queryFn: async () => {
      const result = await getTrips(params);
      if (!result.ok) throw result.error;     // Query needs a rejection to mark error
      return result.value;
    },
  });
}
```

Note the boundary translation: the domain returns a `Result`, and the query hook
converts a failure into a rejection because that is Query's protocol. That is the
**only** place a `Failure` is thrown, and it is caught by Query itself — it never
crosses a layer as an exception.

## 8. Zustand store shape

```ts
interface SessionState {
  user: User | null;
  status: 'idle' | 'authenticating' | 'authenticated';
  signIn: (user: User) => void;
  signOut: () => void;
}

export const useSessionStore = create<SessionState>()((set) => ({
  user: null,
  status: 'idle',
  signIn: (user) => set({ user, status: 'authenticated' }),
  signOut: () => set({ user: null, status: 'idle' }),
}));
```

Rules:
- **Always read with a selector.** `useSessionStore((s) => s.user?.name)`
  re-renders on that slice only; `useSessionStore()` re-renders on every change
  and is a performance defect (`performance.md` §1).
- Selectors returning a new object/array need `useShallow` — otherwise the new
  reference re-renders every time.
- Actions live in the store, not scattered `set` calls at call sites.
- Never store what Query owns (§7), and never store what is derivable.

## 9. Other patterns worth reaching for

- **Strategy** — swap an algorithm at runtime (pricing rules, sort orders).
  Replaces growing `if/switch` chains; supports Open/Closed. In TS this is often
  just a `Record<Kind, Handler>` map.
- **Factory** — centralize creation when construction is non-trivial or picks a
  variant (`makePaymentProcessor(method)`).
- **Adapter** — wrap a third-party SDK (analytics, maps, payments, push) behind
  your own interface so the app depends on *your* contract, not the vendor's.
  In React Native this is non-negotiable for anything with a native module: SDK
  swaps and version bumps then touch one file.
- **Decorator** — layer behavior (a `loggingRepository` wrapping a real one)
  without touching the wrapped implementation.
- **Observer** — already idiomatic via store subscriptions and `AppState`/
  `NetInfo` listeners; don't reinvent it with manual listener arrays.

Pick the simplest pattern that removes the coupling or duplication at hand. A
pattern applied where a plain function would do is over-engineering (YAGNI).
