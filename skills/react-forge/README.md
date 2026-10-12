# react-forge skill

My frontend coding conventions, packaged as a reusable agent skill (Claude Code / opencode /
community-style). It teaches an agent to build React web apps the way I do: **React 19 +
TypeScript + Vite + Tailwind CSS v4 + shadcn/ui**, with React Hook Form + Zod forms, TanStack Query
for server state, react-router for navigation, a typed `env.ts`, and Vitest/Testing Library/MSW/
Playwright tests. It defines a **preferred library set** and a **precedence rule** (user preference
→ existing project conventions → preferred set → contextual choice) so it guides without forcing one
architecture on every project.

It is the frontend member of this repo's `*-forge` family: `api-forge` (backend), `react-forge`
(frontend), `skill-forge` (authoring skills).

## Layout

```
react-forge/
├── SKILL.md    # entry point: precedence rule, preferred set, structure, rule index, golden rules
├── README.md   # this file
└── rules/      # one focused rule per topic, each with a real code example
    ├── project-structure.md   setup-vite-tailwind.md   shadcn-ui.md
    ├── components-and-hooks.md styling-tailwind.md
    ├── forms.md               env-validation.md
    ├── data-fetching.md       api-codegen-orval.md
    ├── routing.md             state-management.md       ui-states.md
    ├── testing.md
    └── example-feature.md     # a full feature end-to-end + build checklist
```

## How to use

Agents that auto-discover skills read `SKILL.md` (its `name` + `description` are the trigger), then
open the relevant `rules/*.md` on demand (progressive disclosure). Install it like any skill in this
repo (see the root `README.md`).

## Maintenance

These conventions are distilled from real projects (`pizzashop` is the backbone — the only one on
the exact Vite + React + Tailwind v4 + shadcn/ui stack; `devstage-web`, `dt-money`, and
`frontend-contacteme` contributed Orval, context/hooks, and `env.ts` ideas, adapted to the target
stack). The code is the source of truth: when the stack or style evolves, re-read the projects and
update the matching `rules/*.md`. Verify Tailwind/shadcn setup steps against the official docs, as
those change between versions.
