---
title: Full feature end-to-end (Product domain)
tags: example, end-to-end, walkthrough, product
---

## Full feature, end-to-end

A complete new feature built following every other rule in this skill. Example domain:
**Product** (public listing/detail + admin create), reusing the `Product` model from
`db-schema`. Build order mirrors the dependency flow: repository → use case → factory →
controller → route → register → tests.

> Replace `Product`/`product` with your entity. Keep file/symbol names exactly as shown
> (`naming-conventions`).

### 1. Repository interface — `src/repositories/products-repository.ts`

```ts
import type { Product } from "../../generated/prisma";

export interface ProductInput {
  sku: string;
  name: string;
  description: string;
  price: number;
  promotional_price: number | null;
}

// ISP: only what the Product use cases need today.
export interface ProductsRepository {
  findManyActive(): Promise<Product[]>;
  findById(id: number): Promise<Product | null>;
  findBySku(sku: string): Promise<Product | null>;
  create(data: ProductInput): Promise<Product>;
}
```

### 2. Prisma impl — `src/repositories/prisma/prisma-products-repository.ts`

```ts
import { prisma } from "@/lib/prisma";
import type {
  ProductsRepository,
  ProductInput,
} from "@/repositories/products-repository";

export class PrismaProductsRepository implements ProductsRepository {
  findManyActive() {
    return prisma.product.findMany({
      where: { active: true },
      orderBy: { id: "asc" },
    });
  }

  findById(id: number) {
    return prisma.product.findUnique({ where: { id } });
  }

  findBySku(sku: string) {
    return prisma.product.findUnique({ where: { sku } });
  }

  create(data: ProductInput) {
    return prisma.product.create({ data });
  }
}
```

### 3. In-memory fake — `src/repositories/in-memory/in-memory-products-repository.ts`

```ts
import type { Product } from "../../../generated/prisma";
import type {
  ProductsRepository,
  ProductInput,
} from "@/repositories/products-repository";

export class InMemoryProductsRepository implements ProductsRepository {
  public items: Product[] = [];
  private nextId = 1;

  async findManyActive(): Promise<Product[]> {
    return this.items.filter((p) => p.active).sort((a, b) => a.id - b.id);
  }

  async findById(id: number): Promise<Product | null> {
    return this.items.find((p) => p.id === id) ?? null;
  }

  async findBySku(sku: string): Promise<Product | null> {
    return this.items.find((p) => p.sku === sku) ?? null;
  }

  async create(data: ProductInput): Promise<Product> {
    const product: Product = {
      id: this.nextId++,
      ...data,
      active: true,
      created_at: new Date(),
    };
    this.items.push(product);
    return product;
  }
}
```

### 4. Domain error — `src/use-cases/errors/sku-already-exists-error.ts`

```ts
export class SkuAlreadyExistsError extends Error {
  constructor() {
    super("A product with this SKU already exists");
    this.name = "SkuAlreadyExistsError";
  }
}
```

### 5. Use cases — `src/use-cases/products/`

```ts
// list-products.ts
import type { Product } from "../../../generated/prisma";
import type { ProductsRepository } from "@/repositories/products-repository";

interface ListProductsUseCaseResponse {
  products: Product[];
}

export class ListProductsUseCase {
  constructor(private productsRepository: ProductsRepository) {}

  async execute(): Promise<ListProductsUseCaseResponse> {
    const products = await this.productsRepository.findManyActive();
    return { products };
  }
}
```

```ts
// create-product.ts — business rule: SKU must be unique
import type { Product } from "../../../generated/prisma";
import type { ProductsRepository } from "@/repositories/products-repository";
import { SkuAlreadyExistsError } from "@/use-cases/errors/sku-already-exists-error";

interface CreateProductUseCaseRequest {
  sku: string;
  name: string;
  description: string;
  price: number;
  promotional_price?: number | null;
}

interface CreateProductUseCaseResponse {
  product: Product;
}

export class CreateProductUseCase {
  constructor(private productsRepository: ProductsRepository) {}

  async execute({
    sku,
    name,
    description,
    price,
    promotional_price,
  }: CreateProductUseCaseRequest): Promise<CreateProductUseCaseResponse> {
    const existing = await this.productsRepository.findBySku(sku);
    if (existing) {
      throw new SkuAlreadyExistsError();
    }

    const product = await this.productsRepository.create({
      sku,
      name,
      description,
      price,
      promotional_price: promotional_price ?? null,
    });

    return { product };
  }
}
```

### 6. Unit test — `src/use-cases/products/create-product.spec.ts`

```ts
import { beforeEach, describe, expect, it } from "vitest";
import { InMemoryProductsRepository } from "@/repositories/in-memory/in-memory-products-repository";
import { SkuAlreadyExistsError } from "@/use-cases/errors/sku-already-exists-error";
import { CreateProductUseCase } from "./create-product";

let productsRepository: InMemoryProductsRepository;
let sut: CreateProductUseCase;

describe("Create Product Use Case", () => {
  beforeEach(() => {
    productsRepository = new InMemoryProductsRepository();
    sut = new CreateProductUseCase(productsRepository);
  });

  it("should be able to create a product", async () => {
    const { product } = await sut.execute({
      sku: "TSHIRT-001",
      name: "Camiseta",
      description: "100% algodão",
      price: 79.9,
    });

    expect(product.id).toEqual(expect.any(Number));
    expect(product.active).toBe(true);
    expect(productsRepository.items).toHaveLength(1);
  });

  it("should not allow two products with the same SKU", async () => {
    await sut.execute({ sku: "TSHIRT-001", name: "A", description: "x", price: 10 });

    await expect(() =>
      sut.execute({ sku: "TSHIRT-001", name: "B", description: "y", price: 20 }),
    ).rejects.toBeInstanceOf(SkuAlreadyExistsError);
  });
});
```

### 7. Factory — `src/use-cases/factories/make-product-use-cases.ts`

```ts
import { PrismaProductsRepository } from "@/repositories/prisma/prisma-products-repository";
import { ListProductsUseCase } from "@/use-cases/products/list-products";
import { CreateProductUseCase } from "@/use-cases/products/create-product";

export function makeListProductsUseCase() {
  return new ListProductsUseCase(new PrismaProductsRepository());
}

export function makeCreateProductUseCase() {
  return new CreateProductUseCase(new PrismaProductsRepository());
}
```

### 8. Schemas — `src/http/controllers/products/schemas.ts`

```ts
import { z } from "zod";

export const productIdParamSchema = z.object({
  id: z.coerce.number().int().positive(),
});

export const productBodySchema = z.object({
  sku: z.string().min(1).max(60),
  name: z.string().min(1).max(120),
  description: z.string().min(1),
  price: z.number().positive(),
  promotional_price: z.number().positive().nullish(),
});

export const productSchema = z.object({
  id: z.number().int(),
  sku: z.string(),
  name: z.string(),
  description: z.string(),
  price: z.number(),
  promotional_price: z.number().nullable(),
  active: z.boolean(),
  created_at: z.date(),
});

export type ProductIdParam = z.infer<typeof productIdParamSchema>;
export type ProductBody = z.infer<typeof productBodySchema>;
```

### 9. Map the new domain error — `src/http/map-domain-error.ts`

```ts
import { SkuAlreadyExistsError } from "@/use-cases/errors/sku-already-exists-error";

export function mapDomainError(error: unknown, reply: FastifyReply) {
  if (error instanceof ResourceNotFoundError) {
    return reply.status(404).send({ error: { code: "NOT_FOUND", message: error.message } });
  }
  if (error instanceof SkuAlreadyExistsError) {
    return reply.status(409).send({ error: { code: "SKU_ALREADY_EXISTS", message: error.message } });
  }
  throw error;
}
```

### 10. Controllers — `src/http/controllers/products/`

```ts
// list.ts (public)
import type { FastifyReply, FastifyRequest } from "fastify";
import { makeListProductsUseCase } from "@/use-cases/factories/make-product-use-cases";

export async function list(_request: FastifyRequest, reply: FastifyReply) {
  const { products } = await makeListProductsUseCase().execute();
  return reply.status(200).send({ data: products });
}
```

```ts
// create.ts (admin) — can throw a domain error, so wrap + mapDomainError
import type { FastifyReply, FastifyRequest } from "fastify";
import { makeCreateProductUseCase } from "@/use-cases/factories/make-product-use-cases";
import { mapDomainError } from "@/http/map-domain-error";
import type { ProductBody } from "./schemas";

export async function create(
  request: FastifyRequest<{ Body: ProductBody }>,
  reply: FastifyReply,
) {
  const { sku, name, description, price, promotional_price } = request.body;

  try {
    const { product } = await makeCreateProductUseCase().execute({
      sku,
      name,
      description,
      price,
      promotional_price: promotional_price ?? null,
    });
    return reply.status(201).send({ data: product });
  } catch (error) {
    return mapDomainError(error, reply);
  }
}
```

### 11. Routes — `src/http/controllers/products/routes.ts`

```ts
import { z } from "zod";
import type { FastifyInstance } from "fastify";
import type { ZodTypeProvider } from "fastify-type-provider-zod";
import {
  dataResponse,
  errorResponseSchema,
  validationErrorResponseSchema,
} from "@/http/http-schemas";
import { verifyJwt } from "@/http/middlewares/verify-jwt";
import { list } from "./list";
import { create } from "./create";
import { productBodySchema, productSchema } from "./schemas";

const tags = ["Products"];

// Public: list active products.
export async function productsRoutes(app: FastifyInstance) {
  const router = app.withTypeProvider<ZodTypeProvider>();

  router.get(
    "/products",
    { schema: { tags, response: { 200: dataResponse(z.array(productSchema)) } } },
    list,
  );
}

// Admin: create a product (separate group, guarded by JWT).
export async function adminProductsRoutes(app: FastifyInstance) {
  const router = app.withTypeProvider<ZodTypeProvider>();

  router.addHook("onRequest", verifyJwt);

  router.post(
    "/admin/products",
    {
      schema: {
        tags: ["Admin — Products"],
        body: productBodySchema,
        response: {
          201: dataResponse(productSchema),
          400: validationErrorResponseSchema,
          401: errorResponseSchema,
          409: errorResponseSchema,
        },
      },
    },
    create,
  );
}
```

### 12. Register in `src/app.ts`

```ts
import { productsRoutes, adminProductsRoutes } from "@/http/controllers/products/routes";
// ...
app.register(productsRoutes);
app.register(adminProductsRoutes);
```

### 13. e2e test — `src/http/controllers/products/products.spec.ts`

```ts
import { afterAll, beforeAll, beforeEach, describe, expect, it } from "vitest";
import request from "supertest";
import { app } from "@/app";
import { prisma } from "@/lib/prisma";
import { resetDatabase } from "@/utils/test/reset-database";
import { createAndAuthenticate } from "@/utils/test/create-and-authenticate";

describe("Products e2e", () => {
  beforeAll(async () => { await app.ready(); });
  afterAll(async () => { await app.close(); });
  beforeEach(async () => { await resetDatabase(); });

  it("should list only active products", async () => {
    await prisma.product.createMany({
      data: [
        { sku: "A", name: "Ativo", description: "x", price: 10, active: true },
        { sku: "B", name: "Inativo", description: "y", price: 20, active: false },
      ],
    });

    const response = await request(app.server).get("/products");

    expect(response.statusCode).toBe(200);
    expect(response.body.data).toHaveLength(1);
    expect(response.body.data[0].sku).toBe("A");
  });

  it("should require auth to create a product", async () => {
    const response = await request(app.server)
      .post("/admin/products")
      .send({ sku: "C", name: "X", description: "z", price: 30 });

    expect(response.statusCode).toBe(401);
    expect(response.body.error.code).toBe("UNAUTHORIZED");
  });

  it("should create a product and reject a duplicate SKU", async () => {
    const { token } = await createAndAuthenticate(app);
    const body = { sku: "C", name: "X", description: "z", price: 30 };

    const created = await request(app.server)
      .post("/admin/products")
      .set("Authorization", `Bearer ${token}`)
      .send(body);
    expect(created.statusCode).toBe(201);

    const duplicate = await request(app.server)
      .post("/admin/products")
      .set("Authorization", `Bearer ${token}`)
      .send(body);
    expect(duplicate.statusCode).toBe(409);
    expect(duplicate.body.error.code).toBe("SKU_ALREADY_EXISTS");
  });
});
```

### Checklist for any new feature

1. `repositories/<x>-repository.ts` interface (ISP) + `prisma/` impl + `in-memory/` fake.
2. `use-cases/<domain>/*.ts` (one class each, `execute`, typed Request/Response) + domain errors.
3. Unit specs against the in-memory repo.
4. `factories/make-<domain>-use-cases.ts`.
5. `http/controllers/<domain>/schemas.ts` (Zod + inferred types).
6. Map any new domain error in `http/map-domain-error.ts` (and add the response status to routes).
7. Controllers (one file per action) + `routes.ts` (public/admin split as needed).
8. Register route groups in `app.ts`.
9. e2e spec; if the feature adds a model, add its `deleteMany` to `utils/test/reset-database.ts`
   (FK-safe order) and keep `docs/openapi.yaml` in sync.
