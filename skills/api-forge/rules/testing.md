---
title: Testing — unit (in-memory) + e2e (Supertest)
tags: testing, jest, supertest, in-memory
---

## Test strategy

**Jest** with `ts-jest` (ESM preset), everything as `*.spec.ts`, two levels:

- **Unit** (`use-cases/<domain>/*.spec.ts`): the use case against the **in-memory repository** (a
  real fake, not a mock lib). Rebuild the repos + use case in `beforeEach`.
- **e2e** (`http/controllers/<domain>/*.spec.ts`): Supertest against `app.server`, with
  `app.ready()`/`app.close()` and a `DatabaseCleaner` reset per test.

```ts
// unit — use-cases/clients/create-client.spec.ts
describe("Create Client Use Case", () => {
  let clientRepository: InMemoryClientRepository;
  let sut: CreateClientUseCase;

  beforeEach(() => {
    clientRepository = new InMemoryClientRepository();
    sut = new CreateClientUseCase(clientRepository /*, …other in-memory repos */);
  });

  it("should not allow a duplicate email", async () => {
    await sut.execute({ clients: [{ name: "Alice", email: "a@ex.com" }] });
    await expect(() =>
      sut.execute({ clients: [{ name: "Bob", email: "a@ex.com" }] }),
    ).rejects.toBeInstanceOf(ClientAlreadyExistsError);
  });
});
```

```ts
// e2e — http/controllers/clients/create-client-controller.spec.ts
describe("Create Client (e2e)", () => {
  let cleanerDb: DatabaseCleaner;

  beforeAll(async () => {
    cleanerDb =
      env.TYPE_DB === "firebird"
        ? await DatabaseCleanerFactory.createFirebird()
        : await DatabaseCleanerFactory.createPrisma();
    await app.ready();
  });
  beforeEach(async () => { await cleanerDb.cleanDatabase(); });
  afterAll(async () => { await app.close(); });

  it("should create a client", async () => {
    const { body } = await request(app.server).post("/users").send({ name: "U", email: "u@ex.com", password: "123456" });
    const response = await request(app.server)
      .post("/client")
      .set("Authorization", `Bearer ${body.token}`)
      .send({ clients: [{ name: "John", email: "john@ex.com" }] });

    expect(response.statusCode).toEqual(201);
  });
});
```

Infrastructure:
- `jest.config.js`: `ts-jest/presets/default-esm`, `testEnvironment: "node"`,
  `moduleNameMapper` maps `@/(.*)` → `<rootDir>/src/$1`.
- The e2e `DatabaseCleaner` is built by `DatabaseCleanerFactory` for the active provider
  (`createPrisma()` / `createFirebird()`), so the same specs run against either backend.
- Scripts: `npm test` → `jest src/use-cases` (unit), `npm run test:e2e` → `jest src/http` (e2e).

**Don't** reach into `prisma` from a use-case spec — assert against the in-memory repository.
