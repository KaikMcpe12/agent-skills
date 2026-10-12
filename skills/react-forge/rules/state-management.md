---
title: State — local vs shared vs server
tags: state, context, use-context-selector, tanstack-query
---

## Choosing where state lives

Classify state first, then pick the mechanism:

1. **Local UI state** → `useState`/`useReducer` in the component. Default; most state is this.
2. **Server state** (anything from an API) → **TanStack Query** (see `data-fetching`). Not Context,
   not `useState`.
3. **URL state** (filters, pagination, selected tab) → `useSearchParams` (see `routing`).
4. **Shared client state** (genuinely cross-tree, non-server: theme, auth user, a multi-step
   wizard) → React **Context**, optionally with **use-context-selector** to avoid re-rendering every
   consumer.

## Context with use-context-selector

Reach for Context only when multiple, distant components need the same *client* state. Plain
`useContext` re-renders every consumer on any context change; `use-context-selector` lets a
component subscribe to one slice:

```tsx
import { createContext } from 'use-context-selector'

interface ThemeContextValue { theme: Theme; setTheme: (t: Theme) => void }
export const ThemeContext = createContext({} as ThemeContextValue)

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('dark')
  return <ThemeContext.Provider value={{ theme, setTheme }}>{children}</ThemeContext.Provider>
}
```

```tsx
import { useContextSelector } from 'use-context-selector'
// subscribes only to `theme`; won't re-render when unrelated context fields change
const theme = useContextSelector(ThemeContext, (ctx) => ctx.theme)
```

Guidelines:
- A derive-from-context hook (e.g. `useSummary`) pairs `useContextSelector` + `useMemo` (see
  `components-and-hooks`).
- You rarely need a dedicated global-state library (Redux/Zustand) in these apps — Query covers
  server state and Context covers the little shared client state that remains. Add one only for a
  concrete need the above can't meet.

**Don't** use Context as a server cache (fetching into a provider and sharing it) — that reinvents a
worse TanStack Query, with manual refetch and stale copies. **Don't** lift state to Context "just in
case"; keep it local until it's actually shared.
