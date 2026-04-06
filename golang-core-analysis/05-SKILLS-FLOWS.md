# 05 - Skills, Hooks, Flows, Cron & Tasks

## Resumo

O OpenClaw possui 5 subsistemas de extensibilidade e automação:
- **Skills**: Módulos de conhecimento/capacidade carregados no prompt do agente
- **Hooks**: Sistema event-driven para extensibilidade em pontos-chave do lifecycle
- **Flows**: Wizards interativos de configuração
- **Cron**: Agendamento de jobs periódicos
- **Tasks**: Registry de tarefas distribuídas (subagents, ACP, CLI)

## 1. Skills System

### O que são Skills

Skills são arquivos `SKILL.md` que contêm instruções, conhecimento, e metadados. São carregados no prompt do agente como contexto adicional. Exemplo: skill de "weather" ensina o agente a consultar clima.

### Fluxo

```
~/.agents/skills/weather/SKILL.md
        │
        ▼
workspace.ts (discovery)
        │
        ▼
frontmatter.ts (parse metadata)
        │
        ▼
config.ts (check eligibility)
        │
        ▼
Agent prompt (XML formatted)
```

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `src/agents/skills/workspace.ts` | **Loader principal** (~27KB). Carrega skills de múltiplas fontes (workspace, config, plugins). Filtros: agent-level, config-level, bundled allowlists. Limites: max 150 no prompt, 30KB prompt size, 256KB file size |
| `src/agents/skills/local-loader.ts` | Carrega skills individuais de diretórios. Lê `SKILL.md` com boundary checks de segurança |
| `src/agents/skills/frontmatter.ts` | Parse de frontmatter: name, description, invocation policy, install specs (brew, node, npm, go, uv, download) |
| `src/agents/skills/plugin-skills.ts` | Integração com plugins: resolve skill dirs de plugins carregados |
| `src/agents/skills/bundled-dir.ts` | Discovery de skills bundled da distribuição |
| `src/agents/skills/tools-dir.ts` | Discovery de skills de tools |
| `src/agents/skills/config.ts` | `resolveSkillConfig()`, `shouldIncludeSkill()`, `isBundledSkillAllowed()` |
| `src/agents/skills/agent-filter.ts` | Filtros de skills por agente |
| `src/agents/skills/filter.ts` | Normalização de filtros |
| `src/agents/skills/refresh.ts` | File watcher (Chokidar) para live reload de skills |
| `src/agents/skills/refresh-state.ts` | Estado de refresh: snapshot versions, change listeners |
| `src/agents/skills/env-overrides.ts` | Mapeamento de env requirements para config/process |
| `src/agents/skills/serialize.ts` | Task queue serializada para operações de skill |
| `src/agents/skills/skill-contract.ts` | Tipo `Skill` e `formatSkillsForPrompt()` (XML) |
| `src/agents/skills/types.ts` | `SkillEntry`, `OpenClawSkillMetadata`, `SkillInvocationPolicy`, `SkillExposure`, `SkillSnapshot` |

### Formato do Skill no Prompt

```xml
<available_skills>
  <skill name="weather" description="Get weather forecasts" user_invocable="true">
    Content of SKILL.md...
  </skill>
  <skill name="github" description="GitHub operations" user_invocable="true">
    Content of SKILL.md...
  </skill>
</available_skills>
```

## 2. Hooks System

### O que são Hooks

Hooks são handlers event-driven que executam em pontos-chave do lifecycle: boot, mensagem recebida/enviada, sessão iniciada/encerrada, etc.

### Tipos de Eventos

| Evento | Quando dispara |
|--------|----------------|
| `agent:bootstrap` | Inicialização do agente com bootstrap files |
| `gateway:startup` | Gateway inicia |
| `message:received` | Mensagem inbound |
| `message:sent` | Mensagem outbound |
| `message:transcribed` | Áudio transcrito |
| `message:preprocessed` | Mensagem pré-processada com enrichments |
| `session:end` | Sessão encerrada |

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `src/hooks/internal-hooks.ts` | **Tipos de evento** (~150+ linhas): `AgentBootstrapHookEvent`, `GatewayStartupHookEvent`, `MessageReceivedHookEvent`, `MessageSentHookEvent`, etc. |
| `src/hooks/hooks.ts` | Re-exports públicos |
| `src/hooks/loader.ts` | **Carregamento dinâmico** (~150+ linhas): carrega hooks de bundled, managed, workspace dirs. Import dinâmico de módulos handler |
| `src/hooks/workspace.ts` | Discovery de hooks: lê `HOOK.md` files e package.json |
| `src/hooks/frontmatter.ts` | Parse de frontmatter de HOOK.md: events, export name, invocation policy |
| `src/hooks/install.ts` | Instalação de hooks |
| `src/hooks/config.ts` | Configuração e eligibilidade de hooks |
| `src/hooks/policy.ts` | Policy de eligibilidade |
| `src/hooks/plugin-hooks.ts` | Discovery e registro de hooks de plugins |
| `src/hooks/message-hook-mappers.ts` | Mapeamento de eventos de mensagem para handlers |
| `src/hooks/message-hooks.ts` | Triggering de hooks de mensagem |
| `src/hooks/module-loader.ts` | Import dinâmico e resolução de exports |
| `src/hooks/fire-and-forget.ts` | Execução fire-and-forget (non-blocking) |

### Hooks Bundled

| Arquivo | Propósito |
|---------|-----------|
| `src/hooks/bundled/boot-md/handler.ts` | Carrega bootstrap markdown no startup |
| `src/hooks/bundled/session-memory/handler.ts` | **Captura transcripts de sessão** para o sistema de memória |
| `src/hooks/bundled/bootstrap-extra-files/handler.ts` | Bootstrap de files extras |
| `src/hooks/bundled/command-logger/handler.ts` | Logging de execução de comandos |

## 3. Flows System

### O que são Flows

Flows são wizards interativos para configuração de canais, seleção de modelos, e health checks.

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `src/flows/types.ts` | Tipos: `FlowContributionKind` (channel, core, provider, search), `FlowContributionSurface` (auth-choice, health, model-picker, setup), `FlowOption`, `FlowContribution` |
| `src/flows/channel-setup.ts` | **Setup de canal** (~100+ linhas): coordena wizard de configuração, instalação de plugins, post-write hooks |
| `src/flows/channel-setup.prompts.ts` | Prompts do wizard de canal |
| `src/flows/channel-setup.status.ts` | Status e seleção de canal |
| `src/flows/provider-flow.ts` | Flow de seleção de provider/modelo |
| `src/flows/model-picker.ts` | Integração com model picker |
| `src/flows/doctor-health.ts` | Flow de health check/diagnóstico |
| `src/flows/doctor-health-contributions.ts` | Contribuições de status de saúde |
| `src/flows/search-setup.ts` | Flow de setup de search provider |

## 4. Cron System

### O que é o Cron

Sistema de agendamento de jobs: executa agente em horários programados com delivery configurável.

### Tipos de Schedule

```
- "at": Horário absoluto (ISO date/time)
- "every": Intervalo (5m, 1h, 1d)
- "cron": Expressão cron (0 9 * * 1-5)
```

### Delivery Targets

```
- channel: Envia para canal específico
- DM: Envia como mensagem direta
- webhook: POST para URL
- thread: Responde em thread
```

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `src/cron/types.ts` | **Tipos** (~150+ linhas): `CronSchedule`, `CronSessionTarget`, `CronWakeMode`, `CronDelivery`, `CronPayload`, `CronJobState`, `CronJob` |
| `src/cron/service.ts` | **API do CronService**: `start()`, `stop()`, `status()`, `list()`, `add()`, `update()`, `remove()`, `run()`, `enqueueRun()`, `wake()` |
| `src/cron/service/state.ts` | Gerenciamento de estado do service |
| `src/cron/service/ops.ts` | Operações: list, add, update, remove, run |
| `src/cron/service/store.ts` | Store abstrato de jobs |
| `src/cron/service/jobs.ts` | Operações de gerenciamento de jobs |
| `src/cron/service/timer.ts` | Lógica de scheduling/timer |
| `src/cron/service/timeout-policy.ts` | Políticas de timeout |
| `src/cron/service/locked.ts` | Locking de concorrência |
| `src/cron/store.ts` | Persistência SQLite de jobs |
| `src/cron/schedule.ts` | Parse e computação de schedules |
| `src/cron/parse.ts` | Parse de tempos absolutos |
| `src/cron/normalize.ts` | Normalização de expressões |

### Execução Isolada

| Arquivo | Propósito |
|---------|-----------|
| `src/cron/isolated-agent.ts` | **Runner isolado**: executa cron jobs em sessões isoladas, delivery e followup |
| `src/cron/isolated-agent/run.ts` | Execução do agent turn com message e model |
| `src/cron/isolated-agent/run-executor.ts` | Executor wrapper com lifecycle e failover |
| `src/cron/isolated-agent/run-config.ts` | Build de config de execução |
| `src/cron/isolated-agent/session.ts` | Criação de sessões isoladas |
| `src/cron/isolated-agent/delivery-dispatch.ts` | Dispatch de delivery para targets |
| `src/cron/isolated-agent/delivery-target.ts` | Resolução de delivery targets |
| `src/cron/isolated-agent/model-selection.ts` | Seleção de modelo com fallback |
| `src/cron/isolated-agent/skills-snapshot.ts` | Snapshot de skills para o job |

### Delivery

| Arquivo | Propósito |
|---------|-----------|
| `src/cron/delivery.ts` | Execução de delivery para channels, webhooks |
| `src/cron/delivery-plan.ts` | Planejamento de delivery multi-target |
| `src/cron/webhook-url.ts` | Build de URLs de webhook |

## 5. Tasks System

### O que são Tasks

Registry de tarefas para trabalho distribuído: subagents, ACP, CLI, cron.

### Status de Task

```
queued → running → succeeded | failed | timed_out | cancelled | lost
```

### Arquivos

| Arquivo | Propósito |
|---------|-----------|
| `src/tasks/task-registry.types.ts` | **Tipos**: `TaskStatus`, `TaskRuntime` (subagent, acp, cli, cron), `TaskScopeKind` (session, system), `TaskRecord`, `TaskDeliveryStatus`, `TaskNotifyPolicy` |
| `src/tasks/task-registry.ts` | **API principal**: create, update, query task records |
| `src/tasks/task-registry.store.ts` | Interface de store abstrato |
| `src/tasks/task-registry.store.sqlite.ts` | Implementação SQLite |
| `src/tasks/task-registry.maintenance.ts` | Cleanup e manutenção |
| `src/tasks/task-registry.reconcile.ts` | Reconciliação de consistência |
| `src/tasks/task-registry.audit.ts` | Audit logging |
| `src/tasks/task-registry.summary.ts` | Estatísticas |
| `src/tasks/task-executor.ts` | **Coordenação de execução**: cria runs, linka tasks a flows, delivery notifications |
| `src/tasks/task-executor-policy.ts` | Políticas de execução |
| `src/tasks/task-owner-access.ts` | Controle de acesso por owner |

### Task Flows (Workflows Parent-Child)

| Arquivo | Propósito |
|---------|-----------|
| `src/tasks/task-flow-registry.ts` | **API de flows**: cria e gerencia task flow records (parent-child relationships) |
| `src/tasks/task-flow-registry.types.ts` | Tipos de flow |
| `src/tasks/task-flow-registry.store.ts` | Store abstrato de flows |
| `src/tasks/task-flow-registry.store.sqlite.ts` | SQLite |
| `src/tasks/task-flow-registry.maintenance.ts` | Cleanup |
| `src/tasks/task-flow-owner-access.ts` | Controle de acesso |

## Interações entre Sistemas

```
SKILLS ──→ Carregadas no prompt do agent
              │
              ├──→ Agent pode invocar skills via tool call
              └──→ Agent tem conhecimento das skills disponíveis

HOOKS ──→ Disparam em eventos do lifecycle
              │
              ├──→ session-memory hook → alimenta LTM
              ├──→ bootstrap hook → enriquece prompt
              └──→ message hooks → preprocessing/postprocessing

FLOWS ──→ Wizards de configuração
              │
              └──→ Configuram channels, providers, etc.

CRON ──→ Jobs agendados
              │
              ├──→ Cria sessão isolada
              ├──→ Executa agent turn
              ├──→ Delivery para channels/webhooks
              └──→ Registra no task registry

TASKS ──→ Registry centralizado
              │
              ├──→ Subagents registram tasks
              ├──→ Cron jobs registram tasks
              └──→ UI consulta status
```

## Mapeamento para Go + GORM

### Structs

```go
// Skills
type Skill struct {
    gorm.Model
    Name            string `gorm:"uniqueIndex"`
    Description     string
    Content         string // SKILL.md content
    Source          string // bundled, workspace, plugin
    InvocationPolicy string
    Enabled         bool
    Requirements    datatypes.JSON // bins, env, config
}

// Hooks
type Hook struct {
    gorm.Model
    Name      string `gorm:"uniqueIndex"`
    Events    datatypes.JSON // []string of event types
    Handler   string // module path
    Source    string // bundled, managed, workspace
    Enabled   bool
}

// Cron Jobs
type CronJob struct {
    gorm.Model
    Name          string
    ScheduleType  string // at, every, cron
    ScheduleExpr  string
    SessionTarget string // main, isolated, current
    Payload       datatypes.JSON
    DeliveryType  string // channel, dm, webhook
    DeliveryTarget string
    Enabled       bool
    LastRunAt     *time.Time
    NextRunAt     *time.Time
    RunCount      int
    ErrorCount    int
    LastError     *string
}

// Tasks
type Task struct {
    gorm.Model
    ExternalID    string `gorm:"uniqueIndex"`
    Runtime       string // subagent, acp, cli, cron
    ScopeKind     string // session, system
    ScopeID       string
    Status        string // queued, running, succeeded, failed, etc.
    NotifyPolicy  string
    ParentFlowID  *uint
    Input         datatypes.JSON
    Output        datatypes.JSON
    Error         *string
    QueuedAt      time.Time
    StartedAt     *time.Time
    CompletedAt   *time.Time
}

// Task Flows
type TaskFlow struct {
    gorm.Model
    Name     string
    Status   string
    Tasks    []Task `gorm:"foreignKey:ParentFlowID"`
    Metadata datatypes.JSON
}
```
