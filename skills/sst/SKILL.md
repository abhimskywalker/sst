---
name: sst
description: Build full-stack applications with SST (Serverless Stack). Use this skill when creating, configuring, or deploying SST applications, working with sst.config.ts files, understanding SST CLI commands (sst dev, sst deploy, sst remove), or setting up SST in a new or existing project.
license: MIT
metadata:
  author: sst
  version: "3.0"
---

# SST (Serverless Stack)

SST is a framework for building modern full-stack applications on your own infrastructure. It uses Infrastructure as Code (IaC) where your entire app is defined in a single `sst.config.ts` file.

## When to use this skill

Use this skill when:
- Creating a new SST application
- Configuring the `sst.config.ts` file
- Understanding SST CLI commands
- Setting up stages for development, staging, and production
- Working with SST components and providers

## Key Concepts

### 1. The sst.config.ts File

Every SST app has a `sst.config.ts` file at the root. This file defines your entire application:

```typescript
/// <reference path="./.sst/platform/config.d.ts" />

export default $config({
  app(input) {
    return {
      name: "my-app",
      removal: input?.stage === "production" ? "retain" : "remove",
      home: "aws",
    };
  },
  async run() {
    // Define your infrastructure here
    const bucket = new sst.aws.Bucket("MyBucket");
    
    new sst.aws.Function("MyFunction", {
      handler: "src/handler.main",
      link: [bucket]
    });
  }
});
```

### 2. App Configuration

The `app` function returns configuration for your app:

```typescript
app(input) {
  return {
    name: "my-app",           // Required: unique app name
    removal: "remove",         // What happens on `sst remove`
    home: "aws",              // Cloud provider: "aws" or "cloudflare"
    providers: {              // Optional: configure providers
      aws: { region: "us-east-1" }
    }
  };
}
```

**Removal policies:**
- `"remove"` - Remove all resources (default for non-production)
- `"retain"` - Retain critical resources like databases and buckets
- `"retain-all"` - Retain all resources

### 3. Stages

Stages are separate environments of your app. Use them for:
- Personal development stages (auto-created from username)
- Shared development (`dev`)
- Production (`production`)
- PR previews (`pr-123`)

```bash
# Deploy to a specific stage
sst deploy --stage production

# The stage name is used to namespace resources
```

### 4. Components

SST components are high-level abstractions that create cloud resources:

```typescript
// AWS components are under sst.aws.*
new sst.aws.Bucket("MyBucket");
new sst.aws.Function("MyFunction", { handler: "src/handler.main" });
new sst.aws.Nextjs("MyWeb");
new sst.aws.Postgres("MyDB", { vpc });

// Cloudflare components are under sst.cloudflare.*
new sst.cloudflare.Worker("MyWorker");
```

### 5. Providers

SST supports 150+ Pulumi/Terraform providers:

```typescript
app(input) {
  return {
    name: "my-app",
    home: "aws",
    providers: {
      aws: { region: "us-east-1" },
      cloudflare: true,
      stripe: true
    }
  };
}

async run() {
  // Use native Pulumi providers
  new stripe.Product("MyProduct", {
    name: "Premium Plan"
  });
}
```

## CLI Commands

### sst init

Initialize SST in an existing project:

```bash
sst init
```

### sst dev

Start local development:

```bash
sst dev
```

This command:
1. Deploys your app to your personal stage
2. Runs Lambda functions "live" (changes reload in milliseconds)
3. Creates a tunnel for VPC access
4. Starts frontend/container services locally

### sst deploy

Deploy to a stage:

```bash
# Deploy to personal stage
sst deploy

# Deploy to production
sst deploy --stage production
```

### sst remove

Remove all resources in a stage:

```bash
sst remove --stage dev
```

**Warning:** Be careful with `sst remove` in production. Use `removal: "retain"` to protect critical resources.

### sst shell

Run commands with linked resources available:

```bash
sst shell npm run migrate
```

## Project Structure

### Drop-in Mode

For simple apps, add `sst.config.ts` to an existing project:

```
my-nextjs-app/
├── next.config.js
├── sst.config.ts    # SST config
├── package.json
└── app/
```

### Monorepo Mode

For larger apps, use a monorepo structure:

```
my-app/
├── sst.config.ts
├── package.json
├── packages/
│   ├── functions/   # Lambda functions
│   ├── frontend/    # Web frontend
│   └── core/        # Shared code
└── infra/           # Infrastructure modules
    ├── api.ts
    └── storage.ts
```

## Working with Outputs

Component properties return `Output<T>` types that resolve during deployment:

```typescript
const bucket = new sst.aws.Bucket("MyBucket");

// Use outputs in other components
new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  environment: {
    BUCKET_NAME: bucket.name  // Output<string>
  }
});

// String operations with $interpolate
const url = $interpolate`https://${domain.name}`;

// JSON operations with $jsonStringify
const config = $jsonStringify({ bucket: bucket.name });
```

## Best Practices

1. **Use personal stages for development**: Run `sst dev` in your own stage
2. **Protect production**: Set `removal: "retain"` for production stages
3. **Link resources**: Use resource linking instead of hardcoding values
4. **Use $transform for defaults**: Set default configurations across components
5. **Split infrastructure**: Use the `infra/` directory for large apps

## Common Patterns

### Environment-specific configuration

```typescript
app(input) {
  return {
    name: "my-app",
    removal: input?.stage === "production" ? "retain" : "remove",
    home: "aws",
    providers: {
      aws: {
        region: input?.stage === "production" ? "us-east-1" : "us-west-2"
      }
    }
  };
}
```

### Conditional resources

```typescript
async run() {
  const stage = $app.stage;
  
  if (stage === "production") {
    new sst.aws.Cron("DailyReport", {
      job: "src/report.handler",
      schedule: "rate(1 day)"
    });
  }
}
```

### Returning outputs

```typescript
async run() {
  const api = new sst.aws.Function("Api", {
    url: true,
    handler: "src/api.handler"
  });

  return {
    apiUrl: api.url
  };
}
```

## Troubleshooting

### State Issues

If your state gets corrupted:

```bash
sst refresh
```

### Deployment Stuck

If a deployment hangs, you can cancel and retry:

```bash
sst cancel
sst deploy
```

### Check Logs

View function logs in the SST Console or use:

```bash
sst dev  # See live logs in the multiplexer
```
