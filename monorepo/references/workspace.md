# Workspace

`pnpm-workspace.yaml` is the whole configuration: which directories are packages, which versions are shared, which installs may run code.

```yaml
packages:
  - apps/*
  - packages/*

allowBuilds:
  esbuild: true
  nx: true

catalogs:
  react19:
    react: "19.2.0"
    react-dom: "19.2.0"
```

`@acme`, `orders` and `api` below stand in for the repo's own scope and package names.

## Globs

Two, and the distinction is deployability: `apps/*` is started, `packages/*` is imported. A glob listed but empty is worse than absent — `libs/*` sitting next to `packages/*` gives the next library two legal homes, and which one it lands in becomes a coin toss. Delete it from the yaml and from disk together, or the directory alone will re-suggest it.

The glob is one level deep. Grouping under `packages/contexts/*` means adding that glob explicitly; pnpm does not recurse.

## Internal dependencies

```json
"dependencies": {
  "@acme/orders": "workspace:*"
}
```

`workspace:*` means *the copy in this repo, whatever version it claims*. pnpm symlinks `node_modules/@acme/orders` to `packages/orders`, so the import resolves through the package's own `exports` — the same path a published consumer would take, which is what makes "it worked in the monorepo" a meaningful signal.

The line does double duty: it is also the edge nx orders builds along. A package that is built too late is usually a package nobody declared.

**Declare it in the layer that imports it.** A dependency only the app needs does not belong in the library, and the reverse mistake — the app declaring what the library imports — survives until the library is used somewhere else.

## Catalogs

A catalog is a named set of versions; a package references an entry with `catalog:<name>` instead of a range:

```json
"dependencies": {
  "react": "catalog:react19",
  "react-dom": "catalog:react19"
}
```

Two packages resolving two copies of the same library produce failures that read as bugs in your code — a hook that throws about the wrong renderer, an `instanceof` that is false across a boundary, a decorator that never registers. The catalog makes that unrepresentable, and a bump is one line in one file instead of a grep across every `package.json`.

Name the catalog after what pins it, not after where it is used: `react19` bumps to `react20` and the diff says what happened. The default `catalog:` with no name works too, but a repo with one unnamed bucket eventually puts unrelated things in it.

A version that only one package uses stays in that package. The catalog is for what must agree.

## Install scripts

`allowBuilds` is an allowlist of dependencies permitted to run lifecycle scripts on install. Everything else is skipped, and pnpm reports what it skipped rather than failing.

Adding a name is a decision — a postinstall script runs with your credentials on every machine that clones the repo. The list is short on purpose: things that genuinely need to compile a binary or place one.

When a package misbehaves after install and its error mentions a missing binary or a native module, check this list before anything else.

## The lockfile

Committed, always, and the only place a transitive version is written down. `pnpm install --frozen-lockfile` is what CI runs; if that fails locally, a `package.json` was edited without an install.

## Filtering

```
pnpm --filter @acme/orders build       # one package
pnpm --filter @acme/api... build       # it and its dependencies
pnpm -r exec <cmd>                     # every package
```

Useful while a package is being stood up. Once nx is configured, `nx run-many` is what you reach for — it does the same fan-out and adds ordering and caching.
