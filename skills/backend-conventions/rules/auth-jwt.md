---
title: Auth — Bearer JWT only
tags: auth, jwt, fastify, security
---

## Authentication

Bearer-only JWT via `@fastify/jwt` (no refresh cookie). Tokens are signed with `env.JWT_SECRET`
and expire in `1d` (configured in `app.ts`). A `verify-jwt` middleware guards protected routes
and replies `401` in the standard error envelope. The JWT payload shape is augmented globally.

```ts
// http/middlewares/verify-jwt.ts
export async function verifyJwt(request: FastifyRequest, reply: FastifyReply) {
  try {
    await request.jwtVerify();
  } catch {
    return reply.status(401).send({ error: { code: "UNAUTHORIZED", message: "Unauthorized" } });
  }
}
```

```ts
// src/@types/fastify-jwt.d.ts — typed payload
import "@fastify/jwt";
declare module "@fastify/jwt" {
  interface FastifyJWT {
    user: { sub: number }; // sub = authenticated admin id; single role, so no `role` claim
  }
}
```

Conventions:
- Read the authenticated id via `request.user.sub`.
- Hash passwords with `bcryptjs` (`await hash(password, 6)`), store as `password_hash`.
- Apply `verifyJwt` as a group `onRequest` hook for admin route groups; public routes have none.
