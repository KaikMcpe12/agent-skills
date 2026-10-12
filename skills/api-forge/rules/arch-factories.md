---
title: Factories — async DI that picks the provider by env.TYPE_DB
tags: architecture, dependency-injection, factories, multi-provider
---

## Factories (manual dependency injection + provider selection)

The factory is the **only** place that knows the concrete repository implementation. Each
`make-*-use-case.ts` is **async**: it opens a DB handle, picks the provider by `env.TYPE_DB`
(`sqlite` → Prisma, anything else → the second provider), wires the repositories into the use
case, and returns it. One factory file per use case.

**Writes** open a transaction handle and return `{ useCase, db }` so the controller can
`commitAndClose()` / `rollbackAndClose()`:

```ts
// use-cases/factories/make-create-client-use-case.ts
import { env } from "../../env";
import { PrismaDB } from "../../lib/prisma";
import { FirebirdDB } from "../../lib/firebird";
import { PrismaClientRepository } from "../../repositories/prisma/prisma-client-repository";
import { FirebirdClientRepository } from "../../repositories/firebird/firebird-client-repository";
import { FirebirdService } from "../../services/firebird-service";
import { CreateClientUseCase } from "../clients/create-client-use-case";

export async function makeCreateClientUseCase() {
  let db;
  let clientRepository;

  if (env.TYPE_DB === "sqlite") {
    db = await PrismaDB.startTransaction();
    clientRepository = new PrismaClientRepository(db.prisma);
  } else {
    db = await FirebirdDB.startTransaction();
    clientRepository = new FirebirdClientRepository(new FirebirdService(db));
  }

  const createClientUseCase = new CreateClientUseCase(clientRepository);
  return { createClientUseCase, db };
}
```

**Reads** use `getConnection()` instead of a transaction. A read factory may return the use case
directly (no `db` to commit) or `{ useCase, db }` — match the surrounding code:

```ts
// use-cases/factories/make-get-conversation-by-id-use-case.ts (read)
export async function makeGetConversationByIdUseCase() {
  let conversationRepository;
  if (env.TYPE_DB === "sqlite") {
    const db = await PrismaDB.getConnection();
    conversationRepository = new PrismaConversationRepository(db);
  } else {
    const db = await FirebirdDB.getConnection();
    conversationRepository = new FirebirdConversationRepository(new FirebirdService(db));
  }
  return new GetConversationByIdUseCase(conversationRepository, /* … */);
}
```

**Why:** controllers and use cases stay agnostic of the provider and of persistence — swapping
Prisma for the second provider (or an in-memory fake in tests) touches only the factory/constructor.

**Don't** call `new SomeUseCase(...)` in a controller — always go through its factory.

> Note: `startTransaction()` returns a handle whose `commitAndClose()`/`rollbackAndClose()` manage
> the connection lifecycle and signal success/failure; on the Prisma path they are **not** a real
> `prisma.$transaction` (each repository call commits on its own). Treat them as the lifecycle
> contract the controllers expect — see `http-controllers`.
