# KreatorsHub — Arquitetura Definitiva

## Guia de referência para escala: 50.000 tenants · 50M contatos · 1000 automações por tenant

*Versão 7.7 — D40: party_id null em enrollments de formulário*

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
- `sendgrid-webhook` → receiver webhook-driven | pipeline ativo (escrita direta em `email.*`) | **rename pendente → `sendgrid-receiver-webhook`**
- `webhooks-sendgrid` → receiver alternativo (inativo — pipeline v2 planejado, ver D5)
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

- Trigger `segment_entry` → journeys-worker scheduler (60s) → verifica `analytics.segment_parties` → cria enrollment → enfileira `process_enrollment`
- Trigger `form_entry` → `trigger-form-journey` (Edge Function, imediato) ou journeys-worker scheduler (60s, fallback)
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
- Tipos de nó: `send_email`, `add_tag`, `remove_tag`, `condition`, `wait`, `ab_split`, `webhook`, `goal`, `exit`, `create_opportunity`

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
| `public.claim_jobs(worker_id, batch_size, provider)` | Clama jobs para processamento (SKIP LOCKED) |
| `public.complete_job(job_id, result)` | Marca success |
| `public.fail_job(job_id, error)` | Retry com backoff exponencial (60s→86400s + jitter 20%) ou dead |
| `public.block_job(job_id, reason)` | Bloqueia job sem consumir attempts |
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

## 11. pg_cron — Jobs Agendados

| Job | Schedule | O que faz |
|---|---|---|
| `segment-eval-fallback` | `*/5 * * * *` | Detecta contatos sem segmentação |
| `expire-soft-bounce-suppressions` | `0 * * * *` | Expira bounces suaves |
| `cleanup-old-analytics-jobs` | `0 4 * * *` | Limpa jobs antigos |
| `create-contact-events-partition` | `0 2 25 * *` | Cria partição mensal |
| `cleanup-event-processing-jobs` | `0 3 * * *` | DELETE success com > 7 dias |
| `rfm-rolling-refresh` | `0 * * * *` | Recalcula RFM de 1/24 dos tenants por hora via `MOD(HASHTEXT(tenant_id), 24)` — garante atualização em até 24h para tenants sem atividade |

> **Pull-based providers (Sympla, Doare):** sync periódico gerenciado pelo pull-sync-worker via `integrations.sync_schedules`. Não usam pg_cron.

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

`crm.opportunities.party_id` é NOT NULL enforçado — oportunidade sem contato é violação do modelo party-first. Trigger `crm.sync_opportunity_summary` (AFTER INSERT/UPDATE(status,party_id)/DELETE) mantém `has_open_opportunity` e `open_opportunity_count` em `contact_state` sincronizados em tempo real.

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

**Coexistência com legacy:** `refreshSegments.ts` usa `rules_json_v2` se `ast_version >= 1`, senão fallback para `build_segment_where_clause`. Métrica `v2_ratio` logada por ciclo. Meta: 100%.

**Estado da migração:** 24/24 segmentos do tenant principal migrados para `rules_json_v2`. Colunas adicionadas em `analytics.segments`: `rules_json_v2`, `ast_version`, `ast_hash`, `migrated_at`, `migration_warnings`.

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

**Typeform — provider futuro:**
Mesmo fluxo, origem diferente. Schema `typeform.*`, `field_definitions.source_system = 'typeform'`, `source_field_id` = ID do campo no Typeform. Webhook para ingestão contínua + Responses API para backfill histórico. Implementação: ver checklist de novo provider (seção 14).

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
| R37 | **Impact Analysis obrigatória antes de mudança em contrato compartilhado.** Antes de alterar formato, renomear ou deprecar qualquer conceito listado em `DEPENDENCY_MAP.md`: (1) listar todos os readers e writers; (2) confirmar que TODOS os readers suportam o novo formato; (3) dual-write por pelo menos 1 sprint antes de dropar formato antigo; (4) campo vazio usado para filtrar = fail-closed (zero resultados, nunca "todos"). | Incidente Sprint 12 — campanha blast radius: migração `rules_json` → `rules_json_v2` quebrou email-campaign-worker que ainda lia v1 vazio, disparando para base inteira do tenant. |
| R38 | **Fail-closed para membership vazia.** Se `segment_parties` retorna 0 rows para um `segment_id` que tem regras definidas (`rules_json_v2 IS NOT NULL`), a operação dependente (envio, export, enrollment) DEVE recusar, nunca prosseguir sem filtro. Zero members + regras definidas = refresh pendente, não "enviar para todos". **Implementações:** (1) `CampaignSend.tsx` — `getSegmentRecipientsFailClosed()` tenta refresh e rejeita se ainda 0; (2) `generate-export/index.ts` — enfileira refresh e aborta export; (3) `checkSegmentEntries.ts` (worker handler) — guard com verificação de `rules_json_v2` + enqueue refresh; (4) `check-journey-entries/index.ts` (Edge Function legada) — guard equivalente como safety net. | Incidente Sprint 12 — mesma origem de R37. |
| R39 | **Skip em nó de automação DEVE retornar `success: false` + `waitUntil`.** Quando um nó `send_email` detecta que a campanha não está pronta (status não-enviável), o handler deve retornar `{ success: false, waitUntil }` para que o enrollment fique em espera e reprocesse. Retornar `success: true` com skip faz o enrollment avançar para o próximo nó sem enviar o email, perdendo o envio silenciosamente. Status `automation` é um status válido para envio em contexto de journey — não deve ser rejeitado. **Lição adicional: Docker layer cache.** Workers containerizados (Railway, ECS, etc.) podem reutilizar layer de build anterior mesmo após push de novo código. Sempre incluir `BUILD_VERSION` constante no entrypoint do worker para: (1) invalidar cache do `COPY src` layer a cada mudança, (2) confirmar nos logs qual versão está rodando. | Incidente Sprint 13 — commit `c3f172e` corrigiu send_email mas Railway serviu código antigo por cache Docker. Worker logava `campaign_not_ready/automation` com `success: true` (código antigo) em vez de enviar o email. 19 enrollments ficaram stuck. |

---

# PARTE D — DÍVIDAS TÉCNICAS & ROADMAP

## 22. Dívidas Arquiteturais

| # | Item | Prioridade |
|---|---|---|
| D5 | Migrar pipeline SendGrid para versão unificada | Pendente |
| D7 | Producer `responded_form_ids` em tempo real (hoje só backfill) | Pendente |
| D8 | Producer `import_ids` em tempo real (hoje só backfill) | Pendente |
| D9 | `journey_ids` em `contact_state` — producer não existe | Pendente |
| D10 | `has_tag` GIN Phase 2 — bloqueado por product tags transitivas não estarem em `tag_ids` | Pendente |
| D11 | `refreshFeatures` cleanup de parties com customer role revogada — zerar `monetary`/`frequency` órfãos em `contact_state` | Pendente |
| D12 | Normalização de telefone (`phone_normalized` E.164) — `crm.party_person`, `sympla.participants`, todos os providers. Função utilitária compartilhada. Backfill + UI de indicador de cobertura. Campanhas WhatsApp filtram por `phone_normalized IS NOT NULL` | Pendente |
| D13 | `DROP TABLE eduzz.integrations` — tabela legada. Toda leitura/escrita já migrada para `integrations.accounts`. RPCs atualizadas. Dropar após validação. | Pendente |
| D14 | Auditoria preventiva de TTL em todas as filas do sistema — verificar se todas as filas têm: (1) cron de purge para rows terminais, (2) índice parcial cobrindo status ativo, (3) NOT EXISTS filtrando apenas status ativos. Ver R20. | Pendente |
| D18 | **Journey fanout não-atômico** — enrollment INSERT e `process_enrollment` job INSERT são statements separadas sem transação. **Fix:** (1) envolver ambos INSERTs em `sql.begin()` em `processEventJob.ts`, `processBackfillJob.ts` e `checkEventEntries.ts`; (2) criar sweeper pg_cron para enrollments órfãos; (3) remover comentários incorretos sobre sql.begin(). Ref: Audit F1. | **P1** |
| D19 | **Observabilidade backend** — Sentry cobre apenas frontend React. Workers sem aggregação, dashboards ou alertas. **Fix:** (1) Sentry nos Railway workers; (2) dashboard de filas (depth, drain rate, p95 latency); (3) alertas: dead jobs > 0, queue depth crescendo, worker sem heartbeat >5min. Ref: Audit F10. | **P2** |
| D20 | **Journeys batch claim** — `journeys.claim_next_job` retorna 1 job por vez. **Fix:** (1) criar `journeys.claim_next_jobs(worker_id, batch_size)` retornando SETOF; (2) batch processing em journeys-worker e email-campaign-worker; (3) per-tenant cap e `maxCycleRuntimeMs`. Ref: Audit F8. | **P3** |
| D21 | **Segment-eval tenant scan** — Worker faz O(tenants) queries por ciclo mesmo sem trabalho. **Fix:** criar `claim_next_segment_jobs_global(worker_id, batch_size, shard_index, total_shards)`. Ref: Audit F2. | **P4** |
| D22 | **claim_jobs tenant fairness** — ordena por `priority, created_at` sem cap por tenant. **Fix futuro:** round-robin CTE com `ROW_NUMBER() OVER (PARTITION BY tenant_id)`. Ref: Audit F3. | Defer |
| D23 | **Idempotency constraint no enqueue** — `enqueue_webhook_job` usa SELECT-then-INSERT (TOCTOU). Risco baixo. **Hardening:** partial unique index em `integrations.jobs(webhook_event_id)`. Ref: Audit F5. | Defer |
| D24 | `DROP TABLE analytics.segment_customers` — período de segurança cumprido | Pendente |
| D25 | `refresh_tenant_distribution` + tela de monitoramento | Pendente |
| D26 | `trg_record_email_engagement` → atualizar `last_email_opened_at` / `last_email_clicked_at` em `contact_state` | Pendente |
| D27 | UI: novos blocos de condição no segment builder (product picker, time window, CRM condition) | Pendente |
| D28 | D6 Fase 3 — `DROP COLUMN customer_id` (tabelas restantes) + 7 FKs + recriar 4 views | Pendente |
| D29 | Performance loading timeout — tenant escola-do-fluxo. Profiling de RPCs lentas (`get_dashboard_data`, `get_pareto_analysis`) + lazy loading analytics. Sentry JAVASCRIPT-REACT-22, 21, 1S | Investigar |
| D30 | Decompor blocos contact_info e address em traits individuais — hoje kind: 'skip', sem dados reais ainda | Pendente |
| D31 | Integração Typeform — schema raw + Edge Function receiver + Responses API backfill + field_definitions com source_system='typeform' | Pendente |
| D32 | `evaluation_mode` coluna em analytics.segments — inferir event_driven vs time_driven por segmento para otimizar re-scan periódico | Pendente |
| D33 | UI: segment builder migrado para AST v2 (`rules_json_v2`) — save/load dual-write, conversão bidirecional, catálogo de campos dinâmico. Ver [`docs/D33-segment-builder-v2.md`](D33-segment-builder-v2.md) | Pendente |
| D34 | Scoping do unique index `processing_jobs_tenant_job_type_pending_uidx` — excluir `project_traits` e `sync_field_definitions` da condição WHERE para permitir múltiplos jobs pendentes per-entity. Ver R32 | **P2** |
| D35 | Completar backfill de traits de formulários — 16 de 30 submissions ainda sem traits projetados. Rodar `backfillFormTraits.ts` com DATABASE_URL de produção | **P1** |
| D36 | **RLS não reconhece system_admin** — policies de RLS usam `tenant_id IN (SELECT tenant_id FROM core.tenant_users WHERE user_id = auth.uid())`, mas system admins podem não ter registro em `tenant_users` para todos os tenants. O RPC `get_user_tenant_memberships` faz LEFT JOIN e retorna todos os tenants, mas o RLS bloqueia queries. **Fix:** criar helper `core.user_visible_tenant_ids(uid)` que retorna todos os tenants para system admins (via `core.system_admins`) e apenas memberships para users normais. Substituir subquery de RLS por chamada a essa função. Workaround atual: inserir system admins em `tenant_users` de cada tenant manualmente. | **P2** |
| D37 | **Cleanup dead code segmentação v1:** (1) branch `ast_version !== 1` no evaluator (`segmentEvaluator.ts`) — fallback v1 nunca atingido, remover; (2) import de `build_segment_where_clause` — função dropada do banco, import morto; (3) referências em scripts de equivalence gate (`scripts/`) que usam `build_segment_where_clause` para comparação v1↔v2 — não mais necessárias. Zero segmentos rule-based em v1_only confirmado via diagnóstico. | Baixa prioridade |
| D38 | **Alerta de anomalia pós-envio.** pg_cron job diário que detecta campanhas com `recipient_source = 'segment'` onde `recipient_count_at_send > segment_parties_snapshot * 1.5` OU `recipient_count_at_send > 80%` da base do tenant. INSERT em tabela de alertas. Depende das colunas de rastreabilidade adicionadas no Sprint 12 (Guard 1). | **P2** |
| D39 | **Backend validation RPC para envio de campanha.** RPC `email.validate_campaign_send(p_tenant_id, p_recipient_count, p_segment_ids, p_recipient_source)` que valida server-side: (1) se source='segment', recipient_count <= SUM(segment_parties) * 1.1; (2) se source='all', recipient_count <= total_contacts do tenant; (3) retorna ok/reject com reason. Defense-in-depth — proteção redundante independente do frontend. | **P2** |
| D40 | **party_id null em enrollments de formulário.** Jornada "Caminho de Madalena" teve 3 enrollments com `party_id` null, causando nó `create_opportunity` stuck (`next_execution_at = null`). Causa provável: `form_submission` criada antes do party ser vinculado, ou `checkFormEntries` não copiando `party_id`. **Investigar:** (1) verificar se `checkFormEntries` garante `party_id NOT NULL` antes de criar enrollment; (2) verificar se `form_submissions` com `party_id` null deveriam ser filtradas; (3) adicionar guard no enrollment: se `party_id IS NULL`, não criar enrollment e logar warning. | **P1** |

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
