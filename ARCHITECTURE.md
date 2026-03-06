# KreatorsHub — Arquitetura Definitiva

## Guia de referência para escala: 50.000 tenants · 50M contatos · 1000 automações por tenant

*Versão 6.6 — ✅ Eduzz OAuth app global ✅ webhook_verified_at ✅ TokenExpiredError ✅ core.kreatorshub.com.br slug ✅ import_sales_page Phase 2 ✅ Clusters/Lookalikes arquitetados (Sprint 19+)*

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
- `eduzz-receiver-webhook` → receiver webhook-driven | salva raw em `integrations.webhook_events` + enfileira jobs (provider=`eduzz-processor-worker`) | URL pública: `core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{slug}` | aceita slug no path ou `?tenant_id=uuid` (legado)
- `eduzz-processor-worker` → processor job-driven | `eduzz:process_invoice`, `eduzz:process_contract`, `eduzz:process_cart_abandonment`, `eduzz:process_commission`, `eduzz:process_learning`, `eduzz:ping`
- `sympla-sync` → cron-driven (sem webhooks nativos) | `sympla:sync_orders`, `sympla:sync_participants`
- `sendgrid-webhook` → receiver webhook-driven | pipeline ativo (escrita direta em `email.*`) | **rename pendente → `sendgrid-receiver-webhook`**
- `webhooks-sendgrid` → receiver alternativo (inativo — pipeline v2 planejado, ver D5)
- `integrations-worker` → órfão sem tráfego (desmembrado em Sprint 6)
- `[outros: oauth, utils]` → funções de suporte (ex: `eduzz-oauth-callback`)

### Convenção de nomenclatura — Edge Functions

| Padrão | Papel | Exemplo |
|---|---|---|
| `{provider}-receiver-webhook` | Recebe requisições HTTP externas do provider. URL pública registrada no painel do provider por tenant. Nunca processa — apenas salva raw e enfileira job. | `eduzz-receiver-webhook` ✅ |
| `{provider}-processor-worker` | Processa jobs da fila com `provider = '{provider}-processor-worker'`. Nunca exposto externamente. | `eduzz-processor-worker` ✅ |
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
- `analytics-worker` → inteligência e segmentação (segment-eval-loop, refreshRfm, computeClusters, computeLookalikes, refreshFeatures). `refreshRfm` escreve em `contact_state` (`customer_features` dropada em Sprint 8)
- `journeys-worker` → execução de nós de automação (`WORKER_ONLY=true`)
- `journey-scheduler` → enfileira checks periódicos (`IS_SCHEDULER=true`): `checkSegmentEntries`, `checkFormEntries`, `checkEventEntries` (apenas `trigger_type='purchase'`), `enqueueReadyWaits`
- **`journeys-event-worker`** → fan-out push-based de eventos. Processa `event_processing_jobs` + `backfill_jobs`
- `eduzz-enrichment` → import histórico Eduzz (`ENABLED_HANDLERS=eduzz:enrich_sales,eduzz:enrich_invoice`)
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

eduzz-enrichment (Railway)
  → busca 1 página da API
  → salva em provider.raw_table
  → mapeia → NormalizedTransaction[]
  → cria job 'commerce:process_batch' para o lote

ingestion-worker → process_transactions_batch → commerce + analytics
```

**Fluxo Eduzz especificamente:**
```
Webhook Eduzz chega em core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{slug}
  → eduzz-receiver-webhook: resolve slug → tenant_id, valida HMAC, salva raw, enfileira job
  → eduzz-processor-worker: claim job → eduzz:process_invoice
  → [priority 1] monta NormalizedTransaction direto do payload
  → job commerce:process_batch
  → ingestion-worker → commerce.transactions
  → [priority 9] job eduzz:enrich_invoice
  → eduzz_upsert_buyer, eduzz_upsert_invoice, enriquecimento crm.party_person (phone, document, address)
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
| Job errors | `integrations.job_errors` | `process_transactions_batch` (erros por item) | observabilidade + reprocess |

---

## 11. pg_cron — Jobs Agendados

| Job | Schedule | O que faz |
|---|---|---|
| `doare-sync-hourly` | `0 * * * *` | Inicia sync Doare |
| `segment-eval-fallback` | `*/5 * * * *` | Detecta contatos sem segmentação |
| `expire-soft-bounce-suppressions` | `0 * * * *` | Expira bounces suaves |
| `cleanup-old-analytics-jobs` | `0 4 * * *` | Limpa jobs antigos |
| `create-contact-events-partition` | `0 2 25 * *` | Cria partição mensal |
| `cleanup-event-processing-jobs` | `0 3 * * *` | DELETE success com > 7 dias |
| `rfm-rolling-refresh` | `0 * * * *` | Recalcula RFM de 1/24 dos tenants por hora via `MOD(HASHTEXT(tenant_id), 24)` — garante atualização em até 24h para tenants sem atividade |
| `sympla-sync-daily` | `0 2 * * *` | Sync diário Sympla (02:00 UTC = 23:00 BRT) — orders + participants por janela de ciclo de vida |

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

**Fases pendentes:**
- Fase 5: checkin → `journeys.journey_events` (type `sympla_checkin`)
- Fase 6: `contact_state.event_checkin_count` + `analytics.contact_checkin_stats`

---

## 14.2 Provider Eduzz — Detalhes de Implementação

**Status:** ✅ Operacional — Webhook-driven + OAuth + import histórico

**Tipo:** Webhook-driven (real-time). Credenciais OAuth + webhook_secret em `integrations.accounts.config`.

**Edge Functions:**
- `eduzz-receiver-webhook` — URL pública, recebe webhooks da Eduzz. Valida HMAC (`x-eduzz-signature`), salva raw em `integrations.webhook_events`, enfileira job com `provider = 'eduzz-processor-worker'`.
- `eduzz-processor-worker` — Worker interno, processa jobs da fila.
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

**Railway worker:** `eduzz-enrichment` — import histórico (`ENABLED_HANDLERS=eduzz:enrich_sales,eduzz:enrich_invoice`).

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
| R25 | Edge Function nunca é worker. Se precisa de poll loop, claim de jobs ou processar lógica pesada, é Railway worker. Sintomas de violação: Edge Function com `while(true)`, `claimJobs()`, múltiplas chamadas RPC em sequência, timeout > 10s, ou lógica que cresce com volume de dados. Qualquer um desses sinais = mover para Railway worker imediatamente. | Separação Edge vs Railway |
| R26 | Webhook de provider deve criar jobs com `provider='ingestion'`. O ingestion-worker é o único consumidor de processamento downstream de webhooks. Edge Function dedicada por provider só existe para recepção e normalização O(1). | Webhook → ingestion pipeline |
| R27 | OAuth providers usam app centralizado da plataforma, nunca app por tenant. Client credentials em env vars globais. Token por tenant salvo em `integrations.accounts.config`. Dependência de app OAuth por tenant = risco operacional: cliente revoga → todos os tenants perdem acesso. | Eduzz OAuth migration Sprint 10 |

---

# PARTE D — ROADMAP & DÍVIDAS TÉCNICAS

## 22. Dívidas Arquiteturais

| # | Item | Status |
|---|---|---|
| D5 | Migrar pipeline SendGrid para versão unificada | Pendente Sprint 10 |
| D7 | Producer `responded_form_ids` em tempo real (hoje só backfill) | Pendente |
| D8 | Producer `import_ids` em tempo real (hoje só backfill) | Pendente |
| D9 | `journey_ids` em `contact_state` — producer não existe | Pendente |
| D10 | `has_tag` GIN Phase 2 — bloqueado por product tags transitivas não estarem em `tag_ids` | Pendente |
| D11 | `refreshFeatures` cleanup de parties com customer role revogada — zerar `monetary`/`frequency` órfãos em `contact_state` | Pendente Sprint 10+ |
| D12 | Normalização de telefone (`phone_normalized` E.164) — `crm.party_person`, `sympla.participants`, todos os providers. Função utilitária compartilhada. Backfill + UI de indicador de cobertura. Campanhas WhatsApp filtram por `phone_normalized IS NOT NULL` | Pendente Sprint 11 |
| D13 | `DROP TABLE eduzz.integrations` — tabela legada. Toda leitura/escrita já migrada para `integrations.accounts` (Sprint 14). RPCs (`eduzz.*`, `public.eduzz_get_integration`, `public.get_integration_config`) já atualizadas. Tabela mantida como fallback; dropar após 30 dias sem incidente. | Pendente Sprint 15 |
| D14 | Auditoria preventiva de TTL em todas as filas do sistema — após incidente de queue bloat em segment_eval_queue (1.1M rows, 482MB, banco travado). Verificar se todas as filas têm: (1) cron de purge para rows terminais, (2) índice parcial cobrindo status ativo, (3) NOT EXISTS filtrando apenas status ativos. Ver R20. | Pendente Sprint 11 |
| D15 | URL customizada `core.kreatorshub.com.br` — custom domain Supabase + slug no path do receiver. Backward compatible com `?tenant_id=uuid`. | ✅ Sprint 11 |
| D16 | `import_sales_page` Phase 2 — raw → `NormalizedTransaction[]` → `commerce:process_batch`. Import histórico agora alimenta o mesmo pipeline do webhook. | ✅ Sprint 11 |

---

## 23. Sprint 10 — Parcialmente Concluído

**Concluído:**
- [x] Rename Edge Functions: `webhooks-eduzz` → `eduzz-receiver-webhook`, `eduzz-webhook` → `eduzz-processor-worker` (funções antigas deletadas do Supabase)
- [x] Custom domain `core.kreatorshub.com.br` + webhook URLs com slug no path
- [x] `claim_jobs` RPC: workers agora passam `p_handlers text[]` no SQL em vez de filtrar em app code
- [x] Provider migration: 87 jobs e webhook_events migrados de `eduzz-webhook` → `eduzz-processor-worker`
- [x] Integration Hub UI: `IntegrationHub.tsx` substitui `EduzzIntegrationTab` direto — grid multi-provider (Eduzz, Sympla, Hotmart em breve) com Sheet lateral por provider
- [x] Sympla Integration UI: `SymplaIntegrationDialog.tsx` — setup com token, seleção de eventos, sync, stats
- [x] Eduzz OAuth app global: `client_id`/`client_secret` em env vars, `access_token` por tenant em `integrations.accounts.config`
- [x] Webhook verification: `webhook_verified_at` + ping test + UI polling automático
- [x] TokenExpiredError: fluxo defensivo 401 → invalidate → blockJob → banner UI
- [x] `import_sales_page` Phase 2: raw → NormalizedTransaction → commerce:process_batch (D16)

**Pendente:**
- [ ] `DROP TABLE analytics.segment_customers` (período de segurança cumprido)
- [ ] D6 Fase 3 — `DROP COLUMN customer_id` (tabelas restantes) + 7 FKs + recriar 4 views
- [ ] D5 — Migrar pipeline SendGrid para versão unificada
- [ ] `refresh_tenant_distribution` + tela de monitoramento
- [ ] Conectar `trg_record_email_engagement` para atualizar `last_email_opened_at` / `last_email_clicked_at` em `contact_state`
- [ ] UI: novos blocos de condição no segment builder (product picker, time window, CRM condition)
- [ ] Sympla Fase 5: checkin → `journeys.journey_events` (type `sympla_checkin`)
- [ ] Sympla Fase 6: `contact_state.event_checkin_count` + `analytics.contact_checkin_stats` (Tier 1.5)

---

## 24. Sprint 11 — Pendente

### Sympla Integration v5

**Tabela `sympla.participants`** criada com PK `(tenant_id, id)`, campos de ticket (`ticket_number`, `ticket_name`, `ticket_discount`), checkin (`checkin_at`, `checkin_synced`), e extração de `custom_form` (`phone`, `cpf`). Índices: order lookup, email lookup, checkin_pending (partial).

**Edge Function `sympla-sync` v5** — mudanças em relação à v4:
- **Timezone fix:** `parseSymplaDate` agora detecta offset no payload; datas sem offset recebem `-03:00` (BRT) em vez de `Z` (UTC). Fix retroativo: 775 `commerce.transactions` corrigidas (`paid_at + 3h`).
- **Novas colunas:** events (17 cols: `reference_id`, `detail`, `host_*`, `category_*`, `address_*`, `checkin_synced`) e orders (5 cols: `presentation_id`, `address_*`, `user_agent`).
- **Participants sync:** `syncParticipants()` com janela temporal — future (>-1d), active (-1d a +3d), recently-ended (+3d a +7d), archived (skip se `checkin_synced`). Extrai phone/CPF de `custom_form` via regex case-insensitive.
- **Mapper:** `NormalizedTransaction.extra` inclui `city`, `state`, `presentation_id`.

**pg_cron:** `sympla-sync-daily` às 02:00 UTC (23:00 BRT).

### Performance: Loading Timeout Recorrente (escola-do-fluxo)

**Status:** Identificado, não resolvido
**Tenant:** `escola-do-fluxo` (`48ef8a5c-283b-4943-8552-53e8f8e92c3a`)
**User:** `68d43ac4-f65c-479a-a7b9-84889424965c`
**Sentry Issues:** JAVASCRIPT-REACT-22, 21, 1S

**Problema:** Watchdog `LOADING_TIMEOUT` recorrente — 10 queries ficam penduradas por >10s em múltiplas rotas (`/admin/analytics`, `/contatos`, `/admin/clients/*`). Logs do Supabase mostram status codes 520 (internal error) e 525 (SSL handshake failed) em RPCs como `get_user_features`, `get_tenant_account`, `get_dashboard_data`, `get_pareto_analysis`, `transactions`.

**Ocorrências registradas no Sentry:**
- 04/03: `/admin/analytics` — 10 fetching, 520/525 nos logs Supabase
- 03/03: `/contatos` — 10 fetching
- 12-27/02: `/admin/clients/*` — 17 páginas diferentes, 7 fetching (issue 1S, 21 ocorrências)

**Hipóteses a investigar:**
1. Queries pesadas para este tenant (volume de dados grande?)
2. Conexão de rede instável do usuário (520/525 = falha SSL/conexão)
3. RPCs lentas que excedem timeout do PostgREST (~30s default)
4. Concorrência de queries — 10 RPCs simultâneas saturando pool de conexões

**Ações necessárias:**
- [ ] Verificar tamanho dos dados do tenant (transações, contatos, contact_state)
- [ ] Profiling das RPCs mais lentas (`get_dashboard_data`, `get_pareto_analysis`, `get_analytics_data`)
- [ ] Considerar lazy loading / paginação na página de analytics
- [ ] Avaliar se 10 queries simultâneas no mount é necessário ou pode ser sequencial/priorizado
- [ ] D14 — Auditoria de TTL em todas as filas: inventariar todas as tabelas com coluna `status`, verificar acúmulo de rows terminais, criar crons de purge e índices parciais onde faltarem. Modelo de referência: correção aplicada em `segment_eval_queue` após incidente de queue bloat.

---

## 25. Sprint 14 — Unificação `integrations.accounts`

- [x] Migrar `eduzz.integrations` → `integrations.accounts` (colunas OAuth/sync/webhook genéricas)
- [x] Copiar dados de 4 tenants via UPSERT (paridade 100%)
- [x] Atualizar 7 RPCs (`eduzz.init_sync`, `eduzz.increment_sync_progress`, `eduzz.fail_sync`, `eduzz.reset_sync`, `public.eduzz_get_integration`, `public.eduzz_update_integration_stats`, `public.get_integration_config`)
- [x] Edge Functions: `eduzz-oauth-callback` e `eduzz-receiver-webhook` (ex-`webhooks-eduzz`) — removido fallback legado
- [x] Worker `eduzz/supabase.ts`: `getEduzzIntegration`, `updateEduzzToken`, `invalidateEduzzToken` → `integrations.accounts`
- [x] Frontend: `eduzzStorage.ts` CRUD migrado, `IntegrationHub.tsx` sem `eduzzLegacy`, `EduzzIntegrationTab.tsx` usa `status` em vez de `is_enabled`
- [x] Tipo `EduzzIntegration` reflete `integrations.accounts` (com `config` JSONB, `status`, colunas OAuth)
- [ ] D13: `DROP TABLE eduzz.integrations` — aguardar 30 dias sem incidente (Sprint 15)

---

## 26. Sprints Futuros

**Sprint 17 — Personalização e cache**
- Vercel KV para cache de edge (`contact_state` hot path)
- Identity resolution em checkout e formulários
- `visitor_sessions` conectado ao frontend

**Sprint 18 — Dashboard Piloto Automático**
- API de oportunidades de receita, cards de revenue em risco
- Attribution dashboard, product ladder visualization

**Sprint 19+ — ML Real**
- Schema `cluster_runs` / `cluster_definitions` / `cluster_members` / `cluster_segments`
- `source_run_id` indexável em `segment_parties`
- `computeClusters.ts` — K-means k=5 lendo `contact_state`
- UI de renomear label de cluster, features comportamentais
- HDBSCAN/GMM via microserviço Python, lookalike por embeddings, A/B testing com bandit
