---
title: Tooling — lint, format, build, Docker, swagger
tags: tooling, eslint, prettier, tsup, docker, swagger
---

## Tooling & build

- **TypeScript:** `strict`, `NodeNext` module/resolution, `rootDir: src`, alias `@/* → src/*`.
- **ESLint** (`eslint.config.js`, flat config): `@eslint/js` + `typescript-eslint` recommended +
  `eslint-config-prettier`. Custom rules:
  - `id-denylist` (warn) for common Portuguese terms → keeps identifiers English.
  - `no-unused-vars` with `^_` ignore pattern (so `_request` is fine).
- **Prettier:** `semi: true`, `singleQuote: false`, `trailingComma: "all"`, `printWidth: 80`, `tabWidth: 2`.
- **Build:** `tsup` → CJS, `target: node22`, entry `src/server.ts`, externals
  (`better-sqlite3`, `@prisma/adapter-better-sqlite3`).
- **Scripts:** `dev` (`tsx watch`), `build`, `start`, `lint`/`lint:fix`, `format`, `test`/`test:watch`/
  `test:coverage`, and `db:migrate|generate|seed|reset` (all via `scripts/prisma.mjs`).

**Swagger / OpenAPI:** the spec is a **static `docs/openapi.yaml`** served by `@fastify/swagger`
in `mode: "static"` — it is the source of truth, *not* generated from the routes. UI at `/docs`,
JSON at `/docs/json`. Keep `openapi.yaml` in sync when adding/changing endpoints; the per-route
Zod `response` schemas still drive runtime validation/serialization.

**Docker:** multi-stage `node:22-bookworm-slim` (build → runtime), `npx prisma generate && npm run
build`, runs as non-root `node`, SQLite file on a named volume (`docker-compose.yml`) at `/data`.
Entry via `docker-entrypoint.sh`.

> Note: a repo `CLAUDE.md` may predate the current code (it mentions Portuguese field names,
> a refresh cookie, port 3333). **The code is authoritative** — identifiers are English, auth is
> Bearer-only, default port is 3000, and the DB is multi-provider via adapters.
