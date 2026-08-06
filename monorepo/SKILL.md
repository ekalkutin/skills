---
name: monorepo
description: "How a TypeScript monorepo is set up here — pnpm workspaces with a catalog for shared versions, one root tsconfig every package extends, two tsconfigs per package, `build/` as the only output, and nx ordering and caching plain npm scripts. Use when starting a repo, adding a package or app to one, wiring one package to import another, or fixing a build that runs out of order, rebuilds what has not changed, or resolves an import to source instead of output."
---

# Monorepo

Names in the examples — the scope `@acme`, the library `orders`, the app `api` — are placeholders. Substitute the repo's own; nothing here depends on them, or on a particular framework.

## Rules

### Shape

- **The root holds no source.** `apps/*` are deployables, `packages/*` are libraries, and there is no third glob — an extra `libs/*` is a second name for `packages/*`, and drift starts the day someone guesses wrong. Delete the empty one from `pnpm-workspace.yaml` and from disk.
- The root `package.json` is `"private": true` with no `main`: it exists to hold the workspace, the devDependencies the root's own scripts need, and the scripts that fan out. Nothing imports the root, so nothing there describes an entry point.
- A root devDependency is only what a root script runs — the task runner, the compiler, `@types/node`. **What a package imports is that package's dependency**, declared in its own `package.json`, even when every package needs it.
- Every package is `@<scope>/<name>` under **one scope for the whole repo**, and the directory matches the name after the scope, so `@acme/orders` is findable at `packages/orders` without a lookup.
- The package manager is pinned once, in root `devEngines.packageManager` with `"onFail": "download"`, so a fresh clone runs the version the lockfile was written by.

### Workspace

- An internal dependency is `"@acme/orders": "workspace:*"` in the consumer's `dependencies`. That line is the edge the task graph is built from — a package nx builds first is one that some other package declared.
- **One version of a shared dependency lives in `catalogs:`**, and packages reference it as `catalog:<name>`. Two packages resolving two copies of the same framework is a class of bug that reads as a bug in your code, worth a config block to make impossible; the bump is then one line.
- `allowBuilds` names the exact dependencies permitted to run install scripts. The default is deny; adding a name is a decision, not a formality.
- The lockfile is committed, and it is the only place a transitive version is written down.

### TypeScript

- One root `tsconfig.json`, `compilerOptions` only — no `include`, no `files`, no `paths`. It is a base to extend, never a project to build.
- **No `paths` aliases pointing at another package's `src/`.** A package is resolved by node, through its `exports`, to its built output — so an import behaves identically inside the repo and after publishing, and a stale build fails loudly instead of being silently papered over by a source alias.
- Two tsconfigs per package: `tsconfig.json` includes everything so specs are type-checked, and `tsconfig.build.json` extends it and excludes `src/**/*.spec.ts` so they never reach the output. `exclude` is replaced rather than merged when a config extends another, so repeat `build` and `node_modules` there.
- `"type": "module"` with `"module": "NodeNext"` — every relative import carries a `.js` extension, naming the emitted file rather than the source: `./order.service.js`.
- **A library declares `exports`, emits declarations, and ships `files: ["build"]`; an app declares none of it.** Nothing imports an app, so `declaration`, `declarationMap` and an entry map on one are output nobody reads. Drop `"main"` from a library once `exports` exists — it points at a path that does not exist, and the tools that still read it are the ones that will break.
- `build/` is the output directory in every package, with the same name everywhere, so `.gitignore`, nx `outputs` and CI each say it once.
- `strict` and its family are set at the root and never relaxed in a package.

### Task graph

- **A target is an ordinary npm script; nx only orders and caches it.** `"build": "tsc -p tsconfig.build.json"` runs by hand, in CI, and under nx unchanged — which is what keeps the repo debuggable when the task runner misbehaves.
- `targetDefaults.build.dependsOn: ["^build"]` is the entire ordering rule: build my dependencies first. Do not restate it per package.
- **A cacheable target declares both `"cache": true` and `"outputs"`.** With neither, nx re-runs every task on every invocation and quietly reports `0/2 hit` — the caching that justified adding a task runner is off until you write those two keys.
- A target that does not exit is `"continuous": true` — `watch`, `dev`, `serve`. A continuous target `dependsOn` the one-shot `build`, so a watcher starts against output that already exists.
- Root scripts are fan-outs and nothing else: `nx run-many -t build`, `nx run-many -t watch dev`.

### Adding a package

1. `packages/<name>/package.json` — scoped name, `type: module`, `exports` at `build/`, `files`, the two scripts, dependencies through the catalog where one exists.
2. Copy the package's two tsconfigs verbatim; they extend the root and differ only in `rootDir`/`outDir`.
3. `src/index.ts` is the public surface — export what the outside may use, nothing internal.
4. Declare `"workspace:*"` in the consumer, install, then `nx run-many -t build` and read the order nx chose.

## Reference Guide

| Intent                                            | Reference                                   |
| ------------------------------------------------- | ------------------------------------------- |
| Standing a new repo up, in order, from empty      | [bootstrap.md](./references/bootstrap.md)   |
| Workspace globs, catalogs, `workspace:*`, builds  | [workspace.md](./references/workspace.md)   |
| Root and per-package tsconfigs, `exports`, output | [typescript.md](./references/typescript.md) |
| nx targets, dependsOn, cache, continuous          | [nx.md](./references/nx.md)                 |

## Not covered here

Deliberately absent rather than forgotten — ask before inventing a convention:

- CI: what runs on a pull request, and how the cache is shared between machines
- publishing and versioning (changesets, release tags), and whether anything is published at all
- Docker: which packages a build stage copies and in what order
- lint and format config, and the import rule that would enforce a layer boundary
- the test runner and how a spec is discovered
- TypeScript project references and `composite`, which this setup deliberately does not use
- nx plugins and inferred targets — every target here is written by hand
