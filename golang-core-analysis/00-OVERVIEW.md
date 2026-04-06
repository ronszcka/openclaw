# OpenClaw Core - Análise para Migração Golang

## Visão Geral

O OpenClaw é um assistente pessoal de IA multi-canal construído em TypeScript/Node.js. Este documento mapeia os componentes **core** do sistema para viabilizar uma reimplementação em Go + GORM.

## Arquitetura Core

O mecanismo central do OpenClaw consiste em 7 subsistemas principais:

```
┌─────────────────────────────────────────────────────────────┐
│                      GATEWAY (HTTP Server)                   │
│              src/gateway/ - Control Plane HTTP               │
├──────────┬──────────┬──────────┬──────────┬─────────────────┤
│  AGENT   │  TOOL    │ MEMORY   │ REASONING│   SKILLS &      │
│  LOOP    │  SYSTEM  │ SYSTEM   │ /INFERE  │   FLOWS         │
│          │          │          │ NCE      │                 │
│ src/     │ src/     │ src/     │ src/     │ src/agents/     │
│ agents/  │ agents/  │ sessions/│ agents/  │   skills/       │
│          │ tools/   │ config/  │ pi-*     │ src/flows/      │
│          │          │ sessions/│          │ src/hooks/      │
│          │ src/mcp/ │          │ src/     │ src/cron/       │
│          │          │ context- │ plugin-  │ src/tasks/      │
│          │          │ engine/  │ sdk/     │                 │
├──────────┴──────────┴──────────┴──────────┴─────────────────┤
│               CONFIG & PLUGIN SYSTEM                         │
│      src/config/ | src/plugins/ | src/plugin-sdk/            │
├─────────────────────────────────────────────────────────────┤
│               CHANNELS & ROUTING                             │
│      src/channels/ | src/routing/ | src/auto-reply/          │
└─────────────────────────────────────────────────────────────┘
```

## Documentos por Feature

| # | Documento | Feature Core |
|---|-----------|-------------|
| 01 | [01-AGENT-LOOP.md](./01-AGENT-LOOP.md) | Loop Agêntico (Think → Act → Observe) |
| 02 | [02-TOOL-SYSTEM.md](./02-TOOL-SYSTEM.md) | Sistema de Tools (definição, registro, execução, MCP) |
| 03 | [03-MEMORY-SYSTEM.md](./03-MEMORY-SYSTEM.md) | Memória de curto, médio e longo prazo |
| 04 | [04-REASONING-INFERENCE.md](./04-REASONING-INFERENCE.md) | Raciocínio, inferência LLM, prompt assembly |
| 05 | [05-SKILLS-FLOWS.md](./05-SKILLS-FLOWS.md) | Skills, Hooks, Flows, Cron, Tasks |
| 06 | [06-CONFIG-PLUGINS.md](./06-CONFIG-PLUGINS.md) | Configuração, plugins, bootstrap |
| 07 | [07-ROUTING-CHANNELS.md](./07-ROUTING-CHANNELS.md) | Roteamento de mensagens, canais, auto-reply |
| 08 | [08-GOLANG-MIGRATION-GUIDE.md](./08-GOLANG-MIGRATION-GUIDE.md) | Guia de migração para Go + GORM |
| 09 | [09-CLOUD-RUN-ARCHITECTURE.md](./09-CLOUD-RUN-ARCHITECTURE.md) | Arquitetura Cloud Run (wake → execute → sleep) |

## Stack Tecnológico Original

- **Linguagem**: TypeScript (ESM)
- **Runtime**: Node.js 22+ / Bun
- **Testes**: Vitest
- **Build**: tsdown
- **Package Manager**: pnpm (workspaces)
- **Formato/Lint**: oxfmt, oxlint
- **Schemas**: Zod
- **Persistência**: SQLite (via plugin), filesystem JSON/JSONL

## Princípios de Design

1. **Plugin-first**: Quase tudo é extensível via plugins
2. **Channel-agnostic**: Core não sabe detalhes de canais específicos
3. **Provider-agnostic**: Core não sabe detalhes de providers LLM específicos
4. **Event-driven**: Hooks para extensibilidade em pontos-chave
5. **Session-isolated**: Cada conversa é uma sessão isolada
6. **Streaming-native**: Respostas são streamed por padrão

## Mapeamento de Diretórios Core

```
src/
├── agents/                    # Agent runtime & orchestration (CORE)
│   ├── pi-embedded-runner/    # Main agent execution engine
│   ├── skills/                # Skill loading & management
│   ├── tools/                 # Built-in tool implementations
│   ├── sandbox/               # Sandboxed execution
│   ├── command/               # Agent CLI command
│   └── schema/                # Agent config schemas
├── auto-reply/                # Message handling & command dispatch
│   └── reply/                 # Core reply execution pipeline
├── channels/                  # Channel abstraction layer
│   └── plugins/               # Channel plugin contracts
├── config/                    # Configuration loading & schemas
│   └── sessions/              # Session store config
├── context-engine/            # Context building for agent turns
├── cron/                      # Scheduled job execution
│   ├── service/               # Cron service state/ops
│   └── isolated-agent/        # Isolated agent execution
├── flows/                     # Interactive setup wizards
├── gateway/                   # HTTP control plane
│   ├── protocol/              # Wire protocol schemas
│   └── server-methods/        # HTTP method handlers
├── hooks/                     # Event-driven extensibility
│   └── bundled/               # Built-in hooks
├── mcp/                       # Model Context Protocol integration
├── memory-host-sdk/           # Memory system SDK
├── plugin-sdk/                # Public plugin contract
├── plugins/                   # Plugin discovery, registry, loader
│   └── contracts/             # Plugin-to-core contracts
├── routing/                   # Message routing & session keys
├── sessions/                  # Session lifecycle
├── tasks/                     # Task registry & execution
└── shared/                    # Shared utilities
```
