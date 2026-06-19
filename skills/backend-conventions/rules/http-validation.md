---
title: Validation — Zod schemas per domain
tags: zod, validation, schemas, type-inference
---

## Input validation

Each controller domain has a `schemas.ts` exporting the Zod schemas (params/body/response shapes)
and their inferred TypeScript types. The route references them in its `schema`; the controller
imports the inferred type to type its `FastifyRequest`. Validation happens at the Fastify layer
(via the type provider), so controllers don't call `.parse()`.

```ts
// http/controllers/admin-categories/schemas.ts
import { z } from "zod";

export const categoryIdParamSchema = z.object({
  id: z.coerce.number().int().positive(), // coerce: params/query arrive as strings
});

export const categoryBodySchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(255).nullish(), // optional + nullable
});

export type CategoryIdParam = z.infer<typeof categoryIdParamSchema>;
export type CategoryBody = z.infer<typeof categoryBodySchema>;
```

Conventions:
- `z.coerce.number()` for path/query numerics; plain `z.number()` only for JSON bodies.
- Response/entity schemas (e.g. `categorySchema`) also live in `schemas.ts` and are reused in the
  route `response` map and shared across public/admin route groups.
- Enum-like DB columns get their closed value set enforced here with `z.enum([...])`.
- Always infer the type from the schema (`z.infer`) — never hand-write a parallel interface.
