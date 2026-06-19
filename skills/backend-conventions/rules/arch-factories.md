---
title: Manual DI via make-*-use-cases factories
tags: architecture, dependency-injection, factories
---

## Factories (manual dependency injection)

The factory is the **only** place that knows the concrete repository implementation. It wires
the use case to the Prisma repository and returns the ready-to-use instance. One file per domain,
exporting one `make*UseCase` function per use case.

```ts
// use-cases/factories/make-category-use-cases.ts
import { PrismaCategoriesRepository } from "@/repositories/prisma/prisma-categories-repository";
import { CreateCategoryUseCase } from "@/use-cases/categories/create-category";
import { UpdateCategoryUseCase } from "@/use-cases/categories/update-category";

export function makeCreateCategoryUseCase() {
  return new CreateCategoryUseCase(new PrismaCategoriesRepository());
}

export function makeUpdateCategoryUseCase() {
  return new UpdateCategoryUseCase(new PrismaCategoriesRepository());
}
```

Controllers call the factory, never `new SomeUseCase(...)`:

```ts
const { category } = await makeUpdateCategoryUseCase().execute({ id, name, description });
```

**Why:** controllers and use cases stay agnostic of the persistence layer; swapping the
implementation (or injecting an in-memory fake in tests) touches only the factory/constructor.
