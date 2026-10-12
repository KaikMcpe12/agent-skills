---
title: Error handling — domain errors + global handler
tags: errors, error-handling, fastify, zod
---

## Error handling (two layers)

1. **Domain error classes** (`use-cases/errors/`) thrown by use cases — semantic, no HTTP
   knowledge. Controllers translate them with `instanceof` → `reply.status(x).send({ message })`.
2. **Global `errorHandler`** (`src/error-handler.ts`, wired in `app.ts`) is the safety net for
   everything a controller re-throws: Zod / Fastify validation → `400`, token expiry → `400`,
   anything else → `500`. It is the only place that logs.

```ts
// use-cases/errors/resource-not-found-error.ts
export class ResourceNotFoundError extends Error {
  constructor() {
    super("Resource not found");
  }
}
```

```ts
// src/error-handler.ts
import { ZodError } from "zod";
import { TokenExpiredError } from "jsonwebtoken";

export function errorHandler(error: any, req: any, res: any) {
  if (error.validation && error.code === "FST_ERR_VALIDATION") {
    return res.status(400).send({ message: "Validation failed", errors: error.validation });
  }
  if (error instanceof ZodError) {
    return res.status(400).send({ message: "Invalid input", errors: error.flatten().fieldErrors });
  }
  if (error instanceof TokenExpiredError) {
    return res.status(400).send({ message: error.message });
  }
  console.log("Error:", error);
  return res.status(500).send({ message: "Internal server error" });
}
```

Response shapes:
- **Error:** `{ message }`. Validation failures add `errors` (Fastify's `error.validation` array,
  or Zod's `flatten().fieldErrors`). **There is no `{ error: { code } }` envelope.**
- **Success:** a plain object shaped by the route's response DTO (see `http-controllers`).

Conventions:
- Domain errors `extend Error` and set the message in the constructor; the user-facing message may
  be Portuguese, the class name stays English.
- Map each domain error to its HTTP status **in the controller** with `instanceof`; `throw` unknown
  errors so the global handler turns them into a `500`.

**Don't** build a `mapDomainError` helper or a code-based envelope — the convention is the direct
`instanceof` check in the controller plus the global handler. **Don't** send error responses from a
use case; throw a domain error instead. **Don't** catch validation errors in a controller; let them
reach the global handler.
