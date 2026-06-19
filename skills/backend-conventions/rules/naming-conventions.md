---
title: Naming and export conventions
tags: naming, conventions, exports, casing
---

## Naming conventions

- **Files:** kebab-case, named by role/action — `update-category.ts`, `make-category-use-cases.ts`,
  `prisma-categories-repository.ts`, `resource-not-found-error.ts`, `verify-jwt.ts`,
  `categories.spec.ts`.
- **Controllers are named by action** (`create.ts`, `update.ts`, `delete.ts` → exported `remove`,
  `get.ts`, `list.ts`). No `.controller.ts` / `.service.ts` suffixes.
- **Suffixes:** `*-repository.ts` (interface), `prisma-*-repository.ts` / `in-memory-*-repository.ts`
  (impls), `make-*-use-cases.ts` (factory), `*-error.ts` (domain error), `*.spec.ts` (test).
- **Exports are always named.** No `export default` in `src/`.
- **Casing:** classes PascalCase; functions/handlers camelCase; **DB columns snake_case**
  (`password_hash`, `created_at`, `product_id`).
- **All code identifiers are English.** Portuguese is allowed only in user-facing message strings,
  comments, and seed/example data — enforced by an ESLint `id-denylist` of common PT terms.
- **Imports** use the `@/` alias (`@/use-cases/...`), except the generated Prisma client which is
  imported by relative path from `generated/prisma`.
- **Use case shape:** one class per use case, single `execute(...)`, a typed
  `*UseCaseRequest`/`*UseCaseResponse` pair.
