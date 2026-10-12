---
title: Layered architecture (controller → use case → repository)
tags: architecture, layers, dependency-inversion, entities
---

## Layered architecture

Every feature flows: **route → controller → async factory → use case → repository (interface) →
Prisma/in-memory impl**, and the data passed between layers is a **domain entity**, not a Prisma
row. The use case depends on the repository **interface**, never the concrete class — so business
rules are testable with an in-memory fake and independent of the provider.

- **Controller** (`http/controllers/<domain>/<action>-controller.ts`): reads validated input, calls
  the use case via its factory, maps domain errors to HTTP. No business rules, no Prisma.
- **Use case** (`use-cases/<domain>/<action>-use-case.ts`): one class, single `execute(...)`, a
  typed `*Request`/`*Response` pair, business rules + domain errors. Repositories injected via
  the constructor.
- **Repository** (`repositories/<name>-repository.ts`): the interface (contract). Concrete impl in
  `repositories/prisma/`, fake in `repositories/in-memory/`. Both speak **entities**.

```ts
// use-cases/conversations/get-conversation-by-id.ts
import { Conversation } from "../../entities/conversation";
import { ConversationRepository } from "../../repositories/conversation-repository";
import { ResourceNotFoundError } from "../errors/resource-not-found-error";

interface GetConversationByIdUseCaseRequest {
  conversationId: string;
  authenticatedUserId: string;
}
interface GetConversationByIdUseCaseResponse {
  conversation: Conversation;
}

export class GetConversationByIdUseCase {
  constructor(
    private conversationRepository: ConversationRepository, // <- interface, injected
    private verifyUserAccess: VerifyUserAccessUseCase,
  ) {}

  async execute({
    conversationId,
    authenticatedUserId,
  }: GetConversationByIdUseCaseRequest): Promise<GetConversationByIdUseCaseResponse> {
    const conversation = await this.conversationRepository.findById(conversationId);
    if (!conversation) throw new ResourceNotFoundError(); // domain error, not an HTTP reply

    await this.verifyUserAccess.execute({ userId: authenticatedUserId, companyId: conversation.companyId });
    return { conversation };
  }
}
```

**Don't:** call `prisma.*` from a controller or use case; import a `Prisma*Repository` into a use
case; return `@prisma/client` row types from a repository (return an entity — see `db-mappers`);
put business rules in a controller or repository.
