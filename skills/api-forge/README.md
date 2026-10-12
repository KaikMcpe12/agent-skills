# api-forge skill

My backend coding conventions, packaged as a reusable agent skill (Claude Code / opencode /
community-style). It teaches an agent to write backend code in **any project** the way I do:
**Fastify 5 + Prisma 6 + Zod 3**, a layered **controller → use case → repository** flow with rich
domain **entities + mappers**, manual DI through async factories, multi-provider DB chosen at
runtime by `env.TYPE_DB`, Bearer JWT (`@fastify/jwt` + bcrypt), OpenAPI generated from the Zod
route schemas, and Jest/Supertest tests.

## Layout

```
api-forge/
├── SKILL.md    # entry point: frontmatter (name/description) + rule index + golden rules
├── README.md   # this file
└── rules/      # one focused rule per topic, each with a real code example
    ├── arch-layers.md        arch-bootstrap.md     arch-factories.md     arch-entities.md
    ├── env-validation.md
    ├── db-prisma.md          db-schema.md          db-repository.md       db-mappers.md
    ├── http-routes.md        http-controllers.md   http-dto.md            http-errors.md
    ├── swagger-openapi.md
    ├── auth-jwt.md
    ├── naming-conventions.md testing.md            tooling.md
    └── example-feature.md    # full feature end-to-end (Project domain) + build checklist
```

## How to use

Agents that auto-discover skills read `SKILL.md`, then open the relevant `rules/*.md` on demand
(progressive disclosure — there is no compiled single-file copy to keep in sync). Install it like
any skill in this repo (see the root `README.md`).

## Maintenance

These conventions are distilled from a reference implementation (`backend_contacteme`). The code is
the source of truth: when the style evolves, re-read the real project and update the matching
`rules/*.md` (and `SKILL.md` if the stack or golden rules change). The rules are domain-agnostic —
entity names in examples (`Client`, `Conversation`, `Project`) are placeholders for whatever the
target project models. Teach the clean convention, not the reference's occasional typos
(see `rules/naming-conventions.md`).
