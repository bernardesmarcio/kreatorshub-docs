# Plano de Refatoração — Segmentation Pipeline

**Versão:** 1.0
**Iniciado:** 2026-05-02
**Status:** Aprovado, aguardando início Sprint 1
**Owner:** Marcio (architectural decisions) + Claude (orchestration) + Claude Code (execution)
**Janela alvo:** 6 sprints (~12 semanas)
**Documento companheiro:** [PROGRESS.md](./PROGRESS.md), [BACKLOG.md](./BACKLOG.md)

---

## 1. Contexto e motivação

A funcionalidade de segmentação é o coração da plataforma KreatorsHub. Define quem recebe campanhas, quem entra em journeys, quem é exportado para audiences pagas, quem aciona automações.

Em 4 dias de diagnóstico estruturado (29/04 a 02/05/2026), identificamos que o pipeline tem débito arquitetural crônico mascarado por convergência eventual e por safety nets (cron fallback) introduzidos sem registro arquitetural.

Estado em 02/05/2026, medido em produção:

- 100% dos jobs de segmentação nas últimas 6h vieram do cron fallback (`triggered_by='manual'`)
- 1.845.469 hits do fallback nos últimos 7 dias (P50 8.6k/h, P95 26.4k/h, max 36.8k/h por hora — medido em `analytics.fallback_log`)
- ~11.000 parties tiveram mudança real de estado em 24h, mas 275.402 enqueues foram feitos pelo cron — amplificação de ~25x
- Latência média de 7 minutos para propagação de mudança em segmentação (avg wait pre-claim = 431s medido em `segment_eval_queue.claimed_at - created_at`)
- Caminho event-driven projetado na constituição §4.3 está praticamente desligado em produção (em 24h, 1 job `triggered_by='tag_change'` versus 37.483 `triggered_by='manual'`)

Em escala alvo (50.000 tenants, 50 milhões de parties), os mesmos parâmetros geram pressão de banco insustentável. **A arquitetura atual não escala** sem essa refatoração.

## 2. Visão arquitetural alvo

A plataforma é um motor de decisão orientado a estado (constituição §3). Mutações de estado real geram trabalho real para segmentação. Trabalho é coalescido por party. Avaliação respeita watermark per-party. Membership é projeção, não input. Workers competem em fairness global. Observability é camada própria.

**Em uma linha:** segmentação deixa de ser dirigida por timestamp de snapshot e passa a ser dirigida por versão semântica de input com fila coalescida por party.

**Em duas linhas:** mutações de estado e enqueues de avaliação são emitidos por funções canônicas; nenhum writer ad hoc encosta em `contact_state` ou `segment_eval_queue`.

## 3. Princípios não-negociáveis durante a refatoração

Estes princípios são lei. Toda PR que viola um deles requer ADR (Architecture Decision Record) explícito antes de merge.

**P1 — Single writer per state column.** Cada coluna de `analytics.contact_state` tem exatamente uma rota canônica de escrita. Sem exceção.

**P2 — Decision Write API obrigatório.** Toda mutação em `contact_state` passa por função SQL canônica. Defesa em profundidade via trigger BEFORE UPDATE bloqueando escrita ad hoc.

**P3 — Versão semântica por dimensão.** `segmentation_input_version` separado de `state_version` separado de outras versões futuras. Cada consumer tem seu watermark.

**P4 — Membership é projeção, não input.** `apply_segment_membership_diff` não bumpa `segmentation_input_version`. Diff de membership não retroalimenta segmentação.

**P5 — pg_cron é só housekeeping.** Conforme R50. Lógica de domínio vive em worker.

**P6 — Worker > cron sempre que houver tenant awareness.** Loop bloqueante de tenants em PG é proibido.

**P7 — Eventos são imutáveis e first-class.** `contact_events` é fonte da verdade temporal. Tudo é derivado dele.

**P8 — Dependency-aware evaluation.** Worker só reavalia segmentos cuja dependência intersecta com o que mudou.

**P9 — Coalescing real via UNIQUE constraint.** Fila tem `(tenant_id, party_id) WHERE status active`. UPSERT vira contrato.

**P10 — Observability é camada própria.** Conforme R52. Telemetria não acoplada a tabelas operacionais.

## 4. Critérios de sucesso da refatoração

Mensuráveis. Validáveis via telemetria de D-CRON-3.

| Métrica | Valor atual (02/05) | Valor alvo (fim da Fase 6) |
|---|---|---|
| `fallback_hits` per dia | ~270.000 | < 1.000 (apenas drift residual) |
| `event_driven_hits` per dia | ~0 | > 90% do total |
| Latência média de propagação | 7 minutos | < 30 segundos |
| Workers ad hoc escrevendo `contact_state` | 12 | 0 (apenas via Decision Write API) |
| Producers de `segment_parties` | 6 | 1 (apenas via canonical fn) |
| pg_cron com lógica de domínio | 2 (categoria B) | 0 |
| Latência p99 de claim por tenant | desconhecido | < 60 segundos |

## 5. As 8 fases

### Fase 1 — Estabilização (1 sprint, baixo risco)

**Objetivo:** parar a hemorragia de amplificação sem quebrar nada. Fixes aditivos com rollback trivial.

**Entrada:** plano aprovado.

**Escopo:**
- Adicionar `segmentation_input_version` em `contact_state`
- Adicionar `last_evaluated_segmentation_version` em `contact_state`
- Backfill: ambas começam iguais a `state_version` atual
- Trocar predicate do cron fallback de `state_updated_at` para versão semântica
- Patch worker: atualizar watermark ao terminar processamento
- `apply_segment_membership_diff` para de bumpar `state_version` (passa a bumpar apenas `state_updated_at` + novo `segment_membership_updated_at`)
- Telemetria nova em `event_health_metrics`: métrica `eval_drift_count`

**Saída:** redução medida de ≥70% em `fallback_hits`. Métricas estáveis.

**Risco:** baixo. Migration aditiva, watermark COALESCE em backward compat. Predicate antigo permanece disponível via feature flag durante 7 dias para rollback instantâneo.

**Rollback:** SQL para restaurar predicate antigo + DROP COLUMN das versões novas. Tempo: < 5 minutos.

**Tarefas:** R-1.1 a R-1.7 no BACKLOG.

### Fase 2 — Decision Write API (1 sprint, médio risco)

**Objetivo:** eliminar 12 writers ad hoc. Único contrato de escrita.

**Entrada:** Fase 1 estável por 7 dias.

**Escopo:**
- Função `analytics.apply_contact_state_mutation()` canônica
- Função `analytics.enqueue_segment_eval_canonical()` canônica
- Migrar 12 writers para usar APIs canônicas (PR independente por writer ou cluster de writers)
- Trigger BEFORE UPDATE em `contact_state` bloqueando escrita ad hoc (defesa em profundidade)
- Whitelist explícita de "dimensões relevantes para segmentação" registrada em ARCHITECTURE.md

**Saída:** zero escritas em `contact_state` fora da Decision API. Validado via trigger em modo "log + permit" durante 7 dias, depois "log + reject".

**Risco:** médio. Mexe em hot path de ingestion. Mitigação: testes de regressão; staging com replay de 24h de produção; deploy escalonado por worker.

**Rollback:** trigger volta para modo "log + permit" (loga violação mas não bloqueia) se for detectada regressão pós-cutover ou writer crítico não migrado em produção. Cada writer migrado é PR independente — rollback de 1 writer não afeta outros.

**Tarefas:** R-2.1 a R-2.13 no BACKLOG (12 writers + função canônica).

### Fase 3 — Coalescing real na queue (1 sprint, baixo risco)

**Objetivo:** idempotência estrutural. UPSERT vira contrato.

**Entrada:** Fase 2 estável por 7 dias.

**Escopo:**
- Adicionar colunas em `segment_eval_queue`: `desired_state_version`, `dirty_dimensions`, `trigger_count`
- `CREATE UNIQUE INDEX seg_eval_queue_active_uniq ON (tenant_id, party_id) WHERE status IN ('pending','claimed')`
- Função canônica usa `ON CONFLICT (tenant_id, party_id)` para colapsar N eventos em 1 row
- Worker considera `dirty_dimensions` na decisão de quais segmentos avaliar (intersecção com `depends_on_fields`)

**Saída:** N eventos no mesmo party = 1 row pendente. Validado via queries de integridade.

**Risco:** baixo. Migração aditiva da queue + função canônica.

**Rollback:** DROP UNIQUE INDEX + reverter função canônica.

**Tarefas:** R-3.1 a R-3.4 no BACKLOG.

### Fase 4 — Repair worker substitui fallback (1 sprint, baixo risco)

**Objetivo:** D-CRON-4 fechado. R50 enforcement de verdade.

**Entrada:** Fase 3 estável por 7 dias + Fase 5 (audit) com gaps fechados.

**Escopo:**
- Implementa `runSegmentRepairCheck` em `journeys-worker.runScheduler` (cadence 60s)
- Query intencional: drift sustentado (`segmentation_input_version > last_evaluated_segmentation_version AND state_updated_at < now() - 5min`)
- Limit por tenant para fairness
- Telemetria nova: `repair_hits` em `event_health_metrics`
- Após 7 dias com `repair_hits` próximo de zero: DROP cron `segment-eval-fallback` + DROP function `run_segment_eval_fallback`

**Saída:** `fallback_hits` cessa. `repair_hits` baixo e estável.

**Risco:** baixo — repair worker roda em paralelo ao fallback inicialmente. Drop só após confirmação.

**Rollback:** reativar cron via migration. Função SQL preservada em backup por 30 dias antes do DROP definitivo.

**Tarefas:** R-4.1 a R-4.5 no BACKLOG.

### Fase 5 — Audit cobertura event-driven (paralela a Fases 2-3, P0)

**Objetivo:** garantir que todos os eventos de domínio enfileiram em `segment_eval_queue`. Sem isso, eliminar fallback é arriscado.

**Entrada:** Fase 1 deployada (telemetria de drift disponível).

**Escopo:**
- Mapear todos os triggers em `crm.*`, `commerce.*`, `email.*`, `forms.*`, `journeys.*`
- Mapear todas as funções SQL que escrevem em `contact_state`
- Mapear todas as Edge Functions que chamam RPCs que mutam estado
- Para cada um, validar enqueue em `segment_eval_queue` ou via Decision Write API
- Reportar gaps (eventos que mutam estado mas não enfileiram eval)
- Cada gap vira fix calibrado independente

**Saída:** telemetria mostra `event_driven_hits` ≥ 70% do total de jobs. Lista zerada de gaps conhecidos.

**Risco:** baixo (audit sem mexer em produção até gaps virarem fix individual).

**Rollback:** N/A — fase de audit + fixes pequenos isolados.

**Tarefas:** R-5.1 a R-5.5 no BACKLOG (macro — refinaremos quando entrar).

### Fase 6 — Worker com fairness real (1 sprint, médio risco)

**Objetivo:** eliminar starvation tenant-grande-bloqueia-tenant-pequeno.

**Entrada:** Fases 1-4 estáveis.

**Escopo:**
- Refatorar `runSegmentEvalLoop` para claim global com `ROW_NUMBER() OVER (PARTITION BY tenant_id)`
- Limit por tenant por ciclo (max 10 jobs/tenant/round)
- Múltiplos workers paralelos sem coordenação manual
- Métrica nova: `tenant_fairness_p99` em `event_health_metrics`

**Saída:** p99 de wait time pre-claim por tenant < 60 segundos.

**Risco:** médio (mexe no claim path).

**Rollback:** reverter SQL do claim para versão atual per-tenant.

**Tarefas:** R-6.1 a R-6.3 no BACKLOG (macro).

### Fase 7 — Journey trigger por evento real (1 sprint, médio risco)

**Objetivo:** matar Edge function `check-journey-entries`. `segment_entered` em `contact_events` vira trigger first-class.

**Entrada:** Fase 4 estável.

**Escopo:**
- Trigger AFTER INSERT em `analytics.contact_events` que copia `segment_entered`/`segment_exited` para `journeys.journey_events` quando há journey ativo
- Edge function `check-journey-entries` é dropada
- Frequency guard em journey actions (validar R45 existente)

**Saída:** Edge function fora do código. Latência de journey trigger por segment < 5 segundos.

**Risco:** médio (mexe em path de journeys).

**Rollback:** reativar Edge function (preservada em git).

**Tarefas:** R-7.1 a R-7.3 no BACKLOG (macro).

### Fase 8 — Dead letter + observability completa (1 sprint, baixo risco)

**Objetivo:** observabilidade completa do pipeline. Invariantes monitorados.

**Entrada:** Fases 1-7 estáveis.

**Escopo:**
- Status `'dead'` em `segment_eval_queue` + `max_retries`
- Worker move jobs com `retry_count >= max_retries` para dead
- Métricas finais em `event_health_metrics`: `claim_latency`, `eval_latency`, `skip_rate` (jobs ignorados por watermark `last_evaluated_segmentation_version >= desired`), `diff_useful_rate` (frações de eval que produziram entered/exited > 0), `trigger_amplification_ratio` (`fallback_hits` / `event_driven_hits`, alvo < 0.1 pós-Fase 6)
- R52.1 nova: invariantes monitorados via D-CRON-3 view

**Saída:** painel completo de saúde do pipeline. Alertas configurados.

**Risco:** baixo (aditivo).

**Rollback:** drop das colunas/métricas novas.

**Tarefas:** R-8.1 a R-8.4 no BACKLOG (macro).

## 6. Dependências entre fases

```
Fase 1 ───────┬─── Fase 5 (paralela, P0)
              │
              ├─── Fase 2 ─── Fase 3 ───┬─── Fase 4
              │                          │
              └──────────────────────────┴─── Fase 6
                                         │
                                         ├─── Fase 7
                                         │
                                         └─── Fase 8
```

Fases 1 e 5 podem iniciar em paralelo. Fase 2 espera Fase 1 estável. Fases 3, 4 são sequenciais. Fases 6, 7, 8 podem ser paralelas após Fase 4.

## 7. Janela de execução

**Sprint 1 (semana 1-2):** Fase 1 + início Fase 5
**Sprint 2 (semana 3-4):** Fase 2 + finalização Fase 5
**Sprint 3 (semana 5-6):** Fase 3
**Sprint 4 (semana 7-8):** Fase 4 (incluso buffer de validação)
**Sprint 5 (semana 9-10):** Fase 6 + Fase 7
**Sprint 6 (semana 11-12):** Fase 8 + buffer final

Buffer de 2 semanas absorve imprevistos. Janela alvo: até final de Q3 2026.

## 8. Limpeza dos 20% de cicatriz

Durante a refatoração, eliminar progressivamente o que NÃO faríamos do zero:

**Eliminado em Fase 2:**
- 12 writers ad hoc em `contact_state`
- Heurística `lifecycle_stage` como proxy de party_role (D-SEG-10 já corrigiu, validar coverage)
- Triggers em domain tables que bumpam `state_version` (movem para Decision API)
- Concatenação de `triggered_by` em string

**Eliminado em Fase 3:**
- 6 writers ad hoc em `segment_parties` (resta apenas Decision API)
- D-SEG-12 (lookalike direct write) — migra para canonical fn

**Eliminado em Fase 4:**
- pg_cron `segment-eval-fallback`
- Function `run_segment_eval_fallback`
- Tabela `analytics.fallback_log` (migra dados para `event_health_metrics`, depois drop)
- D-SEG-9 (refresh_segment_parties legacy v1) — migra trigger para `_unified`

**Eliminado em Fase 7:**
- Edge function `check-journey-entries`

**Eliminado em Fase 8:**
- Múltiplas filas redundantes — possível unificar `processing_jobs`, `event_processing_jobs`, `segment_eval_queue` em padrão único — investigar custo/benefício antes

## 9. Coexistência com features novas

Durante a refatoração, features novas continuam sendo desenvolvidas. Regra: **toda feature nova já nasce conforme princípios P1-P10**. Não-negociável.

Se alguma feature requer mudar `contact_state`, ela usa Decision API (mesmo antes da Fase 2 estar completa) — Decision API é construída na Fase 1 já em modo permissivo.

Se alguma feature parece exigir cron novo, ela é repensada para worker. Fim.

## 10. Critério de cancelamento

A refatoração só é cancelada se:
1. Vulnerabilidade crítica em produção exige reescrita maior (improvável)
2. Mudança estratégica de produto (ex: pivot de modelo) torna o escopo obsoleto
3. Métrica de saída de qualquer fase regredir vs estado pré-refatoração

Atrasos por mais de 2 sprints requerem revisão do plano com ADR. Não cancelamento automático — replano.

## 11. Comunicação

**Cadência de check-ins?** Marco semanal com resumo do progresso e bloqueios. Ad-hoc quando uma fase muda de estado (entrou em deploy, deploy estável após 7 dias, etc.) ou quando aparece bloqueador que justifica replano.

**Quem aprova mudanças no plano?** Marcio. Claude propõe, Marcio decide.

**Quem documenta?** Claude Code documenta progresso no PROGRESS.md a cada sessão. Claude (chat) atualiza BACKLOG.md quando tarefas mudam de estado.

## 12. Referências

- Constituição: `docs/ARCHITECTURE.md` v7.27 (atualizar conforme bump)
- Segurança: `docs/SECURITY.md` v1.5
- Dependências: `docs/DEPENDENCY_MAP.md`
- Diagnóstico que originou este plano: chat de 02/05/2026 (transcript em `transcripts/2026-05-02-refactor-decision/`)

---

**Última revisão:** 2026-05-02
**Próxima revisão programada:** ao fim de cada sprint
