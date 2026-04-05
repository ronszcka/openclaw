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

## 5. Config Processing Pipeline (Detalhado)

A config passa por múltiplas camadas de processamento antes de ficar disponível:

| Arquivo | Propósito |
|---------|-----------|
| `src/config/io.ts` | **I/O principal** - `loadConfig()`, `readConfigFileSnapshot()`, `writeConfigFile()`, `createConfigIO()`. Escreve `config-audit.jsonl` e `config-health.json` |
| `src/config/env-substitution.ts` | Resolve `${VAR}` env vars na config |
| `src/config/env-preserve.ts` | Preserva referências originais de env vars |
| `src/config/env-vars.ts` | Aplica overrides via `OPENCLAW_*` env vars |
| `src/config/includes.ts` | Suporte a file inclusion com detecção de circular |
| `src/config/merge-patch.ts` | JSON Patch merging (ex: `/agents/` rebase) |
| `src/config/materialize.ts` | Materializa source → runtime config |
| `src/config/runtime-overrides.ts` | Overrides via CLI |
| `src/config/legacy.ts` | Detecta padrões deprecated |
| `src/config/zod-schema.ts` | Schema Zod de validação (~266 linhas) |
| `src/config/validation.ts` | **Engine de validação** - `validateConfigObjectWithPlugins()`, `validateConfigObjectRawWithPlugins()`. Valida contra Zod, detecta legacy, valida channel schemas e secret refs |
| `src/config/runtime-snapshot.ts` | **Snapshot global** - `setRuntimeConfigSnapshot()`, `getRuntimeConfigSnapshot()`, `setRuntimeConfigSnapshotRefreshHandler()` |
| `src/config/mutate.ts` | **Mutação segura** - `mutateConfigFile()` com read-modify-write, conflict detection via `previousHash` (optimistic locking), `ConfigMutationConflictError` |
| `src/config/backup-rotation.ts` | Rotação de backups na escrita |

### Pipeline de Processamento

```
openclaw.json (disco)
      │
      ▼ readConfigFileSnapshot()
      │
      ├──→ includes.ts (resolve file inclusions)
      ├──→ env-substitution.ts (resolve ${VAR})
      ├──→ env-vars.ts (apply OPENCLAW_* overrides)
      ├──→ runtime-overrides.ts (CLI overrides)
      │
      ▼ validation.ts
      │
      ├──→ zod-schema.ts (schema validation)
      ├──→ legacy.ts (detect deprecated patterns)
      │
      ▼ materialize.ts
      │
      └──→ runtime-snapshot.ts (cache global)
```

## 6. Plugin Discovery e Loading (Detalhado)

| Arquivo | Propósito |
|---------|-----------|
| `src/plugins/discovery.ts` | `discoverOpenClawPlugins()` - Scan filesystem, cache 1s, retorna `PluginCandidate[]` |
| `src/plugins/roots.ts` | Resolução de diretórios raiz de plugins. Respeita `OPENCLAW_PLUGIN_LOAD_PATHS` |
| `src/plugins/manifest.ts` | Schema de `openclaw.plugin.json`: id, configSchema, providers, channels, skills, version |
| `src/plugins/manifest-registry.ts` | `loadPluginManifestRegistry()` - Registry in-memory de manifests (antes do runtime load) |
| `src/plugins/bundled-plugin-metadata.ts` | Registro de plugins bundled, scan de dirs, cache |
| `src/plugins/bundled-plugin-scan.ts` | Scanner de plugins bundled |
| `src/plugins/bundle-manifest.ts` | Manifest de bundle IDE (`.claude-plugin/plugin.json`, `.cursor-plugin/plugin.json`) |

### Loading Options

```typescript
loadOpenClawPlugins({
    mode: "full" | "validate",     // Carrega tudo ou só manifest
    activate: boolean,              // Ativa plugins (chama register())
    loadModules: boolean,           // Importa módulos runtime
    onlyPluginIds: string[],        // Loading seletivo
    pluginSdkResolution: "...",     // Preferência de alias SDK
})
```

## 7. Plugin Activation State

| Arquivo | Propósito |
|---------|-----------|
| `src/plugins/config-state.ts` | `resolveEffectivePluginActivationState()` - `PluginActivationSource`: disabled, explicit, auto, default. `normalizePluginId()` para aliases legacy |
| `src/config/plugin-auto-enable.ts` | **Auto-enable** baseado em uso da config: analisa providers, models, auth methods referenciados e auto-habilita plugins que os possuem |
| `src/plugins/bundled-compat.ts` | Compatibilidade backwards: allowlist mode, enablement mode, vitest compat |

## 8. Plugin Runtime e Registry

| Arquivo | Propósito |
|---------|-----------|
| `src/plugins/registry.ts` | **PluginRegistry** - Holds all loaded/activated plugins. `PluginRecord` com status/error. Contém provider registrations, channel registrations, CLI commands, HTTP routes, hooks. Rastreia diagnostics e warnings |
| `src/plugins/registry-empty.ts` | Placeholder vazio para startup |
| `src/plugins/runtime.ts` | **Singleton global** `PLUGIN_REGISTRY_STATE` em `globalThis`. `setActivePluginRegistry()`, `getActivePluginRegistry()`. Synca para HTTP route e channel surfaces. Version tracking |
| `src/plugins/runtime/index.ts` | `PluginRuntime` facade - lazy-loads TTS, media, model auth |
| `src/plugins/runtime/types.ts` | Contrato `PluginRuntime`: config, agent, channel, system, media, TTS |
| `src/plugins/api-builder.ts` | Builds `OpenClawPluginApi` passado para `register()` |
| `src/plugins/command-registration.ts` | `registerCommand()` com schema validation e conflict detection |
| `src/plugins/captured-registration.ts` | Captura registrations durante loading para diagnostics |
| `src/plugins/interactive-registry.ts` | Registro de interactive handlers |

## 9. Secrets e Auth (Detalhado)

| Arquivo | Propósito |
|---------|-----------|
| `src/secrets/runtime.ts` | **Runtime principal** - `prepareSecretsRuntimeSnapshot()`, `getActiveRuntimeSecretsSnapshot()`. Cache com refresh context |
| `src/secrets/resolve.ts` | `resolveSecretRefValues()` - Resolve `SecretRef` para valores reais via auth profile stores e provider env vars |
| `src/secrets/target-registry.ts` | Mapeamento de SecretRef targets. Query interface para encontrar targets |
| `src/secrets/provider-env-vars.ts` | Mapeamento de provider IDs para env var names |
| `src/secrets/runtime-config-collectors.ts` | Entry point para coletores config-based |
| `src/secrets/runtime-config-collectors-channels.ts` | Secrets de canais |
| `src/secrets/runtime-config-collectors-plugins.ts` | Secrets de plugins |
| `src/secrets/runtime-config-collectors-core.ts` | Secrets core (models, gateway) |
| `src/secrets/runtime-config-collectors-tts.ts` | Secrets de TTS |
| `src/secrets/runtime-auth-collectors.ts` | Coletores de auth store |
| `src/secrets/configure.ts` | Configuração interativa de secrets |
| `src/secrets/apply.ts` | Aplicação de secrets do store |
| `src/secrets/audit.ts` | Auditoria de segurança |
| `src/agents/auth-profiles.ts` | Storage de auth profiles por agente (JSONC files em state dir) |

## 10. State Management (3 Camadas)

```
┌─────────────────────────────────────────────┐
│  CONFIG STATE                                │
│  Source: openclaw.json                       │
│  Snapshot: runtime-snapshot.ts               │
│  Lifecycle: Load once, refresh on watchers   │
├─────────────────────────────────────────────┤
│  PLUGIN REGISTRY STATE                       │
│  Source: Loaded & activated plugins          │
│  Snapshot: runtime.ts (global singleton)     │
│  Lifecycle: Built on loading, synced         │
├─────────────────────────────────────────────┤
│  SECRETS STATE                               │
│  Source: Auth stores + config refs           │
│  Snapshot: runtime.ts cached                 │
│  Lifecycle: Prepared on startup, refreshed   │
└─────────────────────────────────────────────┘

Todos usam global singleton para suportar:
- Gateway long-lived com hot-reloading
- CLI short-lived com single load
- Refresh handlers para config watchers
- Version tracking para change detection
```

## 11. Plugin Lifecycle Completo

```
1. DISCOVERY
   discovery.ts → PluginCandidate[]
   (scan filesystem + loadPaths + bundled dirs)

2. MANIFEST LOADING
   manifest.ts → PluginManifest (static, no JS loaded)
   manifest-registry.ts → in-memory registry

3. CONFIG VALIDATION
   validation.ts usa manifest registry (sem JS de plugin)
   Valida channel configs, secret refs, allowed values

4. ACTIVATION DECISION
   config-state.ts + plugin-auto-enable.ts
   → NormalizedPluginsConfig
   → PluginActivationState per plugin
   (disabled | explicit | auto | default)

5. METADATA LOADING
   bundled-plugin-metadata.ts → BundledPluginMetadata
   Prepara paths de módulos runtime

6. RUNTIME LOADING
   loader.ts importa módulo do plugin via jiti
   Cria PluginRegistry com plugins carregados

7. REGISTRATION
   Plugin chama api.register({ channels, providers, tools, ... })
   captured-registration.ts grava todas as registrations

8. ACTIVATION
   setActivePluginRegistry() [runtime.ts]
   Torna capabilities do plugin disponíveis para o core

9. EXECUTION
   Plugin hooks, tools, channels, providers disponíveis no runtime
```

## 12. Entry Point e Boot Sequence

| Arquivo | Propósito |
|---------|-----------|
| `src/entry.ts` | **Entry point CLI** - Handles respawning, env setup, routes para main CLI |
| `src/entry.respawn.ts` | Lógica de respawn: setup de Node.js env vars (TLS, experimental warnings) |
| `src/bootstrap/node-startup-env.ts` | Resolve Node.js TLS env (NODE_EXTRA_CA_CERTS, NODE_USE_SYSTEM_CA) |
| `src/bootstrap/node-extra-ca-certs.ts` | Detecção platform-specific de CA bundle |
| `src/cli/run-main.ts` | `runCli()` → `tryRouteCli()` |
| `src/cli/route.ts` | Command routing |

### Boot Sequence Detalhado

```
entry.ts
  ├─ buildCliRespawnPlan() [entry.respawn.ts]
  │   └─ resolveNodeStartupTlsEnvironment() [bootstrap/node-startup-env.ts]
  │
  ├─ parseCliProfileArgs() [cli/profile.ts]
  │
  └─ runCli() [cli/run-main.ts]
      │
      ├─ loadConfig() [config/io.ts]
      │   ├─ Resolve path via paths.ts
      │   ├─ Read & parse openclaw.json
      │   ├─ Resolve includes, env substitution
      │   ├─ Validate against zod-schema.ts
      │   ├─ Materialize runtime config
      │   └─ Cache in runtime-snapshot.ts
      │
      ├─ applyPluginAutoEnable() [config/plugin-auto-enable.ts]
      │
      ├─ loadOpenClawPlugins() [plugins/loader.ts]
      │   ├─ discoverOpenClawPlugins() [plugins/discovery.ts]
      │   ├─ loadPluginManifestRegistry() [plugins/manifest-registry.ts]
      │   ├─ normalizePluginsConfig() [plugins/config-state.ts]
      │   ├─ resolveEffectivePluginActivationState()
      │   ├─ Load plugin modules via jiti
      │   ├─ Call plugin.register() functions
      │   └─ setActivePluginRegistry() [plugins/runtime.ts]
      │
      ├─ prepareSecretsRuntimeSnapshot() [secrets/runtime.ts]
      │   ├─ collectConfigAssignments() [secrets/runtime-config-collectors.ts]
      │   ├─ collectAuthStoreAssignments() [secrets/runtime-auth-collectors.ts]
      │   ├─ resolveSecretRefValues() [secrets/resolve.ts]
      │   └─ Return resolved config + auth stores
      │
      └─ Execute command (gateway run, agent, etc.)
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
