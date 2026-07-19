# Prisma

How a bounded context is wired to its database. Everything here is Prisma-specific — swap this file, not the layout, if a context uses something else.

## Where things live

```
<context>/
  prisma.config.ts                the CLI has to find it at the root
  src/infrastructure/persistence/
    schema/       *.prisma        this context's models
    migrations/                   this context's migration history
    generated/                    this context's client
    board.repository.ts  board.mapper.ts
```

One schema, one client, one migration history per context — never a shared package.

## Config

```ts
// <context>/prisma.config.ts
import "dotenv/config"; // a Prisma config file switches off Prisma's own .env loading

import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "./src/infrastructure/persistence/schema",
  migrations: { path: "./src/infrastructure/persistence/migrations" },
  datasource: { url: env("WORK_DATABASE_URL") },
});
```

Reference: <https://www.prisma.io/docs/orm/reference/prisma-config-reference>

## Schema

```prisma
datasource db {
  provider = "postgresql"
}

generator client {
  provider     = "prisma-client"   // TypeScript, so tsc compiles it like any other source
  output       = "../generated/client"
  moduleFormat = "esm"             // "cjs" in a CommonJS package
  runtime      = "nodejs"
}
```

The generated client is **not** excluded from the tsconfigs. It is TypeScript and has to be compiled.

## Client

```ts
import { PrismaPg } from "@prisma/adapter-pg";

import { PrismaClient } from "./generated/client/client.js";

@Injectable()
export class WorkPrismaService extends PrismaClient implements OnModuleDestroy {
  constructor() {
    super({
      adapter: new PrismaPg({
        connectionString: process.env["WORK_DATABASE_URL"],
      }),
    });
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

Nothing outside the context may inject it.

## Scripts

```json
{
  "scripts": {
    "db:generate": "prisma generate",
    "db:migrate": "prisma migrate dev",
    "db:deploy": "prisma migrate deploy",
    "db:reset": "prisma migrate reset --force"
  }
}
```
