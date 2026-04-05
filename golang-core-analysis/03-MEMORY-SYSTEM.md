# 03 - Memory System (Sistema de Memória)

## Resumo

O OpenClaw implementa memória em 3 camadas: **curto prazo** (contexto da conversa atual), **médio prazo** (sessões persistidas) e **longo prazo** (memory plugin com vector store). O sistema de contexto monta o prompt completo para cada turn do agente.

## Camadas de Memória

```
┌─────────────────────────────────────────────────────┐
│           LONGO PRAZO (Long-term Memory)             │
│  extensions/memory-core/ + extensions/memory-lancedb/│
│  Vector store, embeddings, semantic search           │
│  Persiste entre sessões, aprende do usuário          │
├─────────────────────────────────────────────────────┤
│           MÉDIO PRAZO (Session Persistence)           │
│  src/sessions/ + src/config/sessions/                │
│  Sessões persistidas em disco (JSONL)                │
│  Mantém histórico de conversa entre restarts         │
├─────────────────────────────────────────────────────┤
│           CURTO PRAZO (Context Window)               │
│  src/context-engine/ + pi-embedded-runner/           │
│  Mensagens no contexto atual do LLM                  │
│  Compaction quando excede limite                     │
└─────────────────────────────────────────────────────┘
```

## 1. Memória de Curto Prazo (Context Window)

A memória de curto prazo é o contexto atual da conversa no LLM. Inclui system prompt, mensagens anteriores, tool results, e a mensagem atual.

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `src/context-engine/context-engine.ts` | **Engine principal** de construção de contexto. Monta o prompt completo para cada turn: system prompt + bootstrap files + skills + tools + histórico |
| `src/context-engine/context-budget.ts` | Gerenciamento de orçamento de tokens. Calcula quanto espaço disponível para cada componente |
| `src/context-engine/context-builder.ts` | Builder pattern para montar o contexto incrementalmente |
| `src/context-engine/context-types.ts` | Tipos: `ContextWindow`, `ContextBudget`, `ContextComponent` |
| `src/context-engine/context-compaction.ts` | **Compaction** - quando o contexto excede o limite, resume/trunca mensagens antigas preservando as recentes |
| `src/context-engine/context-pruning.ts` | Poda de contexto: remove tool results grandes, imagens antigas |
| `src/context-engine/bootstrap-files.ts` | Carrega bootstrap files (AGENTS.md, SKILL.md) para o contexto |

### Compaction (Gerenciamento de Overflow)

Quando o contexto excede o limite de tokens:

1. **Detecção**: O stream retorna evento `compaction` quando o provider sinaliza overflow
2. **Resume**: Mensagens antigas são resumidas (summarized) em texto mais curto
3. **Poda**: Tool results grandes são removidos, imagens antigas descartadas
4. **Retry**: O attempt é retentado com contexto compactado e grace period estendido

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-runner/run/attempt.ts` | Lógica de compaction retry no attempt |
| `src/agents/pi-embedded-runner/prompt-cache-retention.ts` | Preserva cache de prompt durante compaction |
| `src/agents/system-prompt-cache-boundary.ts` | Boundary de cache do system prompt |
| `src/agents/pi-embedded-runner/session-manager-cache.ts` | Cache de gerenciador de sessão |

## 2. Memória de Médio Prazo (Session Persistence)

Sessões são persistidas em disco para manter contexto entre restarts do gateway.

### Arquivos de Sessão

| Arquivo | Propósito |
|---------|-----------|
| `src/sessions/session-store.ts` | **Store abstrato** de sessões. Interface para load/save/delete |
| `src/sessions/session-store-jsonl.ts` | Implementação em **JSONL** (JSON Lines). Cada mensagem é uma linha. Append-only para performance |
| `src/sessions/session-store-sqlite.ts` | Implementação alternativa em **SQLite** |
| `src/sessions/session-lifecycle.ts` | Lifecycle da sessão: create, load, save, archive, delete |
| `src/sessions/session-types.ts` | Tipos: `Session`, `SessionState`, `SessionMetadata` |
| `src/sessions/session-id.ts` | Geração e validação de IDs de sessão |
| `src/sessions/session-compaction.ts` | Compaction de sessões antigas |
| `src/sessions/session-transcript.ts` | Transcrição completa da sessão |

### Configuração de Sessões

| Arquivo | Propósito |
|---------|-----------|
| `src/config/sessions/session-config.ts` | Config de sessão: dmScope, isolation, TTL |
| `src/config/sessions/session-store-config.ts` | Config do store: tipo (jsonl, sqlite), path, limites |
| `src/config/sessions/session-routing-config.ts` | Config de roteamento de sessões |

### Localização em Disco

Sessões ficam em `~/.openclaw/agents/<agentId>/sessions/`:
- `*.jsonl` - Logs de sessão (append-only)
- `metadata.json` - Metadados da sessão

### Roteamento de Sessão

| Arquivo | Propósito |
|---------|-----------|
| `src/routing/session-key.ts` | **Build de session key** a partir de agent/channel/peer. Funções: `buildAgentSessionKey()`, `buildAgentPeerSessionKey()`, `buildAgentMainSessionKey()` |
| `src/routing/resolve-route.ts` | Resolve qual sessão usar para uma mensagem inbound |
| `src/routing/bindings.ts` | Bindings agent→channel→account. Precedência: peer → guild → team → account → channel |

### Isolamento de Sessão (DM Scope)

```
session.dmScope options:
- "main"                    → Todos DMs compartilham uma sessão
- "per-peer"                → Isolado por sender (entre canais)
- "per-channel-peer"        → Isolado por canal + sender (recomendado)
- "per-account-channel-peer"→ Isolado por account + canal + sender
```

## 3. Memória de Longo Prazo (Learning & Vector Store)

A memória de longo prazo é implementada como plugins de extensão, não no core.

### Memory Host SDK

| Arquivo | Propósito |
|---------|-----------|
| `src/memory-host-sdk/memory-host.ts` | **SDK do host** - Interface que o core expõe para plugins de memória registrarem handlers |
| `src/memory-host-sdk/memory-types.ts` | Tipos: `MemoryEntry`, `MemoryQuery`, `MemorySearchResult`, `MemorySlot` |
| `src/memory-host-sdk/memory-store.ts` | Interface abstrata de store de memória |
| `src/memory-host-sdk/memory-hooks.ts` | Hooks para interceptar operações de memória |

### Memory Core Extension

| Arquivo | Propósito |
|---------|-----------|
| `extensions/memory-core/src/index.ts` | Plugin principal de memória. Registra handlers para save/query/delete |
| `extensions/memory-core/src/memory-service.ts` | Serviço de memória: CRUD de entries, categorização |
| `extensions/memory-core/src/memory-tool.ts` | Tool `memory_save` e `memory_search` disponibilizadas ao agente |
| `extensions/memory-core/src/memory-schema.ts` | Schema de entries de memória |
| `extensions/memory-core/openclaw.plugin.json` | Manifesto do plugin |

### Memory LanceDB Extension

| Arquivo | Propósito |
|---------|-----------|
| `extensions/memory-lancedb/src/index.ts` | Plugin de vector store usando LanceDB |
| `extensions/memory-lancedb/src/vector-store.ts` | Implementação de vector store: embeddings, similarity search |
| `extensions/memory-lancedb/src/embeddings.ts` | Geração de embeddings para memória semântica |
| `extensions/memory-lancedb/openclaw.plugin.json` | Manifesto do plugin |

### Como a Memória de Longo Prazo Funciona

```
1. Agente processa mensagem
2. Hook `session-memory` (src/hooks/bundled/session-memory/handler.ts)
   captura o transcript da conversa
3. Memory Core plugin decide o que memorizar
4. Embeddings são gerados
5. LanceDB armazena com vector embedding
6. Em futuras conversas, contexto é enriquecido com memórias relevantes
   via busca semântica (similarity search)
```

### Hook de Session Memory

| Arquivo | Propósito |
|---------|-----------|
| `src/hooks/bundled/session-memory/handler.ts` | **Hook bundled** que captura transcripts de sessão e alimenta o sistema de memória. Intercepta `session:end` events |

## 4. Context Engine (Montagem de Contexto)

O Context Engine monta o prompt completo para cada turn, combinando todas as camadas de memória.

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `src/context-engine/context-engine.ts` | Engine principal |
| `src/context-engine/context-budget.ts` | Orçamento de tokens por componente |
| `src/context-engine/context-builder.ts` | Builder incremental |
| `src/context-engine/context-types.ts` | Tipos compartilhados |
| `src/context-engine/bootstrap-files.ts` | Bootstrap files (AGENTS.md, etc.) |

### Composição do Contexto

```
Context Window = [
    System Prompt (fixo)
    + Bootstrap Files (AGENTS.md, etc.)
    + Skills Prompt (XML formatado)
    + Memory Context (memórias relevantes da LTM)
    + Session History (mensagens anteriores da sessão)
    + Tool Definitions (schema das tools disponíveis)
    + Current Message (mensagem do usuário)
]
```

### Orçamento de Tokens

```
Total Context Budget (ex: 128k tokens)
├── System Prompt:    ~2k tokens (fixo)
├── Bootstrap Files:  ~4k tokens (variável)
├── Skills:           ~30k tokens (max 150 skills)
├── Memory:           ~8k tokens (top-k relevantes)
├── Tools:            ~10k tokens (schema)
├── History:          ~60k tokens (flexível)
└── Current Message:  ~14k tokens (variável)
```

## Interações entre Arquivos

```
Mensagem Inbound
      │
      ▼
routing/session-key.ts ──→ Resolve session key
      │
      ▼
sessions/session-store.ts ──→ Load/create session
      │
      ▼
context-engine/context-engine.ts
      │
      ├──→ bootstrap-files.ts (AGENTS.md, etc.)
      ├──→ agents/skills/workspace.ts (skills prompt)
      ├──→ memory-host-sdk/ → extensions/memory-core/ (LTM search)
      ├──→ sessions/session-transcript.ts (STM history)
      └──→ agents/openclaw-tools.ts (tool definitions)
      │
      ▼
Contexto completo → pi-embedded-runner/run.ts
      │
      ▼
[Agent Loop executa]
      │
      ▼
sessions/session-store.ts ──→ Persiste mensagens (MTM)
hooks/bundled/session-memory/ ──→ Captura para LTM
```

## Mapeamento para Go + GORM

### Structs

```go
// Sessão
type Session struct {
    gorm.Model
    SessionKey  string `gorm:"uniqueIndex"`
    AgentID     uint
    ChannelID   string
    PeerID      string
    DmScope     string
    Metadata    datatypes.JSON
    LastActiveAt time.Time
}

// Mensagem de sessão (médio prazo)
type SessionMessage struct {
    gorm.Model
    SessionID  uint   `gorm:"index"`
    Role       string // system, user, assistant, tool
    Content    string
    ToolCallID *string
    Sequence   int
    TokenCount int
}

// Memória de longo prazo
type Memory struct {
    gorm.Model
    AgentID    uint     `gorm:"index"`
    Content    string
    Category   string   // fact, preference, instruction, context
    Embedding  []byte   // vector embedding (pgvector ou similar)
    Source     string   // session_id que originou
    Relevance  float64
    AccessCount int
    LastAccessedAt time.Time
}

// Context budget tracking
type ContextBudget struct {
    TotalTokens    int
    SystemPrompt   int
    BootstrapFiles int
    Skills         int
    Memory         int
    Tools          int
    History        int
    CurrentMessage int
    Remaining      int
}
```

### Tabelas GORM

```go
func AutoMigrate(db *gorm.DB) {
    db.AutoMigrate(
        &Session{},
        &SessionMessage{},
        &Memory{},
    )
}
```

### Operações Chave

```go
// Busca semântica de memórias relevantes
func (ms *MemoryStore) Search(ctx context.Context, query string, topK int) ([]Memory, error)

// Compaction de contexto
func (ce *ContextEngine) Compact(messages []SessionMessage, budget int) []SessionMessage

// Persistência de sessão
func (ss *SessionStore) AppendMessage(sessionID uint, msg SessionMessage) error

// Build de context window
func (ce *ContextEngine) BuildContext(session *Session, currentMessage string) (*ContextWindow, error)
```
