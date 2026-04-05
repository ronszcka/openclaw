# 01 - Agent Loop (Loop Agêntico)

## Resumo

O Agent Loop é o coração do OpenClaw. Implementa o ciclo **Think → Act → Observe** que permite ao agente raciocinar, chamar ferramentas, observar resultados e decidir se precisa continuar ou parar.

## Fluxo Principal

```
Mensagem do Usuário
        │
        ▼
┌─────────────────┐
│  PROMPT ASSEMBLY │  ← System prompt + context + tools + skills
│  (Bootstrap)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   THINK          │  ← LLM gera texto e/ou tool calls
│   (Inference)    │
└────────┬────────┘
         │
    ┌────┴────┐
    │ Tool    │ Não → Resposta final
    │ calls?  │
    └────┬────┘
    Sim  │
         ▼
┌─────────────────┐
│   ACT            │  ← Executa tool calls em paralelo/sequencial
│   (Tool Execute) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   OBSERVE        │  ← Resultados das tools alimentam o próximo turn
│   (Feed Results) │
└────────┬────────┘
         │
         ▼
    Loop volta para THINK
    (até não haver mais tool calls)
```

## Arquivos Principais

### Entry Point do Agent

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/agent-command.ts` | Comando CLI principal do agent (~1005 linhas). Orquestra a criação de sessão, configuração de modelo, e disparo do agent runner |
| `src/agents/pi-embedded-runner/run.ts` | **Engine principal do agent** (~2114 linhas). Função `runEmbeddedPiAgent()` - monta o contexto, cria sessão, inicia stream, processa eventos |
| `src/agents/pi-embedded-runner/run/attempt.ts` | **Attempt de execução** (~2114 linhas). Uma "tentativa" completa de run: monta tools, inicia stream, processa tool calls, gerencia compaction |

### Ciclo Think-Act-Observe

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-subscribe.ts` | **Processamento de stream** (~795 linhas). Subscreve eventos do stream LLM: `text_delta`, `tool_use`, `stop_reason`, `usage`, `compaction` |
| `src/agents/pi-embedded-runner/run/attempt.ts:850-879` | Dispatch de tool calls: tools built-in separadas de custom, `toClientToolDefinitions()` registra todas no agente |
| `src/agents/pi-tool-definition-adapter.ts` | Adapta execução de tools com hooks before-tool-call, normaliza resultados, sanitiza parâmetros para console |

### Controle de Fluxo e Parada

O agent para quando:
1. **Sem tool calls na resposta** - LLM produziu apenas texto (resposta final)
2. **Stop token / end_turn** - LLM sinalizou fim
3. **Timeout** - Tempo limite atingido (`attempt.ts:1406-1459`)
4. **Context overflow** - Compaction retry com extensão de grace period
5. **Abort signal** - Usuário cancelou ou tool `sessions_yield` chamada
6. **Stream completo** - Sem operações pendentes

### Concorrência e Filas

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-runner/lanes.ts` | **Session Lanes** - Garante execução sequencial por sessão. Lane global para coordenação. Previne runs concorrentes na mesma sessão |
| `src/agents/pi-embedded-runner/runs.ts` | **Active Run Tracking** - Singleton `activeRuns` Map. Snapshots para crash recovery. Rastreamento de model switch requests |
| `src/agents/pi-embedded-runner/queue.ts` | Fila de mensagens para processamento sequencial |

### Streaming

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-runner/run/attempt.ts:936-980` | **Stream Chain** - Compõe wrappers: provider-specific → cache trace → payload logging → idle timeout → malformed tool call repair → tool call normalization |
| `src/agents/pi-embedded-subscribe.ts` | Processa eventos do stream: `text_delta` → acumulação de texto, `tool_use` → extração de tool, `stop_reason` → detecção de conclusão, `usage` → rastreamento de tokens |
| `src/agents/pi-embedded-runner/stream-payload-utils.ts` | Utilidades para manipulação de payloads de stream |

### Compaction (Gestão de Contexto)

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-runner/run/attempt.ts` | Quando o contexto excede o limite, realiza compaction: resume mensagens antigas, mantém as recentes, e retenta |
| `src/agents/pi-embedded-runner/prompt-cache-retention.ts` | Retenção de cache de prompt durante compaction |
| `src/agents/system-prompt-cache-boundary.ts` | Boundary de cache do system prompt para estabilidade |

### Estado do Agent

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-runner/run/types.ts` | **AttemptResult** - Estado completo de um attempt: `aborted`, `timedOut`, `messagesSnapshot`, `assistantTexts`, `toolMetas`, `attemptUsage`, `itemLifecycle`, `replayMetadata` |
| `src/agents/pi-embedded-runner/types.ts` | **RunResult** - Resultado do run: `payloads` (blocos formatados), `meta`, `didSendViaMessagingTool`, `successfulCronAdds` |
| `src/agents/pi-embedded-runner/run/state.ts` | Gerenciamento de estado durante execução |

### Exports Públicos

```typescript
// Entry point principal
export { runEmbeddedPiAgent }

// Controle de run
export { abortEmbeddedPiRun, isEmbeddedPiRunActive, isEmbeddedPiRunStreaming }

// Fila de mensagens
export { queueEmbeddedPiMessage }

// Resolução de sessão
export { resolveActiveEmbeddedRunSessionId, resolveEmbeddedSessionLane, waitForEmbeddedPiRunEnd }
```

## Interações entre Arquivos

```
agent-command.ts
    │
    ├──→ pi-embedded-runner/run.ts          [monta contexto, cria sessão]
    │       │
    │       ├──→ run/attempt.ts             [executa um attempt completo]
    │       │       │
    │       │       ├──→ pi-tools.ts        [monta lista de tools]
    │       │       ├──→ pi-embedded-subscribe.ts  [processa stream]
    │       │       │       │
    │       │       │       ├──→ text_delta → acumula texto
    │       │       │       ├──→ tool_use → extrai tool call
    │       │       │       └──→ stop_reason → marca conclusão
    │       │       │
    │       │       ├──→ pi-tool-definition-adapter.ts  [executa tools]
    │       │       └──→ session-tool-result-guard.ts   [valida resultados]
    │       │
    │       ├──→ lanes.ts                   [controle de concorrência]
    │       ├──→ runs.ts                    [rastreamento de runs ativos]
    │       └──→ queue.ts                   [fila de mensagens]
    │
    └──→ pi-embedded-runner/types.ts        [tipos de resultado]
```

## Mapeamento para Go

### Structs Necessárias

```go
type AgentRun struct {
    ID            string
    SessionID     string
    Status        RunStatus // running, completed, aborted, timed_out
    Messages      []Message
    ToolCalls     []ToolCall
    Usage         TokenUsage
    StartedAt     time.Time
    CompletedAt   *time.Time
}

type AttemptResult struct {
    Aborted         bool
    TimedOut        bool
    Messages        []Message
    AssistantTexts  []string
    ToolMetas       []ToolMeta
    Usage           TokenUsage
}

type RunResult struct {
    Payloads           []Payload
    Meta               RunMeta
    SentViaMessaging   bool
}
```

### Tabelas GORM Sugeridas

```go
// agents - Configuração de agentes
type Agent struct {
    gorm.Model
    Name        string
    SystemPrompt string
    ModelID     string
    Config      datatypes.JSON
}

// agent_runs - Execuções do agent
type AgentRun struct {
    gorm.Model
    AgentID     uint
    SessionID   uint
    Status      string
    TokensIn    int
    TokensOut   int
    StartedAt   time.Time
    CompletedAt *time.Time
}

// agent_messages - Mensagens de uma run
type AgentMessage struct {
    gorm.Model
    RunID       uint
    Role        string // system, user, assistant, tool
    Content     string
    ToolCallID  *string
    Sequence    int
}
```
