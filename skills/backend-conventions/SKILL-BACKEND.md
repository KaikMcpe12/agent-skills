# SKILL-BACKEND — Backend Conventions (compiled)

Single-file version of the `backend-conventions` skill: all my conventions for building
Node/TypeScript HTTP APIs, reusable across projects. The granular per-topic versions live in
`rules/*.md`; this file compiles them for pasting as context.

> Conventions distilled from a reference implementation (`tny-backend`). They are domain-agnostic:
> entity names in examples are placeholders. Defaults: English identifiers, Bearer-only auth,
> default port 3000, multi-provider DB (SQLite dev / PostgreSQL prod).

---

## Stack

- **Runtime:** Node.js 22 + TypeScript. `tsx` in dev, `tsup` (CJS) for prod. Alias `@/*` → `src/*`.
- **HTTP:** Fastify 5 + `fastify-type-provider-zod` (+ `@fastify/jwt`, `@fastify/cors`, `@fastify/swagger`, `@fastify/swagger-ui`).
- **ORM:** Prisma 7 with driver adapters — `@prisma/adapter-better-sqlite3` (dev), `@prisma/adapter-pg` (prod), chosen at runtime by `DATABASE_PROVIDER`. Client output at `generated/prisma`.
- **Validation:** Zod 4 (request validation + response serialization via the type provider).
- **Auth:** `@fastify/jwt` Bearer-only; `bcryptjs` for hashing.
- **Tests:** Vitest (unit + e2e) + Supertest.

## Folder structure

```
src/
├── @types/            # type augmentation (fastify-jwt payload)
├── env/index.ts       # Zod-validated env loader
├── http/
│   ├── controllers/<domain>/   # routes.ts + one file per action + schemas.ts + *.spec.ts (e2e)
│   ├── middlewares/            # verify-jwt
│   ├── error-handler.ts        # global error handler
│   ├── map-domain-error.ts     # domain error → HTTP
│   └── http-schemas.ts         # shared data/error envelopes
├── lib/prisma.ts      # PrismaClient singleton (adapter by env)
├── repositories/
│   ├── *-repository.ts         # interfaces (contracts)
│   ├── prisma/                 # real impls
│   └── in-memory/              # fakes for unit tests
├── use-cases/
│   ├── <domain>/      # one class per use case (execute) + *.spec.ts (unit)
│   ├── errors/        # domain error classes
│   └── factories/     # make-*-use-cases.ts (manual DI)
├── utils/test/        # e2e helpers (create-and-authenticate, reset-database)
├── app.ts             # builds & exports Fastify instance (no listen)
└── server.ts          # only entry that calls app.listen
```

## Golden rules

1. Never touch Prisma outside `repositories/prisma/` + `lib/prisma.ts` (exception: e2e helpers in `utils/test/`). Use cases/controllers depend on the repository **interface**.
2. Never read `process.env` directly — import `env` from `@/env`.
3. Never `new SomeUseCase()` in a controller — use a `make-*` factory.
4. All exports named. No `export default` in `src/`.
5. Code identifiers English; snake_case only for DB columns. Portuguese only in messages/comments/seed.
6. Response envelope fixed: success `{ data }`, error `{ error: { code, message } }`.
7. Domain errors are classes thrown by use cases, mapped by `mapDomainError`; validation/unknown → global handler.

---

## Architecture

**Flow:** route → controller → factory → use case → repository (interface) → Prisma impl.

- `app.ts` builds and **exports** the Fastify instance (plugins, Zod compilers, swagger, error
  handler, manual route registration) **without** `listen`, so tests import `app`. `server.ts` is
  the only place that calls `app.listen`.
- **Factories** (`make-*-use-cases.ts`) are the only code that knows the concrete repository; they
  wire the use case to `PrismaXRepository` and return the instance. Controllers call the factory.
- **Use case:** one class, single `execute`, typed `*Request`/`*Response`, injected repo
  interface, throws domain errors.

```ts
// app.ts (essence)
export const app = fastify().withTypeProvider<ZodTypeProvider>();
app.setValidatorCompiler(validatorCompiler);
app.setSerializerCompiler(serializerCompiler);
app.register(cors, { origin: env.CORS_ORIGIN === "*" ? true : env.CORS_ORIGIN.split(",").map(o => o.trim()) });
app.register(fastifyJwt, { secret: env.JWT_SECRET, sign: { expiresIn: "1d" } });
app.setErrorHandler(errorHandler);
app.setNotFoundHandler((_r, reply) => reply.status(404).send({ error: { code: "NOT_FOUND", message: "Resource not found" } }));
app.register(categoriesRoutes);
app.register(adminCategoriesRoutes);
```

```ts
// factory
export function makeUpdateCategoryUseCase() {
  return new UpdateCategoryUseCase(new PrismaCategoriesRepository());
}
// use case
export class UpdateCategoryUseCase {
  constructor(private categoriesRepository: CategoriesRepository) {}
  async execute({ id, name, description }: UpdateCategoryUseCaseRequest): Promise<UpdateCategoryUseCaseResponse> {
    const category = await this.categoriesRepository.update(id, { name, description: description ?? null });
    if (!category) throw new ResourceNotFoundError();
    return { category };
  }
}
```

## Environment

`src/env/index.ts` validates `process.env` with Zod (`safeParse`) at import; on failure logs
issues and throws. Import `env` everywhere; never read `process.env` raw. Add new vars to the
schema **and** `.env.example`.

```ts
const envSchema = z.object({
  NODE_ENV: z.enum(["dev", "test", "production"]).default("dev"),
  PORT: z.coerce.number().default(3000),
  DATABASE_PROVIDER: z.enum(["sqlite", "postgres"]).default("sqlite"),
  DATABASE_URL: z.string(),
  JWT_SECRET: z.string(),
  CORS_ORIGIN: z.string().default("*"),
});
```

## Database (Prisma, multi-provider)

- One schema, two providers; client uses driver adapters chosen by `env.DATABASE_PROVIDER`.
  Singleton in `lib/prisma.ts`, query logging only in `dev`.
- Custom client output `generated/prisma` — import models from there.
- Migrations per provider (`prisma/migrations/{sqlite,postgres}`), wired in `prisma.config.ts`.
- `scripts/prisma.mjs` syncs the schema `provider` line to `DATABASE_PROVIDER` before each prisma
  command; always use `npm run db:migrate|generate|seed|reset`.
- Schema (reusable, domain-agnostic): English identifiers, PascalCase models, snake_case columns
  via `@@map("plural")`. No native enums (portability) — enum-like columns are `String` validated
  by Zod. Soft delete via an `active` flag when history matters. Snapshot/freeze values that must
  not change retroactively by copying them onto the row at write time. `@@index` on filtered
  columns; explicit N:N join model + explicit cascade rules.

```ts
// lib/prisma.ts
const adapter = env.DATABASE_PROVIDER === "postgres"
  ? new PrismaPg({ connectionString: env.DATABASE_URL })
  : new PrismaBetterSqlite3({ url: env.DATABASE_URL });
export const prisma = new PrismaClient({ adapter, log: env.NODE_ENV === "dev" ? ["query"] : [] });
```

## Repositories

Interface at root of `repositories/` (ISP — only needed methods). Impls in `prisma/` and
`in-memory/`, both implementing the same interface (LSP). "Not found" is a **return value**
(`null`/`false`), not a thrown error; the use case decides if that's a `ResourceNotFoundError`.
Prisma `P2025` is caught and turned into `null`/`false`.

```ts
export interface CategoriesRepository {
  findMany(): Promise<Category[]>;
  findById(id: number): Promise<Category | null>;
  create(data: CategoryInput): Promise<Category>;
  update(id: number, data: CategoryInput): Promise<Category | null>;
  delete(id: number): Promise<boolean>;
}
```

## HTTP — routes, controllers, validation

- **Route** = `async (app) =>` plugin; `app.withTypeProvider<ZodTypeProvider>()`; per-route
  `schema` with `tags`, `params/body/query`, and a `response` map with one Zod schema per status.
  Auth via `addHook("onRequest", verifyJwt)` (group) or `{ onRequest: [verifyJwt] }` (per route).
  Public vs admin = separate route groups differing only by the auth hook.
- **Controller** = one file per action; type the request with `FastifyRequest<{ Body; Params }>`;
  call the factory; send the envelope; wrap in `try/catch` + `mapDomainError` only when a domain
  error is possible. Status: 201 create, 200 read/update, 204 delete.
- **Validation** = Zod schemas in the domain's `schemas.ts`, types via `z.infer`. `z.coerce.number()`
  for path/query numerics. No `.parse()` in controllers — Fastify validates via the type provider.

```ts
router.put("/admin/categories/:id", {
  schema: { tags, params: categoryIdParamSchema, body: categoryBodySchema,
    response: { 200: dataResponse(categorySchema), 400: validationErrorResponseSchema, 401: errorResponseSchema, 404: errorResponseSchema } },
}, update);

export async function update(request: FastifyRequest<{ Params: CategoryIdParam; Body: CategoryBody }>, reply: FastifyReply) {
  const { id } = request.params; const { name, description } = request.body;
  try {
    const { category } = await makeUpdateCategoryUseCase().execute({ id, name, description: description ?? null });
    return reply.status(200).send({ data: category });
  } catch (error) { return mapDomainError(error, reply); }
}
```

## Error handling & response envelope

Three layers: domain error classes (thrown by use cases) → `mapDomainError` (controller catch →
HTTP) → global `errorHandler` (Zod/validation → 400, serialization → 500, else → 500; the only
logger). Envelope: success `{ data }`; error `{ error: { code, message } }` (+ `fields` for
validation). `error.code` is stable UPPER_SNAKE (`NOT_FOUND`, `VALIDATION_ERROR`, `UNAUTHORIZED`,
`INTERNAL_SERVER_ERROR`, `RESPONSE_SERIALIZATION_ERROR`). Shared shapes + `dataResponse()` helper
in `http/http-schemas.ts`.

## Auth

Bearer-only JWT (`@fastify/jwt`), `expiresIn: 1d`, secret from env. `verify-jwt` middleware
replies 401 in the standard envelope. Payload `{ sub: number }` augmented in
`@types/fastify-jwt.d.ts`; read id via `request.user.sub`. Passwords hashed with `bcryptjs`
(`hash(pw, 6)`), stored as `password_hash`.

## Testing

Vitest, `*.spec.ts`. **Unit**: use case + in-memory repo, `sut` rebuilt in `beforeEach`.
**e2e**: Supertest vs `app.server`, `app.ready()`/`app.close()`, `resetDatabase()` per test.
Isolated SQLite file built by `test/global-setup.ts` (`prisma migrate deploy`, deleted on
teardown); `fileParallelism: false`; test env vars set in `vitest.config.mts`; coverage on
`src/use-cases/**` at 90% thresholds. Helpers in `utils/test/` (`resetDatabase`,
`createAndAuthenticate`) — the only place outside `repositories/`/`lib/` allowed to touch Prisma.

## Tooling

`strict` TS, `NodeNext`, alias `@/*`. ESLint flat config (`typescript-eslint` recommended +
prettier) with `id-denylist` of Portuguese terms and `^_` unused-arg ignore. Prettier:
semicolons, double quotes, trailing commas, width 80, 2 spaces. `tsup` → CJS, node22. Swagger
served from a **static `docs/openapi.yaml`** (source of truth, not generated) at `/docs`; keep it
in sync. Docker: multi-stage node:22-slim, non-root, SQLite on a named volume.

## Anti-patterns

- Prisma in a controller/use case; business rules in a controller/repository.
- `new UseCase()` in a controller (use the factory); `export default`; reading `process.env` raw.
- Sending error responses from a use case (throw a domain error instead); handling validation
  errors in a controller (let them reach the global handler).
- Portuguese identifiers; native Prisma enums; hand-written types parallel to a Zod schema.
- Running raw `prisma` instead of the `db:*` scripts (provider would desync).
