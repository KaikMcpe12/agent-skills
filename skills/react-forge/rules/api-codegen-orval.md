---
title: API client generation with Orval
tags: orval, openapi, codegen, types, api
---

## Orval — generate the client from an OpenAPI spec

**When the backend exposes an OpenAPI spec**, generate the client functions and types with Orval
instead of hand-writing `src/api/*.ts`. This keeps the frontend contracts in lockstep with the API
and removes a whole class of drift. If there's no spec (or the project already has a working
hand-written API layer), stay with the hand-written approach in `data-fetching` — don't introduce
Orval speculatively.

> Synergy with `api-forge`: that backend serves a generated OpenAPI document at `/docs/json`, which
> is exactly what Orval consumes below.

```ts
// orval.config.ts
import { defineConfig } from 'orval'

export default defineConfig({
  api: {
    input: 'http://localhost:3333/docs/json', // the backend's OpenAPI JSON
    output: {
      target: './src/http/api.ts', // generated client + types (one file)
      client: 'fetch',             // or 'react-query' to emit hooks
      clean: true,
      baseUrl: 'http://localhost:3333',
    },
  },
})
```

Run it as a script (e.g. `"generate:api": "orval"`); re-run when the API changes. **Never edit the
generated file** — it's overwritten.

The generated functions and types are imported directly; pair them with React Hook Form and
TanStack Query like any `src/api` function:

```tsx
import { subscribeToEvent, type SubscribeToEventBody } from '@/http/api'

async function onSubmit({ name, email }: SubscribeToEventBody) {
  const { subscriberId } = await subscribeToEvent({ name, email })
  // navigate / toast …
}
```

Conventions:
- `client: 'fetch'` emits plain functions (wrap them in `useQuery`/`useMutation` yourself);
  `client: 'react-query'` emits ready-made hooks — pick one per project and be consistent.
- Form **validation** schemas are still your own Zod schemas (Orval generates request/response
  *types*, not form validation). Keep using `zodResolver` (see `forms`).
- Commit the generated file so builds don't require a running backend.

**Don't** assume every project has an OpenAPI spec, and don't mix generated and hand-written clients
for the same API — choose one source of truth.
