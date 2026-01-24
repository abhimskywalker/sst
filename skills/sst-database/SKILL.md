---
name: sst-database
description: Set up databases with SST including Postgres, Aurora, MySQL, DynamoDB, Redis, and vector databases. Use this skill when provisioning databases, configuring VPCs, setting up database migrations with Drizzle or Prisma, or connecting applications to databases.
license: MIT
metadata:
  author: sst
  version: "3.0"
---

# SST Databases

SST provides components for various database types, from relational databases like Postgres and MySQL to NoSQL options like DynamoDB and Redis.

## When to use this skill

Use this skill when:
- Setting up Postgres, Aurora, or MySQL databases
- Configuring DynamoDB tables
- Setting up Redis for caching
- Running database migrations with Drizzle or Prisma
- Connecting databases to Lambda functions or containers

## Postgres (RDS)

### Basic Setup

```typescript
const vpc = new sst.aws.Vpc("MyVpc");

const database = new sst.aws.Postgres("MyDatabase", {
  vpc
});
```

### With RDS Proxy

For Lambda functions, use a proxy to manage connections:

```typescript
const database = new sst.aws.Postgres("MyDatabase", {
  vpc,
  proxy: true  // Enables RDS Proxy
});
```

### Full Configuration

```typescript
const vpc = new sst.aws.Vpc("MyVpc", {
  bastion: true,  // For local development access
  nat: "ec2"      // For Lambda in private subnets
});

const database = new sst.aws.Postgres("MyDatabase", {
  vpc,
  proxy: true,
  version: "16.3",
  instance: "db.t4g.medium",
  storage: "100 GB",
  transform: {
    instance: (args) => {
      args.deletionProtection = true;
    }
  }
});
```

### Connecting Functions

```typescript
new sst.aws.Function("MyApi", {
  vpc,
  url: true,
  handler: "src/api.handler",
  link: [database]
});
```

### Accessing in Code

```typescript
// src/api.ts
import { Resource } from "sst";
import postgres from "postgres";

const sql = postgres({
  host: Resource.MyDatabase.host,
  port: Resource.MyDatabase.port,
  database: Resource.MyDatabase.database,
  username: Resource.MyDatabase.username,
  password: Resource.MyDatabase.password,
});

export async function handler() {
  const users = await sql`SELECT * FROM users`;
  return { statusCode: 200, body: JSON.stringify(users) };
}
```

## Aurora (Serverless v2)

For auto-scaling databases:

```typescript
const vpc = new sst.aws.Vpc("MyVpc");

const database = new sst.aws.Aurora("MyDatabase", {
  vpc,
  engine: "postgres",  // or "mysql"
  scaling: {
    min: "0.5 ACU",
    max: "4 ACU"
  }
});
```

### Aurora MySQL

```typescript
const database = new sst.aws.Aurora("MyDatabase", {
  vpc,
  engine: "mysql",
  version: "8.0.mysql_aurora.3.05.1"
});
```

## MySQL (RDS)

```typescript
const vpc = new sst.aws.Vpc("MyVpc");

const database = new sst.aws.Mysql("MyDatabase", {
  vpc,
  version: "8.0",
  instance: "db.t4g.micro"
});
```

## DynamoDB

### Basic Table

```typescript
const table = new sst.aws.Dynamo("MyTable", {
  fields: {
    pk: "string",
    sk: "string"
  },
  primaryIndex: { hashKey: "pk", rangeKey: "sk" }
});
```

### With Global Secondary Index

```typescript
const table = new sst.aws.Dynamo("MyTable", {
  fields: {
    pk: "string",
    sk: "string",
    gsi1pk: "string",
    gsi1sk: "string"
  },
  primaryIndex: { hashKey: "pk", rangeKey: "sk" },
  globalIndexes: {
    gsi1: { hashKey: "gsi1pk", rangeKey: "gsi1sk" }
  }
});
```

### Stream Handler

```typescript
const table = new sst.aws.Dynamo("MyTable", {
  fields: { pk: "string", sk: "string" },
  primaryIndex: { hashKey: "pk", rangeKey: "sk" },
  stream: "new-and-old-images"
});

table.subscribe("src/stream.handler");
```

### Accessing in Code

```typescript
// src/api.ts
import { Resource } from "sst";
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, GetCommand, PutCommand } from "@aws-sdk/lib-dynamodb";

const client = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export async function handler() {
  await client.send(new PutCommand({
    TableName: Resource.MyTable.name,
    Item: { pk: "user#123", sk: "profile", name: "John" }
  }));

  const { Item } = await client.send(new GetCommand({
    TableName: Resource.MyTable.name,
    Key: { pk: "user#123", sk: "profile" }
  }));

  return { statusCode: 200, body: JSON.stringify(Item) };
}
```

## Redis (ElastiCache)

### Basic Setup

```typescript
const vpc = new sst.aws.Vpc("MyVpc");

const redis = new sst.aws.Redis("MyRedis", { vpc });
```

### Cluster Mode

```typescript
const redis = new sst.aws.Redis("MyRedis", {
  vpc,
  cluster: true
});
```

### Accessing in Code

```typescript
import { Resource } from "sst";
import { createClient } from "redis";

const client = createClient({
  url: `redis://${Resource.MyRedis.host}:${Resource.MyRedis.port}`
});

await client.connect();
await client.set("key", "value");
const value = await client.get("key");
```

## Vector Database

For AI/ML applications:

```typescript
const vpc = new sst.aws.Vpc("MyVpc");

const vector = new sst.aws.Vector("MyVector", { vpc });
```

### Usage

```typescript
import { Resource } from "sst";
import { VectorClient } from "sst/vector";

const client = new VectorClient(Resource.MyVector);

// Store embeddings
await client.put({
  id: "doc-1",
  vector: [0.1, 0.2, 0.3, ...],
  metadata: { title: "My Document" }
});

// Query
const results = await client.query({
  vector: [0.1, 0.2, 0.3, ...],
  topK: 10
});
```

## Database Migrations

### With Drizzle

```typescript
const vpc = new sst.aws.Vpc("MyVpc", { bastion: true, nat: "ec2" });
const database = new sst.aws.Postgres("MyDatabase", { vpc, proxy: true });

new sst.aws.Function("MyApi", {
  vpc,
  url: true,
  handler: "src/api.handler",
  link: [database]
});

// DevCommand for Drizzle Studio
new sst.x.DevCommand("Studio", {
  link: [database],
  dev: {
    command: "npx drizzle-kit studio"
  }
});
```

Drizzle config:

```typescript
// drizzle.config.ts
import { Resource } from "sst";
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./migrations",
  dialect: "postgresql",
  dbCredentials: {
    host: Resource.MyDatabase.host,
    port: Resource.MyDatabase.port,
    database: Resource.MyDatabase.database,
    user: Resource.MyDatabase.username,
    password: Resource.MyDatabase.password,
  }
});
```

Running migrations:

```bash
# Generate migration
sst shell npx drizzle-kit generate

# Run migration
sst shell npx drizzle-kit migrate

# Open Drizzle Studio (during sst dev)
npx drizzle-kit studio
```

### With Prisma

```typescript
const vpc = new sst.aws.Vpc("MyVpc", { bastion: true, nat: "ec2" });
const database = new sst.aws.Postgres("MyDatabase", { vpc, proxy: true });

new sst.aws.Function("MyApi", {
  vpc,
  url: true,
  handler: "src/api.handler",
  link: [database],
  copyFiles: [{ from: "prisma/schema.prisma" }]
});

new sst.x.DevCommand("Studio", {
  link: [database],
  dev: {
    command: "npx prisma studio"
  }
});
```

Prisma schema:

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
}
```

Access in code:

```typescript
// src/api.ts
import { Resource } from "sst";
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient({
  datasources: {
    db: {
      url: `postgresql://${Resource.MyDatabase.username}:${Resource.MyDatabase.password}@${Resource.MyDatabase.host}:${Resource.MyDatabase.port}/${Resource.MyDatabase.database}`
    }
  }
});
```

Running migrations:

```bash
# Generate Prisma client
sst shell npx prisma generate

# Run migration
sst shell npx prisma migrate dev

# Push schema (without migration)
sst shell npx prisma db push
```

## VPC Configuration

Most databases require a VPC:

### Basic VPC

```typescript
const vpc = new sst.aws.Vpc("MyVpc");
```

### With Bastion (for local access)

```typescript
const vpc = new sst.aws.Vpc("MyVpc", {
  bastion: true  // Creates SSH bastion for tunnel
});
```

### With NAT Gateway

```typescript
const vpc = new sst.aws.Vpc("MyVpc", {
  bastion: true,
  nat: "managed"  // or "ec2" for cheaper option
});
```

## Local Development

### Using Tunnel

For connecting to VPC resources locally:

1. Enable bastion in VPC:
```typescript
const vpc = new sst.aws.Vpc("MyVpc", { bastion: true });
```

2. Install tunnel:
```bash
sudo sst tunnel install
```

3. Run dev (tunnel starts automatically):
```bash
sst dev
```

### Using Local Database

For faster development, use a local database:

```typescript
// src/db.ts
import { Resource } from "sst";
import postgres from "postgres";

const isLocal = process.env.SST_DEV === "true";

const sql = postgres(isLocal 
  ? "postgres://localhost:5432/mydb"
  : {
      host: Resource.MyDatabase.host,
      port: Resource.MyDatabase.port,
      database: Resource.MyDatabase.database,
      username: Resource.MyDatabase.username,
      password: Resource.MyDatabase.password,
    }
);
```

## Best Practices

1. **Use RDS Proxy for Lambda**: Manages connection pooling
2. **Enable bastion for development**: Allows local database access via tunnel
3. **Use Drizzle or Prisma**: Type-safe database access
4. **Set deletion protection in production**: Prevent accidental data loss
5. **Use appropriate instance sizes**: Start small, scale as needed
6. **Separate databases per stage**: Each stage should have its own database

## Common Issues

### Connection Timeouts

Lambda functions in VPC need NAT for internet access:

```typescript
const vpc = new sst.aws.Vpc("MyVpc", {
  nat: "ec2"  // or "managed"
});
```

### Too Many Connections

Use RDS Proxy to manage connection pooling:

```typescript
const database = new sst.aws.Postgres("MyDatabase", {
  vpc,
  proxy: true
});
```

### Local Access Issues

Ensure tunnel is installed and running:

```bash
sudo sst tunnel install
sst dev  # Tunnel tab should appear
```
