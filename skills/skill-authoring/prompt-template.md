---
title: Prompt template for extracting a skill
tags: prompt, template, extraction
---

## Prompt template

Copy, fill in `<PATH>`, and send. (Or just say: "use skill-authoring to extract a skill from
`<PATH>`" — the skill workflow covers the rest.)

```
Extraia as convenções de código do projeto em <PATH> e gere uma skill reutilizável
para o meu repo agent-skills, seguindo a skill skill-authoring.

- LEIA O CÓDIGO como fonte da verdade (não confie só em README/CLAUDE.md/docs;
  aponte divergências, o código vence).
- A skill é um GUIA DE ESTILO genérico/reutilizável, não documentação do domínio
  do projeto. Nomes de entidade nos exemplos são placeholders.
- Cubra: estrutura de pastas, arquitetura/camadas, env, ORM/banco, validação, erros,
  auth, naming, testes, tooling — e um example-feature end-to-end com checklist.
- Formato: pasta skills/<nome>/ com SKILL.md (frontmatter name+description de QUANDO
  usar) + rules/*.md (um tópico por arquivo, com exemplo de código real) +
  SKILL-<TOPIC>.md consolidado + README.
- Me pergunte (formato, nome da pasta) antes de gerar se houver ambiguidade.
- NÃO faça commit/push sem eu pedir. Quando eu pedir, adicione a linha na tabela
  do README raiz.
```

### Variations

- **From an idea (no codebase):** replace the first line with a description of the conventions and
  add: "Marque o que você inferiu para eu confirmar."
- **Quick/single-file:** add "Formato: apenas um SKILL-<TOPIC>.md consolidado, sem pasta rules/."
- **Update an existing skill:** "Atualize a skill skills/<nome> para refletir as mudanças em
  <PATH>; ajuste os rules afetados e o doc consolidado."
