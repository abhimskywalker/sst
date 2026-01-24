---
name: opencode-sandbox
description: Create isolated sandbox environments for AI coding agents using OpenCode and SST. Use this skill when setting up remote development environments, creating persistent agent sessions with tmux, deploying OpenCode in containers on AWS, or building teleport-like secure access to agent sandboxes.
license: MIT
metadata:
  author: sst
  version: "1.0"
---

# OpenCode Sandbox Environments

Create isolated, persistent sandbox environments for AI coding agents using OpenCode with SST infrastructure. This enables teleport-like secure access to long-running, resumable agent sessions.

## When to use this skill

Use this skill when:
- Setting up remote sandbox environments for AI agents
- Creating persistent OpenCode sessions that survive disconnects
- Deploying OpenCode in containers on AWS ECS
- Building secure, tunnel-based access to agent environments
- Running long-running AI tasks that need to resume

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Your Machine                              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐       │
│  │   Browser   │     │   Terminal  │     │    IDE      │       │
│  │ (Web UI)    │     │  (Attach)   │     │ (Plugin)    │       │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘       │
│         │                   │                   │               │
│         └───────────────────┼───────────────────┘               │
│                             │                                   │
│                     ┌───────▼───────┐                           │
│                     │  SST Tunnel   │                           │
│                     └───────┬───────┘                           │
└─────────────────────────────┼───────────────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   AWS VPC         │
                    │  ┌─────────────┐  │
                    │  │ ECS Service │  │
                    │  │ ┌─────────┐ │  │
                    │  │ │  tmux   │ │  │
                    │  │ │ ┌─────┐ │ │  │
                    │  │ │ │Open │ │ │  │
                    │  │ │ │Code │ │ │  │
                    │  │ │ │serve│ │ │  │
                    │  │ │ └─────┘ │ │  │
                    │  │ └─────────┘ │  │
                    │  └─────────────┘  │
                    └───────────────────┘
```

## SST Infrastructure

### Basic Sandbox Service

```typescript
// sst.config.ts
const vpc = new sst.aws.Vpc("SandboxVpc", {
  bastion: true,  // Enable tunnel access
  nat: "ec2"
});

const cluster = new sst.aws.Cluster("SandboxCluster", { vpc });

// OpenCode sandbox service
const sandbox = new sst.aws.Service("OpenCodeSandbox", {
  cluster,
  cpu: "2 vCPU",
  memory: "4 GB",
  image: {
    dockerfile: "Dockerfile.sandbox"
  },
  scaling: {
    min: 1,
    max: 1  // Single persistent instance
  },
  dev: {
    command: "opencode serve --hostname 0.0.0.0 --port 4096"
  }
});
```

### Dockerfile for Sandbox

```dockerfile
# Dockerfile.sandbox
FROM node:20-bookworm

# Install dependencies
RUN apt-get update && apt-get install -y \
    tmux \
    git \
    curl \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install OpenCode
RUN npm install -g opencode

# Install common dev tools
RUN npm install -g typescript tsx bun

# Create workspace
WORKDIR /workspace

# Copy OpenCode config
COPY opencode.json /root/.config/opencode/opencode.json

# Start script
COPY start-sandbox.sh /start-sandbox.sh
RUN chmod +x /start-sandbox.sh

EXPOSE 4096

CMD ["/start-sandbox.sh"]
```

### Start Script with tmux

```bash
#!/bin/bash
# start-sandbox.sh

# Create or attach to tmux session
SESSION_NAME="opencode"

# Check if session exists
if tmux has-session -t $SESSION_NAME 2>/dev/null; then
    echo "Session $SESSION_NAME already exists"
else
    # Create new session
    tmux new-session -d -s $SESSION_NAME
    
    # Start OpenCode server in the session
    tmux send-keys -t $SESSION_NAME "opencode serve --hostname 0.0.0.0 --port 4096" Enter
fi

# Keep container running
tail -f /dev/null
```

## Secure Access with Tunnel

### Install SST Tunnel

```bash
sudo sst tunnel install
```

### Access Sandbox Locally

```bash
# Start dev (creates tunnel automatically)
sst dev

# OpenCode is now accessible at the service's internal address
# Connect from another terminal:
opencode attach --hostname <service-internal-ip> --port 4096
```

### Web Access Through Load Balancer

```typescript
// sst.config.ts - Add load balancer for web access
const sandbox = new sst.aws.Service("OpenCodeSandbox", {
  cluster,
  cpu: "2 vCPU",
  memory: "4 GB",
  image: { dockerfile: "Dockerfile.sandbox" },
  loadBalancer: {
    ports: [
      { listen: "443/https", forward: "4096/http" }
    ],
    domain: "sandbox.my-app.com"
  },
  environment: {
    OPENCODE_SERVER_PASSWORD: process.env.SANDBOX_PASSWORD!,
    OPENCODE_SERVER_USERNAME: "agent"
  }
});
```

## Long-Running Sessions

### Session Persistence

OpenCode sessions are stored locally. For persistent sessions across container restarts, mount an EFS volume:

```typescript
// sst.config.ts
const efs = new sst.aws.Efs("SandboxStorage", { vpc });

const sandbox = new sst.aws.Service("OpenCodeSandbox", {
  cluster,
  cpu: "2 vCPU",
  memory: "4 GB",
  image: { dockerfile: "Dockerfile.sandbox" },
  volumes: [
    {
      efs,
      path: "/root/.local/share/opencode"  // OpenCode data directory
    },
    {
      efs,
      path: "/workspace"  // Project files
    }
  ]
});
```

### tmux Session Management

Create a wrapper script for managing tmux sessions:

```bash
#!/bin/bash
# opencode-session.sh

SESSION="$1"
shift

case "$1" in
  create)
    tmux new-session -d -s "$SESSION"
    tmux send-keys -t "$SESSION" "cd /workspace && opencode" Enter
    echo "Created session: $SESSION"
    ;;
  attach)
    tmux attach -t "$SESSION"
    ;;
  list)
    tmux list-sessions
    ;;
  kill)
    tmux kill-session -t "$SESSION"
    ;;
  send)
    shift
    tmux send-keys -t "$SESSION" "$*" Enter
    ;;
  *)
    echo "Usage: $0 <session-name> {create|attach|list|kill|send <command>}"
    ;;
esac
```

### Multiple Agent Sessions

Run multiple OpenCode instances for different tasks:

```bash
# In container

# Create project-specific sessions
tmux new-session -d -s frontend
tmux send-keys -t frontend "cd /workspace/frontend && opencode serve --port 4097" Enter

tmux new-session -d -s backend
tmux send-keys -t backend "cd /workspace/backend && opencode serve --port 4098" Enter

tmux new-session -d -s infra
tmux send-keys -t infra "cd /workspace/infra && opencode serve --port 4099" Enter
```

## Multi-User Sandbox

### User Isolation with Tasks

```typescript
// sst.config.ts
const cluster = new sst.aws.Cluster("SandboxCluster", { vpc });

// Task for on-demand user sandboxes
const sandboxTask = new sst.aws.Task("UserSandbox", {
  cluster,
  cpu: "1 vCPU",
  memory: "2 GB",
  handler: "sandbox/handler.main",
  vpc
});

// API to spawn sandboxes
const api = new sst.aws.Function("SandboxApi", {
  url: true,
  handler: "api/sandbox.handler",
  link: [sandboxTask, cluster],
  vpc
});
```

### Sandbox Spawner

```typescript
// api/sandbox.ts
import { Resource } from "sst";
import { ECSClient, RunTaskCommand } from "@aws-sdk/client-ecs";

const ecs = new ECSClient({});

export async function handler(event: any) {
  const { userId, project } = JSON.parse(event.body);
  
  // Spawn sandbox for user
  const result = await ecs.send(new RunTaskCommand({
    cluster: Resource.SandboxCluster.name,
    taskDefinition: Resource.UserSandbox.taskDefinition,
    launchType: "FARGATE",
    networkConfiguration: {
      awsvpcConfiguration: {
        subnets: Resource.SandboxVpc.privateSubnets,
        securityGroups: [Resource.SandboxVpc.securityGroup]
      }
    },
    overrides: {
      containerOverrides: [{
        name: "sandbox",
        environment: [
          { name: "USER_ID", value: userId },
          { name: "PROJECT", value: project },
          { name: "OPENCODE_SERVER_PASSWORD", value: generateToken() }
        ]
      }]
    }
  }));
  
  return {
    statusCode: 200,
    body: JSON.stringify({
      taskArn: result.tasks?.[0]?.taskArn,
      // Return connection info
    })
  };
}
```

## OpenCode Configuration

### Sandbox-Optimized Config

```json
{
  "models": {
    "default": "anthropic/claude-sonnet-4-20250514"
  },
  "tools": {
    "bash": true,
    "edit": true,
    "read": true,
    "webfetch": true
  },
  "permissions": {
    "bash": {
      "allow": ["git:*", "npm:*", "bun:*", "python:*"],
      "deny": ["rm -rf /*", "sudo:*"]
    }
  },
  "agents": {
    "sandbox": {
      "description": "Sandbox development agent with full access",
      "mode": "primary",
      "tools": {
        "bash": true,
        "edit": true
      }
    }
  }
}
```

## SDK Integration

### Remote Sandbox Client

```typescript
import { opencode } from "opencode/sdk";

async function connectToSandbox(host: string, password: string) {
  const client = await opencode.connect({
    hostname: host,
    port: 4096,
    auth: {
      username: "agent",
      password
    }
  });
  
  // Create or resume session
  let session = await client.session.list()
    .then(sessions => sessions.find(s => s.agent === "sandbox"));
  
  if (!session) {
    session = await client.session.create({ agent: "sandbox" });
  }
  
  return { client, session };
}

// Usage
const { client, session } = await connectToSandbox(
  "sandbox.internal",
  process.env.SANDBOX_PASSWORD!
);

// Send task
await client.message.create({
  sessionID: session.id,
  content: "Implement the user authentication feature"
});

// Subscribe to updates
client.on("session.idle", async () => {
  console.log("Task completed!");
  // Trigger next task or notify
});
```

## Best Practices

1. **Use EFS for persistence**: Mount workspace and OpenCode data to survive restarts
2. **tmux for resumability**: Sessions survive disconnects
3. **VPC + Tunnel for security**: Keep sandboxes private, access via tunnel
4. **Resource limits**: Set CPU/memory limits per sandbox
5. **Session timeouts**: Implement idle shutdown to save costs
6. **Audit logging**: Track agent actions via plugins

## Cost Optimization

### Spot Instances

```typescript
const cluster = new sst.aws.Cluster("SandboxCluster", { vpc });

const sandbox = new sst.aws.Service("OpenCodeSandbox", {
  cluster,
  capacity: {
    spot: true,
    spotMaxPrice: "0.05"
  },
  // ... other config
});
```

### Auto-Shutdown

Create a plugin that shuts down idle sandboxes:

```typescript
// .opencode/plugins/auto-shutdown.ts
import type { Plugin } from "opencode/plugin";

let lastActivity = Date.now();
const IDLE_TIMEOUT = 30 * 60 * 1000; // 30 minutes

export default function autoShutdown(): Plugin {
  return {
    name: "auto-shutdown",
    setup({ on }) {
      // Track activity
      on("message.updated", () => {
        lastActivity = Date.now();
      });
      
      // Check idle timeout
      setInterval(() => {
        if (Date.now() - lastActivity > IDLE_TIMEOUT) {
          console.log("Idle timeout reached, shutting down...");
          process.exit(0);
        }
      }, 60000);
    }
  };
}
```

## Troubleshooting

### Container Won't Start

Check ECS task logs:

```bash
aws logs get-log-events \
  --log-group-name /ecs/OpenCodeSandbox \
  --log-stream-name <stream-name>
```

### Can't Connect via Tunnel

1. Ensure bastion is enabled in VPC
2. Run `sudo sst tunnel install`
3. Check tunnel status in `sst dev` multiplexer

### Session Lost After Restart

Mount EFS for persistent storage:

```typescript
volumes: [
  { efs, path: "/root/.local/share/opencode" }
]
```
