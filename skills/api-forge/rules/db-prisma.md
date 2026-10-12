---
title: Prisma client — singleton + libsql adapter, provider by env
tags: prisma, database, adapters, libsql, multi-provider
---

## Prisma setup

`src/lib/prisma.ts` exports a single `PrismaClient`. Prisma runs with the `driverAdapters` preview
feature. By default it uses the stock client; when `DATABASE_TURSOR` is enabled (and in dev) it
swaps in the libsql adapter (`@prisma/adapter-libsql`) pointed at a Turso/libsql URL. Query logging
is verbose in dev, errors-only otherwise.

```ts
// src/lib/prisma.ts
import { PrismaClient } from "@prisma/client";
import { PrismaLibSQL } from "@prisma/adapter-libsql";
import { env } from "../env";

let prisma: PrismaClient;

if (env.DATABASE_TURSOR && env.NODE_ENV === "dev") {
  const adapter = new PrismaLibSQL({
    url: env.LIBSQL_DATABASE_URL!,
    authToken: env.LIBSQL_DATABASE_TOKEN,
  });
  prisma = new PrismaClient({
    adapter,
    log: env.NODE_ENV === "dev" ? ["query", "info", "warn", "error"] : ["error"],
  });
} else {
  prisma = new PrismaClient({
    log: env.NODE_ENV === "dev" ? ["query", "info", "warn", "error"] : ["error"],
  });
}

export { prisma };
```

```prisma
// prisma/schema.prisma (generator)
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["driverAdapters"]
}
```

Multi-provider note:
- The ORM is **one of two** providers. `env.TYPE_DB` (`sqlite` → Prisma, `firebird` → the second
  provider) decides which repository implementation a factory wires up (see `arch-factories`).
  This Prisma path is the primary one these conventions document.
- The project wraps the client in a small `PrismaDB` helper exposing `getConnection()` (reads) and
  `startTransaction()` (writes); factories use those, not `prisma` directly.

**Don't** import `prisma` outside `lib/prisma.ts`, `repositories/prisma/`, and e2e test helpers —
everything else depends on a repository interface.
