# Refactor Backlog — Tarefas Atômicas

**Documento companheiro:** [PLAN.md](./PLAN.md), [PROGRESS.md](./PROGRESS.md)

> 🏁 **REFACTOR COMPLETO em 2026-05-06.** 6/6 sprints fechadas. Conteúdo
> abaixo dividido em (1) **Pendentes pós-refactor** — items priorizados para
> trabalho futuro, e (2) **Items concluídos (apêndice)** — histórico para
> referência. Detalhes de cada sprint em `PROGRESS.md`. Playbooks
> operacionais em `docs/ARCHITECTURE.md` §22.10 (adicionar producer)
> e §22.11 (troubleshooting).

---

# Pendente pós-refactor

## Alta prioridade

### D-2026-05-06-13 — Sprint 7: Incremental refresh (cron rolling-refresh filtra por state_updated_at)

**Status:** todo
**Estimativa:** 1 sprint dedicada
**Risco:** médio (mexe em hot path de cron)
**Justificativa:** antes de 5M+ contatos. Rolling-refresh hoje recalcula features/RFM/reactivation para todas as parties qualificadas a cada hora. Em escala, vai dominar throughput. Filtro por `state_updated_at > last_run_at` reduz drasticamente o working set (só processa parties com mudança real desde último run).

**Critérios de aceite (preliminares):**
- `analytics.tenant_processing_state` ganha colunas `last_features_at`, `last_rfm_at`, `last_reactivation_at` (já tem) usadas como watermark
- Workers de refresh filtram parties via `WHERE state_updated_at > last_<type>_at`
- Cron run vazio (sem mudanças) retorna em <1s sem trabalho
- Validação: 1 hora de produção com 0 mudanças → 0 jobs enfileirados

## Prioridade normal

- **Decisão de produto:** observability stack externa (Datadog / Grafana / Metabase) — desbloqueada pós-Sprint 6 (views portáveis prontas). Escopo a definir.
- **Cleanup TOTAL_SHARDS / MY_SHARD env vars** (D-2026-05-05-11) — ~30min, remover dead code do `segment-eval-worker.ts` quando 30 dias sem warning.
- **Suíte de testes para workers/historical-sync** (D-2026-05-05-01) — 1 sprint dedicada de testabilidade (Vitest + mock SQL via DI).
- **Unificar evaluators legacy ↔ v2** (D-2026-05-05-03) — 1 sprint dedicada. Migração `rules_json_v2` → `rules_json` para todos os segmentos OU evaluator passa a ler v2 nativamente.
- **Cleanup `state_updated_at` redundante** em `refreshFeatures` UPSERT (D-2026-05-05-04) — ~30min.
- **Canal dedicado para journey notifications** — separar dos 2 usos semânticos do `segment_eval_queue` (D-2026-05-05-06). Investigar event bus / dedicated queue.
- **Encryption credenciais via Supabase Vault** (B-09) — segurança.

## Backlog futuro

- Dashboard system admin (DOCS-2 — sucessor desta task DOCS-1).
- Monitoramento adicional dos health alerts em janela 7-30 dias para calibrar thresholds (`oldest_pending`, `errors`, `dlq_new_24h`).

---

# Items concluídos (apêndice — histórico do refactor)

> Mantido para referência cronológica. Detalhes técnicos completos em
> `PROGRESS.md`. Versão consolidada da arquitetura em `docs/ARCHITECTURE.md`
> §22.4 a §22.9.

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
- **R-2.6:** ✅ DONE em 2026-05-05 (R-2.7 do plano operacional) — `recordFormSubmitted` (workers/historical-sync/src/handlers/webhook/analytics-events.ts:105) migrado para Decision API single em enforce. `triggered_by` passou de `'form_submitted'` → `'recordFormSubmitted'`. UPDATE+Decision API atômicos via `sql.begin()`. Removidos: `state_version+1` e INSERT direto em `segment_eval_queue`. Aguardando próxima form submission real para checkpoint.
- **R-2.7:** ✅ DONE em 2026-05-05 (R-2.8 do plano operacional) — `recordImportAdded` (workers/historical-sync/src/handlers/webhook/analytics-events.ts:169) migrado para Decision API single em enforce. `triggered_by` passou de `'import_added'` → `'recordImportAdded'`. UPDATE+Decision API atômicos via `sql.begin()`. 1 dimensão whitelist (`import_ids`). Priority=6 preservada (background batch). Removidos: `state_version+1`, INSERT direto em `segment_eval_queue`, try/catch externo. Aguardando próximo import real para checkpoint (volume muito baixo, 0 jobs/7d via legacy).
- **R-2.8:** ✅ DONE em 2026-05-05 (R-2.12 do plano operacional) — `traitProducer.ts:727-755` (function `processSubmission`) migrado para Decision API single em enforce com **BULK_ONLY pattern**. Estratégia granular `['party_trait_<key>']` se ≤5 keys, genérico `['party_traits']` caso contrário. Decision API entende BULK_ONLY via `analytics.is_bulk_only_dimension` (bumpa `state_updated_at` + enfileira eval, NÃO bumpa `seg_version` — traits vivem em `crm.party_trait_*`). `triggered_by`: `'form_submission'` → `'traitProducer'`. **Priority=3 preservada** (mais urgente que `recordFormSubmitted=5`; trait change pode disparar regra de segmentação imediata — princípio: sem bug claro, não mudar). Reusa `db.begin` existente (linha 631) sem aninhar transação. Test legacy atualizado para validar chamada Decision API. **TS-side completo** (10/12 + 2 out-of-scope).
- **R-2.9:** ✅ DONE em 2026-05-05 — `linkProducts.ts:175-209` migrado para `analytics.apply_contact_state_mutation_batch` (priority=4, dimensão `bought_product_ids`). Pattern bulk em N parties via `paidPartyIds[]` (sem loop de chunks). Atomicidade UPDATE+Decision API via `sql.begin()`. `triggered_by`: `'product_link'` → `'linkProducts'`. Removidos: `state_version+1` e INSERT direto em `segment_eval_queue`. `state_updated_at` no UPDATE preservado (D-2026-05-05-04). Validação produção assíncrona (volume 0/7d, admin-triggered).
- **R-2.10:** ✅ DONE em 2026-05-05 (R-2.5 do plano operacional) — `refreshFeatures.ts:167-189` migrado para `analytics.apply_contact_state_mutation_batch` (priority=6, 10 dimensões reais — whitelist §22.3 validada). Atomicidade UPSERT+Decision API por chunk (500) via `sql.begin()`. Bug latente fechado: features de domínio (frequency/monetary/recency_days — base do RFM) não enfileiravam eval; agora enfileiram. `state_updated_at` no UPSERT redundante registrado como D-2026-05-05-04 (débito menor). **Validado em produção (commit `befc80e`):** job manual completou em 57.9s, 10.304 jobs `triggered_by='refreshFeatures'` enfileirados, drenando 10-12 jobs/s, 0 errors. D-2026-05-05-05 registrado: orquestração refreshFeatures→refreshRfm gera 2 jobs/party — coalescing natural via R-3.2 vai resolver.
- **R-2.11:** ✅ DONE em 2026-05-05 (R-2.4 do plano operacional) — `refreshRfm.ts` migrado para `analytics.apply_contact_state_mutation_batch` (priority=6, dimensions=`r_score,f_score,m_score,rfm_segment,value_tier,lifecycle_stage`). Atomicidade UPSERT+Decision API por chunk via `sql.begin()`. Bug latente fechado: 9.857 parties/6h estavam invisíveis à segmentação (D-2026-05-05-02). Pattern bulk producer estabelecido para R-2.10 (refreshFeatures) e R-2.12 (refreshReactivation). Aguardando próximo cron run real para checkpoint.

### R-2.4.1 — Adicionar operators faltantes no segment-evaluator

**Status:** ✅ DONE em 2026-05-05
**Contexto:** bug pré-existente exposto por R-2.4. 14 segmentos do tenant
Escola do Fluxo usavam `=` (numeric/text) e `is_not_empty` (text), todos
ausentes do switch case do evaluator. Comportamento anterior: warning
"Unknown operator" + retorna false (segmento nunca evaluado).

**Aplicado:**
- `workers/analytics/src/segment-rules-evaluator.ts` — 3 operators
  adicionados: `'='` (alias de `'equals'`), `'is_empty'`, `'is_not_empty'`
- 6 testes novos em `segment-rules-evaluator.test.ts` (55/55 passing,
  zero regressão dos 49 prévios)
- typecheck ✓, build ✓

**Cross-check pós-deploy (2026-05-05):** mapeamento completo revelou
2 evaluators paralelos (D-2026-05-05-03 — coexistência aceita como
débito). Path v2 (SQL canonical via `refresh_segment_parties_unified`)
já cobria os operators dos 3 segmentos nativamente. Path legacy
(memória, hot path) era o único com gap, fechado por d460945. Validação
em produção: 14 segmentos do Escola do Fluxo com `last_calculated_at`
recente, membership consistente entre `segments` counts e
`segment_parties`. **R-2.4.1 fechado sem trabalho novo.**

Adicionalmente: threshold slow_job ajustado 200ms → 1500ms (commit
`0a9c50e`) para restaurar signal-to-noise pós-Fase 1.
- **R-2.12:** ✅ DONE em 2026-05-05 (R-2.6 do plano operacional) — `refreshReactivation.ts` migrado em **2 escopos**: (1) UPSERT principal de scoring (chunks de 1000) e (2) `cleanReactivatedCustomers` (UPDATE de cleaning), ambos com Decision API batch atômico via `sql.begin()`. 1 dimensão whitelist (`reactivation_priority`); `reactivation_score` é metadata interno (ranking) fora da whitelist. Priority=6, `triggered_by='refreshReactivation'` em ambos. Bug latente fechado em 2 frentes (UPSERT + cleaner não enfileiravam eval). **Sprint 2 milestone — 3/3 producers bulk migrados** (refreshRfm + refreshFeatures + refreshReactivation). **Validado em produção (commit `6fa8abe`):** job manual completou em 42s, 5.044 jobs `triggered_by='refreshReactivation'` enfileirados com `fields_changed='{reactivation_priority}'`, 0 errors.
- **R-2.13:** ⛔ OUT-OF-SCOPE em 2026-05-05 (R-2.10 + R-2.11 do plano operacional) — `computeClusters.ts:691-704` + `computeClusterSubgroups.ts:807-820`. **Sem mudança de código.** Esses producers não escrevem em `contact_state`; apenas notificam journey engine via `segment_eval_queue` com `fields_changed = ['segment_ids']` (output, não input — fora da whitelist §22.3 por design). Migrar literalmente causaria regressão funcional (Decision API descartaria INSERT). Rename cosmético sem ganho. Decisão registrada em D-2026-05-05-06: `segment_eval_queue` tem 2 usos semânticos distintos (re-eval por mudança de input vs notificação cross-system). Refactor para canal dedicado fica como item futuro fora do escopo Sprint 2.
- **R-2.14:** ✅ DONE em 2026-05-05 (cross-check MCP) — Triggers `sync_party_type`/`sync_party_types_batch` + `trigger_auto_populate_contact_state`. **Migração histórica não-documentada** descoberta na closure da Sprint 2. Validado via `pg_get_functiondef` em 2026-05-05: ambas chamam `apply_contact_state_mutation`, removeram `state_version+1` (no caso de `auto_populate`, `state_version=1` no INSERT branch é inicialização, não bump — comentário explícito "REMOVIDO: state_version+1" no UPDATE), removeram INSERT direto em `segment_eval_queue`. D-2026-05-05-07 registrado.
- **R-2.15:** Frontend `analyticsStorage.ts:527` (D-SEG-12) — *fora do escopo de Decision Write API: arquivo passa `triggered_by` em payload de RPC, não escreve direto na queue. Mantido na lista por compatibilidade com plano original.*

### R-2.13.b — Producers SQL adicionais migrados (cross-check MCP)

**Status:** ✅ DONE em 2026-05-05 — descoberta via cross-check MCP na closure da Sprint 2

Funções SQL identificadas no inventário como pendentes mas que **já estavam
migradas** silenciosamente (provavelmente via `apply_migration` em algum
momento da sessão). Validadas via `pg_get_functiondef` em 2026-05-05:

| Função | Schema | Decision API call |
|---|---|---|
| `sync_tag_ids_to_contact_state` | analytics | `'recordTagChange'`-like (chama API, não bumpa state_version, não INSERT direto) |
| `record_email_engagement_event` | email | `'recordEmailEngagement'` (priority=3, enforce, dimensão `last_email_*_at`) |
| `trg_product_name_mapped` | commerce | chama API |

Decisão: marcar como done sem código novo. **Custo evitado:** ~3 prompts
R-2.x duplicando migração já feita. Lição registrada em D-2026-05-05-07.

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

### R-3.1 + R-3.2 — Coalescing real (UNIQUE INDEX parcial + UPSERT merge)

**Status:** ✅ DONE em 2026-05-05 (aplicado via MCP em migration única, ~30min, sem patch TS — D-2026-05-05-08)

**Aplicado:**
- **UNIQUE INDEX parcial** `idx_seg_eval_queue_pending_unique_party ON
  analytics.segment_eval_queue (tenant_id, party_id) WHERE status='pending'`
  — escopo simplificado vs. plano original (`status IN ('pending','claimed')`):
  apenas `pending` é coalescível, `claimed` representa job já em processamento
- **`apply_contact_state_mutation` + `apply_contact_state_mutation_batch`**
  reescritas com `ON CONFLICT (tenant_id, party_id) WHERE status='pending'
  DO UPDATE` que mescla:
  - `fields_changed` = união DISTINCT ordenada
  - `priority = LEAST(EXCLUDED.priority, segment_eval_queue.priority)` (mais
    urgente vence)
  - `triggered_by = 'coalesced'` (sinaliza merge para observability)
  - `retry_count` reset para 0 (job efetivamente novo)
  - `created_at` preservado (FIFO honesto)
- Constraint `segment_eval_queue_triggered_by_check` estendida para aceitar `'coalesced'`

**Decisão:** colunas `desired_state_version` / `dirty_dimensions` / `trigger_count`
do plano original **não foram criadas** — `fields_changed` (já existia) absorve a
função de `dirty_dimensions`; `priority` LEAST + FIFO via `created_at` resolve o
papel de `desired_state_version` na prática. `trigger_count` cabe como métrica
futura se a observability pedir.

**Validação E2E (via MCP, 2026-05-05):**
- 2 inserts seguidos (refreshFeatures + refreshRfm) mesmo party → 1 row pending
- `fields_changed = ['frequency','monetary','r_score','recency_days']` (união)
- `priority = 3` venceu sobre `priority = 5`
- `was_coalesced = true` retornado pela Decision API
- Worker R-1.6 processou em 584ms sem erro

**Resolve débitos:**
- ✅ D-2026-05-05-05 (refreshFeatures orquestra refresh_rfm — 2 jobs/party)
- ✅ Edge case 2x-3x jobs/party de R-2.5/R-2.6
- ✅ Edge case BULK_ONLY de R-2.12 (form com múltiplos blocos)

**Beleza arquitetural:** 0 mudança em código TS (D-2026-05-05-08). Sprint 2
preparou o terreno: tudo passa pelo único INSERT que agora coalesce.

### R-3.3 — Função canônica usa ON CONFLICT para coalescer

**Status:** ✅ DONE em 2026-05-05 — **absorvido por R-3.1+R-3.2** (aplicação única via MCP)

A spec original previa essa tarefa como passo separado, mas a aplicação MCP
fez `apply_contact_state_mutation` / `_batch` adotarem o `ON CONFLICT DO UPDATE`
merge na mesma migration de R-3.1+R-3.2. Não há `enqueue_segment_eval_canonical`
separada — Decision API É a função canônica desde Sprint 2.

Adaptações vs spec original:
- `desired_state_version = GREATEST` substituído por **`priority = LEAST`** + FIFO
  via `created_at` (semântica equivalente para o caso de uso real: mais urgente
  vence, ordem temporal preservada)
- `dirty_dimensions = array_distinct(existing || new)` mapeado para
  **`fields_changed = união DISTINCT ordenada`** (`fields_changed` já existia,
  evitou criar coluna nova)
- `trigger_count` **não criado** — fica como métrica futura se observability pedir

Testes cobrindo "10 events same party em janela = 1 row final" validados via
MCP em produção (2 events → 1 row pending).

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

## Fase 4 — Repair via coalescing (escopo redimensionado — D-2026-05-05-09)

**🎯 Sprint 4 FECHADA em 2026-05-06 via mudança cirúrgica em
`run_segment_eval_fallback`. Plano original (worker dedicado +
4 sub-tarefas R-4.1 a R-4.5) foi substituído por 1 migration MCP única.**

**Por que redimensionado:**
- Pré-Sprint 3: cron fallback detectava ~120k parties/dia (justificava worker)
- Pós-Sprint 3: 20 jobs/dia (causa raiz do drift transitório morta pelas
  Sprints 1+2+3)
- Worker dedicado seria overkill para volume edge raro

### R-4.1 — Repair via coalescing pattern

**Status:** ✅ DONE em 2026-05-06 (aplicado via MCP, 1 migration cirúrgica)

**Aplicado:**
- `analytics.run_segment_eval_fallback` reescrita: INSERT direto em queue →
  **`INSERT ... ON CONFLICT DO UPDATE`** (mesmo merge pattern de R-3.2)
- `fields_changed`: lista hardcoded de 13 dimensões → **`['repair_full_eval']`**
  (sinal especial para o evaluator significando "re-avalia tudo, não filtra
  por interseção `depends_on_fields`")
- `priority = 9` preservada (baixa urgência, última a processar)
- `triggered_by = 'repair'` preservado (identifica origem)

**Validação E2E (via MCP, 2026-05-06):**
- Drift artificial criado em 1 party
- `run_segment_eval_fallback()` detectou
- Job enfileirado com `triggered_by='repair'`, `fields_changed=['repair_full_eval']`
- Worker processou em 568ms, drift drenado para 0
- Sprint 3 não quebrado: 10.639 jobs `coalesced` em 2h continuam aparecendo

**Decisão arquitetural (D-2026-05-05-09):** repair NÃO usa Decision API porque
não muta `contact_state` (P2 aplica-se a contact_state, não a queue inserts).
Mas usa o mesmo coalescing pattern via `ON CONFLICT DO UPDATE` direto.

### R-4.2 a R-4.5 — Substituídos pelo R-4.1

**Status:** ⛔ NOT DONE — escopo absorvido / não-aplicável após Sprint 1+2+3

- **R-4.2 (telemetria repair_hits):** parcialmente coberto via
  `event_health_metrics` que já existe. Métrica granular pode ser adicionada
  como item futuro se a observability pedir.
- **R-4.3 (validação 7 dias paralelo):** não-aplicável — não há worker novo
  rodando em paralelo, apenas a função SQL existente foi reescrita.
- **R-4.4 (drop pg_cron segment-eval-fallback):** **não fazer** — fallback
  agora é o único caminho de repair (proporcional ao volume real). Drop seria
  perder a safety net.
- **R-4.5 (drop function run_segment_eval_fallback):** **não fazer** — função
  permanece, agora alinhada com pattern de coalescing.

---

## Fase 5 — Audit cobertura event-driven (macro)

### R-5.1 — Mapear triggers em crm/commerce/email/forms/journeys

### R-5.2 — Mapear funções SQL que escrevem em contact_state

### R-5.3 — Mapear Edge Functions com RPC

### R-5.4 — Reportar gaps (eventos sem enqueue)

### R-5.5 — Cada gap vira PR de fix individual

---

## Fase 6 — Worker com fairness real (macro)

**🎯 Sprint 5 do plano operacional aplicada em 2026-05-06**: SQL via MCP +
patch TS commitado. R-6.1 done via implementação two-phase (round-robin +
FIFO). R-6.2/R-6.3 absorvidos / opcionais.

### R-6.1 — Claim com fairness (two-phase: round-robin + FIFO global)

**Status:** ✅ DONE + VALIDADO EM PRODUÇÃO em 2026-05-06 (Sprint 5 do plano operacional)

**Implementação (D-2026-05-05-10):** algoritmo em SQL ao invés de
ROW_NUMBER OVER PARTITION em TS. `analytics.claim_next_segment_jobs_fair(worker_id, batch_size)`:

- **Phase 1 — round-robin:** `batch_size / tenant_count_active` por tenant ativo
- **Phase 2 — FIFO global:** preenche capacity restante por ordem de `created_at`
- Lock via `FOR UPDATE SKIP LOCKED` (compatível multi-replica)
- View `analytics.v_queue_fairness_status` para observability em tempo real

**Patch TS aplicado (`workers/analytics/src/segment-eval-worker.ts`, commit `d93bc65`):**
- Removido loop `for (tenant of tenants) claim(tenant_id)` + query de tenants
  por hash sharding
- Substituído por claim único via `claim_next_segment_jobs_fair(WORKER_ID, BATCH_SIZE)`
- TOTAL_SHARDS/MY_SHARD preservados como dead code com warning (D-2026-05-05-11)
- Logger `fair_claim_received` para validação pós-deploy

**Validação E2E em produção (2026-05-06):** burst de 500 jobs Escola + 28 SaudeNow simultâneos.

| Métrica | Pré-Sprint 5 | Pós-Sprint 5 | Improvement |
|---|---|---|---|
| Wait máximo Escola | 1189s (19.8 min) | 76s | **~16x** |
| Wait máximo SaudeNow | ~5 min | 31s | **~10x** |
| SaudeNow drenagem | sequencial atrás Escola | **paralelo** | anti-starv ✓ |
| Throughput total | 70 jobs/min | ~430 jobs/min | **~6x** |

Throughput 6x veio de eliminar overhead artificial do sharding (D-2026-05-05-11).

### R-6.2 — Limit por tenant por ciclo

**Status:** ✅ ABSORVIDO por R-6.1. A função `claim_next_segment_jobs_fair`
implementa limit per-tenant via Phase 1 (`batch_size / tenant_count_active`).

### R-6.3 — Métrica tenant_fairness_p99

**Status:** parcial — `analytics.v_queue_fairness_status` cobre `oldest_pending_seconds`
per-tenant em tempo real. Métrica histórica p99 fica como item futuro
(observability complementar) — pode ser absorvido pela Sprint 6 (dead letter +
observability).

---

## Fase 7 — Journey trigger por evento real (macro)

### R-7.1 — Trigger AFTER INSERT em contact_events copia para journey_events
### R-7.2 — Drop Edge function check-journey-entries
### R-7.3 — Validar frequency guard R45 em journey actions

---

## Fase 8 — Dead letter + observability portátil

**🎯 Sprint 6 do plano operacional FECHADA em 2026-05-06** (~30min via MCP,
**0 patch TS**, 3 migrations cirúrgicas). Escopo redimensionado de "worker
dedicado + dashboards integrados" para "safety net + views portáveis" —
mesmo padrão de redução da Sprint 4 (D-2026-05-05-09): problema original
encolheu após Sprints 1-5 consolidarem o pipeline (0 errors/7d, 0 retries
esgotados).

### R-8.1 — Dead Letter Queue (safety net)

**Status:** ✅ DONE em 2026-05-06

- `analytics.dead_letter_queue` (clone schema das duas queues)
- `analytics.move_to_dead_letter(queue text, job_id bigint)` — entrada manual
  para jobs irrecuperáveis identificados via inspeção
- TTL 90 dias via cron purge dedicado
- View `analytics.v_dlq_summary` para inspeção rápida

### R-8.2 — Health Views (observability portátil)

**Status:** ✅ DONE em 2026-05-06

4 views SQL portáveis (qualquer dashboard via `SELECT`):
- `v_pipeline_status` — snapshot multidimensional
- `v_queue_health` — throughput / latency / errors em janelas (5m / 15m / 24h)
- `v_drift_by_tenant` — drift summary per-tenant
- `v_worker_throughput` — performance por worker (p50/p95/p99 latency)

### R-8.3 — Health Alerts

**Status:** ✅ DONE em 2026-05-06

`analytics.check_health_alerts()` retorna `jsonb` com violações detectadas.
Limites configurados:

| Métrica | Threshold | Severity |
|---|---|---|
| `oldest_pending` | > 5 min | warning |
| `errors` | > 10/h | critical |
| `dlq_new_24h` | > 0 | warning |
| `throughput` (com fila) | < 10/min | critical |
| `active_workers` (com fila) | < 3 | critical |

Cron a cada 5min loga alertas críticos em `analytics.health_alert_log` (TTL 30d).

**Validação real-time pós-deploy:** sistema detectou warnings ATIVOS
imediatamente (esperado durante drenamento de burst rolling-refresh):
queue_backlog (oldest 521s), drift_high (5172 > 1000), `has_critical=false`,
`jobs_processed_5m=2961` (~600/min), `coalescing_rate_pct=94.8%`.

### R-8.4 — Documentação final em ARCHITECTURE.md

**Status:** ✅ DONE em 2026-05-06 — bump v7.33 → v8.0 (marcador de refactor
completo). §22.8 (Sprint 6) + §22.9 (REFACTOR COMPLETO — métricas finais).

### Out-of-scope — decisão de produto

Integração com observability stack externa (Datadog / Grafana / Metabase /
similar) é decisão de produto, não de banco. Views `v_*` são portáveis para
qualquer dashboard via `SELECT`. Worker logs estruturados já existem (Railway).

---

# 🏁 REFACTOR COMPLETO em 2026-05-06

**6/6 sprints fechadas** entre 2026-05-02 e 2026-05-06. Ver
`docs/refactor/PROGRESS.md` para métricas pré/pós e D-2026-05-05-12 (lição
arquitetural sobre custo upfront de centralização).

**Backlog futuro (não-bloqueador):**
- Decisão observability stack (Datadog / Grafana / Metabase) — produto
- Cleanup TOTAL_SHARDS env vars (D-2026-05-05-11)
- Suíte de testes para workers/historical-sync (D-2026-05-05-01)
- Unificar evaluators legacy (AST v1) ↔ v2 (AST v2) (D-2026-05-05-03)
- Remover `state_updated_at` redundante de refreshFeatures UPSERT
  (D-2026-05-05-04)
- Investigar canal dedicado para journey notifications (separar dos 2 usos
  semânticos do `segment_eval_queue` — D-2026-05-05-06)

---

**Total:** ~30 tarefas micro detalhadas + entries macro para fases tardias.
