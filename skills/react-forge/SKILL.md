---
name: react-forge
description: My frontend conventions for React + TypeScript single-page apps built with Vite, Tailwind CSS v4, and shadcn/ui — covering project structure, component and custom-hook organization, forms (React Hook Form + Zod), server state (TanStack Query) and API layers (hand-written or Orval-generated from an OpenAPI spec), client/shared state, routing (react-router), typed env validation, UI states (loading/empty/error/toasts), and testing (Vitest + Testing Library + MSW + Playwright). Defines a preferred library set and a precedence rule so agents don't re-decide every project. Use when building, reviewing, or refactoring a React web frontend, choosing frontend libraries, or setting up a Vite + Tailwind + shadcn/ui app.
license: MIT
metadata:
  author: kaik
  version: "1.0.0"
---

# react-forge

The way I build web frontends, reusable across projects: a **React 19 + TypeScript** SPA on
**Vite**, styled with **Tailwind CSS v4** and **shadcn/ui**, with **React Hook Form + Zod** forms,
**TanStack Query** for server state, **react-router** for navigation, and a typed **`env.ts`**.
This skill gives an agent a clear *preferred path* plus the judgment to deviate when the user, the
project, or the problem calls for something simpler.

> Opinionated, not mandatory. Entity/domain names in examples are placeholders. The examples are
> adapted from real projects (`pizzashop` is the backbone — the only one on this exact stack).

## Precedence rule (apply this before choosing any library or pattern)

1. **Explicit user preference** — if the user picks a library/approach, use it (when viable). Flag
   a real risk and propose an alternative; never swap silently.
2. **Existing project conventions** — otherwise preserve what the project already uses. Don't add a
   second library for a solved problem; don't migrate the stack without a concrete reason.
3. **The preferred set** (below) — otherwise, use it.
4. **Contextual fit** — if a recommended tool is unnecessary, incompatible, or overkill for the
   project's size, pick the simpler option and explain any significant divergence.

## Preferred set

| Concern | Preferred | When |
|---|---|---|
| Build/dev | **Vite** | SPA (non-SSR) projects |
| Components | **shadcn/ui** (Radix + `cva` + `cn`) | any non-trivial UI; see `shadcn-ui` |
| Styling | **Tailwind CSS v4** (CSS-first tokens) | always; see `styling-tailwind` |
| Icons | **lucide-react** | shadcn's default icon set |
| Forms | **React Hook Form + Zod** (`zodResolver`) | validated/multi-field forms; see `forms` |
| Server state | **TanStack Query** (+ `axios`) | any data from an API; see `data-fetching` |
| API types | hand-written `src/api/*.ts`, or **Orval** | Orval **when** an OpenAPI spec exists; see `api-codegen-orval` |
| Routing | **react-router-dom** (`createBrowserRouter`) | multi-page SPA; see `routing` |
| Shared client state | React Context + **use-context-selector** | only when genuinely shared; see `state-management` |
| Env | **`env.ts` + Zod** over `import.meta.env` | any env var; see `env-validation` |
| Toasts | **sonner** | user feedback |
| Tests | **Vitest + Testing Library** (+ MSW, + Playwright) | contextual; see `testing` |

Not every category needs a library — plain React (`useState`, props) is the default for local
state and simple UI. Don't stack overlapping libraries.

## When to Apply

- Scaffolding a Vite + React + TS app, or adding Tailwind v4 / shadcn/ui to one.
- Building a feature: route → page → form → data fetching → UI states.
- Choosing a frontend library, or reviewing/refactoring components, hooks, forms, or state.

## Folder Structure

```
src/
├── api/              # one file per endpoint (typed axios fns) + mocks/ (MSW); OR http/api.ts (Orval)
├── assets/           # imported static assets (svg, images)
├── components/
│   ├── ui/           # shadcn/ui components (added via CLI)
│   └── *.tsx         # shared app components (header, nav-link, dialogs…)
├── hooks/            # custom hooks (only when they earn their place)
├── lib/              # axios.ts, react-query.ts, utils.ts (cn)
├── pages/
│   ├── _layouts/     # layout routes (Outlet) — app shell, auth shell
│   ├── <area>/       # route pages grouped by area, co-located subcomponents + *.spec.tsx
│   ├── 404.tsx       # not-found (router path "*")
│   └── error.tsx     # router errorElement
├── env.ts            # Zod-validated import.meta.env
├── routes.tsx        # createBrowserRouter
├── app.tsx           # providers (Theme → Toaster → QueryClient → Router)
└── main.tsx          # createRoot (+ enableMsw in test)
```

> Scale the structure to the app: a tiny app may collapse `pages/` into a few files. Don't impose
> a large-app layout on a small one (precedence rule #4).

## Rule Index

Read the file in `rules/` for the pattern + a real example of each.

| Rule | What it covers |
|------|----------------|
| `project-structure` | folder layout, `main`/`app`/`routes`/`env` wiring, provider nesting, scaling to app size |
| `setup-vite-tailwind` | Vite + `@tailwindcss/vite`, `@import "tailwindcss"`, `@/*` alias, v4 CSS-first `@theme` tokens |
| `shadcn-ui` | `init`/`components.json`, adding components, composition (`asChild`/`cva`/`cn`), icons, when not to use |
| `components-and-hooks` | component organization; when to extract a component or a custom hook; avoid premature abstraction |
| `styling-tailwind` | `cn()` composition, responsive/state/dark variants, CSS-var tokens, class ordering |
| `forms` | React Hook Form + `zodResolver` + `z.infer`; shadcn inputs; errors; simple vs validated |
| `env-validation` | `env.ts` with Zod over `import.meta.env`; `VITE_` prefix; client-exposure caveat |
| `data-fetching` | TanStack Query queries/mutations, optimistic cache, `lib/axios.ts`, typed `src/api/*.ts` |
| `api-codegen-orval` | Orval when an OpenAPI spec exists (`/docs/json` → `src/http/api.ts`); wiring into Query/RHF |
| `routing` | `createBrowserRouter`, layout routes, `errorElement`, 404, `useSearchParams` as URL state |
| `state-management` | local vs shared (Context + `use-context-selector`) vs server (Query); what *not* to put in context |
| `ui-states` | loading (Skeleton), error (errorElement/boundary), empty states, toast feedback |
| `testing` | Vitest + Testing Library (+ MSW, Playwright); contextual adoption |
| `example-feature` | a full feature end-to-end + a build checklist |

## Golden Rules (the non-negotiables)

1. **Follow the precedence rule.** User preference > existing project convention > preferred set >
   simpler contextual choice. State the reason when you diverge from the preferred set.
2. **Server data is not state.** Fetch and cache it with **TanStack Query**, not `useState` +
   `useEffect`, and not React Context. Context is for genuinely shared *client* state only.
3. **Validate at the boundaries with Zod** — env vars, form input, and untrusted external data —
   and derive the TS type with `z.infer`. Don't hand-write a type parallel to a schema.
4. **Prefer shadcn/ui components** (composed with `cn` + `cva`) over hand-rolled generic ones; add
   them via the CLI into `components/ui`. Don't recreate what a shadcn component already does.
5. **Type the `@/*` alias to `src/`** and import with it. Validate `import.meta.env` through
   `env.ts`; never read `import.meta.env` directly elsewhere.
6. **Match specificity to fragility** — a simple form needs no resolver; a trivial app needs no
   router. Add a library when it removes real repetition or risk, not by default.
