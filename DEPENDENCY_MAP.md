# KreatorsHub — Dependency Map

> **REGRA R37**: Antes de alterar qualquer item listado aqui, verificar TODOS os readers/writers.
> Atualizar este arquivo sempre que adicionar novo consumidor.
>
> Gerado em: 2026-03-19

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

- **Writers (pipeline atômico apenas):**
  - `workers/analytics/src/handlers/refreshFeatures.ts` — UPSERT features (monetary, frequency, recency, etc.)
  - `workers/analytics/src/handlers/refreshRfm.ts` — UPSERT RFM scores
  - `workers/analytics/src/handlers/refreshReactivation.ts` — UPSERT reactivation fields
  - `workers/analytics/src/handlers/computeClusters.ts` — UPSERT cluster assignments
- **Readers (principais):**
  - Views `analytics.v_segment_base`, `analytics.v_segment_unified`, `analytics.v_segment_leads_base` — source de preview/refresh
  - `supabase/functions/generate-export/index.ts:593` — export enrichment
  - `workers/analytics/src/segment-eval-worker.ts:66` — real-time eval per-contact
  - `workers/analytics/src/handlers/computeClusters.ts:430` — full scan para cluster training
  - `supabase/functions/process-journey/index.ts` — journey condition checks
  - `src/pages/admin/CustomerDetail.tsx` — customer analytics display
- **Regra:** Nunca atualizar diretamente. Apenas via pipeline atômico do analytics-worker.

---

## Filas

### integrations.jobs

- **Producers:**
  - Edge Functions: `eduzz-receiver-webhook`, `nylas-oauth-callback`, `nylas-imap-connect`, `eduzz-oauth-callback` (bulk_enqueue_jobs)
  - Workers: `ingestion/processWebhook.ts:150,165`, `email-sync/supabase.ts:109,157`, `pull-sync` (Sympla, Typeform), `historical-sync` (Sympla, invoice)
  - DB triggers: `trg_enqueue_field_sync` (smart_forms publish)
  - RPCs: `enqueue_job_from_webhook` (idempotent INSERT)
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
- **Consumers:** journeys-worker APENAS — via `journeys.claim_next_job` (SKIP LOCKED)

### journeys.event_processing_jobs

- **Producers:** `trg_enqueue_event_processing` (trigger AFTER INSERT em journey events)
- **Consumers:** journeys-event-worker APENAS — via claim/complete/fail cycle (SKIP LOCKED)

### email.campaign_sends

- **Producers:**
  - `src/pages/admin/CampaignSend.tsx:697` — user triggers send from UI
  - `supabase/functions/process-journey/index.ts:461` — journey email node
  - `workers/journeys/src/handlers/processJourneyNode.ts:1629` — journey worker email node
- **Consumers:** email-campaign-worker APENAS — via `prepareEmailBatch.ts` (claim) + `sendEmailBatch.ts` (process)
- **Readers:** `CampaignSend.tsx`, `CampaignResults.tsx`, `Campaigns.tsx`, `JourneyAnalytics.tsx`

---

## RPCs Críticas

### analytics.refresh_segment_parties(segment_id)

- **Callers:** `process_segment_refresh_queue()` (via fila), `check-journey-entries/index.ts:202` (síncrono), `api.refresh_segment_customers` RPC wrapper
- **Delega para:** `refresh_segment_parties_unified()` — usa `build_unified_where_clause_v2()` (v2 ONLY)
- **Regra:** Preferir enfileirar via `segment_refresh_queue`. Chamada síncrona apenas em hot paths (journey enrollment, fail-closed retry).

### analytics.build_segment_where_clause — DROPPED

- **Status:** Função removida do banco. Substituída por `build_unified_where_clause_v2()`.
- **Referências residuais:** `segmentEvaluator.ts:98` (dead code — fallback v1 nunca atingido). Ver D37.

### api.get_segment_preview

- **Assinatura nova (5 args):** `(p_rules_json, p_rules_json_v2, p_ast_version, p_limit, p_tenant_id)` — usada pelo frontend
- **Assinatura antiga (3 args):** `(p_tenant_id, p_rules_json, p_limit)` — removida de `generate-export` (R38). Verificar se existe em outros lugares.
