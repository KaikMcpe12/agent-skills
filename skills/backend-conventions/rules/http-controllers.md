---
title: Controllers — one file per action
tags: fastify, controllers, error-mapping
---

## Controller pattern

One file per action (`create.ts`, `update.ts`, `delete.ts`, `get.ts`, `list.ts`). The handler:
reads input from the (already-validated) request, types it with `FastifyRequest<{ Body; Params }>`,
calls the use case via its factory, and sends the response envelope. When the use case can throw a
domain error, wrap the call in `try/catch` and delegate to `mapDomainError`.

```ts
// http/controllers/admin-categories/update.ts
export async function update(
  request: FastifyRequest<{ Params: CategoryIdParam; Body: CategoryBody }>,
  reply: FastifyReply,
) {
  const { id } = request.params;
  const { name, description } = request.body;

  try {
    const { category } = await makeUpdateCategoryUseCase().execute({
      id, name, description: description ?? null,
    });
    return reply.status(200).send({ data: category });
  } catch (error) {
    return mapDomainError(error, reply);
  }
}
```

```ts
// a handler that can't fail with a domain error skips the try/catch
export async function create(request: FastifyRequest<{ Body: CategoryBody }>, reply: FastifyReply) {
  const { name, description } = request.body;
  const { category } = await makeCreateCategoryUseCase().execute({ name, description: description ?? null });
  return reply.status(201).send({ data: category });
}
```

Conventions:
- Trust the route schema for validation — type the request generic instead of re-parsing with Zod
  in the controller.
- Status codes: `201` create, `200` read/update, `204` delete (empty body).
- Prefix unused handler args with `_` (`_request`) to satisfy the lint rule.
- No business logic here — just I/O mapping.
