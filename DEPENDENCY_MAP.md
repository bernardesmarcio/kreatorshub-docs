# KreatorsHub — Dependency Map

> **REGRA R37**: Antes de alterar qualquer item listado aqui, verificar TODOS os readers/writers.
> Atualizar este arquivo sempre que adicionar novo consumidor.
>
> Gerado em: 2026-03-19 · Última atualização: 2026-03-19

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
- **Status:** DEPRECATED. Nenhum reader deve depender de v1 para lógica de avaliação.

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
