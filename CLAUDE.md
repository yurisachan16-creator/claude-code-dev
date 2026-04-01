# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Context

This is a **source snapshot archive** of Anthropic's Claude Code CLI, made publicly accessible on 2026-03-31 via an exposed `.map` file in the npm distribution. It is maintained for educational study of agentic developer tooling architecture and supply-chain security research. It is **not** an official Anthropic repository and has no build infrastructure (no `package.json`, no test runner, no lint config).

The codebase is read-only for analysis purposes. Do not attempt to build or run it.

## Tech Stack

- **Runtime**: Bun
- **Language**: TypeScript (strict mode)
- **Terminal UI**: React + [Ink](https://github.com/vadimdemedes/ink)
- **CLI parsing**: Commander.js
- **Schema validation**: Zod v4
- **Code search**: ripgrep
- **Linting**: Biome (`// biome-ignore` comments throughout)
- **Feature flags / A-B**: GrowthBook
- **Protocols**: MCP (Model Context Protocol), LSP
- **Telemetry**: OpenTelemetry + gRPC (lazy-loaded)

## Architecture

### Entry Point

`src/main.tsx` is the CLI entry point. It uses Commander.js for argument parsing, launches the React/Ink renderer, and fires parallel prefetch calls (MDM settings, keychain, API preconnect) before module evaluation to reduce startup latency.

### Core Pipeline

```
User input → QueryEngine (src/QueryEngine.ts)
           → streams Anthropic API response
           → detects tool_use blocks
           → dispatches to tool handlers
           → loops until no more tool calls
```

`src/query.ts` and `src/QueryEngine.ts` together implement this streaming tool-call loop, including thinking mode, retry logic, and token counting.

### Tool System (`src/tools/`)

Each tool is a self-contained module defining its Zod input schema, permission model, and `call()` implementation. The registry is in `src/tools.ts`. Key tools:

| Tool | Purpose |
|---|---|
| `BashTool` | Shell execution |
| `FileReadTool` | Files, images, PDFs, Jupyter notebooks |
| `FileWriteTool` / `FileEditTool` | Write and partial-edit files |
| `GlobTool` / `GrepTool` | File search (ripgrep-backed) |
| `AgentTool` | Spawn sub-agents |
| `SkillTool` | Execute reusable skill workflows |
| `MCPTool` | Invoke MCP server tools |
| `LSPTool` | Language Server Protocol integration |
| `TaskCreateTool` / `TaskUpdateTool` | Task lifecycle |
| `SendMessageTool` | Inter-agent messaging |
| `EnterPlanModeTool` / `ExitPlanModeTool` | Plan mode |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git worktree isolation |
| `CronCreateTool` | Scheduled triggers |
| `RemoteTriggerTool` | Remote execution |

### Command System (`src/commands/`)

User-facing `/slash` commands. Registry in `src/commands.ts`. Commands are conditionally imported using Bun feature flags to eliminate dead code at build time.

### Service Layer (`src/services/`)

| Service | Purpose |
|---|---|
| `api/` | Anthropic API client, Files API, retry logic, error types |
| `mcp/` | MCP server connection, auth, channel permissions |
| `oauth/` | OAuth 2.0 flow |
| `lsp/` | Language Server Protocol manager |
| `analytics/` | GrowthBook feature flags + analytics |
| `compact/` | Conversation context compression |
| `plugins/` | Plugin loader |
| `policyLimits/` | Org policy enforcement |
| `extractMemories/` | Auto memory extraction |

### UI Layer (`src/components/`)

~140 Ink components. `src/components/App.tsx` is the main shell. State lives in `src/state/AppState.tsx`.

### Bridge System (`src/bridge/`)

Bidirectional communication layer for IDE extensions (VS Code, JetBrains). Uses JWT authentication. `bridgeMain.ts` orchestrates the bridge; `replBridge.ts` handles REPL sessions.

### Multi-Agent Systems

- **Coordinator** (`src/coordinator/`): orchestrates parallel agent work
- **Teams** (`TeamCreateTool`): group agents for coordinated tasks
- **Skills** (`src/skills/`): reusable workflow definitions executed via `SkillTool`
- **Plugins** (`src/plugins/`): dynamically loaded at runtime

### Feature Flags (Bun Build-Time)

Dead code is stripped via `bun:bundle` feature flags. Key flags:

```
PROACTIVE         – proactive/background mode
KAIROS            – Kairos assistant mode
BRIDGE_MODE       – IDE extension bridge
DAEMON            – daemon mode
VOICE_MODE        – voice input
AGENT_TRIGGERS    – agent trigger system
COORDINATOR_MODE  – multi-agent coordinator
MONITOR_TOOL      – monitoring tool
```

### Permission System

Centralized in `src/hooks/toolPermission/`. Every tool invocation passes through permission checks. Permission modes: `default`, `plan`, `bypassPermissions`, `auto`.

### Context Collection

`src/context.ts` collects system/user context (git status, environment, CLAUDE.md files) once per session and caches it. `src/utils/claudemd.ts` handles CLAUDE.md discovery and memory file injection.
