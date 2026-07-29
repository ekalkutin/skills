---
name: backend
description: "How backends are written here — a package per bounded context, layered domain/application/infrastructure/api, value objects as classes, aggregates, commands and queries that never chain, exceptions typed by layer, Prisma per context, and unit tests beside their source. Use when writing or reviewing any server-side TypeScript in this style: domain models, use cases, repositories, controllers, schemas, or their tests."
---

# Backend

## Rules

### Layout

- One package per bounded context; layers are its top-level folders: `domain/`, `application/`, `infrastructure/`, `api/`.
- `domain/` imports nothing from the other layers, and nothing from NestJS, the ORM or HTTP. No decorators on domain classes, ever.
- `api/` is how the outside world drives the context, one folder per transport (`http/`, `graphql/`, `ws/`, `consumers/`); `infrastructure/` is how the context drives the outside world. A transport translates a request into a command or query and does nothing else.
- A context owns its database — schema, client and migration history under `infrastructure/persistence/`, never in a shared package.
- Two shared _backend_ packages, and no third: **`kernel`** holds what a domain model extends (`AggregateRoot`, `EntityId`, `DomainEvent`, the exception bases) and has no dependencies whatsoever; **`application`** holds the machinery that runs a use case (transaction boundary, command bus, event dispatch) and may know NestJS. Neither ever holds a domain concept — something one context uses lives in that context. `api-contracts` is not a third: it is the wire, not the backend.
- Repository _interfaces_ live in `domain/repositories/`; `application/ports/` is only for what the domain cannot express — clock, id generator, outbound calls.
- Every folder that holds classes has a barrel `index.ts`, so a file imports another folder in one line instead of one per class. A layer folder — `domain/`, `application/`, `infrastructure/`, `api/` — has none: nothing imports a layer as a whole.
- **Import any folder but your own through its barrel, however far away it is; import a sibling file inside your own folder directly.** Going through your own barrel is a cycle.
- The package's own `index.ts` is different: it is the context's public surface, and exports only the module, its commands and queries, and public id types. Never aggregates, entities, value objects or repositories.
- kebab-case with a role suffix (`board.aggregate.ts`, `title.vo.ts`, `place-task.handler.ts`). A use case is a folder holding its command and handler together.

### Value objects

- A class with a `private constructor` and a `static create` that throws when the input breaks a rule — the only door in, so an invalid instance cannot exist. `of` instead of `create` where there is nothing to normalise.
- Add a method where the answer belongs to the value (`wipLimit.allows(n)`), not a getter that re-exposes `value`.
- Represent absence as a value (`Description.empty()`), not as `null`. A nullable source gets its own named constructor — `fromNullable` — returning that value; `create` and `of` never widen their signature to swallow `null`.
- Identifiers extend `EntityId`, validate nothing, and declare a private `__type` marker so they are nominal rather than interchangeable.
- Enumerations are classes too — named static instances plus an `of` that refuses the unknown. An aggregate's signatures speak in value objects only, never in a bare `string`.
- A value object never imports `api-contracts`: the rule has exactly one owner and this is it.

### Aggregates

- Extends `AggregateRoot<Id>`, `private constructor`, a `static create` that applies an event and a `static restore` that applies nothing and is called only by mappers.
- Draw the boundary around the **invariant**. If a rule spans several children they belong to one aggregate; otherwise keep aggregates small and reference by id.
- A command method takes the acting account **first**, returns what it made, and throws a `DomainException` when it refuses. Guard clauses come first, so a refusal leaves the aggregate untouched.
- State lives in `#private` fields behind readonly getters; collections are handed out `readonly`. Every aggregate carries a `version` that takes part in every write.
- Events are added only from inside via the protected `apply`. `getUncommittedEvents()` reads without clearing, `commit()` clears without publishing — an aggregate never publishes anything itself.
- Entities inside an aggregate are never loaded or saved on their own. Prefer immutable ones that return a new instance on change.

### Commands and queries

- **A command never dispatches another command, and a query never calls another query.** One request, one handler, no chains through the bus.
- A command handler loads, delegates, saves and returns an id; a query reads through a port and never loads an aggregate. Neither holds a business rule — that is the aggregate's job.

### Exceptions

- Three base classes, one per layer: **`DomainException`** (a business rule was refused), **`ApplicationException`** (the use case could not proceed — the aggregate named does not exist, the caller is not authenticated), **`InfrastructureException`** (the machinery failed).
- **Every class name ends in `Exception`** — `TaskNotFoundException`, never `TaskNotFound`. A throwable outside the three layers is a built-in (`Error`, `RangeError`), never a class of ours.
- All three carry a stable `code`, which is what a client matches on — never the class name.
- **A `DomainException` carries no status**: HTTP is a transport the domain knows nothing about, so the api layer maps `code` to one. The outer two do carry a status, defaulting to `400` and `500`.
- Absence _inside_ a loaded aggregate is a `DomainException`; absence _of_ the aggregate is an `ApplicationException`.
- Every refusal is thrown. There is no `Result` type and no error union in a signature.
- Exceptions live in an `exceptions/` folder of their own layer, grouped by subject (`board.exceptions.ts`) — never by what raised them.

### Unit tests

- One `.spec.ts` beside every file worth testing, named after it: `title.vo.ts` → `title.vo.spec.ts`.
- **No `testing/` folder and no shared fixture module.** A spec is self-contained; builders and helpers live in the spec that needs them.
- No test doubles: build the thing, call the method, assert on what came back. **Never mock your own classes** — hand-written doubles are for the clock, the id generator and outbound calls, and nothing else.
- Assert a refusal with `expect(() => …).toThrow(TheExceptionClass)`. Always name the class: a bare `toThrow()` also passes when the code throws for an unrelated reason.

### Comments

- **Obvious code gets no comment**, and most code here is obvious. A name that needs explaining is the wrong name — rename it, extract the method, or lift the condition into a value object. Same for tests: the test name is the sentence a comment wanted to be.
- No JSDoc, no file headers, no section banners, no commented-out code. The signature already says what a doc block would repeat.
- Comment only what the code cannot say — a workaround for someone else's bug, an order that looks arbitrary but is not. One line, above what it defends.

## Reference Guide

Ordered as the rules above are.

| Intent                                          | Reference                                         |
| ----------------------------------------------- | ------------------------------------------------- |
| Package and layer structure, contracts, naming  | [layout.md](./references/layout.md)               |
| Writing a value object, id, or enumeration      | [value-objects.md](./references/value-objects.md) |
| Writing an aggregate, its entities and events   | [aggregates.md](./references/aggregates.md)       |
| Commands, queries and their handlers            | [cqrs.md](./references/cqrs.md)                   |
| Exception classes, which layer each belongs to  | [exceptions.md](./references/exceptions.md)       |
| Unit tests, builders, what may be doubled       | [testing.md](./references/testing.md)             |
| Wiring a context to its database (Prisma)       | [prisma.md](./references/prisma.md)               |

## Not covered here

Deliberately absent rather than forgotten — ask before inventing a convention:

- repositories and aggregate-to-table mappers
- the command bus itself and how a handler is registered
- transaction handling
- the transactional outbox and integration events
- the api layer itself: controllers, the validation pipe, the exception filter and its `code → status` table
- any test that is not a unit test
