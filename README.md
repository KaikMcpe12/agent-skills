# agent-skills

A collection of my agent skills — reusable conventions and guidelines for AI coding agents
(Claude Code, opencode, and other `.agents`-aware tools). Each skill lives under `skills/` as a
self-contained folder with a `SKILL.md` entry point.

## Skills

| Skill | What it does |
|-------|--------------|
| [`backend-conventions`](skills/backend-conventions/) | My backend coding style for Node/TypeScript APIs — Fastify 5 + Prisma 7 + Zod 4, layered architecture (controller → use case → repository), manual DI, multi-provider DB, Bearer JWT, Vitest/Supertest. |
| [`skill-authoring`](skills/skill-authoring/) | Meta-skill: how to extract a reusable convention skill from a project (or idea) and package it for this repo. Say "use skill-authoring to extract a skill from `<path>`". |

## Layout

```
agent-skills/            ← repo root
├── README.md            ← this index
└── skills/
    └── <skill-name>/
        ├── SKILL.md      ← entry point (frontmatter: name + description)
        └── ...           ← rules/, examples, compiled docs
```

## Usage

The `skills/` folder maps to `.agents/skills/` in a consuming project. Pick one:

```bash
# A) clone the whole collection as a project's .agents folder
git clone git@github.com:<user>/agent-skills.git .agents

# B) symlink a single skill into an existing .agents/skills
ln -s /path/to/agent-skills/skills/backend-conventions .agents/skills/backend-conventions

# C) Claude Code personal skills
ln -s /path/to/agent-skills/skills/backend-conventions ~/.claude/skills/backend-conventions
```

Agents that auto-discover skills read each `SKILL.md`, then pull in the relevant `rules/*.md`
on demand.

## Adding a skill

1. Create `skills/<kebab-name>/SKILL.md` with frontmatter `name` (matching the folder) and a
   `description` that states *when* to use it.
2. Keep granular guidance in `rules/*.md`, one topic per file with a code example.
3. Add a row to the table above.
