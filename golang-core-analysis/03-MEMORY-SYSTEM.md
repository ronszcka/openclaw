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

## 5. Memory Host SDK Detalhado (`src/memory-host-sdk/`)

O SDK é muito mais rico do que a interface básica. Contém contratos para embedding, storage, QMD queries e dreaming.

### Arquivos do Host SDK

| Arquivo | Propósito |
|---------|-----------|
| `src/memory-host-sdk/engine.ts` | Contrato agregado de todas as superfícies do memory engine |
| `src/memory-host-sdk/engine-foundation.ts` | Contrato de fundação: agent scope, memory search config, state dir |
| `src/memory-host-sdk/engine-storage.ts` | Contrato de storage/index helpers |
| `src/memory-host-sdk/engine-qmd.ts` | Contrato QMD (Query Markdown) - session/query helpers |
| `src/memory-host-sdk/engine-embeddings.ts` | Contrato de embedding providers e batch helpers |
| `src/memory-host-sdk/dreaming.ts` | **Tipos de dreaming**: light, deep, REM sleep |
| `src/memory-host-sdk/multimodal.ts` | Handling de conteúdo multimodal |
| `src/memory-host-sdk/query.ts` | Contrato de processamento de queries |
| `src/memory-host-sdk/runtime-core.ts` | Definições de runtime core |
| `src/memory-host-sdk/runtime-cli.ts` | Utilidades CLI de runtime |
| `src/memory-host-sdk/runtime-files.ts` | Runtime de handling de arquivos |
| `src/memory-host-sdk/status.ts` | Formatação de status |

### Embedding Infrastructure (`src/memory-host-sdk/host/`)

~60 arquivos para infraestrutura de embeddings:

| Arquivo | Propósito |
|---------|-----------|
| `src/memory-host-sdk/host/embeddings.ts` | Orquestração de embedding providers |
| `src/memory-host-sdk/host/embeddings-gemini.ts` | Provider Gemini |
| `src/memory-host-sdk/host/embeddings-mistral.ts` | Provider Mistral |
| `src/memory-host-sdk/host/embeddings-ollama.ts` | Provider Ollama |
| `src/memory-host-sdk/host/embeddings-openai.ts` | Provider OpenAI |
| `src/memory-host-sdk/host/embeddings-voyage.ts` | Provider Voyage |
| `src/memory-host-sdk/host/batch-*.ts` | Batch embedding runners por provider |
| `src/memory-host-sdk/host/embedding-chunk-limits.ts` | Limites de chunking |
| `src/memory-host-sdk/host/embedding-input-limits.ts` | Limites de input |
| `src/memory-host-sdk/host/embedding-model-limits.ts` | Limites por modelo |
| `src/memory-host-sdk/host/query-expansion.ts` | Expansão de queries para melhor recall |
| `src/memory-host-sdk/host/qmd-process.ts` | Execução de comandos QMD CLI |
| `src/memory-host-sdk/host/qmd-query-parser.ts` | Parse de resultados QMD |
| `src/memory-host-sdk/host/qmd-scope.ts` | Constraints de scope QMD |
| `src/memory-host-sdk/host/memory-schema.ts` | Schema SQLite |
| `src/memory-host-sdk/host/sqlite.ts` | Driver SQLite |
| `src/memory-host-sdk/host/sqlite-vec.ts` | Driver SQLite com vector search |

## 6. Memory Core Extension Detalhado (`extensions/memory-core/`)

### Core Memory Manager (~35 arquivos)

| Arquivo | Propósito |
|---------|-----------|
| `extensions/memory-core/src/memory/manager.ts` | **MemoryIndexManager** - gerencia SQLite vector/FTS indices, embedding sync, readonly recovery, batch operations |
| `extensions/memory-core/src/memory/manager-embedding-ops.ts` | Operações de batch embedding |
| `extensions/memory-core/src/memory/manager-search.ts` | Busca em memória (vector + keyword) |
| `extensions/memory-core/src/memory/manager-sync-ops.ts` | File sync e indexação |

### Algoritmos de Busca

| Arquivo | Propósito |
|---------|-----------|
| `extensions/memory-core/src/memory/hybrid.ts` | **Busca híbrida** BM25 + vector similarity |
| `extensions/memory-core/src/memory/mmr.ts` | **MMR** (Maximal Marginal Relevance) - ranking por diversidade |
| `extensions/memory-core/src/memory/temporal-decay.ts` | **Temporal decay** - scoring por recência |

### Short-Term Promotion (Promoção de Memória)

| Arquivo | Propósito |
|---------|-----------|
| `extensions/memory-core/src/short-term-promotion.ts` | **Promoção weights-based** (~40KB) - promove de daily files para long-term memory. Rastreia recall history com métricas: frequency, relevance, diversity, recency, consolidation, conceptual |
| `extensions/memory-core/src/concept-vocabulary.ts` | Deriva concept tags do conteúdo |
| `extensions/memory-core/src/flush-plan.ts` | Planeja flush de memória baseado em token budgets |
| `extensions/memory-core/src/prompt-section.ts` | Gera contexto de memória para prompts |

### Tools Expostas ao Agente

| Arquivo | Propósito |
|---------|-----------|
| `extensions/memory-core/src/tools.ts` | Tools `memory_search` e `memory_get` |
| `extensions/memory-core/src/tools.citations.ts` | Tracking de citações de recall |
| `extensions/memory-core/src/tools.recall-tracking.ts` | Rastreia o que foi recalled e quando |

### CLI de Memória

| Arquivo | Propósito |
|---------|-----------|
| `extensions/memory-core/src/cli.ts` | Interface CLI (~7KB) |
| `extensions/memory-core/src/cli.runtime.ts` | Implementação completa (~47KB, 20+ comandos) |

## 7. Sistema de Dreaming (Processamento de Memória em Background)

O OpenClaw implementa um sistema inspirado em sono humano para consolidar memórias:

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `extensions/memory-core/src/dreaming.ts` | **Dreaming principal** (~40KB) - promoção de short-term com cron scheduling |
| `extensions/memory-core/src/dreaming-command.ts` | Comando CLI para dreaming manual |
| `extensions/memory-core/src/dreaming-phases.ts` | **Fases de dreaming** (~200+ linhas) - light e REM sleep |
| `extensions/memory-core/src/dreaming-markdown.ts` | Escrita de relatórios de dreaming em markdown |

### Fases de Dreaming

```
LIGHT SLEEP (A cada 6 horas)
├── Deduplica entries recentes dos daily memory files
├── Source: últimos 2 dias de memory files + session recall
├── Limite: 100 entries max
└── Output: relatórios inline ou separados

DEEP SLEEP (Diariamente às 3 AM)
├── Promove high-scoring short-term recalls para MEMORY.md
├── Sources: daily files, memory indices, sessions, logs, recalls
├── Scoring: fórmula ponderada
│   ├── frequency:     0.24
│   ├─�� relevance:     0.30
│   ���── diversity:     0.15
│   ├── recency:       0.15
│   ├── consolidation: 0.10
│   └��─ conceptual:    0.06
├── Thresholds: min score 0.8, min recall count 3, min unique queries 3
├── Recovery mode: auto-repair se health < 35%
└── Limite: 10 entries max por run

REM SLEEP (Semanalmente, Domingo 5 AM)
├── Descoberta de padrões e síntese cross-cutting
├── Lookback: 7 dias
├── Identifica meta-patterns
└���─ Limite: 10 entries max
```

### Configuração de Dreaming

```
Speed:    fast | balanced | slow
Thinking: low | medium | high
Budget:   cheap | medium | expensive
(Configurável por fase)
```

## 8. Agent-Level Memory

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/memory-search.ts` | **Config de busca por agente** (~100+ linhas). Hybrid search (vector + keyword) com MMR e temporal decay. FTS tokenizer (unicode61 ou trigram). Chunking params (token size 400 default, overlap 80). Sync strategies (on start, on search, watch, interval) |
| `src/agents/subagent-registry-memory.ts` | Handling de memória para spawning de subagents |
| `src/agents/pi-embedded-runner/run/attempt.memory-flush-forwarding.ts` | Forwarding de memory flush durante agent runs |

## 9. Plugin SDK Facades para Memória

| Arquivo | Propósito |
|---------|-----------|
| `src/plugin-sdk/memory-core.ts` | Superfície principal do SDK de memória |
| `src/plugin-sdk/memory-core-host-engine-foundation.ts` | Contratos de fundação |
| `src/plugin-sdk/memory-core-host-engine-storage.ts` | Contratos de storage |
| `src/plugin-sdk/memory-core-host-engine-qmd.ts` | Contratos de query QMD |
| `src/plugin-sdk/memory-core-host-engine-embeddings.ts` | Contratos de embedding |
| `src/plugin-sdk/memory-core-host-runtime-core.ts` | Tipos de runtime core |
| `src/plugin-sdk/memory-core-host-runtime-cli.ts` | Helpers de runtime CLI |
| `src/plugin-sdk/memory-core-host-runtime-files.ts` | Operações de runtime de arquivos |
| `src/plugin-sdk/memory-core-host-status.ts` | Resolução de config de dreaming |
| `src/plugin-sdk/memory-core-host-query.ts` | Utilidades de query |
| `src/plugin-sdk/memory-core-host-secret.ts` | Handling de secrets |
| `src/plugin-sdk/memory-core-host-multimodal.ts` | Suporte multimodal |
| `src/plugin-sdk/memory-lancedb.ts` | SDK do plugin LanceDB |
| `src/plugin-sdk/webhook-memory-guards.ts` | Guards de segurança para operações webhook |

## 10. Session Store Detalhado (`src/config/sessions/`)

~26 arquivos para gerenciamento completo de sessões:

| Arquivo | Propósito |
|---------|-----------|
| `src/config/sessions/store.ts` | **Store principal** com disk locking, caching, maintenance |
| `src/config/sessions/store-cache.ts` | Camada de cache |
| `src/config/sessions/store-load.ts` | Loading de session stores do disco |
| `src/config/sessions/store-lock-state.ts` | Fila de write locks |
| `src/config/sessions/store-maintenance.ts` | Pruning, rotation, disk budget |
| `src/config/sessions/store-pruning.ts` | Lógica de pruning de entries |
| `src/config/sessions/store-migrations.ts` | Migrações de dados |
| `src/config/sessions/store-read.ts` | Acesso read-only |
| `src/config/sessions/transcript.ts` | **Append e gerenciamento** de transcript entries |
| `src/config/sessions/transcript-mirror.ts` | Mirroring de texto de transcript |
| `src/config/sessions/transcript-events.ts` | Broadcasting de eventos de transcript |
| `src/config/sessions/targets.ts` | Resolução e filtragem de targets |
| `src/config/sessions/metadata.ts` | Derivação e atualização de metadata |
| `src/config/sessions/paths.ts` | Resolução de paths (10+ helpers) |
| `src/config/sessions/session-key.ts` | Normalização e parsing de session key |
| `src/config/sessions/session-id.ts` | Handling de session ID |
| `src/config/sessions/main-session.ts` | Gerenciamento de sessão principal |
| `src/config/sessions/group.ts` | Lógica de agrupamento de sessões |
| `src/config/sessions/disk-budget.ts` | Enforcement de quota de disco |
| `src/config/sessions/reset.ts` | Operações de reset/cleanup |
| `src/config/sessions/delivery-info.ts` | Info de contexto de delivery |
| `src/config/sessions/send-policy.ts` | Enforcement de send policy |

## 11. Estruturas de Dados Chave

### Short-Term Recall Entry
```typescript
{
  key: string,
  path: string,
  startLine: number,
  endLine: number,
  snippet: string,
  recallCount: number,
  totalScore: number,
  maxScore: number,
  firstRecalledAt: Date,
  lastRecalledAt: Date,
  queryHashes: string[],     // max 32
  recallDays: string[],      // max 16
  conceptTags: string[],
  promotedAt?: Date
}
```

### Memory Entry (LanceDB)
```typescript
{
  id: string,
  text: string,
  vector: number[],
  importance: number,
  category: string,          // facts, interactions, learnings, preferences, goals
  createdAt: Date
}
```

### Session Entry
```typescript
{
  sessionId: string,
  updatedAt: Date,
  sessionFile?: string,
  spawnedBy?: string,
  parentSessionKey?: string,
  forkedFromParent?: boolean,
  spawnDepth?: number,
  subagentRole?: string,
  lastHeartbeatText?: string,
  lastHeartbeatSentAt?: Date
}
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

### Structs Adicionais para Dreaming/Recall

```go
// Short-term recall tracking
type RecallEntry struct {
    gorm.Model
    AgentID         uint      `gorm:"index;not null"`
    Key             string    `gorm:"not null"`
    Path            string
    StartLine       int
    EndLine         int
    Snippet         string    `gorm:"type:text"`
    RecallCount     int       `gorm:"default:0"`
    TotalScore      float64   `gorm:"default:0"`
    MaxScore        float64   `gorm:"default:0"`
    FirstRecalledAt time.Time
    LastRecalledAt  time.Time
    QueryHashes     datatypes.JSON // []string max 32
    RecallDays      datatypes.JSON // []string max 16
    ConceptTags     datatypes.JSON // []string
    PromotedAt      *time.Time
}

// Dreaming job tracking
type DreamingRun struct {
    gorm.Model
    AgentID       uint      `gorm:"index;not null"`
    Phase         string    `gorm:"not null"` // light, deep, rem
    Status        string    `gorm:"not null"` // running, completed, failed
    EntriesInput  int
    EntriesOutput int
    PromotedCount int
    Report        string    `gorm:"type:text"`
    StartedAt     time.Time
    CompletedAt   *time.Time
    Error         *string
}

// Embedding cache
type EmbeddingCache struct {
    gorm.Model
    ContentHash string    `gorm:"uniqueIndex;not null"`
    Provider    string    `gorm:"not null"`
    Model       string    `gorm:"not null"`
    Vector      []byte    `gorm:"not null"` // serialized float32 array
    Dimensions  int
    CreatedAt   time.Time
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
