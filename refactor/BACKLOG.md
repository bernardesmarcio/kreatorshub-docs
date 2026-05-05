# Refactor Backlog — Tarefas Atômicas

**Documento companheiro:** [PLAN.md](./PLAN.md), [PROGRESS.md](./PROGRESS.md)

> Lista canônica de tarefas. Cada uma com ID estável, critérios de aceite, owner, status.
> Granularidade: micro nas Fases 1-3 (próximas), macro nas Fases 4-8 (refinaremos quando entrar).
> Status: `todo` / `in_progress` / `done` / `blocked` / `cancelled`

---

## Fase 1 — Estabilização ✅ CONCLUÍDA (2026-05-03 02:00 UTC)

> R-1.0 a R-1.7 done. Loop de amplificação 25x quebrado estruturalmente.
> Telemetria nativa `eval_drift_count` ativa para guard de Fase 2.

### R-1.0 — Definir whitelist de dimensões em ARCHITECTURE.md (PRÉ-REQUISITO)

**Status:** done (pelo cleanup PR v1.1 — v7.28)
**Owner:** Claude (chat) + Marcio (review)
**Estimativa:** 2h
**Risco:** baixo
**Bloqueia:** R-1.1 a R-2.16 (toda Fase 1 e 2)
**Referência:** ARCHITECTURE.md §22.3 (criada em v7.28)

**Critérios de aceite:**
- Seção §22.3 publicada em ARCHITECTURE.md
- Whitelist explicita TODAS as dimensões que bumpam `segmentation_input_version`
- Blacklist explicita dimensões que NÃO bumpam (predições, cosméticos, membership, timestamps internos)
- Categoria "Campos BULK_ONLY (R36)" cobrindo opportunity_* (fora do contrato)
- Procedimento documentado para adicionar/remover dimensão futuramente (com bump de `analytics_segmentation_contract_version`)
- Validação: cada dimensão referenciada por qualquer segmento ativo deve estar na whitelist OU ter justificativa explícita do contrário (snapshot 2026-05-02 validado: 21 dims em uso)

**Por que pré-requisito:** sem whitelist explícita, R-2.1 (Decision Write API) tem
critério de aceite ambíguo. Risco de cada writer migrado decidir whitelist sozinho =
12 implementações divergentes. Whitelist tem que ser fonte única, escrita ANTES dos
PRs da Fase 2 começarem.

### R-1.1 — Pre-flight checklist em produção (substitui criação de branch)

**Status:** done
**Owner:** Marcio + Claude (chat)
**Concluído em:** 2026-05-02
**Decisão arquitetural:** D-2026-05-02-08 (cancelar branch develop)

**Critérios de aceite (atendidos):**
- ✅ Snapshot baseline em PROGRESS.md ("Snapshot pré-Fase 1")
- ✅ Definições atuais de funções críticas preservadas via md5:
  - `apply_segment_membership_diff` md5 `3e35fb8c1cf3b40c3251564bc5ac5fbc`
  - `run_segment_eval_fallback` md5 `6a41cfe1c50bbeb6def9c9413e4b1196`
- ✅ Sistema em estado consistente (0 pending, 0 claimed em 2026-05-02 19:10 UTC)
- ✅ Backup automático Supabase ativo (validar via dashboard antes de R-1.3)

**Não fazer:** branch develop (cancelada via D-2026-05-02-08).

### R-1.2 — Migration: adicionar versões semânticas em contact_state

**Status:** done
**Owner:** Dev (via Claude Code)
**Concluído em:** 2026-05-02 19:14 UTC
**Estimativa:** 30min (real: ~25min com validações)
**Risco:** baixo
**Bloqueado por:** R-1.1
**Bloqueia:** R-1.3

**Critérios de aceite (atendidos):**
- ✅ `ALTER TABLE analytics.contact_state ADD COLUMN segmentation_input_version bigint NOT NULL DEFAULT 0`
- ✅ `ALTER TABLE analytics.contact_state ADD COLUMN last_evaluated_segmentation_version bigint NOT NULL DEFAULT 0`
- ✅ `ALTER TABLE analytics.contact_state ADD COLUMN segment_membership_updated_at timestamptz`
- ⏸️ Index `contact_state_drift_idx` — postergado para R-1.3 (criar pós-backfill, evita escrita inútil em 42k rows com versões zeradas)
- ✅ Migration aplicada direto em produção (D-2026-05-02-08 — sem branch develop)
- ✅ Queries existentes validadas, sem regressão

**Concluído em:** 2026-05-02 19:14 UTC
**Migration aplicada:** `20260502191444 r12_add_segmentation_versioning_columns` em `pbfpwfkgjaqotbudihpy`
**Validações realizadas:**
- 3 colunas presentes com tipos/defaults corretos (V1)
- 3 comments com referência a §22.3 e R-1.2 aplicados (V2)
- 42.632 parties com defaults zerados (V3)
- Smoke test: queries críticas (`state_version > 0`, `MAX(state_updated_at)`, `segment_ids LIMIT 1`, `COUNT(*)`) sem regressão (V4)
- Migration listada em `supabase_migrations.schema_migrations` (V5)

### R-1.3 — Backfill: zerar drift histórico

**Status:** done
**Owner:** Dev (via Claude Code)
**Concluído em:** 2026-05-02 19:30 UTC
**Estimativa:** 1h (real: ~15min — janela 100% quiescente)
**Risco:** baixo
**Bloqueado por:** R-1.2
**Bloqueia:** R-1.4

**Critérios de aceite (atendidos):**
- ✅ One-shot UPDATE: `UPDATE analytics.contact_state SET segmentation_input_version = state_version, last_evaluated_segmentation_version = state_version WHERE state_version > 0`
- ⚠️ Branch develop pulada via D-2026-05-02-08 (cancelar branch). Aplicada direto em produção.
- ✅ Em produção: aplicada em janela quiescente (0 pending, 0 claimed, 0 writes em 5min — sábado 2026-05-02 19:30 UTC)
- ✅ Drift_idx pós-backfill = 0 rows (validado por tenant: 7/7 com drift=0)

**Migrations aplicadas:**
- `20260502193017 r13_backfill_segmentation_versioning` — UPDATE em massa
- `20260502193133 r13_create_contact_state_drift_idx` — índice parcial (postergado de R-1.2)

**Validações pós-backfill:**
- 42.632 parties: `segmentation_input_version = state_version` e `last_evaluated_segmentation_version = state_version` em 100%
- `max_seg_version = max_watermark = 35` (consistente com `max(state_version)` pré-R-1.3)
- Distribuição por tenant: drift = 0 em todos os 7 tenants ativos (28.735 + 7.012 + 6.069 + 781 + 28 + 5 + 2 = 42.632)
- Índice `contact_state_drift_idx` criado, 8192 bytes (mínimo, esperado pós-backfill)
- Comment com referência a R-1.3 aplicado

### R-1.4 — apply_segment_membership_diff para de bumpar state_version

**Status:** done
**Owner:** Dev (via Claude Code)
**Concluído em:** 2026-05-02 19:45 UTC
**Estimativa:** 2h (real: ~30min com audit + validações)
**Risco:** médio (função core) — mitigado por audit independente (D-2026-05-02-09)
**Bloqueado por:** R-1.3
**Bloqueia:** R-1.5

**Critérios de aceite (atendidos):**
- ✅ Função `apply_segment_membership_diff` modificada:
  - REMOVE `state_version = state_version + 1` (ambos os UPDATEs — entered + exited)
  - MANTÉM `state_updated_at = now()` (cache invalidation, D-2026-05-02-03)
  - ADICIONA `segment_membership_updated_at = now()` (rastreabilidade dedicada)
- ⚠️ Branch develop pulada via D-2026-05-02-08 — replay de 100 jobs substituído por audit independente de readers (D-2026-05-02-09)
- ✅ Plano de rollback: function preservada como `apply_segment_membership_diff__pre_refactor_2026_05` para backup. Drop após 30 dias com Fase 1 estável (registrar em PROGRESS.md).

**Migration aplicada:**
- `r14_apply_segment_membership_diff_no_state_version_bump` (em `pbfpwfkgjaqotbudihpy`)

**Validações pós-R-1.4:**
1. ✅ 2 funções presentes em `analytics`: `apply_segment_membership_diff` (nova) + `apply_segment_membership_diff__pre_refactor_2026_05` (backup), ambas com mesmo signature
2. ✅ Função nova **não bumpa** `state_version` (regex check em `pg_get_functiondef` retornou `false`)
3. ✅ Função nova **bumpa** `segment_membership_updated_at` (regex check retornou `true`)
4. ✅ Backup preserva comportamento antigo (regex check retornou `true` para `state_version+1`) — rollback funcional

**Audit pré-aplicação (D-2026-05-02-09):** 6 funções SQL bumpam `state_version`, apenas `apply_segment_membership_diff` removida. 2 readers (`populate_contact_state_from_existing`, `trigger_auto_populate_contact_state`) — ambos init/populate, não decisão. Cron fallback usa `state_updated_at` (preservado). Zero readers dependem do bump específico. P4 implementado.

### R-1.5 — Trocar predicate do cron fallback para versão semântica

**Status:** done
**Owner:** Dev (via Claude Code)
**Concluído em:** 2026-05-02 22:44 UTC (migration aplicada) — 1º ciclo validado 22:45 UTC
**Estimativa:** 1h (real: ~30min com cross-check de contrato)
**Risco:** médio (predicate é hot path) — mitigado por backup fiel da versão atual
**Bloqueado por:** R-1.6 (ordem invertida — D-2026-05-02-10)
**Bloqueia:** R-1.7

**Critérios de aceite (atendidos com adaptações de contrato):**
- ✅ Função `run_segment_eval_fallback` modificada:
  - Predicate antigo: `state_updated_at >= now() - interval '15 minutes'`
  - Predicate novo: `segmentation_input_version > last_evaluated_segmentation_version`
- ⚠️ Feature flag `USE_LEGACY_FALLBACK_PREDICATE` substituída por backup nomeado (`run_segment_eval_fallback__pre_refactor_2026_05`). Justificativa: rollback de <1min via DROP+RENAME é equivalente em velocidade e mais simples que feature flag em SQL function (sem branch dynamic em hot path).
- ⚠️ Branch develop pulada via D-2026-05-02-08. Substituída por:
  - Cross-check de contrato pré-aplicação (md5, return shape, guard, permission model — preservados)
  - Validação por execução natural do cron 30s após migration
- ✅ Telemetria adicional: `triggered_by` mudou de `'manual'` para `'repair'` em `segment_eval_queue` para distinguir runs do predicate novo dos antigos em queries históricas.

**Migration aplicada:**
- `20260502224424 r15_run_segment_eval_fallback_uses_watermark` (em `pbfpwfkgjaqotbudihpy`)

**Validações imediatas pós-aplicação (Passo 2):**
1. ✅ 2 funções presentes em `analytics`: `run_segment_eval_fallback` (nova) + `run_segment_eval_fallback__pre_refactor_2026_05` (backup), ambas com `(p_tenant_id uuid) RETURNS jsonb`
2. ✅ Função nova: regex `segmentation_input_version > cs.last_evaluated_segmentation_version` retorna true; `state_updated_at` retorna false; schema do INSERT inalterado; usa `'repair'` como triggered_by
3. ✅ Backup: regex `state_updated_at >= now() - v_window` retorna true, `'manual'` retorna true. md5 backup `920ac6e3db149734c7d81681301156b0` (md5 da função original era `6a41cfe1c50bbeb6def9c9413e4b1196` — diferença explicada pelo nome da função no body)
4. ✅ Cron job 7 (`segment-eval-fallback`, schedule `*/5 * * * *`) inalterado, continua chamando `analytics.run_segment_eval_fallback(id)`

**Validações pós 1º ciclo (Passo 3, run às 22:45 UTC):**
- ✅ Cron rodou após R-1.5 (`status='succeeded'`, `return_message='7 rows'`)
- ✅ Drift global = 0 mantido (42.633 parties, 42.632 com watermark > 0)
- ✅ Fallback enqueue: 0 entries em `fallback_log` no run pós-R-1.5 (predicate semântico em estado limpo + guard `IF v_detected > 0`)
- ✅ Tempo de execução do cron: 37ms para 7 tenants (vs 126–735ms nos 3 runs anteriores) — sinal direto de que predicate filtra muito mais rapidamente

**Validação multi-ciclo (Passo 4 — Checkpoint final 23:32 UTC, 6 ciclos pós-deploy):**
- ✅ Drift global = 0 mantido por 50min consecutivos
- ✅ 0 entries em `fallback_log` ao longo de todos os 6 ciclos (22:45, 23:05, 23:10, 23:15, 23:20, 23:25, 23:30)
- ✅ 100% dos cron runs `status='succeeded'`, todos retornando `7 rows`
- ✅ Tempo de execução por ciclo (ms): 14.6 / 24.5 / 33.3 / 34.6 / 30.5 / 111.0 — mediana 32ms, avg 41ms (outlier de 111ms isolado, ainda 7x melhor que pré-R-1.5)
- ✅ Distribuição `triggered_by` em `segment_eval_queue` (últimos 30min): 0 rows com `'repair'` (predicate filtra antes), 0 rows com `'manual'` (predicate antigo desativado), apenas 1 row natural event-driven com `'purchase'` (sistema event-driven funcionando normal)

**Tendência observada (event_health_metrics.fallback_hits hourly):**
- Baseline sábado (17:00–20:00 UTC): 2–10 hits/h
- Pré-R-1.4 (21:00 UTC): 18.622 hits/h (legado + R-1.4 quebrando bump)
- Pós-R-1.4 + ~15min R-1.5 (22:00 UTC): 10 hits/h
- 23:00 UTC: ETL hourly do bucket popula às 00:05 — esperado próximo de zero baseado em fallback_log direto (zero rows pós-R-1.5)

**Plano de rollback (validado):**
```sql
DROP FUNCTION analytics.run_segment_eval_fallback(uuid);
ALTER FUNCTION analytics.run_segment_eval_fallback__pre_refactor_2026_05(uuid)
  RENAME TO run_segment_eval_fallback;
```
Cron continua chamando `run_segment_eval_fallback` — função restaurada com predicate antigo retoma comportamento pré-R-1.5 em <1min.

**Notas de contrato preservado (vs prompt original):**
- Permission model: mantido `LANGUAGE plpgsql` sem `SECURITY DEFINER` (R-1.5 não escopa mudar perm; trade-off de hardening fica para Sprint 2 se houver demanda específica)
- Guard `IF v_detected > 0 THEN INSERT INTO fallback_log`: preservado (evita rows com zero detected/enqueued, mantém shape histórico de telemetria)
- Return shape: 4 keys preservadas (`tenant_id, contacts_detected, contacts_enqueued, window_minutes`) — `window_minutes` mantido com valor literal `15` para preservar contrato externo, mesmo que a janela não seja mais usada

### R-1.6 — Worker atualiza watermark ao terminar processamento

**Status:** done
**Owner:** Dev (via Claude Code)
**Concluído em:** 2026-05-02 21:50 UTC (deploy Railway) — Checkpoint B fechado 22:00 UTC
**Estimativa:** 2h (real: ~2h30min com testes + cross-check)
**Risco:** médio (hot path do worker)
**Bloqueado por:** R-1.4 (precisa de `segmentation_input_version` populado)
**Bloqueia:** R-1.5 (predicate cron) e R-1.7 (telemetria drift)
**Decisão arquitetural relacionada:** D-2026-05-02-10 (ordem R-1.6 antes de R-1.5)

**Critérios de aceite (atendidos):**
- ✅ `workers/analytics/src/segment-eval-worker.ts` patch:
  - No início do `processEvalJob`: lê `segmentation_input_version` (`observed_version`)
  - Ao terminar com sucesso (após `mark_job_result`): chama `analytics.update_segmentation_watermark(tenant_id, party_id, observed_version)` via helper exportado `advanceSegmentationWatermark`
  - Guard atômico: RPC faz `UPDATE ... WHERE segmentation_input_version = p_observed_version AND last_evaluated_segmentation_version < p_observed_version` (CAS, sem race window)
  - Falha não-fatal: erros da RPC são logados (`segment_eval_watermark_error`) mas não derrubam o job
- ✅ Testes unit cobrindo (`segmentationWatermark.test.ts`, 7 testes):
  - Happy path: observed == current → advanced=true
  - Race condition: outro writer bumpou seg_version antes do worker chamar RPC → advanced=false
  - Idempotência: replay de job já em sync → advanced=false sem erro
  - Falha de conexão na RPC: captura erro, retorna null
  - Resposta vazia (sem rows): retorna null
  - args passthrough (`tenant_id`, `party_id`, `observed_version` literalmente repassados)
  - Coerção bigint-as-string → number (driver postgres.js)

**Migrations aplicadas:**
- `r16_update_segmentation_watermark_rpc` — função `analytics.update_segmentation_watermark(uuid, uuid, bigint) RETURNS jsonb` SECURITY DEFINER

**Validações pós-deploy (Checkpoint B):**

*Cross-check independente via `pg_stat_statements` (Marcio, 22:00 UTC):*
1. ✅ RPC `analytics.update_segmentation_watermark` executada 2x com formato prepared statement = driver postgres.js do worker (não Supabase MCP nem outro caller)
2. ✅ Latência média 5.85ms/call (UPDATE pequeno por PK + SELECT por PK)
3. ✅ Drift global = 0 mantido pós-deploy (integridade preservada)
4. ✅ Cron fallback continua rodando normal (4 runs em 30min) — predicate antigo sem regressão
5. ✅ 0 erros em logs de Postgres relacionados à nova RPC nos primeiros 30min
6. ✅ 0 jobs em status `pending` ou `claimed` durante a janela

*Sanity 15min pós-deploy via Supabase MCP:*
- 9 jobs processados (3 parties distintas — sistema fim-de-semana baixo tráfego)
- 100% dos parties pós-job em `last_evaluated_segmentation_version = segmentation_input_version` (in_sync)
- `state_updated_at` propagou para parties cujo watermark avançou (trigger nativo do `contact_state` continua funcionando — efeito colateral aceitável)
- Latest_processed: 2026-05-02 21:56:08.881351+00

**Plano de rollback:**
- Helper `advanceSegmentationWatermark` é não-fatal: revert do worker → jobs continuam processando, watermark para de avançar (drift cresce, mas funcional). Recovery: cron fallback compensa via predicate atual `state_updated_at`.
- RPC pode ficar instalada mesmo após revert (apenas não é chamada). Se houver razão para drop, `DROP FUNCTION analytics.update_segmentation_watermark(uuid, uuid, bigint)` é seguro pós-revert do worker.

**Notas:**
- Deploy strategy: NÃO escalou Railway para 1 réplica. Justificativa Marcio: sistema quieto, sharding já distribui jobs, watermark tem CAS atômico, falha é não-fatal. Confirmado: 0 conflitos observados.

### R-1.7 — Telemetria de drift em event_health_metrics

**Status:** done
**Owner:** Dev (via Claude Code)
**Concluído em:** 2026-05-03 02:00 UTC
**Estimativa:** 1h (real: ~20min — purely additive)
**Risco:** baixo (zero efeito comportamental — só emite uma métrica nova)
**Bloqueado por:** R-1.5 (done)
**Bloqueia:** Sprint 2 (Fase 2 — Decision Write API)

**Critérios de aceite (atendidos com escopo mínimo):**
- ✅ Nova métrica `eval_drift_count` emitida em `event_health_metrics` por tenant ativo
- ✅ ETL emite 1 row per tenant ativo (incluindo tenants com drift=0, para visualização contínua de saúde)
- ✅ Bucket atual (`date_trunc('hour', now())`) usado como referência (drift é estado "agora", não aggregate de eventos passados)
- ⚠️ View `v_event_driven_health` não atualizada — fora do escopo R-1.7 conforme Marcio (Fase 8 R-8.2 quando definirmos thresholds com base em dados reais)
- ⚠️ Alarmes não criados — fora do escopo R-1.7 (Fase 8)

**Migration aplicada:**
- `r17_etl_event_health_metrics_drift` (em `pbfpwfkgjaqotbudihpy`)

**Mudanças preservadas (contrato externo):**
- `etl_event_health_metrics(p_window_hours integer DEFAULT 2)` mesma signature
- 5 métricas existentes (`fallback_hits`, `event_driven_hits`, `eval_queue_throughput`, `eval_queue_error_rate`, `eval_queue_p95_latency_ms`) preservadas inalteradas
- `emit_health_metric` é UPSERT (`ON CONFLICT (tenant_id, metric_name, bucket_hour) DO UPDATE`) — re-execução do ETL é idempotente
- Cron job 20 (`etl-event-health-metrics`, schedule `5 * * * *`) inalterado

**Validações pós-aplicação (Passo 3, 5/5 ✓):**
1. ✅ ETL manual retornou `drift_emitted: 7`, `drift_breakdown: {tenants_with_drift: 0, tenants_at_zero: 7}`
2. ✅ 7 rows em `event_health_metrics` com `metric_name='eval_drift_count'`, `value=0`, bucket `2026-05-03 02:00:00+00`
3. ✅ `SUM(value) = 0`, `tenants_reporting = 7` no bucket atual
4. ✅ Cross-check com SELECT direto em `contact_state`: `drift_via_query = 0` (bate com soma da métrica)
5. ✅ Cron job 20 inalterado, continua chamando `analytics.etl_event_health_metrics(2)` — próxima execução automática às `*:05 UTC`

**Tenant_ids reportando drift=0:**
- `00000000-0000-0000-0000-000000000001`
- `35047211-dc6c-46dc-9c05-61fc0edc9189`
- `48ef8a5c-283b-4943-8552-53e8f8e92c3a` (Instituto Socrates)
- `5c6db7d1-2e24-4000-af38-7825bda2d3d5`
- `61d64a99-728e-43ff-9a67-ef9547600280`
- `7659b701-66aa-436b-8706-4d634b020ffe`
- `fe793fcd-7564-4d7c-b628-12a25e6d6656`

**Plano de rollback (se necessário):** restaurar definição anterior via `CREATE OR REPLACE`. md5 original: `62dadeaa6b4dcd21da9f8f1919b6d0dd`.

**Critérios de aceite:**
- Nova métrica `eval_drift_count` em `event_health_metrics`
- ETL pulse (`etl_event_health_metrics`) coleta: `COUNT(*) FROM contact_state WHERE segmentation_input_version > last_evaluated_segmentation_version` agrupado por tenant
- View `v_event_driven_health` mostra drift como nova dimensão
- Documentação atualizada em ARCHITECTURE.md §17.1

---

## Fase 2 — Decision Write API

> **D-2026-05-03-01:** R-2.1 quebrada em a/b para reduzir risco de cutover.
> R-2.1a (dry-run + 1 producer paralelo) → R-2.1b (enforce no producer migrado).

### R-2.1a — Decision API em dry-run + process_purchase_analytics paralelo

**Status:** done (aplicação) / **aguardando validação com tráfego natural**
**Owner:** Dev (via Claude Code)
**Concluído em:** 2026-05-03 02:40 UTC (aplicação) — Checkpoint final pendente
**Estimativa:** 2-3h (real: ~30min com cross-check de definição real)
**Risco:** baixo (modo dryrun não escreve em contact_state)
**Bloqueado por:** R-1.7 (done)
**Bloqueia:** R-2.1b

**Critérios de aceite (atendidos):**
- ✅ Função `analytics.apply_contact_state_mutation(uuid, uuid, text, text[], jsonb, smallint, text)` SECURITY DEFINER com modo `'dryrun'` | `'enforce'`
- ✅ Helper `analytics.is_segmentation_relevant_dimension(text)` IMMUTABLE PARALLEL SAFE — whitelist canônica em SQL
- ✅ Tabela `analytics.decision_api_dryrun_log` com TTL 7d via cron categoria A
- ✅ `process_purchase_analytics` chama Decision API em dryrun **paralelamente** (bloco `BEGIN/EXCEPTION WHEN OTHERS`, não-fatal)
- ✅ Path antigo de escrita preservado byte-a-byte (definição base = md5 `55fc1f5d0946955e0c18cb96621dd134`)
- ✅ 4/4 testes unit SQL passing (whitelist, blacklist, mistura, enforce error)
- ⏸️ Validação produção: aguardando tráfego natural (madrugada de domingo BRT — 0 purchases nos últimos 60min)

**Migrations aplicadas:**
- `20260503023813 r21a_apply_contact_state_mutation_dryrun`
- `20260503023928 r21a_process_purchase_analytics_parallel_dryrun`

**Validação imediata (4 testes unit):**
- ✅ Test 1: `dimensions=[frequency, monetary]` → `segmentation_relevant=true, relevant_dims=[frequency, monetary]`
- ✅ Test 2: `dimensions=[churn_score, engagement_score]` → `segmentation_relevant=false, relevant_dims=[]`
- ✅ Test 3: `dimensions=[churn_score, frequency]` → `segmentation_relevant=true, relevant_dims=[frequency]` (filtra blacklist)
- ✅ Test 4: `mode='enforce'` → RAISE EXCEPTION 'enforce mode not implemented in R-2.1a — wait for R-2.1b'

**Validações de integridade (6/6):**
- ✅ Função chama Decision API
- ✅ Usa modo dryrun
- ✅ Tem EXCEPTION WHEN OTHERS (não-fatal)
- ✅ Preserva Step 2b (`contact_product_stats`)
- ✅ Preserva Step 4 (`tenant_distribution` stale)
- ✅ Preserva acquisition tracking (`first_touch_revenue`, `first_product_id/at`)

**Critérios de validação produção (pendentes):**
- N ≥ 1 purchase real chamando função em janela de observação
- `dryrun_calls` cruza com `real_purchase_jobs` (delta < 5%)
- 100% das chamadas com `segmentation_relevant=true` (parity com whitelist)
- Drift global mantém zero
- Zero erros de Decision API em logs Postgres

**Notas de cross-check ao prompt original:**
- Definição de `process_purchase_analytics` no prompt era simplificada e omitia: `products_sequence`, `first_touch_revenue`, `first_product_id/at`, `acquisition_source/utm/at`, Step 2b `contact_product_stats`, Step 4 `tenant_distribution`, transições de `lifecycle_stage` (cooling/at_risk/churned → active). Aplicação usou definição **real** capturada via MCP, preservando 100% do contrato existente.

### R-2.1b — Promover Decision API para enforce em process_purchase_analytics

**Status:** done (aplicação) / **aguardando validação com tráfego real**
**Owner:** Marcio (Decision API enforce + constraint) + Claude Code (cutover producer)
**Concluído em:** 2026-05-03 02:58 UTC
**Estimativa:** 3-4h (real: <30min — Marcio adiantou Decision API enforce em paralelo)
**Risco:** médio — primeiro cutover real de producer
**Bloqueado por:** R-2.1a (done)

**Critérios de aceite (atendidos):**
- ✅ Branch ENFORCE em `apply_contact_state_mutation`:
  - SE `segmentation_relevant`: UPDATE contact_state SET segmentation_input_version=+1, state_updated_at=now() + INSERT segment_eval_queue (triggered_by=p_producer, fields_changed=p_dimensions_changed, priority=p_priority)
  - SENÃO: UPDATE state_updated_at=now() apenas (cache invalidation)
  - **NÃO** bumpa `state_version` legacy (P3 — versão semântica separada)
  - INSERT cego em segment_eval_queue (coalescing real via UNIQUE INDEX vem em R-3.2)
- ✅ Constraint `segment_eval_queue_triggered_by_check` estendida para aceitar nomes de producers (process_purchase_analytics, recordTagChange, recordFormSubmitted, recordImportAdded, refreshFeatures, refreshRfm, refreshReactivation, linkProducts, computeClusters, computeClusterSubgroups, traitProducer, syncPartyType, autoPopulateContactState, r21b_validation_test)
- ✅ `process_purchase_analytics` migrado para `mode='enforce'`:
  - Trocado `'dryrun'` → `'enforce'`
  - **Removido** bloco BEGIN/EXCEPTION WHEN OTHERS — em enforce, falha = abort transação (silenciar criaria mudança perdida)
  - **Removido** `state_version = ... + 1` do UPSERT branch UPDATE
  - **Removido** Step 3 INSERT segment_eval_queue (Decision API já enfileira)
  - **Mantido**: contact_events insert, todos campos de domínio do contact_state UPSERT, contact_product_stats, tenant_distribution stale, state_updated_at=now()
  - INSERT VALUES literal `state_version=1` mantido (default sensível para row nova)
- ✅ Backup `process_purchase_analytics__pre_r21b_2026_05` (cópia da versão R-2.1a)
- ⏸️ Validação produção: aguardando tráfego real (madrugada de domingo BRT — 0 purchases na janela de teste)

**Migrations aplicadas:**
- `20260503025203 r21b_apply_contact_state_mutation_enforce` (Marcio via MCP)
- `20260503025250 r21b_fix_triggered_by_constraint` (Marcio via MCP)
- `20260503025832 r21b_cutover_process_purchase_analytics` (Claude Code via MCP)

**Validações regex pós-cutover (7/7 ✓):**
- ✅ `calls_decision_api`: true
- ✅ `uses_enforce_mode`: true
- ✅ `still_uses_dryrun`: false
- ✅ `still_bumps_state_version`: false
- ✅ `still_has_step3_insert`: false
- ✅ `preserves_state_updated_at`: true
- ✅ `still_has_exception_isolation`: false (removido para garantir consistência)

**Critérios de validação produção (pendentes — aguardando purchase real):**
- 1 row em `segment_eval_queue` com `triggered_by='process_purchase_analytics'`
- `contact_state.segmentation_input_version` incrementado em +1
- `contact_state.state_version` NÃO incrementado (Decision API não bumpa legacy)
- 0 rows em `decision_api_dryrun_log` (não é mais dryrun)
- Drift global mantém 0 (worker R-1.6 alinha em seguida)
- 0 erros em logs Postgres relacionados a `apply_contact_state_mutation`

**Notas técnicas:**
- **Edge case primeira compra:** Decision API roda ANTES do UPSERT em contact_state. Se row inexistente: UPDATE 0 rows (no-op), INSERT segment_eval_queue OK. Path antigo cria row com `segmentation_input_version=0` (default). Worker processa job normalmente; primeira compra "perde" o bump da Decision API mas funcionalmente eval correto. Não-ideal mas não-quebrando — pode ser revisitado em R-2.3+.
- **Decisão de remover EXCEPTION isolation:** divergência consciente do prompt de Marcio ("trocar 1 caractere"). Em enforce, falha silenciosa = mudança perdida (path antigo escreve sem bump nem enqueue). Falha DEVE abortar transação para garantir consistência. Reportado pra Marcio.

**Plano de rollback (validado):**
```sql
DROP FUNCTION analytics.process_purchase_analytics(uuid, uuid, numeric, text, text, uuid, timestamptz, text, text, text, text);
ALTER FUNCTION analytics.process_purchase_analytics__pre_r21b_2026_05(uuid, uuid, numeric, text, text, uuid, timestamptz, text, text, text, text)
  RENAME TO process_purchase_analytics;
```
Função retorna a R-2.1a (Decision API em dryrun paralelo, path antigo escrevendo). <1min.

### R-2.1 (legado — superseded by R-2.1a/b — manter referência)

**Status:** superseded (D-2026-05-03-01)

### R-2.2 — Função analytics.enqueue_segment_eval_canonical

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 2h
**Risco:** baixo
**Bloqueado por:** R-2.1

**Critérios de aceite:**
- Função canônica de enqueue
- UPSERT compatível com unique index (será criado em R-3.2)
- Funciona ANTES e DEPOIS do unique index ser criado (idempotente)
- Documentado padrão de uso

### R-2.3 a R-2.13 — Migrar 12 writers (PRs independentes)

Lista detalhada — cada um vira PR isolado com testes:

- **R-2.3:** ✅ DONE em R-2.1b — `process_purchase_analytics` SQL fn → usa Decision API
- **R-2.4:** `apply_segment_membership_diff` → migração para API (já parcial em R-1.4)
- **R-2.5:** ✅ DONE em 2026-05-04 — `recordTagChange` (workers/historical-sync/src/handlers/webhook/analytics-events.ts:24) migrado para Decision API enforce. triggered_by passou de `'tag_change'` → `'recordTagChange'`. UPDATE+Decision API agora atômicos via `sql.begin()`. Aguardando tag change real em produção para checkpoint.
- **R-2.6:** `recordFormSubmitted` (idem:133)
- **R-2.7:** `recordImportAdded` (idem:198)
- **R-2.8:** `traitProducer.ts:730`
- **R-2.9:** `linkProducts.ts:198`
- **R-2.10:** `refreshFeatures.ts:163,187`
- **R-2.11:** ✅ DONE em 2026-05-05 (R-2.4 do plano operacional) — `refreshRfm.ts` migrado para `analytics.apply_contact_state_mutation_batch` (priority=6, dimensions=`r_score,f_score,m_score,rfm_segment,value_tier,lifecycle_stage`). Atomicidade UPSERT+Decision API por chunk via `sql.begin()`. Bug latente fechado: 9.857 parties/6h estavam invisíveis à segmentação (D-2026-05-05-02). Pattern bulk producer estabelecido para R-2.10 (refreshFeatures) e R-2.12 (refreshReactivation). Aguardando próximo cron run real para checkpoint.
- **R-2.12:** `refreshReactivation.ts:246,337`
- **R-2.13:** `computeClusters.ts:692` + `computeClusterSubgroups.ts:808`
- **R-2.14:** Triggers `trg_sync_party_type` + `trigger_auto_populate_contact_state`
- **R-2.15:** Frontend `analyticsStorage.ts:527` (D-SEG-12)

**Cada um:**
- Estimativa: 2-4h por PR
- Risco: baixo (incremental, isolado)
- Critérios de aceite: writer ad hoc removido, todas escritas via Decision API, testes regredindo passando, telemetria mostra job sendo enfileirado corretamente

### R-2.16 — Trigger BEFORE UPDATE em contact_state (defesa em profundidade)

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 3h
**Risco:** alto (pode bloquear escritas legítimas se mal configurado)
**Bloqueado por:** R-2.3 a R-2.15

**Critérios de aceite:**
- Trigger BEFORE UPDATE que valida origem da escrita via `current_setting('app.write_via_canonical_api', true)`
- Modo "log + permit" durante 7 dias: registra violações em audit table mas permite
- Após 7 dias com zero violações: modo "log + reject"
- Plano de rollback: alterar trigger para "log + permit" em < 1 minuto via configuração

---

## Fase 3 — Coalescing real

### R-3.1 — Adicionar colunas de coalescing em segment_eval_queue

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 1h
**Risco:** baixo

**Critérios de aceite:**
- `ALTER TABLE analytics.segment_eval_queue ADD COLUMN desired_state_version bigint NOT NULL DEFAULT 0`
- `ALTER TABLE analytics.segment_eval_queue ADD COLUMN dirty_dimensions text[] NOT NULL DEFAULT '{}'`
- `ALTER TABLE analytics.segment_eval_queue ADD COLUMN trigger_count int NOT NULL DEFAULT 1`

### R-3.2 — Unique index parcial para coalescing

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 30min (mas exige downtime curto)
**Risco:** médio (criar unique index em tabela grande pode bloquear)
**Bloqueado por:** R-3.1

**Critérios de aceite:**
- `CREATE UNIQUE INDEX CONCURRENTLY seg_eval_queue_active_uniq ON analytics.segment_eval_queue (tenant_id, party_id) WHERE status IN ('pending','claimed')`
- CONCURRENTLY para não bloquear writes
- Validado: nenhum row pendente duplicado existia antes (preventiva)

### R-3.3 — Função canônica usa ON CONFLICT para coalescer

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 2h
**Risco:** baixo
**Bloqueado por:** R-3.2

**Critérios de aceite:**
- `enqueue_segment_eval_canonical` atualizada: `ON CONFLICT (tenant_id, party_id) WHERE status IN ('pending','claimed') DO UPDATE`
- `desired_state_version = GREATEST(existing, new)`
- `dirty_dimensions = array_distinct(existing || new)`
- `trigger_count = existing + 1`
- Testes cobrindo: 10 events same party em janela = 1 row final

### R-3.4 — Worker filtra segmentos por interseção dirty_dimensions ∩ depends_on_fields

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 2h (reduzido — infra de depends_on_fields JÁ existe)
**Risco:** médio (mexe em hot path do worker)
**Bloqueado por:** R-3.3

**Pré-condições já satisfeitas (validado em 2026-05-02 via Supabase MCP):**
- `analytics.segments.depends_on_fields text[]` existe e está populado em 52/78
  segmentos ativos. Os 26 restantes têm array vazio `{}` (sentinela
  `all_contacts`/`all_customers`/`all_leads` ou segmentos pós D-SEG-10 sem
  re-derivação) — recebem skip-filter (sempre avalia).
- Producers de `segment_eval_queue` já emitem `fields_changed text[]` (será renomeado
  para `dirty_dimensions` em R-3.1).

**Critérios de aceite:**
- `workers/analytics/src/segment-rules-evaluator.ts` patch:
  - Antes de avaliar regras de cada segmento: validar `dirty_dimensions ∩
    segment.depends_on_fields ≠ ∅`
  - Se interseção vazia: skip avaliação, registrar em telemetria (`dependency_aware_skip_count`)
- Casos especiais a tratar:
  - `all_contacts` / `all_customers` / `all_leads` (segmentos sentinela): SEM filtro
    de dependência — sempre avalia
  - Segmentos com `depends_on_fields = '{}'` (vazio, ~26 atualmente): SEM filtro —
    sempre avalia (fail-open conservador até R-3.4-followup re-derivar)
  - Segmentos com `depends_on_fields = ['*']` (wildcard, se existir): SEM filtro —
    sempre avalia
- Telemetria: nova métrica `dependency_aware_skip_count` em `event_health_metrics`
- Testes mostrando: mudança em `tag_ids` não dispara reavaliação de segmento RFM puro
- Validação produção: queries comparando `event_driven_hits` antes/depois mostram
  redução proporcional ao % de segmentos não tocados pela dimensão

**Sub-tarefa potencial (R-3.4-followup):** se 26 segmentos com `{}` representarem
gap real (não sentinelas), abrir tarefa para re-derivar `depends_on_fields` desses
segmentos a partir do `rules_json_v2` antes de habilitar filter agressivo.

---

## Fase 4 — Repair worker substitui fallback (macro)

### R-4.1 — Implementar runSegmentRepairCheck em journeys-worker

**Status:** todo
**Estimativa:** 4h
**Risco:** baixo

### R-4.2 — Telemetria repair_hits em event_health_metrics

**Status:** todo
**Estimativa:** 1h

### R-4.3 — Validação 7 dias em paralelo ao fallback

**Status:** todo
**Estimativa:** 7 dias monitoramento

### R-4.4 — Drop pg_cron segment-eval-fallback

**Status:** todo
**Estimativa:** 30min
**Bloqueado por:** R-4.3 com `repair_hits` próximo de zero

### R-4.5 — Drop function run_segment_eval_fallback

**Status:** todo
**Estimativa:** 30min
**Bloqueado por:** R-4.4 + 30 dias

---

## Fase 5 — Audit cobertura event-driven (macro)

### R-5.1 — Mapear triggers em crm/commerce/email/forms/journeys

### R-5.2 — Mapear funções SQL que escrevem em contact_state

### R-5.3 — Mapear Edge Functions com RPC

### R-5.4 — Reportar gaps (eventos sem enqueue)

### R-5.5 — Cada gap vira PR de fix individual

---

## Fase 6 — Worker com fairness real (macro)

### R-6.1 — Refactor claim para ROW_NUMBER OVER PARTITION
### R-6.2 — Limit por tenant por ciclo
### R-6.3 — Métrica tenant_fairness_p99

---

## Fase 7 — Journey trigger por evento real (macro)

### R-7.1 — Trigger AFTER INSERT em contact_events copia para journey_events
### R-7.2 — Drop Edge function check-journey-entries
### R-7.3 — Validar frequency guard R45 em journey actions

---

## Fase 8 — Dead letter + observability completa (macro)

### R-8.1 — Status 'dead' + max_retries em segment_eval_queue
### R-8.2 — Métricas finais em event_health_metrics
### R-8.3 — R52.1 nova: invariantes monitorados
### R-8.4 — Documentação final em ARCHITECTURE.md v8.0

---

**Total:** ~30 tarefas micro detalhadas + entries macro para fases tardias.
