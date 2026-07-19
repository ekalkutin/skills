# Unit tests

One `.spec.ts` beside every file that has behaviour worth testing, named after it: `title.vo.ts` → `title.vo.spec.ts`, `board.aggregate.ts` → `board.aggregate.spec.ts`. There is no `testing/` folder and no shared fixture package — a spec is self-contained.

## Nothing is doubled

No mocks, no NestJS testing module, no database. Build the thing, call the method, assert on what came back.

```ts
it("appends to the end of the column, keeping positions contiguous", () => {
  const { board, owner, columnNamed } = aBoard();
  const todo = columnNamed("Todo");

  const first = board.place(owner, aTaskId(board.id), todo);
  const second = board.place(owner, aTaskId(board.id), todo);

  expect([first.position.value, second.position.value]).toEqual([0, 1]);
  expect(board.placementsIn(todo)).toHaveLength(2);
});
```

**Never mock your own classes.** `vi.mock`, `jest.mock` and `spyOn` against your own repositories, services or aggregates are banned: a mock freezes the current shape of a collaborator, so renaming a method leaves the tests green and the application broken. Hand-written doubles are allowed for exactly three things, all of them non-deterministic or external — the clock, the id generator, and outbound network calls.

## Asserting a refusal

Always name the error class. A bare `toThrow()` also passes when the code throws for a completely unrelated reason — a typo in a property name, for one — which is exactly the bug a test is supposed to catch.

```ts
it("refuses a column that is at its WIP limit", () => {
  const { board, owner, columnNamed } = aBoard({ columns: [["Doing", 1]] });
  const doing = columnNamed("Doing");
  board.place(owner, aTaskId(board.id), doing);

  expect(() => board.place(owner, aTaskId(board.id), doing)).toThrow(
    WipLimitExceededException,
  );
});
```

Where the particulars matter, assert on the message or catch the error and read its fields:

```ts
expect(() => board.place(owner, aTaskId(board.id), doing)).toThrow(
  /WIP limit of 1/,
);
```

A refusal should also leave the aggregate untouched, and that is worth its own test:

```ts
it("changes nothing when it refuses", () => {
  const { board, owner, columnNamed } = aBoard({ columns: [["Doing", 1]] });
  const doing = columnNamed("Doing");
  board.place(owner, aTaskId(board.id), doing);
  board.commit();

  expect(() => board.place(owner, aTaskId(board.id), doing)).toThrow(
    WipLimitExceededException,
  );

  expect(board.placementsIn(doing)).toHaveLength(1);
  expect(board.getUncommittedEvents()).toEqual([]);
});
```

## Builders

Where constructing a valid aggregate is noisy, build it in the spec that needs it — not in a shared fixture module.

```ts
function aBoard(
  options: { columns?: readonly (readonly [string, number | null])[] } = {},
) {
  const owner = AccountId.of(anId());
  const columns = (options.columns ?? [["Todo", null]]).map(
    ([name, wipLimit], order) => aColumn(name, wipLimit, order),
  );
  const board = Board.create(
    BoardId.of(anId()),
    BoardName.create("Delivery"),
    owner,
    columns,
  );

  return {
    board,
    owner,
    columnNamed: (name: string) => columnIdNamed(board, name),
  };
}
```

Ids are sequential rather than random, so a failure is reproducible and readable:

```ts
let sequence = 0;

function anId(): string {
  sequence += 1;
  return `0190a1b2-c3d4-7e5f-8a9b-${sequence.toString(16).padStart(12, "0")}`;
}
```

## Names

A test name states the rule, in the domain's language:

- "refuses a column that is at its WIP limit"
- "appends to the end of the column, keeping positions contiguous"
- "records nothing when restored from storage"

not "should return error" or "test place()".
