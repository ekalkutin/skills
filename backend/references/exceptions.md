# Exceptions

Three base classes, one per layer. Every exception extends one of them, and which one it extends is the answer to "whose fault is this".

| Base                      | Meaning                                                               | Raised by                                | Carries a status   |
| ------------------------- | --------------------------------------------------------------------- | ---------------------------------------- | ------------------ |
| `DomainException`         | A rule of the business was refused                                    | Value objects, aggregate command methods | **No**             |
| `ApplicationException`    | The use case could not proceed for a reason the domain does not model | Repositories, use case handlers          | Yes, default `400` |
| `InfrastructureException` | The machinery failed                                                  | Persistence, messaging, outbound calls   | Yes, default `500` |

Siblings, not a hierarchy — one `@Catch(DomainException, ApplicationException, InfrastructureException)` filter serves all three.

```ts
export abstract class ApplicationException extends Error {
  abstract readonly code: string;

  /** The caller asked for something that could not be acted on. */
  readonly status: number = 400;

  protected constructor(message: string) {
    super(message);
    this.name = new.target.name;
  }
}
```

`DomainException` is the same without the `status`; `InfrastructureException` the same with `500`.

## Status belongs to the outer layers

**A `DomainException` has no status.** HTTP is a transport, and the domain does not know it exists: `WipLimitExceededException` says a rule was refused, and whether that becomes a `409` or a `422` is a decision about how to answer a caller. The api layer holds a `code → status` table and makes it.

The outer two do carry one, because they already speak about the world outside the domain. The layer sets a default and an individual exception overrides it:

```ts
export class BoardNotFoundException extends ApplicationException {
  override readonly code = "BOARD_NOT_FOUND";
  override readonly status = 404;

  constructor(public readonly boardId: BoardId) {
    super(`No board ${boardId.value}`);
  }
}
```

The override is ordinary field initialisation: the base's initialiser runs first, the subclass's replaces it. Do not read `this.status` in a base constructor — the override has not been applied yet.

## Choosing between them

The line that matters is **domain versus application**: _was there anything to apply the rules to?_

- `ColumnNotFoundException` is a **domain** exception. The board _is_ loaded and simply has no such column — absence inside an aggregate is a rule.
- `BoardNotFoundException` is an **application** exception. The aggregate the command names does not exist, so no rule ever got a chance to run.
- `ConcurrencyConflictException` is an **application** exception too: the state moved under the use case, but nothing was misconfigured and no rule was broken.

`InfrastructureException` is narrower than it looks — a component that actually failed: the database unreachable, a message unpublishable, an outbound call timed out. If the request could have succeeded on a retry against healthy infrastructure, it is not one of these.

## Files

An `exceptions/` folder in the layer they belong to, grouped by subject:

```
domain/exceptions/       board.exceptions.ts   task.exceptions.ts
application/exceptions/  aggregate-not-found.exceptions.ts
```

Never grouped by what raised them: a "value object exceptions" file cuts across the domain in a way nothing else does, and leaves nowhere obvious for the next one to go. `InvalidTitleException` sits with the other Task rules; `InvalidWipLimitException` sits with the other Board rules.

## Bugs are none of the three

A value the code computed for itself and got wrong is not a layer exception. Throw a plain `Error` or `RangeError`, so it can never be mapped to a client-facing status:

```ts
static atIndex(index: number): Position {
  if (!Number.isInteger(index) || index < 0) {
    throw new RangeError(`Position index must be a non-negative integer, received ${index}`);
  }

  return new Position(index);
}
```

## Naming

- **Every class ends in `Exception`** — `TaskNotFoundException`, not `TaskNotFound`. A `catch` block, an import list and a stack trace then all say what they are holding.
- A throwable outside the three layers is a built-in — `Error`, `RangeError` — not a class of ours. Anything we define and name is one of the three.
- One class per rule. `InvalidTitleException("empty")` and `InvalidTitleException("too_long")` share a class because they are the same rule; `NotAMemberException` and `NotAnAdminException` do not.
- Carry the particulars as fields — ids, limits, the offending value — so a caller and a log line both have them without parsing the message.
- The `code` is `SCREAMING_SNAKE_CASE` and stable. Renaming the class must not change it, which is why clients match on `code` and never on the name.
