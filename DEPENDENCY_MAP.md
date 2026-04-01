# KreatorsHub — Dependency Map

> **REGRA R37**: Antes de alterar qualquer item listado aqui, verificar TODOS os readers/writers.
> Atualizar este arquivo sempre que adicionar novo consumidor.
>
> Gerado em: 2026-03-19 · Última atualização: 2026-03-20

---

## Queue Routing Map

> **REGRA R33**: Job enfileirado na tabela/worker errado fica pending infinitamente sem erro.
> Antes de enfileirar: (1) confirmar a tabela-fila correta, (2) confirmar que o job_type está na allowlist do worker consumidor.

### Roteamento por tabela-fila

| Tabela-fila | Worker consumidor | Claim RPC | Job types aceitos |
|---|---|---|---|
| `integrations.jobs` | ingestion-worker | `worker.claim_jobs(worker_id, handler_list)` | `commerce:process_batch`, `eduzz:process_invoice`, `eduzz:process_contract`, `eduzz:process_cart_abandonment` |
| `integrations.jobs` | historical-sync-worker | `worker.claim_jobs(worker_id, handler_list)` | `eduzz:import_sales_page`, `eduzz:import_products`, `eduzz:enrich_sales`, `eduzz:enrich_invoice`, `eduzz:get_sales_count`, `sympla:historical_sync`, `typeform:sync_forms`, `typeform:sync_responses`, `typeform:enrich_themes`, `typeform:promote_forms`, `typeform:promote_responses`, `typeform:backfill_settings` |
| `integrations.jobs` | email-sync-worker | `worker.claim_jobs(worker_id, handler_list)` | `nylas:deep_sync`, `nylas:deep_sync_folder`, `nylas:initial_sync`, `cache_attachments_scan`, `cache_attachment_batch`, `cleanup_jobs` |
| `integrations.sync_schedules` | pull-sync-worker | `integrations.claim_sync_schedules(worker_id, providers)` | N/A (schedule-based por provider: `sympla`, `doare`, `typeform`) |
| `analytics.processing_jobs` | analytics-worker APENAS | `analytics.claim_next_jobs(worker_id, batch)` | `refresh_features`, `refresh_rfm`, `refresh_reactivation`, `refresh_segments`, `compute_clusters`, `compute_lookalikes`, `compute_cluster_subgroups`, `sync_customers`, `compute_lookalike_audience`, `link_products`, `project_traits`, `sync_field_definitions` |
| `journeys.processing_jobs` | journeys-worker | `journeys.claim_next_job(worker_id)` | `check_segment_entries`, `check_form_entries`, `check_event_entries`, `process_enrollment`, `link_submission_to_lead` |
| `journeys.processing_jobs` | email-campaign-worker | `journeys.claim_next_job(worker_id)` | `prepare_email_batch`, `send_email_batch`, `retry_soft_bounce` |
| `journeys.event_processing_jobs` | journeys-event-worker APENAS | FOR UPDATE SKIP LOCKED direto | Status-based (`pending`/`retry`), sem job_type — processa todos os eventos disponíveis |
| `analytics.segment_eval_queue` | segment-eval-worker (analytics) | `analytics.claim_next_jobs` | N/A (item-based, não job_type) |
| `analytics.segment_refresh_queue` | DB-only (RPC) | `analytics.process_segment_refresh_queue()` | N/A (item-based) |

### Erros comuns de roteamento

| Erro | Sintoma | Regra violada |
|---|---|---|
| `project_traits` enfileirado em `integrations.jobs` | Job pending infinito — ingestion-worker não reconhece o tipo | R33 |
| Job sem `capability` em `integrations.jobs` | Worker filtra por capability e ignora o job | R31 |
| `commerce:process_batch` enviado para analytics | analytics-worker não tem handler, job pending | R33 |
| Qualquer job novo sem verificar allowlist | Pending sem erro até alguém investigar | R33 |

### Regra para novo job_type

1. Decidir: job coalescente (1 por tenant, ex: `refresh_rfm`) ou per-entity (1 por submission, ex: `project_traits`)
2. Escolher tabela-fila com base no worker que vai consumir
3. Verificar allowlist (`ENABLED_JOB_TYPES` / `ENABLED_HANDLERS` / handler map) do worker
4. Se per-entity: verificar se unique index de coalescing não bloqueia (R32)
5. Adicionar mapeamento nesta tabela

---

## Pipeline Invariants

> **Propósito:** Cada pipeline tem pré-condições que DEVEM ser verdade antes do dado avançar.
> Violação de invariante = dado corrompido propagando silenciosamente.
> Regra geral: **falhar ruidosamente no ponto de entrada, não silenciosamente no ponto de consumo.**

### Form → Traits (`traitProducer.ts`)

| Invariante | Onde verificar | O que acontece se violado |
|---|---|---|
| `party_id IS NOT NULL` na submission | `traitProducer.ts:550` — desestrutura `party_id` da submission | Traits escritos com `party_id` do submission — se null, INSERT falha por FK constraint em `party_trait_*` |
| `field_definitions` existem para o form | `traitProducer.ts:571` — `if (!fd) continue` | Retorna 0 traits silenciosamente (skip individual, não erro) |
| `form_version.status = 'published'` | Trigger `trg_enqueue_field_sync` | `sync_field_definitions` só roda para published — se form não publicou, field_definitions não existem |
| Rodar `sync_field_definitions` ANTES de `project_traits` | Ordem de execução no pipeline | Sem field_definitions, `processSubmission` retorna 0 traits. Guard: `if (!fd) continue` na linha 571 |

### Form → Journey Enrollment (`checkFormEntries`)

| Invariante | Onde verificar | O que acontece se violado |
|---|---|---|
| `party_id IS NOT NULL` na enrollment | `checkFormEntries.ts:148-156` — 3-layer resolve (v7.13): (1) `vw_parties_unified` lookup, (2) `crm.upsert_party_person` fallback, (3) skip gracioso se ainda null | ✅ D40 resolvido (v7.13). Guard presente — enrollment com `party_id = null` impossível; `create_opportunity` node também tem fallback próprio. |
| `email IS NOT NULL` | `checkFormEntries.ts:129` — `if (!email) { skipped++ }` | ✅ Guard presente — submissions sem email são ignoradas |
| Journey `status` ativa | `checkFormEntries.ts:37` — `if journey not active → skip` | ✅ Guard presente — retorna `{ skipped: true, reason: 'journey_not_active' }` |
| Submission não duplicada | `checkFormEntries.ts:135` — `enrolledSubmissionIds.has()` | ✅ Guard presente — skip se já enrolled |

### Form → Journey Enrollment (`trigger-form-journey` Edge Function)

| Invariante | Onde verificar | O que acontece se violado |
|---|---|---|
| `party_id` resolvido via `upsert_party_person` | `trigger-form-journey/index.ts:131-139` — RPC upsert | Cria party se não existe (melhor que checkFormEntries). `partyId` pode ser `null` se RPC falha, mas enrollment é criado na linha 290 |
| `respondentEmail` presente | `trigger-form-journey/index.ts:282-290` — INSERT enrollment | Enrollment com `customer_email = null` se sem email |

### Ingestion → Analytics (`process_transactions_batch` RPC)

| Invariante | Onde verificar | O que acontece se violado |
|---|---|---|
| `_version = 1` | RPC linha ~27 — `IF v_version != 1 THEN` | ✅ Guard presente — erro individual, batch continua. Registrado em `v_errors` |
| `email` presente e não-vazio | RPC linha ~37 — `IF NULLIF(TRIM(email), '') IS NULL` | ✅ Guard presente — `v_skipped++`, CONTINUE (não consegue criar party) |
| `product_name` presente | RPC linha ~42 — `IF v_product_name IS NULL` | ✅ Guard presente — erro registrado em `integrations.job_errors`, CONTINUE |
| `external_transaction_id` presente | ON CONFLICT clause na RPC | Idempotência via `(tenant_id, source_name, external_transaction_id)` — sem ID, duplicatas possíveis |
| Status normalizado válido | CHECK constraint em `commerce.transactions.status` | ✅ Guard presente (R31) — INSERT rejeitado se status inválido |
| CPF ≤ 14 chars | RPC linha ~51 — `IF length(v_cpf) > 14 THEN NULL` | ✅ Guard presente — sanitiza documentos não-brasileiros |

### Campaign Send (R38 fail-closed)

| Invariante | Onde verificar | O que acontece se violado |
|---|---|---|
| `segment_parties` não vazio se segmento tem regras | `CampaignSend.tsx:607` — `getSegmentRecipientsFailClosed()` | **R38:** membership vazia = refresh pendente, não "enviar para todos". Fail-closed retorna 0 recipients |
| Blast radius ≤ 1.1× segment size | Guard de blast radius no frontend | Envio abortado se ratio excede threshold |
| Segmento não deletado (`deleted_at IS NULL`) | Query de seleção de segmentos | Segmento fantasma retorna 0 members via `segment_parties` |

### Journey Node Execution

| Invariante | Onde verificar | O que acontece se violado |
|---|---|---|
| `enrollment.party_id IS NOT NULL` | Nodes que dependem de party | ✅ D40 resolvido (v7.13). `checkFormEntries` tem 3-layer resolve; `create_opportunity` tem fallback próprio com `upsert_party_person` + skip gracioso. |
| `send_email` skip retorna `success: false` | `process-journey` Edge Function | **R39:** skip com `success: true` perdia envio silenciosamente. Corrigido: `success: false` + `waitUntil` para retry |
| `enrollment.status = 'active'` | Claim do job no worker | Enrollment completed/failed não deveria gerar novos jobs |

### Checklist para novo pipeline

Antes de implementar qualquer novo pipeline, responder:

1. Qual a **pré-condição mínima** do dado na entrada? (ex: `party_id NOT NULL`)
2. O que acontece se a pré-condição for violada? (erro ruidoso ou falha silenciosa?)
3. O guard está no **ponto de entrada** ou no ponto de consumo?
4. Existe **CHECK constraint** no banco para os valores esperados?
5. O pipeline depende de **outro pipeline ter rodado antes**? (ex: `field_definitions` antes de `project_traits`)

---

## Segmentação

### analytics.segments.rules_json (v1) — DEPRECATED

- **Writers:** nenhum writer ativo. Frontend não grava mais v1 desde Sprint 12 (D33). 40 segmentos existentes ainda têm v1 populado (dual-write histórico).
- **Readers:**
  - `workers/analytics/src/handlers/refreshSegments.ts:64,73` — SELECT para passagem ao evaluator (fallback v1 nunca atingido — todos os rule-based têm v2)
  - `workers/analytics/src/segmentation/segmentEvaluator.ts:93-100` — fallback legado: chama `build_segment_where_clause` (DROPADA — dead code)
  - `supabase/functions/generate-export/index.ts:711` — SELECT para check de has-rules no guard fail-closed
  - `src/pages/admin/CampaignSend.tsx:612` — check de has-rules no guard fail-closed
  - `src/lib/segmentStorage.ts:687,844` — passagem para `loadSegmentRules()` (display only)
- **Status:** DEPRECATED. Nenhum reader depende de v1 para avaliação. Dead code confirmado (D37, 2026-03-20): `segmentEvaluator.ts:86-100` (chama `build_segment_where_clause` que foi dropada do banco — executar causaria erro SQL), `scripts/runEquivalenceGate.ts` (inteiro), e testes em `equivalence.test.ts`.

### analytics.segments.rules_json_v2

- **Writers:**
  - `src/lib/segmentStorage.ts:157` — `createSegment()` INSERT
  - `src/lib/segmentStorage.ts:207` — `updateSegment()` UPDATE
  - `src/lib/segmentation/segmentService.ts:155,220` — create/update alternativo
  - `workers/analytics/src/segmentation/scripts/applyMigration.ts:182` — batch migration script
- **Readers:**
  - `workers/analytics/src/handlers/refreshSegments.ts:64,73` — SELECT para compilação v2
  - `workers/analytics/src/segmentation/segmentEvaluator.ts:57,65` — compilação v2 → WHERE clause
  - `supabase/functions/generate-export/index.ts:711,812` — has-rules check (fail-closed)
  - `src/pages/admin/CampaignSend.tsx:611` — has-rules check (fail-closed)
  - `src/lib/segmentStorage.ts:291,356` — passagem para preview RPCs
  - `src/lib/segmentation/astV2ToBuilder.ts:432` — hydrate builder UI
  - `src/components/segments/SegmentBuilderV2.tsx:46` — preview RPC
  - `src/pages/admin/SegmentationList.tsx:258` — duplicate segment
  - `src/pages/admin/SegmentationBuilder.tsx:103` — load segment for editing
- **Regra:** Source of truth para regras de segmentação rule-based. Avaliação feita por `build_unified_where_clause_v2()`.

### analytics.segment_parties

- **Writers (SQL):**
  - `refresh_segment_parties_unified()` — INSERT/DELETE diff após avaliação v2 (chamada via `process_segment_refresh_queue`)
  - `apply_segment_membership_diff()` — diff incremental (triggers de transação)
  - `sync_cluster_segment_customers()` — clusters/subgroups
- **Writers (Node.js):**
  - `workers/analytics/src/segmentation/segmentEvaluator.ts:112` — `refreshSegmentV2()` INSERT/DELETE dentro de transaction
  - `workers/analytics/src/handlers/computeClusters.ts` — cluster party_ids
  - `workers/analytics/src/handlers/computeLookalikeAudience.ts` — lookalike party_ids
- **Readers:**
  - `src/lib/segmentStorage.ts:555` — `getSegmentPartiesForCampaign()` → envio de campanha
  - `src/pages/admin/CampaignSend.tsx` — via `getSegmentRecipientsFailClosed()` (R38)
  - `supabase/functions/generate-export/index.ts:738` — export de segmento
  - `supabase/functions/check-journey-entries/index.ts:213` — enrollment de journey
  - Views: `analytics.v_segment_unified` (preview)
- **Regra R38:** ÚNICA fonte para membership de segmento. Todo código que precisa de lista de membros/recipients DEVE ler daqui. NUNCA re-avaliar regras no momento do consumo.

### analytics.segment_refresh_queue

- **Producers:**
  - `trg_queue_segment_refresh` — trigger BEFORE INSERT/UPDATE em `analytics.segments` (detecta mudança em `rules_json` OU `rules_json_v2`)
  - `supabase/functions/generate-export/index.ts:821` — fail-closed: enfileira quando segment_parties vazio + has rules
- **Consumers:**
  - `workers/journeys/src/index.ts:157,257` — `processSegmentRefreshQueue(10)` a cada 5s
  - → chama `analytics.process_segment_refresh_queue()` (SKIP LOCKED)
  - → chama `analytics.refresh_segment_parties()` → `refresh_segment_parties_unified()` (v2)
- **Regra:** Único caminho assíncrono para refresh de segment_parties. Trigger nunca chama refresh diretamente.

---

## Contact State

### analytics.contact_state

- **Writers — Pipeline atômico (analytics-worker):**
  - `workers/analytics/src/handlers/refreshFeatures.ts:156` — UPSERT features (monetary, frequency, recency, total_purchases, etc.)
  - `workers/analytics/src/handlers/refreshRfm.ts:237` — UPSERT RFM scores (rfm_segment, rfm_recency, rfm_frequency, rfm_monetary)
  - `workers/analytics/src/handlers/refreshReactivation.ts:246,337` — UPSERT reactivation fields
  - `workers/analytics/src/handlers/computeClusters.ts` — UPSERT cluster assignments
  - `workers/analytics/src/handlers/linkProducts.ts:180` — UPDATE `bought_product_ids` (recalc from transactions)
- **Writers — Real-time array mutations (historical-sync-worker):**
  - `workers/historical-sync/src/handlers/webhook/analytics-events.ts:44,57` — `tag_ids` array append/remove
  - `workers/historical-sync/src/handlers/webhook/analytics-events.ts:120` — `responded_form_ids` array append
  - `workers/historical-sync/src/handlers/webhook/analytics-events.ts:185` — `import_ids` array append
- **Writers — Purchase ingestion pipeline:**
  - `workers/ingestion/src/handlers/shared/analytics-pipeline.ts:42` — chama `analytics.process_purchase_analytics()` RPC (upsert interno)
- **Writers — segment_ids (SQL triggers):**
  - `apply_segment_membership_diff()` — array_union/array_except em `segment_ids` (múltiplas versões nas migrations)
  - `trg_cleanup_segment_ids_on_delete` — `array_remove(segment_ids, segment.id)` ao soft-delete segmento
- **Readers (principais):**
  - Views `analytics.v_segment_base`, `analytics.v_segment_unified`, `analytics.v_segment_leads_base` — source de preview/refresh
  - `supabase/functions/generate-export/index.ts:593` — export enrichment
  - `workers/analytics/src/segment-eval-worker.ts:66` — real-time eval per-contact
  - `workers/analytics/src/handlers/computeClusters.ts:441` — full scan para cluster training
  - `workers/analytics/src/handlers/computeLookalikeAudience.ts:327` — lookalike training
  - `supabase/functions/process-journey/index.ts` — journey condition checks
  - `src/lib/analyticsStorage.ts:145,331` — frontend analytics display
- **Regra:** Nunca atualizar diretamente fora dos paths listados. Pipeline atômico via analytics-worker.

---

## Email Pipeline

### email.campaign_sends — Metadados do envio (1 row por disparo)

- **Writers:**
  - `src/pages/admin/CampaignSend.tsx:697` — INSERT: user triggers send from UI
  - `supabase/functions/process-journey/index.ts:461` — INSERT: journey email node
  - `workers/journeys/src/handlers/processJourneyNode.ts:1629` — INSERT: journey worker email node
  - `workers/email-campaign/src/handlers/prepareEmailBatch.ts:24,47,58,85` — UPDATE status (sending/failed/completed)
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:630,658,691` — UPDATE status + counts
- **Readers:**
  - `src/pages/admin/CampaignSend.tsx:426`, `CampaignResults.tsx:143`, `Campaigns.tsx:179`, `JourneyAnalytics.tsx:352`
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:119` — load send for processing
- **Colunas de rastreabilidade (Guard 1):** `recipient_source` (segment/manual/all/test), `segment_ids` (uuid[]), `segment_parties_snapshot` (int), `recipient_count_at_send` (int). Adicionadas Sprint 12 para auditoria de blast radius.
- **Regra:** NÃO contém lista de recipients. É apenas metadados (template, subject, status, counts). A lista real está em `campaign_recipients`.

### email.campaign_recipients — Lista congelada de envio (R38)

- **Writers:**
  - `src/pages/admin/CampaignSend.tsx:732` — INSERT bulk: congela lista de recipients ao disparar campanha
  - `supabase/functions/process-journey/index.ts:477` — INSERT: single recipient (journey node)
  - `workers/journeys/src/handlers/processJourneyNode.ts:1652` — INSERT: single recipient (journey worker)
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:376,393,408` — UPDATE status → `skipped` (suppressed/unsubscribed)
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:503,520,561` — UPDATE status → `sent`/`pending`/`failed`
  - `workers/email-campaign/src/handlers/retrySoftBounce.ts:126,301,333` — UPDATE status (bounce retry)
  - `supabase/functions/integrations-worker/handlers/sendgrid.ts:82,99` — UPDATE tracking fields via `sendgrid_message_id`
  - `supabase/functions/unsubscribe/index.ts:280` — UPDATE status on unsubscribe
- **Readers:**
  - `workers/email-campaign/src/handlers/prepareEmailBatch.ts:76` — COUNT pending recipients
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:616,650,677,684` — COUNT by status (completion checks)
  - `workers/email-campaign/src/handlers/retrySoftBounce.ts:90` — SELECT single recipient for retry
  - `supabase/functions/webhooks-sendgrid/index.ts:131` — lookup `tenant_id` by `sendgrid_message_id`
  - `src/pages/admin/CampaignSend.tsx:502` — SELECT failed/bounced for display
  - `src/pages/admin/CampaignResults.tsx:186` — SELECT all (non-skipped) for results view
  - `workers/journeys/src/handlers/processJourneyNode.ts:1615` — SELECT dedup guard (already sent?)
- **Regra R38:** Lista CONGELADA de recipients. O email-campaign-worker é blind sender — só processa o que está aqui. Nunca re-avaliar segmento no momento do envio. Se 0 rows para um `campaign_send_id`, o envio não acontece (fail-closed by design).

### email.suppression_list — Bounce/spam suppression

- **Writers:**
  - `supabase/functions/integrations-worker/handlers/sendgrid.ts:173-201` — INSERT/UPDATE: soft_bounce, hard_bounce, spam, blocked
  - `supabase/functions/eduzz-processor-worker/handlers/sendgrid.ts:173-201` — (duplicate handler)
- **Readers:**
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:214` — SELECT suppressed emails (pre-send gate)
  - `workers/email-campaign/src/handlers/retrySoftBounce.ts:115` — SELECT before retry
  - `workers/journeys/src/handlers/processJourneyNode.ts:1528` — SELECT before journey send
- **Regra:** Check obrigatório antes de qualquer envio. Tripla camada: suppression + global unsub + group unsub.

### email.unsubscribes — Global/group unsubscribes

- **Writers:**
  - `supabase/functions/integrations-worker/handlers/sendgrid.ts:217` — UPSERT on SendGrid unsubscribe/spamreport event
  - `supabase/functions/unsubscribe/index.ts:230` — UPSERT global unsubscribe (one-click link)
  - `supabase/functions/save-preferences/index.ts:246,286,314,340` — UPSERT/UPDATE global + group unsub/resub
- **Readers:**
  - `src/pages/admin/UnsubscribeGroupDetail.tsx:87` — SELECT for group detail
  - `src/pages/admin/UnsubscribeGroups.tsx:87` — SELECT for group counts

---

## Filas

### integrations.jobs

- **Producers (Edge Functions):**
  - `eduzz-receiver-webhook/index.ts:279` — via `enqueueWebhookJob()` RPC
  - `webhooks-nylas/index.ts:250` — via `enqueueWebhookJob()` RPC
  - `webhooks-sendgrid/index.ts:313` — via `enqueueWebhookJob()` RPC
  - `nylas-oauth-callback/index.ts:112` — INSERT direto (initial sync)
  - `nylas-imap-connect/index.ts:177` — INSERT direto (IMAP sync)
  - `nylas-sync/handlers/nylas.ts:352,563` — INSERT (deep_sync + continuation)
  - `eduzz-oauth-callback/index.ts:362` — via `bulk_enqueue_jobs` RPC (historical import)
- **Producers (Workers):**
  - `workers/ingestion/src/handlers/eduzz/processWebhook.ts:150,165` — `commerce:process_batch`, `eduzz:enrich_invoice`
  - `workers/email-sync/src/supabase.ts:109,157` — email sync jobs
  - `workers/pull-sync/src/handlers/sympla/index.ts:279` — Sympla batch
  - `workers/pull-sync/src/handlers/typeform/index.ts:84` — Typeform responses
  - `workers/historical-sync/src/handlers/sympla/historicalSync.ts:238` — Sympla historical
  - `workers/historical-sync/src/handlers/webhook/processInvoice.ts:113` — invoice processing
- **Producers (DB triggers):**
  - `trg_enqueue_field_sync` — smart_forms publish → `forms:sync_field_definitions`
- **Consumers:** ingestion-worker, email-sync-worker, historical-sync-worker, pull-sync-worker — via `claim_next_jobs` (SKIP LOCKED)
- **Job types principais:** `commerce:process_batch`, `eduzz:enrich_invoice`, `nylas:deep_sync`, `sympla:sync_responses`, `typeform:sync_responses`

### analytics.processing_jobs

- **Producers:**
  - RPCs: `enqueue_job_internal` (INSERT com coalescing window)
  - DB triggers: `trg_enqueue_trait_projection` (trait changes), `trg_enqueue_field_sync` (form publish)
- **Consumers:** analytics-worker APENAS — via `public.claim_next_jobs` (SKIP LOCKED)
- **Job types:** `project_traits`, `sync_field_definitions`, `refresh_features`, `refresh_rfm`, `link_products`

### journeys.processing_jobs

- **Producers:**
  - `workers/journeys-event/src/handlers/processEventJob.ts:221` — `process_enrollment`
  - `workers/journeys-event/src/handlers/processBackfillJob.ts:332` — backfill enrollments
  - `supabase/functions/check-journey-entries/index.ts:364` — via `journeys.enqueue_job` RPC
  - `supabase/functions/trigger-form-journey/index.ts:319` — via `journeys.enqueue_job` RPC
- **Consumers:** journeys-worker APENAS — via `journeys.claim_next_job` (SKIP LOCKED)

### journeys.event_processing_jobs

- **Producers:** `trg_enqueue_event_processing` (trigger AFTER INSERT em journey events)
- **Consumers:** journeys-event-worker APENAS — via claim/complete/fail cycle (SKIP LOCKED)

---

## Journeys

### journeys.journey_enrollments

- **Writers:**
  - `workers/journeys/src/handlers/checkEventEntries.ts:196` — INSERT: event-triggered enrollment
  - `workers/journeys/src/handlers/checkSegmentEntries.ts:224` — INSERT: segment-triggered enrollment
  - `workers/journeys/src/handlers/checkFormEntries.ts:166` — INSERT: form-triggered enrollment
  - `workers/journeys-event/src/handlers/processEventJob.ts:196` — INSERT: event worker enrollment
  - `workers/journeys-event/src/handlers/processBackfillJob.ts:301` — INSERT: backfill enrollment
  - `supabase/functions/check-journey-entries/index.ts:309` — INSERT: edge function enrollment
  - `supabase/functions/trigger-form-journey/index.ts:238,253,269,285` — UPSERT: form-triggered edge function
  - `workers/journeys/src/handlers/processJourneyNode.ts:248,275,327,1717,1739` — UPDATE: status/state during node processing
  - `supabase/functions/process-journey/index.ts:107,196,214,227,239` — UPDATE: node execution updates
- **Readers:**
  - `workers/journeys/src/handlers/checkEventEntries.ts:97,239` — SELECT dedup (existing enrollment?)
  - `workers/journeys/src/handlers/checkSegmentEntries.ts:96,187` — SELECT dedup
  - `workers/journeys/src/handlers/checkFormEntries.ts:81` — SELECT dedup
  - `workers/journeys/src/handlers/processJourneyNode.ts:57,141,1707` — SELECT by id, status, context
  - `supabase/functions/check-journey-entries/index.ts:116` — SELECT dedup
  - `src/pages/admin/JourneyAnalytics.tsx:246-288` — SELECT for analytics display
  - `src/pages/admin/Journeys.tsx:106` — COUNT aggregate
- **Regra D18:** INSERT enrollment + INSERT `process_enrollment` job DEVEM ser atômicos (`sql.begin`). Sweeper pg_cron para enrollments órfãos pendente.

---

## Commerce

### commerce.transactions

- **Writers:**
  - `commerce.upsert_provider_transaction()` RPC — canonical upsert (ON CONFLICT UPDATE) from any provider
  - `workers/ingestion/src/handlers/eduzz/processInvoice.ts:119` — INSERT ON CONFLICT (Eduzz ingestion)
  - `workers/historical-sync/src/supabase.ts:654` — INSERT ON CONFLICT DO NOTHING (historical sync)
  - `workers/analytics/src/handlers/linkProducts.ts:160` — UPDATE `product_id` (product linking)
- **Readers:**
  - `src/lib/partyStorage.ts:371` — party transaction history
  - `src/pages/admin/Transactions.tsx:290,490,674` — transactions list, CSV export, detail
  - `src/pages/admin/ClientDetail.tsx:143,249` — customer detail
  - `src/pages/admin/ProductCustomers.tsx:350` — product customers
  - `src/components/segments/useSegmentOptions.ts:21` — segment product options
  - `workers/analytics/src/handlers/linkProducts.ts:97` — product linking pipeline
  - `supabase/functions/eduzz-processor-worker/handlers/eduzz.ts:224` — lookup by `external_transaction_id`
  - SQL views/functions: `build_segment_where_clause_v2`, `v_segment_base` — segment evaluation
- **Regra:** Idempotência via `(tenant_id, source_name, external_transaction_id)`. ON CONFLICT DO UPDATE apenas para transições válidas de status.

### commerce.asaas_webhook_events — Raw storage Asaas

- **Writers:**
  - `supabase/functions/asaas-receiver-webhook/index.ts` — INSERT via `public.upsert_asaas_webhook_event()` RPC (contorna limitação PostgREST + partial unique index)
  - `public.mark_asaas_webhook_processed()` — UPDATE `processed=true, job_id` após pipeline
- **Readers:**
  - `supabase/functions/asaas-receiver-webhook/index.ts` — dedup check via `event_id` (dentro da RPC)
- **TTL:** pg_cron diário — DELETE WHERE `processed = true AND created_at < now() - 90 days`
- **Regra:** Unique index parcial `(tenant_id, event_id) WHERE event_id IS NOT NULL`. Pipeline E2E validado com PIX real (2026-03-20).

### commerce.asaas_config — Configuração Asaas por tenant

- **Writers:**
  - `src/lib/checkoutStorage.ts:311` — `upsertAsaasConfig()` UPSERT via frontend admin
- **Readers:**
  - `src/lib/checkoutStorage.ts:291` — `getAsaasConfig()` SELECT para tela de configuração
  - `supabase/functions/asaas-receiver-webhook/index.ts:68` — SELECT `webhook_token` para validação do header `asaas-access-token`
- **Triggers:** `audit_asaas_config` (INSERT/UPDATE/DELETE → `audit.log_change()`)
- **Regra:** `api_key_encrypted` usa criptografia. `webhook_token` validado no receiver. `commerce.sync_checkout_order_from_transaction` trigger em `commerce.transactions` sincroniza status do `checkout_orders` quando transação muda.

---

## Forms Pipeline

### forms.form_submissions (+ smart_forms.form_submissions)

- **Writers:**
  - `src/lib/formStorage.ts:981` — INSERT: `submitResponse()` (public form)
  - `src/lib/smartFormStorage.ts:629,880,1046` — INSERT/UPDATE: smart form submit lifecycle
  - `workers/historical-sync/src/handlers/typeform/promoteResponses.ts:277` — INSERT bulk (Typeform historical)
- **Readers:**
  - `src/lib/formStorage.ts:102,233,1068,1142,1170,1252,1352,1621,2497` — counts, lists, detail, analytics
  - `src/lib/smartFormStorage.ts:249,714,879,960` — smart form reads
  - `src/lib/partyStorage.ts:326`, `src/lib/leadStorage.ts:947` — party/lead submissions
  - `workers/journeys/src/handlers/checkFormEntries.ts:106,115` — journey enrollment check
  - `workers/analytics/src/segmentation/scripts/backfillFormTraits.ts:127` — trait backfill
- **Regra:** Jobs devem incluir `capability='process_submission'`. Status `submitted` + `party_id NOT NULL` = trigger enfileira `project_traits`.

### forms.form_answers (+ smart_forms.form_answers)

- **Writers:**
  - `src/lib/formStorage.ts:1045,1224` — INSERT bulk / UPSERT single
  - `src/lib/smartFormStorage.ts:932,1097` — UPSERT single / INSERT bulk
  - `workers/historical-sync/src/handlers/typeform/promoteResponses.ts:340` — INSERT bulk (Typeform)
- **Readers:**
  - `src/lib/formStorage.ts:1071,1271,1403` — joined in submission queries, extract email, journey payload
  - `src/lib/smartFormStorage.ts:691,979,1097` — smart form reads
  - `workers/analytics/src/segmentation/traitProducer.ts:559` — trait projection SQL
  - Migration triggers: `link_public_submission_to_lead` — extract email from answers

---

## Traits

### crm.party_trait_* (text, number, timestamp, boolean, multi_value)

- **Writers:** `workers/analytics/src/segmentation/traitProducer.ts:637-702` APENAS — INSERT ON CONFLICT UPDATE para scalar types, DELETE+INSERT para multi_value. Triggered por `project_traits` jobs from form submissions.
- **Readers:**
  - `workers/analytics/src/segmentation/resolvers.ts:524` — gera `EXISTS` subquery SQL para segment evaluation
  - `workers/analytics/src/segmentation/fieldCatalog.ts:109` — dynamic field catalog
  - Leitura indireta: trait tables são acessadas via SQL dinâmico em segment where-clauses, nunca via `.from()` direto
- **Regra R30:** Traits NUNCA materializados em `contact_state`. Avaliados via `EXISTS` subquery direta.

### crm.field_definitions

- **Writers:**
  - `workers/analytics/src/segmentation/traitProducer.ts:280` — INSERT: syncs field definitions from form blocks on publish
  - DB trigger: `trg_enqueue_field_sync` → `sync_field_definitions` job → traitProducer INSERT
- **Readers:**
  - `workers/analytics/src/segmentation/traitProducer.ts:214` — reads to project traits (match block→field)
  - `workers/analytics/src/segmentation/fieldCatalog.ts:147` — loads dynamic fields for segmentation catalog
- **Regra:** Pré-requisito para projeção de traits. Sem `field_definition` mapeada, `processSubmission` retorna 0 traits silenciosamente.

---

## Integrations

### integrations.webhook_events — Raw storage

- **Writers:**
  - `integrations.receive_webhook()` RPC — INSERT ON CONFLICT (chamada por todos os receivers)
  - Callers: `eduzz-receiver-webhook`, `webhooks-nylas`, `webhooks-sendgrid` — via `receiveWebhook()` shared helper
  - `workers/historical-sync/src/supabase.ts:244` — UPDATE marks as processed
- **Readers:**
  - `workers/historical-sync/src/supabase.ts:213` — reads pending events for processing
  - `src/lib/eduzzStorage.ts:162,199,360,374` — frontend event list, detail, stats
- **Regra:** Raw imutável. Nunca deletar. Nunca usar para lógica downstream — apenas debug/reprocessing.

### integrations.accounts

- **Writers:**
  - `src/lib/eduzzStorage.ts:49,81` — INSERT/UPDATE Eduzz account
  - `src/lib/symplaStorage.ts:36,66,87` — INSERT/UPDATE Sympla account
  - `supabase/functions/eduzz-oauth-callback/index.ts:141,227` — INSERT/UPDATE on OAuth
  - `supabase/functions/typeform-oauth-callback/index.ts:302,314` — INSERT/UPDATE Typeform
  - Workers: `pull-sync` (Sympla, Typeform, Doare), `historical-sync` — UPDATE tokens/sync state
- **Readers:**
  - `src/lib/eduzzStorage.ts:24,48,100`, `src/lib/symplaStorage.ts:13,34,101` — frontend config
  - `src/components/settings/IntegrationHub.tsx:96` — all integrations for UI
  - Edge Functions: `eduzz-receiver-webhook:116`, `nylas-*`, `webhooks-sendgrid:167` — read credentials/status
  - Workers: `pull-sync`, `historical-sync` — read tokens/config
- **Regra R21/R34:** Único local para config de integração. Nunca criar tabela `{provider}.integrations`.

---

## RPCs Críticas

### analytics.refresh_segment_parties(segment_id)

- **Callers:** `process_segment_refresh_queue()` (via fila), `check-journey-entries/index.ts:202` (síncrono), `api.refresh_segment_customers` RPC wrapper
- **Delega para:** `refresh_segment_parties_unified()` — usa `build_unified_where_clause_v2()` (v2 ONLY)
- **Regra:** Preferir enfileirar via `segment_refresh_queue`. Chamada síncrona apenas em hot paths (journey enrollment, fail-closed retry).

### analytics.build_segment_where_clause — DROPPED

- **Status:** Função removida do banco. Substituída por `build_unified_where_clause_v2()`.
- **Referências residuais:** `segmentEvaluator.ts:93-100` (dead code — fallback v1 nunca atingido). Ver D37.

### api.get_segment_preview

- **Assinatura nova (5 args):** `(p_rules_json, p_rules_json_v2, p_ast_version, p_limit, p_tenant_id)` — usada pelo frontend
- **Assinatura antiga (3 args):** `(p_tenant_id, p_rules_json, p_limit)` — removida de `generate-export` (R38). Verificar se existe em outros lugares.

---

## Audit

### audit.trail — Log de auditoria imutável

- **Writers (triggers ONLY — nunca INSERT direto):**
  - `analytics.segments` — trigger `audit_segments` (INSERT/UPDATE/DELETE)
  - `commerce.asaas_config` — trigger `audit_asaas_config` (INSERT/UPDATE/DELETE)
  - `core.system_admins` — trigger `audit_system_admins` (INSERT/UPDATE/DELETE)
  - `core.tenant_users` — trigger `audit_tenant_users` (INSERT/UPDATE/DELETE)
  - `core.tenants` — trigger `audit_tenants` (INSERT/UPDATE/DELETE)
  - `email.tenant_config` — trigger `audit_tenant_config` (INSERT/UPDATE/DELETE)
  - `forms.forms` — trigger `audit_forms` (INSERT/UPDATE/DELETE)
  - `integrations.accounts` — trigger `audit_integrations_accounts` (INSERT/UPDATE/DELETE)
  - `journeys.journeys` — trigger `audit_journeys` (INSERT/UPDATE/DELETE)
  - `smart_forms.forms` — trigger `audit_smart_forms` (INSERT/UPDATE/DELETE)
  - `whatsapp.instances` — trigger `audit_whatsapp_instances` (INSERT/UPDATE/DELETE)
  - Todos delegam para `audit.log_change()` — captura `user_id`, `user_email`, `role_name`, `schema_name`, `table_name`, `operation`, `record_id`, `tenant_id`, `old_data`, `new_data`, `changed_fields`, `ip_address`, `user_agent`, `request_id`, `transaction_id`
- **Readers:** Nenhum reader frontend implementado. Acesso via SQL direto (admin).
- **Funções:**
  - `audit.log_change()` — trigger function principal
  - `audit.log_change_simple()` — versão simplificada
  - `audit.get_record_history(record_id)` — histórico de um registro
  - `audit.get_tenant_changes(tenant_id)` — mudanças de um tenant
  - `audit.get_user_changes(user_id)` — mudanças de um usuário
  - `audit.get_stats()` — estatísticas do audit trail
  - `audit.export_trail()` — exportação
- **Regra:** Append-only — nunca UPDATE ou DELETE. RLS habilitado. 33 triggers total (11 tabelas × 3 operações). Gap: `checkout_offers`, `checkout_orders`, `opportunities`, `eduzz.integrations` sem audit.

---

## Code Comments Standard

> **Propósito:** Comentários existem nos **pontos de junção** — onde um erro silencioso propaga para outro sistema.
> Não comentar "o que o código faz" (isso é legível). Comentar "por que essa decisão, e o que quebra se mudar".

### Estado atual (diagnóstico 2026-03-23)

- **R38** referenciada em 7 lugares (CampaignSend, check-journey-entries, generate-export, checkSegmentEntries) — bom padrão
- **R28** referenciada em 1 lugar (traitProducer) — isolada
- **D-series:** zero referências no código — workarounds não estão marcados
- **TODOs:** zero nos workers e libs — limpo, mas pontos críticos sem anotação

### Onde comentar (obrigatório)

| Ponto de junção | Por quê | Exemplo |
|---|---|---|
| **Enqueue de job** | Job na fila errada = pending infinito (R33) | `// Queue: analytics.processing_jobs → analytics-worker. R33: NÃO usar integrations.jobs` |
| **ON CONFLICT** | Idempotência errada = duplicata ou perda de update | `// Idempotência: (tenant_id, source_name, external_transaction_id). ON CONFLICT DO UPDATE apenas paid→refunded` |
| **Guard / fail-closed** | Remoção acidental do guard = blast radius (R38) | `// R38 FAIL-CLOSED: segment_parties vazio + has rules = refresh pendente, NUNCA "enviar para todos"` |
| **Pipeline pré-condição** | Dado incompleto propaga silenciosamente | `// INVARIANTE: party_id NOT NULL. D40 resolvido (v7.13): 3-layer resolve + fallback em create_opportunity` |
| **CHECK constraint enum** | Novo valor sem migration = INSERT rejeitado (R31) | `// R31: valores devem estar na CHECK constraint do banco. Novo valor → migration ANTES do deploy` |
| **Status transitions** | Transição inválida corrompe state | `// Status: pending→paid OK, paid→pending NUNCA. Ver process_transactions_batch` |
| **contact_state write** | Múltiplos writers = divergência | `// WRITER: este é um dos N writers de contact_state. Ver DEPENDENCY_MAP.md > contact_state` |
| **Array mutation (GIN)** | Array vazio = segmento silenciosamente errado (R17) | `// R17: array_append/array_remove. Validar que array não fica [] — segmento retornaria 0 members` |

### Template de header para funções de pipeline

```typescript
/**
 * [Nome da função] — [O que faz em uma linha]
 *
 * Pipeline: [origem] → [trigger/enqueue] → [tabela-fila] → [worker] → esta função
 *
 * Invariantes de entrada:
 *   - [campo] NOT NULL (sem isso → [consequência])
 *   - [pré-condição] (sem isso → [consequência])
 *
 * Rules: [R-series relevantes]
 * Debts: [D-series se houver workaround ativo]
 * Ref: ARCHITECTURE.md seção [N], DEPENDENCY_MAP.md > [tabela]
 */
```

### Template de comentário inline para pontos críticos

```typescript
// R[N]: [descrição curta da regra]
// Ref: DEPENDENCY_MAP.md > [seção] | ARCHITECTURE.md seção [N]

// D[N] WORKAROUND: [descrição do workaround temporário]
```

### Anti-patterns de comentários

| ❌ Não fazer | ✅ Fazer |
|---|---|
| `// Enfileira o job` | `// Queue: analytics.processing_jobs → analytics-worker (R33)` |
| `// Verifica se tem party_id` | `// INVARIANTE D40 (resolvido v7.13): 3-layer resolve garante party_id NOT NULL` |
| `// Atualiza contact_state` | `// WRITER contact_state.tag_ids — ver DEPENDENCY_MAP.md para lista completa de writers` |
| `// ON CONFLICT DO NOTHING` | `// Idempotência: (tenant_id, source_name, external_id). Duplicata = skip (camada 2)` |
| Comentário sem referência a regra | Sempre incluir R[N], D[N] ou seção do ARCHITECTURE.md |

### Regra de manutenção

Ao resolver uma dívida D[N]:
1. Buscar `D[N]` no código: `grep -rn "D[N]" workers/ src/ supabase/`
2. Remover comentários de workaround marcados com `// D[N] WORKAROUND`
3. Manter comentários de regra R[N] (são permanentes)
