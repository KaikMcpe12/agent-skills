---
title: Tooling — TypeScript, dev/build, scripts
tags: tooling, typescript, tsup, tsx, jest
---

## Tooling & build

- **TypeScript** (`tsconfig.json`): `strict`, `module: commonjs`, `target: es2022`,
  `lib: ["es2023"]`, `isolatedModules`, `esModuleInterop`, `skipLibCheck`,
  `moduleResolution: node`, `outDir: ./dist`.
- **Dev:** `tsx watch src/server.ts` (`npm run dev`).
- **Build:** `tsup` bundles `src/server.ts`, then `tsc-alias` rewrites path aliases; Prisma client
  is generated first and the Prisma/libsql packages are kept external. (`npm run build`; a
  `vercel-build` variant targets CJS only.)
- **Run:** `node dist/server.js` (`npm start`).
- **Tests:** Jest (`ts-jest` ESM) — see `testing`. `npm test` (unit), `test:e2e`, `test:coverage`.
- **DB scripts:** `db:reset` (`prisma migrate reset` + seed), `db:seed` (`tsx prisma/seed.ts`).
- **Path alias:** `@/ → src/*` is configured for Jest (`moduleNameMapper`) and the build
  (`tsc-alias`), but source imports are overwhelmingly relative — see `naming-conventions`.

**Swagger / OpenAPI** is generated from the Zod route schemas via `jsonSchemaTransform`, served at
`/docs` — see `swagger-openapi` (not a static yaml file).

> No ESLint, Prettier, or Biome config is present in the project. Don't invent lint/format rules or
> claim a formatter; match the existing style of the file you're editing (2-space indent, single
> quotes, mostly no semicolons in app code).
