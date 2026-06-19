---
name: backend-conventions
description: My backend coding conventions for Node/TypeScript APIs — Fastify 5 + Prisma 7 + Zod 4 in a layered architecture (controller → use case → repository) with manual DI, multi-provider database (SQLite/PostgreSQL via driver adapters), Bearer JWT auth, and Vitest/Supertest tests. Use this skill whenever writing, reviewing, or refactoring backend code in any project that should follow these conventions: new routes, use cases, repositories, env handling, validation, error handling, Prisma schema, or tests.
license: MIT
metadata:
  author: kaik
  version: "1.0.0"
---

# Backend Conventions

The way I build backend APIs, reusable in any project: a Node + TypeScript HTTP API on
**Fastify 5**, **Prisma 7** (multi-provider via driver adapters), and **Zod 4**, organized in
clean layers (**controller → use case → repository**) with **manual dependency injection** through
factories. This skill encodes the folder structure, the per-layer patterns, naming,
validation, error handling, auth, database setup, and the test strategy so an agent can
generate code that looks and behaves like mine.

## When to Apply

Reference these rules when:
- Adding a new feature end-to-end (route → controller → use case → repository).
- Writing or changing Prisma schema, migrations, or env configuration.
- Wiring validation, error handling, auth, or the HTTP response envelope.
- Writing unit (in-memory) or e2e (Supertest) tests.
- Reviewing/refactoring backend code for consistency with these conventions.

## Stack

- **Runtime:** Node.js 22 + TypeScript. Source runs via `tsx` (dev), bundled with `tsup` (CJS) for prod. Path alias `@/*` → `src/*`.
- **HTTP:** Fastify 5 + `fastify-type-provider-zod` (`@fastify/jwt`, `@fastify/cors`, `@fastify/swagger`, `@fastify/swagger-ui`).
- **ORM:** Prisma 7 with **driver adapters** — `@prisma/adapter-better-sqlite3` (dev) and `@prisma/adapter-pg` (prod), selected at runtime by `DATABASE_PROVIDER`. Custom client output at `generated/prisma`.
- **Validation:** Zod 4, via the Fastify type provider (validation **and** response serialization).
- **Auth:** `@fastify/jwt`, Bearer-only (no refresh cookie), `bcryptjs` for hashing.
- **Tests:** Vitest (unit + e2e) + Supertest.

## Folder Structure

```
src/
├── @types/            # type augmentation (e.g. fastify-jwt payload)
├── env/               # Zod-validated env loader (index.ts)
├── http/
│   ├── controllers/<domain>/   # routes.ts + one file per action + schemas.ts + *.spec.ts (e2e)
│   ├── middlewares/            # verify-jwt, etc.
│   ├── error-handler.ts        # global Fastify error handler
│   ├── map-domain-error.ts     # domain error → HTTP, called from controller catch
│   └── http-schemas.ts         # shared response shapes (data/error envelopes)
├── lib/               # infra singletons (prisma.ts)
├── repositories/
│   ├── *-repository.ts         # INTERFACES (contracts)
│   ├── prisma/                 # real Prisma implementations
│   └── in-memory/              # fakes for unit tests
├── use-cases/
│   ├── <domain>/      # one class per use case (single execute method) + *.spec.ts (unit)
│   ├── errors/        # domain error classes
│   └── factories/     # make-*-use-cases.ts (manual DI)
├── utils/test/        # e2e helpers (create-and-authenticate, reset-database)
├── app.ts             # builds & exports Fastify instance (no listen)
└── server.ts          # only entry that calls app.listen
```

## Rule Index

Read the individual files in `rules/` for the pattern + real code example of each.

| Rule | What it covers |
|------|----------------|
| `arch-layers` | controller → use case → repository flow; depend on interfaces, not impls |
| `arch-bootstrap` | `app.ts` (build/export) vs `server.ts` (listen) split |
| `arch-factories` | manual DI via `make-*-use-cases.ts`; never `new UseCase()` in a controller |
| `env-validation` | Zod-validated env in `src/env`; never read `process.env` directly |
| `db-prisma-setup` | multi-provider adapters, custom client output, schema-sync script |
| `db-schema` | English identifiers, snake_case `@@map`, enum-as-String validated by Zod |
| `db-repository` | interface + prisma + in-memory; ISP, P2025 handling, return shapes |
| `http-routes` | `withTypeProvider`, per-route `schema`, `tags`, `response` per status, auth hook |
| `http-controllers` | one file per action, typed `FastifyRequest`, factory call, `mapDomainError` |
| `http-validation` | Zod schemas in `schemas.ts`, `z.coerce`, inferred types |
| `http-errors` | global error handler + `mapDomainError` + domain error classes |
| `http-response-envelope` | `{ data }` success / `{ error: { code, message } }` failure shapes |
| `auth-jwt` | Bearer-only JWT, `verify-jwt` middleware, `@fastify/jwt` payload augmentation |
| `naming-conventions` | kebab-case files, named exports, English identifiers, snake_case DB |
| `testing` | unit (in-memory + `sut`) and e2e (Supertest + `resetDatabase`) patterns |
| `tooling` | ESLint id-denylist, Prettier, tsconfig, tsup, Docker |
| `example-feature` | full new feature end-to-end (Product domain), with a build checklist |

## Golden Rules (the non-negotiables)

1. **Never touch Prisma outside `repositories/prisma/` and `lib/prisma.ts`** (the only exception is e2e test helpers in `utils/test/`). Use cases and controllers depend on the repository **interface**.
2. **Never read `process.env` directly** — import `env` from `@/env`.
3. **Never instantiate a use case with `new` in a controller** — use a `make-*` factory.
4. **All exports are named.** No `export default` anywhere in `src/`.
5. **Code identifiers are English; snake_case only for DB columns.** Portuguese is allowed only in user-facing message strings, comments, and seed data.
6. **Response envelope is fixed:** success → `{ data }`, error → `{ error: { code, message } }`.
7. **Domain errors are classes**, thrown by use cases, mapped to HTTP by `mapDomainError`; validation/unknown errors fall through to the global handler.

## Full Compiled Document

For everything in one file (e.g. to paste as context): `../../../SKILL-BACKEND.md` — or the
copy shipped beside this skill. Each `rules/*.md` is self-contained with a code example.
