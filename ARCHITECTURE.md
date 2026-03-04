# KreatorsHub — Arquitetura Definitiva

## Guia de referência para escala: 50.000 tenants · 50M contatos · 1000 automações por tenant

*Versão 5.52 — ✅ RFM fonte única de verdade ✅ contact_state canonical ✅ Clusters/Lookalikes arquitetados (Sprint 19+)*

---

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
- `eduzz-webhook` → webhook-driven | `eduzz:process_invoice`, `eduzz:process_contract`, `eduzz:process_cart_abandonment`, `eduzz:enrich_invoice` (placeholder)
- `sendgrid-webhook` → webhook-driven | pipeline ativo (escrita direta em `email.*`)
- `webhooks-sendgrid` → receiver alternativo (inativo — pipeline v2 planejado, ver D5)
- `integrations-worker` → órfão sem tráfego (desmembrado em Sprint 6)
- `[outros: oauth, utils]` → funções de suporte

**RAILWAY WORKERS** (processamento — Node.js, postgres.js, PgBouncer)

- `ingestion-worker` → `commerce.transactions` + analytics pipeline (handler genérico, provider-agnóstico)
- `analytics-worker` → inteligência e segmentação (segment-eval-loop, refreshRfm, computeClusters, computeLookalikes, refreshFeatures). `refreshRfm` escreve em `contact_state` (`customer_features` dropada em Sprint 8)
- `journeys-worker` → execução de nós de automação (`WORKER_ONLY=true`)
- `journey-scheduler` → enfileira checks periódicos (`IS_SCHEDULER=true`): `checkSegmentEntries`, `checkFormEntries`, `checkEventEntries` (apenas `trigger_type='purchase'`), `enqueueReadyWaits`
- **`journeys-event-worker`** → fan-out push-based de eventos (NOVO — Sprint 6). Processa `event_processing_jobs` + `backfill_jobs`
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
Webhook Eduzz chega
  → eduzz:process_invoice (Edge Function)
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

**Status:** ✅ Operacional — Sprint 6 (26/02)

**Problema com design anterior (polling):** O design original usava `check_event_entries` com polling a cada 60s e marcava eventos como `processed = true` globalmente. Isso criava dois problemas:

1. Ledger mutável — jornada criada após o evento nunca o via, mesmo dentro de sua janela de tempo
2. Re-scan repetitivo — a cada 60s re-lia todo o histórico desde `last_activated_at`

**Princípios do novo design:**
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

**Comparativo design antigo vs novo:**

| Aspecto | Design antigo (polling) | Design novo (push) |
|---|---|---|
| Latência | ~60s | < 5s |
| `journey_events` | Mutável (`processed = true`) | Ledger imutável |
| Jornada criada depois | Não via eventos históricos | Via backfill explícito |
| Carga no scheduler | Proporcional a jornadas × tenants | Zero polling para event_entry |
| Histórico | Re-scan completo a cada 60s | Cursor avança, sem re-scan |

**Shared util — `journeyMatcher.ts`:** `workers/journeys-event/src/shared/journeyMatcher.ts`
Lógica de filtro de jornada compartilhada entre `processEventJob.ts` e `processBackfillJob.ts`.

- `extractMatchContext(eventData)` → extrai `product_id`, `campaign_id`, `url` do evento
- `resolveProductIds(tenantId, ctx)` → resolve IDs reais de `commerce.products` (com cache in-memory por batch)
- `matchesJourneyFilters(conditions, ctx, resolvedIds)` → avalia filtros de produto, campanha e URL

**Scheduler — check_event_entries desabilitado:** A função SQL `journeys.enqueue_journey_checks()` foi atualizada em Sprint 6 (26/02) para remover `'event_entry'` do loop. O handler `checkEventEntries.ts` permanece no código como **fallback emergencial** — nunca deve ser agendado salvo incidente no journeys-event-worker.

**ATENÇÃO — trigger_type no banco:** O valor correto salvo no banco é `'event_entry'` (não `'event'`). O frontend foi corrigido em Sprint 6 (26/02):
- `src/types/journey.ts`: TriggerType inclui `'event_entry'`
- `JourneyBuilder.tsx`: `detectTriggerFromNodes()` salva `trigger_type: 'event_entry'`
- `NodeInspector.tsx`: `sourceType` usa `'event_entry'`

As 3 jornadas existentes com `trigger_type='event'` foram migradas via UPDATE.

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
3. Faz INSERT com `ON CONFLICT DO UPDATE` — transições unidirecionais: `paid` permanece `paid` exceto para `refunded`; qualquer outro status pode transicionar para `paid`
4. `paid_at` é preenchido via `COALESCE` — nunca sobrescreve se já existia
5. Se o status não mudou (`v_previous_status = v_status`) → skip, não roda analytics
6. Analytics pipeline só roda na transição PARA `paid`: novo insert `paid`, ou mudança de `pending`/`cancelled` para `paid`

---

## 7. analytics.process_purchase_analytics — Pipeline consolidado

Uma única RPC executada dentro do loop de `process_transactions_batch`. Substitui 4 chamadas separadas.

**Executa 4 operações em sequência na mesma transação:**

1. `INSERT contact_events` — ledger imutável (idempotência por `transaction_id`)
2. `INSERT contact_state ON CONFLICT DO UPDATE` — snapshot vivo
3. `INSERT segment_eval_queue priority=1` — fila de segmentação
4. `INSERT tenant_distribution ON CONFLICT` — marca tenant como stale

**Benchmark medido:** 7.170ms → 1.174ms por job (-83%), P95 de 13s → 1.2s (-91%).

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
| `cleanup-event-processing-jobs` | `0 3 * * *` | DELETE success com > 7 dias (Sprint 7+) |
| `rfm-rolling-refresh` | `0 * * * *` | Recalcula RFM de 1/24 dos tenants por hora via `MOD(HASHTEXT(tenant_id), 24)` — garante atualização em até 24h para tenants sem atividade |

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

### Campos array — estado dos producers (Sprint 9)

| Campo | Producer | Status |
|---|---|---|
| `bought_product_ids` | `process_purchase_analytics` + `linkProducts.ts` (recalcula array completo após link) | ✅ Ativo |
| `tag_ids` | Triggers DB em `crm.customer_tags` + `crm.lead_tags` (after insert/delete) | ✅ Ativo |
| `import_ids` | Backfill aplicado — 31.389 parties | ✅ Populado — producer tempo real pendente (D8) |
| `responded_form_ids` | Backfill aplicado — 12 parties | ✅ Populado — producer tempo real pendente (D7) |
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

**Dedup:** busca oportunidade `status='open'` no mesmo `pipeline_id` pelo contato. Chave preferencial e única: `party_id`. Fallback `contact_email` removido em Sprint 7 — migração party-first completa.

**Se encontrou:** calcula diff de campos → UPDATE oportunidade → INSERT em `crm.opportunity_stage_history` se etapa mudou → INSERT em `crm.opportunity_activities` → retorna `opportunity_updated`.

**Se não encontrou:** cria nova oportunidade → INSERT em `crm.opportunity_activities` com `activity_type='created'` → retorna `opportunity_created`.

**Regras da timeline (`crm.opportunity_activities`):**
- Append-only — nunca fazer UPDATE
- Idempotência via UNIQUE INDEX parcial em `(enrollment_id, node_id)` — reprocessamento não duplica
- Backfill usa `source='system'` com `source_meta=null` para não colidir com INSERTs de jornada (54 oportunidades backfilladas em Sprint 6)
- Activity é obrigatória: se o INSERT falhar, o node falha inteiro e o journeys-worker faz retry — nunca usar `try/catch` isolado no INSERT de activity

`email_opened` / `email_clicked` — emitidos por trigger AFTER INSERT em `email.email_events` (resolve party via `recipient_id → campaign_recipients → crm.parties`)

`segment_entered` / `segment_exited` — emitidos por `apply_segment_membership_diff` a cada mudança de membership

`crm.opportunities.party_id` é NOT NULL enforçado desde Sprint 8 — oportunidade sem contato é violação do modelo party-first. Trigger `crm.sync_opportunity_summary` (AFTER INSERT/UPDATE(status,party_id)/DELETE) mantém `has_open_opportunity` e `open_opportunity_count` em `contact_state` sincronizados em tempo real.

---

## 14. Checklist para Novo Provider

Adicionar um provider novo é criar **1 arquivo de mapper**. Zero mudança no banco.

**Edge Function:**

- [ ] Criar Edge Function dedicada:
  ```
  supabase/functions/<provider>/
  ├── _core.ts          (copiar de qualquer Edge Function existente)
  ├── handlers/<provider>.ts    (lógica do provider)
  └── index.ts          (só registrar handlers do provider, claim com provider='<provider>')
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

## 16. Roadmap de Sprints

### Sprint 9 — pendência de segmentação

- Conectar `trg_record_email_engagement` para atualizar `last_email_opened_at` / `last_email_clicked_at` em `contact_state` (trigger do Sprint 7 só escreve em `contact_events` — falta o UPDATE em `contact_state`)
- UI: novos blocos de condição no segment builder (product picker, time window, CRM condition)

### Sprint 10

- [ ] `DROP TABLE analytics.segment_customers` (período de segurança cumprido)
- [ ] D6 Fase 3 — `DROP COLUMN customer_id` (tabelas restantes) + 7 FKs + recriar 4 views
- [ ] D5 — Migrar pipeline SendGrid para versão unificada (ver seção 20)
- [ ] `refresh_tenant_distribution` + tela de monitoramento

### Sprint 17 — Personalização e cache

- [ ] Vercel KV para cache de edge (`contact_state` hot path)
- [ ] Identity resolution em checkout e formulários
- [ ] `visitor_sessions` conectado ao frontend

### Sprint 18 — Dashboard Piloto Automático

- [ ] API de oportunidades de receita
- [ ] Cards de revenue em risco
- [ ] Attribution dashboard
- [ ] Product ladder visualization

### Sprint 19+ — ML Real

- [ ] Implementar schema `cluster_runs` / `cluster_definitions` / `cluster_members` / `cluster_segments`
- [ ] Adicionar `source_run_id` indexável em `segment_parties`
- [ ] `computeClusters.ts` — K-means k=5 lendo `contact_state`, chamando `apply_cluster_membership`
- [ ] UI de renomear label de cluster (`label_overridden_at`)
- [ ] Fase 2: features comportamentais (`avg_order_value`, `email_open_rate_30d`)
- [ ] Fase 3: HDBSCAN/GMM via microserviço Python
- [ ] Lookalike por embeddings
- [ ] A/B testing com bandit algorithm

---

## 17. Deploy no Railway — Configuração por Service

Todos os services usam build context = raiz do repo. Cada service define seu Dockerfile via env var `RAILWAY_DOCKERFILE_PATH`.

**Separação de workers por perfil, não por provider:** Worker único com múltiplas réplicas é o modelo correto para escala. Separar só quando perfis divergem em SLA: realtime (webhooks, segundos) vs batch (imports, minutos). `journeys-event-worker` foi separado do `journeys-worker` por ter SLA < 5s vs minutos.

---

## 18. Observabilidade — O que monitorar

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

## 19. Incidentes e Lições Aprendidas

Esta seção registra conhecimento arquitetural extraído de incidentes reais. Não é um log operacional — é o que o sistema aprendeu e nunca deve esquecer.

---

### Lição 1 — Dedup cross-source (Incidente Eduzz, Fev 2026)

O índice de idempotência é `UNIQUE ON (tenant_id, source_name, external_transaction_id)`. `source_name` faz parte da chave **intencionalmente** — permite rastrear múltiplas fontes do mesmo dado (ex: webhook + import histórico).

**Consequência obrigatória:** o enqueue de qualquer provider DEVE verificar todas as fontes antes de criar jobs, sem filtrar por `source_name`. Se o enqueue filtrar só a própria fonte, re-importações criam duplicatas que passam pela constraint sem serem detectadas.

A proteção depende de todos os pipelines do mesmo provider usarem o mesmo `source_name` canônico. Documentado e auditado em Sprint 6 — zero duplicatas em 35.312 transações.

---

### Lição 2 — Amount: sempre bruto, nunca líquido

Mappers devem usar explicitamente o campo de valor bruto do provider. Nunca usar valor líquido (após taxas) como `amount`. Para providers com moeda estrangeira, converter para BRL usando cotação disponível no payload, com fallback para 1.

---

### Lição 3 — Controle de idempotência do enqueue

O controle de "já foi processado?" deve usar a **tabela de destino** (`commerce.transactions`), nunca um campo de status na tabela raw.

**Errado:** `WHERE processing_status = 'pending'` em `doare.donations` — dessincroniza quando o worker não faz callback, cria dependência de estado na tabela raw.

**Correto:** `NOT EXISTS (SELECT 1 FROM commerce.transactions WHERE ...)` — a fonte de verdade é o destino, idempotência garantida pela constraint UNIQUE.

`doare.donations.processing_status` foi deprecated em Fev 2026. Coluna não existia no DB no momento da auditoria (Sprint 8) — type stale removido de `workers/ingestion/src/types.ts`.

---

### Lição 4 — Edge Functions: responsabilidade única

Edge Functions que crescem sem limite viram gargalo de manutenção e deploy. Solução aplicada no Sprint 6 (25/02): `integrations-worker` (5 providers) → 4 Edge Functions dedicadas.

**Regra permanente:** um provider = uma Edge Function. Deploy de um provider não arrisca os outros.

---

### Lição 5 — raw_source é opcional quando existe schema raw dedicado

| Provider | Rastreabilidade raw |
|---|---|
| doare | `doare.donations` via `external_transaction_id = doare_id` |
| eduzz | `eduzz.invoices` via `external_transaction_id = eduzz_invoice_id` |
| Provider sem schema raw | `raw_source` obrigatório no payload do job |

---

### Lição 6 — Validar o deploy, não só o código

**Checklist obrigatório após qualquer mudança em Edge Function:**

- [ ] `supabase functions deploy <nome>` executado
- [ ] Versão confirmada no dashboard (deve ser > versão anterior)
- [ ] Primeiro ciclo do cron/webhook validado nos logs
- [ ] Resultado do job inclui o campo novo esperado

---

### Lição 7 — Contrato de tipos frontend → banco deve ser explícito (Incidente event_entry, Fev 2026)

O frontend salvava `trigger_type = 'event'` mas toda a infraestrutura push-based foi construída com `trigger_type = 'event_entry'`. A divergência ficou invisível porque não havia validação no INSERT de jornadas.

**Consequência:** routing index, trigger GENERATED, `enqueue_journey_checks` — tudo estava correto mas nunca encontrava jornadas reais porque o valor no banco era diferente.

**Resolução:** 3 arquivos TypeScript corrigidos + UPDATE nas 3 jornadas existentes + migration futura para adicionar CHECK constraint em `journeys.journeys(trigger_type)`.

**Regra permanente:** qualquer valor de enum salvo no banco deve ter CHECK constraint correspondente. Tipos TypeScript e valores de banco devem ser auditados juntos em qualquer PR que crie novo `trigger_type`, `status`, ou campo categórico.

---

### Lição 8 — Campos de controle de fluxo não pertencem ao settings_json (Incidente journeys-event-worker, Fev 2026)

O worker tentou `SELECT allow_reentry FROM journeys.journeys` — coluna não existia, valor estava em `settings_json`. Isso causou 35 dead jobs.

**Causa raiz dupla:** (1) schema real não foi verificado antes de escrever o SELECT; (2) campos de controle de fluxo foram enterrados em JSON sem type safety, sem NOT NULL, sem default garantido pelo banco.

**Resolução:** `allow_reentry`, `force_reentry` e `enroll_existing` migrados para colunas `boolean NOT NULL DEFAULT false`. 9 arquivos atualizados. `settings_json` mantido apenas para configurações verdadeiramente opcionais e variáveis por tipo de jornada.

**Regras permanentes:**
- Antes de qualquer SELECT em tabela existente, confirmar schema real — nunca assumir que campos TypeScript têm colunas 1:1
- Campos que controlam lógica de negócio (reentrada, elegibilidade, limites) devem ser colunas tipadas, nunca JSON
- `settings_json` é para preferências de UI e metadados opcionais — nunca para flags de comportamento do worker

---

### Lição 9 — Duas fontes de verdade para RFM (Sprint 6, 26/02)

`v_segment_base` calculava RFM on-the-fly via `ntile(5)`, enquanto `refreshRfm` materializava em `customer_features`. UI e export divergiam.

**Resolução:** `v_customer_rfm_base` e `v_customer_rfm_scores` dropadas, `v_segment_base` reescrita para ler de `contact_state`, `refreshRfm` sincroniza scores para `contact_state` após cada ciclo.

**Regra permanente:** nenhuma view deve recalcular RFM em runtime — `contact_state` é o snapshot, workers são os únicos produtores.

---

### Lição 10 — Migração party-first: cortar o modelo antigo, não adaptar (Sprint 7, 26/02)

`crm.leads` existia como entidade com tabela própria. A tentativa de "consertar" o fluxo que passava por ela (`ensure_lead_for_party`, FK em opportunities, `segment_leads`) gerava patches em cima de modelo errado.

**Resolução:** cortar os fluxos na raiz — sem adaptar, sem fallback temporário. Lead é role de um party (`crm.party_role_assignments`), não entidade separada.

**Sequência executada:**
1. FK `opportunities_lead_id_fkey` → removida
2. `ContactSelector` → upsert direto em `crm.parties`, sem `createLead`
3. `segment_leads` → dropada, substituída por `segment_parties` (`source='lead'`)
4. 6 RPCs que referenciavam `segment_leads` → reescritas
5. `crm.leads` → write protection enforced no banco → depois dropada
6. 5 tabelas filhas (`lead_tags`, `lead_notes`, etc.) → `lead_id` nullable, queries migradas para `party_id`
7. `ensure_lead_for_party` → removida de todos os callers ativos

**Regra permanente:** quando um modelo de dados está errado, dropar é mais seguro que adaptar. Adaptar prolonga a coexistência de dois modelos e aumenta a superfície de bugs. CASCADE com auditoria prévia de dependências é o caminho.

---

### Lição 11 — Migração de coluna em produção: dados primeiro, código depois, DROP por último (D6, Sprint 7, 26/02)

A migração `customer_id → party_id` revelou que backfill de dados (100% `party_id` populado) e migração de código são etapas independentes e devem ser executadas separadamente. Tentar dropar colunas antes de migrar todo o código que as lê quebra produção.

**Sequência correta:**
1. Backfill dados — garantir cobertura 100% na coluna nova
2. Migrar escritas — sistema para de popular coluna antiga
3. Migrar leituras — functions, workers, frontend
4. `DROP COLUMN` — só após período de segurança

Efeito colateral positivo: ao parar de chamar `ensure_customer_for_party`, 11 parties que eram silenciosamente excluídas do RFM (sem row em `crm.customers`) passaram a ser incluídas.

**Regra permanente:** nunca dropar coluna na mesma sessão em que migra o código. Período de segurança mínimo de 1 sprint entre Fase 2 (código) e Fase 3 (DROP).

---

### Lição 12 — Trigger de role assignment deve manter invariante lead→customer (Sprint 7, 27/02)

`assign_customer_role_on_paid_transaction` atribuía customer role mas não revogava o lead role. `upsert_party_person` fazia corretamente (step 4), mas o trigger não.

**Efeito:** 1.750 parties com ambos os roles ativos. Dashboard mostrava 925 clientes em vez de 2.675 — diferença de 1.750 caindo na categoria "both" invisível para o card.

**Causa:** dois caminhos de atribuição de role com comportamentos divergentes.

**Resolução:** migration `fix_trigger_revoke_lead_on_customer_assignment` + backfill 1.751 lead roles revogados cross-tenant. `both=0` confirmado em todos os tenants.

**Regra permanente:** qualquer função ou trigger que atribui um role superior deve revogar os roles inferiores da mesma hierarquia. Invariante: lead e customer são mutuamente exclusivos — customer sempre prevalece.

---

### Lição 13 — RFM tem um único produtor (Sprint 7+)

O sistema tinha 5 locais calculando RFM com regras divergentes: `get_customer_rfm()` recalculava NTILE on-the-fly sobre todos os clientes (sem regra dos 365 dias), `v_segment_base` + 4 RPCs replicavam o CASE de 365 dias independentemente. UI e export mostravam scores diferentes para o mesmo contato.

**Regra dos 365 dias canônica:** ativos = `last_purchase_at >= NOW() - INTERVAL '365 days'`. NTILE calculado apenas entre ativos. Perdidos recebem `r_score=0`, `rfm_segment='Perdido'`.

**Resolução:** `get_customer_rfm()` eliminada (lê direto de `contact_state`). `v_customer_rfm_base` e `v_customer_rfm_scores` dropadas. 4 RPCs migradas de `customer_features` → `contact_state`. 1.832 contatos com segmentos legacy ("Hibernating"/"Champions") corrigidos para "Perdido". Cron `rfm-rolling-refresh` adicionado para tenants sem atividade (spread de 1/24 por hora).

**Regra permanente:** NTILE só existe em `refreshRfm.ts`. Todo consumidor lê `contact_state.rfm_segment` diretamente. Dois produtores de RFM = bug.

---

### Lição 14 — Índices redundantes e dead tuples custam 18x em performance (Sprint 7+)

`customer_features` tinha 12 índices para tabela de escrita intensiva — cada upsert atualizava 12 índices desnecessariamente. `contact_state` e `commerce.transactions` acumulavam milhares de dead tuples sem VACUUM adequado.

**Otimizações:** VACUUM em `transactions` (5.977 dead tuples) e `contact_state` (6.535 dead tuples). Drop de 5 índices redundantes em `customer_features` e `legacy_party_map`. Novo índice composto em `legacy_party_map (tenant_id, legacy_type, legacy_id)`.

**Resultados medidos:** `get_customer_ids_paginated` 208ms → 11ms (18x), disk reads -98%. `refreshRfm` fetch 229ms → 13ms (17.6x). `customer_features`: 12 → 7 índices.

**Regra permanente:** auditar índices e dead tuples de tabelas quentes a cada sprint. Tabela com upserts constantes não deve ter mais índices do que queries distintas que a lêem.

---

### Lição 15 — Clusters e Lookalikes: contrato output-table agnóstico ao algoritmo (Sprint 19+)

Arquitetura planejada (não implementada). O produto precisa expor clusters como segmentos acionáveis em jornadas sem conhecer o algoritmo que os gerou.

**Princípio central:** o algoritmo é plugável, o output é um contrato fixo.

**Tabelas do contrato:**
- `analytics.cluster_runs` — auditoria: qual algoritmo, quais features, qual versão, status
- `analytics.cluster_definitions` — um registro por cluster: label (gerado pelo worker, editável pelo usuário), centróide, `label_overridden_at` (null = worker gerou, not null = usuário editou)
- `analytics.cluster_members` — membros com score por `run_id` (semântica de score definida pelo run, nunca comparar entre runs)
- `analytics.cluster_segments` — vínculo 1:1 `cluster_definition_id → segment_id` (unique constraint garante que um cluster nunca vira dois segmentos)

**Guardrails não-negociáveis:**
1. `segment_parties` deve ter coluna `source_run_id` indexável — necessário para DELETE seguro no replace (nunca filtrar por JSON)
2. Ninguém fora do compute depende de `cluster_index` como identidade semântica — apenas `cluster_definition_id` e `segment_id` são estáveis
3. `apply_cluster_membership` processa cluster por cluster (evita storm em tenants grandes)
4. `features_version` é migração versionada (ex: `contact_state_v3`), não número mágico — permite rollback e comparação entre versões

**Fluxo:**
```
contact_state (input)
  → compute (K-means, HDBSCAN, etc.)
  → cluster_members
  → apply_cluster_membership(run_id)
  → segment_parties
  → segment_eval_queue
  → jornadas
```
O publish dispara avaliação automática — sem confirmação manual.

**Roadmap algoritmo:**
- Fase 1: K-means k=5, features RFM 3D, semanal
- Fase 2: +comportamento (`avg_order_value`, `email_open_rate_30d`, `distinct_products`)
- Fase 3: HDBSCAN/GMM via microserviço Python
- Fase 4: embeddings

---

### Lição 16 — Membership e metadata de algoritmo são tabelas diferentes (Sprint 8, Mar 2026)

`segment_customers` acumulava dois tipos de dado conceitualmente distintos: membership (quais parties pertencem a qual segmento) e metadata de algoritmo (score, tier, `explanation_json` para lookalikes).

Tentar migrar `segment_customers` diretamente para `segment_parties` revelou o gap: `segment_parties` é tabela de membership pura, não tem colunas de score/tier.

**Resolução correta:** separar por responsabilidade.
- `segment_parties` = membership canônica para todas as fontes (cluster, lookalike, rule-based)
- `customer_lookalike_scores` = metadata do algoritmo com `segment_id` (score, tier, explanation)

O JOIN é feito on-demand por `get_lookalike_members()` quando o frontend precisa de ambos. `segment_customers` entrou em modo zero-write e será dropada no Sprint 10.

**Regra permanente:** tabelas de membership não devem carregar metadata de algoritmo. Membership = pertence/não pertence. Score/tier = output do compute, vive na tabela do compute.

---

### Lição 17 — Condições correlacionadas requerem tabelas de summary, não JSONB dinâmico

O problema "primeira compra do produto X nos últimos 30 dias" não pode ser resolvido com AND de condições independentes contra `contact_state` — o filtro temporal e o filtro de produto precisam ser avaliados juntos contra o mesmo registro.

Tentativa de solução via JSONB (`product_stats` em `contact_state`) descartada: GIN não resolve range queries em campos dinâmicos, evolução de schema é custosa, debug é difícil.

**Solução correta:** tabela dedicada `contact_product_stats(tenant_id, party_id, product_id, first_purchase_at, ...)` com índices compostos reais. EXISTS na avaliação do segmento. O mesmo padrão se aplica a qualquer dimensão com alta cardinalidade (campanhas, afiliados, etc.).

**Regra permanente:** campos de summary com cardinalidade alta (por produto, por campanha) vivem em tabelas dedicadas, nunca em JSONB. JSONB é para campos globais estáveis e de baixa dimensionalidade.

### Lição 18 — Campo array vazio = segmento silenciosamente errado (Sprint 9)

`bought_product_ids` tinha 32% dos contatos com produtos faltando. O evaluator usava subquery em `commerce.transactions` (lenta mas correta). Ao migrar para GIN, contagens colapsaram: `linkProducts.ts` atualizava `commerce.transactions` com `product_id` mas não propagava para `contact_state.bought_product_ids`. O campo ficava com 1 produto quando o contato tinha 4. O producer de `process_purchase_analytics` faz `array_append` incremental — funciona apenas para compras novas via webhook, não para retroativos.

**Fix:** (1) `linkProducts.ts` recalcula array completo após UPDATE transactions, (2) backfill cross-tenant corrigiu 3.454 contatos, (3) validação divergência = 0 antes de reaplicar GIN.

Rollback bem executado evitou semanas de contagens erradas em produção.

**Regra permanente:** antes de migrar qualquer condition de subquery para GIN, rodar validação de divergência com EXPLAIN ANALYZE e comparação de contagens por segmento. Divergência > 0 = não migrar. O rollback é a decisão correta.

---

## 20. Bugs e Dívidas Pendentes — Sprint 7+

### Dívidas arquiteturais

| # | Item | Status |
|---|---|---|
| D5 | Migrar pipeline SendGrid para versão unificada | Pendente Sprint 10 |
| D7 | Producer `responded_form_ids` em tempo real (hoje só backfill) | Pendente |
| D8 | Producer `import_ids` em tempo real (hoje só backfill) | Pendente |
| D9 | `journey_ids` em `contact_state` — producer não existe | Pendente |
| D10 | `has_tag` GIN Phase 2 — bloqueado por product tags transitivas não estarem em `tag_ids` | Pendente |

### D5 — Migrar pipeline SendGrid para versão unificada

**Pipeline atual** (`sendgrid-webhook`, ativo): escreve direto em `email.email_events` sem passar por `integrations.jobs`. Bugs conhecidos: 68 eventos `'blocked'` ignorados, soft bounce suprime no 1º bounce (51 emails suprimidos indevidamente), `spamreport` não registra em `unsubscribes`.

**Handler v2** (`sendgrid.ts`, nunca ativado): tem soft bounce retry, threshold adaptativo, `ALLOWED_PREVIOUS`.

**Quando resolver:**
1. Criar pipeline unificado em `sendgrid-events/handlers/sendgrid.ts`
2. Testar em staging com eventos reais
3. Configurar webhook SendGrid para `/functions/v1/sendgrid-events`
4. Desativar `sendgrid-webhook` após validação

---

## 21. Rule Engine v2 — Segmentação Escalável

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
- Backfill histórico executado: 21.299 registros, 15.489 contatos, desde 2021

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

### Estado do evaluator (Sprint 9)

| Condition | Implementação | Performance |
|---|---|---|
| `bought_product` / `not_bought_product` | GIN `bought_product_ids && ARRAY[]` | ✅ 530x — ativo desde Sprint 9 |
| `has_tag` / `not_has_tag` | 4 subqueries (diretas + product tags transitivas) | Bloqueado — product tags não estão em `tag_ids` |
| `responded_form` / `not_responded_form` | Subquery `lead_form_submissions` | Phase 2 — após producer tempo real |
| `from_import` / `not_from_import` | Subquery `lead_import_history` | Phase 2 — após producer tempo real |
| Campos RFM / lifecycle / scores | Colunas diretas via `v_segment_base` | Já O(1) |

### Soft-delete — segments e forms (Sprint 9)

- `analytics.segments`: coluna `deleted_at timestamptz` adicionada. Soft-delete via `UPDATE SET deleted_at = NOW(), is_active = false` — nunca DELETE físico
- `forms.forms` e `smart_forms.forms`: `deleted_at timestamptz` adicionado
- 7 RPCs corrigidas para filtrar `deleted_at IS NULL`
- `segment-eval-worker` filtra `deleted_at IS NULL` no claim
- `refreshSegments` filtra `deleted_at IS NULL`
- Dropdowns de automações no builder filtram deletados + aviso visual quando segmento/form foi deletado

### Jobs stuck — prevenção (Sprint 9)

`analytics.cleanup_stale_segment_eval_jobs(interval)` — RPC criada. Libera jobs claimed há mais que o threshold (default 10min). Chamada a cada ~60s no `segment-eval-worker`. Configurável via `SEGMENT_EVAL_STALE_MINUTES`.

