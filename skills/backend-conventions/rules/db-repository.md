---
title: Repository — interface + Prisma impl + in-memory fake
tags: repository, prisma, interface-segregation, testing
---

## Repository pattern

The contract is an **interface** at the root of `repositories/`, exposing only the methods the
use cases actually need (ISP). Two implementations: `prisma/` (real) and `in-memory/` (fake for
unit tests). Both implement the same interface (LSP), so a use case can't tell them apart.

```ts
// repositories/categories-repository.ts — contract
export interface CategoryInput { name: string; description: string | null; }

export interface CategoriesRepository {
  findMany(): Promise<Category[]>;
  findById(id: number): Promise<Category | null>;
  create(data: CategoryInput): Promise<Category>;
  update(id: number, data: CategoryInput): Promise<Category | null>; // null = not found
  delete(id: number): Promise<boolean>;                              // false = id didn't exist
}
```

```ts
// repositories/prisma/prisma-categories-repository.ts — translate "not found" to a return value
async update(id: number, data: CategoryInput) {
  try {
    return await prisma.category.update({ where: { id }, data });
  } catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2025") {
      return null; // record to update not found
    }
    throw error;
  }
}
```

Conventions:
- "Not found" is a **return value** (`null` / `false`), not a thrown error — the use case decides
  whether that's a `ResourceNotFoundError`.
- The in-memory fake mirrors behaviour exactly (auto-increment id, same sort order, same null/false
  semantics) and exposes `public items` for assertions.
- Repositories return the Prisma model type directly; no custom entity/mapper layer in this project.
