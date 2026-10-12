---
title: Auth — Bearer JWT via @fastify/jwt
tags: auth, jwt, fastify, bcrypt, security
---

## Authentication

Bearer-only JWT via `@fastify/jwt` (registered in `app.ts` with `sign.expiresIn: "7d"`). A
`verifyJWT` middleware guards protected routes and replies `401`; a `verifyUserRole` factory guards
by role. The payload carries `{ sub, role }` and is typed by augmenting `@fastify/jwt`.

```ts
// src/@types/fastify-jwt.d.ts — typed payload
import "@fastify/jwt";
declare module "@fastify/jwt" {
  interface FastifyJWT {
    user: { sub: string; role: "ADMIN" | "MEMBER" };
  }
}
```

```ts
// http/middleware/verify-jwt.ts
import { FastifyReply, FastifyRequest } from "fastify";

export async function verifyJWT(request: FastifyRequest, reply: FastifyReply) {
  try {
    await request.jwtVerify();
  } catch {
    return reply.status(401).send({ message: "Unauthorized" });
  }
}
```

```ts
// http/middleware/verify-user-role.ts
export function verifyUserRole(roleToVerify: "ADMIN" | "MEMBER") {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    if (request.user.role !== roleToVerify) {
      return reply.status(401).send({ message: "Unauthorized" });
    }
  };
}
```

Signing happens in the login/register controller with `reply.jwtSign`; passwords are hashed with
**bcrypt**:

```ts
// in the controller
const token = await reply.jwtSign(
  { role: user.level === 1 ? "ADMIN" : "MEMBER" },
  { sign: { sub: user.id, expiresIn: "7d" } },
);
return reply.status(201).send({ token });

// in the use case
import bcrypt from "bcrypt";
const passwordHash = await bcrypt.hash(password, 10);   // create
const match = await bcrypt.compare(password, user.password as string); // login
```

Conventions:
- Apply auth per route with `preHandler: [verifyJWT]` (and `verifyUserRole("ADMIN")` where a role
  is required); read the authenticated id via `request.user.sub`.
- Hash with `bcrypt` (`hash(pw, 10)`), store as the password hash column; compare with
  `bcrypt.compare`.
- Middleware lives in `http/middleware/` (singular).

**Don't** use a refresh cookie or a second auth scheme — it's Bearer-only.
