# 04 - Reasoning & Inference (Raciocínio e Inferência LLM)

## Resumo

O sistema de reasoning/inference é responsável por: montar prompts, chamar LLMs, processar streams de resposta, e gerenciar cache de prompts. Usa uma abstração de providers para suportar múltiplos LLMs (Anthropic, OpenAI, Google, etc.).

## Fluxo de Inferência

```
Contexto Montado (context-engine)
        │
        ▼
┌─────────────────────┐
│  PROMPT ASSEMBLY     │  ← System prompt + messages + tools
│  (Composição)        │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  PROVIDER SELECTION  │  ← Resolve qual LLM usar (model config)
│  (Plugin SDK)        │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  CACHE MANAGEMENT    │  ← Prompt cache stability
│  (Cache Boundary)    │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  LLM API CALL        │  ← Streaming request ao provider
│  (Stream)            │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  STREAM PROCESSING   │  ← text_delta, tool_use, stop_reason
│  (Events)            │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  THINKING MODE       │  ← Extended thinking / chain-of-thought
│  (Opcional)          │
└─────────────────────┘
```

## Arquivos por Camada

### 1. Prompt Assembly (Composição de Prompt)

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-runner/run.ts` | Monta o prompt completo: system prompt base + bootstrap + skills + context + tools. Função principal `runEmbeddedPiAgent()` |
| `src/agents/pi-embedded-runner/run/attempt.ts` | Em cada attempt, reconfigura o prompt baseado no estado atual. Adiciona mensagens de retry se necessário |
| `src/agents/prompt-composition-scenarios.ts` | Cenários de composição de prompt para teste e validação |
| `src/context-engine/context-engine.ts` | Engine de contexto que alimenta a composição |
| `src/context-engine/bootstrap-files.ts` | Carrega AGENTS.md, SKILL.md e outros bootstrap files |

### System Prompt Structure

```
[System Prompt]
You are {agent_name}, a personal AI assistant...

[Bootstrap Files]
# AGENTS.md content
# Project-specific instructions

[Skills]
<available_skills>
  <skill name="weather" description="..." />
  <skill name="github" description="..." />
</available_skills>

[Memory Context]
Relevant memories from long-term storage...

[Conversation History]
User: ...
Assistant: ...
Tool Result: ...

[Tools are passed separately via the API tools parameter]
```

### 2. Provider Abstraction (Abstração de Providers)

| Arquivo | Propósito |
|---------|-----------|
| `src/plugin-sdk/provider-entry.ts` | **Contrato público** de plugins de provider. Define interface que providers implementam: `createCompletion()`, `listModels()`, `getModelCapabilities()` |
| `src/plugin-sdk/provider-auth.ts` | Autenticação de providers: API keys, OAuth, custom auth |
| `src/plugin-sdk/provider-catalog-shared.ts` | Catálogo compartilhado de modelos |
| `src/plugin-sdk/provider-model-shared.ts` | Metadata compartilhada de modelos |
| `src/plugin-sdk/provider-tools.ts` | Adaptação de tools para providers específicos |
| `src/plugins/types.ts` | Tipos do sistema de plugins incluindo provider types |

### Provider Extensions (Exemplos)

| Extensão | Provider |
|----------|---------|
| `extensions/openai/` | OpenAI (GPT-4, GPT-4o, etc.) |
| `extensions/anthropic/` | Anthropic (Claude) |
| `extensions/google-gemini/` | Google (Gemini) |
| `extensions/ollama/` | Ollama (modelos locais) |
| `extensions/groq/` | Groq |
| `extensions/deepseek/` | DeepSeek |
| `extensions/openrouter/` | OpenRouter (multi-provider) |

### Como um Provider é Registrado

```typescript
// extensions/anthropic/src/index.ts (simplificado)
export default {
    id: "anthropic",
    name: "Anthropic",
    createCompletion: async (params) => {
        // Chama API da Anthropic
        return stream;
    },
    listModels: async () => [...],
    getModelCapabilities: (modelId) => ({
        streaming: true,
        tools: true,
        thinking: true,
        vision: true,
    }),
};
```

### 3. Streaming e Processamento de Resposta

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-subscribe.ts` | **Subscriber principal** (~795 linhas). Processa eventos do stream: |
| | `text_delta` → acumula texto da resposta |
| | `tool_use` → extrai tool calls |
| | `stop_reason` → detecta conclusão |
| | `usage` → rastreia tokens |
| | `compaction` → sinaliza overflow de contexto |
| `src/agents/pi-embedded-runner/run/attempt.ts:936-980` | **Stream Chain** - pipeline de wrappers: |
| | 1. Provider-specific wrapper |
| | 2. Cache trace wrapping |
| | 3. Payload logging |
| | 4. Idle timeout detection |
| | 5. Malformed tool call repair |
| | 6. Tool call normalization |
| `src/agents/pi-embedded-runner/stream-payload-utils.ts` | Utilidades para manipulação de payloads do stream |
| `src/agents/pi-embedded-payloads.ts` | Formatação de payloads do agent para delivery |

### Eventos do Stream

```typescript
// Tipos de eventos processados pelo subscriber
type StreamEvent =
    | { type: "text_delta"; text: string }           // Fragmento de texto
    | { type: "tool_use"; id: string; name: string; input: any }  // Tool call
    | { type: "stop_reason"; reason: string }        // end_turn, tool_use, max_tokens
    | { type: "usage"; input_tokens: number; output_tokens: number }
    | { type: "compaction"; summary: string }        // Context overflow
    | { type: "thinking"; text: string }             // Extended thinking
```

### 4. Prompt Cache (Estabilidade de Cache)

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/prompt-cache-stability.ts` | **Estabilidade de cache** - Garante que o prefixo do prompt permanece byte-idêntico entre turns para aproveitar o cache do provider |
| `src/agents/system-prompt-cache-boundary.ts` | Define boundary do system prompt para cache. Separa parte cacheável da dinâmica |
| `src/agents/bootstrap-cache.ts` | Cache de bootstrap files para evitar re-leitura |
| `src/agents/cache-trace.ts` | Tracing de cache hits/misses |
| `src/agents/context-cache.ts` | Cache de contexto computado |
| `src/agents/pi-embedded-runner/anthropic-cache-control-payload.ts` | Payload de cache control para Anthropic |
| `src/agents/pi-embedded-runner/anthropic-family-cache-semantics.ts` | Semântica de cache para família Anthropic |
| `src/agents/pi-embedded-runner/cache-ttl.ts` | TTL de cache |
| `src/agents/pi-embedded-runner/google-prompt-cache.ts` | Cache de prompt para Google |
| `src/agents/pi-embedded-runner/prompt-cache-observability.ts` | Observabilidade de cache |
| `src/agents/pi-embedded-runner/prompt-cache-retention.ts` | Retenção de cache durante compaction |
| `src/agents/pi-embedded-runner/session-manager-cache.ts` | Cache do session manager |

### 5. Thinking / Extended Reasoning

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/reply/thinking.ts` | Integração de **extended thinking mode**. Configura o LLM para raciocínio estendido (chain-of-thought visível) |
| `src/agents/pi-embedded-runner/run.ts` | Configura thinking mode no session setup |
| `src/auto-reply/reply/directive-handling.ts` | Processa diretiva `/thinking` do usuário para ativar/desativar |

### Níveis de Thinking

```
- off: Sem thinking estendido (padrão)
- low: Thinking básico
- medium: Thinking moderado
- high: Thinking profundo (mais tokens, mais tempo)
```

### 6. Payload Logging e Observabilidade

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/anthropic-payload-log.ts` | Logging de payloads Anthropic para debug |
| `src/agents/anthropic-payload-policy.ts` | Políticas de logging (redação de dados sensíveis) |
| `src/agents/openai-responses-payload-policy.ts` | Políticas de logging para OpenAI |
| `src/agents/payload-redaction.ts` | Redação de dados sensíveis em payloads |
| `src/agents/pi-embedded-runner/run/payloads.ts` | Gerenciamento de payloads durante runs |

### 7. Adaptação para Providers Específicos

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-tools.schema.ts` | `cleanToolSchemaForGemini()` - Remove keywords não suportadas |
| `src/agents/pi-embedded-runner/anthropic-family-tool-payload-compat.ts` | Compatibilidade de tool payloads para Anthropic |
| `src/agents/openai-tool-schema.ts` | Compatibilidade de schema para OpenAI |
| `src/agents/model-tool-support.ts` | Detecção de capabilities por modelo |

### 8. Seleção de Modelo

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/reply/directive-handling.ts` | Processa diretiva `/model` para trocar modelo |
| `src/auto-reply/reply/session-reset-model.ts` | Reset de modelo entre turns |
| `src/cron/isolated-agent/model-selection.ts` | Seleção de modelo para cron jobs |
| `src/agents/pi-embedded-runner/run.ts` | Configuração de modelo no session setup |

## Interações entre Arquivos

```
User Message
      │
      ▼
context-engine/context-engine.ts
      │ (monta contexto completo)
      ▼
pi-embedded-runner/run.ts
      │ (configura sessão, modelo, thinking)
      │
      ├──→ plugin-sdk/provider-entry.ts
      │       │ (resolve provider)
      │       ▼
      │    extensions/{provider}/
      │       │ (implementação específica)
      │       ▼
      │    LLM API Call (streaming)
      │
      ├──→ prompt-cache-stability.ts
      │       (garante cache hit)
      │
      ├──→ system-prompt-cache-boundary.ts
      │       (separa cacheável de dinâmico)
      │
      ▼
pi-embedded-subscribe.ts
      │ (processa eventos do stream)
      │
      ├── text_delta → acumula texto
      ├── tool_use → dispatch para tool system
      ├── thinking → captura reasoning
      ├── usage → contabiliza tokens
      └── stop_reason → finaliza ou continua loop
```

## 9. Transport Layer (Camada de Transporte HTTP)

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/provider-transport-stream.ts` | Criação de stream function com awareness de transporte |
| `src/agents/provider-transport-fetch.ts` | HTTP fetch wrapper com error handling |
| `src/agents/openai-transport-stream.ts` | Streaming da API OpenAI (ChatCompletion, Responses) |
| `src/agents/anthropic-transport-stream.ts` | Streaming da API Anthropic Messages |
| `src/agents/google-transport-stream.ts` | Streaming da API Google Generative AI |
| `src/agents/openai-ws-stream.ts` | Suporte WebSocket para OpenAI |
| `src/agents/anthropic-vertex-stream.ts` | Suporte a AWS Bedrock vertex |
| `src/agents/transport-stream-shared.ts` | Utilidades compartilhadas de transporte |
| `src/agents/transport-message-transform.ts` | Transformação de formato de mensagens entre APIs |

## 10. Error Handling e Recovery

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-helpers/errors.ts` | **Classificação de erros**: billing, rate limit, auth, context overflow |
| `src/agents/pi-embedded-helpers/provider-error-patterns.ts` | Pattern matching de erros por provider |
| `src/agents/pi-embedded-runner/run/assistant-failover.ts` | Orquestração de failover |
| `src/agents/pi-embedded-runner/run/stop-reason-recovery.ts` | Recovery de stops incompletos |
| `src/agents/pi-embedded-runner/run/incomplete-turn.ts` | Recovery de turns incompletos |
| `src/agents/pi-embedded-runner/run/failover-policy.ts` | Policy de failover de modelo |
| `src/agents/configured-provider-fallback.ts` | Seleção de provider fallback |

### Estratégias de Recovery

```
1. Rate limit → Retry com backoff exponencial
2. Auth error → Tenta próximo provider configurado
3. Context overflow → Compaction + retry
4. Thinking error → Reduz thinking level e retenta
5. Billing error → Alerta e para
6. Incomplete stop → Continua o turn
7. Model unavailable → Failover para modelo alternativo
```

## 11. Usage Tracking

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/usage.ts` | Agregação e normalização de usage |
| `src/agents/pi-embedded-runner/usage-accumulator.ts` | Acumulação entre attempts |
| `src/agents/pi-embedded-runner/usage-reporting.ts` | Reporting de usage |
| `src/agents/provider-attribution.ts` | Atribuição de usage a providers |

## 12. Model Configuration e Seleção

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/models-config.ts` | Loading de config de modelos |
| `src/agents/models-config.providers.ts` | Definições de modelos por provider |
| `src/agents/models-config.providers.normalize.ts` | Normalização de model IDs |
| `src/agents/model-selection.ts` | Resolução de modelo a partir de strings e aliases |
| `src/agents/model-auth.ts` | Resolução de API keys e auth headers |

## 13. Provider Stream Wrappers

| Arquivo | Propósito |
|---------|-----------|
| `src/plugin-sdk/provider-stream.ts` | Factory e composição de stream wrappers |
| `src/plugin-sdk/provider-stream-family.ts` | Famílias de stream built-in (google-thinking, moonshot, openai-responses) |
| `src/plugin-sdk/provider-stream-shared.ts` | Utilidades de streaming compartilhadas |
| `src/agents/pi-embedded-runner/openai-stream-wrappers.ts` | Wrapping de reasoning output OpenAI |
| `src/agents/pi-embedded-runner/google-stream-wrappers.ts` | Wrapping de thinking payload Google |
| `src/agents/pi-embedded-runner/moonshot-thinking-stream-wrappers.ts` | Wrapper Moonshot thinking |
| `src/agents/pi-embedded-runner/proxy-stream-wrappers.ts` | Suporte a reasoning via proxy (OpenRouter, Kilocode) |

## Mapeamento para Go

### Structs

```go
// Provider abstraction
type LLMProvider interface {
    CreateCompletion(ctx context.Context, req CompletionRequest) (*CompletionStream, error)
    ListModels(ctx context.Context) ([]Model, error)
    GetModelCapabilities(modelID string) ModelCapabilities
}

type CompletionRequest struct {
    Model       string
    Messages    []Message
    Tools       []ToolDefinition
    SystemPrompt string
    MaxTokens   int
    Temperature float64
    Thinking    ThinkingLevel
    Stream      bool
}

type CompletionStream struct {
    Events <-chan StreamEvent
    Done   <-chan struct{}
    Err    error
}

type StreamEvent struct {
    Type       string // text_delta, tool_use, stop_reason, usage, thinking
    Text       string
    ToolCall   *ToolCall
    StopReason string
    Usage      *TokenUsage
}

type ModelCapabilities struct {
    Streaming bool
    Tools     bool
    Thinking  bool
    Vision    bool
    MaxTokens int
}

// Prompt cache
type PromptCache struct {
    mu         sync.RWMutex
    prefix     []byte
    prefixHash string
    hits       int64
    misses     int64
}

type ThinkingLevel string
const (
    ThinkingOff    ThinkingLevel = "off"
    ThinkingLow    ThinkingLevel = "low"
    ThinkingMedium ThinkingLevel = "medium"
    ThinkingHigh   ThinkingLevel = "high"
)
```

### Tabelas GORM

```go
type InferenceLog struct {
    gorm.Model
    RunID        uint   `gorm:"index"`
    Provider     string
    Model        string
    InputTokens  int
    OutputTokens int
    ThinkingLevel string
    CacheHit     bool
    LatencyMs    int
    StopReason   string
}

type ProviderConfig struct {
    gorm.Model
    ProviderID   string `gorm:"uniqueIndex"`
    Name         string
    APIKey       string // encrypted
    BaseURL      string
    DefaultModel string
    Enabled      bool
    Config       datatypes.JSON
}
```
