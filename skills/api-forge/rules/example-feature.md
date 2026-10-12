---
title: Full feature end-to-end (Project domain)
tags: example, end-to-end, walkthrough
---

## Full feature, end-to-end

A complete new feature built following every other rule. Placeholder domain: **Project** (create +
get by id). Build order mirrors the dependency flow: entity → mapper → repository → use case →
unit spec → factory → DTO → controller → route → register → e2e.

> Replace `Project`/`project` with your entity. Keep the file/symbol conventions
> (`naming-conventions`): kebab-case files, named exports, relative imports, English identifiers.

### 1. Entity — `src/entities/project.ts`

```ts
import { randomUUID } from "crypto";
import { Replace } from "../helpers/replace";

export interface ProjectProps {
  companyId: string;
  name: string;
  status?: number | null;
  dateadded: Date;
  deleted: boolean;
}

export class Project {
  private _id: string;
  private props: ProjectProps;

  constructor(props: Replace<ProjectProps, { dateadded?: Date; deleted?: boolean }>, id?: string) {
    this._id = id ?? randomUUID();
    this.props = { ...props, dateadded: props.dateadded ?? new Date(), status: props.status ?? 1, deleted: props.deleted ?? false };
  }

  public get id(): string { return this._id; }
  public get companyId(): string { return this.props.companyId; }
  public get name(): string { return this.props.name; }
  public set name(value: string) { this.props.name = value; }
  public isDeleted(): boolean { return this.props.deleted ?? false; }
  public delete() { this.props.deleted = true; }
}
```

### 2. Mapper — `src/repositories/prisma/mappers/prisma-project-mapper.ts`

```ts
import { Project } from "../../../entities/project";
import { Project as PrismaProject } from "@prisma/client";

export class PrismaProjectMapper {
  static toDomain(raw: PrismaProject): Project {
    return new Project(
      { companyId: raw.companyId, name: raw.projectName, status: raw.projectStatus, dateadded: raw.projectDateAdded, deleted: raw.projectDeleted === 1 },
      raw.projectId,
    );
  }
  static toPrisma(project: Project) {
    return {
      projectId: project.id,
      companyId: project.companyId,
      projectName: project.name,
      projectStatus: 1,
      projectDateAdded: new Date(),
      projectDeleted: project.isDeleted() ? 1 : 0,
    };
  }
}
```

### 3. Repository — interface + impls

```ts
// src/repositories/project-repository.ts
import { Project } from "../entities/project";

export interface ProjectRepository {
  findById(id: string): Promise<Project | null>;
  findByName(companyId: string, name: string): Promise<Project | null>;
  create(data: Project): Promise<Project>;
}
```

```ts
// src/repositories/prisma/prisma-project-repository.ts
import { Prisma, PrismaClient } from "@prisma/client";
import { ProjectRepository } from "../project-repository";
import { Project } from "../../entities/project";
import { PrismaProjectMapper } from "./mappers/prisma-project-mapper";

export class PrismaProjectRepository implements ProjectRepository {
  constructor(private prisma: PrismaClient | Prisma.TransactionClient) {}

  async findById(id: string): Promise<Project | null> {
    const row = await this.prisma.project.findUnique({ where: { projectId: id } });
    return row ? PrismaProjectMapper.toDomain(row) : null;
  }
  async findByName(companyId: string, name: string): Promise<Project | null> {
    const row = await this.prisma.project.findFirst({ where: { companyId, projectName: name, projectDeleted: 0 } });
    return row ? PrismaProjectMapper.toDomain(row) : null;
  }
  async create(data: Project): Promise<Project> {
    await this.prisma.project.create({ data: PrismaProjectMapper.toPrisma(data) });
    return data;
  }
}
```

```ts
// src/repositories/in-memory/in-memory-project-repository.ts
import { ProjectRepository } from "../project-repository";
import { Project } from "../../entities/project";

export class InMemoryProjectRepository implements ProjectRepository {
  private projects = new Map<string, Project>();

  async findById(id: string): Promise<Project | null> { return this.projects.get(id) ?? null; }
  async findByName(companyId: string, name: string): Promise<Project | null> {
    for (const p of this.projects.values()) if (p.companyId === companyId && p.name === name && !p.isDeleted()) return p;
    return null;
  }
  async create(data: Project): Promise<Project> { this.projects.set(data.id, data); return data; }
}
```

### 4. Domain error — `src/use-cases/errors/project-already-exists-error.ts`

```ts
export class ProjectAlreadyExistsError extends Error {
  constructor() {
    super("Project already exists");
  }
}
```

### 5. Use cases — `src/use-cases/projects/`

```ts
// create-project-use-case.ts
import { Project } from "../../entities/project";
import { ProjectRepository } from "../../repositories/project-repository";
import { ProjectAlreadyExistsError } from "../errors/project-already-exists-error";

interface CreateProjectUseCaseRequest { companyId: string; name: string; }
interface CreateProjectUseCaseResponse { project: Project; }

export class CreateProjectUseCase {
  constructor(private projectRepository: ProjectRepository) {}

  async execute({ companyId, name }: CreateProjectUseCaseRequest): Promise<CreateProjectUseCaseResponse> {
    const existing = await this.projectRepository.findByName(companyId, name);
    if (existing) throw new ProjectAlreadyExistsError();

    const project = await this.projectRepository.create(new Project({ companyId, name }));
    return { project };
  }
}
```

```ts
// get-project-by-id-use-case.ts
import { Project } from "../../entities/project";
import { ProjectRepository } from "../../repositories/project-repository";
import { ResourceNotFoundError } from "../errors/resource-not-found-error";

export class GetProjectByIdUseCase {
  constructor(private projectRepository: ProjectRepository) {}
  async execute({ id }: { id: string }): Promise<{ project: Project }> {
    const project = await this.projectRepository.findById(id);
    if (!project) throw new ResourceNotFoundError();
    return { project };
  }
}
```

### 6. Unit spec — `src/use-cases/projects/create-project.spec.ts`

```ts
import { InMemoryProjectRepository } from "../../repositories/in-memory/in-memory-project-repository";
import { ProjectAlreadyExistsError } from "../errors/project-already-exists-error";
import { CreateProjectUseCase } from "./create-project-use-case";

describe("Create Project Use Case", () => {
  let projectRepository: InMemoryProjectRepository;
  let sut: CreateProjectUseCase;

  beforeEach(() => {
    projectRepository = new InMemoryProjectRepository();
    sut = new CreateProjectUseCase(projectRepository);
  });

  it("should create a project", async () => {
    const { project } = await sut.execute({ companyId: "c1", name: "Alpha" });
    expect(project.id).toEqual(expect.any(String));
    expect(await projectRepository.findById(project.id)).not.toBeNull();
  });

  it("should reject a duplicate name in the same company", async () => {
    await sut.execute({ companyId: "c1", name: "Alpha" });
    await expect(() => sut.execute({ companyId: "c1", name: "Alpha" })).rejects.toBeInstanceOf(ProjectAlreadyExistsError);
  });
});
```

### 7. Factories — `src/use-cases/factories/`

```ts
// make-create-project-use-case.ts (write → { useCase, db })
import { env } from "../../env";
import { PrismaDB } from "../../lib/prisma";
import { FirebirdDB } from "../../lib/firebird";
import { PrismaProjectRepository } from "../../repositories/prisma/prisma-project-repository";
import { FirebirdProjectRepository } from "../../repositories/firebird/firebird-project-repository";
import { FirebirdService } from "../../services/firebird-service";
import { CreateProjectUseCase } from "../projects/create-project-use-case";

export async function makeCreateProjectUseCase() {
  let db, projectRepository;
  if (env.TYPE_DB === "sqlite") {
    db = await PrismaDB.startTransaction();
    projectRepository = new PrismaProjectRepository(db.prisma);
  } else {
    db = await FirebirdDB.startTransaction();
    projectRepository = new FirebirdProjectRepository(new FirebirdService(db));
  }
  return { createProjectUseCase: new CreateProjectUseCase(projectRepository), db };
}
```

```ts
// make-get-project-by-id-use-case.ts (read → use case)
export async function makeGetProjectByIdUseCase() {
  let projectRepository;
  if (env.TYPE_DB === "sqlite") {
    projectRepository = new PrismaProjectRepository(await PrismaDB.getConnection());
  } else {
    projectRepository = new FirebirdProjectRepository(new FirebirdService(await FirebirdDB.getConnection()));
  }
  return new GetProjectByIdUseCase(projectRepository);
}
```

### 8. DTOs — `src/http/dto/project/`

```ts
// project-dto.ts
import z from "zod";

export const createProjectDTO = z.object({ companyId: z.string().uuid(), name: z.string().min(1) });
export type CreateProjectDTO = z.infer<typeof createProjectDTO>;

export const getProjectParamsDTO = z.object({ id: z.string().uuid() });

export const projectResponseDTO = z.object({
  project: z.object({ id: z.string(), companyId: z.string(), name: z.string() }),
});
```

### 9. Controllers — `src/http/controllers/projects/`

```ts
// create-project-controller.ts (write: commit / rollback)
import { FastifyReply, FastifyRequest } from "fastify";
import { createProjectDTO } from "../../dto/project/project-dto";
import { makeCreateProjectUseCase } from "../../../use-cases/factories/make-create-project-use-case";
import { ProjectAlreadyExistsError } from "../../../use-cases/errors/project-already-exists-error";

export async function create(request: FastifyRequest, reply: FastifyReply) {
  const data = createProjectDTO.parse(request.body);
  const { createProjectUseCase, db } = await makeCreateProjectUseCase();

  try {
    const { project } = await createProjectUseCase.execute(data);
    await db.commitAndClose();
    return reply.status(201).send({ project: { id: project.id, companyId: project.companyId, name: project.name } });
  } catch (error) {
    await db.rollbackAndClose();
    if (error instanceof ProjectAlreadyExistsError) return reply.status(409).send({ message: error.message });
    throw error;
  }
}
```

```ts
// get-project-by-id-controller.ts (read)
export async function getProjectById(request: FastifyRequest, reply: FastifyReply) {
  const { id } = getProjectParamsDTO.parse(request.params);
  try {
    const useCase = await makeGetProjectByIdUseCase();
    const { project } = await useCase.execute({ id });
    return reply.status(200).send({ project: { id: project.id, companyId: project.companyId, name: project.name } });
  } catch (error) {
    if (error instanceof ResourceNotFoundError) return reply.status(404).send({ message: error.message });
    throw error;
  }
}
```

### 10. Routes — `src/http/controllers/projects/routes.ts`

```ts
import { FastifyInstance } from "fastify";
import { ZodTypeProvider } from "fastify-type-provider-zod";
import { z } from "zod";
import { verifyJWT } from "../../middleware/verify-jwt";
import { createProjectDTO, getProjectParamsDTO, projectResponseDTO } from "../../dto/project/project-dto";
import { create } from "./create-project-controller";
import { getProjectById } from "./get-project-by-id-controller";

export async function projectRoutes(app: FastifyInstance) {
  app.withTypeProvider<ZodTypeProvider>().post("/projects", {
    preHandler: [verifyJWT],
    schema: {
      tags: ["Projects"],
      body: createProjectDTO,
      headers: z.object({ authorization: z.string().describe("Bearer token (required)").optional() }),
      response: { 201: projectResponseDTO, 409: z.object({ message: z.string() }), 401: z.object({ message: z.string() }) },
    },
  }, create);

  app.withTypeProvider<ZodTypeProvider>().get("/projects/:id", {
    preHandler: [verifyJWT],
    schema: {
      tags: ["Projects"],
      params: getProjectParamsDTO,
      headers: z.object({ authorization: z.string().describe("Bearer token (required)").optional() }),
      response: { 200: projectResponseDTO, 404: z.object({ message: z.string() }), 401: z.object({ message: z.string() }) },
    },
  }, getProjectById);
}
```

### 11. Register in `src/app.ts`

```ts
import { projectRoutes } from "./http/controllers/projects/routes";
// ...
app.register(projectRoutes);
```

### 12. e2e spec — `src/http/controllers/projects/create-project-controller.spec.ts`

```ts
import request from "supertest";
import { app } from "../../../app";
import { env } from "../../../env";
import { DatabaseCleaner, DatabaseCleanerFactory } from "../../../../test/factories/database-cleaner-factory";

describe("Create Project (e2e)", () => {
  let cleanerDb: DatabaseCleaner;
  beforeAll(async () => {
    cleanerDb = env.TYPE_DB === "firebird" ? await DatabaseCleanerFactory.createFirebird() : await DatabaseCleanerFactory.createPrisma();
    await app.ready();
  });
  beforeEach(async () => { await cleanerDb.cleanDatabase(); });
  afterAll(async () => { await app.close(); });

  it("should create a project", async () => {
    const { body } = await request(app.server).post("/users").send({ name: "U", email: "u@ex.com", password: "123456" });
    const response = await request(app.server)
      .post("/projects")
      .set("Authorization", `Bearer ${body.token}`)
      .send({ companyId: "11111111-1111-1111-1111-111111111111", name: "Alpha" });
    expect(response.statusCode).toEqual(201);
  });
});
```

### Checklist for any new feature

1. `entities/<x>.ts` (rich class) + `repositories/prisma/mappers/prisma-<x>-mapper.ts`.
2. `repositories/<x>-repository.ts` interface (ISP, entities) + `prisma/` impl + `in-memory/` fake.
3. `use-cases/<domain>/*-use-case.ts` (one class, `execute`, typed Request/Response) + domain errors.
4. Unit spec against the in-memory repo.
5. `factories/make-*-use-case.ts` (async; `env.TYPE_DB` switch; writes → `{ useCase, db }`).
6. `http/dto/<domain>/*-dto.ts` (body/params/query/response, `z.infer` types).
7. Controllers (one per action; writes commit/rollback) + `routes.ts` (per-route schema + auth).
8. `app.register(<domain>Routes)` in `app.ts`.
9. e2e spec using `DatabaseCleanerFactory`; add the new model to the Prisma schema (`db-schema`).
