---
title: Testing — Vitest, Testing Library, MSW, Playwright
tags: testing, vitest, testing-library, msw, playwright
---

## Testing strategy

Adopt tests in proportion to risk and complexity (precedence rule #4): a throwaway prototype may
need none; anything with real logic or regression risk should have them. The preferred tools:

- **Vitest + Testing Library** — unit/component tests, co-located as `*.spec.tsx`.
- **MSW** — mock the API at the network layer (shared between tests and a dev mock mode).
- **Playwright** — end-to-end for critical flows.

### Vitest + Testing Library

Vitest shares Vite's config (same `@/` alias). Use a DOM environment and a setup file:

```ts
// vitest.config.ts
import path from 'node:path'
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vitest/config'

export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
  test: { globals: true, environment: 'happy-dom', setupFiles: ['./test/setup.ts'] },
})
```

Test behavior, query by accessible role/text, and wrap components that need router/query context:

```tsx
// components/nav-link.spec.tsx
import { render } from '@testing-library/react'
import { MemoryRouter } from 'react-router-dom'
import { describe, expect, it } from 'vitest'
import { NavLink } from './nav-link'

describe('NavLink', () => {
  it('marks the active link', () => {
    const wrapper = render(
      <>
        <NavLink to="/">Home</NavLink>
        <NavLink to="/about">About</NavLink>
      </>,
      { wrapper: ({ children }) => <MemoryRouter initialEntries={['/about']}>{children}</MemoryRouter> },
    )
    expect(wrapper.getByText('Home').dataset.current).toEqual('false')
    expect(wrapper.getByText('About').dataset.current).toEqual('true')
  })
})
```

### MSW — mock the API

Define handlers once; start the worker only in the mock/test mode. The same handlers back both the
test run and a `MODE=test` dev mode:

```ts
// api/mocks/index.ts
import { setupWorker } from 'msw/browser'
import { env } from '@/env'
import { signInMock } from './sign-in.mock'

export const worker = setupWorker(signInMock)

export async function enableMsw() {
  if (env.MODE !== 'test') return
  await worker.start()
}
```

```ts
// api/mocks/sign-in.mock.ts — typed against the endpoint's request body
import { http, HttpResponse } from 'msw'
import type { SignInBody } from '../sign-in'

export const signInMock = http.post<never, SignInBody>('/authenticate', async ({ request }) => {
  const { email } = await request.json()
  return email === 'john.doe@example.com'
    ? new HttpResponse(null, { status: 200 })
    : new HttpResponse(null, { status: 401 })
})
```

Playwright drives the built app against this mock mode for e2e flows.

**Don't** stand up a full test pyramid for a trivial app, and **don't** skip tests for critical,
regression-prone logic (auth, money, data transforms). Test what matters.
