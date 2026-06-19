---
title: Testing — unit (in-memory) + e2e (Supertest)
tags: testing, vitest, supertest, in-memory
---

## Test strategy

Vitest, two levels, both as `*.spec.ts`:

- **Unit** (`use-cases/<domain>/*.spec.ts`): test the use case against the **in-memory repository**
  (real fake, not a mock lib). Convention: module-level `sut` + repository, rebuilt in `beforeEach`.
- **e2e** (`http/controllers/<domain>/*.spec.ts`): Supertest against `app.server`, with
  `app.ready()`/`app.close()` and `resetDatabase()` in `beforeEach`.

```ts
// unit — use-cases/categories/create-category.spec.ts
let categoriesRepository: InMemoryCategoriesRepository;
let sut: CreateCategoryUseCase;

describe("Create Category Use Case", () => {
  beforeEach(() => {
    categoriesRepository = new InMemoryCategoriesRepository();
    sut = new CreateCategoryUseCase(categoriesRepository);
  });

  it("should default description to null when omitted", async () => {
    const { category } = await sut.execute({ name: "Bermudas" });
    expect(category.description).toBeNull();
  });
});
```

```ts
// e2e — http/controllers/categories/categories.spec.ts
describe("Categories (public) e2e", () => {
  beforeAll(async () => { await app.ready(); });
  afterAll(async () => { await app.close(); });
  beforeEach(async () => { await resetDatabase(); });

  it("should return 404 for a missing category", async () => {
    const response = await request(app.server).get("/categories/999");
    expect(response.statusCode).toBe(404);
    expect(response.body.error.code).toBe("NOT_FOUND");
  });
});
```

Infrastructure (`vitest.config.mts` + `test/global-setup.ts`):
- e2e runs against an isolated SQLite file (`prisma/test.db`) created via `prisma migrate deploy`
  in a global setup and deleted on teardown.
- `fileParallelism: false` — test files share the DB, so they run sequentially.
- Test env vars (`DATABASE_URL`, `NODE_ENV=test`, `JWT_SECRET`, `CORS_ORIGIN`) are set in the
  Vitest config.
- Coverage is measured on `src/use-cases/**` with **90%** thresholds (lines/functions/branches/statements).
- Helpers in `utils/test/`: `resetDatabase()` (FK-safe `deleteMany` order) and
  `createAndAuthenticate(app)` (creates an admin + returns a signed Bearer token).
- e2e helpers are the **only** place outside `repositories/`/`lib/` allowed to touch `prisma` directly.
