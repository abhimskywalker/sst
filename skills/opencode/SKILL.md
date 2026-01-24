---
name: opencode
description: Use OpenCode, the open source AI coding agent. Use this skill when working with OpenCode CLI, TUI, web interface, configuring agents, creating custom tools, setting up MCP servers, or using the OpenCode SDK for programmatic access.
license: MIT
metadata:
  author: anomaly
  version: "1.1"
---

# OpenCode

OpenCode is an open source AI coding agent available as a terminal-based interface (TUI), desktop app, or IDE extension. It supports multiple LLM providers, custom tools, agents, and integrates with the Agent Skills format.

## When to use this skill

Use this skill when:
- Setting up and configuring OpenCode
- Creating custom agents with specific behaviors
- Building custom tools for OpenCode
- Configuring MCP servers
- Using the OpenCode SDK programmatically
- Running OpenCode in server/headless mode

## Installation

### Install Script (recommended)

```bash
curl -fsSL https://opencode.ai/install | bash
```

### Other Methods

```bash
# Using Homebrew
brew install anomalyco/tap/opencode

# Using npm
npm install -g opencode

# Using bun
bun install -g opencode
```

## Basic Usage

### Start TUI

```bash
opencode
```

### Start with a Prompt

```bash
opencode run "explain this codebase"
```

### Initialize Project

```bash
opencode init
```

This creates an `AGENTS.md` file for project-specific instructions.

## Configuration

OpenCode uses `opencode.json` for configuration:

```json
{
  "models": {
    "default": "anthropic/claude-sonnet-4-20250514",
    "fast": "anthropic/claude-haiku-3-5"
  },
  "tools": {
    "bash": true,
    "edit": true,
    "read": true
  },
  "mcp": {},
  "agents": {}
}
```

### Configuration Locations

- **Project**: `./opencode.json`
- **Global**: `~/.config/opencode/opencode.json`

## Agents

Agents are specialized AI assistants with custom prompts, models, and tool access.

### Built-in Agents

| Agent | Mode | Description |
|-------|------|-------------|
| **Build** | Primary | Default agent with all tools enabled |
| **Plan** | Primary | Read-only, creates plans without changes |
| **General** | Subagent | Multi-step tasks, has full tool access |
| **Explore** | Subagent | Fast, read-only codebase exploration |

### Custom Agent (Markdown)

Create `.opencode/agents/review.md`:

```markdown
---
description: Code review specialist that analyzes changes for quality
mode: subagent
temperature: 0.2
tools:
  edit: false
  bash: false
---

# Code Review Agent

You are a code review specialist. Analyze code changes for:
- Code quality and best practices
- Potential bugs or security issues
- Performance considerations
- Documentation completeness

Provide actionable feedback without making changes.
```

### Custom Agent (JSON)

In `opencode.json`:

```json
{
  "agents": {
    "security": {
      "description": "Security-focused code analyzer",
      "mode": "subagent",
      "temperature": 0.1,
      "prompt": "./prompts/security.md",
      "tools": {
        "edit": false,
        "bash": false
      }
    }
  }
}
```

### Switching Agents

- **Tab key**: Cycle through primary agents
- **@ mention**: Invoke subagent (e.g., `@explore find all API routes`)

## Custom Tools

Create custom tools in TypeScript/JavaScript:

### Single Tool

Create `.opencode/tools/database.ts`:

```typescript
import { tool } from "opencode/tool";

export default tool({
  description: "Query the database",
  args: (z) => ({
    query: z.string().describe("SQL query to execute")
  }),
  async execute({ args }) {
    // Execute query...
    return { result: "Query executed" };
  }
});
```

### Multiple Tools

Create `.opencode/tools/math.ts`:

```typescript
import { tool } from "opencode/tool";

export const add = tool({
  description: "Add two numbers",
  args: (z) => ({
    a: z.number(),
    b: z.number()
  }),
  execute: ({ args }) => ({ result: args.a + args.b })
});

export const multiply = tool({
  description: "Multiply two numbers",
  args: (z) => ({
    a: z.number(),
    b: z.number()
  }),
  execute: ({ args }) => ({ result: args.a * args.b })
});
```

### Using External Scripts

```typescript
import { tool } from "opencode/tool";

export default tool({
  description: "Run Python analysis",
  args: (z) => ({
    data: z.string()
  }),
  async execute({ args, $ }) {
    const result = await $`python analyze.py ${args.data}`.text();
    return { result };
  }
});
```

## MCP Servers

Add Model Context Protocol servers for external tools:

### Local MCP Server

```json
{
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "@modelcontextprotocol/server-filesystem", "/path"]
    }
  }
}
```

### Remote MCP Server

```json
{
  "mcp": {
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/sse"
    }
  }
}
```

### Authenticate with MCP

```bash
opencode mcp auth sentry
```

## Server Mode

Run OpenCode as a headless HTTP server:

```bash
# Start server
opencode serve --port 4096

# With authentication
OPENCODE_SERVER_PASSWORD=secret opencode serve

# Attach TUI to running server
opencode attach --port 4096
```

### Web Interface

```bash
# Start web interface
opencode web --port 8080

# Access at http://localhost:8080
```

## SDK Usage

```typescript
import { opencode } from "opencode/sdk";

// Create instance (starts server + client)
const oc = await opencode();

// Or connect to existing server
const client = await opencode.connect({ port: 4096 });

// Create session
const session = await oc.session.create({
  agent: "build"
});

// Send message
await oc.message.create({
  sessionID: session.id,
  content: "Explain this codebase"
});

// Subscribe to events
oc.on("message.updated", (msg) => {
  console.log(msg.content);
});
```

## Skills

OpenCode supports the Agent Skills format (agentskills.io):

### Skill Locations

- **Project**: `.opencode/skills/<name>/SKILL.md`
- **Global**: `~/.config/opencode/skills/<name>/SKILL.md`
- **Claude-compatible**: `.claude/skills/<name>/SKILL.md`

### Skill Permissions

```json
{
  "skills": {
    "allow": ["*"],
    "deny": ["internal-*"]
  }
}
```

## Plugins

Create plugins in `.opencode/plugins/`:

```typescript
import type { Plugin } from "opencode/plugin";

export default function myPlugin(): Plugin {
  return {
    name: "my-plugin",
    setup({ on, client }) {
      on("session.created", (session) => {
        console.log("New session:", session.id);
      });
      
      on("tool.execute.before", (event) => {
        if (event.tool === "bash" && event.args.command.includes("rm -rf")) {
          return { cancel: true, message: "Dangerous command blocked" };
        }
      });
    }
  };
}
```

## CLI Commands

| Command | Description |
|---------|-------------|
| `opencode` | Start TUI |
| `opencode run "prompt"` | Non-interactive mode |
| `opencode serve` | Headless server |
| `opencode web` | Web interface |
| `opencode attach` | Attach to running server |
| `opencode models` | List available models |
| `opencode mcp list` | List MCP servers |
| `opencode mcp auth <name>` | Authenticate MCP server |

## Key Bindings (TUI)

| Key | Action |
|-----|--------|
| `Tab` | Switch primary agent |
| `@` | Fuzzy file search |
| `Ctrl+C` | Interrupt |
| `/undo` | Undo last change |
| `/redo` | Redo change |
| `/share` | Share conversation |

## Best Practices

1. **Use agents for different tasks**: Plan first, then build
2. **Create project-specific tools**: Automate repetitive tasks
3. **Use MCP for integrations**: Connect to external services
4. **Commit AGENTS.md**: Share project context with team
5. **Use server mode for automation**: Enable programmatic access
