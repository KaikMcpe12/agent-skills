---
title: Routing — react-router
tags: routing, react-router, layouts, error, search-params
---

## Routing

Use **react-router-dom** with `createBrowserRouter`. Group routes under **layout routes** (a layout
renders shared chrome + `<Outlet />`), attach an `errorElement` for render/loader errors, and catch
unknown paths with `path: "*"`.

```tsx
// routes.tsx
import { createBrowserRouter } from 'react-router-dom'
import { AppLayout } from './pages/_layouts/app'
import { AuthLayout } from './pages/_layouts/auth'
import { Dashboard } from './pages/app/dashboard/dashboard'
import { SignIn } from './pages/auth/sign-in'
import { NotFound } from './pages/404'
import { Error } from './pages/error'

export const router = createBrowserRouter([
  {
    path: '/',
    element: <AppLayout />,
    errorElement: <Error />,
    children: [{ path: '/', element: <Dashboard /> }],
  },
  { path: '/', element: <AuthLayout />, children: [{ path: '/sign-in', element: <SignIn /> }] },
  { path: '*', element: <NotFound /> },
])
```

```tsx
// pages/_layouts/app.tsx — shared shell via <Outlet/>
export function AppLayout() {
  return (
    <div className="flex min-h-screen flex-col">
      <Header />
      <div className="flex flex-1 flex-col gap-4 p-8 pt-6">
        <Outlet />
      </div>
    </div>
  )
}
```

Patterns:
- A layout is the place for cross-cutting concerns — e.g. an axios response interceptor that
  redirects to `/sign-in` on a `401`, registered in `useEffect` and ejected on cleanup.
- **URL as state:** keep list filters/pagination in the query string with `useSearchParams` instead
  of component state, so the view is shareable and refetches naturally (the params feed the query
  key — see `data-fetching`). Coerce with Zod: `z.coerce.number().parse(searchParams.get('page') ?? '1')`.
- 404 (`pages/404.tsx`) and the router error page (`pages/error.tsx`, via `useRouteError()`) are
  plain components — see `ui-states`.

**Don't** force a router onto a genuinely single-screen app, and don't mirror the current route into
React state — read it from the router (`useLocation`, `useSearchParams`, `data-current`).
