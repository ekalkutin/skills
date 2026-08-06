# nx

nx is added last and does two things: it runs tasks in dependency order, and it skips a task whose inputs have not changed. It does not build anything — the scripts do.

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "cache": true,
      "outputs": ["{projectRoot}/build"]
    },
    "watch": { "continuous": true, "dependsOn": ["build"] },
    "dev": { "continuous": true, "dependsOn": ["build"] }
  },
  "analytics": false
}
```

There is no `project.json` anywhere. A target is a script in a `package.json`, and nx picks it up by name. `@acme/orders` below stands in for any package in the repo.

## Targets are scripts

```json
"scripts": {
  "build": "tsc -p tsconfig.build.json",
  "watch": "tsc -p tsconfig.build.json --watch --preserveWatchOutput"
}
```

Every one of these runs by hand, in CI, and under nx, unchanged. That is the property worth protecting: when a build behaves oddly, `cd packages/orders && pnpm build` answers whether the problem is the compiler or the task runner. A target defined only inside nx config cannot be bisected that way.

`targetDefaults` applies to every project that has a script by that name, so a new package inherits the whole pipeline by naming its scripts correctly and adding no configuration at all.

## Ordering

```json
"build": { "dependsOn": ["^build"] }
```

`^build` is *the `build` target of my dependencies*. The dependency graph comes from `workspace:*` entries — nothing is declared twice.

```
pnpm nx run-many -t build --skip-nx-cache
```

reports the order it chose. If a package builds too early, the missing edge is a missing `workspace:*` line, not an nx problem.

## Caching

**Both keys or neither works:**

```json
"cache": true,
"outputs": ["{projectRoot}/build"]
```

Without them nx runs every task on every invocation and says so quietly, in a line under the summary:

```
Cache:  0/2 hit (0%)          # not configured — full compile, every time
Cache:  2/2 hit (100%)        # configured, nothing changed — tens of milliseconds
```

`cache: true` makes the target eligible; `outputs` tells nx what to restore on a hit. A cacheable target with no `outputs` replays the terminal output and leaves `build/` untouched, which is worse than no cache — the next task reads stale or absent files.

`{projectRoot}` is literal; nx expands it per project. Add `.nx` to `.gitignore`.

Read `Cache: n/m hit` after any change to this config. It is the only feedback there is.

Never mark a target cacheable when its result depends on something outside the inputs nx hashes — anything that touches a database, a network, or the clock.

## Continuous targets

```json
"watch": { "continuous": true, "dependsOn": ["build"] },
"dev":   { "continuous": true, "dependsOn": ["build"] }
```

`continuous: true` tells nx this process is not going to exit, so it does not wait for it before starting the next task. Without it, `run-many -t watch` starts one watcher and hangs.

Each depends on the one-shot `build`, so a watcher and a `node --watch` start against output that already exists rather than racing the first compile.

The dev loop is then one command at the root:

```json
"dev": "nx run-many -t watch dev"
```

Every package's `watch` and every app's `dev` come up together: a change in a library is recompiled by its own watcher, and the app's `node --watch` sees the changed file in `build/` and restarts. Nothing coordinates them beyond the filesystem.

## Root scripts

```json
"build": "nx run-many -t build",
"dev": "nx run-many -t watch dev"
```

Fan-outs only. Anything more specific is a `--filter`/`-p` flag typed at the moment it is needed, not a script.

## Commands worth knowing

```
pnpm nx run-many -t build                  # everything, in order, cached
pnpm nx run-many -t build --skip-nx-cache  # same, ignoring the cache
pnpm nx build @acme/orders                 # one project and its dependencies
pnpm nx graph                              # the dependency graph, in a browser
pnpm nx reset                              # clear the cache and stop the daemon
```

`nx reset` is the first thing to try when nx behaves in a way the config does not explain — a stale daemon holding an old project graph looks exactly like a configuration bug.

## What is deliberately not here

No nx plugins, and so no inferred targets: every target in this repo is a script someone wrote. Plugins infer targets from tool config, which removes boilerplate and adds a layer between `pnpm build` and what actually runs. Not worth it at this size.

No remote cache. Local caching is what makes the inner loop fast; sharing it between machines is a CI decision, made when CI exists.
