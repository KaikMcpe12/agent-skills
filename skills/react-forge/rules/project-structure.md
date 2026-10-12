---
title: Project structure and app wiring
tags: structure, vite, providers, organization
---

## Structure

Group by role under `src/`, co-locating a page's subcomponents and tests with the page:

```
src/
├── api/              # one typed fn per endpoint (+ mocks/ for MSW); or http/api.ts when Orval-generated
├── components/ui/    # shadcn components;  components/*.tsx = shared app components
├── hooks/            # custom hooks (only when earned — see components-and-hooks)
├── lib/              # axios.ts, react-query.ts, utils.ts (cn)
├── pages/<area>/     # route pages + co-located subcomponents + *.spec.tsx; _layouts/, 404.tsx, error.tsx
├── env.ts  routes.tsx  app.tsx  main.tsx
```

## Wiring

`main.tsx` mounts the root (optionally starting MSW first in test); `app.tsx` nests the providers;
`routes.tsx` defines the router. Keep provider order stable: **Theme → Toaster → QueryClient →
Router**.

```tsx
// main.tsx — default
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { App } from './app'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

Only if the project uses **MSW** (see `testing`), gate the render on the worker so mocks are ready
before the first request — otherwise keep the plain `createRoot` above and don't scaffold `api/mocks`:

```tsx
// main.tsx — variant when MSW is used
import { enableMsw } from './api/mocks' // no-ops outside the mock mode

enableMsw().then(() => {
  createRoot(document.getElementById('root')!).render(
    <StrictMode>
      <App />
    </StrictMode>,
  )
})
```

```tsx
// app.tsx
import { QueryClientProvider } from '@tanstack/react-query'
import { RouterProvider } from 'react-router-dom'
import { Toaster } from 'sonner'
import { ThemeProvider } from './components/theme/theme-provider'
import { queryClient } from './lib/react-query'
import { router } from './routes'

export function App() {
  return (
    <ThemeProvider defaultTheme="dark" storageKey="app-theme">
      <Toaster richColors />
      <QueryClientProvider client={queryClient}>
        <RouterProvider router={router} />
      </QueryClientProvider>
    </ThemeProvider>
  )
}
```

- `lib/` holds infra singletons: `axios.ts` (the API client), `react-query.ts` (`new QueryClient()`),
  `utils.ts` (`cn`).
- Reusable formatters live in a small `utils/` module (e.g. `Intl.NumberFormat`/`DateTimeFormat`
  instances), not inline in components.
- **Static assets:** import assets that are part of the build from `src/assets`
  (`import logo from '@/assets/logo.svg'`) so Vite fingerprints and bundles them; put only files
  that must be served verbatim at a stable path (e.g. `favicon`, `robots.txt`) in `public/` and
  reference them by root-absolute URL (`/favicon.ico`).
- Provider nesting above is the *full* form; include only the providers the app actually uses
  (e.g. drop `ThemeProvider`/`Toaster` if there's no theming/toasts yet).

**Scale to the app.** A tiny app can keep a handful of files and skip `pages/<area>` nesting; a
larger one splits by area. Don't impose a large-app layout on a small one, and don't scatter a
large app into one giant `components/` folder.
