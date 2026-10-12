---
title: Data fetching — TanStack Query + axios
tags: tanstack-query, axios, api, mutations, cache
---

## Server state with TanStack Query

All data that comes from an API is **server state**: fetch and cache it with TanStack Query, not
`useState`/`useEffect` and not Context. One `QueryClient` lives in `lib/react-query.ts` and is
provided in `app.tsx` (see `project-structure`).

### The API client and typed endpoint functions

```ts
// lib/axios.ts
import axios from 'axios'
import { env } from '@/env'

export const api = axios.create({
  baseURL: env.VITE_API_URL,
  withCredentials: true, // only for cookie-based auth; omit for token/header auth
})
```

```ts
// api/get-orders.ts — one typed function per endpoint
import { api } from '@/lib/axios'

export interface GetOrdersQuery { pageIndex?: number | null; status?: string | null }
export interface GetOrdersResponse {
  orders: { orderId: string; customerName: string; total: number; status: string }[]
  meta: { pageIndex: number; perPage: number; totalCount: number }
}

export async function getOrders({ pageIndex, status }: GetOrdersQuery) {
  const response = await api.get<GetOrdersResponse>('/orders', { params: { pageIndex, status } })
  return response.data
}
```

> When the backend exposes an OpenAPI spec, generate these functions + types with Orval instead of
> writing them by hand — see `api-codegen-orval`. Either way they're consumed by Query the same way.

### Queries

The `queryKey` is the cache identity: include every input it depends on, so changing a filter
refetches. Read list/filter params from the URL (`useSearchParams`) rather than duplicating them in
state (see `routing`).

```tsx
const { data: result, isLoading } = useQuery({
  queryKey: ['orders', pageIndex, status],
  queryFn: () => getOrders({ pageIndex, status }),
})
```

### Mutations (with optimistic cache update)

Mutations write; after a write, update or invalidate the affected query. For instant feedback,
update the cache optimistically in `onMutate` and roll back in `onError`:

```tsx
const queryClient = useQueryClient()

const { mutateAsync: updateProfile } = useMutation({
  mutationFn: updateProfileFn,
  onMutate: ({ name }) => {
    const previous = queryClient.getQueryData<Profile>(['profile'])
    if (previous) queryClient.setQueryData<Profile>(['profile'], { ...previous, name })
    return { previous }
  },
  onError: (_err, _vars, context) => {
    if (context?.previous) queryClient.setQueryData(['profile'], context.previous)
  },
})
```

- Simpler alternative: after a successful mutation call
  `queryClient.invalidateQueries({ queryKey: ['orders'] })` to refetch. Use optimistic updates only
  where the snappier UX is worth the rollback code.
- Set `staleTime` for data that rarely changes (e.g. `staleTime: Infinity` for a profile loaded
  once per session).

**Don't** store fetched data in `useState`/Context (that creates stale copies and manual refetch
logic — the exact thing Query removes). **Don't** fetch in `useEffect` when a `useQuery` will do.
