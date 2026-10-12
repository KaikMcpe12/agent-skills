---
title: Prompt template for extracting a skill
tags: prompt, template, extraction
---

## Prompt template

Copy, fill in `<PATH>`, and send. (Or just say: "use skill-forge to extract a skill from
`<PATH>`" — the skill workflow covers the rest.)

```
Extraia as convenções de código do projeto em <PATH> e gere uma skill reutilizável
para o meu repo agent-skills, seguindo a skill skill-forge.

- LEIA O CÓDIGO como fonte da verdade (não confie só em README/CLAUDE.md/docs;
  aponte divergências, o código vence). Distinga padrão observado de defeito
  acidental (typos, inconsistências) — ensine a convenção limpa e sinalize o defeito.
- A skill é um GUIA DE ESTILO genérico/reutilizável, não documentação do domínio
  do projeto. Nomes de entidade nos exemplos são placeholders.
- Cubra: estrutura de pastas, arquitetura/camadas, entidades/mappers, env, ORM/banco,
  validação, erros, auth, naming, testes, tooling, docs (Swagger/OpenAPI) — e um
  example-feature end-to-end com checklist.
- Formato: pasta skills/<nome>/ com SKILL.md (frontmatter name + description de QUANDO
  usar, 3ª pessoa) + rules/*.md (um tópico por arquivo, com exemplo de código real).
  Progressive disclosure: SKILL.md referencia cada rule UM nível de profundidade.
- Me pergunte (nome da pasta) antes de gerar se houver ambiguidade.
- NÃO faça commit/push sem eu pedir. Quando eu pedir, adicione a linha na tabela
  do README raiz.
```

### Variations

- **From an idea (no codebase):** replace the first line with a description of the conventions and
  add: "Marque o que você inferiu para eu confirmar."
- **Update an existing skill:** "Atualize a skill skills/<nome> para refletir as mudanças em
  <PATH>; releia o código real e ajuste os rules afetados e o SKILL.md."
