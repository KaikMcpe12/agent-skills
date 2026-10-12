---
title: Swagger / OpenAPI — generated from the Zod route schemas
tags: swagger, openapi, zod, fastify-type-provider-zod, documentation
---

## Swagger / OpenAPI

The OpenAPI document is **generated from the same Zod schemas** that validate requests and
serialize responses — there is no hand-written spec file. The chain is: `ZodTypeProvider` +
Zod compilers make each route's `schema` authoritative, then `@fastify/swagger` with
`transform: jsonSchemaTransform` turns those schemas into OpenAPI, and `@fastify/swagger-ui` serves
the UI at `/docs`.

### 1. Wire it once in `app.ts`

```ts
import { jsonSchemaTransform, serializerCompiler, validatorCompiler } from "fastify-type-provider-zod";
import fastifySwagger from "@fastify/swagger";
import fastifySwaggerUi from "@fastify/swagger-ui";

app.setSerializerCompiler(serializerCompiler); // Zod serializes responses
app.setValidatorCompiler(validatorCompiler);   // Zod validates requests

app.register(fastifySwagger, {
  openapi: {
    info: { title: "API ContacteMe", description: "Documentação da API", version: "2.0.0" },
  },
  transform: jsonSchemaTransform, // Zod route schemas → OpenAPI
});
app.register(fastifySwaggerUi, { routePrefix: "/docs" });
```

> Order matters: set the compilers and register swagger **before** the route plugins (see
> `arch-bootstrap`). Use `await app.ready()` before `listen` so the spec is fully built.

### 2. Make each route carry a complete, typed schema

Every route uses the typed router and declares `tags` + input schemas + a `response` entry per
status. Because the type provider drives serialization, the `response` schema is also the public
contract — declare one for **every** status you return.

```ts
app.withTypeProvider<ZodTypeProvider>().get("/conversations/:id", {
  preHandler: [verifyJWT],
  schema: {
    tags: ["Conversations"],                 // groups the route in the UI
    params: getConversationByIdParamsDTO,
    headers: z.object({
      authorization: z.string().describe("Bearer token (required)").optional(), // documents the header
    }),
    response: {
      200: getConversationResponseDTO,        // documented + serialized from this Zod schema
      401: z.object({ message: z.string() }),
      404: z.object({ message: z.string() }),
    },
  },
}, getConversationByIdController);
```

Conventions:
- **One source of truth:** the Zod schema = validation + serialization + docs. Don't maintain a
  separate OpenAPI file, and don't document a shape that differs from what the route actually
  returns.
- `tags` (one per domain) organize the UI; `.describe(...)` on fields/headers enriches the spec.
- Reuse response **DTOs** from `http/dto/<domain>/` so the documented shape matches the controller's
  payload exactly.

**Don't** add `@fastify/swagger` in `mode: "static"` with a yaml file — this project generates the
spec from the routes via `jsonSchemaTransform`.
