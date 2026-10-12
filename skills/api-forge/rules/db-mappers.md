---
title: Mappers — translate Prisma rows ↔ domain entities
tags: mappers, prisma, entities, boundary
---

## Mappers

A mapper is a small static class in `repositories/prisma/mappers/` that converts between a Prisma
row and a domain entity. It is the **only** place that knows both shapes, keeping
`@prisma/client` types out of use cases, controllers, and entities. One mapper per entity, with
`toDomain(raw)` and `toPrisma(entity)`.

```ts
// repositories/prisma/mappers/prisma-client-mapper.ts
import { Client } from "../../../entities/client";
import { Client as PrismaClient } from "@prisma/client";

export class PrismaClientMapper {
  static toDomain(raw: PrismaClient): Client {
    return new Client(
      {
        name: raw.clientName,
        email: raw.clientEmail,
        status: raw.clientStatus,
        dateadded: raw.clientDateAdded,
        deleted: raw.clientDeleted === 1, // Int 0/1 → boolean
      },
      raw.clientId, // physical id → entity id
    );
  }

  static toPrisma(client: Client) {
    return {
      clientId: client.id,
      clientName: client.name,
      clientEmail: client.email ?? null,
      clientStatus: client.status ?? 1,
      clientDateAdded: client.dateadded,
      clientDeleted: client.isDeleted() ? 1 : 0, // boolean → Int 0/1
    };
  }
}
```

Conventions:
- `toDomain` rebuilds the entity (passing the physical id as the second constructor arg);
  `toPrisma` produces the `data` payload for `create`/`update`.
- Translate the storage encoding here: `Int 0/1` ⇄ `boolean`, camelCase entity props ⇄ the Prisma
  field names, nullable normalization (`?? null`).
- The second provider (Firebird) has its own parallel mappers under
  `repositories/firebird/mappers/` doing the same job for its row shape.

**Don't** do this translation inline in a repository method or a use case — route every
row↔entity conversion through the mapper so the boundary stays in one place.
