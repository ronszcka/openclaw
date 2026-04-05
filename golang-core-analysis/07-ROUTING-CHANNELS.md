# 07 - Routing & Channels (Roteamento e Canais)

## Resumo

O sistema de roteamento e canais gerencia como mensagens chegam ao agente (inbound), como são processadas (auto-reply), e como respostas são entregues (delivery). Suporta 30+ canais de messaging.

## Fluxo de Mensagem Completo

```
Canal (Telegram, WhatsApp, etc.)
        │
        ▼
┌─────────────────────┐
│  CHANNEL ADAPTER     │  ← Converte formato do canal para formato interno
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  ROUTING             │  ← Resolve agent, sessão, account
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  INBOUND PROCESSING  │  ← Dedupe, sanitize, command detect
└────────┬────────────┘
         │
    ┌────┴────┐
    │Command? │ Sim → Command Handler
    └────┬────┘
    Não  │
         ▼
┌─────────────────────┐
│  DIRECTIVE PARSING   │  ← /model, /thinking, etc.
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  AGENT RUNNER        │  ← Executa agent loop
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  REPLY DELIVERY      │  ← Chunking, formatting, sending
└────────┬────────────┘
         │
         ▼
Canal (resposta entregue)
```

## 1. Channel System (Canais)

### Arquivos Core

| Arquivo | Propósito |
|---------|-----------|
| `src/channels/channel-types.ts` | **Tipos base** - `Channel`, `ChannelMessage`, `ChannelAdapter`, `ChannelCapabilities` |
| `src/channels/chat-type.ts` | Tipos de chat: `dm`, `group`, `room`, `thread` |
| `src/channels/session.ts` | Sessão de canal |
| `src/channels/targets.ts` | Targets de delivery (channel + peer + thread) |
| `src/channels/ids.ts` | IDs de canais e peers |
| `src/channels/transport/` | Camada de transporte de mensagens |
| `src/channels/allowlists/` | Allow/block lists por canal |
| `src/channels/web/` | Canal web (WhatsApp Web) |

### Plugin SDK de Canal

| Arquivo | Propósito |
|---------|-----------|
| `src/plugin-sdk/channel-contract.ts` | **Contrato público** que plugins de canal implementam |
| `src/channels/plugins/types.plugin.ts` | Tipos para plugins de canal |
| `src/channels/plugins/types.core.ts` | Tipos internos do core de canal |
| `src/channels/plugins/types.adapters.ts` | Tipos de adaptadores |

### Canais Built-in

| Diretório | Canal |
|-----------|-------|
| `src/telegram/` | Telegram Bot API |
| `src/discord/` | Discord Bot |
| `src/slack/` | Slack App |
| `src/signal/` | Signal Messenger |
| `src/imessage/` | iMessage (macOS) |
| `src/web/` | WhatsApp Web |

### Canais via Plugin

| Extensão | Canal |
|----------|-------|
| `extensions/matrix/` | Matrix |
| `extensions/zalo/` | Zalo |
| `extensions/voice-call/` | Voice Call |
| `extensions/mattermost/` | Mattermost |
| `extensions/teams/` | Microsoft Teams |
| `extensions/irc/` | IRC |
| `extensions/line/` | LINE |
| `extensions/nostr/` | Nostr |

## 2. Routing System (Roteamento)

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `src/routing/session-key.ts` | **Build de session key** (~150+ linhas). Funções: `buildAgentSessionKey()`, `buildAgentPeerSessionKey()`, `buildAgentMainSessionKey()`. Constants: `DEFAULT_ACCOUNT_ID`, `DEFAULT_MAIN_KEY` |
| `src/routing/resolve-route.ts` | **Resolução de rota** (~150+ linhas). `resolveInboundLastRouteSessionKey()`. `ResolvedAgentRoute` com `matchedBy` (binding type, default, etc.). Cache e normalização |
| `src/routing/account-id.ts` | Handling de account ID |
| `src/routing/account-lookup.ts` | Lookup de account por canal/peer |
| `src/routing/bindings.ts` | **Bindings** - Mapeamentos agent→channel→account. Precedência: peer → guild → team → account → channel |
| `src/routing/default-account-warnings.ts` | Warnings para uso de account default |

### Lógica de Roteamento

```
Mensagem chega com: channelId, peerId, groupId, accountId
        │
        ▼
1. Lookup de binding (src/routing/bindings.ts)
   Precedência:
   ├── peer binding (mais específico)
   ├── guild binding
   ├── team binding
   ├── account binding
   └── channel binding (mais genérico)
        │
        ▼
2. Build de session key (src/routing/session-key.ts)
   Formato: {agentId}:{channelId}:{peerId}:{dmScope}
        │
        ▼
3. Load/create session
```

## 3. Auto-Reply System (Processamento de Mensagens)

### Entry Point

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/reply.ts` | Exports públicos do sistema de reply |
| `src/auto-reply/reply/get-reply.ts` | **Orquestrador principal** (~200+ linhas). Carrega config, init sessão, aplica directives, invoca agent runner |
| `src/auto-reply/reply/get-reply-run.ts` | Execução do agent turn |
| `src/auto-reply/reply/agent-runner.ts` | Invocação do agente com streaming |

### Processamento Inbound

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/reply/inbound-context.ts` | Enriquece mensagem com metadata |
| `src/auto-reply/reply/inbound-dedupe.ts` | **Deduplicação** - Previne processamento duplicado |
| `src/auto-reply/reply/inbound-text.ts` | Normalização de texto: CRLF→LF, sanitização de system tags |
| `src/auto-reply/reply/inbound-meta.ts` | Extração de metadata |
| `src/auto-reply/reply/mentions.ts` | Detecção e strip de menções |

### Command Detection

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/commands-registry.ts` | **Registry de comandos** - `listChatCommands()`, `isCommandEnabled()`. Text aliases e skill command integration |
| `src/auto-reply/commands-registry.types.ts` | `ChatCommandDefinition`, `CommandDetection`, `CommandArgDefinition` |
| `src/auto-reply/commands-registry.data.ts` | Definições de comandos built-in |
| `src/auto-reply/command-detection.ts` | Detecção de comandos no texto |
| `src/auto-reply/commands-text-routing.ts` | Roteamento de comandos de texto |
| `src/auto-reply/commands-args.ts` | Parse de argumentos de comando |
| `src/auto-reply/command-auth.ts` | Autorização de comandos |

### Directives

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/reply/directive-handling.ts` | **Extração e aplicação** de diretivas (`/model`, `/thinking`, etc.) |
| `src/auto-reply/reply/directive-parsing.ts` | Parsing de sintaxe de diretivas |
| `src/auto-reply/reply/directives.ts` | Definições de diretivas |

### Diretivas Disponíveis

| Diretiva | Efeito |
|----------|--------|
| `/model <name>` | Troca o modelo LLM |
| `/thinking <level>` | Ativa/desativa extended thinking |
| `/auth <profile>` | Troca o perfil de autenticação |
| `/verbose` | Ativa modo verboso |
| `/fast` | Ativa fast lane |

### Session Management no Reply

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/reply/session.ts` | `initSessionState()` - Cria sessão a partir de config |
| `src/auto-reply/reply/session-reset-model.ts` | Reset de modelo entre turns |
| `src/auto-reply/reply/session-reset-prompt.ts` | Gerenciamento de system prompt |
| `src/auto-reply/reply/session-updates.ts` | Handling de atualizações de sessão |
| `src/auto-reply/reply/session-hooks.ts` | Integração com hooks de sessão |
| `src/auto-reply/reply/session-usage.ts` | Rastreamento de uso de tokens |
| `src/auto-reply/reply/session-delivery.ts` | Config de delivery da sessão |
| `src/auto-reply/reply/session-fork.ts` | Fork de sessão para subagents |

### Reply Delivery

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/reply/reply-payloads.ts` | Build de payloads de resposta |
| `src/auto-reply/reply/reply-delivery.ts` | Envio de respostas para canais |
| `src/auto-reply/reply/route-reply.ts` | Roteamento de respostas |
| `src/auto-reply/reply/reply-dispatcher.ts` | Dispatcher registry e routing |
| `src/auto-reply/reply/reply-threading.ts` | Suporte a threads |
| `src/auto-reply/reply/block-streaming.ts` | Streaming baseado em blocos |
| `src/auto-reply/chunk.ts` | **Chunking** de respostas grandes |

### Commands Implementation

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/reply/commands.ts` | Dispatcher de execução de comandos |
| `src/auto-reply/reply/commands-core.ts` | Comandos de sistema (start, stop) |
| `src/auto-reply/reply/commands-session.ts` | Comandos de sessão |
| `src/auto-reply/reply/commands-models.ts` | Comandos de seleção de modelo |
| `src/auto-reply/reply/commands-status.ts` | Comandos de status/info |
| `src/auto-reply/reply/commands-config.ts` | Comandos de config |
| `src/auto-reply/reply/commands-context.ts` | Comandos de manipulação de contexto |
| `src/auto-reply/reply/commands-bash.ts` | Comandos de execução bash |
| `src/auto-reply/reply/commands-plugins.ts` | Comandos de plugins |
| `src/auto-reply/reply/commands-tasks.ts` | Comandos de tasks |
| `src/auto-reply/reply/commands-subagents.ts` | Comandos de subagents |
| `src/auto-reply/reply/commands-tts.ts` | Comandos de text-to-speech |

### Especiais

| Arquivo | Propósito |
|---------|-----------|
| `src/auto-reply/reply/groups.ts` | Handling de grupos: `resolveGroupRequireMention()` |
| `src/auto-reply/reply/thinking.ts` | Extended thinking mode |
| `src/auto-reply/reply/heartbeat.ts` | Heartbeat status messages |
| `src/auto-reply/reply/abort.ts` | Abort/stop signal |
| `src/auto-reply/reply/media-note.ts` | Handling de media attachments |
| `src/auto-reply/reply/subagents-utils.ts` | Utilidades de subagents |
| `src/auto-reply/templating.ts` | Template variable substitution |
| `src/auto-reply/tokens.ts` | Token counting e budgeting |

## 4. Gateway HTTP

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `src/gateway/server/server.ts` | **HTTP server** principal |
| `src/gateway/server/router.ts` | Roteamento de requests HTTP |
| `src/gateway/protocol/schema.ts` | **Schema do protocolo** wire |
| `src/gateway/protocol/index.ts` | Exports do protocolo |
| `src/gateway/tools-invoke-http.ts` | Endpoint de invocação de tools |
| `src/gateway/tool-resolution.ts` | Resolução de tools para sessão |
| `src/gateway/server-methods/` | Implementações dos métodos HTTP |

## Interações entre Arquivos

```
Channel (ex: Telegram)
      │
      ├──→ src/telegram/webhook.ts (recebe webhook)
      │
      ▼
channels/channel-types.ts (normaliza mensagem)
      │
      ▼
routing/resolve-route.ts
      │ (resolve agent + session)
      ▼
routing/session-key.ts
      │ (build session key)
      ▼
auto-reply/reply/get-reply.ts
      │
      ├──→ reply/inbound-dedupe.ts (check duplicate)
      ├──→ reply/inbound-text.ts (normalize text)
      ├──→ reply/mentions.ts (strip mentions)
      │
      ├──→ commands-registry.ts (check command)
      │       │
      │       └──→ reply/commands-*.ts (execute if command)
      │
      ├──→ reply/directive-handling.ts (parse directives)
      │
      ├──→ reply/session.ts (init session)
      │
      ├──→ reply/agent-runner.ts (run agent)
      │       │
      │       └──→ agents/pi-embedded-runner/ (agent loop)
      │
      ├──→ reply/reply-payloads.ts (format response)
      ├──→ chunk.ts (split large responses)
      └──→ reply/reply-delivery.ts (send to channel)
```

## Mapeamento para Go + GORM

### Structs

```go
// Channel
type Channel struct {
    gorm.Model
    ChannelID   string `gorm:"uniqueIndex"`
    Type        string // telegram, discord, slack, etc.
    Name        string
    Config      datatypes.JSON
    Enabled     bool
}

// Inbound Message
type InboundMessage struct {
    gorm.Model
    ChannelID   string `gorm:"index"`
    PeerID      string
    GroupID     *string
    ThreadID    *string
    ChatType    string // dm, group, room, thread
    Text        string
    MediaURLs   datatypes.JSON
    Metadata    datatypes.JSON
    DedupeKey   string `gorm:"uniqueIndex"`
    ProcessedAt *time.Time
}

// Route Binding
type RouteBinding struct {
    gorm.Model
    AgentID     uint   `gorm:"index"`
    ChannelID   string
    AccountID   string
    PeerID      *string
    GuildID     *string
    TeamID      *string
    SessionKey  string
    Priority    int // peer > guild > team > account > channel
}

// Command
type Command struct {
    Name        string
    Description string
    Aliases     []string
    Scope       string // dm, group, all
    Handler     func(ctx context.Context, args CommandArgs) (*CommandResult, error)
}

// Directive
type Directive struct {
    Name   string
    Value  string
}

// Reply Delivery
type ReplyDelivery struct {
    gorm.Model
    SessionID   uint
    ChannelID   string
    PeerID      string
    ThreadID    *string
    Content     string
    Chunks      int
    Status      string // pending, delivered, failed
    DeliveredAt *time.Time
}
```

### Interfaces Go

```go
// Channel adapter interface
type ChannelAdapter interface {
    ID() string
    Name() string
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Send(ctx context.Context, target DeliveryTarget, message OutboundMessage) error
    OnMessage(handler func(InboundMessage))
}

// Router
type Router interface {
    ResolveRoute(msg InboundMessage) (*ResolvedRoute, error)
    BuildSessionKey(agentID, channelID, peerID, dmScope string) string
}

// Auto-reply pipeline
type ReplyPipeline interface {
    Process(ctx context.Context, msg InboundMessage) (*ReplyResult, error)
}
```
