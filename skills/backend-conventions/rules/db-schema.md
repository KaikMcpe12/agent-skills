---
title: Prisma schema conventions
tags: prisma, schema, snake-case, modeling
---

## Schema conventions

Reusable modeling conventions — they apply to any domain, not a specific one. The `Entity`/`Child`
models below are illustrative placeholders; replace them with your own.

- **Identifiers in English, PascalCase models, snake_case columns** mapped with
  `@@map("plural_table")`. The DB is snake_case; code reads the fields as declared.
- **No native enums** (keeps schema portable across SQLite/PostgreSQL). Model enum-like columns as
  `String` with a documented closed value set, **enforced by Zod at the app layer** (not the DB).
- **Soft delete** via an `active Boolean @default(true)` flag instead of deleting rows, when
  history/audit matters for that entity.
- **Snapshot/freeze** any value that must not change retroactively by copying it onto the row at
  write time (rather than resolving it later through a relation).
- **`@@index([...])`** on columns you filter or sort by.
- **Explicit N:N** via a join model with a composite `@@id`; set **explicit cascade rules**
  (`onDelete: Cascade` for owned children, `Restrict`/`SetNull` where history matters).
- Surrogate `Int @id @default(autoincrement())` PKs; `created_at DateTime @default(now())` for
  audit timestamps; `@unique` on natural keys.

```prisma
model Entity {
  id             Int      @id @default(autoincrement())
  code           String   @unique          // natural key
  name           String
  status         String   @default("new")  // enum-like: validated by Zod, not a DB enum
  active         Boolean  @default(true)    // soft delete
  created_at     DateTime @default(now())

  children       Child[]

  @@index([name])
  @@map("entities")
}

model Child {
  id        Int    @id @default(autoincrement())
  entity_id Int
  // snapshot fields: copied at write time so later changes to Entity don't rewrite this row
  name_at_creation String

  entity Entity @relation(fields: [entity_id], references: [id], onDelete: Cascade)

  @@map("children")
}
```
