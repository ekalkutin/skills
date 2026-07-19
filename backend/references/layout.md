# Layout

## A bounded context

```
packages/contexts/<context>/
  src/
    domain/
      aggregates/     board.aggregate.ts
      entities/       column.entity.ts  placement.entity.ts
      value-objects/  title.vo.ts  ids.vo.ts  wip-limit.vo.ts
      events/         board.events.ts
      exceptions/     board.exceptions.ts  task.exceptions.ts
      repositories/   board.repository.ts        (interfaces only)
    application/
      use-cases/
        place-task/   place-task.command.ts  place-task.handler.ts
      ports/          clock.port.ts  id-generator.port.ts
      exceptions/     aggregate-not-found.exceptions.ts
    infrastructure/
      persistence/    schema, migrations, generated client, repositories, mappers
    api/              inbound transports: how the outside world drives this context
      http/           controllers, validating with the schemas from api-contracts
      graphql/        resolvers
    <context>.module.ts
    index.ts
```

The dependency rule reads off the import path: an import of `../../infrastructure/...` inside `domain/` is visibly wrong, and a lint rule can enforce it.

One folder per transport under `api/` — `http/`, `graphql/`, `ws/`, `consumers/` for a queue. They differ only in how a request arrives; each one translates it into the same command or query and does nothing else. There is no business rule in this layer, and a second transport must not be a second implementation.

A context owns its database: its own schema, its own client, its own migration history, all inside `infrastructure/persistence/` and never in a shared package. That is what makes "a transaction never spans two contexts" something the wiring cannot express, rather than a rule anyone has to remember. Wiring it up is in [prisma.md](./prisma.md).

## Shared packages

Two backend ones, and the line between them is what they are allowed to know:

```
packages/shared/
  kernel/src/
    domain/       aggregate-root.ts  domain-event.ts  entity-id.ts
    exceptions/   domain-exception.ts  application-exception.ts  infrastructure-exception.ts
  application/src/
    transactions/ transaction-context.ts  transactional-command-bus.ts
    events/       domain-event-dispatcher.ts
    exceptions/   concurrency-conflict.exception.ts
```

**`kernel` is what a domain model is built out of** — the classes an aggregate, an id or an exception extends. It has no dependencies at all: no framework, no ORM, no I/O. That is what lets a context's `domain/` layer import it while still importing nothing else.

**`application` is the machinery that runs a use case** — the transaction boundary, the retry, the event dispatch. It knows NestJS, because a command bus is a framework thing, and it is imported by a context's `application/` and `infrastructure/`, never by its `domain/`.

Neither ever contains a domain concept. There is no `Money` in `kernel` and no `BoardRepository` in `application`: **something used by exactly one context lives in that context**, however reusable it looks. A shared package earns a class only when a second context needs it, and moving it there later costs one import rewrite — far less than untangling a shared package that has quietly become a second domain model.

## Contracts

`packages/api-contracts/` holds zod schemas and their inferred types, one file per endpoint. It sits outside `packages/shared/`, which is backend-only: `api/http/` validates with these schemas and a client imports the same file, so neither side can drift. It imports nothing from a context — that would drag NestJS and Prisma into a browser.

**Zod checks the shape, the value object checks the rule.** `z.string()` says a name arrived as a string; `BoardName.create` says whether it is a name. Copying `.min(1).max(100)` into the schema puts one rule in two places, and the copy here is the one nobody remembers to change — which is also why no value object may import this package.

## Barrels

Every folder that holds classes carries an `index.ts` re-exporting what it holds, so one import line replaces one per class:

```ts
// domain/value-objects/index.ts
export { BoardName } from "./board-name.vo.js";
export { BoardRole } from "./board-role.vo.js";
export { AccountId, BoardId, ColumnId, TaskId } from "./ids.vo.js";
```

```ts
// domain/aggregates/board.aggregate.ts
import { BoardMember, type Column, Placement } from "../entities/index.js";
import { NotAMemberException, WipLimitExceededException } from "../exceptions/index.js";
import { type BoardId, BoardRole, Position } from "../value-objects/index.js";
```

One rule keeps this safe: **any folder but your own is imported through its barrel, a sibling file inside your own folder is imported directly.** A file that reaches its own folder's `index.ts` imports itself in a circle, and with classes that means a half-initialised module at load time rather than a compile error.

A folder has one door, and distance does not open a second one — crossing a layer is not an exception:

```ts
// application/exceptions/aggregate-not-found.exceptions.ts
import type { BoardId, TaskId } from "../../domain/value-objects/index.js";
```

A layer folder gets no barrel at all, for the same reason: `domain/index.ts` would give every class in the layer a second legal path, and which one a file uses becomes a coin toss.

## The module surface

```ts
/**
 * The public surface of this context. Aggregates, value objects, entities and
 * repositories are deliberately absent: another context that could construct a
 * `Board` would make the boundary decorative.
 */
export { WorkModule } from "./work.module.js";
export { PlaceTaskCommand } from "./application/use-cases/place-task/place-task.command.js";
```

## Two tsconfigs per package

`tsconfig.json` includes everything, so specs are type-checked.
`tsconfig.build.json` excludes them, so they never reach the build output.

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["build", "node_modules", "src/**/*.spec.ts"]
}
```

`exclude` is replaced rather than merged when a config extends another, so anything the base excludes has to be repeated here.

## File naming

| Kind              | Name                                             |
| ----------------- | ------------------------------------------------ |
| Aggregate         | `board.aggregate.ts`                             |
| Entity            | `placement.entity.ts`                            |
| Value object      | `title.vo.ts`, `ids.vo.ts`                       |
| Events            | `board.events.ts`                                |
| Exceptions        | `board.exceptions.ts`                            |
| Repository        | `board.repository.ts`                            |
| Command / handler | `place-task.command.ts`, `place-task.handler.ts` |
| Port              | `clock.port.ts`                                  |
| Spec              | source name plus `.spec.ts`, beside the source   |

kebab-case throughout, with no exceptions — `task-id.vo.ts`, never `taskid.vo.ts`.
