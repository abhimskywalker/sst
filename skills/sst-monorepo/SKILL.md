---
name: sst-monorepo
description: Set up monorepo project structure for SST applications. Use this skill when organizing code into packages (functions, core, frontend), splitting infrastructure into modules, configuring workspaces, or setting up a scalable project architecture for SST apps.
license: MIT
metadata:
  author: sst
  version: "3.0"
---

# SST Monorepo Setup

For larger applications, SST recommends a monorepo structure that separates your code into packages and your infrastructure into modules.

## When to use this skill

Use this skill when:
- Starting a new SST project that will grow
- Organizing existing code into a monorepo structure
- Setting up shared code packages
- Splitting infrastructure into logical modules
- Configuring npm/yarn/pnpm workspaces

## Project Structure

```
my-sst-app/
├── sst.config.ts           # Main SST config
├── package.json            # Root package.json with workspaces
├── tsconfig.json           # Root TypeScript config
├── packages/
│   ├── core/               # Shared business logic
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── functions/          # Lambda function handlers
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── frontend/           # Web frontend (Next.js, etc.)
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── scripts/            # CLI scripts and utilities
│       ├── src/
│       ├── package.json
│       └── tsconfig.json
└── infra/                  # Infrastructure modules
    ├── storage.ts
    ├── api.ts
    └── web.ts
```

## Root Configuration

### package.json

```json
{
  "name": "my-sst-app",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "dev": "sst dev",
    "deploy": "sst deploy",
    "remove": "sst remove",
    "test": "npm run test --workspaces --if-present"
  },
  "devDependencies": {
    "sst": "latest",
    "typescript": "^5.0.0"
  }
}
```

### tsconfig.json (root)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true
  }
}
```

### sst.config.ts

```typescript
/// <reference path="./.sst/platform/config.d.ts" />

export default $config({
  app(input) {
    return {
      name: "my-sst-app",
      removal: input?.stage === "production" ? "retain" : "remove",
      home: "aws",
    };
  },
  async run() {
    // Import infrastructure modules
    const storage = await import("./infra/storage");
    const api = await import("./infra/api");
    const web = await import("./infra/web");

    return {
      bucket: storage.bucket.name,
      api: api.api.url,
      web: web.site.url,
    };
  },
});
```

## Packages

### packages/core

Shared business logic and domain modules:

```
packages/core/
├── src/
│   ├── user/
│   │   └── index.ts
│   ├── order/
│   │   └── index.ts
│   └── utils/
│       └── index.ts
├── package.json
└── tsconfig.json
```

**package.json:**

```json
{
  "name": "@my-app/core",
  "version": "0.0.1",
  "type": "module",
  "exports": {
    "./*": [
      "./src/*/index.ts",
      "./src/*.ts"
    ]
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vitest": "^1.0.0"
  },
  "scripts": {
    "test": "sst shell vitest run"
  }
}
```

**src/user/index.ts:**

```typescript
export module User {
  export interface User {
    id: string;
    email: string;
    name: string;
  }

  export async function create(data: Omit<User, "id">): Promise<User> {
    // Implementation
    return { id: crypto.randomUUID(), ...data };
  }

  export async function getById(id: string): Promise<User | null> {
    // Implementation
    return null;
  }
}
```

**Usage in other packages:**

```typescript
import { User } from "@my-app/core/user";

const user = await User.create({ email: "test@example.com", name: "Test" });
```

### packages/functions

Lambda function handlers:

```
packages/functions/
├── src/
│   ├── api/
│   │   ├── users.ts
│   │   └── orders.ts
│   └── events/
│       └── process-order.ts
├── package.json
└── tsconfig.json
```

**package.json:**

```json
{
  "name": "@my-app/functions",
  "version": "0.0.1",
  "type": "module",
  "dependencies": {
    "@my-app/core": "*",
    "sst": "latest"
  },
  "devDependencies": {
    "@types/aws-lambda": "^8.10.0",
    "typescript": "^5.0.0"
  }
}
```

**src/api/users.ts:**

```typescript
import { Resource } from "sst";
import { User } from "@my-app/core/user";
import { APIGatewayProxyEventV2, APIGatewayProxyResultV2 } from "aws-lambda";

export async function list(event: APIGatewayProxyEventV2): Promise<APIGatewayProxyResultV2> {
  const users = await User.list();
  return {
    statusCode: 200,
    body: JSON.stringify(users),
  };
}

export async function create(event: APIGatewayProxyEventV2): Promise<APIGatewayProxyResultV2> {
  const body = JSON.parse(event.body || "{}");
  const user = await User.create(body);
  return {
    statusCode: 201,
    body: JSON.stringify(user),
  };
}
```

### packages/scripts

CLI scripts for maintenance and operations:

```
packages/scripts/
├── src/
│   ├── migrate.ts
│   └── seed.ts
├── package.json
└── tsconfig.json
```

**package.json:**

```json
{
  "name": "@my-app/scripts",
  "version": "0.0.1",
  "type": "module",
  "dependencies": {
    "@my-app/core": "*",
    "sst": "latest"
  },
  "devDependencies": {
    "tsx": "^4.0.0",
    "typescript": "^5.0.0"
  },
  "scripts": {
    "shell": "sst shell tsx"
  }
}
```

**src/seed.ts:**

```typescript
import { Resource } from "sst";
import { User } from "@my-app/core/user";

async function main() {
  console.log("Seeding database...");
  
  await User.create({ email: "admin@example.com", name: "Admin" });
  
  console.log("Done!");
}

main().catch(console.error);
```

**Running scripts:**

```bash
cd packages/scripts
npm run shell src/seed.ts
```

## Infrastructure Modules

### infra/storage.ts

```typescript
export const bucket = new sst.aws.Bucket("MyBucket", {
  access: "public"
});

export const uploads = new sst.aws.Bucket("Uploads");
```

### infra/database.ts

```typescript
export const vpc = new sst.aws.Vpc("MyVpc", {
  bastion: true,
  nat: "ec2"
});

export const database = new sst.aws.Postgres("MyDatabase", {
  vpc,
  proxy: true
});
```

### infra/api.ts

```typescript
import { bucket } from "./storage";
import { database, vpc } from "./database";

export const api = new sst.aws.ApiGatewayV2("MyApi");

api.route("GET /users", {
  handler: "packages/functions/src/api/users.list",
  link: [database]
});

api.route("POST /users", {
  handler: "packages/functions/src/api/users.create",
  link: [database]
});

api.route("POST /upload", {
  handler: "packages/functions/src/api/upload.handler",
  link: [bucket]
});
```

### infra/web.ts

```typescript
import { api } from "./api";
import { bucket } from "./storage";

export const site = new sst.aws.Nextjs("MyWeb", {
  path: "packages/frontend",
  link: [bucket],
  environment: {
    NEXT_PUBLIC_API_URL: api.url
  }
});
```

## Workspace Commands

### npm Workspaces

```bash
# Install all dependencies
npm install

# Run command in specific package
npm run test -w @my-app/core

# Run command in all packages
npm run test --workspaces

# Add dependency to specific package
npm install lodash -w @my-app/core
```

### pnpm Workspaces

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
```

```bash
# Install all dependencies
pnpm install

# Run command in specific package
pnpm --filter @my-app/core test

# Run command in all packages
pnpm -r test
```

### yarn Workspaces

```json
{
  "workspaces": ["packages/*"]
}
```

```bash
# Install all dependencies
yarn install

# Run command in specific package
yarn workspace @my-app/core test

# Run command in all packages
yarn workspaces foreach run test
```

## TypeScript Configuration

### packages/*/tsconfig.json

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

### Path Aliases (optional)

For cleaner imports, configure path aliases:

```json
// tsconfig.json (root)
{
  "compilerOptions": {
    "paths": {
      "@core/*": ["./packages/core/src/*"],
      "@functions/*": ["./packages/functions/src/*"]
    }
  }
}
```

## Testing

### Setting up Vitest

```typescript
// packages/core/vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "node"
  }
});
```

### Test with linked resources

```typescript
// packages/core/src/user/index.test.ts
import { describe, it, expect } from "vitest";
import { User } from "./index";

describe("User", () => {
  it("should create a user", async () => {
    const user = await User.create({
      email: "test@example.com",
      name: "Test User"
    });
    
    expect(user.id).toBeDefined();
    expect(user.email).toBe("test@example.com");
  });
});
```

Run tests with `sst shell` for linked resources:

```bash
npm test -w @my-app/core
# or
sst shell vitest run
```

## DevCommands

Set up development tools that need linked resources:

```typescript
// sst.config.ts or infra/dev.ts
import { database } from "./database";

new sst.x.DevCommand("Studio", {
  link: [database],
  dev: {
    command: "npx drizzle-kit studio",
    directory: "packages/functions"
  }
});

new sst.x.DevCommand("PrismaStudio", {
  link: [database],
  dev: {
    command: "npx prisma studio",
    directory: "packages/functions"
  }
});
```

## Best Practices

1. **Use Domain Driven Design**: Organize `core` package by domain (user, order, etc.)
2. **Keep functions thin**: Business logic in `core`, handlers in `functions`
3. **Share types**: Export interfaces from `core` for use everywhere
4. **Split infrastructure logically**: One module per concern (storage, api, web)
5. **Use consistent naming**: `@my-app/core`, `@my-app/functions`, etc.
6. **Test the core**: Focus tests on business logic in `core` package

## Template

SST provides a monorepo template to get started quickly:

```bash
# Use the template
git clone https://github.com/sst/monorepo-template my-app
cd my-app

# Rename the app
npx replace-in-file /monorepo-template/g my-app **/*.* --verbose

# Install dependencies
npm install

# Start development
npx sst dev
```

## Common Issues

### Package not found

Ensure the package is listed in workspaces and installed:

```bash
npm install
```

### Types not resolving

Check that `exports` in package.json matches your file structure:

```json
{
  "exports": {
    "./*": [
      "./src/*/index.ts",
      "./src/*.ts"
    ]
  }
}
```

### Circular dependencies

Avoid importing between `infra/` modules. Instead, export and import at the top level:

```typescript
// sst.config.ts - Good
async run() {
  const storage = await import("./infra/storage");
  await import("./infra/api");  // Can use storage.bucket
}
```
