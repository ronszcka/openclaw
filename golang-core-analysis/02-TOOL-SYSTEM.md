# 02 - Tool System (Sistema de Ferramentas)

## Resumo

O sistema de tools permite ao agente interagir com o mundo externo: executar comandos, ler/escrever arquivos, buscar na web, enviar mensagens, gerar imagens, etc. As tools são definidas, registradas, filtradas por policy, e executadas durante o loop agêntico.

## Fluxo de Vida de uma Tool Call

```
1. DEFINIÇÃO       → Tool definida (built-in ou plugin)
2. REGISTRO        → Tool registrada no catálogo
3. POLICY          → Filtrada por policies (owner, profile, provider, group)
4. SCHEMA ADAPT    → Schema normalizado para o provider LLM (Gemini, OpenAI, etc.)
5. PROMPT          → Tool incluída no prompt do LLM
6. LLM DECIDE      → LLM decide chamar a tool
7. DISPATCH        → Gateway recebe tool invoke request
8. HOOK            → before-tool-call hooks executados
9. EXECUTE         → Tool.execute() chamada
10. RESULT         → Resultado sanitizado, truncado, alimenta próximo turn
```

## Arquivos por Camada

### 1. Definição e Criação de Tools

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/openclaw-tools.ts` | **Factory principal** `createOpenClawTools()` - monta todas as built-in tools (~20+): canvas, nodes, cron, message, tts, gateway, agents_list, sessions_*, image, pdf, web_fetch, web_search. Condicional por config, provider e feature flags |
| `src/agents/pi-tools.ts` | `createOpenClawCodingTools()` - Assembly de alto nível com policy enforcement. Cria coding tools (read, write, edit, exec, apply_patch). Wraps com before-tool-call hooks e abort signal handlers |
| `src/agents/pi-tools.types.ts` | Tipo `AnyAgentTool` - wrapper de `AgentTool<any, unknown>` |
| `src/agents/pi-tools.schema.ts` | Normalização de schema de tools: `cleanToolSchemaForGemini()`, `normalizeToolParameters()`. Flatten de anyOf/oneOf/allOf. Strip de keywords provider-specific |

### 2. Implementações de Tools Built-in

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/tools/message-tool.ts` | Tool de envio de mensagens para canais |
| `src/agents/tools/image-tool.ts` | Tool de geração de imagens |
| `src/agents/tools/pdf-tool.ts` | Tool de leitura/geração de PDF |
| `src/agents/tools/cron-tool.ts` | Tool de agendamento de cron jobs |
| `src/agents/tools/web-tools.ts` | Criação de tools web (fetch + search) |
| `src/agents/tools/web-fetch.ts` | Implementação de web fetch com proteção SSRF |
| `src/agents/tools/web-search.ts` | Tool de busca na web |
| `src/agents/tools/web-search-provider-*.ts` | Implementações específicas por provider de busca |
| `src/agents/tools/common.ts` | Helpers compartilhados: param readers, error types (`ToolInputError`, `ToolAuthorizationError`), action gates |

### 3. Tools de Bash/Exec

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/bash-tools.ts` | Definição principal de exec e process tools |
| `src/agents/bash-tools.exec.ts` | Implementação de execução de comandos |
| `src/agents/bash-tools.process.ts` | Gerenciamento de processos background |
| `src/agents/bash-tools.exec-approval-request.ts` | Fluxo de aprovação do usuário para exec |
| `src/agents/bash-tools.exec-runtime.ts` | Configuração runtime de exec |
| `src/agents/bash-tools.shared.ts` | Utilidades compartilhadas de exec |

### 4. Registro e Integração com Plugins

| Arquivo | Propósito |
|---------|-----------|
| `src/plugins/tools.ts` | `resolvePluginTools()` - Resolve tools fornecidas por plugins: invoca factory com context, detecta conflitos, aplica allowlist, rastreia metadata via WeakMap |
| `src/plugins/loader.ts` | Carregamento runtime do registry de plugins |
| `src/plugins/config-state.ts` | Normalização de configuração de plugins |
| `src/plugin-sdk/provider-tools.ts` | Helpers de schema para tools de providers: `stripUnsupportedSchemaKeywords()` para XAI, Gemini |
| `src/plugin-sdk/setup-tools.ts` | Hooks de setup/init para tools de plugins |
| `src/plugin-sdk/tool-send.ts` | Envio de tool results de plugins |

### 5. Dispatch e Execução

| Arquivo | Propósito |
|---------|-----------|
| `src/gateway/tools-invoke-http.ts` | **Endpoint HTTP** para invocação de tools. Parseia nome, action, args do request body. Valida owner-only tools. Suporta dry-run |
| `src/gateway/tool-resolution.ts` | `resolveGatewayScopedTools()` - Monta tool set efetivo para uma sessão. Aplica policies (profile, provider, group, subagent) |
| `src/agents/pi-tool-definition-adapter.ts` | **Adapter de execução** - Wraps tool execution com: before-tool-call hook invocation, result normalization, error handling, parameter sanitization |
| `src/agents/pi-tools.before-tool-call.ts` | Sistema de hooks pré-execução. Async hook invocation antes da execução real |

### 6. MCP (Model Context Protocol)

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/mcp-transport.ts` | Resolução de transporte: stdio, SSE, streamable-http. `resolveMcpTransport()` cria transporte baseado na config do server |
| `src/agents/mcp-transport-config.ts` | Parse de config de servidores MCP |
| `src/agents/mcp-stdio.ts` | Transporte MCP via stdio |
| `src/agents/mcp-sse.ts` | Transporte MCP via SSE |
| `src/agents/mcp-http.ts` | Transporte MCP via HTTP |
| `src/agents/pi-bundle-mcp-runtime.ts` | `SessionMcpRuntime` - Gerencia lifecycle de servidores MCP por sessão. Carregamento de catálogo, resolução de tools, cache |
| `src/agents/pi-bundle-mcp-materialize.ts` | `materializeBundleMcpToolsForRun()` - Converte tools do catálogo MCP para formato AgentTool. Nome seguro para evitar conflitos |
| `src/agents/pi-bundle-mcp-types.ts` | Tipos MCP: `McpToolCatalog`, `BundleMcpToolRuntime` |
| `src/agents/pi-bundle-mcp-names.ts` | Normalização de nomes de tools MCP |
| `src/mcp/channel-tools.ts` | Serving de tools MCP específicas de canais |
| `src/mcp/channel-server.ts` | Servidor MCP para operações de canal |
| `src/mcp/plugin-tools-serve.ts` | Serving de tools de plugins via protocolo MCP |

### 7. Policy e Autorização

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/tool-policy.ts` | **Policy core** - `applyOwnerOnlyToolPolicy()`, `isOwnerOnlyToolName()`. Tipos de policy: allow/deny lists. Expansão de tool groups |
| `src/agents/tool-policy-shared.ts` | Constantes e helpers: TOOL_GROUPS ("minimal", "coding", "messaging", "full") |
| `src/agents/pi-tools.policy.ts` | **Policy de alto nível** - `resolveEffectiveToolPolicy()` (combina global/agent/provider), `resolveGroupToolPolicy()` (channel/group), `resolveSubagentToolPolicy()`, `isToolAllowedByPolicies()` |
| `src/agents/tool-policy-pipeline.ts` | `applyToolPolicyPipeline()` - Aplicação sequencial de policies. Warning collection |
| `src/agents/tool-fs-policy.ts` | Controle de acesso ao filesystem |
| `src/agents/tool-policy-match.ts` | Lógica de matching de policies |

### 8. Tratamento de Resultados

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-subscribe.tools.ts` | `sanitizeToolResult()` - Trunca texto, omite dados de imagem. `extractToolResultText()`. TRUSTED_TOOL_RESULT_MEDIA - allowlist de media |
| `src/agents/session-tool-result-guard.ts` | Validação de resultados de tools |
| `src/agents/session-tool-result-guard-wrapper.ts` | Wrapper para guards de resultado |
| `src/agents/session-tool-result-state.ts` | Estado de resultados de tools |
| `src/agents/pi-embedded-runner/tool-result-truncation.ts` | Gerenciamento de tamanho de resultados |
| `src/agents/pi-embedded-runner/tool-result-context-guard.ts` | Proteção de context window |
| `src/agents/pi-embedded-runner/tool-result-char-estimator.ts` | Estimativa de caracteres para truncamento |

### 9. Normalização e Reparo

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/pi-embedded-runner/attempt.tool-call-normalization.ts` | Normaliza tool calls de diferentes modelos |
| `src/agents/pi-embedded-runner/attempt.tool-call-argument-repair.ts` | Repara argumentos malformados |
| `src/agents/tool-loop-detection.ts` | Detecta tool calls repetitivas idênticas |
| `src/agents/tool-mutation.ts` | Modificações e wrappers de tools |
| `src/agents/pi-tools.abort.ts` | `wrapToolWithAbortSignal()` - Suporte a cancelamento |

### 10. Display e Logging

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/tool-display.ts` | Formatação de display de tools |
| `src/agents/tool-display-config.ts` | Configuração de display |
| `src/agents/tool-display-exec.ts` | Display de tools de execução |
| `src/agents/tool-display-exec-shell.ts` | Display de execução shell |
| `src/agents/tool-display-common.ts` | Utilidades compartilhadas de display |
| `src/agents/tool-description-presets.ts` | Descrições predefinidas |
| `src/agents/tool-description-summary.ts` | Sumarização de descrições |
| `src/agents/tool-error-summary.ts` | Sumarização de erros |

### 11. Catálogo e Inventário

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/tool-catalog.ts` | Metadata de tools e gerenciamento de IDs |
| `src/agents/tools-effective-inventory.ts` | Inventário efetivo de tools |
| `src/gateway/server-methods/tools-effective.ts` | Endpoint HTTP para listar tools efetivas |
| `src/gateway/server-methods/tools-effective.runtime.ts` | Runtime de resolução de inventário |
| `src/gateway/server-methods/tools-catalog.ts` | Enumeração de catálogo de tools |

### 12. Configuração

| Arquivo | Propósito |
|---------|-----------|
| `src/config/types.tools.ts` | Tipos de config de tools: `ToolLoopDetectionConfig`, `ToolProfileId` ("minimal", "coding", "messaging", "full"), `MediaToolsConfig`, `LinkToolsConfig` |
| `src/config/mcp-config.ts` | Config de servidores MCP |
| `src/config/types.mcp.ts` | Tipos MCP |

## Interações entre Arquivos

```
openclaw-tools.ts ──────────┐
    (built-in tools)        │
                            ▼
pi-tools.ts ──────────────→ TOOL REGISTRY ←── resolvePluginTools()
    (coding tools)          │                  (plugin tools)
                            │
                            ▼
                    tool-policy-pipeline.ts
                    ┌───────┼───────┐
                    │       │       │
                    ▼       ▼       ▼
            owner  profile  group  subagent
            policy  policy  policy  policy
                    │
                    ▼
            pi-tools.schema.ts
            (normalize for provider)
                    │
                    ▼
            LLM receives tools in prompt
                    │
                    ▼
            LLM generates tool_use
                    │
                    ▼
            pi-tool-definition-adapter.ts
            ├── before-tool-call hook
            ├── tool.execute()
            └── result normalization
                    │
                    ▼
            pi-embedded-subscribe.tools.ts
            (sanitize result, truncate)
                    │
                    ▼
            Feed back to LLM context
```

## Mapeamento para Go

### Structs Necessárias

```go
type Tool struct {
    Name        string
    Description string
    Schema      json.RawMessage // JSON Schema dos parâmetros
    Execute     func(ctx context.Context, params map[string]any) (*ToolResult, error)
    Category    ToolCategory    // builtin, plugin, mcp
    OwnerOnly   bool
}

type ToolCall struct {
    ID         string
    ToolName   string
    Arguments  map[string]any
    Status     ToolCallStatus // pending, executing, completed, failed
    Result     *ToolResult
    StartedAt  time.Time
    Duration   time.Duration
}

type ToolResult struct {
    Content   []ContentBlock // text, image, etc.
    IsError   bool
    Truncated bool
}

type ToolPolicy struct {
    Profile    string   // minimal, coding, messaging, full
    AllowList  []string
    DenyList   []string
}

type ToolRegistry struct {
    mu       sync.RWMutex
    tools    map[string]*Tool
    policies []ToolPolicy
}
```

### Tabelas GORM Sugeridas

```go
type ToolDefinition struct {
    gorm.Model
    Name        string `gorm:"uniqueIndex"`
    Description string
    Schema      datatypes.JSON
    Category    string // builtin, plugin, mcp
    OwnerOnly   bool
    PluginID    *uint
}

type ToolExecution struct {
    gorm.Model
    RunID       uint   `gorm:"index"`
    ToolName    string
    Arguments   datatypes.JSON
    Result      datatypes.JSON
    IsError     bool
    DurationMs  int
    TokensUsed  int
}

type ToolPolicy struct {
    gorm.Model
    Name       string
    Profile    string
    AllowList  datatypes.JSON
    DenyList   datatypes.JSON
    Priority   int
}
```
