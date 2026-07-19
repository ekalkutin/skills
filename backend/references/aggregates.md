# Aggregates

## The base class

Small on purpose. Note what is _missing_: there is no `publish`, so an aggregate cannot send anything anywhere.

```ts
export abstract class AggregateRoot<TEntityId extends EntityId> {
  #events: DomainEvent[] = [];

  protected constructor(
    public readonly id: TEntityId,
    /** Part of the WHERE clause of every update. */
    public readonly version: number,
  ) {}

  protected apply(event: DomainEvent): void {
    this.#events.push(event);
  }

  /** Reading does not clear: a repository may look before it decides. */
  getUncommittedEvents(): readonly DomainEvent[] {
    return this.#events;
  }

  /** Forgets the events. Publishes nothing — it cannot. */
  commit(): void {
    this.#events = [];
  }
}
```

`apply` is `protected`, so an event can only be added by the aggregate itself, and `commit()` means only "they have been handed over, forget them" — publishing belongs to the repository, which is also its only caller: `getUncommittedEvents()` then `commit()`, inside `save()`.

Write this base class rather than inheriting one. A ready-made aggregate root brings a public `apply`, a `publish` and an auto-commit switch, and any of the three will send events past whatever delivery guarantee the system relies on.

## Boundary

Draw it around the invariant. A WIP limit spans every card in a column, so the board owns the columns and the placements; a task's title spans nothing, so the task is its own aggregate and the placement refers to it by id.

The consequence to accept deliberately: an aggregate is the unit of concurrency, so two people changing unrelated parts of one board still conflict.

## The shape

```ts
export type BoardState = {
  readonly id: BoardId;
  readonly version: number;
  readonly name: BoardName;
  readonly columns: readonly Column[];
  readonly placements: readonly Placement[];
  readonly members: readonly BoardMember[];
};

export class Board extends AggregateRoot<BoardId> {
  readonly #name: BoardName;
  readonly #columns: readonly Column[];
  readonly #placements: Placement[];
  readonly #members: BoardMember[];

  private constructor(state: BoardState) {
    super(state.id, state.version);
    this.#name = state.name;
    this.#columns = [...state.columns].sort(
      (left, right) => left.order - right.order,
    );
    this.#placements = [...state.placements];
    this.#members = [...state.members];
  }

  static create(
    id: BoardId,
    name: BoardName,
    owner: AccountId,
    columns: readonly Column[],
  ): Board {
    if (columns.length === 0) throw new BoardMustHaveAColumnException();

    const board = new Board({
      id,
      version: 0,
      name,
      columns,
      placements: [],
      members: [new BoardMember(owner, BoardRole.ADMIN)],
    });

    board.apply(new BoardCreated(id, owner));

    return board;
  }

  /** Rebuilds from storage. Only mappers call this, and it applies nothing. */
  static restore(state: BoardState): Board {
    return new Board(state);
  }

  get columns(): readonly Column[] {
    return this.#columns;
  }
}
```

`create` applies an event; `restore` never does. Getting that backwards republishes the entire history every time a row is read.

A factory with no rule left to break simply returns the aggregate:

```ts
static create(id: TaskId, boardId: BoardId, title: Title, description: Description): Task {
  const task = new Task({ id, version: 0, boardId, title, description, assignee: null });
  task.apply(new TaskCreated(id, boardId, title));
  return task;
}
```

## Command methods

The acting account comes first, every time. Authorisation lives here because the rule needs the aggregate's own state to decide — a guard would have to load the board separately and rule on a different copy of it.

```ts
place(actor: AccountId, taskId: TaskId, columnId: ColumnId): Placement {
  if (!this.isMember(actor)) throw new NotAMemberException(actor, this.id);

  const column = this.#columns.find((candidate) => candidate.id.equals(columnId));
  if (!column) throw new ColumnNotFoundException(columnId, this.id);

  if (this.#placements.some((placement) => placement.taskId.equals(taskId))) {
    throw new TaskAlreadyPlacedException(taskId, this.id);
  }

  const occupied = this.placementsIn(columnId).length;
  if (!column.wipLimit.allows(occupied)) {
    throw new WipLimitExceededException(columnId, column.wipLimit);
  }

  const placement = new Placement(taskId, columnId, Position.atIndex(occupied));
  this.#placements.push(placement);
  this.apply(new TaskPlaced(this.id, taskId, columnId, placement.position));

  return placement;
}
```

Guard clauses first, mutation last, event applied with the mutation. Nothing is changed before every check has passed, so a refused command leaves the aggregate exactly as it found it.

## Entities inside an aggregate

Immutable, and never loaded or saved on their own. Returning a new instance on change keeps the aggregate's collection straightforward to diff when it is written.

```ts
export class Placement {
  constructor(
    public readonly taskId: TaskId,
    public readonly columnId: ColumnId,
    public readonly position: Position,
  ) {}

  movedTo(columnId: ColumnId, position: Position): Placement {
    return new Placement(this.taskId, columnId, position);
  }
}
```

## Events

Internal to the context, so they may carry value objects and their shape is free to change. No timestamp: that is stamped by infrastructure when the event is written, which keeps `new Date()` out of the domain.

```ts
export abstract class DomainEvent {
  abstract readonly type: string;
}

export class TaskPlaced extends DomainEvent {
  override readonly type = "work.TaskPlaced";

  constructor(
    public readonly boardId: BoardId,
    public readonly taskId: TaskId,
    public readonly columnId: ColumnId,
    public readonly position: Position,
  ) {
    super();
  }
}
```
