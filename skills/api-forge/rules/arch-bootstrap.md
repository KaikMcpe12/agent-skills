---
title: app.ts builds the instance, server.ts listens
tags: architecture, bootstrap, fastify, swagger, testing
---

## app.ts / server.ts split

`app.ts` **builds and exports** the Fastify instance — plugins, Zod compilers, swagger, the global
error handler, and the manually registered route plugins — **without** calling `listen`, so tests
can import `app` and hit `app.server`. `server.ts` registers `cors`, then `app.ready()` and
`app.listen`. Routes are registered **manually** (one `app.register(xRoutes)` per domain, no
autoload).

```ts
// src/app.ts
import { fastify } from "fastify";
import { jsonSchemaTransform, serializerCompiler, validatorCompiler } from "fastify-type-provider-zod";
import fastifyJwt from "@fastify/jwt";
import fastifySwagger from "@fastify/swagger";
import fastifySwaggerUi from "@fastify/swagger-ui";
import { env } from "./env";
import { errorHandler } from "./error-handler";
import { userRoutes } from "./http/controllers/users/routes";
import { conversationsRoutes } from "./http/controllers/conversations/routes";

export const app = fastify();

app.register(fastifyJwt, { secret: env.JWT_SECRET_KEY, sign: { expiresIn: "7d" } });

app.setErrorHandler(errorHandler);
app.setSerializerCompiler(serializerCompiler);
app.setValidatorCompiler(validatorCompiler);

app.register(fastifySwagger, {
  openapi: { info: { title: "API", description: "API docs", version: "2.0.0" } },
  transform: jsonSchemaTransform, // generate OpenAPI from the Zod route schemas
});
app.register(fastifySwaggerUi, { routePrefix: "/docs" });
app.register(require("@fastify/formbody"));

// routes — one register per domain, added manually
app.register(userRoutes);
app.register(conversationsRoutes);
```

```ts
// src/server.ts — the only place that listens
import { app } from "./app";
import { env } from "./env";
import cors from "@fastify/cors";

app.register(cors, { origin: env.ORIGIN });

async function start() {
  await app.ready(); // let swagger/plugins finish wiring
  app.listen({ port: env.PORT, host: "0.0.0.0" });
}
start();
```

**When adding a route group:** import its plugin in `app.ts` and add one `app.register(...)`. Set
the Zod compilers and the swagger `transform` **before** registering routes so every route's
schema is validated, serialized, and documented consistently (see `swagger-openapi`).
