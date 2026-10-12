---
title: Controllers — one file per action
tags: fastify, controllers, error-mapping, transactions
---

## Controller pattern

One file per action (`create-client-controller.ts`, `login-user-controller.ts`, …), exporting a
named handler. The handler: reads input, calls the use case via its **async factory**, sends a
plain response object, and maps known domain errors to HTTP with `instanceof`. Unknown errors are
re-thrown to the global handler (see `http-errors`).

**Write** (opens a transaction via the factory → commit on success, rollback in the catch):

```ts
// http/controllers/clients/create-client-controller.ts
import { FastifyReply, FastifyRequest } from "fastify";
import { createClientDTO } from "../../dto/client/client-dto";
import { makeCreateClientUseCase } from "../../../use-cases/factories/make-create-client-use-case";
import { ClientAlreadyExistsError } from "../../../use-cases/errors/client-already-exists-error";

export async function create(request: FastifyRequest, reply: FastifyReply) {
  const data = createClientDTO.parse(request.body);
  const { createClientUseCase, db } = await makeCreateClientUseCase();

  try {
    await createClientUseCase.execute(data);
    await db.commitAndClose();
    return reply.status(201).send();
  } catch (error) {
    await db.rollbackAndClose();
    if (error instanceof ClientAlreadyExistsError) {
      return reply.status(409).send({ message: error.message });
    }
    throw error; // unknown → global handler
  }
}
```

**Read** (no transaction; returns a plain object shaped by the route's response DTO):

```ts
// http/controllers/conversations/get-conversation-by-id-controller.ts
export async function getConversationByIdController(request: FastifyRequest, reply: FastifyReply) {
  const { id } = getConversationByIdParamsDTO.parse(request.params);

  try {
    const useCase = await makeGetConversationByIdUseCase();
    const { conversation } = await useCase.execute({ conversationId: id, authenticatedUserId: request.user.sub });
    return reply.status(200).send({ conversation });
  } catch (error) {
    if (error instanceof ResourceNotFoundError) return reply.status(404).send({ message: error.message });
    if (error instanceof NotAllowedError) return reply.status(403).send({ message: error.message });
    throw error;
  }
}
```

Conventions:
- Type the request with `FastifyRequest<{ Body: XDTO }>` when the body shape matters; read the
  authenticated id with `request.user.sub`.
- **Status codes:** `201` create (often empty `send()` or `{ token }`), `200` read/update,
  `403`/`404`/`409`/`401` for the matching domain error. Use the status your route `response` map
  declares.
- Success payloads are **plain objects** named after the resource (`{ conversation }`,
  `{ clients, totalCount }`, `{ token }`) — there is no `{ data }` wrapper.
- Only `try/catch` when a domain error is possible; for writes, always pair
  `commitAndClose()` (success) with `rollbackAndClose()` (catch).

> The route `schema` already validates input via the type provider. Some reference controllers
> also call `dto.parse(request.body)` for a typed value — either is fine, but don't add a third
> hand-written type; infer from the DTO.
