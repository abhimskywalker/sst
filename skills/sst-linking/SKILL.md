---
name: sst-linking
description: Link SST resources together for type-safe access in runtime code. Use this skill when connecting infrastructure to functions or frontends, accessing resource properties in code using the SST SDK, creating custom linkable resources, or setting up permissions between resources.
license: MIT
metadata:
  author: sst
  version: "3.0"
---

# SST Resource Linking

Resource Linking is one of SST's most powerful features. It allows you to access your infrastructure in your runtime code in a type-safe and secure way, without hardcoding values.

## When to use this skill

Use this skill when:
- Connecting resources like buckets, databases, or queues to functions
- Accessing infrastructure properties in your application code
- Creating custom linkable values or resources
- Setting up IAM permissions between resources
- Working with the SST SDK

## Basic Linking

### 1. Create a resource

```typescript
const bucket = new sst.aws.Bucket("MyBucket");
```

### 2. Link it to a function or frontend

```typescript
new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  link: [bucket]
});

// Or to a frontend
new sst.aws.Nextjs("MyWeb", {
  link: [bucket]
});
```

### 3. Access in code using the SDK

```typescript
// src/handler.ts
import { Resource } from "sst";

export async function main() {
  // Type-safe access to bucket properties
  console.log(Resource.MyBucket.name);
  console.log(Resource.MyBucket.arn);
}
```

## What Gets Linked

Each component exposes specific properties through linking. These are documented in the component's API reference.

### Common Components and Their Links

| Component | Linked Properties |
|-----------|-------------------|
| `Bucket` | `name`, `arn` |
| `Queue` | `url`, `arn` |
| `SnsTopic` | `arn`, `name` |
| `Postgres` | `host`, `port`, `database`, `username`, `password` |
| `Dynamo` | `name`, `arn` |
| `Redis` | `host`, `port` |
| `Secret` | `value` |

## Linking to Different Targets

### Functions

```typescript
const bucket = new sst.aws.Bucket("MyBucket");
const database = new sst.aws.Postgres("MyDB", { vpc });

new sst.aws.Function("MyFunction", {
  handler: "src/api.handler",
  link: [bucket, database]
});
```

### Containers/Services

```typescript
new sst.aws.Service("MyService", {
  cluster,
  link: [bucket, database],
  loadBalancer: {
    ports: [{ listen: "80/http" }]
  }
});
```

### Frontends

```typescript
new sst.aws.Nextjs("MyWeb", {
  link: [bucket, api]
});
```

### Cron Jobs

```typescript
new sst.aws.Cron("DailyJob", {
  job: {
    handler: "src/cron.handler",
    link: [database]
  },
  schedule: "rate(1 day)"
});
```

## The SST SDK

### Installation

```bash
npm install sst
```

### JavaScript/TypeScript

```typescript
import { Resource } from "sst";

// Access linked resources
const bucketName = Resource.MyBucket.name;
const dbHost = Resource.MyDB.host;
```

### Python

```python
from sst import Resource

bucket_name = Resource.MyBucket.name
db_host = Resource.MyDB.host
```

### Golang

```go
import "github.com/sst/sst/v3/sdk/golang/resource"

bucketName := resource.MyBucket.Name
dbHost := resource.MyDB.Host
```

### Rust

```rust
use sst::Resource;

let bucket_name = Resource::my_bucket().name;
```

## Secrets

Link secrets for sensitive values:

```typescript
const stripeKey = new sst.Secret("StripeKey");

new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  link: [stripeKey]
});
```

Set the secret value:

```bash
sst secret set StripeKey sk_live_...
```

Access in code:

```typescript
import { Resource } from "sst";

const stripe = new Stripe(Resource.StripeKey.value);
```

## Custom Linkable Values

Link any value using `sst.Linkable`:

```typescript
const myConfig = new sst.Linkable("MyConfig", {
  properties: {
    apiUrl: "https://api.example.com",
    region: "us-east-1"
  }
});

new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  link: [myConfig]
});
```

Access in code:

```typescript
import { Resource } from "sst";

console.log(Resource.MyConfig.apiUrl);    // https://api.example.com
console.log(Resource.MyConfig.region);    // us-east-1
```

### With Permissions

Include IAM permissions with your linkable:

```typescript
const customLinkable = new sst.Linkable("MyCustom", {
  properties: {
    bucketName: bucket.name,
    queueUrl: queue.url
  },
  include: [
    sst.aws.permission({
      actions: ["s3:GetObject", "s3:PutObject"],
      resources: [$interpolate`${bucket.arn}/*`]
    }),
    sst.aws.permission({
      actions: ["sqs:SendMessage"],
      resources: [queue.arn]
    })
  ]
});
```

## Making Any Resource Linkable

Use `Linkable.wrap` to make Pulumi/Terraform resources linkable:

```typescript
// Make DynamoDB tables linkable
sst.Linkable.wrap(aws.dynamodb.Table, (table) => ({
  properties: {
    tableName: table.name,
    tableArn: table.arn
  },
  include: [
    sst.aws.permission({
      actions: ["dynamodb:*"],
      resources: [table.arn]
    })
  ]
}));

// Now use it like any SST component
const table = new aws.dynamodb.Table("MyTable", {
  attributes: [{ name: "pk", type: "S" }],
  hashKey: "pk",
  billingMode: "PAY_PER_REQUEST"
});

new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  link: [table]
});
```

Access in code:

```typescript
import { Resource } from "sst";

console.log(Resource.MyTable.tableName);
```

## Modifying Built-in Links

Override SST's default linking behavior:

```typescript
// Change permissions for Bucket links
sst.Linkable.wrap(sst.aws.Bucket, (bucket) => ({
  properties: { name: bucket.name },
  include: [
    sst.aws.permission({
      actions: ["s3:GetObject"],  // Read-only
      resources: [$interpolate`${bucket.arn}/*`]
    })
  ]
}));
```

## Local Development

### With sst dev

The `sst dev` multiplexer automatically injects linked resources:

```bash
sst dev
```

### Basic Mode

In basic mode, wrap your dev command:

```bash
sst dev --mode=basic
sst dev next dev
```

### Checking Links

View what's linked in the types file:

```typescript
// sst-env.d.ts (generated)
declare module "sst" {
  export interface Resource {
    MyBucket: {
      name: string;
      arn: string;
    };
    MyDB: {
      host: string;
      port: number;
      database: string;
      username: string;
      password: string;
    };
  }
}
```

## Permissions

When you link a resource, SST automatically grants the necessary IAM permissions.

### Default Permissions

| Resource | Permissions Granted |
|----------|---------------------|
| `Bucket` | Full S3 access to the bucket |
| `Queue` | Full SQS access to the queue |
| `Dynamo` | Full DynamoDB access to the table |
| `Postgres` | Connection credentials (no IAM) |

### Custom Permissions

Add extra permissions to a function:

```typescript
new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  link: [bucket],
  permissions: [
    {
      actions: ["ses:SendEmail"],
      resources: ["*"]
    }
  ]
});
```

## Linking Across Stacks

Link resources from one stack to another using outputs:

```typescript
// infra/storage.ts
export const bucket = new sst.aws.Bucket("MyBucket");

// infra/api.ts
import { bucket } from "./storage";

new sst.aws.Function("MyApi", {
  handler: "src/api.handler",
  link: [bucket]
});
```

## Best Practices

1. **Link instead of hardcode**: Always use linking for resource access
2. **Use secrets for sensitive data**: Never hardcode API keys
3. **Keep links minimal**: Only link what you need
4. **Type your resources**: Commit `sst-env.d.ts` for team visibility
5. **Use custom Linkables for config**: Group related config values

## Common Patterns

### Database Connection String

```typescript
const database = new sst.aws.Postgres("MyDB", { vpc, proxy: true });

// Create a linkable with connection string
const dbUrl = new sst.Linkable("DatabaseUrl", {
  properties: {
    url: $interpolate`postgresql://${database.username}:${database.password}@${database.host}:${database.port}/${database.database}`
  }
});

new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  link: [dbUrl]
});
```

### External Services

```typescript
const externalApi = new sst.Linkable("ExternalApi", {
  properties: {
    url: process.env.EXTERNAL_API_URL!,
    key: process.env.EXTERNAL_API_KEY!
  }
});
```

### Feature Flags

```typescript
const features = new sst.Linkable("Features", {
  properties: {
    newDashboard: $app.stage === "production" ? "false" : "true",
    betaFeatures: "true"
  }
});
```

## Troubleshooting

### Resource Not Found

Ensure the resource name matches exactly:

```typescript
// sst.config.ts
const bucket = new sst.aws.Bucket("MyBucket");  // Name is "MyBucket"

// src/handler.ts
Resource.MyBucket.name  // Correct
Resource.mybucket.name  // Wrong - case sensitive
```

### Types Not Generated

Run `sst dev` or `sst deploy` to generate types:

```bash
sst dev
```

Check `sst-env.d.ts` exists in your project.

### Links Not Available Client-Side

Linked resources are only available server-side:

```typescript
// app/page.tsx (Next.js)

// Server component - works
export default function Page() {
  console.log(Resource.MyBucket.name);  // ✓
}

// Client component - doesn't work
"use client"
export default function Page() {
  console.log(Resource.MyBucket.name);  // ✗
}
```

Pass values to client components explicitly.
