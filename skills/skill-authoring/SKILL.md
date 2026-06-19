---
name: skill-authoring
description: How I want agents to extract a reusable coding-convention skill from an existing project (or an idea) and package it for my agent-skills repo. Use this skill whenever I ask to "create/extract a skill" from a codebase, turn a project's conventions into a skill, or author a new skill folder. It defines the extraction workflow, the folder/file format, generalization rules, and the prompt template.
license: MIT
metadata:
  author: kaik
  version: "1.0.0"
---

# Skill Authoring

Meta-skill: the process and format for turning a project's coding conventions (or a described
idea) into a **reusable, domain-agnostic** skill that fits my `agent-skills` repo. Follow this
whenever I ask to extract/create a skill.

## The two inputs

- **From a project** (most common): I give a path. The **code is the source of truth** — read it,
  don't trust docs alone.
- **From an idea**: I describe conventions in prose. Capture them in the same format; mark
  anything you inferred so I can confirm.

## Workflow (follow in order)

1. **Explore.** List the directory tree (ignore `node_modules`, `dist`, `build`, `generated`,
   `coverage`, `.git`). Read `package.json`/manifests and config files to learn the stack.
2. **Read the code in bulk**, not just docs. Cover:
   - Infrastructure: app/server bootstrap, env loading, DB/ORM client, error handling, auth.
   - One full feature end-to-end (every layer it passes through).
   - Repositories/data access, tests, build/lint/format config, Dockerfile/CI.
3. **Reconcile.** Where README/CLAUDE.md/docs disagree with the code, **the code wins** — note the
   divergence explicitly in the skill (a short "the docs are stale about X" line).
4. **Confirm with me** before generating, using `AskUserQuestion`, when ambiguous:
   - Format: Vercel/community style (folder + `SKILL.md` + `rules/`), single markdown file, or both.
   - Skill folder name (kebab-case, generic — not tied to the project's name/domain).
5. **Generate** the files (see Format below).
6. **Generalize.** Strip domain coupling — entity/table names become placeholders
   (`Entity`/`Resource`/`Product` as an *example*, with a "replace with your entity" note). The
   skill teaches *style*, not the source project's domain model.
7. **Version only when I ask.** If I do: place under `skills/<name>/`, add a row to the repo root
   `README.md` table, commit (don't push unless told).

## Output format

Mirror the existing skills in this repo (`backend-conventions` is the reference):

```
skills/<skill-name>/
├── SKILL.md            # entry point — see frontmatter rules below
├── SKILL-<TOPIC>.md    # OPTIONAL: the whole skill compiled into one file (paste-as-context)
├── README.md           # what it is, layout, how to use, how to maintain
└── rules/              # one focused rule per topic, each with a real code example
    ├── <prefix>-<topic>.md
    └── example-feature.md   # a full feature end-to-end + a build checklist
```

### `SKILL.md` requirements
- **Frontmatter:** `name` (must equal the folder name), `description`, `license`, `metadata`.
- **`description` is a trigger**, not a summary: state *what it covers* AND *when to use it*
  ("Use this skill whenever writing/reviewing/refactoring …"). This is what an agent matches on.
- **Body:** short intro · Stack · Folder structure · a **Rule Index** table (rule → what it covers)
  · **Golden Rules** (the non-negotiables) · pointer to the compiled doc.

### `rules/*.md` requirements
- Frontmatter `title` + `tags`.
- One topic per file. Lead with the rule, then a **real, copy-pasteable code example** from (or
  faithful to) the source. Add a short "don't" / anti-pattern line where useful.
- Use a consistent prefix per area (e.g. `arch-`, `http-`, `db-`, `env-`, `auth-`, `test-`,
  `naming-`, `tooling-`).
- Always include an `example-feature.md` that builds one feature through every layer, ending with
  a reusable checklist.

## Quality bar

- **Faithful:** examples match how the code actually looks (naming, exports, casing, imports).
- **Generic:** no business domain leaks; conventions apply to any project of that type.
- **Actionable:** each rule tells the agent exactly what to do, with code — not vague advice.
- **Honest:** call out stale docs and intentional anti-patterns instead of papering over them.

## Prompt template (what I'll typically send)

See `prompt-template.md` in this folder — paste/adapt it, or just say
"use skill-authoring to extract a skill from `<path>`".
