# Refactor Progress — Diário Operacional

**Documento companheiro:** [PLAN.md](./PLAN.md), [BACKLOG.md](./BACKLOG.md)

> Este documento é o "savepoint" do refactor. Toda sessão de Claude Code começa lendo daqui.
> Atualizado a cada PR + a cada decisão arquitetural intra-sprint.
> Idioma: português. Tom: factual, sem ceremônia.

---

## Estado atual

**Sprint:** 1 — Fase 1 (Estabilização)
**Fase ativa:** Fase 1, R-1.2 done, R-1.3 desbloqueada
**Última atualização:** 2026-05-02 19:14 UTC
**Quem atualizou:** Claude Code via R-1.2 + Marcio aprovação

## Próximas 3 ações

1. **R-1.3** — backfill: `UPDATE analytics.contact_state SET segmentation_input_version = state_version, last_evaluated_segmentation_version = state_version`. One-shot, ~42.6k rows. Aplicar fora de janela de pico.
2. **R-1.4** — `apply_segment_membership_diff` para de bumpar `state_version`. Backup como `apply_segment_membership_diff__pre_refactor_2026_05`.
3. **PAUSE 1h após R-1.4** — observar baseline antes de R-1.5/1.6.

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
**Tarefas concluídas:** R-1.1 (pre-flight), R-1.2 (versões semânticas)
**Tarefas em andamento:** —
**Tarefas pendentes:** R-1.3, R-1.4, R-1.5, R-1.6, R-1.7

**Notas:**
- R-1.2 aplicada direto em produção (`pbfpwfkgjaqotbudihpy`) via D-2026-05-02-08
- Migration version: `20260502191444 r12_add_segmentation_versioning_columns`
- 3 colunas novas em `analytics.contact_state` com defaults zerados em 42.632 parties
- 5 validações pós-migration ✓ (colunas, comments, defaults, smoke, list_migrations)
- Próximo: R-1.3 (backfill `segmentation_input_version = state_version` + `last_evaluated_segmentation_version = state_version`)

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
