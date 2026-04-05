# 08 - Guia de Migração para Go + GORM

## Visão Geral

Este documento consolida o mapeamento do OpenClaw (TypeScript) para uma reimplementação em Go com GORM. Foca nos componentes core: agent loop, tools, memória, reasoning, e configuração.

## Arquitetura Proposta em Go

```
openclaw-go/
├── cmd/
│   └── openclaw/
│       └── main.go              # Entry point CLI
├── internal/
│   ├── agent/                   # Agent loop (Think → Act → Observe)
│   │   ├── loop.go              # Main agent loop
│   │   ├── attempt.go           # Single execution attempt
│   │   ├── stream.go            # Stream event processing
│   │   ├── compaction.go        # Context compaction
│   │   ├── lane.go              # Concurrency lanes
│   │   └── types.go             # Agent types
│   ├── tool/                    # Tool system
│   │   ├── registry.go          # Tool registry
│   │   ├── executor.go          # Tool execution
│   │   ├── policy.go            # Tool policies
│   │   ├── schema.go            # Tool JSON Schema
│   │   ├── builtin/             # Built-in tools
│   │   │   ├── exec.go
│   │   │   ├── web_fetch.go
│   │   │   ├── web_search.go
│   │   │   ├── message.go
│   │   │   └── cron.go
│   │   └── mcp/                 # MCP integration
│   │       ├── client.go
│   │       ├── transport.go
│   │       └── materialize.go
│   ├── memory/                  # Memory system
│   │   ├── short_term.go        # Context window (in-memory)
│   │   ├── medium_term.go       # Session persistence (GORM)
│   │   ├── long_term.go         # Vector store / embeddings
│   │   ├── context_engine.go    # Context assembly
│   │   ├── compaction.go        # Context compaction
│   │   └── types.go
│   ├── inference/               # LLM inference
│   │   ├── provider.go          # Provider interface
│   │   ├── prompt.go            # Prompt assembly
│   │   ├── stream.go            # Stream processing
│   │   ├── cache.go             # Prompt cache
│   │   ├── thinking.go          # Extended thinking
│   │   └── providers/           # Provider implementations
│   │       ├── anthropic.go
│   │       ├── openai.go
│   │       └── ollama.go
│   ├── skill/                   # Skills system
│   │   ├── loader.go            # Skill discovery & loading
│   │   ├── frontmatter.go       # SKILL.md parsing
│   │   ├── registry.go          # Skill registry
│   │   └── types.go
│   ├── hook/                    # Hooks system
│   │   ├── registry.go          # Hook registry
│   │   ├── loader.go            # Hook loading
│   │   ├── dispatcher.go        # Event dispatch
│   │   └── types.go
│   ├── cron/                    # Cron system
│   │   ├── service.go           # Cron service
│   │   ├── scheduler.go         # Schedule computation
│   │   ├── executor.go          # Isolated execution
│   │   ├── delivery.go          # Result delivery
│   │   └── types.go
│   ├── task/                    # Task registry
│   │   ├── registry.go          # Task CRUD
│   │   ├── executor.go          # Task execution
│   │   ├── flow.go              # Task flows (parent-child)
│   │   └── types.go
│   ├── channel/                 # Channel system
│   │   ├── adapter.go           # Channel adapter interface
│   │   ├── router.go            # Message routing
│   │   ├── session_key.go       # Session key building
│   │   ├── binding.go           # Route bindings
│   │   └── adapters/            # Channel implementations
│   │       ├── telegram.go
│   │       ├── discord.go
│   │       └── webhook.go
│   ├── reply/                   # Auto-reply pipeline
│   │   ├── pipeline.go          # Main reply pipeline
│   │   ├── inbound.go           # Inbound processing
│   │   ├── command.go           # Command detection & dispatch
│   │   ├── directive.go         # Directive parsing
│   │   ├── delivery.go          # Reply delivery
│   │   └── chunk.go             # Response chunking
│   ├── config/                  # Configuration
│   │   ├── config.go            # Config loading
│   │   ├── schema.go            # Config validation
│   │   ├── defaults.go          # Default values
│   │   └── types.go
│   ├── plugin/                  # Plugin system
│   │   ├── registry.go          # Plugin registry
│   │   ├── loader.go            # Plugin loading (Go plugins or gRPC)
│   │   ├── manifest.go          # Manifest parsing
│   │   └── types.go
│   └── gateway/                 # HTTP gateway
│       ├── server.go            # HTTP server
│       ├── router.go            # Request routing
│       ├── handlers/            # HTTP handlers
│       └── protocol/            # Wire protocol types
├── pkg/
│   ├── models/                  # GORM models
│   │   ├── agent.go
│   │   ├── session.go
│   │   ├── message.go
│   │   ├── memory.go
│   │   ├── tool.go
│   │   ├── task.go
│   │   ├── cron.go
│   │   ├── plugin.go
│   │   └── migrate.go           # Auto-migration
│   └── types/                   # Shared types
│       ├── message.go
│       ├── stream.go
│       └── common.go
├── go.mod
├── go.sum
└── Makefile
```

## Schema Completo GORM

```go
package models

import (
    "time"
    "gorm.io/datatypes"
    "gorm.io/gorm"
)

// ============================================
// AGENT & RUNS
// ============================================

type Agent struct {
    gorm.Model
    Name         string `gorm:"not null"`
    SystemPrompt string
    ModelID      string `gorm:"not null;default:'sonnet-4.6'"`
    Config       datatypes.JSON
    Enabled      bool `gorm:"default:true"`
}

type AgentRun struct {
    gorm.Model
    AgentID      uint      `gorm:"index;not null"`
    Agent        Agent     `gorm:"foreignKey:AgentID"`
    SessionID    uint      `gorm:"index;not null"`
    Session      Session   `gorm:"foreignKey:SessionID"`
    Status       string    `gorm:"not null;default:'running'"` // running, completed, aborted, timed_out
    InputTokens  int       `gorm:"default:0"`
    OutputTokens int       `gorm:"default:0"`
    ThinkingLevel string   `gorm:"default:'off'"`
    Provider     string
    Model        string
    CacheHit     bool      `gorm:"default:false"`
    LatencyMs    int
    StopReason   string
    Error        *string
    StartedAt    time.Time `gorm:"not null"`
    CompletedAt  *time.Time
}

// ============================================
// SESSIONS & MESSAGES
// ============================================

type Session struct {
    gorm.Model
    SessionKey   string `gorm:"uniqueIndex;not null"`
    AgentID      uint   `gorm:"index;not null"`
    Agent        Agent  `gorm:"foreignKey:AgentID"`
    ChannelID    string
    PeerID       string
    DmScope      string `gorm:"default:'main'"`
    Metadata     datatypes.JSON
    LastActiveAt time.Time
}

type SessionMessage struct {
    gorm.Model
    SessionID  uint    `gorm:"index;not null"`
    Session    Session `gorm:"foreignKey:SessionID"`
    Role       string  `gorm:"not null"` // system, user, assistant, tool
    Content    string  `gorm:"type:text"`
    ToolCallID *string
    ToolName   *string
    Sequence   int     `gorm:"not null"`
    TokenCount int     `gorm:"default:0"`
}

// ============================================
// MEMORY (LONG-TERM)
// ============================================

type Memory struct {
    gorm.Model
    AgentID        uint    `gorm:"index;not null"`
    Agent          Agent   `gorm:"foreignKey:AgentID"`
    Content        string  `gorm:"type:text;not null"`
    Category       string  `gorm:"not null"` // fact, preference, instruction, context
    Embedding      []byte  // vector embedding
    SourceSessionID *uint
    Relevance      float64 `gorm:"default:1.0"`
    AccessCount    int     `gorm:"default:0"`
    LastAccessedAt *time.Time
}

// ============================================
// TOOLS
// ============================================

type ToolDefinition struct {
    gorm.Model
    Name        string `gorm:"uniqueIndex;not null"`
    Description string
    Schema      datatypes.JSON // JSON Schema dos parâmetros
    Category    string `gorm:"not null"` // builtin, plugin, mcp
    OwnerOnly   bool   `gorm:"default:false"`
    PluginID    *uint
    Enabled     bool   `gorm:"default:true"`
}

type ToolExecution struct {
    gorm.Model
    RunID      uint   `gorm:"index;not null"`
    Run        AgentRun `gorm:"foreignKey:RunID"`
    ToolName   string `gorm:"not null"`
    Arguments  datatypes.JSON
    Result     datatypes.JSON
    IsError    bool   `gorm:"default:false"`
    DurationMs int
    Sequence   int
}

type ToolPolicy struct {
    gorm.Model
    Name      string `gorm:"uniqueIndex;not null"`
    Profile   string // minimal, coding, messaging, full
    AllowList datatypes.JSON
    DenyList  datatypes.JSON
    Priority  int    `gorm:"default:0"`
}

// ============================================
// TASKS & CRON
// ============================================

type Task struct {
    gorm.Model
    ExternalID   string  `gorm:"uniqueIndex"`
    Runtime      string  `gorm:"not null"` // subagent, acp, cli, cron
    ScopeKind    string  `gorm:"not null"` // session, system
    ScopeID      string
    Status       string  `gorm:"not null;default:'queued'"` // queued, running, succeeded, failed, timed_out, cancelled
    NotifyPolicy string  `gorm:"default:'done_only'"`
    ParentFlowID *uint
    ParentFlow   *TaskFlow `gorm:"foreignKey:ParentFlowID"`
    Input        datatypes.JSON
    Output       datatypes.JSON
    Error        *string
    QueuedAt     time.Time  `gorm:"not null"`
    StartedAt    *time.Time
    CompletedAt  *time.Time
}

type TaskFlow struct {
    gorm.Model
    Name     string
    Status   string `gorm:"not null;default:'running'"`
    Tasks    []Task `gorm:"foreignKey:ParentFlowID"`
    Metadata datatypes.JSON
}

type CronJob struct {
    gorm.Model
    Name           string `gorm:"not null"`
    AgentID        uint   `gorm:"index;not null"`
    Agent          Agent  `gorm:"foreignKey:AgentID"`
    ScheduleType   string `gorm:"not null"` // at, every, cron
    ScheduleExpr   string `gorm:"not null"`
    SessionTarget  string `gorm:"default:'isolated'"`
    Payload        datatypes.JSON
    DeliveryType   string // channel, dm, webhook
    DeliveryTarget string
    Enabled        bool       `gorm:"default:true"`
    LastRunAt      *time.Time
    NextRunAt      *time.Time
    RunCount       int        `gorm:"default:0"`
    ErrorCount     int        `gorm:"default:0"`
    LastError      *string
}

// ============================================
// PLUGINS & CONFIG
// ============================================

type Plugin struct {
    gorm.Model
    PluginID     string `gorm:"uniqueIndex;not null"`
    Name         string `gorm:"not null"`
    Version      string
    Description  string
    Capabilities datatypes.JSON // []string
    Source       string `gorm:"not null"` // builtin, plugin, external
    Enabled      bool   `gorm:"default:true"`
    Config       datatypes.JSON
}

type Secret struct {
    gorm.Model
    Key            string `gorm:"uniqueIndex;not null"`
    EncryptedValue []byte `gorm:"not null"`
    Source         string // config, env, store
}

type Skill struct {
    gorm.Model
    Name        string `gorm:"uniqueIndex;not null"`
    Description string
    Content     string `gorm:"type:text"`
    Source      string // bundled, workspace, plugin
    Enabled     bool   `gorm:"default:true"`
}

type Channel struct {
    gorm.Model
    ChannelID string `gorm:"uniqueIndex;not null"`
    Type      string `gorm:"not null"` // telegram, discord, slack, etc.
    Name      string
    Config    datatypes.JSON
    Enabled   bool `gorm:"default:true"`
}

type RouteBinding struct {
    gorm.Model
    AgentID    uint   `gorm:"index;not null"`
    ChannelID  string `gorm:"not null"`
    AccountID  string
    PeerID     *string
    GuildID    *string
    TeamID     *string
    SessionKey string `gorm:"not null"`
    Priority   int    `gorm:"default:0"`
}

// ============================================
// MIGRATION
// ============================================

func AutoMigrate(db *gorm.DB) error {
    return db.AutoMigrate(
        &Agent{},
        &AgentRun{},
        &Session{},
        &SessionMessage{},
        &Memory{},
        &ToolDefinition{},
        &ToolExecution{},
        &ToolPolicy{},
        &Task{},
        &TaskFlow{},
        &CronJob{},
        &Plugin{},
        &Secret{},
        &Skill{},
        &Channel{},
        &RouteBinding{},
    )
}
```

## Interfaces Core em Go

```go
package agent

// AgentLoop is the main agent execution loop
type AgentLoop interface {
    Run(ctx context.Context, req RunRequest) (*RunResult, error)
    Abort(runID string) error
    IsActive(sessionID string) bool
}

// RunRequest contains everything needed for an agent run
type RunRequest struct {
    AgentID     uint
    SessionID   uint
    Message     string
    Images      [][]byte
    Thinking    ThinkingLevel
    ModelOverride *string
    ToolPolicy  *ToolPolicy
}

// RunResult contains the output of an agent run
type RunResult struct {
    Payloads      []Payload
    TokensIn      int
    TokensOut     int
    ToolCalls     []ToolCallResult
    StopReason    string
    Duration      time.Duration
}
```

```go
package inference

// LLMProvider is the interface all LLM providers implement
type LLMProvider interface {
    ID() string
    Name() string
    CreateCompletion(ctx context.Context, req CompletionRequest) (*CompletionStream, error)
    ListModels(ctx context.Context) ([]Model, error)
    GetCapabilities(modelID string) ModelCapabilities
}
```

```go
package tool

// Tool is the interface all tools implement
type Tool interface {
    Name() string
    Description() string
    Schema() json.RawMessage
    Execute(ctx context.Context, params map[string]any) (*ToolResult, error)
}

// ToolRegistry manages available tools
type ToolRegistry interface {
    Register(tool Tool) error
    Get(name string) (Tool, bool)
    List(policy *ToolPolicy) []Tool
    Execute(ctx context.Context, name string, params map[string]any) (*ToolResult, error)
}
```

```go
package memory

// MemoryStore manages all memory layers
type MemoryStore interface {
    // Short-term (context window)
    GetContextMessages(sessionID uint, budget int) ([]SessionMessage, error)
    CompactContext(sessionID uint, messages []SessionMessage) ([]SessionMessage, error)
    
    // Medium-term (session persistence)
    AppendMessage(sessionID uint, msg SessionMessage) error
    LoadSession(sessionKey string) (*Session, error)
    
    // Long-term (vector store)
    SaveMemory(entry Memory) error
    SearchMemory(ctx context.Context, agentID uint, query string, topK int) ([]Memory, error)
}
```

## Dependências Go Sugeridas

```go
// go.mod
module github.com/your-org/openclaw-go

go 1.22

require (
    // ORM
    gorm.io/gorm v1.25.x
    gorm.io/driver/postgres v1.5.x    // ou sqlite
    gorm.io/datatypes v1.2.x
    
    // HTTP
    github.com/gin-gonic/gin v1.9.x   // ou chi, fiber
    
    // LLM APIs
    github.com/anthropics/anthropic-sdk-go v1.x
    github.com/sashabaranov/go-openai v1.x
    
    // Vector store (para LTM)
    github.com/pgvector/pgvector-go v0.2.x  // se usar pgvector
    
    // Cron
    github.com/robfig/cron/v3 v3.0.x
    
    // Config
    github.com/spf13/viper v1.18.x
    
    // CLI
    github.com/spf13/cobra v1.8.x
    
    // Streaming
    github.com/r3labs/sse/v2 v2.10.x
    
    // MCP
    // github.com/anthropics/mcp-go (se disponível)
    
    // Utils
    github.com/google/uuid v1.6.x
    go.uber.org/zap v1.27.x           // logging
)
```

## Ordem de Implementação Sugerida

### Fase 1: Fundação
1. **Models GORM** (`pkg/models/`) - Todas as tabelas
2. **Config** (`internal/config/`) - Loading e validação
3. **Gateway HTTP** (`internal/gateway/`) - Server básico

### Fase 2: Inference
4. **Provider interface** (`internal/inference/provider.go`)
5. **Anthropic provider** (`internal/inference/providers/anthropic.go`)
6. **Stream processing** (`internal/inference/stream.go`)
7. **Prompt assembly** (`internal/inference/prompt.go`)

### Fase 3: Agent Loop
8. **Agent loop** (`internal/agent/loop.go`)
9. **Attempt** (`internal/agent/attempt.go`)
10. **Context compaction** (`internal/agent/compaction.go`)

### Fase 4: Tools
11. **Tool registry** (`internal/tool/registry.go`)
12. **Tool executor** (`internal/tool/executor.go`)
13. **Built-in tools** (`internal/tool/builtin/`)
14. **Tool policies** (`internal/tool/policy.go`)

### Fase 5: Memory
15. **Session persistence** (`internal/memory/medium_term.go`)
16. **Context engine** (`internal/memory/context_engine.go`)
17. **Vector store** (`internal/memory/long_term.go`)

### Fase 6: Messaging
18. **Channel adapter** (`internal/channel/adapter.go`)
19. **Routing** (`internal/channel/router.go`)
20. **Auto-reply pipeline** (`internal/reply/pipeline.go`)

### Fase 7: Automação
21. **Skills** (`internal/skill/`)
22. **Hooks** (`internal/hook/`)
23. **Cron** (`internal/cron/`)
24. **Tasks** (`internal/task/`)

## Diferenças Chave TypeScript → Go

| Aspecto | TypeScript (OpenClaw) | Go (Proposta) |
|---------|----------------------|---------------|
| **Plugins** | Dynamic import() | Go plugins ou gRPC |
| **Streaming** | AsyncIterator/EventEmitter | Channels (`chan StreamEvent`) |
| **Schema validation** | Zod | struct tags + validator |
| **Concurrency** | Promises/async-await | Goroutines + channels |
| **Session store** | JSONL files | GORM/PostgreSQL |
| **Vector store** | LanceDB plugin | pgvector ou Qdrant |
| **Config** | JSON5 file | Viper (YAML/JSON/env) |
| **CLI** | Custom argparse | Cobra |
| **HTTP** | Custom server | Gin/Chi |
| **Tool schemas** | JSON Schema (runtime) | JSON Schema (runtime) |
| **Hooks** | Dynamic module import | Interface + registry |
| **MCP** | SDK with stdio/SSE | Custom client |

## Considerações Importantes

1. **O agent loop é o mais crítico** - Comece por ele. O ciclo Think→Act→Observe é o coração do sistema.

2. **Streaming é essencial** - Use Go channels para eventos do stream. Cada provider retorna `chan StreamEvent`.

3. **Memória de longo prazo é um plugin** - No Go, pode ser integrado diretamente com pgvector no PostgreSQL via GORM.

4. **Tool system precisa ser extensível** - Use interfaces Go. Cada tool implementa `Tool` interface.

5. **Context compaction é complexo** - Quando o contexto excede o limite, você precisa resumir mensagens antigas. Isso requer uma chamada LLM separada.

6. **Prompt cache stability** - Se usar Anthropic, mantenha o prefixo do prompt byte-idêntico entre turns.

7. **Session isolation** - Use GORM transactions para garantir isolamento entre sessões concorrentes.

8. **MCP** - O Model Context Protocol permite integrar tools externas. Implemente o cliente MCP para stdio e HTTP.
