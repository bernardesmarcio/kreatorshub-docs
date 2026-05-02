# Refactor Progress — Diário Operacional

**Documento companheiro:** [PLAN.md](./PLAN.md), [BACKLOG.md](./BACKLOG.md)

> Este documento é o "savepoint" do refactor. Toda sessão de Claude Code começa lendo daqui.
> Atualizado a cada PR + a cada decisão arquitetural intra-sprint.
> Idioma: português. Tom: factual, sem ceremônia.

---

## Estado atual

**Sprint:** 1 — Fase 1 (Estabilização)
**Fase ativa:** Fase 1, R-1.5 done (1º ciclo validado, monitorando +30min), R-1.7 desbloqueada
**Última atualização:** 2026-05-02 22:48 UTC
**Quem atualizou:** Claude Code via R-1.5 (migration `r15_run_segment_eval_fallback_uses_watermark` em 22:44:24 UTC)

## Próximas 3 ações

1. **Monitorar R-1.5 +30min** — observar 3–6 ciclos do cron pós-R-1.5 (até ~23:15 UTC). Confirmar drift mantém zero, fallback_log segue sem entries (guard `IF v_detected > 0` faz com que predicate semântico em estado limpo não gere row), tempo de execução do cron permanece <50ms (vs 126–735ms pré-R-1.5).
2. **R-1.7** — telemetria `eval_drift_count` em `event_health_metrics` (ETL hourly + view `v_event_driven_health`). Desbloqueada com R-1.6 e R-1.5 estáveis. Métrica deve mostrar zero global, sinalizar imediatamente se algum tenant divergir.
3. **PAUSE 24h pós-Fase 1** — observar baseline pós-refactor antes de iniciar Sprint 2 (Decision Write API). Comparar amplification ratio (era 24.6x), throughput jobs, tempo de execução do fallback. Marcio decide quando abrir Sprint 2.

## Bloqueadores ativos

Nenhum.

## Métricas de baseline (medidas em 2026-05-02)

Estado pré-refactor. Valores que vamos comparar contra ao longo das fases.

| Métrica | Valor | Fonte |
|---|---|---|
| Total parties | 42.632 | `SELECT COUNT(*) FROM analytics.contact_state` |
| Tenants ativos | 7 | `core.tenants WHERE is_active` |
| Fallback hits 7d | 1.845.469 | `event_health_metrics WHERE metric_name='fallback_hits'` |
| Throughput jobs 7d | 324.668 | `event_health_metrics WHERE metric_name='eval_queue_throughput'` |
| Mudanças reais state 24h | 11.188 | `contact_state WHERE state_updated_at > now()-24h` |
| Enqueues fallback 24h | 275.402 | `fallback_log WHERE ran_at > now()-24h` |
| Amplification ratio | 24.6x | enqueues / mudanças reais |
| Latência média job | 7m20s | `segment_eval_queue` |
| Triggered_by 'manual' (fallback) | 100% últimas 6h | confirmação devastadora |
| Producers de contact_state | 12 | inventário do dev |
| Producers de segment_parties | 6 | idem |
| pg_cron categoria B (lógica de domínio) | 2 | rfm-rolling-refresh, segment-eval-fallback |

## Snapshot pré-Fase 1 (2026-05-02 19:10 UTC)

Capturado via Supabase MCP imediatamente antes de aplicar R-1.2. Travado para
comparações posteriores ao longo da Fase 1.

| Métrica | Valor |
|---|---|
| Total parties (`contact_state`) | 42.632 |
| Total memberships (`segment_parties`) | 130.548 |
| Pending jobs (`segment_eval_queue`) | 0 |
| Claimed jobs | 0 |
| Max `state_version` | 35 |
| Avg `state_version` | 5.16 |
| Fallback runs 24h | 173 |
| Enqueues fallback 24h | 284.501 |
| Jobs criados 24h | 20.144 |
| Active segments | 78 |
| Manual segments | 52 |
| Active tenants | 7 |

**Definições preservadas para rollback (md5 captured):**
- `analytics.apply_segment_membership_diff` — md5 `3e35fb8c1cf3b40c3251564bc5ac5fbc` (3.705 chars)
- `analytics.run_segment_eval_fallback` — md5 `6a41cfe1c50bbeb6def9c9413e4b1196` (1.652 chars)

## Decisões arquiteturais tomadas durante o plano

### D-2026-05-02-01 — Versão semântica separada de state_version

**Contexto:** o consultor externo sugeriu `segmentation_input_version` separado de `state_version`. O plano inicial considerava manter `state_version` único com whitelist de dimensões dentro da função canônica.

**Decisão:** versão separada (`segmentation_input_version`).

**Justificativa:** `state_version` carrega 5+ semânticas misturadas hoje. Mesmo com whitelist na função canônica, o risco de outro feature futuro voltar a usar `state_version` para fim novo é alto. Versão dedicada é defesa em profundidade.

**Alternativa rejeitada:** whitelist embutida em função canônica usando `state_version` único.

**Custo:** 1 coluna nova em `contact_state`. Negligível.

### D-2026-05-02-02 — Manter 1 tabela de queue, não separar em pending + attempts

**Contexto:** consultor externo sugeriu separar `segment_eval_pending` (hot path) de `segment_eval_attempts` (ledger). Em greenfield seria mais limpo.

**Decisão:** manter `segment_eval_queue` como única tabela. Adicionar UNIQUE INDEX parcial para coalescing.

**Justificativa:** queue tem TTL 4h via `purge-eval-queue` (R20). Histórico longo prazo já vai para `event_health_metrics` (Camada A do D-CRON-3). Custo de migrar para 2 tabelas (rewrite, atomicidade entre escritas) supera benefício de pureza arquitetural na nossa escala atual.

**Alternativa rejeitada:** rewrite em 2 tabelas separadas.

**Custo:** UNIQUE INDEX parcial extra. Negligível.

### D-2026-05-02-03 — apply_segment_membership_diff continua bumpando state_updated_at

**Contexto:** consultor externo sugeriu remover bump de ambos `state_version` e `state_updated_at` da função.

**Decisão:** remover bump de `state_version` (quebra loop), mas manter bump de `state_updated_at`. Adicionar coluna nova `segment_membership_updated_at`.

**Justificativa:** outros consumers legitimamente leem `state_updated_at` para cache invalidation (UI, exports). Remover quebra contratos não relacionados a segmentação. Solução cirúrgica: separar dirty trigger (`segmentation_input_version`) de cache invalidation (`state_updated_at`).

**Alternativa rejeitada:** remover ambos os bumps (quebraria UI cache).

**Custo:** 1 coluna nova. Negligível.

### D-2026-05-02-04 — Localização dos documentos de refactor

**Contexto:** onde colocar PLAN.md, PROGRESS.md, BACKLOG.md.

**Decisão:** subpasta `docs/refactor/`.

**Justificativa:** isola da constituição, facilita git history limpa, permite drop pós-refactor sem afetar docs principais.

**Alternativa rejeitada:** raiz de `docs/`.

### D-2026-05-02-05 — Idioma dos documentos de refactor

**Contexto:** PT vs EN.

**Decisão:** PT, coerente com ARCHITECTURE.md, SECURITY.md.

**Justificativa:** custo de switch para EN agora sem retorno imediato. Internacionalização possível futura, fora de escopo deste refactor.

### D-2026-05-02-06 — Cadência ETL de event_health_metrics confirmada como hourly

**Contexto:** durante validação Marcio + Claude, surgiu dúvida se cron
`etl-event-health-metrics` estava em `*/5 * * * *` (a cada 5min) ou `5 * * * *`
(hourly no minuto 5). Inspeção em `cron.job` confirmou: schedule é `5 * * * *` =
hourly, exatamente como spec original do D-CRON-3 v7.24.

**Decisão:** manter hourly. Nada muda na implementação atual.

**Justificativa:** ETL pulse hourly é suficiente para tier classification (que usa
rolling 7d). Cadência mais agressiva geraria overhead sem benefício de detecção
real-time (handler `runEventDrivenHealthCheck` em journeys-worker já roda 5min e dá
real-time check sobre métricas hourly).

**Alternativa considerada:** mover ETL para 5min para detecção mais granular —
rejeitada por overhead 12x sem ganho real (handler já cobre real-time).

### D-2026-05-02-07 — depends_on_fields como fonte canônica de dependências de segmento

**Contexto:** validação descobriu que `analytics.segments.depends_on_fields text[]`
já existe e está populado em 52/78 segmentos ativos (78 com a coluna não-NULL, 26
com array vazio `{}` — sentinelas ou pós D-SEG-10 sem re-derivar). Field não tinha
sido referenciado no plano original.

**Decisão:** usar `depends_on_fields` como fonte canônica para R-3.4 — não criar
campo novo ou estrutura paralela.

**Justificativa:** infra já existe e está populada para 67% dos segmentos
ativos. Reaproveitar reduz escopo da Fase 3 e elimina migration de schema. Worker
apenas precisa começar a respeitar o campo, com fail-open (skip-filter) para os
casos vazios.

**Validação adicional necessária:** confirmar que QUEM popula `depends_on_fields`
hoje (provavelmente derivação de `rules_json_v2` em `extractDependsOnFields` no
`src/lib/segmentation/segmentService.ts`) está completo e sincronizado para os 26
segmentos com `{}`. Se houver gap, registrar como sub-tarefa R-3.4-followup.

**Custo:** zero. Apenas usar o que já existe + fail-open conservador.

### D-2026-05-02-08 — Cancelar criação de branch develop Supabase

**Contexto:** plano original previa branch develop ($9.68/mês, ~$28 total) para
testar migrations da Fase 1 antes de produção. Marcio questionou necessidade.

**Decisão:** cancelar branch. Aplicar todas as migrations da Fase 1 diretamente
em produção (`pbfpwfkgjaqotbudihpy`) com checkpoints de validação cruzada entre
cada uma.

**Justificativa:**
- Migrations Fase 1 são aditivas + CREATE OR REPLACE — risco intrínseco baixo
- Branch é schema-only (zero dados) — não testa backfill 42k rows nem
  comportamento de worker, que é onde está o risco real
- Custo cognitivo de manter branch sincronizada por 12 semanas supera benefício
- D-CRON-3 v7.24 (3 camadas, 19 testes, 3 crons) foi aplicado direto em
  produção com sucesso usando checkpoint cruzado

**Alternativa rejeitada:** criar branch para Fase 1 inteira.

**Mitigação:** R-1.1 redefinida como pre-flight checklist (snapshot, backup
auto Supabase confirmado, definições antigas capturadas para rollback via md5).

**Reavaliação:** se aparecer tarefa específica de alto risco (ex: R-2.16
trigger BEFORE UPDATE em contact_state), criar branch ad-hoc no ciclo da
tarefa (criar/usar/deletar em horas, não semanas).

**Custo evitado:** ~$28 USD + 12 semanas de overhead operacional.

### D-2026-05-02-10 — Inverter ordem R-1.5 ↔ R-1.6 (worker antes do cron)

**Contexto:** plano original tinha R-1.5 (predicate cron usa watermark) antes de
R-1.6 (worker atualiza watermark). Ao revisar, percebemos que essa ordem deixa o
sistema em estado degradado entre as duas tarefas: o cron começaria a filtrar por
`segmentation_input_version > last_evaluated_segmentation_version`, mas como
nenhum worker ainda atualizaria `last_evaluated_segmentation_version`, **todos** os
parties pareceriam dirty para sempre — disparando reavaliação 100% do tempo até
R-1.6 entrar.

**Decisão:** executar R-1.6 antes de R-1.5. R-1.6 deploya worker que escreve no
watermark; cron continua usando `state_updated_at` (predicate antigo) até R-1.5.

**Justificativa:** com R-1.6 estável, o predicate antigo do cron continua
funcionando exatamente como antes (zero side effect). Quando R-1.5 trocar o
predicate, o watermark já estará populado e refletindo realidade. Nenhuma janela
de degradação.

**Custo:** zero. Apenas alteração de ordem; tarefas e contratos inalterados.

### D-2026-05-02-09 — Audit pré-R-1.4 confirma zero risco de regressão

**Contexto:** R-1.4 muda comportamento de `apply_segment_membership_diff` (deixa
de bumpar `state_version`). Dev solicitou audit antes de aplicar.

**Decisão:** prosseguir com R-1.4 conforme spec.

**Audit realizada via Supabase MCP (Claude chat) em 2026-05-02 19:35 UTC:**
- 6 funções SQL no schema `analytics` bumpam `state_version`. Apenas
  `apply_segment_membership_diff` foi removida do conjunto.
- 2 funções SQL leem `state_version` (apenas em paths de init/populate, não decisão):
  `populate_contact_state_from_existing` (init/backfill) e
  `trigger_auto_populate_contact_state` (init de party novo).
- Cron fallback (`run_segment_eval_fallback`) usa `state_updated_at` (preservado em
  R-1.4 — D-2026-05-02-03), **não** `state_version`. Continua disparando inalterado
  até R-1.5.
- Workers de membership reagem a `contact_events.segment_entered` (preservado em R-1.4),
  **não** a mudança de `state_version`.
- 5 outros producers de `state_version` (`process_purchase_analytics`,
  `upsert_contact_state_on_purchase`, `sync_party_type`, `sync_party_types_batch`,
  `sync_tag_ids_to_contact_state`) continuam bumpando — apenas
  `apply_segment_membership_diff` para.

**Conclusão:** `state_version` continua refletindo mudanças reais de estado, mas
membership recalculation deixa de ser side effect (P4 implementado).

**Custo:** 10 minutos de audit. Zero gambiarra evitada.

## Histórico de sprints

### Sprint 0 — Setup (CONCLUÍDO 2026-05-02)

**Início:** 2026-05-02
**Fim:** 2026-05-02
**Tarefas concluídas:** R-1.0 (whitelist em ARCHITECTURE.md §22.3), R-1.1 (pre-flight checklist via D-2026-05-02-08)

**Notas:**
- 3 documentos de refactor publicados (PLAN, PROGRESS, BACKLOG)
- ARCHITECTURE.md §22.3 (whitelist canônica) publicada
- D-2026-05-02-01..08 registrados
- Cleanup pós-validação aplicado em v7.28
- Sprint 1 inicia com R-1.2 (R-1.1 redefinida como pre-flight, sem branch)

### Sprint 1 — Fase 1 Estabilização (em andamento)

**Início:** 2026-05-02
**Fim previsto:** ~2 semanas (~16/05)
**Tarefas concluídas:** R-1.1 (pre-flight), R-1.2 (versões semânticas), R-1.3 (backfill + drift_idx), R-1.4 (apply_segment_membership_diff sem state_version bump), R-1.6 (worker watermark per-party com CAS atômico), R-1.5 (cron fallback usa watermark semântico)
**Tarefas em andamento:** monitoramento R-1.5 +30min (até ~23:15 UTC) — coleta de tendência multi-ciclo
**Tarefas pendentes:** R-1.7

**Notas:**
- R-1.2 aplicada direto em produção (`pbfpwfkgjaqotbudihpy`) via D-2026-05-02-08
- Migration version: `20260502191444 r12_add_segmentation_versioning_columns`
- 3 colunas novas em `analytics.contact_state` com defaults zerados em 42.632 parties
- 5 validações pós-migration ✓ (colunas, comments, defaults, smoke, list_migrations)
- R-1.3 aplicada em janela quiescente (0 pending, 0 claimed, 0 writes em 5min)
- 2 migrations: `r13_backfill_segmentation_versioning` + `r13_create_contact_state_drift_idx`
- 42.632 parties backfilled (`segmentation_input_version = last_evaluated_segmentation_version = state_version`)
- Drift count = 0 em todos os 7 tenants ativos (validado via `contact_state_drift_idx`)
- R-1.4 aplicada após audit independente (D-2026-05-02-09 — zero risco de regressão)
- Migration `r14_apply_segment_membership_diff_no_state_version_bump`
- Backup: `apply_segment_membership_diff__pre_refactor_2026_05` (rollback em <1min via DROP+CREATE OR REPLACE)
- Função nova: NÃO bumpa `state_version` (P4), MANTÉM `state_updated_at` (D-03), ADICIONA `segment_membership_updated_at`
- 4 validações ✓ (2 funções presentes, no-bump-state_version=false, bumps-membership_at=true, backup-still-bumps=true)
- R-1.6 aplicada antes de R-1.5 (ordem invertida no plano original; ver D-2026-05-02-10): worker precisa atualizar watermark antes do cron começar a depender dele
- Migration `r16_update_segmentation_watermark_rpc` (RPC `analytics.update_segmentation_watermark` SECURITY DEFINER, CAS atômico via WHERE clause na UPDATE)
- Worker patch: helper `advanceSegmentationWatermark` exportado em `workers/analytics/src/segment-eval-worker.ts`. Lê `observed_version` no início de `processEvalJob`, chama RPC após `mark_job_result`, falha não-fatal (loga e segue)
- 7 testes unitários novos cobrindo happy path, race condition, idempotência, falha RPC, resposta vazia, args passthrough, coerção bigint→number (`segmentationWatermark.test.ts`)
- Cross-check independente via `pg_stat_statements` (Marcio, 2026-05-02 22:00 UTC): RPC executada 2x com formato prepared statement (= driver postgres.js do worker), avg 5.85ms, drift global = 0, 0 erros em 30min
- Sanity 15min pós-deploy: 9 jobs processados em segment_eval_queue (3 parties distintas), 100% in_sync após RPC, latest_processed às 21:56:08 UTC
- R-1.5 aplicada em 2026-05-02 22:44:24 UTC após cross-check de contrato e captura do md5 fiel da função pré-refactor
- Migration `20260502224424 r15_run_segment_eval_fallback_uses_watermark`. Backup `analytics.run_segment_eval_fallback__pre_refactor_2026_05` (cópia byte-a-byte da definição original; rollback em <1min via DROP+RENAME)
- Função nova: predicate trocado de `state_updated_at >= now() - 15min` para `segmentation_input_version > last_evaluated_segmentation_version`. `triggered_by` mudou de `'manual'` para `'repair'` para distinguir nova semântica em telemetria. Permission model, guard de fallback_log e shape do JSON retornado preservados (contrato externo intacto)
- 4 validações imediatas ✓: 2 funções presentes com mesma signature, nova usa novo predicate (regex match), backup preserva predicate antigo, cron job 7 (`segment-eval-fallback`) inalterado
- 1º ciclo pós-R-1.5 (22:45 UTC): tempo de execução 37ms para 7 tenants (vs 126–735ms nos runs anteriores), zero entries em `fallback_log` (predicate em estado limpo + guard `IF v_detected > 0`), zero rows novos com `triggered_by='repair'` em `segment_eval_queue`. Drift global manteve zero
- Tendência hourly em `event_health_metrics.fallback_hits`: 17:00–20:00 estabilizou em 2–10/h (baseline sábado), 21:00 spike 18.622/h (legado + R-1.4 quebrando bump), 22:00 já em 2/h. Esperado bucket 23:00 próximo de zero
- Próximo: monitorar até 23:15 UTC (5–6 ciclos), depois R-1.7 (telemetria drift) e PAUSE 24h Fase 1

## Lições aprendidas (acumulando)

Esta seção cresce ao longo da refatoração. Lições registradas viram regra na constituição (R-rules) ou viram critério de PR review.

### L-001 — Validar distribuição de timestamps antes de declarar BULK
**Aprendido em:** 2026-05-01 (incidente Amazônia)
**Contexto:** Claude declarou "BULK 4168 rows às 14:49" sem validar distribuição. Dev (correta e cirurgicamente) pediu validação. Distribuição mostrou 4124 rows às 19:10 (operação anterior) + rows esparsos depois. Hipótese estava errada.
**Regra:** Antes de chamar "BULK" em diagnóstico de escrita, sempre rodar `COUNT(*) GROUP BY date_trunc('second', created_at)`.

### L-002 — Triggers em produção: SECURITY DEFINER ou GRANT explícito
**Aprendido em:** 2026-05-01 (trap de segment_parties)
**Contexto:** trigger de auditoria sem `SECURITY DEFINER`. Workers que escreviam em `segment_parties` mas não tinham GRANT em audit table teriam falhado em produção. Dev pegou antes de causar dano. Trigger foi dropado em < 5 minutos sem incident.
**Regra:** Triggers de auditoria/diagnóstico em produção SEMPRE com `SECURITY DEFINER` ou `GRANT INSERT` explícito para todas as roles que escrevem na tabela monitorada. `DROP TRIGGER IF EXISTS` antes de `CREATE` é mandatório.

### L-003 — Convergência eventual mascara bugs estruturais
**Aprendido em:** 2026-04-30 a 2026-05-02 (incidente Amazônia + diagnóstico fallback)
**Contexto:** sistema parecia funcionar porque cron fallback compensava silenciosamente bugs do event-driven path. 1.58M hits em 7 dias passaram despercebidos por meses.
**Regra:** Toda safety net (cron fallback, retry agressivo, backfill periódico) deve ter alarme dedicado quando hits > N. Sem alarme, vira mascarador de bug. R52 (D-CRON-3) materializa essa regra para o pipeline de segmentação. Padrão deve ser estendido a outros pipelines.

---

**Final do PROGRESS.md.** Atualizado por Claude / Claude Code a cada sessão.
