# skill-forge skill

Meta-skill: how I want an agent to **extract a reusable coding-convention skill** from a project
(or an idea) and package it for this `agent-skills` repo. It defines the extraction workflow, the
output format, the frontmatter/naming rules, generalization rules, the quality bar, and a ready
prompt template.

## Layout

```
skill-forge/
├── SKILL.md            # the workflow + format + golden rules
├── prompt-template.md  # copy-paste prompt for "extract a skill from <path>"
└── README.md           # this file
```

## How to use

Just say, in any future session:

> "Use skill-forge to extract a skill from `<path>`."

or paste the template in `prompt-template.md`. The agent reads `SKILL.md`, runs the workflow
(explore → read code → reconcile → confirm name → generate → generalize → version on request), and
produces a new `skills/<name>/` folder matching the repo's conventions (the reference example is
`api-forge`).
