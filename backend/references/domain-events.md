# Domain events

`apply` puts an event in the aggregate, `save` moves it to a table, a relay publishes it. Nothing in between skips a step, and nothing publishes inside the transaction that produced the event.

## The event

Primitives on the way out. The aggregate carries value objects; `payload()` is where they stop, so the row stays readable by a consumer that has never seen `BoardId`.

```ts
export abstract class DomainEvent {
  abstract readonly type: string;
  abstract payload(): Record<string, unknown>;
}

export class TaskPlaced extends DomainEvent {
  override readonly type = "work.TaskPlaced";

  constructor(
    private readonly boardId: BoardId,
    private readonly taskId: TaskId,
  ) {
    super();
  }

  payload(): Record<string, unknown> {
    return { boardId: this.boardId.value, taskId: this.taskId.value };
  }
}
```

## The aggregate applies

The one manual call in the chain, and it sits with the mutation it describes — see [aggregates.md](./aggregates.md).

```ts
place(actor: AccountId, taskId: TaskId): void {
  if (!this.isMember(actor)) throw new NotAMemberException(actor, this.id);

  this.#taskIds.push(taskId);
  this.apply(new TaskPlaced(this.id, taskId));
}
```

## The sink writes rows

It takes the transaction from its caller and opens none of its own. That parameter is the whole guarantee: the row and the state change commit together or not at all.

```ts
@Injectable()
export class OutboxSink {
  async record(events: readonly DomainEvent[], tx: Tx): Promise<void> {
    if (events.length === 0) return;

    await tx.outboxEvent.createMany({
      data: events.map((event) => ({
        type: event.type,
        payload: event.payload(),
      })),
    });
  }
}
```

```prisma
model OutboxEvent {
  id          String    @id @default(uuid(7))
  type        String
  payload     Json
  occurredAt  DateTime  @default(now())
  publishedAt DateTime?
}
```

`occurredAt` is stamped by the database, which is how `new Date()` stays out of the domain.

## The repository hands them over

The only caller of `getUncommittedEvents()` and `commit()`, and the only place a transaction is opened.

```ts
@Injectable()
export class BoardRepository {
  constructor(
    private readonly prisma: WorkPrismaService,
    private readonly outbox: OutboxSink,
  ) {}

  async save(board: Board): Promise<void> {
    const events = board.getUncommittedEvents();

    await this.prisma.$transaction(async (tx) => {
      const { count } = await tx.board.updateMany({
        where: { id: board.id.value, version: board.version },
        data: { ...BoardMapper.toRow(board), version: board.version + 1 },
      });
      if (count === 0) throw new ConcurrentModificationException(board.id);

      await this.outbox.record(events, tx);
    });

    board.commit();
  }
}
```

`version` in the `WHERE` is why `updateMany` is used rather than `update`: nothing matched means someone else wrote first, and `count === 0` is how you find out.

`commit()` runs after the transaction resolves. Clearing before it means a rollback leaves an aggregate that has forgotten what it did.

## The relay publishes

Its own tick, outside every request. Delivery is at-least-once, so a subscriber that runs twice must end in the same place as one that ran once.

```ts
@Injectable()
export class OutboxRelay {
  constructor(
    private readonly prisma: WorkPrismaService,
    private readonly bus: EventBus,
  ) {}

  @Interval(500)
  async drain(): Promise<void> {
    const rows = await this.prisma.outboxEvent.findMany({
      where: { publishedAt: null },
      orderBy: { occurredAt: "asc" },
      take: 100,
    });

    for (const row of rows) {
      await this.bus.emit(row.type, row.payload);
      await this.prisma.outboxEvent.update({
        where: { id: row.id },
        data: { publishedAt: new Date() },
      });
    }
  }
}
```

## What reacts

The handler publishes nothing and knows nothing about the outbox — it loads, delegates, saves, as in [cqrs.md](./cqrs.md). Work that has to follow the write subscribes on the other side of it, which is the alternative to a command dispatching a command.

```ts
@CommandHandler(PlaceTaskCommand)
export class PlaceTaskHandler {
  constructor(private readonly boards: BoardRepository) {}

  async execute(command: PlaceTaskCommand): Promise<void> {
    const board = await this.boards.findById(command.boardId);
    if (!board) throw new BoardNotFoundException(command.boardId);

    board.place(command.actor, command.taskId);
    await this.boards.save(board);
  }
}

@Injectable()
export class NotifyOnPlacement {
  @OnEvent("work.TaskPlaced")
  async handle(payload: { boardId: string; taskId: string }): Promise<void> {
    await this.notifications.taskPlaced(payload.taskId);
  }
}
```

## The whole flow

```
handler   board.place()      apply()      event held in memory
handler   boards.save()      BEGIN  UPDATE board + INSERT outbox  COMMIT
                             board.commit()                       memory cleared
relay     every 500ms        SELECT unpublished → emit → mark published
subscriber                   reacts, outside the write transaction
```

Crash before the `COMMIT` and nothing happened. Crash after it and the relay still finds the row. Publish inside the transaction instead — `eventBus.publish()` in `save()` — and you get both failures the outbox exists to prevent: events announcing a write that then rolls back, and a write with no event because the process died a line later.

## Deliberately left out

Add each when the reason to arrives, not before:

- `FOR UPDATE SKIP LOCKED` on the relay's read, once more than one instance runs.
- A unit of work, once one command writes two aggregates. Until then `save()` is already the transaction boundary.
- Retries and a dead letter, once a subscriber can fail permanently rather than transiently.
