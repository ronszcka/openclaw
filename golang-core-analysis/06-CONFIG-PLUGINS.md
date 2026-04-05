# 06 - Config & Plugin System

## Resumo

O sistema de configuração e plugins é o backbone do OpenClaw. Permite extensibilidade total via plugins para channels, providers, tools, e skills. A configuração é schema-validated com Zod e persistida em arquivo JSON.

## 1. Configuration System

### Fluxo de Configuração

```
~/.openclaw/config.json (ou config.json5)
        │
        ▼
config/loader.ts (read + parse)
        │
        ▼
config/schema.ts (Zod validation)
        │
        ▼
config/state.ts (runtime state)
        │
        ▼
Disponível para todo o sistema
```

### Arquivos de Config

| Arquivo | Propósito |
|---------|-----------|
| `src/config/config.ts` | **Config principal** - Loading, merging, defaults. Entry point para todo o sistema de config |
| `src/config/config-loader.ts` | Loader de config: lê arquivo, faz parse JSON5, aplica defaults |
| `src/config/config-schema.ts` | **Schema Zod** - Validação completa da config. Define todos os campos, tipos, defaults |
| `src/config/config-types.ts` | Tipos TypeScript gerados do schema |
| `src/config/config-state.ts` | Estado runtime da config: singleton acessível globalmente |
| `src/config/config-defaults.ts` | Valores default para todos os campos |
| `src/config/config-merge.ts` | Merge de configs (base + overrides) |
| `src/config/config-write.ts` | Escrita de config atualizada para disco |
| `src/config/config-paths.ts` | Paths de config: `~/.openclaw/config.json`, etc. |
| `src/config/config-env.ts` | Override de config via environment variables |
| `src/config/config-migration.ts` | Migração de configs legacy |

### Tipos de Configuração Específicos

| Arquivo | Propósito |
|---------|-----------|
| `src/config/types.tools.ts` | Config de tools: `ToolLoopDetectionConfig`, `ToolProfileId`, `MediaToolsConfig` |
| `src/config/types.mcp.ts` | Config de servidores MCP |
| `src/config/mcp-config.ts` | Parsing de config MCP |
| `src/config/sessions/session-config.ts` | Config de sessão: dmScope, isolation, TTL |
| `src/config/sessions/session-store-config.ts` | Config do session store: tipo (jsonl, sqlite), path |
| `src/config/sessions/session-routing-config.ts` | Config de roteamento de sessões |

### Estrutura da Config

```json5
{
  // Modelo padrão
  "model": "sonnet-4.6",
  
  // Providers de LLM
  "providers": {
    "anthropic": { "apiKey": "sk-..." },
    "openai": { "apiKey": "sk-..." }
  },
  
  // Canais de messaging
  "channels": {
    "telegram": { "botToken": "..." },
    "discord": { "botToken": "..." }
  },
  
  // Sessões
  "session": {
    "dmScope": "per-channel-peer",
    "store": "jsonl"
  },
  
  // Tools
  "tools": {
    "profile": "full",
    "exec": { "enabled": true }
  },
  
  // Skills
  "skills": {
    "dirs": ["~/.agents/skills"],
    "entries": {}
  },
  
  // Hooks
  "hooks": {
    "internal": { "entries": {} }
  },
  
  // MCP servers
  "mcp": {
    "servers": {}
  },
  
  // Plugins
  "plugins": {
    "entries": {}
  },
  
  // Gateway
  "gateway": {
    "mode": "local",
    "port": 18789
  }
}
```

## 2. Plugin System

### Lifecycle de um Plugin

```
1. DISCOVERY    → Plugin encontrado (workspace, npm, bundled)
2. MANIFEST     → openclaw.plugin.json lido e validado
3. REGISTRATION → Plugin registrado no registry
4. LOADING      → Módulo TypeScript carregado dinamicamente
5. ACTIVATION   → Plugin ativado: registra tools, providers, channels
6. RUNTIME      → Plugin responde a chamadas do core
```

### Arquivos do Plugin System

| Arquivo | Propósito |
|---------|-----------|
| `src/plugins/loader.ts` | **Loader principal** - Carrega plugins do registry, importa módulos dinamicamente |
| `src/plugins/discovery.ts` | Discovery de plugins: busca em workspace, npm global, bundled dirs |
| `src/plugins/manifest.ts` | Parse e validação de `openclaw.plugin.json` |
| `src/plugins/registry.ts` | **Registry central** - Mapa de plugins carregados, lookup por ID |
| `src/plugins/types.ts` | Tipos: `PluginEntry`, `PluginManifest`, `PluginCapability`, `PluginState` |
| `src/plugins/config-state.ts` | Normalização de config de plugins |
| `src/plugins/tools.ts` | `resolvePluginTools()` - Resolve tools fornecidas por plugins |
| `src/plugins/contracts/registry.ts` | Contratos de registro de plugins |
| `src/plugins/public-artifacts.ts` | Artefatos públicos de plugins |

### Plugin SDK (Contrato Público)

| Arquivo | Propósito |
|---------|-----------|
| `src/plugin-sdk/core.ts` | **Core do SDK** - Tipos e interfaces base que plugins importam |
| `src/plugin-sdk/plugin-entry.ts` | **Entry point** - Interface que plugins implementam |
| `src/plugin-sdk/provider-entry.ts` | Interface para plugins de provider |
| `src/plugin-sdk/provider-auth.ts` | Autenticação de providers |
| `src/plugin-sdk/provider-catalog-shared.ts` | Catálogo de modelos |
| `src/plugin-sdk/provider-model-shared.ts` | Metadata de modelos |
| `src/plugin-sdk/provider-tools.ts` | Helpers de tools para providers |
| `src/plugin-sdk/channel-contract.ts` | Contrato para plugins de canal |
| `src/plugin-sdk/setup-tools.ts` | Setup hooks de tools |
| `src/plugin-sdk/tool-send.ts` | Envio de tools de plugins |

### Manifesto de Plugin

```json
{
  "id": "anthropic",
  "name": "Anthropic",
  "version": "1.0.0",
  "description": "Anthropic Claude provider",
  "openclaw": {
    "install": {
      "npmSpec": "@openclaw/anthropic"
    },
    "channel": {
      "id": "anthropic"
    },
    "capabilities": ["provider", "tools"],
    "entrypoint": "src/index.ts"
  }
}
```

### Tipos de Plugin

| Tipo | Propósito | Exemplo |
|------|-----------|---------|
| **Provider** | Fornece acesso a LLMs | anthropic, openai, google-gemini |
| **Channel** | Adiciona canal de messaging | matrix, zalo, voice-call |
| **Tool** | Adiciona tools ao agente | browser, diagnostics |
| **Memory** | Adiciona backend de memória | memory-lancedb |
| **Skill** | Fornece skills adicionais | skill-creator |

## 3. Secrets System

| Arquivo | Propósito |
|---------|-----------|
| `src/secrets/secret-store.ts` | Store de secrets: lê/escreve secrets encriptados |
| `src/secrets/secret-ref.ts` | `SecretRef` - Referência a um secret armazenado (não expõe o valor) |
| `src/secrets/secret-resolution.ts` | Resolução de secrets: de config, env, store |
| `src/secrets/encryption.ts` | Encriptação/decriptação de secrets |

## 4. Bootstrap System

| Arquivo | Propósito |
|---------|-----------|
| `src/bootstrap/bootstrap.ts` | **Inicialização** - Sequência de boot: load config → discover plugins → load plugins → start gateway |
| `src/bootstrap/bootstrap-steps.ts` | Steps individuais de bootstrap |
| `src/bootstrap/bootstrap-validation.ts` | Validação pós-bootstrap |

### Sequência de Boot

```
1. Load config from ~/.openclaw/config.json
2. Validate config with Zod schema
3. Resolve secrets (API keys, tokens)
4. Discover plugins (workspace, npm, bundled)
5. Load plugin manifests
6. Register plugins in registry
7. Activate plugins (tools, providers, channels)
8. Load skills from workspace
9. Load hooks and register event handlers
10. Initialize session stores
11. Start gateway HTTP server
12. Fire gateway:startup hooks
13. Start cron service
14. Ready!
```

## Interações entre Arquivos

```
bootstrap/bootstrap.ts
      │
      ├──→ config/config.ts (load config)
      │       │
      │       ├──→ config-schema.ts (validate)
      │       └──→ config-state.ts (set runtime state)
      │
      ├──→ secrets/secret-resolution.ts (resolve API keys)
      │
      ├──→ plugins/discovery.ts (find plugins)
      │       │
      │       ├──→ plugins/manifest.ts (parse manifests)
      │       └──→ plugins/registry.ts (register)
      │
      ├──→ plugins/loader.ts (load plugin modules)
      │       │
      │       ├──→ plugins/tools.ts (register tools)
      │       ├──→ plugin-sdk/provider-entry.ts (register providers)
      │       └──→ plugin-sdk/channel-contract.ts (register channels)
      │
      ├──→ agents/skills/workspace.ts (load skills)
      │
      ├──→ hooks/loader.ts (load hooks)
      │
      ├──→ sessions/ (init session stores)
      │
      └──→ gateway/ (start HTTP server)
```

## Mapeamento para Go + GORM

### Structs

```go
// Config
type AppConfig struct {
    Model      string
    Providers  map[string]ProviderConfig
    Channels   map[string]ChannelConfig
    Session    SessionConfig
    Tools      ToolsConfig
    Skills     SkillsConfig
    Hooks      HooksConfig
    MCP        MCPConfig
    Plugins    PluginsConfig
    Gateway    GatewayConfig
}

// Plugin
type Plugin struct {
    gorm.Model
    PluginID     string `gorm:"uniqueIndex"`
    Name         string
    Version      string
    Description  string
    Capabilities datatypes.JSON // []string
    Entrypoint   string
    Source       string // bundled, workspace, npm
    Enabled      bool
    Config       datatypes.JSON
}

// Plugin Registry (in-memory)
type PluginRegistry struct {
    mu       sync.RWMutex
    plugins  map[string]*LoadedPlugin
}

type LoadedPlugin struct {
    Manifest  PluginManifest
    Provider  LLMProvider     // if provider plugin
    Tools     []Tool          // if tool plugin
    Channel   ChannelHandler  // if channel plugin
}

// Secret
type Secret struct {
    gorm.Model
    Key           string `gorm:"uniqueIndex"`
    EncryptedValue []byte
    Source        string // config, env, store
}
```

### Tabelas GORM

```go
func AutoMigrate(db *gorm.DB) {
    db.AutoMigrate(
        &Plugin{},
        &Secret{},
        &ConfigSnapshot{}, // versioned config snapshots
    )
}
```
