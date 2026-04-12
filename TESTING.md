# TESTING.md

Testing rules for all AI agents working in this repository.
Applies to every piece of generated or modified code — no PR is complete without tests.

---

## General Rules

- **Test files live next to the source file.** `foo.service.ts` → `foo.service.spec.ts` (NestJS), `useFoo.ts` → `useFoo.test.ts` (React).
- **One `describe` block per class or function.** Nest inner `describe` blocks per method.
- **Test behavior, not implementation.** Assert on outputs and side effects, not on internal state or which private method was called.
- **Use builder functions for test data.** Define `buildEntity(overrides = {})` helpers at the top of each spec file. Never duplicate full object literals across tests.
- **Always define UUID constants** — never use inline string literals for IDs. Use different constants per role (e.g. `COMPANY_ID`, `OTHER_COMPANY_ID`) to keep cross-tenant tests readable.
- **Each test must be independent.** Call `jest.clearAllMocks()` / `vi.clearAllMocks()` in `afterEach`. Never rely on state left by a previous test.
- **Test the unhappy path as much as the happy path.** Every error branch (`NotFoundException`, `ForbiddenException`, `BadRequestException`, invalid UUID) needs its own `it()`.

---

## NestJS — Jest (`apps/api`)

**Module setup:**
```ts
const module = await Test.createTestingModule({
  providers: [
    SubjectUnderTest,
    { provide: DependencyService, useValue: mockDependency },
    { provide: getRepositoryToken(Entity), useValue: mockRepository },
  ],
}).compile();
```
- Always use `useValue` mocks — never instantiate real services or hit the real database.
- For the repository mock, include all methods the service calls: `create`, `save`, `findOne`, `softDelete`, `createQueryBuilder` (with full builder chain returning `this`).

**What to test per layer:**

| Layer | What to cover |
|---|---|
| **Service** | Happy path for each method; `NotFoundException` when entity not found; `BadRequestException` for invalid UUID inputs; business logic branches (status transitions, conditional fields) |
| **Resolver** | `resolveAuthorizedCompanyId` is called; query/mutation returns correct shape; `ForbiddenException` when cross-tenant access is attempted; `NotFoundException` when entity missing |
| **Mapper** | All scalar fields map correctly; optional relations map when present and are `undefined` when absent |

**Tenant isolation — always required in resolver tests:**
```ts
it('should throw ForbiddenException when resource belongs to another company', async () => {
  mockService.findOne.mockResolvedValue(
    buildEntity({ companyId: OTHER_COMPANY_ID }),
  );
  await expect(resolver.findOne(VALID_UUID, buildCurrentUser())).rejects.toThrow(
    ForbiddenException,
  );
});
```
Every resolver query/mutation that fetches by ID must have a cross-tenant test.

**UUID validation — always required in service tests:**
```ts
it('should throw BadRequestException for invalid UUID', async () => {
  await expect(service.findOne('not-a-uuid')).rejects.toThrow(BadRequestException);
});
```

**Assert both exception class and message:**
```ts
await expect(service.update('bad', {})).rejects.toThrow(NotFoundException);
await expect(service.update('bad', {})).rejects.toThrow('expected message substring');
```

---

## React — Vitest + React Testing Library (`apps/client-app`)

**Hooks — use `renderHook` + `AllMockedProviders`:**
```ts
const { result } = renderHook(() => useMyHook(), {
  wrapper: ({ children }) => AllMockedProviders({ children, mocks: [apolloMock] }),
});
await waitFor(() => expect(result.current.data).toHaveLength(1));
```
- Always `await waitFor(...)` before asserting on async state — never assert immediately after `renderHook`.
- Build Apollo mock helpers at the top of the file: `buildMock(variables, items)`.

**Components/Pages — use accessible queries:**
```ts
screen.getByRole('button', { name: /submit/i })  // ✅
screen.getByTestId('submit-btn')                  // ❌ only if no accessible alternative
```
- Always use `userEvent.setup()` for user interactions, not `fireEvent`.
- Mock external modules (`useNavigate`, toast libraries) with `vi.mock()` at the top.
- Assert what the user sees and experiences — not which function was called internally.

**What to test per layer:**

| Layer | What to cover |
|---|---|
| **Custom hook** | Returns correct initial state; updates state after Apollo response; handles empty results; passes correct variables to the query |
| **Component** | Renders all key UI elements; shows loading/empty states; user interactions trigger correct mutations; validation errors are displayed |
| **Page** | Renders with mocked data; protected actions require correct role; navigation happens after successful mutation |
| **Utils** | Pure function — input/output pairs covering normal cases, edge cases, and invalid inputs |
| **Context** | State changes propagate to consumers; initial state is correct |

**Mock external dependencies minimally:**
```ts
// ✅ Mock only what you need to isolate the unit
vi.mock('@/contexts', () => ({ useAuth: vi.fn() }));

// ❌ Don't mock the entire module if you only need one export
vi.mock('@/contexts');
```

---

## What NOT to Test

- Implementation details (which private method was called, internal variable values).
- Third-party library internals (Apollo, TypeORM query builder internals).
- Snapshot tests — they break on any UI change and communicate nothing about intent.
- The same assertion in multiple tests — extract to a shared `it.each` or helper.
