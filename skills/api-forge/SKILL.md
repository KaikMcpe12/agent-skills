---
name: api-forge
description: My backend conventions for Node/TypeScript HTTP APIs on Fastify 5, Prisma 6 (multi-provider, chosen at runtime by env.TYPE_DB) and Zod 3 — a layered architecture (route → controller → async factory → use case → repository interface → Prisma/in-memory impl) with rich domain entities and mappers, manual dependency injection, Bearer JWT auth (@fastify/jwt + bcrypt), and OpenAPI/Swagger generated from the Zod route schemas via fastify-type-provider-zod. Use when writing, reviewing, or refactoring backend code that should follow these conventions: new routes, controllers, use cases, repositories, entities, DTOs/validation, error handling, Prisma schema, JWT auth, Swagger docs, or Jest/Supertest tests.
license: MIT
metadata:
  author: kaik
  version: "2.0.0"
---

# api-forge

The way I build backend APIs, reusable across projects: a Node + TypeScript HTTP API on
**Fastify 5**, **Prisma 6** (multi-provider, selected at runtime by `env.TYPE_DB`), and
**Zod 3**, organized in clean layers (**controller → use case → repository**) with **rich domain
entities + mappers** and **manual dependency injection** through factories. This skill encodes the
folder structure, the per-layer patterns, naming, validation, error handling, auth, database
setup, the Swagger/OpenAPI flow, and the test strategy so an agent can generate code that looks
and behaves like mine.

> Conventions distilled from a reference implementation. They are domain-agnostic: the entity
> names in examples (`Client`, `Conversation`, …) are placeholders — replace them with yours.
> Teach the clean convention, not the reference's occasional typos (see `naming-conventions`).

## When to Apply

Reference these rules when:
- Adding a feature end-to-end (route → controller → factory → use case → repository → entity).
- Writing or changing a Prisma schema, migration, entity, mapper, or env configuration.
- Wiring validation (DTOs), error handling, auth, or Swagger/OpenAPI documentation.
- Writing unit (in-memory repo) or e2e (Supertest) tests.
- Reviewing/refactoring backend code for consistency with these conventions.

## Stack

- **Runtime:** Node.js + TypeScript (CommonJS, ES2022). `tsx` in dev; `tsup` + `tsc-alias` to build.
- **HTTP:** Fastify 5 + `fastify-type-provider-zod` (`@fastify/jwt`, `@fastify/cors`, `@fastify/swagger`, `@fastify/swagger-ui`, `@fastify/formbody`).
- **ORM:** Prisma 6 with the `driverAdapters` preview. Default path uses the stock client; the libsql adapter (`@prisma/adapter-libsql`) is used when `DATABASE_TURSOR` is on. A second provider (Firebird) implements the same repository interfaces; the active one is chosen by `env.TYPE_DB` (`firebird` | `sqlite`).
- **Validation:** Zod 3, via the Fastify type provider — one schema drives request validation, response serialization, **and** the generated OpenAPI spec.
- **Auth:** `@fastify/jwt`, Bearer-only, `bcrypt` for password hashing.
- **Tests:** Jest (`ts-jest`, ESM preset) + Supertest.

## Folder Structure

```
src/
├── @types/            # type augmentation (fastify-jwt payload)
├── entities/          # rich domain classes — the currency passed between layers
├── env.ts             # Zod-validated env, parsed at import
├── error-handler.ts   # global Fastify error handler
├── http/
│   ├── controllers/<domain>/   # routes.ts + one file per action (*-controller.ts) + *.spec.ts (e2e)
│   ├── dto/<domain>/           # Zod DTOs (*-dto.ts): body / params / query / response
│   └── middleware/             # verify-jwt, verify-user-role
├── lib/               # infra singletons: prisma.ts (also firebird.ts, websocket.ts)
├── repositories/
│   ├── <name>-repository.ts    # INTERFACES — take and return entities
│   ├── prisma/                 # Prisma implementations + prisma/mappers/
│   └── in-memory/              # fakes for unit tests
├── use-cases/
│   ├── <domain>/      # one class per use case (single execute) + *.spec.ts (unit)
│   ├── errors/        # domain error classes
│   └── factories/     # make-*-use-case.ts (async; picks the provider by env.TYPE_DB)
├── app.ts             # builds & exports the Fastify instance (+ swagger); no listen
└── server.ts          # registers cors, then calls app.listen
```

> The reference project also has `gateways/`, `services/`, `http/handlers/` (webhooks), and a
> `socket.io` plugin. Those are domain/integration specifics, **out of scope** for these
> conventions — don't treat them as patterns to replicate.

## Rule Index

Read the individual files in `rules/` for the pattern + a real code example of each.

| Rule | What it covers |
|------|----------------|
| `arch-layers` | route → controller → factory → use case → repository(interface) flow; depend on interfaces; entities are the currency |
| `arch-bootstrap` | `app.ts` (build + register plugins/swagger, no listen) vs `server.ts` (cors + listen) |
| `arch-factories` | async `make-*-use-case.ts`; provider switch on `env.TYPE_DB`; `{ useCase, db }` + commit/rollback |
| `arch-entities` | rich domain entity classes: `Replace` constructor, `randomUUID`, getters/behavior, separate from Prisma |
| `http-routes` | `withTypeProvider`, per-route `schema` (tags, params/body/query/headers, response-per-status), `preHandler` auth |
| `http-controllers` | one file per action; factory call; `instanceof` error mapping; `commitAndClose`/`rollbackAndClose` |
| `http-dto` | Zod DTOs in `dto/<domain>/`; `z.coerce` for params/query; inferred types; `.describe()` for docs |
| `http-errors` | global `errorHandler` + domain error classes; `{ message }` / `{ message, errors }` shapes |
| `db-prisma` | `lib/prisma.ts` singleton; libsql adapter via `DATABASE_TURSOR`; `driverAdapters`; `TYPE_DB` note |
| `db-schema` | PascalCase models, camelCase fields `@map("UPPERCASE")`, Int 0/1 soft-delete, String enum-likes, indexes, join models |
| `db-repository` | interface + prisma impl (via mapper) + in-memory fake; take/return entities; a 2nd provider fits the same interface |
| `db-mappers` | `PrismaXMapper.toDomain` / `toPrisma` — translate Prisma rows ↔ entities at the boundary |
| `env-validation` | `src/env.ts`, `envSchema.parse(process.env)`; never read `process.env` directly |
| `auth-jwt` | Bearer JWT via `@fastify/jwt`; `reply.jwtSign`; `{ sub, role }` payload; `verify-jwt`/`verify-user-role`; bcrypt |
| `swagger-openapi` | `jsonSchemaTransform` → OpenAPI generated from the Zod route schemas; swagger-ui at `/docs` |
| `naming-conventions` | kebab-case files, named exports, English identifiers, relative imports |
| `testing` | Jest; unit (in-memory repo) + e2e (Supertest + `DatabaseCleanerFactory`) |
| `tooling` | tsconfig, `tsx`/`tsup`/`tsc-alias`, scripts; no linter/formatter configured |
| `example-feature` | a full new feature end-to-end, with a build checklist |

## Golden Rules (the non-negotiables)

1. **Entities are the domain currency.** Repositories take and return entity instances;
   `@prisma/client` row types stay behind **mappers** in `repositories/prisma/`. Never leak Prisma
   types into use cases or controllers.
2. **Use cases depend on repository interfaces**, never concrete impls. Business rules live in use
   cases and **throw domain errors** — a use case never sends an HTTP reply.
3. **Factories are the only code that knows the concrete provider.** `make-*-use-case.ts` is
   **async**, switches on `env.TYPE_DB`, and returns the wired use case. Writes return
   `{ useCase, db }` so the controller can `db.commitAndClose()` / `db.rollbackAndClose()`. Never
   `new SomeUseCase()` in a controller.
4. **Controllers map I/O only.** Call the factory, `try/catch`, translate known domain errors with
   `instanceof` → `reply.status(x).send({ message })`, and **re-`throw`** unknown errors to the
   global handler. For writes: commit on success, rollback in the catch.
5. **One Zod schema per route drives everything** — validation, serialization, and the generated
   OpenAPI. DTOs live in `http/dto/<domain>/`. The route `schema` is the validator; don't
   hand-write parallel TypeScript types (`z.infer` instead).
6. **Never read `process.env` directly** — import `env` from `src/env.ts`.
7. **All exports are named** (no `export default` in `src/`). Code identifiers are English;
   Portuguese is allowed only in user-facing message strings, comments, and seed data.
8. **No `{ data }` envelope.** Success responses are plain objects shaped by the route's response
   DTO; errors are `{ message }` (validation also adds `errors`).
