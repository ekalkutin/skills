# Bootstrap

Empty directory to two packages, one importing the other, built in the right order. Each step ends in a working state, and a commit.

`@acme`, `orders` and `api` stand in for the repo's own scope and package names.

## 1. The root

```
mkdir <repo> && cd <repo> && git init
pnpm init
```

`package.json` — the root is not a package anyone installs:

```json
{
  "name": "acme",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "nx run-many -t build",
    "dev": "nx run-many -t watch dev"
  },
  "devEngines": {
    "packageManager": {
      "name": "pnpm",
      "version": "^11.10.0",
      "onFail": "download"
    }
  },
  "devDependencies": {
    "@types/node": "^26.1.2",
    "nx": "^23.1.1",
    "typescript": "^7.0.2"
  }
}
```

`pnpm-workspace.yaml`:

```yaml
packages:
  - apps/*
  - packages/*
```

`.gitignore`:

```
node_modules
build

.nx
```

`tsconfig.json` — [typescript.md](./typescript.md) explains the choices:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,

    "module": "NodeNext",
    "moduleResolution": "nodenext",
    "esModuleInterop": true,
    "isolatedModules": true,
    "forceConsistentCasingInFileNames": true,

    "target": "esnext",
    "types": ["node"],
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "inlineSources": true,
    "skipLibCheck": true,
    "stripInternal": true
  }
}
```

Two options belong here only when a framework downstream needs them, and both are then set once, at the root, where the concession is visible:

```json
"emitDecoratorMetadata": true,
"experimentalDecorators": true,
"strictPropertyInitialization": false
```

The last is off because a container assigns injected properties after construction and the compiler cannot see it. A repo with no decorator-driven framework keeps all three out.

## 2. The first library

```
packages/orders/
  package.json
  tsconfig.json
  tsconfig.build.json
  src/index.ts
```

```json
{
  "name": "@acme/orders",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./build/index.d.ts",
      "import": "./build/index.js"
    }
  },
  "files": ["build"],
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "watch": "tsc -p tsconfig.build.json --watch --preserveWatchOutput"
  }
}
```

`tsconfig.json`, and the build variant beside it:

```json
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
{
  "extends": "./tsconfig.json",
  "exclude": ["build", "node_modules", "src/**/*.spec.ts"]
}
```

Then `pnpm install`, `pnpm --filter @acme/orders build`, and check that `build/index.js` and `build/index.d.ts` exist. **Get one package compiling before adding a second** — a resolution failure with one package is a typo, with two it is an argument about the task runner.

## 3. The catalog

The moment a second package needs the same dependency, move the version into `pnpm-workspace.yaml` rather than copying it:

```yaml
packages:
  - apps/*
  - packages/*

catalogs:
  react19:
    react: "19.2.0"
    react-dom: "19.2.0"
```

A backend framework is pinned exactly the same way — one catalog per thing whose version must agree, named after what pins it.

## 4. The app

`apps/api` is the same shape minus everything that exists for consumers — no `exports`, no `files`, no `declaration` — plus the dependency on the library and a run script:

```json
{
  "name": "@acme/api",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "watch": "tsc -p tsconfig.build.json --watch --preserveWatchOutput",
    "dev": "node --watch build/main.js"
  },
  "dependencies": {
    "@acme/orders": "workspace:*"
  }
}
```

`pnpm install` links `node_modules/@acme/orders` to the package directory, and the import resolves through its `exports` to `build/`. The two import forms differ, and the difference is the whole of the ESM convention here:

```ts
// apps/api/src/main.ts
import { OrderBook } from "@acme/orders";        // a package: no extension, ever

import { serve } from "./serve.js";              // relative: .js, always

async function bootstrap(): Promise<void> {
  await serve(new OrderBook(), 3000);
}

void bootstrap();
```

`./serve.js` names the file tsc will emit, which is what node loads. Its source is `src/serve.ts`.

## 5. The task graph

`nx.json` is last, and it is the smallest file in the repo — see [nx.md](./nx.md):

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

Verify both halves of it, because both fail silently:

```
pnpm nx run-many -t build --skip-nx-cache   # orders must run before api
pnpm nx run-many -t build                   # then again: 2/2 cache hit
```

## Order, and why

Root before packages, one package before two, and nx last. nx reads the workspace — the projects, their names, their dependency edges — from what pnpm already knows, so every file it needs is in place by the time it is added, and its first run is a check on the four steps before it rather than a step of its own.
