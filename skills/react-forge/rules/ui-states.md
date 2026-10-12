---
title: UI states — loading, error, empty, feedback
tags: loading, error, empty, skeleton, toast, accessibility
---

## Handle every async state

A view that fetches data has four states — render each deliberately: **loading**, **error**,
**empty**, and **data**.

### Loading — Skeletons

Show a skeleton that mirrors the final layout (shadcn `Skeleton`), not a bare spinner, for
content-heavy views:

```tsx
{isLoading && <OrderTableSkeleton />}
{result && result.orders.map((order) => <OrderTableRow key={order.orderId} order={order} />)}
```

```tsx
// a skeleton mirrors the real rows
export function OrderTableSkeleton() {
  return Array.from({ length: 10 }).map((_, i) => (
    <TableRow key={i}>
      <TableCell><Skeleton className="h-4 w-[172px]" /></TableCell>
      <TableCell><Skeleton className="h-4 w-[110px]" /></TableCell>
    </TableRow>
  ))
}
```

### Empty state

When a successful query returns nothing, render an explicit empty state (a short message / call to
action), not a blank area or a stuck skeleton.

### Error

- **Route-level:** react-router's `errorElement` catches render/loader errors; read it with
  `useRouteError()` and show a recovery link (see `routing`).
- **Expected request errors:** handle them where they happen — a `try/catch` around a mutation with
  a toast, or an axios interceptor for cross-cutting cases (e.g. `401` → redirect).

```tsx
// pages/error.tsx
export function Error() {
  const error = useRouteError() as Error
  return (
    <div className="flex h-screen flex-col items-center justify-center gap-2">
      <h1 className="text-4xl font-bold">Algo deu errado</h1>
      <pre>{error?.message}</pre>
      <Link to="/" className="text-sky-600 dark:text-sky-400">Voltar ao início</Link>
    </div>
  )
}
```

### Feedback — toasts

Use **sonner** for transient feedback on actions (`<Toaster richColors />` mounted once in
`app.tsx`):

```tsx
import { toast } from 'sonner'
toast.success('Perfil atualizado com sucesso!')
toast.error('Não foi possível salvar, tente novamente.')
```

Accessibility: icon-only controls get an `sr-only` label; error text uses `role="alert"`; keep the
Radix-provided semantics from shadcn components.

**Don't** leave a view in a permanent spinner on error, and don't swallow a failed mutation without
telling the user.
