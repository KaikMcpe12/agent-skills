---
title: Domain entities — rich classes, the currency between layers
tags: architecture, entities, domain-model
---

## Domain entities

Each domain concept is a **class** in `src/entities/`, independent of Prisma. It holds a private
`props` object plus an identity, exposes **getters** (and setters only where mutation is a real
behavior), and carries domain behavior (`isDeleted()`, `delete()`, `enable()`, …). Repositories
produce and consume these entities; the Prisma row ↔ entity translation lives in a mapper
(see `db-mappers`).

The constructor takes the props and an optional id. Fields with sensible defaults are made
optional at the constructor boundary using the `Replace` helper, so callers don't pass
`dateadded`/`deleted` for a brand-new entity:

```ts
// src/helpers/replace.ts
export type Replace<T, R> = Omit<T, keyof R> & R;
```

```ts
// src/entities/client.ts
import { randomUUID } from "crypto";
import { Replace } from "../helpers/replace";

export interface ClientProps {
  name: string;
  email: string | null;
  status?: number | null;
  dateadded: Date;
  deleted: boolean;
}

export class Client {
  private _id: string;
  private props: ClientProps;

  constructor(props: Replace<ClientProps, { dateadded?: Date; deleted?: boolean }>, id?: string) {
    this._id = id ?? randomUUID();
    this.props = {
      ...props,
      dateadded: props.dateadded ?? new Date(),
      status: props.status ?? 1,
      deleted: props.deleted ?? false,
    };
  }

  public get id(): string { return this._id; }
  public get name(): string { return this.props.name; }
  public set name(value: string) { this.props.name = value; }
  public get email(): string | null { return this.props.email; }

  public isDeleted(): boolean { return this.props.deleted ?? false; }
  public delete() { this.props.deleted = true; }
  public enable() { this.props.status = 1; }
  public disable() { this.props.status = 0; }
}
```

Conventions:
- **Id** is a private `_id`, defaulted to `randomUUID()` when not supplied; expose it read-only.
- **Behavior over flags:** model soft-delete/status as methods (`delete()`, `enable()`), not raw
  field writes from the outside.
- Use cases build entities (`new Client({...})`) and pass them to repositories; they never build
  Prisma payloads directly.

**Don't** import `@prisma/client` types into an entity, and don't let controllers touch `props` —
go through getters/behavior.
