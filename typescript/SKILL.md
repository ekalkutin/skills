# TypeScript

## Rules

- Prefer explicit types.
- Never use `any`.
- Prefer `readonly`.
- Enable strict mode.

## Good

```ts
interface User {
    readonly id: string;
}
```

## Bad

```ts
let user: any;
```
