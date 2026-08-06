# Skills

Agent skills: [backend](./backend), [monorepo](./monorepo), [react](./react), [typescript](./typescript).

New skills start from [SKILL.template.md](./SKILL.template.md). [GUIDE.md](./GUIDE.md) keeps the links behind it. Most of the work happens in `backend` first; whatever survives there gets moved into the template.

## Done

`backend` covers:

- Layout: package per bounded context, layers, barrels, tsconfigs, file naming
- Value objects and identifiers
- Aggregates: boundary, command methods, entities, events
- Exceptions typed by layer
- Unit tests with no doubles
- Prisma per context

`monorepo` covers the repo `backend` sits in:

- Shape: `apps/*` and `packages/*`, a private root, scoped names
- pnpm workspaces: `workspace:*` edges, catalogs, install-script allowlist
- One root tsconfig, two per package, `build/` out, no path aliases
- nx: ordering off `^build`, caching that needs `cache` *and* `outputs`, continuous targets

`react` and `typescript` are thinner: components and state for one, explicit types and readability for the other.

## TODO

- [ ] Anti-patterns. Rules only show the right shape, but people write the near-miss. Collect the ones that actually happened.
- [ ] Verification checklist. Plenty of rules are mechanically checkable: `@nestjs` imported under `domain/`, a class in `exceptions/` without the `Exception` suffix, a source file with no `.spec.ts` next to it. Either prose or `scripts/verify.sh`.
- [ ] Exit criteria. _The implementation is complete only if_ ... starting with `domain/` having no framework dependency. Might end up merged with the checklist above.
- [ ] Generate the Reference Guide. The table is hand-written and has drifted already.
- [ ] Guard against drift. Nothing here is compiled or tested, and one stale example made it into real code. After changing a canonical pattern, grep its name across the references.
