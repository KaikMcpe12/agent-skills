---
title: Error handling — domain errors + mapDomainError + global handler
tags: errors, error-handling, fastify, zod
---

## Error handling (three layers)

1. **Domain error classes** (`use-cases/errors/`) thrown by use cases — semantic, no HTTP knowledge.
2. **`mapDomainError`** translates known domain errors to HTTP in the controller's catch block;
   unknown errors are re-thrown to the global handler.
3. **Global `errorHandler`** (`app.ts`) is the safety net: Zod/validation → 400, serialization
   error → 500, everything else → 500. It's the only place that logs.

```ts
// use-cases/errors/resource-not-found-error.ts
export class ResourceNotFoundError extends Error {
  constructor() {
    super("Resource not found");
    this.name = "ResourceNotFoundError";
  }
}
```

```ts
// http/map-domain-error.ts
export function mapDomainError(error: unknown, reply: FastifyReply) {
  if (error instanceof ResourceNotFoundError) {
    return reply.status(404).send({ error: { code: "NOT_FOUND", message: error.message } });
  }
  throw error; // unknown → global handler
}
```

```ts
// http/error-handler.ts (shape)
export const errorHandler: FastifyErrorHandler = (error, _request, reply) => {
  if (hasZodFastifySchemaValidationErrors(error)) {
    return reply.status(400).send({
      error: { code: "VALIDATION_ERROR", message: "Validation error",
        fields: error.validation.map((e) => ({
          field: e.instancePath.replace(/^\//, "").replace(/\//g, "."), message: e.message })) },
    });
  }
  if (isResponseSerializationError(error)) {
    return reply.status(500).send({ error: { code: "RESPONSE_SERIALIZATION_ERROR", message: "..." } });
  }
  if (error instanceof ZodError) { /* same VALIDATION_ERROR shape from error.issues */ }
  if (env.NODE_ENV !== "production") console.error(error); // else: report to Sentry/Datadog (TODO)
  return reply.status(500).send({ error: { code: "INTERNAL_SERVER_ERROR", message: "Internal server error" } });
};
```

**Don't** send error responses from a use case — throw a domain error. **Don't** handle validation
errors in a controller — let them reach the global handler.
