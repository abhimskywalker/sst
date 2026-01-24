---
name: sst-frontend
description: Deploy frontend applications with SST including Next.js, Remix, Astro, SvelteKit, SolidStart, Nuxt, TanStack Start, and static sites to AWS or Cloudflare. Use this skill when setting up frontend frameworks, configuring custom domains, optimizing builds, or connecting frontends to backend resources.
license: MIT
metadata:
  author: sst
  version: "3.0"
---

# SST Frontend Frameworks

SST makes it easy to deploy frontend frameworks to AWS or Cloudflare with built-in optimizations, custom domains, and seamless linking to backend resources.

## When to use this skill

Use this skill when:
- Deploying Next.js, Remix, Astro, or other frontend frameworks
- Setting up custom domains for frontends
- Configuring build settings and environment variables
- Linking frontends to backend resources (buckets, databases, APIs)
- Deploying static sites

## Supported Frameworks

### AWS Components

| Framework | Component |
|-----------|-----------|
| Next.js | `sst.aws.Nextjs` |
| Remix | `sst.aws.Remix` |
| Astro | `sst.aws.Astro` |
| SvelteKit | `sst.aws.SvelteKit` |
| SolidStart | `sst.aws.SolidStart` |
| Nuxt | `sst.aws.Nuxt` |
| TanStack Start | `sst.aws.TanStackStart` |
| Angular | `sst.aws.Angular` |
| Static Sites | `sst.aws.StaticSite` |

### Cloudflare Components

| Framework | Component |
|-----------|-----------|
| Workers | `sst.cloudflare.Worker` |
| Static Sites | `sst.cloudflare.StaticSite` |

## Next.js

### Basic Setup

```typescript
new sst.aws.Nextjs("MyWeb");
```

### With Custom Domain

```typescript
new sst.aws.Nextjs("MyWeb", {
  domain: "my-app.com"
});
```

### With Subdomain

```typescript
new sst.aws.Nextjs("MyWeb", {
  domain: {
    name: "app.my-domain.com",
    dns: sst.aws.dns()
  }
});
```

### Full Configuration

```typescript
new sst.aws.Nextjs("MyWeb", {
  path: "packages/web",           // Path to Next.js app
  domain: "my-app.com",
  buildCommand: "npm run build",
  environment: {
    NEXT_PUBLIC_API_URL: api.url
  },
  link: [bucket, database],       // Link resources
  imageOptimization: {
    memory: "512 MB"
  },
  warm: 10,                       // Keep 10 instances warm
  transform: {
    server: (args) => {
      args.memory = "1024 MB";
    }
  }
});
```

### Container Mode

For apps that need more resources or longer execution times:

```typescript
new sst.aws.Nextjs("MyWeb", {
  domain: "my-app.com",
  container: true,
  vpc
});
```

## Remix

### Basic Setup

```typescript
new sst.aws.Remix("MyWeb");
```

### Full Configuration

```typescript
new sst.aws.Remix("MyWeb", {
  path: "packages/remix",
  domain: "my-app.com",
  buildCommand: "npm run build",
  environment: {
    DATABASE_URL: database.url
  },
  link: [bucket, database]
});
```

## Astro

### Basic Setup

```typescript
new sst.aws.Astro("MyWeb");
```

### Full Configuration

```typescript
new sst.aws.Astro("MyWeb", {
  path: "packages/astro",
  domain: "my-app.com",
  buildCommand: "npm run build",
  link: [bucket]
});
```

### Container Mode

```typescript
new sst.aws.Astro("MyWeb", {
  container: true,
  vpc,
  domain: "my-app.com"
});
```

## SvelteKit

```typescript
new sst.aws.SvelteKit("MyWeb", {
  domain: "my-app.com",
  link: [bucket]
});
```

## SolidStart

```typescript
new sst.aws.SolidStart("MyWeb", {
  domain: "my-app.com",
  link: [bucket]
});
```

## Nuxt

```typescript
new sst.aws.Nuxt("MyWeb", {
  domain: "my-app.com",
  link: [bucket]
});
```

## TanStack Start

```typescript
new sst.aws.TanStackStart("MyWeb", {
  domain: "my-app.com",
  link: [bucket]
});
```

## Static Sites

For sites without server-side rendering:

```typescript
new sst.aws.StaticSite("MyDocs", {
  path: "packages/docs",
  build: {
    command: "npm run build",
    output: "dist"
  },
  domain: "docs.my-app.com"
});
```

### With Custom Index/Error Pages

```typescript
new sst.aws.StaticSite("MySPA", {
  path: "packages/spa",
  build: {
    command: "npm run build",
    output: "dist"
  },
  errorPage: "index.html"  // For SPA routing
});
```

## Custom Domains

### Using Route 53 (AWS DNS)

```typescript
new sst.aws.Nextjs("MyWeb", {
  domain: "my-app.com"
});
```

### Using Cloudflare DNS

```typescript
new sst.aws.Nextjs("MyWeb", {
  domain: {
    name: "my-app.com",
    dns: sst.cloudflare.dns()
  }
});
```

### With Aliases

```typescript
new sst.aws.Nextjs("MyWeb", {
  domain: {
    name: "my-app.com",
    aliases: ["www.my-app.com"]
  }
});
```

### Redirect www to apex

```typescript
new sst.aws.Nextjs("MyWeb", {
  domain: {
    name: "my-app.com",
    redirects: ["www.my-app.com"]
  }
});
```

## Linking Resources

Link backend resources to access them in your frontend:

```typescript
const bucket = new sst.aws.Bucket("MyBucket");
const database = new sst.aws.Postgres("MyDB", { vpc });

new sst.aws.Nextjs("MyWeb", {
  link: [bucket, database]
});
```

Access in your frontend (server-side only):

```typescript
// app/api/route.ts
import { Resource } from "sst";

export async function GET() {
  console.log(Resource.MyBucket.name);
  console.log(Resource.MyDB.host);
}
```

## Environment Variables

### Build-time Variables

```typescript
new sst.aws.Nextjs("MyWeb", {
  environment: {
    NEXT_PUBLIC_API_URL: api.url,
    STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY
  }
});
```

### Using SST Secrets

```typescript
const stripeKey = new sst.Secret("StripeKey");

new sst.aws.Nextjs("MyWeb", {
  link: [stripeKey]
});
```

## Development

### Local Development

The `sst dev` command starts your frontend locally while linking it to deployed resources:

```bash
sst dev
```

This:
1. Deploys your infrastructure
2. Starts your frontend in dev mode (e.g., `next dev`)
3. Injects linked resources as environment variables

### Basic Mode

For more control, use basic mode:

```bash
sst dev --mode=basic
```

Then wrap your dev command:

```bash
sst dev next dev
```

## Cloudflare Workers

### Basic Worker

```typescript
new sst.cloudflare.Worker("MyWorker", {
  handler: "src/worker.ts"
});
```

### With Custom Domain

```typescript
new sst.cloudflare.Worker("MyWorker", {
  handler: "src/worker.ts",
  domain: "api.my-app.com"
});
```

### With Bindings

```typescript
const kv = new sst.cloudflare.Kv("MyKv");
const d1 = new sst.cloudflare.D1("MyD1");

new sst.cloudflare.Worker("MyWorker", {
  handler: "src/worker.ts",
  link: [kv, d1]
});
```

## Best Practices

1. **Use resource linking**: Access backend resources with type safety
2. **Set up custom domains early**: Avoid changing domains after launch
3. **Use container mode for long operations**: If your app needs > 30s execution
4. **Warm instances for production**: Use `warm` option for consistent cold starts
5. **Separate build from runtime env vars**: Use `NEXT_PUBLIC_*` for client-side

## Common Issues

### Build Failures

Check that your build command is correct and dependencies are installed:

```typescript
new sst.aws.Nextjs("MyWeb", {
  buildCommand: "npm install && npm run build"
});
```

### Domain Not Working

Ensure your DNS is properly configured:
- For Route 53: Domain should be in same AWS account
- For external DNS: Add the required CNAME records

### Environment Variables Missing

Remember:
- Linked resources are only available server-side
- Use `NEXT_PUBLIC_*` prefix for client-side access in Next.js
