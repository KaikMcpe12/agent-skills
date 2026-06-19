---
title: Layered architecture (controller → use case → repository)
tags: architecture, layers, dependency-inversion
---

## Layered architecture

Every feature flows: **route → controller → factory → use case → repository (interface) → Prisma impl**.
The use case depends on the repository **interface**, never on the concrete Prisma class — that
keeps business rules testable with an in-memory fake and DB-agnostic.

- **Controller** (`http/controllers/<domain>/<action>.ts`): reads validated input, calls the
  use case via its factory, maps domain errors to HTTP. No business rules, no Prisma.
- **Use case** (`use-cases/<domain>/<action>.ts`): one class, single `execute(...)`, a typed
  `*Request`/`*Response` pair, business rules + domain errors. Receives repositories via constructor.
- **Repository** (`repositories/*-repository.ts`): interface (contract). Concrete impl in
  `repositories/prisma/`, fake in `repositories/in-memory/`.

```ts
// use-cases/categories/update-category.ts
interface UpdateCategoryUseCaseRequest { id: number; name: string; description?: string | null; }
interface UpdateCategoryUseCaseResponse { category: Category; }

export class UpdateCategoryUseCase {
  constructor(private categoriesRepository: CategoriesRepository) {} // <- interface, injected

  async execute({ id, name, description }: UpdateCategoryUseCaseRequest):
    Promise<UpdateCategoryUseCaseResponse> {
    const category = await this.categoriesRepository.update(id, { name, description: description ?? null });
    if (!category) throw new ResourceNotFoundError(); // domain error, not an HTTP reply
    return { category };
  }
}
```

**Don't:** call `prisma.*` from a controller or use case; put business rules in a controller or
repository; have the use case import a `Prisma*Repository`.
