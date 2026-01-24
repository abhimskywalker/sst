---
name: sst-backend
description: Build backend APIs with SST using Lambda functions, containers, API Gateway, Hono, Express, and other frameworks. Use this skill when creating serverless APIs, deploying containerized services, setting up API routes, configuring CORS, or building real-time applications with WebSockets.
license: MIT
metadata:
  author: sst
  version: "3.0"
---

# SST Backend & APIs

SST provides powerful components for building backend services, from serverless Lambda functions to containerized applications running on ECS.

## When to use this skill

Use this skill when:
- Creating Lambda functions or API endpoints
- Deploying containerized applications (Express, Hono, NestJS)
- Setting up API Gateway with custom routes
- Configuring WebSocket APIs
- Building event-driven architectures with queues and topics

## Lambda Functions

### Basic Function

```typescript
new sst.aws.Function("MyFunction", {
  handler: "src/handler.main"
});
```

### Function with URL

Create a function with a public HTTP endpoint:

```typescript
new sst.aws.Function("MyApi", {
  url: true,
  handler: "src/api.handler"
});
```

### Full Configuration

```typescript
new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  url: true,
  timeout: "30 seconds",
  memory: "1024 MB",
  runtime: "nodejs20.x",
  architecture: "arm64",
  environment: {
    MY_VAR: "value"
  },
  link: [bucket, database],
  permissions: [
    {
      actions: ["ses:SendEmail"],
      resources: ["*"]
    }
  ],
  vpc,                          // Deploy in VPC
  transform: {
    function: (args) => {
      args.reservedConcurrentExecutions = 10;
    }
  }
});
```

### Function Handler

```typescript
// src/handler.ts
import { Resource } from "sst";

export async function main(event: any) {
  console.log(Resource.MyBucket.name);
  
  return {
    statusCode: 200,
    body: JSON.stringify({ message: "Hello, World!" })
  };
}
```

### Different Runtimes

```typescript
// Node.js (default)
new sst.aws.Function("NodeFunction", {
  handler: "src/handler.main",
  runtime: "nodejs20.x"
});

// Python
new sst.aws.Function("PythonFunction", {
  handler: "src/handler.main",
  runtime: "python3.11"
});

// Go
new sst.aws.Function("GoFunction", {
  handler: "bootstrap",
  runtime: "provided.al2023",
  bundle: "functions/go"
});

// Rust
new sst.aws.Function("RustFunction", {
  handler: "bootstrap",
  runtime: "provided.al2023",
  architecture: "arm64"
});
```

## Hono API

Hono is a lightweight web framework perfect for serverless:

```typescript
new sst.aws.Function("Hono", {
  url: true,
  handler: "src/index.handler",
  link: [bucket]
});
```

Handler:

```typescript
// src/index.ts
import { Hono } from "hono";
import { handle } from "hono/aws-lambda";
import { Resource } from "sst";

const app = new Hono();

app.get("/", (c) => c.json({ message: "Hello!" }));

app.get("/bucket", (c) => {
  return c.json({ bucket: Resource.MyBucket.name });
});

app.post("/upload", async (c) => {
  const body = await c.req.json();
  // Upload to S3...
  return c.json({ success: true });
});

export const handler = handle(app);
```

## Express in Containers

For Express apps or when you need more resources:

### Basic Container Service

```typescript
const vpc = new sst.aws.Vpc("MyVpc");
const cluster = new sst.aws.Cluster("MyCluster", { vpc });

new sst.aws.Service("MyService", {
  cluster,
  loadBalancer: {
    ports: [{ listen: "80/http" }]
  }
});
```

### With Dockerfile

```dockerfile
# Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 80
CMD ["node", "server.js"]
```

```typescript
new sst.aws.Service("MyService", {
  cluster,
  loadBalancer: {
    ports: [{ listen: "80/http", forward: "3000/http" }]
  },
  image: {
    dockerfile: "Dockerfile"
  }
});
```

### Full Configuration

```typescript
const vpc = new sst.aws.Vpc("MyVpc");
const cluster = new sst.aws.Cluster("MyCluster", { vpc });

new sst.aws.Service("MyService", {
  cluster,
  cpu: "1 vCPU",
  memory: "2 GB",
  scaling: {
    min: 1,
    max: 10,
    cpuUtilization: 70,
    memoryUtilization: 70
  },
  loadBalancer: {
    ports: [
      { listen: "443/https", forward: "3000/http" }
    ],
    domain: "api.my-app.com"
  },
  link: [bucket, database],
  environment: {
    NODE_ENV: "production"
  },
  dev: {
    command: "npm run dev"
  }
});
```

## API Gateway

### HTTP API (v2)

```typescript
const api = new sst.aws.ApiGatewayV2("MyApi");

api.route("GET /", "src/handlers/index.main");
api.route("GET /users", "src/handlers/users.list");
api.route("POST /users", "src/handlers/users.create");
api.route("GET /users/{id}", "src/handlers/users.get");
api.route("PUT /users/{id}", "src/handlers/users.update");
api.route("DELETE /users/{id}", "src/handlers/users.delete");
```

### With Custom Domain

```typescript
const api = new sst.aws.ApiGatewayV2("MyApi", {
  domain: "api.my-app.com"
});
```

### With Authorization

```typescript
const api = new sst.aws.ApiGatewayV2("MyApi");

// JWT Authorizer
api.route("GET /protected", {
  handler: "src/protected.handler",
  auth: {
    jwt: {
      issuer: "https://auth.my-app.com",
      audiences: ["my-api"]
    }
  }
});

// IAM Authorization
api.route("GET /admin", {
  handler: "src/admin.handler",
  auth: { iam: true }
});
```

### REST API (v1)

```typescript
const api = new sst.aws.ApiGatewayV1("MyApi");

api.route("GET /", "src/index.handler");
api.route("POST /webhook", "src/webhook.handler");
```

## WebSocket API

```typescript
const ws = new sst.aws.ApiGatewayWebSocket("MyWebSocket");

ws.route("$connect", "src/connect.handler");
ws.route("$disconnect", "src/disconnect.handler");
ws.route("$default", "src/default.handler");
ws.route("sendMessage", "src/sendMessage.handler");
```

Handler for sending messages:

```typescript
// src/sendMessage.ts
import { ApiGatewayManagementApiClient, PostToConnectionCommand } from "@aws-sdk/client-apigatewaymanagementapi";

export async function handler(event: any) {
  const { connectionId, domainName, stage } = event.requestContext;
  const client = new ApiGatewayManagementApiClient({
    endpoint: `https://${domainName}/${stage}`
  });

  await client.send(new PostToConnectionCommand({
    ConnectionId: connectionId,
    Data: JSON.stringify({ message: "Hello!" })
  }));

  return { statusCode: 200 };
}
```

## Realtime (AWS AppSync Events)

For real-time subscriptions:

```typescript
const realtime = new sst.aws.Realtime("MyRealtime", {
  authorizer: "src/authorizer.handler"
});

new sst.aws.Nextjs("MyWeb", {
  link: [realtime]
});
```

Client usage:

```typescript
import { Resource } from "sst/client";
import { realtime } from "sst/realtime";

const client = realtime({
  authorizer: async () => {
    // Return token for authorization
    return { token: "..." };
  }
});

client.subscribe("chat/room-1", (message) => {
  console.log(message);
});

client.publish("chat/room-1", { text: "Hello!" });
```

## Cron Jobs

```typescript
new sst.aws.Cron("DailyReport", {
  job: "src/cron/daily-report.handler",
  schedule: "rate(1 day)"
});

// Or with cron expression
new sst.aws.Cron("HourlySync", {
  job: "src/cron/sync.handler",
  schedule: "cron(0 * * * ? *)"  // Every hour
});
```

## Queues

### Basic Queue

```typescript
const queue = new sst.aws.Queue("MyQueue");

queue.subscribe("src/subscriber.handler");
```

### Queue Handler

```typescript
// src/subscriber.ts
import { SQSEvent } from "aws-lambda";

export async function handler(event: SQSEvent) {
  for (const record of event.Records) {
    const body = JSON.parse(record.body);
    console.log("Processing:", body);
  }
}
```

### FIFO Queue

```typescript
const queue = new sst.aws.Queue("MyFifoQueue", {
  fifo: true
});
```

### Dead Letter Queue

```typescript
const dlq = new sst.aws.Queue("MyDLQ");

const queue = new sst.aws.Queue("MyQueue", {
  dlq: dlq.arn
});
```

## SNS Topics

```typescript
const topic = new sst.aws.SnsTopic("MyTopic");

topic.subscribe("src/subscriber.handler");

// Or subscribe a queue
topic.subscribeQueue(queue.arn);
```

## EventBridge Bus

```typescript
const bus = new sst.aws.Bus("MyBus");

bus.subscribe("src/handler.handler", {
  pattern: {
    source: ["my-app"],
    detailType: ["OrderCreated"]
  }
});
```

Publishing events:

```typescript
import { EventBridgeClient, PutEventsCommand } from "@aws-sdk/client-eventbridge";
import { Resource } from "sst";

const client = new EventBridgeClient({});

await client.send(new PutEventsCommand({
  Entries: [{
    EventBusName: Resource.MyBus.name,
    Source: "my-app",
    DetailType: "OrderCreated",
    Detail: JSON.stringify({ orderId: "123" })
  }]
}));
```

## Tasks (Long-running Jobs)

For jobs that run longer than Lambda's 15-minute limit:

```typescript
const vpc = new sst.aws.Vpc("MyVpc");
const cluster = new sst.aws.Cluster("MyCluster", { vpc });

const task = new sst.aws.Task("MyTask", {
  cluster,
  handler: "src/task.handler",
  link: [bucket]
});
```

Triggering tasks:

```typescript
import { Resource } from "sst";
import { ECSClient, RunTaskCommand } from "@aws-sdk/client-ecs";

const client = new ECSClient({});

await client.send(new RunTaskCommand({
  cluster: Resource.MyCluster.name,
  taskDefinition: Resource.MyTask.taskDefinition,
  launchType: "FARGATE",
  // ... network configuration
}));
```

## Best Practices

1. **Use Hono for simple APIs**: Lightweight and fast
2. **Use containers for complex apps**: Better for Express/NestJS with many dependencies
3. **Set appropriate timeouts**: Don't over-provision Lambda timeouts
4. **Use ARM architecture**: Better price/performance for Lambda
5. **Link resources**: Use SST's linking for type-safe resource access
6. **Use queues for async work**: Decouple long-running operations

## Common Patterns

### API with Authentication (OpenAuth)

```typescript
// Set up OpenAuth
const auth = new sst.aws.Auth("MyAuth", {
  issuer: "auth/index.handler"
});

const api = new sst.aws.ApiGatewayV2("MyApi");

api.route("POST /public", "src/public.handler");
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

### Microservices

```typescript
const vpc = new sst.aws.Vpc("MyVpc");
const cluster = new sst.aws.Cluster("MyCluster", { vpc });

// User Service
new sst.aws.Service("UserService", {
  cluster,
  loadBalancer: { ports: [{ listen: "80/http" }] }
});

// Order Service
new sst.aws.Service("OrderService", {
  cluster,
  loadBalancer: { ports: [{ listen: "80/http" }] }
});
```
