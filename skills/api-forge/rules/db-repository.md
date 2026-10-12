---
title: Repository — interface + Prisma impl + in-memory fake
tags: repository, prisma, interface-segregation, entities, testing
---

## Repository pattern

The contract is an **interface** at the root of `repositories/`, exposing only the methods the use
cases need (ISP) and speaking in **entities** (not Prisma rows). Two implementations:
`repositories/prisma/` (real, translating via a mapper) and `repositories/in-memory/` (a fake for
unit tests). A second provider (Firebird) implements the very same interface, so a use case can't
tell them apart.

```ts
// repositories/client-repository.ts — contract, in entities
import { Client } from "../entities/client";

export interface ClientRepository {
  findByEmail(email: string): Promise<Client | null>;
  findById(id: string): Promise<Client | null>;
  create(data: Client): Promise<Client>;
  update(data: Client): Promise<Client>;
  delete(id: string): Promise<void>;
}
```

```ts
// repositories/prisma/prisma-client-repository.ts — maps rows ↔ entities
import { Prisma, PrismaClient } from "@prisma/client";
import { ClientRepository } from "../client-repository";
import { Client } from "../../entities/client";
import { PrismaClientMapper } from "./mappers/prisma-client-mapper";

export class PrismaClientRepository implements ClientRepository {
  constructor(private prisma: PrismaClient | Prisma.TransactionClient) {}

  async findByEmail(email: string): Promise<Client | null> {
    const client = await this.prisma.client.findUnique({ where: { clientEmail: email } });
    return client ? PrismaClientMapper.toDomain(client) : null;
  }

  async create(data: Client): Promise<Client> {
    await this.prisma.client.create({ data: PrismaClientMapper.toPrisma(data) });
    return data;
  }
  // update / delete: soft-delete writes set clientDeleted = 1 (see db-schema)
}
```

```ts
// repositories/in-memory/in-memory-client-repository.ts — a real fake, not a mock
import { ClientRepository } from "../client-repository";
import { Client } from "../../entities/client";

export class InMemoryClientRepository implements ClientRepository {
  private clients = new Map<string, Client>();

  async findByEmail(email: string): Promise<Client | null> {
    for (const client of this.clients.values()) if (client.email === email) return client;
    return null;
  }
  async findById(id: string): Promise<Client | null> { return this.clients.get(id) ?? null; }
  async create(data: Client): Promise<Client> { this.clients.set(data.id, data); return data; }
  async update(data: Client): Promise<Client> { this.clients.set(data.id, data); return data; }
  async delete(id: string): Promise<void> { this.clients.get(id)?.delete(); }
}
```

Conventions:
- **"Not found" is a return value** (`null`), not a thrown error — the use case decides whether
  that's a `ResourceNotFoundError`.
- The Prisma impl's constructor accepts `PrismaClient | Prisma.TransactionClient` so a write
  factory can hand it the transaction handle (`db.prisma`).
- The in-memory fake mirrors behavior exactly (same null semantics, respects `isDeleted()`), using
  a `Map` keyed by entity id.

**Don't** return `@prisma/client` row types from a repository — always map to an entity
(`db-mappers`).
