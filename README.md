# Claude Code — Architecture Study Notes

<p align="center">
  <a href="./README_zh.md"><img src="https://img.shields.io/badge/文档-中文版-blue?style=for-the-badge" alt="中文版" /></a>
</p>

---

## What This Is

Personal learning notes built around reading the Claude Code source code — Anthropic's official CLI for AI-assisted software engineering. The goal is to understand how a production-grade agentic CLI is architected: how it wires tools, manages permissions, streams LLM responses, and coordinates multi-agent workflows.

> **Ownership disclaimer:** The original Claude Code source code is the property of Anthropic. This repository is not affiliated with, endorsed by, or maintained by Anthropic. Code here is used solely for educational analysis.

---

## Tech Stack

| Category | Technology |
|---|---|
| Runtime | [Bun](https://bun.sh) |
| Language | TypeScript (strict) |
| Terminal UI | [React](https://react.dev) + [Ink](https://github.com/vadimdemedes/ink) |
| CLI Parsing | [Commander.js](https://github.com/tj/commander.js) |
| Schema Validation | [Zod v4](https://zod.dev) |
| Code Search | [ripgrep](https://github.com/BurntSushi/ripgrep) |
| Protocols | MCP SDK, LSP |
| API Client | [Anthropic SDK](https://docs.anthropic.com) |
| Telemetry | OpenTelemetry + gRPC (lazy-loaded) |
| Feature Flags | GrowthBook |
| Auth | OAuth 2.0, JWT, macOS Keychain |
| Linting | Biome |

---

## Architecture Overview

### Entry Point & Startup

`src/main.tsx` is the CLI entry point. It uses Commander.js for argument parsing and launches the React/Ink renderer. A key startup optimization fires MDM settings reads, keychain prefetch, and API preconnect **in parallel** before heavy modules are evaluated, shaving meaningful latency off every cold start.

### Core Query Pipeline

```
User input
  → QueryEngine (src/QueryEngine.ts)
    → streams Anthropic API response
    → detects tool_use blocks
    → dispatches to tool handlers (src/tools/)
    → loops until no more tool calls
  → renders output via Ink components
```

`src/QueryEngine.ts` (~46K lines) is the heart of the system. It handles streaming, the tool-call loop, thinking mode, retry logic, and token counting.

### Tool System (`src/tools/`)

Every capability Claude can invoke is a self-contained module with three things: a Zod input schema, a permission declaration, and a `call()` implementation. The registry lives in `src/tools.ts`.

| Tool | Purpose |
|---|---|
| `BashTool` | Shell command execution |
| `FileReadTool` | Files, images, PDFs, Jupyter notebooks |
| `FileWriteTool` / `FileEditTool` | Write and partial-edit files |
| `GlobTool` / `GrepTool` | File search (ripgrep-backed) |
| `AgentTool` | Spawn sub-agents |
| `SkillTool` | Execute reusable skill workflows |
| `MCPTool` | Invoke MCP server tools |
| `LSPTool` | Language Server Protocol integration |
| `TaskCreateTool` / `TaskUpdateTool` | Task lifecycle management |
| `SendMessageTool` | Inter-agent messaging |
| `EnterPlanModeTool` / `ExitPlanModeTool` | Plan mode toggle |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git worktree isolation |
| `CronCreateTool` | Scheduled triggers |
| `RemoteTriggerTool` | Remote execution |

### Command System (`src/commands/`)

User-facing `/slash` commands. All ~50 commands are registered in `src/commands.ts`. Commands use Bun's `bun:bundle` feature flags for dead-code elimination at build time — inactive features are completely stripped from the bundle.

```typescript
// Example: voice command only included when VOICE_MODE flag is enabled
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Notable flags: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `COORDINATOR_MODE`.

### Service Layer (`src/services/`)

| Service | Purpose |
|---|---|
| `api/` | Anthropic API client, Files API, retry logic, error types |
| `mcp/` | MCP server connections, auth, channel permissions |
| `oauth/` | OAuth 2.0 flow |
| `lsp/` | Language Server Protocol manager |
| `analytics/` | GrowthBook feature flags + event logging |
| `compact/` | Conversation context compression |
| `plugins/` | Plugin loader |
| `policyLimits/` | Org policy enforcement |
| `extractMemories/` | Automatic memory extraction |

### Permission System (`src/hooks/toolPermission/`)

Every tool invocation passes through a centralized permission check. Permission modes: `default`, `plan`, `bypassPermissions`, `auto`. The system either prompts the user for approval or resolves automatically based on the configured mode.

### Bridge System (`src/bridge/`)

A bidirectional communication layer between IDE extensions (VS Code, JetBrains) and the CLI. Uses JWT authentication. `bridgeMain.ts` runs the main bridge loop; `replBridge.ts` handles interactive REPL sessions.

### Multi-Agent Architecture

- **Coordinator** (`src/coordinator/`): orchestrates parallel agents
- **Teams** (`TeamCreateTool`): group agents for coordinated tasks
- **Skills** (`src/skills/`): reusable named workflows, user-extensible
- **Sub-agents** (`AgentTool`): spawn isolated agent instances with their own context

---

## Directory Map

```
src/
├── main.tsx              # CLI entry point (Commander.js + Ink renderer)
├── QueryEngine.ts        # Core LLM streaming + tool-call loop
├── Tool.ts               # Tool type definitions & permission model
├── commands.ts           # Slash command registry
├── tools.ts              # Tool registry
├── context.ts            # System/user context collection (cached per session)
├── cost-tracker.ts       # Token counting & USD cost tracking
│
├── commands/             # ~50 slash command implementations
├── tools/                # ~40 agent tool implementations
├── components/           # ~140 Ink terminal UI components
├── services/             # External integrations (API, MCP, OAuth, LSP…)
├── bridge/               # IDE extension bridge
├── coordinator/          # Multi-agent coordination
├── skills/               # Reusable skill workflows
├── plugins/              # Plugin system
├── hooks/                # React hooks (incl. permission system)
├── screens/              # Full-screen UIs (Doctor, REPL, Resume)
├── state/                # App state (AppState.tsx)
├── types/                # TypeScript type definitions
├── utils/                # Utility functions (config, auth, git, shell…)
├── schemas/              # Zod config schemas
├── entrypoints/          # Initialization & Agent SDK types
├── memdir/               # Persistent memory directory
├── tasks/                # Task management
├── migrations/           # Config migrations
├── vim/                  # Vim mode
├── voice/                # Voice input
├── remote/               # Remote sessions
├── server/               # Server mode
└── keybindings/          # Keybinding configuration
```
