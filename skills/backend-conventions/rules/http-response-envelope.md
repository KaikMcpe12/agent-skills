---
title: Response envelope — data / error shapes
tags: api-design, response, envelope, openapi
---

## Response envelope

Every JSON response uses one of two fixed shapes, declared once in `http/http-schemas.ts` and
reused across routes:

- **Success:** `{ data: <payload> }` (single resource or array).
- **Error:** `{ error: { code, message } }`, and for validation also `fields: [{ field, message }]`.

```ts
// http/http-schemas.ts
export const errorResponseSchema = z.object({
  error: z.object({ code: z.string(), message: z.string() }),
});

export const validationErrorResponseSchema = z.object({
  error: z.object({
    code: z.string(),
    message: z.string(),
    fields: z.array(z.object({ field: z.string(), message: z.string() })),
  }),
});

// helper: wrap any payload schema in the data envelope
export const dataResponse = <T extends z.ZodTypeAny>(schema: T) => z.object({ data: schema });
```

Usage in routes / controllers:

```ts
response: { 200: dataResponse(z.array(categorySchema)), 404: errorResponseSchema }
// ...
return reply.status(201).send({ data: category });
```

`error.code` is a stable UPPER_SNAKE string (`NOT_FOUND`, `VALIDATION_ERROR`, `UNAUTHORIZED`,
`INTERNAL_SERVER_ERROR`, `RESPONSE_SERIALIZATION_ERROR`). The 404 not-found and notFound handler
both emit this same envelope. `204` responses send no body.
