# KreatorsHub — Dependency Map

> **REGRA R37**: Antes de alterar qualquer item listado aqui, verificar TODOS os readers/writers.
> Atualizar este arquivo sempre que adicionar novo consumidor.
>
> Gerado em: 2026-03-19 · Última atualização: 2026-11-05 (dados sociais fase 2: edge fn `social-ingest` passa a writer de `social_media_posts`/`_post_daily`/`social_audience_snapshot`/`social_ingest_runs`/breakdown de `social_account_daily`/`social_accounts.profile`; fila `social:ingest` (migration `20261105130000`); gancho na `social-accounts-sync`; `_shared/adsBreakdowns.ts`/`adsWindows.ts`; escrito, NÃO aplicado/deployado); 2026-11-05 (Ads Intelligence: schema `ads`, edge `ads-sync`, RPCs `api.ads_*`, Insights Center); 2026-11-04 (dados sociais fase 1: `integrations.social_media_posts`/`_post_daily`/`social_audience_snapshot`/`social_ingest_runs`, `ads.metrics_window`, `_shared/socialProvider.ts` + `windsorProvider.ts`; migrations escritas, NÃO aplicadas); 2026-10-29 (consentimento manual: `api.set_email_consent` limpa grupos inativos; push: 2026-10-20, não aplicado) · Frontend mobile: 2026-10-21 (casca e navegação, kit mobile, leitores novos; acompanha ARCHITECTURE v10.0 §25)

---

## Email builder — contrato aditivo (2026-09-10)

| Contrato | Writers | Readers / compatibilidade |
|---|---|---|
| Rich text em título/hero/produto/CTA | IframeCanvas, Inspector | `mjmlRenderer` aceita texto antigo e HTML sanitizado; CTA conserva URL separada |
| `block.props.mobile` opcional | Inspector | `mobileOverrides`, `designToHtml`, `renderEmail`/`renderEmailSync`; ausência usa desktop; CSS reaplicado após hardener |
| HTML compilado / renderer `1.1.0` | CampaignEditor e TemplateEditor via renderer canônico; exports do editor | email-campaign worker consome HTML e compara constante compartilhada; versão mínima aceita permanece inalterada |
| Biblioteca e versões locais | BlockPalette, useDesignPersistence | Biblioteca por tenant; recuperação por tenant/entidade; sem alteração de schema nem sincronização entre dispositivos |

Frontend e worker compartilham `workers/shared/src/email-renderer-version.ts`. Publicar ambos ao promover esta alteração. O formato `design_json` mantém versão e propriedades anteriores; não há remoção de campos.

## Queue Routing Map

> **REGRA R33**: Job enfileirado na tabela/worker errado fica pending infinitamente sem erro.
> Antes de enfileirar: (1) confirmar a tabela-fila correta, (2) confirmar que o job_type está na allowlist do worker consumidor.

### Roteamento por tabela-fila

| Tabela-fila | Worker consumidor | Claim RPC | Job types aceitos |
|---|---|---|---|
| `integrations.jobs` | ingestion-worker | `worker.claim_jobs(worker_id, handler_list)` | `commerce:process_batch`, `eduzz:process_invoice`, `eduzz:process_contract`, `eduzz:process_cart_abandonment` |
| `integrations.jobs` | historical-sync-worker | `worker.claim_jobs(worker_id, handler_list)` | `eduzz:import_sales_page`, `eduzz:import_products`, `eduzz:enrich_sales`, `eduzz:enrich_invoice`, `eduzz:get_sales_count`, `sympla:historical_sync`, `typeform:sync_forms`, `typeform:sync_responses`, `typeform:enrich_themes`, `typeform:promote_forms`, `typeform:promote_responses`, `typeform:backfill_settings` |
| `integrations.jobs` | email-sync-worker | `worker.claim_jobs(worker_id, handler_list)` | `nylas:deep_sync`, `nylas:deep_sync_folder`, `nylas:initial_sync`, `nylas:apply_message_ops`, `nylas:sync_folders`, `cache_attachments_scan`, `cache_attachment_batch`, `cleanup_jobs` |
| `integrations.sync_schedules` | pull-sync-worker | `integrations.claim_sync_schedules(worker_id, providers)` | N/A (schedule-based por provider: `sympla`, `doare`, `typeform`) |
| `analytics.processing_jobs` | analytics-worker APENAS | `analytics.claim_next_jobs(worker_id, batch)` | `refresh_features`, `refresh_rfm`, `refresh_reactivation`, `refresh_segments`, `compute_clusters`, `compute_lookalikes`, `compute_cluster_subgroups`, `sync_customers`, `compute_lookalike_audience`, `link_products`, `project_traits`, `sync_field_definitions` |
| `journeys.processing_jobs` | journeys-worker | `journeys.claim_next_job(worker_id)` | `check_segment_entries`, `check_form_entries`, `check_event_entries`, `process_enrollment`, `link_submission_to_lead` |
| `journeys.processing_jobs` | email-campaign-worker | `journeys.claim_next_job(worker_id)` | `prepare_email_batch`, `send_email_batch`, `retry_soft_bounce` |
| `journeys.event_processing_jobs` | journeys-event-worker APENAS | FOR UPDATE SKIP LOCKED direto | Status-based (`pending`/`retry`), sem job_type — processa todos os eventos disponíveis |
| `journeys.backfill_jobs` | journeys-event-worker APENAS (loop de backfill, tick 5s) | `claimBackfillJobs` — UPDATE … FOR UPDATE SKIP LOCKED direto (pending/retry prontos + running com claim > 10 min) | N/A (1 job por jornada, UNIQUE `journey_id`); 1 lote de 500 por claim |
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
| `segment_parties` não vazio se segmento tem regras | Envio: `email.create_campaign_send` (R61) → `segment_not_materialized`/`exclusion_segment_not_materialized`/`segment_empty`; preview: `CampaignSend.tsx` `getSegmentRecipientsFailClosed()` | **R38:** membership vazia = refresh pendente, não "enviar para todos". A UI mostra "Calcular agora e enviar" (`api.refresh_segment_customers` + 1 retentativa) |
| Teto de destinatários (50.000) | `email.create_campaign_send` (R61) → `too_many_recipients` | Envio recusado sem gravar nada. O guard de blast radius do front (lista ≤ 1,1× segmento) saiu em 2026-10-26: a lista não é mais montada no cliente |
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
  - `src/pages/admin/CampaignSend.tsx` `getSegmentRecipientsFailClosed()` — check de has-rules no guard fail-closed (só preview de destinatários desde 2026-10-26; o envio é `api.create_campaign_send`)
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
  - `src/pages/admin/CampaignSend.tsx` — has-rules check (fail-closed; só preview desde 2026-10-26)
  - `src/lib/segmentStorage.ts:291,356` — passagem para preview RPCs
  - `src/lib/segmentation/astV2ToBuilder.ts:432` — hydrate builder UI (legado, não usado pela page atual)
  - `src/lib/segmentation/normalizeAstV2.ts` — adapter de shape no load (R48)
  - `src/components/segments/SegmentBuilderV2.tsx:46` — preview RPC
  - `src/pages/admin/SegmentationList.tsx:258` — duplicate segment
  - `src/pages/admin/SegmentationBuilder.tsx` — load segment for editing (chama `normalizeAstV2ForUI` + `fromBackendAST`)
- **Regra:** Source of truth para regras rule-based. Avaliação por `build_unified_where_clause_v2()`. Shape canônico: `root.children` apenas grupos (R48). UI aplica `normalizeAstV2ForUI` no load para resiliência a shapes legacy até Fase 2/4.

### analytics.segment_parties

- **Writers (SQL):**
  - `refresh_segment_parties_unified()` — INSERT/DELETE diff após avaliação v2 (canonical, chamada via `process_segment_refresh_queue`)
  - `refresh_segment_parties()` — legacy v1 (D-SEG-9: `trg_refresh_segment_parties_on_change` ainda chama esta versão; suspeita de parte da causa-raiz do drift)
  - `apply_segment_membership_diff()` — diff incremental (triggers de transação)
  - `sync_cluster_segment_customers()` — clusters/subgroups
- **Writers (Node.js):**
  - `workers/analytics/src/segmentation/segmentEvaluator.ts` — `refreshSegmentV2()` é **wrapper de `analytics.refresh_segment_parties_unified()` desde D-SEG-10/v7.21** (não escreve direto)
  - `workers/analytics/src/handlers/computeClusters.ts:737` — DELETE orphan ids
  - `workers/analytics/src/handlers/computeClusterSubgroups.ts:908` — DELETE orphan ids
  - `workers/analytics/src/handlers/computeLookalikeAudience.ts:607,644` — DELETE+INSERT bulk para lookalike (`source='customer'` é correto — todos os membros são customers por definição)
- **Writers (Frontend — irregular, D-SEG-12):**
  - `src/lib/analyticsStorage.ts:527` — `createSegmentFromLookalikes()` faz BULK INSERT direto via `supabase.schema("analytics").from("segment_parties").insert(batch)` com `source: "customer"` hardcoded. Bypassa funções canonical. Migrar para função SQL canonical (D-SEG-12).
- **Readers:**
  - `email-campaign-worker` — campanhas (fail-closed por R37/R38)
  - `journeys-worker` — segment-trigger journeys
  - `generate-export` Edge Function — export de segmentos
  - `src/lib/segmentStorage.ts:555` — `getSegmentPartiesForCampaign()` → envio de campanha
  - `src/pages/admin/CampaignSend.tsx` — via `getSegmentRecipientsFailClosed()` (R38), só no preview "Ver destinatários" (UI 2026-10-26, não deployada)
  - `api.get_segments_recipient_count(tenant_id, segment_ids[], exclude_segment_ids[])` — COUNT DISTINCT lower(email) entre múltiplos segmentos, descontando segmentos de exclusão; `segment_ids NULL` = todos os contatos do tenant. Usado por `CampaignSend.tsx` para exibir contagem exata de destinatários únicos no botão de envio (somente leitura, mesma semântica de `get_segment_parties_for_campaign`)
  - `CampaignSend.tsx` — `getExclusionEmailSet()` lê membros de segmentos marcados como EXCLUSÃO para o preview de destinatários (desde a UI de 2026-10-26 o disparo manda só `exclude_segment_ids` e o servidor aplica). IDs auditados em `email.campaign_sends.excluded_segment_ids`
  - `email.create_campaign_send` (via `api.create_campaign_send`, R61, 2026-10-26) — materializa destinatários de campanha (`mode=segments`) e aplica segmentações de exclusão (`segments`/`all`/`resend_unopened`) com `INSERT … SELECT` no servidor; fail-closed igual ao front (segmento com regras e 0 membros → `segment_not_materialized`; exclusão idem → `exclusion_segment_not_materialized`; segmento sem regras e vazio → `segment_empty`). Substitui `getSegmentRecipientsFailClosed()`/`getExclusionEmailSet()` no envio na UI de 2026-10-26 (não deployada); o preview continua usando-os.
  - `supabase/functions/check-journey-entries/index.ts:213` — enrollment de journey
  - UI — counts, listagens (`get_segment_parties_preview`)
  - Views: `analytics.v_segment_unified` (preview)
- **Regra R38:** ÚNICA fonte para membership de segmento. Todo código que precisa de lista de membros/recipients DEVE ler daqui. NUNCA re-avaliar regras no momento do consumo.
- **Drift conhecido (D-SEG-5):** sem causa-raiz identificada, divergência entre `segment_parties` e avaliação fresh já foi observada em 22/52 segmentos manuais ativos (auditoria 30/04, "LEADS COM TELEFONE" com +9.153 contatos a mais que a regra permite). Operação A em 30/04 reconciliou tudo via `refresh_segment_parties_unified` em massa. Auditoria periódica recomendada (D-SEG-8) até D-SEG-5 ser fechado.

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
- **Writers — tag_ids (SQL triggers `trg_sync_tag_ids_customer` / `trg_sync_tag_ids_lead` em `crm.customer_tags` / `crm.lead_tags`):**
  - `analytics.sync_tag_ids_to_contact_state()` — append/remove em `tag_ids` + `apply_contact_state_mutation('syncTagIds')`
  - Writers de `crm.customer_tags`/`crm.lead_tags` que disparam o trigger: `workers/journeys/src/handlers/processJourneyNode.ts` `executeAddTagNode` (R62: `ON CONFLICT (party_id, tag_id)`; cliente sem `customer_id` grava só `party_id`), `executeRemoveTagNode` (DELETE)
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
  - **`api.create_campaign_send(p jsonb)` → `email.create_campaign_send` (R61, 2026-10-26, migration `20261026120100_create_campaign_send_atomic`)** — INSERT do envio + destinatários + job numa transação; copia `message_type`/`group_id`/`transactional_reason` da campanha; agendado → marca a campanha `scheduled`. Único writer de envio direto da UI depois do deploy da UI nova.
  - `src/pages/admin/CampaignSend.tsx` — **UI 2026-10-26 (não deployada):** não insere mais; `createSendMutation`/`sendTestMutation` → `src/lib/email/campaignSendRpc.ts` `submitCampaignSend()` → `api.create_campaign_send`. Até o deploy, o bundle em produção ainda faz o INSERT direto legado.
  - `email.cancel_orphan_campaign_sends` / `api.cancel_orphan_campaign_sends` (R61) — UPDATE `pending` sem destinatários e sem job há > 30 min → `cancelled` (`error_message='orphan_send_without_recipients: …'`). Chamador: `CampaignSend.tsx` ao abrir a tela (uma vez por tenant, em background, só loga; UI 2026-10-26)
  - `supabase/functions/process-journey/index.ts:461` — INSERT: journey email node
  - `workers/journeys/src/handlers/processJourneyNode.ts:1629` — INSERT: journey worker email node
  - `workers/email-campaign/src/handlers/prepareEmailBatch.ts:24,47,58,85` — UPDATE status (sending/failed/completed)
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:630,658,691` — UPDATE status + counts
- **Readers:**
  - `src/pages/admin/CampaignSend.tsx:426`, `CampaignResults.tsx:143`, `Campaigns.tsx:179`, `JourneyAnalytics.tsx:352`
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:119` — load send for processing
  - `api.get_campaign_unsubscribe_stats` (Onda 3) — filtra `send_id`s do tenant; `delivered_count` como fallback de entregues
  - `api.get_tenant_consent_summary` (Onda 3) — contagem de envios marketing/transacional e `sum(sent_count)` transacional na janela (`is_test` fora; status `cancelled/scheduled/pending/draft` fora; data `coalesce(started_at, created_at)`)
  - `api.get_account_overview` (2026-10-28) — `count`/`sum(sent_count)`/`sum(delivered_count)` do tenant em mês (SP), 30 dias e total, mesmo filtro do `get_tenant_consent_summary`; consumidor `src/lib/accountOverview.ts` → `AccountBilling.tsx` (aba Visão geral)
- **Colunas de rastreabilidade (Guard 1):** `recipient_source` (segment/manual/all/test/resend_unopened — o último desde 2026-10-26, gravado só por `api.create_campaign_send`), `segment_ids` (uuid[]), `segment_parties_snapshot` (int), `recipient_count_at_send` (int). Adicionadas Sprint 12 para auditoria de blast radius.
- **Validação no banco (R60, 2026-10-21, migration `20261021120000_campaign_message_type_validation`):** trigger `trg_campaign_sends_validate_message_type` (BEFORE INSERT OR UPDATE OF `message_type, group_id, transactional_reason`) → `email.campaign_sends_validate_message_type()`. Marketing sem `group_id` → `marketing_requires_group`; grupo de outro tenant → `invalid_group`; transacional → `group_id` forçado a NULL e `transactional_reason` obrigatório quando `is_test=false` (vem do send ou, se ausente, de `email_campaigns.transactional_reason`) → senão `transactional_requires_reason`. Todos SQLSTATE `23514`. Em UPDATE só valida quando essas colunas mudam de fato; SET NULL da FK de grupo é aceito. **Todo writer novo** de `campaign_sends` precisa mandar grupo (marketing) ou ter motivo na campanha (transacional).
  - `workers/journeys/src/handlers/processJourneyNode.ts` passa `transactional_reason` da campanha e trata a rejeição (`sendValidationErrorCode`): nó falha uma vez com mensagem clara, sem retry (não deployado).
  - `supabase/functions/process-journey/index.ts` (legado, sem callers conhecidos) insere sem `message_type`/`group_id` → seria rejeitado por `marketing_requires_group`.
- **Regra:** NÃO contém lista de recipients. É apenas metadados (template, subject, status, counts). A lista real está em `campaign_recipients`.

### email.campaign_recipients — Lista congelada de envio (R38)

- **Writers:**
  - **`api.create_campaign_send(p jsonb)` → `email.create_campaign_send` (R61, 2026-10-26)** — `INSERT … SELECT` no servidor, na mesma transação do `campaign_sends` e do job: `segments` (`analytics.segment_parties` ⋈ `crm.parties`), `all` (`crm.vw_parties_unified` cliente ou lead), `emails` (lista explícita/teste, minúsculas) e `resend_unopened` (destinatários `sent|delivered` sem abertura/clique/bounce/descadastro/spam de um envio anterior da mesma campanha). Dedupe por `lower(btrim(email))`; segmentações de exclusão aplicadas (exceto `emails`); `party_id` preenchido direto da fonte (no modo `emails` a trigger `trg_resolve_recipient_party_id` resolve). Teto 50.000. **O front nunca mais insere aqui (R61).**
  - `src/pages/admin/CampaignSend.tsx` — INSERT bulk direto **removido na UI de 2026-10-26** (não deployada; causava envio órfão por timeout de 8 s). O bundle em produção ainda o faz até o deploy.
  - `supabase/functions/process-journey/index.ts:477` — INSERT: single recipient (journey node)
  - `workers/journeys/src/handlers/processJourneyNode.ts:1652` — INSERT: single recipient (journey worker)
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:376,393,408` — UPDATE status → `skipped` (suppressed/unsubscribed)
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:503,520,561` — UPDATE status → `sent`/`pending`/`failed`
  - `workers/email-campaign/src/handlers/retrySoftBounce.ts` — UPDATE status (bounce retry; `skipped` quando marketing e descadastrado global/grupo, mesmo `error_message` do lote — 2026-10-20)
  - `supabase/functions/integrations-worker/handlers/sendgrid.ts:82,99` — UPDATE tracking fields via `sendgrid_message_id`
  - ~~`supabase/functions/unsubscribe/index.ts` UPDATE direto no one-click~~ — desde a Onda 1b (2026-10-20, não deployado) o one-click passa por `email.apply_consent_change` (`channel=one_click`)
  - `email.apply_consent_change` — UPDATE `unsubscribed_at` (1ª vez vence) em todo descadastro com destinatário resolvido; `status='unsubscribed'` só no one-click global
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts` / `retrySoftBounce.ts` — UPDATE `status='failed'`, `error_message='preference_center_url_unavailable'` em marketing quando o tenant não tem domínio e `PREFERENCE_CENTER_FALLBACK_URL` não existe (P18, Onda 3, não deployado)
  - Migration `20261022120300_consent_backfill_d3_d4` (D4, aplicada) — preencheu `unsubscribed_at` em 1.607 linhas a partir de `unsubscribes.send_id` (sem tocar `status`)
- **Readers:**
  - `workers/email-campaign/src/handlers/prepareEmailBatch.ts:76` — COUNT pending recipients
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:616,650,677,684` — COUNT by status (completion checks)
  - `workers/email-campaign/src/handlers/retrySoftBounce.ts:90` — SELECT single recipient for retry
  - `supabase/functions/webhooks-sendgrid/index.ts:131` — lookup `tenant_id` by `sendgrid_message_id`
  - `src/pages/admin/CampaignSend.tsx:502` — SELECT failed/bounced for display
  - `src/pages/admin/CampaignResults.tsx:186` — SELECT all (non-skipped) for results view; desde a Onda 3 a tabela "Contatos que cancelaram" e o fallback do card "Descadastros" usam `unsubscribed_at` (não mais `status='unsubscribed'`, que só o one-click grava)
  - `workers/journeys/src/handlers/processJourneyNode.ts:1615` — SELECT dedup guard (already sent?)
  - `api.get_campaign_unsubscribe_stats` (Onda 3) — `delivered_at`, `unsubscribed_at`, `spam_reported_at` por envio (métrica de descadastro/spam da UI de resultados)
    - UI (Onda 3, não deployado): `src/lib/email/consentMetrics.ts` `fetchCampaignUnsubscribeStats()` (lotes de 500 `send_id`s = limite da RPC; taxas recalculadas no front a partir das contagens) → `CampaignResults.tsx` (cards "Descadastros"/"Reclamações de spam" + banner de alerta) e `Campaigns.tsx` (uma chamada com todos os envios das campanhas `sent`; taxa na linha e no painel "Metricas"). Alerta > 0,5% descadastro / > 0,1% spam com ≥ 100 entregues.
- **Regra R38:** Lista CONGELADA de recipients. O email-campaign-worker é blind sender — só processa o que está aqui. Nunca re-avaliar segmento no momento do envio. Se 0 rows para um `campaign_send_id`, o envio não acontece (fail-closed by design).

### email.suppression_list — Bounce/spam suppression

- **Writer canônico (R59, 2026-10-20):** `email.apply_consent_change` (action `suppress`/`unsuppress`; tipos `hard_bounce|spam_report|invalid|unsubscribe|manual`). `spam_report`/`unsubscribe` também gravam descadastro global.
- **Via RPC (Onda 1b, 2026-10-20 — não deployado):** `supabase/functions/integrations-worker/handlers/sendgrid.ts` `handleSuppression()` — hard_bounce (`bounce` type=bounce), invalid (`dropped`), spam_report (`spamreport`), unsubscribe (`unsubscribe`) via `applyConsentChange` (`_shared/consent.ts`, `channel=webhook_sendgrid`); erro da RPC lança e o job é re-tentado.
- **Writers diretos (exceções da R59):**
  - `supabase/functions/integrations-worker/handlers/sendgrid.ts` `handleSoftBounceCounter()` — INSERT/UPDATE soft_bounce (contador/threshold; `bounce` não-hard e `blocked`). Erros agora lançam (antes eram ignorados).
  - `email.expire_soft_bounce_suppressions()` (pg_cron/worker) — UPDATE is_active=false de soft bounce expirado (housekeeping, exceção documentada na R59)
- **Readers:**
  - `api.get_tenant_consent_summary` (Onda 3) — ativos por `suppression_type` (exclui soft bounce expirado). `suppression_list` não tem `send_id`: o spam por envio vem de `email_events`/`campaign_recipients`.
  - `api.get_email_consent_status` (2026-10-29) — supressões ativas e não expiradas de um email (`suppression_type`, `reason`, `suppressed_at`) → `effective=suppressed`.
    - UI: `TenantConsentSummaryCard` (suprimidos por tipo) e `estimateBlockedRecipients()` (soma de `suppressed_by_type` no "até N podem ser pulados" de `CampaignSend.tsx`). O front não tem GRANT em `suppression_list`; esta RPC é o único caminho.
  - Migration `20261022120300_consent_backfill_d3_d4` (D3) — leu `spam_report` ativos sem global
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:253` — SELECT suppressed emails (pre-send gate)
  - `workers/email-campaign/src/handlers/retrySoftBounce.ts:120` — SELECT before retry
  - `workers/journeys/src/handlers/processJourneyNode.ts:1883` — SELECT before journey send
  - `email.consent_state_snapshot()` — before/after da trilha de consentimento
  - `api.get_email_consent_status(p_tenant_id, p_email)` (2026-10-29) — global (ativo ou não) + unsubs de grupo ativos com nome/`is_active` do grupo; unsub ativo em grupo inativo → `effective=global_unsubscribed` (R60). Qualquer membro do tenant.
- **Regra:** Check obrigatório antes de qualquer envio. Tripla camada: suppression + global unsub + group unsub.

### email.unsubscribes — Global/group unsubscribes

- **Writer canônico (R59, 2026-10-20):** `email.apply_consent_change` / `email.apply_consent_changes` (SECURITY DEFINER, service_role) via `supabase/functions/_shared/consent.ts`. Upsert global usa o índice parcial `idx_unsubscribes_tenant_email_global`; grupo usa a constraint `unsubscribes_tenant_id_email_group_id_key`.
- **Via RPC (Onda 1b, não deployado):** webhook SendGrid `spamreport`/`unsubscribe` → `email.apply_consent_change` `suppress` (`spam_report`/`unsubscribe`) grava também o global (`source=webhook`) — substitui o upsert quebrado (P3). `group_unsubscribe`/`group_resubscribe` da SendGrid só gravam `email_events` (grupo ASM não mapeia para `email.unsubscribe_groups`).
- **Via RPC (Onda 1b — página e edge functions, 2026-10-20, não deployado):**
  - `supabase/functions/unsubscribe/index.ts` → `_shared/unsubscribeEndpoint.ts` — POST one-click: `apply_consent_change` global `channel=one_click` (POST sem corpo one-click: `footer_get`); GET só `preferences_view`
  - `supabase/functions/save-preferences/index.ts` → `_shared/preferenceCenter.ts` — `apply_consent_changes` `channel=preference_center` (actions `global_unsubscribe`/`global_resubscribe`/`save_groups`/`feedback`; contrato em `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` "Execução — Onda 1b (página e edge functions)")
- **Writers legados:** nenhum nas edge functions (o upsert direto foi removido de `unsubscribe` e `save-preferences`).
  - ~~`supabase/functions/eduzz-processor-worker/handlers/sendgrid.ts`~~ — removido em 2026-10-20 (código morto, P19)
- **Via RPC (Onda 2.2, 2026-10-21):** `api.deactivate_unsubscribe_group(...)` — `convert_to_global`/`migrate_to_group` geram unsubs via `email.apply_consent_changes` (`channel=admin_manual`, `request_payload.reason='group_deactivated'`).
- **Via RPC (Onda 3, 2026-10-22; `admin_manual` desde 2026-10-29):** `email.apply_consent_change` `resubscribe` global com `channel=preference_center` (inclui o `confirm_resubscribe` do double opt-in) ou `channel=admin_manual` (`api.set_email_consent`, migration `20261029180000_manual_consent_rpcs_fixes`) também faz `is_active=false` nos unsubs ativos de grupos INATIVOS do mesmo email/tenant (ids em `request_payload.inactive_groups_cleared`, uma linha de auditoria). Demais canais não limpam. Migration `20261022120300_consent_backfill_d3_d4` (D3) criou 49 globais `source='manual'` (`channel=system`, `request_id='backfill_spam_report_D3_20261022'`) para spam_report ativos sem global.
- **Via RPC (consentimento manual, 2026-10-29):** `api.set_email_consent(p_tenant_id, p_email, p_action, p_scope, p_group_id, p_reason)` — tela do tenant; `email.apply_consent_change` `channel=admin_manual` (`source=manual`), `actor_user_id=auth.uid()`, `request_payload {reason (≤ 500 chars), ui:'tenant_admin'}`. Unsubscribe: membro não-viewer; resubscribe: owner/admin. Resubscribe global também limpa unsubs ativos de grupos inativos (`inactive_groups_cleared`). Retorno **sem `before_state`/`after_state`** desde 2026-10-29 (consumidores não devem ler essas chaves; o estado vem de `api.get_email_consent_status`). Consumidores: `src/lib/email/manualConsent.ts` (`setEmailConsent`, `submitManualUnsubscribe` — 1 chamada global ou N por grupo) → `useManualConsentActions` → `ManualUnsubscribeDialog`/`EmailConsentStatusCard` (detalhe do contato via `ConsentSection`) e `UnsubscribeGroupDetail.tsx` ("Adicionar email" e "Reinscrever" por linha). Status lido por `api.get_email_consent_status` (`getEmailConsentStatus`, cache `["email-consent-status", tenantId, email]`).
- **FK `unsubscribes_group_id_fkey`:** `ON DELETE RESTRICT` desde 2026-10-21 (antes CASCADE apagava o histórico). Grupo com unsubs não pode ser excluído; o caminho é desativar. Cascata de exclusão de tenant testada e funcionando.
- **IP:** `ip_address` sempre NULL desde 2026-10-21 (Onda 2.5; históricos anonimizados). IP só como `ip_hash` em `consent_audit_log`.
- **Readers:**
  - `workers/email-campaign/src/unsubscribeFilter.ts` `loadActiveUnsubscribes()` — SELECT global + unsubs de grupo INATIVO (valem como global, R60) + grupo do envio; usado por `sendEmailBatch.ts` (pre-send gate, relido a cada lote) e por `retrySoftBounce.ts` (P11). Transacional não checa. Skip `inactive_group_unsubscribe` grava `error_message` próprio. Não deployado.
  - `workers/journeys/src/handlers/processJourneyNode.ts` `executeSendEmailNode` — SELECT global + grupo inativo (R60, output `inactive_group_unsubscribed`) e depois grupo da campanha, antes do envio da jornada (não deployado)
  - Mesma regra nas duas cópias: mudou uma, mude a outra.
  - `email.get_preference_center_data()` — página de preferências (via save-preferences `action=load`; email devolvido mascarado)
  - `src/pages/admin/UnsubscribeGroupDetail.tsx:87` — SELECT for group detail (sem `ip_address` desde a Onda 2.5)
  - `src/pages/admin/UnsubscribeGroups.tsx:87` — SELECT for group counts
  - `src/lib/email/unsubscribeGroups.ts` `countActiveGroupUnsubscribes()` — count exato por `group_id` ativo no diálogo "Desativar grupo" (Onda 2.2)
  - `supabase/functions/_shared/preferenceCenter.ts` `loadActiveUnsubscribesFromDb()` — SELECT `group_id, source` ativos + `unsubscribe_groups.is_active` dos grupos (service role) para o gate de double opt-in do save-preferences (Onda 2.3): reativação imediata só se `source='preference_center'` e token ≤ 7 dias. Desde a Onda 3 o `global_resubscribe` também considera as linhas de grupos INATIVOS. Mudar valores de `source` exige revisar `DIRECT_RESUBSCRIBE_SOURCES`.
  - `api.get_campaign_unsubscribe_stats` (Onda 3) — emails distintos com `send_id` do envio (ativos ou não); índice parcial novo `idx_unsubscribes_send`
  - `api.get_tenant_consent_summary` (Onda 3) — contagens de ativos (global / grupo ativo / grupo inativo / pessoas bloqueadas) e descadastros na janela por `unsubscribed_at`
    - UI (Onda 3, não deployado): `src/components/email/TenantConsentSummaryCard.tsx` no topo de `UnsubscribeGroups.tsx` (7/30/90 dias; `42501` esconde o card) via `fetchTenantConsentSummary()` (`src/lib/email/consentMetrics.ts`)
  - `src/lib/email/consentMetrics.ts` `estimateBlockedRecipients()` (Onda 3, P21) — count exato (head) de ativos global + grupo do envio + grupos inativos, para o "até N podem ser pulados" de `CampaignSend.tsx` (marketing; transacional não lê). Teto, não interseção com o segmento.
  - `email.consent_state_snapshot()` — before/after da trilha de consentimento

### email.unsubscribe_groups — Grupos de descadastro

- **Writers:** UI `UnsubscribeGroups.tsx` (INSERT/UPDATE de nome/descrição direto por RLS; desativação só por `api.deactivate_unsubscribe_group` via `DeactivateUnsubscribeGroupDialog` desde a Onda 2 — UPDATE direto de `is_active` removido da UI, não deployado), `api.deactivate_unsubscribe_group` (UPDATE `is_active=false`).
- **Readers que dependem de `is_active`:** `email.apply_consent_change` (rejeita descadastro/reativação de grupo inativo: `invalid_group`), `api.set_campaign_message_type` (marketing exige grupo ativo), `unsubscribeFilter.ts` e `processJourneyNode.ts` (unsub de grupo inativo = global, R60), `email.get_preference_center_data` (lista só ativos), `src/lib/email/consentMetrics.ts` `estimateBlockedRecipients()` (ids de grupos inativos entram na estimativa de pulados, R60).
- **Lacuna conhecida:** quem tem unsub ativo num grupo inativo fica bloqueado para marketing, mas a página de preferências não mostra esse grupo nem o trata como global; "voltar a receber" (resubscribe global) não remove esse bloqueio.

### email.campaign_type_changes — Trilha de troca de tipo de campanha (2026-10-21)

- **Writer:** `api.set_campaign_message_type` (só quando algo muda) e `email.duplicate_campaign` (cópia de transacional sem motivo convertida em marketing, 2026-10-26). RLS sem policies, sem grants a anon/authenticated; service_role SELECT/INSERT.
- **Readers:** nenhum na UI ainda (operacional via service_role).

### email.email_campaigns.message_type / transactional_reason (R60)

- **Writer na duplicação (2026-10-26, migration `20261026120000_duplicate_campaign_copies_transactional_reason`):** `email.duplicate_campaign` (via `api.duplicate_campaign`; chamadores `Campaigns.tsx`, `CampaignResults.tsx`, `CampaignMobileSummary.tsx`) copia `transactional_reason`; transacional com motivo → cópia transacional sem grupo; transacional sem motivo → cópia **marketing** com o grupo da origem (se ativo) ou o padrão ativo do tenant, e uma linha em `email.campaign_type_changes` (`campaign_id` = cópia). Antes a cópia nascia sem motivo e o envio era recusado (`transactional_requires_reason`).
- **Writers:** `api.set_campaign_message_type` (owner/admin; transacional exige motivo e zera `group_id`; marketing exige grupo ativo). A UI legada ainda faz UPDATE direto de `message_type` (sem motivo) até o deploy da Onda 2 — campanha virada para transacional por esse caminho depois do backfill fica sem motivo e seus envios não-teste são rejeitados pela trigger de `campaign_sends` (`transactional_requires_reason`). Na Onda 2 (interface, não deployado) a UI passa a usar a RPC: `CampaignEditor.tsx` → `CampaignMessageTypeField` → `src/lib/email/messageType.ts` `setCampaignMessageType()`; os saves do editor não mandam mais `message_type` e só mandam `group_id` em marketing. `CampaignSend.tsx` (UI 2026-10-26) duplica via `api.duplicate_campaign` e, no diálogo de próxima ação (`SendOutcomeDialog`), chama `setCampaignMessageType()` para converter para marketing, registrar o motivo ou trocar o grupo (owner/admin; editor troca só o grupo por UPDATE de `group_id` em campanha de marketing, como o editor).
- **Backfill 2026-10-21:** 139 campanhas transacionais receberam `transactional_reason='legacy: classificado antes da validação (2026-10-21)'` (tipo e `group_id` da campanha inalterados).
- **Readers:** trigger de `campaign_sends` (fallback do motivo), `processJourneyNode.ts`.

### email.unsubscribe_feedback — Motivo do descadastro

- **Writer canônico:** `email.apply_consent_change` (action `feedback`; falha vira `result=error` na trilha sem derrubar a transação). CHECK `unsubscribe_feedback_reason_check` inclui `no_more` e `accidental` desde 2026-10-20 (P6).
- **Writer legado:** nenhum — `save-preferences` grava feedback pela RPC desde a Onda 1b (2026-10-20, não deployado); falha → 200 `feedback_saved:false`.
- **Readers:** UI admin de descadastros (RLS por membership).
- **IP:** `ip_address` sempre NULL desde 2026-10-21 (Onda 2.5; 210 históricos anonimizados).

### email.consent_audit_log — Trilha de consentimento (append-only, 2026-10-20)

- **Writers (somente SECURITY DEFINER):** `email.apply_consent_change`/`apply_consent_changes` (via `email.consent_audit_write`), `email.log_consent_error` (caller grava erro quando a RPC lança), migration `20261020120400_consent_audit_baseline_snapshot` (`channel=system`, `request_id='baseline_snapshot_20261020'`).
- **Readers:** `api.get_consent_history` (authenticated; membership + owner/admin), `email.get_consent_history` (service_role). Nenhum SELECT direto de anon/authenticated.
  - Rate limit (Onda 3, 3.4): `email.count_recent_resubscribe_requests(p_tenant_id, p_email, p_hours=24)` conta `preferences_view` com `request_payload.double_opt_in.status='requested'` (save-preferences → 429 a partir do 4º pedido/24 h); `email.count_recent_preference_loads_by_ip(p_ip, p_minutes=60)` conta `preferences_view`/`preference_center` do mesmo `ip_hash` (save-preferences `load` → 429 a partir de 60/h). Ambas service_role. Quem mudar o formato de `request_payload.double_opt_in` ou o canal do `load` quebra o limite. Índice novo `idx_consent_audit_ip_time (ip_hash, occurred_at desc) where ip_hash is not null`.
  - UI: `src/components/contacts/ConsentHistoryCard.tsx` (em `ContactDetail.tsx`, só owner/admin; `42501` esconde a seção) via `src/lib/email/consentHistory.ts` (Onda 2.2).
- **Contrato:** UPDATE/DELETE/TRUNCATE bloqueados por trigger. `ip_hash` = `email.hash_ip()` (HMAC, segredo no Vault `consent_ip_hash_secret`). Enums de `action`/`channel`/`result` espelhados em `_shared/consent.ts` — mudar os dois juntos (R7/R31).

### journeys.journey_enrollments — saída por descadastro global

- `email.consent_cancel_journey_enrollments()` (chamado pela RPC de consentimento) — UPDATE `status='exited'`, `context_json.cancellation_reason='global_unsubscribe'`.
- `public.cancel_enrollments_on_global_unsubscribe()` — corrigido em 2026-10-20 (migration `20261020130000`, aplicada): wrapper que delega a `email.consent_cancel_journey_enrollments()` (`exited`), mesma assinatura/retorno; EXECUTE só `service_role`. Sem callers nas edge functions desde a Onda 1b (2026-10-20, não deployado): `unsubscribe` e `save-preferences` usam a RPC, que já cancela as jornadas.

### email.email_events — Email engagement tracking

- **Writers:**
  - `supabase/functions/integrations-worker/handlers/sendgrid.ts` — INSERT com campaign context via `resolveCampaignContext()` (open, click, delivered, bounce, etc.)
  - `email.apply_consent_change` — INSERT `unsubscribe`/`group_unsubscribe` (`sendgrid_message_id='self:<recipient_id>'`) quando há `send_id` e o canal não é webhook
  - ~~`supabase/functions/unsubscribe/index.ts` INSERT direto do one-click~~ — removido na Onda 1b (a RPC grava o evento)
- **Readers:**
  - `workers/shared/src/segmentation/resolvers.ts` — `email_engagement` resolver compileSQL (EXISTS subquery por `send_id` + `event_type`)
  - `workers/analytics/src/segmentation/resolvers.ts` — mesma cópia do resolver
  - `workers/journeys/src/conditionEngine.ts` — pre-enrichment query (`extractEngagementSendIds` → `__email_engagement` no contactState)
  - `api.get_campaign_unsubscribe_stats` (Onda 3) — `event_type='spamreport'` por envio: `send_id` da linha OU `raw_payload->>'send_id'` (custom_arg; o webhook grava spamreport sem `send_id` na coluna — só 1 de 69 tinha). Se o webhook passar a preencher a coluna, nada quebra.
  - `api.get_tenant_consent_summary` (Onda 3) — `spamreport` distintos na janela
- **Indexes:** `idx_email_events_send` (btree on `send_id` WHERE `send_id IS NOT NULL`)
- **Regra R47:** Engagement data lida via subquery/enrichment, nunca materializada em `contact_state`.

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
- **Producers (RPCs SECURITY DEFINER — inbox outbox/folders):**
  - `inbox.enqueue_message_op` (inline) e `inbox.enqueue_ops_drainer` — job drenador `nylas:apply_message_ops` (dedupe: máx 1 pending/running por conta)
  - `inbox.enqueue_folder_sync_job` — `nylas:sync_folders` via `mark_inbox_active` (throttle 15min) e `api.request_folder_sync` (refresh manual da sidebar)
  - Handlers do email-sync se re-enfileiram: drainer com `run_after` = próximo retry; `sync_folders` +1h e após cada folder op aplicada
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
  - `workers/journeys-event/src/handlers/processBackfillJob.ts` (`processBackfillJob`, CTE `jobs`) — `process_enrollment` priority 3 com `reason='backfill'` (no mesmo statement do INSERT da inscrição)
  - `supabase/functions/check-journey-entries/index.ts:364` — via `journeys.enqueue_job` RPC
  - `supabase/functions/trigger-form-journey/index.ts:319` — via `journeys.enqueue_job` RPC
- **Consumers:** journeys-worker APENAS — via `journeys.claim_next_job` (SKIP LOCKED)

### journeys.event_processing_jobs

- **Producers:** `trg_enqueue_event_processing` (trigger AFTER INSERT em journey events)
- **Consumers:** journeys-event-worker APENAS — via claim/complete/fail cycle (SKIP LOCKED)

### journeys.backfill_jobs (2026-10-16)

Pedido de carga do histórico de uma jornada `event_entry` ativada com "Incluir toda a base existente" (`enroll_existing=true`) ou "a partir de uma data" (`settings_json.enroll_from_date`).

- **Writers:**
  - `journeys.request_journey_backfill(journey_id, from, to)` — UPSERT idempotente (UNIQUE `journey_id`; reativação reseta cursor/contadores). SECURITY DEFINER, EXECUTE só `service_role`.
  - Trigger `trg_journey_enqueue_backfill` (AFTER UPDATE OF status em `journeys.journeys`, transição → `active`) — chama `request_journey_backfill`; em ativação "apenas novos" marca job anterior como `blocked` (`superseded_by_new_only_activation`).
  - `workers/journeys-event/src/handlers/processBackfillJob.ts` — claim (`running`/`locked_by`), cursor + contadores (`events_processed`, `enrollments_created`, `skipped_*`) no MESMO statement do lote; `blocked` se jornada não está ativa; `retry`/`failed` em erro.
  - `workers/journeys-event/src/index.ts` (`runBackfillLoop`) — devolve claims não processados no shutdown.
  - pg_cron `cleanup-backfill-jobs` — DELETE `completed` > 30d.
- **Readers:**
  - `workers/journeys-event/src/handlers/processBackfillJob.ts` (`claimBackfillJobs`, CTE `bj`)
  - `workers/journeys-event/src/monitoring.sql` — observabilidade
- **FK:** `journey_id → journeys.journeys(id) ON DELETE CASCADE` (antes sem cascade — bloquearia DELETE de jornada).

### journeys.backfill_batch_candidates() (2026-10-16)

Fonte ÚNICA da seleção do backfill — mudar aqui muda worker, preview e dry-run juntos.

- **Readers (callers):**
  - `workers/journeys-event/src/handlers/processBackfillJob.ts` — CTE `c` (lote real)
  - `api.preview_journey_backfill(tenant_id, journey_id, from)` — contagem do diálogo de ativação (`src/pages/admin/JourneyBuilder.tsx`, só jornadas por evento)
- **Lê:** `journeys.journeys` (trigger_config.conditions), `commerce.products`, `commerce.transactions` (purchase: `status='paid'`), `journeys.journey_events` (demais eventos), `crm.parties` (party por email quando a fonte não tem), `journeys.journey_enrollments` (dedupe por email/party, qualquer status).

### journeys.journeys.enroll_window_from (2026-10-16)

- **Writer:** `journeys.trg_journey_status_lifecycle` (BEFORE UPDATE, transição → `active`): `enroll_from_date` > `1970-01-01` (enroll_existing) > `last_activated_at`.
- **Readers:** `workers/journeys/src/handlers/checkFormEntries.ts`, `workers/journeys/src/handlers/checkEventEntries.ts` via `resolveEntryWindowStart()` (`workers/journeys/src/utils/enrollmentUtils.ts`). NULL (ativação anterior a 2026-10-16) = regra legada de cada handler.

---

## Journeys

### journeys.journey_enrollments

- **Writers:**
  - `workers/journeys/src/handlers/checkEventEntries.ts:196` — INSERT: event-triggered enrollment
  - `workers/journeys/src/handlers/checkSegmentEntries.ts:224` — INSERT: segment-triggered enrollment
  - `workers/journeys/src/handlers/checkFormEntries.ts:166` — INSERT: form-triggered enrollment
  - `workers/journeys-event/src/handlers/processEventJob.ts:196` — INSERT: event worker enrollment
  - `workers/journeys-event/src/handlers/processBackfillJob.ts` (CTE `ins`) — INSERT … SELECT FROM `journeys.backfill_batch_candidates()` ON CONFLICT DO NOTHING: backfill enrollment (`context_json.enrolled_via='backfill'`)
  - `supabase/functions/check-journey-entries/index.ts:309` — INSERT: edge function enrollment
  - `supabase/functions/trigger-form-journey/index.ts:238,253,269,285` — UPSERT: form-triggered edge function
  - `workers/journeys/src/handlers/processJourneyNode.ts:248,275,327,1717,1739` — UPDATE: status/state during node processing
  - `supabase/functions/process-journey/index.ts:107,196,214,227,239` — UPDATE: node execution updates
- **Readers:**
  - `workers/journeys/src/handlers/checkEventEntries.ts:97,239` — SELECT dedup (existing enrollment?)
  - `workers/journeys/src/handlers/checkSegmentEntries.ts:96,187` — SELECT dedup
  - `workers/journeys/src/handlers/checkFormEntries.ts:81` — SELECT dedup
  - `journeys.backfill_batch_candidates()` — NOT EXISTS por `lower(customer_email)` (índice `idx_journey_enrollments_journey_email_lower`) e por `party_id`
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

## email.email_events.party_id — engajamento na segmentação

- **Writers:** trigger `trg_resolve_event_party_id` (BEFORE INSERT, resolve o party pelo e-mail na escrita). Backfill único em 2026-08-11: 884.448/884.573.
- **Readers:** `analytics.build_email_engagement_clause` (subconsulta dos 4 filtros de engajamento), servido por `idx_email_events_seg_period` e `idx_email_events_seg_send` (índices COBERTOS — index-only scan, heap intocada).
- **Regra (R52):** a subconsulta SEMPRE correlaciona `ee.tenant_id = v_segment_unified.tenant_id`. Todos os call sites usam `FROM analytics.v_segment_unified` sem alias — se algum passar a usar alias, a correlação quebra e a query volta a varrer todos os tenants.
- **Campos atendidos:** `last_email_opened_at`, `last_email_clicked_at` (qualquer email) e `campaign_opened`, `campaign_clicked` (send_ids específicos, com `url_contains`/`url_exact`).

### email.campaign_recipients.party_id — pertencimento à campanha

- **Writers:** trigger `trg_resolve_recipient_party_id` (BEFORE INSERT). Backfill 2026-08-13: 348.761/348.789.
- **Readers:** `analytics.build_campaign_membership_clause`, via índice coberto `idx_campaign_recipients_seg_send`.
- **Semântica:** "está na campanha" = consta na lista de destinatários com `status <> 'skipped'` — `skipped` é supressão explícita (descadastrado/suprimido), nunca recebeu o e-mail.

### Campo unificado `campaign` (segmentação)

- Seis operadores: `in` / `not_in` (destinatários) e `opened` / `not_opened` / `clicked` / `not_clicked` (eventos), mais `url_contains` / `url_exact`.
- Substitui `campaign_opened` / `campaign_clicked`, que seguem funcionando no SQL para compatibilidade (os 4 segmentos que os usavam foram migrados em 2026-08-13).
- **Regra (R54):** subconsultas de segmentação usam `EXISTS` / `NOT EXISTS`, nunca `IN` / `NOT IN`. Com correlação de tenant, `NOT IN` vira SubPlan reexecutado por linha — medido 5,3 s vs 107 ms com anti-join no filtro "recebeu e não abriu".

### journeys.journey_enrollments — campo `journey` (segmentação)

- **Readers:** `analytics.build_journey_clause`, via índice coberto `idx_journey_enrollments_seg`.
- **`party_id` já existia preenchido** (4.452/4.454) — diferente de `email_events` e `campaign_recipients`, não precisou desnormalização.
- **Operadores:** `in` (tem enrollment, qualquer status — `exited` é o estado normal de quem percorreu), `not_in`, `completed`, `not_completed` (= entrou E não concluiu; sem a primeira metade quem nunca entrou cairia no filtro), `active`.
- Campo temporal: o intervalo do grupo filtra `enrolled_at` ("entrou na automação em julho").

### analytics.contact_matches_segment(tenant, party, segment)

- **Callers:** `workers/analytics/src/segment-eval-worker.ts` (segmentos em `diff.deferred`)
- **Contrato:** `true`/`false` = decisão; `null` = indecidível → o chamador PRESERVA a associação atual (R53). Reusa `build_unified_where_clause_v2`, então nunca diverge do refresh em lote.

---

## email.campaign_sends — agendamento e cancelamento

- **Writers:** `api.create_campaign_send` (R61; com `cancel_previous_scheduled` — padrão `true` quando agendado — chama `email.cancel_campaign_scheduled_sends` na mesma transação; sem ele, conflito no índice R49 vira `error_code='already_scheduled'`), `CampaignSend.tsx` (legado em produção: insert do envio; reagendar chama `email.cancel_campaign_scheduled_sends` antes. UI 2026-10-26: só `api.create_campaign_send`, com `cancel_previous_scheduled=true` apenas quando a confirmação mostrou o agendamento vivo ou o usuário escolheu "Cancelar o agendamento anterior e agendar este" no diálogo de `already_scheduled`), `workers/email-campaign` (prepare/send batch: `scheduled|pending` → `sending` → `completed`), `email.cancel_scheduled_send` (→ `cancelled`; o front chama o wrapper `api.cancel_scheduled_send` em `CampaignSend.tsx` e `Campaigns.tsx` desde a UI de 2026-10-26), `reconcile-sendgrid` (contadores), `process-journey` (insert com status `pending`)
- **Readers:** `Campaigns.tsx` (agendados + em andamento, com botão cancelar/parar), `CampaignSend.tsx` (Envios Recentes), `CampaignResults.tsx`
- **Invariante (R49):** índice `campaign_sends_one_scheduled_per_campaign` — no máximo UM envio `scheduled` não-teste por campanha. Jornadas criam `pending`, então não colidem.
- **Cancelamento (R50):** `email.cancel_scheduled_send(send_id, tenant_id)` aceita `scheduled|pending|sending`; marca recipients `pending`→`skipped`, remove jobs pendentes do envio em `journeys.processing_jobs` (por `payload->>'send_id'`) e devolve a campanha para `ready` (nada enviado) ou `sent` (envio parcial). Recipients `processing` são deixados em paz — o lote em voo termina e os seguintes abortam pela checagem de status em `sendEmailBatch.ts`.

---

## Chat (Whaticket) — módulo separado do WhatsApp/Apidozap

### public.chat_contatos / chat_mensagens / chat_config / chat_agentes

- **Writers:** `chat-webhook/index.ts` (mensagens recebidas + resposta IA, service role), `chat-enviar/index.ts` (envio humano, service role), `chat-template-enviar/index.ts` (template, service role), **`chat-historico-importar/index.ts`** (importação do histórico do Whaticket por conexão, service role: `chat_contatos`/`chat_mensagens` com `origem='importado'`, `criado_em` = instante real da API; recalcula `ultima_msg_em/preview` e `ultima_recebida_em`; nunca incrementa `nao_lidas`; grava `chat_conexoes.historico_*`), frontend `src/lib/data/chat.ts` (contatos/config/agentes via RLS)
- **Ordem das mensagens:** `useChatMensagens` (`chat.ts`) ordena por `criado_em` e depois `id` (`compararMensagens` também no realtime) — histórico importado depois tem `id` maior que mensagens mais novas. Índice `chat_mensagens_contato_criado_idx (contato_id, criado_em, id)`.
- **Readers:** `src/pages/admin/Chat.tsx` via hooks de `src/lib/data/chat.ts`; realtime na publication `supabase_realtime`
- **RPCs:** `public.chat_incrementa_nao_lidas` (service_role only), `public.chat_zerar_nao_lidas` (authenticated, valida `app_auth.assert_tenant_access`)
- **Regra:** port 1:1 do módulo campanhas do Carreta Oftalmo — mudanças estruturais devem preservar o mapeamento `campanhas_*` → `chat_*` (ver ARCHITECTURE §22.14)

### public.chat_conexoes / chat_provedor_rota / chat_templates / chat_config_segredos (migrations 2026-11-02, aplicação pendente)

- **chat_conexoes** — token protegido: `authenticated` sem GRANT. Writers: `chat_conexao_salvar` (admin), `chat_conexao_status` (admin), `chat_conexao_token` (system admin; grava `token_visto_*`), `chat_conexao_verificada` (service_role; edge `chat-conexao-testar`, Onda 2). Readers: `chat_conexoes_listar(p_tenant)` (sem token), `chat_contato_conexao(p_contato)`, `chat_conexao_rotas(p_id)`; service_role: `chat_conexao_por_token(p_tenant, p_token)`, `chat_conexoes_oficiais_sync(p_tenant)` (devolvem `base_url` efetiva; a segunda devolve token), `chat-webhook`/`chat-enviar` (tabela direto). **Front (Onda 3, só RPC — nada lê a tabela direto, token nunca chega ao navegador):** `src/lib/data/chatConexoes.ts` chama `chat_conexoes_listar` (`useChatConexoes`, reexportado por `chat.ts`), `chat_conexao_salvar` (`useSalvarConexao`), `chat_conexao_status` (`useDefinirConexaoAtiva`), `chat_conexao_token` (`useRevelarToken`, sob clique) e `chat_conexao_rotas` (`useConexaoRotas`); consumidores `components/chat-config/CardConexoes.tsx`, `FormularioConexao.tsx`, `DialogoCriarConexaoQr.tsx`. Edge functions chamadas de lá: `chat-conexao-testar` (`useTestarConexao`; grava verificação via `chat_conexao_verificada`) e `chat-conexao-criar` (`useCriarConexaoWhaticket`/`useQrConexao`). Os hooks antigos `useCriarConexao`/`useAtualizarConexao` (PostgREST direto em `chat.ts`) e `CardNumeros.tsx` foram removidos.
- **chat_contatos.conexao_token** — casa por VALOR com `chat_conexoes.token`; `chat_conexao_salvar` reescreve ao trocar o token; `chat-historico-importar` grava o token da conexão nos contatos importados (e preenche quando estava nulo; nunca sobrescreve outro token). **chat_contatos.ultima_recebida_em** — writers: `chat-webhook` (Onda 2) e `chat-historico-importar` (máximo entre banco e lote); backfill na migration; reader: `chat_contato_conexao` (janela 24h).
- **chat_conexoes.historico_importado_em / historico_lote_em / historico_resumo** (migration `20261103130000`) — writer: só `chat-historico-importar` (service role; `historico_lote_em` é o lock com heartbeat de 5 min). Nenhum reader no front ainda (`chat_conexoes_listar` não devolve). Front: `useImportarHistorico` (`chatConexoes.ts`) → `DialogoImportarHistorico.tsx` / `CardConexoes.tsx`, laço por `proximo_cursor`; invalida `["chat","contatos"]` e a lista de conexões ao terminar.
- **chat_provedor_rota** — seed na migration (whaticket × oficial × texto_livre/template/midia); leitura authenticated; reader: `chat_conexao_rotas`, edge functions (referência).
- **chat_templates** — writer: só service_role (`chat_template_upsert` no sync; edge `chat-template-criar` grava `origem='plataforma'`, `painel_template_id`, `submetido_em`). Writers (edge functions, service_role): `chat-templates-sync` (espelha `/api/whatsapptemplates`; também por cron), `chat-template-criar` (criar/submeter no Painel; `acao: testar_painel`/WABAs). Readers: RLS select por membro; front `src/lib/data/chatTemplates.ts` via `chat_templates_listar(p_conexao)` (`useTemplatesConexao` → `chat-templates/TemplatesConexao.tsx`), `chat_templates_aprovados(p_contato)` (`useTemplatesAprovados` → `SeletorTemplates.tsx`) e `chat_contato_conexao(p_contato)` (`useContatoConexao` → `pages/admin/Chat.tsx`, `ChatDetailsSheet.tsx`: conexão oficial + janela 24h). Envio: `chat-template-enviar` (`useEnviarTemplate`).
- **chat_mensagens.template_nome / template_componentes** — writer: edge `chat-template-enviar` (Onda 2); reader: `useChatMensagens` (`src/lib/data/chat.ts`, no select e no INSERT do realtime) → bolha de template em `pages/admin/Chat.tsx` (campo ausente/null = mensagem normal).
- **409 `janela_24h_fechada`** — writer: edge `chat-enviar` (conexão oficial fora da janela); reader: `useEnviarMensagem` lança `ChatTemplateErro` (`src/lib/data/chatErros.ts`) com o `codigo`; `Chat.tsx` abre o seletor de templates via `ehErroJanelaFechada`.
- **chat_config.painel_email + chat_config_segredos** — writer navegador só `chat_config_definir_painel` (admin); reader `chat_config_painel_status` (sem segredo; front: `usePainelStatus` único em `chatConexoes.ts`, reexportado por `chatTemplates.ts`, chave `["chat","painel",tenant]`); service_role (edge `chat-template-criar`) lê senha e grava `painel_jwt`/`painel_jwt_em`.

---

## Notificações push (Web Push) — port do Carreta Oftalmo (ARCHITECTURE §22.17; não aplicado)

### push.fila / push.entregas / push.execucoes — fila própria do push

- **Writers:**
  - `push.enfileirar_evento` / `push.enfileirar` / `push.enfileirar_campanha` — chamados por gatilhos `SECURITY DEFINER` dos módulos (nenhum ainda; exemplo em `docs/push/APLICAR.md` §9) e por `api.push_testar_aparelho` (notificação de teste)
  - Edge Function `supabase/functions/push-processador/` (`banco.ts`): `push.adotar_execucao`, `iniciar_residente`, `reservar_lote`, `renovar`, `resolver_destinatarios` (cria `entregas`), `entregas_validas` (descarta/expira), `finalizar`, `devolver_reservas`, `encerrar_execucao` — conexão postgres.js direta (`SUPABASE_DB_URL`)
  - `push.sinalizar` (gatilho adiado `fila_sinalizar_commit` e cron) → INSERT em `execucoes` + `net.http_post`
  - cron `kreators-push-recuperar` → `push.recuperar()` (reservas/execuções vencidas, expiração, reservas excedidas)
  - `api.push_inbox_marcar_lida` / `_marcar_todas_lidas` / `_arquivar` / `_restaurar` (só `lida_em`/`arquivada_em` do próprio usuário); `api.push_confirmar_exibicao` / `api.push_registrar_clique` (anon, token por entrega)
- **Readers:** `api.push_inbox_listar` / `api.push_inbox_nao_lidas` (front `src/lib/data/push.ts`), view `push.painel_adocao` (via `api.push_painel_adocao`, só system admin)
- **Regra:** o processador só sai por `push.encerrar_execucao`; `finalizar` recebe todas as entregas de uma notificação numa chamada; estado final de fila/entregas/execuções é imutável (gatilho `23514`).

### push.catalogo / push.preferencias / push.inscricoes

- **Writers:** catálogo só por migração (nasce vazio); `api.push_preferencia_salvar` / `api.push_preferencia_restaurar_padrao`; `api.push_inscrever` / `api.push_desinscrever` / `api.push_desinscrever_outros`; `push.finalizar` (desativa inscrição em 404/410, `ultimo_sucesso_em`)
- **Readers:** `api.push_catalogo_meu`, `api.push_aparelho_status`, `api.push_aparelhos_resumo`, `push.resolver_destinatarios`, `push._motivo_fila`, `push.enfileirar_evento`

### Dependências do push em tabelas de OUTROS módulos (somente leitura)

| Objeto lido | Onde | Para quê |
|---|---|---|
| `core.tenant_users` | `push._usuario_tem_acesso`, `_tem_feature`, `_motivo_fila`, `enfileirar_evento`, `enfileirar_campanha`, `painel_adocao`, `api.push_inscrever`, `api.push_testar_aparelho` | quem é membro (destinatário) e o papel (owner/admin vs member/viewer) |
| `core.features` (`code`, `module`) | gatilho `catalogo_validar_permissao`, `_motivo_fila` (`permissao_inativa`), `api.push_catalogo_meu` (`module_key`/`grupo`) | `permissao_ref` do tipo e agrupamento da tela |
| `core.tenant_features`, `core.user_features` | `push._tem_feature` | mesma regra de `core.has_feature_access` (D-PUSH-2) |
| `core.system_admins` | `push._tem_feature`, `api.push_painel_adocao`, `api.push_inscrever`, `api.push_testar_aparelho` | acesso total a features; painel de adoção |
| `core.tenants` (`is_active`, `archived_at`) | `_usuario_tem_acesso`, `_motivo_fila` (`tenant_inativo`), `enfileirar_evento` | tenant ativo (push) |
| `core.tenants` (`is_active`) | `app_auth.assert_tenant_access` (lança `tenant_inactive` para membros de tenant inativo; system admin passa) | gate UI/RPC — R64 |
| `core.tenants` (`is_active`) | `worker.claim_jobs`, `integrations.claim_jobs` (EXISTS sublink) | gate fila de ingestão — R64 |
| `core.tenants` (`is_active`) | `journeys.claim_next_jobs`, `journeys.claim_next_job` (EXISTS sublink) | gate fila de jornadas/email — R64 |
| `core.tenants` (`is_active`) | `analytics.claim_next_jobs` (EXISTS sublink) | gate fila de analytics — R64 |
| `core.tenants` (`is_active`) | `analytics.claim_next_segment_jobs_fair`, `analytics.claim_next_segment_jobs` (EXISTS sublink) | gate fila de segment eval — R64 |
| `auth.users` (`deleted_at`, `banned_until`) | `push._usuario_ativo` | usuário ativo |
| `app_auth.assert_tenant_access` | wrappers `api.push_*` com `p_tenant_id` | R56 |
| Vault `service_role_key` (e opcional `push_processador_segredo`) | `push._chamar_processador` | Bearer do sinal pg_net → Edge Function |

FKs com `on delete cascade` para `core.tenants(id)` e `auth.users(id)` (preferências, inscrições, fila, campanhas): apagar tenant/usuário limpa o push. Mudança de semântica em qualquer linha acima exige revisar `push._tem_feature`/`_usuario_ativo` e rodar `scripts/push/ambiente-teste/fumaca.sql`.

---

## Inbox (Nylas/IMAP) — outbox, labels e pastas

### inbox.ops — Outbox local-first (FIFO por conta, ver ARCHITECTURE §22.15)

- **Writers:**
  - `api.enqueue_message_op` RPC — frontend `src/lib/inboxStorage.ts` (`updateMessageFlags`, `moveMessageToFolder`): INSERT do op + mutação otimista + job drenador na mesma transação
  - `api.create_folder` / `api.rename_folder` / `api.delete_folder` RPCs — frontend `src/lib/inboxStorage.ts` + `src/components/inbox/InboxSidebar.tsx`: folder ops (delete com mensagens gera ops `move` em massa ANTES do `folder_delete` — FIFO por `seq`)
  - `api.retry_folder_op` / `api.discard_folder_op` RPCs — badge "falhou" da sidebar (re-enfileira op failed / cancela e reverte estado local)
  - `workers/email-sync/src/handlers/applyMessageOps.ts` + `folderOps.ts` — UPDATE de status (applying/applied/failed/cancelled, backoff, rollback com cascata)
- **Readers:**
  - email-sync-worker handler `nylas:apply_message_ops` — drena FIFO (`ORDER BY seq`): coalescing por mensagem; folder ops como barreiras individuais
  - `src/lib/inboxStorage.ts` `getOutboxStatus` — chip "Sincronizando N…" (SELECT count, RLS read-only)
  - `inbox.upsert_message` — blindagem de campo: op pending/applying vence sync/webhook
- **Regra:** frontend NUNCA muda status de op (só INSERT via RPC + SELECT); status é exclusivo do worker/RPCs de retry-discard.

### inbox.labels / inbox.message_labels — labels 100% locais

- **Writers:** RPCs `api.create_label` / `update_label` / `delete_label` / `set_message_labels` — frontend `src/lib/inboxStorage.ts` (sem outbox, sem worker, sem sync IMAP)
- **Readers:** `api.get_labels_with_counts` (contagem on-demand — nunca materializar), `api.get_inbox_messages_v2` (filtro `p_label_id`), SELECT direto em `inbox.message_labels` (picker/lista)
- **Regra:** nunca sincronizar labels com IMAP (decisão §8.1 do INBOX_REVAMP_PLAN).

### crm.get_contact_context (RPC — painel CRM do inbox)

- **Reader:** frontend `src/lib/inboxStorage.ts` `getContactContext` → `ContactContextPanel` (via wrapper `api.get_contact_context`, 1 round-trip jsonb)
- **Fontes lidas:** `crm.parties`/roles/tags, `analytics.contact_state`, `commerce.transactions`, `journeys.journey_enrollments`, `crm.opportunities`

### inbox.link_message_to_crm (reescrita 2026-08-27)

- **Callers:** `inbox.upsert_message` (todo sync/webhook de mensagem) e `inbox.batch_link_messages_to_crm`
- **Regras:** customer supersede lead (badge não fica stale após conversão); o endereço da PRÓPRIA conta nunca linka (recebido → casa só o remetente; enviado → casa destinatários exceto a conta); leads via `crm.parties` + role ativa (`crm.leads` não existe mais)

### Handlers email-sync `nylas:apply_message_ops` / `nylas:sync_folders`

- **Producers:** ver "Producers (RPCs SECURITY DEFINER)" em `integrations.jobs`; ambos com dedupe de 1 job pending/running por conta
- **`apply_message_ops`** (`applyMessageOps.ts` + `folderOps.ts`): escreve em `inbox.ops`, `inbox.messages` (remap de folders pós-rename, restore em rollback), `inbox.folders` (adota nylas_folder_id no create, remap, hard-delete no delete confirmado, `sync_status='error'` em falha permanente); enfileira `nylas:sync_folders` após folder op aplicada
- **`sync_folders`** (`syncFolders.ts`): reconcilia `inbox.folders` com GET /folders — blindagem (linha com `sync_status != 'active'` intocada), soft-delete com mensagens MANTIDAS, eleição canônica SPECIAL-USE (1 pasta por papel de sistema), `parent_id` derivado do path; atualiza `inbox.accounts.last_folder_sync_at`

---

## Domínios Customizados

### core.custom_domains

- **Writers:**
  - `src/lib/customDomains.ts` — CRUD via RLS (add/toggle/delete pelo `DomainSettingsDialog`)
  - `supabase/functions/domain-delegation/index.ts` (`handleValidate`) — upsert automático do `forms.<fqdn>` com `status='active'` quando a delegação NS valida
  - `supabase/functions/domain-delegation/index.ts` (`handleRecheckAllDomains`/`handleRecheckDomain`) — re-verificação de DNS: seta `dns_broken` (2 falhas consecutivas no cron; imediato no check manual), restaura `active`, mantém `last_dns_check_at`/`dns_failure_count`
- **Readers:**
  - `src/lib/customDomains.ts` (`getActiveDomainByService`) — links de forms/lp/app na UI (FormBuilder, FormsDashboard, SmartFormsDashboard, EmbedFormBuilder, CheckoutOffers). Filtra `status='active'`
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts` — URL do preference center (`app.%`, `status='active'`)
- **Status:** `pending` → `active` (verificado) | `failed` (verificação inicial falhou) | `disabled` (desligado pelo usuário — nunca reativado automaticamente) | `dns_broken` (estava ativo, cron `custom-domains-recheck` não encontra mais o DNS; volta a `active` sozinho quando o registro reaparece)
- **Regra:** `dns_broken` REMOVE o domínio dos readers acima (filtram `active`) — links gerados voltam ao domínio da plataforma até o DNS ser restaurado. CNAME esperado: `cname.vercel-dns.com` (constante `VERCEL_CNAME` no Edge Function; espelhada em `DomainSettingsDialog.tsx`).

### core.delegated_subdomains

- **Writers:**
  - `supabase/functions/domain-delegation/index.ts` — ciclo de vida completo (create/validate/delete); re-check NS periódico atualiza `observed_nameservers` e flaga `metadata.dns_broken`/`metadata.ns_failure_count` sem tocar `status`
- **Readers:**
  - `src/components/settings/DomainDelegationTab.tsx` via action `status` — UI de delegação (alerta destrutivo quando `metadata.dns_broken`)
- **Regra:** o state machine de `status` (`pending_delegation` → `validating` → `active`) pertence ao fluxo de onboarding; saúde pós-ativação vive em `metadata` para não regredir a UI de setup.

---

## Casca e navegação (frontend mobile, ARCHITECTURE §25, 2026-10-21)

### `src/config/nav-config.ts` — registro único de navegação

Arquivo puro (sem React/Supabase). Mudar rótulo, ordem, `access`, `ROUTE_META` ou `resolveTabs` afeta todos os
leitores abaixo; o teste de equivalência da sidebar tem que continuar passando.

| Leitor | O que consome |
|---|---|
| `src/components/layout/AppLayout.tsx` | `getNavGroups` (sidebar desktop/tablet), `resolveTabs` (abas da BottomNav), `matchRouteMeta` (`layout`, `hideBottomNav`), `getParentPath` (passado ao `ShellProvider`), `isNavItemActive`, `isTabRoot` |
| `src/components/shell/bottom-nav.tsx` | `getActiveTabId`, `MORE_PATH`, `TabItem`; badges por `NavItem.badge` (`useNavBadges`) |
| `src/components/shell/mobile-more-menu.tsx` + `src/pages/Mais.tsx` (`/mais`) | `getMobileMoreSections`, `resolveTabs`, `getActiveNavItem` |
| `src/components/shell/command-palette.tsx` | `NAV_ITEMS`/`ACCOUNT_NAV_ITEMS` com `keywords` (busca ⌘K) |
| `src/components/shell/route-header.tsx` | `getRouteTitle`, `getParentPath`, `getParentLabel` (header automático sem `PageHeader`) |
| `src/App.tsx`, `src/components/auth/FeatureRoute.tsx` | `matchRouteMeta` (skeleton da rota durante o carregamento) |
| `src/lib/home-route.ts` | tipo `FeatureCtx` (`useFeatureCtx`, `getHomeRoute`/`useHomeRoute`: home do `/`, `/app` e do voltar) |
| `src/lib/recent-routes.ts`, `src/lib/shell/recent-nav.ts` | `matchRouteMeta`, `getRouteTitle`, `getActiveNavItem`, `getNavItemById`, `isTabRoot`, `resolveTabs`, `NAV_ITEMS`/`ACCOUNT_NAV_ITEMS`, `MORE_PATH` (Recentes do `/mais`) |
| `src/lib/shell/conversation-badges.ts` | tipo `NavItem["badge"]` |
| `src/pages/Upgrade.tsx` | `NAV_ITEMS`, `getRouteTitle` (nome do recurso bloqueado) |
| `src/pages/admin/AccountBilling.tsx` | `MORE_PATH` (voltar) |
| Testes | `src/config/__tests__/nav-config.test.ts` (equivalência da sidebar), `src/lib/__tests__/home-route.test.ts`, `src/lib/__tests__/recent-routes.test.ts` |
| Bancada | `src/pages/dev/kit/casca.demo.tsx` |
| Script | `scripts/check-overflow.ts` (paths de `NAV_ITEMS`/`ACCOUNT_NAV_ITEMS` e `ROUTE_META` sem parâmetro; `hideBottomNav` = editor, fora por padrão) |

### Variáveis CSS da casca (gravadas em `<html>`)

| Variável | Writer | Readers |
|---|---|---|
| `--shell-top` | `ShellProvider` (`shell/shell-context.tsx`) | `index.css`, `shell/full-bleed-page.tsx`, `ui/{search-input,responsive-tabs,filter-bar,data-list}.tsx` (sticky sob o header), `layout/{AppLayout,SystemLayout}.tsx`, `system/DocLayout.tsx`, `contacts/detail/ContactDetailScreen.tsx`, `pages/admin/{Transactions,CampaignResults,Opportunities}.tsx` |
| `--shell-bottom` | `ShellProvider`; `ui/bottom-action-bar.tsx` enquanto registrada | `index.css`, `ui/{sonner,fab,bottom-action-bar}.tsx`, `shell/full-bleed-page.tsx`, `layout/AppLayout.tsx`, `contacts/detail/NotesComposer.tsx` |
| `--app-height`, `--app-offset-top`, `--kb-inset` | `hooks/use-visual-viewport.ts` (`publishCssVars`, montado só no `ShellProvider`) | `index.css`, `shell/full-bleed-page.tsx` (`FullBleedPage`/`ImmersiveScreen`), `conversation/chat-composer.tsx` |
| `--bottom-bar-h` | `ui/bottom-action-bar.tsx` | `index.css`, `shell/shell-context.tsx` |
| `--impersonate-h` | `system/ImpersonateBanner.tsx` (ResizeObserver; evento `impersonate-height-change`) | `layout/SystemLayout.tsx` |

## Kit mobile — primitivas e principais consumidores (ARCHITECTURE §25.3)

Contagem de arquivos de produção que importam a primitiva (sem `/__dev/kit` e testes), 2026-10-21. Mudança de props
ou de comportamento padrão afeta todos; regras M1–M3 valem para qualquer edição.

| Primitiva | Arquivo | Consumidores |
|---|---|---|
| `ResponsiveDialog` | `ui/responsive-dialog.tsx` | 88 arquivos (formulários e diálogos do app inteiro) |
| `DataList` / `ListRow` | `ui/data-list.tsx` | 83 |
| `PageHeader` | `shell/page-header.tsx` | 80 (toda tela com título) |
| `useConfirm` / `ConfirmSwitch` | `ui/confirm-dialog.tsx` (host no `App.tsx`) | 78 |
| `ActionMenu` | `ui/action-menu.tsx` | 77 |
| `StatGrid` / `StatTile` | `ui/stat-grid.tsx` | 27 |
| `StatusScreen` | `ui/status-screen.tsx` | 22 |
| `FilterBar` | `ui/filter-bar.tsx` | 21 (Contatos, Transações, Pedidos, Assinaturas, Campanhas, Jornadas, Segmentação, Inbox, WhatsApp, Chat, Oportunidades, Atividades, Formulários, Analytics...) |
| `BottomActionBar` | `ui/bottom-action-bar.tsx` | 17 (CampaignSend, WhatsAppBroadcast, OpportunityDetail, CheckoutPage, builders em modo celular, HomeEditor...) |
| `EmptyState` | `ui/empty-state.tsx` | 16 |
| `ResponsiveTabs` | `ui/responsive-tabs.tsx` | 13 (ContactDetailScreen, OpportunityDetail, Campaigns, Tags, SegmentationList, BaseRFM, SystemHealth, SystemTenantDetail...) |
| `ChartCard` | `ui/chart-card.tsx` | 9 (Dashboard, Analytics, dashboards de CRM, produtos, saúde do sistema) |
| `FullBleedPage` / `ImmersiveScreen` | `shell/full-bleed-page.tsx` | 8 (AppLayout, Inbox, MessageDetail, Chat, WhatsApp, ChatDetail, Opportunities, LandingPageBuilder) |
| `MasterDetail` | `shell/master-detail.tsx` | 5 (Inbox, Chat, WhatsApp, LookalikePage, PipelineSettings) |

Guarda-corpos que leem o código (não o banco): `eslint.config.js` (regras de §25.5), `src/test/guardrails.test.ts` +
`src/test/guardrails-baseline.json`, `src/dev/overflow-detector.ts` (montado em `src/main.tsx` só em DEV).

## Leitores novos do frontend mobile (Ondas 5–11, somente leitura)

Nenhum cria objeto no banco: são leituras novas ou mais baratas de tabelas que já existiam. Quem mudar coluna, índice
ou RLS destas tabelas precisa conferir estes leitores.

| Leitor | Arquivo | Lê | Como | Consumidores |
|---|---|---|---|---|
| Vendas de hoje | `src/lib/data/sales-today.ts` (`useSalesToday`, `useRecentSales`) | `commerce.transactions` | `status='paid'` e `transaction_date` no dia LOCAL do navegador (`localDayStartISO`/`localDayEndISO`, as mesmas de Transações); páginas de 1000 até 20 (depois `truncated`); somas no cliente; `staleTime` 60 s; chaves `transactions-today`/`transactions-recent` (limpas na troca de tenant) | `components/dashboard/{TodayHero,RecentSales}.tsx` (`/dashboard` "Hoje") |
| Badges da navegação | `src/lib/shell/conversation-badges.ts` (`useConversationsBadge`, `useNotificationsBadge`, `useNavBadges`) | `public.chat_contatos` (`nao_lidas > 0`, `count` com `head: true`); não lidas do push via `usePushInboxNaoLidas` (`lib/data/push.ts`) | só com a feature; sem realtime; `staleTime` 60 s; chave começa com `["chat","contatos"]` (invalidações do Chat atualizam o badge). E-mail e WhatsApp fora (contagem cara) | `shell/bottom-nav.tsx`, `shell/mobile-more-menu.tsx`, `dashboard/PendingCard.tsx` |
| Canal realtime com ressincronização | `src/lib/realtime/use-realtime-channel.ts` (`useRealtimeChannel`) | `inbox.messages` (Inbox), `whatsapp.{messages,chats,instances}` (WhatsApp), `imports.import_runs` (Transactions, TransactionImportHistory), `integrations.accounts` (IntegrationHub), `public.chat_mensagens` (`lib/data/chat.ts`) | invalida as queries informadas ao reconectar (`CHANNEL_ERROR`/`TIMED_OUT`/`CLOSED` → `SUBSCRIBED`), ao voltar do background (> 30 s) e no `online`; recria o canal se caiu | `pages/admin/{Inbox,WhatsApp,Transactions,TransactionImportHistory}.tsx`, `settings/IntegrationHub.tsx`, `lib/data/chat.ts` |
| Estatísticas de atividades | `src/lib/opportunityStorage.ts` (`getActivityStats`) | `crm.opportunity_activities` | 4 contagens no servidor (`count: "exact", head: true`): total, pendentes, atrasadas, concluídas hoje; atividades de CRM do tenant sem automação de jornada. Antes baixava as linhas e cortava em 1000 (PostgREST) | `pages/admin/Activities.tsx`, `dashboard/PendingCard.tsx` (chave `["activity-stats", tenant]`) |
| Analytics de jornada | `src/components/journey-builder/journey-analytics-data.ts` (`fetchJourneyAnalytics`) | `journeys.journeys`, `journeys.journey_enrollments`, `journeys.journey_executions`, `email.campaign_sends` (métricas por envio) | `email.campaign_sends` lido em lotes por `selectInChunks` (`src/lib/supabase-chunk.ts`: 150 ids por lote, 4 lotes simultâneos); antes um `.in()` de ~800 ids estourava a URL (400) | `pages/admin/JourneyAnalytics.tsx`, `journey-builder/JourneyMobileView.tsx` |
| Respostas de formulário paginadas | `src/lib/formStorage.ts` (`getFormResponsesPage`), `src/lib/smartFormStorage.ts` (`getSmartFormResponsesPage`) | `forms.form_submissions` (+ respostas), `smart_forms.form_submissions` | `.range()` de 30 por página (`RESPONSES_PAGE_SIZE`) em `useInfiniteQuery`; formulário descarta parciais sem resposta e emenda até 5 páginas do banco; exportação continua com `getFormResponses`/`getSmartFormResponses` (tudo) | `components/forms/responses/ResponsesView.tsx` via `pages/forms/FormResponses.tsx` e `pages/smart-forms/SmartFormResponses.tsx` |

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

## Platform Health Observability (D-CRON-3)

> **R52** (v7.25): Camada de saúde (view) e camada de alerta (handler) leem APENAS `analytics.event_health_metrics`. Nunca tabelas operacionais (`fallback_log`, `segment_eval_queue`, `contact_events`).
> Antes de adicionar reader/writer aqui: verificar §17.1 do ARCHITECTURE.md e a migration `20260501100000_dcron3_observability.sql`.

### analytics.event_health_metrics — Telemetria normalizada (Camada A)

- **Writers (sempre via RPC `analytics.emit_health_metric`):**
  - `analytics.etl_event_health_metrics()` — pulse hourly (cron `etl-event-health-metrics`). Agrega `analytics.fallback_log` + `analytics.contact_events` + `analytics.segment_eval_queue` em 5 métricas (`fallback_hits`, `event_driven_hits`, `eval_queue_throughput`, `eval_queue_error_rate`, `eval_queue_p95_latency_ms`).
  - `workers/analytics/src/segment-eval-worker.ts:flushHealthMetricBatch` — flush 60s acumula `segment_membership_changes` em memória e emite via `emit_health_metric` (primeiro caso de worker emitting telemetry direto).
- **Readers (R52 — apenas via Camada B):**
  - `analytics.v_event_driven_health` — view de tier classification. Único reader operacional em produção.
- **Schema:** `PARTITION BY RANGE (bucket_hour)` mensal. PK `(tenant_id, metric_name, bucket_hour)`. RLS service-only.
- **Indexes:** `event_health_metrics_metric_bucket_idx (metric_name, bucket_hour DESC)`, `event_health_metrics_tenant_metric_idx (tenant_id, metric_name, bucket_hour DESC)`.
- **Lifecycle:** `analytics.event_health_metrics_partition_create_if_needed()` cria próxima partição mensal (cron dia 25 às 02:30); `analytics.event_health_metrics_purge_old(retention_days)` faz DROP partition de >90d (cron daily 04:30).

### analytics.platform_alert_state — Dedup state (Camada C)

- **Writers:**
  - `workers/journeys/src/handlers/healthCheck.ts:runEventDrivenHealthCheck` — UPSERT `last_tier`, `last_changed_at`, `last_alert_at` por `(tenant_id, metric_name)`.
- **Readers:**
  - `workers/journeys/src/handlers/healthCheck.ts:shouldAlert` — decide se emite Sentry (transição p/ tier pior, dedup 1h em mesmo tier ≥ ERROR, recovery silencioso 30min).
- **Schema:** PK `(tenant_id, metric_name)`. RLS service-only. Index `platform_alert_state_tier_idx` em `last_tier` para query de dashboard "tenants em ERROR/CRITICAL agora".

### analytics.v_event_driven_health — View de saúde (Camada B)

- **Source:** `analytics.event_health_metrics` (rolling 7d P50/P95/P99 per-tenant excluindo a hora atual — evita auto-inflação).
- **Readers:**
  - `workers/journeys/src/handlers/healthCheck.ts:runEventDrivenHealthCheck` — único reader em produção (5min cadence dentro do `runSchedulerLoop`/`runCombinedLoop`).
  - Possíveis dashboards admin futuros (não implementados em v7.25).
- **Tier output:** `HEALTHY`, `INSUFFICIENT_DATA` (< 168 samples warmup), `WARN` (`hits_1h > p50 * 1.5`), `ERROR` (`hits_1h > p95`), `CRITICAL` (`error_sustained_1h AND hits_1h > p99`).
- **Whitelist:** `LEFT JOIN core.platform_health_whitelist` exclui tenants nominalmente listados.

### core.platform_health_whitelist — Tenants excluídos do alerta

- **Writers (apenas manual via migration):** INSERT nominal com `reason` (CHECK in `seed_test`, `demo_internal`, `paused_investigation`, `opcit`), `added_by`, `expires_at`, `notes`. Auditável.
- **Readers:**
  - `analytics.v_event_driven_health` — filtro embutido via LEFT JOIN.
- **Schema:** PK `tenant_id` com FK ON DELETE CASCADE → `core.tenants(id)`. RLS service-only. Começa vazia em v7.25.

### analytics.emit_health_metric() — RPC de UPSERT (escrita única)

- **Callers (Camada A producers):**
  - `analytics.etl_event_health_metrics()` — cron hourly.
  - `workers/analytics/src/segment-eval-worker.ts:flushHealthMetricBatch` — worker batch 60s.
  - **Futuros workers em D-CRON-4** (substituto do `segment-eval-fallback`) — emitirão `fallback_hits` direto, sem cron intermediário.
- **Behavior:** SECURITY DEFINER. ON CONFLICT `(tenant_id, metric_name, bucket_hour)` DO UPDATE — idempotente para retries.
- **Regra R52:** Único caminho válido de escrita em `event_health_metrics`. INSERT direto = violação.

### workers/shared/src/sentry.ts — Helper compartilhado

- **Callers:**
  - `workers/journeys/src/index.ts` — `initSentryWorker` no startup, `flushSentry` no shutdown. Primeiro consumer.
  - Workers futuros podem importar `captureWorkerException` para alertar incidentes.
- **Behavior:** No-op silencioso se `SENTRY_DSN_WORKERS` ausente. Importação dinâmica de `@sentry/node` — workers que não chamam Sentry não pagam custo de bundle.
- **Env vars:** `SENTRY_DSN_WORKERS`, `SENTRY_ENVIRONMENT`, `RELEASE_SHA` (todos opcionais — sem DSN, handler escreve em `platform_alert_state` + log e segue).

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


## core.tenant_billing (Platform Billing)

Vínculo tenant ↔ cliente Asaas da plataforma (conta do tenant Kreators Platform).

| Papel | Quem |
|---|---|
| Writer | edge fn `platform-billing` (service role): `admin_link_customer` (upsert), `admin_unlink_customer` (is_active=false) |
| Reader | edge fn `platform-billing`: `generate_share_link`, `get_billing`, `get_payment_artifacts` |
| Frontend | `src/lib/platformBilling.ts` → `TenantBillingTab` (/system/tenants/:id), `AccountBilling` (/admin/account), `PlatformBillingPage` (/cobranca/:token) |

Leitura adicional: `platform-billing` lê `commerce.asaas_config` e `commerce.checkout_orders` APENAS do tenant `PLATFORM_TENANT_ID` (credencial + sugestão de vínculo).


## api.get_account_overview (Minha conta — Visão geral, 2026-10-28)

Leitura só (migration `20261028120000_account_overview_rpc`). Nenhum writer.

| Papel | Quem |
|---|---|
| Lê | `core.tenants` (id, name, slug, created_at), `core.tenant_plans` + `core.plans` (plano vigente), `email.tenant_config` (SÓ `monthly_send_limit`/`daily_send_limit`), `core.tenant_users` (por role, sem system admins), `core.team_invitations` (pendentes), `email.campaign_sends` (envios), `core.tenant_features` (ativas) |
| Frontend | `src/lib/accountOverview.ts` → `src/pages/admin/AccountBilling.tsx` (/admin/account, aba Visão geral) |

Mudou coluna/semântica de qualquer tabela acima (ex.: limites novos em `core.plans`) → revisar a RPC e o contrato do front.

## Fila de jornadas — espera por pré-condição (Fase 0, 29/10/2026, em produção)

Contrato completo em `docs/JOURNEY_ISOLATION.md`. **Antes de mudar:** status enviáveis de campanha, valores de `wait_reason`, chave de dedupe `enrollment_<id>` ou a semântica de `next_execution_at` → revisar todos os readers e writers abaixo.

**`journeys.journey_enrollments.wait_reason` / `wait_ref_id` / `waiting_since`**

| Papel | Quem |
|---|---|
| Writers (espera) | `journeys.apply_enrollment_blocker`, chamada por `enqueue_ready_waits`, `sweep_orphan_enrollments` e pela trigger `trg_execution_campaign_not_ready` (AFTER INSERT em `journey_executions`, `send_email` + `campaign_not_ready`) |
| Writers (limpeza) | `journeys.release_campaign_waits`, chamada pela trigger `trg_campaign_release_journey_waits` e pela rede de segurança do `enqueue_ready_waits`; `journeys.fail_campaign_waits`, chamada pela trigger `trg_campaign_delete_journey_waits` e pelo cron; trigger `trg_enrollment_clear_stale_wait`, quando o worker avança de nó ou encerra a inscrição |
| Readers | `enqueue_ready_waits` e `sweep_orphan_enrollments` (ignoram quem tem espera); `journeys.enrollment_blocker` (diagnóstico); futuro "agora nesta etapa" do canvas (Fase 1) |

**Portão `journeys.evaluate_enrollment_blocker`**
- Lê `journeys.journeys.status`, o nó atual (versão congelada → grafo vivo) e `email.email_campaigns.status`.
- O worker `journeys` (`processJourneyNode.ts`) mantém a própria lista `['ready','sent','automation']`. As duas listas precisam ser iguais.

**`email.email_campaigns.status`** (reader novo)
- As triggers `trg_campaign_release_journey_waits` (AFTER UPDATE OF status, WHEN inline) e `trg_campaign_delete_journey_waits` (AFTER DELETE) agora reagem a mudanças nesse campo.
- Quem escreve: `CampaignEditor.tsx` (Salvar → `ready`), `CampaignSend.tsx` ("Salvar para Automação" → `automation`), `sendEmailBatch` (`ready` → `sent`) e `Campaigns.tsx` (DELETE físico).

**`journeys.journeys.status`** (reader novo)
- As triggers `trg_journey_cancel_pending_jobs` e `trg_journey_delete_cancel_pending_jobs` disparam ao sair de `active`.
- Elas cancelam os `process_enrollment` pendentes, mas retêm os de entrada, que vão para `paused`/`draft` e são cancelados ao arquivar ou excluir.
- O claim também filtra `process_enrollment` de jornada não ativa.

**`journeys.processing_jobs`**
- O claim (`claim_next_jobs`/`claim_next_job`) é compartilhado com o worker `email-campaign`, com a mesma assinatura e retorno.
- A reserva anti-inanição vale por `job_type`.
- A dedupe de `process_enrollment` usa a chave única `enrollment_<id>`, nos produtores `enqueue_ready_waits` e `sweep_orphan_enrollments`.

**Permissões**
- Todas as funções de `journeys` só têm EXECUTE para `postgres` e `service_role`.
- Exceção: `journeys.safe_timestamptz`, usada pela trigger SECURITY INVOKER `trg_journey_status_lifecycle` na ativação pelo navegador.
- Ver R63.

---

## Redes sociais (Windsor.ai) — ARCHITECTURE §14.3 (Onda 1, aplicado em produção em 2026-11-03)

### integrations.social_accounts
- **Writers (só service role):** edge `social-accounts-sync` (insert, ou update da linha do mesmo tenant para `active` com `connected_by`/`connected_at`; no fluxo co-user também `connect_link_id`, `windsor_datasource`, `metadata.windsor_co_user`), edge `social-disconnect` (`status='disconnected'`, `error_message` e `metadata.windsor_unlink` com o resultado do DELETE na Windsor), seed `supabase/scripts/social_seed_maconequi.sql`. Nenhum writer `authenticated` (sem policy e sem GRANT de escrita).
- **Readers:** `api.social_accounts_list(p_tenant_id)` (aba Configurações › Redes sociais, `src/components/settings/SocialAccountsTab.tsx` via `src/lib/data/socialAccounts.ts`, query key `["social-accounts", tenantId]`; status da aba em `settings-sections.tsx`; e a edge fn `social-insights` que alimenta a UI `/admin/social`, ver subseção abaixo), RLS select por `is_tenant_member`, edges `social-accounts-sync` (contas `active` de todos os tenants por connector, para `conta_em_uso`; linhas do tenant para reativar) e `social-disconnect` (`windsor_datasource`, `connect_link_id`, `metadata`).
- **Chamadores das edge fns:** só `src/lib/data/socialAccounts.ts` (`useSocialConnectUrl` → `social-connect-url`, `useSocialAccountsSync` → `social-accounts-sync`, `useSocialDisconnect` → `social-disconnect`); o estado "aguardando autorização" (`link_id` + `connect_url`) fica em `sessionStorage` até `expiraEm` (≤ 4 dias, prazo do link na Windsor).
- **Sem writer ainda:** `last_sync_at`, `status='error'` (reservados para a coleta de métricas).
- **Reader novo (2026-11-04, migration `20261104160000`, NÃO aplicada):** RPC `integrations.social_report_contexto` (contas `active` do conector, `connected_at desc nulls last, id asc`), chamada pelas edges `social-insights` e `social-ads-insights` no lugar da query direta.

### schema `ads` — Ads Intelligence (2026-11-04/05, migrations `20261105100000`–`20261105100500`, APLICADAS; ARCHITECTURE §22.19)
- **Writers (só service role, via RPC `ads.*` — nenhuma policy de escrita para `authenticated`):** edge `ads-sync` (`_shared/ads/sync.ts`): `ads.ensure_account` (de `integrations.social_accounts`), `ads.update_account_info` (moeda/fuso/nome, só quando vazios), `ads.upsert_entities` (`campaigns/adsets/ads`), `ads.upsert_metrics_daily` (`metrics_daily`; sobrescreve o dia inteiro; marca `restated_at`), `ads.start_sync_run`/`finish_sync_run` (`sync_runs`, `accounts.last_sync_at`), `ads.enqueue_sync`/`enqueue_backfill`/`enqueue_daily_all`/`requeue_sync_job`/`release_stale_sync_jobs` (**`integrations.jobs`**, handler `ads:sync` — writer novo da fila; índice parcial `idx_jobs_handler_ads_account_active`), `ads.run_data_quality` → `data_quality_runs/checks/incidents` + `insights` `dado_*`; trigger `metrics_daily` → `metrics_daily_history`. Front (authenticated, R56): `api.ads_recommendation_set_status` (`recommendations` + `recommendation_events`), `api.ads_insight_set_status` (`insights`), `api.ads_account_goals_set` (`accounts` metas/`automation_mode`), `api.ads_setting_set` (`settings`, owner/admin), `api.ads_data_quality_run_now`.
- **Readers:** RPCs `api.ads_accounts`, `ads_sync_status`, `ads_insights`, `ads_insight`, `ads_recommendations`, `ads_recommendation_events`, `ads_data_quality_incidents`, `ads_data_state`, `ads_settings`, `ads_overview`, `ads_metrics_series`, `ads_campaigns_overview`, `ads_funnel` ← `src/lib/data/adsIntelligence.ts` (hooks `useAdsOverview`, `useAdsSeries`, `useAdsCampaignsOverview`, `useAdsInsights`, `useAdsRecommendations`, `useAdsIncidents`, `useAdsAccounts`) ← `src/components/ads-intelligence/*` e `src/pages/admin/Inteligencia.tsx` (`/admin/inteligencia`). Edge `ads-sync` lê `ads.accounts` + `integrations.social_accounts` (`carregarConta`) e `worker.claim_jobs(p_handlers => '{ads:sync}')`. Crons: `ads.invoke_sync` lê `integrations.jobs` (guarda) ; `ads.invoke_enqueue_daily` → edge.
- **Contratos acoplados (mudar juntos):** (1) colunas de `ads.metrics_daily` = nomes dos campos da Windsor em `REPORTS_ADS` (`_shared/socialAds.ts`) + listas `FIELDS_DIARIOS` em `metaWindsorAdapter.ts` — campo novo na ingestão exige coluna nova; (2) payload do job `{ads_account_id, kind, date_from, date_to}` entre `ads.enqueue_sync` e `ads-sync/pedido.ts`; (3) JSON de `api.ads_overview`/`ads_funnel`/`ads_insight` e os tipos em `adsIntelligence.ts`; (4) `ads.data_state` chamada por `api.ads_overview` e `api.ads_data_state`; (5) chaves de `ads.settings` lidas por `ads.dq_*` e `api.ads_campaigns_overview` (`ux.piorou_cpa_pct`); (6) estados (`data_state`, status de insight/recomendação/ação) replicados como rótulos pt-BR no front.
- **Fora de `ads` que o módulo lê:** `commerce.transactions` (`currency`, `currency_local`, `status` text, `paid_at`, `amount`, `tenant_id`) em `dq_moeda_divergente` e, na F2, na atribuição; `core.tenants.is_active` (R64) em `enqueue_daily_all`, `invoke_sync`, modos manual/connect; `core.tenant_users.role` em `ads_setting_set`.
- **F5 (migration `20261105101100` + edges `ads-actions`/`ads-chat`):** writers front → `api.ads_action_propose/approve/cancel/revert_request`, `api.ads_recommendation_approve` → `ads.actions`, `ads.action_events`, `integrations.jobs` (`ads:action`); edge `ads-actions` → `ads.action_claim_start/record_before/finish/mark_reverted`, `ads.release_stale_actions` → `ads.actions`, `ads.action_outcomes`; `ads.evaluate_outcomes` → `action_outcomes`; edge `ads-chat` → `ads.chat_thread_upsert`, `chat_message_append`, `usage_add` → `ads.chat_threads/messages`, `usage_daily`; `api.ads_chat_memory_set` → `ads.chat_memory`; `api.ads_actions_execution_toggle` (system admin) → `ads.settings` `actions.execucao_habilitada`. Readers: `api.ads_actions`, `ads_action`, `ads_action_catalog`, `ads_outcomes`, `ads_actions_execution_status`, `ads_chat_threads/thread`, `ads_chat_memory_get` ← front; `ads.action_guardrails` lê `accounts.automation_mode`, `campaign_tags` (`protegida`), `ads.automation_allowed`, `ads.agent_blocked`, `ads.actions` (cooldown/conflito), `campaigns/adsets.budget_daily`; `ads.chat_context` lê contas, metas, tags, cobertura, estado dos dados, insight/recomendações, últimas ações; as 18 tools do chat leem as mesmas `api.ads_*` do front com o JWT do usuário. Contratos: catálogo `action_type` fechado (5 tipos) ↔ `ads-actions` (`provider_ops`) ↔ `ads-chat` (`propor_acao`) ↔ front; mapeamento de `action_finish` (p_error/p_verified → status); `RPC_CHAT` em `_shared/ads/chatStore.ts` espelha os nomes das RPCs de chat.
- **F4 (migration `20261105101000` + edge `ads-creatives`):** writers `ads.detect_campaign_tags` → `ads.campaign_tags` (dentro de `ads.det_run`; manual via `api.ads_campaign_tag_set`), `ads.attribute_launch_cohort` → `ads.attributed_events` (`model` `coorte_lancamento`), `api.ads_launch_upsert/delete` → `ads.launches`, edge `ads-creatives` → `ads.creative_tags` (`ads.upsert_creative_tags`; `api.ads_creative_tag_review` para correção). Readers: `api.ads_creatives`, `ads_creatives_leaderboard`, `ads_creative_brief`, `ads_campaign_tags`, `ads_launches`, `ads_launch_overview`, `ads_account_structure`, detectores F4 (`ads.det_fadiga_criativo` lê `metrics_daily` nível ad + `creative_tags`; `det_baixa_diversidade` lê `creative_tags.concept`; `det_alteracoes_frequentes_escala` lê `ads.actions` + `recommendations`), `ads.agent_blocked` (F5), `api.ads_overview.por_tag`. Contratos: taxonomia fechada em `creativeTaxonomy.ts` ↔ valores aceitos em `ads.creative_tags` (check) ↔ rótulos do front; `tags.padroes` (regex ARE) ↔ `ads.detect_campaign_tags`; `creatives.model_version` ↔ `creatives_pending_tagging`.
- **Digest e alertas (F3, migration `20261105100800`):** writers edge `ads-digest` → `ads.digests` (`digest_upsert`, `digest_mark_delivery`), `push.fila` via `ads.digest_push_enfileirar` → `push.enfileirar` (catálogo `ads_resumo_diario`, seed em `push.catalogo`), `integrations.jobs` handler `ads:digest` (via `api.ads_digest_request`); front → `ads.notification_preferences` (`api.ads_notification_preferences_set`), `ads.settings` `ux.digest_nivel`. Readers: `ads.digest_build` (lê `metrics_daily`, `attributed_events`, `insights`, `recommendations`, `data_quality_incidents`, `accounts`), `ads.digest_recipients` (lê `core.tenant_users`, `auth.users` e-mail, `notification_preferences`), `api.ads_digest_latest`/`ads_digests` ← front (bloco "Resumo de ontem"), `api.ads_run_now_status` (lê `data_quality_runs`, `accounts.attribution_run_at`, `detector_runs`, `digests`). Contratos acoplados: JSON v1 do `digest_build` ↔ `_shared/ads/digestEmail.ts` (tipo `AdsDigest`) ↔ `digestNarrative.ts` (validação numérica) ↔ front; `delivery.hour_local` ↔ `digest_accounts_due`.
- **Insight Engine (F3, migration `20261105100700`):** writers `ads.det_*` via `ads.run_detectors` (edge `ads-sync` após atribuição; `api.ads_detectors_run_now`) → `ads.insights` (+`priority`, `misses`), `ads.detector_runs/checks`; `ads.recommend_from_insight` → `ads.recommendations` + `recommendation_events`. Leem `ads.metrics_daily`, `attributed_events`, `accounts` (metas), `actions` (mudança de orçamento recente), `settings` (`det.*`), `ads.data_state`. Readers: `api.ads_insights` (ordenada por `priority`), `api.ads_insight`, `api.ads_recommendations`, `api.ads_detector_catalog` ← front `adsIntelligence.ts` (normalizadores de `evidence`/`hypotheses`/`suggested_action` acoplados ao formato das funções `ads.det_*` — mudar o JSON exige mudar os dois). Limiares `det.*` também lidos por `api.ads_detector_catalog`.
- **Atribuição (F2, migration `20261105100600`):** writers `ads.attribute_transactions`/`attribute_leads` (via `ads.run_attribution`, chamada por `ads.run_data_quality` e pela edge `ads-sync`, e à mão no backfill) → `ads.attributed_events`, `ads.accounts.attribution_*`. Lê `commerce.transactions` (`tenant_id`, `status` enum `public.invoice_status`, `paid_at`, `transaction_date`, `amount`, `currency`, `party_id`, `utm_*`; índices `idx_transactions_tenant_paid_at`, `idx_transactions_status`, `idx_transactions_party_id`), `forms.form_submissions` e `smart_forms.form_submissions` (`submitted_at`, `status`, `utm_*`, `party_id`). Readers: `ads.ux_real_kpis`/`ux_atribuicao` ← `api.ads_overview`, `ads_campaigns_overview`, `ads_metrics_series`, `ads_attribution_summary`, `ads_attributed_events`; `ads.dq_purchase_vs_vendas`, `dq_receita_reportada_vs_real`, `dq_reembolsos`, `dq_gasto_vs_comercial`, `dq_rastreio_utm`. **Contrato:** mudar o significado de `status`/`paid_at` em `commerce.transactions` ou o padrão de UTM (`utm_term` = id de conjunto, `utm_content` = id de anúncio, `utm_campaign` = id/nome) exige rever `ads.attr_match`/`attr_canal`.
- **Sem writer ainda:** `metrics_breakdown_daily` (sessão "dados sociais em casa" vai criar `ads.upsert_metrics_breakdown_daily` e `ads.metrics_window`, migration `20261105120100`), `recommendations`/`actions`/`action_outcomes` (F3/F5 geram), `launches`, `campaign_tags`, `product_costs`, `usage_daily`.

### integrations.social_report_cache (2026-11-04, migration `20261104160000`, NÃO aplicada)
- **Writers (só service role):** edges `social-insights` e `social-ads-insights` via `_shared/socialCache.ts` (`marcarPreparando`: insert/update do marcador `preparando`; `gravarCache`: upsert `ok` com rows; `apagarMarcador`: delete do marcador ou volta a `ok` depois de erro da Windsor); edge `social-cache-warm` (`aquecerChave`: posse do marcador + fetch + gravar; disparada pelo cron `kreators-social-cache-warm` → `integrations.invoke_social_cache_warm()`, migration `20261104170000`, e por `social-accounts-sync`); cron `kreators-social-report-cache-limpar` → `integrations.social_report_cache_limpar()` (delete do vencido há > 1 dia).
- **Readers (só service role):** RPC `integrations.social_report_contexto` (linha da chave), `_shared/socialCache.ts` (`lerCache`, `esperarCache`) e `social-cache-warm` (`lerEstadosCache`, sem as rows; também lê `integrations.social_accounts` ativas). Nenhum reader `authenticated`.
- **Contrato da chave:** `chaveCache` em `_shared/socialCache.ts` e o cálculo para a conta padrão dentro da RPC (`sha256(account_id || '|' || sufixo)`) precisam mudar juntos. As janelas e reports de abertura em `_shared/socialCacheWarm.ts` (e `_shared/socialInsightsReports.ts`) espelham o front (`periodo.ts`, abas sociais): mudar o período padrão de uma aba exige mudar o aquecimento (`socialCacheWarm.test.ts` compara com as funções do front).

### integrations.social_connect_links (2026-11-04, migration `20261104100000`, NÃO aplicada)
- **Writers (só service role):** edge `social-connect-url` (insert `pending` com `access_token`; `cancelled` nos pendentes além dos 5 mais novos do tenant/provider), edge `social-accounts-sync` (`expired` nos pendentes vencidos ou vencidos na Windsor; `consumed` + `consumed_at` + `metadata.windsor_link`/`resultado` no link que devolveu conta ou que foi usado — `revoked_at` — há mais de 2 h sem conta).
- **Readers (só service role):** `social-accounts-sync` (links do tenant/provider, `access_token` para `co-user-linked-accounts`), `social-disconnect` (`access_token` do link de origem via `social_accounts.connect_link_id`, para o DELETE na Windsor). **Nenhum reader `authenticated`** (sem grant, sem policy): o token não vai ao navegador.
- **FK:** `social_accounts.connect_link_id → social_connect_links.id on delete set null`; `tenant_id → core.tenants on delete cascade`.

### Edge functions (JWT ON, owner/admin do tenant ou system admin)
- `social-connect-url` → Windsor `generate-co-user-url/`; grava `integrations.social_connect_links`; devolve `connect_url` + `link_id`.
- `social-accounts-sync` → Windsor `co-user-linked-accounts/?access_token=` (um por link pendente) + `ds-accounts` (nome; reconexão por `account_id`); lê/grava `integrations.social_connect_links` e `integrations.social_accounts`. Ganchos fire-and-forget com a service role ao gravar/reativar uma conta: `dispararSnapshotConnect` (`social-snapshot`), `dispararIngestConnect` (fase 2, só Instagram: `POST social-ingest {modo:"connect", social_account_id}` → backfill na fila `social:ingest`; escrito, NÃO deployado) e `dispararCacheWarm` (`social-cache-warm`).
- `social-disconnect` → Windsor `DELETE co-user-linked-accounts/`; lê `integrations.social_connect_links` (token), grava `integrations.social_accounts`.
- Helpers: `_shared/windsor.ts` (fetch Windsor sem seguir redirect, `windsorGenerateCoUserUrl`/`windsorCoUserLinkedAccounts`/`windsorUnlinkCoUserAccount`/`windsorListAccounts`, `requireTenantAdmin`, lê `core.tenant_users`, `core.system_admins`, `core.tenants`), `_shared/socialAccounts.ts` (puro: `PROVIDER_TO_CONNECTOR`, `normalizeDatasource`/`classifyDatasource`, `parseCoUserUrl`, `windsorLinkState`/`parseWindsorDate`, `parseCoUserLinkedAccounts`, `accountsForLink`, `matchDsAccount`, `selectLinksToCheck`, `unlinkOutcome`, `redactWindsorKey`/`tokenSuffix`; testes em `socialAccounts.test.ts`). Front puro testado em `src/lib/data/socialAccounts.test.ts` (`pendingFromConnect`, `parsePendingConnect`, `ehErroDeLink`).
- **Contrato compartilhado:** `link_id` sai de `social-connect-url` e volta em `social-accounts-sync`; o resultado do sync é `{ok:true, accounts:[…], conta_em_uso}` (lista, não mais `account`) e os erros de link (`link_expirado|link_nao_encontrado|nenhum_link_pendente`) levam a tela a pedir link novo. Mudar esses códigos exige mudar `SyncResult`/`ehErroDeLink` no front.

### core.features `social`
- Inserida pela migration `20261102150000` (module `social`, sort_order 42, ícone `share-2`); `core.tenant_features` só para a Maconequi.

### Edge fn `social-insights` (leitura, 2026-11-02)

Leitura só (ARCHITECTURE §22.18). Nenhum writer no banco: a função só consulta e repassa a Windsor.ai.

| Papel | Quem |
|---|---|
| Lê | `core.tenant_users` (membership), `core.system_admins`, `integrations.social_accounts` (SÓ `account_id`, `account_name`, filtro `tenant_id` + `windsor_connector='instagram'` + `status='active'`) |
| Externo | `GET https://connectors.windsor.ai/instagram` com `WINDSOR_API_KEY` (secret da edge function) e `select_accounts=<account_id>` |
| Frontend | `src/lib/data/socialInsights.ts` → `src/pages/admin/Social.tsx` e `src/components/social/*` (/admin/social, feature `social`) |

Mudou o nome/forma de `integrations.social_accounts` (colunas `account_id`, `windsor_connector`, `status`) ou os campos do conector da Windsor → revisar a função e os tipos de linha em `socialInsights.ts`. **`account_id` no body (2026-11-04, sem redeploy):** a função lê `account_id?` e escolhe a conta por `_shared/socialInsightsConta.ts` (`escolherContaInstagram`: ativa do tenant pedida, ou a mais recente por `connected_at desc nulls last, id asc`); o front manda a conta do `ContaInstagramContext` (`socialInsights.ts`) e a inclui na chave de cache — mudou a regra de conta padrão → mudar nos dois lados e em `contasAdsElegiveis`/`useContaAds` (mesma ordem do seletor).

### integrations.social_account_daily (histórico diário do Instagram; migration `20261104190000`, aplicada 2026-11-04)

Uma linha por conta Instagram por dia (ARCHITECTURE §22.18 "Histórico além de 30 dias"; plano em `docs/social-instagram-historico-plano.md`). O que a Meta descarta (`new_followers` após 30 dias, série de `followers_total`) só existe aqui.

| Papel | Quem |
|---|---|
| Writer (só service role) | edge fn `social-snapshot` via RPC `integrations.upsert_social_account_daily(p_social_account_id, p_kind, p_source, p_rows)` (`daily` sobrescreve as 9 métricas; `followers` e `profile` com `coalesce(excluded, atual)`). Disparada pelo cron `kreators-social-snapshot-diario` (`integrations.invoke_social_snapshot`, pulse) e por `social-accounts-sync` na conexão (`dispararSnapshotConnect`, `{modo:"connect"}` = backfill 729 dias em 4 janelas de ≤ 183 dias, retomável: janela ≥ 95 % coberta é pulada). |
| Reader `authenticated` | RPC `api.social_account_daily(p_tenant_id, p_account_id, p_from, p_to)` (R56, `assert_tenant_access`) ← `src/lib/data/socialHistory.ts` (`useSocialAccountHistory`, chave `["social","instagram",tenant,"history",account,from,to]`) ← `InstagramOverview.tsx` (períodos 90 d / 12 m / 24 m) e `InstagramFollowersChart.tsx`. |
| Reader service role | `social-snapshot` não lê a tabela (upsert cego); `integrations.logs` recebe os erros (`provider='instagram'`, `capability='social_snapshot'`). |
| FK | `social_account_id → integrations.social_accounts(id) on delete cascade`; `tenant_id → core.tenants on delete cascade` (histórico some com a conta/tenant). |

Mudou uma coluna da tabela → mudar `_shared/socialSnapshot.ts` (`linhasDiarias`/`linhasSeguidores`/`linhaPerfil`), a RPC de upsert (lista fixa de colunas por `kind`), a RPC de leitura (`returns table`) e `HistoricoDiaRow`/`linhasParaDiario` em `socialHistory.ts`. Mudou a janela de 30 d (`janelaDiaria`) → o cache de reports da sessão de desempenho e o pré-aquecimento erram a chave. Mudou o limite de 729 dias da Meta → `janelaBackfill`/`janelasBackfill` (`socialSnapshot.ts`), `MAX_DIAS_META` (`periodo.ts`) e `MAX_DIAS` da `social-insights`. **Fase 1 de D-SOCIAL-16 (migration `20261105120000`, escrita, NÃO aplicada):** 5 colunas de breakdown (`reach_followers`, `reach_non_followers`, `views_followers`, `views_non_followers`, `follows_and_unfollows`) e `p_kind = 'breakdown'` na RPC de upsert (toca só essas 5 colunas e sobrescreve, como `daily`); **Writer do `p_kind = 'breakdown'` (só service role, fase 2 escrita em 2026-11-05, NÃO deployada): edge fn `social-ingest` via RPC `integrations.upsert_social_account_daily(p_kind 'breakdown')` (job kind `breakdown`)**, com `WindsorProvider.buscarSerieContaInstagram(..., { partes: ['breakdown'] })` (D7: 1 chamada) e gravando só nas datas de `datas.breakdown` (a RPC sobrescreve). `daily`/`followers`/`profile` continuam só do `social-snapshot`. `api.social_account_daily` é recriada com as 5 colunas no fim do retorno (o front atual ignora as colunas que não conhece em `normalizarHistorico`).

### Edge fn `social-ads-insights` (Meta Ads e Google Ads, leitura; v1 em produção 2026-11-03, v2 Meta 2026-11-04 14:00 UTC, v3 com Google Ads 2026-11-04 15:05 UTC — commit 596ae14)

Leitura só (ARCHITECTURE §22.18, bullets Meta Ads e Google Ads). Nenhum writer no banco. Roteiros: `supabase/scripts/social_meta_ads_deploy.md` e `social_google_ads_deploy.md`. O provedor é escolhido pelo `connector` do body (`facebook` | `google_ads`, `_shared/socialAdsProviders.ts`).

| Papel | Quem |
|---|---|
| Lê | `core.tenant_users` (membership), `core.system_admins`, `integrations.social_accounts` (SÓ `account_id`, `account_name`, filtro `tenant_id` + `windsor_connector` = `'facebook'` ou `'google_ads'` conforme o `connector` do body + `status='active'`, mais `account_id` quando vem no body; sem ele, `order by connected_at desc nulls last, id asc limit 1`) |
| Externo | `GET https://connectors.windsor.ai/facebook` ou `/google_ads` com `WINDSOR_API_KEY` (secret) e `select_accounts=<account_id>`; sempre `date_from`/`date_to`. Reports Meta `summary|daily|campaigns|adsets|ads|campaign_daily|ad_preview`; Google `summary|daily|campaigns|adgroups|ads` + (2026-11-04, SEM redeploy ainda) `keywords|search_terms|audiences|demographics_age|demographics_gender|locations|locations_city|placements|networks|when|devices|ad_videos|ad_assets` (`campaign_id` obrigatório; `chave` por report descarta a linha "enchimento"; `search_terms`/`locations_city`/`placements` com teto de 500 e `truncado` na resposta) |
| Helpers | `_shared/socialAds.ts` (puro: `REPORTS_ADS`/`ACOES`, `validarPeriodoAds`, `validarIdFiltro`/`validarAccountId`, `filtroObrigatorioAusente` (`ad_preview` exige `ad_id`), `filtrarLinhasAds` (`campaign_id`/`adset_id`/`ad_id`), `montarUrlWindsor`, `classificarRespostaWindsor`, `semChave`; testes em `socialAds.test.ts`). `_shared/socialGoogleAds.ts` (puro: `REPORTS_GOOGLE_ADS`, `validarAccountIdGoogle` (customer id com ou sem hífens), `filtrarLinhasGoogleAds` (`campaign_id`/`ad_group_id`); testes em `socialGoogleAds.test.ts`). `_shared/socialAdsProviders.ts` (`resolverProvedorAds(connector)` → `ProvedorAds` com `defReport`/`validarAccountId`/`filtrarLinhas`; `validarFiltrosAds` valida todos os filtros do provedor, aplica só os do report e recusa obrigatório ausente — enumera `ad_id` em `CAMPOS_META`, `ad_group_id` em `CAMPOS_GOOGLE`, `obrigatorios?` em `DefReportProvedor`; testes em `socialAdsProviders.test.ts`) |
| Frontend | `src/lib/data/socialAdsInsights.ts` (reaproveita `invocar`, `mapearResposta`, `FnErro`, `useTenantId`, `retrySeTransitorio` de `socialInsights.ts`; hooks `useMetaAdsSummary/Daily/Campaigns/Adsets/Ads/CampaignDaily/AdPreview`, todos com `accountId?` na chave) → aba: `src/components/social/MetaAdsTab.tsx` (+ `useContaMetaAds.ts`, seletor de conta), `MetaAdsKpis.tsx` (+ `metaAdsTile.tsx`), `MetaAdsSpendChart.tsx`, `MetaAdsCampaignsTable.tsx` (+ `MetaAdsStatusFilter.tsx`), `metaAdsFormat.ts`; página da campanha: `src/pages/admin/SocialMetaAdsCampaign.tsx` → `MetaAdsCampaignKpis.tsx`, `MetaAdsCampaignConfig.tsx`, `MetaAdsAdsetsTable.tsx` → `MetaAdsCampaignAds.tsx` → `MetaAdsAdPreview.tsx`; rota em `src/App.tsx` e `src/config/nav-config.ts` (`/admin/social/meta-ads/campanhas/:campaignId`, parent `/admin/social?tab=meta-ads`); aba em `src/pages/admin/Social.tsx` (`?tab=`, agora gravado na URL); botão "Ver análises" dos canais `meta_ads`/`google_ads` em `src/components/settings/SocialAccountsTab.tsx` (`TAB_POR_CANAL`). Lê `useSocialAccounts` de `socialAccounts.ts` (lista de contas para o seletor). **Google Ads:** `src/lib/data/socialGoogleAdsInsights.ts` (mesmo transporte; hooks `useGoogleAdsSummary/Daily/Campaigns/AdGroups/Ads`, chave `["social","google_ads",tenantId,report,…]`; puras testadas em `socialGoogleAdsInsights.test.ts`: `taxasGoogle`, `somarResumoGoogle`, `kpisResumoGoogle`, rótulos de status/canal/estratégia/tipo de anúncio, `orcamentoCampanhaGoogle`) → `src/components/social/GoogleAdsTab.tsx` → `GoogleAdsKpis.tsx`, `GoogleAdsSpendChart.tsx`, `GoogleAdsCampaignsTable.tsx` → `GoogleAdsCampaignSheet.tsx` → `GoogleAdsCampaignSections.tsx` (abas; `GoogleAdsSectionKit.tsx`, `GoogleAdsAdGroupsSection.tsx` (grupos → anúncios + `ad_videos`), `GoogleAdsKeywordsSection.tsx`, `GoogleAdsAudiencesSection.tsx`, `GoogleAdsLocationsSection.tsx`, `GoogleAdsPlacementsSection.tsx`, `GoogleAdsWhenSection.tsx`, `GoogleAdsDevicesSection.tsx`), `googleAdsFormat.ts`; hooks novos `useGoogleAdsKeywords/SearchTerms/Audiences/Demographics(Age|Gender)/Locations/LocationsCity/Placements/Networks/When/Devices/AdVideos` e puras testadas (`gradeQuando`, `agruparLocais`, `agruparPorChave`, `taxasVideo`, `videoPrincipalDoAnuncio`, `miniaturaYoutube`, rótulos pt-BR) em `socialGoogleAdsInsights.ts`; retry de consulta fria em `src/lib/data/socialAdsRetry.ts` (`retryAds`, `retryDelayAds`, `textoEsperaFriaGoogle`; teste `socialAdsRetry.test.ts`); seletor de conta genérico `useContaAds.ts` + `AdsAccountSelect.tsx` sobre `src/lib/data/socialAdsConta.ts` (`contasAdsElegiveis`, `contaEscolhidaValida`, `chaveContaAds`; testes em `socialAdsConta.test.ts`) e `socialAdsNum.ts` (`num`, `somaOuNull`, `dividir`) |

Mudou a lista de campos em `REPORTS_ADS`/`ACOES` → revisar os tipos de linha e as puras de `socialAdsInsights.ts` (a derivação de resultados por objetivo lê nomes de campo literais). Mudou `REPORTS_GOOGLE_ADS` → revisar os tipos `GoogleAds*Row` e as puras de `socialGoogleAdsInsights.ts` (as taxas lêem `cost`/`clicks`/`impressions`/`conversions`/`interactions`/`conversions_value` literais). Mudou a interface `ProvedorAds`/`DefReportProvedor` → `index.ts` e os dois módulos de provedor. `ACOES_INSTAGRAM` (`instagram_profile_follow`/`instagram_profile_visits`/`actions_like`, 2026-11-04) vai SÓ nos reports por entidade (`campaigns`/`adsets`/`ads`/`campaign_daily`; nunca em `summary`/`daily`, que a Windsor passaria a devolver por campanha): lidos por `kpisCampanha`, pelo cartão do anúncio (`MetaAdsCampaignAds.tsx`), pelo detalhe do conjunto (`MetaAdsAdsetsTable.tsx`) e, na aba, por `kpisResumo(atual, anterior, { atual: campanhas })`, que soma as linhas de `campaigns` para os tiles Seguidores/Custo por seguidor (`MetaAdsKpis.tsx`). `actions_landing_page_view` (em `ACOES`, todos os reports de métricas; 2026-11-04) é lido por `valoresResumo` (linha do `summary`, com variação), `valoresCampanha`, `taxaLandingPage`, pelo `MetaAdsSpendChart` (série `actions_landing_page_view`, em `CONTAGENS`), por `ExtrasNoDetalhe`/`MetricasNoDetalhe` (`MetaAdsAdsetsTable.tsx`) e por `metricasAnuncio` (`MetaAdsCampaignAds.tsx`); a cor da série é `--social-series-4` (`social-viz.css`) via `SERIES.s4` (`vizTokens.ts`) — adicionar um 5º papel exige novo slot nos dois arquivos. Mudou `periodo.ts` (`intervaloAteOntem`, `diasDoIntervalo`, `rotuloIntervalo`, `intervaloDaUrl`, `periodoDoIntervalo`) → afeta também a aba Instagram. Mudou a chave de query (`metaAdsQueryKey`) → a página da campanha depende de compartilhar o cache com a aba (mesma chave de `campaigns`/`summary` para o mesmo período e conta). Mudou `social_accounts_list` (colunas `provider`, `status`, `connected_at`, `account_id`, `account_name`) → quebra `contasMetaAdsElegiveis`/o seletor. A prévia depende de o Meta continuar servindo `business.facebook.com/ads/api/preview_iframe.php` sem `X-Frame-Options` (não documentado): se mudar, o fallback (miniatura + permalink) assume sem erro.

### Edge fns `social-comment-reply` e `social-comment-hide` (escrita, deployadas em 2026-11-03)

Escrita no Instagram via Windsor.ai (ARCHITECTURE §14.3 "Escrita em comentários"). Nenhum writer no banco: só leem `integrations.social_accounts` para conferir a conta e chamam a Windsor. Roteiro de deploy: `supabase/scripts/social_comentarios_deploy.md`.

| Papel | Quem |
|---|---|
| Lê | `core.tenant_users` (owner/admin), `core.system_admins`, `core.tenants` (via `requireTenantAdmin`); `integrations.social_accounts` (filtro `tenant_id` + `windsor_connector='instagram'` + `account_id` + `status='active'`, colunas `account_id, windsor_connector, status`) |
| Externo | `POST https://mcp.windsor.ai/` (JSON-RPC `tools/call` → `execute_action`, Bearer `WINDSOR_API_KEY`, resposta SSE) — ações `reply_to_comment`, `hide_comment`, `unhide_comment`. Uma chamada por invocação, sem retry |
| Helpers | `_shared/windsor.ts` (`windsorExecuteAction`, `WindsorError` com `acao_recusada`/`windsor_rate_limit`, `windsorErrorResponse` 422/429), `_shared/socialComments.ts` (puro: `validarTextoComentario`, `validarCommentId`, `contaAtivaDoTenant`, `extrairIdComentario`, `parseCorpoMcp`, `interpretarRespostaMcp`, `motivoRecusa`; testes em `socialComments.test.ts`) |
| Frontend | `src/lib/data/socialActions.ts` (`useSocialCommentReply` → `social-comment-reply`, `useSocialCommentHide` → `social-comment-hide`; reaproveita `lerErroFn`/`FnErro`/`rotuloErroSocial`/`socialReportKeyPrefix` de `socialInsights.ts`) → `src/components/social/useCommentActions.ts` (único chamador dos hooks; estado compartilhado) → `CommentThread.tsx` (`ListaFios`/`FormResposta`), usado por `InstagramPostSheet.tsx` e `InstagramCommentsPanel.tsx` (alternância em `InstagramPostsTable.tsx`) |

Contratos acoplados: o front valida o texto com `validarTextoComentario` próprio em `socialActions.ts` (2200 code points) — mudar o limite exige mudar os dois lados; `reply_id` pode vir `null` (o front gera id local). `useSocialCommentReply` invalida `socialReportKeyPrefix(tenantId, "comments")`: mudar a chave `K.report` em `socialInsights.ts` quebra a invalidação. `WindsorError`/`windsorErrorResponse` são compartilhados com `social-connect-url`/`social-accounts-sync`/`social-disconnect`: os códigos novos não alteram os existentes.

### integrations.social_media_posts (+ integrations.social_media_post_daily) — dados sociais fase 1 (migration `20261105120000`, escrita, NÃO aplicada)

Um post do Instagram por linha (`social_media_posts`, PK `(social_account_id, media_id)`, métricas lifetime, legenda/permalink/URLs de mídia) e o lifetime do post em cada dia de coleta (`social_media_post_daily`, PK `(social_account_id, media_id, date)`, só posts com ≤ 30 dias de vida — regra da ingestão, D3). ARCHITECTURE §14.3 D-SOCIAL-16 e §22.18; desenho em `docs/social-dados-plano.md` §3.3–3.4.

| Papel | Quem |
|---|---|
| Writer (só service role) | edge fn `social-ingest` (fase 2, escrita 2026-11-05, NÃO deployada) via RPC `integrations.upsert_social_media_posts(p_social_account_id, p_source, p_rows)` (coalesce — `null` nunca apaga; `first_seen_at` fixo; `tenant_id`/`provider_source` copiados da conta) e `integrations.upsert_social_media_post_daily(p_social_account_id, p_rows)` (sobrescreve o dia; post ainda não gravado em `social_media_posts` é ignorado) — **job kind `posts`** (post_daily só para posts com ≤ 30 d, `date` = dia da coleta; não é kind de job) |
| Reader service role | `social-ingest` conta posts da conta (D9, `contadoresDaCadeia`: janelas vazias e `total_gravado` × `media_total`) no backfill encadeado |
| Reader `authenticated` | fase 3 (RPC `api.social_media_posts`, devolvendo `InstagramMediaRow` de `socialInsights.ts`) — **ainda sem reader**; RLS select por `is_tenant_member`/system admin já existe na migration |
| Retenção | `social_media_post_daily`: cron `kreators-social-snapshots-limpar` → `integrations.social_snapshots_limpar()` (400 d). `social_media_posts`: sem TTL |
| FK | `social_media_posts.social_account_id → integrations.social_accounts(id) on delete cascade`; `tenant_id → core.tenants on delete cascade` (nas duas); `social_media_post_daily (social_account_id, media_id) → social_media_posts on delete cascade` (FK composta) |

Mudou uma coluna → mudar `PostInstagram` (`_shared/socialProvider.ts`), `normalizarPosts` (`_shared/windsorProvider.ts`, e `REPORTS_INSTAGRAM.media` em `socialInsightsReports.ts` se o campo vier da Windsor), as duas RPCs de upsert (lista fixa de colunas) e, na fase 3, `api.social_media_posts` + o mapeamento para `InstagramMediaRow` (prefixo `media_*`). Mudou a regra de ≤ 30 dias (D3) → só a ingestão (`social-ingest`) e a estimativa de volume; o banco não a impõe.

### integrations.social_audience_snapshot — dados sociais fase 1 (migration `20261105120000`, escrita, NÃO aplicada)

Audiência do Instagram por dia de coleta (a Windsor só entrega o valor de hoje), PK `(social_account_id, date, kind, key)`, `kind` `country|gender_age|city`. Agregado, sem dado pessoal.

| Papel | Quem |
|---|---|
| Writer (só service role) | fase 2 (`social-ingest`) via RPC `integrations.upsert_social_audience_snapshot(p_social_account_id, p_date, p_rows)` (substitui o dia de cada `kind` presente no payload; `kind` ausente fica como está) — **edge fn `social-ingest`, job kind `audience`** (uma chamada por job, sem lotes; fase 2 escrita 2026-11-05, NÃO deployada) |
| Reader `authenticated` | fase 3 (RPC `api.social_audience`, último `date` ≤ `p_date`, devolvendo `InstagramAudienceCountryRow`/`GenderAgeRow`/`CityRow`) — **ainda sem reader** |
| Retenção | cron `kreators-social-snapshots-limpar` → `integrations.social_snapshots_limpar()` (400 d) |
| FK | `social_account_id → integrations.social_accounts(id) on delete cascade`; `tenant_id → core.tenants on delete cascade` |

Mudou uma coluna ou um `kind` → mudar o CHECK de `kind`, `TipoAudiencia`/`AudienciaInstagram` (`socialProvider.ts`), `TIPOS_AUDIENCIA`/`normalizarAudiencia` (`windsorProvider.ts`), a RPC de upsert e, na fase 3, `api.social_audience` e os tipos de audiência de `socialInsights.ts`.

### integrations.social_ingest_runs — dados sociais fase 1 (migration `20261105120000`, escrita, NÃO aplicada)

Execuções da ingestão por conta e `kind` (`posts|post_daily|audience|breakdown|profile`), espelho de `ads.sync_runs`; é o "estado por conta" (último run por `kind` + jobs na fila). **Só service role** (sem grant nem policy para clientes, padrão `social_report_cache`).

| Papel | Quem |
|---|---|
| Writer (só service role) | fase 2 (`social-ingest`) via RPCs `integrations.start_social_ingest_run(p_social_account_id, p_kind, p_date_from, p_date_to, p_job_id)` e `integrations.finish_social_ingest_run(p_run_id, p_status, p_rows, p_partial, p_error, p_details)` (`P0002` para conta/run inexistente) — **edge fn `social-ingest`, todos os job kinds** (`breakdown`, `posts`, `audience`, `profile`; também o modo `manual`, `p_job_id` null) e `integrations.release_stale_social_ingest_jobs` (fecha como `falhou`/`travada` runs `running` há > 15 min) — fase 2 escrita 2026-11-05, NÃO deployada |
| Reader `authenticated` | nenhum direto; fase 3 (RPC `api.social_ingest_status`, que junta o último run por `kind` com os jobs `social:ingest` de `integrations.jobs`) — **ainda sem reader** |
| Retenção | cron `kreators-social-ingest-runs-limpar` → `integrations.social_ingest_runs_limpar()` (90 d; também runs abertos há mais de 90 d) |
| FK | `social_account_id → integrations.social_accounts(id) on delete cascade`; `tenant_id → core.tenants on delete cascade`; `job_id` sem FK (a fila `integrations.jobs` é purgada) |

Mudou `kind`/`status` → mudar os CHECKs, a validação nas duas RPCs (`22023`), a ingestão (fase 2) e `api.social_ingest_status` (fase 3). `error` sempre normalizado e sem chave (`semChave`).

### ads.metrics_window (+ ads.metrics_breakdown_daily ampliada) — dados sociais fase 1 (migration `20261105120100`, escrita, NÃO aplicada)

Aditiva sobre o schema `ads` em produção (subseção "schema `ads`" acima; dona: sessão do Ads Intelligence). `ads.metrics_window`: agregados do provedor por janela fixa (7/30/90 d até `as_of`) por entidade, PK `(account_id, level, entity_id, window_days, as_of)` — alcance/frequência não são aditivos (D2). `ads.metrics_breakdown_daily`: CHECK de `breakdown_type` trocado pela constraint nomeada `metrics_breakdown_daily_breakdown_type_chk`, ampliada com `age`, `gender`, `network`, `day_hour`, `country_region`, `keyword` (Google), gravados no nível `campaign`.

| Papel | Quem |
|---|---|
| Writer (só service role) | fase 2 (`ads-sync`, kinds `windows`/`breakdowns`, coordenado com a sessão do Ads) via RPC `ads.upsert_metrics_window(p_account_id, p_level, p_window_days, p_as_of, p_rows, p_source)` e `ads.upsert_metrics_breakdown_daily(p_account_id, p_level, p_breakdown_type, p_rows, p_source)` (mesmo estilo de `ads.upsert_metrics_daily`; conta inexistente → `P0002`; a sessão do Ads reaproveita a de breakdown na F4) — **writer previsto: `ads-sync`, kinds `breakdowns` (Google) / `windows` (Meta) como jobs próprios, via o pacote de integração** (`~/Library/Application Support/Claude/side-session-notes/local_c2da9448-7d1b-4391-90d1-0bd245cfbf3f/social-dados-ads-diff.md`, aplicado pela sessão do Ads; **NÃO aplicado**). Busca/normalização nos módulos nossos `_shared/adsBreakdowns.ts` (`itensRpcBreakdown` monta `p_rows`) e `_shared/adsWindows.ts` (`itensRpcJanela`); as etapas `executarBreakdowns`/`executarJanelas` entram no `_shared/ads/sync.ts` da sessão do Ads pelo pacote; módulos expostos por `WindsorProvider.buscarBreakdownsAds`/`buscarJanelaAds` |
| Reader `authenticated` | fase 3 (RPCs `api.ads_window` e `api.ads_breakdown`, e `api.ads_campaigns_metrics` para alcance/frequência dos presets) — **ainda sem reader**; RLS select padrão do schema `ads` |
| Retenção | `metrics_window`: cron `kreators-ads-metrics-window-limpar` → `ads.metrics_window_limpar()` (90 d). `metrics_breakdown_daily`: sem TTL |
| FK | `metrics_window.account_id → ads.accounts(id) on delete cascade`; `tenant_id → core.tenants on delete cascade` |

Mudou uma coluna de `metrics_window` → mudar `ads.upsert_metrics_window` (lista fixa de colunas), `MetricaJanela` (`socialProvider.ts`) e, na fase 3, `api.ads_window`/`api.ads_campaigns_metrics`; as colunas de ação acompanham as de `ads.metrics_daily` (nomes da Windsor em `REPORTS_ADS`) — coluna nova em `metrics_daily` pede a mesma em `metrics_window`. Mudou a lista de `breakdown_type` → mudar a constraint, `MetricaBreakdown` (`socialProvider.ts`, `breakdown_type` hoje é `string`) e a ingestão. **Guarda do banco compartilhado:** `metrics_window` precisa entrar na lista de tabelas permitidas do bloco 0 da `20261105100000` (arquivo da sessão do Ads; pedido registrado no cabeçalho da `120100`), senão reaplicar aquela migration aborta.

### `_shared/socialProvider.ts` + `_shared/windsorProvider.ts` — contrato do provider de dados sociais (fase 1, escrito, sem chamador)

`socialProvider.ts`: interface `SocialDataProvider` (não `SocialProvider`, que é a união das redes em `socialAccounts.ts`), tipos normalizados do Instagram (`PerfilInstagram`, `LinhaSerieInstagram`, `SerieContaInstagram`, `PostInstagram`, `AudienciaInstagram`) e `MetricaBreakdown`/`MetricaJanela`; Ads reaproveitam os tipos de `_shared/ads/adapter.ts`; seleção por conta em `providerDaConta` pela coluna `social_accounts.provider_source` (`api_oficial` → `ProviderNaoSuportado`). `windsorProvider.ts`: `criarWindsorProvider({ chave, … })`, única implementação hoje; módulo puro (chave, `fetch`, relógio e listagem de contas injetados). Testes `socialProvider.test.ts`/`windsorProvider.test.ts` com fixtures em `_shared/fixtures/windsor/`.

| Papel | Quem |
|---|---|
| Quem usa | fase 2: edge fn `social-ingest` (Instagram) e, coordenado com a sessão do Ads, `buscarBreakdownsAds`/`buscarJanelaAds` para os kinds `breakdowns`/`windows` da `ads-sync` — **chamador escrito em 2026-11-05 (NÃO deployado):** `social-ingest` (Instagram, `partes` em `buscarSerieContaInstagram`, D7); ads só quando a sessão do Ads aplicar o pacote de integração |
| Depende de | `socialInsightsReports.ts` (`REPORTS_INSTAGRAM`, período e URL do Instagram), `socialAds.ts` (`classificarRespostaWindsor`, `semChave`, `MAX_DIAS_ADS`), `socialSnapshot.ts` (`paraInt`, `linhasDiarias`, `linhasSeguidores`, `somarDias`), `ads/adaptadores.ts` (`resolverAdaptadorAds` → `metaWindsorAdapter`/`googleWindsorAdapter`), `ads/adapter.ts` (tipos, `ErroAds`, capacidades), `ads/windsorHttp.ts` (`erroDeStatus`, `criarNormalizadorErroWindsor`, pausa/timeout), `socialAccounts.ts` (`PROVIDER_TO_CONNECTOR`, `SocialProvider`) |
| Externo | Windsor.ai (conectores `instagram`, `facebook`, `google_ads`) com `WINDSOR_API_KEY` injetada (`deps.chave`); nunca logar a URL; erros por `semChave` |

Mudou um report/campo em `REPORTS_INSTAGRAM` ou `linhasDiarias`/`linhasSeguidores` → rodar `windsorProvider.test.ts` (as fixtures fixam os nomes de campo). Mudou o contrato de `AdsProviderAdapter` (arquivo da sessão do Ads) → revisar a delegação de ads do `WindsorProvider`. Nenhum nome de campo ou URL da Windsor deve sair destes módulos e dos módulos de reports (D-SOCIAL-16).

### Edge fn `social-ingest` (ingestão Instagram em tabelas nossas — fase 2; escrita e revisada 2026-11-05, NÃO aplicada/deployada/commitada)

Consome a fila `integrations.jobs` handler `social:ingest` (migration `20261105130000_social_ingest_fila.sql`) e grava pelas RPCs da `20261105120000`. Modos `cron`, `enqueue_daily`, `connect`/`backfill`, `manual`. ARCHITECTURE §3/§11/§22.18; "como ficou" em `docs/social-dados-plano.md` §6; runbook `supabase/scripts/social_dados_deploy.md`.

| Papel | Quem |
|---|---|
| Chamadores | crons `kreators-social-ingest` (`integrations.invoke_social_ingest` → `{modo:"cron"}`) e `kreators-social-ingest-enqueue-daily` (`integrations.invoke_social_ingest_enqueue_daily` → `{modo:"enqueue_daily"}`), ambos via `integrations.pulse_social_ingest` (pg_net, nascem desligados); `social-accounts-sync` (`dispararIngestConnect`, `{modo:"connect"}`); operador (`backfill`/`manual`) |
| Lê | `integrations.jobs` via `worker.claim_jobs(p_handlers => '{social:ingest}')` (+ `worker.complete_job`/`fail_job`/`block_job`, `integrations.requeue_social_ingest_job`, `release_stale_social_ingest_jobs`, `enqueue_social_ingest*`); `integrations.social_accounts` (conta, conector, status, `provider_source`, `profile.media_total`); `core.tenants` (tenant ativo, R64); `integrations.social_media_posts` (contagem D9) |
| Escreve | `social_media_posts`/`_post_daily` (kind `posts`), `social_audience_snapshot` (`audience`), `social_account_daily` `p_kind 'breakdown'` (`breakdown`), `social_accounts.profile` via `upsert_social_profile` (`profile`), `social_ingest_runs` (todos), `social_accounts.last_sync_at`/`error_message` (direto, best effort), `integrations.logs` (provider `instagram`, capability `social_ingest`) |
| Externo | Windsor.ai (conector `instagram`) via `WindsorProvider` (`criarWindsorProvider`), `WINDSOR_API_KEY` injetada; nunca logar a URL |
| Helpers | `_shared/socialIngest.ts` (puro: `executarJobSocial`, `fecharJobSocial`, `criarRpcJob`, `sourceDoJob`, `decidirEncadeamento`/`contadoresDaCadeia`, validação de body/payload/progresso; testes `socialIngest.test.ts`), `_shared/socialProvider.ts`, `_shared/windsorProvider.ts`, `_shared/ads/sync.ts` (orçamento `criarOrcamento`, `ParadaAds`, `ClienteAds`), `_shared/socialSnapshot.ts` (`bearerDe`, `segredoConfere`, datas), `_shared/chatAuth.ts` (`roleDoJwt`), `_shared/socialAds.ts` (`semChave`), `_shared/windsor.ts` (`windsorApiKey`, `windsorListAccounts`) |
| Frontend | nenhum (a UI verá o estado na fase 3 por `api.social_ingest_status`) |

Mudou o payload do job (`social_account_id`, `kind`, `date_from`, `date_to`, `progresso`) → mudar `validarPayloadJob`/`validarProgressoSocial` (`socialIngest.ts`) e as funções `integrations.enqueue_social_ingest*`/`requeue_social_ingest_job`/`social_ingest_etapas_concluidas` juntos. Mudou a regra de prioridade/cobertura do SQL (cobertura só para breakdown) → conferir `sourceDoJob` e o encadeamento D9 (um posts que vire cobertura mata a cadeia: `encadeamento_coberto`). Mudou `ProgressoSocial`/etapas por kind → requeue e retomada do manual (`repetir`). Copia de `ads-sync/pedido.ts` as funções de destino da fila: mudanças lá não chegam aqui sozinhas.
