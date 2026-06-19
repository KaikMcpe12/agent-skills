---
title: Routes — typed plugin with per-route schema
tags: fastify, routes, zod, swagger, auth
---

## Route pattern

Each domain has a `routes.ts` exporting an `async (app: FastifyInstance)` plugin. Inside, grab a
typed router via `app.withTypeProvider<ZodTypeProvider>()` and declare, per route: `tags`,
`params`/`body`/`query` schemas, and a `response` map with **one Zod schema per status code**.
Handlers live in separate per-action files. Auth is applied with an `onRequest` hook.

```ts
// http/controllers/admin-categories/routes.ts
const tags = ["Admin — Categories"];

export async function adminCategoriesRoutes(app: FastifyInstance) {
  const router = app.withTypeProvider<ZodTypeProvider>();

  router.addHook("onRequest", verifyJwt); // protect the whole group

  router.post("/admin/categories", {
    schema: {
      tags,
      body: categoryBodySchema,
      response: {
        201: dataResponse(categorySchema),
        400: validationErrorResponseSchema,
        401: errorResponseSchema,
      },
    },
  }, create);

  router.put("/admin/categories/:id", {
    schema: {
      tags,
      params: categoryIdParamSchema,
      body: categoryBodySchema,
      response: {
        200: dataResponse(categorySchema),
        400: validationErrorResponseSchema,
        401: errorResponseSchema,
        404: errorResponseSchema,
      },
    },
  }, update);
}
```

- Declare **every** relevant status (success + each error) with its shape — the response schema
  is also the contract serializer.
- Group-wide auth → `router.addHook("onRequest", verifyJwt)`. Per-route auth →
  `{ onRequest: [verifyJwt] }` in that route's options.
- Public vs admin variants of the same resource are separate route groups (`categories` vs
  `admin/categories`); the difference is the auth hook.
