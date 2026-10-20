# KreatorsHub — Dependency Map

> **REGRA R37**: Antes de alterar qualquer item listado aqui, verificar TODOS os readers/writers.
> Atualizar este arquivo sempre que adicionar novo consumidor.
>
> Gerado em: 2026-03-19 · Última atualização: 2026-10-22 (push: 2026-10-20, não aplicado)

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
  - `src/pages/admin/CampaignSend.tsx` — via `getSegmentRecipientsFailClosed()` (R38)
  - `api.get_segments_recipient_count(tenant_id, segment_ids[], exclude_segment_ids[])` — COUNT DISTINCT lower(email) entre múltiplos segmentos, descontando segmentos de exclusão; `segment_ids NULL` = todos os contatos do tenant. Usado por `CampaignSend.tsx` para exibir contagem exata de destinatários únicos no botão de envio (somente leitura, mesma semântica de `get_segment_parties_for_campaign`)
  - `CampaignSend.tsx` — `getExclusionEmailSet()` lê membros de segmentos marcados como EXCLUSÃO no disparo (fail-closed: exclusão com regras e 0 membros bloqueia o envio). IDs auditados em `email.campaign_sends.excluded_segment_ids`
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
  - `api.get_campaign_unsubscribe_stats` (Onda 3) — filtra `send_id`s do tenant; `delivered_count` como fallback de entregues
  - `api.get_tenant_consent_summary` (Onda 3) — contagem de envios marketing/transacional e `sum(sent_count)` transacional na janela (`is_test` fora; status `cancelled/scheduled/pending/draft` fora; data `coalesce(started_at, created_at)`)
- **Colunas de rastreabilidade (Guard 1):** `recipient_source` (segment/manual/all/test), `segment_ids` (uuid[]), `segment_parties_snapshot` (int), `recipient_count_at_send` (int). Adicionadas Sprint 12 para auditoria de blast radius.
- **Validação no banco (R60, 2026-10-21, migration `20261021120000_campaign_message_type_validation`):** trigger `trg_campaign_sends_validate_message_type` (BEFORE INSERT OR UPDATE OF `message_type, group_id, transactional_reason`) → `email.campaign_sends_validate_message_type()`. Marketing sem `group_id` → `marketing_requires_group`; grupo de outro tenant → `invalid_group`; transacional → `group_id` forçado a NULL e `transactional_reason` obrigatório quando `is_test=false` (vem do send ou, se ausente, de `email_campaigns.transactional_reason`) → senão `transactional_requires_reason`. Todos SQLSTATE `23514`. Em UPDATE só valida quando essas colunas mudam de fato; SET NULL da FK de grupo é aceito. **Todo writer novo** de `campaign_sends` precisa mandar grupo (marketing) ou ter motivo na campanha (transacional).
  - `workers/journeys/src/handlers/processJourneyNode.ts` passa `transactional_reason` da campanha e trata a rejeição (`sendValidationErrorCode`): nó falha uma vez com mensagem clara, sem retry (não deployado).
  - `supabase/functions/process-journey/index.ts` (legado, sem callers conhecidos) insere sem `message_type`/`group_id` → seria rejeitado por `marketing_requires_group`.
- **Regra:** NÃO contém lista de recipients. É apenas metadados (template, subject, status, counts). A lista real está em `campaign_recipients`.

### email.campaign_recipients — Lista congelada de envio (R38)

- **Writers:**
  - `src/pages/admin/CampaignSend.tsx:732` — INSERT bulk: congela lista de recipients ao disparar campanha
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
    - UI: `TenantConsentSummaryCard` (suprimidos por tipo) e `estimateBlockedRecipients()` (soma de `suppressed_by_type` no "até N podem ser pulados" de `CampaignSend.tsx`). O front não tem GRANT em `suppression_list`; esta RPC é o único caminho.
  - Migration `20261022120300_consent_backfill_d3_d4` (D3) — leu `spam_report` ativos sem global
  - `workers/email-campaign/src/handlers/sendEmailBatch.ts:253` — SELECT suppressed emails (pre-send gate)
  - `workers/email-campaign/src/handlers/retrySoftBounce.ts:120` — SELECT before retry
  - `workers/journeys/src/handlers/processJourneyNode.ts:1883` — SELECT before journey send
  - `email.consent_state_snapshot()` — before/after da trilha de consentimento
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
- **Via RPC (Onda 3, 2026-10-22):** `email.apply_consent_change` `resubscribe` global com `channel=preference_center` (inclui o `confirm_resubscribe` do double opt-in) também faz `is_active=false` nos unsubs ativos de grupos INATIVOS do mesmo email/tenant (ids em `request_payload.inactive_groups_cleared`, uma linha de auditoria). Migration `20261022120300_consent_backfill_d3_d4` (D3) criou 49 globais `source='manual'` (`channel=system`, `request_id='backfill_spam_report_D3_20261022'`) para spam_report ativos sem global.
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

- **Writer:** `api.set_campaign_message_type` (só quando algo muda). RLS sem policies, sem grants a anon/authenticated; service_role SELECT/INSERT.
- **Readers:** nenhum na UI ainda (operacional via service_role).

### email.email_campaigns.message_type / transactional_reason (R60)

- **Writers:** `api.set_campaign_message_type` (owner/admin; transacional exige motivo e zera `group_id`; marketing exige grupo ativo). A UI legada ainda faz UPDATE direto de `message_type` (sem motivo) até o deploy da Onda 2 — campanha virada para transacional por esse caminho depois do backfill fica sem motivo e seus envios não-teste são rejeitados pela trigger de `campaign_sends` (`transactional_requires_reason`). Na Onda 2 (interface, não deployado) a UI passa a usar a RPC: `CampaignEditor.tsx` → `CampaignMessageTypeField` → `src/lib/email/messageType.ts` `setCampaignMessageType()`; os saves do editor não mandam mais `message_type` e só mandam `group_id` em marketing. `CampaignSend.tsx` duplica campanha copiando `transactional_reason` (transacional) e lê o motivo para o insert de `campaign_sends.transactional_reason`.
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

- **Writers:** `CampaignSend.tsx` (insert do envio; reagendar chama `email.cancel_campaign_scheduled_sends` antes), `workers/email-campaign` (prepare/send batch: `scheduled|pending` → `sending` → `completed`), `email.cancel_scheduled_send` (→ `cancelled`), `reconcile-sendgrid` (contadores), `process-journey` (insert com status `pending`)
- **Readers:** `Campaigns.tsx` (agendados + em andamento, com botão cancelar/parar), `CampaignSend.tsx` (Envios Recentes), `CampaignResults.tsx`
- **Invariante (R49):** índice `campaign_sends_one_scheduled_per_campaign` — no máximo UM envio `scheduled` não-teste por campanha. Jornadas criam `pending`, então não colidem.
- **Cancelamento (R50):** `email.cancel_scheduled_send(send_id, tenant_id)` aceita `scheduled|pending|sending`; marca recipients `pending`→`skipped`, remove jobs pendentes do envio em `journeys.processing_jobs` (por `payload->>'send_id'`) e devolve a campanha para `ready` (nada enviado) ou `sent` (envio parcial). Recipients `processing` são deixados em paz — o lote em voo termina e os seguintes abortam pela checagem de status em `sendEmailBatch.ts`.

---

## Chat (Whaticket) — módulo separado do WhatsApp/Apidozap

### public.chat_contatos / chat_mensagens / chat_config / chat_agentes

- **Writers:** `chat-webhook/index.ts` (mensagens recebidas + resposta IA, service role), `chat-enviar/index.ts` (envio humano, service role), frontend `src/lib/data/chat.ts` (contatos/config/agentes via RLS)
- **Readers:** `src/pages/admin/Chat.tsx` via hooks de `src/lib/data/chat.ts`; realtime na publication `supabase_realtime`
- **RPCs:** `public.chat_incrementa_nao_lidas` (service_role only), `public.chat_zerar_nao_lidas` (authenticated, valida `app_auth.assert_tenant_access`)
- **Regra:** port 1:1 do módulo campanhas do Carreta Oftalmo — mudanças estruturais devem preservar o mapeamento `campanhas_*` → `chat_*` (ver ARCHITECTURE §22.14)

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
| `core.tenants` (`is_active`, `archived_at`) | `_usuario_tem_acesso`, `_motivo_fila` (`tenant_inativo`), `enfileirar_evento` | tenant ativo |
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
