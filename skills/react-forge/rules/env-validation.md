---
title: Typed environment variables with Zod
tags: env, zod, vite, configuration
---

## env.ts

Validate environment variables once, at startup, with Zod, and import the typed `env` everywhere —
never read `import.meta.env` directly in app code. In Vite the env object is **`import.meta.env`**
(not `process.env`), and only variables prefixed **`VITE_`** are exposed to the client (plus
built-ins like `MODE`, `DEV`, `PROD`).

```ts
// src/env.ts
import { z } from 'zod'

const envSchema = z.object({
  MODE: z.enum(['development', 'production', 'test']),
  VITE_API_URL: z.url(), // Zod 4 top-level; on Zod 3 use z.string().url()
  // flag: accept only "true"/"false", default to false when absent, reject anything else
  VITE_ENABLE_API_DELAY: z
    .enum(['true', 'false'])
    .default('false')
    .transform((value) => value === 'true'),
})

const _env = envSchema.safeParse(import.meta.env)

if (!_env.success) {
  console.error('Invalid environment variables:', _env.error.format())
  throw new Error('Invalid environment variables')
}

export const env = _env.data
```

- `safeParse` + `error.format()` + `throw` gives a clear startup failure listing every bad var.
  (A bare `.parse(import.meta.env)` also works for a smaller app — it throws too, just with a less
  friendly message.)
- Use `z.coerce.number()` for numeric vars. **Avoid `z.coerce.boolean()` for flags** — it applies
  JS `Boolean()`, so the string `"false"` coerces to `true`. Instead accept only the literal
  strings — `z.enum(['true', 'false']).default('false').transform((v) => v === 'true')` — so an
  absent var defaults to `false` and an invalid value (e.g. `"1"`, `"yes"`) is **rejected** at
  startup rather than silently becoming `false`.
- Consume it typed: `axios.create({ baseURL: env.VITE_API_URL })`.

**Client-exposure caveat:** anything under `VITE_` is bundled into the client and publicly visible.
**Never** put secrets (private API keys, tokens) in `VITE_` vars — those belong on a server. `env.ts`
guarantees shape and presence, not secrecy.
