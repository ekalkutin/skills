# TypeScript

Three files decide how the repo compiles: one root config, and a pair in every package. `@acme/orders` below stands in for any package in the repo.

## The root config

`tsconfig.json` at the root holds `compilerOptions` and nothing else. No `include`, no `files`, no `references`, no `paths`. It is never built — `tsc -p tsconfig.json` at the root should compile nothing, because there is no source at the root.

Everything that must be true everywhere goes here, and a package never relaxes it. `strict` off in one package is how a repo ends up with two dialects.

```json
"strict": true,
"noUncheckedIndexedAccess": true,
"noImplicitOverride": true
```

A concession to a framework is made here too, not in a package. `strictPropertyInitialization: false` is the usual one — a DI container assigns injected properties after construction and the compiler cannot see it — and at the root it is one visible line rather than a habit spreading package by package.

## Module resolution

```json
"module": "NodeNext",
"moduleResolution": "nodenext",
"isolatedModules": true
```

With `"type": "module"` in every `package.json`, this is plain ESM as node runs it, which means **a relative import carries the extension of the emitted file**:

```ts
import { OrderBook } from "./order-book.js";     // relative: .js, always
import { OrderBook } from "@acme/orders";        // package: never
```

`.js` on a file you can see is `.ts` reads wrong for about a week. It is naming the output, which is what node will actually load.

`isolatedModules` keeps every file independently transpilable — the constraint that lets a faster non-`tsc` tool be dropped in later without a rewrite.

## No path aliases

There are no `paths` entries mapping `@acme/*` to `packages/*/src`. Resolution goes through node: pnpm symlinks the package, the package's `exports` names `build/`, and that is the file that gets loaded.

The alias version looks more convenient and costs more. It compiles a consumer against source the consumer will never receive, so a missing `exports` entry or an unbuilt package is invisible until deploy; and it has to be repeated in every tool that resolves modules — the runtime, the test runner, the bundler — each with its own syntax.

The price of not aliasing is that a dependency must be built before a consumer type-checks. That is exactly what `dependsOn: ["^build"]` is for, and a stale build failing loudly is the point.

## Two configs per package

```json
// packages/orders/tsconfig.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./build"
  },
  "include": ["src/**/*"],
  "exclude": ["build", "node_modules"]
}
```

```json
// packages/orders/tsconfig.build.json
{
  "extends": "./tsconfig.json",
  "exclude": ["build", "node_modules", "src/**/*.spec.ts"]
}
```

`tsconfig.json` includes everything, so specs are type-checked by the editor and by a bare `tsc --noEmit`. `tsconfig.build.json` is what the `build` script points at, so specs never reach the output.

**`exclude` is replaced, not merged, when a config extends another** — `build` and `node_modules` have to be repeated in the second file or they come back.

The pair is identical in every package apart from the two paths. Copy it; do not improve it in one package.

## Library or app

A library exists to be imported, and says so:

```json
"exports": {
  ".": {
    "types": "./build/index.d.ts",
    "import": "./build/index.js"
  }
},
"files": ["build"]
```

with `declaration` and `declarationMap` on, so a consumer gets types and ctrl-click lands in the `.ts` source.

An app has none of it — no `exports`, no `files`, no `declaration`. Nothing imports an app, so its `.d.ts` files are output that will never be read.

**Delete `"main"` once `exports` is present.** The `"main": "index.js"` that `pnpm init` writes points at a file that does not exist; node ignores it in favour of `exports`, and the tools that do not are precisely the ones that will fail confusingly.

`"private": true` on every package that is not published, which is usually all of them.

## Output

`build/` in every package, spelled the same way everywhere, so `.gitignore`, nx `outputs`, Docker copies and CI artifacts each name it once. `dist` is equally fine; two names in one repo are not.

`sourceMap` with `inlineSources` puts the original text in the map, so a stack trace from a deployed build resolves without shipping `src/`.

## Watch

```json
"watch": "tsc -p tsconfig.build.json --watch --preserveWatchOutput"
```

`--preserveWatchOutput` stops tsc clearing the terminal on every rebuild, which matters when several watchers share one — without it, whichever package rebuilt last erases the error you were reading.
