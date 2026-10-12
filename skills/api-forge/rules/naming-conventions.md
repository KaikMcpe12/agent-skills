---
title: Naming and export conventions
tags: naming, conventions, exports, imports, casing
---

## Naming conventions

- **Files:** kebab-case, named by role/action with a consistent suffix:
  - `*-controller.ts` (HTTP handler), `*-use-case.ts` (use case), `make-*-use-case.ts` (factory),
    `*-dto.ts` (Zod DTO), `*-repository.ts` (interface),
    `prisma-*-repository.ts` / `in-memory-*-repository.ts` / `firebird-*-repository.ts` (impls),
    `*-error.ts` (domain error), `*.spec.ts` (test).
  - Entities are `src/entities/<name>.ts`; the domain folder under `controllers/`, `use-cases/`,
    and `dto/` is the plural/area name (`clients/`, `conversations/`).
- **Exports are always named.** No `export default` in `src/`.
- **Casing:** classes PascalCase (`CreateClientUseCase`, `Client`, `PrismaClientMapper`);
  functions/handlers/consts camelCase (`createClientDTO`, `verifyJWT`). Prisma **model** names are
  PascalCase, **fields** camelCase, physical columns UPPERCASE (see `db-schema`).
- **Identifiers are English.** Portuguese is allowed only in user-facing message strings, comments,
  and seed/example data.
- **Imports are relative** (`../../entities/client`). A `@/ → src/*` alias exists in the Jest and
  build config but is rarely used — match the file you're editing (relative) rather than
  introducing the alias.
- **Use-case shape:** one class, a single `execute(...)`, a typed `*Request`/`*Response` pair.

> The reference project has a few historical typos and off-pattern names
> (`client-alredy-exists-erros.ts`, `ClientAlredyExistsError`, `list-clients-by-compan.ts`,
> `login-use-dto.ts`, `hellor-world.ts`, a `message.repository.ts` using a dot) and a couple of
> read routes that reply `201` instead of `200`. **Follow the clean convention shown here** —
> correct English spelling, the `-repository.ts` suffix, accurate status codes — don't reproduce
> the typos.
