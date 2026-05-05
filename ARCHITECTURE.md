# KreatorsHub — Arquitetura Definitiva

## Guia de referência para escala: 50.000 tenants · 50M contatos · 1000 automações por tenant

*Versão 7.29 — Sprint 2 do refactor fechada (Decision Write API canônica)*

---

# PARTE A — ARQUITETURA DA PLATAFORMA

## 1. Princípios que nunca mudam

Antes de qualquer detalhe técnico, estes princípios governam toda decisão arquitetural. Quando uma dúvida surgir, volte aqui.

**Escalar horizontalmente = adicionar réplicas, nunca mudar código.** Nenhuma lógica pode depender de "só uma instância sabe disso". Estado vive no banco, não no processo.

**Banco é a fonte da verdade. Worker é stateless.** Um worker pode morrer e reiniciar a qualquer momento sem perder nada. O banco guarda tudo.

**Nunca processar na hora de receber.** Todo evento externo é salvo primeiro, processado depois. Receber é O(1). Processar pode levar o tempo que precisar.

**Idempotência em toda entrada — em duas camadas.** Camada 1: `ON CONFLICT DO UPDATE` na transação com chave `(tenant_id, source_name, external_id)` — atualiza status apenas em transições válidas (ex: pending → paid), nunca retrocede (paid permanece paid exceto para refunded). Camada 2: analytics pipeline só roda na transição para `paid` — novo insert paid ou mudança pending → paid. Status igual ao anterior = skip. Evento duplicado não duplica ledger nem state.

**Falha isolada por tenant.** Um tenant com automação em loop não pode degradar os outros. Filas e rate limiting garantem isso.

**Provider novo = só um mapper. Zero mudança no banco.** O banco nunca conhece o nome do provider. O worker nunca conhece a estrutura do banco além do contrato `NormalizedTransaction`.

**Erro de código não é erro de infra — não misturar retry com blocked.** Erro transitório (deadlock, timeout, conexão) → `retry` automático, consome attempts. Erro de mapper/contrato (schema quebrado, campo obrigatório ausente) → `blocked`, não consome attempts, não reprocessa até fix + deploy explícito.

---

## 2. As três camadas da plataforma

**OBSERVAR — `analytics.contact_events`**
Ledger imutável. Append-only. Particionado. Tudo que aconteceu com cada contato.

**ENTENDER — `analytics.contact_state`**
Snapshot vivo. Uma linha por contato. Zero joins em runtime. O que o contato é hoje.

**AGIR — `analytics.segment_eval_queue` + `journeys.processing_jobs`**
O que fazer com o que sabemos.

> **`contact_state` é a fonte única de verdade para RFM.** Nenhuma view, RPC ou frontend recalcula scores RFM em runtime. NTILE existe em um único lugar: `refreshRfm.ts`. Qualquer outro NTILE no sistema é bug arquitetural.

---

## 3. Arquitetura de Workers — Estado Definitivo

### Visão geral

**SUPABASE EDGE FUNCTIONS** (ingestion — leve, rápida, sem estado)

- `doare-sync` → cron-driven | `doare:sync` + `enqueue_pending_donations`
- `apidozap` → webhook-driven | `apidozap:read_receipt`, `apidozap:process_message`
- `nylas-sync` → webhook-driven | `nylas:sync_message` e sub-handlers
- `eduzz-receiver-webhook` → receiver webhook-driven | salva raw em `integrations.webhook_events` + mapeia → NormalizedTransaction + enfileira jobs (provider=`ingestion`) | URL pública: `core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{slug}` | aceita slug no path ou `?tenant_id=uuid` (legado)
- `sympla-sync` → cron-driven (sem webhooks nativos) | `sympla:sync_orders`, `sympla:sync_participants`
- `webhooks-sendgrid` → receiver webhook-driven O(1) | salva raw em `integrations.webhook_events` + enfileira jobs (`sendgrid:delivery_event`, `sendgrid:engagement_event`, `sendgrid:suppression_event`) | URL: `core.kreatorshub.com.br/functions/v1/webhooks-sendgrid` | verificação ECDSA P-256
- `sendgrid-webhook` → **LEGADO** (desativado — URL migrada para `webhooks-sendgrid`)
- `[outros: oauth, utils]` → funções de suporte (ex: `eduzz-oauth-callback`)

### Convenção de nomenclatura — Edge Functions

| Padrão | Papel | Exemplo |
|---|---|---|
| `{provider}-receiver-webhook` | Recebe requisições HTTP externas do provider. URL pública registrada no painel do provider por tenant. Nunca processa — apenas salva raw e enfileira job. | `eduzz-receiver-webhook` ✅ |
| `{provider}-oauth-callback` | Fluxo OAuth — troca code por token, salva credenciais, dispara import histórico. | `eduzz-oauth-callback` |
| `{provider}-sync` | Sync cron-driven via pg_cron para providers sem webhook nativo. | `sympla-sync`, `doare-sync` |

> **Regra:** o nome da função deve deixar claro o papel sem precisar abrir o código.
> receiver = URL pública, processor = worker interno, sync = cron pull.

### Custom domain — Webhook URLs

**Domínio:** `core.kreatorshub.com.br` (aponta para `pbfpwfkgjaqotbudihpy.supabase.co`)

Todos os webhook URLs gerados pelo frontend usam o domínio customizado. Dois formatos aceitos:
- **Slug (preferencial):** `https://core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{tenant_slug}`
- **Legacy (fallback):** `https://core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook?tenant_id={uuid}`

O receiver resolve o slug via `core.tenants.slug` → `tenant_id`. `getWebhookUrl()` em `eduzzStorage.ts` é async e faz lookup do slug.

### claim_jobs — Filtro de handlers no SQL

Workers passam `p_handlers text[]` direto na chamada RPC `claim_jobs`, evitando claim de jobs que serão descartados em application code. Parâmetro `p_handlers` já existia no RPC mas nenhum worker usava — corrigido em Sprint 10. Ver R23.

**RAILWAY WORKERS** (processamento — Node.js, postgres.js, PgBouncer)

- `ingestion-worker` → `commerce.transactions` + analytics pipeline (handler genérico, provider-agnóstico)
- `analytics-worker` → inteligência e segmentação (segment-eval-loop, refreshRfm, computeClusters, computeLookalikes, refreshFeatures, **project_traits**, **sync_field_definitions**). `refreshRfm` escreve em `contact_state` (`customer_features` dropada em Sprint 8)
- `journeys-worker` → execução de nós de automação (`WORKER_ONLY=true`)
- `journey-scheduler` → enfileira checks periódicos (`IS_SCHEDULER=true`): `checkSegmentEntries`, `checkFormEntries`, `checkEventEntries` (apenas `trigger_type='purchase'`), `enqueueReadyWaits`
- **`journeys-event-worker`** → fan-out push-based de eventos. Processa `event_processing_jobs` + `backfill_jobs`
- `historical-sync-worker` → import histórico e enrich de qualquer provider (`ENABLED_HANDLERS=eduzz:enrich_invoice,eduzz:enrich_sales,eduzz:import_sales_page,eduzz:import_products,sympla:historical_sync`) | `RAILWAY_DOCKERFILE_PATH=historical-sync/Dockerfile`
- `email-campaign-worker` → envio de emails (`prepareEmailBatch`, `sendEmailBatch`, `retrySoftBounce`)
- `analytics-backfill-worker` → jobs de backfill e product linking isolados do pipeline crítico (`ENABLED_JOB_TYPES=link_products`)
- `email-sync-worker` → sincronização de inbox Nylas (`deepSync`, `syncMessage`, `grantReauth`)

**Schema de acesso dos workers (v7.11):** Todos os workers usam `worker.*` para job management (claim, complete, fail, block). Funções de domínio usam o schema nativo: `historical-sync` → `eduzz.*` e `integrations.*`; `email-sync` → `inbox.*`. **Zero stubs `public.*` de worker** — todos removidos.

### Regra de ouro de onde cada coisa fica

| Tipo de trabalho | Onde fica | Por quê |
|---|---|---|
| Receber webhook / pull API | Edge Function | Leve, rápida, sem estado |
| Receber webhook por provider | Edge Function dedicada por provider | Deploy independente, logs isolados, sem risco cruzado |
| Mapear provider → NormalizedTransaction | Edge Function (mapper) | Lógica de provider isolada do banco |
| Criar batch jobs na fila | Edge Function ou pg_cron | Gatilho, não processamento |
| Processar dados → `commerce.*` | ingestion-worker (Railway) | postgres.js, transactions, BEGIN/COMMIT |
| Inteligência / scores / RFM | analytics-worker (Railway) | Batch pesado, longa duração |
| Projeção de traits (formulários) | analytics-worker (Railway) | `project_traits` + `sync_field_definitions` via `analytics.processing_jobs` |
| Import histórico / enrich por provider | historical-sync-worker (Railway) | Batch pesado, longa duração, isolado do hot path |
| Fan-out de eventos → enrollments | journeys-event-worker (Railway) | SLA < 5s, realtime, isolado |
| Executar nós de automação | journeys-worker (Railway) | Estado de enrollment, grafos |
| Enviar email | email-campaign-worker (Railway) | API key, rate limit SendGrid |
| Sincronizar inbox | email-sync-worker (Railway) | API key, longa duração |

---

## 4. O caminho de cada tipo de evento

### 4.1 Webhook / Pull periódico — Fluxo unificado de ingestion

```
Provider
  → Edge Function (receiver)
  → salva raw em provider.raw_table (ledger imutável)
  → mapper: raw → NormalizedTransaction[] (lógica de provider fica aqui — nunca no banco)
  → agrupa em batches (max 100 itens, 512KB, 20 por tenant)
  → INSERT integrations.jobs com handler = 'commerce:process_batch'
  → retorna 200 em < 100ms

ingestion-worker consome integrations.jobs
  → handleProcessBatch — único handler para todos os providers
  → 1 chamada SQL: commerce.process_transactions_batch(tenant_id, transactions[])
  → banco processa todo o batch numa transação
  → loga sumário + grava erros em integrations.job_errors
```

### 4.2 Import histórico (onboarding de novo tenant)

```
Tenant conecta provider
  → Edge Function (oauth callback)
  → descobre volume total (ex: 456 páginas)
  → cria N jobs com scheduled_for espaçados (1 página por job, ~500ms de intervalo)

historical-sync-worker (Railway)
  → busca 1 página da API
  → salva em provider.raw_table
  → mapeia → NormalizedTransaction[]
  → cria job 'commerce:process_batch' para o lote

ingestion-worker → process_transactions_batch → commerce + analytics
```

**Fluxo Eduzz especificamente:**
```
Webhook Eduzz chega em core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{slug}
  → eduzz-receiver-webhook: resolve slug → tenant_id, valida HMAC, salva raw
  → mapeia payload → NormalizedTransaction (direto na Edge Function)
  → enfileira job commerce:process_batch (provider='ingestion', priority=1)
  → ingestion-worker → commerce.transactions
  → [priority 9] job eduzz:enrich_invoice
  → historical-sync-worker → eduzz_upsert_buyer, eduzz_upsert_invoice, enriquecimento crm.party_person
```

### 4.3 Segmentação (quando contact_state muda)

```
Qualquer evento que muda o contact_state
  → INSERT segment_eval_queue com fields_changed e priority (1 para compra, 9 para fallback)

analytics-worker (segment-eval-loop)
  → claim_next_segment_jobs (FOR UPDATE SKIP LOCKED)
  → para cada segmento: avalia depends_on_fields ∩ fields_changed
  → avalia regra em memória contra contact_state
  → apply_segment_membership_diff (adiciona/remove segmentos, atualiza contact_state.segment_ids)
```

**FALLBACK** (pg_cron a cada 5min): detecta contatos com `state_updated_at` recente sem job na fila, enfileira com `priority=9`. Se `fallback_log` tiver registros frequentes → algum handler esqueceu de chamar o analytics pipeline.

### 4.4 Automações / Jornadas

**ENTRADA — como o contato entra na jornada**

- Trigger `segment_entry` → **push-based (Sprint 14):** `refresh_segment_parties_unified` detecta novos membros e enfileira `check_segment_entries` diretamente. Scheduler fallback a cada 5 min (throttled de 60s→5min). Redução ~85% no volume de jobs.
- Trigger `form_entry` → `trigger-form-journey` (Edge Function, push principal) ou scheduler `check_form_entries` (fallback 5 min)
- Trigger `event_entry` → arquitetura push-based (ver seção 4.5) ✅ Operacional desde Sprint 6

**CICLO DE VIDA DA JORNADA**

Três colunas governam o comportamento:
- `activated_at` → primeira ativação (imutável — nunca sobrescrito)
- `last_activated_at` → avança a cada reativação (filtro do checkEventEntries)
- `paused_at` → setado ao pausar, null ao reativar

Trigger no banco (`trg_journey_status_lifecycle`) gerencia essas transições automaticamente. Eventos durante pausa são ignorados porque `checkEventEntries` só processa eventos com `created_at >= journey.last_activated_at`.

**PROCESSAMENTO — como os nós são executados**

journeys-worker worker mode → `claim_next_job` → `processJourneyNode.ts`

- Máximo 50 nós por execução
- Time budget: 3 minutos (re-enfileira se exceder)
- Idempotência via `journeys.journey_executions`
- Tipos de nó: `send_email`, `add_tag`, `remove_tag`, `condition`, `wait`, `ab_split`, `webhook`, `goal`, `exit`, `create_opportunity`, `add_badge`, `remove_badge`, `notify`

**ATOMICIDADE (Sprint 14 — D18/D18b)**

- Enrollment INSERT + process_enrollment job INSERT: `sql.begin()` em 5 entry handlers
- send_email node: 3 INSERTs (campaign_send + recipient + enqueue) em `sql.begin()`
- Safety net: pg_cron `sweep-orphan-enrollments` (*/10 * * * *) detecta enrollments ativos sem job pendente e enfileira recovery com `dedupe_key = 'orphan_' + enrollment_id`

**PERFORMANCE (Sprint 17)**

- Batch claim: `journeys.claim_next_jobs(worker_id, batch_size, job_types, tenant_cap)` com fairness round-robin por tenant via `ROW_NUMBER() OVER (PARTITION BY tenant_id)`. journeys-worker 10 jobs/poll, email-campaign 5 jobs/poll.
- Email paralelo: `p-limit(10)` + rate limiter token bucket (100/sec) + circuit breaker (5 failures). ~50-90 emails/sec. `SEND_CONCURRENCY` configurável.
- Frequency guard (R45): send_email node verifica último email sent antes de enviar. Skip = success (enrollment avança). Decisão persistida em `journey_executions.decision_result` (R42). `EMAIL_FREQUENCY_GUARD_HOURS` configurável, **default=0 (desabilitado, opt-in)**. Automações triggered são contextuais — guard é opt-in via env var no Railway.
- SLA separation (R46): journeys-worker (seconds) e email-campaign-worker (minutes) com zero overlap de job_types.

### 4.5 Trigger event_entry — Arquitetura Push-Based

**Status:** ✅ Operacional

**Princípios do design:**
- `journey_events` é ledger imutável — append-only, NUNCA marcar `processed = true`
- Cada jornada define sua própria janela via `last_activated_at`
- Dedup via UNIQUE em `journey_enrollments`, não via flag no evento

**Fluxo:**
```
Evento ocorre
  → INSERT em journeys.journey_events
  → AFTER INSERT trigger (trg_enqueue_event_processing) insere em journeys.event_processing_jobs [O(1)]
  → journeys-event-worker consome (FOR UPDATE SKIP LOCKED)
  → busca jornadas ativas via routing index (tenant_id + trigger_event_type)
  → para cada jornada match: avalia filtros
  → tenta INSERT enrollment ON CONFLICT DO NOTHING
  → se inseriu: enfileira process_enrollment
  → journeys-worker executa nós
```

**SLA medido:** evento novo → enrollment criado em < 5s.

**Tabelas envolvidas:**
- `journeys.journey_events` → ledger imutável (append-only)
- `journeys.event_processing_jobs` → fila de fan-out
- `journeys.backfill_jobs` → controle de backfill histórico
- `journeys.journey_enrollments` → resultado do processamento
- `journeys.processing_jobs` → execução dos nós (existente, inalterado)

**Routing index:** coluna derivada `trigger_event_type` (`GENERATED ALWAYS AS STORED`) com índice parcial em jornadas ativas para lookup O(log n).

**Event types suportados (Sprint 18):**
| event_type | Trigger SQL | Tabela fonte | Descrição |
|---|---|---|---|
| `purchase` | `trigger_purchase_journey_event()` | `commerce.transactions` | status → `paid` |
| `transaction_pending` | `trigger_purchase_journey_event()` | `commerce.transactions` | status → `pending` (boleto/PIX) |
| `cart_abandonment` | `trigger_cart_abandonment_journey_event()` | `eduzz.cart_abandonments` | Carrinho abandonado |
| `email_opened` | `trigger_email_engagement_journey_event()` | `email.email_events` | Email aberto |
| `email_clicked` | `trigger_email_engagement_journey_event()` | `email.email_events` | Link clicado |
| `form_submitted` | — (scheduler/push) | — | Formulário enviado |

Guards de transição: `purchase` só dispara se OLD.status != 'paid'; `transaction_pending` só dispara se OLD.status != 'pending'. INSERT direto como 'paid' gera APENAS purchase (sem transaction_pending). Pending → paid gera ambos (desejado: jornadas diferentes).

**Backfill — decisão explícita na ativação:** Não é automático. O usuário escolhe ao ativar a jornada: apenas novos eventos (padrão) ou incluir histórico (desde quando?). A fronteira é `backfill_to = last_activated_at` — os dois workers nunca processam o mesmo evento. Rate limiting via `pg_try_advisory_lock` (máximo 1 backfill ativo por tenant). Cursor estável avança por `(created_at, id)` — nunca re-lê histórico completo.

**Shared util — `journeyMatcher.ts`:** `workers/journeys-event/src/shared/journeyMatcher.ts`
Lógica de filtro de jornada compartilhada entre `processEventJob.ts` e `processBackfillJob.ts`.

- `extractMatchContext(eventData)` → extrai `product_id`, `campaign_id`, `url` do evento
- `resolveProductIds(tenantId, ctx)` → resolve IDs reais de `commerce.products` (com cache in-memory por batch)
- `matchesJourneyFilters(conditions, ctx, resolvedIds)` → avalia filtros de produto, campanha e URL

**Scheduler — check_event_entries desabilitado:** A função SQL `journeys.enqueue_journey_checks()` remove `'event_entry'` do loop. O handler `checkEventEntries.ts` permanece no código como **fallback emergencial** — nunca deve ser agendado salvo incidente no journeys-event-worker.

**ATENÇÃO — trigger_type no banco:** O valor correto salvo no banco é `'event_entry'` (não `'event'`). Arquivos relevantes:
- `src/types/journey.ts`: TriggerType inclui `'event_entry'`
- `JourneyBuilder.tsx`: `detectTriggerFromNodes()` salva `trigger_type: 'event_entry'`
- `NodeInspector.tsx`: `sourceType` usa `'event_entry'`

---

## 5. NormalizedTransaction — O contrato entre providers e banco

O banco nunca vê além deste contrato. O mapper de cada provider é responsável por produzir este formato.

**Campos obrigatórios de identidade:**
- `_version: 1` e `_schema: 'normalized_transaction_v1'` — versão desconhecida = erro por item, batch continua
- `external_id` — ID único no provider, chave de idempotência
- `source_name` — `'doare'` | `'eduzz'` | `'hotmart'` | `'stripe'`
- `source_type` — `'webhook'` | `'integration'` | `'manual'`
- `email` — chave de upsert do contato

**Campos opcionais de contato:** `name`, `phone`, `document` (CPF/CNPJ — normalizar no mapper), `city`, `state` (nome completo ou sigla — banco aceita ambos)

**Campos obrigatórios de transação:**
- `amount` — SEMPRE em reais, NUNCA centavos. Usar `Math.round(x * 100) / 100` para evitar float
- `currency` — `'BRL'` default
- `status` — `'paid'` | `'pending'` | `'cancelled'` | `'refunded'` | `'chargeback'`

**Campos opcionais de transação:** `paid_at` (ISO 8601), `product_id`, `product_name`, `utm_source`, `utm_medium`, `utm_campaign`

**Campo de auditoria obrigatório — `raw_source`:**

```
Regra para escala:

- Provider COM schema raw dedicado (doare, eduzz, e todo provider novo):
  raw_source deve ser omitido. A rastreabilidade é garantida via
  external_transaction_id → provider.raw_table. Duplicar o payload
  no job causa bloat e degrada performance em escala.

- Provider SEM schema raw dedicado (integrações pontuais, manual):
  raw_source obrigatório. É a única fonte de auditoria disponível.
  Mas todo provider novo deve ter schema raw dedicado (ver seção 14) —
  raw_source neste caso é temporário até o schema existir.
```

### Mapeamento de status por provider

| Status no provider | Status normalizado |
|---|---|
| Pago, approved, paid, completed | `paid` |
| Pendente, pending, waiting | `pending` |
| Cancelado, cancelled, voided | `cancelled` |
| Reembolsado, refunded, reversed | `refunded` |
| Chargeback, disputed | `chargeback` |

---

## 6. commerce.process_transactions_batch — A RPC genérica

Uma única RPC que processa qualquer provider. Nunca precisa ser modificada para novos providers.

**Assinatura:**
```sql
commerce.process_transactions_batch(p_tenant_id uuid, p_transactions jsonb)
RETURNS jsonb
-- Retorna: { processed, skipped, errors: [{external_id, error_code, message}] }
```

**Garantias internas:**

1. Para cada transação do batch, valida versão do contrato — versão desconhecida gera erro por item, batch continua
2. Captura o status anterior da transação se ela já existir (para detectar transição real)
3. **Role assignment condicional:** `paid` + `amount > 0` → `ARRAY['customer']`; caso contrário → `ARRAY['lead']`. Transações gratuitas (R$0) **nunca** atribuem role customer — contato permanece como lead.
4. Faz INSERT com `ON CONFLICT DO UPDATE` — transições unidirecionais: `paid` permanece `paid` exceto para `refunded`; qualquer outro status pode transicionar para `paid`
5. `paid_at` é preenchido via `COALESCE` — nunca sobrescreve se já existia
6. Se o status não mudou (`v_previous_status = v_status`) → skip, não roda analytics
7. Analytics pipeline só roda na transição PARA `paid`: novo insert `paid`, ou mudança de `pending`/`cancelled` para `paid`

---

## 7. analytics.process_purchase_analytics — Pipeline consolidado

Uma única RPC executada dentro do loop de `process_transactions_batch`. Substitui 4 chamadas separadas.

**Executa 4 operações em sequência na mesma transação:**

1. `INSERT contact_events` — ledger imutável (idempotência por `transaction_id`)
2. `INSERT contact_state ON CONFLICT DO UPDATE` — snapshot vivo
3. `INSERT segment_eval_queue priority=1` — fila de segmentação
4. `INSERT tenant_distribution ON CONFLICT` — marca tenant como stale

---

## 8. Ciclo de vida dos Jobs

**Estados:** `pending` → `running` → `success` / `retry` / `blocked` / `dead`

**Classificação de erros:**

| Classe | Error codes | Comportamento | Resolução |
|---|---|---|---|
| Contract bug | `unknown_contract_version`, `missing_email` | job → `blocked` | fix mapper + deploy + `reprocess_blocked_jobs` |
| Data invalid | `missing_product_name`, `invalid_amount` | item → `job_errors`, job continua | corrigir dado na fonte |
| Infra | `deadlock (40P01)`, timeout, conexão | job → `retry` com backoff exponencial | automático |

**Blocked jobs — fluxo operacional:**

1. Contract error detectado → worker chama `blockJob()` (não `failJob`) → `job.status = 'blocked'`, para de ser reclamado
2. Alerta dispara
3. Fix do mapper → commit → deploy Railway
4. Executar `reprocess_blocked_jobs` com filtros opcionais (`error_code`, `source_name`, `tenant_id`, `job_id`) → jobs voltam para `'pending'`, `attempts = 0`

**RPCs de lifecycle:**

| RPC | O que faz |
|---|---|
| `worker.claim_jobs(worker_id, batch_size, provider, handlers)` | Clama jobs para processamento (SKIP LOCKED) |
| `worker.complete_job(job_id, result)` | Marca success |
| `worker.fail_job(job_id, error)` | Retry com backoff exponencial ou dead |
| `worker.block_job(job_id, reason)` | Bloqueia job sem consumir attempts |
| `integrations.reprocess_blocked_jobs(...)` | Reactiva jobs blocked após fix |

**Estrutura de arquivos do ingestion-worker:**
```
workers/ingestion/src/
├── handlers/
│   ├── commerce/
│   │   └── processBatch.ts        ← handler genérico (não muda nunca)
│   ├── doare/
│   │   ├── mapTransactions.ts     ← mapper Doare → NormalizedTransaction
│   │   └── sync.ts                ← pull API Doare
│   ├── eduzz/
│   │   ├── mapTransactions.ts     ← mapper Eduzz → NormalizedTransaction
│   │   └── processInvoice.ts      ← DEPRECATED — fluxo migrado para Edge Function eduzz.ts
│   └── [hotmart]/
│       └── mapTransactions.ts     ← quando chegar: só criar este arquivo
├── types/
│   └── NormalizedTransaction.ts   ← interface compartilhada
└── worker.ts                      ← BATCH_SIZE + CONCURRENCY via env vars
```

**Estrutura de arquivos do journeys-event-worker:**
```
workers/journeys-event/src/
├── config.ts
├── db.ts
├── logger.ts
├── index.ts                       ← loop principal: event jobs (1s) + backfill tick (5s)
├── handlers/
│   ├── processEventJob.ts         ← fan-out: evento → jornadas matching → enrollments
│   └── processBackfillJob.ts      ← backfill histórico com cursor estável
└── shared/
    └── journeyMatcher.ts          ← filtros compartilhados (produto, campanha, url)
```

**Configuração de performance (Railway env vars):**

| Service | BATCH_SIZE | CONCURRENCY | Pool max |
|---|---|---|---|
| ingestion-worker | 25 | 10 | 12 |
| journeys-event-worker | 20 | 5 | — |

---

## 9. Rate Limiting e Isolamento por Tenant

**Camada 1 — Criação dos jobs (Edge Function):** Max 20 transações por tenant por batch. Tenant com 500 pendentes = 25 jobs de 20, processados em rounds. Nunca monopoliza a fila.

**Camada 2 — Claim da fila (banco):** `FOR UPDATE SKIP LOCKED` + `PARTITION BY tenant_id ORDER BY priority, created_at`. Fairness entre tenants no mesmo worker. 429 do provider: job reagenda com backoff exponencial (60s→86400s + jitter 20%).

**Tenant problemático (loop de automação):** Max 50 nós por execução, time budget 3 min com re-enfileiramento, `releaseStaleClaims()` a cada 60s.

**journeys-event-worker — backfill por tenant:** Advisory lock via `pg_try_advisory_lock` (namespace `987654321`) garante máximo 1 backfill ativo por tenant. Batch de 500 eventos com cursor estável. Máximo 3 backfills simultâneos por tick (5s).

---

## 10. Filas do Sistema

| Fila | Tabela | Quem produz | Quem consome |
|---|---|---|---|
| Ingestion jobs | `integrations.jobs` | Edge Functions (mappers), pg_cron | ingestion-worker |
| Segment eval | `analytics.segment_eval_queue` | `process_purchase_analytics`, `analytics-events.ts`, fallback | analytics-worker |
| Journey processing | `journeys.processing_jobs` | journeys-worker scheduler, journeys-event-worker | journeys-worker |
| **Event processing** | **`journeys.event_processing_jobs`** | **`trg_enqueue_event_processing` (trigger AFTER INSERT)** | **journeys-event-worker** |
| **Backfill** | **`journeys.backfill_jobs`** | **Ativação manual de jornada** | **journeys-event-worker** |
| Email sending | `email.campaign_sends` | journeys-worker (`send_email` node) | email-campaign-worker |
| Analytics jobs | `analytics.jobs` | analytics-worker scheduler | analytics-worker |
| Analytics processing | `analytics.processing_jobs` | Edge Functions (form submit trigger), backfill scripts | analytics-worker (`project_traits`, `sync_field_definitions`, `link_products`) |
| Job errors | `integrations.job_errors` | `process_transactions_batch` (erros por item) | observabilidade + reprocess |

---

## 11. pg_cron — Jobs Agendados (auditados)

Auditoria arquitetural completa em v7.22. Cada cron é classificado por categoria
(R50): **A** = housekeeping puro (manter); **B** = enqueue com lógica embutida
(refatorar para worker no Railway); **C** = lógica de domínio inline (substituir
totalmente); **D** = caduco (deletar). Cron novo requer entrada nesta tabela —
sem registro = violação arquitetural.

| jobname | schedule | command (resumido) | função SQL invocada | categoria | nota |
|---|---|---|---|---|---|
| `archive-opportunity-activities` | `0 2 * * 0` | INSERT archive + DELETE source TTL 2y | inline | **A** | Banco-only. Monitorar lock duration se volume crescer. |
| `cleanup-asaas-webhook-events` | `0 4 * * *` | DELETE TTL 90d processed | inline | **A** | TTL puro. |
| `cleanup-backfill-jobs` | `0 3 * * *` | DELETE TTL 30d completed | inline | **A** | TTL puro. |
| `cleanup-event-processing-jobs` | `0 3 * * *` | DELETE TTL 7d success | inline | **A** | TTL puro. |
| `cleanup-journey-processing-jobs` | `0 4 * * *` | DELETE TTL 7d completed/failed | inline | **A** | TTL puro. |
| `cleanup-old-analytics-jobs` | `0 4 * * *` | DELETE TTL 7d completed/failed | inline | **A** | TTL puro. |
| `purge-eval-queue` | `15 * * * *` | DELETE TTL 4h processed | `analytics.purge_processed_eval_queue` | **A** | TTL puro (R20). |
| `purge-integrations-jobs` | `30 3 * * *` | DELETE TTL 7d/30d | `integrations.purge_old_jobs` | **A** | TTL puro com cleanup de `job_errors`. |
| `purge-integrations-webhook-events` | `45 3 * * *` | DELETE TTL com guard `NOT EXISTS jobs` | `integrations.purge_old_webhook_events` | **A** | TTL puro. |
| `expire-soft-bounce-suppressions` | `0 * * * *` | UPDATE TTL `expires_at <= NOW()` | `email.expire_soft_bounce_suppressions` | **A** | TTL puro. |
| `create-contact-events-partition` | `0 2 25 * *` | DDL `CREATE TABLE PARTITION` idempotente | `analytics.create_contact_events_partition_if_needed` | **A** | DDL maintenance. |
| `etl-event-health-metrics` | `5 * * * *` | Pulse ETL hourly: agrega `fallback_log` + `contact_events` + `segment_eval_queue` em `event_health_metrics` via `emit_health_metric` | `analytics.etl_event_health_metrics` | **A** | D-CRON-3 (v7.25). Banco-only, sem lógica de domínio. Substituível por workers em D-CRON-4 sem mudar view/handler (R52). |
| `event-health-metrics-partition-create` | `30 2 25 * *` | DDL idempotente — cria partição mensal de `event_health_metrics` | `analytics.event_health_metrics_partition_create_if_needed` | **A** | D-CRON-3 (v7.25). Mesmo padrão de `create-contact-events-partition`. |
| `event-health-metrics-purge-old` | `30 4 * * *` | DROP partition para retention 90d em `event_health_metrics` | `analytics.event_health_metrics_purge_old` | **A** | D-CRON-3 (v7.25). TTL puro via DROP (mais barato que DELETE). |
| `sweep-orphan-enrollments` | `*/10 * * * *` | Re-enqueue de enrollments órfãos LIMIT 100 | `journeys.sweep_orphan_enrollments` | **A** | Safety net pós D18 (fanout não-atômico). Sem lógica de domínio. **TODO:** alarmar quando `count > 0` indica bug em fanout. |
| `rfm-rolling-refresh` | `0 * * * *` | Enqueue `refresh_features` para 1/24 dos tenants/hora com threshold 20h via `MOD(HASHTEXT(t.id), 24) = HOUR(NOW())` | `analytics.enqueue_job_internal` | **B** | Loop bloqueante de tenants no banco (viola §2.4). Lógica de sharding e threshold em SQL. **D-CRON-2** — migrar para worker. |
| `segment-eval-fallback` | `*/5 * * * *` | Itera tenants ativos, detecta contatos com `state_updated_at < 15min` sem job pending, enqueue na `segment_eval_queue` | `analytics.run_segment_eval_fallback` | **B** | Safety net pro event-driven path. Loop bloqueante de tenants (viola §2.4). Mascara bugs do path event-driven sem alarme. **D-CRON-3** (instrumentar `analytics.fallback_log`) é pré-requisito de **D-CRON-4** (migrar para worker). |

> **Pull-based providers (Sympla, Doare):** sync periódico gerenciado pelo pull-sync-worker via `integrations.sync_schedules`. Não usam pg_cron.

> **R50 (v7.22):** pg_cron é APENAS para housekeeping (DELETE/UPDATE de TTL, DDL maintenance, pulse simples sem lógica de domínio). Lógica com tenant awareness, sharding heurístico, thresholds dinâmicos ou INSERT em tabelas de domínio = worker no Railway. Cron novo requer entrada nesta tabela + classificação A.

---

## 12. contact_state — O Prontuário do Contato

Uma linha por contato. É o que todo o sistema consulta para tomar decisões. Nunca fazer join em runtime para dados de comportamento — tudo vem daqui.

No campo `lifecycle_stage`, adicionar `dormant` à lista de valores válidos: `never_purchased/new/active/cooling/at_risk/dormant/churned`.

| Categoria | Campos |
|---|---|
| RFM | `recency_days`, `frequency`, `monetary`, `r_score`, `f_score`, `m_score` (1-5), `rfm_segment` |
| Valor | `value_tier` (platinum/gold/silver/bronze), `lifecycle_stage` (never_purchased/new/active/cooling/at_risk/churned) |
| Comportamento | `bought_product_ids[]`, `tag_ids[]`, `responded_form_ids[]`, `import_ids[]`, `segment_ids[]` |
| Engajamento | `email_open_rate_30d`, `engagement_score` (0-100), `last_purchase_at` |
| Preditivo | `churn_score` (0-1), `purchase_propensity` (0-1), `next_best_product_id` |
| Aquisição | `acquisition_source`, `utm_source/medium/campaign`, `first_touch_revenue` |

**contact_state como tabela quente:** Com 5M contatos e updates constantes, requer `fillfactor = 70` (espaço para HOT updates), autovacuum agressivo (`vacuum_scale_factor = 0.01`, `analyze_scale_factor = 0.005`), e monitoramento de dead tuples (alerta se `dead_pct > 10%`).

**Como o contact_state é atualizado:** Toda mutação deve: (1) atualizar `contact_state`, (2) registrar em `contact_events`, (3) inserir em `segment_eval_queue` com `fields_changed`. Sempre os três juntos, sempre em `BEGIN/COMMIT`. Nunca atualizar `contact_state` diretamente sem passar por essas funções.

> **Fronteira permanente do contact_state:** Somente dados comportamentais
> derivados pertencem a contact_state — RFM, lifecycle, scores, arrays de
> membership, datas derivadas de comportamento. Campos de perfil (phone,
> profession, birth_date) vivem em `crm.party_person` / `crm.party_profile`.
> Campos customizados do tenant vivem em `crm.party_trait_*`. Materializar
> campos de perfil ou traits em contact_state é violação arquitetural (ver R30).

### Campos array — estado dos producers

| Campo | Producer | Status |
|---|---|---|
| `bought_product_ids` | `process_purchase_analytics` + `linkProducts.ts` (recalcula array completo após link) | ✅ Ativo |
| `tag_ids` | Triggers DB em `crm.customer_tags` + `crm.lead_tags` (after insert/delete) | ✅ Ativo |
| `import_ids` | Backfill aplicado — producer tempo real pendente (D8) | ✅ Populado |
| `responded_form_ids` | Backfill aplicado — producer tempo real pendente (D7) | ✅ Populado |
| `journey_ids` | — | ❌ 0% populado — producer não existe (D9) |
| `segment_ids` | `apply_segment_membership_diff` | ✅ Ativo |

> **Regra permanente:** campo array vazio = segmento que usa esse campo retorna zero membros silenciosamente, sem erro. Antes de migrar qualquer condition para GIN, validar divergência = 0 contra a fonte normalizada.

**Mapeamento evento → fields_changed:**

| Evento | fields_changed | Priority |
|---|---|---|
| `purchase` | `monetary, frequency, recency_days, rfm_segment, bought_product_ids, last_purchase_at, r/f/m_score, value_tier, lifecycle_stage, churn_score, purchase_propensity` | 1 |
| `tag_added/removed` | `tag_ids` | 3 |
| `form_submitted` | `responded_form_ids` | 3 |
| `rfm_recalculated` | `recency_days, frequency, monetary, rfm_segment, r/f/m_score, value_tier, lifecycle_stage` | 5 |
| `profile_updated` | `name, phone, state, city, profession, birth_date` | 7 |
| `import_added` | `import_ids` | 8 |
| `fallback` | todos os campos | 9 |

---

## 13. CRM — Oportunidades e Timeline

O handler `executeCreateOpportunityNode` (em `processJourneyNode.ts`) cria ou atualiza oportunidades via automação.

**Dedup:** busca oportunidade `status='open'` no mesmo `pipeline_id` pelo contato. Chave preferencial e única: `party_id`. Fallback `contact_email` removido — migração party-first completa.

**Se encontrou:** calcula diff de campos → UPDATE oportunidade → INSERT em `crm.opportunity_stage_history` se etapa mudou → INSERT em `crm.opportunity_activities` → retorna `opportunity_updated`.

**Se não encontrou:** cria nova oportunidade → INSERT em `crm.opportunity_activities` com `activity_type='created'` → retorna `opportunity_created`.

**Regras da timeline (`crm.opportunity_activities`):**
- Append-only — nunca fazer UPDATE
- Idempotência via UNIQUE INDEX parcial em `(enrollment_id, node_id)` — reprocessamento não duplica
- Backfill usa `source='system'` com `source_meta=null` para não colidir com INSERTs de jornada
- Activity é obrigatória: se o INSERT falhar, o node falha inteiro e o journeys-worker faz retry — nunca usar `try/catch` isolado no INSERT de activity

`email_opened` / `email_clicked` — emitidos por trigger AFTER INSERT em `email.email_events` (resolve party via `recipient_id → campaign_recipients → crm.parties`)

`segment_entered` / `segment_exited` — emitidos por `apply_segment_membership_diff` a cada mudança de membership

`crm.opportunities.party_id` é NOT NULL enforçado — oportunidade sem contato é violação do modelo party-first. Trigger `sync_opportunity_summary` removido (Sprint 11). Colunas `has_open_opportunity` e `open_opportunity_count` dropadas de `contact_state` — violavam R36 (1:N nunca denormalizado). Substituídas por 10 campos `opportunity_exists` no worker planner, avaliados via EXISTS subquery direta em `crm.opportunities` com índice `idx_opportunities_party_tenant_status`. Ver R36 para lista completa de campos.

---

## 14. Checklist para Novo Provider

Adicionar um provider novo é criar **1 arquivo de mapper**. Zero mudança no banco.

**Edge Function:**

- [ ] Criar Edge Functions seguindo convenção de nomenclatura (ver R22):
  ```
  supabase/functions/{provider}-receiver-webhook/   ← URL pública, recebe do provider
  supabase/functions/{provider}-processor-worker/   ← worker interno, processa jobs
  supabase/functions/{provider}-oauth-callback/     ← se provider usa OAuth
  ```
- [ ] Criar `integrations.invoke_<provider>()` apontando para `/functions/v1/<provider>`
- [ ] Adicionar ELSIF no `enqueue_webhook_job` (webhook) ou invoke no cron (pull)
- [ ] Receber ou buscar dados brutos
- [ ] Salvar no schema do provider (`provider.raw_table`) — ledger imutável
- [ ] Criar `mapTransactions.ts`: raw → `NormalizedTransaction[]` com `_version: 1`
- [ ] Atenção no mapper: `amount` sempre em reais, status normalizado, `paid_at` em ISO 8601
- [ ] Agrupar em batches: max 100 itens, 512KB, 20 por tenant
- [ ] `INSERT integrations.jobs` com `handler='commerce:process_batch'`
- [ ] Se pull: respeitar rate limit via `scheduled_for` nos jobs
- [ ] Se webhook: salvar em `integrations.webhook_events` com `idempotency_key`

**Banco:**

- [ ] Schema `provider.*` com tabela de dados brutos
- [ ] Schema raw dedicado vira item obrigatório, não opcional
- [ ] `UNIQUE (tenant_id, provider_id)` em toda tabela raw
- [ ] Se precisa de sync periódico: adicionar pg_cron
- [ ] Registrar `source_name` em lowercase nos providers cadastrados

**NÃO precisa:**
- ~~Criar nova RPC no banco~~
- ~~Criar novo handler no ingestion-worker~~
- ~~Modificar `commerce.process_transactions_batch`~~

> `source_name` deve ser sempre o nome canônico do provider em lowercase — `'eduzz'`, `'doare'`, `'hotmart'`. Nunca usar variantes como `eduzz_import`, `eduzz_historical`, `eduzz_backfill`. Um provider = um `source_name` fixo para todos os pipelines (webhook, import histórico, reprocessamento). Variação de `source_name` no mesmo provider quebra o dedup cross-source.

> **Provider pull-based — padrão Sympla:** Providers sem webhook nativo usam cron-driven sync via Edge Function. Credencial por tenant (`s_token`) armazenada em `integrations.accounts.config`. Hierarquia Tenant → Events → Orders → Participants. Sync de participants usa janela de ciclo de vida: futuro (enrich de contato), ativo (-1d a +3d, checkins em tempo real), encerrado recente (sync final, seta `checkin_synced = true`), arquivado (nunca re-fetch). Datas da API sem offset tratadas como `America/Sao_Paulo`, nunca UTC.

---

## 14.1 Provider Sympla — Detalhes de Implementação

**Status:** ✅ Operacional (Sprint 10) — Fases 1–4 completas

**Tipo:** Pull-based (sem webhooks nativos). Credencial `s_token` por tenant em `integrations.accounts.config`.

**Hierarquia de dados:**

```
Tenant
└── Events (GET /events)
    └── Orders (GET /events/{id}/orders)          ← transações
    └── Participants (GET /events/{id}/participants)  ← contatos + checkin
```

**Schema no banco:** `sympla.*`
- `sympla.events` — cache de eventos com campos completos (host, category, address, `checkin_synced`)
- `sympla.orders` — pedidos com invoice_info (CPF ~12%, address ~24%), UTM, user_agent
- `sympla.participants` — participantes com phone extraído de `custom_form`, checkin, `checkin_event_emitted`
- `sympla.sync_cursors` — cursor por evento para sync incremental

**Cobertura de dados (tenant de referência):**

| Campo | Cobertura | Origem |
|---|---|---|
| email | 100% | orders.buyer_email |
| city/state | 23.5% | orders.invoice_info (só boleto/cartão com NF) |
| CPF | 12.3% | orders.invoice_info (boleto 100%, PIX/cartão opcional) |
| phone | 52.6% | participants.custom_form |
| checkin | 31.5% | participants.checkin array |

**Mapeamento de status:**

| Status Sympla | Status normalizado |
|---|---|
| `A` | `paid` |
| `P`, `NP` | `pending` |
| `NA`, `C` | `cancelled` |
| `R` | `refunded` |

**Datas:** API retorna sem offset — tratar como `America/Sao_Paulo` — appender `-03:00` no parse, nunca `Z`.

**Janela de sync de participants:**

| Janela | Critério | Ação |
|---|---|---|
| Futuro | `start_date > hoje` | Fetch para enrich de contato |
| Ativo | `-1d ≤ start_date ≤ +3d` | Fetch frequente — checkins em tempo real |
| Encerrado recente | `+3d < start_date ≤ +7d` AND `checkin_synced = false` | Fetch final → `checkin_synced = true` |
| Arquivado | `checkin_synced = true` | Nunca re-fetch |

**Handler histórico:** `sympla:historical_sync` no historical-sync-worker. Busca todos os eventos/orders/participants sem janela temporal. Atualiza `sync_cursors` após processar para que o pull-sync-worker não reprocesse. Enfileira `commerce:process_batch` (provider=`ingestion`).

**Fases pendentes:**
- Fase 5: checkin → `journeys.journey_events` (type `sympla_checkin`)
- Fase 6: `contact_state.event_checkin_count` + `analytics.contact_checkin_stats`

---

## 14.2 Provider Eduzz — Detalhes de Implementação

**Status:** ✅ Operacional — Webhook-driven + OAuth + import histórico

**Tipo:** Webhook-driven (real-time). Credenciais OAuth + webhook_secret em `integrations.accounts.config`.

**Edge Functions:**
- `eduzz-receiver-webhook` — URL pública, recebe webhooks da Eduzz. Valida HMAC (`x-eduzz-signature`), salva raw em `integrations.webhook_events`, mapeia → NormalizedTransaction, enfileira job com `provider = 'ingestion'`.
- `eduzz-oauth-callback` — Fluxo OAuth, troca code por token, dispara import histórico.

**OAuth — app centralizado KreatorsHub:**
- `client_id` e `client_secret` em env vars (`EDUZZ_CLIENT_ID`, `EDUZZ_CLIENT_SECRET`) — nunca por tenant
- `access_token` de longa duração — API Eduzz não retorna `refresh_token` nem `expires_in`
- Callback salva: `access_token`, `oauth_connected_at`, `oauth_user_id`
- Import histórico disparado automaticamente no callback via `createHistoricalImportJobs()`

**Webhook verification:**
- Ping test: receiver responde 200 + grava `webhook_verified_at = now()`
- Eventos reais: validação HMAC com Origin Key (`webhook_secret`)
- UI faz polling a cada 5s enquanto `webhook_verified_at IS NULL` — badge muda para verde automaticamente

**Token expirado — fluxo defensivo:**
```
Worker recebe 401
  → tenta refresh (1x) → falha (sem refresh_token)
  → invalidateEduzzToken(): access_token = null, oauth_connected_at = null
  → throw TokenExpiredError
  → blockJob('TOKEN_EXPIRED')
  → UI detecta oauth_connected_at = null → banner "Conexão expirada — reconectar"
```

**Import histórico — Phase 2 (D12 implementado):**
```
eduzz:import_sales_page
  Fase 1: fetch API → eduzz.buyers + eduzz.invoices (raw ledger)
  Fase 2: paid invoices → NormalizedTransaction[]
    → jobs commerce:process_batch (provider='ingestion', priority=8)
    → ingestion-worker processa igual webhook
```

**Webhook URL (tenant-facing):**
- Formato slug: `https://core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{tenant_slug}`
- Formato legacy: `https://core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook?tenant_id={uuid}`

**Schema no banco:** `eduzz.*`
- `eduzz.invoices` — invoices completas com dados de buyer, affiliates
- `eduzz.buyers` — compradores com dados de contato
- `eduzz.contracts` — contratos/assinaturas

**Handlers:**

| Event type | Handler |
|---|---|
| `myeduzz.invoice_*` (paid, opened, canceled, refunded, chargeback, etc.) | `eduzz:process_invoice` |
| `myeduzz.contract_*` | `eduzz:process_contract` |
| `sun.cart_abandonment` | `eduzz:process_cart_abandonment` |
| `myeduzz.commission_processed` | `eduzz:process_commission` |
| `nutror.*` (enrollment, lesson, module, certificate) | `eduzz:process_learning` |
| `ping` | `eduzz:ping` |

**Prioridades:** `invoice_paid` = 1, `refunded`/`chargeback` = 2, `cart_abandonment` = 3, default = 5.

**Railway worker:** `historical-sync-worker` — import histórico e enrich (`ENABLED_HANDLERS=eduzz:enrich_invoice,eduzz:enrich_sales,eduzz:import_sales_page,eduzz:import_products,sympla:historical_sync`).

---

## 15. Checklist para Novo Nó de Automação

- [ ] Implementar handler em `workers/journeys/src/handlers/nodes/`
- [ ] Registrar em `processJourneyNode.ts`
- [ ] Implementar idempotência via `journeys.journey_executions`
- [ ] Se modifica o contato: chamar `analytics-events.ts` para atualizar `contact_state`
- [ ] Se envia comunicação: delegar para worker especializado (email, WhatsApp)
- [ ] Se tem estado temporal (wait): salvar `wait_until`, scheduler acorda
- [ ] Se cria/modifica entidade CRM: registrar em tabela de activities correspondente
- [ ] Se tem condição/branch: usar `evaluateConditionSingle()` do Condition Engine (R40)
- [ ] Se tem decisão: persistir em `journey_executions` (`decision_result`, `decision_input_snapshot`) (R42)
- [ ] Se tem skip: registrar nós pulados em `journey_executions` com razão

---

## 16. Deploy no Railway — Configuração por Service

Todos os services usam build context = raiz do repo. Cada service define seu Dockerfile via env var `RAILWAY_DOCKERFILE_PATH`.

**Separação de workers por perfil, não por provider:** Worker único com múltiplas réplicas é o modelo correto para escala. Separar só quando perfis divergem em SLA: realtime (webhooks, segundos) vs batch (imports, minutos). `journeys-event-worker` foi separado do `journeys-worker` por ter SLA < 5s vs minutos.

---

## 17. Observabilidade — O que monitorar

### Filas e saúde do sistema

Monitorar regularmente: `integrations.jobs`, `analytics.segment_eval_queue`, `journeys.processing_jobs`, `journeys.event_processing_jobs`, `email.campaign_sends` — todos por status. Verificar `contact_state`: total de contatos, quantos com `rfm_segment` preenchido, timestamp do update mais recente. Verificar `fallback_log` — se > 0 na última hora, algum handler esqueceu o analytics pipeline.

**Separação de workers por perfil, não por provider.** Mesma imagem Docker, env vars diferentes. Usar `ENABLED_JOB_TYPES` para restringir job types por worker. Usar `WORKER_ONLY` / `IS_SCHEDULER` para separar execução de agendamento. Separar em workers distintos apenas quando SLAs divergem (ex: realtime < 5s vs batch em minutos). Configs detalhadas de cada service vivem no Railway.

### Alertas — se diferente de zero

| Query | O que indica |
|---|---|
| `fallback_log` com registros em 1h | Handler esqueceu de chamar analytics pipeline |
| `segment_eval_queue` com `status='error'` | Erro no evaluator |
| `integrations.jobs` com `status='blocked'` | Mapper/contrato quebrado — requer fix + deploy |
| `integrations.jobs` com `status='dead'` | Esgotou tentativas — verificar `last_error` |
| `journey.processing_jobs pending` > 10min | Journey worker parado |
| `max(now()-state_updated_at)` em `contact_state` > 30min | Pipeline atrasado |
| `job_errors` crescendo com mesmo `error_code` | Erro sistemático — verificar mapper |
| **`event_processing_jobs pending` > 2min** | **journeys-event-worker parado** |
| **`event_processing_jobs` com `status='blocked'`** | **`trigger_config` inválido — requer fix** |
| **Latência P95 evento→enrollment > 5s** | **Worker com sobrecarga ou deadlock** |
| **`backfill_jobs` sem atualização em 10min** | **Backfill travado** |

### Queries de observabilidade — journeys-event-worker

Ver `workers/journeys-event/src/monitoring.sql` para queries completas:

1. Throughput por minuto (última hora)
2. Latência P50/P95/P99
3. Fila atual por status
4. Jobs blocked por motivo
5. Jobs dead (últimos 20)
6. Backfills ativos
7. Alerta: pending > 2min
8. Alerta: P95 > 5s

---

## 17.1 Platform Health Observability (D-CRON-3)

**Status:** ✅ Operacional (v7.25).

Camada própria de observability arquitetural — desacoplada do mecanismo observado
(R52). Mudança no path observado (ex: `segment-eval-fallback` cron → worker em
D-CRON-4) NÃO obriga reescrever a view nem o handler de alerta.

### Arquitetura — três camadas decoupled

```
┌──────────────────────────────────────────────────────────────┐
│  Camada A — TELEMETRIA                                       │
│  analytics.event_health_metrics (particionada mensal, 90d)   │
│  Populada por:                                               │
│    • cron `etl-event-health-metrics` (hourly, categoria A)   │
│    • workers via RPC analytics.emit_health_metric()          │
│      ex: segment-eval-worker emite `segment_membership_      │
│          changes` em batch 60s                               │
└──────────────────────────────────────────────────────────────┘
                          ↓ lida APENAS por
┌──────────────────────────────────────────────────────────────┐
│  Camada B — SAÚDE                                            │
│  analytics.v_event_driven_health (view)                      │
│  Tier classification per-tenant: HEALTHY / WARN / ERROR /    │
│  CRITICAL / INSUFFICIENT_DATA                                │
│  Baseline: rolling 7d P50/P95/P99 do PRÓPRIO tenant,         │
│  excluindo a hora atual (evita auto-inflação).               │
│  Whitelist via core.platform_health_whitelist.               │
└──────────────────────────────────────────────────────────────┘
                          ↓ lida APENAS por
┌──────────────────────────────────────────────────────────────┐
│  Camada C — ALERTA                                           │
│  workers/journeys/src/handlers/healthCheck.ts (5min cadence) │
│  Emite Sentry em transição p/ tier pior. Recovery silencioso │
│  (HEALTHY sustentado 30min). Dedup via                       │
│  analytics.platform_alert_state.                             │
└──────────────────────────────────────────────────────────────┘
```

### Tier classification

| Condição | Tier |
|---|---|
| baseline samples < 168 (7d × 24h) | `INSUFFICIENT_DATA` |
| `error_sustained_1h` E `hits_1h > p99` | `CRITICAL` |
| `hits_1h > p95` | `ERROR` |
| `hits_1h > p50 * 1.5` | `WARN` |
| caso contrário | `HEALTHY` |

### Métricas coletadas (v7.25)

| metric_name | semântica | fonte v7.25 | fonte D-CRON-4 |
|---|---|---|---|
| `fallback_hits` | contacts detectados pelo cron `segment-eval-fallback` | ETL hourly de `analytics.fallback_log` | worker emite direto (cron some) |
| `event_driven_hits` | event-driven entries no segmento | ETL hourly de `analytics.contact_events` (event_type='segment_entered', source='system') | inalterado |
| `eval_queue_throughput` | jobs com status='processed' por hora | ETL hourly de `analytics.segment_eval_queue` | inalterado |
| `eval_queue_error_rate` | errors / (errors + processed) por hora (mín 10 jobs) | ETL hourly de `analytics.segment_eval_queue` | inalterado |
| `eval_queue_p95_latency_ms` | p95 latency `processed_at - claimed_at` (ms) | ETL hourly de `analytics.segment_eval_queue` | inalterado |
| `segment_membership_changes` | sum(entered + exited) por bucket hora | `segment-eval-worker` emite em batch 60s direto via `emit_health_metric` | inalterado |

### Regras de alerta (Camada C)

1. **WARN não dispara Sentry.** Tier informativo apenas — handler trackeia state mas não notifica.
2. **ERROR/CRITICAL alertam em transição PARA pior** (HEALTHY/WARN/null → ERROR/CRITICAL OU ERROR → CRITICAL).
3. **Re-alerta no mesmo tier ≥ ERROR só após 1h.** Evita alert fatigue.
4. **Transição CRITICAL → ERROR não re-alerta** (tier melhor sem ser HEALTHY).
5. **Recovery silencioso:** ERROR/CRITICAL → HEALTHY sustentado 30min apenas atualiza state + log info. Sem Sentry. Auditoria via `platform_alert_state.last_changed_at`.
6. **Whitelist:** `core.platform_health_whitelist` (auditável, com `reason` CHECK constraint, `added_by`, `expires_at`). Começa vazia em v7.25. Adicionar tenant nominalmente via migration.

### Pré-requisito de produção

Sentry DSN do project `workers-railway` deve estar em `SENTRY_DSN_WORKERS` no
Railway. Se ausente, `initSentryWorker` loga `sentry_disabled` e o handler segue
funcionando — escreve em `platform_alert_state` + log estruturado, mas não emite
no Sentry. Permite deploy do código antes do DSN estar configurado.

### Arquivos

| Arquivo | Papel |
|---|---|
| `supabase/migrations/20260501100000_dcron3_observability.sql` | Schema completo (whitelist, tabela particionada, RPC, view, ETL function, partition rotation, TTL purge, 3 crons categoria A) |
| `workers/shared/src/sentry.ts` | `initSentryWorker`, `captureWorkerException`, `flushSentry` — reutilizável por qualquer worker. No-op se DSN ausente. |
| `workers/journeys/src/handlers/healthCheck.ts` | Handler `runEventDrivenHealthCheck` + `shouldAlert`/`isWorseTier` puros (testáveis). |
| `workers/journeys/src/index.ts:runSchedulerLoop` | Hook `lastHealthCheck` (5min interval). |
| `workers/analytics/src/segment-eval-worker.ts` | `flushHealthMetricBatch` (60s) emite `segment_membership_changes`. |

---

# PARTE B — SEGMENTAÇÃO & RULE ENGINE

## 18. Rule Engine v2 — Segmentação Escalável

### Arquitetura de condições — dois tiers

O evaluator opera em dois tiers para equilibrar performance e expressividade:

**Tier 1 — State conditions:** avaliadas diretamente contra `contact_state`. Zero queries adicionais. O(1) por contato. Inclui RFM, lifecycle, tags, segmentos, CRM summary e email engagement (timestamps).

**Tier 1.5 — Product stat conditions:** avaliadas via EXISTS em `analytics.contact_product_stats`. Uma subconsulta por condição de produto, mas sobre conjunto já filtrado pelo Tier 1.

**Tier 2 — Event conditions (futuro):** EXISTS em `contact_events` para casos que não podem ser materializados (ex: "abriu campanha X especificamente"). Reservado para Sprint 9+.

### Tabela de summary por produto — `analytics.contact_product_stats`

- Particionada por `HASH(tenant_id)` — 8 partições
- Chave: `(tenant_id, party_id, product_id)`
- Campos: `first_purchase_at`, `last_purchase_at`, `purchase_count`, `total_spent`
- Populada por `process_purchase_analytics` a cada compra paid

### evaluation_mode em `analytics.segments`

- `event_driven` (default): reavaliado apenas quando `fields_changed` relevantes chegam
- `time_driven`: requer re-scan periódico — condições com `within_last/before/after` expiram com o tempo sem evento
- Inferido automaticamente pelo `inferEvaluationMode()` no service de segmento

### Operadores semânticos (v2)

- Timestamps: `op: 'within_last' | 'before' | 'after'` + `window_days` + `window_unit: 'days' | 'hours'`
- Números/contagens: `op: 'gte' | 'lte' | 'gt' | 'eq'` + `value`
- Nunca `value` em dias — sempre `window_days` explícito

> **Regra v1 de produto — sem rolling counts:** O builder v1 permite apenas timestamps (`first/last_purchase_at`) e contagens lifetime (`purchase_count` total). Rolling counts com janela deslizante (ex: `purchase_count_30d`) são V2 — só implementar quando o `rfm-rolling-refresh` for estendido para suportar decay.

### Arquivos

- `workers/analytics/src/types/segmentCondition.ts` — interfaces: `ProductStatCondition`, `EmailEngagementCondition`, `CrmCondition`
- `workers/analytics/src/segment-rules-evaluator.ts` — avaliação in-memory para crm e email_engagement
- `workers/analytics/src/segment-sql-builder.ts` — SQL builder para product_stat via EXISTS + `inferEvaluationMode()`

### Estado do evaluator

| Condition | Implementação | Performance |
|---|---|---|
| `bought_product` / `not_bought_product` | GIN `bought_product_ids && ARRAY[]` | ✅ 530x — ativo |
| `has_tag` / `not_has_tag` | 4 subqueries (diretas + product tags transitivas) | Bloqueado — product tags não estão em `tag_ids` |
| `responded_form` / `not_responded_form` | Subquery `lead_form_submissions` | Phase 2 — após producer tempo real |
| `from_import` / `not_from_import` | Subquery `lead_import_history` | Phase 2 — após producer tempo real |
| Campos RFM / lifecycle / scores | Colunas diretas via `v_segment_base` | Já O(1) |

---

## 18.1 Segmentation Engine v2 — Implementação

**Status:** ✅ Operacional (Sprint 10)

**Localização:** `workers/analytics/src/segmentation/`

**Arquivos:**

| Arquivo | Responsabilidade |
|---|---|
| `types.ts` | Interfaces: ConditionLeaf, ConditionGroup, SegmentAST, EvaluationContext, ExecutionMeta |
| `normalizer.ts` | toNNF + canonicalize + astHash — elimina NOT via De Morgan |
| `validator.ts` | Validação de AST contra registry — retorna erros tipados |
| `semanticFieldRegistry.ts` | Campos hardcoded com resolver, cost_hint, missing_semantics |
| `executionRegistry.ts` | Mapeamento campo → tabela/coluna de execução |
| `resolvers.ts` | 5 resolvers: snapshot_scalar, snapshot_array_contains, profile_scalar, summary_exists, custom_trait_scalar |
| `planner.ts` | compileSQL() + compileMemory() — recursivo por nó AND/OR |
| `fieldCatalog.ts` | getSegmentFieldCatalog() — combina campos estáticos + field_definitions do tenant |
| `traitProducer.ts` | BLOCK_TYPE_MAP, processSubmission, updatePartyIdentity |

**Resolvers:**

| Resolver | Fonte | Custo | Cobertura |
|---|---|---|---|
| `snapshot_scalar` | `analytics.contact_state` | instant | RFM, lifecycle, scores, timestamps |
| `snapshot_array_contains` | `analytics.contact_state` (GIN) | instant | bought_product_ids, tag_ids, segment_ids |
| `profile_scalar` | `crm.parties` / `crm.party_person` | fast | phone, profession, campos canônicos |
| `summary_exists` | `analytics.contact_product_stats` | fast | product_first_purchase, product_purchase_count |
| `custom_trait_scalar` | `crm.party_trait_*` | fast | qualquer campo de formulário do tenant |

**Operadores suportados:**
- Scalar: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`
- Temporal: `before`, `after`, `within_last`, `between_dates`, `not_between_dates`
- Array: `in`, `not_in`, `contains_any`, `not_contains_any`
- Text: `starts_with`, `contains`
- Existencial: `is_null`, `is_not_null`

**Coexistência com legacy:** `refreshSegments.ts` usa `rules_json_v2` se `ast_version >= 1`, senão fallback para `build_segment_where_clause` (D37: função dropada do banco — fallback nunca atingido, dead code confirmado).

**Estado da migração:** Migração completa — 42/42 segmentos manuais com v2. Ver D33. Colunas adicionadas em `analytics.segments`: `rules_json_v2`, `ast_version`, `ast_hash`, `migrated_at`, `migration_warnings`.

---

## 18.15 Segment Builder — Blocos Paralelos

**Status:** ✅ Operacional (Sprint 11)

### Modelo mental

O builder de segmentação usa **blocos paralelos** no primeiro nível. Cada bloco é uma lista de condições conectadas por AND (implícito). Os blocos se conectam entre si com E ou OU (operador **entre** blocos, não dentro).

```
┌─────────────────────────────────────┐
│ Bloco 1                             │
│   • Comprou produto A   (AND)       │
│   • Comprou produto B   (AND)       │
│   • Comprou produto C   (AND)       │
└─────────────────────────────────────┘
                 ─── OU ───
┌─────────────────────────────────────┐
│ Bloco 2                             │
│   • Comprou produto D               │
└─────────────────────────────────────┘
                 ─── E ───
┌─────────────────────────────────────┐
│ Bloco 3                             │
│   • Cidade = São Paulo              │
└─────────────────────────────────────┘
```

**Lógica:** `(A E B E C) OU (D)` → dessa lista, filtra `cidade = SP`

### Como funciona internamente

**UI:** Cada bloco é um `UIBlock { group: RuleGroupType, joinOp: 'AND' | 'OR' }`. O `joinOp` define como ele se conecta ao bloco anterior (primeiro bloco é ignorado).

**Recompose (UI → Backend AST):** Blocos consecutivos com mesmo `joinOp` são agrupados. Exemplo:
- `[B1, OU:B2, E:B3]` → `AND( OR(B1, B2), B3 )`
- `[B1, E:B2, E:B3]` → `AND(B1, B2, B3)`
- `[B1, OU:B2, OU:B3]` → `OR(B1, B2, B3)`

**Decompose (Backend AST → UI):** Recursivo — achata grupos aninhados de volta para blocos planos com joinOps corretos. `AND(OR(B1,B2), B3)` → `[B1, OU:B2, E:B3]`.

**Controle de ciclo:** `internalChangeRef` evita que mudanças internas (emitChange) trigem re-decompose via useEffect. Só mudanças externas (template, load do DB) re-decompõem.

### Regras permanentes

1. Condições dentro de um bloco são **sempre AND** — sem toggle interno
2. O operador E/OU é **entre blocos**, não dentro
3. Botões "Adicionar bloco E" / "Adicionar bloco OU" definem o `joinOp` do novo bloco — blocos existentes não mudam
4. `RuleGroup.tsx` usa `maxDepth={0}` dentro de blocos — sem sub-grupos aninhados na UI

### Arquivos

| Arquivo | Papel |
|---|---|
| `src/components/segmentation/SegmentBuilder.tsx` | Wrapper: decompose/recompose, state de UIBlocks, preview |
| `src/components/segmentation/RuleGroup.tsx` | Renderiza um bloco (condições + add/remove). `maxDepth` controla nesting |
| `src/components/segmentation/RuleRow.tsx` | Uma condição individual (campo, operador, valor) |
| `src/types/segmentation.ts` | Tipos: `RuleGroup`, `RuleCondition`, `FieldDefinition` |
| `src/types/segmentation-v2.ts` | Tipos AST v2: `ConditionNode`, `GroupNode`, `RulesJsonV2` + helpers de conversão |
| `src/lib/segmentation/normalizeAstV2.ts` | Adapter de normalização aplicado no load (R48). Resiliência a shapes legacy até Fase 2/4 |

### Shape canônico de `rules_json_v2`

O AST v2 dos segmentos rule-based segue o shape canônico:

- `root.children` contém EXCLUSIVAMENTE nós `kind: 'group'` (cada filho é um UIBlock).
- Conditions vivem dentro de cada bloco (AND interno).
- `root.op` define o joinOp entre blocos: AND ou OR.
- Conditions com `__ui_locked: true` são metadado de UI somente — nunca persistido no banco (strip em `toBackendAST`).
- Sentinelas `all_leads` / `all_customers` são deprecated (dropadas em load).
- Fields fora do `FIELD_OPTIONS` da UI mas presentes no `eval_condition_v2` do backend são preservados em modo locked até a UI ganhar suporte.

O adapter `normalizeAstV2ForUI` em `src/lib/segmentation/normalizeAstV2.ts` aplica esse shape em runtime durante o load (defensive). Após Fase 2 (backfill), o adapter vira no-op porque o banco terá shape A em todos os segmentos. Após Fase 4 (trigger guardrail), gravar shape não-canônico fica impossível.

**Classificação de warnings:** o normalizador emite seis tipos de warning. `field_dropped` e `field_locked` são **actionable** (renderizam banner amarelo na UI — usuário precisa revisar/recriar). `shape_wrapped`, `field_aliased`, `op_aliased`, `expanded_array_op` são **cosméticos** (transformações idempotentes que preservam semântica — silenciados no banner, mantidos no array para devs). Helpers: `hasActionableWarnings()` e `summarizeActionableWarnings()`.

---

## 18.16 Segment Refresh Queue — Cache Assíncrono

**Status:** ✅ Operacional (Sprint 11)

### Problema resolvido

O trigger síncrono `refresh_segment_parties()` bloqueava 400-800ms por save de segmento. Com 100+ tenants e segmentos grandes (22k membros), isso causava timeouts.

### Arquitetura

```
User salva segmento
    │
    ▼
Trigger BEFORE (< 10ms)
    ├── SET refresh_pending = true
    └── INSERT INTO segment_refresh_queue

Journeys worker (polling 5s)
    │
    ▼
process_segment_refresh_queue()
    ├── FOR UPDATE SKIP LOCKED
    ├── refresh_segment_parties() (400-800ms em background)
    ├── DELETE da fila
    └── SET refresh_pending = false
```

### Componentes

| Componente | Papel |
|---|---|
| `analytics.segment_refresh_queue` | Tabela fila (PK = segment_id, natural dedup) |
| `analytics.segments.refresh_pending` | Flag booleana para feedback na UI |
| `trg_queue_segment_refresh` | Trigger BEFORE no `analytics.segments` — enfileira em INSERT, mudança de regras, ou cache vazio |
| `analytics.process_segment_refresh_queue()` | RPC batch processor (SKIP LOCKED, limit configurable) |
| `analytics.queue_stale_segments()` | Enfileira segmentos não atualizados há 60+ min |
| `analytics.queue_segment_for_refresh()` | Helper para enfileirar manualmente |
| `workers/journeys/src/index.ts` | Scheduler polls `process_segment_refresh_queue` a cada 5s |
| `workers/journeys/src/handlers/checkSegmentEntries.ts` | Safety net: se cache vazio, enfileira e retorna (próximo tick encontra membros) |

### Regras permanentes

1. Trigger NUNCA chama `refresh_segment_parties()` diretamente — sempre enfileira
2. Worker usa `FOR UPDATE SKIP LOCKED` — sem contenção entre instâncias
3. Após 3 tentativas com erro, item é removido da fila (`refresh_pending` resetado)
4. Segmentos cluster/lookalike são ignorados (têm seus próprios pipelines)

---

## 18.2 Perfil do Contato — party_profile e Traits

**Arquitetura em três camadas:**

```
analytics.contact_state      → comportamento derivado (instant)
                                RFM, lifecycle, scores, memberships
crm.party_person             → identidade base (já existe)
crm.party_profile            → snapshot de perfil (fast)
                                colunas canônicas: phone, profession,
                                birth_date, city, state, country,
                                missing_semantics, source_system,
                                source_field_id (mapeia block_id do formulário)
crm.party_trait_text         ↗
crm.party_trait_number       → projeção tipada para segmentação (fast)
crm.party_trait_timestamp    → índice por (tenant_id, field_key, value_*)
crm.party_trait_boolean      → cobertura completa de operadores por tipo
crm.party_trait_multi_value  ↘
```

**Por que não JSONB puro para segmentação:** GIN cobre igualdade e containment mas não range queries, comparações numéricas e datas com precisão de planner. Traits tipadas cobrem todos os operadores com índice previsível.

**Fluxo de formulário → traits:**

```
forms.form_versions status → 'published'
  → trg_enqueue_field_sync
  → handler forms:sync_field_definitions
  → syncFormFieldDefinitions():
      para cada bloco BLOCK_TYPE_MAP[type] = 'trait':
        INSERT INTO crm.field_definitions
          source_field_id = block.id
          field_key = slugify(block.title)
        ON CONFLICT (tenant_id, field_key) DO UPDATE

forms.form_submissions status → projection
  → handler forms:project_traits
  → processSubmission() em sql.begin():
      1. normalizeAnswer() por tipo de bloco
      2. upsert em party_trait_* por field_type
      3. merge party_profile.custom_fields
      4. updatePartyIdentity() para blocos identity
      5. INSERT segment_eval_queue fields_changed = [field_keys]
```

**BLOCK_TYPE_MAP — 22 tipos:**

| Categoria | Tipos | Kind |
|---|---|---|
| Identidade | first_name, last_name, full_name, email, phone | identity → party_person |
| Texto livre | short_text, long_text | trait → text |
| Seleção única | single_choice, picture_choice, dropdown | trait → single_select |
| Boolean | yes_no | trait → boolean |
| Seleção múltipla | multiple_choice, checkbox | trait → multi_select |
| Numérico | number, rating, nps, ranking | trait → number |
| Data | date | trait → timestamp |
| Compostos | contact_info, address | skip (futuro) |
| Display | statement, website | skip |

**Typeform — integração parcial (implementada):**
Schema `typeform.*` com 6 tabelas (`forms_raw`, `responses_raw`, `import_runs`, `sync_checkpoints`, `themes_cache`, `webhook_events`). Forms e responses migrados via `historical-sync-worker` e `pull-sync-worker`. Handlers: `typeform:sync_forms`, `typeform:sync_responses`, `typeform:promote_forms`, `typeform:promote_responses`, `typeform:enrich_themes`, `typeform:backfill_settings`. Ver R34/R35 para lições aprendidas da implementação.

---

## 19. Soft-delete — segments e forms

- `analytics.segments`: coluna `deleted_at timestamptz` adicionada. Soft-delete via `UPDATE SET deleted_at = NOW(), is_active = false` — nunca DELETE físico
- `forms.forms` e `smart_forms.forms`: `deleted_at timestamptz` adicionado
- 7 RPCs corrigidas para filtrar `deleted_at IS NULL`
- `segment-eval-worker` filtra `deleted_at IS NULL` no claim
- `refreshSegments` filtra `deleted_at IS NULL`
- Dropdowns de automações no builder filtram deletados + aviso visual quando segmento/form foi deletado

---

## 20. Jobs stuck — prevenção

`analytics.cleanup_stale_segment_eval_jobs(interval)` — RPC criada. Libera jobs claimed há mais que o threshold (default 10min). Chamada a cada ~60s no `segment-eval-worker`. Configurável via `SEGMENT_EVAL_STALE_MINUTES`.

---

# PARTE C — REGRAS PERMANENTES

## 21. Regras extraídas de incidentes

Cada regra foi extraída de um incidente real. Sem narrativa — apenas o que o sistema aprendeu e nunca deve esquecer.

| # | Regra | Origem |
|---|-------|--------|
| R1 | Enqueue deve verificar TODAS as fontes, sem filtrar por `source_name`. Um provider = um `source_name` canônico | Dedup cross-source |
| R2 | Mappers usam valor bruto do provider, nunca líquido. Moeda estrangeira → BRL com cotação do payload | Amount |
| R3 | Idempotência de enqueue usa tabela de destino (`commerce.transactions`), nunca campo de status na tabela raw | Enqueue |
| R4 | Um provider = uma Edge Function. Deploy de um isolado do outro | Edge Functions |
| R5 | `raw_source` é opcional quando provider tem schema raw dedicado. Obrigatório caso contrário | Rastreabilidade |
| R6 | Após deploy de Edge Function: confirmar versão no dashboard + validar primeiro ciclo nos logs | Deploy |
| R7 | Enum salvo no banco deve ter CHECK constraint. Tipos TS e valores DB auditados juntos em PRs | Contrato frontend→banco |
| R8 | Campos de controle de fluxo (reentrada, elegibilidade) = colunas tipadas, nunca `settings_json` | JSON vs colunas |
| R9 | Nenhuma view recalcula RFM em runtime. `contact_state` é o snapshot. NTILE só em `refreshRfm.ts` | RFM fonte única |
| R10 | Modelo errado → dropar, não adaptar. Adaptar prolonga coexistência de dois modelos | Party-first |
| R11 | Migração de coluna: (1) backfill dados (2) migrar escritas (3) migrar leituras (4) DROP após 1 sprint | DROP seguro |
| R12 | Trigger que atribui role superior deve revogar roles inferiores. Lead e customer mutuamente exclusivos | Role hierarchy |
| R13 | NTILE só em `refreshRfm.ts`. Dois produtores de RFM = bug. Regra 365d vive ali ONLY | RFM produtor único |
| R14 | Tabelas quentes: auditar índices e dead tuples a cada sprint. Índices ≤ queries distintas | Performance |
| R15 | Tabelas de membership não carregam metadata de algoritmo. Score/tier = tabela do compute | Membership vs metadata |
| R16 | Condições correlacionadas (produto×tempo) → tabela de summary dedicada, nunca JSONB | Summary tables |
| R17 | Campo array vazio = segmento silenciosamente errado. Validar divergência = 0 antes de migrar para GIN | Array producers |
| R18 | Clusters: algoritmo plugável, output é contrato fixo. `segment_parties` com `source_run_id` indexável | Clusters/Lookalikes |
| R19 | Datas de providers brasileiros sem offset explícito = `America/Sao_Paulo` (UTC-3), nunca UTC. Appender `Z` causa erro de +3h em todos os timestamps. | Sympla timezone bug |
| R20 | Toda fila de trabalho com status terminal (`processed`, `success`, `completed`) precisa de TTL de purge definido na criação. Fila não é ledger — não acumula histórico. `segment_eval_queue` sem purge → 1.1M rows, 482MB, banco travado. | Queue bloat incident |
| R21 | Primeiro provider não ganha schema próprio. Config de integração = `integrations.accounts` desde o dia 1. Schema dedicado só para dados raw do provider (`eduzz.invoices`, `eduzz.buyers`). `eduzz.integrations` legada causou workarounds `eduzzLegacy` no frontend e 7 RPCs duplicadas. | Eduzz legacy migration |
| R22 | Edge Functions seguem convenção `{provider}-receiver-webhook` (URL pública) e `{provider}-processor-worker` (worker interno). Nomes ambíguos causam configuração de URL errada no provider e interrupção silenciosa de webhooks. | Rename eduzz Sprint 10 |
| R23 | Workers devem passar `p_handlers text[]` no RPC `claim_jobs` para filtrar no SQL, nunca em application code. Claim sem filtro trava jobs de outros handlers e gasta locks desnecessários. | claim_jobs Sprint 10 |
| R24 | Webhook URLs usam domínio customizado (`core.kreatorshub.com.br`) com slug no path, nunca URL raw do Supabase. Slug é preferencial; `?tenant_id=uuid` é fallback. Receiver resolve slug → tenant_id via `core.tenants`. | Custom domain Sprint 10 |
| R25 | Edge Function nunca é worker. Se precisa de poll loop, claim de jobs ou processar lógica pesada, é Railway worker. Sintomas de violação: Edge Function com `while(true)`, `claimJobs()`, múltiplas chamadas RPC em sequência, timeout > 10s, ou lógica que cresce com volume de dados. Qualquer um desses sinais = mover para Railway worker imediatamente. | eduzz-processor-worker migration Sprint 15 |
| R26 | Webhook de provider deve criar jobs com `provider='ingestion'`. O ingestion-worker é o único consumidor de processamento downstream de webhooks. Edge Function dedicada por provider só existe para recepção e normalização O(1). | eduzz-processor-worker migration Sprint 15 |
| R27 | OAuth providers usam app centralizado da plataforma, nunca app por tenant. Client credentials em env vars globais. Token por tenant salvo em `integrations.accounts.config`. Dependência de app OAuth por tenant = risco operacional: cliente revoga → todos os tenants perdem acesso. | Eduzz OAuth migration Sprint 10 |
| R28 | Multi-step writes que criam estado dependente (enrollment + job, entity + event) devem estar em `sql.begin()`. `postgres.js` suporta transações com PgBouncer transaction mode e `prepare: false`. Comentário `sql.begin() não suportado` é incorreto e causou fanout não-atômico nas jornadas. | Principal Eng Audit D18 |
| R29 | PostgreSQL NÃO propaga `relrowsecurity` para partições existentes. Policies na tabela pai SÃO herdadas, mas cada partição precisa de `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` explícito. Sem isso, acesso direto à partição via PostgREST bypassa RLS. Foram 49 partições habilitadas individualmente no Security Sprint 2. | Security Sprint 2 partition RLS |
| R30 | `contact_state` contém exclusivamente comportamento derivado. Campos de perfil (phone, profession, qualquer atributo estático do contato) pertencem a `crm.party_profile`. Campos customizados do tenant pertencem a `crm.party_trait_*`. Materializar atributos de perfil ou traits em `contact_state` é violação arquitetural que destrói a fronteira entre snapshot comportamental e dados brutos de perfil. | Segmentation Engine v2 — Sprint 10 |
| R31 | CHECK constraints de enums (`triggered_by`, `status`, etc.) devem ser auditadas antes de adicionar novo valor no código. Constraint com lista incompleta causa insert silenciosamente rejeitado. Sempre: (1) verificar constraint existente, (2) migration adicionando novo valor, (3) deploy migration antes do código. | Form trait pipeline — `form_submission` vs `form_submitted` |
| R32 | Unique index de coalescing (`UNIQUE (tenant_id, job_type) WHERE status = 'pending'`) é correto para jobs coalescentes (refresh_features, refresh_rfm) mas bloqueia jobs per-entity (project_traits, sync_field_definitions). Jobs per-entity precisam de `payload->>'submission_id'` no unique ou exclusão do job_type no index parcial. Misturar os dois padrões na mesma tabela requer WHERE clause que exclua os job_types per-entity. | Form trait backfill — unique constraint bloqueou enqueue em massa |
| R33 | Job enfileirado na tabela/worker errado bloqueia silenciosamente. `project_traits` foi enfileirado em `integrations.jobs` mas integrations-worker só consome providers (eduzz, sympla, apidozap). Jobs ficaram pending sem erro. Antes de enfileirar: (1) confirmar qual worker consome a tabela, (2) verificar se job_type está na allowlist (`ENABLED_JOB_TYPES`/`KNOWN_JOB_TYPES`). Trait projection = `analytics.processing_jobs` → analytics-worker. | Form trait pipeline — jobs pending indefinidamente |
| R34 | **Todo novo provider obrigatoriamente passa pelo sistema canônico de integração.** Ao conectar qualquer novo provider (Typeform, HubSpot, Google Calendar, etc.), o caminho é sempre: `integrations.accounts` → `integrations.capabilities` → `integrations.jobs` → `pull-sync-worker` ou `historical-sync-worker`. É **proibido**: (1) criar tabelas `{provider}.import_runs` ou equivalentes para controlar execução, (2) criar workers isolados por provider fora do sistema canônico, (3) fazer chamadas de API de sync diretamente de Edge Functions, (4) implementar scheduling fora de `integrations.jobs`. Padrão de nomeação: `{provider}:{ação}` (ex: `typeform:sync_responses`, `typeform:promote_forms`). **Como identificar violação:** se você está prestes a criar uma tabela de controle de execução ou um worker dedicado a um provider, pare — você está criando uma pipeline paralela. | Typeform implementado com pipeline própria (`typeform.import_runs`, zero capabilities, zero jobs) em Março/2026. Corrigido no mesmo sprint. |
| R35 | **Import de provider externo exige filtro de organização antes de gravar no raw.** Todo sync que busca dados de uma API externa deve filtrar pelo `provider_account_id` da conta vinculada ao tenant antes de inserir no raw. Nunca importar "tudo que a conta vê" sem validar que o dado pertence ao escopo daquele tenant. Checklist obrigatório antes de gravar qualquer registro no raw: (1) `tenant_id` está correto e explícito? (2) `integration_account_id` está preenchido? (3) O dado pertence à organização/conta vinculada ao tenant (não apenas acessível pelo token)? | Typeform importou forms de todas as organizações acessíveis pelo personal token sem filtrar por organização, gerando 271 registros órfãos com contaminação entre tenants. Corrigido com limpeza em Março/2026. |
| R36 | **Campos 1:N nunca são denormalizados em `contact_state`.** `contact_state` é estritamente escalar (um valor por contato). Entidades com cardinalidade 1:N (oportunidades, tickets, forms futuras) são avaliadas via `EXISTS` subquery direta na tabela relacional com índices corretos. O worker planner (`opportunity_exists` resolver) e o SQL function (`eval_condition_v2`) devem gerar SQL equivalente para esses campos. Denormalizar 1:N em `contact_state` (ex: `has_open_opportunity boolean`) cria divergência semântica entre preview (SQL function faz subquery real) e refresh (worker lê snapshot stale), além de exigir pipeline de sync impossível de manter consistente. **Modo de execução: BULK_ONLY** — mudanças em oportunidades (criar, mover estágio, fechar) NÃO disparam reavaliação incremental do segmento; atualiza apenas no próximo ciclo de `evaluate_segment`. Se no futuro for necessário reatividade em tempo real, o pipeline de CRM deve enfileirar `evaluate_segment` jobs ao mudar status de oportunidades. **Campos implementados:** `has_opportunity`, `opportunity_status`, `opportunity_pipeline`, `opportunity_stage`, `opportunity_assigned_to`, `opportunity_lost_reason`, `opportunity_created_days`, `opportunity_closed_days`, `opportunity_won_days`, `opportunity_lost_days`. | `has_open_opportunity` e `open_opportunity_count` foram dropados de `contact_state` em Março/2026 após auditoria confirmar zero segmentos referenciando. Substituídos por 10 campos `opportunity_exists` no worker planner. Índice `idx_opportunities_party_tenant_status` criado para suportar as subqueries. |
| R37 | **Impact Analysis obrigatória antes de mudança em contrato compartilhado.** Antes de alterar formato, renomear ou deprecar qualquer conceito listado em `DEPENDENCY_MAP.md`: (1) listar todos os readers e writers; (2) confirmar que TODOS os readers suportam o novo formato; (3) dual-write por pelo menos 1 sprint antes de dropar formato antigo; (4) campo vazio usado para filtrar = fail-closed (zero resultados, nunca "todos"). **Nota (v7.21):** R37 protege contra migração mal feita, mas pressupõe que `segment_parties` reflita as regras. Drift entre `segment_parties` e avaliação fresh (D-SEG-5) burla a proteção: o worker lê membership stale e dispara para um conjunto que não casa com o segmento atual. Operação A em 30/04 reconciliou 22 segmentos divergentes; causa-raiz do drift segue em investigação. | Incidente Sprint 12 — campanha blast radius: migração `rules_json` → `rules_json_v2` quebrou email-campaign-worker que ainda lia v1 vazio, disparando para base inteira do tenant. |
| R38 | **Fail-closed para membership vazia.** Se `segment_parties` retorna 0 rows para um `segment_id` que tem regras definidas (`rules_json_v2 IS NOT NULL`), a operação dependente (envio, export, enrollment) DEVE recusar, nunca prosseguir sem filtro. Zero members + regras definidas = refresh pendente, não "enviar para todos". **Implementações:** (1) `CampaignSend.tsx` — `getSegmentRecipientsFailClosed()` tenta refresh e rejeita se ainda 0; (2) `generate-export/index.ts` — enfileira refresh e aborta export; (3) `checkSegmentEntries.ts` (worker handler) — guard com verificação de `rules_json_v2` + enqueue refresh; (4) `check-journey-entries/index.ts` (Edge Function legada) — guard equivalente como safety net. **Nota (v7.21):** R38 protege contra membership vazia mas não detecta membership *errada*. Drift cap (D-SEG-5) é o caso "membership populada mas não corresponde às regras". Auditoria recomendada (D-SEG-8 — job diário) até a causa-raiz ser fechada. | Incidente Sprint 12 — mesma origem de R37. |
| R39 | **Skip em nó de automação DEVE retornar `success: false` + `waitUntil`.** Quando um nó `send_email` detecta que a campanha não está pronta (status não-enviável), o handler deve retornar `{ success: false, waitUntil }` para que o enrollment fique em espera e reprocesse. Retornar `success: true` com skip faz o enrollment avançar para o próximo nó sem enviar o email, perdendo o envio silenciosamente. Status `automation` é um status válido para envio em contexto de journey — não deve ser rejeitado. **Lição adicional: Docker layer cache.** Workers containerizados (Railway, ECS, etc.) podem reutilizar layer de build anterior mesmo após push de novo código. Sempre incluir `BUILD_VERSION` constante no entrypoint do worker para: (1) invalidar cache do `COPY src` layer a cada mudança, (2) confirmar nos logs qual versão está rodando. | Incidente Sprint 13 — commit `c3f172e` corrigiu send_email mas Railway serviu código antigo por cache Docker. Worker logava `campaign_not_ready/automation` com `success: true` (código antigo) em vez de enviar o email. 19 enrollments ficaram stuck. |

| R40 | **Condition node deve usar Rule Engine v2.** O nó `condition` em journeys deve reutilizar os mesmos campos, operadores e resolvers do segmentation engine (AST v2). Implementação duplicada (6 campos hardcoded vs 40+ do Rule Engine) viola DRY e causa divergência silenciosa. Evaluator: carregar `ContactState` do party + `evaluateRules(state, ast.root)`. | Gap identificado Sprint 14 — condition node tem 6 campos, Rule Engine v2 tem 40+ |
| R41 | **Journey version_id em enrollments.** Enrollment deve congelar `version_id` no momento da criação. Se a jornada é editada durante execução, enrollments existentes continuam com a versão original. Implementação: coluna `version_id` em `journey_enrollments` + snapshot `nodes_json`/`edges_json` em `journey_versions`. | Requisito Etapa 2 — versionamento de jornadas |
| R42 | **Decisão de condition node deve ser persistida.** O resultado da avaliação (campo, valor, operador, resultado booleano) deve ser gravado em `journey_executions.output_json` para auditoria. Se o contato re-entrar, a decisão anterior é consultável sem re-executar a condição. | Requisito Etapa 2 — auditabilidade |
| R43 | **Trigger context congelado no enrollment.** `context_json` deve capturar o estado do contato no momento do enrollment (valores de campos usados por conditions). Conditions avaliam contra esse snapshot, não contra o estado atual, para garantir determinismo na execução. Exceção: nós explicitamente marcados como "live evaluation" podem consultar estado atual. | Requisito Etapa 2 — determinismo |
| R44 | **Extensibilidade de resolvers via registry.** Novos campos para condition node devem ser adicionados via registro (field → resolver function), não via switch/case hardcoded. O resolver recebe `(partyId, tenantId, fieldConfig)` e retorna o valor. Mesmo padrão do `semanticFieldRegistry.ts` do analytics. | Requisito Etapa 2 — condition engine extensível |
| R45 | **Frequency guard em nós de comunicação.** Nós `send_email`, `webhook`, `notify` devem respeitar limite de frequência por contato configurável no journey settings. Default: max 1 email por contato por 24h na mesma jornada. Guard consultável em `journey_executions` (contagem de executions do mesmo node_type para o mesmo enrollment nas últimas N horas). | Requisito Etapa 2 — proteção contra spam |
| R46 | **Workers separados por SLA.** journeys-event-worker (< 5s, push-based) e journeys-worker (minutos, batch) devem permanecer separados. Condition engine pesado (queries em contact_state) não deve rodar no event-worker. Se condition evaluation for necessária no fanout, delegar para journeys-worker via processing_job. | Confirmado Sprint 14 — audit comprovou separação necessária |
| R47 | **Email engagement via subquery, não snapshot.** Campos `campaign_opened`/`campaign_clicked` usam resolver `email_engagement` com EXISTS em `email.email_events`. Engagement data é per-campaign — nunca materializar em `contact_state`. Condition node usa pre-enrichment (`__email_engagement` injetado pelo conditionEngine). Padrão replicável para futuros campos relacionais (ex: `opened_journey_email_X`). | Sprint 19 |
| R48 | **`analytics.segments.rules_json_v2.root.children` deve conter exclusivamente nós `kind: 'group'`.** Conditions soltas no nível root são deprecated (shape B legado). Adapter UI (`normalizeAstV2ForUI`) é safety net que normaliza shapes legacy em runtime no load — até Fase 2 (backfill SQL canonicalizando todos os segmentos) e Fase 4 (trigger BEFORE INSERT/UPDATE em `analytics.segments` validando o shape). Após Fase 4, R48 vira enforcement estrutural; antes disso, é convenção. | Hotfix v7.21 — 46/52 segmentos manuais ativos com shape B causaram builder vazio em produção; duas rotas de criação coexistiam, escrevendo formatos divergentes. |
| R50 | **pg_cron é APENAS housekeeping.** Permitido: DELETE/UPDATE de TTL em filas/logs/audit; DDL maintenance (criar partição, REINDEX, VACUUM); pulse simples sem lógica de domínio (`SELECT some_fn()` que apenas dispara cleanup banco-only). **Proibido:** loop sobre tenants (`FROM core.tenants WHERE is_active`) — viola §2.4 Isolamento por tenant; lógica de negócio embutida (thresholds dinâmicos, sharding heurístico, condições de domínio); INSERT/UPDATE em tabelas de domínio (não-fila/log); fanout que cresce com volume. Lógica com schedule tenant-aware ou condições de negócio = worker no Railway. Cron novo requer entrada na tabela §11 + classificação A; cron categoria B/C/D requer auditoria arquitetural antes de aprovar. | Auditoria v7.22 — `rfm-rolling-refresh` e `segment-eval-fallback` foram criados sem registro arquitetural, com loop bloqueante de tenants no banco e mascarando bugs do event-driven path. Reclassificados como categoria B; D-CRON-2..4 abertos. |
| R52 | **Observability de plataforma reside em camada própria desacoplada do mecanismo observado.** Telemetria (Camada A — `analytics.event_health_metrics`), view de saúde (Camada B — `analytics.v_event_driven_health`) e alerta (Camada C — handler em worker) são entidades separadas. View e handler leem APENAS a tabela de telemetria — NUNCA tabelas operacionais (`fallback_log`, `segment_eval_queue`, etc). Mudança no mecanismo observado (ex: substituir cron por worker em D-CRON-4) afeta APENAS quem POPULA a métrica, não quem consome. **Como identificar violação:** PR adicionando `JOIN analytics.fallback_log` ou similar dentro de `v_event_driven_health` ou do handler de alerta = violação. Caminho correto: novo emissor escreve em `event_health_metrics` via `analytics.emit_health_metric()`. | D-CRON-3 (v7.25) — primeira camada de observability arquitetural; precedente para futura observabilidade de outros paths (workers, integrações, journey throughput). |
| R51 | **Campos de domínio separado fora de `contact_state` e `v_segment_unified` são avaliados como optimistic (`true`) no worker in-memory; a filtragem definitiva acontece no SQL path via subquery em `eval_condition_v2`.** Extensão operacional de R36: campos 1:N (oportunidades, tickets, etc.) NÃO devem ser denormalizados em `contact_state`. No fast-path do worker (planner/evaluator de jornadas e segmentação), quando o campo não tiver resolver disponível na avaliação in-memory, retornar `true` (optimistic) e deixar o SQL path do `eval_condition_v2` aplicar o filtro real via subquery indexada. Nunca adicionar coluna em `contact_state` para "facilitar" — isso reintroduz o drift entre snapshot stale e regra fresh que motivou R36. **Como identificar violação:** PR adicionando `has_*` ou `*_count` em `contact_state` para resolver gap de avaliação no worker = violação. Caminho correto: criar/ajustar resolver no SQL path. | Sprint de Performance v7.23 — `has_open_opportunity` resolvia false em CRM conditions porque o worker in-memory não conhecia oportunidades; fix correto foi optimistic no worker + subquery no SQL path, não materialização. |
| R53 | **Trigger enrichment via COUNT in-transaction.** Triggers de jornada podem enriquecer `event_data` com campos derivados na mesma transação SQL (ex: `purchase_sequence_number`). O campo é instantâneo (zero pipeline delay) e determinístico para condition nodes. Usar `party_id` como chave de agregação (party-first). Best-effort sob concorrência (READ COMMITTED — dois inserts simultâneos do mesmo party podem ambos receber sequence=N). | Sprint 19 — purchase_sequence_number |

---

# PARTE D — DÍVIDAS TÉCNICAS & ROADMAP

## 22. Dívidas Arquiteturais

| # | Item | Prioridade |
|---|---|---|
| D5 | ~~Migrar pipeline SendGrid para versão unificada~~ | ✅ Resolvido (v7.11) — `webhooks-sendgrid` O(1) + `integrations-worker` v81 handlers + URL migrada para `core.kreatorshub.com.br` em 5 subusers + `domain-delegation` atualizada |
| D7 | Producer `responded_form_ids` em tempo real (hoje só backfill) | Pendente |
| D8 | Producer `import_ids` em tempo real (hoje só backfill) | Pendente |
| D9 | `journey_ids` em `contact_state` — producer não existe | Pendente |
| D10 | `has_tag` GIN Phase 2 — bloqueado por product tags transitivas não estarem em `tag_ids` | Pendente |
| D11 | ~~`refreshFeatures` cleanup de parties com customer role revogada~~ | ✅ Sem ação necessária (v7.11) — `refreshFeatures` recalcula do zero a cada execução, zero órfãos encontrados em auditoria |
| D12 | Normalização de telefone (`phone_normalized` E.164) — `crm.party_person`, `sympla.participants`, todos os providers. Função utilitária compartilhada. Backfill + UI de indicador de cobertura. Campanhas WhatsApp filtram por `phone_normalized IS NOT NULL` | Pendente |
| D13 | `DROP TABLE eduzz.integrations` — tabela legada. Toda leitura/escrita já migrada para `integrations.accounts`. RPCs atualizadas. Dropar após validação. | Pendente |
| D14 | Auditoria preventiva de TTL em todas as filas do sistema — verificar se todas as filas têm: (1) cron de purge para rows terminais, (2) índice parcial cobrindo status ativo, (3) NOT EXISTS filtrando apenas status ativos. Ver R20. | Pendente |
| D18 | ~~Journey fanout não-atômico~~ | ✅ Resolvido (v7.13) — `sql.begin()` em 5 entry handlers + send_email node (D18b). Sweeper `sweep-orphan-enrollments` (*/10 * * * *) como safety net. 3 órfãos históricos recuperados. |
| D19 | **Observabilidade backend** — Sentry cobre apenas frontend React. Workers sem aggregação, dashboards ou alertas. **Fix:** (1) Sentry nos Railway workers; (2) dashboard de filas (depth, drain rate, p95 latency); (3) alertas: dead jobs > 0, queue depth crescendo, worker sem heartbeat >5min. Ref: Audit F10. | **P2** |
| D20 | ~~Journeys batch claim~~ | ✅ Resolvido (Sprint 17) — `claim_next_jobs(worker_id, batch_size, job_types, tenant_cap)` com ROW_NUMBER fairness. journeys-worker 10/poll, email-campaign 5/poll. Singular preservado. |
| D21 | **Segment-eval tenant scan** — Worker faz O(tenants) queries por ciclo mesmo sem trabalho. **Fix:** criar `claim_next_segment_jobs_global(worker_id, batch_size, shard_index, total_shards)`. Ref: Audit F2. | **P4** |
| D22 | **claim_jobs tenant fairness** — ordena por `priority, created_at` sem cap por tenant. **Fix futuro:** round-robin CTE com `ROW_NUMBER() OVER (PARTITION BY tenant_id)`. Ref: Audit F3. | Defer |
| D23 | **Idempotency constraint no enqueue** — `enqueue_webhook_job` usa SELECT-then-INSERT (TOCTOU). Risco baixo. **Hardening:** partial unique index em `integrations.jobs(webhook_event_id)`. Ref: Audit F5. | Defer |
| D24 | DROP TABLE segment_customers — CANCELADO (2026-03-20). Tabela ativa para clusters/lookalikes. | ❌ Cancelado |
| D25 | `refresh_tenant_distribution` + tela de monitoramento | Pendente |
| D26 | trg_record_email_engagement — ✅ Trigger + 4 colunas em contact_state operacionais. | ✅ Done |
| D27 | UI: novos blocos de condição no segment builder (product picker, time window, CRM condition) | Pendente |
| D28 | D6 Fase 3 — `DROP COLUMN customer_id`. **Fase 1 ✅ (2026-04-08, v7.24):** 6 colunas zero-data dropadas — `analytics.customer_cluster_assignments`, `analytics.customer_cluster_subgroup_assignments`, `analytics.customer_lookalike_scores`, `core.eduzz_webhook_events`, `eduzz.webhook_events`, `crm.customer_badges`. **Restante (11 entradas):** 8 base tables — `commerce.transactions`, `commerce.archived_transactions`, `journeys.journey_enrollments`, `crm.opportunities`, `crm.customer_tags`, `analytics.lifecycle_events`, `analytics.segment_customers`, `eduzz.buyers`; 3 views — `analytics.v_segment_base`, `analytics.v_segment_leads_base`, `crm.vw_parties_unified`. 5 FKs ativas → `crm.customers` (bridge table, 16.4K rows 100% party_id). 35 funções com "customer" no nome — auditoria necessária antes da Fase 3. **Próximas fases:** Fase 2 (tabelas com party_id 100%: transactions, enrollments, opportunities), Fase 3 (audit das 35 funções + drop FKs → `crm.customers`), Fase 4 (DROP `crm.customers` + rebuild views). | Fase 1 ✅ · Fase 2–4 Pendente |
| D29 | Performance loading timeout — ✅ Resolvido (v7.23, Sprint de Performance). `crm.get_dashboard_data` 534ms→17ms (3 lead CTEs colapsados em 1 `COUNT FILTER`); `analytics.get_analytics_data` 4 scans→1 (CTE `base_txn`); `crm.list_parties` reescrito de subqueries correlacionadas para CTEs batch. Novas RPCs: `crm.get_parties_stats(p_tenant_id)` (stats globais separados, staleTime 5min) e `analytics.get_transactions_chart_data(p_tenant_id, p_granularity, p_start_date, p_end_date)` (agregação por período: 26K rows→37 buckets, 22ms). `segment_eval_queue` purgada de 533K erros. Polling -99.8%. Frontend: React Query Persist (IndexedDB, 24h, limpa no logout), connection heartbeat (4min, pausa em tab inativa), prefetch durante auth (dashboard, tags, features), loading progressivo (Campaigns, Segmentation, Forms, Analytics), AccessGate bypass em warm path, dynamic imports −924K bundle, vendor chunks (`@xyflow`, `@tiptap`, `zod`), `useTenantTags` (6 query keys→1), `usePartiesStats` (cache 5min), polling 1s→30s, fix `has_open_opportunity` optimistic no worker + SQL subquery (R51). Rollbacks em `docs/rollback/`. Sentry JAVASCRIPT-REACT-22, 21, 1S fechados. | ✅ Done |
| D30 | Decompor blocos contact_info e address em traits individuais — hoje kind: 'skip', sem dados reais ainda | Pendente |
| D31 | Integração Typeform — ✅ Parcial. Schema `typeform.*` implementado (6 tabelas), sync via pull-sync e historical-sync. Pendente: analytics de choice blocks, webhook receiver para ingestão contínua. | Parcial |
| D32 | evaluation_mode em segments — ✅ Coluna populada automaticamente. 70/70 = event_driven. | ✅ Done |
| D33 | Segment builder UI → AST v2 — ✅ Frontend escreve exclusivamente rules_json_v2 desde Sprint 12. | ✅ Done |
| D34 | Scoping unique index `processing_jobs_coalesced_pending_uidx` — ✅ Resolvido (2026-03-30). Index agora exclui `project_traits` e `sync_field_definitions` do WHERE. Múltiplos pending per-entity permitidos. | ✅ Done |
| D35 | Backfill de traits de formulários — ✅ Resolvido (2026-03-30). Function SQL temporária `backfill_form_traits` projetou 8.816 traits (text: 8.464, number: 402, date: 368, boolean: 3) para 4 tenants. `segment_eval_queue` drenou em minutos. Gap restante: ~163K answers sem `field_definitions` (forms cujo `syncFieldDefinitions` nunca rodou — dívida separada, não D35). | ✅ Done |
| D36 | **RLS não reconhece system_admin** — policies de RLS usam `tenant_id IN (SELECT tenant_id FROM core.tenant_users WHERE user_id = auth.uid())`, mas system admins podem não ter registro em `tenant_users` para todos os tenants. O RPC `get_user_tenant_memberships` faz LEFT JOIN e retorna todos os tenants, mas o RLS bloqueia queries. **Fix:** criar helper `core.user_visible_tenant_ids(uid)` que retorna todos os tenants para system admins (via `core.system_admins`) e apenas memberships para users normais. Substituir subquery de RLS por chamada a essa função. Workaround atual: inserir system admins em `tenant_users` de cada tenant manualmente. | **P2** |
| D37 | Cleanup dead code segmentação v1 — ✅ Resolvido (2026-03-30). Removidos: `runEquivalenceGate.ts`, `equivalence.test.ts`, fallback legacy em `segmentEvaluator.ts` (substituído por throw explícito). | ✅ Done |
| D38 | **Alerta de anomalia pós-envio.** pg_cron job diário que detecta campanhas com `recipient_source = 'segment'` onde `recipient_count_at_send > segment_parties_snapshot * 1.5` OU `recipient_count_at_send > 80%` da base do tenant. INSERT em tabela de alertas. Depende das colunas de rastreabilidade adicionadas no Sprint 12 (Guard 1). | **P2** |
| D39 | **Backend validation RPC para envio de campanha.** RPC `email.validate_campaign_send(p_tenant_id, p_recipient_count, p_segment_ids, p_recipient_source)` que valida server-side: (1) se source='segment', recipient_count <= SUM(segment_parties) * 1.1; (2) se source='all', recipient_count <= total_contacts do tenant; (3) retorna ok/reject com reason. Defense-in-depth — proteção redundante independente do frontend. | **P2** |
| D40 | ~~party_id null em enrollments de formulário~~ | ✅ Resolvido (v7.13) — `create_opportunity` node: fallback lookup + last resort `crm.upsert_party_person` + graceful skip. `checkFormEntries` já tinha 3-layer resolve (v7.11). 10 jobs failed corrigidos, 0 órfãos restantes. |
| D41 | Queue audit — filas sem TTL/purge — ✅ Totalmente resolvido. Todas as filas com purge: `integrations.jobs` ✅, `integrations.webhook_events` ✅, `journeys.processing_jobs` ✅, `segment_eval_queue` TTL 24h→4h ✅ (2026-03-30). | ✅ Done |
| D42 | **DROP tabelas Eduzz legadas** — `eduzz.integrations` e `core.eduzz_integrations`. Ambas substituídas por `integrations.accounts`. Bloqueado por Sprint Security — credenciais em texto plano precisam ser migradas antes do DROP. | Bloqueado |
| B-17 | Testes de integração dos workers no tenant SaudeNow (`fe793fcd-7564-4d7c-b628-12a25e6d6656`) — criar jobs sintéticos para historical-sync, email-sync, ingestion e validar fluxo completo claim→running→success→analytics | Pendente |
| B-18 | ~~Mapper Eduzz — `installments`, `fee_value`, `net_gain`~~ ✅ Handler `eduzz:enrich_invoice` atualizado (26/03): popula campos financeiros a partir de `eduzz.invoices`. Backfill aplicado: 22K transações. Cobertura 99.9%. | **Done** |
| D43 | **`segment_entered` events não gerados.** `refresh_segment_parties_unified` faz UPSERT/DELETE direto sem chamar `apply_segment_membership_diff` — eventos `segment_entered`/`segment_exited` não são criados em `journeys.journey_events`. Os 9 eventos segment_entered existentes são de antes de 13/Mar. Impacto: `event_entry` com `trigger_event_type='segment_entered'` não funciona. Fix: gerar eventos inline no refresh ou restaurar chamada a `apply_segment_membership_diff`. | **P3** — nenhuma jornada usa esse trigger hoje |
| D44 | **Migrar template engine para SendGrid dynamic templates.** Atualmente o worker renderiza HTML com `substituteVariables()` (server-side). Migrar para `template_id` + `dynamic_template_data` habilitaria batch real (1000 emails/request via personalizations). Bloqueado por: cada recipient tem HTML diferente + `sendgrid_message_id` é 1 por request. | **P4** — quando precisar >50K emails/envio |
| D-SEG-2-BACKFILL | Canonicalizar todos `rules_json_v2` para shape A no banco. Após 48h estável de v7.21 em produção, rodar migração SQL que reescreve `root.children` envolvendo conditions soltas em grupos. Adapter `normalizeAstV2ForUI` vira no-op defensivo. | **P2** |
| D-SEG-3-CONTROLLED-COMPONENT | `SegmentBuilderV2` deve virar controlled — derivar `ast` interno do prop `initialAST` via `useEffect` de sync, ou eliminar o estado interno e fazer setter sobir todo via `onChange`. Hoje o gate na page (`!initialized → spinner`) elimina o sintoma sem reescrever o componente. | **P3** |
| D-SEG-4-INTEGRATION-TESTS | Aumentar cobertura de testes de integração da page de segmentos. Hoje cobre regression do state-lock (3 cenários). Faltam: salvar/atualizar segmento, locked condition que sobrevive round-trip e salva sem `__ui_locked`, navegação entre segmentos sem refresh da page. | **P3** |
| D-SEG-5-MEMBERSHIP-DRIFT | Causa-raiz do drift entre `segment_parties` e avaliação fresh não identificada. 22/52 segmentos ativos divergentes em 30/04. Operação A reconciliou (sintoma), causa segue aberta. Investigação: scheduler de refresh, trigger event-driven, regra invertida em path antigo. | **P1** |
| D-SEG-6-LAST-CALC-LIES | `last_calculated_at` é atualizado em paths que não fazem refresh real. Coluna não é confiável para detectar staleness. Auditar todos os writers da coluna; ou (1) só atualizar quando `refresh_segment_parties_unified` rodar com sucesso, ou (2) renomear para deixar semântica clara. | **P2** |
| D-SEG-7-EVENT-DRIVEN-AUDIT | Rota event-driven populou complemento semântico no segmento "Segmentação Amazônia" (3 contatos no card vs 4.167 fresh). Suspeita de regra invertida em path antigo (`apply_segment_membership_diff` ou trigger relacionado). Auditar inputs/outputs do event-driven path antes de Fase 4. | **P2** |
| D-SEG-8-DAILY-AUDIT-JOB | Job diário (pg_cron) que para cada segmento ativo: avalia regras fresh, compara com `segment_parties`, registra divergência em tabela de alertas e auto-refresh quando |drift| > N. Bloqueado por D-SEG-5 (sem causa-raiz, auto-refresh corre risco de mascarar problema). | **P2** |
| D-SEG-9-LEGACY-TRIGGER | `trg_refresh_segment_parties_on_change` em `analytics.segments` chama `refresh_segment_parties` (legacy v1) em vez de `_unified` (canonical v2). Possível parte da causa-raiz do drift (D-SEG-5). Auditar o trigger e migrar para a função unificada. | **P2** |
| D-RFM-LABELS | Lista canônica dos 11 labels `rfm_segment` está hardcoded em `workers/analytics/src/handlers/refreshRfm.ts:449-476` e duplicada em `src/components/segments/fieldConfig.ts FIELD_OPTIONS.rfm_segment`. Acentuação inconsistente em produção (D-DATA-1). Extrair enum compartilhado e re-exportar de ambos os lados. | **P3** |
| D-PRODUCT-FIRST-PURCHASE-UI | Campo `product_first_purchase` (temporal scoped por `product_id`) é válido no backend (`semanticFieldRegistry`) e usado por 1 segmento ativo, mas exige UI nova: scope picker (produto) + temporal window (`within_last N dias`). Atualmente preservado em modo locked após v7.21. Construir UI dedicada para destravar edição. | **P3** |
| D-OPP-FIELDS-UI | Campos `opportunity_pipeline`/`opportunity_stage`/`opportunity_assigned_to` precisam de pickers UUID com nome humanizado. Hooks (`usePipelines` etc.) já existem em `useSegmentOptions.ts`, mas não estão plugados no novo builder. Não ocorre em segmentos ativos hoje; se ocorrer ficará locked. | **P3** |
| D-RECENCY-DAYS-UNIT | `recency_days` foi registrado como `number` no fieldConfig, mas conceitualmente representa dias. Adicionar suffix "dias" no input (suporte ao campo `suffix` que já existe no `SemanticFieldMeta` do worker). | **P4** |
| D-CONVERSION-TEST-FIX | 4 testes em `src/lib/segmentation/conversion.test.ts` quebrados desde commit `8e38c4d` (anterior ao hotfix v7.21). Testam `builderToAstV2`/`astV2ToBuilder` (paths legacy não usados pelo builder novo). Avaliar se ainda têm valor antes da Fase 3 (eliminação do builder antigo). | **P4** |
| D-DATA-1-RFM-ACCENT | Acentuação inconsistente em `analytics.contact_state.rfm_segment` em produção: `Clientes Fiéis` (com agudo) coexiste com `Potenciais Fieis` (sem agudo). Outros labels OK. UI replica o estado do banco para preservar igualdade de string no SQL. Resolver junto com D-RFM-LABELS quando enum compartilhado for criado. | **P4** |
| D-CRON-1 | **Documentar todos os pg_cron jobs em `ARCHITECTURE.md`.** Mandatório (independente de categoria A/B/C/D). Tabela em §11 com jobname, schedule, command resumido, função SQL invocada, categoria, owner. Cron novo daqui pra frente requer entrada nessa tabela antes de ser criado. | ✅ Done (v7.22) |
| D-CRON-2 | **`rfm-rolling-refresh` migrar para worker no Railway.** Cron some, lógica de sharding (`MOD(HASHTEXT, 24)`) + threshold (20h staleness) vai pro código TS no `journeys-worker` ou novo `analytics-scheduler`. Risco: timing/SLA de RFM crítico — fazer com testes A/B em staging. Bloqueado por: revisar threshold 20h e estratégia de sharding em código antes de migrar. | **P3** |
| D-CRON-3 | **`segment-eval-fallback` instrumentar.** ✅ **Resolvido (v7.25).** Implementação muito além do "alarme em fallback_log" original: 3-camada Platform Health Observability decoupled (Camada A `event_health_metrics` particionada mensal + RPC `emit_health_metric` + 3 crons categoria A; Camada B view `v_event_driven_health` com tier classification per-tenant baseado em rolling 7d P50/P95/P99; Camada C handler `runEventDrivenHealthCheck` em `journeys-worker` com dedup, recovery silencioso, integração Sentry via `workers/shared/src/sentry.ts` reutilizável). 19 testes unit/integration. R52 estabelece o decoupling como regra para futura observability arquitetural. Pré-requisito de produção: criar Sentry project `workers-railway` e setar `SENTRY_DSN_WORKERS`. Auto-noop sem DSN. | ✅ Done |
| D-CRON-4 | **`segment-eval-fallback` migrar para worker.** Após baseline de D-CRON-3 estabelecido, mover loop de tenants para Railway. Cron vira pulse simples ou desaparece se métrica de hits zerar. Bloqueado por: D-CRON-3 + D-SEG-7 (event-driven audit) fechado. | **P3** |
| D-SEG-12-LOOKALIKE-DIRECT-WRITE | **`src/lib/analyticsStorage.ts:527 createSegmentFromLookalikes`** faz BULK INSERT direto no `analytics.segment_parties` da UI com `source: "customer"` hardcoded — bypassa todas as funções canonical. Para segmentos lookalike (todos os membros são customers por definição) é semanticamente correto, mas é mais um produtor fora do path canonical. Migrar para função SQL canonical (ex: `analytics.create_lookalike_segment_from_scores`) que reuse `apply_segment_membership_diff` para determinar `source` via `party_type` e mantenha audit trail. | **P3** |

---

### §22.3 Whitelist de dimensões que disparam `segmentation_input_version`

A coluna `segmentation_input_version` em `analytics.contact_state` (criada na Fase 1
do refactor de Maio/2026 — ver `docs/refactor/PLAN.md`) é o gatilho semântico
exclusivo de re-avaliação de segmentação.

A função canônica `analytics.apply_contact_state_mutation()` (Fase 2) decide bumpar
`segmentation_input_version` baseada na lista abaixo. **Mudar essa lista requer bump
de `analytics_segmentation_contract_version` (constante registrada em código + tabela
`core.platform_config`).**

#### Dimensões que SEMPRE bumpam (whitelist)

Identidade e role do contato:
- `party_type` — customer vs lead (R12). Toda mudança de role afeta segmentação.

Comportamento de compra (insumo direto de RFM e segmentos baseados em produto):
- `frequency` — número de compras
- `monetary` — valor total
- `recency_days` — dias desde última compra
- `bought_product_ids[]` — produtos comprados
- `total_purchases`, `total_revenue`, `avg_order_value`, `distinct_products` — agregados de compra
- `last_purchase_at`, `first_product_id`, `first_product_at` — primeiras/últimas compras

Derivados RFM (usados como input por segmentos hoje — confirmado em `analytics.segments.depends_on_fields`):
- `r_score`, `f_score`, `m_score`
- `rfm_segment` (mapeamento texto)
- `value_tier`
- `lifecycle_stage`

Tags, formulários, imports (insumos de segmentação manual):
- `tag_ids[]`
- `responded_form_ids[]`
- `import_ids[]`
- `acquisition_source`, `acquisition_campaign_id`

Reativação (usado em segmentos hoje):
- `reactivation_priority`

Engajamento agregado (insumo de segmentos baseados em comportamento de email):
- `last_email_opened_at`, `last_email_clicked_at`
- `email_open_rate_30d`, `email_click_rate_30d`
- `whatsapp_read_rate_30d`
- `last_engagement_at`

Identidade adicional (usado em segmentos baseados em perfil):
- Quaisquer `crm.party_traits` cujo `field_definition` esteja referenciado em
  `depends_on_fields` de algum segmento ativo (ex: `city`, `phone`, `profession` —
  detecção dinâmica via JOIN; lista evolui com novos traits)

#### Dimensões que NUNCA bumpam (blacklist)

Predições de ML (cosméticos para UI; segmentos não devem depender):
- `churn_score`
- `purchase_propensity`
- `ltv_predicted`
- `next_purchase_days`
- `next_best_product_id`
- `engagement_score`

Recomendações:
- `best_channel`
- `best_send_hour`
- `current_ladder_position`

Atribuição cosmética:
- `last_attributed_campaign_id`
- `total_attributed_revenue`
- `first_touch_revenue`

Membership (princípio P4 — projeção, não input):
- `segment_ids[]`
- `journey_ids[]`

Timestamps internos:
- `state_updated_at` (cache invalidation, não dirty trigger)
- `scores_updated_at`
- `features_version`
- `state_version` (legado pré-refactor)

#### Campos `BULK_ONLY` (R36) — fora do contrato `segmentation_input_version`

Campos de domínios 1:N (oportunidades, traits, clusters) que aparecem em
`depends_on_fields` mas NÃO vivem em `contact_state`. Avaliação ocorre via subquery
no SQL path (`eval_condition_v2`), não via watermark per-party. Nunca bumpam
`segmentation_input_version` por definição.

A função `analytics.is_bulk_only_dimension(text)` é a fonte canônica desta lista.
Decision API (`apply_contact_state_mutation`) classifica dimensões em 3 grupos:
whitelist (bumpa seg_version + enfileira) | bulk_only (bumpa apenas
`state_updated_at` + enfileira) | blacklist (nada).

Lista canônica:

**Opportunity (R36):**
- `has_opportunity`, `opportunity_status`, `opportunity_pipeline`, `opportunity_stage`
- `opportunity_assigned_to`, `opportunity_lost_reason`
- `opportunity_created_days`, `opportunity_closed_days`, `opportunity_won_days`, `opportunity_lost_days`

**Cluster (R-2.10/R-2.11 — out-of-scope no refactor de Maio/2026):**
- `cluster_id`
- `cluster_subgroup_id`

**Traits (R-2.12 — formalizado no refactor):**
- `party_traits` (genérico — "qualquer trait mudou", usado quando producer toca >5 traits)
- `party_trait_<key>` (granular — `LIKE 'party_trait_%'`, ex: `party_trait_idade`, `party_trait_cidade`).
  Usado quando producer toca ≤5 traits específicos. Avaliação real ocorre por trait_key
  via JOIN com `crm.party_trait_*` (5 tabelas tipadas).

Detecção em runtime: o worker `segment-eval-worker` reavalia esses segmentos via
fluxo BULK_ONLY (refresh_segments handler) — não via fila per-party. R36 cobre.

#### Como adicionar dimensão à whitelist

1. Argumentar em ADR (Architecture Decision Record) por que a dimensão é insumo real
   de segmentação (não derivado cosmético)
2. Validar que algum segmento ativo realmente depende dessa dimensão (`depends_on_fields`)
3. Migration adicionando entrada nesta seção do ARCHITECTURE.md
4. Bumpar `analytics_segmentation_contract_version` (constante em código + tabela
   `core.platform_config` para track)
5. Validar que `apply_contact_state_mutation()` reconhece a dimensão nova

#### Detecção runtime

A whitelist vive como CONST em `workers/shared/src/segmentation/dimensions.ts`
(será criada em R-2.1 — Decision Write API). Função canônica importa esta constante.
Mudança de whitelist sem regenerar build do worker tem efeito apenas após deploy.
Workers que invocam Decision API recebem `dirty_dimensions` como parâmetro
explícito — não fazem detecção runtime. Lista de dimensions emitidas por cada producer
está documentada em `DEPENDENCY_MAP.md` seção "Producers de contact_state".

#### Validação de cobertura (snapshot 2026-05-02)

Universo real medido em produção (`analytics.segments` ativos, `depends_on_fields`):
- 78 segmentos ativos
- 52 com `depends_on_fields` populado (≥ 1 entrada)
- 26 com array vazio `{}` — sentinela `all_contacts`/`all_customers`/`all_leads` ou
  segmentos pós D-SEG-10 que ainda não tiveram `depends_on_fields` re-derivado.
  R-3.4 trata esse caso (skip filter de dependência para segmentos sentinela).
- 21 dimensões distintas em uso (cross-check da whitelist acima):
  `all_contacts`, `bought_product_ids`, `city`, `from_import`, `has_opportunity`,
  `has_tag`, `m_score`, `not_responded_form`, `opportunity_pipeline`,
  `opportunity_stage`, `opportunity_status`, `phone`, `product_first_purchase`,
  `profession`, `r_score`, `reactivation_priority`, `recency_days`,
  `responded_form`, `responded_form_ids`, `rfm_segment`, `tag_ids`.

  Notas de validação:
  - `has_opportunity` + 4 `opportunity_*` → categoria BULK_ONLY (acima).
  - `from_import` → equivalente a `import_ids` (alias semântico em uso na UI legacy).
  - `not_responded_form` / `responded_form` → cobertos pelo aliasing de
    `normalizeAstV2ForUI` (D-SEG-10 v7.21).
  - `product_first_purchase` → temporal scoped, fora do contact_state (resolver
    `summary_exists`, BULK_ONLY também).

---

## Refactor em andamento (Maio 2026)

A plataforma está em refatoração estrutural do pipeline de segmentação. Detalhes em:
- [docs/refactor/PLAN.md](./refactor/PLAN.md) — visão geral, fases, princípios
- [docs/refactor/PROGRESS.md](./refactor/PROGRESS.md) — estado atual, decisões
- [docs/refactor/BACKLOG.md](./refactor/BACKLOG.md) — tarefas atômicas

**Princípios não-negociáveis durante refactor (P1-P10):**

P1: Single writer per state column.
P2: Decision Write API obrigatório.
P3: Versão semântica por dimensão.
P4: Membership é projeção, nunca input.
P5: pg_cron é só housekeeping (R50).
P6: Worker > cron sempre que houver tenant awareness.
P7: Eventos são imutáveis e first-class.
P8: Dependency-aware evaluation.
P9: Coalescing real via UNIQUE constraint.
P10: Observability é camada própria (R52).

Toda PR durante o refactor deve referenciar tarefa do BACKLOG (R-X.Y) e respeitar P1-P10.

---

## §22.4 Sprint 2 do refactor — Decision Write API canônica

**Status:** ✅ FECHADA em **2026-05-05** (~3 dias de execução).
**Documentação operacional viva:** `docs/refactor/PROGRESS.md` + `docs/refactor/BACKLOG.md`.

### Objetivo da Sprint 2

Centralizar a escrita em `analytics.contact_state` e o enqueue em
`analytics.segment_eval_queue` numa única função canônica
(`analytics.apply_contact_state_mutation` / `_batch`) — a **Decision Write API**.

Antes da Sprint 2, cada producer tinha que:
1. Bumpar `state_version` (legacy) manualmente
2. Decidir sozinho se enfileirava `segment_eval_queue`
3. Construir o INSERT direto com `triggered_by` arbitrário

Pós-Sprint 2, producers só escrevem suas dimensões de domínio + chamam a Decision API
com `(tenant_id, party_id, producer, dimensions_changed[], payload, priority, mode)`.
A API decide via **whitelist §22.3** se a mudança é segmentation-relevant, bumpa
`segmentation_input_version` quando aplicável, e enfileira atomicamente.

### 17 producers endereçados

**9 producers TypeScript migrados:**

| R-* | Producer | Tipo | Localização |
|---|---|---|---|
| R-2.1b | `process_purchase_analytics` | SQL fn | `analytics.process_purchase_analytics` |
| R-2.3 | `recordTagChange` | TS single-row | `historical-sync/.../analytics-events.ts:24` |
| R-2.4 | `refreshRfm` | TS bulk per-chunk | `analytics/handlers/refreshRfm.ts:241` |
| R-2.5 | `refreshFeatures` | TS bulk per-chunk | `analytics/handlers/refreshFeatures.ts:175` |
| R-2.6 | `refreshReactivation` | TS bulk + cleaner (2 escopos) | `analytics/handlers/refreshReactivation.ts:252,361` |
| R-2.7 | `recordFormSubmitted` | TS single-row | `historical-sync/.../analytics-events.ts:105` |
| R-2.8 | `recordImportAdded` | TS single-row | `historical-sync/.../analytics-events.ts:169` |
| R-2.9 | `linkProducts` | TS bulk via array | `analytics/handlers/linkProducts.ts:175` |
| R-2.12 | `traitProducer` | TS single-row BULK_ONLY | `analytics/segmentation/traitProducer.ts:727` |

**6 producers SQL migrados** (descobertos via cross-check MCP — D-2026-05-05-07):

| Função | Schema |
|---|---|
| `sync_tag_ids_to_contact_state` | analytics |
| `trigger_auto_populate_contact_state` | analytics |
| `record_email_engagement_event` | email |
| `trg_product_name_mapped` | commerce |
| `sync_party_type` | analytics |
| `sync_party_types_batch` | analytics |

**2 fora de escopo** (D-2026-05-05-06):
- `computeClusters`, `computeClusterSubgroups` — usam `segment_eval_queue` como
  canal de notificação cross-system para journey engine, não como re-eval por
  mudança de input. `segment_ids` é output, não input.

### Pattern BULK_ONLY (R36) formalizado

Dimensões em `depends_on_fields` mas que **não vivem em `contact_state`**.
Decision API classifica via `analytics.is_bulk_only_dimension(text)`:
bumpa `state_updated_at` + enfileira eval, **não** bumpa `segmentation_input_version`.
Avaliação real ocorre via SQL path (`eval_condition_v2`), não watermark per-party.

Categorias BULK_ONLY canônicas:

- **Cluster** (R-2.10/R-2.11): `cluster_id`, `cluster_subgroup_id`
- **Opportunity** (R36 original): `has_opportunity`, `opportunity_status`,
  `opportunity_pipeline`, `opportunity_stage`, `opportunity_assigned_to`,
  `opportunity_lost_reason`, `opportunity_created_days`, `opportunity_closed_days`,
  `opportunity_won_days`, `opportunity_lost_days`
- **Traits** (R-2.12 — formalizado nesta sprint): `party_traits` (genérico,
  usado quando producer toca >5 traits no mesmo evento), `party_trait_<key>`
  (granular via `LIKE 'party_trait_%'`, usado quando ≤5 traits específicos)

Detalhes da whitelist regular vs blacklist vs BULK_ONLY: §22.3.

### Decisões arquiteturais registradas (D-records)

| ID | Tema | Resumo |
|---|---|---|
| D-2026-05-03-01 | Arquitetura de migração | R-2.1 quebrada em a/b — dryrun em paralelo antes do cutover. Reduz blast radius de "tudo ou nada" no hot path de ingestion. |
| D-2026-05-03-02 | Atomicidade > resiliência local | Producers migrados não têm `EXCEPTION WHEN OTHERS` ou try/catch externo silenciador. Falha aborta transação. Justificativa: mudança parcial silenciada é pior que abort retornado ao caller. |
| D-2026-05-05-01 | Testabilidade workers/historical-sync | Worker não tem `scripts.test` nem suíte. Validação reduzida a `typecheck + build + observação produção`. Sprint dedicada de testabilidade pós-refactor (Vitest + mock SQL via DI). |
| D-2026-05-05-02 | refreshRfm gap event-driven pré-Fase 1 | Antes do refactor, refreshRfm refresha ~9k parties/6h em dimensões whitelist mas **nunca** enfileirava eval. Cron antigo (predicate `state_updated_at`) também não pegava (UPSERT não tocava esse campo). Mudanças de RFM eram invisíveis à segmentação na história da plataforma. R-2.4 fechou esse gap. |
| D-2026-05-05-03 | Coexistência de 2 evaluators | `segment-rules-evaluator.ts` (memória, hot path por job) lê `rules_json` (AST v1) — patchado em R-2.4.1. `segmentation/segmentEvaluator.ts` + `resolvers.ts` (SQL canonical via `refresh_segment_parties_unified`) lê `rules_json_v2`. Coexistência aceita como débito; migração legacy → v2 fica fora do escopo de Decision Write API. |
| D-2026-05-05-04 | `state_updated_at` redundante em refreshFeatures | UPSERT mantém `state_updated_at = now()` mesmo a Decision API batch também bumpando. Inócuo (mesma transação, último timestamp vence). Remover quando todos producers bulk forem 100% canonical. |
| D-2026-05-05-05 | refreshFeatures orquestra refresh_rfm | Cada cron run gera 2 batches em queue (refreshFeatures + refreshRfm) para os mesmos parties. Comportamento correto (cada producer declara dimensões reais). Custo transitório até R-3.2 (UNIQUE INDEX coalescente + UPSERT mesclando `fields_changed`). |
| D-2026-05-05-06 | BULK_ONLY pattern para cluster/traits/opportunity | `segment_eval_queue` tem 2 usos semânticos: (1) re-eval por mudança de input (whitelist) → Decision API; (2) notificação cross-system (segment_ids = output) → fora de escopo. Refactor para canal dedicado fica como item futuro. |
| D-2026-05-05-07 | Cross-check MCP > inventário textual | Inventário em PROGRESS.md ≠ estado real do banco. Antes de criar R-2.x novo, consultar `pg_get_functiondef` da função candidata. Cross-check final da Sprint 2 revelou 6 funções SQL já migradas silenciosamente (R-2.13–R-2.17 fechados sem código novo). |

### Métricas pré/pós Sprint 2 (combinadas com Fase 1)

| Métrica | Pré-refactor (2026-05-02) | Pós-Sprint 2 (2026-05-05) | Comentário |
|---|---|---|---|
| Amplification ratio (enqueues / mudanças reais) | 24.6x | **~1.0x** | Fase 1 (R-1.5/R-1.7) baixou predicate cron + Sprint 2 eliminou writes ad-hoc duplicados. |
| Drift global (`seg_input_version > last_evaluated`) | crescente, mascarado por cron fallback | **sustentado em 0** | Validado em produção pós-R-2.4/R-2.5/R-2.6 com ~95k jobs drenados. |
| Single-source-of-truth para writes em `contact_state` | ❌ — 12 producers escreviam ad-hoc, cada um decidia versão e enqueue | ✅ — todos producers passam por Decision API; whitelist §22.3 dita o que é segmentation-relevant | Princípio P1 (single producer per state column) materializado para metadados de versão. |
| Producers fora do contrato Decision API | 12 (todos) | 2 (out-of-scope D-2026-05-05-06) | computeClusters/Subgroups por design (notificação cross-system). |
| Triggers SQL com `state_version+1` legacy | 6 funções | **0** | Cross-check MCP em 2026-05-05. |

### Próximo: Sprint 3

Coalescing real (R-3.1/R-3.2):
- **R-3.1:** UNIQUE INDEX parcial em `analytics.segment_eval_queue (tenant_id, party_id) WHERE status='pending'`
- **R-3.2:** UPSERT que mescla `fields_changed` em vez de criar nova row

Resolve naturalmente:
- 2x jobs/party da orquestração refreshFeatures→refreshRfm (D-2026-05-05-05)
- 2x jobs/party do refreshReactivation (UPSERT scoring + cleaner no mesmo cron run)
- BULK_ONLY pattern do traitProducer (form com múltiplos blocos enfileira N jobs)

---

## 23. Roadmap Futuro

**Personalização e cache**
- Vercel KV para cache de edge (`contact_state` hot path)
- Identity resolution em checkout e formulários
- `visitor_sessions` conectado ao frontend

**Dashboard Piloto Automático**
- API de oportunidades de receita, cards de revenue em risco
- Attribution dashboard, product ladder visualization

**ML Real**
- Schema `cluster_runs` / `cluster_definitions` / `cluster_members` / `cluster_segments`
- `source_run_id` indexável em `segment_parties`
- `computeClusters.ts` — K-means k=5 lendo `contact_state`
- UI de renomear label de cluster, features comportamentais
- HDBSCAN/GMM via microserviço Python, lookalike por embeddings, A/B testing com bandit

---

## 24. Pipeline de Formulários → Traits

### Fluxo end-to-end

```
form_submission (status='submitted', party_id NOT NULL)
  → trigger enfileira job `project_traits` em analytics.processing_jobs
  → analytics-worker consome job
  → traitProducer.processSubmission()
    1. Carrega field_definitions (form_block_id → trait metadata)
    2. Busca form_answers do submission
    3. Filtra por COMPATIBLE_ANSWER_TYPES (single_choice→single_select, etc.)
    4. Upsert em crm.party_trait_text / party_trait_number / party_trait_timestamp / party_trait_boolean / party_trait_multi_value
    5. Merge party_profile.custom_fields
    6. updatePartyIdentity (blocos de identidade)
    7. Enfileira segment_eval_queue com triggered_by='form_submission'
```

### Arquitetura de dados — 3 camadas

**`crm.party_person` / `crm.party_profile`**
Campos de identidade (email, phone, nome) — atualizados via `updatePartyIdentity()`.

**`crm.field_definitions`**
Contrato semântico dos campos do tenant. `(tenant_id, field_key, field_type, operator_family, source_system='forms', source_field_id=block_id)`. Populado por `sync_field_definitions` a cada versão publicada.

**`crm.party_trait_text/number/timestamp/boolean/multi_value`**
Traits projetados por submission. Índice por `(tenant_id, field_key, value_*)`. Trait só é gravado se existir `field_definition` mapeada para o block. Idempotente: `ON CONFLICT (tenant_id, party_id, field_key) DO UPDATE`.

### Mapeamento block_type → destino

| block_type | Destino |
|---|---|
| `email`, `phone`, `first_name`, `last_name`, `full_name` | `crm.party_person` (identidade) |
| `short_text`, `long_text`, `single_choice`, `picture_choice`, `dropdown` | `crm.party_trait_text` |
| `yes_no` | `crm.party_trait_boolean` |
| `multiple_choice`, `checkbox` | `crm.party_trait_multi_value` |
| `number`, `rating`, `nps`, `ranking` | `crm.party_trait_number` |
| `date` | `crm.party_trait_timestamp` |
| `contact_info`, `address` | skip — D30 (decomposição futura) |
| `statement`, `website` | skip |

### Regras permanentes

- **R30:** traits NUNCA são materializados em `contact_state` — violação arquitetural
- **R36:** campos 1:N (oportunidades, tickets) NUNCA são denormalizados em `contact_state` — usar EXISTS subquery via `opportunity_exists` resolver
- **`project_traits` e `sync_field_definitions` NÃO são jobs coalesced** — múltiplos pending simultâneos por tenant são esperados e corretos
- A unique index de deduplicação em `analytics.processing_jobs` cobre APENAS jobs coalesced (`refresh_features`, `refresh_rfm`, etc.) — `project_traits` e `sync_field_definitions` estão fora dessa lista intencionalmente
- `tenant_id` vem sempre do job, nunca do payload
- `field_definitions` são pré-requisito: sem elas, `processSubmission` retorna 0 traits. Sempre rodar `sync_field_definitions` antes de `project_traits`
- `COMPATIBLE_ANSWER_TYPES`: mapeia tipos de resposta do formulário para tipos de campo. `single_choice` e `picture_choice` são compatíveis com `single_select`
- Backfill usa processamento direto: `backfillFormTraits.ts` chama `processSubmission()` diretamente, sem passar pela fila de jobs. Evita problemas com unique constraint de coalescing
- `DbConnection` adapter: scripts que usam `postgres.js` nativo precisam do adapter `{ unsafe, begin }` para compatibilidade com traitProducer

### Arquivos principais

| Arquivo | Papel |
|---|---|
| `workers/analytics/src/segmentation/traitProducer.ts` | Core: `processSubmission`, `syncFormFieldDefinitions`, `resolveFieldDefinitions`, `normalizeAnswer`, `updatePartyIdentity` |
| `workers/analytics/src/handlers/formTraits.ts` | Handlers de job: `handleProjectTraits`, `handleSyncFieldDefinitions` |
| `workers/analytics/src/segmentation/scripts/backfillFormTraits.ts` | Script de backfill (Phase 0: field_definitions, Phase 1: submissions) |
| `supabase/migrations/*_create_party_profile_traits.sql` | Schema das tabelas de traits e field_definitions |
| `supabase/migrations/*_fix_trg_enqueue_trait_projection.sql` | Trigger de enqueue para `project_traits` |
| `supabase/migrations/*_fix_processing_jobs_coalesced_index_scope.sql` | Scoping do unique index para excluir jobs per-entity |

---

# PARTE E — CHANGELOG

## [v7.28] — 2026-05-02
### Refactor Plan v1.1 — Cleanup pós-validação

**Cleanup dos documentos de refactor v1.0:**
- `docs/REFACTOR_PLAN.md` (duplicado na raiz) já tinha sido movido em `git mv` no commit `8b2e6f7` para `docs/refactor/PLAN.md`. Verificado no working tree: apenas 1 cópia existe.
- `docs/refactor/PLAN.md` §12: referência ARCHITECTURE.md v7.24 → v7.27.
- `docs/refactor/BACKLOG.md`:
  - R-1.0 nova (pré-requisito): definir whitelist antes da Fase 1.
  - R-1.4: naming de backup padronizado (`__pre_refactor_2026_05`).
  - R-3.4: critérios atualizados — `depends_on_fields` já existe e está populado em 52/78 segmentos (26 com array vazio recebem skip-filter).
- `docs/refactor/PROGRESS.md`:
  - D-2026-05-02-06: cadência ETL confirmada hourly (`5 * * * *`).
  - D-2026-05-02-07: `depends_on_fields` como fonte canônica.

**Nova seção §22.3 em ARCHITECTURE.md:**
- Whitelist canônica de dimensões que disparam `segmentation_input_version`
- Blacklist de dimensões que NÃO bumpam (predições, cosméticos, membership, timestamps internos)
- Categoria nova "Campos BULK_ONLY (R36)" para opportunity_* (não vivem em `contact_state`)
- Procedimento documentado de adição/remoção de dimensões + bump de `analytics_segmentation_contract_version`
- Snapshot de validação 2026-05-02: 78 segmentos ativos, 52 com depends_on_fields populado, 21 dimensões distintas em uso.

**Validações arquiteturais via Supabase MCP:**
- 21 dimensões em uso por segmentos ativos hoje (medido em `depends_on_fields`)
- `analytics.segments.depends_on_fields` populado em 52/78 segmentos ativos (26 com array vazio — sentinela ou pós D-SEG-10 sem re-derivar)
- Cron `etl-event-health-metrics` confirmado em `5 * * * *` (hourly)

**R-1.0 closes by this PR.** Próximo passo: iniciar Sprint 1 com R-1.1 (criar branch develop no Supabase) após aprovação explícita de Marcio. Sprint 1 unblocked.

---

## [v7.27] — 2026-05-02
### Refactor Plan v1.0 — Segmentation Pipeline

**Documentos de refactor publicados** em `docs/refactor/`:
- `PLAN.md` — visão arquitetural, 10 princípios não-negociáveis (P1-P10), 8 fases sequenciais (~6 sprints), critérios de sucesso mensuráveis via D-CRON-3 telemetry, plano de migração com gates de estabilidade.
- `PROGRESS.md` — savepoint operacional (estado, próximas 3 ações, bloqueadores, baseline metrics, decisões arquiteturais formalizadas D-2026-05-02-01..05, lições aprendidas L-001..L-003).
- `BACKLOG.md` — ~30 tarefas atômicas com IDs estáveis (R-1.1 a R-8.4), critérios de aceite, dependências entre tarefas. Granularidade micro nas Fases 1-3, macro nas 4-8.

**Mudança em ARCHITECTURE.md** (nova seção "Refactor em andamento (Maio 2026)" antes do §23 Roadmap):
- Link para os 3 documentos do refactor.
- Lista pública dos 10 princípios P1-P10.
- Regra: toda PR durante o refactor deve referenciar tarefa do BACKLOG (R-X.Y) e respeitar P1-P10.

**Sync workflow** atualizado (`.github/workflows/sync-architecture-docs.yml`):
- Trigger paths inclui `docs/refactor/**`.
- Constituição (ARCHITECTURE/SECURITY/DEPENDENCY_MAP) continua sincronizando para raiz de `kreatorshub-docs`.
- Documentos de refactor sincronizam para `kreatorshub-docs/refactor/` (subpasta espelhando D-2026-05-02-04).

**Status:** plano aprovado, Sprint 0 (setup) concluído, Sprint 1 inicia com R-1.1 (criar branch develop no Supabase) após aprovação de Marcio.

**Não inicia tarefas R-X.Y neste commit.** Apenas documentação publicada.

---

## [v7.26] — 2026-05-01
### Trigger enrichment — purchase_sequence_number

**`trigger_purchase_journey_event()`** enriquecido com `purchase_sequence_number` no `event_data` do bloco `purchase`. Valor computado via `COUNT(*) + 1` em `commerce.transactions` filtrado por `(tenant_id, party_id, status='paid')` — mesma transação SQL, zero delay, zero dependência do analytics pipeline.

**Design:** campo numérico genérico, não boolean. Suporta todos os operadores do Rule Engine: `=`, `>`, `<`, `>=`, `between`. Exemplos: primeira compra (`= 1`), recompra (`> 1`), terceira compra (`= 3`), entre N e M (`between 2 e 5`).

**Chave de agregação:** `party_id` (party-first, 100% cobertura em transactions pagas). Índice `idx_transactions_party_id` confirmado — 0.234ms por insert.

**Frontend:** campo `trigger.purchase_sequence_number` registrado em `TRIGGER_FIELD_CATEGORIES` (type: number, label: "Numero da compra (sequencia)").

**Padrão replicável (R53):** Qualquer trigger recorrente pode enriquecer `event_data` com sequence_number via COUNT na mesma transação. Candidatos futuros: `pending_sequence_number` (transaction_pending), `abandonment_sequence_number` (cart_abandonment), `submission_sequence_number` (form_submitted).

**Race condition:** READ COMMITTED permite que dois `paid` simultâneos do mesmo party vejam o mesmo COUNT — ambos recebem sequence=N. Aceito como best-effort (pagamento concorrente do mesmo party é raro).

---

## [v7.25] — 2026-05-01
### Platform Health Observability (D-CRON-3) + R52

**Contexto:** D-CRON-3 evoluiu de "alarme simples em `fallback_log`" para a primeira camada de observability arquitetural da plataforma. Princípio guia: instrumentação não pode ficar acoplada ao mecanismo observado — quando D-CRON-4 substituir o cron `segment-eval-fallback` por worker, view e handler permanecem inalterados.

**Migration `20260501100000_dcron3_observability.sql`:**
- `core.platform_health_whitelist` — whitelist auditável (reason CHECK, added_by, expires_at). Começa vazia. Tenant `00000000-...001` (Escola do Fluxo) NÃO entra: é tenant real.
- `analytics.event_health_metrics` — tabela particionada mensal (PK `tenant_id, metric_name, bucket_hour`), RLS service-only. Partições 2026-04 + 2026-05 criadas para warmup retroativo dos 7d iniciais de baseline.
- `analytics.emit_health_metric()` — RPC SECURITY DEFINER para UPSERT. Workers consomem.
- `analytics.platform_alert_state` — dedup state (last_tier, last_changed_at, last_alert_at).
- `analytics.v_event_driven_health` — tier classification per-tenant baseada em rolling 7d P50/P95/P99 do PRÓPRIO tenant, excluindo a hora atual. Tier `INSUFFICIENT_DATA` durante warmup (< 168 samples).
- `analytics.etl_event_health_metrics(window_hours)` — pulse banco-only (R50 categoria A). Agrega `fallback_log` + `contact_events` + `segment_eval_queue` em 5 métricas. Substituível por workers em D-CRON-4 sem mudar consumidores (R52).
- `analytics.event_health_metrics_partition_create_if_needed()` — DDL idempotente para rotação mensal.
- `analytics.event_health_metrics_purge_old(retention_days)` — DROP partition (90d retention).
- 3 novos pg_cron jobs categoria A registrados em §11: `etl-event-health-metrics` (5 * * * *), `event-health-metrics-partition-create` (30 2 25 * *), `event-health-metrics-purge-old` (30 4 * * *).

**Worker code:**
- `workers/shared/src/sentry.ts` — helper reutilizável. `initSentryWorker(opts)` no-op silencioso se `SENTRY_DSN_WORKERS` ausente — permite deploy do código antes do DSN. Adicionado `@sentry/node ^8.55.2` em deps de `@creator-hub/shared`. Re-exportado em `index.ts` + entrada em `exports` map.
- `workers/journeys/src/handlers/healthCheck.ts` — handler `runEventDrivenHealthCheck` (5min cadence). `shouldAlert` puro, testável. Lê apenas `v_event_driven_health` + `platform_alert_state`. Recovery silencioso (HEALTHY sustentado 30min).
- `workers/journeys/src/index.ts` — hook `lastHealthCheck` em `runSchedulerLoop` E `runCombinedLoop`. Sentry init no startup. Flush Sentry no shutdown.
- `workers/analytics/src/segment-eval-worker.ts` — `flushHealthMetricBatch` (60s) acumula `segment_membership_changes` em memória e emite via `emit_health_metric`. Custo zero no hot path.

**R52 nova:** Observability em camada própria desacoplada do mecanismo observado. View e handler nunca consultam tabelas operacionais — apenas `event_health_metrics`. Garante que substituir cron por worker (D-CRON-4) não invalide instrumentação. Precedente para futura observabilidade arquitetural.

**Testes (19 passing):**
- Pure fns: `isWorseTier`, `shouldAlert` (8 cenários — transições, dedup, intervalos).
- Integration: tier classification, multi-tenant isolation, recovery, WARN não emite, CRITICAL→ERROR não re-alerta.

**Baseline real coletado em Fase 1.b** (P50=8.613 hits/h, P95=26.420, P99=36.348 fallback_hits global) usado como sanity check da view; thresholds são per-tenant e bootstrap automático após 168h de coleta.

**Pré-requisito de produção:**
- Criar Sentry project `workers-railway` em org `creators-hub-ur`.
- Setar `SENTRY_DSN_WORKERS`, `SENTRY_ENVIRONMENT`, `RELEASE_SHA` em env vars Railway do `journeys-worker`.
- Sem isso: handler funciona normalmente, escreve em `platform_alert_state` + log, mas nada vai pro Sentry. **Não bloqueia deploy do código.**

**Pendências encadeadas:**
- D-CRON-4 (migrar `segment-eval-fallback` cron para worker) agora desbloqueado — D-CRON-3 forneceu observability + métrica `fallback_hits` que o substituto deve emitir.
- D-SEG-7 (event-driven audit) ganha visibility via `event_driven_hits` métrica.

---

## [v7.24] — 2026-04-08
### D28 Fase 1 — DROP customer_id em 6 tabelas zero-data

**Contexto:** diagnóstico de 2026-03-20 listou 17 tabelas/views ainda com coluna `customer_id`. Re-auditoria em 2026-04-08 identificou que 6 dessas tabelas tinham a coluna mas com **zero linhas populadas** — seguro dropar sem backfill nem migração de leitores.

**Tabelas afetadas (6 colunas dropadas):**
- `analytics.customer_cluster_assignments`
- `analytics.customer_cluster_subgroup_assignments`
- `analytics.customer_lookalike_scores`
- `core.eduzz_webhook_events`
- `eduzz.webhook_events`
- `crm.customer_badges`

**Procedimento:**
1. `SELECT COUNT(customer_id)` em cada tabela → todas retornaram 0.
2. `ALTER TABLE … DROP COLUMN IF EXISTS customer_id` nas 6 tabelas.
3. Re-query do `information_schema.columns` confirmou 0 ocorrências restantes nas 6.

**Inventário D28 restante (11 entradas):**
- **Base tables (8):** `commerce.transactions` (35K rows, 100% `party_id`), `commerce.archived_transactions` (7.9K), `journeys.journey_enrollments` (17, 99.8% `party_id`), `crm.opportunities` (22, 100% `party_id`), `crm.customer_tags` (7), `analytics.lifecycle_events` (9.4K, 5 órfãos), `analytics.segment_customers` (112), `eduzz.buyers` (46).
- **Views (3):** `analytics.v_segment_base`, `analytics.v_segment_leads_base`, `crm.vw_parties_unified`.
- **FKs ativas:** 5 → `crm.customers` (bridge table, 16.4K rows 100% party_id).
- **Funções:** 35 com "customer" no nome — auditoria obrigatória antes da Fase 3.

**Mudanças nos docs:**
- **D28** atualizada: Fase 1 ✅; restante e roadmap (Fase 2–4) detalhados.

**Próximas fases (não executadas nesta sprint):**
- **Fase 2:** tabelas com `party_id` 100% — `transactions`, `journey_enrollments`, `opportunities` ganham DROP de `customer_id` direto.
- **Fase 3:** audit das 35 funções com "customer" + drop das 5 FKs → `crm.customers`.
- **Fase 4:** DROP `crm.customers` + rebuild das 3 views (`v_segment_base`, `v_segment_leads_base`, `vw_parties_unified`).

---

## [v7.23] — 2026-05-01
### Sprint de Performance — RPCs, frontend e cache

**Contexto:** loading timeouts em tenants grandes (escola-do-fluxo, Sentry JAVASCRIPT-REACT-22/21/1S) escalaram para sprint dedicado de performance. Foco: matar full scans em RPCs quentes, eliminar polling agressivo, segmentar bundle, cachear o que é estável e arrumar fast-path do CRM condition. D29 era a dívida-mãe; ficou resolvida.

**Backend — RPCs novas:**
- `crm.get_parties_stats(p_tenant_id)` — stats globais separados da listagem paginada. Frontend cacheia com `staleTime` 5min, evita recalcular contagem a cada paginação.
- `analytics.get_transactions_chart_data(p_tenant_id, p_granularity, p_start_date, p_end_date)` — substitui query de 26K rows brutas por agregação por período (37 buckets, 22ms). Granularidade controlada server-side.

**Backend — RPCs otimizadas in-place** (rollbacks em `docs/rollback/`):
- `crm.list_parties` — subqueries correlacionadas → CTEs batch.
- `analytics.get_analytics_data` — 4 scans da tabela de transactions → 1 CTE `base_txn` reusada.
- `analytics.get_dashboard_data` — 3 lead CTEs colapsados em 1 `COUNT FILTER`. **534ms → 17ms**.

**Frontend:**
- React Query Persist com IndexedDB, TTL 24h, limpeza no logout.
- Connection heartbeat (4min, pausa quando tab inativa).
- Prefetch durante auth: dashboard, tags, features.
- Loading progressivo: Campaigns, Segmentation, Forms, Analytics.
- `AccessGate` com bypass em warm path (cache válido).
- Dynamic imports cortando **−924K** do bundle inicial.
- Vendor chunks dedicados: `@xyflow`, `@tiptap`, `zod`.
- `useTenantTags` — 6 query keys colapsados em 1.
- `usePartiesStats` — stats separados da listagem, cache 5min.
- Polling **1s → 30s** (redução -99.8% de requests em sessões idle).

**Worker / SQL — fix `has_open_opportunity`:**
- Worker in-memory não tem oportunidades carregadas → retorna optimistic (`true`).
- Filtragem definitiva no SQL path via subquery indexada em `eval_condition_v2`.
- **Não adicionou coluna em `contact_state`** (R36 + R51).

**Mudanças nos docs:**
- **R51** registrada: campos de domínio separado avaliam optimistic no worker, SQL path filtra de fato.
- **D29** marcada como ✅ Done com checklist completo do sprint.

**Pendências encadeadas:**
- D-CRON-3 (P2) segue como próximo fix calibrado (alarme sobre `analytics.fallback_log`).
- Padrão optimistic+subquery vira referência para futuros campos 1:N (tickets, custom relations).

---

## [v7.22] — 2026-05-01
### Auditoria pg_cron + R50 + D-CRON-1..4 + D-SEG-12

**Contexto:** investigação dupla pós-fix D-SEG-10. Q1 confirmou que NÃO existe rota oculta corrompendo `segment_parties` — a percepção de "BULK 4168 hoje" referia-se ao bulk de 30/04 19:10:52 UTC (Operação A pré-fix); pós-deploy, apenas 44 rows entraram via `apply_segment_membership_diff` per-party com `source` correto baseado em `contact_state.party_type`. Q2 auditou 14 pg_cron jobs ativos — 12 são housekeeping puro (categoria A), 2 violam §2.4 com loop bloqueante de tenants (`rfm-rolling-refresh`, `segment-eval-fallback`).

**Mudanças nos docs:**
- §11 expandida: tabela completa dos 14 crons com schedule, command, função SQL, categoria A/B/C/D, notas. Substitui a lista incompleta anterior.
- **R50** adicionada: pg_cron é APENAS housekeeping. Loop de tenants, lógica de negócio embutida, INSERT em tabelas de domínio, fanout dependente de volume — tudo proibido em SQL. Vai pra worker.
- **D-CRON-1** marcado como Done (auditoria + tabela já em §11).
- **D-CRON-2** (rfm-rolling-refresh → worker), **D-CRON-3** (instrumentar fallback_log), **D-CRON-4** (segment-eval-fallback → worker), **D-SEG-12-LOOKALIKE-DIRECT-WRITE** (`analyticsStorage.ts:527` migrar para SQL canonical) registrados.

**Pendências encadeadas:**
- D-CRON-3 (P2) é o próximo fix calibrado — alarme sobre `analytics.fallback_log` antes de qualquer migração.
- D-CRON-4 bloqueado por D-CRON-3 + D-SEG-7 (event-driven audit).
- D-SEG-12 é PR independente, pode ir junto com qualquer outro de cleanup do segmentation path.

**Próximas fases:**
- Fase 2 (backfill SQL canonicalizando shape A) e Fase 4 (trigger guardrail de R48) seguem planejadas mas não-iniciadas — aguardam 48h estável de v7.21 em produção.

---

## [v7.21] — 2026-04-30
### Hotfix segment builder + auditoria de membership

**Frontend (UI):**
- **Sintoma reportado:** segmentos antigos abriam vazios no builder; banner amarelo aparecia em loop.
- **Causa-raiz 1 (shape AST v2):** 46/52 segmentos manuais ativos tinham `rules_json_v2.root.children` com conditions diretas (shape B legado), enquanto o builder novo espera grupos (shape A "Blocos Paralelos"). 23 desses foram criados nativamente entre 10/03 e 01/04 — duas rotas de criação coexistiam.
- **Causa-raiz 2 (state-lock):** `SegmentBuilderV2` é uncontrolled — `useState(initialAST)` capturava o valor inicial vazio antes do useEffect de hidratação, ignorando o prop atualizado.
- **Fix 1 — adapter no load:** `normalizeAstV2ForUI` aplicado antes de `fromBackendAST`. Wraps loose conditions em grupos AND preservando `root.op`. Aliasing de fields/ops deprecated. Postura preserve > drop: fields desconhecidos viram conditions read-only com flag `__ui_locked` (strip antes de salvar). Apenas sentinelas (`all_leads`, `all_customers`) são dropadas.
- **Fix 2 — gate na page:** `SegmentationBuilder.tsx` não monta `<SegmentBuilderV2>` até `initialized === true`. Spinner inline durante hidratação. Banner de warnings no mesmo gate (não aparece antes do builder).
- **Fix 3 — banner só para warnings actionable:** apenas `field_dropped` e `field_locked` geram banner amarelo. `shape_wrapped`, `field_aliased`, `op_aliased`, `expanded_array_op` continuam logados internamente mas são silenciosos no UI (transformações cosméticas que preservam semântica).
- **Validação:** smoke test sobre 52 segmentos, 0 falhas. Suite de 30+ testes incluindo fixtures dos 7 segmentos críticos do banco. Build verde.

**Banco (operação A):**
- **Sintoma reportado:** segmento "Segmentação Amazônia" (3a487e5d) com 3 contatos no card mas 4.167 na lista; campanhas atingindo 3 errados em vez de 4.167 corretos.
- **Auditoria:** 22 de 52 segmentos manuais ativos com `segment_parties` divergente da avaliação fresh via `build_unified_where_clause_v2`. Top 3 com >700 contatos de excesso; "LEADS COM TELEFONE" sozinho com 9.153 contatos a mais do que a regra permite — violação direta de R37 (blast radius cap) e R38 (fail-closed).
- **Operação executada:** `refresh_segment_parties_unified` em massa nos 22 divergentes, ~13s total, zero erros. Snapshot pré-operação preservado em `analytics._segment_parties_audit_20260430` por 7 dias.
- **Resultado:** 22/22 com `segment_parties` = avaliação fresh = cache (`customer_count`/`lead_count`). Estado matematicamente correto.

**Pendências registradas:**
- D-SEG-3-CONTROLLED-COMPONENT (P3): `SegmentBuilderV2` deve virar controlled.
- D-SEG-4-INTEGRATION-TESTS (P3): cobertura de integração da page.
- D-SEG-5-MEMBERSHIP-DRIFT (P1): causa-raiz do drift entre segment_parties e regras não identificada. Operação A trata sintoma, não causa.
- D-SEG-6-LAST-CALC-LIES (P2): `last_calculated_at` é atualizado sem refresh real.
- D-SEG-7-EVENT-DRIVEN-AUDIT (P2): rota event-driven populou complemento semântico no Amazônia. Suspeita de regra invertida em path antigo.
- D-SEG-8-DAILY-AUDIT-JOB (P2): implementar job diário de auditoria + auto-refresh.
- D-SEG-9-LEGACY-TRIGGER (P2): `trg_refresh_segment_parties_on_change` chama `refresh_segment_parties` (legacy) em vez de `_unified` — possível parte da causa-raiz.
- D-RFM-LABELS (P3): RFM labels duplicados worker↔frontend.
- D-PRODUCT-FIRST-PURCHASE-UI (P3): campo válido no backend, exige UI nova.
- D-OPP-FIELDS-UI (P3): pickers UUID humanizados para campos de oportunidade.
- D-RECENCY-DAYS-UNIT (P4): adicionar suffix "dias" no input.
- D-CONVERSION-TEST-FIX (P4): 4 testes pré-existentes broken em `conversion.test.ts`.
- D-DATA-1-RFM-ACCENT (P4): acentuação inconsistente de labels rfm_segment em produção.
- D-SEG-2-BACKFILL (P2): canonicalizar todos rules_json_v2 para shape A no banco.

**Próximas fases planejadas:**
- Fase 2 (backfill SQL): após 48h estável em produção, canonicalizar shape A em todos os segmentos. Adapter `normalizeAstV2ForUI` vira no-op defensivo.
- Fase 3: eliminar `SegmentBuilder.tsx` antigo, `astV2ToBuilder.ts`, `conversion.test.ts`.
- Fase 4: guardrail estrutural — trigger BEFORE INSERT/UPDATE em `analytics.segments` validando shape canônico de `rules_json_v2`. R48 vira enforcement.
- Investigação D-SEG-5: diagnóstico do worker, do scheduler e do trigger event-driven para descobrir por que o drift acontece.

---

## [v7.20] — 2026-04-08
### Sprint 19 — Email Engagement Conditions (Segmentação + Jornadas)

**Handler fix:** `sendgrid:engagement_event` e `sendgrid:delivery_event` agora propagam `campaign_id`, `send_id`, `recipient_id` no INSERT de `email.email_events` via `resolveCampaignContext()` (1 query JOIN `campaign_recipients` → `campaign_sends`). Backfill aplicado: 75.2% opens e 68.0% clicks linkados (restante = transacionais/teste sem campaign — esperado).

**Novo resolver `email_engagement`:** Dual-mode (compileSQL para segmentação bulk + compileMemory para condition node). EXISTS subquery em `email.email_events` filtrado por `send_id` + `event_type`. Suporta operadores `is_true`, `is_false`, `url_contains`.

**Campos registrados:**
- `campaign_opened` — "Abriu campanha" (uuid[] send_ids, operator_family engagement)
- `campaign_clicked` — "Clicou em campanha" (uuid[] send_ids, + url_contains para filtro de URL)
- `last_email_opened_at` / `last_email_clicked_at` — expostos no frontend (já existiam no registry)

**Pre-enrichment no conditionEngine:** `extractEngagementSendIds()` scanneia AST, faz 1 query em `email.email_events`, injeta `__email_engagement` no contactState. Custo: 1 query extra apenas quando condição de engagement é usada.

**Frontend:** Categoria "Engajamento de email" no SegmentBuilderV2 com 4 campos. `useCampaignSends()` hook busca `campaign_sends` + `email_campaigns` JOIN. `CampaignSelectInput` via MultiSelectInput pattern (mesmo de bought_product). Operador `url_contains` renderiza input text adicional.

**Regra R47:** Campos de email engagement usam resolver `email_engagement` com EXISTS subquery — nunca materializar engagement data em `contact_state` (dados são per-campaign, não per-contact snapshot). Pre-enrichment no conditionEngine é o padrão para campos que dependem de dados relacionais no modo SINGLE.

## [v7.19] — 2026-03-30
### Cleanup sprint — G-010, D34, D37, D41
- **G-010:** revogados TRUNCATE/REFERENCES/TRIGGER de `authenticated` em 63 tabelas / 10 schemas
- **D34:** unique index `processing_jobs_coalesced_pending_uidx` agora exclui `project_traits` e `sync_field_definitions`
- **D37:** dead code v1 removido — `runEquivalenceGate.ts`, `equivalence.test.ts`, fallback legacy em `segmentEvaluator.ts`
- **D41:** `segment_eval_queue` TTL 24h→4h (projeção: 193MB→~50MB)

## [v7.17] — 2026-03-30
### D35 Done — Backfill de traits completo
- 8.816 traits projetados via SQL function temporária (text: 8.464, number: 402, date: 368, boolean: 3)
- 4 tenants processados: Escola do Fluxo (+8.694), Cone Island (+112), Thais Zacharias (+2), Instituto Socrates (+8)
- `segment_eval_queue`: 1.187 entries processados, pipeline drenou em minutos
- Function temporária `backfill_form_traits` dropada após uso
- Gap documentado: ~163K answers sem `field_definitions` (`syncFieldDefinitions` pendente)

## [v7.16] — 2026-03-26
### B-18 Done — Campos financeiros Eduzz
- Handler `eduzz:enrich_invoice` atualizado: popula `fee_value`, `net_gain`, `installments` em `commerce.transactions` a partir de `eduzz.invoices`
- Backfill aplicado: ~22K transações Eduzz existentes atualizadas (cobertura 99.9%)
- Edge function `integrations-worker` deployed (v86)
- B-18: ✅ Done

## [v7.18] — 2026-03-30
### Sprint 18 — Automações v2, Etapa 4: Novos Nodes (parcial)
- **If/Then/Else (Condition Engine no Canvas):** SegmentBuilderV2 integrado no NodeInspector via Dialog popup. 40+ campos de profile + 10 campos de trigger (`trigger.amount`, `trigger.product_name`, `trigger.status`, `trigger.utm_*`, `trigger.payment_method`, etc.) via `TRIGGER_FIELD_CATEGORIES`. Dual path: v2 (`rules_json_v2` via `toBackendAST()`) + legacy (`conditions[]`) com badge "Condição legada" e botão de upgrade. `FieldSelectV2` com Dialog + busca + grid 2 colunas por categoria. `configSummary` descritivo gerado do AST.
- **Cart abandonment trigger:** trigger SQL `on_cart_abandonment_journey_event` em `eduzz.cart_abandonments`. `event_data`: cart_id, product_ids, product_name, payment_method, UTMs, abandoned_at. Pipeline E2E: webhook → upsert cart → trigger → `journey_events` → `event_processing_jobs` → enrollment.
- **Transaction pending trigger:** `trigger_purchase_journey_event` estendido. `status='pending'` (e OLD != 'pending') gera `event_type='transaction_pending'`. INSERT direto como 'paid' gera apenas purchase. `event_data`: transaction_id, amount, product_id, product_name, payment_type.
- **Wait Until Date/Time:** nó wait expandido com 2 modos — `duration` (minutos/horas/dias) e `until_date` (data absoluta + timezone). Quando data já passou, 3 comportamentos via `if_passed`: `execute` (continua imediatamente), `skip_to_next_wait` (pula nós intermediários, registra em `journey_executions`), `exit_journey` (encerra enrollment). `convertLocalToUTC()` via Intl API nativa (Node 18+), sem dependência externa. `findNextWaitOrExit()` com safety limit 50 iterações.
- **Frontend:** EventEntryConfig expandido (6 event_types). WaitConfig com seletor de tipo, date/time picker, timezone selector (8 opções BR/US/EU), dropdown `if_passed`. ConditionConfig com Dialog popup + SegmentBuilderV2 compact.
- **Frequency guard default:** alterado de 16h para **0 (desabilitado)**. Opt-in via `EMAIL_FREQUENCY_GUARD_HOURS` env var no Railway.
- **semanticFieldRegistry fix:** 14 campos adicionados ao `workers/shared/src/segmentation/semanticFieldRegistry.ts` e `executionRegistry.ts` — campos financeiros (`total_revenue`, `frequency`, `monetary`) + 10 trigger fields (`trigger.amount`, `trigger.product_name`, `trigger.payment_type`, etc.). Sincronizado com `workers/analytics/src/segmentation/semanticFieldRegistry.ts` (8 campos financeiros). Fix identificado durante E2E testing — Condition Engine retornava `null` para campos não registrados.

## [v7.15] — 2026-03-26
### Sprint 17 — Automações v2, Etapa 3: Batch, Fairness & Performance
- **D20 ✅:** Batch claim com fairness — `claim_next_jobs(batch_size, job_types, tenant_cap)` com `ROW_NUMBER() OVER (PARTITION BY tenant_id)`. journeys-worker 10 jobs/poll, email-campaign 5 jobs/poll. Singular preservado.
- **Email paralelo:** `p-limit(10)` + rate limiter compartilhado. ~9→~50-90 emails/sec. `SEND_CONCURRENCY` configurável. Circuit breaker preservado.
- **R45 ✅:** Channel frequency guard — **default=0 (desabilitado, opt-in)** entre emails por contato (cross-journey). Skip com razão persistida em `journey_executions` (R42). `EMAIL_FREQUENCY_GUARD_HOURS` configurável. Index parcial `idx_campaign_recipients_frequency_guard`.
- **R46 ✅:** SLA separation verificada — zero overlap de job_types. journeys-worker (seconds SLA), email-campaign (minutes SLA).
- **D44 registrada:** Migrar template engine para SendGrid dynamic templates (P4).

## [v7.14] — 2026-03-26
### Automações v2, Etapa 2: Condition Engine + Versionamento
- **R40 ✅:** Condition Engine — Rule Engine v2 modo SINGLE. 8 arquivos extraídos para `@creator-hub/shared/segmentation/`. `evaluateConditionSingle()` com 40+ campos/operadores. Dual path: v2 (rules_json_v2) + legacy fallback.
- **R41 ✅:** Journey versions — `journey_versions` table, `publish_journey_version()` RPC, auto-publish on activation. `processJourneyNode` lê da versão congelada. Backfill: 8 jornadas + 39 enrollments vinculados.
- **R42 ✅:** Decisão persistida — `decision_result`, `decision_input_snapshot`, `resolved_edge_id` em `journey_executions`.
- **R43 ✅:** Trigger context — `context_json.trigger_payload` populado nos event handlers. `conditionEngine` flatten trigger fields como `trigger_<key>`.
- **Entry policy:** `entry_policy` (once/after_exit/always), `reentry_cooldown_hours`, `max_concurrent_enrollments`. `canEnterJourney()` v2 com fallback legacy.

## [v7.13] — 2026-03-25
### Sprint 14 — Automações v2, Etapa 1: Estabilizar Runtime
- **D18 ✅:** Fanout atômico — `sql.begin()` em 5 entry handlers (processEventJob, processBackfillJob, checkSegmentEntries, checkFormEntries, checkEventEntries). Zero enrollments órfãos possíveis.
- **D18b ✅:** send_email node — 3 INSERTs (campaign_sends → campaign_recipients → enqueue) atômicos via `sql.begin()`
- **D40 ✅:** create_opportunity guard — lookup parties → `crm.upsert_party_person` → skip gracioso. 10 jobs failed corrigidos.
- **D41 ✅ completo:** TTL cleanup em TODAS as filas de journeys: processing_jobs (pg_cron 4AM + scheduler 1h), event_processing_jobs (3AM), backfill_jobs (3AM)
- **Push-based segments:** `refresh_segment_parties_unified` detecta novos membros e enfileira `check_segment_entries` apenas para jornadas afetadas
- **Scheduler throttle:** `enqueue_journey_checks` skipa jornadas com check nos últimos 5 min. ~12.9K→~2K jobs/dia (85% redução)
- **Sweeper:** pg_cron `sweep-orphan-enrollments` (*/10 * * * *) — recuperou 3 órfãos históricos
- **R40-R46:** Regras arquiteturais para Automações v2 Etapa 2
- **D43 registrada:** `segment_entered` events não gerados pelo evaluator atual (P3)
- **Orphan sweeper:** `journeys.sweep_orphan_enrollments()` + pg_cron `sweep-orphan-enrollments` (*/10 * * * *) — recuperou 3 órfãos históricos
- **Push-based segments:** `refresh_segment_parties_unified` detecta novos membros e enfileira `check_segment_entries` apenas para jornadas afetadas. Zero push se zero novos.
- **Scheduler throttle:** `enqueue_journey_checks` skipa jornadas com check nos últimos 5 min. Projeção: ~12.9K/dia → ~2K/dia (fallback only) + push on-demand
- Wait node UI: métricas separadas em "Aguardando" (amber) + "Concluídos" (verde) no flow e na Performance dos Nós

## [v7.12] — 2026-03-24
### Correções baseadas em validação direta no banco
- **RPCs de lifecycle:** tabela na seção do ingestion-worker corrigida — `public.*` → `worker.*` (stubs confirmados removidos)
- **B-18:** colunas `fee_value`, `installments`, `net_gain` confirmadas em `commerce.transactions` — fix é no mapper, não no schema
- **D41:** `journeys.processing_jobs` atualizado — 90K rows / 66 MB (cresceu de 76K/55MB desde v7.11)

## [v7.11] — 2026-03-20

### Workers Railway — migração public.* → worker.* concluída
- `ingestion-worker`: já usava `worker.*` (confirmado, sem alteração)
- `historical-sync`: migrado — `public.claim_jobs/complete_job/fail_job/get_jobs_count_grouped/bulk_enqueue_jobs` → `worker.*` e `integrations.*`
- `email-sync`: migrado — `public.claim_jobs/complete_job/fail_job` → `worker.*`
- Funções de domínio eduzz: `public.eduzz_increment_sync_progress` → `eduzz.increment_sync_progress` (atualizada com lógica robusta), `eduzz.claim_invoices_for_enrichment` criada (SKIP LOCKED)
- Funções de domínio inbox: `public.upsert_message/batch_link_messages_to_crm/claim_active_accounts` → `inbox.*`
- `public.*` stubs de job management: **zero** — 6 stubs dropados + 4 stubs de domínio dropados
- `worker.*`: 10 funções canônicas (block_job, claim_jobs, claim_next_jobs, complete_job, enqueue_job, enqueue_webhook_job, fail_job, get_jobs_count_grouped, log_event, receive_webhook)

### TTL/Purge — integrations (R20 cumprida)
- `integrations.purge_old_jobs()` criada (retention: success=7d, terminal=30d; deleta `job_errors` dependentes)
- `integrations.purge_old_webhook_events()` criada (retention: processed=7d, failed=30d; NOT EXISTS guard para FK)
- pg_cron `purge-integrations-jobs` ativo (30 3 * * *)
- pg_cron `purge-integrations-webhook-events` ativo (45 3 * * *)
- Purge inicial: 10.722 jobs + 1.609 webhook_events removidos (11.480→758 jobs, 1.941→332 events)

### D5 — SendGrid pipeline unificado ✅
- `sendgrid-webhook` (inline, legado): URL migrada, não recebe mais tráfego
- `webhooks-sendgrid` Edge Fn ativa: O(1) — recebe → salva raw → enfileira job → 200 (verificação ECDSA P-256)
- `integrations-worker` v81: `registerSendgridHandlers` ativado
- Pipeline: SendGrid → `webhooks-sendgrid` → `integrations.jobs` (provider='sendgrid') → `integrations-worker` → `email.*`
- Handlers: `sendgrid:delivery_event` (bounce retry + supressão adaptativa), `sendgrid:engagement_event` (open/click tracking), `sendgrid:suppression_event` (unsubscribe + spam report)
- URL webhook migrada para `https://core.kreatorshub.com.br/functions/v1/webhooks-sendgrid` em 5 subusers via action `migrate_webhooks`
- `domain-delegation`: URL hardcoded atualizada + auth fix (JWT decode fallback para service_role)

### Itens auditados e fechados
- D11 (refreshFeatures roles revogados): ✅ zero órfãos em contact_state — refreshFeatures recalcula do zero
- D41 (queue audit): ✅ parcial — integrations.jobs e webhook_events com purge; journeys.processing_jobs pendente
- B-04 (webhooks eduzz orphaned 23–25/02): ✅ sem reprocessamento — transações já existiam via historical sync
- B-05 (segment_parties não populado): ✅ trigger `trg_queue_segment_refresh` já existia e funcionava

## [v7.9] — 2026-03-20
### Auditoria Fase 2 — D-series consolidação
- D26: confirmado completo — trigger `trg_record_email_engagement` + 4 colunas em `contact_state`
- D32: confirmado completo — `evaluation_mode` populado automaticamente, 70/70 = `event_driven`
- D28: atualizado com inventário factual — 17 tabelas/views com `customer_id` em 6 schemas
- D41: queue audit — 3 filas sem purge (`journeys` 76K, `integrations` 10K+1.9K), `segment_eval_queue` TTL 24h→4h
- D42: DROP tabelas Eduzz legadas (bloqueado por Sprint Security)
- Eduzz webhooks confirmados ativos (último evento: 2026-03-20 20:12)

## [v7.8] — 2026-03-20
### Auditoria da base de conhecimento
- D33: confirmado completo (Sprint 12). Tipo `rule_based` renomeado para `manual`. 42/42 com v2.
- D24: cancelado — `segment_customers` é ativa para clusters/lookalikes (~112 rows, 3 writers, 2 readers)
- D37: dead code confirmado — `build_segment_where_clause` dropada do banco, fallback v1 no evaluator chamaria função inexistente
- IC-5: seção 13 (CRM) atualizada — referências a `sync_opportunity_summary` e colunas dropadas removidas
- Seção 13 alinhada com R36 (`opportunity` via EXISTS, não denormalizado em `contact_state`)

## [v7.7] — 2026-03-19
### Sprint 12 — Documentação final + D40
- D40: party_id null em enrollments de formulário (P1 — investigar checkFormEntries)
- Fix: duplicate_campaign RPC — SECURITY DEFINER + assert_tenant_access + qualified extensions.digest()

## [v7.6] — 2026-03-20
### Journey Node Skip + Docker Cache Fix
- R39: send_email node skip deve retornar `success: false` + `waitUntil`, não `success: true`
- Status `automation` é válido para envio em contexto de journey
- BUILD_VERSION marker adicionado ao journey worker para deploy verification
- Lição: Docker layer cache pode servir código antigo — sempre ter marker de versão no entrypoint

## [v7.5] — 2026-03-19
### Campaign Safety Guards
- Guard 1: colunas de rastreabilidade em `campaign_sends` (`recipient_source`, `segment_ids`, `segment_parties_snapshot`, `recipient_count_at_send`)
- Guard 2: blast radius cap (ratio > 1.1 = abort), confirmação "Todos" com digitação obrigatória, hard cap 50K
- R38: implementação completa em 4 pontos (CampaignSend, generate-export, checkSegmentEntries worker, check-journey-entries Edge Function)
- D38: alerta pós-envio (P2 — pendente)
- D39: backend validation RPC (P2 — pendente)

## [v7.4] — 2026-03-19
### Segmentation Engine — Oportunidades
- Implementado resolver `opportunity_exists` com 10 campos via EXISTS subquery em `crm.opportunities`
- Removidas colunas órfãs `has_open_opportunity` e `open_opportunity_count` de `analytics.contact_state`
- Criado índice `idx_opportunities_party_tenant_status` em `crm.opportunities`
- R36 expandido: campos 1:N nunca denormalizados em contact_state, modo BULK_ONLY documentado, lista completa de 10 campos implementados
