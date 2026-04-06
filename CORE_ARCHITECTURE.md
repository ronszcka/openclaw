# OpenClaw - Arquitetura Core para Clonagem em Golang/GORM

## Visao Geral

Este documento mapeia 100%% dos arquivos que compoem o mecanismo core do OpenClaw.
O objetivo e servir como guia para recriar o engine em Go com GORM.

## Arquitetura em Camadas

O core do OpenClaw segue esta hierarquia:

```
Comando do Usuario
  -> Agent Command (entry point)
    -> Embedded Runner (main loop com retry/failover)
      -> Attempt (execucao de um turno)
        -> Streaming/Subscription (eventos em tempo real)
          -> Handlers (mensagens, tools, lifecycle)
            -> Tool Execution
            -> Compaction (quando context overflow)
            -> Memory Flush
```

---

## 1. AGENTIC LOOP (Loop Principal do Agente)

O coracao do sistema. Um loop `while(true)` que executa turnos de raciocinio ate completar a tarefa.

### 1.1 Entry Point e Orquestracao

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/agent-command.ts` | Entry point principal. Orquestra sessao, modelo, parametros |
| `src/agents/command/attempt-execution.ts` | Coordenacao de alto nivel para rodar agentes embedded |
| `src/agents/command/run-context.ts` | Resolve contexto runtime (workspace, sessao, config) |

### 1.2 Loop Principal (Retry + Failover)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-embedded-runner/run.ts` | **CRITICO** - Loop principal `while(true)`. Gerencia retry, failover entre modelos, compaction em overflow, rotacao de auth profiles |
| `src/agents/pi-embedded-runner/runs.ts` | Singleton global de runs ativos. Queue de mensagens, abort, status de streaming |
| `src/agents/pi-embedded-runner/abort.ts` | Tratamento de sinais de abort e cleanup |

### 1.3 Execucao de Turno Individual (Attempt)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-embedded-runner/run/attempt.ts` | **CRITICO** - Executa um turno. Cria AgentSession, configura tools, system prompt, inicia subscription |
| `src/agents/pi-embedded-runner/run/attempt.stop-reason-recovery.ts` | Recuperacao de stop reasons especiais (tool_use, text_delta) |
| `src/agents/pi-embedded-runner/run/attempt.sessions-yield.ts` | Mecanismo de yield (coroutine-style) entre sessoes |
| `src/agents/pi-embedded-runner/run/attempt.tool-call-normalization.ts` | Normalizacao de tool calls recebidas do modelo |
| `src/agents/pi-embedded-runner/run/attempt.tool-call-argument-repair.ts` | Reparo de argumentos malformados em tool calls |
| `src/agents/pi-embedded-runner/run/attempt.tool-run-context.ts` | Contexto de execucao de tools |
| `src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts` | Helpers para construcao de prompts |
| `src/agents/pi-embedded-runner/run/attempt.thread-helpers.ts` | Helpers para threads |
| `src/agents/pi-embedded-runner/run/attempt.context-engine-helpers.ts` | Helpers do context engine |
| `src/agents/pi-embedded-runner/run/attempt.subscription-cleanup.ts` | Cleanup de subscriptions |
| `src/agents/pi-embedded-runner/run/params.ts` | Tipos e validacao de parametros |
| `src/agents/pi-embedded-runner/run/setup.ts` | Resolve streaming function e modelo para o run |
| `src/agents/pi-embedded-runner/run/helpers.ts` | Utilidades gerais (usage, error metadata, auth state) |
| `src/agents/pi-embedded-runner/run/types.ts` | Type definitions |
| `src/agents/pi-embedded-runner/run/trigger-policy.ts` | Politica de trigger para runs |
| `src/agents/pi-embedded-runner/run/incomplete-turn.ts` | Tratamento de turnos incompletos |
| `src/agents/pi-embedded-runner/run/retry-limit.ts` | Exaustao de limite de retry |
| `src/agents/pi-embedded-runner/run/payloads.ts` | Construcao de payloads para LLM (messages, tools, context) |
| `src/agents/pi-embedded-runner/run/images.ts` | Tratamento de imagens em runs |
| `src/agents/pi-embedded-runner/run/history-image-prune.ts` | Poda de imagens do historico |

### 1.4 Streaming e Subscription (Eventos em Tempo Real)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-embedded-subscribe.ts` | **CRITICO** - Cria subscription ao fluxo de streaming. State machine para texto, tools, thinking, block reply |
| `src/agents/pi-embedded-subscribe.handlers.ts` | **CRITICO** - Factory de event handlers. Roteamento e scheduling de eventos |
| `src/agents/pi-embedded-subscribe.handlers.messages.ts` | Handlers de mensagem: start, update, end. Acumula texto, trata reasoning tags |
| `src/agents/pi-embedded-subscribe.handlers.tools.ts` | Handlers de tool execution: start, update, end. Processa resultados |
| `src/agents/pi-embedded-subscribe.handlers.lifecycle.ts` | Handlers de lifecycle: agent start/end. Emite eventos |
| `src/agents/pi-embedded-subscribe.handlers.compaction.ts` | Handlers de compaction: start/end |
| `src/agents/pi-embedded-subscribe.handlers.compaction.runtime.ts` | Runtime de compaction em handlers |
| `src/agents/pi-embedded-subscribe.handlers.types.ts` | Tipos para handlers |
| `src/agents/pi-embedded-subscribe.tools.ts` | Utilidades de tool no subscription |
| `src/agents/pi-embedded-subscribe.tools.extract.ts` | Extracao de resultados de tools |
| `src/agents/pi-embedded-subscribe.handlers.tools.media.ts` | Tratamento de midia em tool results |
| `src/agents/pi-embedded-subscribe.promise.ts` | Promise handling para subscription |
| `src/agents/pi-embedded-subscribe.raw-stream.ts` | Log de raw stream para debug |

### 1.5 Failover e Recuperacao de Erros

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-embedded-runner/run/failover-policy.ts` | Decide failover: retry mesmo modelo, fallback outro modelo, ou desistir |
| `src/agents/pi-embedded-runner/run/failover-observation.ts` | Logging e observacao de decisoes de failover |
| `src/agents/pi-embedded-runner/run/assistant-failover.ts` | Failover quando assistant retorna erro |
| `src/agents/failover-error.ts` | Classificacao de erros (rate limit, auth, billing, context overflow) |
| `src/agents/pi-embedded-runner/run/auth-controller.ts` | Controlador de autenticacao |
| `src/agents/pi-embedded-runner/run/llm-idle-timeout.ts` | Timeout por inatividade do LLM |

---

## 2. TOOL CALLS (Sistema de Chamada de Ferramentas)

O agente raciocina e decide quais tools usar. O sistema gerencia definicao, permissao, execucao e resultado.

### 2.1 Catalogo e Inventario de Tools

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/tool-catalog.ts` | Perfis de tools (minimal, coding, messaging, full) e secoes core |
| `src/agents/tools-effective-inventory.ts` | Resolucao de inventario efetivo de tools |
| `src/agents/tool-description-presets.ts` | Presets de descricao de tools |
| `src/agents/tool-description-summary.ts` | Geracao de descricoes de tools |
| `src/agents/openclaw-tools.ts` | Factory principal para criar OpenClaw tools |
| `src/agents/openclaw-tools.runtime.ts` | Runtime de gerenciamento de tools |
| `src/agents/pi-tools.ts` | Implementacao principal de pi-tools (coding tools) |
| `src/agents/pi-tools.types.ts` | Tipos core de tools |
| `src/agents/pi-tools.params.ts` | Tratamento de parametros de tools |
| `src/agents/pi-tools.schema.ts` | Normalizacao e limpeza de schemas de tools |

### 2.2 Implementacoes de Tools Built-in

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/bash-tools.ts` | Tool de execucao Bash (exec, process) |
| `src/agents/bash-tools.exec.ts` | Funcionalidade de exec |
| `src/agents/bash-tools.exec-runtime.ts` | Runtime de execucao bash |
| `src/agents/bash-tools.exec-types.ts` | Tipos de bash tool |
| `src/agents/bash-tools.process.ts` | Gerenciamento de processos |
| `src/agents/bash-tools.shared.ts` | Utilidades compartilhadas de bash |
| `src/agents/bash-tools.exec-host-node.ts` | Execucao bash no host (Node) |
| `src/agents/bash-tools.exec-host-gateway.ts` | Execucao bash via gateway |
| `src/agents/bash-tools.exec-host-shared.ts` | Utilidades compartilhadas de host |
| `src/agents/bash-process-registry.ts` | Registro de processos bash |
| `src/agents/pi-tools.read.ts` | Tool de leitura de arquivos |
| `src/agents/pi-tools.host-edit.ts` | Tool de edicao de arquivos |
| `src/agents/pi-tools.abort.ts` | Abort de tools |
| `src/agents/channel-tools.ts` | Tools especificos de canal |

### 2.3 Tools Individuais (diretorio tools/)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/tools/common.ts` | Utilidades e tipos comuns |
| `src/agents/tools/web-tools.ts` | Tools de web fetch/search |
| `src/agents/tools/web-fetch.ts` | Implementacao de web fetch |
| `src/agents/tools/web-search.ts` | Implementacao de web search |
| `src/agents/tools/image-tool.ts` | Manipulacao de imagem |
| `src/agents/tools/image-generate-tool.ts` | Geracao de imagem |
| `src/agents/tools/video-generate-tool.ts` | Geracao de video |
| `src/agents/tools/pdf-tool.ts` | Tratamento de PDF |
| `src/agents/tools/canvas-tool.ts` | Tool de canvas |
| `src/agents/tools/message-tool.ts` | Envio de mensagens |
| `src/agents/tools/cron-tool.ts` | Agendamento cron |
| `src/agents/tools/tts-tool.ts` | Text-to-speech |
| `src/agents/tools/gateway-tool.ts` | Tool de gateway |
| `src/agents/tools/nodes-tool.ts` | Acoes em nodes (canvas) |
| `src/agents/tools/sessions-spawn-tool.ts` | Spawn de subagent |
| `src/agents/tools/sessions-list-tool.ts` | Listar sessoes |
| `src/agents/tools/sessions-history-tool.ts` | Historico de sessoes |
| `src/agents/tools/sessions-send-tool.ts` | Enviar para sessao |
| `src/agents/tools/sessions-yield-tool.ts` | Yield de controle |
| `src/agents/tools/session-status-tool.ts` | Status de sessao |
| `src/agents/tools/agents-list-tool.ts` | Listar agentes |
| `src/agents/tools/subagents-tool.ts` | Gerenciamento de subagentes |
| `src/agents/tools/update-plan-tool.ts` | Atualizacao de plano |

### 2.4 Politica e Permissoes de Tools

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/tool-policy.ts` | Enforcement principal de politica de tools |
| `src/agents/tool-policy-pipeline.ts` | Pipeline composavel de politicas |
| `src/agents/tool-policy-match.ts` | Logica de matching de politica |
| `src/agents/tool-policy-shared.ts` | Utilidades compartilhadas de politica |
| `src/agents/tool-policy.conformance.ts` | Teste de conformidade de politica |
| `src/agents/sandbox-tool-policy.ts` | Politica de tools em sandbox |
| `src/agents/sandbox/tool-policy.ts` | Implementacao sandbox de tool policy |
| `src/agents/tool-fs-policy.ts` | Politica de file system |
| `src/agents/pi-tools.policy.ts` | Politica de pi-tools |

### 2.5 Schema e Adaptadores de Tools

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-tool-definition-adapter.ts` | Adaptador de definicoes para diferentes formatos de modelo |
| `src/agents/openai-tool-schema.ts` | Adaptacao de schema para OpenAI |
| `src/agents/schema/typebox.ts` | Utilidades TypeBox para schemas |
| `src/agents/schema/clean-for-gemini.ts` | Limpeza de schema para Gemini |
| `src/agents/schema/clean-for-xai.ts` | Limpeza de schema para XAI |
| `src/agents/model-tool-support.ts` | Validacao de suporte de tools por modelo/provider |
| `src/agents/anthropic-family-tool-payload-compat.ts` | Compatibilidade de payloads Anthropic |

### 2.6 Resultado e Truncamento de Tools

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-embedded-runner/tool-result-truncation.ts` | Truncamento de resultados de tools |
| `src/agents/pi-embedded-runner/tool-result-context-guard.ts` | Guard de contexto para resultados |
| `src/agents/pi-embedded-runner/tool-result-char-estimator.ts` | Estimativa de caracteres de resultados |
| `src/agents/session-tool-result-guard.ts` | Guard de resultados por sessao |
| `src/agents/session-tool-result-guard-wrapper.ts` | Wrapper de guard de resultados |
| `src/agents/session-tool-result-state.ts` | Tracking de estado de resultados |
| `src/agents/tool-error-summary.ts` | Sumarizacao de erros de tools |
| `src/agents/tool-call-id.ts` | Gerenciamento de IDs de tool calls |
| `src/agents/tool-images.ts` | Gerenciamento de imagens em tools |

### 2.7 Loop Detection e Mutation

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/tool-loop-detection.ts` | Detecta quando agente entra em loop de tools |
| `src/agents/tool-mutation.ts` | Tracking de estado de mutacao de tool calls |

### 2.8 Before/After Tool Call Hooks

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-tools.before-tool-call.ts` | Hook antes de tool call |
| `src/agents/pi-tools.before-tool-call.runtime.ts` | Runtime do hook before-call |
| `src/agents/pi-tool-definition-adapter.after-tool-call.ts` | Adaptador after tool call |

### 2.9 Approval System (Exec)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/bash-tools.exec-approval-request.ts` | Geracao de request de aprovacao |
| `src/agents/bash-tools.exec-approval-followup.ts` | Followup de aprovacao |
| `src/agents/exec-approval-result.ts` | Parsing de resultado de aprovacao |

### 2.10 MCP Tools (Model Context Protocol)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-bundle-mcp-tools.ts` | Bundling de MCP tools |
| `src/agents/pi-bundle-mcp-tools.materialize.ts` | Materializacao de MCP tools |
| `src/agents/pi-bundle-mcp-runtime.ts` | Runtime de MCP |
| `src/agents/pi-bundle-mcp-names.ts` | Nomes de MCP tools |
| `src/agents/embedded-pi-mcp.ts` | Integracao MCP embedded |
| `src/agents/mcp-http.ts` | Transporte MCP HTTP |
| `src/agents/mcp-sse.ts` | Transporte MCP SSE |
| `src/agents/mcp-stdio.ts` | Transporte MCP stdio |
| `src/agents/mcp-transport.ts` | Abstracao de transporte MCP |
| `src/agents/pi-embedded-runner/tool-name-allowlist.ts` | Allowlisting de nomes de tools |
| `src/agents/pi-embedded-runner/tool-split.ts` | Split/batching de tools |
| `src/agents/pi-embedded-runner/tool-schema-runtime.ts` | Gerenciamento de schema runtime |

### 2.11 Display de Tools

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/tool-display.ts` | Formatacao de display de tools |
| `src/agents/tool-display-config.ts` | Configuracao de display |
| `src/agents/tool-display-common.ts` | Utilidades comuns de display |
| `src/agents/tool-display-exec-shell.ts` | Display de tool bash |
| `src/agents/tool-display-exec.ts` | Display de tool exec |

---

## 3. MEMORIA (Curto, Medio e Longo Prazo)

### 3.1 Memoria de Curto Prazo (Context Window / Historico de Conversa)

A janela de contexto do LLM. Tudo que o agente "ve" no turno atual.

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-embedded-runner/history.ts` | Gerenciamento de historico de mensagens |
| `src/agents/pi-embedded-runner/replay-history.ts` | Replay de historico durante construcao de contexto |
| `src/agents/context.ts` | Gerenciamento de contexto e tracking de window |
| `src/agents/context-cache.ts` | Cache de tokens de contexto |
| `src/agents/context-tokens.runtime.ts` | Calculo de tokens |
| `src/agents/context-window-guard.ts` | Protecao contra overflow de context window |
| `src/agents/context-runtime-state.ts` | Estado runtime de contexto |
| `src/agents/internal-runtime-context.ts` | Contexto runtime interno |
| `src/agents/pi-embedded-runner/context-engine-maintenance.ts` | Lifecycle do context engine |

### 3.2 Memoria de Medio Prazo (Persistencia de Sessao / Transcripts)

Sessoes persistidas em disco. Permite continuar conversas entre reinicializacoes.

| Arquivo | Descricao |
|---------|-----------|
| `src/config/sessions/store.ts` | Store principal de metadados de sessao (sessions.json) |
| `src/config/sessions/store-load.ts` | Carregamento de dados de sessao |
| `src/config/sessions/store-read.ts` | Leitura do store de sessoes |
| `src/config/sessions/store-cache.ts` | Cache do store de sessoes |
| `src/config/sessions/store-maintenance.ts` | Poda e rotacao de sessoes |
| `src/config/sessions/store-migrations.ts` | Migracoes de dados |
| `src/config/sessions/store-lock-state.ts` | Controle de acesso concorrente |
| `src/config/sessions/store-summary.ts` | Sumarios de sessao |
| `src/config/sessions/store.runtime.ts` | Runtime do store |
| `src/config/sessions/transcript.ts` | Gerenciamento de arquivos transcript (*.jsonl) |
| `src/config/sessions/transcript.runtime.ts` | Operacoes runtime de transcript |
| `src/config/sessions/transcript-mirror.ts` | Mirror/backup de transcripts |
| `src/config/sessions/session-file.ts` | Arquivo de sessao |
| `src/config/sessions/session-key.ts` | Chave de sessao |
| `src/config/sessions/metadata.ts` | Metadados de sessao |
| `src/config/sessions/paths.ts` | Caminhos de sessao |
| `src/config/sessions/disk-budget.ts` | Gerenciamento de espaco em disco |
| `src/config/sessions/artifacts.ts` | Artefatos de sessao |
| `src/config/sessions/group.ts` | Grupo de sessoes |
| `src/config/sessions/delivery-info.ts` | Info de entrega |
| `src/config/sessions/thread-info.ts` | Info de thread |
| `src/config/sessions/targets.ts` | Targets de sessao |
| `src/config/sessions/reset.ts` | Reset de sessao |
| `src/config/sessions/main-session.ts` | Sessao principal |
| `src/config/sessions/main-session.runtime.ts` | Runtime da sessao principal |
| `src/config/sessions/inbound.runtime.ts` | Runtime de inbound |
| `src/sessions/transcript-events.ts` | Tracking de eventos de transcript |
| `src/agents/pi-embedded-runner/session-manager-cache.ts` | Cache de session manager |
| `src/agents/pi-embedded-runner/session-manager-init.ts` | Inicializacao de sessao |
| `src/agents/session-dirs.ts` | Organizacao de diretorios de sessao |
| `src/agents/session-slug.ts` | Identificacao de sessao |
| `src/agents/session-file-repair.ts` | Reparo de arquivo de sessao |
| `src/agents/session-transcript-repair.ts` | Reparo de transcripts |
| `src/agents/session-write-lock.ts` | Write lock de sessao |
| `src/agents/transcript-policy.ts` | Politica de persistencia de transcript |

### 3.3 Memoria de Longo Prazo (Embeddings, Vector DB, Knowledge Base)

Sistema de memoria persistente com busca semantica via embeddings vetoriais.

#### 3.3.1 SDK de Memoria (Host)

| Arquivo | Descricao |
|---------|-----------|
| `src/memory-host-sdk/runtime.ts` | Interface runtime principal |
| `src/memory-host-sdk/runtime-core.ts` | Core runtime |
| `src/memory-host-sdk/runtime-cli.ts` | Runtime CLI |
| `src/memory-host-sdk/runtime-files.ts` | Runtime de arquivos |
| `src/memory-host-sdk/engine.ts` | Engine principal de memoria |
| `src/memory-host-sdk/engine-embeddings.ts` | Engine de embeddings |
| `src/memory-host-sdk/engine-storage.ts` | Engine de armazenamento |
| `src/memory-host-sdk/engine-foundation.ts` | Fundacao do engine |
| `src/memory-host-sdk/engine-qmd.ts` | Engine QMD (Queryable Markdown) |
| `src/memory-host-sdk/query.ts` | Interface de query de memoria |
| `src/memory-host-sdk/multimodal.ts` | Suporte multi-modal |
| `src/memory-host-sdk/dreaming.ts` | Sistema de consolidacao em background (dreaming) |
| `src/memory-host-sdk/embeddings.ts` | Camada de abstracao de embeddings |
| `src/memory-host-sdk/secret.ts` | Gerenciamento de secrets |
| `src/memory-host-sdk/status.ts` | Status do sistema de memoria |

#### 3.3.2 Providers de Embedding (Host)

| Arquivo | Descricao |
|---------|-----------|
| `src/memory-host-sdk/host/embeddings.ts` | Provider de embeddings principal |
| `src/memory-host-sdk/host/embeddings-openai.ts` | Provider OpenAI embeddings |
| `src/memory-host-sdk/host/embeddings-gemini.ts` | Provider Gemini embeddings |
| `src/memory-host-sdk/host/embeddings-mistral.ts` | Provider Mistral embeddings |
| `src/memory-host-sdk/host/embeddings-voyage.ts` | Provider Voyage embeddings |
| `src/memory-host-sdk/host/embeddings-ollama.ts` | Provider Ollama embeddings |
| `src/memory-host-sdk/host/embedding-inputs.ts` | Tratamento de inputs |
| `src/memory-host-sdk/host/embedding-vectors.ts` | Tratamento de vetores |
| `src/memory-host-sdk/host/memory-schema.ts` | Schema do banco de dados |
| `src/memory-host-sdk/host/sqlite.ts` | Backend SQLite |
| `src/memory-host-sdk/host/sqlite-vec.ts` | SQLite com suporte vetorial |
| `src/memory-host-sdk/host/temporal-decay.ts` | Decay temporal para frescor de memoria |
| `src/memory-host-sdk/host/qmd-query-parser.ts` | Parser de sintaxe de query |
| `src/memory-host-sdk/host/batch-*.ts` | Operacoes batch de embedding (Gemini, OpenAI, Voyage, HTTP) |

#### 3.3.3 Plugin de Memoria Core (extension)

| Arquivo | Descricao |
|---------|-----------|
| `extensions/memory-core/src/tools.ts` | Tools de memoria (memory_search, memory_get, memory_write) |
| `extensions/memory-core/src/tools.shared.ts` | Utilidades compartilhadas de tools |
| `extensions/memory-core/src/tools.runtime.ts` | Runtime de tools de memoria |
| `extensions/memory-core/src/tools.citations.ts` | Citacoes para referencias de memoria |
| `extensions/memory-core/src/tools.recall-tracking.ts` | Tracking de padrao de recall |
| `extensions/memory-core/src/prompt-section.ts` | Injecao de memoria no prompt |
| `extensions/memory-core/src/flush-plan.ts` | Agendamento de flush de memoria |
| `extensions/memory-core/src/concept-vocabulary.ts` | Conceitos semanticos para memoria |
| `extensions/memory-core/src/runtime-provider.ts` | Provider runtime |
| `extensions/memory-core/src/cli.ts` | CLI de memoria |
| `extensions/memory-core/src/cli.types.ts` | Tipos CLI |
| `extensions/memory-core/src/cli.runtime.ts` | Runtime CLI |
| `extensions/memory-core/src/cli.host.runtime.ts` | Runtime CLI host |

#### 3.3.4 Gerenciamento e Busca de Memoria

| Arquivo | Descricao |
|---------|-----------|
| `extensions/memory-core/src/memory/manager.ts` | Gerenciador central de memoria |
| `extensions/memory-core/src/memory/manager-search.ts` | Operacoes de busca |
| `extensions/memory-core/src/memory/search-manager.ts` | Gerenciador de busca |
| `extensions/memory-core/src/memory/manager-embedding-ops.ts` | Operacoes de embedding |
| `extensions/memory-core/src/memory/manager-sync-ops.ts` | Operacoes de sincronizacao/indexacao |
| `extensions/memory-core/src/memory/hybrid.ts` | Busca hibrida (vetor + keyword) |
| `extensions/memory-core/src/memory/mmr.ts` | Maximal Marginal Relevance (reranking) |
| `extensions/memory-core/src/memory/temporal-decay.ts` | Relevancia temporal |
| `extensions/memory-core/src/memory/qmd-manager.ts` | Backend QMD |
| `extensions/memory-core/src/memory/embeddings.ts` | Wrapper de embeddings |
| `extensions/memory-core/src/memory/provider-adapters.ts` | Adaptadores de providers de embedding |

#### 3.3.5 Integracao de Memoria no Core

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/memory-search.ts` | Integracao de busca de memoria |
| `src/agents/agent-scope.ts` | Resolucao de config de busca de memoria |
| `src/plugins/memory-runtime.ts` | Runtime de plugin de memoria |
| `src/plugins/memory-state.ts` | Estado de memoria |
| `src/plugins/memory-embedding-providers.ts` | Registro de providers de embedding |
| `src/plugins/memory-embedding-provider-runtime.ts` | Runtime de providers |
| `src/plugin-sdk/memory-core.ts` | Exports do SDK de memoria |

---

## 4. COMPACTION (Compressao de Contexto / Sumarizacao)

Quando o context window atinge o limite, o sistema compacta o historico criando um resumo.

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/compaction.ts` | **CRITICO** - Logica principal de compaction (sumarizacao de conversa) |
| `src/agents/compaction-real-conversation.ts` | Compaction de conversa real |
| `src/agents/pi-embedded-runner/compact.ts` | Compaction de sessao embedded |
| `src/agents/pi-embedded-runner/compact.runtime.ts` | Dispatch runtime de compaction |
| `src/agents/pi-embedded-runner/compaction-hooks.ts` | Hooks pre/pos compaction |
| `src/agents/pi-embedded-runner/compaction-runtime-context.ts` | Contexto runtime de compaction |
| `src/agents/pi-embedded-runner/compaction-safety-timeout.ts` | Timeout de seguranca |
| `src/agents/pi-embedded-runner/compact-reasons.ts` | Razoes de trigger de compaction |
| `src/agents/pi-embedded-runner/run/compaction-timeout.ts` | Tratamento de timeout |
| `src/agents/pi-embedded-runner/run/compaction-retry-aggregate-timeout.ts` | Logica de retry |
| `src/agents/pi-hooks/compaction-safeguard.ts` | Guardas de seguranca para compaction |
| `src/agents/pi-hooks/compaction-instructions.ts` | Instrucoes de sistema para compaction |
| `src/agents/pi-hooks/compaction-safeguard-quality.ts` | Verificacao de qualidade |

---

## 5. RACIOCINIO (Reasoning / Thinking)

### 5.1 Extended Thinking

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-embedded-runner/thinking.ts` | Integracao de modo thinking estendido |
| `src/agents/pi-embedded-helpers/thinking.ts` | Utilidades e helpers de modo thinking |

### 5.2 System Prompt (Instrucoes do Sistema)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/system-prompt.ts` | **CRITICO** - Builder de system prompt. Monta instrucoes visiveis ao agente de multiplas fontes (agents.md, soul.md, identity.md, user.md, bootstrap.md, memory.md) |
| `src/agents/system-prompt-params.ts` | Parametros do system prompt |
| `src/agents/system-prompt-report.ts` | Report/debug de construcao de prompt |
| `src/agents/system-prompt-contribution.ts` | Contribuicoes de providers ao system prompt |
| `src/agents/system-prompt-cache-boundary.ts` | Fronteira de cache do system prompt |

### 5.3 Bootstrap (Contexto Inicial do Agente)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/bootstrap-files.ts` | Resolve arquivos de bootstrap para o run |
| `src/agents/bootstrap-budget.ts` | Analise de orcamento de bootstrap (truncation) |
| `src/agents/bootstrap-cache.ts` | Cache de bootstrap |
| `src/agents/bootstrap-hooks.ts` | Hooks de bootstrap |

---

## 6. APRENDIZADO (Learning / Dreaming / Consolidacao)

### 6.1 Dreaming (Consolidacao em Background)

O mecanismo de "sonhar" - processa memorias em background, promove dados de curto prazo para longo prazo.

| Arquivo | Descricao |
|---------|-----------|
| `extensions/memory-core/src/dreaming.ts` | **CRITICO** - Engine principal de dreaming/consolidacao |
| `extensions/memory-core/src/dreaming-phases.ts` | Fases do processo (extracao, analise, consolidacao) |
| `extensions/memory-core/src/dreaming-command.ts` | Interface CLI para dreaming |
| `extensions/memory-core/src/dreaming-markdown.ts` | Formato de saida markdown para dreams |
| `src/memory-host-sdk/dreaming.ts` | Sistema de consolidacao/promocao em background |

### 6.2 Promocao de Memoria de Curto para Longo Prazo

| Arquivo | Descricao |
|---------|-----------|
| `extensions/memory-core/src/short-term-promotion.ts` | **CRITICO** - Promove notas diarias para memoria de longo prazo durante dreaming |

### 6.3 Vocabulario de Conceitos

| Arquivo | Descricao |
|---------|-----------|
| `extensions/memory-core/src/concept-vocabulary.ts` | Extracao de conceitos e construcao de vocabulario semantico |

### 6.4 Memory Flush (Salvamento Proativo)

| Arquivo | Descricao |
|---------|-----------|
| `src/auto-reply/reply/memory-flush.ts` | Logica de gating de memory flush |
| `src/auto-reply/reply/agent-runner-memory.ts` | Execucao de memory flush |
| `src/auto-reply/reply/agent-runner-memory.runtime.ts` | Runtime de memory handling |
| `src/agents/pi-embedded-runner/wait-for-idle-before-flush.ts` | Timing de flush |

---

## 7. SKILLS (Habilidades Extensiveis)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/skills/workspace.ts` | Gerenciamento e carregamento de skills workspace |
| `src/agents/skills/command-specs.ts` | Sistema de especificacao de comandos de skills |
| `src/agents/skills/frontmatter.ts` | Parsing de frontmatter para metadata de skills |
| `src/agents/skills/skill-contract.ts` | Contratos de definicao de skills |
| `src/agents/skills/types.ts` | Tipos de skills |
| `src/agents/skills/plugin-skills.ts` | Integracao com skills de plugins |
| `src/agents/skills/local-loader.ts` | Carregamento de skills locais |
| `src/agents/skills/refresh.ts` | Refresh da lista de skills |
| `src/agents/skills/bundled-dir.ts` | Diretorio de skills bundled |
| `src/agents/skills/bundled-context.ts` | Contexto de skills bundled |
| `src/agents/skills/filter.ts` | Filtro de skills |
| `src/agents/skills/agent-filter.ts` | Filtro de skills por agente |
| `src/agents/skills/config.ts` | Configuracao de skills |
| `src/agents/skills/source.ts` | Fonte de skills |
| `src/agents/skills/serialize.ts` | Serializacao de skills |
| `src/agents/skills/runtime-config.ts` | Config runtime |
| `src/agents/skills/tools-dir.ts` | Diretorio de tools de skills |
| `src/agents/skills/env-overrides.ts` | Overrides de ambiente |
| `src/agents/skills/env-overrides.runtime.ts` | Runtime de overrides |
| `src/agents/skills/refresh-state.ts` | Estado de refresh |
| `src/agents/skills.ts` | Modulo principal de skills |

---

## 8. HOOKS (Sistema de Hooks)

| Arquivo | Descricao |
|---------|-----------|
| `src/hooks/hooks.ts` | Definicao core de hooks |
| `src/hooks/loader.ts` | Carregador/runtime de hooks |
| `src/hooks/install.ts` | Sistema de instalacao de hooks |
| `src/hooks/install.runtime.ts` | Runtime de instalacao |
| `src/hooks/installs.ts` | Tracking de instalacoes |
| `src/hooks/internal-hooks.ts` | Hooks built-in do sistema |
| `src/hooks/message-hooks.test.ts` | Hooks de mensagem |
| `src/hooks/message-hook-mappers.ts` | Mapeadores de hooks de mensagem |
| `src/hooks/plugin-hooks.ts` | Hooks especificos de plugins |
| `src/hooks/policy.ts` | Politica de execucao de hooks |
| `src/hooks/frontmatter.ts` | Frontmatter de hooks |
| `src/hooks/module-loader.ts` | Carregador de modulos |
| `src/hooks/config.ts` | Configuracao de hooks |
| `src/hooks/hooks-status.ts` | Status de hooks |
| `src/hooks/import-url.ts` | Import de URL |
| `src/hooks/legacy-config.ts` | Config legada |
| `src/hooks/fire-and-forget.ts` | Hooks fire-and-forget |
| `src/hooks/bundled-dir.ts` | Diretorio de hooks bundled |
| `src/plugins/hooks.ts` | Runner principal de plugin hooks (before-tool-call, after-tool-call, etc.) |
| `src/plugins/hook-runner-global.ts` | Runner global de hooks |

---

## 9. PROVIDERS DE MODELO (Abstracao de LLMs)

### 9.1 Streaming e Wrappers por Provider

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/provider-stream.ts` | Registro de implementacoes de streaming por provider |
| `src/agents/anthropic-transport-stream.ts` | Streaming da API Anthropic |
| `src/agents/anthropic-vertex-stream.ts` | Streaming Vertex AI (modelos Anthropic) |
| `src/agents/openai-stream-wrappers.ts` | Streaming OpenAI |
| `src/agents/google-stream-wrappers.ts` | Streaming Google |
| `src/agents/bedrock-stream-wrappers.ts` | Streaming AWS Bedrock |
| `src/agents/minimax-stream-wrappers.ts` | Streaming Minimax |
| `src/agents/moonshot-stream-wrappers.ts` | Streaming Moonshot |
| `src/agents/proxy-stream-wrappers.ts` | Streaming via proxy |
| `src/agents/zai-stream-wrappers.ts` | Streaming ZAI |
| `src/agents/openrouter-model-capabilities.ts` | Capacidades OpenRouter |

### 9.2 Selecao e Config de Modelo

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/model-selection.ts` | Selecao de modelo padrao para agente |
| `src/agents/model-selection-display.ts` | Display de selecao de modelo |
| `src/agents/model-auth.ts` | Modo de autenticacao de modelo |
| `src/agents/model-alias-lines.ts` | Linhas de alias de modelo |
| `src/agents/synthetic-models.ts` | Modelos sinteticos |
| `src/agents/configured-provider-fallback.ts` | Fallback de provider configurado |
| `src/agents/api-key-rotation.ts` | Rotacao de API keys |

### 9.3 Prompt Caching

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/pi-embedded-runner/anthropic-cache-control-payload.ts` | Prompt caching Anthropic |
| `src/agents/pi-embedded-runner/google-prompt-cache.ts` | Prompt caching Google |
| `src/agents/pi-embedded-runner/prompt-cache-retention.ts` | Politica de retencao de cache |
| `src/agents/pi-embedded-runner/cache-ttl.ts` | Gerenciamento de TTL de cache |
| `src/agents/pi-embedded-runner/prompt-cache-observability.ts` | Metricas de cache |
| `src/agents/cache-trace.ts` | Trace de cache |

### 9.4 Auth Profiles

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/auth-profiles.ts` | Gerenciamento de perfis de autenticacao |
| `src/agents/auth-profiles.runtime.ts` | Runtime de auth profiles |
| `src/agents/auth-health.ts` | Health check de autenticacao |

---

## 10. SUBAGENTES (Multi-Agent Orchestration)

| Arquivo | Descricao |
|---------|-----------|
| `src/agents/subagent-spawn.ts` | Spawn de subagentes |
| `src/agents/subagent-spawn.runtime.ts` | Runtime de spawn |
| `src/agents/subagent-registry.ts` | Registro central de subagentes |
| `src/agents/subagent-registry.types.ts` | Tipos do registro |
| `src/agents/subagent-registry-state.ts` | Estado do registro |
| `src/agents/subagent-registry-lifecycle.ts` | Lifecycle de subagentes |
| `src/agents/subagent-registry-queries.ts` | Queries no registro |
| `src/agents/subagent-registry-read.ts` | Leitura do registro |
| `src/agents/subagent-registry-helpers.ts` | Helpers do registro |
| `src/agents/subagent-registry-cleanup.ts` | Cleanup do registro |
| `src/agents/subagent-registry-completion.ts` | Completacao de subagentes |
| `src/agents/subagent-registry-memory.ts` | Tracking de memoria de subagentes |
| `src/agents/subagent-registry-run-manager.ts` | Gerenciador de runs de subagentes |
| `src/agents/subagent-registry.store.ts` | Store do registro |
| `src/agents/subagent-registry.runtime.ts` | Runtime do registro |
| `src/agents/subagent-capabilities.ts` | Capacidades de subagentes |
| `src/agents/subagent-control.ts` | Controle de subagentes |
| `src/agents/subagent-depth.ts` | Profundidade de subagentes |
| `src/agents/subagent-attachments.ts` | Anexos de subagentes |
| `src/agents/subagent-announce.ts` | Anuncio de subagentes |
| `src/agents/subagent-announce-delivery.ts` | Entrega de anuncio |
| `src/agents/subagent-announce-dispatch.ts` | Dispatch de anuncio |
| `src/agents/subagent-announce-output.ts` | Saida de anuncio |
| `src/agents/subagent-announce-queue.ts` | Fila de anuncios |
| `src/agents/subagent-lifecycle-events.ts` | Eventos de lifecycle |
| `src/agents/subagent-orphan-recovery.ts` | Recuperacao de orfaos |
| `src/agents/acp-spawn.ts` | Spawn via ACP |
| `src/agents/acp-spawn-parent-stream.ts` | Stream do parent |
| `src/agents/spawned-context.ts` | Contexto de spawn |

---

## 11. AUTO-REPLY (Dispatch de Respostas)

O sistema que orquestra a resposta automatica a mensagens recebidas.

| Arquivo | Descricao |
|---------|-----------|
| `src/auto-reply/dispatch.ts` | Dispatch principal de auto-reply |
| `src/auto-reply/envelope.ts` | Envelope de mensagem |
| `src/auto-reply/command-detection.ts` | Deteccao de comandos |
| `src/auto-reply/command-auth.ts` | Autenticacao de comandos |
| `src/auto-reply/commands-registry.ts` | Registro de comandos |
| `src/auto-reply/commands-registry.types.ts` | Tipos do registro |
| `src/auto-reply/commands-registry.shared.ts` | Compartilhados do registro |
| `src/auto-reply/commands-registry.data.ts` | Data do registro |
| `src/auto-reply/commands-registry.runtime.ts` | Runtime do registro |
| `src/auto-reply/commands-text-routing.ts` | Roteamento de texto |
| `src/auto-reply/commands-args.ts` | Argumentos de comandos |
| `src/auto-reply/chunk.ts` | Chunking de respostas |
| `src/auto-reply/heartbeat.ts` | Sistema de heartbeat |
| `src/auto-reply/fallback-state.ts` | Estado de fallback |
| `src/auto-reply/group-activation.ts` | Ativacao de grupo |
| `src/auto-reply/inbound-debounce.ts` | Debounce de inbound |
| `src/auto-reply/media-note.ts` | Notas de midia |
| `src/auto-reply/reply/agent-runner-execution.ts` | Execucao do agent runner |
| `src/auto-reply/reply/agent-runner-execution.runtime.ts` | Runtime de execucao |
| `src/auto-reply/reply/agent-runner-helpers.ts` | Helpers do runner |
| `src/auto-reply/reply/agent-runner-payloads.ts` | Payloads do runner |
| `src/auto-reply/reply/agent-runner-utils.ts` | Utilidades do runner |
| `src/auto-reply/reply/agent-runner-auth-profile.ts` | Auth profile do runner |
| `src/auto-reply/reply/agent-runner-reminder-guard.ts` | Guard de lembretes |
| `src/auto-reply/reply/agent-runner-usage-line.ts` | Linha de uso |
| `src/auto-reply/reply/agent-runner.runtime.ts` | Runtime do runner |
| `src/auto-reply/reply/abort.ts` | Abort de reply |
| `src/auto-reply/reply/abort.runtime.ts` | Runtime de abort |
| `src/auto-reply/reply/abort-cutoff.ts` | Cutoff de abort |
| `src/auto-reply/reply/abort-cutoff.runtime.ts` | Runtime de cutoff |
| `src/auto-reply/reply/abort-primitives.ts` | Primitivas de abort |
| `src/auto-reply/reply/history.ts` | Tracking de historico de reply |
| `src/auto-reply/reply/dispatch-acp.ts` | ACP dispatch para tools |
| `src/auto-reply/reply/dispatch-acp-manager.runtime.ts` | Runtime do ACP manager |
| `src/auto-reply/reply/acp-projector.ts` | Projector ACP |
| `src/auto-reply/reply/acp-stream-settings.ts` | Settings de stream ACP |
| `src/auto-reply/reply/acp-reset-target.ts` | Reset target ACP |
| `src/auto-reply/reply/commands-mcp.ts` | Comandos MCP |
| `src/auto-reply/reply/commands-registry.ts` | Registro de comandos |
| `src/auto-reply/reply/commands-session-store.ts` | Store de sessao de comandos |
| `src/auto-reply/tool-meta.ts` | Metadados de tools para auto-reply |

---

## 12. PLUGIN SYSTEM (Nucleo)

| Arquivo | Descricao |
|---------|-----------|
| `src/plugins/discovery.ts` | Descoberta de plugins |
| `src/plugins/loader.ts` | Carregador de plugins (lifecycle completo) |
| `src/plugins/manifest.ts` | Tratamento de manifesto |
| `src/plugins/manifest-registry.ts` | Registro de manifestos |
| `src/plugins/registry.ts` | Registro central de plugins |
| `src/plugins/types.ts` | Tipos de plugins |
| `src/plugins/runtime.ts` | Execucao runtime de plugins |
| `src/plugins/install.ts` | Mecanismo de instalacao |
| `src/plugins/enable.ts` | Sistema de enable/disable |
| `src/plugins/uninstall.ts` | Desinstalacao |
| `src/plugins/update.ts` | Atualizacoes |
| `src/plugins/config-policy.ts` | Politicas de config |
| `src/plugins/config-state.ts` | Estado de config |
| `src/plugins/config-schema.ts` | Schemas de config |
| `src/plugins/status.ts` | Status de plugins |
| `src/plugins/tools.ts` | Resolucao de tools de plugins |
| `src/plugins/conversation-binding.ts` | Binding de plugins a conversas |
| `src/plugins/captured-registration.ts` | Captura de registros |
| `src/plugins/bundled-plugin-metadata.ts` | Metadata de plugins bundled |
| `src/plugins/bundled-dir.ts` | Diretorio de plugins bundled |
| `src/plugins/bundled-sources.ts` | Sources de plugins bundled |
| `src/plugins/marketplace.ts` | Integracao marketplace |
| `src/plugin-sdk/core.ts` | SDK core de plugins |
| `src/plugin-sdk/index.ts` | Entry point do SDK |
| `src/plugin-sdk/plugin-entry.ts` | Implementacao de entry |

---

## 13. CONFIGURACAO DO AGENTE

| Arquivo | Descricao |
|---------|-----------|
| `src/config/types.agent-defaults.ts` | Tipos de defaults de agente (modelos, sistema, compaction, heartbeat) |
| `src/config/zod-schema.agent-defaults.ts` | Schema Zod para defaults |
| `src/config/types.agents.ts` | Tipos para configuracao individual de agentes |
| `src/config/zod-schema.agents.ts` | Schema Zod para agentes |
| `src/config/types.tools.ts` | Tipos de configuracao de tools |
| `src/config/types.hooks.ts` | Tipos de configuracao de hooks |
| `src/config/types.skills.ts` | Tipos de configuracao de skills |
| `src/config/defaults.ts` | Defaults globais de configuracao |
| `src/agents/defaults.ts` | Defaults de agente (DEFAULT_CONTEXT_TOKENS, etc.) |
| `src/agents/pi-settings.ts` | Settings de personalidade do agente |
| `src/agents/pi-project-settings.ts` | Settings de projeto |

---

## 14. SEGURANCA

| Arquivo | Descricao |
|---------|-----------|
| `src/security/dangerous-tools.ts` | Deteccao de tools perigosos |
| `src/security/audit-tool-policy.ts` | Auditoria de politica de tools |
| `src/agents/sandbox.ts` | Resolve contexto de sandbox |
| `src/agents/sandbox/registry.ts` | Registro de sandbox |
| `src/agents/sandbox/runtime-status.ts` | Status runtime de sandbox |
| `src/agents/sandbox/fs-bridge-mutation-helper.ts` | Tracking de mutacao em sandbox |

---

## 15. DIAGRAMA DE FLUXO DO AGENTIC LOOP

```
                    USUARIO ENVIA MENSAGEM
                            |
                            v
                   agent-command.ts (entry point)
                            |
                            v
                 attempt-execution.ts (coordenacao)
                            |
                            v
            +=======================================+
            |    run.ts - LOOP PRINCIPAL            |
            |    while (true) {                     |
            |                                       |
            |      1. Selecionar modelo             |
            |      2. Resolver auth profile         |
            |      3. Executar attempt ----------+  |
            |                                    |  |
            |      4. Avaliar resultado:         |  |
            |         - Sucesso -> SAIR          |  |
            |         - Retry -> continua loop   |  |
            |         - Failover -> troca modelo |  |
            |         - Overflow -> COMPACTAR    |  |
            |         - Max retries -> SAIR      |  |
            |    }                                  |
            +=======================================+
                            |
                            v
            +=======================================+
            |    attempt.ts - TURNO INDIVIDUAL      |
            |                                       |
            |    1. Criar AgentSession              |
            |    2. Configurar tools                |
            |    3. Montar system prompt            |
            |    4. Iniciar streaming               |
            |    5. Subscription processa eventos   |
            +=======================================+
                            |
                            v
            +=======================================+
            |    SUBSCRIPTION / EVENT HANDLERS      |
            |                                       |
            |    message_start  -> Inicia resposta  |
            |    message_update -> Acumula texto    |
            |    message_end    -> Finaliza msg     |
            |    tool_exec_start -> Inicia tool     |
            |    tool_exec_end   -> Processa result |
            |    agent_end       -> Finaliza run    |
            |    compaction_*    -> Compacta ctx     |
            +=======================================+
                            |
                            v
                   RESPOSTA AO USUARIO


        CICLO DE MEMORIA:

        Curto Prazo (context window)
              |  overflow -> Compaction (sumariza)
              v
        Medio Prazo (sessao/transcript *.jsonl)
              |  dreaming -> Promocao
              v
        Longo Prazo (embeddings vetoriais / SQLite-vec)
              |
              v
        Busca Semantica (memory_search tool)
              |
              v
        Injecao no Prompt (prompt-section.ts)
```

---

## 16. GUIA PARA CLONAGEM EM GO/GORM

### Mapeamento de Conceitos TS -> Go

| Conceito TypeScript | Equivalente Go/GORM |
|---------------------|---------------------|
| AgentSession | struct AgentSession com metodos |
| SessionManager | GORM models (Session, Message, ToolCall) |
| Tool definitions (TypeBox) | Go structs com JSON schema tags |
| Streaming (AsyncIterator) | Go channels ou SSE |
| Compaction | Goroutine com LLM call para sumarizar |
| Memory/Embeddings | pgvector ou SQLite-vec via GORM |
| Plugin system | Go plugin ou interface-based DI |
| Hooks | Go interfaces/callbacks |
| MCP | HTTP/stdio clients |
| Auth profiles | GORM model AuthProfile |

### Modelos GORM Sugeridos

```go
// Sessao (medio prazo)
type Session struct {
    gorm.Model
    AgentID     string
    SessionKey  string `gorm:"uniqueIndex"`
    Status      string
    Metadata    datatypes.JSON
}

// Mensagem no historico (curto prazo)
type Message struct {
    gorm.Model
    SessionID   uint
    Role        string // user, assistant, system, tool
    Content     string
    TokenCount  int
    Metadata    datatypes.JSON
}

// Tool Call
type ToolCall struct {
    gorm.Model
    MessageID   uint
    ToolName    string
    Arguments   datatypes.JSON
    Result      string
    Status      string // pending, running, completed, error
    Duration    int64
}

// Memoria de longo prazo
type Memory struct {
    gorm.Model
    AgentID     string
    Content     string
    Embedding   pgvector.Vector `gorm:"type:vector(1536)"`
    Source      string // conversation, dreaming, manual
    Importance  float64
    LastAccess  time.Time
    DecayFactor float64
}

// Compaction (resumos)
type CompactionSummary struct {
    gorm.Model
    SessionID   uint
    Summary     string
    TokensSaved int
    TurnRange   string // "1-50"
}

// Tool Definition
type ToolDefinition struct {
    gorm.Model
    Name        string
    Description string
    Schema      datatypes.JSON
    Category    string // builtin, mcp, plugin
    Enabled     bool
}

// Agent Config
type AgentConfig struct {
    gorm.Model
    AgentID          string `gorm:"uniqueIndex"`
    SystemPrompt     string
    Model            string
    MaxContextTokens int
    Skills           datatypes.JSON
    Hooks            datatypes.JSON
}
```

### Estrutura de Diretorios Go Sugerida

```
cmd/
  openclaw/main.go          # Entry point
internal/
  agent/
    loop.go                 # Agentic loop (run.ts)
    attempt.go              # Single turn (attempt.ts)
    failover.go             # Failover logic
    compaction.go           # Context compaction
  tools/
    registry.go             # Tool catalog
    executor.go             # Tool execution
    policy.go               # Tool permissions
    builtin/                # Built-in tools
      bash.go
      web.go
      file.go
  memory/
    short_term.go           # Context window mgmt
    medium_term.go          # Session persistence
    long_term.go            # Embeddings/vector search
    dreaming.go             # Background consolidation
    promotion.go            # Short->long term
  provider/
    interface.go            # LLM provider interface
    anthropic.go            # Anthropic implementation
    openai.go               # OpenAI implementation
    streaming.go            # Streaming handler
  prompt/
    system.go               # System prompt builder
    bootstrap.go            # Bootstrap files
    cache.go                # Prompt caching
  session/
    manager.go              # Session management
    transcript.go           # Transcript persistence
    store.go                # Session store (GORM)
  hooks/
    runner.go               # Hook execution
    types.go                # Hook interfaces
  skills/
    loader.go               # Skill loading
    registry.go             # Skill registry
  config/
    agent.go                # Agent configuration
    types.go                # Config types
  models/
    models.go               # GORM models
    migrations.go           # DB migrations
pkg/
  schema/                   # JSON schema helpers
  streaming/                # SSE/streaming utilities
```

---

## RESUMO DE CONTAGEM DE ARQUIVOS CORE

| Subsistema | Arquivos (aprox.) |
|------------|-------------------|
| Agentic Loop (run, attempt, subscription, handlers) | ~45 |
| Tool Calls (catalog, execution, policy, schema, MCP) | ~80 |
| Memoria Curto Prazo (context, history) | ~10 |
| Memoria Medio Prazo (sessions, transcripts) | ~35 |
| Memoria Longo Prazo (embeddings, vector, dreaming) | ~40 |
| Compaction | ~13 |
| Raciocinio (thinking, system prompt, bootstrap) | ~10 |
| Aprendizado (dreaming, promotion, flush) | ~10 |
| Skills | ~20 |
| Hooks | ~20 |
| Providers de Modelo | ~15 |
| Subagentes | ~30 |
| Auto-Reply | ~40 |
| Plugin System | ~25 |
| Config | ~10 |
| Seguranca | ~6 |
| **TOTAL** | **~400+ arquivos** |

> **Nota:** Este documento lista apenas arquivos de producao (sem `.test.ts`). O codebase total com testes e muito maior.
