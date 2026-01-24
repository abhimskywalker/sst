---
name: sst-auth
description: Add authentication to SST applications using OpenAuth, JWT authorizers, and IAM authorization. Use this skill when implementing user login, protecting API routes, setting up OAuth providers, or integrating authentication with Next.js, Remix, or other frontends.
license: MIT
metadata:
  author: sst
  version: "3.0"
---

# SST Authentication

SST uses OpenAuth for authentication, providing a flexible, self-hosted auth solution that supports multiple providers and works seamlessly with SST's linking system.

## When to use this skill

Use this skill when:
- Adding user authentication to your app
- Protecting API routes with JWT
- Setting up OAuth providers (Google, GitHub, etc.)
- Implementing email/password or magic link auth
- Integrating auth with frontend frameworks

## OpenAuth Overview

OpenAuth is a self-hosted authentication server that:
- Runs as a Lambda function in your AWS account
- Stores data in DynamoDB (managed by SST)
- Supports multiple auth providers
- Issues JWTs for API authorization

## Basic Setup

### 1. Create Auth Component

```typescript
// sst.config.ts
const auth = new sst.aws.Auth("MyAuth", {
  issuer: "auth/index.handler"
});

new sst.aws.Nextjs("MyWeb", {
  link: [auth]
});
```

### 2. Create Issuer Function

```typescript
// auth/index.ts
import { handle } from "hono/aws-lambda";
import { issuer } from "@openauthjs/openauth";
import { CodeUI } from "@openauthjs/openauth/ui/code";
import { CodeProvider } from "@openauthjs/openauth/provider/code";
import { subjects } from "./subjects";

const app = issuer({
  subjects,
  allow: async () => true,  // Configure allowed redirects
  providers: {
    code: CodeProvider(
      CodeUI({
        sendCode: async (email, code) => {
          console.log(`Send ${code} to ${email}`);
          // In production, send via email
        }
      })
    )
  },
  success: async (ctx, value) => {
    if (value.provider === "code") {
      const userId = await getOrCreateUser(value.claims.email);
      return ctx.subject("user", { id: userId });
    }
    throw new Error("Invalid provider");
  }
});

export const handler = handle(app);
```

### 3. Define Subjects

```typescript
// auth/subjects.ts
import { createSubjects } from "@openauthjs/openauth/subject";

export const subjects = createSubjects({
  user: {
    id: "string"
  }
});
```

### 4. Install Dependencies

```bash
npm install @openauthjs/openauth hono
```

## Auth Providers

### Code Provider (Magic Link/OTP)

```typescript
import { CodeUI } from "@openauthjs/openauth/ui/code";
import { CodeProvider } from "@openauthjs/openauth/provider/code";

const app = issuer({
  providers: {
    code: CodeProvider(
      CodeUI({
        sendCode: async (email, code) => {
          // Send code via email
          await sendEmail(email, `Your code: ${code}`);
        }
      })
    )
  },
  // ...
});
```

### Password Provider

```typescript
import { PasswordUI } from "@openauthjs/openauth/ui/password";
import { PasswordProvider } from "@openauthjs/openauth/provider/password";

const app = issuer({
  providers: {
    password: PasswordProvider(
      PasswordUI({
        copy: {
          register_title: "Create Account",
          login_title: "Sign In"
        }
      }),
      async (email, password) => {
        // Validate credentials
        const user = await validateUser(email, password);
        if (user) return user.id;
        throw new Error("Invalid credentials");
      }
    )
  },
  // ...
});
```

### Google OAuth

```typescript
import { GoogleProvider } from "@openauthjs/openauth/provider/google";

const app = issuer({
  providers: {
    google: GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!
    })
  },
  success: async (ctx, value) => {
    if (value.provider === "google") {
      const user = await getOrCreateUser(value.claims.email);
      return ctx.subject("user", { id: user.id });
    }
    // ...
  }
});
```

### GitHub OAuth

```typescript
import { GithubProvider } from "@openauthjs/openauth/provider/github";

const app = issuer({
  providers: {
    github: GithubProvider({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
      scopes: ["user:email"]
    })
  },
  // ...
});
```

### Multiple Providers

```typescript
const app = issuer({
  providers: {
    code: CodeProvider(/* ... */),
    google: GoogleProvider(/* ... */),
    github: GithubProvider(/* ... */)
  },
  success: async (ctx, value) => {
    let email: string;
    
    switch (value.provider) {
      case "code":
        email = value.claims.email;
        break;
      case "google":
        email = value.claims.email;
        break;
      case "github":
        email = value.claims.email;
        break;
      default:
        throw new Error("Unknown provider");
    }
    
    const user = await getOrCreateUser(email);
    return ctx.subject("user", { id: user.id });
  }
});
```

## Protecting API Routes

### JWT Authorization

```typescript
// sst.config.ts
const auth = new sst.aws.Auth("MyAuth", {
  issuer: "auth/index.handler"
});

const api = new sst.aws.ApiGatewayV2("MyApi");

// Public route
api.route("GET /public", "src/public.handler");

// Protected route
api.route("GET /protected", {
  handler: "src/protected.handler",
  auth: {
    jwt: {
      issuer: auth.url,
      audiences: ["my-api"]
    }
  }
});
```

### Accessing User in Handler

```typescript
// src/protected.ts
import { APIGatewayProxyEventV2 } from "aws-lambda";

export async function handler(event: APIGatewayProxyEventV2) {
  // JWT claims are in the request context
  const claims = event.requestContext.authorizer?.jwt?.claims;
  const userId = claims?.sub;
  
  return {
    statusCode: 200,
    body: JSON.stringify({ userId })
  };
}
```

### IAM Authorization

For service-to-service auth:

```typescript
api.route("POST /internal", {
  handler: "src/internal.handler",
  auth: { iam: true }
});
```

## Frontend Integration

### Next.js

```typescript
// app/auth/callback/route.ts
import { Resource } from "sst";
import { createClient } from "@openauthjs/openauth/client";
import { cookies } from "next/headers";

const client = createClient({
  issuer: Resource.MyAuth.url
});

export async function GET(request: Request) {
  const url = new URL(request.url);
  const code = url.searchParams.get("code");
  
  if (code) {
    const tokens = await client.exchange(code);
    
    const cookieStore = await cookies();
    cookieStore.set("access_token", tokens.access, {
      httpOnly: true,
      secure: true,
      sameSite: "lax"
    });
    cookieStore.set("refresh_token", tokens.refresh, {
      httpOnly: true,
      secure: true,
      sameSite: "lax"
    });
    
    return Response.redirect(new URL("/", request.url));
  }
  
  return Response.redirect(client.authorize("code", "openid profile"));
}
```

### Login Button

```typescript
// components/LoginButton.tsx
"use client";

export function LoginButton() {
  return (
    <a href="/auth/callback">
      Sign In
    </a>
  );
}
```

### Protected Server Component

```typescript
// app/dashboard/page.tsx
import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import { Resource } from "sst";
import { createClient } from "@openauthjs/openauth/client";

const client = createClient({
  issuer: Resource.MyAuth.url
});

export default async function Dashboard() {
  const cookieStore = await cookies();
  const accessToken = cookieStore.get("access_token")?.value;
  
  if (!accessToken) {
    redirect("/auth/callback");
  }
  
  const verified = await client.verify(accessToken);
  
  if (!verified) {
    redirect("/auth/callback");
  }
  
  return <div>Welcome, {verified.subject.properties.id}</div>;
}
```

## Custom Domain

```typescript
const auth = new sst.aws.Auth("MyAuth", {
  issuer: "auth/index.handler",
  domain: "auth.my-app.com"
});
```

## With Database

Store user data alongside auth:

```typescript
// sst.config.ts
const vpc = new sst.aws.Vpc("MyVpc");
const database = new sst.aws.Postgres("MyDatabase", { vpc });

const auth = new sst.aws.Auth("MyAuth", {
  issuer: {
    handler: "auth/index.handler",
    link: [database],
    vpc
  }
});
```

```typescript
// auth/index.ts
import { Resource } from "sst";
import postgres from "postgres";

const sql = postgres({
  host: Resource.MyDatabase.host,
  // ... other config
});

async function getOrCreateUser(email: string) {
  const [user] = await sql`
    INSERT INTO users (email)
    VALUES (${email})
    ON CONFLICT (email) DO UPDATE SET email = ${email}
    RETURNING id
  `;
  return user.id;
}
```

## Sending Emails

Integrate with SST Email component:

```typescript
// sst.config.ts
const email = new sst.aws.Email("MyEmail", {
  sender: "auth@my-app.com"
});

const auth = new sst.aws.Auth("MyAuth", {
  issuer: {
    handler: "auth/index.handler",
    link: [email]
  }
});
```

```typescript
// auth/index.ts
import { Resource } from "sst";
import { SESClient, SendEmailCommand } from "@aws-sdk/client-ses";

const ses = new SESClient({});

const app = issuer({
  providers: {
    code: CodeProvider(
      CodeUI({
        sendCode: async (email, code) => {
          await ses.send(new SendEmailCommand({
            Source: Resource.MyEmail.sender,
            Destination: { ToAddresses: [email] },
            Message: {
              Subject: { Data: "Your verification code" },
              Body: { Text: { Data: `Your code is: ${code}` } }
            }
          }));
        }
      })
    )
  },
  // ...
});
```

## Best Practices

1. **Use custom domains in production**: Set `allow` to check redirect URIs
2. **Store refresh tokens securely**: Use HTTP-only cookies
3. **Link to database**: Store user profiles alongside auth
4. **Use environment variables**: Keep OAuth secrets out of code
5. **Implement token refresh**: Handle expired access tokens

## Troubleshooting

### Redirect URI Mismatch

Configure allowed redirects:

```typescript
const app = issuer({
  allow: async (input, req) => {
    const allowed = [
      "http://localhost:3000",
      "https://my-app.com"
    ];
    return allowed.some(url => input.redirectURI.startsWith(url));
  },
  // ...
});
```

### Token Expired

Implement token refresh:

```typescript
const client = createClient({ issuer: Resource.MyAuth.url });

const tokens = await client.refresh(refreshToken);
```

### CORS Issues

The Auth component automatically configures CORS. For custom needs:

```typescript
const auth = new sst.aws.Auth("MyAuth", {
  issuer: "auth/index.handler",
  transform: {
    cdn: (args) => {
      args.origins = [/* custom origins */];
    }
  }
});
```
