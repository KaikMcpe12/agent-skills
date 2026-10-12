---
title: DTOs — Zod schemas per domain
tags: zod, validation, dto, type-inference, swagger
---

## DTOs (validation + docs)

Validation and response shapes live in `src/http/dto/<domain>/<name>-dto.ts` as Zod schemas, one
responsibility per export: `body`, `params`, `querystring`, and `response`. Export the schema as a
`camelCaseDTO` const and its inferred type as `PascalCaseDTO` via `z.infer`. The route references
the schema in its `schema` block; the controller imports the inferred type (never a hand-written
parallel interface).

```ts
// http/dto/user/create-user-dto.ts — body
import z from "zod";

export const createUserDTO = z.object({
  name: z.string(),
  email: z.string().email(),
  password: z.string(),
  phoneNumber: z.string().optional(),
  companyId: z.string().optional(),
});

export type CreateUserDTO = z.infer<typeof createUserDTO>;
```

```ts
// http/dto/conversation/list-conversations-by-company-dto.ts — querystring
import { z } from "zod";

export const listConversationsByCompanyQueryDTO = z.object({
  companyId: z.string().uuid(),
  page: z.coerce.number().int().positive().optional().default(1),     // query arrives as string
  pageSize: z.coerce.number().int().positive().optional().default(10),
  status: z.enum(["waiting", "active", "paused", "closed", "archived"]).optional(),
});

export type ListConversationsByCompanyQueryDTO = z.infer<typeof listConversationsByCompanyQueryDTO>;
```

```ts
// http/dto/conversation/list-conversations-dto.ts — response (used in the route's response map)
export const listConversationsByCompanyResponseDTO = z.object({
  conversations: z.array(conversationSchema),
  pagination: z.object({ page: z.number(), limit: z.number(), total: z.number() }),
});
```

Conventions:
- `z.coerce.number()` / `z.coerce.date()` for path & query values (they arrive as strings); plain
  `z.number()` for JSON bodies.
- Enum-like values use `z.enum([...])` — this is where the closed value set for a `String` DB
  column is enforced (see `db-schema`).
- Add `.describe(...)` to fields that need documentation; it flows into the generated OpenAPI
  (see `swagger-openapi`).
- Always `z.infer` the type from the schema — never hand-write a parallel interface.

**Don't** scatter inline body/response schemas across routes when they're reused; give them a DTO
file. Keep one schema as the single source of truth for a shape.
