---
title: Zod-validated env, never read process.env directly
tags: env, configuration, zod, validation
---

## Environment configuration

`src/env/index.ts` loads `.env` with `dotenv` and validates `process.env` with Zod via
`safeParse`. On failure it logs the issues and **throws**, killing the process at startup.
Everywhere else, import `env` from `@/env` — never read `process.env` raw.

```ts
// src/env/index.ts
import "dotenv/config";
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["dev", "test", "production"]).default("dev"),
  PORT: z.coerce.number().default(3000),
  DATABASE_PROVIDER: z.enum(["sqlite", "postgres"]).default("sqlite"),
  DATABASE_URL: z.string(),
  JWT_SECRET: z.string(),
  CORS_ORIGIN: z.string().default("*"),
});

const _env = envSchema.safeParse(process.env);
if (_env.success === false) {
  console.error("Invalid environment variables", _env.error.issues);
  throw new Error("Invalid environment variables");
}

export const env = _env.data;
```

**When adding a variable:** add it to the Zod schema **and** to `.env.example` (with a comment).
`z.coerce.number()` for numeric vars; sensible `.default()` for anything non-secret.
