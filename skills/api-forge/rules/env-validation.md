---
title: Zod-validated env, never read process.env directly
tags: env, configuration, zod, validation
---

## Environment configuration

`src/env.ts` loads `.env` with `dotenv` and validates `process.env` with a Zod schema at import
time via `.parse(...)` — which **throws** on the first invalid/missing variable, killing the
process at startup. Everywhere else, import `env` from `src/env.ts`; never read `process.env` raw.

```ts
// src/env.ts
import dotenv from "dotenv";
import { z } from "zod";

dotenv.config();

const envSchema = z.object({
  NODE_ENV: z.enum(["dev", "test", "production"]),
  TYPE_DB: z.enum(["firebird", "sqlite"]).default("firebird"),
  DATABASE_TURSOR: z.coerce.boolean().default(false),
  LIBSQL_DATABASE_URL: z.string().optional(),
  LIBSQL_DATABASE_TOKEN: z.string().optional(),

  PORT: z.coerce.number().default(3333),
  JWT_SECRET_KEY: z.string(),
  ORIGIN: z.string(),
});

export const env = envSchema.parse(process.env);
```

Conventions:
- `z.coerce.number()` / `z.coerce.boolean()` for numeric/boolean vars (env values are strings);
  give non-secret vars a sensible `.default()`, mark optional infra vars `.optional()`.
- `TYPE_DB` selects the active persistence provider (see `arch-factories`); `DATABASE_TURSOR`
  toggles the libsql adapter (see `db-prisma`).
- **When adding a variable:** add it to the schema **and** to `.env.example`.

**Don't** call `process.env.X` anywhere but `src/env.ts` — import `env` instead so every value is
validated and typed.
