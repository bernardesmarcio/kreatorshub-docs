# Refactor Backlog — Tarefas Atômicas

**Documento companheiro:** [PLAN.md](./PLAN.md), [PROGRESS.md](./PROGRESS.md)

> Lista canônica de tarefas. Cada uma com ID estável, critérios de aceite, owner, status.
> Granularidade: micro nas Fases 1-3 (próximas), macro nas Fases 4-8 (refinaremos quando entrar).
> Status: `todo` / `in_progress` / `done` / `blocked` / `cancelled`

---

## Fase 1 — Estabilização

### R-1.1 — Criar branch develop no Supabase + aplicar migrations base

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 1h
**Risco:** baixo
**Bloqueia:** R-1.2

**Critérios de aceite:**
- Branch `develop` criada via `Supabase:create_branch`
- `project_id` da branch registrado em PROGRESS.md
- Schemas espelhados de produção
- ARCHITECTURE.md, SECURITY.md, DEPENDENCY_MAP.md commitados na branch

### R-1.2 — Migration: adicionar versões semânticas em contact_state

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 30min
**Risco:** baixo
**Bloqueado por:** R-1.1
**Bloqueia:** R-1.3

**Critérios de aceite:**
- `ALTER TABLE analytics.contact_state ADD COLUMN segmentation_input_version bigint NOT NULL DEFAULT 0`
- `ALTER TABLE analytics.contact_state ADD COLUMN last_evaluated_segmentation_version bigint NOT NULL DEFAULT 0`
- `ALTER TABLE analytics.contact_state ADD COLUMN segment_membership_updated_at timestamptz`
- Index novo: `CREATE INDEX contact_state_drift_idx ON analytics.contact_state (tenant_id) WHERE segmentation_input_version > last_evaluated_segmentation_version`
- Migration aplicada em branch develop primeiro
- Validado: queries existentes não regrediram

### R-1.3 — Backfill: zerar drift histórico

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 1h
**Risco:** baixo
**Bloqueado por:** R-1.2
**Bloqueia:** R-1.4

**Critérios de aceite:**
- One-shot UPDATE: `UPDATE analytics.contact_state SET segmentation_input_version = state_version, last_evaluated_segmentation_version = state_version`
- Em branch develop primeiro, validar no banco completo
- Em produção: aplicar fora de janela de pico (final de semana ou madrugada)
- Drift_idx pós-backfill = 0 rows

### R-1.4 — apply_segment_membership_diff para de bumpar state_version

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 2h
**Risco:** médio (função core)
**Bloqueado por:** R-1.3
**Bloqueia:** R-1.5

**Critérios de aceite:**
- Função `apply_segment_membership_diff` modificada:
  - REMOVE `state_version = state_version + 1`
  - MANTÉM `state_updated_at = now()`
  - ADICIONA `segment_membership_updated_at = now()`
- Testes existentes passando
- Validado em branch develop com replay de 100 jobs reais
- Plano de rollback: function preservada como `apply_segment_membership_diff_v1` para backup

### R-1.5 — Trocar predicate do cron fallback para versão semântica

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 1h
**Risco:** médio (predicate é hot path)
**Bloqueado por:** R-1.4
**Bloqueia:** R-1.6

**Critérios de aceite:**
- Função `run_segment_eval_fallback` modificada:
  - Predicate antigo: `state_updated_at >= now() - interval '15 minutes'`
  - Predicate novo: `segmentation_input_version > last_evaluated_segmentation_version`
- Predicate antigo preservado em comentário + feature flag `USE_LEGACY_FALLBACK_PREDICATE` (default false, configurável via Supabase config)
- Validado em branch develop: queries retornam resultado equivalente quando feature flag toggle
- Telemetria adicional: log antes/depois para diff em volume

### R-1.6 — Worker atualiza watermark ao terminar processamento

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 2h
**Risco:** médio
**Bloqueado por:** R-1.5
**Bloqueia:** R-1.7

**Critérios de aceite:**
- `workers/analytics/src/segment-eval-worker.ts` patch:
  - No início do `processEvalJob`: ler `segmentation_input_version` atual
  - Ao terminar com sucesso: `UPDATE contact_state SET last_evaluated_segmentation_version = X WHERE state_version_unchanged_check`
  - Guard: se `segmentation_input_version` mudou durante processamento, NÃO avança watermark (próxima eval refaz)
- Testes unit cobrindo:
  - Watermark avança em processamento normal
  - Watermark NÃO avança se versão mudou durante processamento
  - Watermark NÃO avança em erro

### R-1.7 — Telemetria de drift em event_health_metrics

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 1h
**Risco:** baixo
**Bloqueado por:** R-1.6

**Critérios de aceite:**
- Nova métrica `eval_drift_count` em `event_health_metrics`
- ETL pulse (`etl_event_health_metrics`) coleta: `COUNT(*) FROM contact_state WHERE segmentation_input_version > last_evaluated_segmentation_version` agrupado por tenant
- View `v_event_driven_health` mostra drift como nova dimensão
- Documentação atualizada em ARCHITECTURE.md §17.1

---

## Fase 2 — Decision Write API

### R-2.1 — Função analytics.apply_contact_state_mutation

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 4h
**Risco:** médio (define contrato central)
**Bloqueia:** R-2.2 a R-2.13

**Critérios de aceite:**
- Função criada com signature documentada
- SECURITY DEFINER, search_path explícito
- Whitelist de dimensões registrada como CONST no código + documentada em ARCHITECTURE.md
- Decide bumpa `segmentation_input_version` se dimensão na whitelist
- Sempre bumpa `state_updated_at`
- Decide enfileira em `segment_eval_queue` baseado em prioridade
- Testes unit cobrindo cada combinação de dimension + relevance

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

- **R-2.3:** `process_purchase_analytics` SQL fn → usa Decision API
- **R-2.4:** `apply_segment_membership_diff` → migração para API (já parcial em R-1.4)
- **R-2.5:** `recordTagChange` (workers/historical-sync/src/handlers/webhook/analytics-events.ts:67)
- **R-2.6:** `recordFormSubmitted` (idem:133)
- **R-2.7:** `recordImportAdded` (idem:198)
- **R-2.8:** `traitProducer.ts:730`
- **R-2.9:** `linkProducts.ts:198`
- **R-2.10:** `refreshFeatures.ts:163,187`
- **R-2.11:** `refreshRfm.ts:237`
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

### R-3.4 — Worker considera dirty_dimensions em depends_on_fields

**Status:** todo
**Owner:** Dev (via Claude Code)
**Estimativa:** 3h
**Risco:** médio (mexe em hot path do worker)
**Bloqueado por:** R-3.3

**Critérios de aceite:**
- `workers/analytics/src/segment-rules-evaluator.ts` patch:
  - Filtra segmentos: `dirty_dimensions ∩ depends_on_fields ≠ ∅`
- Testes mostrando: mudança em `tag_ids` não dispara reavaliação de segmentos baseados apenas em RFM
- Telemetria: nova métrica `dependency_aware_skip_count` em `event_health_metrics`

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
