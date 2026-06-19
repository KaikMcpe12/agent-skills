# backend-conventions skill

My backend coding conventions, packaged as a reusable agent skill (Claude Code / opencode /
community-style). It teaches an agent to write backend code in **any project** the way I do:
**Fastify 5 + Prisma 7 + Zod 4**, layered **controller → use case → repository** with manual DI,
multi-provider DB (SQLite dev / PostgreSQL prod via driver adapters), Bearer JWT, and
Vitest/Supertest tests.

## Layout

```
backend-conventions/
├── SKILL.md            # entry point: frontmatter (name/description) + rule index + golden rules
├── SKILL-BACKEND.md    # everything compiled into one markdown file (paste-as-context)
├── README.md           # this file
└── rules/              # one focused rule per topic, each with a code example
    ├── arch-layers.md          arch-bootstrap.md       arch-factories.md
    ├── env-validation.md
    ├── db-prisma-setup.md      db-schema.md            db-repository.md
    ├── http-routes.md          http-controllers.md     http-validation.md
    ├── http-errors.md          http-response-envelope.md
    ├── auth-jwt.md
    ├── naming-conventions.md   testing.md              tooling.md
    └── example-feature.md      # full feature end-to-end (example entity) + build checklist
```

## How to use

- **Agents that auto-discover skills:** drop this folder under a discovered skills path
  (it lives at `/tny/.agents/skills/backend-conventions/`). The agent reads `SKILL.md`, then opens
  the relevant `rules/*.md` on demand.
- **Manual context:** paste `SKILL-BACKEND.md` (the compiled single file) into the conversation.

## Maintenance

These conventions were distilled from a reference implementation (`tny-backend`). When the style
evolves, update the matching `rules/*.md` and the corresponding section of `SKILL-BACKEND.md`.
The rules are domain-agnostic — entity names in examples (e.g. `Category`, `Product`) are just
placeholders for whatever the target project models.
