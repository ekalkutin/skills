# Value Objects

## The shape

```ts
import { InvalidTitleException } from "../exceptions/task.exceptions.js";

export class Title {
  static readonly MAX_LENGTH = 200;

  private constructor(public readonly value: string) {}

  static create(raw: string): Title {
    const trimmed = raw.trim();

    if (trimmed.length === 0) throw new InvalidTitleException("empty");
    if (trimmed.length > Title.MAX_LENGTH)
      throw new InvalidTitleException("too_long");

    return new Title(trimmed);
  }

  equals(other: Title): boolean {
    return this.value === other.value;
  }
}
```

`private constructor` plus a `static create` is the whole idea: one door in, and it checks the rule. A `Title` that breaks it cannot be constructed anywhere, including by a mapper reading a corrupted row.

`create` and `equals` are on almost every value object. `create` is called `of` where there is nothing to normalise — an id, a currency code.

## Identifiers

```ts
export abstract class EntityId {
  protected constructor(public readonly value: string) {}

  equals(other: this): boolean {
    return this.value === other.value;
  }

  toString(): string {
    return this.value;
  }

  toJSON(): string {
    return this.value;
  }
}
```

```ts
export class BoardId extends EntityId {
  declare private readonly __type: "BoardId";

  static of(value: string): BoardId {
    return new BoardId(value);
  }
}

/** A reference to another context: this one never loads an account, it only names one. */
export class AccountId extends EntityId {
  declare private readonly __type: "AccountId";

  static of(value: string): AccountId {
    return new AccountId(value);
  }
}
```

The `__type` marker makes ids nominal — without it they are all structurally `{ value: string }` and `move(taskId, columnId)` takes its arguments in either order. `declare` keeps it type-only, so no runtime field is added to the thousands of ids an aggregate may hold.

An id validates nothing: alone among value objects it has no rule of its own. Nor does it mint itself — a constructor falling back to `randomUUID()` makes every test non-deterministic, so ids come from an `IdGenerator` port.

## Methods beyond create and equals

Add one where the answer belongs to the value rather than to whoever holds it. Here the caller never has to think about what `null` means:

```ts
export class WipLimit {
  private constructor(public readonly value: number | null) {}

  static of(raw: number): WipLimit {
    if (!Number.isInteger(raw) || raw < 1)
      throw new InvalidWipLimitException(raw);

    return new WipLimit(raw);
  }

  static unlimited(): WipLimit {
    return new WipLimit(null);
  }

  /** Behaviour, not a getter for `value`. */
  allows(currentCount: number): boolean {
    return this.value === null || currentCount < this.value;
  }
}
```

Note `unlimited()`: a second named constructor for a second legitimate state. Represent absence as a value this way — `Description.empty()`, `WipLimit.unlimited()` — rather than as `null`, whenever callers would otherwise branch on it.

The source, however, is often genuinely nullable — a `wip_limit Int?` column, an optional field in a request. That gets a named constructor of its own rather than a wider `of`, which would hand every caller who does have a number a signature saying they might not:

```ts
static fromNullable(raw: number | null | undefined): WipLimit {
  return raw === null || raw === undefined ? WipLimit.unlimited() : WipLimit.of(raw);
}
```

Keep it to what the value genuinely knows. A value object is not a place to park helpers, and a pile of getters that only re-expose `value` means the behaviour belongs somewhere else.

## Enumerations are classes too

A string union would be lighter and is the wrong shape. Named instances of a class, with a private constructor like every other value object:

```ts
export class BoardRole {
  static readonly ADMIN = new BoardRole("admin");
  static readonly MEMBER = new BoardRole("member");

  private constructor(public readonly value: string) {}

  static get all(): readonly BoardRole[] {
    return [BoardRole.ADMIN, BoardRole.MEMBER];
  }

  static of(raw: string): BoardRole {
    const role = BoardRole.all.find((candidate) => candidate.value === raw);

    if (!role) throw new InvalidBoardRoleException(raw);

    return role;
  }

  get isAdmin(): boolean {
    return this.equals(BoardRole.ADMIN);
  }

  equals(other: BoardRole): boolean {
    return this.value === other.value;
  }
}
```

The reason is that **an aggregate's signatures speak in value objects only**. With a union, `addMember(actor, account, "member")` compiles from anywhere a plausible-looking string can be produced, and the domain has a bare `string` in the middle of it. With a class the only way to obtain a role is `BoardRole.MEMBER` or `BoardRole.of(raw)`, and the second one is the single place where an unknown value is refused.

It also gives the enumeration somewhere to keep its behaviour: `isAdmin` belongs to the role, not to every caller that compares against `"admin"`.

Use a static getter for `all` rather than a static field, so it cannot be evaluated before `ADMIN` and `MEMBER` are initialised.
