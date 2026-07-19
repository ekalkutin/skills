---
name: react
description: React development conventions covering components, state management, styling, and project structure.
---

# React

## Rules

- Define props with a `type Props` alias, not `interface Props`.
- Mark props as `readonly` unless mutation is intentional.
- Prefer arrow functions with explicit `React.FC<Props>` typing.
- Destructure props inside the component body.
- Keep components small, composable, and easy to scan.
- Use the Reference Guide to find patterns matching your intent.

## Reference Guide

| Intent              | Reference                                 |
| ------------------- | ----------------------------------------- |
| Component structure | [component.md](./references/component.md) |
