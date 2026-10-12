# agent-skills

A collection of my agent skills — reusable conventions and guidelines for AI coding agents
(Claude Code, opencode, and other skill-aware tools). Each skill lives under `skills/` as a
self-contained folder with a `SKILL.md` entry point.

## Skills

| Skill | What it does |
|-------|--------------|
| [`api-forge`](skills/api-forge/) | My backend coding style for Node/TypeScript APIs — Fastify 5 + Prisma 6 + Zod 3, layered architecture (controller → use case → repository) with domain entities + mappers, async DI factories, multi-provider DB via `env.TYPE_DB`, Bearer JWT (`@fastify/jwt` + bcrypt), OpenAPI generated from Zod, Jest/Supertest. |
| [`skill-forge`](skills/skill-forge/) | Meta-skill: how to extract a reusable convention skill from a project (or idea) and package it for this repo. Say "use skill-forge to extract a skill from `<path>`". |

## Layout

```
agent-skills/            ← repo root
├── README.md            ← this index
└── skills/
    └── <skill-name>/
        ├── SKILL.md      ← entry point (frontmatter: name + description)
        ├── README.md     ← what it is / how to use
        └── rules/        ← one focused rule per topic, each with a code example
```

Agents that auto-discover skills read each `SKILL.md` (its `name` + `description` are the trigger),
then pull in the relevant `rules/*.md` on demand (progressive disclosure).

## Usage

The `skills/` folder maps to a consuming project's discovered skills path. Pick one:

```bash
# A) clone the whole collection as a project's .agents folder
git clone git@github.com:<user>/agent-skills.git .agents

# B) symlink a single skill into an existing skills folder
ln -s /path/to/agent-skills/skills/api-forge .agents/skills/api-forge

# C) Claude Code personal skills
ln -s /path/to/agent-skills/skills/api-forge ~/.claude/skills/api-forge
```

## Adding a skill

1. Create `skills/<kebab-name>/SKILL.md` with frontmatter `name` (matching the folder, lowercase
   letters/numbers/hyphens, ≤64 chars) and a `description` (≤1024 chars) that states *what it
   covers* and *when* to use it.
2. Keep granular guidance in `rules/*.md`, one topic per file with a real code example; reference
   each rule directly from `SKILL.md` (one level deep).
3. Add a row to the table above.

Use the [`skill-forge`](skills/skill-forge/) skill to generate a new skill folder from a project or
an idea following these conventions.
