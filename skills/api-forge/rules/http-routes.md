---
title: Routes — typed plugin with per-route Zod schema
tags: fastify, routes, zod, swagger, auth
---

## Route pattern

Each domain has a `routes.ts` exporting an `async (app: FastifyInstance)` plugin. Grab a typed
router with `app.withTypeProvider<ZodTypeProvider>()` and declare, per route: `tags`, the input
schemas (`params` / `body` / `querystring` / `headers`), and a `response` map with **one Zod schema
per status code**. Handlers live in separate per-action files. Protected routes get
`preHandler: [verifyJWT]`.

```ts
// http/controllers/conversations/routes.ts
import { FastifyInstance } from "fastify";
import { ZodTypeProvider } from "fastify-type-provider-zod";
import { z } from "zod";
import { verifyJWT } from "../../middleware/verify-jwt";
import { listConversationsByCompanyQueryDTO } from "../../dto/conversation/list-conversations-by-company-dto";
import { listConversationsByCompanyResponseDTO } from "../../dto/conversation/list-conversations-dto";
import { listConversationsByCompany } from "./list-conversations-by-company-controller";

export async function conversationsRoutes(app: FastifyInstance) {
  app.withTypeProvider<ZodTypeProvider>().get("/conversations", {
    preHandler: [verifyJWT],
    schema: {
      tags: ["Conversations"],
      querystring: listConversationsByCompanyQueryDTO,
      headers: z.object({
        authorization: z.string().describe("Bearer token (required)").optional(),
      }),
      response: {
        200: listConversationsByCompanyResponseDTO,
        401: z.object({ message: z.string() }),
      },
    },
  }, listConversationsByCompany);
}
```

Conventions:
- Declare **every** relevant status (success + each error) with its shape — the response schema is
  both the serializer and the OpenAPI contract (see `swagger-openapi`).
- Error statuses use an inline `z.object({ message: z.string() })`; success statuses reference a
  response DTO from `http/dto/<domain>/` (see `http-dto`).
- Auth is per route via `preHandler: [verifyJWT]`. Document the header with
  `z.object({ authorization: z.string().describe("Bearer token (required)").optional() })` so it
  shows up in Swagger.
- `tags` groups the route in the Swagger UI (one tag per domain).

**Don't** register routes with autoload — each domain's plugin is added by hand in `app.ts`.
