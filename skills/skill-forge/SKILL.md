---
name: skill-forge
description: Extracts a reusable, domain-agnostic coding-convention skill from an existing project (the code is the source of truth) or a described idea, and packages it for this agent-skills repo. Defines the extraction workflow, the SKILL.md + rules/ folder format, the frontmatter and naming rules, generalization rules, and the quality bar. Use when the user asks to create, extract, author, or update a skill, to turn a project's conventions into a skill folder, or to add a new skill to this repo.
license: MIT
metadata:
  author: kaik
  version: "2.0.0"
---

# skill-forge

Meta-skill: the process and format for turning a project's coding conventions (or a described
idea) into a **reusable, domain-agnostic** skill that fits my `agent-skills` repo. Follow this
whenever I ask to extract/create/update a skill. The reference example in this repo is
**`api-forge`** — mirror its shape.

## The two inputs

- **From a project** (most common): I give a path. The **code is the source of truth** — read it,
  don't trust docs alone.
- **From an idea**: I describe conventions in prose. Capture them in the same format; mark anything
  you inferred so I can confirm.

## Workflow (follow in order)

1. **Explore.** List the directory tree (ignore `node_modules`, `dist`, `build`, `generated`,
   `coverage`, `.git`). Read `package.json`/manifests and config files to learn the real stack and
   versions.
2. **Read the code in bulk**, not just docs. Cover:
   - Infrastructure: app/server bootstrap, env loading, DB/ORM client, error handling, auth.
   - One full feature end-to-end (every layer it passes through).
   - Repositories/data access, entities/mappers, tests, build/lint/format config.
3. **Reconcile.** Where README/CLAUDE.md/docs disagree with the code, **the code wins** — note the
   divergence explicitly (a short "the docs are stale about X" line). Distinguish an **observed
   pattern** from an accidental **defect** (typos, one-off inconsistencies): teach the clean
   intended convention and flag the defect; don't reproduce it, don't silently "fix reality".
4. **Confirm with me** (via `AskUserQuestion`) only when a choice can't be derived from evidence:
   - Skill folder name (see Naming below) and, when relevant, how deep to document optional/secondary
     paths.
5. **Generate** the files (see Format below).
6. **Generalize.** Strip domain coupling — entity/table names become placeholders (`Entity`,
   `Project` as an *example*, with a "replace with your entity" note). The skill teaches *style*,
   not the source project's domain model. Keep examples faithful to real casing/imports/signatures.
7. **Version only when I ask.** If I do: place under `skills/<name>/`, add a row to the repo root
   `README.md` table, commit (don't push unless told).

## Output format

Mirror `api-forge`:

```
skills/<skill-name>/
├── SKILL.md    # entry point — see frontmatter rules below
├── README.md   # what it is, layout, how to use, how to maintain
└── rules/      # one focused rule per topic, each with a real code example
    ├── <prefix>-<topic>.md
    └── example-feature.md   # a full feature end-to-end + a build checklist
```

- Prefer **SKILL.md + `rules/`** (progressive disclosure). A compiled single-file
  `SKILL-<TOPIC>.md` is **optional and discouraged** — it duplicates `rules/` and drifts out of
  sync. Only add one if I explicitly ask for a paste-as-context blob.

### `SKILL.md` requirements (per Anthropic's Agent Skills format)
- **Frontmatter — only `name` and `description` are required:**
  - `name`: must equal the folder name; **lowercase letters/numbers/hyphens only**, ≤64 chars; no
    XML tags; must not contain the reserved words `anthropic` or `claude`.
  - `description`: non-empty, ≤1024 chars; state **what it covers AND when to use it** ("Use when
    …"). This is the trigger Claude matches on — be specific, include key terms, write in the
    third person.
  - `license`/`metadata` are optional extras this repo keeps for provenance.
- **Body (keep under ~500 lines):** short intro · **Stack** · **Folder structure** · a **Rule
  Index** table (rule → what it covers) · **Golden Rules** (the non-negotiables). Reference each
  `rules/*.md` directly from here — **one level deep**, never a reference-to-a-reference.

### `rules/*.md` requirements
- Frontmatter `title` + `tags`.
- One topic per file. Lead with the rule, then a **real, copy-pasteable code example** from (or
  faithful to) the source. Add a short "don't" / anti-pattern line where useful.
- Use a consistent prefix per area (e.g. `arch-`, `http-`, `db-`, `env-`, `auth-`, `testing`,
  `naming-`, `tooling-`, `swagger-`).
- Always include an `example-feature.md` that builds one feature through every layer, ending with a
  reusable checklist.

### Naming (identity + clarity)
- Prefer **gerund** (`authoring-skills`) or clear **noun phrases** (`api-forge`); a consistent
  family (`*-forge`) is good. Avoid vague names (`helper`, `utils`, `tools`) and filler suffixes
  (`pro`, `ultimate`, `advanced`). The name can have identity; the **description** must carry the
  precise *what + when*.

## Quality bar

- **Faithful:** examples match how the code actually looks (naming, exports, casing, imports,
  versions).
- **Generic:** no business domain leaks; conventions apply to any project of that type.
- **Actionable:** each rule tells the agent exactly what to do, with code — not vague advice.
- **Concise:** the context window is shared; cut anything Claude already knows. Use consistent
  terminology throughout.
- **Honest:** call out stale docs, defects, and intentional anti-patterns instead of papering over
  them; separate observed practice from recommendation from proposal.

## Prompt template (what I'll typically send)

See `prompt-template.md` in this folder — paste/adapt it, or just say
"use skill-forge to extract a skill from `<path>`".
