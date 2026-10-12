---
title: Prisma schema conventions
tags: prisma, schema, modeling, mapping
---

## Schema conventions

Reusable modeling conventions — the `Client`/`Conversation` models below are illustrative
placeholders; replace them with your own.

- **PascalCase models, camelCase fields**, each mapped to an **UPPERCASE** physical column with
  `@map("COLUMNNAME")`, and the table mapped with `@@map("UPPERCASE_PLURAL")`. (The uppercase
  physical names keep the schema portable to the second provider, Firebird.)
- **Soft delete** via an `Int` flag `xDeleted @default(0)` (`0` = live, `1` = deleted) rather than
  deleting rows; the entity exposes it as a boolean (`isDeleted()`). **Status** is likewise an
  `Int` (`0`/`1`).
- **Enum-like columns are `String`** with a documented closed value set (e.g. `status`,
  `priority`), enforced by `z.enum([...])` at the app layer (see `http-dto`) — no native Prisma
  enums.
- **Ids are `String`**, either assigned by the app (`randomUUID()` in the entity) or
  `@default(uuid())`.
- `@@index([...])` on columns you filter/sort by; composite `@@unique([...])` for natural keys;
  N:N via an explicit join model.

```prisma
model Client {
  clientId      String    @id @map("CLIENTID")
  clientName    String    @map("CLIENTNAME")
  clientEmail   String?   @unique @map("CLIENTEMAIL")
  clientStatus  Int       @default(1) @map("CLIENTSTATUS") // 0 inactive, 1 active
  clientDateAdded DateTime @default(now()) @map("CLIENTDATEADDED")
  clientDeleted Int       @default(0) @map("CLIENTDELETED") // 0 live, 1 deleted

  conversations Conversation[]

  @@map("CLIENTS")
}

model Conversation {
  conversationId String   @id @default(uuid()) @map("CONVERSATIONID")
  clientId       String   @map("CLIENTID")
  companyId      String   @map("COMPANYID")
  status         String   @default("waiting") @map("STATUS")   // enum-like, validated by Zod
  priority       String   @default("normal") @map("PRIORITY")
  createdAt      DateTime @default(now()) @map("CREATED_AT")
  updatedAt      DateTime @updatedAt @map("UPDATED_AT")

  client Client @relation(fields: [clientId], references: [clientId])

  @@unique([clientId, companyId])
  @@index([companyId])
  @@index([status])
  @@map("CONVERSATIONS")
}
```

**Don't** use native Prisma enums or `Boolean` soft-delete columns — the physical layer is shared
with the second provider, so enum-likes are `String` and flags are `Int 0/1`.
