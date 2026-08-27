# Testing — Unit, Component, Snapshot, E2E

Read this when writing tests, deciding what to test, or wiring test
infrastructure. The point of the clean architecture in `architect.md` is that it
makes this layer cheap: pure domain code is trivial to test, and a component that
takes props instead of reaching into stores renders in three lines.

## 1. The pyramid

Many fast unit tests, fewer component tests, few E2E tests.

```
        /\      E2E         — real device/app (Detox/Maestro), slow, few
       /  \     snapshot    — design-system components, structure lock-in
      /----\    component   — a screen/component in RNTL, user-visible behavior
     /------\   unit        — use cases, repositories, mappers, hooks, pure logic
```

**Target: >90% coverage of business logic** (domain + data + hooks). Do not chase
100% overall — trivial prop pass-throughs and generated code aren't worth it.
Coverage is a tool to find untested logic, not a scoreboard.

Toolchain baseline: **Jest** + **React Native Testing Library** (RNTL) +
**MSW** for network + **Detox or Maestro** for E2E.

## 2. Mock at the boundary, not everywhere

Anything crossing a boundary — network, disk, native modules, the clock,
randomness — is faked so tests are deterministic and fast.

The composition root (`patterns.md` §6) makes this a one-liner: build a container
with fake use cases and render the tree with it. Prefer that over
`jest.mock('../../data/…')` module surgery, which couples tests to file paths and
breaks on every refactor.

```ts
// unit test for a use case — no React involved at all
describe('getTrips', () => {
  it('returns trips on success', async () => {
    const repository: TripRepository = {
      getTrips: jest.fn().mockResolvedValue(ok([tripFixture()])),
      getTripById: jest.fn(),
    };
    const getTrips = makeGetTrips(repository);

    const result = await getTrips({ page: 1 });

    expect(result).toEqual(ok([tripFixture()]));
    expect(repository.getTrips).toHaveBeenCalledWith({ page: 1 });
  });

  it('propagates failure', async () => {
    const repository = {
      getTrips: jest.fn().mockResolvedValue(err({ kind: 'server', message: 'boom' })),
      getTripById: jest.fn(),
    } as unknown as TripRepository;

    const result = await makeGetTrips(repository)({ page: 1 });

    expect(result.ok).toBe(false);
  });
});
```

Structure tests as **Arrange / Act / Assert**. One behavior per test. Name tests
as sentences: `shows the error state and retries when the request fails`.

## 3. Repository & mapper tests

These are where the real bugs live — field renames, null handling, error
classification.

```ts
it('maps the wire shape to the domain entity', () => {
  const dto = tripDtoSchema.parse(tripDtoFixture());   // parse the fixture too:
  expect(toDomain(dto)).toEqual({                      // it proves the fixture is
    id: 'trip-1',                                      // still a valid payload
    origin: 'Tashkent',
    destination: 'Samarkand',
    status: 'active',
    price: money(1500000, 'UZS'),
  });
});

it('returns a network failure when offline', async () => {
  const repository = makeTripRepository({
    remote: { getTrips: jest.fn() },
    network: { isConnected: async () => false },
  });

  const result = await repository.getTrips({ page: 1 });

  expect(result).toEqual(err({ kind: 'network', message: 'no_internet' }));
});
```

Keep DTO fixtures as **real captured payloads**, and parse them through the zod
schema in the test. A hand-written fixture that no longer matches the API passes
forever while production breaks.

## 4. Component tests with RNTL

Test what the **user** sees and does. Query by accessible role/label/text, not by
`testID` where a real query works — that way the test also proves the screen is
accessible.

```tsx
function renderWithProviders(ui: React.ReactElement, container = fakeContainer()) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false, gcTime: 0 } },   // no retries in tests
  });
  return render(
    <ContainerProvider container={container}>
      <QueryClientProvider client={queryClient}>
        <ThemeProvider theme={lightTheme}>{ui}</ThemeProvider>
      </QueryClientProvider>
    </ContainerProvider>,
  );
}

it('shows the error state and retries', async () => {
  const getTrips = jest.fn()
    .mockResolvedValueOnce(err({ kind: 'server', message: 'boom' }))
    .mockResolvedValueOnce(ok([tripFixture()]));

  renderWithProviders(<TripsScreen />, fakeContainer({ getTrips }));

  expect(await screen.findByText('trips.error.title')).toBeOnTheScreen();

  fireEvent.press(screen.getByRole('button', { name: 'common.retry' }));

  expect(await screen.findByText('Tashkent → Samarkand')).toBeOnTheScreen();
});
```

Rules:
- **Never assert on implementation details** — no reaching into state, no
  `UNSAFE_getByType`, no snapshotting an entire screen to "check it works".
- `findBy*` (async) for anything after a fetch; `getBy*` only for what is already
  on screen. A `waitFor` wrapping a bare `getBy*` is usually a `findBy*`.
- Use `userEvent` where the interaction is complex (typing, scrolling);
  `fireEvent.press` is fine for a simple tap.
- Turn **retries off** in the test query client, or a failing-request test waits
  for three back-offs and then times out.
- Test the **four states** of a screen (loading / empty / error / success), not
  just the happy path. That is the single highest-value habit in this file.
- Fake timers for debounce/timeout logic (`jest.useFakeTimers()`), and advance
  them explicitly.

### 4.1 Mocking the network with MSW

Prefer faking at the container/use-case level for component tests. When you do
want to exercise the real data layer (repository + parsing + mapping), MSW gives
you a real HTTP boundary without a real server — and it catches contract drift
that a hand-written mock hides.

```ts
server.use(
  http.get('*/trips', () => HttpResponse.json([tripDtoFixture()])),
);
```

### 4.2 Native modules

Native modules don't exist in Jest. Use the library's official mock
(`jest.setup.js` + `jest.mock('react-native-keychain', …)`), and keep every such
mock in one setup file rather than scattered per test. If a library has no mock,
that is another argument for the Adapter pattern (`patterns.md` §9) — you can
fake **your** interface instead.

## 5. Snapshot tests — narrow and deliberate

Snapshots are useful for **design-system components** with fixed rendering, and
harmful as a substitute for assertions on screens.

```tsx
it('renders the primary variant', () => {
  const tree = render(<AppButton title="Save" variant="primary" onPress={jest.fn()} />);
  expect(tree.toJSON()).toMatchSnapshot();
});
```

- A snapshot nobody reads is a rubber stamp: reviewers must actually diff it, or
  the test only enforces "it changed".
- Never snapshot a whole screen with data — it fails on every copy tweak and
  teaches the team to run `-u` reflexively.
- For real visual regressions, prefer a screenshot-diff tool (Maestro/Detox
  screenshots, or a hosted visual-testing service) over JSON snapshots.

## 6. E2E tests

Full-app flows on a real device/emulator via **Maestro** (fast to write, YAML) or
**Detox** (JS, tighter synchronization). Reserve these for critical end-to-end
journeys (login → create trip → see it in the list), because they're slow and
flakier.

```yaml
# maestro/login.yaml
appId: com.example.app
---
- launchApp
- tapOn: { id: "phone_field" }
- inputText: "+998901234567"
- tapOn: { id: "send_otp" }
- assertVisible: { id: "otp_field" }
```

- Keep the count small and the flows high-value.
- Point them at a **sandbox backend or a mocked one** so they're repeatable; an
  E2E suite that depends on production data is a flaky suite.
- `testID` is the right selector here (stable across copy and locale changes).
- Run them on CI on both platforms, nightly if they're slow — but run them.

## 7. Test data & TDD

- Centralize fixtures/builders (`tripFixture({ status: 'delivered' })`) so tests
  read cleanly and one schema change updates one place. A builder with defaults
  beats a frozen object.
- Prefer **TDD for domain logic and bug fixes**: write the failing test that
  reproduces the bug, then fix until green — it becomes a permanent regression
  guard.
- Make tests independent (no shared mutable state, no ordering dependence). Reset
  mocks, stores, and the query cache between tests.
- `jest.config.js` should fail the run on `console.error` from React (act
  warnings, key warnings) — those are real defects, and silence trains people to
  ignore them.

## What to test vs. skip

**Test:** use cases, repositories (caching/offline branching, failure mapping),
mappers, zod schemas against real payloads, hooks with logic, screen states,
critical flows, accessibility of interactive components.

**Skip:** generated code, trivial prop pass-throughs, third-party internals,
pure layout with no logic (a screenshot diff covers that better than an
assertion-heavy component test), and anything whose test would just re-state the
implementation.
