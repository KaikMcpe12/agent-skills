---
title: Components and custom hooks
tags: components, hooks, composition, abstraction
---

## Components

- One component per file, named by role; co-locate a page's private subcomponents next to the page
  (`pages/<area>/`), and put genuinely shared ones in `components/`.
- Keep components focused: presentational components take props and render; data/wiring lives in the
  page or a hook. Type props with an explicit interface (or `React.ComponentProps<'el'>` when
  wrapping an element).
- Extract a component when a piece of UI is **reused** or when a page gets too large to read — not
  preemptively. Prefer composing shadcn components over building generic ones (see `shadcn-ui`).

## Custom hooks

Write a custom hook when it encapsulates **reused stateful logic** or a non-trivial derivation —
not to hide a couple of lines. A good hook has one clear responsibility and a typed return.

```ts
// hooks/use-summary.ts — derive a value from shared context, memoized
import { useMemo } from 'react'
import { useContextSelector } from 'use-context-selector'
import { TransactionsContext } from '@/contexts/transactions-context'

export function useSummary() {
  const transactions = useContextSelector(TransactionsContext, (ctx) => ctx.transactions)

  return useMemo(
    () =>
      transactions.reduce(
        (acc, t) => {
          if (t.type === 'income') { acc.income += t.price; acc.total += t.price }
          else { acc.outcome += t.price; acc.total -= t.price }
          return acc
        },
        { income: 0, outcome: 0, total: 0 },
      ),
    [transactions],
  )
}
```

Notes:
- `useContextSelector` subscribes to **one slice** of a context, so the component re-renders only
  when that slice changes (see `state-management`).
- `useMemo` here avoids recomputing the summary on every render; use it for genuinely expensive
  derivations, not reflexively.

**Don't** create a hook that just wraps a single `useState`, or a component so generic it takes a
dozen props to configure. Favor explicit, readable code; reach for an abstraction when it removes
real duplication or clarifies a page.
