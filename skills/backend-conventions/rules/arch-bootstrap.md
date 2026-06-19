---
title: app.ts builds the instance, server.ts listens
tags: architecture, bootstrap, fastify, testing
---

## app.ts / server.ts split

`app.ts` **builds and exports** the Fastify instance (plugins, compilers, swagger, error
handler, routes) **without** calling `listen` — so tests can import `app` and hit `app.server`.
`server.ts` is the only entry that calls `app.listen`. Routes are registered **manually**
(no autoload).

```ts
// src/app.ts
export const app = fastify().withTypeProvider<ZodTypeProvider>();

app.setValidatorCompiler(validatorCompiler);
app.setSerializerCompiler(serializerCompiler);

app.register(cors, {
  origin: env.CORS_ORIGIN === "*" ? true : env.CORS_ORIGIN.split(",").map((o) => o.trim()),
});
app.register(fastifySwagger, { mode: "static", specification: { path: /* docs/openapi.yaml */ } });
app.register(fastifySwaggerUi, { routePrefix: "/docs" });
app.register(fastifyJwt, { secret: env.JWT_SECRET, sign: { expiresIn: "1d" } });

app.setErrorHandler(errorHandler);
app.setNotFoundHandler((_req, reply) =>
  reply.status(404).send({ error: { code: "NOT_FOUND", message: "Resource not found" } }));

app.register(healthRoutes);
app.register(categoriesRoutes);
app.register(adminCategoriesRoutes); // add new route groups here, manually
```

```ts
// src/server.ts — the only place that listens
app.listen({ host: "0.0.0.0", port: env.PORT }).then(() => {
  console.log(`🚀 HTTP Server running on http://localhost:${env.PORT}`);
});
```

**When adding a route group:** import its plugin in `app.ts` and add one `app.register(...)`.
