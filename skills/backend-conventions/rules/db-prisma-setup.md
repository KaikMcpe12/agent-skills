---
title: Multi-provider Prisma via driver adapters
tags: prisma, database, adapters, sqlite, postgres
---

## Prisma setup (SQLite dev / PostgreSQL prod)

A single Prisma schema serves two providers. The client uses **driver adapters** chosen at
runtime by `env.DATABASE_PROVIDER`. Client output is a custom path (`generated/prisma`), so
import models from there, not `@prisma/client`.

```ts
// src/lib/prisma.ts — singleton, adapter picked by env
import { PrismaBetterSqlite3 } from "@prisma/adapter-better-sqlite3";
import { PrismaPg } from "@prisma/adapter-pg";
import { env } from "@/env";
import { PrismaClient } from "../../generated/prisma";

const adapter =
  env.DATABASE_PROVIDER === "postgres"
    ? new PrismaPg({ connectionString: env.DATABASE_URL })
    : new PrismaBetterSqlite3({ url: env.DATABASE_URL });

export const prisma = new PrismaClient({
  adapter,
  log: env.NODE_ENV === "dev" ? ["query"] : [],
});
```

Because migrations are dialect-specific, they live in one folder per provider
(`prisma/migrations/{sqlite,postgres}`), wired in `prisma.config.ts`. A small script syncs the
schema `provider` line to `DATABASE_PROVIDER` before running any prisma command:

```js
// scripts/prisma.mjs (used by npm run db:* scripts)
const provider = process.env.DATABASE_PROVIDER === "postgres" ? "postgresql" : "sqlite";
const schema = readFileSync("prisma/schema.prisma", "utf8")
  .replace(/provider = "(sqlite|postgresql)"/, `provider = "${provider}"`);
writeFileSync("prisma/schema.prisma", schema);
execSync(`npx prisma ${process.argv.slice(2).join(" ")}`, { stdio: "inherit" });
```

Use `npm run db:migrate`, `db:generate`, `db:seed`, `db:reset` — not raw `prisma` — so the
provider stays in sync. Keep queries portable; avoid DB-specific features.
