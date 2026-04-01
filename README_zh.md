# Claude Code — 架构学习笔记

<p align="center">
  <a href="./README.md"><img src="https://img.shields.io/badge/Docs-English-blue?style=for-the-badge" alt="English" /></a>
</p>

---

## 这是什么

个人学习笔记，通过阅读 Claude Code 源码来理解一个生产级 Agentic CLI 的架构设计——它如何串联工具、管理权限、流式处理 LLM 响应、以及协调多 Agent 工作流。

Claude Code 是 Anthropic 官方发布的 AI 辅助软件工程命令行工具，约 1,900 个文件、51 万余行 TypeScript 代码。

> **版权声明：** Claude Code 原始源代码归 Anthropic 所有。本仓库与 Anthropic 无任何关联，仅用于个人学习和技术架构分析。

---

## 技术栈

| 类别 | 技术 |
|---|---|
| 运行时 | [Bun](https://bun.sh) |
| 语言 | TypeScript（严格模式） |
| 终端 UI | [React](https://react.dev) + [Ink](https://github.com/vadimdemedes/ink) |
| CLI 解析 | [Commander.js](https://github.com/tj/commander.js) |
| Schema 验证 | [Zod v4](https://zod.dev) |
| 代码搜索 | [ripgrep](https://github.com/BurntSushi/ripgrep) |
| 协议 | MCP SDK、LSP |
| API 客户端 | [Anthropic SDK](https://docs.anthropic.com) |
| 遥测 | OpenTelemetry + gRPC（懒加载） |
| Feature Flag | GrowthBook |
| 认证 | OAuth 2.0、JWT、macOS Keychain |
| 代码检查 | Biome |

---

## 架构解析

### 入口与启动优化

`src/main.tsx` 是 CLI 入口。Commander.js 负责参数解析，React/Ink 负责渲染终端 UI。启动时有一个关键优化：MDM 配置读取、Keychain 预取、API 预连接**并行触发**，在重模块加载完成之前就已完成，有效降低冷启动延迟。

### 核心查询流水线

```
用户输入
  → QueryEngine（src/QueryEngine.ts）
    → 流式请求 Anthropic API
    → 检测 tool_use 块
    → 分发到对应工具处理器（src/tools/）
    → 循环直到无更多工具调用
  → 通过 Ink 组件渲染输出
```

`src/QueryEngine.ts`（约 4.6 万行）是整个系统的核心，处理流式响应、工具调用循环、思考模式（thinking mode）、重试逻辑和 token 计数。

### 工具系统（`src/tools/`）

每个工具都是独立模块，包含三要素：Zod 输入 Schema、权限声明、`call()` 实现。工具注册表在 `src/tools.ts`。

| 工具 | 用途 |
|---|---|
| `BashTool` | Shell 命令执行 |
| `FileReadTool` | 读取文件、图片、PDF、Jupyter Notebook |
| `FileWriteTool` / `FileEditTool` | 写入文件 / 局部编辑 |
| `GlobTool` / `GrepTool` | 文件搜索（ripgrep 支持） |
| `AgentTool` | 启动子 Agent |
| `SkillTool` | 执行可复用的 Skill 工作流 |
| `MCPTool` | 调用 MCP 服务工具 |
| `LSPTool` | 语言服务器协议集成 |
| `TaskCreateTool` / `TaskUpdateTool` | 任务生命周期管理 |
| `SendMessageTool` | Agent 间消息传递 |
| `EnterPlanModeTool` / `ExitPlanModeTool` | 计划模式切换 |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git Worktree 隔离 |
| `CronCreateTool` | 定时触发器 |
| `RemoteTriggerTool` | 远程执行 |

### 命令系统（`src/commands/`）

用户可调用的 `/斜杠命令`，约 50 个，统一注册在 `src/commands.ts`。命令通过 Bun 的 `bun:bundle` Feature Flag 实现**编译期死代码消除**——未启用的功能在打包时完全从产物中剔除。

```typescript
// 示例：仅在 VOICE_MODE flag 开启时才打包语音命令
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

主要 Flag：`PROACTIVE`、`KAIROS`、`BRIDGE_MODE`、`DAEMON`、`VOICE_MODE`、`AGENT_TRIGGERS`、`COORDINATOR_MODE`。

### 服务层（`src/services/`）

| 服务 | 用途 |
|---|---|
| `api/` | Anthropic API 客户端、Files API、重试逻辑、错误类型 |
| `mcp/` | MCP 服务器连接、认证、Channel 权限 |
| `oauth/` | OAuth 2.0 认证流程 |
| `lsp/` | 语言服务器协议管理 |
| `analytics/` | GrowthBook Feature Flag + 事件上报 |
| `compact/` | 对话上下文压缩 |
| `plugins/` | 插件加载器 |
| `policyLimits/` | 组织策略限制执行 |
| `extractMemories/` | 自动记忆提取 |

### 权限系统（`src/hooks/toolPermission/`）

每次工具调用都经过集中式权限检查。权限模式：`default`、`plan`、`bypassPermissions`、`auto`。系统根据配置模式自动放行或弹出用户确认。

### Bridge 系统（`src/bridge/`）

连接 IDE 扩展（VS Code、JetBrains）与 CLI 的双向通信层，使用 JWT 认证。`bridgeMain.ts` 负责主循环，`replBridge.ts` 处理交互式 REPL 会话。

### 多 Agent 架构

- **Coordinator**（`src/coordinator/`）：调度并行 Agent
- **Teams**（`TeamCreateTool`）：将多个 Agent 编组协同工作
- **Skills**（`src/skills/`）：可复用的命名工作流，支持用户自定义扩展
- **Sub-agents**（`AgentTool`）：启动拥有独立上下文的子 Agent 实例

---

## 目录结构

```
src/
├── main.tsx              # CLI 入口（Commander.js + Ink 渲染器）
├── QueryEngine.ts        # 核心 LLM 流式处理 + 工具调用循环
├── Tool.ts               # 工具类型定义与权限模型
├── commands.ts           # 斜杠命令注册表
├── tools.ts              # 工具注册表
├── context.ts            # 系统/用户上下文收集（每会话缓存）
├── cost-tracker.ts       # Token 计数与费用追踪
│
├── commands/             # ~50 个斜杠命令实现
├── tools/                # ~40 个 Agent 工具实现
├── components/           # ~140 个 Ink 终端 UI 组件
├── services/             # 外部集成（API、MCP、OAuth、LSP…）
├── bridge/               # IDE 扩展 Bridge
├── coordinator/          # 多 Agent 协调
├── skills/               # 可复用 Skill 工作流
├── plugins/              # 插件系统
├── hooks/                # React Hooks（含权限系统）
├── screens/              # 全屏 UI（Doctor、REPL、Resume）
├── state/                # 应用状态（AppState.tsx）
├── types/                # TypeScript 类型定义
├── utils/                # 工具函数（config、auth、git、shell…）
├── schemas/              # Zod 配置 Schema
├── entrypoints/          # 初始化逻辑与 Agent SDK 类型
├── memdir/               # 持久化记忆目录
├── tasks/                # 任务管理
├── migrations/           # 配置迁移
├── vim/                  # Vim 模式
├── voice/                # 语音输入
├── remote/               # 远程会话
├── server/               # Server 模式
└── keybindings/          # 快捷键配置
```
