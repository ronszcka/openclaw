# 09 - Cloud Run Architecture (Wake → Execute → Sleep)

## Resumo

O padrão ideal para Cloud Run é **scale-to-zero**: a instância dorme com 0 containers, acorda ao receber um request (webhook de mensagem ou cron trigger), executa o agent loop completo, e volta a dormir. Custo só quando processa.

## Arquitetura

```
                    ┌─────────────────────────┐
                    │   Cloud Scheduler        │
                    │   (cron triggers)        │
                    └──────────┬──────────────┘
                               │ HTTP POST
                               ▼
┌──────────┐    HTTP POST    ┌─────────────────────────┐
│ Telegram │ ──────────────→ │                         │
│ Webhook  │                 │   CLOUD RUN             │
├──────────┤                 │   (min instances: 0)    │
│ Discord  │ ──────────────→ │                         │
│ Webhook  │                 │   ┌───────────────────┐ │
├──────────┤                 │   │ Go HTTP Server    │ │
│ WhatsApp │ ──────────────→ │   │                   │ │    ┌─────────────┐
│ Webhook  │                 │   │ /webhook/telegram │─┼──→ │ Cloud SQL   │
├──────────┤                 │   │ /webhook/discord  │ │    │ (PostgreSQL)│
│ Slack    │ ──────────────→ │   │ /webhook/slack    │─┼──→ │ + pgvector  │
│ Events   │                 │   │ /cron/trigger     │ │    └─────────────┘
└──────────┘                 │   │ /api/v1/*         │ │
                             │   └───────────────────┘ │    ┌─────────────┐
                             │                         │──→ │ LLM APIs    │
                             │   Processa → Responde   │    │ (Anthropic, │
                             │   → Scale to zero       │    │  OpenAI)    │
                             └─────────────────────────┘    └─────────────┘
```

## Por que Cloud Run funciona perfeitamente

| Característica | Benefício |
|---------------|-----------|
| **Scale to zero** | Custo $0 quando inativo |
| **Cold start ~500ms** | Go compila para binário, boot instantâneo |
| **Request timeout até 60min** | Tempo suficiente para agent loops complexos |
| **Concurrency configurável** | 1 request por container = isolamento total |
| **Cloud SQL Auth Proxy** | Conexão segura ao PostgreSQL |
| **Secret Manager** | API keys seguras |
| **Cloud Scheduler** | Cron nativo integrado |
| **Min/Max instances** | Controle de custo e escala |

## Modos de Wake

### 1. Wake por Mensagem (Webhook)

```
Telegram envia webhook → Cloud Run acorda → Agent processa → Responde → Dorme

Latência total:
├── Cold start:     ~300-500ms (Go binary)
├── DB connection:  ~100ms (Cloud SQL proxy)
├── Agent loop:     ~2-30s (depende do LLM e tools)
└── Total:          ~3-31s (aceitável para chat)
```

### 2. Wake por Cron (Cloud Scheduler)

```
Cloud Scheduler (0 9 * * *) → POST /cron/trigger → Agent executa task → Dorme

Exemplos:
├── Daily briefing:    "Resuma as notícias do dia"
├── Health check:      "Verifique status dos serviços"
├── Report:            "Gere relatório semanal"
└── Memory dreaming:   "Consolide memórias recentes"
```

### 3. Wake por Pub/Sub (Event-driven)

```
Cloud Pub/Sub → Push subscription → Cloud Run → Processa evento → Dorme

Exemplos:
├── Novo email:        Gmail push notification
├── CI/CD:             GitHub webhook
├── Monitoring alert:  Alertas de infraestrutura
└── Async tasks:       Tasks de longa duração
```

## Estrutura Go para Cloud Run

```go
// cmd/openclaw/main.go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "time"

    "github.com/gin-gonic/gin"
    "gorm.io/gorm"

    "your-org/openclaw-go/internal/agent"
    "your-org/openclaw-go/internal/channel"
    "your-org/openclaw-go/internal/config"
    "your-org/openclaw-go/internal/cron"
    "your-org/openclaw-go/internal/memory"
    "your-org/openclaw-go/pkg/models"
)

func main() {
    // Cloud Run fornece PORT
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    // Init config (from env/Secret Manager)
    cfg := config.Load()

    // Init DB (Cloud SQL via proxy)
    db := initDB(cfg)
    models.AutoMigrate(db)

    // Init core services
    memoryStore := memory.NewStore(db)
    agentLoop := agent.NewLoop(cfg, db, memoryStore)
    cronService := cron.NewService(cfg, db, agentLoop)

    // Init channel adapters
    adapters := channel.InitAdapters(cfg)

    // HTTP server
    r := gin.New()
    r.Use(gin.Recovery())

    // Webhook endpoints (wake on message)
    webhooks := r.Group("/webhook")
    {
        webhooks.POST("/telegram", handleTelegramWebhook(adapters, agentLoop))
        webhooks.POST("/discord", handleDiscordWebhook(adapters, agentLoop))
        webhooks.POST("/slack", handleSlackWebhook(adapters, agentLoop))
        webhooks.POST("/whatsapp", handleWhatsAppWebhook(adapters, agentLoop))
        webhooks.POST("/generic", handleGenericWebhook(adapters, agentLoop))
    }

    // Cron endpoint (wake on schedule)
    r.POST("/cron/trigger", handleCronTrigger(cronService))

    // API endpoints
    api := r.Group("/api/v1")
    {
        api.POST("/chat", handleChat(agentLoop))
        api.GET("/health", handleHealth(db))
        api.GET("/sessions", handleListSessions(db))
    }

    // Graceful shutdown (Cloud Run sends SIGTERM)
    srv := &http.Server{
        Addr:         ":" + port,
        Handler:      r,
        ReadTimeout:  30 * time.Second,
        WriteTimeout: 300 * time.Second, // 5min para agent loops longos
    }

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %s\n", err)
        }
    }()

    // Wait for SIGTERM (Cloud Run graceful shutdown)
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}
```

## Handlers

```go
// Webhook handler (wake on message)
func handleTelegramWebhook(adapters *channel.Adapters, loop *agent.Loop) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. Parse webhook payload
        msg, err := adapters.Telegram.ParseWebhook(c.Request)
        if err != nil {
            c.JSON(400, gin.H{"error": "invalid webhook"})
            return
        }

        // 2. Route to session
        session, err := adapters.Router.ResolveRoute(c.Request.Context(), msg)
        if err != nil {
            c.JSON(500, gin.H{"error": "routing failed"})
            return
        }

        // 3. Run agent loop (Think → Act → Observe)
        result, err := loop.Run(c.Request.Context(), agent.RunRequest{
            SessionID: session.ID,
            Message:   msg.Text,
            Images:    msg.Images,
        })
        if err != nil {
            c.JSON(500, gin.H{"error": "agent failed"})
            return
        }

        // 4. Send reply via channel
        err = adapters.Telegram.Send(c.Request.Context(), msg.ReplyTarget(), result.Text())
        if err != nil {
            c.JSON(500, gin.H{"error": "send failed"})
            return
        }

        // 5. Cloud Run pode dormir agora
        c.JSON(200, gin.H{"ok": true})
    }
}

// Cron handler (wake on schedule)
func handleCronTrigger(cronSvc *cron.Service) gin.HandlerFunc {
    return func(c *gin.Context) {
        var req struct {
            JobID   string `json:"job_id"`
            Payload string `json:"payload"`
        }
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(400, gin.H{"error": "invalid request"})
            return
        }

        // Execute cron job
        result, err := cronSvc.ExecuteJob(c.Request.Context(), req.JobID, req.Payload)
        if err != nil {
            c.JSON(500, gin.H{"error": err.Error()})
            return
        }

        // Deliver result (channel, webhook, etc.)
        if err := cronSvc.DeliverResult(c.Request.Context(), req.JobID, result); err != nil {
            c.JSON(500, gin.H{"error": "delivery failed"})
            return
        }

        c.JSON(200, gin.H{"ok": true, "result": result.Summary()})
    }
}
```

## Cloud Run Config

```yaml
# service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: openclaw-agent
spec:
  template:
    metadata:
      annotations:
        # Scale to zero (essencial!)
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/maxScale: "5"
        # CPU só durante request processing
        run.googleapis.com/cpu-throttling: "true"
        # Cloud SQL proxy
        run.googleapis.com/cloudsql-instances: "project:region:instance"
        # Startup CPU boost para cold start rápido
        run.googleapis.com/startup-cpu-boost: "true"
    spec:
      # 1 request por container = isolamento de sessão
      containerConcurrency: 1
      # Timeout de 5 min para agent loops longos
      timeoutSeconds: 300
      containers:
        - image: gcr.io/your-project/openclaw-agent:latest
          ports:
            - containerPort: 8080
          env:
            - name: DB_CONNECTION
              valueFrom:
                secretKeyRef:
                  name: db-connection
                  key: latest
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: anthropic-api-key
                  key: latest
            - name: TELEGRAM_BOT_TOKEN
              valueFrom:
                secretKeyRef:
                  name: telegram-bot-token
                  key: latest
          resources:
            limits:
              cpu: "2"
              memory: "1Gi"
```

## Cloud Scheduler (Cron)

```bash
# Daily briefing às 9h
gcloud scheduler jobs create http openclaw-daily-briefing \
  --schedule="0 9 * * *" \
  --uri="https://openclaw-agent-xxxxx.run.app/cron/trigger" \
  --http-method=POST \
  --headers="Content-Type=application/json" \
  --body='{"job_id":"daily-briefing","payload":"Resuma as principais notícias do dia"}' \
  --oidc-service-account-email=scheduler@your-project.iam.gserviceaccount.com \
  --time-zone="America/Sao_Paulo"

# Memory dreaming (consolidação) às 3h
gcloud scheduler jobs create http openclaw-memory-dreaming \
  --schedule="0 3 * * *" \
  --uri="https://openclaw-agent-xxxxx.run.app/cron/trigger" \
  --http-method=POST \
  --body='{"job_id":"memory-dreaming","payload":"deep-sleep"}' \
  --oidc-service-account-email=scheduler@your-project.iam.gserviceaccount.com

# Health check a cada 6h
gcloud scheduler jobs create http openclaw-health \
  --schedule="0 */6 * * *" \
  --uri="https://openclaw-agent-xxxxx.run.app/cron/trigger" \
  --http-method=POST \
  --body='{"job_id":"health-check","payload":"Verifique status de todos os serviços"}' \
  --oidc-service-account-email=scheduler@your-project.iam.gserviceaccount.com
```

## Terraform (IaC)

```hcl
# Cloud Run service
resource "google_cloud_run_v2_service" "openclaw" {
  name     = "openclaw-agent"
  location = "southamerica-east1"

  template {
    scaling {
      min_instance_count = 0  # Scale to zero!
      max_instance_count = 5
    }

    containers {
      image = "gcr.io/${var.project}/openclaw-agent:latest"

      ports {
        container_port = 8080
      }

      env {
        name = "DB_CONNECTION"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.db_connection.id
            version = "latest"
          }
        }
      }

      env {
        name = "ANTHROPIC_API_KEY"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.anthropic_key.id
            version = "latest"
          }
        }
      }

      resources {
        limits = {
          cpu    = "2"
          memory = "1Gi"
        }
        cpu_idle          = true   # CPU throttled quando idle
        startup_cpu_boost = true   # Boost no cold start
      }
    }

    volumes {
      name = "cloudsql"
      cloud_sql_instance {
        instances = [google_sql_database_instance.main.connection_name]
      }
    }

    timeout = "300s"
    max_instance_request_concurrency = 1
  }
}

# Cloud SQL (PostgreSQL + pgvector)
resource "google_sql_database_instance" "main" {
  name             = "openclaw-db"
  database_version = "POSTGRES_16"
  region           = "southamerica-east1"

  settings {
    tier = "db-f1-micro"  # Escala conforme necessário

    database_flags {
      name  = "cloudsql.enable_pgvector"
      value = "on"
    }
  }
}

# Cloud Scheduler jobs
resource "google_cloud_scheduler_job" "daily_briefing" {
  name      = "openclaw-daily-briefing"
  schedule  = "0 9 * * *"
  time_zone = "America/Sao_Paulo"

  http_target {
    uri         = "${google_cloud_run_v2_service.openclaw.uri}/cron/trigger"
    http_method = "POST"
    headers     = { "Content-Type" = "application/json" }
    body        = base64encode(jsonencode({
      job_id  = "daily-briefing"
      payload = "Resuma as principais notícias do dia"
    }))

    oidc_token {
      service_account_email = google_service_account.scheduler.email
    }
  }
}

# Secrets
resource "google_secret_manager_secret" "anthropic_key" {
  secret_id = "anthropic-api-key"
  replication { auto {} }
}

resource "google_secret_manager_secret" "db_connection" {
  secret_id = "db-connection"
  replication { auto {} }
}
```

## Dockerfile Multi-stage

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /openclaw ./cmd/openclaw

# Runtime stage
FROM gcr.io/distroless/static-debian12
COPY --from=builder /openclaw /openclaw
EXPOSE 8080
ENTRYPOINT ["/openclaw"]
```

## Estimativa de Custo

```
Cloud Run (scale-to-zero):
├── CPU:     $0.00002400/vCPU-second
├── Memory:  $0.00000250/GiB-second
├── Request: $0.40/million requests
└── Free tier: 2M requests/month, 360K vCPU-seconds

Cenário: 100 mensagens/dia, ~10s cada
├── CPU:     100 × 10s × 2 vCPU = 2000 vCPU-s/dia = $0.048/dia
├── Memory:  100 × 10s × 1 GiB  = 1000 GiB-s/dia  = $0.0025/dia
├── Request: 100/dia = ~$0.00
├── Cloud SQL: db-f1-micro = ~$7.67/mês
├── LLM API:  ~$0.50-5/dia (depende do modelo)
└── Total:    ~$10-15/mês (sem contar LLM API)

Com free tier: potencialmente $0/mês para uso leve!
```

## Otimizações para Cold Start

```go
// 1. Connection pooling com Cloud SQL
func initDB(cfg *config.Config) *gorm.DB {
    db, _ := gorm.Open(postgres.Open(cfg.DBConnection), &gorm.Config{
        // Preparar statements para reuso
        PrepareStmt: true,
    })
    sqlDB, _ := db.DB()
    // Pool mínimo para cold start rápido
    sqlDB.SetMaxOpenConns(5)
    sqlDB.SetMaxIdleConns(2)
    sqlDB.SetConnMaxLifetime(5 * time.Minute)
    return db
}

// 2. Lazy initialization de providers
type LazyProvider struct {
    once     sync.Once
    provider inference.LLMProvider
    cfg      *config.ProviderConfig
}

func (lp *LazyProvider) Get() inference.LLMProvider {
    lp.once.Do(func() {
        lp.provider = inference.NewProvider(lp.cfg)
    })
    return lp.provider
}

// 3. Warmup endpoint para min-instances > 0
func handleWarmup(c *gin.Context) {
    // Cloud Run chama /_ah/warmup no startup
    c.JSON(200, gin.H{"status": "warm"})
}
```

## Padrão Async para Operações Longas

Para agent loops que excedem 30s, use Pub/Sub:

```go
// Webhook recebe, publica no Pub/Sub, responde 200 imediatamente
func handleAsyncWebhook(pubsub *pubsub.Client, agentLoop *agent.Loop) gin.HandlerFunc {
    return func(c *gin.Context) {
        msg, _ := parseWebhook(c)

        // Publica no Pub/Sub para processamento async
        topic := pubsub.Topic("agent-requests")
        result := topic.Publish(c.Request.Context(), &pubsub.Message{
            Data: marshal(msg),
        })

        // Responde 200 imediatamente (Telegram não espera resposta)
        c.JSON(200, gin.H{"ok": true, "message_id": result.Get(c.Request.Context())})
    }
}

// Outro Cloud Run service processa do Pub/Sub (push subscription)
func handlePubSubPush(agentLoop *agent.Loop, adapters *channel.Adapters) gin.HandlerFunc {
    return func(c *gin.Context) {
        msg := parsePubSubMessage(c)

        // Agent loop sem pressão de timeout do webhook
        result, _ := agentLoop.Run(c.Request.Context(), msg.ToRunRequest())

        // Envia resposta via API do canal
        adapters.Send(c.Request.Context(), msg.ReplyTarget(), result)

        c.JSON(200, gin.H{"ok": true})
    }
}
```

## Resumo da Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                    GOOGLE CLOUD                              │
│                                                             │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────┐  │
│  │ Cloud        │   │ Cloud Run      │   │ Cloud SQL    │  │
│  │ Scheduler    │──→│ (scale to 0)   │──→│ PostgreSQL   │  │
│  │ (cron)       │   │                │   │ + pgvector   │  │
│  └──────────────┘   │  Go binary     │   └──────────────┘  │
│                     │  Agent Loop    │                      │
│  ┌──────────────┐   │  Tool System   │   ┌──────────────┐  │
│  │ Secret       │──→│  Memory Store  │──→│ LLM APIs     │  │
│  │ Manager      │   │  Session Mgmt  │   │ (external)   │  │
│  └──────────────┘   └────────────────┘   └──────────────┘  │
│                           ↑                                 │
│  ┌──────────────┐         │                                 │
│  │ Pub/Sub      │─────────┘  (async processing)            │
│  │ (opcional)   │                                           │
│  └──────────────┘                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
         ↑ webhooks
    ┌────┴────┐
    │Telegram │  Discord │  Slack │  WhatsApp │ ...
    └─────────┘
```
