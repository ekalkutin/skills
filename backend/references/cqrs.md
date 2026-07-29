# CQRS

Every request into a context arrives as a command or a query, one handler answers it, and neither kind calls its own kind.

## Where things live

A use case is a folder holding the message and its handler together, commands and queries side by side:

```
application/
  use-cases/
    place-task/   place-task.command.ts  place-task.handler.ts  place-task.handler.spec.ts
    get-board/    get-board.query.ts     get-board.handler.ts
  ports/          board-view.port.ts
```

The command and the query are the only two things a context exports besides its module and its public id types — see [layout.md](./layout.md).

## A command

Data, and nothing else. No behaviour, no defaults, no validation:

```ts
export class PlaceTaskCommand {
  constructor(
    public readonly actor: AccountId,
    public readonly boardId: BoardId,
    public readonly taskId: TaskId,
    public readonly columnId: ColumnId,
  ) {}
}
```

It speaks in value objects, so a command that exists is a command whose ids are real. The transport builds it: `api/http/` validates the request with its zod schema, turns the strings into value objects, and dispatches. That is also why the package's `index.ts` exports the id types — a caller needs them to construct the command.

## A command handler

Load, delegate, save:

```ts
@CommandHandler(PlaceTaskCommand)
export class PlaceTaskHandler {
  constructor(private readonly boards: BoardRepository) {}

  async execute(command: PlaceTaskCommand): Promise<PlacementId> {
    const board = await this.boards.findById(command.boardId);
    if (!board) throw new BoardNotFoundException(command.boardId);

    const placement = board.place(command.actor, command.taskId, command.columnId);
    await this.boards.save(board);

    return placement.id;
  }
}
```

Three lines of orchestration and no business rule: whether the placement is allowed is the board's decision, and the handler never checks a WIP limit or a membership itself. What it does own is the absence of the aggregate — an `ApplicationException`, per [exceptions.md](./exceptions.md).

It returns an id. Not the aggregate, not a view — whoever needs to render the board asks for it with a query.

## A query handler

A query never loads an aggregate. An aggregate exists to refuse writes, and rebuilding one to read three fields pays for every invariant it protects and uses none of them:

```ts
@QueryHandler(GetBoardQuery)
export class GetBoardHandler {
  constructor(private readonly view: BoardViewPort) {}

  async execute(query: GetBoardQuery): Promise<BoardView> {
    const board = await this.view.byId(query.boardId, query.actor);
    if (!board) throw new BoardNotFoundException(query.boardId);

    return board;
  }
}
```

The read goes through a port in `application/ports/`, implemented in `infrastructure/persistence/` against the same database as the writes. What comes back is a plain readonly type shaped for the caller — never an aggregate, an entity or a repository result.

## Neither kind chains

**A command never dispatches another command. A query never calls another query.**

A command inside a command hides a second transaction boundary inside the first, and the failure that follows belongs to neither use case. A query inside a query is the same read done twice, or an N+1 nobody can see from either handler.

So when two handlers need the same work:

- the rule belongs to the aggregate, or to a domain service both handlers call;
- a command that needs to read goes to its repository, never to a query handler;
- a query does its own single read through its own port.

Work that has to follow a command reacts to the event the aggregate applied, on the other side of the write — not to a second dispatch inside the handler.

## What a handler may not become

A handler that has grown a rule is the rule in the wrong place. Two shapes to catch early:

```ts
if (board.placementsIn(columnId).length >= column.wipLimit.value) throw ...
```

The check reads the board's state to decide something about the board: it is a command method on `Board`, and leaving it here means the next caller forgets it.

```ts
const board = await this.boards.findById(id);
return { id: board.id.value, name: board.name.value };
```

An aggregate loaded to be flattened into a view. That is a query, and it wants a port and a read model.
