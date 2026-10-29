# KreatorsHub — Arquitetura Definitiva

*Versão 10.7 — Fila de jornadas, Fase 0 aplicada (migration `20261029190619_journey_queue_phase0_gates`):*
- *Portão de pré-condições antes de enfileirar; a espera vira estado na inscrição (`wait_reason`/`wait_ref_id`/`waiting_since`), sem prazo.*
- *Liberação por trigger da campanha, em lotes de 200/min por jornada; campanha excluída coloca a inscrição em erro.*
- *Pausa real, anti-inanição no claim e uma única chave de dedupe.*
- *Adendo R39 reescrito. R63 nova: SECURITY DEFINER exposta = REVOKE PUBLIC + checagem no corpo (correções de 29/10 em `SECURITY.md` §12.6).*
- *O desenho do Codex (`20261026193000`, worker de `dd5ea54`) foi descartado e o worker revertido em `f8ae28f`. Ver `docs/JOURNEY_ISOLATION.md`.*

*Versão 10.6 — §22.16/R60: correções do consentimento manual (migration `20261029180000_manual_consent_rpcs_fixes`) — `api.set_email_consent` não devolve mais `before_state`/`after_state` e limita `p_reason` a 500 chars; resubscribe global `channel=admin_manual` também limpa unsubs ativos de grupos inativos (antes só `preference_center`).*

*Versão 10.5 — §22.16: consentimento manual pela UI do tenant — `api.set_email_consent` (unsubscribe: membro não-viewer; resubscribe: owner/admin; via `email.apply_consent_change` `channel=admin_manual`) e `api.get_email_consent_status` (qualquer membro; `effective` suppressed|global_unsubscribed|partial|subscribed).*

*Versão 10.4 — R62 nova: `ON CONFLICT` em `crm.customer_tags`/`crm.lead_tags`/`crm.customer_badges` usa o alvo party-first `(party_id, …)`; nó `add_tag` do worker `journeys` corrigido (falhava para todo contato cliente desde abr/2026).*

*Versão 10.3 — RPC `api.get_account_overview(p_tenant_id) → jsonb` (Minha conta / Visão geral; owner/admin; seção "Minha conta — Visão geral" após Platform Billing).*

*Versão 10.2 — (substituída pela 10.7: este desenho foi descartado e nunca aplicado) isolamento de jornadas implementado localmente pelo Codex: distribuição por tenant, cotas globais, raias, espera durável por campanha com prazo, orçamento por nó e disjuntor.*

## Guia de referência para escala: 50.000 tenants · 50M contatos · 1000 automações por tenant

*Versão 10.1 — R61 nova: criação de envio de campanha é atômica no servidor (`api.create_campaign_send`: `campaign_sends` + `campaign_recipients` + job numa transação, `statement_timeout` 120 s; o front nunca insere `campaign_recipients`); `api.cancel_orphan_campaign_sends` para envios órfãos. Adendo R60: `email.duplicate_campaign` copia `transactional_reason` (transacional sem motivo vira marketing com grupo). Incidente 2026-10-26 em `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` (§22.16).*
*Versão 10.0 — §25 nova: frontend mobile (redesign Ondas 3–12) — breakpoints e modos celular/tablet/desktop (D3), casca (`navConfig` como fonte única, `ShellProvider`/`PageHeader`/`BottomNav`/`/mais`, `FullBleedPage`/`ImmersiveScreen`, variáveis CSS da casca), kit de primitivas responsivas, regras M1–M3 do twMerge, guarda-corpos (ESLint para `confirm`/`alert`/`prompt`, `use-toast` e import estático de libs pesadas; teste de baseline `src/test/guardrails.test.ts`; detector de overflow em DEV; `npm run check:overflow`), PWA (`/app`, fontes self-hosted, status bar `black-translucent`, D8/D11 pendentes) e regras de validação com dados reais.*
*Versão 9.9 — §22.17 nova: notificações push (Web Push) portadas do Carreta Oftalmo — schema `push` (fila própria, fora do PostgREST), wrappers `api.push_*`, Edge Function `push-processador` (Bearer service role), cron `kreators-push-recuperar` (30 s, categoria A em §11), fila `push.fila` em §10. Destinatário = membro de `core.tenant_users` com a feature (regra de `core.has_feature_access`). D-PUSH-1..3 novas. Migrações `20261023100000`–`20261023100500` validadas só no isolado; nada aplicado.*
*Versão 9.8 — Adendo R59/R60 (Onda 3 do `docs/EMAIL_UNSUBSCRIBE_AUDIT.md`): resubscribe global confirmado pela pessoa (`channel=preference_center`, inclusive double opt-in) também desativa unsubs de grupos inativos; token do centro de preferências vale 90 dias para tudo exceto descadastrar; rate limit do email de reativação e do `load`; Sentry nas edge functions de consentimento; worker não envia marketing sem link de preferências (P18); RPCs `api.get_campaign_unsubscribe_stats` e `api.get_tenant_consent_summary` (§22.16).*
*Versão 9.7 — R60 nova: `email.campaign_sends` valida tipo no banco (marketing exige grupo; transacional sem grupo e com motivo), unsub de grupo inativo vale como global nos workers; FK `unsubscribes_group_id_fkey` → RESTRICT; RPCs `api.set_campaign_message_type` e `api.deactivate_unsubscribe_group`; políticas "manage" removidas de `unsubscribes`/`suppression_list`; IP bruto não é mais gravado (§22.16, Onda 2 do `docs/EMAIL_UNSUBSCRIBE_AUDIT.md`).*
*Versão 9.6 — §22.16 nova: consentimento de email com trilha append-only (`email.consent_audit_log`) e RPC única `email.apply_consent_change` / `apply_consent_changes` / `log_consent_error` (Onda 1 do `docs/EMAIL_UNSUBSCRIBE_AUDIT.md`, itens 1.1/1.2). R59 nova. Bug corrigido no caminho novo: `public.cancel_enrollments_on_global_unsubscribe` grava status `cancelled`, proibido pelo CHECK — descadastro global nunca cancelou jornada.*
*Versão 9.5 — §4 Backfill de jornadas reescrito: "Incluir toda a base existente" passa a funcionar (produtor = trigger `trg_journey_enqueue_backfill` na ativação; consumidor = loop de backfill do journeys-event-worker; seleção única em `journeys.backfill_batch_candidates()`; compra lê `commerce.transactions` status `paid`). `journeys.enroll_window_from` (janela efetiva por ativação). R58 nova. D-JRN-BF-1..3 novas.*
*Versão 9.4 — §22.15: Inbox local-first — outbox `inbox.ops` (FIFO por conta, mutação otimista + blindagem de campo), labels 100% locais e CRUD de pastas IMAP via worker. R56 nova (tenant check em wrapper `api.*` = `app_auth.assert_tenant_access`, nunca o claim) e R57 nova (schemas de extensão com efeito colateral — `net`/pg_net — nunca expostos a `authenticated`/`anon`).*
*Versão 9.3 — Platform Billing (§ nova ao final): cobrança da plataforma aos tenants via tenant Kreators Platform + edge fn `platform-billing` (token HMAC com expiração, PIX/boleto nativos). Dívida: tokens stateless sem revogação individual.*
*Versão 9.2 — §11: cron `custom-domains-recheck` (re-verificação periódica de DNS — `core.custom_domains` ganha status `dns_broken`; delegações ativas flagadas via `metadata.dns_broken`). Incidente: forms.alemdaescuridao.org ficou "ativo" após cliente remover o CNAME.*
*Versão 9.2 — R55 nova (RLS por participação, não só claim do JWT).
*Versão 9.1 — R54 nova (EXISTS vs IN em segmentação) e campo unificado `campaign` com seis operadores.
*Versão 9.0 — R51/R52/R53 novas (auditoria 2026-08-11: segmentação por engajamento de email nunca funcionou; motor vivo esvaziava segmentos v2). R49/R50 (incidente 2026-08-10: campanha duplicada e impossível de cancelar). §22.14: Módulo Chat (Whaticket + IA).*

---

# PARTE A — ARQUITETURA DA PLATAFORMA

## 1. Princípios que nunca mudam

Antes de qualquer detalhe técnico, estes princípios governam toda decisão arquitetural. Quando uma dúvida surgir, volte aqui.

**Escalar horizontalmente = adicionar réplicas, nunca mudar código.** Nenhuma lógica pode depender de "só uma instância sabe disso". Estado vive no banco, não no processo.

**Banco é a fonte da verdade. Worker é stateless.** Um worker pode morrer e reiniciar a qualquer momento sem perder nada. O banco guarda tudo.

**Nunca processar na hora de receber.** Todo evento externo é salvo primeiro, processado depois. Receber é O(1). Processar pode levar o tempo que precisar.

**Idempotência em toda entrada — em duas camadas.** Camada 1: `ON CONFLICT DO UPDATE` na transação com chave `(tenant_id, source_name, external_transaction_id)` — atualiza status apenas em transições válidas (ex: pending → paid), nunca retrocede (paid permanece paid exceto para refunded). Camada 2: analytics pipeline só roda na transição para `paid` — novo insert paid ou mudança pending → paid. Status igual ao anterior = skip. Evento duplicado não duplica ledger nem state.

**Falha isolada por tenant.** Um tenant com automação em loop não pode degradar os outros. Filas e rate limiting garantem isso.

**Provider novo = só um mapper. Zero mudança no banco.** O banco nunca conhece o nome do provider. O worker nunca conhece a estrutura do banco além do contrato `NormalizedTransaction`.

**Erro de código não é erro de infra — não misturar retry com blocked.** Erro transitório (deadlock, timeout, conexão) → `retry` automático, consome attempts. Erro de mapper/contrato (schema quebrado, campo obrigatório ausente) → `blocked`, não consome attempts, não reprocessa até fix + deploy explícito.

---

## 2. As três camadas da plataforma

**OBSERVAR — `analytics.contact_events`**
Ledger imutável. Append-only. Particionado. Tudo que aconteceu com cada contato.

**ENTENDER — `analytics.contact_state`**
Snapshot vivo. Uma linha por contato. Zero joins em runtime. O que o contato é hoje.

**AGIR — `analytics.segment_eval_queue` + `journeys.processing_jobs`**
O que fazer com o que sabemos.

> **`contact_state` é a fonte única de verdade para RFM.** Nenhuma view, RPC ou frontend recalcula scores RFM em runtime. NTILE existe em um único lugar: `refreshRfm.ts`. Qualquer outro NTILE no sistema é bug arquitetural.

---

## 3. Arquitetura de Workers — Estado Definitivo

### Visão geral

**SUPABASE EDGE FUNCTIONS** (ingestion — leve, rápida, sem estado)

- `doare-sync` → cron-driven | `doare:sync` + `enqueue_pending_donations`
- `apidozap` → webhook-driven | `apidozap:read_receipt`, `apidozap:process_message`
- `nylas-sync` → webhook-driven | `nylas:sync_message` e sub-handlers
- `eduzz-receiver-webhook` → receiver webhook-driven | salva raw em `integrations.webhook_events` + mapeia → NormalizedTransaction + enfileira jobs (provider=`ingestion`) | URL pública: `core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{slug}` | aceita slug no path ou `?tenant_id=uuid` (legado)
- `sympla-sync` → cron-driven (sem webhooks nativos) | `sympla:sync_orders`, `sympla:sync_participants`
- `webhooks-sendgrid` → receiver webhook-driven O(1) | salva raw em `integrations.webhook_events` + enfileira jobs (`sendgrid:delivery_event`, `sendgrid:engagement_event`, `sendgrid:suppression_event`) | URL: `core.kreatorshub.com.br/functions/v1/webhooks-sendgrid` | verificação ECDSA P-256
- `sendgrid-webhook` → **LEGADO** (desativado — URL migrada para `webhooks-sendgrid`)
- `push-processador` → pulse-driven (pg_net do gatilho adiado de `push.fila` / cron `kreators-push-recuperar` / auto-encadeamento) | drena `push.fila` e envia Web Push (RFC 8291/8292) | `verify_jwt=false`, autenticação Bearer = service role no handler | ver §22.17
- `[outros: oauth, utils]` → funções de suporte (ex: `eduzz-oauth-callback`)

### Convenção de nomenclatura — Edge Functions

| Padrão | Papel | Exemplo |
|---|---|---|
| `{provider}-receiver-webhook` | Recebe requisições HTTP externas do provider. URL pública registrada no painel do provider por tenant. Nunca processa — apenas salva raw e enfileira job. | `eduzz-receiver-webhook` ✅ |
| `{provider}-oauth-callback` | Fluxo OAuth — troca code por token, salva credenciais, dispara import histórico. | `eduzz-oauth-callback` |
| `{provider}-sync` | Sync cron-driven via pg_cron para providers sem webhook nativo. | `sympla-sync`, `doare-sync` |

> **Regra:** o nome da função deve deixar claro o papel sem precisar abrir o código.
> receiver = URL pública, processor = worker interno, sync = cron pull.

### Custom domain — Webhook URLs

**Domínio:** `core.kreatorshub.com.br` (aponta para `pbfpwfkgjaqotbudihpy.supabase.co`)

Todos os webhook URLs gerados pelo frontend usam o domínio customizado. Dois formatos aceitos:
- **Slug (preferencial):** `https://core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{tenant_slug}`
- **Legacy (fallback):** `https://core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook?tenant_id={uuid}`

O receiver resolve o slug via `core.tenants.slug` → `tenant_id`. `getWebhookUrl()` em `eduzzStorage.ts` é async e faz lookup do slug.

### claim_jobs — Filtro de handlers no SQL

Workers passam `p_handlers text[]` direto na chamada RPC `claim_jobs`, evitando claim de jobs que serão descartados em application code. Parâmetro `p_handlers` já existia no RPC mas nenhum worker usava — corrigido em Sprint 10. Ver R23.

**RAILWAY WORKERS** (processamento — Node.js, postgres.js, PgBouncer)

- `ingestion-worker` → `commerce.transactions` + analytics pipeline (handler genérico, provider-agnóstico)
- `analytics-worker` → inteligência e segmentação (segment-eval-loop, refreshRfm, computeClusters, computeLookalikes, refreshFeatures, **project_traits**, **sync_field_definitions**). `refreshRfm` escreve em `contact_state` (`customer_features` dropada em Sprint 8)
- `journeys-worker` → execução de nós de automação (`WORKER_ONLY=true`)
- `journey-scheduler` → enfileira checks periódicos (`IS_SCHEDULER=true`): `checkSegmentEntries`, `checkFormEntries`, `checkEventEntries` (apenas `trigger_type='purchase'`), `enqueueReadyWaits`
- **`journeys-event-worker`** → fan-out push-based de eventos. Processa `event_processing_jobs` + `backfill_jobs`
- `historical-sync-worker` → import histórico e enrich de qualquer provider (`ENABLED_HANDLERS=eduzz:enrich_invoice,eduzz:enrich_sales,eduzz:import_sales_page,eduzz:import_products,sympla:historical_sync`) | `RAILWAY_DOCKERFILE_PATH=historical-sync/Dockerfile`
- `email-campaign-worker` → envio de emails (`prepareEmailBatch`, `sendEmailBatch`, `retrySoftBounce`)
- `analytics-backfill-worker` → jobs de backfill e product linking isolados do pipeline crítico (`ENABLED_JOB_TYPES=link_products`)
- `email-sync-worker` → sincronização de inbox Nylas (`deepSync`, `syncMessage`, `grantReauth`)

**Schema de acesso dos workers (v7.11):** Todos os workers usam `worker.*` para job management (claim, complete, fail, block). Funções de domínio usam o schema nativo: `historical-sync` → `eduzz.*` e `integrations.*`; `email-sync` → `inbox.*`. **Zero stubs `public.*` de worker** — todos removidos.

### Regra de ouro de onde cada coisa fica

| Tipo de trabalho | Onde fica | Por quê |
|---|---|---|
| Receber webhook / pull API | Edge Function | Leve, rápida, sem estado |
| Receber webhook por provider | Edge Function dedicada por provider | Deploy independente, logs isolados, sem risco cruzado |
| Mapear provider → NormalizedTransaction | Edge Function (mapper) | Lógica de provider isolada do banco |
| Criar batch jobs na fila | Edge Function ou pg_cron | Gatilho, não processamento |
| Processar dados → `commerce.*` | ingestion-worker (Railway) | postgres.js, transactions, BEGIN/COMMIT |
| Inteligência / scores / RFM | analytics-worker (Railway) | Batch pesado, longa duração |
| Projeção de traits (formulários) | analytics-worker (Railway) | `project_traits` + `sync_field_definitions` via `analytics.processing_jobs` |
| Import histórico / enrich por provider | historical-sync-worker (Railway) | Batch pesado, longa duração, isolado do hot path |
| Fan-out de eventos → enrollments | journeys-event-worker (Railway) | SLA < 5s, realtime, isolado |
| Executar nós de automação | journeys-worker (Railway) | Estado de enrollment, grafos |
| Enviar email | email-campaign-worker (Railway) | API key, rate limit SendGrid |
| Sincronizar inbox | email-sync-worker (Railway) | API key, longa duração |

---

## 4. O caminho de cada tipo de evento

### 4.1 Webhook / Pull periódico — Fluxo unificado de ingestion

```
Provider
  → Edge Function (receiver)
  → salva raw em provider.raw_table (ledger imutável)
  → mapper: raw → NormalizedTransaction[] (lógica de provider fica aqui — nunca no banco)
  → agrupa em batches (max 100 itens, 512KB, 20 por tenant)
  → INSERT integrations.jobs com handler = 'commerce:process_batch'
  → retorna 200 em < 100ms

ingestion-worker consome integrations.jobs
  → handleProcessBatch — único handler para todos os providers
  → 1 chamada SQL: commerce.process_transactions_batch(tenant_id, transactions[])
  → banco processa todo o batch numa transação
  → loga sumário + grava erros em integrations.job_errors
```

### 4.2 Import histórico (onboarding de novo tenant)

```
Tenant conecta provider
  → Edge Function (oauth callback)
  → descobre volume total (ex: 456 páginas)
  → cria N jobs com scheduled_for espaçados (1 página por job, ~500ms de intervalo)

historical-sync-worker (Railway)
  → busca 1 página da API
  → salva em provider.raw_table
  → mapeia → NormalizedTransaction[]
  → cria job 'commerce:process_batch' para o lote

ingestion-worker → process_transactions_batch → commerce + analytics
```

**Fluxo Eduzz especificamente:**
```
Webhook Eduzz chega em core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{slug}
  → eduzz-receiver-webhook: resolve slug → tenant_id, valida HMAC, salva raw
  → mapeia payload → NormalizedTransaction (direto na Edge Function)
  → enfileira job commerce:process_batch (provider='ingestion', priority=1)
  → ingestion-worker → commerce.transactions
  → [priority 9] job eduzz:enrich_invoice
  → historical-sync-worker → eduzz_upsert_buyer, eduzz_upsert_invoice, enriquecimento crm.party_person
```

### 4.3 Segmentação (quando contact_state muda)

```
Qualquer evento que muda o contact_state
  → INSERT segment_eval_queue com fields_changed e priority (1 para compra, 9 para fallback)

analytics-worker (segment-eval-loop)
  → claim_next_segment_jobs (FOR UPDATE SKIP LOCKED)
  → para cada segmento: avalia depends_on_fields ∩ fields_changed
  → avalia regra em memória contra contact_state
  → apply_segment_membership_diff (adiciona/remove segmentos, atualiza contact_state.segment_ids)
```

**FALLBACK** (pg_cron a cada 5min): detecta contatos com `state_updated_at` recente sem job na fila, enfileira com `priority=9`. Se `fallback_log` tiver registros frequentes → algum handler esqueceu de chamar o analytics pipeline.

### 4.4 Automações / Jornadas

**ENTRADA — como o contato entra na jornada**

- Trigger `segment_entry` → **push-based (Sprint 14):** `refresh_segment_parties_unified` detecta novos membros e enfileira `check_segment_entries` diretamente. Scheduler fallback a cada 5 min (throttled de 60s→5min). Redução ~85% no volume de jobs.
- Trigger `form_entry` → `trigger-form-journey` (Edge Function, push principal) ou scheduler `check_form_entries` (fallback 5 min)
- Trigger `event_entry` → arquitetura push-based (ver seção 4.5) ✅ Operacional desde Sprint 6

**CICLO DE VIDA DA JORNADA**

Três colunas governam o comportamento:
- `activated_at` → primeira ativação (imutável — nunca sobrescrito)
- `last_activated_at` → avança a cada reativação (filtro do checkEventEntries)
- `paused_at` → setado ao pausar, null ao reativar

Trigger no banco (`trg_journey_status_lifecycle`) gerencia essas transições automaticamente. Eventos durante pausa são ignorados porque `checkEventEntries` só processa eventos com `created_at >= journey.last_activated_at`.

**PROCESSAMENTO — como os nós são executados**

journeys-worker worker mode → `claim_next_job` → `processJourneyNode.ts`

- Máximo 50 nós por execução
- Time budget: 3 minutos (re-enfileira se exceder)
- Idempotência via `journeys.journey_executions`
- Tipos de nó: `send_email`, `add_tag`, `remove_tag`, `condition`, `wait`, `ab_split`, `webhook`, `goal`, `exit`, `create_opportunity`, `add_badge`, `remove_badge`, `notify`

**ATOMICIDADE (Sprint 14 — D18/D18b)**

- Enrollment INSERT + process_enrollment job INSERT: `sql.begin()` em 5 entry handlers
- send_email node: 3 INSERTs (campaign_send + recipient + enqueue) em `sql.begin()`
- Safety net: pg_cron `sweep-orphan-enrollments` (*/10 * * * *) detecta enrollments ativos sem job pendente e enfileira recovery com `dedupe_key = 'orphan_' + enrollment_id`

**PERFORMANCE (Sprint 17)**

- Batch claim: `journeys.claim_next_jobs(worker_id, batch_size, job_types, tenant_cap)`.
  - **Até 26/10/2026:** o ROW_NUMBER por tenant só distribuía candidatos já cortados pela prioridade global (`batch*3`), e isso permitia inanição (incidente de 22–28/10).
  - **Desde a Fase 0 (29/10/2026):** até metade do lote vai para o pendente mais antigo de cada `job_type`, independentemente da prioridade. `process_enrollment` de jornada não ativa não é entregue.
  - **Fase 1 (pendente):** revezamento justo entre tenants e raias. Ver `JOURNEY_ISOLATION.md`.
  - journeys-worker 10 jobs/poll, email-campaign 5 jobs/poll.
- Email paralelo: `p-limit(10)` + rate limiter token bucket (100/sec) + circuit breaker (5 failures). ~50-90 emails/sec. `SEND_CONCURRENCY` configurável.
- Frequency guard (R45): send_email node verifica último email sent antes de enviar. Skip = success (enrollment avança). Decisão persistida em `journey_executions.decision_result` (R42). `EMAIL_FREQUENCY_GUARD_HOURS` configurável, **default=0 (desabilitado, opt-in)**. Automações triggered são contextuais — guard é opt-in via env var no Railway.
- SLA separation (R46): journeys-worker (seconds) e email-campaign-worker (minutes) com zero overlap de job_types.

### 4.5 Trigger event_entry — Arquitetura Push-Based

**Status:** ✅ Operacional

**Princípios do design:**
- `journey_events` é ledger imutável — append-only, NUNCA marcar `processed = true`
- Cada jornada define sua própria janela via `last_activated_at`
- Dedup via UNIQUE em `journey_enrollments`, não via flag no evento

**Fluxo:**
```
Evento ocorre
  → INSERT em journeys.journey_events
  → AFTER INSERT trigger (trg_enqueue_event_processing) insere em journeys.event_processing_jobs [O(1)]
  → journeys-event-worker consome (FOR UPDATE SKIP LOCKED)
  → busca jornadas ativas via routing index (tenant_id + trigger_event_type)
  → para cada jornada match: avalia filtros
  → tenta INSERT enrollment ON CONFLICT DO NOTHING
  → se inseriu: enfileira process_enrollment
  → journeys-worker executa nós
```

**SLA medido:** evento novo → enrollment criado em < 5s.

**Tabelas envolvidas:**
- `journeys.journey_events` → ledger imutável (append-only)
- `journeys.event_processing_jobs` → fila de fan-out
- `journeys.backfill_jobs` → controle de backfill histórico
- `journeys.journey_enrollments` → resultado do processamento
- `journeys.processing_jobs` → execução dos nós (existente, inalterado)

**Routing index:** coluna derivada `trigger_event_type` (`GENERATED ALWAYS AS STORED`) com índice parcial em jornadas ativas para lookup O(log n).

**Event types suportados (Sprint 18):**
| event_type | Trigger SQL | Tabela fonte | Descrição |
|---|---|---|---|
| `purchase` | `trigger_purchase_journey_event()` | `commerce.transactions` | status → `paid` |
| `transaction_pending` | `trigger_purchase_journey_event()` | `commerce.transactions` | status → `pending` (boleto/PIX) |
| `cart_abandonment` | `trigger_cart_abandonment_journey_event()` | `eduzz.cart_abandonments` | Carrinho abandonado |
| `email_opened` | `trigger_email_engagement_journey_event()` | `email.email_events` | Email aberto |
| `email_clicked` | `trigger_email_engagement_journey_event()` | `email.email_events` | Link clicado |
| `form_submitted` | — (scheduler/push) | — | Formulário enviado |

Guards de transição: `purchase` só dispara se OLD.status != 'paid'; `transaction_pending` só dispara se OLD.status != 'pending'. INSERT direto como 'paid' gera APENAS purchase (sem transaction_pending). Pending → paid gera ambos (desejado: jornadas diferentes).

**Backfill — decisão explícita na ativação (implementado de ponta a ponta em 2026-10-16):** O usuário escolhe no diálogo de ativação: "Apenas novos entrantes" (padrão), "Incluir toda a base existente" (`enroll_existing=true`) ou "a partir de uma data" (`settings_json.enroll_from_date`). Até 2026-10-16 a opção só gravava a coluna: `journeys.backfill_jobs` nunca recebia linhas e `processBackfillJob` nunca era chamado (e, se fosse, descartaria todo evento anterior à ativação).

Fluxo atual (jornadas `event_entry`; segment/form ver abaixo):
1. **Produtor** — trigger `trg_journey_enqueue_backfill` (AFTER UPDATE OF status, transição → `active`) chama `journeys.request_journey_backfill(journey_id, from, to)`: UPSERT idempotente (UNIQUE `journey_id`) com `backfill_from = enroll_from_date` (NULL = todo o histórico) e `backfill_to = last_activated_at`. Trigger e não RPC porque a ativação é um UPDATE direto do frontend (não existe RPC de ativação) e o trigger cobre qualquer caminho de ativação; o BEFORE `trg_journey_status_lifecycle` já fixou `last_activated_at` e o UPDATE traz `enroll_existing`/`settings_json`/`trigger_config` da mesma ativação. Ativação "apenas novos" marca qualquer job anterior como `blocked`. **Reativação** reseta cursor e contadores do mesmo job — seguro porque o dedupe pula quem já tem QUALQUER inscrição na jornada.
2. **Consumidor** — loop de backfill do `journeys-event-worker` (`runBackfillLoop`, tick 5s, até 3 jobs/tick, 1 lote de 500 por job), separado do loop de eventos. Cada lote é UMA transação: `pg_try_advisory_xact_lock(20250225, hashtext(tenant_id))` (1 lote por tenant por vez; xact-level, nunca vaza entre conexões do pool), revalida o claim (`FOR UPDATE`), exige jornada `active` (senão `blocked`), e num único statement: candidatos → INSERT inscrições (`ON CONFLICT DO NOTHING`) → INSERT `process_enrollment` priority 3 (push path usa 8) → avanço do cursor + contadores. Lote incompleto = `completed`. Erro = rollback do lote inteiro + `retry` com backoff (5 tentativas p/ transiente, 2 p/ demais) → `failed`.
3. **Seleção** — `journeys.backfill_batch_candidates(journey, from, to, cursor_ts, cursor_id, limit)` é a fonte ÚNICA (worker, `api.preview_journey_backfill` do diálogo e dry-run). Fronteira `[from, to)` — `to` exclusivo: `>= last_activated_at` é do push path. Cursor estável `(created_at, id)`.
   - `purchase` → **`commerce.transactions` `status='paid'`** (refunded/cancelled/pending fora), produto por `product_id` OU `product_name` (mesma semântica do push path), corte superior por `created_at` da transação e inferior pela data da compra (`paid_at`). `journey_events` não serve como fonte do histórico de compra: eventos antigos têm `product_id` NULL (produto vinculado depois por `link_products`, com nome diferente) e continuam existindo após refund.
   - demais eventos → `journeys.journey_events` do `event_type`, filtros `product_ids`/`campaign_ids`/`link_urls`.
   - dedupe → pula quem já tem inscrição (qualquer status) por `lower(customer_email)` ou `party_id`, **independentemente de `entry_policy`**: o backfill só traz quem nunca entrou; 1 linha por pessoa por lote.
4. **Supressão** — não é filtrada no backfill (a jornada pode ter nós além de email): o nó `send_email` (`processJourneyNode.executeSendEmailNode`) bloqueia `email.suppression_list`, descadastro global e descadastro do grupo da campanha (`email.unsubscribes.group_id = campaign.group_id`, só `message_type='marketing'`) e "já recebeu esta campanha"; o `sendEmailBatch` do email-campaign-worker repete supressão + unsub como segunda camada.

**Segment / form:** não usam `backfill_jobs`. `on_journey_status_change` enfileira `check_segment_entries`/`check_form_entries` na ativação. Segmento inscreve todos os membros atuais (honra "incluir base"; ver D-JRN-BF-1 para "apenas novos"). Formulário: `checkFormEntries` usa `journeys.enroll_window_from` (gravado por `trg_journey_status_lifecycle` em toda ativação: `enroll_from_date` > `1970-01-01` se `enroll_existing` > `last_activated_at`) — antes cortava "toda a base" na data de criação da jornada. Jornadas ativadas antes de 2026-10-16 (`enroll_window_from` NULL) mantêm a regra legada até serem reativadas (sem inscrição retroativa em clientes que não pediram).

**Filtros de evento:** a lógica de match (produto por id/nome, campanha, URL) vive em `processEventJob.ts` (push, em memória) e em `journeys.backfill_batch_candidates()` (SQL). Não existe `shared/journeyMatcher.ts` (citado em versões antigas deste documento). Mudou a semântica de um, mude a do outro (R58).

**Scheduler — check_event_entries desabilitado:** A função SQL `journeys.enqueue_journey_checks()` remove `'event_entry'` do loop. O handler `checkEventEntries.ts` permanece no código como **fallback emergencial** — nunca deve ser agendado salvo incidente no journeys-event-worker.

**ATENÇÃO — trigger_type no banco:** O valor correto salvo no banco é `'event_entry'` (não `'event'`). Arquivos relevantes:
- `src/types/journey.ts`: TriggerType inclui `'event_entry'`
- `JourneyBuilder.tsx`: `detectTriggerFromNodes()` salva `trigger_type: 'event_entry'`
- `NodeInspector.tsx`: `sourceType` usa `'event_entry'`

---

## 5. NormalizedTransaction — O contrato entre providers e banco

O banco nunca vê além deste contrato. O mapper de cada provider é responsável por produzir este formato.

**Campos obrigatórios de identidade:**
- `_version: 1` e `_schema: 'normalized_transaction_v1'` — versão desconhecida = erro por item, batch continua
- `external_transaction_id` — ID único no provider, chave de idempotência. **Atenção:** o nome do campo no JSON é `external_transaction_id` (é o que `process_transactions_batch` lê via `v_tx->>'external_transaction_id'`). Enviar `external_id` faz o campo ser silenciosamente ignorado e a transação entra sem ID — perdendo a idempotência.
- `source_name` — `'doare'` | `'eduzz'` | `'hotmart'` | `'stripe'`
- `source_type` — `'webhook'` | `'integration'` | `'manual'`
- `email` — chave de upsert do contato

**Campos opcionais de contato:** `name`, `phone`, `document` (CPF/CNPJ — normalizar no mapper), `city`, `state` (nome completo ou sigla — banco aceita ambos)

**Campos obrigatórios de transação:**
- `amount` — SEMPRE em reais, NUNCA centavos. Usar `Math.round(x * 100) / 100` para evitar float
- `currency` — `'BRL'` default
- `status` — `'paid'` | `'pending'` | `'cancelled'` | `'refunded'` | `'chargeback'`

**Campos opcionais de transação:** `paid_at` (ISO 8601), `product_id`, `product_name`, `utm_source`, `utm_medium`, `utm_campaign`

**Campo de auditoria obrigatório — `raw_source`:**

```
Regra para escala:

- Provider COM schema raw dedicado (doare, eduzz, e todo provider novo):
  raw_source deve ser omitido. A rastreabilidade é garantida via
  external_transaction_id → provider.raw_table. Duplicar o payload
  no job causa bloat e degrada performance em escala.

- Provider SEM schema raw dedicado (integrações pontuais, manual):
  raw_source obrigatório. É a única fonte de auditoria disponível.
  Mas todo provider novo deve ter schema raw dedicado (ver seção 14) —
  raw_source neste caso é temporário até o schema existir.
```

### Mapeamento de status por provider

| Status no provider | Status normalizado |
|---|---|
| Pago, approved, paid, completed | `paid` |
| Pendente, pending, waiting | `pending` |
| Cancelado, cancelled, voided | `cancelled` |
| Reembolsado, refunded, reversed | `refunded` |
| Chargeback, disputed | `chargeback` |

---

## 6. commerce.process_transactions_batch — A RPC genérica

Uma única RPC que processa qualquer provider. Nunca precisa ser modificada para novos providers.

**Assinatura:**
```sql
commerce.process_transactions_batch(p_tenant_id uuid, p_transactions jsonb)
RETURNS jsonb
-- Retorna: { processed, skipped, errors: [{external_id, error_code, message}] }
```

**Garantias internas:**

1. Para cada transação do batch, valida versão do contrato — versão desconhecida gera erro por item, batch continua
2. Captura o status anterior da transação se ela já existir (para detectar transição real)
3. **Role assignment condicional:** `paid` + `amount > 0` → `ARRAY['customer']`; caso contrário → `ARRAY['lead']`. Transações gratuitas (R$0) **nunca** atribuem role customer — contato permanece como lead.
4. Faz INSERT com `ON CONFLICT DO UPDATE` — transições unidirecionais: `paid` permanece `paid` exceto para `refunded`; qualquer outro status pode transicionar para `paid`
5. `paid_at` é preenchido via `COALESCE` — nunca sobrescreve se já existia
6. Se o status não mudou (`v_previous_status = v_status`) → skip, não roda analytics
7. Analytics pipeline só roda na transição PARA `paid`: novo insert `paid`, ou mudança de `pending`/`cancelled` para `paid`

---

## 7. analytics.process_purchase_analytics — Pipeline consolidado

Uma única RPC executada dentro do loop de `process_transactions_batch`. Substitui 4 chamadas separadas.

**Executa 4 operações em sequência na mesma transação:**

1. `INSERT contact_events` — ledger imutável (idempotência por `transaction_id`)
2. `INSERT contact_state ON CONFLICT DO UPDATE` — snapshot vivo
3. `INSERT segment_eval_queue priority=1` — fila de segmentação
4. `INSERT tenant_distribution ON CONFLICT` — marca tenant como stale

---

## 8. Ciclo de vida dos Jobs

**Estados:** `pending` → `running` → `success` / `retry` / `blocked` / `dead`

**Classificação de erros:**

| Classe | Error codes | Comportamento | Resolução |
|---|---|---|---|
| Contract bug | `unknown_contract_version`, `missing_email` | job → `blocked` | fix mapper + deploy + `reprocess_blocked_jobs` |
| Data invalid | `missing_product_name`, `invalid_amount` | item → `job_errors`, job continua | corrigir dado na fonte |
| Infra | `deadlock (40P01)`, timeout, conexão | job → `retry` com backoff exponencial | automático |

**Blocked jobs — fluxo operacional:**

1. Contract error detectado → worker chama `blockJob()` (não `failJob`) → `job.status = 'blocked'`, para de ser reclamado
2. Alerta dispara
3. Fix do mapper → commit → deploy Railway
4. Executar `reprocess_blocked_jobs` com filtros opcionais (`error_code`, `source_name`, `tenant_id`, `job_id`) → jobs voltam para `'pending'`, `attempts = 0`

**RPCs de lifecycle:**

| RPC | O que faz |
|---|---|
| `worker.claim_jobs(worker_id, batch_size, provider, handlers)` | Clama jobs para processamento (SKIP LOCKED) |
| `worker.complete_job(job_id, result)` | Marca success |
| `worker.fail_job(job_id, error)` | Retry com backoff exponencial ou dead |
| `worker.block_job(job_id, reason)` | Bloqueia job sem consumir attempts |
| `integrations.reprocess_blocked_jobs(...)` | Reactiva jobs blocked após fix |

**Estrutura de arquivos do ingestion-worker:**
```
workers/ingestion/src/
├── handlers/
│   ├── commerce/
│   │   └── processBatch.ts        ← handler genérico (não muda nunca)
│   ├── doare/
│   │   ├── mapTransactions.ts     ← mapper Doare → NormalizedTransaction
│   │   └── sync.ts                ← pull API Doare
│   ├── eduzz/
│   │   ├── mapTransactions.ts     ← mapper Eduzz → NormalizedTransaction
│   │   └── processInvoice.ts      ← DEPRECATED — fluxo migrado para Edge Function eduzz.ts
│   └── [hotmart]/
│       └── mapTransactions.ts     ← quando chegar: só criar este arquivo
├── types/
│   └── NormalizedTransaction.ts   ← interface compartilhada
└── worker.ts                      ← BATCH_SIZE + CONCURRENCY via env vars
```

**Estrutura de arquivos do journeys-event-worker:**
```
workers/journeys-event/src/
├── config.ts
├── db.ts
├── logger.ts
├── index.ts                       ← dois loops: event jobs (1s) + backfill (tick 5s, BACKFILL_* env)
└── handlers/
    ├── processEventJob.ts         ← fan-out: evento → jornadas matching → enrollments
    ├── processBackfillJob.ts      ← backfill histórico: claim + 1 lote/transação via journeys.backfill_batch_candidates()
    └── processBackfillJob.test.ts ← `npm test` (node:test + tsx)
```

Env vars do backfill: `BACKFILL_ENABLED` (default `true`), `BACKFILL_POLL_INTERVAL_MS` (5000), `BACKFILL_BATCH_SIZE` (500), `BACKFILL_JOBS_PER_TICK` (3), `BACKFILL_STALE_MINS` (10).

**Configuração de performance (Railway env vars):**

| Service | BATCH_SIZE | CONCURRENCY | Pool max |
|---|---|---|---|
| ingestion-worker | 25 | 10 | 12 |
| journeys-event-worker | 20 | 5 | — |

---

## 9. Rate Limiting e Isolamento por Tenant

**Camada 1 — Criação dos jobs (Edge Function):** Max 20 transações por tenant por batch. Tenant com 500 pendentes = 25 jobs de 20, processados em rounds. Nunca monopoliza a fila.

**Camada 2 — Claim da fila (banco):** `FOR UPDATE SKIP LOCKED` + `PARTITION BY tenant_id ORDER BY priority, created_at`. Fairness entre tenants no mesmo worker. 429 do provider: job reagenda com backoff exponencial (60s→86400s + jitter 20%).

**Tenant problemático (loop de automação):** Max 50 nós por execução, time budget 3 min com re-enfileiramento, `releaseStaleClaims()` a cada 60s.

**journeys-event-worker — backfill por tenant:** Advisory lock transacional `pg_try_advisory_xact_lock(20250225, hashtext(tenant_id))` garante no máximo 1 lote de backfill por tenant por vez (lock ocupado → job volta em 30s). Lote de 500 linhas de origem com cursor estável; até 3 jobs por tick (5s), processados em sequência. `process_enrollment` do backfill entra com priority 3 (push path = 8; claim é `priority DESC`), então o backfill nunca atrasa entradas ao vivo.

---

## 10. Filas do Sistema

| Fila | Tabela | Quem produz | Quem consome |
|---|---|---|---|
| Ingestion jobs | `integrations.jobs` | Edge Functions (mappers), pg_cron | ingestion-worker |
| Segment eval | `analytics.segment_eval_queue` | `process_purchase_analytics`, `analytics-events.ts`, fallback | analytics-worker |
| Journey processing | `journeys.processing_jobs` | journeys-worker scheduler, journeys-event-worker | journeys-worker |
| **Event processing** | **`journeys.event_processing_jobs`** | **`trg_enqueue_event_processing` (trigger AFTER INSERT)** | **journeys-event-worker** |
| **Backfill** | **`journeys.backfill_jobs`** | **Trigger `trg_journey_enqueue_backfill` na ativação com "incluir base" (`journeys.request_journey_backfill`)** | **journeys-event-worker (`runBackfillLoop`)** |
| Email sending | `email.campaign_sends` | journeys-worker (`send_email` node) | email-campaign-worker |
| Analytics jobs | `analytics.jobs` | analytics-worker scheduler | analytics-worker |
| Analytics processing | `analytics.processing_jobs` | Edge Functions (form submit trigger), backfill scripts | analytics-worker (`project_traits`, `sync_field_definitions`, `link_products`) |
| Job errors | `integrations.job_errors` | `process_transactions_batch` (erros por item) | observabilidade + reprocess |
| **Push (notificações da equipe)** | **`push.fila`** (+ `push.entregas` por aparelho, `push.execucoes`) | **`push.enfileirar_evento`/`push.enfileirar` (gatilhos SECURITY DEFINER dos módulos), `api.push_testar_aparelho`** | **Edge Function `push-processador` (claim `push.reservar_lote`, SKIP LOCKED); cron `kreators-push-recuperar` devolve reservas vencidas** |

---

## 11. pg_cron — Jobs Agendados (auditados)

Auditoria arquitetural completa em v7.22. Cada cron é classificado por categoria
(R50): **A** = housekeeping puro (manter); **B** = enqueue com lógica embutida
(refatorar para worker no Railway); **C** = lógica de domínio inline (substituir
totalmente); **D** = caduco (deletar). Cron novo requer entrada nesta tabela —
sem registro = violação arquitetural.

| jobname | schedule | command (resumido) | função SQL invocada | categoria | nota |
|---|---|---|---|---|---|
| `archive-opportunity-activities` | `0 2 * * 0` | INSERT archive + DELETE source TTL 2y | inline | **A** | Banco-only. Monitorar lock duration se volume crescer. |
| `cleanup-asaas-webhook-events` | `0 4 * * *` | DELETE TTL 90d processed | inline | **A** | TTL puro. |
| `cleanup-backfill-jobs` | `0 3 * * *` | DELETE TTL 30d completed | inline | **A** | TTL puro. |
| `cleanup-event-processing-jobs` | `0 3 * * *` | DELETE TTL 7d success | inline | **A** | TTL puro. |
| `cleanup-journey-processing-jobs` | `0 4 * * *` | DELETE TTL 7d completed/failed | inline | **A** | TTL puro. |
| `cleanup-old-analytics-jobs` | `0 4 * * *` | DELETE TTL 7d completed/failed | inline | **A** | TTL puro. |
| `purge-eval-queue` | `15 * * * *` | DELETE TTL 4h processed | `analytics.purge_processed_eval_queue` | **A** | TTL puro (R20). |
| `purge-integrations-jobs` | `30 3 * * *` | DELETE TTL 7d/30d | `integrations.purge_old_jobs` | **A** | TTL puro com cleanup de `job_errors`. |
| `purge-integrations-webhook-events` | `45 3 * * *` | DELETE TTL com guard `NOT EXISTS jobs` | `integrations.purge_old_webhook_events` | **A** | TTL puro. |
| `expire-soft-bounce-suppressions` | `0 * * * *` | UPDATE TTL `expires_at <= NOW()` | `email.expire_soft_bounce_suppressions` | **A** | TTL puro. |
| `create-contact-events-partition` | `0 2 25 * *` | DDL `CREATE TABLE PARTITION` idempotente | `analytics.create_contact_events_partition_if_needed` | **A** | DDL maintenance. |
| `etl-event-health-metrics` | `5 * * * *` | Pulse ETL hourly: agrega `fallback_log` + `contact_events` + `segment_eval_queue` em `event_health_metrics` via `emit_health_metric` | `analytics.etl_event_health_metrics` | **A** | D-CRON-3 (v7.25). Banco-only, sem lógica de domínio. Substituível por workers em D-CRON-4 sem mudar view/handler (R52). |
| `event-health-metrics-partition-create` | `30 2 25 * *` | DDL idempotente — cria partição mensal de `event_health_metrics` | `analytics.event_health_metrics_partition_create_if_needed` | **A** | D-CRON-3 (v7.25). Mesmo padrão de `create-contact-events-partition`. |
| `event-health-metrics-purge-old` | `30 4 * * *` | DROP partition para retention 90d em `event_health_metrics` | `analytics.event_health_metrics_purge_old` | **A** | D-CRON-3 (v7.25). TTL puro via DROP (mais barato que DELETE). |
| `sweep-orphan-enrollments` | `*/10 * * * *` | Re-enqueue de enrollments órfãos LIMIT 100 | `journeys.sweep_orphan_enrollments` | **A** | Safety net pós D18 (fanout não-atômico). Sem lógica de domínio. **TODO:** alarmar quando `count > 0` indica bug em fanout. |
| `rfm-rolling-refresh` | `0 * * * *` | Enqueue `refresh_features` para 1/24 dos tenants/hora com threshold 20h via `MOD(HASHTEXT(t.id), 24) = HOUR(NOW())` | `analytics.enqueue_job_internal` | **B** | Loop bloqueante de tenants no banco (viola §2.4). Lógica de sharding e threshold em SQL. **D-CRON-2** — migrar para worker. |
| `segment-eval-fallback` | `*/5 * * * *` | Itera tenants ativos, detecta contatos com `state_updated_at < 15min` sem job pending, enqueue na `segment_eval_queue` | `analytics.run_segment_eval_fallback` | **B** | Safety net pro event-driven path. Loop bloqueante de tenants (viola §2.4). Mascara bugs do path event-driven sem alarme. **D-CRON-3** (instrumentar `analytics.fallback_log`) é pré-requisito de **D-CRON-4** (migrar para worker). |
| `custom-domains-recheck` | `20 */6 * * *` | Pulse → Edge Function `domain-delegation` action `recheck_domains` (re-valida DNS de `core.custom_domains` ativos + NS de delegações ativas) | `core.invoke_domain_recheck` | **A** | v9.2. SQL é pulse puro (`net.http_post`); toda a lógica de DNS vive no Edge Function (mesmo padrão de `integrations.invoke_worker`). Domínio ativo vira `dns_broken` após 2 checks consecutivos falhos; volta a `active` quando o CNAME (`cname.vercel-dns.com`) reaparece. Delegações quebradas são flagadas em `metadata.dns_broken` sem tocar o status. |

| `kreators-push-recuperar` | `30 seconds` | Housekeeping da fila push (reservas/execuções vencidas → `pendente`/`nao_iniciada`/`abandonada`, expiração, reservas excedidas) + pulse (`push.sinalizar('cron')` → pg_net → Edge Function `push-processador` quando há pendente pronta; no modo residente, reacende a corrente) | `push.recuperar` | **A** | v9.9 (§22.17). Banco-only + pulse, sem regra de negócio nem loop de tenants; try-lock próprio + do despachante (rodada concorrente pula). Prefixo `kreators-`: banco compartilhado entre produtos. **Ainda não aplicado** (docs/push/APLICAR.md). |

> **Pull-based providers (Sympla, Doare):** sync periódico gerenciado pelo pull-sync-worker via `integrations.sync_schedules`. Não usam pg_cron.

> **R50 (v7.22):** pg_cron é APENAS para housekeeping (DELETE/UPDATE de TTL, DDL maintenance, pulse simples sem lógica de domínio). Lógica com tenant awareness, sharding heurístico, thresholds dinâmicos ou INSERT em tabelas de domínio = worker no Railway. Cron novo requer entrada nesta tabela + classificação A.

---

## 12. contact_state — O Prontuário do Contato

Uma linha por contato. É o que todo o sistema consulta para tomar decisões. Nunca fazer join em runtime para dados de comportamento — tudo vem daqui.

No campo `lifecycle_stage`, adicionar `dormant` à lista de valores válidos: `never_purchased/new/active/cooling/at_risk/dormant/churned`.

| Categoria | Campos |
|---|---|
| RFM | `recency_days`, `frequency`, `monetary`, `r_score`, `f_score`, `m_score` (1-5), `rfm_segment` |
| Valor | `value_tier` (platinum/gold/silver/bronze), `lifecycle_stage` (never_purchased/new/active/cooling/at_risk/churned) |
| Comportamento | `bought_product_ids[]`, `tag_ids[]`, `responded_form_ids[]`, `import_ids[]`, `segment_ids[]` |
| Engajamento | `email_open_rate_30d`, `engagement_score` (0-100), `last_purchase_at` |
| Preditivo | `churn_score` (0-1), `purchase_propensity` (0-1), `next_best_product_id` |
| Aquisição | `acquisition_source`, `utm_source/medium/campaign`, `first_touch_revenue` |

**contact_state como tabela quente:** Com 5M contatos e updates constantes, requer `fillfactor = 70` (espaço para HOT updates), autovacuum agressivo (`vacuum_scale_factor = 0.01`, `analyze_scale_factor = 0.005`), e monitoramento de dead tuples (alerta se `dead_pct > 10%`).

**Como o contact_state é atualizado:** Toda mutação deve: (1) atualizar `contact_state`, (2) registrar em `contact_events`, (3) inserir em `segment_eval_queue` com `fields_changed`. Sempre os três juntos, sempre em `BEGIN/COMMIT`. Nunca atualizar `contact_state` diretamente sem passar por essas funções.

> **Fronteira permanente do contact_state:** Somente dados comportamentais
> derivados pertencem a contact_state — RFM, lifecycle, scores, arrays de
> membership, datas derivadas de comportamento. Campos de perfil (phone,
> profession, birth_date) vivem em `crm.party_person` / `crm.party_profile`.
> Campos customizados do tenant vivem em `crm.party_trait_*`. Materializar
> campos de perfil ou traits em contact_state é violação arquitetural (ver R30).

### Campos array — estado dos producers

| Campo | Producer | Status |
|---|---|---|
| `bought_product_ids` | `process_purchase_analytics` + `linkProducts.ts` (recalcula array completo após link) | ✅ Ativo |
| `tag_ids` | Triggers DB em `crm.customer_tags` + `crm.lead_tags` (after insert/delete) | ✅ Ativo |
| `import_ids` | Backfill aplicado — producer tempo real pendente (D8) | ✅ Populado |
| `responded_form_ids` | Backfill aplicado — producer tempo real pendente (D7) | ✅ Populado |
| `journey_ids` | — | ❌ 0% populado — producer não existe (D9) |
| `segment_ids` | `apply_segment_membership_diff` | ✅ Ativo |

> **Regra permanente:** campo array vazio = segmento que usa esse campo retorna zero membros silenciosamente, sem erro. Antes de migrar qualquer condition para GIN, validar divergência = 0 contra a fonte normalizada.

**Mapeamento evento → fields_changed:**

| Evento | fields_changed | Priority |
|---|---|---|
| `purchase` | `monetary, frequency, recency_days, rfm_segment, bought_product_ids, last_purchase_at, r/f/m_score, value_tier, lifecycle_stage, churn_score, purchase_propensity` | 1 |
| `tag_added/removed` | `tag_ids` | 3 |
| `form_submitted` | `responded_form_ids` | 3 |
| `rfm_recalculated` | `recency_days, frequency, monetary, rfm_segment, r/f/m_score, value_tier, lifecycle_stage` | 5 |
| `profile_updated` | `name, phone, state, city, profession, birth_date` | 7 |
| `import_added` | `import_ids` | 8 |
| `fallback` | todos os campos | 9 |

---

## 13. CRM — Oportunidades e Timeline

O handler `executeCreateOpportunityNode` (em `processJourneyNode.ts`) cria ou atualiza oportunidades via automação.

**Dedup:** busca oportunidade `status='open'` no mesmo `pipeline_id` pelo contato. Chave preferencial e única: `party_id`. Fallback `contact_email` removido — migração party-first completa.

**Se encontrou:** calcula diff de campos → UPDATE oportunidade → INSERT em `crm.opportunity_stage_history` se etapa mudou → INSERT em `crm.opportunity_activities` → retorna `opportunity_updated`.

**Se não encontrou:** cria nova oportunidade → INSERT em `crm.opportunity_activities` com `activity_type='created'` → retorna `opportunity_created`.

**Regras da timeline (`crm.opportunity_activities`):**
- Append-only — nunca fazer UPDATE
- Idempotência via UNIQUE INDEX parcial em `(enrollment_id, node_id)` — reprocessamento não duplica
- Backfill usa `source='system'` com `source_meta=null` para não colidir com INSERTs de jornada
- Activity é obrigatória: se o INSERT falhar, o node falha inteiro e o journeys-worker faz retry — nunca usar `try/catch` isolado no INSERT de activity

`email_opened` / `email_clicked` — emitidos por trigger AFTER INSERT em `email.email_events` (resolve party via `recipient_id → campaign_recipients → crm.parties`)

`segment_entered` / `segment_exited` — emitidos por `apply_segment_membership_diff` a cada mudança de membership

`crm.opportunities.party_id` é NOT NULL enforçado — oportunidade sem contato é violação do modelo party-first. Trigger `sync_opportunity_summary` removido (Sprint 11). Colunas `has_open_opportunity` e `open_opportunity_count` dropadas de `contact_state` — violavam R36 (1:N nunca denormalizado). Substituídas por 10 campos `opportunity_exists` no worker planner, avaliados via EXISTS subquery direta em `crm.opportunities` com índice `idx_opportunities_party_tenant_status`. Ver R36 para lista completa de campos.

---

## 14. Checklist para Novo Provider

Adicionar um provider novo é criar **1 arquivo de mapper**. Zero mudança no banco.

**Edge Function:**

- [ ] Criar Edge Functions seguindo convenção de nomenclatura (ver R22):
  ```
  supabase/functions/{provider}-receiver-webhook/   ← URL pública, recebe do provider
  supabase/functions/{provider}-processor-worker/   ← worker interno, processa jobs
  supabase/functions/{provider}-oauth-callback/     ← se provider usa OAuth
  ```
- [ ] Criar `integrations.invoke_<provider>()` apontando para `/functions/v1/<provider>`
- [ ] Adicionar ELSIF no `enqueue_webhook_job` (webhook) ou invoke no cron (pull)
- [ ] Receber ou buscar dados brutos
- [ ] Salvar no schema do provider (`provider.raw_table`) — ledger imutável
- [ ] Criar `mapTransactions.ts`: raw → `NormalizedTransaction[]` com `_version: 1`
- [ ] Atenção no mapper: `amount` sempre em reais, status normalizado, `paid_at` em ISO 8601
- [ ] Agrupar em batches: max 100 itens, 512KB, 20 por tenant
- [ ] `INSERT integrations.jobs` com `handler='commerce:process_batch'`
- [ ] Se pull: respeitar rate limit via `scheduled_for` nos jobs
- [ ] Se webhook: salvar em `integrations.webhook_events` com `idempotency_key`

**Banco:**

- [ ] Schema `provider.*` com tabela de dados brutos
- [ ] Schema raw dedicado vira item obrigatório, não opcional
- [ ] `UNIQUE (tenant_id, provider_id)` em toda tabela raw
- [ ] Se precisa de sync periódico: adicionar pg_cron
- [ ] Registrar `source_name` em lowercase nos providers cadastrados

**NÃO precisa:**
- ~~Criar nova RPC no banco~~
- ~~Criar novo handler no ingestion-worker~~
- ~~Modificar `commerce.process_transactions_batch`~~

> `source_name` deve ser sempre o nome canônico do provider em lowercase — `'eduzz'`, `'doare'`, `'hotmart'`. Nunca usar variantes como `eduzz_import`, `eduzz_historical`, `eduzz_backfill`. Um provider = um `source_name` fixo para todos os pipelines (webhook, import histórico, reprocessamento). Variação de `source_name` no mesmo provider quebra o dedup cross-source.

> **Provider pull-based — padrão Sympla:** Providers sem webhook nativo usam cron-driven sync via Edge Function. Credencial por tenant (`s_token`) armazenada em `integrations.accounts.config`. Hierarquia Tenant → Events → Orders → Participants. Sync de participants usa janela de ciclo de vida: futuro (enrich de contato), ativo (-1d a +3d, checkins em tempo real), encerrado recente (sync final, seta `checkin_synced = true`), arquivado (nunca re-fetch). Datas da API sem offset tratadas como `America/Sao_Paulo`, nunca UTC.

---

## 14.1 Provider Sympla — Detalhes de Implementação

**Status:** ✅ Operacional (Sprint 10) — Fases 1–4 completas

**Tipo:** Pull-based (sem webhooks nativos). Credencial `s_token` por tenant em `integrations.accounts.config`.

**Hierarquia de dados:**

```
Tenant
└── Events (GET /events)
    └── Orders (GET /events/{id}/orders)          ← transações
    └── Participants (GET /events/{id}/participants)  ← contatos + checkin
```

**Schema no banco:** `sympla.*`
- `sympla.events` — cache de eventos com campos completos (host, category, address, `checkin_synced`)
- `sympla.orders` — pedidos com invoice_info (CPF ~12%, address ~24%), UTM, user_agent
- `sympla.participants` — participantes com phone extraído de `custom_form`, checkin, `checkin_event_emitted`
- `sympla.sync_cursors` — cursor por evento para sync incremental

**Cobertura de dados (tenant de referência):**

| Campo | Cobertura | Origem |
|---|---|---|
| email | 100% | orders.buyer_email |
| city/state | 23.5% | orders.invoice_info (só boleto/cartão com NF) |
| CPF | 12.3% | orders.invoice_info (boleto 100%, PIX/cartão opcional) |
| phone | 52.6% | participants.custom_form |
| checkin | 31.5% | participants.checkin array |

**Mapeamento de status:**

| Status Sympla | Status normalizado |
|---|---|
| `A` | `paid` |
| `P`, `NP` | `pending` |
| `NA`, `C` | `cancelled` |
| `R` | `refunded` |

**Datas:** API retorna sem offset — tratar como `America/Sao_Paulo` — appender `-03:00` no parse, nunca `Z`.

**Janela de sync de participants:**

| Janela | Critério | Ação |
|---|---|---|
| Futuro | `start_date > hoje` | Fetch para enrich de contato |
| Ativo | `-1d ≤ start_date ≤ +3d` | Fetch frequente — checkins em tempo real |
| Encerrado recente | `+3d < start_date ≤ +7d` AND `checkin_synced = false` | Fetch final → `checkin_synced = true` |
| Arquivado | `checkin_synced = true` | Nunca re-fetch |

**Handler histórico:** `sympla:historical_sync` no historical-sync-worker. Busca todos os eventos/orders/participants sem janela temporal. Atualiza `sync_cursors` após processar para que o pull-sync-worker não reprocesse. Enfileira `commerce:process_batch` (provider=`ingestion`).

**Fases pendentes:**
- Fase 5: checkin → `journeys.journey_events` (type `sympla_checkin`)
- Fase 6: `contact_state.event_checkin_count` + `analytics.contact_checkin_stats`

---

## 14.2 Provider Eduzz — Detalhes de Implementação

**Status:** ✅ Operacional — Webhook-driven + OAuth + import histórico

**Tipo:** Webhook-driven (real-time). Credenciais OAuth + webhook_secret em `integrations.accounts.config`.

**Edge Functions:**
- `eduzz-receiver-webhook` — URL pública, recebe webhooks da Eduzz. Valida HMAC (`x-eduzz-signature`), salva raw em `integrations.webhook_events`, mapeia → NormalizedTransaction, enfileira job com `provider = 'ingestion'`.
- `eduzz-oauth-callback` — Fluxo OAuth, troca code por token, dispara import histórico.

**OAuth — app centralizado KreatorsHub:**
- `client_id` e `client_secret` em env vars (`EDUZZ_CLIENT_ID`, `EDUZZ_CLIENT_SECRET`) — nunca por tenant
- `access_token` de longa duração — API Eduzz não retorna `refresh_token` nem `expires_in`
- Callback salva: `access_token`, `oauth_connected_at`, `oauth_user_id`
- Import histórico disparado automaticamente no callback via `createHistoricalImportJobs()`

**Webhook verification:**
- Ping test: receiver responde 200 + grava `webhook_verified_at = now()`
- Eventos reais: validação HMAC com Origin Key (`webhook_secret`)
- UI faz polling a cada 5s enquanto `webhook_verified_at IS NULL` — badge muda para verde automaticamente

**Token expirado — fluxo defensivo:**
```
Worker recebe 401
  → tenta refresh (1x) → falha (sem refresh_token)
  → invalidateEduzzToken(): access_token = null, oauth_connected_at = null
  → throw TokenExpiredError
  → blockJob('TOKEN_EXPIRED')
  → UI detecta oauth_connected_at = null → banner "Conexão expirada — reconectar"
```

**Import histórico — Phase 2 (D12 implementado):**
```
eduzz:import_sales_page
  Fase 1: fetch API → eduzz.buyers + eduzz.invoices (raw ledger)
  Fase 2: paid invoices → NormalizedTransaction[]
    → jobs commerce:process_batch (provider='ingestion', priority=8)
    → ingestion-worker processa igual webhook
```

**Webhook URL (tenant-facing):**
- Formato slug: `https://core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook/{tenant_slug}`
- Formato legacy: `https://core.kreatorshub.com.br/functions/v1/eduzz-receiver-webhook?tenant_id={uuid}`

**Schema no banco:** `eduzz.*`
- `eduzz.invoices` — invoices completas com dados de buyer, affiliates
- `eduzz.buyers` — compradores com dados de contato
- `eduzz.contracts` — contratos/assinaturas

**Handlers:**

| Event type | Handler |
|---|---|
| `myeduzz.invoice_*` (paid, opened, canceled, refunded, chargeback, etc.) | `eduzz:process_invoice` |
| `myeduzz.contract_*` | `eduzz:process_contract` |
| `sun.cart_abandonment` | `eduzz:process_cart_abandonment` |
| `myeduzz.commission_processed` | `eduzz:process_commission` |
| `nutror.*` (enrollment, lesson, module, certificate) | `eduzz:process_learning` |
| `ping` | `eduzz:ping` |

**Prioridades:** `invoice_paid` = 1, `refunded`/`chargeback` = 2, `cart_abandonment` = 3, default = 5.

**Railway worker:** `historical-sync-worker` — import histórico e enrich (`ENABLED_HANDLERS=eduzz:enrich_invoice,eduzz:enrich_sales,eduzz:import_sales_page,eduzz:import_products,sympla:historical_sync`).

---

## 15. Checklist para Novo Nó de Automação

- [ ] Implementar handler em `workers/journeys/src/handlers/nodes/`
- [ ] Registrar em `processJourneyNode.ts`
- [ ] Implementar idempotência via `journeys.journey_executions`
- [ ] Se modifica o contato: chamar `analytics-events.ts` para atualizar `contact_state`
- [ ] Se envia comunicação: delegar para worker especializado (email, WhatsApp)
- [ ] Se tem estado temporal (wait): salvar `wait_until`, scheduler acorda
- [ ] Se cria/modifica entidade CRM: registrar em tabela de activities correspondente
- [ ] Se tem condição/branch: usar `evaluateConditionSingle()` do Condition Engine (R40)
- [ ] Se tem decisão: persistir em `journey_executions` (`decision_result`, `decision_input_snapshot`) (R42)
- [ ] Se tem skip: registrar nós pulados em `journey_executions` com razão

---

## 16. Deploy no Railway — Configuração por Service

Todos os services usam build context = raiz do repo. Cada service define seu Dockerfile via env var `RAILWAY_DOCKERFILE_PATH`.

**Separação de workers por perfil, não por provider:** Worker único com múltiplas réplicas é o modelo correto para escala. Separar só quando perfis divergem em SLA: realtime (webhooks, segundos) vs batch (imports, minutos). `journeys-event-worker` foi separado do `journeys-worker` por ter SLA < 5s vs minutos.

---

## 17. Observabilidade — O que monitorar

### Filas e saúde do sistema

Monitorar regularmente: `integrations.jobs`, `analytics.segment_eval_queue`, `journeys.processing_jobs`, `journeys.event_processing_jobs`, `email.campaign_sends` — todos por status. Verificar `contact_state`: total de contatos, quantos com `rfm_segment` preenchido, timestamp do update mais recente. Verificar `fallback_log` — se > 0 na última hora, algum handler esqueceu o analytics pipeline.

**Separação de workers por perfil, não por provider.** Mesma imagem Docker, env vars diferentes. Usar `ENABLED_JOB_TYPES` para restringir job types por worker. Usar `WORKER_ONLY` / `IS_SCHEDULER` para separar execução de agendamento. Separar em workers distintos apenas quando SLAs divergem (ex: realtime < 5s vs batch em minutos). Configs detalhadas de cada service vivem no Railway.

### Alertas — se diferente de zero

| Query | O que indica |
|---|---|
| `fallback_log` com registros em 1h | Handler esqueceu de chamar analytics pipeline |
| `segment_eval_queue` com `status='error'` | Erro no evaluator |
| `integrations.jobs` com `status='blocked'` | Mapper/contrato quebrado — requer fix + deploy |
| `integrations.jobs` com `status='dead'` | Esgotou tentativas — verificar `last_error` |
| `journey.processing_jobs pending` > 10min | Journey worker parado |
| `max(now()-state_updated_at)` em `contact_state` > 30min | Pipeline atrasado |
| `job_errors` crescendo com mesmo `error_code` | Erro sistemático — verificar mapper |
| **`event_processing_jobs pending` > 2min** | **journeys-event-worker parado** |
| **`event_processing_jobs` com `status='blocked'`** | **`trigger_config` inválido — requer fix** |
| **Latência P95 evento→enrollment > 5s** | **Worker com sobrecarga ou deadlock** |
| **`backfill_jobs` sem atualização em 10min** | **Backfill travado** |

### Queries de observabilidade — journeys-event-worker

Ver `workers/journeys-event/src/monitoring.sql` para queries completas:

1. Throughput por minuto (última hora)
2. Latência P50/P95/P99
3. Fila atual por status
4. Jobs blocked por motivo
5. Jobs dead (últimos 20)
6. Backfills ativos
7. Alerta: pending > 2min
8. Alerta: P95 > 5s

---

## 17.1 Platform Health Observability (D-CRON-3)

**Status:** ✅ Operacional (v7.25).

Camada própria de observability arquitetural — desacoplada do mecanismo observado
(R52). Mudança no path observado (ex: `segment-eval-fallback` cron → worker em
D-CRON-4) NÃO obriga reescrever a view nem o handler de alerta.

### Arquitetura — três camadas decoupled

```
┌──────────────────────────────────────────────────────────────┐
│  Camada A — TELEMETRIA                                       │
│  analytics.event_health_metrics (particionada mensal, 90d)   │
│  Populada por:                                               │
│    • cron `etl-event-health-metrics` (hourly, categoria A)   │
│    • workers via RPC analytics.emit_health_metric()          │
│      ex: segment-eval-worker emite `segment_membership_      │
│          changes` em batch 60s                               │
└──────────────────────────────────────────────────────────────┘
                          ↓ lida APENAS por
┌──────────────────────────────────────────────────────────────┐
│  Camada B — SAÚDE                                            │
│  analytics.v_event_driven_health (view)                      │
│  Tier classification per-tenant: HEALTHY / WARN / ERROR /    │
│  CRITICAL / INSUFFICIENT_DATA                                │
│  Baseline: rolling 7d P50/P95/P99 do PRÓPRIO tenant,         │
│  excluindo a hora atual (evita auto-inflação).               │
│  Whitelist via core.platform_health_whitelist.               │
└──────────────────────────────────────────────────────────────┘
                          ↓ lida APENAS por
┌──────────────────────────────────────────────────────────────┐
│  Camada C — ALERTA                                           │
│  workers/journeys/src/handlers/healthCheck.ts (5min cadence) │
│  Emite Sentry em transição p/ tier pior. Recovery silencioso │
│  (HEALTHY sustentado 30min). Dedup via                       │
│  analytics.platform_alert_state.                             │
└──────────────────────────────────────────────────────────────┘
```

### Tier classification

| Condição | Tier |
|---|---|
| baseline samples < 168 (7d × 24h) | `INSUFFICIENT_DATA` |
| `error_sustained_1h` E `hits_1h > p99` | `CRITICAL` |
| `hits_1h > p95` | `ERROR` |
| `hits_1h > p50 * 1.5` | `WARN` |
| caso contrário | `HEALTHY` |

### Métricas coletadas (v7.25)

| metric_name | semântica | fonte v7.25 | fonte D-CRON-4 |
|---|---|---|---|
| `fallback_hits` | contacts detectados pelo cron `segment-eval-fallback` | ETL hourly de `analytics.fallback_log` | worker emite direto (cron some) |
| `event_driven_hits` | event-driven entries no segmento | ETL hourly de `analytics.contact_events` (event_type='segment_entered', source='system') | inalterado |
| `eval_queue_throughput` | jobs com status='processed' por hora | ETL hourly de `analytics.segment_eval_queue` | inalterado |
| `eval_queue_error_rate` | errors / (errors + processed) por hora (mín 10 jobs) | ETL hourly de `analytics.segment_eval_queue` | inalterado |
| `eval_queue_p95_latency_ms` | p95 latency `processed_at - claimed_at` (ms) | ETL hourly de `analytics.segment_eval_queue` | inalterado |
| `segment_membership_changes` | sum(entered + exited) por bucket hora | `segment-eval-worker` emite em batch 60s direto via `emit_health_metric` | inalterado |

### Regras de alerta (Camada C)

1. **WARN não dispara Sentry.** Tier informativo apenas — handler trackeia state mas não notifica.
2. **ERROR/CRITICAL alertam em transição PARA pior** (HEALTHY/WARN/null → ERROR/CRITICAL OU ERROR → CRITICAL).
3. **Re-alerta no mesmo tier ≥ ERROR só após 1h.** Evita alert fatigue.
4. **Transição CRITICAL → ERROR não re-alerta** (tier melhor sem ser HEALTHY).
5. **Recovery silencioso:** ERROR/CRITICAL → HEALTHY sustentado 30min apenas atualiza state + log info. Sem Sentry. Auditoria via `platform_alert_state.last_changed_at`.
6. **Whitelist:** `core.platform_health_whitelist` (auditável, com `reason` CHECK constraint, `added_by`, `expires_at`). Começa vazia em v7.25. Adicionar tenant nominalmente via migration.

### Pré-requisito de produção

Sentry DSN do project `workers-railway` deve estar em `SENTRY_DSN_WORKERS` no
Railway. Se ausente, `initSentryWorker` loga `sentry_disabled` e o handler segue
funcionando — escreve em `platform_alert_state` + log estruturado, mas não emite
no Sentry. Permite deploy do código antes do DSN estar configurado.

### Arquivos

| Arquivo | Papel |
|---|---|
| `supabase/migrations/20260501100000_dcron3_observability.sql` | Schema completo (whitelist, tabela particionada, RPC, view, ETL function, partition rotation, TTL purge, 3 crons categoria A) |
| `workers/shared/src/sentry.ts` | `initSentryWorker`, `captureWorkerException`, `flushSentry` — reutilizável por qualquer worker. No-op se DSN ausente. |
| `workers/journeys/src/handlers/healthCheck.ts` | Handler `runEventDrivenHealthCheck` + `shouldAlert`/`isWorseTier` puros (testáveis). |
| `workers/journeys/src/index.ts:runSchedulerLoop` | Hook `lastHealthCheck` (5min interval). |
| `workers/analytics/src/segment-eval-worker.ts` | `flushHealthMetricBatch` (60s) emite `segment_membership_changes`. |

---

# PARTE B — SEGMENTAÇÃO & RULE ENGINE

## 18. Rule Engine v2 — Segmentação Escalável

### Arquitetura de condições — dois tiers

O evaluator opera em dois tiers para equilibrar performance e expressividade:

**Tier 1 — State conditions:** avaliadas diretamente contra `contact_state`. Zero queries adicionais. O(1) por contato. Inclui RFM, lifecycle, tags, segmentos, CRM summary e email engagement (timestamps).

**Tier 1.5 — Product stat conditions:** avaliadas via EXISTS em `analytics.contact_product_stats`. Uma subconsulta por condição de produto, mas sobre conjunto já filtrado pelo Tier 1.

**Tier 2 — Event conditions (futuro):** EXISTS em `contact_events` para casos que não podem ser materializados (ex: "abriu campanha X especificamente"). Reservado para Sprint 9+.

### Tabela de summary por produto — `analytics.contact_product_stats`

- Particionada por `HASH(tenant_id)` — 8 partições
- Chave: `(tenant_id, party_id, product_id)`
- Campos: `first_purchase_at`, `last_purchase_at`, `purchase_count`, `total_spent`
- Populada por `process_purchase_analytics` a cada compra paid

### evaluation_mode em `analytics.segments`

- `event_driven` (default): reavaliado apenas quando `fields_changed` relevantes chegam
- `time_driven`: requer re-scan periódico — condições com `within_last/before/after` expiram com o tempo sem evento
- Inferido automaticamente pelo `inferEvaluationMode()` no service de segmento

### Operadores semânticos (v2)

- Timestamps: `op: 'within_last' | 'before' | 'after'` + `window_days` + `window_unit: 'days' | 'hours'`
- Números/contagens: `op: 'gte' | 'lte' | 'gt' | 'eq'` + `value`
- Nunca `value` em dias — sempre `window_days` explícito

> **Regra v1 de produto — sem rolling counts:** O builder v1 permite apenas timestamps (`first/last_purchase_at`) e contagens lifetime (`purchase_count` total). Rolling counts com janela deslizante (ex: `purchase_count_30d`) são V2 — só implementar quando o `rfm-rolling-refresh` for estendido para suportar decay.

### Arquivos

- `workers/analytics/src/types/segmentCondition.ts` — interfaces: `ProductStatCondition`, `EmailEngagementCondition`, `CrmCondition`
- `workers/analytics/src/segment-rules-evaluator.ts` — avaliação in-memory para crm e email_engagement
- `workers/analytics/src/segment-sql-builder.ts` — SQL builder para product_stat via EXISTS + `inferEvaluationMode()`

### Estado do evaluator

| Condition | Implementação | Performance |
|---|---|---|
| `bought_product` / `not_bought_product` | GIN `bought_product_ids && ARRAY[]` | ✅ 530x — ativo |
| `has_tag` / `not_has_tag` | 4 subqueries (diretas + product tags transitivas) | Bloqueado — product tags não estão em `tag_ids` |
| `responded_form` / `not_responded_form` | Subquery `lead_form_submissions` | Phase 2 — após producer tempo real |
| `from_import` / `not_from_import` | Subquery `lead_import_history` | Phase 2 — após producer tempo real |
| Campos RFM / lifecycle / scores | Colunas diretas via `v_segment_base` | Já O(1) |

---

## 18.1 Segmentation Engine v2 — Implementação

**Status:** ✅ Operacional (Sprint 10)

**Localização:** `workers/analytics/src/segmentation/`

**Arquivos:**

| Arquivo | Responsabilidade |
|---|---|
| `types.ts` | Interfaces: ConditionLeaf, ConditionGroup, SegmentAST, EvaluationContext, ExecutionMeta |
| `normalizer.ts` | toNNF + canonicalize + astHash — elimina NOT via De Morgan |
| `validator.ts` | Validação de AST contra registry — retorna erros tipados |
| `semanticFieldRegistry.ts` | Campos hardcoded com resolver, cost_hint, missing_semantics |
| `executionRegistry.ts` | Mapeamento campo → tabela/coluna de execução |
| `resolvers.ts` | 5 resolvers: snapshot_scalar, snapshot_array_contains, profile_scalar, summary_exists, custom_trait_scalar |
| `planner.ts` | compileSQL() + compileMemory() — recursivo por nó AND/OR |
| `fieldCatalog.ts` | getSegmentFieldCatalog() — combina campos estáticos + field_definitions do tenant |
| `traitProducer.ts` | BLOCK_TYPE_MAP, processSubmission, updatePartyIdentity |

**Resolvers:**

| Resolver | Fonte | Custo | Cobertura |
|---|---|---|---|
| `snapshot_scalar` | `analytics.contact_state` | instant | RFM, lifecycle, scores, timestamps |
| `snapshot_array_contains` | `analytics.contact_state` (GIN) | instant | bought_product_ids, tag_ids, segment_ids |
| `profile_scalar` | `crm.parties` / `crm.party_person` | fast | phone, profession, campos canônicos |
| `summary_exists` | `analytics.contact_product_stats` | fast | product_first_purchase, product_purchase_count |
| `custom_trait_scalar` | `crm.party_trait_*` | fast | qualquer campo de formulário do tenant |

**Operadores suportados:**
- Scalar: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`
- Temporal: `before`, `after`, `within_last`, `between_dates`, `not_between_dates`
- Array: `in`, `not_in`, `contains_any`, `not_contains_any`
- Text: `starts_with`, `contains`
- Existencial: `is_null`, `is_not_null`

**Coexistência com legacy:** `refreshSegments.ts` usa `rules_json_v2` se `ast_version >= 1`, senão fallback para `build_segment_where_clause` (D37: função dropada do banco — fallback nunca atingido, dead code confirmado).

**Estado da migração:** Migração completa — 42/42 segmentos manuais com v2. Ver D33. Colunas adicionadas em `analytics.segments`: `rules_json_v2`, `ast_version`, `ast_hash`, `migrated_at`, `migration_warnings`.

---

## 18.15 Segment Builder — Blocos Paralelos

**Status:** ✅ Operacional (Sprint 11)

### Modelo mental

O builder de segmentação usa **blocos paralelos** no primeiro nível. Cada bloco é uma lista de condições conectadas por AND (implícito). Os blocos se conectam entre si com E ou OU (operador **entre** blocos, não dentro).

```
┌─────────────────────────────────────┐
│ Bloco 1                             │
│   • Comprou produto A   (AND)       │
│   • Comprou produto B   (AND)       │
│   • Comprou produto C   (AND)       │
└─────────────────────────────────────┘
                 ─── OU ───
┌─────────────────────────────────────┐
│ Bloco 2                             │
│   • Comprou produto D               │
└─────────────────────────────────────┘
                 ─── E ───
┌─────────────────────────────────────┐
│ Bloco 3                             │
│   • Cidade = São Paulo              │
└─────────────────────────────────────┘
```

**Lógica:** `(A E B E C) OU (D)` → dessa lista, filtra `cidade = SP`

### Como funciona internamente

**UI:** Cada bloco é um `UIBlock { group: RuleGroupType, joinOp: 'AND' | 'OR' }`. O `joinOp` define como ele se conecta ao bloco anterior (primeiro bloco é ignorado).

**Recompose (UI → Backend AST):** Blocos consecutivos com mesmo `joinOp` são agrupados. Exemplo:
- `[B1, OU:B2, E:B3]` → `AND( OR(B1, B2), B3 )`
- `[B1, E:B2, E:B3]` → `AND(B1, B2, B3)`
- `[B1, OU:B2, OU:B3]` → `OR(B1, B2, B3)`

**Decompose (Backend AST → UI):** Recursivo — achata grupos aninhados de volta para blocos planos com joinOps corretos. `AND(OR(B1,B2), B3)` → `[B1, OU:B2, E:B3]`.

**Controle de ciclo:** `internalChangeRef` evita que mudanças internas (emitChange) trigem re-decompose via useEffect. Só mudanças externas (template, load do DB) re-decompõem.

### Regras permanentes

1. Condições dentro de um bloco são **sempre AND** — sem toggle interno
2. O operador E/OU é **entre blocos**, não dentro
3. Botões "Adicionar bloco E" / "Adicionar bloco OU" definem o `joinOp` do novo bloco — blocos existentes não mudam
4. `RuleGroup.tsx` usa `maxDepth={0}` dentro de blocos — sem sub-grupos aninhados na UI

### Arquivos

| Arquivo | Papel |
|---|---|
| `src/components/segmentation/SegmentBuilder.tsx` | Wrapper: decompose/recompose, state de UIBlocks, preview |
| `src/components/segmentation/RuleGroup.tsx` | Renderiza um bloco (condições + add/remove). `maxDepth` controla nesting |
| `src/components/segmentation/RuleRow.tsx` | Uma condição individual (campo, operador, valor) |
| `src/types/segmentation.ts` | Tipos: `RuleGroup`, `RuleCondition`, `FieldDefinition` |
| `src/types/segmentation-v2.ts` | Tipos AST v2: `ConditionNode`, `GroupNode`, `RulesJsonV2` + helpers de conversão |
| `src/lib/segmentation/normalizeAstV2.ts` | Adapter de normalização aplicado no load (R48). Resiliência a shapes legacy até Fase 2/4 |

### Shape canônico de `rules_json_v2`

O AST v2 dos segmentos rule-based segue o shape canônico:

- `root.children` contém EXCLUSIVAMENTE nós `kind: 'group'` (cada filho é um UIBlock).
- Conditions vivem dentro de cada bloco (AND interno).
- `root.op` define o joinOp entre blocos: AND ou OR.
- Conditions com `__ui_locked: true` são metadado de UI somente — nunca persistido no banco (strip em `toBackendAST`).
- Sentinelas `all_leads` / `all_customers` são deprecated (dropadas em load).
- Fields fora do `FIELD_OPTIONS` da UI mas presentes no `eval_condition_v2` do backend são preservados em modo locked até a UI ganhar suporte.

O adapter `normalizeAstV2ForUI` em `src/lib/segmentation/normalizeAstV2.ts` aplica esse shape em runtime durante o load (defensive). Após Fase 2 (backfill), o adapter vira no-op porque o banco terá shape A em todos os segmentos. Após Fase 4 (trigger guardrail), gravar shape não-canônico fica impossível.

**Classificação de warnings:** o normalizador emite seis tipos de warning. `field_dropped` e `field_locked` são **actionable** (renderizam banner amarelo na UI — usuário precisa revisar/recriar). `shape_wrapped`, `field_aliased`, `op_aliased`, `expanded_array_op` são **cosméticos** (transformações idempotentes que preservam semântica — silenciados no banner, mantidos no array para devs). Helpers: `hasActionableWarnings()` e `summarizeActionableWarnings()`.

---

## 18.16 Segment Refresh Queue — Cache Assíncrono

**Status:** ✅ Operacional (Sprint 11)

### Problema resolvido

O trigger síncrono `refresh_segment_parties()` bloqueava 400-800ms por save de segmento. Com 100+ tenants e segmentos grandes (22k membros), isso causava timeouts.

### Arquitetura

```
User salva segmento
    │
    ▼
Trigger BEFORE (< 10ms)
    ├── SET refresh_pending = true
    └── INSERT INTO segment_refresh_queue

Journeys worker (polling 5s)
    │
    ▼
process_segment_refresh_queue()
    ├── FOR UPDATE SKIP LOCKED
    ├── refresh_segment_parties() (400-800ms em background)
    ├── DELETE da fila
    └── SET refresh_pending = false
```

### Componentes

| Componente | Papel |
|---|---|
| `analytics.segment_refresh_queue` | Tabela fila (PK = segment_id, natural dedup) |
| `analytics.segments.refresh_pending` | Flag booleana para feedback na UI |
| `trg_queue_segment_refresh` | Trigger BEFORE no `analytics.segments` — enfileira em INSERT, mudança de regras, ou cache vazio |
| `analytics.process_segment_refresh_queue()` | RPC batch processor (SKIP LOCKED, limit configurable) |
| `analytics.queue_stale_segments()` | Enfileira segmentos não atualizados há 60+ min |
| `analytics.queue_segment_for_refresh()` | Helper para enfileirar manualmente |
| `workers/journeys/src/index.ts` | Scheduler polls `process_segment_refresh_queue` a cada 5s |
| `workers/journeys/src/handlers/checkSegmentEntries.ts` | Safety net: se cache vazio, enfileira e retorna (próximo tick encontra membros) |

### Regras permanentes

1. Trigger NUNCA chama `refresh_segment_parties()` diretamente — sempre enfileira
2. Worker usa `FOR UPDATE SKIP LOCKED` — sem contenção entre instâncias
3. Após 3 tentativas com erro, item é removido da fila (`refresh_pending` resetado)
4. Segmentos cluster/lookalike são ignorados (têm seus próprios pipelines)

---

## 18.2 Perfil do Contato — party_profile e Traits

**Arquitetura em três camadas:**

```
analytics.contact_state      → comportamento derivado (instant)
                                RFM, lifecycle, scores, memberships
crm.party_person             → identidade base (já existe)
crm.party_profile            → snapshot de perfil (fast)
                                colunas canônicas: phone, profession,
                                birth_date, city, state, country,
                                missing_semantics, source_system,
                                source_field_id (mapeia block_id do formulário)
crm.party_trait_text         ↗
crm.party_trait_number       → projeção tipada para segmentação (fast)
crm.party_trait_timestamp    → índice por (tenant_id, field_key, value_*)
crm.party_trait_boolean      → cobertura completa de operadores por tipo
crm.party_trait_multi_value  ↘
```

**Por que não JSONB puro para segmentação:** GIN cobre igualdade e containment mas não range queries, comparações numéricas e datas com precisão de planner. Traits tipadas cobrem todos os operadores com índice previsível.

**Fluxo de formulário → traits:**

```
forms.form_versions status → 'published'
  → trg_enqueue_field_sync
  → handler forms:sync_field_definitions
  → syncFormFieldDefinitions():
      para cada bloco BLOCK_TYPE_MAP[type] = 'trait':
        INSERT INTO crm.field_definitions
          source_field_id = block.id
          field_key = slugify(block.title)
        ON CONFLICT (tenant_id, field_key) DO UPDATE

forms.form_submissions status → projection
  → handler forms:project_traits
  → processSubmission() em sql.begin():
      1. normalizeAnswer() por tipo de bloco
      2. upsert em party_trait_* por field_type
      3. merge party_profile.custom_fields
      4. updatePartyIdentity() para blocos identity
      5. INSERT segment_eval_queue fields_changed = [field_keys]
```

**BLOCK_TYPE_MAP — 22 tipos:**

| Categoria | Tipos | Kind |
|---|---|---|
| Identidade | first_name, last_name, full_name, email, phone | identity → party_person |
| Texto livre | short_text, long_text | trait → text |
| Seleção única | single_choice, picture_choice, dropdown | trait → single_select |
| Boolean | yes_no | trait → boolean |
| Seleção múltipla | multiple_choice, checkbox | trait → multi_select |
| Numérico | number, rating, nps, ranking | trait → number |
| Data | date | trait → timestamp |
| Compostos | contact_info, address | skip (futuro) |
| Display | statement, website | skip |

**Typeform — integração parcial (implementada):**
Schema `typeform.*` com 6 tabelas (`forms_raw`, `responses_raw`, `import_runs`, `sync_checkpoints`, `themes_cache`, `webhook_events`). Forms e responses migrados via `historical-sync-worker` e `pull-sync-worker`. Handlers: `typeform:sync_forms`, `typeform:sync_responses`, `typeform:promote_forms`, `typeform:promote_responses`, `typeform:enrich_themes`, `typeform:backfill_settings`. Ver R34/R35 para lições aprendidas da implementação.

---

## 19. Soft-delete — segments e forms

- `analytics.segments`: coluna `deleted_at timestamptz` adicionada. Soft-delete via `UPDATE SET deleted_at = NOW(), is_active = false` — nunca DELETE físico
- `forms.forms` e `smart_forms.forms`: `deleted_at timestamptz` adicionado
- 7 RPCs corrigidas para filtrar `deleted_at IS NULL`
- `segment-eval-worker` filtra `deleted_at IS NULL` no claim
- `refreshSegments` filtra `deleted_at IS NULL`
- Dropdowns de automações no builder filtram deletados + aviso visual quando segmento/form foi deletado

---

## 20. Jobs stuck — prevenção

`analytics.cleanup_stale_segment_eval_jobs(interval)` — RPC criada. Libera jobs claimed há mais que o threshold (default 10min). Chamada a cada ~60s no `segment-eval-worker`. Configurável via `SEGMENT_EVAL_STALE_MINUTES`.

---

# PARTE C — REGRAS PERMANENTES

## 21. Regras extraídas de incidentes

Cada regra foi extraída de um incidente real. Sem narrativa — apenas o que o sistema aprendeu e nunca deve esquecer.

| # | Regra | Origem |
|---|-------|--------|
| R1 | Enqueue deve verificar TODAS as fontes, sem filtrar por `source_name`. Um provider = um `source_name` canônico | Dedup cross-source |
| R2 | Mappers usam valor bruto do provider, nunca líquido. Moeda estrangeira → BRL com cotação do payload | Amount |
| R3 | Idempotência de enqueue usa tabela de destino (`commerce.transactions`), nunca campo de status na tabela raw | Enqueue |
| R4 | Um provider = uma Edge Function. Deploy de um isolado do outro | Edge Functions |
| R5 | `raw_source` é opcional quando provider tem schema raw dedicado. Obrigatório caso contrário | Rastreabilidade |
| R6 | Após deploy de Edge Function: confirmar versão no dashboard + validar primeiro ciclo nos logs | Deploy |
| R7 | Enum salvo no banco deve ter CHECK constraint. Tipos TS e valores DB auditados juntos em PRs | Contrato frontend→banco |
| R8 | Campos de controle de fluxo (reentrada, elegibilidade) = colunas tipadas, nunca `settings_json` | JSON vs colunas |
| R9 | Nenhuma view recalcula RFM em runtime. `contact_state` é o snapshot. NTILE só em `refreshRfm.ts` | RFM fonte única |
| R10 | Modelo errado → dropar, não adaptar. Adaptar prolonga coexistência de dois modelos | Party-first |
| R11 | Migração de coluna: (1) backfill dados (2) migrar escritas (3) migrar leituras (4) DROP após 1 sprint | DROP seguro |
| R12 | Trigger que atribui role superior deve revogar roles inferiores. Lead e customer mutuamente exclusivos | Role hierarchy |
| R13 | NTILE só em `refreshRfm.ts`. Dois produtores de RFM = bug. Regra 365d vive ali ONLY | RFM produtor único |
| R14 | Tabelas quentes: auditar índices e dead tuples a cada sprint. Índices ≤ queries distintas | Performance |
| R15 | Tabelas de membership não carregam metadata de algoritmo. Score/tier = tabela do compute | Membership vs metadata |
| R16 | Condições correlacionadas (produto×tempo) → tabela de summary dedicada, nunca JSONB | Summary tables |
| R17 | Campo array vazio = segmento silenciosamente errado. Validar divergência = 0 antes de migrar para GIN | Array producers |
| R18 | Clusters: algoritmo plugável, output é contrato fixo. `segment_parties` com `source_run_id` indexável | Clusters/Lookalikes |
| R19 | Datas de providers brasileiros sem offset explícito = `America/Sao_Paulo` (UTC-3), nunca UTC. Appender `Z` causa erro de +3h em todos os timestamps. | Sympla timezone bug |
| R20 | Toda fila de trabalho com status terminal (`processed`, `success`, `completed`) precisa de TTL de purge definido na criação. Fila não é ledger — não acumula histórico. `segment_eval_queue` sem purge → 1.1M rows, 482MB, banco travado. | Queue bloat incident |
| R21 | Primeiro provider não ganha schema próprio. Config de integração = `integrations.accounts` desde o dia 1. Schema dedicado só para dados raw do provider (`eduzz.invoices`, `eduzz.buyers`). `eduzz.integrations` legada causou workarounds `eduzzLegacy` no frontend e 7 RPCs duplicadas. | Eduzz legacy migration |
| R22 | Edge Functions seguem convenção `{provider}-receiver-webhook` (URL pública) e `{provider}-processor-worker` (worker interno). Nomes ambíguos causam configuração de URL errada no provider e interrupção silenciosa de webhooks. | Rename eduzz Sprint 10 |
| R23 | Workers devem passar `p_handlers text[]` no RPC `claim_jobs` para filtrar no SQL, nunca em application code. Claim sem filtro trava jobs de outros handlers e gasta locks desnecessários. | claim_jobs Sprint 10 |
| R24 | Webhook URLs usam domínio customizado (`core.kreatorshub.com.br`) com slug no path, nunca URL raw do Supabase. Slug é preferencial; `?tenant_id=uuid` é fallback. Receiver resolve slug → tenant_id via `core.tenants`. | Custom domain Sprint 10 |
| R25 | Edge Function nunca é worker. Se precisa de poll loop, claim de jobs ou processar lógica pesada, é Railway worker. Sintomas de violação: Edge Function com `while(true)`, `claimJobs()`, múltiplas chamadas RPC em sequência, timeout > 10s, ou lógica que cresce com volume de dados. Qualquer um desses sinais = mover para Railway worker imediatamente. | eduzz-processor-worker migration Sprint 15 |
| R26 | Webhook de provider deve criar jobs com `provider='ingestion'`. O ingestion-worker é o único consumidor de processamento downstream de webhooks. Edge Function dedicada por provider só existe para recepção e normalização O(1). | eduzz-processor-worker migration Sprint 15 |
| R27 | OAuth providers usam app centralizado da plataforma, nunca app por tenant. Client credentials em env vars globais. Token por tenant salvo em `integrations.accounts.config`. Dependência de app OAuth por tenant = risco operacional: cliente revoga → todos os tenants perdem acesso. | Eduzz OAuth migration Sprint 10 |
| R28 | Multi-step writes que criam estado dependente (enrollment + job, entity + event) devem estar em `sql.begin()`. `postgres.js` suporta transações com PgBouncer transaction mode e `prepare: false`. Comentário `sql.begin() não suportado` é incorreto e causou fanout não-atômico nas jornadas. | Principal Eng Audit D18 |
| R29 | PostgreSQL NÃO propaga `relrowsecurity` para partições existentes. Policies na tabela pai SÃO herdadas, mas cada partição precisa de `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` explícito. Sem isso, acesso direto à partição via PostgREST bypassa RLS. Foram 49 partições habilitadas individualmente no Security Sprint 2. | Security Sprint 2 partition RLS |
| R30 | `contact_state` contém exclusivamente comportamento derivado. Campos de perfil (phone, profession, qualquer atributo estático do contato) pertencem a `crm.party_profile`. Campos customizados do tenant pertencem a `crm.party_trait_*`. Materializar atributos de perfil ou traits em `contact_state` é violação arquitetural que destrói a fronteira entre snapshot comportamental e dados brutos de perfil. | Segmentation Engine v2 — Sprint 10 |
| R31 | CHECK constraints de enums (`triggered_by`, `status`, etc.) devem ser auditadas antes de adicionar novo valor no código. Constraint com lista incompleta causa insert silenciosamente rejeitado. Sempre: (1) verificar constraint existente, (2) migration adicionando novo valor, (3) deploy migration antes do código. | Form trait pipeline — `form_submission` vs `form_submitted` |
| R32 | Unique index de coalescing (`UNIQUE (tenant_id, job_type) WHERE status = 'pending'`) é correto para jobs coalescentes (refresh_features, refresh_rfm) mas bloqueia jobs per-entity (project_traits, sync_field_definitions). Jobs per-entity precisam de `payload->>'submission_id'` no unique ou exclusão do job_type no index parcial. Misturar os dois padrões na mesma tabela requer WHERE clause que exclua os job_types per-entity. | Form trait backfill — unique constraint bloqueou enqueue em massa |
| R33 | Job enfileirado na tabela/worker errado bloqueia silenciosamente. `project_traits` foi enfileirado em `integrations.jobs` mas integrations-worker só consome providers (eduzz, sympla, apidozap). Jobs ficaram pending sem erro. Antes de enfileirar: (1) confirmar qual worker consome a tabela, (2) verificar se job_type está na allowlist (`ENABLED_JOB_TYPES`/`KNOWN_JOB_TYPES`). Trait projection = `analytics.processing_jobs` → analytics-worker. | Form trait pipeline — jobs pending indefinidamente |
| R34 | **Todo novo provider obrigatoriamente passa pelo sistema canônico de integração.** Ao conectar qualquer novo provider (Typeform, HubSpot, Google Calendar, etc.), o caminho é sempre: `integrations.accounts` → `integrations.capabilities` → `integrations.jobs` → `pull-sync-worker` ou `historical-sync-worker`. É **proibido**: (1) criar tabelas `{provider}.import_runs` ou equivalentes para controlar execução, (2) criar workers isolados por provider fora do sistema canônico, (3) fazer chamadas de API de sync diretamente de Edge Functions, (4) implementar scheduling fora de `integrations.jobs`. Padrão de nomeação: `{provider}:{ação}` (ex: `typeform:sync_responses`, `typeform:promote_forms`). **Como identificar violação:** se você está prestes a criar uma tabela de controle de execução ou um worker dedicado a um provider, pare — você está criando uma pipeline paralela. | Typeform implementado com pipeline própria (`typeform.import_runs`, zero capabilities, zero jobs) em Março/2026. Corrigido no mesmo sprint. |
| R35 | **Import de provider externo exige filtro de organização antes de gravar no raw.** Todo sync que busca dados de uma API externa deve filtrar pelo `provider_account_id` da conta vinculada ao tenant antes de inserir no raw. Nunca importar "tudo que a conta vê" sem validar que o dado pertence ao escopo daquele tenant. Checklist obrigatório antes de gravar qualquer registro no raw: (1) `tenant_id` está correto e explícito? (2) `integration_account_id` está preenchido? (3) O dado pertence à organização/conta vinculada ao tenant (não apenas acessível pelo token)? | Typeform importou forms de todas as organizações acessíveis pelo personal token sem filtrar por organização, gerando 271 registros órfãos com contaminação entre tenants. Corrigido com limpeza em Março/2026. |
| R36 | **Campos 1:N nunca são denormalizados em `contact_state`.** `contact_state` é estritamente escalar (um valor por contato). Entidades com cardinalidade 1:N (oportunidades, tickets, forms futuras) são avaliadas via `EXISTS` subquery direta na tabela relacional com índices corretos. O worker planner (`opportunity_exists` resolver) e o SQL function (`eval_condition_v2`) devem gerar SQL equivalente para esses campos. Denormalizar 1:N em `contact_state` (ex: `has_open_opportunity boolean`) cria divergência semântica entre preview (SQL function faz subquery real) e refresh (worker lê snapshot stale), além de exigir pipeline de sync impossível de manter consistente. **Modo de execução: BULK_ONLY** — mudanças em oportunidades (criar, mover estágio, fechar) NÃO disparam reavaliação incremental do segmento; atualiza apenas no próximo ciclo de `evaluate_segment`. Se no futuro for necessário reatividade em tempo real, o pipeline de CRM deve enfileirar `evaluate_segment` jobs ao mudar status de oportunidades. **Campos implementados:** `has_opportunity`, `opportunity_status`, `opportunity_pipeline`, `opportunity_stage`, `opportunity_assigned_to`, `opportunity_lost_reason`, `opportunity_created_days`, `opportunity_closed_days`, `opportunity_won_days`, `opportunity_lost_days`. | `has_open_opportunity` e `open_opportunity_count` foram dropados de `contact_state` em Março/2026 após auditoria confirmar zero segmentos referenciando. Substituídos por 10 campos `opportunity_exists` no worker planner. Índice `idx_opportunities_party_tenant_status` criado para suportar as subqueries. |
| R37 | **Impact Analysis obrigatória antes de mudança em contrato compartilhado.** Antes de alterar formato, renomear ou deprecar qualquer conceito listado em `DEPENDENCY_MAP.md`: (1) listar todos os readers e writers; (2) confirmar que TODOS os readers suportam o novo formato; (3) dual-write por pelo menos 1 sprint antes de dropar formato antigo; (4) campo vazio usado para filtrar = fail-closed (zero resultados, nunca "todos"). **Nota (v7.21):** R37 protege contra migração mal feita, mas pressupõe que `segment_parties` reflita as regras. Drift entre `segment_parties` e avaliação fresh (D-SEG-5) burla a proteção: o worker lê membership stale e dispara para um conjunto que não casa com o segmento atual. Operação A em 30/04 reconciliou 22 segmentos divergentes; causa-raiz do drift segue em investigação. | Incidente Sprint 12 — campanha blast radius: migração `rules_json` → `rules_json_v2` quebrou email-campaign-worker que ainda lia v1 vazio, disparando para base inteira do tenant. |
| R38 | **Fail-closed para membership vazia.** Se `segment_parties` retorna 0 rows para um `segment_id` que tem regras definidas (`rules_json_v2 IS NOT NULL`), a operação dependente (envio, export, enrollment) DEVE recusar, nunca prosseguir sem filtro. Zero members + regras definidas = refresh pendente, não "enviar para todos". **Implementações:** (1) `CampaignSend.tsx` — `getSegmentRecipientsFailClosed()` tenta refresh e rejeita se ainda 0; (2) `generate-export/index.ts` — enfileira refresh e aborta export; (3) `checkSegmentEntries.ts` (worker handler) — guard com verificação de `rules_json_v2` + enqueue refresh; (4) `check-journey-entries/index.ts` (Edge Function legada) — guard equivalente como safety net. **Nota (v7.21):** R38 protege contra membership vazia mas não detecta membership *errada*. Drift cap (D-SEG-5) é o caso "membership populada mas não corresponde às regras". Auditoria recomendada (D-SEG-8 — job diário) até a causa-raiz ser fechada. | Incidente Sprint 12 — mesma origem de R37. |
| R39 | **Skip em nó de automação DEVE retornar `success: false` + `waitUntil`.** Quando um nó `send_email` detecta que a campanha não está pronta (status não-enviável), o handler deve retornar `{ success: false, waitUntil }` para que o enrollment fique em espera e reprocesse. Retornar `success: true` com skip faz o enrollment avançar para o próximo nó sem enviar o email, perdendo o envio silenciosamente. Status `automation` é um status válido para envio em contexto de journey — não deve ser rejeitado. **Adendo 29/10/2026 (Fase 0 em produção):** `waitUntil` sozinho virou loop sem limite no incidente de 22–28/10 (cerca de 605 mil execuções `campaign_not_ready` e a fila travada para todos os tenants).
- **Esperar não é retry.** Campanha não enviável é detectada pelo portão `journeys.evaluate_enrollment_blocker` antes de criar o job. A inscrição fica `active` com `wait_reason='campaign_not_ready'` e `wait_ref_id=campaign_id`, **sem prazo**, e é ignorada por `enqueue_ready_waits` e pelo sweeper.
- **Com o worker antigo**, a trigger em `journey_executions` estaciona a inscrição já na primeira execução: no pior caso, 1 execução por inscrição.
- **Liberação:** a trigger em `email.email_campaigns` dispara na transição para `ready`/`sent`/`automation` e libera em lotes de 200/min por jornada; o cron serve de rede de segurança.
- **Campanha excluída ou inexistente:** a inscrição vai para `error` (`campaign_deleted`/`campaign_missing`), nunca para espera infinita.
- **Nunca** avançar sem enviar e nunca criar jobs ilimitados.

Ver `JOURNEY_ISOLATION.md`. **Lição adicional: Docker layer cache.** Workers containerizados (Railway, ECS, etc.) podem reutilizar layer de build anterior mesmo após push de novo código. Sempre incluir `BUILD_VERSION` constante no entrypoint do worker para: (1) invalidar cache do `COPY src` layer a cada mudança, (2) confirmar nos logs qual versão está rodando. | Incidente Sprint 13 — commit `c3f172e` corrigiu send_email mas Railway serviu código antigo por cache Docker. Worker logava `campaign_not_ready/automation` com `success: true` (código antigo) em vez de enviar o email. 19 enrollments ficaram stuck. |

| R40 | **Condition node deve usar Rule Engine v2.** O nó `condition` em journeys deve reutilizar os mesmos campos, operadores e resolvers do segmentation engine (AST v2). Implementação duplicada (6 campos hardcoded vs 40+ do Rule Engine) viola DRY e causa divergência silenciosa. Evaluator: carregar `ContactState` do party + `evaluateRules(state, ast.root)`. | Gap identificado Sprint 14 — condition node tem 6 campos, Rule Engine v2 tem 40+ |
| R41 | **Journey version_id em enrollments.** Enrollment deve congelar `version_id` no momento da criação. Se a jornada é editada durante execução, enrollments existentes continuam com a versão original. Implementação: coluna `version_id` em `journey_enrollments` + snapshot `nodes_json`/`edges_json` em `journey_versions`. | Requisito Etapa 2 — versionamento de jornadas |
| R42 | **Decisão de condition node deve ser persistida.** O resultado da avaliação (campo, valor, operador, resultado booleano) deve ser gravado em `journey_executions.output_json` para auditoria. Se o contato re-entrar, a decisão anterior é consultável sem re-executar a condição. | Requisito Etapa 2 — auditabilidade |
| R43 | **Trigger context congelado no enrollment.** `context_json` deve capturar o estado do contato no momento do enrollment (valores de campos usados por conditions). Conditions avaliam contra esse snapshot, não contra o estado atual, para garantir determinismo na execução. Exceção: nós explicitamente marcados como "live evaluation" podem consultar estado atual. | Requisito Etapa 2 — determinismo |
| R44 | **Extensibilidade de resolvers via registry.** Novos campos para condition node devem ser adicionados via registro (field → resolver function), não via switch/case hardcoded. O resolver recebe `(partyId, tenantId, fieldConfig)` e retorna o valor. Mesmo padrão do `semanticFieldRegistry.ts` do analytics. | Requisito Etapa 2 — condition engine extensível |
| R45 | **Frequency guard em nós de comunicação.** Nós `send_email`, `webhook`, `notify` devem respeitar limite de frequência por contato configurável no journey settings. Default: max 1 email por contato por 24h na mesma jornada. Guard consultável em `journey_executions` (contagem de executions do mesmo node_type para o mesmo enrollment nas últimas N horas). | Requisito Etapa 2 — proteção contra spam |
| R46 | **Workers separados por SLA.** journeys-event-worker (< 5s, push-based) e journeys-worker (minutos, batch) devem permanecer separados. Condition engine pesado (queries em contact_state) não deve rodar no event-worker. Se condition evaluation for necessária no fanout, delegar para journeys-worker via processing_job. | Confirmado Sprint 14 — audit comprovou separação necessária |
| R47 | **Email engagement via subquery, não snapshot.** Campos `campaign_opened`/`campaign_clicked` usam resolver `email_engagement` com EXISTS em `email.email_events`. Engagement data é per-campaign — nunca materializar em `contact_state`. Condition node usa pre-enrichment (`__email_engagement` injetado pelo conditionEngine). Padrão replicável para futuros campos relacionais (ex: `opened_journey_email_X`). | Sprint 19 |
| R55 | **RLS de tabela consultada pela UI valida PARTICIPAÇÃO (`is_tenant_member`/`is_tenant_admin`), não apenas o claim `active_tenant_id`.** O claim não está presente em todos os JWTs; sem ele `app_auth.active_tenant_id()` devolve NULL e a policy zera silenciosamente o resultado — sem erro, sem log. Usar o claim só como atalho, sempre com o `OR` de participação. | 2026-08-14: usuários viam a aba Chat mas nenhuma conversa carregava; 2 de 9 membros da Escola do Fluxo tinham o claim correto. 23 tabelas internas seguem só com o claim (dívida aberta). |
| R54 | **Subconsulta de segmentação usa `EXISTS`/`NOT EXISTS`, nunca `IN`/`NOT IN`.** Com correlação de tenant/party, `NOT IN` não pode virar anti-join (o Postgres precisaria provar ausência de NULL) e degrada para SubPlan reexecutado por linha. Correlacionar por `party_id` **e** `tenant_id` é o que habilita semi/anti-join. Após backfill em massa, rodar `VACUUM ANALYZE` — sem isso o índice coberto perde o index-only scan. | Medido 2026-08-13 no filtro "recebeu a campanha e não abriu": 5,3 s (SubPlan, 8.975 loops) → 107 ms (Hash Anti Join). |
| R51 | **Campo oferecido na UI de segmentação DEVE ter tradutor correspondente em `analytics.eval_condition_v2`.** Sem tratamento explícito, o campo cai no template genérico `campo operador valor` e gera SQL inválido — que só falha em runtime, no preview e no refresh. Ao adicionar campo em `fieldConfig.ts`, adicionar o branch no plpgsql E um caso na varredura de regressão (gerar a cláusula de todos os segmentos ativos e executá-la). | Auditoria 2026-08-11: os 4 filtros de engajamento de email (`last_email_opened_at`, `last_email_clicked_at`, `campaign_opened`, `campaign_clicked`) existiam na UI desde sempre e **nunca** funcionaram — geravam `last_email_opened_at >= 7` (timestamptz vs int) e `campaign_opened = NULL` (coluna inexistente). 2 segmentos em produção com 0 membros. |
| R52 | **Filtro por período sobre engajamento usa o EVENTO (`email.email_events`), nunca o snapshot (`contact_state.last_email_*_at`).** O snapshot guarda só a última ocorrência: quem abriu em junho e de novo em agosto não apareceria em "abriu em junho". Os campos de engajamento são temporais (`is_temporal_field`) e herdam o intervalo do grupo. Escala: `party_id` desnormalizado em `email_events` + índices cobertos (`INCLUDE (party_id)`) + correlação obrigatória `ee.tenant_id = v_segment_unified.tenant_id` — sem o filtro de tenant a subconsulta varre os eventos de todos os tenants (custo 19× maior medido com 884k eventos). | Auditoria 2026-08-11 |
| R53 | **O avaliador em memória do motor vivo NUNCA pode remover contato de segmento que não sabe avaliar.** Regras sem `rules_json` v1 (ou com campos relacionais) vão para `diff.deferred` e são decididas por `analytics.contact_matches_segment` — uma consulta indexada de UMA linha, com `null` significando "não sei" (preserva a associação). Refresh completo por evento é proibido: a 1000 tenants seria inviável. | Auditoria 2026-08-11: o worker lia `rules_json` (v1) vazio nos segmentos do builder novo, concluía "não pertence" e esvaziava 71 dos 121 segmentos ativos, brigando com o refresh em lote que os repopulava. |
| R49 | **Estado "vivo" único por entidade deve ser garantido por índice único parcial, não por disciplina de código.** Agendamento de campanha: `email.campaign_sends` tem `unique (campaign_id) where status='scheduled' and is_test is not true`. Reagendar CANCELA o anterior (`email.cancel_campaign_scheduled_sends`) antes de inserir o novo — nunca cria um segundo em paralelo. | Incidente 2026-08-10 (Escola do Fluxo): reagendar criava um 2º `campaign_sends` sem cancelar o 1º; ambos dispararam e 8.854 pessoas receberam a mesma newsletter duas vezes. |
| R50 | **Toda operação de longa duração precisa de botão de parada acessível no lugar onde o usuário percebe o problema.** Se o motor já sabe abortar (worker confere status antes de cada lote), a UI e a RPC DEVEM expor isso — inclusive na tela de listagem, não só numa tela interna. Cancelamento de envio aceita `scheduled`, `pending` e `sending`. | Mesmo incidente: o botão só existia na tela de envio e só para `scheduled`; quando o usuário viu os e-mails saindo, não havia nada a clicar por 13 minutos. |
| R48 | **`analytics.segments.rules_json_v2.root.children` deve conter exclusivamente nós `kind: 'group'`.** Conditions soltas no nível root são deprecated (shape B legado). Adapter UI (`normalizeAstV2ForUI`) é safety net que normaliza shapes legacy em runtime no load — até Fase 2 (backfill SQL canonicalizando todos os segmentos) e Fase 4 (trigger BEFORE INSERT/UPDATE em `analytics.segments` validando o shape). Após Fase 4, R48 vira enforcement estrutural; antes disso, é convenção. | Hotfix v7.21 — 46/52 segmentos manuais ativos com shape B causaram builder vazio em produção; duas rotas de criação coexistiam, escrevendo formatos divergentes. |
| R50 | **pg_cron é APENAS housekeeping.** Permitido: DELETE/UPDATE de TTL em filas/logs/audit; DDL maintenance (criar partição, REINDEX, VACUUM); pulse simples sem lógica de domínio (`SELECT some_fn()` que apenas dispara cleanup banco-only). **Proibido:** loop sobre tenants (`FROM core.tenants WHERE is_active`) — viola §2.4 Isolamento por tenant; lógica de negócio embutida (thresholds dinâmicos, sharding heurístico, condições de domínio); INSERT/UPDATE em tabelas de domínio (não-fila/log); fanout que cresce com volume. Lógica com schedule tenant-aware ou condições de negócio = worker no Railway. Cron novo requer entrada na tabela §11 + classificação A; cron categoria B/C/D requer auditoria arquitetural antes de aprovar. | Auditoria v7.22 — `rfm-rolling-refresh` e `segment-eval-fallback` foram criados sem registro arquitetural, com loop bloqueante de tenants no banco e mascarando bugs do event-driven path. Reclassificados como categoria B; D-CRON-2..4 abertos. |
| R52 | **Observability de plataforma reside em camada própria desacoplada do mecanismo observado.** Telemetria (Camada A — `analytics.event_health_metrics`), view de saúde (Camada B — `analytics.v_event_driven_health`) e alerta (Camada C — handler em worker) são entidades separadas. View e handler leem APENAS a tabela de telemetria — NUNCA tabelas operacionais (`fallback_log`, `segment_eval_queue`, etc). Mudança no mecanismo observado (ex: substituir cron por worker em D-CRON-4) afeta APENAS quem POPULA a métrica, não quem consome. **Como identificar violação:** PR adicionando `JOIN analytics.fallback_log` ou similar dentro de `v_event_driven_health` ou do handler de alerta = violação. Caminho correto: novo emissor escreve em `event_health_metrics` via `analytics.emit_health_metric()`. | D-CRON-3 (v7.25) — primeira camada de observability arquitetural; precedente para futura observabilidade de outros paths (workers, integrações, journey throughput). |
| R51 | **Campos de domínio separado fora de `contact_state` e `v_segment_unified` são avaliados como optimistic (`true`) no worker in-memory; a filtragem definitiva acontece no SQL path via subquery em `eval_condition_v2`.** Extensão operacional de R36: campos 1:N (oportunidades, tickets, etc.) NÃO devem ser denormalizados em `contact_state`. No fast-path do worker (planner/evaluator de jornadas e segmentação), quando o campo não tiver resolver disponível na avaliação in-memory, retornar `true` (optimistic) e deixar o SQL path do `eval_condition_v2` aplicar o filtro real via subquery indexada. Nunca adicionar coluna em `contact_state` para "facilitar" — isso reintroduz o drift entre snapshot stale e regra fresh que motivou R36. **Como identificar violação:** PR adicionando `has_*` ou `*_count` em `contact_state` para resolver gap de avaliação no worker = violação. Caminho correto: criar/ajustar resolver no SQL path. | Sprint de Performance v7.23 — `has_open_opportunity` resolvia false em CRM conditions porque o worker in-memory não conhecia oportunidades; fix correto foi optimistic no worker + subquery no SQL path, não materialização. |
| R53 | **Trigger enrichment via COUNT in-transaction.** Triggers de jornada podem enriquecer `event_data` com campos derivados na mesma transação SQL (ex: `purchase_sequence_number`). O campo é instantâneo (zero pipeline delay) e determinístico para condition nodes. Usar `party_id` como chave de agregação (party-first). Best-effort sob concorrência (READ COMMITTED — dois inserts simultâneos do mesmo party podem ambos receber sequence=N). | Sprint 19 — purchase_sequence_number |
| R54 | **Cast de jsonb para UUID em RPC de ingestão deve usar `public.safe_cast_uuid()`.** Toda RPC que consome `NormalizedTransaction` ou payload externo e precisa extrair `uuid` de campo jsonb deve usar `public.safe_cast_uuid(p_text text) → uuid` (returns NULL se não casar com regex de UUID canônico). Cast direto `(jsonb->>'field')::uuid` aborta a linha inteira com `22P02` quando o provider envia ID não-UUID (numeric, slug, etc.) e o batch inteiro vai para retry/DLQ — mesmo que o fallback de resolução (mapping table) cobrisse o caso. **Como identificar violação:** PR adicionando `(v_tx->>'something_id')::uuid` em RPC `process_*_batch` = violação. Caminho correto: `public.safe_cast_uuid(v_tx->>'something_id')`. Mappers de provider também NÃO devem enviar IDs nativos do provider em campos contratuais UUID — preservar em `extra.{provider_id}`. | Sprint 5 — Sympla mapper enviava `event_id` numérico ("3416371") em `product_id`; 10 vendas da tenant `thais-zacharias` ficaram presas (3 retry + 2 dead-lettered) entre 14/04 e 10/05. Fix: helper `safe_cast_uuid` + remover `product_id: eventId` dos 3 mappers Sympla (sympla-sync edge, pull-sync, historical-sync). |
| R55 | **Toda chamada `supabase.rpc()` em Edge Function ou frontend que invoque função fora do schema `public` DEVE usar `.schema("<schema>").rpc(...)` explicitamente.** PostgREST resolve `.rpc("foo", ...)` no schema padrão (`public`); se a função existe apenas em outro schema (`integrations`, `inbox`, `whatsapp`, `analytics`, `email`, etc.) o erro retornado é `function public.foo does not exist`. Quando esse erro é capturado dentro de um try/catch genérico que apenas loga e continua (padrão "fire-and-forget"), o resultado é uma falha silenciosa em produção sem regressão de testes nem alerta. **Como identificar violação:** `grep -rn '\.rpc("[a-z_]\+"' supabase/functions/ src/` e cruzar cada nome contra `pg_proc/pg_namespace`; qualquer nome cuja função SÓ exista fora de `public` é violação. **Mecanismo defensivo:** sempre que um catch para o `createHistoricalImportJobs` (ou equivalente "background task após sucesso") loga `error` mas continua o fluxo principal, o handler deve no mínimo gravar a falha em `integrations.logs` para visibilidade — ou propagar até alertas. | Sprint v7.26 — Instituto Socrates conectou Eduzz mas historical não disparou; `eduzz-oauth-callback` chamava `bulk_enqueue_jobs` sem `.schema("integrations")`, o catch externo do OAuth engoliu o erro. Auditoria do codebase encontrou 13 call-sites Nylas (`upsert_message`, `batch_link_messages_to_crm` no schema `inbox`) com o mesmo padrão. Fix: adicionar `.schema("inbox")` em todos. |
| R56 | **Tenant check em wrapper `api.*` usa `PERFORM app_auth.assert_tenant_access(p_tenant_id)` — NUNCA `app_auth.active_tenant_id() IS DISTINCT FROM p_tenant_id`.** O claim `active_tenant_id` só é gravado por `core.switch_tenant`, que o frontend nunca chama: 14/22 usuários em produção não têm o claim (check estrito sempre nega) e claims stale apontam para tenant revogado (check estrito CONCEDERIA acesso indevido). `assert_tenant_access` valida membership em `core.tenant_users` OU system_admin. **Como identificar violação:** `SELECT ... FROM pg_proc WHERE prosrc ILIKE '%IS DISTINCT FROM p_tenant_id%'` — a migração `20260827150000` roda esse sweep e corrige/falha alto. | Auditoria 2026-08-27 (F-Labels do inbox): todos os wrappers novos nasceram com o padrão estrito e nenhum membro sem claim conseguia usar labels; hotfix `20260827140000` + padronização `20260827150000`. |
| R57 | **Schema de extensão com efeito colateral (ex.: `net`/pg_net) NUNCA fica com USAGE/EXECUTE para `authenticated`/`anon`.** pg_net faz o BANCO emitir HTTP arbitrário — exposto via PostgREST é SSRF pronto (metadata da cloud, endpoints internos, port scan a partir do nosso IP). Revogar USAGE + EXECUTE e travar `ALTER DEFAULT PRIVILEGES` no schema; consumidores legítimos (triggers, cron, SECURITY DEFINER, service_role) não são afetados. **Como identificar violação:** `has_schema_privilege('authenticated', 'net', 'USAGE')` = true. | Auditoria 2026-08-27: `net.http_get/http_post` executáveis por qualquer usuário logado desde a instalação da extensão; lockdown em `20260827160000`. |
| R58 | **Opção de UI que promete efeito precisa de produtor E consumidor ligados — e de uma fonte única de seleção.** Uma flag gravada sem quem a leia é bug silencioso: o usuário acredita que a ação aconteceu. Para backfill de jornada: produtor = trigger de ativação, consumidor = loop do journeys-event-worker, seleção = `journeys.backfill_batch_candidates()` (o preview do diálogo e o dry-run usam a MESMA função). Histórico de compra vem de `commerce.transactions status='paid'`, nunca de `journey_events` (eventos sem `product_id` e de transações hoje refunded). **Como identificar violação:** tabela-fila com 0 linhas e handler sem import (`grep -rn "processXxx" workers/*/src/index.ts`). | Incidente 2026-10-16 (Instituto Socrates): jornada ativada com "Incluir toda a base existente" inscreveu 16 compradores novos e 0 dos 414 anteriores; `backfill_jobs` vazia no banco inteiro, `processBackfillJob` nunca chamado e descartaria eventos anteriores à ativação. 11 jornadas de compra ativas tinham `enroll_existing=true` sem efeito. |
| R59 | **Toda escrita em `email.unsubscribes`, `email.suppression_list` e `email.unsubscribe_feedback` passa por `email.apply_consent_change` (ou `apply_consent_changes` em lote).** A RPC é transacional, valida grupo/tenant, faz upsert idempotente, propaga efeitos (`campaign_recipients.unsubscribed_at`, `email_events`, saída de jornada no global) e grava SEMPRE uma linha em `email.consent_audit_log` (append-only). Se a RPC lançar, o caller chama `email.log_consent_error`. Edge functions usam `supabase/functions/_shared/consent.ts`. Exceções transitórias (a migrar): contador de soft bounce do webhook e `email.expire_soft_bounce_suppressions` (housekeeping). **Adendo Onda 3 (2026-10-22):** backfills de consentimento também passam pela RPC (`channel=system`, `request_id` próprio, ex. D3 `backfill_spam_report_D3_20261022`); falha da RPC é reportada ao Sentry por `logConsentError` (`feature:email-consent`, email mascarado). | `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` (P3/P5/P13/P14): 4 writers diferentes, cada um sobrescrevendo a mesma linha sem histórico; webhook nunca gravou descadastro (`onConflict` incompatível); feedback perdido por CHECK. |
| R60 | **`email.campaign_sends`: marketing exige grupo; transacional não tem grupo e exige motivo; unsub de grupo inativo vale como global.** Trigger `trg_campaign_sends_validate_message_type` rejeita (SQLSTATE `23514`) marketing sem `group_id` (`marketing_requires_group`) ou com grupo de outro tenant (`invalid_group`); em transacional força `group_id=NULL` e exige `transactional_reason` não vazio quando `is_test=false` — vindo do send ou, se ausente, de `email_campaigns.transactional_reason` (`transactional_requires_reason`). Troca de tipo da campanha só por `api.set_campaign_message_type` (owner/admin, trilha em `email.campaign_type_changes`). Workers (`workers/email-campaign/src/unsubscribeFilter.ts` e `processJourneyNode.ts`) tratam unsub ativo de grupo com `is_active IS NOT TRUE` como global para marketing; desativar grupo só por `api.deactivate_unsubscribe_group` (FK de `unsubscribes` é RESTRICT). **Como identificar violação:** writer novo de `campaign_sends` que não copia `group_id`/motivo da campanha; filtro de descadastro que consulta só `group_id IS NULL` e `group_id = <grupo do envio>`. **Adendo Onda 3 (2026-10-22):** como o unsub de grupo inativo vale como global, um *resubscribe global* com `channel=preference_center` (inclusive o `confirm_resubscribe` do double opt-in) também desativa os unsubs ativos de grupos inativos do mesmo email/tenant (uma linha de auditoria, ids em `request_payload.inactive_groups_cleared`); unsubs de grupos ativos não mudam e outros canais não limpam. **Desde 2026-10-29** o resubscribe global com `channel=admin_manual` (owner/admin pela UI do tenant, `api.set_email_consent`) também limpa — sem isso o unsub de grupo inativo ficava impossível de desfazer pela UI, porque resubscribe por grupo inativo é rejeitado (`invalid_group`); `one_click`, `footer_get`, `webhook_sendgrid`, `import`, `api` e `system` continuam sem limpar. A página trata esse bloqueio como `global_unsubscribed:true` e o gate de double opt-in considera essas linhas como a global. **Adendo 2026-10-26 (duplicação):** `email.duplicate_campaign` (via `api.duplicate_campaign`) copia `transactional_reason`; cópia de transacional com motivo nasce transacional e sem grupo; cópia de transacional **sem** motivo nasce **marketing** com o grupo da origem (se ativo) ou o grupo padrão ativo do tenant (NULL se não houver — o envio fica bloqueado por `marketing_requires_group`), com linha em `email.campaign_type_changes`. Antes a cópia herdava um bloqueio (`transactional_requires_reason`, 12 tentativas falhas de uma usuária em 2026-10-26). **Como identificar violação:** função que cria campanha a partir de outra sem copiar `message_type`/`group_id`/`transactional_reason` juntos. | `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` P4/P9/P12: marketing classificado como transacional furava todo descadastro (742 envios / 38.740 destinatários em 90d no tenant 0…001); trava "marketing sem grupo" só no navegador; grupo desativado deixava de proteger quem saiu dele e excluir grupo apagava o histórico (CASCADE). |
| R61 | **Criação de envio de campanha é atômica no servidor via `api.create_campaign_send(p jsonb)`; o front nunca insere `email.campaign_recipients` (nem `campaign_sends` de envio direto).** A RPC (SECURITY DEFINER, `SET statement_timeout='120s'` aplicado pelo PostgREST, membro não-viewer) faz numa transação: cancela o agendamento anterior se pedido, insere `campaign_sends` com tipo/grupo/motivo da campanha (trigger R60 valida), materializa os destinatários com `INSERT … SELECT` a partir da mesma fonte da UI (`segment_parties` R38, `vw_parties_unified`, lista explícita ou envio anterior), atualiza `total_recipients`, enfileira `prepare_email_batch` (`email.enqueue_campaign_send`) e, se agendado, marca a campanha `scheduled`. Qualquer falha devolve `{ok:false, error_code}` sem deixar linha. Envios órfãos antigos: `api.cancel_orphan_campaign_sends(p_tenant_id)` (pending, sem `scheduled_at`, > 30 min, 0 destinatários, sem job) — não é cron (R50: UPDATE em tabela de domínio). **Como identificar violação:** `.from("campaign_recipients").insert(` ou `.from("campaign_sends").insert(` em `src/`; writer que cria o envio e os destinatários em requests separados. | Incidente 2026-10-26 (`docs/EMAIL_UNSUBSCRIBE_AUDIT.md`): o INSERT de todos os destinatários (944 a 7.500 linhas) num request sob o `statement_timeout` de 8 s do `authenticator` expirava sob carga (57014; 123 timeouts no banco em 30 min em 20/10) e deixava `campaign_sends` `pending` sem destinatários e sem job (ex. `3c175050…`). |
| R62 | **`ON CONFLICT` em tabela de vínculo de contato usa o alvo party-first: `crm.customer_tags` e `crm.lead_tags` → `(party_id, tag_id)`; `crm.customer_badges` → `(party_id, badge_id)`.** Desde a migração party-first não existe índice único em `(customer_id, tag_id)` nem coluna `customer_id` em `customer_badges`. Em `customer_tags`, `customer_id` é opcional: party cliente sem linha em `crm.customers` (`customer_id` nulo na `vw_parties_unified`) recebe a tag só com `party_id`; com `party_id` nulo, o trigger `trg_z_party_id_from_customer` resolve pelo `customer_id`. **Como identificar violação:** `ON CONFLICT (customer_id, tag_id)`, `onConflict: "customer_id,tag_id"`, `ON CONFLICT (customer_id, badge_id)`; condição `party.customer_id && party.is_customer` antes de gravar tag. | Incidente 2026-10-26: o nó `add_tag` do worker `journeys` falhava para todo contato cliente desde abr/2026 ("there is no unique or exclusion constraint matching the ON CONFLICT specification"; 19 falhas em 2 tenants, 23/04–27/10) e pulava em silêncio cliente sem `customer_id` (`tag_skipped`). Corrigido em `workers/journeys/src/handlers/processJourneyNode.ts` (`executeAddTagNode`); tags reaplicadas na Escola do Fluxo (8 contatos, 2026-10-28) e no AEF (5 contatos, 2026-10-29). Até o deploy do worker `journeys`, novos clientes continuam falhando ou sendo pulados. **Pendente:** `executeAddBadgeNode` (insere coluna `customer_id` inexistente) e a Edge Function legada `process-journey` (`onConflict: "customer_id,…"`). |
| R63 | **Função SECURITY DEFINER em schema exposto ao PostgREST precisa de duas coisas: `REVOKE ALL … FROM PUBLIC, anon, authenticated` com GRANT explícito a quem chama, e checagem de acesso no corpo.** O Postgres dá EXECUTE a PUBLIC em toda função nova, e default privilege por schema não remove esse grant global. Os schemas expostos (`pgrst.db_schemas`) dão USAGE a anon e authenticated. Sem o REVOKE, a função roda como `postgres`, ignora RLS e fica chamável pela anon key (pública, no bundle) via `POST /rest/v1/rpc/<fn>` com o header `Content-Profile`. Como o signup é público, "só authenticated" não protege dados de outro tenant.<br>**Regras por tipo de chamador:**<br>• Worker, edge, cron ou SQL interno: só `postgres` e `service_role`.<br>• Navegador: `authenticated` + `PERFORM app_auth.assert_tenant_access(...)`, seguindo a R56. O tenant vem do parâmetro ou é derivado da linha.<br>• Fluxo público por design (formulário, checkout, landing, convite, push): validação por token ou slug, e o mínimo de parâmetros aceitos.<br>• Função chamada por trigger SECURITY INVOKER ou por condição `WHEN` disparada pelo navegador precisa de EXECUTE para `authenticated` (ex.: `journeys.safe_timestamptz`). Prefira condição WHEN inline, sem chamar função.<br>**Como identificar violação:** migration com `CREATE FUNCTION … SECURITY DEFINER` sem o REVOKE correspondente, ou função que lê ou grava por `p_tenant_id` sem checagem. Toda migration de função termina com um bloco de verificação que aborta se `has_function_privilege('anon', …)` for verdadeiro onde não deveria. | Auditoria de 29/10/2026:<br>• `public.read_secret` devolvia qualquer segredo do Vault (inclusive `service_role_key` e `ANTHROPIC_API_KEY`) para a anon key; `public.eduzz_get_integration` devolvia tokens OAuth.<br>• As funções de fila (`journeys.claim_next_job`, `enqueue_job`, `release_stale_claims`, `cleanup_old_jobs`; o mesmo em analytics, email, integrations, inbox, eduzz, doare) podiam ser chamadas por qualquer um.<br>• Cerca de 167 funções liam ou gravavam PII sem checar tenant.<br>Corrigido pelas migrations `20261029184147`, `20261029190718` e `20261029191215`, mais a Onda 2 (`20261029205501` e `20261029205748`). Nenhum sinal de abuso nos edge_logs de 31/07 a 29/10. Detalhes em `SECURITY.md` §12.6. |

---

# PARTE D — DÍVIDAS TÉCNICAS & ROADMAP

## 22. Dívidas Arquiteturais

| # | Item | Prioridade |
|---|---|---|
| D5 | ~~Migrar pipeline SendGrid para versão unificada~~ | ✅ Resolvido (v7.11) — `webhooks-sendgrid` O(1) + `integrations-worker` v81 handlers + URL migrada para `core.kreatorshub.com.br` em 5 subusers + `domain-delegation` atualizada |
| D7 | Producer `responded_form_ids` em tempo real (hoje só backfill) | Pendente |
| D8 | Producer `import_ids` em tempo real (hoje só backfill) | Pendente |
| D9 | `journey_ids` em `contact_state` — producer não existe | Pendente |
| D10 | `has_tag` GIN Phase 2 — bloqueado por product tags transitivas não estarem em `tag_ids` | Pendente |
| D11 | ~~`refreshFeatures` cleanup de parties com customer role revogada~~ | ✅ Sem ação necessária (v7.11) — `refreshFeatures` recalcula do zero a cada execução, zero órfãos encontrados em auditoria |
| D12 | Normalização de telefone (`phone_normalized` E.164) — `crm.party_person`, `sympla.participants`, todos os providers. Função utilitária compartilhada. Backfill + UI de indicador de cobertura. Campanhas WhatsApp filtram por `phone_normalized IS NOT NULL` | Pendente |
| D13 | `DROP TABLE eduzz.integrations` — tabela legada. Toda leitura/escrita já migrada para `integrations.accounts`. RPCs atualizadas. Dropar após validação. | Pendente |
| D14 | Auditoria preventiva de TTL em todas as filas do sistema — verificar se todas as filas têm: (1) cron de purge para rows terminais, (2) índice parcial cobrindo status ativo, (3) NOT EXISTS filtrando apenas status ativos. Ver R20. | Pendente |
| D18 | ~~Journey fanout não-atômico~~ | ✅ Resolvido (v7.13) — `sql.begin()` em 5 entry handlers + send_email node (D18b). Sweeper `sweep-orphan-enrollments` (*/10 * * * *) como safety net. 3 órfãos históricos recuperados. |
| D19 | **Observabilidade backend** — Sentry cobre apenas frontend React. Workers sem aggregação, dashboards ou alertas. **Fix:** (1) Sentry nos Railway workers; (2) dashboard de filas (depth, drain rate, p95 latency); (3) alertas: dead jobs > 0, queue depth crescendo, worker sem heartbeat >5min. Ref: Audit F10. | **P2** |
| D20 | ~~Journeys batch claim~~ | ✅ Resolvido (Sprint 17) — `claim_next_jobs(worker_id, batch_size, job_types, tenant_cap)` com ROW_NUMBER fairness. journeys-worker 10/poll, email-campaign 5/poll. Singular preservado. |
| D21 | **Segment-eval tenant scan** — Worker faz O(tenants) queries por ciclo mesmo sem trabalho. **Fix:** criar `claim_next_segment_jobs_global(worker_id, batch_size, shard_index, total_shards)`. Ref: Audit F2. | **P4** |
| D22 | **claim_jobs tenant fairness** — ordena por `priority, created_at` sem cap por tenant. **Fix futuro:** round-robin CTE com `ROW_NUMBER() OVER (PARTITION BY tenant_id)`. Ref: Audit F3. | Defer |
| D23 | **Idempotency constraint no enqueue** — `enqueue_webhook_job` usa SELECT-then-INSERT (TOCTOU). Risco baixo. **Hardening:** partial unique index em `integrations.jobs(webhook_event_id)`. Ref: Audit F5. | Defer |
| D24 | DROP TABLE segment_customers — CANCELADO (2026-03-20). Tabela ativa para clusters/lookalikes. | ❌ Cancelado |
| D25 | `refresh_tenant_distribution` + tela de monitoramento | Pendente |
| D26 | trg_record_email_engagement — ✅ Trigger + 4 colunas em contact_state operacionais. | ✅ Done |
| D27 | UI: novos blocos de condição no segment builder (product picker, time window, CRM condition) | Pendente |
| D28 | D6 Fase 3 — `DROP COLUMN customer_id`. **Fase 1 ✅ (2026-04-08, v7.24):** 6 colunas zero-data dropadas — `analytics.customer_cluster_assignments`, `analytics.customer_cluster_subgroup_assignments`, `analytics.customer_lookalike_scores`, `core.eduzz_webhook_events`, `eduzz.webhook_events`, `crm.customer_badges`. **Restante (11 entradas):** 8 base tables — `commerce.transactions`, `commerce.archived_transactions`, `journeys.journey_enrollments`, `crm.opportunities`, `crm.customer_tags`, `analytics.lifecycle_events`, `analytics.segment_customers`, `eduzz.buyers`; 3 views — `analytics.v_segment_base`, `analytics.v_segment_leads_base`, `crm.vw_parties_unified`. 5 FKs ativas → `crm.customers` (bridge table, 16.4K rows 100% party_id). 35 funções com "customer" no nome — auditoria necessária antes da Fase 3. **Próximas fases:** Fase 2 (tabelas com party_id 100%: transactions, enrollments, opportunities), Fase 3 (audit das 35 funções + drop FKs → `crm.customers`), Fase 4 (DROP `crm.customers` + rebuild views). | Fase 1 ✅ · Fase 2–4 Pendente |
| D29 | Performance loading timeout — ✅ Resolvido (v7.23, Sprint de Performance). `crm.get_dashboard_data` 534ms→17ms (3 lead CTEs colapsados em 1 `COUNT FILTER`); `analytics.get_analytics_data` 4 scans→1 (CTE `base_txn`); `crm.list_parties` reescrito de subqueries correlacionadas para CTEs batch. Novas RPCs: `crm.get_parties_stats(p_tenant_id)` (stats globais separados, staleTime 5min) e `analytics.get_transactions_chart_data(p_tenant_id, p_granularity, p_start_date, p_end_date)` (agregação por período: 26K rows→37 buckets, 22ms). `segment_eval_queue` purgada de 533K erros. Polling -99.8%. Frontend: React Query Persist (IndexedDB, 24h, limpa no logout), connection heartbeat (4min, pausa em tab inativa), prefetch durante auth (dashboard, tags, features), loading progressivo (Campaigns, Segmentation, Forms, Analytics), AccessGate bypass em warm path, dynamic imports −924K bundle, vendor chunks (`@xyflow`, `@tiptap`, `zod`), `useTenantTags` (6 query keys→1), `usePartiesStats` (cache 5min), polling 1s→30s, fix `has_open_opportunity` optimistic no worker + SQL subquery (R51). Rollbacks em `docs/rollback/`. Sentry JAVASCRIPT-REACT-22, 21, 1S fechados. | ✅ Done |
| D30 | Decompor blocos contact_info e address em traits individuais — hoje kind: 'skip', sem dados reais ainda | Pendente |
| D31 | Integração Typeform — ✅ Parcial. Schema `typeform.*` implementado (6 tabelas), sync via pull-sync e historical-sync. Pendente: analytics de choice blocks, webhook receiver para ingestão contínua. | Parcial |
| D32 | evaluation_mode em segments — ✅ Coluna populada automaticamente. 70/70 = event_driven. | ✅ Done |
| D33 | Segment builder UI → AST v2 — ✅ Frontend escreve exclusivamente rules_json_v2 desde Sprint 12. | ✅ Done |
| D34 | Scoping unique index `processing_jobs_coalesced_pending_uidx` — ✅ Resolvido (2026-03-30). Index agora exclui `project_traits` e `sync_field_definitions` do WHERE. Múltiplos pending per-entity permitidos. | ✅ Done |
| D35 | Backfill de traits de formulários — ✅ Resolvido (2026-03-30). Function SQL temporária `backfill_form_traits` projetou 8.816 traits (text: 8.464, number: 402, date: 368, boolean: 3) para 4 tenants. `segment_eval_queue` drenou em minutos. Gap restante: ~163K answers sem `field_definitions` (forms cujo `syncFieldDefinitions` nunca rodou — dívida separada, não D35). | ✅ Done |
| D36 | **RLS não reconhece system_admin** — policies de RLS usam `tenant_id IN (SELECT tenant_id FROM core.tenant_users WHERE user_id = auth.uid())`, mas system admins podem não ter registro em `tenant_users` para todos os tenants. O RPC `get_user_tenant_memberships` faz LEFT JOIN e retorna todos os tenants, mas o RLS bloqueia queries. **Fix:** criar helper `core.user_visible_tenant_ids(uid)` que retorna todos os tenants para system admins (via `core.system_admins`) e apenas memberships para users normais. Substituir subquery de RLS por chamada a essa função. Workaround atual: inserir system admins em `tenant_users` de cada tenant manualmente. | **P2** |
| D37 | Cleanup dead code segmentação v1 — ✅ Resolvido (2026-03-30). Removidos: `runEquivalenceGate.ts`, `equivalence.test.ts`, fallback legacy em `segmentEvaluator.ts` (substituído por throw explícito). | ✅ Done |
| D38 | **Alerta de anomalia pós-envio.** pg_cron job diário que detecta campanhas com `recipient_source = 'segment'` onde `recipient_count_at_send > segment_parties_snapshot * 1.5` OU `recipient_count_at_send > 80%` da base do tenant. INSERT em tabela de alertas. Depende das colunas de rastreabilidade adicionadas no Sprint 12 (Guard 1). | **P2** |
| D39 | **Backend validation RPC para envio de campanha.** RPC `email.validate_campaign_send(p_tenant_id, p_recipient_count, p_segment_ids, p_recipient_source)` que valida server-side: (1) se source='segment', recipient_count <= SUM(segment_parties) * 1.1; (2) se source='all', recipient_count <= total_contacts do tenant; (3) retorna ok/reject com reason. Defense-in-depth — proteção redundante independente do frontend. | **P2** |
| D40 | ~~party_id null em enrollments de formulário~~ | ✅ Resolvido (v7.13) — `create_opportunity` node: fallback lookup + last resort `crm.upsert_party_person` + graceful skip. `checkFormEntries` já tinha 3-layer resolve (v7.11). 10 jobs failed corrigidos, 0 órfãos restantes. |
| D41 | Queue audit — filas sem TTL/purge — ✅ Totalmente resolvido. Todas as filas com purge: `integrations.jobs` ✅, `integrations.webhook_events` ✅, `journeys.processing_jobs` ✅, `segment_eval_queue` TTL 24h→4h ✅ (2026-03-30). | ✅ Done |
| D42 | **DROP tabelas Eduzz legadas** — `eduzz.integrations` e `core.eduzz_integrations`. Ambas substituídas por `integrations.accounts`. Bloqueado por Sprint Security — credenciais em texto plano precisam ser migradas antes do DROP. | Bloqueado |
| B-17 | Testes de integração dos workers no tenant SaudeNow (`fe793fcd-7564-4d7c-b628-12a25e6d6656`) — criar jobs sintéticos para historical-sync, email-sync, ingestion e validar fluxo completo claim→running→success→analytics | Pendente |
| B-18 | ~~Mapper Eduzz — `installments`, `fee_value`, `net_gain`~~ ✅ Handler `eduzz:enrich_invoice` atualizado (26/03): popula campos financeiros a partir de `eduzz.invoices`. Backfill aplicado: 22K transações. Cobertura 99.9%. | **Done** |
| D43 | **`segment_entered` events não gerados.** `refresh_segment_parties_unified` faz UPSERT/DELETE direto sem chamar `apply_segment_membership_diff` — eventos `segment_entered`/`segment_exited` não são criados em `journeys.journey_events`. Os 9 eventos segment_entered existentes são de antes de 13/Mar. Impacto: `event_entry` com `trigger_event_type='segment_entered'` não funciona. Fix: gerar eventos inline no refresh ou restaurar chamada a `apply_segment_membership_diff`. | **P3** — nenhuma jornada usa esse trigger hoje |
| D44 | **Migrar template engine para SendGrid dynamic templates.** Atualmente o worker renderiza HTML com `substituteVariables()` (server-side). Migrar para `template_id` + `dynamic_template_data` habilitaria batch real (1000 emails/request via personalizations). Bloqueado por: cada recipient tem HTML diferente + `sendgrid_message_id` é 1 por request. | **P4** — quando precisar >50K emails/envio |
| D-SEG-2-BACKFILL | Canonicalizar todos `rules_json_v2` para shape A no banco. Após 48h estável de v7.21 em produção, rodar migração SQL que reescreve `root.children` envolvendo conditions soltas em grupos. Adapter `normalizeAstV2ForUI` vira no-op defensivo. | **P2** |
| D-SEG-3-CONTROLLED-COMPONENT | `SegmentBuilderV2` deve virar controlled — derivar `ast` interno do prop `initialAST` via `useEffect` de sync, ou eliminar o estado interno e fazer setter sobir todo via `onChange`. Hoje o gate na page (`!initialized → spinner`) elimina o sintoma sem reescrever o componente. | **P3** |
| D-SEG-4-INTEGRATION-TESTS | Aumentar cobertura de testes de integração da page de segmentos. Hoje cobre regression do state-lock (3 cenários). Faltam: salvar/atualizar segmento, locked condition que sobrevive round-trip e salva sem `__ui_locked`, navegação entre segmentos sem refresh da page. | **P3** |
| D-SEG-5-MEMBERSHIP-DRIFT | Causa-raiz do drift entre `segment_parties` e avaliação fresh não identificada. 22/52 segmentos ativos divergentes em 30/04. Operação A reconciliou (sintoma), causa segue aberta. Investigação: scheduler de refresh, trigger event-driven, regra invertida em path antigo. | **P1** |
| D-SEG-6-LAST-CALC-LIES | `last_calculated_at` é atualizado em paths que não fazem refresh real. Coluna não é confiável para detectar staleness. Auditar todos os writers da coluna; ou (1) só atualizar quando `refresh_segment_parties_unified` rodar com sucesso, ou (2) renomear para deixar semântica clara. | **P2** |
| D-SEG-7-EVENT-DRIVEN-AUDIT | Rota event-driven populou complemento semântico no segmento "Segmentação Amazônia" (3 contatos no card vs 4.167 fresh). Suspeita de regra invertida em path antigo (`apply_segment_membership_diff` ou trigger relacionado). Auditar inputs/outputs do event-driven path antes de Fase 4. | **P2** |
| D-SEG-8-DAILY-AUDIT-JOB | Job diário (pg_cron) que para cada segmento ativo: avalia regras fresh, compara com `segment_parties`, registra divergência em tabela de alertas e auto-refresh quando |drift| > N. Bloqueado por D-SEG-5 (sem causa-raiz, auto-refresh corre risco de mascarar problema). | **P2** |
| D-SEG-9-LEGACY-TRIGGER | `trg_refresh_segment_parties_on_change` em `analytics.segments` chama `refresh_segment_parties` (legacy v1) em vez de `_unified` (canonical v2). Possível parte da causa-raiz do drift (D-SEG-5). Auditar o trigger e migrar para a função unificada. | **P2** |
| D-RFM-LABELS | Lista canônica dos 11 labels `rfm_segment` está hardcoded em `workers/analytics/src/handlers/refreshRfm.ts:449-476` e duplicada em `src/components/segments/fieldConfig.ts FIELD_OPTIONS.rfm_segment`. Acentuação inconsistente em produção (D-DATA-1). Extrair enum compartilhado e re-exportar de ambos os lados. | **P3** |
| D-PRODUCT-FIRST-PURCHASE-UI | Campo `product_first_purchase` (temporal scoped por `product_id`) é válido no backend (`semanticFieldRegistry`) e usado por 1 segmento ativo, mas exige UI nova: scope picker (produto) + temporal window (`within_last N dias`). Atualmente preservado em modo locked após v7.21. Construir UI dedicada para destravar edição. | **P3** |
| D-OPP-FIELDS-UI | Campos `opportunity_pipeline`/`opportunity_stage`/`opportunity_assigned_to` precisam de pickers UUID com nome humanizado. Hooks (`usePipelines` etc.) já existem em `useSegmentOptions.ts`, mas não estão plugados no novo builder. Não ocorre em segmentos ativos hoje; se ocorrer ficará locked. | **P3** |
| D-RECENCY-DAYS-UNIT | `recency_days` foi registrado como `number` no fieldConfig, mas conceitualmente representa dias. Adicionar suffix "dias" no input (suporte ao campo `suffix` que já existe no `SemanticFieldMeta` do worker). | **P4** |
| D-CONVERSION-TEST-FIX | 4 testes em `src/lib/segmentation/conversion.test.ts` quebrados desde commit `8e38c4d` (anterior ao hotfix v7.21). Testam `builderToAstV2`/`astV2ToBuilder` (paths legacy não usados pelo builder novo). Avaliar se ainda têm valor antes da Fase 3 (eliminação do builder antigo). | **P4** |
| D-DATA-1-RFM-ACCENT | Acentuação inconsistente em `analytics.contact_state.rfm_segment` em produção: `Clientes Fiéis` (com agudo) coexiste com `Potenciais Fieis` (sem agudo). Outros labels OK. UI replica o estado do banco para preservar igualdade de string no SQL. Resolver junto com D-RFM-LABELS quando enum compartilhado for criado. | **P4** |
| D-CRON-1 | **Documentar todos os pg_cron jobs em `ARCHITECTURE.md`.** Mandatório (independente de categoria A/B/C/D). Tabela em §11 com jobname, schedule, command resumido, função SQL invocada, categoria, owner. Cron novo daqui pra frente requer entrada nessa tabela antes de ser criado. | ✅ Done (v7.22) |
| D-CRON-2 | **`rfm-rolling-refresh` migrar para worker no Railway.** Cron some, lógica de sharding (`MOD(HASHTEXT, 24)`) + threshold (20h staleness) vai pro código TS no `journeys-worker` ou novo `analytics-scheduler`. Risco: timing/SLA de RFM crítico — fazer com testes A/B em staging. Bloqueado por: revisar threshold 20h e estratégia de sharding em código antes de migrar. | **P3** |
| D-CRON-3 | **`segment-eval-fallback` instrumentar.** ✅ **Resolvido (v7.25).** Implementação muito além do "alarme em fallback_log" original: 3-camada Platform Health Observability decoupled (Camada A `event_health_metrics` particionada mensal + RPC `emit_health_metric` + 3 crons categoria A; Camada B view `v_event_driven_health` com tier classification per-tenant baseado em rolling 7d P50/P95/P99; Camada C handler `runEventDrivenHealthCheck` em `journeys-worker` com dedup, recovery silencioso, integração Sentry via `workers/shared/src/sentry.ts` reutilizável). 19 testes unit/integration. R52 estabelece o decoupling como regra para futura observability arquitetural. Pré-requisito de produção: criar Sentry project `workers-railway` e setar `SENTRY_DSN_WORKERS`. Auto-noop sem DSN. | ✅ Done |
| D-CRON-4 | **`segment-eval-fallback` migrar para worker.** Após baseline de D-CRON-3 estabelecido, mover loop de tenants para Railway. Cron vira pulse simples ou desaparece se métrica de hits zerar. Bloqueado por: D-CRON-3 + D-SEG-7 (event-driven audit) fechado. | **P3** |
| D-SEG-12-LOOKALIKE-DIRECT-WRITE | **`src/lib/analyticsStorage.ts:527 createSegmentFromLookalikes`** faz BULK INSERT direto no `analytics.segment_parties` da UI com `source: "customer"` hardcoded — bypassa todas as funções canonical. Para segmentos lookalike (todos os membros são customers por definição) é semanticamente correto, mas é mais um produtor fora do path canonical. Migrar para função SQL canonical (ex: `analytics.create_lookalike_segment_from_scores`) que reuse `apply_segment_membership_diff` para determinar `source` via `party_type` e mantenha audit trail. | **P3** |
| D-OPP-SOURCE-1-BACKFILL | ✅ **Resolvido (2026-05-07).** Migration `20260507160000_opportunity_source_journey_link` adicionou coluna `source_journey_id` (FK `journeys.journeys` ON DELETE SET NULL) e fez backfill lendo `metadata->>'created_by_journey'` (gravado historicamente pelo handler `executeCreateOpportunityNode` em `workers/journeys`). Resultado: 633/750 oportunidades reclassificadas como `source_origin='automation'` + journey vinculada, 117 mantidas como `'manual'`. Webhooks/forms não criam opportunity diretamente hoje — não houve nada a backfillar nesses paths. | ✅ Done |
| D-OPP-SOURCE-2-AUTO-TAG | ✅ **Parcial (2026-05-07).** Único writer não-UI hoje (handler `executeCreateOpportunityNode` em `workers/journeys/src/handlers/processJourneyNode.ts`) instrumentado para gravar `source_origin='automation'` + `source_journey_id` direto no INSERT. Webhooks (Eduzz/Asaas) e form submissions **não criam opportunity diretamente** hoje — só atualizam estado de transações ou criam leads. Quando esses paths forem adicionados, este item reabre como **P2** com prefix `D-OPP-SOURCE-2-FORM` / `D-OPP-SOURCE-2-WEBHOOK`. | ✅ Done (parcial) |
| D-OPP-SOURCE-3-METADATA-PROVENANCE | ✅ **Resolvido (2026-05-07).** Provenance já existia em `metadata.created_by_journey` (UUID) + `metadata.enrollment_id` + `metadata.journey_name` para 100% das 633 opportunities criadas por journey — permitiu backfill exato em D-OPP-SOURCE-1. Padronização para futuros writers (form/webhook) fica embutida no item D-OPP-SOURCE-2 quando esses paths forem adicionados. | ✅ Done |
| D-OPP-SOURCE-4-REPORT-UI | **Relatório de oportunidades por origem.** Implementar tela `/admin/opportunities/insights` ou aba no dashboard atual: ganhas/perdidas agrupadas por (a) `source_id` (origem cadastrada), (b) `source_origin` (canal: manual/automation/etc.), (c) `source_journey_id` (qual automação especificamente). Cross-tab com `lost_reason_id` quando `status='lost'` para análise de funil. | **P3** |
| D-JRN-BF-1 | **Segmento ignora "Apenas novos entrantes".** `check_segment_entries` (ativação + scheduler) inscreve todos os membros atuais do segmento qualquer que seja o modo escolhido. "Incluir base" funciona; "apenas novos" não. Corrigir exige snapshot de membros na ativação (ou `added_at` em `segment_parties`) — mudar sem isso inscreveria/deixaria de inscrever retroativamente jornadas ativas de outros tenants. | P2 |
| D-JRN-BF-2 | **11 jornadas de compra ativas (vários tenants) com `enroll_existing=true` ativadas antes de 2026-10-16 nunca receberam o histórico.** Decisão de produto (2026-10-16): não fazer backfill retroativo nelas; só a jornada do Instituto Socrates (`5ce0951f…`) será disparada via `journeys.request_journey_backfill` com aprovação. Reativar qualquer uma delas com "incluir base" agora cria o backfill automaticamente. Idem 3 jornadas de formulário ativas com `enroll_existing` (corte legado = criação da jornada). | Decisão registrada |
| D-PUSH-1 | **Retenção do push.** `push.fila`/`entregas`/`execucoes` crescem sem limpeza (a caixa de entrada mostra 90 dias; as linhas ficam). Criar TTL (sugestão 90-180 dias, apagando execuções finais sem item reservado) quando o volume justificar — job novo exige entrada em §11. | P3 |
| D-PUSH-2 | **Regra de feature duplicada.** `push._tem_feature` reproduz inline `core.has_feature_access` (SQL embutível, sem `auth.uid()`, para a revalidação em lote). Mudou a regra de features (plans, deny, expiração) → mudar os dois; a fumaça `scripts/push/ambiente-teste/fumaca.sql` prova a equivalência numa matriz usuário × tenant × feature. Obs.: `core.get_user_features` dá todas as features do tenant a qualquer membro quando o CHAMADOR (`auth.uid()`) é system admin — o push não herda esse atalho. | P2 |
| D-PUSH-3 | **pg_net compartilhado atrasa o sinal.** O lote do pg_net só fecha quando a requisição mais lenta termina (no Carretas, 2 min; no Kreators há timeouts de DNS de 5 s em `net._http_response`). Mitigado por `push.adotar_execucao` e pelo processador residente (`modo_residente`, ligar após o deploy). | P2 |
| D-JRN-BF-3 | **`journeys.enqueue_journey_checks()` tem ramo `'purchase'` morto** (CHECK de `trigger_type` não admite `purchase`; compra é `event_entry`) — `check_event_entries` nunca é agendado, conforme pretendido. Limpar o ramo e o comentário. Edge function `check-journey-entries` (honra `force_reentry`) não tem caller. | P3 |

---

### §22.3 Whitelist de dimensões que disparam `segmentation_input_version`

A coluna `segmentation_input_version` em `analytics.contact_state` (criada na Fase 1
do refactor de Maio/2026 — ver `docs/refactor/PLAN.md`) é o gatilho semântico
exclusivo de re-avaliação de segmentação.

A função canônica `analytics.apply_contact_state_mutation()` (Fase 2) decide bumpar
`segmentation_input_version` baseada na lista abaixo. **Mudar essa lista requer bump
de `analytics_segmentation_contract_version` (constante registrada em código + tabela
`core.platform_config`).**

#### Dimensões que SEMPRE bumpam (whitelist)

Identidade e role do contato:
- `party_type` — customer vs lead (R12). Toda mudança de role afeta segmentação.

Comportamento de compra (insumo direto de RFM e segmentos baseados em produto):
- `frequency` — número de compras
- `monetary` — valor total
- `recency_days` — dias desde última compra
- `bought_product_ids[]` — produtos comprados
- `total_purchases`, `total_revenue`, `avg_order_value`, `distinct_products` — agregados de compra
- `last_purchase_at`, `first_product_id`, `first_product_at` — primeiras/últimas compras

Derivados RFM (usados como input por segmentos hoje — confirmado em `analytics.segments.depends_on_fields`):
- `r_score`, `f_score`, `m_score`
- `rfm_segment` (mapeamento texto)
- `value_tier`
- `lifecycle_stage`

Tags, formulários, imports (insumos de segmentação manual):
- `tag_ids[]`
- `responded_form_ids[]`
- `import_ids[]`
- `acquisition_source`, `acquisition_campaign_id`

Reativação (usado em segmentos hoje):
- `reactivation_priority`

Engajamento agregado (insumo de segmentos baseados em comportamento de email):
- `last_email_opened_at`, `last_email_clicked_at`
- `email_open_rate_30d`, `email_click_rate_30d`
- `whatsapp_read_rate_30d`
- `last_engagement_at`

Identidade adicional (usado em segmentos baseados em perfil):
- Quaisquer `crm.party_traits` cujo `field_definition` esteja referenciado em
  `depends_on_fields` de algum segmento ativo (ex: `city`, `phone`, `profession` —
  detecção dinâmica via JOIN; lista evolui com novos traits)

#### Dimensões que NUNCA bumpam (blacklist)

Predições de ML (cosméticos para UI; segmentos não devem depender):
- `churn_score`
- `purchase_propensity`
- `ltv_predicted`
- `next_purchase_days`
- `next_best_product_id`
- `engagement_score`

Recomendações:
- `best_channel`
- `best_send_hour`
- `current_ladder_position`

Atribuição cosmética:
- `last_attributed_campaign_id`
- `total_attributed_revenue`
- `first_touch_revenue`

Membership (princípio P4 — projeção, não input):
- `segment_ids[]`
- `journey_ids[]`

Timestamps internos:
- `state_updated_at` (cache invalidation, não dirty trigger)
- `scores_updated_at`
- `features_version`
- `state_version` (legado pré-refactor)

#### Campos `BULK_ONLY` (R36) — fora do contrato `segmentation_input_version`

Campos de domínios 1:N (oportunidades, traits, clusters) que aparecem em
`depends_on_fields` mas NÃO vivem em `contact_state`. Avaliação ocorre via subquery
no SQL path (`eval_condition_v2`), não via watermark per-party. Nunca bumpam
`segmentation_input_version` por definição.

A função `analytics.is_bulk_only_dimension(text)` é a fonte canônica desta lista.
Decision API (`apply_contact_state_mutation`) classifica dimensões em 3 grupos:
whitelist (bumpa seg_version + enfileira) | bulk_only (bumpa apenas
`state_updated_at` + enfileira) | blacklist (nada).

Lista canônica:

**Opportunity (R36):**
- `has_opportunity`, `opportunity_status`, `opportunity_pipeline`, `opportunity_stage`
- `opportunity_assigned_to`, `opportunity_lost_reason`
- `opportunity_created_days`, `opportunity_closed_days`, `opportunity_won_days`, `opportunity_lost_days`

**Cluster (R-2.10/R-2.11 — out-of-scope no refactor de Maio/2026):**
- `cluster_id`
- `cluster_subgroup_id`

**Traits (R-2.12 — formalizado no refactor):**
- `party_traits` (genérico — "qualquer trait mudou", usado quando producer toca >5 traits)
- `party_trait_<key>` (granular — `LIKE 'party_trait_%'`, ex: `party_trait_idade`, `party_trait_cidade`).
  Usado quando producer toca ≤5 traits específicos. Avaliação real ocorre por trait_key
  via JOIN com `crm.party_trait_*` (5 tabelas tipadas).

Detecção em runtime: o worker `segment-eval-worker` reavalia esses segmentos via
fluxo BULK_ONLY (refresh_segments handler) — não via fila per-party. R36 cobre.

#### Como adicionar dimensão à whitelist

1. Argumentar em ADR (Architecture Decision Record) por que a dimensão é insumo real
   de segmentação (não derivado cosmético)
2. Validar que algum segmento ativo realmente depende dessa dimensão (`depends_on_fields`)
3. Migration adicionando entrada nesta seção do ARCHITECTURE.md
4. Bumpar `analytics_segmentation_contract_version` (constante em código + tabela
   `core.platform_config` para track)
5. Validar que `apply_contact_state_mutation()` reconhece a dimensão nova

#### Detecção runtime

A whitelist vive como CONST em `workers/shared/src/segmentation/dimensions.ts`
(será criada em R-2.1 — Decision Write API). Função canônica importa esta constante.
Mudança de whitelist sem regenerar build do worker tem efeito apenas após deploy.
Workers que invocam Decision API recebem `dirty_dimensions` como parâmetro
explícito — não fazem detecção runtime. Lista de dimensions emitidas por cada producer
está documentada em `DEPENDENCY_MAP.md` seção "Producers de contact_state".

#### Validação de cobertura (snapshot 2026-05-02)

Universo real medido em produção (`analytics.segments` ativos, `depends_on_fields`):
- 78 segmentos ativos
- 52 com `depends_on_fields` populado (≥ 1 entrada)
- 26 com array vazio `{}` — sentinela `all_contacts`/`all_customers`/`all_leads` ou
  segmentos pós D-SEG-10 que ainda não tiveram `depends_on_fields` re-derivado.
  R-3.4 trata esse caso (skip filter de dependência para segmentos sentinela).
- 21 dimensões distintas em uso (cross-check da whitelist acima):
  `all_contacts`, `bought_product_ids`, `city`, `from_import`, `has_opportunity`,
  `has_tag`, `m_score`, `not_responded_form`, `opportunity_pipeline`,
  `opportunity_stage`, `opportunity_status`, `phone`, `product_first_purchase`,
  `profession`, `r_score`, `reactivation_priority`, `recency_days`,
  `responded_form`, `responded_form_ids`, `rfm_segment`, `tag_ids`.

  Notas de validação:
  - `has_opportunity` + 4 `opportunity_*` → categoria BULK_ONLY (acima).
  - `from_import` → equivalente a `import_ids` (alias semântico em uso na UI legacy).
  - `not_responded_form` / `responded_form` → cobertos pelo aliasing de
    `normalizeAstV2ForUI` (D-SEG-10 v7.21).
  - `product_first_purchase` → temporal scoped, fora do contact_state (resolver
    `summary_exists`, BULK_ONLY também).

---

## Refactor em andamento (Maio 2026)

A plataforma está em refatoração estrutural do pipeline de segmentação. Detalhes em:
- [docs/refactor/PLAN.md](./refactor/PLAN.md) — visão geral, fases, princípios
- [docs/refactor/PROGRESS.md](./refactor/PROGRESS.md) — estado atual, decisões
- [docs/refactor/BACKLOG.md](./refactor/BACKLOG.md) — tarefas atômicas

**Princípios não-negociáveis durante refactor (P1-P10):**

P1: Single writer per state column.
P2: Decision Write API obrigatório.
P3: Versão semântica por dimensão.
P4: Membership é projeção, nunca input.
P5: pg_cron é só housekeeping (R50).
P6: Worker > cron sempre que houver tenant awareness.
P7: Eventos são imutáveis e first-class.
P8: Dependency-aware evaluation.
P9: Coalescing real via UNIQUE constraint.
P10: Observability é camada própria (R52).

Toda PR durante o refactor deve referenciar tarefa do BACKLOG (R-X.Y) e respeitar P1-P10.

---

## §22.4 Sprint 2 do refactor — Decision Write API canônica

**Status:** ✅ FECHADA em **2026-05-05** (~3 dias de execução).
**Documentação operacional viva:** `docs/refactor/PROGRESS.md` + `docs/refactor/BACKLOG.md`.

### Objetivo da Sprint 2

Centralizar a escrita em `analytics.contact_state` e o enqueue em
`analytics.segment_eval_queue` numa única função canônica
(`analytics.apply_contact_state_mutation` / `_batch`) — a **Decision Write API**.

Antes da Sprint 2, cada producer tinha que:
1. Bumpar `state_version` (legacy) manualmente
2. Decidir sozinho se enfileirava `segment_eval_queue`
3. Construir o INSERT direto com `triggered_by` arbitrário

Pós-Sprint 2, producers só escrevem suas dimensões de domínio + chamam a Decision API
com `(tenant_id, party_id, producer, dimensions_changed[], payload, priority, mode)`.
A API decide via **whitelist §22.3** se a mudança é segmentation-relevant, bumpa
`segmentation_input_version` quando aplicável, e enfileira atomicamente.

### 17 producers endereçados

**9 producers TypeScript migrados:**

| R-* | Producer | Tipo | Localização |
|---|---|---|---|
| R-2.1b | `process_purchase_analytics` | SQL fn | `analytics.process_purchase_analytics` |
| R-2.3 | `recordTagChange` | TS single-row | `historical-sync/.../analytics-events.ts:24` |
| R-2.4 | `refreshRfm` | TS bulk per-chunk | `analytics/handlers/refreshRfm.ts:241` |
| R-2.5 | `refreshFeatures` | TS bulk per-chunk | `analytics/handlers/refreshFeatures.ts:175` |
| R-2.6 | `refreshReactivation` | TS bulk + cleaner (2 escopos) | `analytics/handlers/refreshReactivation.ts:252,361` |
| R-2.7 | `recordFormSubmitted` | TS single-row | `historical-sync/.../analytics-events.ts:105` |
| R-2.8 | `recordImportAdded` | TS single-row | `historical-sync/.../analytics-events.ts:169` |
| R-2.9 | `linkProducts` | TS bulk via array | `analytics/handlers/linkProducts.ts:175` |
| R-2.12 | `traitProducer` | TS single-row BULK_ONLY | `analytics/segmentation/traitProducer.ts:727` |

**6 producers SQL migrados** (descobertos via cross-check MCP — D-2026-05-05-07):

| Função | Schema |
|---|---|
| `sync_tag_ids_to_contact_state` | analytics |
| `trigger_auto_populate_contact_state` | analytics |
| `record_email_engagement_event` | email |
| `trg_product_name_mapped` | commerce |
| `sync_party_type` | analytics |
| `sync_party_types_batch` | analytics |

**2 fora de escopo** (D-2026-05-05-06):
- `computeClusters`, `computeClusterSubgroups` — usam `segment_eval_queue` como
  canal de notificação cross-system para journey engine, não como re-eval por
  mudança de input. `segment_ids` é output, não input.

### Pattern BULK_ONLY (R36) formalizado

Dimensões em `depends_on_fields` mas que **não vivem em `contact_state`**.
Decision API classifica via `analytics.is_bulk_only_dimension(text)`:
bumpa `state_updated_at` + enfileira eval, **não** bumpa `segmentation_input_version`.
Avaliação real ocorre via SQL path (`eval_condition_v2`), não watermark per-party.

Categorias BULK_ONLY canônicas:

- **Cluster** (R-2.10/R-2.11): `cluster_id`, `cluster_subgroup_id`
- **Opportunity** (R36 original): `has_opportunity`, `opportunity_status`,
  `opportunity_pipeline`, `opportunity_stage`, `opportunity_assigned_to`,
  `opportunity_lost_reason`, `opportunity_created_days`, `opportunity_closed_days`,
  `opportunity_won_days`, `opportunity_lost_days`
- **Traits** (R-2.12 — formalizado nesta sprint): `party_traits` (genérico,
  usado quando producer toca >5 traits no mesmo evento), `party_trait_<key>`
  (granular via `LIKE 'party_trait_%'`, usado quando ≤5 traits específicos)

Detalhes da whitelist regular vs blacklist vs BULK_ONLY: §22.3.

### Decisões arquiteturais registradas (D-records)

| ID | Tema | Resumo |
|---|---|---|
| D-2026-05-03-01 | Arquitetura de migração | R-2.1 quebrada em a/b — dryrun em paralelo antes do cutover. Reduz blast radius de "tudo ou nada" no hot path de ingestion. |
| D-2026-05-03-02 | Atomicidade > resiliência local | Producers migrados não têm `EXCEPTION WHEN OTHERS` ou try/catch externo silenciador. Falha aborta transação. Justificativa: mudança parcial silenciada é pior que abort retornado ao caller. |
| D-2026-05-05-01 | Testabilidade workers/historical-sync | Worker não tem `scripts.test` nem suíte. Validação reduzida a `typecheck + build + observação produção`. Sprint dedicada de testabilidade pós-refactor (Vitest + mock SQL via DI). |
| D-2026-05-05-02 | refreshRfm gap event-driven pré-Fase 1 | Antes do refactor, refreshRfm refresha ~9k parties/6h em dimensões whitelist mas **nunca** enfileirava eval. Cron antigo (predicate `state_updated_at`) também não pegava (UPSERT não tocava esse campo). Mudanças de RFM eram invisíveis à segmentação na história da plataforma. R-2.4 fechou esse gap. |
| D-2026-05-05-03 | Coexistência de 2 evaluators | `segment-rules-evaluator.ts` (memória, hot path por job) lê `rules_json` (AST v1) — patchado em R-2.4.1. `segmentation/segmentEvaluator.ts` + `resolvers.ts` (SQL canonical via `refresh_segment_parties_unified`) lê `rules_json_v2`. Coexistência aceita como débito; migração legacy → v2 fica fora do escopo de Decision Write API. |
| D-2026-05-05-04 | `state_updated_at` redundante em refreshFeatures | UPSERT mantém `state_updated_at = now()` mesmo a Decision API batch também bumpando. Inócuo (mesma transação, último timestamp vence). Remover quando todos producers bulk forem 100% canonical. |
| D-2026-05-05-05 | refreshFeatures orquestra refresh_rfm | Cada cron run gera 2 batches em queue (refreshFeatures + refreshRfm) para os mesmos parties. Comportamento correto (cada producer declara dimensões reais). Custo transitório até R-3.2 (UNIQUE INDEX coalescente + UPSERT mesclando `fields_changed`). |
| D-2026-05-05-06 | BULK_ONLY pattern para cluster/traits/opportunity | `segment_eval_queue` tem 2 usos semânticos: (1) re-eval por mudança de input (whitelist) → Decision API; (2) notificação cross-system (segment_ids = output) → fora de escopo. Refactor para canal dedicado fica como item futuro. |
| D-2026-05-05-07 | Cross-check MCP > inventário textual | Inventário em PROGRESS.md ≠ estado real do banco. Antes de criar R-2.x novo, consultar `pg_get_functiondef` da função candidata. Cross-check final da Sprint 2 revelou 6 funções SQL já migradas silenciosamente (R-2.13–R-2.17 fechados sem código novo). |

### Métricas pré/pós Sprint 2 (combinadas com Fase 1)

| Métrica | Pré-refactor (2026-05-02) | Pós-Sprint 2 (2026-05-05) | Comentário |
|---|---|---|---|
| Amplification ratio (enqueues / mudanças reais) | 24.6x | **~1.0x** | Fase 1 (R-1.5/R-1.7) baixou predicate cron + Sprint 2 eliminou writes ad-hoc duplicados. |
| Drift global (`seg_input_version > last_evaluated`) | crescente, mascarado por cron fallback | **sustentado em 0** | Validado em produção pós-R-2.4/R-2.5/R-2.6 com ~95k jobs drenados. |
| Single-source-of-truth para writes em `contact_state` | ❌ — 12 producers escreviam ad-hoc, cada um decidia versão e enqueue | ✅ — todos producers passam por Decision API; whitelist §22.3 dita o que é segmentation-relevant | Princípio P1 (single producer per state column) materializado para metadados de versão. |
| Producers fora do contrato Decision API | 12 (todos) | 2 (out-of-scope D-2026-05-05-06) | computeClusters/Subgroups por design (notificação cross-system). |
| Triggers SQL com `state_version+1` legacy | 6 funções | **0** | Cross-check MCP em 2026-05-05. |

### Próximo: Sprint 3 ✅ FECHADA

Ver §22.5 abaixo.

---

## §22.5 Sprint 3 do refactor — Coalescing real

**Status:** ✅ FECHADA em **2026-05-05** (~30min via MCP, sem patch TS).
**Documentação operacional viva:** `docs/refactor/PROGRESS.md` + `docs/refactor/BACKLOG.md`.

### Objetivo

Garantir que múltiplos enqueues para o mesmo `(tenant_id, party_id)` em janela
de tempo curta produzam **1 única row pending** com `fields_changed` mesclado,
em vez de N rows duplicadas que o worker processaria sequencialmente.

Antes: cada producer chamava Decision API → INSERT na queue → worker processava
N jobs idênticos para o mesmo party. Após R-2.5/R-2.6, observou-se 2x-3x jobs
por cron run (D-2026-05-05-05).

### Implementação

Aplicação única via Supabase MCP (D-2026-05-05-08):

**1. UNIQUE INDEX parcial:**
```sql
CREATE UNIQUE INDEX idx_seg_eval_queue_pending_unique_party
  ON analytics.segment_eval_queue (tenant_id, party_id)
  WHERE status = 'pending';
```
Garante 1 row pending por (tenant, party). `claimed` é excluído do escopo —
representa job em processamento, não coalescível.

**2. Decision API com `ON CONFLICT DO UPDATE`:**
`apply_contact_state_mutation` (single) e `apply_contact_state_mutation_batch`
reescritas para detectar conflito no UNIQUE INDEX e mesclar.

### Merge semantics

| Coluna | Lógica | Justificativa |
|---|---|---|
| `fields_changed` | união DISTINCT ordenada | Worker filtra segmentos por interseção `fields_changed ∩ depends_on_fields` — união cobre todas as dimensões que merecem re-eval |
| `priority` | `LEAST(EXCLUDED, existente)` | Mais urgente vence (priority menor = mais urgente no schema atual). Trait change priority=3 vence sobre form_submit priority=5. |
| `triggered_by` | `'coalesced'` | Sinaliza para observability que essa row passou por merge. Constraint estendida para aceitar valor. |
| `retry_count` | reset para 0 | Job efetivamente novo após merge — falhas anteriores não devem contar |
| `created_at` | preservado | FIFO honesto — ordem temporal do primeiro insert preservada |
| `updated_at` | `now()` | Auditoria do último merge |

### Filtragem natural pelo worker

Worker R-1.6 (`segment-eval-worker`) já filtra segmentos via interseção
`fields_changed ∩ depends_on_fields` (linha 215 de `segment-rules-evaluator.ts`,
princípio P8 dependency-aware evaluation). Após merge, o array mesclado é
processado em uma única passagem — segmentos que dependem de qualquer dimensão
no array são re-avaliados, segmentos que não dependem são pulados.

**Sem mudança no worker.** A filtragem que já existia agora opera sobre o array
mesclado naturalmente.

### Validação E2E (via MCP, 2026-05-05)

- 2 inserts seguidos (`refreshFeatures` + `refreshRfm`) mesmo party → **1 row pending**
- `fields_changed = ['frequency','monetary','r_score','recency_days']` (união)
- `priority = 3` venceu sobre `priority = 5`
- `was_coalesced = true` retornado pela Decision API
- Worker R-1.6 processou em **584ms** sem erro

### Beleza arquitetural — 0 mudança em código TS

Sprint 2 (Decision Write API canônica) preparou o terreno: tudo passa pelo
**único INSERT** que agora coalesce. Princípio **P2** da constituição
(*single producer, single writer per dimension*) materializado: features
arquiteturais novas são adições **centralizadas**, não mudanças distribuídas
em N producers.

Sprint 3 endereçou comportamento de 9 producers TS + 6 SQL alterando 2 funções
SQL — fator de redução de complexidade de **~15x** vs. cenário pré-Sprint 2.

### Débitos resolvidos

- ✅ **D-2026-05-05-05** — refreshFeatures orquestra refresh_rfm gerando 2 jobs/party
- ✅ Edge case 2x-3x jobs/party de R-2.5/R-2.6 (orquestração + cleaner)
- ✅ Edge case BULK_ONLY de R-2.12 (form com múltiplos blocos enfileira N jobs)

### Métricas a monitorar (próximas 6h pós-deploy)

- Aparição de jobs com `triggered_by='coalesced'` na queue
- Volume total da queue: queda esperada de **30-50%**
- `avg_ms_processing` estável ou melhor (menos jobs duplicados = menos contenção)
- 0 errors funcionais

### Próximo: Sprint 4 ✅ FECHADA

Ver §22.6 abaixo.

---

## §22.6 Sprint 4 do refactor — Repair via coalescing pattern

**Status:** ✅ FECHADA em **2026-05-06** (~15min via MCP, sem patch TS, **escopo
90% menor que planejado**).
**Documentação operacional viva:** `docs/refactor/PROGRESS.md` + `docs/refactor/BACKLOG.md`.

### Objetivo (redimensionado)

O plano original (R-4.1 a R-4.5) previa criar um **repair worker dedicado** com
observability própria, validação 7 dias em paralelo, e eventual drop do
`pg_cron segment-eval-fallback` + função SQL legada. Estimativa: ~5 dias.

**O que mudou:** as Sprints 1+2+3 mataram a causa raiz do drift transitório.
Pré-Sprint 3, fallback detectava ~120k parties/dia (worker dedicado fazia
sentido). Pós-Sprint 3: 20 jobs/dia. Worker dedicado virou overkill.

**Sprint 4 redimensionada:** alinhar `run_segment_eval_fallback` com o pattern
de coalescing do R-3.2. 1 migration MCP única, sem worker novo.

### Implementação (R-4.1)

`analytics.run_segment_eval_fallback` reescrita:

| Antes | Depois |
|---|---|
| `INSERT INTO segment_eval_queue (...)` direto | `INSERT ... ON CONFLICT (tenant_id, party_id) WHERE status='pending' DO UPDATE` (mesmo merge de R-3.2) |
| `fields_changed = ARRAY[...13 dimensões hardcoded...]` | `fields_changed = ARRAY['repair_full_eval']` (sinal especial) |
| `priority = 9` | `priority = 9` (preservada — baixa urgência) |
| `triggered_by = 'repair'` | `triggered_by = 'repair'` (preservado — identifica origem) |

### `repair_full_eval` — sinal especial para o evaluator

Worker R-1.6 normalmente filtra segmentos via interseção
`fields_changed ∩ depends_on_fields` (princípio P8 — dependency-aware
evaluation). Se um segmento não declara dependência das dimensões mudadas, é
pulado.

`repair_full_eval` é **sinal especial** que diz ao evaluator: *"não filtra,
re-avalia o party contra todos os segmentos do tenant"*. Apropriado para
repair porque o fallback detecta drift por causas que não conhece a priori.

Implementação no evaluator: se `fields_changed` contém `'repair_full_eval'`,
o filtro `depends_on_fields` é bypassado para esse job.

### Decisão arquitetural — Decision API vs queue inserts (D-2026-05-05-09)

Sprint 4 clarificou um critério arquitetural não-óbvio: **P2 aplica-se a
escritas em `contact_state`, não a inserts em `segment_eval_queue`**.

| Caller | Muta `contact_state`? | Decision API? | Coalescing pattern? |
|---|---|---|---|
| Producer de dimensão whitelisted | ✅ | ✅ obrigatório (P2) | ✅ herda de R-3.2 |
| Producer BULK_ONLY (cluster, traits, opportunity) | ❌ | ✅ por uniformidade | ✅ herda |
| Notificação cross-system (computeClusters, repair) | ❌ | ❌ não obrigatório | ✅ obrigatório |

`run_segment_eval_fallback` cai na 3ª linha: notificação cross-system, sinaliza
ao worker "preciso re-avaliar esse party" sem mutar estado canônico. Não passa
pela Decision API; usa coalescing pattern via `ON CONFLICT DO UPDATE` direto.

### Validação E2E (via MCP, 2026-05-06)

- Drift artificial criado em 1 party
- `analytics.run_segment_eval_fallback()` detectou
- Job enfileirado com `triggered_by='repair'`, `fields_changed=['repair_full_eval']`
- Worker processou em **568ms**
- Drift drenado para 0
- Sprint 3 intacto: **10.639 jobs `coalesced`** em 2h continuam aparecendo

### Tarefas absorvidas / canceladas

| Tarefa original | Status | Justificativa |
|---|---|---|
| R-4.1 — runSegmentRepairCheck worker | ✅ DONE (redimensionado) | Substituída por mudança em SQL fn existente |
| R-4.2 — Telemetria `repair_hits` | parcial | `event_health_metrics` já cobre o essencial; métrica granular fica como item futuro |
| R-4.3 — Validação 7 dias em paralelo | n/a | Não há worker novo rodando em paralelo |
| R-4.4 — Drop pg_cron `segment-eval-fallback` | ❌ não fazer | Fallback agora é o único caminho de repair, proporcional ao volume real (20/dia). Drop perderia safety net |
| R-4.5 — Drop function `run_segment_eval_fallback` | ❌ não fazer | Função permanece, alinhada com pattern de coalescing |

### Próximo: Sprint 5 ✅ APLICADA (deploy + validação em curso)

Ver §22.7 abaixo.

---

## §22.7 Sprint 5 do refactor — Worker fairness via SQL two-phase claim

**Status:** ✅ FECHADA em **2026-05-06**. SQL aplicado via MCP + patch TS
deployado + **validado em produção via teste E2E** (burst de 528 jobs
simultâneos disparado em produção).
**Documentação operacional viva:** `docs/refactor/PROGRESS.md` + `docs/refactor/BACKLOG.md`.

### Problema observado

Diagnóstico via MCP em 2026-05-06 confirmou starvation real entre tenants:

| Tenant | Jobs/24h | Wait médio | Wait máx |
|---|---|---|---|
| Escola do Fluxo | 41k | **9.6min** | **19.8min** |
| Instituto Socrates | 2.7k | 2.7min | 5.4min |

Capacidade média OK (70 jobs/min, 10 réplicas). O problema era **concentração
temporal** (bursts de cron) + sharding por hash de tenant não distribuía
fairly entre tenants ativos no mesmo momento. Tenant grande monopolizava
réplicas que tinham seu hash, enquanto outros esperavam.

### Solução

`analytics.claim_next_segment_jobs_fair(worker_id text, batch_size int = 10)`:

**Phase 1 — round-robin per-tenant:**
- Conta `tenant_count_active` (tenants com pending > 0)
- Aloca `floor(batch_size / tenant_count_active)` jobs por tenant ativo
- Garante que cada tenant ativo recebe parcela igual da capacity

**Phase 2 — FIFO global:**
- Preenche capacity restante por `created_at ASC`
- Cobre o caso de só 1 tenant ter jobs pendentes (Phase 1 retorna pouco)

**Concorrência multi-replica:** `FOR UPDATE SKIP LOCKED` no INSERT de claim.
Réplicas competem pelas mesmas pending rows; SKIP LOCKED garante exclusão
mútua sem coordenação aplicacional. Algoritmo é **atômico** dentro da função
SQL — não há janela de raça.

### Por que SQL ao invés de TS (D-2026-05-05-10)

Alternativa considerada: round-robin local em TS com state per-replica
(possivelmente sincronizado via Redis ou similar).

Decisão: **SQL é melhor** porque:
- Single source of truth (1 função, não N réplicas com cópia de estado)
- Atomicidade real via `FOR UPDATE SKIP LOCKED`, sem coordenação cross-replica
- Worker TS fica magro: chama 1 função, processa o que volta
- Mudar política (round-robin → priority-weighted, etc.) é alterar 1 função
- View de observability (`v_queue_fairness_status`) é trivial

Continuação da tese de centralização (P2 + D-2026-05-05-08): primitivas que
existem no banco (SKIP LOCKED + funções SQL) devem hospedar coordenação;
lift to TS apenas se houver razão específica (cross-system, side effects de
processo).

### Worker TS — patch (`segment-eval-worker.ts:319-356`)

| Antes | Depois |
|---|---|
| `SELECT id FROM core.tenants WHERE hash % TOTAL_SHARDS = MY_SHARD` | (removido) |
| `for (tenant of tenants) claim(tenant_id, ...)` | claim único `claim_next_segment_jobs_fair(WORKER_ID, BATCH_SIZE)` |
| Loop processa jobs por tenant | Loop processa jobs retornados (já distribuídos) |

**TOTAL_SHARDS / MY_SHARD preservados como dead code** com warning de
inicialização se valor ≠ default (compat com Railway envs existentes; remoção
em sprint futura de cleanup). Não quebra deployments rolling.

**Logger `fair_claim_received`** adicionado para janela de validação.

### Observability

`analytics.v_queue_fairness_status`: view em tempo real com
- `tenant_id`, `tenant_name`
- `pending_count`, `claimed_count`
- `oldest_pending_seconds`

Query padrão para checkpoint pós-deploy:
```sql
SELECT * FROM analytics.v_queue_fairness_status
WHERE oldest_pending_seconds > 60
ORDER BY oldest_pending_seconds DESC;
```
**Esperado: 0 rows.**

### Validação E2E em produção (2026-05-06)

Burst artificial de **500 jobs Escola do Fluxo + 28 jobs SaudeNow simultâneos**
disparado em produção pra exercitar starvation cross-tenant.

| Métrica | Pré-Sprint 5 (baseline 24h) | Pós-Sprint 5 (burst test) | Improvement |
|---|---|---|---|
| Wait máximo Escola | 1189s (19.8 min) | **76s** | **~16x** |
| Wait máximo SaudeNow | ~5 min | **31s** | **~10x** |
| SaudeNow drenagem | sequencial atrás Escola | **paralelo** com Escola | anti-starv ✓ |
| Throughput total | 70 jobs/min | **~430 jobs/min** | **~6x** |

**Insight do throughput 6x (D-2026-05-05-11):** o salto não veio só de
eliminar starvation. O sharding artificial via `TOTAL_SHARDS/MY_SHARD`
particionava tenants entre réplicas e cada réplica iterava sequencialmente o
subset do seu shard. Capacidade real estava sendo desperdiçada por overhead
que a `claim_next_segment_jobs_fair` (claim global atomic) eliminou. Sharding
hash → deprecated.

**Critério de aceite — todos atendidos:**
- ✅ `v_queue_fairness_status`: 0 rows após teste (queue limpa)
- ✅ Logs Railway emitiram `fair_claim_received` corretamente
- ✅ 0 errors funcionais
- ✅ TOTAL_SHARDS warning não ocorreu (envs default no Railway)

### Próximo: Sprint 6 ✅ FECHADA

Ver §22.8 abaixo.

---

## §22.8 Sprint 6 do refactor — Dead letter + observability portátil

**Status:** ✅ FECHADA em **2026-05-06** (~30min via MCP, 3 migrations
cirúrgicas, **0 patch TS**).
**Documentação operacional viva:** `docs/refactor/PROGRESS.md` + `docs/refactor/BACKLOG.md`.

### Redimensionamento (mesmo padrão da Sprint 4)

Plano original previa worker dedicado de dead letter + dashboards integrados
(estimativa ~1 semana). Após Sprints 1-5 consolidarem o pipeline:

- 0 errors funcionais em 7 dias
- 0 retries esgotados
- Drift sustentado em 0

Worker dedicado virou overkill. Sprint redimensionada para **safety net
manual + views portáveis** — solução proporcional ao problema real.

### R-8.1 — Dead Letter Queue (safety net)

- `analytics.dead_letter_queue` — clone schema das duas queues
  (`processing_jobs` e `segment_eval_queue`)
- `analytics.move_to_dead_letter(queue text, job_id bigint)` — entrada manual
  para jobs irrecuperáveis identificados via inspeção
- TTL 90 dias via cron purge dedicado
- View `analytics.v_dlq_summary`

### R-8.2 — Health Views (4 views portáveis)

| View | Conteúdo |
|---|---|
| `v_pipeline_status` | Snapshot multidimensional (queues, workers, drift, throughput) |
| `v_queue_health` | Throughput / latency / errors em janelas (5m, 15m, 24h) |
| `v_drift_by_tenant` | Drift summary per-tenant |
| `v_worker_throughput` | Performance por worker (p50/p95/p99 latency) |

Portáveis: qualquer dashboard via `SELECT` (Grafana, Metabase, Datadog SQL,
admin UI próprio).

### R-8.3 — Health Alerts

`analytics.check_health_alerts()` retorna `jsonb` com violações detectadas:

| Métrica | Threshold | Severity |
|---|---|---|
| `oldest_pending` | > 5 min | warning |
| `errors` | > 10/h | critical |
| `dlq_new_24h` | > 0 | warning |
| `throughput` (com fila pending) | < 10/min | critical |
| `active_workers` (com fila pending) | < 3 | critical |

Cron a cada 5min loga alertas **críticos** em `analytics.health_alert_log`
(TTL 30d). Warnings transitórios aparecem na função mas não são persistidos
(evita ruído em log; query view direto pra estado real-time).

### Validação real-time pós-deploy

Sistema detectou warnings ATIVOS imediatamente (esperado, drenamento de
burst rolling-refresh):
- `queue_backlog`: oldest 521s
- `drift_high`: 5172 > 1000
- `has_critical = false` (apenas warnings, comportamento correto)
- `jobs_processed_5m = 2961` (~600/min)
- `coalescing_rate_pct = 94.8%`

### Out-of-scope — decisão de produto

Integração com observability stack externa (Datadog / Grafana / Metabase) é
decisão de produto, não de banco. Views `v_*` são portáveis. Worker logs
estruturados já existem (Railway).

---

## §22.9 🏁 REFACTOR COMPLETO — métricas finais e lições

**Status:** ✅ COMPLETO em **2026-05-06**. 6/6 sprints fechadas em **~5 dias**
(2026-05-02 a 2026-05-06). 17 producers de `contact_state` /
`segment_eval_queue` endereçados.

### Métricas pré/pós (acumuladas)

| Métrica | Pré-refactor (2026-05-02) | Pós-refactor (2026-05-06) | Improvement |
|---|---|---|---|
| Amplification ratio (enqueues / mudanças reais) | 24.6x | **~1.0x** | **~25x** |
| Wait máximo per-tenant | 1189s (19.8 min) | **~76s** | **~16x** |
| Throughput pico | 70 jobs/min | **~600 jobs/min** | **~8.5x** |
| Coalescing rate | 0% (não existia) | **94.8%** | feature nova |
| Errors funcionais (7d) | legacy desconhecido | **0** | — |
| Drift sustentado | crescente, mascarado por fallback agressivo (1.58M hits/7d) | **0 transitório, drena <10min** | — |
| Producers fora do contrato canônico | 12 ad-hoc, cada um decidia versão+enqueue | **2 out-of-scope (notificação cross-system)** | semantic boundary |
| Triggers SQL com `state_version+1` legacy | 6 funções | **0** | — |

### Distribuição do trabalho

| Sprint | Modo | Duração efetiva |
|---|---|---|
| 1 — Watermark semântico + worker per-party | Múltiplas tarefas TS+SQL | ~3-5 dias |
| 2 — Decision Write API canônica (17 producers) | Migração distribuída TS+SQL, hot path | ~3 dias |
| 3 — Coalescing real | **1 migration MCP, 0 patch TS** | **~30min** |
| 4 — Repair via coalescing | **1 migration MCP, 0 patch TS** | **~15min** |
| 5 — Worker fairness | 1 migration MCP + 1 patch TS | **~1h** |
| 6 — Dead letter + observability | **3 migrations MCP, 0 patch TS** | **~30min** |

**4 das 6 sprints fechadas via MCP em <1h cada.** Sprint 2 (centralização) é
a explicação: criou o ponto canônico que tornou Sprints 3-6 cirúrgicas.

### Pattern arquitetural emergente (D-2026-05-05-12)

> **P2 (Decision API canônica) + P3 (workers stateless) + coalescing
> centralizado tornam features novas em extensões cirúrgicas, não refactors
> distribuídos.**

Custo upfront da Sprint 2 (centralização canônica) foi alto: 17 producers
tocados ao longo de 3 dias. Mas habilitou todas as Sprints subsequentes a
serem fechadas em **horas via MCP** ao invés de **dias via patches
distribuídos**. ROI da centralização visível imediatamente.

**Lição operacional:** investir em consolidação canônica antes de evoluções
arquiteturais. Cada producer migrado para o ponto canônico transforma feature
futura de "atualizar N callers" em "atualizar 1 função". Custo upfront de
centralização tem payback rápido.

### Constituição inviolável (resumo)

| Princípio | Status pós-refactor |
|---|---|
| **P1** — Single producer per state column (writes em `contact_state`) | ✅ materializado para metadados de versão (Decision API canônica) |
| **P2** — Single writer per state column (RFM, features, etc.) | ✅ materializado per-dimensão whitelist |
| **P3** — Workers stateless | ✅ workers leem fila, processam, sem estado local |
| **P4** — Membership é projeção, não input | ✅ `segment_ids` mantido fora da whitelist por design |
| **P8** — Dependency-aware evaluation | ✅ filtragem `fields_changed ∩ depends_on_fields` no worker |
| **P9** — Coalescing real via UNIQUE constraint | ✅ R-3.2 (UNIQUE INDEX parcial + UPSERT merge) |
| **P10** — Observability é camada própria | ✅ R-8.2/R-8.3 (views portáveis + health alerts) |

### Débitos remanescentes (não-bloqueadores)

| Débito | Categoria | Esforço estimado |
|---|---|---|
| **D-2026-05-05-01** — workers/historical-sync sem suíte de testes | Testabilidade | 1 sprint dedicada |
| **D-2026-05-05-03** — coexistência 2 evaluators (legacy AST v1 ↔ v2) | Cleanup arquitetural | 1 sprint dedicada |
| **D-2026-05-05-04** — `state_updated_at` redundante em refreshFeatures | Cleanup pequeno | ~30min |
| **D-2026-05-05-06** — `segment_eval_queue` 2 usos semânticos (canal dedicado para journey notifications) | Refactor médio | 1 sprint dedicada |
| **D-2026-05-05-11** — Cleanup `TOTAL_SHARDS`/`MY_SHARD` envs deprecated | Cleanup pequeno | ~30min |
| Decisão de produto: observability stack externa (Datadog / Grafana / Metabase) | Produto | escopo a definir |

---

## §22.10 — Playbook: Adicionar um Novo Producer

Producer = qualquer código que detecta mudança no mundo real (compra,
tag, form, import, trait, cluster) e precisa que segmentos sejam
re-avaliados.

### Decisão 1: o producer muta `contact_state`?

**Sim** (ex: producer atualiza `frequency`, `monetary`, `tag_ids`):
→ MUST chamar `apply_contact_state_mutation` (P2 enforced)
→ NÃO pode bumpar `segmentation_input_version` manualmente
→ NÃO pode INSERT direto em `segment_eval_queue`

**Não** (ex: producer só sinaliza mudança em outra tabela como
`cluster_assignments`, `party_traits`, `opportunities`):
→ É BULK_ONLY pattern (R36)
→ Decision API enfileira eval mas não bumpa seg_version
→ Dimensão deve estar registrada em `analytics.is_bulk_only_dimension()`

### Decisão 2: TS handler ou SQL trigger?

| Critério | TS handler | SQL trigger |
|---|---|---|
| Lógica de domínio complexa | ✅ | ❌ |
| Reage a INSERT/UPDATE em tabela | ❌ | ✅ |
| Volume alto (10k+/dia) | ✅ (workers escalam) | ⚠️ (trigger síncrono) |
| Lógica em PL/pgSQL aceitável | ❌ | ✅ |

### Template TS single-row producer

Localização sugerida: `workers/<service>/src/handlers/<feature>.ts` ou
`workers/<service>/src/handlers/webhook/analytics-events.ts`.

```typescript
async function recordSomething(
  sql: SqlInstance,
  input: { tenantId: string; partyId: string; /* ... */ }
) {
  await sql.begin(async (tx) => {
    // 1. (opcional) registrar evento imutável
    await tx`SELECT analytics.record_contact_event(...)`;

    // 2. UPDATE de domínio
    //    - NÃO bumpar state_version (Decision API faz)
    //    - state_updated_at = now() é OK (redundante mas inócuo)
    await tx`
      UPDATE analytics.contact_state
      SET <coluna_relevante> = <novo_valor>,
          state_updated_at = now()
      WHERE tenant_id = ${input.tenantId}::uuid
        AND party_id = ${input.partyId}::uuid
    `;

    // 3. Decision API (gating + enqueue + atomic merge via coalescing)
    await tx`
      SELECT analytics.apply_contact_state_mutation(
        ${input.tenantId}::uuid,
        ${input.partyId}::uuid,
        '<producerName>',                        -- camelCase, ex: 'recordTagChange'
        ARRAY['<dimensão1>','<dimensão2>']::text[],
        ${JSON.stringify({ /* contexto */ })}::jsonb,
        5::smallint,                              -- priority (números menores = mais urgente)
        'enforce'
      )
    `;
  });
}
```

### Template TS bulk producer

```typescript
async function recomputeBatch(sql: SqlInstance, /* ... */) {
  for (const chunk of chunks) {
    const partyIds = chunk.map((c) => c.party_id);

    await sql.begin(async (tx) => {
      // 1. UPSERT bulk de domínio
      await tx`
        INSERT INTO analytics.contact_state ${tx(rows)}
        ON CONFLICT (tenant_id, party_id) DO UPDATE SET
          <colunas>,
          state_updated_at = now()
      `;

      // 2. Decision API batch
      await tx`
        SELECT analytics.apply_contact_state_mutation_batch(
          ${tenantId}::uuid,
          ${partyIds}::uuid[],
          '<producerName>',
          ARRAY[<dimensões reais tocadas>]::text[],
          <priority>::smallint
        )
      `;
    });
  }
}
```

### Template SQL trigger

```sql
CREATE OR REPLACE FUNCTION <schema>.<function_name>()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
  v_tenant_id uuid;
  v_party_id uuid;
BEGIN
  -- Resolver party_id e tenant_id do trigger context
  -- (varia por tabela de origem)

  -- Decision API via PERFORM
  PERFORM analytics.apply_contact_state_mutation(
    v_tenant_id,
    v_party_id,
    '<producerName>',                  -- ex: 'syncTagIds'
    ARRAY['<dimensão>']::text[],
    jsonb_build_object('context', NEW.id),
    5::smallint,
    'enforce'
  );

  RETURN NEW;
END;
$$;
```

### Dimensões whitelist (consultar antes de codificar)

Whitelist regular (Decision API bumpa `segmentation_input_version`):
- `frequency`, `monetary`, `recency_days`
- `total_purchases`, `total_revenue`, `avg_order_value`, `distinct_products`
- `last_purchase_at`, `first_product_at`, `bought_product_ids`
- `r_score`, `f_score`, `m_score`, `rfm_segment`, `value_tier`, `lifecycle_stage`
- `tag_ids`, `responded_form_ids`, `import_ids`
- `acquisition_source`, `acquisition_campaign_id`
- `reactivation_priority`, `party_type`
- `last_email_opened_at`, `last_email_clicked_at`, `email_open_rate_30d`,
  `email_click_rate_30d`, `whatsapp_read_rate_30d`, `last_engagement_at`

BULK_ONLY (R36 — vivem fora de `contact_state`):
- `cluster_id`, `cluster_subgroup_id`
- `has_opportunity`, `opportunity_pipeline`, `opportunity_stage`,
  `opportunity_status`
- `party_traits` (genérico, >5 traits) e `party_trait_<key>` (granular, ≤5 keys)

Para adicionar dimensão nova: editar `analytics.is_segmentation_relevant_dimension`
(whitelist regular) ou `analytics.is_bulk_only_dimension` (BULK_ONLY).

### Antes de PR — checklist obrigatório

- [ ] Producer name em camelCase, descritivo (ex: `recordTagChange`,
      não `tag_change`)
- [ ] Constraint `segment_eval_queue_triggered_by_check` aceita o nome
      novo (adicionar via migration se necessário)
- [ ] Producer NÃO faz UPDATE direto em `state_version` ou `segmentation_input_version`
- [ ] Producer NÃO faz INSERT direto em `segment_eval_queue`
- [ ] `dimensions_changed` passado contém SOMENTE dimensões realmente
      modificadas pelo producer (evitar arrays inflados)
- [ ] Tudo dentro de `sql.begin()` — atomicidade total
- [ ] Sem `try/catch` envolvendo Decision API (atomicity > resilience local — D-2026-05-03-02)
- [ ] Producer registrado nesta seção §22.10 (lista de producers ativos abaixo)

### Workers Edge Function que processam `integrations.jobs` — filtro por handlers

Lição D-2026-05-06-18: `worker.claim_jobs(p_provider=NULL, p_handlers=NULL)`
captura QUALQUER job pendente, independentemente do worker que invoca.
Workers que registram só um subconjunto de handlers (`integrations-worker`
trata só Eduzz/Sendgrid; `apidozap` Edge Function trata só apidozap, etc.)
DEVEM passar lista de handlers conhecidos quando `provider` não vem
explícito na URL.

Pattern correto:

```typescript
const handlers: Record<string, JobHandler> = {};
registerEduzzHandlers(handlers);
registerSendgridHandlers(handlers);

const REGISTERED_HANDLER_KEYS = Object.keys(handlers);

async function processJobs(provider?: string) {
  const jobs = await claimJobs(
    WORKER_ID,
    DEFAULT_BATCH_SIZE,
    provider,
    // Se vem provider explícito, claim_jobs já filtra por ele.
    // Se NÃO vem, filtra por handlers conhecidos (defesa em profundidade).
    provider ? undefined : REGISTERED_HANDLER_KEYS,
  );
  // ...
}
```

Sem isso, worker captura jobs de outros providers (`apidozap:*`,
`nylas:*`, `doare:*`) → "No handler found" → 10 retries → DLQ.

Pattern derivado do registry, não hardcoded — adicionar handler novo
ao worker registra automaticamente no filtro. Aplicar em qualquer
Edge Function nova que invoque `claim_jobs`.

### Antes de criar UNIQUE INDEX partial em fila/tabela quente — checklist obrigatório

Lição D-2026-05-06-14: invariante introduzida por índice unique partial
quebra TODOS os writers que não respeitam, não apenas o caminho principal.
Bug latente até cenário raro disparar (ex: cleanup só roda quando há
stale claimed).

- [ ] `grep` por TODOS os writers da tabela (UPDATE, INSERT, DELETE)
      em SQL functions, triggers e código TS dos workers
- [ ] Garantir que TODOS respeitam a invariante do índice
      (ON CONFLICT DO UPDATE, DELETE-then-INSERT, ou skip-if-exists)
- [ ] Caminhos auxiliares são writers também: `cleanup_*`, `repair_*`,
      `recover_*`, `rollback_*`, `reset_*` — auditar individualmente
- [ ] Testar com pending duplicado artificial antes de aplicar o índice:
      ```sql
      -- Reproduzir cenário "stale claimed + novo pending para mesma chave"
      INSERT INTO <tabela> (<chaves>, status) VALUES (..., 'pending');
      UPDATE <tabela> SET status='claimed' WHERE id=<above_id>;
      INSERT INTO <tabela> (<chaves mesmas>, status) VALUES (..., 'pending');
      -- Rodar cleanup. Se quebra → fix antes do índice.
      ```
- [ ] Se algum writer auxiliar não puder ser idempotente, considerar
      DELETE-first-then-UPDATE ou substituir por DELETE + reenfileiramento

### Producers ativos (atualizar quando adicionar/remover)

**TS workers:**
- `process_purchase_analytics` (SQL fn chamada via webhook ingestion)
- `recordTagChange`, `recordFormSubmitted`, `recordImportAdded`
- `refreshFeatures`, `refreshRfm`, `refreshReactivation` (cron-driven)
- `linkProducts`, `traitProducer`

**SQL triggers:**
- `syncTagIds`, `syncPartyType`, `autoPopulateContactState`
- `recordEmailEngagement`, `productNameMapped`

**Out-of-scope (não usam Decision API por design):**
- `computeClusters`, `computeClusterSubgroups` (D-2026-05-05-06: notificação
  cross-system, não mutam `contact_state`)
- `run_segment_eval_fallback` / repair (D-2026-05-05-09: usa coalescing
  pattern direto, não Decision API)

---

## §22.11 — Playbook: Troubleshooting

### Pergunta 1: o sistema está saudável?

```sql
SELECT analytics.check_health_alerts();
```

Retorno traz `alerts[]`, `has_critical`, `pipeline_status{}`.

- `has_critical = true` → atenção imediata (ver Pergunta 4)
- `alert_count > 0` mas só warnings → degradação tolerável (ver Pergunta 2)
- `alert_count = 0` → sistema em estado verde

### Pergunta 2: tem warnings — preciso me preocupar?

```sql
SELECT * FROM analytics.v_pipeline_status;
SELECT * FROM analytics.v_drift_by_tenant;
```

Decisão:
- **drift_total > 1000 mas jobs_processed_5m alto** → drenando, OK (típico em burst do cron rolling-refresh)
- **drift_total alto E jobs_processed_5m baixo** → workers travados (ver Pergunta 4)
- **drift concentrado em 1 tenant específico** → tenant com producer travado, investigar

### Pergunta 3: qual worker está mais lento?

```sql
SELECT * FROM analytics.v_worker_throughput;
```

- **p95_ms muito acima de outros workers** → réplica problemática (Railway: rolling restart)
- **p95_ms uniformemente alto** → segmentos pesados (otimizar regras)

### Pergunta 4: tem critical alert ou sistema travado

Diagnóstico passo-a-passo:

1. **Workers vivos?**
   ```sql
   SELECT
     COUNT(DISTINCT worker_id) AS active,
     array_agg(DISTINCT worker_id) AS workers
   FROM analytics.segment_eval_queue
   WHERE claimed_at > now() - interval '5 minutes';
   ```
   < 3 workers = problema infra Railway. Verificar dashboard Railway.

2. **Algum job travado em `claimed`?**
   ```sql
   SELECT id, party_id, claimed_at, worker_id
   FROM analytics.segment_eval_queue
   WHERE status = 'claimed'
     AND claimed_at < now() - interval '10 minutes';
   ```
   Se houver, worker que claim ouviu `kill -9`. Correção: cron `cleanup_stale_jobs` reseta após 30min, ou manual:
   ```sql
   UPDATE analytics.segment_eval_queue
   SET status = 'pending', claimed_at = NULL, worker_id = NULL
   WHERE status = 'claimed' AND claimed_at < now() - interval '10 minutes';
   ```

3. **Tem job em loop de retry?**
   ```sql
   SELECT id, party_id, retry_count, error_message
   FROM analytics.segment_eval_queue
   WHERE retry_count >= 3
   ORDER BY retry_count DESC LIMIT 20;
   ```
   Se sim, mover para DLQ:
   ```sql
   SELECT analytics.move_to_dead_letter('segment_eval_queue', '<id>');
   ```

4. **Tem segmento gerando erro consistente?**
   ```sql
   SELECT error_message, COUNT(*) AS occurrences
   FROM analytics.segment_eval_queue
   WHERE status = 'error'
     AND created_at > now() - interval '1 hour'
   GROUP BY error_message
   ORDER BY occurrences DESC LIMIT 10;
   ```

### Pergunta 5: como saber se um producer específico está bumpando

```sql
SELECT
  triggered_by,
  COUNT(*) AS jobs_24h,
  COUNT(*) FILTER (WHERE status='processed') AS processed,
  COUNT(*) FILTER (WHERE status='error') AS errors,
  AVG(EXTRACT(EPOCH FROM (processed_at - claimed_at))*1000)::int AS avg_ms
FROM analytics.segment_eval_queue
WHERE created_at > now() - interval '24 hours'
GROUP BY triggered_by
ORDER BY jobs_24h DESC;
```

Se producer esperado não aparece → producer quebrado ou enfileirando com nome errado.

### Pergunta 6: coalescing está funcionando?

```sql
SELECT
  ROUND(100.0 * COUNT(*) FILTER (WHERE triggered_by='coalesced')
        / NULLIF(COUNT(*), 0), 2) AS coalescing_rate_pct
FROM analytics.segment_eval_queue
WHERE created_at > now() - interval '1 hour';
```

Esperado: > 70%. Se < 30% sustentado → producers podem estar enfileirando
fora da Decision API.

### Pergunta 7: histórico de incidentes

```sql
SELECT logged_at, alert_count, alerts
FROM analytics.health_alert_log
WHERE logged_at > now() - interval '7 days'
ORDER BY logged_at DESC;
```

### Casos comuns

| Sintoma | Causa provável | Ação |
|---|---|---|
| Drift sustentado em 1 tenant | Producer específico não chama Decision API | Investigar logs Railway por tenant |
| Errors uniforme em todos workers | Bug em evaluator/migration recente | Rollback migration |
| Wait máximo > 5min | Burst grande sem capacity | Aumentar réplicas Railway |
| DLQ recebeu jobs | Jobs em loop irrecuperável | Inspecionar `analytics.dead_letter_queue` |
| Queue cresce mas workers vivos | Producer mal-comportado em loop | `SELECT triggered_by, COUNT(*) FROM segment_eval_queue WHERE status='pending' GROUP BY 1` |
| Worker em loop com `duplicate key value violates unique constraint` (`segment_eval_loop_error`) | Writer auxiliar (cleanup/repair/recovery) faz UPDATE/INSERT sem respeitar índice unique partial | Ver Pergunta 8 abaixo |

### Pergunta 8: worker em loop com `duplicate key violation`

Sintoma típico (Railway logs):
```json
{
  "message": "segment_eval_loop_error",
  "error": "duplicate key value violates unique constraint \"idx_seg_eval_queue_pending_unique_party\""
}
```

Diagnóstico:

1. **Detectar colisões pending+claimed para mesma chave:**
   ```sql
   SELECT COUNT(*) FROM (
     SELECT tenant_id, party_id
     FROM analytics.segment_eval_queue
     WHERE status IN ('pending','claimed')
     GROUP BY tenant_id, party_id
     HAVING COUNT(DISTINCT status) > 1
   ) t;
   ```
   Se > 0 → tem stale claimed que tentaria voltar para pending mas violaria
   índice. Isso bloqueia `cleanup_stale_segment_eval_jobs` em loop.

2. **Limpar stale claimed com pending duplicado (operação idempotente, segura):**
   ```sql
   DELETE FROM analytics.segment_eval_queue q
   WHERE q.status = 'claimed'
     AND q.claimed_at < now() - interval '10 minutes'
     AND EXISTS (
       SELECT 1 FROM analytics.segment_eval_queue p
       WHERE p.status = 'pending'
         AND p.tenant_id = q.tenant_id
         AND p.party_id  = q.party_id
     );
   ```
   O job pending duplicado já cobre o trabalho — deletar o claimed é seguro.

3. **Auditar writers da fila:** `grep` em SQL functions e workers TS por
   `INSERT INTO segment_eval_queue` ou `UPDATE analytics.segment_eval_queue
   SET status='pending'`. Cada writer DEVE respeitar o índice unique
   partial (ON CONFLICT DO UPDATE, ou DELETE-first quando duplicado existe).
   Ver §22.10 checklist "UNIQUE INDEX partial".

4. **Aplicar fix idempotente** (template para qualquer writer auxiliar):
   ```sql
   -- Passo 1: deletar candidatos com pending duplicado
   DELETE FROM <tabela> q
   WHERE <condicao_writer> AND EXISTS (
     SELECT 1 FROM <tabela> p
     WHERE p.status='pending' AND p.<chaves_unicas> = q.<chaves_unicas>
   );
   -- Passo 2: writer original (UPDATE/INSERT) sobre os restantes — agora seguro
   ```

---

## §22.12 — RFM-driven state-change marketing automation

**Status:** ✅ FECHADO em **2026-05-07**. Sprint 9 (compute_rfm_thresholds desacoplado para platform-scheduler) + Sprint 10 phase 1+2 (state-change transitions ledger + journey events) operacionais em produção.
**Documentação operacional:** `docs/refactor/PROGRESS.md` (D-2026-05-07-01 a -04).

### Visão arquitetural

Sistema **state-change-driven** (não apenas RFM-driven). Pipeline completo:

```
Cliente compra (event)
  ↓
process_purchase_analytics (Step 5)
  ↓ apply_rfm_buckets_for_party (cache thresholds + score real-time)
contact_state.rfm_segment changes
  ↓ trigger AFTER UPDATE (zero coupling com handlers)
INSERT analytics.rfm_transitions  (audit ledger imutável, 180d TTL)
INSERT journeys.journey_events    (event_type='rfm_segment_changed')
  ↓ worker journeys consume queue
journey_enrollments criados (se journey config matches trigger_event_type)
  ↓ worker journeys-event executa nodes
Email sent / tag applied / segment update / etc.
```

### Componentes por estágio

| Estágio | Componente | Origem | Custo |
|---|---|---|---|
| Input capture | `commerce.transactions` → triggers → `contact_state` | Sprint pré-existente | <5ms/tx |
| Score per-party real-time | `analytics.apply_rfm_buckets_for_party` | Sprint 8 | ~1ms/batch |
| Features handler | `refresh_features` (incremental) | Sprint 7 | linear no #parties |
| Thresholds globais | `analytics.compute_rfm_thresholds` | Sprint pré-existente | ~50ms/13k, ~3min/50M |
| Thresholds gate | `analytics.tenant_should_recompute_rfm` | Sprint 9 + fix | <1ms |
| Thresholds scheduler | worker journeys (`IS_SCHEDULER`) handler `refreshRfmThresholds` | Sprint 9 | 30min cycle |
| Transition detection | trigger `trg_record_rfm_transition` AFTER UPDATE OF `rfm_segment` | Sprint 10 P1 | <1ms |
| Audit ledger | `analytics.rfm_transitions` (append-only, 180d TTL) | Sprint 10 P1 | <1ms INSERT |
| Journey event emit | INSERT `journeys.journey_events` `'rfm_segment_changed'` | Sprint 10 P2 | <1ms INSERT |
| Worker consume | worker journeys (worker mode) | pré-existente, sem mudança | — |

### Trigger types disponíveis para journeys reagirem

Sistema atual já cobre 100% dos casos via 3 trigger types existentes:

| Caminho | `trigger_type` | Quando usar |
|---|---|---|
| **A — Segment-based** | `segment` | Cliente está em cluster X. Cobre 80% dos casos sem código novo. |
| **B — Event-based** | `event_entry` + `event_type='rfm_segment_changed'` | Quando timing da transição importa (campaign "Welcome Loyal" só faz sentido no momento da promoção). Permite filtrar por `event_data.from_segment` / `to_segment`. |
| **C — Form** | `form` | Submissão de formulário (não relacionado a RFM diretamente). |

UI builder Phase 3 **não foi implementada**. Decisão: **não construir UI específica** — admin pode criar journeys via SQL com Caminho B se necessário, e Caminho A já cobre 80% dos casos via UI existente.

### Decisões arquiteturais relacionadas

- **D-2026-05-07-02:** Sprint 9 — desacoplar `compute_rfm_thresholds` do handler `refresh_features`. Features são tempo-driven (recency aging diário); thresholds são volume-driven (compras). Migrado para platform-scheduler com gate condicional via `tenant_should_recompute_rfm`.
- **D-2026-05-07-03:** Sprint 9 fix — gate retorna false se tenant tem < 10 parties com purchase (sample estatisticamente insuficiente para RFM). Evita loop infinito de no-op em tenants pequenos.
- **D-2026-05-07-04:** Sprint 10 — append-only state transitions ledger via trigger AFTER UPDATE. Phase 1 (rfm_transitions ledger) + Phase 2 (journey_events emit). Phase 3 (UI builder) **descartada** — Caminho A + Caminho B existentes já cobrem casos.

### Generalização (parking lot)

O padrão "**trigger AFTER UPDATE em contact_state → INSERT em ledger + journey_event**" é generalizável para outras dimensões de mudança de estado. Candidatas para futuras Phases:

- `lifecycle_stage` (lead → MQL → SQL → customer → ...)
- `engagement_score` crossing thresholds
- `churn_score` crossing thresholds
- tag added/removed
- custom field changes

Cada dimensão seguiria pattern similar com trigger separado e `event_type` específico, mantendo zero-coupling com handlers existentes.

### Compliance constituição

| Princípio | Validação |
|---|---|
| P1 Escala horizontal absoluta | Workers stateless, SQL functions replicáveis |
| P2 Recepção O(1) | Triggers AFTER UPDATE são O(1) per row |
| P3 Async + idempotente | ON CONFLICT em journey_events, append-only ledger |
| P4 Isolation por tenant | RLS + tenant_id em todas tabelas |
| P5 Append-only ledger | `rfm_transitions` immutable |
| P6 Erros classificados | scheduler logs distinguish `complete`/`noop`/`failed` |
| P7 Mutação atômica | Trigger same-transaction com UPDATE |
| R20 TTL queue/log tables | `rfm-transitions-purge` cron 180d |

### Rules adicionados (registrar em §21)

- **R-NEW:** queries divergentes entre INPUT (SELECT) e RPC de cálculo precisam validação de consistência de filtros — caso R-7-FIX (D-2026-05-07-01)
- **R-NEW:** UPDATE crítico em `contact_state` deve usar COALESCE para preservar valores quando dado novo é NULL
- **R-NEW:** testar primeiro em tenant pequeno antes de production deployment

### Performance comprovada (dataset atual)

| Operação | Latência | Escala |
|---|---|---|
| `apply_rfm_buckets_for_party` | ~1ms | OK até 50M parties |
| `compute_rfm_thresholds` (13k) | ~50ms | ~3min @ 50M parties |
| `tenant_should_recompute_rfm` | <1ms (indexed) | OK escala 50k tenants |
| `trg_record_rfm_transition` | <1ms (1 INSERT × 2 tabelas) | OK escala milhões/dia |

### Validação operacional

- **Test 1 Escola** (2026-05-07): stale forçada 20:58 UTC → recomputed 21:13 UTC (15min) ✅
- **Test 2 Sócrates** (2026-05-07): stale forçada 21:59 UTC → recomputed 22:11 UTC (12min) ✅
- **Sprint 10 Phase 1+2** (2026-05-07): validated E2E via smoke tests com transitions reais
- **Overnight 2026-05-08:** **74 transições reais** capturadas em produção, 53 unique parties (atividade real, sem testes sintéticos)
- **0 failed jobs** últimas 24h durante toda implementação

### Phase 3 (UI builder) — descartada

Caminhos A + B existentes cobrem 100% dos casos:
- **Caminho A (segment-based):** UI atual já permite criar journey reagindo a presença em cluster RFM (cobre 80% dos casos)
- **Caminho B (event-based):** admin cria journey via SQL com `trigger_type='event_entry'` + `trigger_event_type='rfm_segment_changed'`. Worker já processa sem modificação. Permite filtrar por `event_data.from_segment`/`to_segment` para timing-sensitive campaigns.

Construir UI específica para Phase 3 não justifica investimento — duplicaria UI da Caminho A sem ganho funcional, e Caminho B é admin-only por design (low volume).

---

## Email builder — contrato de edição e renderização (2026-09-10)

O builder mantém `design_json` como fonte de verdade. Campos textuais de título, hero, produto e CTA aceitam strings antigas ou HTML sanitizado; labels de CTA permitem somente elementos inline, com URL separada. `block.props.mobile` é opcional e aditivo (`fontSize`, `padding`, `stackColumns`); ausência preserva os valores desktop. `designToHtml` emite classes estáveis por bloco. `renderEmail`/`renderEmailSync` reaplicam CSS responsivo validado após o hardener, garantindo sua presença no HTML de envio.

O callback de salvamento retorna a Promise da API; o editor serializa requisições, captura o conteúdo inline antes da compilação e exibe “Salvo” somente após confirmação. Biblioteca de blocos e recuperação de versões usam localStorage separado por tenant (e entidade para versões), sem nova tabela. Não representam histórico compartilhado no servidor. A revisão editorial não altera os gates de envio.

Renderer compartilhado atualizado para `1.1.0`, mantendo `MIN_ACCEPTED_RENDERER_VERSION` e suporte a campanhas anteriores. Frontend e worker devem publicar a constante compartilhada juntos; o worker continua consumindo o HTML compilado e registra drift de versão. Detalhes e validação: `docs/EMAIL_BUILDER_UX_PLAN.md`.

## 22.13 Email Importer — Render parity invariant

**Plan ref:** P-IMPORT v3.2, Sprint 3 (PRs #6, #7, #8, #11).

O importer de HTML (`src/import/`) recebe HTML externo (Anima, Figma, Mailchimp), converte em `design_json` via rules determinísticas, e persiste como campanha rascunho que o user edita no `CampaignEditor` normal.

### Invariante: Import preview rendering

> O preview do split-view do importer (`src/import/ui/steps/ImportPreviewStep.tsx`)
> renderiza usando o **mesmo BlockRenderer** do `CampaignEditor` —
> via `getBlockComponent(block.type)` em `src/components/email-editor/blocks/index.ts`.
>
> Importer **NÃO** introduz path de render adicional. O save final
> (`ImportFinalizingStep`) também usa `designToHtml()` existente —
> mesmo MJML pipeline que campanhas criadas do zero.

### Por que esse invariante existe

Render parity entre 3 momentos é crítico pra confiança do user:
1. **Preview no importer** — o que o user vê no split-view
2. **Editor canvas** — o que o user vê no `CampaignEditor`
3. **HTML enviado** — o que chega no inbox

Manter os 3 paths usando o mesmo renderer elimina a chance de "preview diferente do enviado". Importer puxa os mesmos block components (TextBlock, ImageBlock, etc.) e os mesmos extractors MJML do editor existente.

### Hard requirement

Nenhum PR do importer pode introduzir lógica de render de HTML email. Apenas chama os renderers existentes via:
- `getBlockComponent(type)` para preview interativo
- `designToHtml(design)` para HTML final

### P-NEXT (não-bloqueante)

`email-render-parity-audit` — projeto separado pra consolidar/auditar os 3 paths de render. Não bloqueia entrega do importer; apenas formaliza o que já é prática.

### Handoff: send gates

Worker de envio (`workers/email-campaign/`) deve, antes de enviar:
1. Ler `design_json.importMetadata.sendBlocked` — se true, abort job
2. Re-rodar `validateRenderedEmailLinks(rendered_html)` — defense in depth
3. Verificar `validateUnsubscribePresence` no design — compliance gate

Worker integration está catalogado como **P-IMPORT-FOLLOWUP**: as funções (`validateRenderedEmailLinks`, `validateUnsubscribePresence` do `src/import/validators/`) estão exportadas e prontas pra worker importar; só falta o wire-up no Railway-side.

### Acknowledge semântica

`importMetadata.acknowledgedAt` + `acknowledgedBy` SÓ limpam `requiresReview`. **Nunca** desbloqueiam `sendBlocked` — corrigir o erro no design é o único caminho. Implementação em `src/import/acknowledgeImportReview.ts`.

### Sprint 4 — Robustez (2026-05-10)

Após smoke test em produção (campanha `24c7840b-...`), identificadas e corrigidas 6 issues:

1. **Inline CSS no fallback** (`src/import/utils/cssInliner.ts` + `rules/fallbackHtml.ts`). Blocos `type=html` em fallback agora recebem `<style>` global do source inlineado (juice-like sem deps externas). Substitui o caso onde blocos cru apareciam sem estilo (ex.: "Pílula de Sabedoria" do template Cadu).

2. **Hero overlay preserved** (`src/import/rules/heroOverlayPreserved.ts`). Nova rule com precedência sobre `hero-stacked-decomposition`. Detecta padrão "img + `hero-card` + textos sobrepostos" e emite UM bloco HTML com `<table background=...>` + `text-shadow`, preservando overlay visualmente. Confidence: high (vs medium do stacked).

3. **Dedupe `blockReasons`** (`converter.ts` + `ImportFinalizingStep.tsx`). Antes: 8 `href="#"` viravam 8 frases idênticas. Agora: agrupa por `(field, value, reason)` com prefixo `"N ocorrências: ..."`. Banner amarelo mostra hint quando `sendBlocked=true`: "botão Aprovar disponível depois de corrigir erros vermelhos".

4. **JWT fix em `import-assets`** (`supabase/functions/import-assets/index.ts:269-276`). `auth.getUser()` precisa receber JWT explicitamente em vez de depender de `global.headers`. Resolve 401 spurious.

5. **SVG + background-image extraction** (`assetMigration.ts`). Adiciona `<image href>`, `<image xlink:href>` e `background-image: url()` dentro de `<style>` blocks à extração de assets.

6. **Charset detection no upload** (`src/import/utils/charsetDecoder.ts`). Detecta UTF-8/latin-1/windows-1252 via BOM, `<meta charset>`, `<meta http-equiv>` e heurística (3+ bytes inválidos em UTF-8). Substitui `file.text()` no `ImportUploadStep`. Fix mojibake ("ImersÃ£o" → "Imersão") em exports antigos.

### Sprint 4.5 — AI visual diff (validação opt-in)

**Plan ref:** `import-visual-diff` Edge Function + `ImportReviewStep` UI.

Após convert, antes de salvar, AI compara screenshots do source HTML vs design final renderizado:

```
Upload → Detect → Converting → Preview → Review (AI) → Finalizing (save)
                                              ↑
                          Snapshots (html2canvas) → Claude Vision (haiku-4-5)
                          → Lista estruturada de divergências
                          → User aprova/ignora cada uma
                          → Decisões persistidas em importMetadata.visualDiff
```

**Invariante crítico mantido:** AI **NÃO** auto-aplica correções. Cada `VisualDivergence` tem `status: pending|accepted|ignored` decidido pelo user na tela `ImportReviewStep`. Render parity preservado — caminho do converter permanece determinístico.

**Onde a AI fica:**
- Validação visual pós-convert (Edge Function `import-visual-diff`)
- Vision usa imagens (não vê o design_json) — comparação puramente visual
- Custo ~$0.005/import (Haiku 4.5)
- Falha gracefully — botão "Pular análise" sempre disponível

**Schema persistido em `email_campaigns.design_json.importMetadata.visualDiff`:**

```ts
{
  divergences: [{
    id: string,
    severity: 'high' | 'medium' | 'low',
    section: string,
    issue: string,
    suggestion: string | null,
    limitationOfEmail?: boolean,
    status: 'pending' | 'accepted' | 'ignored',
    decidedAt?: string,
    decidedBy?: string,
  }],
  overallScore: number, // 0-100
  summary: string,
  modelUsed: string,
  computedAt: string,
}
```

**Roadmap AI (Sprint 5+, não implementado):**
- Sugestões de URL para botões com `href="#"` (a partir de `aria-label`)
- AI generation de patches de correção (com user approval — nunca auto-aplica)
- Padrões recorrentes detectados pela AI → viram rules determinísticas no backlog

---

## 22.14 Módulo Chat (Whaticket + IA) — port do Carreta Oftalmo

Módulo **separado** do WhatsApp/Apidozap (decisão de produto 2026-08-06: não misturar — o chat Whaticket já roda validado no Carreta Oftalmo e foi portado como está). Escopo desta fase: só conversas + IA (sem disparo em massa, sem tela de contatos). Diferença deliberada da origem (decisão 2026-08-07): mensagem de número desconhecido **auto-cria o contato** (comportamento de inbox; o Carretas descartava por privacidade do número pessoal). O contato nasce sem tag — IA só atende após etiquetar.

- **Tabelas:** `public.chat_contatos`, `public.chat_mensagens`, `public.chat_config`, `public.chat_agentes` (exceção consciente à regra "domínio em schema próprio": prefixo `chat_*` em `public` mantém o diff 1:1 com a origem para sincronizar atualizações futuras).
- **Edge Functions:** `chat-enviar` (JWT ON; autorização por `core.tenant_users.role != viewer`; envia via Whaticket e grava com service role) e `chat-webhook` (JWT OFF; segredo `?t=` + `&tenant=` na URL; secret `CHAT_WEBHOOK_SEGREDO`).
- **Exceção consciente ao princípio "nunca processar na hora de receber":** o `chat-webhook` processa inline (responde 200 imediato e segue em `EdgeRuntime.waitUntil`) porque o pull-wait de 30s que agrupa rajadas antes da resposta da IA é parte do design validado na origem. Não usar como referência para novos receivers de integração.
- **IA:** Claude via `ANTHROPIC_API_KEY` (structured output JSON: resposta, encaminhar_humano, motivo, confirmacao_presenca). Agente por tag (`chat_agentes`); contato sem tag não é atendido. Anti-corrida: dedup determinística (`dedup_key`, janela 2 min), revalidação pós-pull-wait e pós-geração.
- **Realtime:** `chat_mensagens` (INSERT) e `chat_contatos` (*) na publication `supabase_realtime`; frontend invalida React Query (`src/lib/data/chat.ts`).
- **Feature gate:** `core.features.code = 'chat'` (concedido via `core.tenant_features`, espelhando quem tem `whatsapp`). Rota `/admin/chat`.
- **Origem do port:** repo `carreta-oftalmo-estoque` (módulo campanhas, versão local 2026-08-06). Ao sincronizar atualizações de lá, mapear `campanhas_*` → `chat_*`.
- **API Whaticket/AGPT:** referência validada (endpoints, webhooks primário/secundário, formatos de evento, gotchas) em `docs/chat-whaticket-api.md`.

## 22.15 Inbox local-first — outbox `inbox.ops`, labels locais e CRUD de pastas IMAP

O inbox (Nylas / IMAP Bluehost, ~10s por operação) segue "banco é a fonte da verdade": nenhuma ação do usuário espera o servidor de email. Plano completo em `docs/INBOX_REVAMP_PLAN.md` (§2, §8).

- **Outbox `inbox.ops`:** fila FIFO única por conta (`seq bigint identity`, drenagem `ORDER BY seq`) com `op_type` `set_flags|move|delete|send|folder_create|folder_rename|folder_delete`. As RPCs de enqueue (`api.enqueue_message_op`; `api.create_folder/rename_folder/delete_folder`) fazem na MESMA transação: mutação local otimista + INSERT do op durável + garantia de 1 job drenador pendente por conta (`nylas:apply_message_ops`, dedupe por `payload.account_id`). Idempotência de enqueue por `client_op_id`. Na drenagem, ops de mensagem são coalescidas por mensagem (last-writer-wins por campo; `delete` é terminal); ops de pasta são **barreiras individuais** — "criar pasta → mover mensagens pra ela" chega ao IMAP nessa ordem exata.
- **Blindagem de campo:** `inbox.upsert_message` nunca deixa sync/webhook sobrescrever campo coberto por op `pending`/`applying` da mesma mensagem (`set_flags{unread}` blinda `is_unread`; `move` blinda `folders`; `delete` blinda tudo). Op aplicada → blindagem cai e o servidor volta a vencer. Mesmo padrão no `nylas:sync_folders`: linha de pasta com `sync_status != 'active'` pertence ao op local e não é reconciliada naquele ciclo.
- **CRUD de pastas (§8.4):** mutação otimista em `inbox.folders` (`sync_status creating|renaming|deleting`, spinner na sidebar) + worker aplica via `POST/PUT/DELETE /v3/grants/{g}/folders` (`workers/email-sync/src/handlers/folderOps.ts`). Gotcha do conector IMAP v3: o folder id é opaco no formato `v0:{grant}:{path IMAP}` — rename muda o id da subárvore INTEIRA, então há remap pós-rename (GET /folders, casa por path, atualiza `inbox.folders.nylas_folder_id` e reescreve o JSONB `inbox.messages.folders`; a UI filtra por UUID local e não quebra). Delete revalida vazia NO SERVIDOR antes do DELETE. Falha permanente → op `failed` + pasta `sync_status='error'` (badge "falhou — tentar de novo/descartar" com `api.retry_folder_op`/`api.discard_folder_op`) + cascata fail-fast cancelando moves dependentes com restore de `payload.prev_folders`. Coalescing no enqueue: rename de pasta ainda `creating` edita o op de create; delete de pasta `creating` cancela o create e remove a linha.
- **Labels 100% locais (`inbox.labels`/`inbox.message_labels`):** camada semântica sem sync IMAP — INSERT local instantâneo, sem outbox, sem reconciliação; sobrevivem a move/arquivamento. Trade-off aceito: não aparecem no webmail. Contadores on-demand — nunca materializar antes de dor medida (lição do incidente 581GB da segmentação).
- **Tenant check nos wrappers `api.*` (R56):** `PERFORM app_auth.assert_tenant_access(p_tenant_id)` — membership em `core.tenant_users` OU system_admin. Nunca comparar com o claim `active_tenant_id` (ausente em 14/22 usuários; pode apontar para tenant revogado). A migração `20260827150000` varre `pg_proc` e corrige qualquer regressão do padrão estrito, falhando alto se não conseguir.
- **Lockdown pg_net (R57, `20260827160000`):** schema `net` revogado de `authenticated`/`anon` — estava exposto via PostgREST e permitia SSRF (qualquer usuário logado fazia o banco emitir HTTP arbitrário). Chamadas internas (triggers/cron/SECURITY DEFINER/service_role) intactas; `ALTER DEFAULT PRIVILEGES` previne regressão em funções futuras do schema.

---

## 22.16 Consentimento de email — trilha e RPC única (2026-10-20)

Origem: `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` (Onda 1, itens 1.1 e 1.2). Migrations `20261020120000_consent_audit_log`, `20261020120100_apply_consent_change_rpc`, `20261020120200_consent_cancel_enrollments_exited`, `20261020120300_consent_audit_attempted_group`, `20261020120400_consent_audit_baseline_snapshot`.

- **Trilha `email.consent_audit_log` (append-only):** uma linha por tentativa de mudança de consentimento — `action` (unsubscribe|resubscribe|suppress|unsuppress|feedback|preferences_view), `channel` (preference_center|one_click|footer_get|webhook_sendgrid|admin_manual|import|api|system), `result` (success|noop|rejected|error), `before_state`/`after_state` (snapshot das linhas de `unsubscribes` + `suppression_list` do email no tenant), contexto do token (`send_id`, `recipient_id`, `campaign_id`, `token_issued_at`, `token_group_id`), `ip_hash` e `user_agent` (≤512). `group_id` sem FK (histórico sobrevive à exclusão do grupo). Trigger bloqueia UPDATE/DELETE/TRUNCATE para qualquer papel; retenção (sugestão 5 anos, a fechar com jurídico) só por migration que desabilita o trigger explicitamente.
- **IP nunca em claro na trilha:** `email.hash_ip(text)` = HMAC-SHA256 com segredo do Supabase Vault `consent_ip_hash_secret` (gerado dentro do banco na migration; fallback GUC `app.ip_hash_secret`; sem segredo → NULL). As colunas legadas `unsubscribes.ip_address` / `unsubscribe_feedback.ip_address` / `email_events.ip_address` deixaram de receber IP pela RPC na Onda 2.5 (2026-10-21) e os históricos foram anonimizados; `email_events.ip_address` de open/click continua vindo do webhook SendGrid.
- **Escrita — `email.apply_consent_change(p jsonb) → jsonb`** (SECURITY DEFINER, `search_path=''`, EXECUTE só `service_role`; chamada com `.schema("email").rpc(...)`, mesmo padrão de `email.get_preference_center_data`): normaliza email, valida tenant (inexistente → `rejected/invalid_tenant` sem linha), serializa por `pg_advisory_xact_lock(tenant+email)`, valida grupo do tenant e ativo (`rejected/invalid_group`, auditado com o grupo tentado), descarta `send_id`/`recipient_id`/`campaign_id` de outro tenant (warning). Efeitos: unsubscribe global/grupo = upsert idempotente (reativa, zera `resubscribed_at`; já ativo = `noop` preservando `unsubscribed_at` original); `campaign_recipients.unsubscribed_at` (1ª vez vence; `status='unsubscribed'` só no one-click global, como antes); `email_events` `unsubscribe`/`group_unsubscribe` (`sendgrid_message_id='self:<recipient>'`, exceto canal webhook, que já grava o evento da SendGrid); global → `email.consent_cancel_journey_enrollments` (status `exited`, `context_json.cancellation_reason='global_unsubscribe'`, em sub-bloco: falha vira warning e não derruba o descadastro). Resubscribe = `is_active=false` + `resubscribed_at` (nunca apaga). Suppress (`hard_bounce|spam_report|invalid|unsubscribe|manual`; `soft_bounce` rejeitado — contador fica no webhook) e, para `spam_report`/`unsubscribe`, também descadastro global. Feedback em sub-bloco: falha → `result=error` auditado sem derrubar a transação. `preferences_view` = só auditoria.
- **Lote — `email.apply_consent_changes(p jsonb)`**: `{<contexto comum>, changes:[...]}` (≤100) na mesma transação; cada item gera sua linha de auditoria com o mesmo `request_id`. Usado pelo save-preferences (global + grupos + feedback atômicos — P13).
- **Erro — `email.log_consent_error(p jsonb) → uuid`**: Postgres não tem transação autônoma; se a RPC lançar, o caller grava `result=error` com este helper (nunca lança).
- **Leitura:** `api.get_consent_history(p_tenant_id, p_email, p_limit=100)` — `app_auth.assert_tenant_access` (R56) + `core.is_tenant_admin` (owner/admin ou system admin); `email.get_consent_history` (service_role).
- **Helper TS:** `supabase/functions/_shared/consent.ts` (`applyConsentChange`, `applyConsentChanges`, `logConsentError`, `requestClientInfo`, tipos) e `_shared/unsubscribeToken.ts` (cópia canônica do verificador HMAC do token).
- **Baseline (D5):** 2.436 linhas `channel=system`, `request_id='baseline_snapshot_20261020'` — 1.593 unsubscribes ativos (299 globais, 1.294 de grupo) e 843 supressões ativas.
- **Bug achado:** `public.cancel_enrollments_on_global_unsubscribe` usa `status='cancelled'`, fora do CHECK de `journeys.journey_enrollments` (`active|completed|completed_with_errors|exited|error`). As edge functions ignoram o erro → nenhuma jornada foi cancelada por descadastro global até hoje (o envio era barrado pelo pré-check de `processJourneyNode`). Corrigida em 2026-10-20 (migration `20261020130000_fix_cancel_enrollments_on_global_unsubscribe`): virou wrapper de `email.consent_cancel_journey_enrollments`, com EXECUTE só para `service_role`.
- **Onda 1b (2026-10-20, não deployado):** o webhook SendGrid (`integrations-worker/handlers/sendgrid.ts`) grava supressões via `applyConsentChange` (`channel=webhook_sendgrid`) e lança em qualquer erro de escrita. `webhooks-sendgrid` exige assinatura quando `SENDGRID_WEBHOOK_VERIFICATION_KEY` existe (`_shared/sendgridSignature.ts`, DER→P1363). `retrySoftBounce` respeita descadastro em marketing (`workers/email-campaign/src/unsubscribeFilter.ts`). Detalhes em `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` § Execução — Onda 1b.
- **Onda 2 — banco e workers (2026-10-21; migrations aplicadas, workers não deployados):** migrations `20261021120000_campaign_message_type_validation`, `20261021120100_unsubscribe_group_deactivation`, `20261021120200_consent_rls_and_ip_cleanup`. (1) R60: trigger de validação em `campaign_sends`, colunas `transactional_reason` em `email_campaigns`/`campaign_sends`, trilha `email.campaign_type_changes`, RPC `api.set_campaign_message_type(p_tenant_id, p_campaign_id, p_message_type, p_reason, p_group_id default null) → jsonb`; 139 campanhas transacionais receberam motivo `legacy: …`. (2) `api.deactivate_unsubscribe_group(p_tenant_id, p_group_id, p_strategy, p_target_group_id default null) → jsonb {deactivated, converted, migrated, skipped, rejected}` (`convert_to_global`/`migrate_to_group` via `apply_consent_changes` `channel=admin_manual`; `keep`); FK `unsubscribes_group_id_fkey` RESTRICT; workers tratam unsub de grupo inativo como global. (3) Políticas "manage" `ALL` removidas de `unsubscribes`/`suppression_list` (escrita só service_role/RPC); `email.consent_normalize_input` não emite mais `ip_raw` e `consent_upsert_unsubscribe` ignora `p_ip` → a RPC grava `ip_address=NULL` nas três tabelas; históricos anonimizados (1.592 unsubscribes, 210 feedbacks, 3 email_events de consentimento). Detalhes em `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` § Execução — Onda 2 (banco e workers).
- **Onda 3 — backend (2026-10-22; migrations aplicadas, edge functions e worker não deployados):** migrations `20261022120000_consent_resubscribe_clears_inactive_groups` (nova versão de `email.apply_consent_change`: resubscribe global `preference_center` limpa unsubs de grupos inativos; retorno ganha `inactive_groups_cleared`; estendido a `admin_manual` em 2026-10-29 por `20261029180000_manual_consent_rpcs_fixes`), `20261022120100_consent_rate_limit_helpers` (`email.count_recent_resubscribe_requests`, `email.count_recent_preference_loads_by_ip`, índice `idx_consent_audit_ip_time`), `20261022120200_consent_metrics_rpcs` (`api.get_campaign_unsubscribe_stats(p_tenant_id, p_send_ids) → table(send_id, delivered, unsubscribed, spam_reports, unsubscribe_rate, spam_rate)` e `api.get_tenant_consent_summary(p_tenant_id, p_days=30) → jsonb`, índice `idx_unsubscribes_send`), `20261022120300_consent_backfill_d3_d4` (D3: 49 globais para spam_report; D4: 1.607 `campaign_recipients.unsubscribed_at`). Edge: token ≤ 90 dias para ver/reativar/feedback (descadastro nunca expira; `400 token_expired`), ≤ 3 emails de confirmação por email+tenant/24 h (`429 too_many_requests`), `load` ≤ 60/h por `ip_hash`; Sentry mínimo `_shared/sentry.ts` (envelope HTTP, `SENTRY_DSN`, no-op sem DSN). Worker: marketing sem domínio e sem `PREFERENCE_CENTER_FALLBACK_URL` → destinatários `failed` com `preference_center_url_unavailable` (P18). Detalhes em `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` § Execução — Onda 3 (backend).
- **Incidente 2026-10-26 (migrations aplicadas; UI não adaptada):** `20261026120000_duplicate_campaign_copies_transactional_reason` (adendo R60: `email.duplicate_campaign` copia o motivo; transacional sem motivo → marketing com grupo; EXECUTE de `email.duplicate_campaign` só `service_role`) e `20261026120100_create_campaign_send_atomic` (R61: `api.create_campaign_send(p jsonb) → jsonb` e `api.cancel_orphan_campaign_sends(p_tenant_id) → int`; `recipient_source` aceita `resend_unopened`). Medido no tenant 0…001 (transação revertida): 3 segmentos (62.997 vínculos → 28.658 destinatários) em 2,4 s; todos os contatos (29.754) em 2,3 s; reenvio a não abertos (7.511) em 0,5 s. Contrato completo em `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` § Incidente 2026-10-26.
- **Consentimento manual pela UI (2026-10-29; migrations `20261029120000_manual_consent_rpcs` e `20261029180000_manual_consent_rpcs_fixes` aplicadas; UI em paralelo):** fecha A.6 do `docs/EMAIL_UNSUBSCRIBE_AUDIT.md` (antes não havia tela; `source=manual` só por SQL). Ambas `SECURITY DEFINER`, `search_path=''`, EXECUTE `authenticated`+`service_role` (sem `anon`); `service_role` pula as checagens.
  - `api.set_email_consent(p_tenant_id uuid, p_email text, p_action text, p_scope text, p_group_id uuid default null, p_reason text default null) → jsonb` — `p_action` unsubscribe|resubscribe, `p_scope` global|group. Chama `email.apply_consent_change` com `channel=admin_manual`, `actor_user_id=auth.uid()`, `request_id=gen_random_uuid()`, `request_payload {reason (btrim, vazio→null, ≤ 500 chars), ui:'tenant_admin'}` e devolve o jsonb dela **sem `before_state`/`after_state`** (o snapshot completo — histórico, `send_id`, `bounce_count` — fica só na auditoria; antes vazava para qualquer membro não-viewer, inclusive em chamadas `rejected`) + `email` normalizado. Resubscribe `global` também desativa unsubs ativos de grupos **inativos** (R60; `inactive_groups_cleared` no retorno, warning `inactive_groups_cleared:<n>` e ids em `request_payload.inactive_groups_cleared` da mesma linha de auditoria), porque resubscribe no grupo inativo é `rejected/invalid_group`; `rejected` (ex.: `invalid_group`, `scope_group_mismatch`) volta normal, o front lê `error_code`; idempotente (`noop`). **Permissão:** `app_auth.assert_tenant_access` (R56) e então *unsubscribe* → membro com `core.tenant_users.role <> 'viewer'` no `p_tenant_id` ou system admin (mesma regra de `api.create_campaign_send`/`useCanEdit`; o role é lido da tabela, não do JWT, porque `app_auth.is_member_or_above()` reflete só o tenant ativo), senão `42501 edit_access_required`; *resubscribe* → `core.is_tenant_admin` (owner/admin ou system admin), senão `42501 admin_required`. Validação: `22023` `invalid_email`/`invalid_action`/`invalid_scope`. Exceção da RPC interna é re-lançada **sem** `email.log_consent_error` (mesma transação: o RAISE desfaria a linha; o helper só serve a callers fora do banco).
  - `api.get_email_consent_status(p_tenant_id uuid, p_email text) → jsonb` — qualquer membro (`assert_tenant_access`). `{email, global:{active, unsubscribed_at, resubscribed_at, source}|null, groups:[{group_id, name, is_active (do grupo), unsubscribed_at, source}] (só unsubs ativos), suppressed:[{suppression_type, reason, suppressed_at}] (ativas e não expiradas), effective}`. `effective`: `suppressed` > `global_unsubscribed` (global ativo **ou** unsub ativo em grupo inativo, R60) > `partial` (só grupos ativos) > `subscribed`. Lê `suppression_list` por SECURITY DEFINER (authenticated continua sem GRANT na tabela).

---

## 22.17 Notificações push (Web Push) — port do Carreta Oftalmo (2026-10-20, não aplicado)

Central de notificações da EQUIPE do tenant (não de contatos): caixa de entrada em `/admin/notificacoes` + Web Push no PWA. Port 1:1 do módulo validado no Carreta Oftalmo (bateria de concorrência de 15 cenários, revisão adversarial, produção desde 10/10/2026), com identificadores em português como o chat (§22.14). Relatório: `docs/push/backend-port-2026-10-20.md`; aplicação: `docs/push/APLICAR.md`; front: `docs/push/frontend-port-2026-10-20.md`.

- **Schema `push` (fora do PostgREST):** `config` (linha única), `catalogo` (tipos de aviso; nasce vazio; `permissao_ref` = `core.features.code`, validado por gatilho, sem FK para não travar o DELETE de features), `preferencias` (única por tenant/usuário/tipo; `catalogo.ligado_por_padrao`), `inscricoes` (aparelhos, do usuário e não do tenant; teto 15 que preserva quem recebe), `fila` (uma notificação por usuário; `lida_em`/`arquivada_em` da caixa), `entregas` (por aparelho, token de confirmação), `execucoes`, `campanhas`, view `painel_adocao`. RLS ligada em todas.
- **Destinatário:** membro de `core.tenant_users` do tenant, com a feature pela regra de `core.has_feature_access` (`push._tem_feature`, D-PUSH-2), usuário ativo em `auth.users` (sem `deleted_at`, sem `banned_until` vigente), tenant ativo (`is_active` e `archived_at` nulo), preferência efetiva ligada, e nunca o autor do evento. Revalidado no envio (`push._motivo_fila`: `tipo_inativo`, `permissao_inativa`, `tenant_inativo`, `usuario_inativo`, `sem_permissao`, `preferencia_desligada`).
- **Produtores:** `push.enfileirar_evento(tenant, chave, origem_tipo, origem_id, payload, autor)` (uma chamada por evento, `INSERT … SELECT`, idempotente pela chave `tipo:origem_tipo:origem_id:usuario`) e `push.enfileirar` (um destinatário), chamados por gatilhos `SECURITY DEFINER` dos módulos (exemplo em `docs/push/APLICAR.md` §9). Nunca levantam erro: push não derruba a transação de negócio.
- **Sinal:** constraint trigger `fila_sinalizar_commit` (adiado para o commit) → `push.sinalizar` (trava do despachante, `lock_timeout 2s`) → execução `agendada` + `net.http_post` com `Authorization: Bearer <service role>` (GUC `supabase.service_role_key`, senão Vault `service_role_key`, como `integrations.invoke_worker`; `x-push-segredo` opcional). Modo residente (`push.config.modo_residente`): uma execução viva ~120 s escuta `LISTEN push_sinal` e encadeia a sucessora; o pg_net vira rede de segurança.
- **Consumidor:** Edge Function `push-processador` (202 + `EdgeRuntime.waitUntil`; `push.adotar_execucao` → `reservar_lote` (SKIP LOCKED) → `resolver_destinatarios` → `entregas_validas` (revalida) → envio com paralelismo 20 e timeout 5 s → `finalizar` (todas as entregas de uma notificação numa chamada) → `encerrar_execucao`). Cifra própria sobre WebCrypto (RFC 8291 byte a byte nos testes). Teto de parede 150 s (o runtime descarrega requisição ociosa em ~170 s), cifra ≤ 1 s de CPU por invocação.
- **Rede de segurança:** cron `kreators-push-recuperar` (§11, categoria A). Tentativa conta na finalização, não na reserva; reserva vencida volta sem queimar tentativa; `max_reservas` 20.
- **Client (`api.push_*`, 19 wrappers):** `SECURITY DEFINER`, `search_path=''`, `auth.uid()` na 1ª linha, tenant por `app_auth.assert_tenant_access` (R56). `anon` só executa `push_vapid_chave_publica`, `push_confirmar_exibicao` e `push_registrar_clique` (Service Worker sem JWT, com token por entrega). Painel de adoção só para `core.system_admins`.
- **Payload (contrato com o SW):** `{v:1, titulo, corpo, url, tag, renotify, entrega_id, token, icone, badge}`; `tag` = id da fila (`evento`), `<chave>:<AAAA-MM-DD>` (`dia`) ou `teste`. Sem dado pessoal de contato (aparece na tela bloqueada).

---

## 23. Roadmap Futuro

**Tracking de origem das oportunidades** (D-OPP-SOURCE-4 segue aberta)
- ✅ Backfill executado em 2026-05-07: 633/750 reclassificadas como `automation` com `source_journey_id` populado
- ✅ Handler `executeCreateOpportunityNode` instrumentado — novas oportunidades de automação já nascem etiquetadas
- ⏳ Relatório `/insights` cruzando `source_id` × `source_origin` × `source_journey_id` × `lost_reason_id`
- ⏳ Quando webhooks/forms passarem a criar opportunity (não criam hoje), instrumentar igual ao caminho automation

**Personalização e cache**
- Vercel KV para cache de edge (`contact_state` hot path)
- Identity resolution em checkout e formulários
- `visitor_sessions` conectado ao frontend

**Dashboard Piloto Automático**
- API de oportunidades de receita, cards de revenue em risco
- Attribution dashboard, product ladder visualization

**ML Real**
- Schema `cluster_runs` / `cluster_definitions` / `cluster_members` / `cluster_segments`
- `source_run_id` indexável em `segment_parties`
- `computeClusters.ts` — K-means k=5 lendo `contact_state`
- UI de renomear label de cluster, features comportamentais
- HDBSCAN/GMM via microserviço Python, lookalike por embeddings, A/B testing com bandit

---

## 24. Pipeline de Formulários → Traits

### Fluxo end-to-end

```
form_submission (status='submitted', party_id NOT NULL)
  → trigger enfileira job `project_traits` em analytics.processing_jobs
  → analytics-worker consome job
  → traitProducer.processSubmission()
    1. Carrega field_definitions (form_block_id → trait metadata)
    2. Busca form_answers do submission
    3. Filtra por COMPATIBLE_ANSWER_TYPES (single_choice→single_select, etc.)
    4. Upsert em crm.party_trait_text / party_trait_number / party_trait_timestamp / party_trait_boolean / party_trait_multi_value
    5. Merge party_profile.custom_fields
    6. updatePartyIdentity (blocos de identidade)
    7. Enfileira segment_eval_queue com triggered_by='form_submission'
```

### Arquitetura de dados — 3 camadas

**`crm.party_person` / `crm.party_profile`**
Campos de identidade (email, phone, nome) — atualizados via `updatePartyIdentity()`.

**`crm.field_definitions`**
Contrato semântico dos campos do tenant. `(tenant_id, field_key, field_type, operator_family, source_system='forms', source_field_id=block_id)`. Populado por `sync_field_definitions` a cada versão publicada.

**`crm.party_trait_text/number/timestamp/boolean/multi_value`**
Traits projetados por submission. Índice por `(tenant_id, field_key, value_*)`. Trait só é gravado se existir `field_definition` mapeada para o block. Idempotente: `ON CONFLICT (tenant_id, party_id, field_key) DO UPDATE`.

### Mapeamento block_type → destino

| block_type | Destino |
|---|---|
| `email`, `phone`, `first_name`, `last_name`, `full_name` | `crm.party_person` (identidade) |
| `short_text`, `long_text`, `single_choice`, `picture_choice`, `dropdown` | `crm.party_trait_text` |
| `yes_no` | `crm.party_trait_boolean` |
| `multiple_choice`, `checkbox` | `crm.party_trait_multi_value` |
| `number`, `rating`, `nps`, `ranking` | `crm.party_trait_number` |
| `date` | `crm.party_trait_timestamp` |
| `contact_info`, `address` | skip — D30 (decomposição futura) |
| `statement`, `website` | skip |

### Regras permanentes

- **R30:** traits NUNCA são materializados em `contact_state` — violação arquitetural
- **R36:** campos 1:N (oportunidades, tickets) NUNCA são denormalizados em `contact_state` — usar EXISTS subquery via `opportunity_exists` resolver
- **`project_traits` e `sync_field_definitions` NÃO são jobs coalesced** — múltiplos pending simultâneos por tenant são esperados e corretos
- A unique index de deduplicação em `analytics.processing_jobs` cobre APENAS jobs coalesced (`refresh_features`, `refresh_rfm`, etc.) — `project_traits` e `sync_field_definitions` estão fora dessa lista intencionalmente
- `tenant_id` vem sempre do job, nunca do payload
- `field_definitions` são pré-requisito: sem elas, `processSubmission` retorna 0 traits. Sempre rodar `sync_field_definitions` antes de `project_traits`
- `COMPATIBLE_ANSWER_TYPES`: mapeia tipos de resposta do formulário para tipos de campo. `single_choice` e `picture_choice` são compatíveis com `single_select`
- Backfill usa processamento direto: `backfillFormTraits.ts` chama `processSubmission()` diretamente, sem passar pela fila de jobs. Evita problemas com unique constraint de coalescing
- `DbConnection` adapter: scripts que usam `postgres.js` nativo precisam do adapter `{ unsafe, begin }` para compatibilidade com traitProducer

### Arquivos principais

| Arquivo | Papel |
|---|---|
| `workers/analytics/src/segmentation/traitProducer.ts` | Core: `processSubmission`, `syncFormFieldDefinitions`, `resolveFieldDefinitions`, `normalizeAnswer`, `updatePartyIdentity` |
| `workers/analytics/src/handlers/formTraits.ts` | Handlers de job: `handleProjectTraits`, `handleSyncFieldDefinitions` |
| `workers/analytics/src/segmentation/scripts/backfillFormTraits.ts` | Script de backfill (Phase 0: field_definitions, Phase 1: submissions) |
| `supabase/migrations/*_create_party_profile_traits.sql` | Schema das tabelas de traits e field_definitions |
| `supabase/migrations/*_fix_trg_enqueue_trait_projection.sql` | Trigger de enqueue para `project_traits` |
| `supabase/migrations/*_fix_processing_jobs_coalesced_index_scope.sql` | Scoping do unique index para excluir jobs per-entity |

---

## 25. Frontend mobile — casca, kit e guarda-corpos (2026-10-21)

Resultado do programa de redesign mobile (`docs/mobile-redesign/PLANO-MESTRE.md`, Ondas 3 a 12). Esta seção é o
contrato vigente; o plano e os `BRIEFS-ONDA-*.md` são o histórico das decisões. Regras próprias desta área usam o
prefixo **M** (M1–M3) para não colidir com as R1–R63 da Parte C.

### 25.1 Breakpoints e modos (D3)

| Modo | Largura | Casca | Primitivas do kit | Builders (e-mail, jornada, formulário, LP, segmento) |
|---|---|---|---|---|
| Celular | `< 768` | header contextual de 56px + safe area, BottomNav (4 abas + Mais), sem sidebar/gaveta (o `/mais` substitui) | modo mobile (Drawer, lista, sheet) | leitura/resumo (D22) |
| Tablet | `768–1023` | rail de 80px; "Expandir menu" abre a sidebar de 256px por cima do conteúdo; sem BottomNav | modo desktop | leitura/resumo (abaixo de `lg`) |
| Desktop | `>= 1024` | sidebar de sempre (256/80, recolher, auto-recolher na Inbox) | modo desktop | completos |

- Primitivas trocam de modo em `md` (768, igual ao `useIsMobile`); a casca troca em `lg` (1024).
- Hooks síncronos no primeiro render (`useSyncExternalStore`, sem flash Dialog→Drawer): `useMediaQuery`,
  `useBreakpoint`, `useIsMobile`, `useIsTouch` em `src/hooks/use-media-query.ts` (`use-mobile.tsx` reexporta).
- Toque não se decide por largura: variantes Tailwind `touch:` (`pointer: coarse`), `hover-hover:` e `standalone:`;
  `future.hoverOnlyWhenSupported` ligado (`hover:` só onde há hover de verdade).
- CSS antes de JS: classe responsiva quando o DOM é o mesmo; hook só quando a árvore muda (Dialog ou Drawer, tabela
  ou lista), para não montar as duas.

### 25.2 Casca

**`navConfig` é a fonte única de navegação** (`src/config/nav-config.ts`, arquivo puro, roda no vitest):
`NAV_ITEMS`/`ACCOUNT_NAV_ITEMS` (rótulo, ícone, grupo, `access` por feature, `badge`, `desktopOnly`, `keywords`),
`NAV_GROUPS`, `ROUTE_META` (título, `parent` para o voltar, `navId`, `layout: "fullbleed"`, `hideBottomNav`,
`skeleton`), `resolveTabs` (abas da BottomNav por feature/papel, D1/D4), `getMobileMoreSections` (`/mais`),
`getParentPath`/`getRouteTitle`. Rota nova = entrada em `ROUTE_META` (e `NavItem` se aparece no menu); nunca array
solto no `AppLayout`. O teste `src/config/__tests__/nav-config.test.ts` garante que a sidebar gerada é a mesma de antes
(grupos, itens, rótulos, ordem). Leitores no `DEPENDENCY_MAP.md` ("Casca e navegação").

| Peça | Arquivo | Papel |
|---|---|---|
| `AppLayout` | `src/components/layout/AppLayout.tsx` | monta os três modos, mede header/BottomNav e passa `topInset`/`bottomNavHeight` ao `ShellProvider` |
| `ShellProvider` / `useShell` / `useShellChrome` | `src/components/shell/shell-context.tsx` | contrato tela ↔ casca: registro do header, da `BottomActionBar` e do modo imersivo; publica as variáveis CSS. Sem provider (páginas públicas, `/__dev/kit`) é no-op com `hasShell: false` e as primitivas renderizam inline |
| `PageHeader` + `ShellHeaderOutlet` | `src/components/shell/page-header.tsx` | no celular registra título/voltar/ações no header fixo (até 2 ícones, resto no ⋯); no desktop renderiza inline. Sem registro, o outlet mostra título e voltar do `ROUTE_META` |
| `BottomNav` | `src/components/shell/bottom-nav.tsx` | 4 abas + Mais, badges (`useNavBadges`); some com teclado aberto, `BottomActionBar` montada, tela imersiva ou `hideBottomNav` da rota |
| `/mais` | `src/components/shell/mobile-more-menu.tsx`, `src/pages/Mais.tsx` | destinos por feature, troca de tenant, Recentes, sair (com `useConfirm`) |
| `CommandPalette` | `src/components/shell/command-palette.tsx` | busca global sobre o `navConfig` (⌘K) |
| `FullBleedPage` / `ImmersiveScreen` | `src/components/shell/full-bleed-page.tsx` | `FullBleedPage`: altura exata entre header e BottomNav (`--app-height − --shell-top − --shell-bottom − safe area`), quem rola é o filho. `ImmersiveScreen` (conversa, builder): esconde header/BottomNav, `fixed inset-0`, `height: var(--app-height)`, trava o scroll do body, `overscroll-behavior: none` |
| `MasterDetail` | `src/components/shell/master-detail.tsx` | seleção na query string com push (`?chat=`, `?contato=`, `?id=`); no celular uma tela por vez e o voltar fecha o detalhe; fora do celular lista e detalhe lado a lado, `aside` em `ResponsiveSheet` |
| `ShellSkeleton` / `RouteSkeleton`, `RouteScrollManager` | `src/components/shell/*`, `src/components/routing/RouteScrollManager.tsx` | silhueta da casca no boot; skeleton por `ROUTE_META.skeleton`; restauração de scroll por rota |

**Variáveis CSS da casca** (todas em `<html>`; padrões em `src/index.css`):

| Variável | Quem grava | Significado |
|---|---|---|
| `--shell-top` | `ShellProvider` | altura total do header fixo (com safe area e faixa de impersonação); 0 fora do celular |
| `--shell-bottom` | `ShellProvider` (ou a `BottomActionBar` registrada) | espaço ocupado embaixo SEM a safe area: barra de ação, senão BottomNav visível, senão 0 |
| `--app-height`, `--app-offset-top`, `--kb-inset` | `useVisualViewport({ publishCssVars: true })`, montado só no `ShellProvider` | viewport visível (segue o teclado do iOS) e altura do teclado |
| `--bottom-bar-h` | `BottomActionBar` (enquanto montada e visível) | altura total da barra, com safe area |
| `--impersonate-h` | `ImpersonateBanner` (`src/components/system/ImpersonateBanner.tsx`) | altura da faixa "visualizando como" no `/system` |

### 25.3 Kit de primitivas responsivas

Catálogo completo (props, comportamento desktop × celular, consumidores): `PLANO-MESTRE.md` §2.1. Kit genérico em
`src/components/ui/`, peças da casca em `src/components/shell/`, hooks em `src/hooks/`, utilitários em `src/lib/`.
Bancada viva: `/__dev/kit` (só DEV).

| Necessidade na tela | Use | Não use |
|---|---|---|
| Título, voltar, ações da página | `PageHeader` | `<h1>` + `flex justify-between` solto |
| Tabela | `DataList` (`<Table>` no desktop, linhas de 64–72px no celular) | `<Table>`/`<table>` crus |
| Formulário em modal | `ResponsiveDialog` (Drawer/tela cheia no celular, `dirty` pede "Descartar?") | `DialogContent` direto |
| Painel lateral | `ResponsiveSheet` | `Sheet` lateral fixo |
| Confirmação | `useConfirm()` / `ConfirmSwitch` (`ui/confirm-dialog.tsx`) | `confirm()`/`alert()`/`prompt()` (ESLint barra) |
| Ações de linha / ⋯ | `ActionMenu` (DropdownMenu no desktop, action sheet no celular) | ação só no hover |
| Abas | `ResponsiveTabs` | `TabsList grid grid-cols-N` |
| Busca e filtros | `FilterBar` + `FilterSheet` + `SearchInput` (16px) | `<input>` cru de 14px |
| KPIs / gráfico | `StatGrid` + `StatTile` / `ChartCard` | grade fixa de 4 colunas; informação só em tooltip |
| CTA de formulário longo | `BottomActionBar` (ou `Fab` para criar) | botão no fim da página escondido pelo teclado |
| Vazio / erro de tela | `EmptyState` / `StatusScreen` | texto solto |
| Página pública | `PublicShell` (gutter 16px, `pt-safe`/`pb-safe`) | `min-h-screen` próprio |
| Link interno em "nova aba" | `useOpenInApp()` (`navigate` no celular/PWA) ou `<Link>` | `target="_blank"` com rota interna (D15) |
| Exportar/compartilhar | `shareOrDownload`, `ShareButton`, `CopyField` | download direto no PWA iOS |
| Campos | `FormField` + `fieldPresets` + `masks` | `type`/`autoComplete`/`inputMode` à mão |
| Tela que não cabe no celular | `DesktopOnlyNotice` | esconder a rota |
| Botão-ícone | `IconButton` (`label` obrigatório) / `.hit-44` | ícone de 16px sem área de toque |
| Toast | `toast` do `sonner` (bottom-center no celular, 4 s; erro 6 s) | `use-toast` do shadcn (removido na 12.2) |

### 25.4 Regras M1–M3 (twMerge no `cn()`)

Valem para toda edição em `src/components/ui/*` e `src/components/shell/*` (origem: `PLANO-MESTRE.md` §2.2, lá
chamadas R1–R3).

| # | Regra | Por quê |
|---|---|---|
| M1 | **Adicionar limites na base é seguro.** Classes sem prefixo que só limitam (`max-h-*`, `max-w-*`, `overflow-y-auto`, `overscroll-contain`) são vencidas por qualquer override do consumidor. | O consumidor com `max-h-[90vh]`/`w-[95vw]`/`p-0` continua vencendo. |
| M2 | **Nunca acrescentar variante de largura mínima (`sm:`/`md:`/`lg:`) a propriedade que consumidores sobrescrevem** (padding, altura, largura, raio, fonte). Mudança só-celular usa `max-md:`, `touch:` ou `[@media(pointer:coarse)]:` e precisa ser aceitável mesmo com override sem prefixo. | `p-6` → `p-4 sm:p-6` no `CardContent` faria o `className="p-4"` de um consumidor perder o base e manter o `sm:p-6`: o desktop regride. Idem `h-10` → `h-11 md:h-10` no `Button`. |
| M3 | **Safe area na base por valor arbitrário dentro do utilitário** (`pt-[max(1.5rem,env(safe-area-inset-top,0px))]`), nunca com `.pt-safe*`/`.pb-safe*` do `index.css`. | Essas classes ficam fora de `@layer` e venceriam o `p-0` de um consumidor (sheet com header próprio). |

Observação: `cn()` (`src/lib/utils.ts`) usa `extendTailwindMerge` com o grupo `bg-brand-gradient`: `gradient-primary`
e `bg-destructive` no mesmo `cn()` agora se resolvem (o último vence). Mudança de comportamento listada para veto no
`RESUMO-EXECUTIVO.md`.

### 25.5 Padrões proibidos e guarda-corpos

Os dez padrões que explicavam ~80% dos defeitos no celular (PLANO-MESTRE §0) estão barrados em duas camadas.
Guarda-corpo não é proibição retroativa: o estado de 2026-10-21 é o baseline; só **piorar** falha.

**ESLint** (`eslint.config.js`, só `src/**/*.{ts,tsx}`):

| Regra | Proíbe | Exceções (comentadas no config) |
|---|---|---|
| `no-restricted-globals` + `no-restricted-properties` | `confirm`/`alert`/`prompt` globais e `window.`/`globalThis.`/`self.` | `email-editor/IframeCanvas.tsx` ("Recomeçar" no card de erro de render) e `email-editor/RawHtmlPreviewFrame.tsx` ("Cole o link" na barra do iframe); os dois só montam no desktop |
| `@typescript-eslint/no-restricted-imports` | `@/hooks/use-toast`, `@/components/ui/use-toast`; import **estático** de `exceljs`, `mjml-browser`, `jspdf`, `pptxgenjs` (`import type` e `await import()` passam) | `src/lib/excel.ts` (chamado só por `await import("@/lib/excel")`) e `email-editor/utils/mjmlRenderer.tsx` (renderizador canônico, só em chunks lazy) |

**Teste de baseline** (`src/test/guardrails.test.ts` + `src/test/guardrails-baseline.json`): varre `src/**/*.tsx`
(sem testes), conta por arquivo e falha se um arquivo aumentar ou se um arquivo novo aparecer com ocorrência.
Variante de breakpoint para cima (`sm:`…`2xl:`, `min-[…]:`) e elemento `hidden … lg:block` não contam; `max-md:` conta.

| Regra | Padrão | Baseline (arquivos / ocorrências) |
|---|---|---|
| `viewport-height` | `h-screen`, `min-h-screen`, `100vh`, `100dvh` fora da casca/kit/moldura pública (allowlist no teste) | 36 / 56 |
| `internal-target-blank` | `target="_blank"` na mesma tag de `href`/`to` começando com `/` | 2 / 3 |
| `hover-only-action` | elemento clicável (`button`/`a`/`Link`/`role="button"`/`onClick`) com `opacity-0` + `group-hover:opacity-100` sem `data-hover-only` e sem `touch:`/`max-md:opacity-100` | 4 / 5 |
| `grid-cols-unprefixed` | `grid-cols-3`…`grid-cols-12` sem prefixo | 24 / 39 |
| `fixed-width-px` | `w-[NNNpx]`/`min-w-[NNNpx]` ≥ 320px sem prefixo | 8 / 9 |
| `tiny-text` | `text-[8px]`…`text-[10px]` | 61 / 138 |

- `npm run guardrails` roda o teste; `npm run guardrails:update` regrava o baseline (só quando a contagem caiu ou com
  justificativa no PR). Quando algo cai, o teste avisa para regravar.
- Rede de segurança do hover (`src/index.css`, "Rede de segurança de hover"): sem hover real, `opacity-0` com revelação
  `group-hover:opacity-100` fica visível. Decoração que deve continuar escondida no toque recebe `data-hover-only`.
- Contêiner com rolagem horizontal intencional (carrossel, tabela) recebe `data-overflow-ok`: os detectores ignoram.

**Detector em desenvolvimento** (`src/dev/overflow-detector.ts`, montado em `src/main.tsx` só com `import.meta.env.DEV`
por import dinâmico): abaixo de 768px, quando `scrollWidth` passa da largura do viewport de layout
(`documentElement.clientWidth`), faz `console.warn` com a rota e até 5 elementos culpados. Checa na carga, a cada
navegação e a cada mudança do DOM (debounce de 400ms, um aviso por rota + culpados).

**`npm run check:overflow`** (`scripts/check-overflow.ts`, Playwright, 375x812 com toque): abre cada rota contra
`BASE_URL` (padrão `http://localhost:8080`, dev server já rodando) e mede overflow horizontal e quantidade de `<h1>`;
sai com código 1 se alguma rota estourar (`STRICT_H1=1` também falha por h1 ≠ 1). Sem sessão, só as rotas públicas
fixas; com `E2E_STORAGE_STATE` (arquivo gerado pelo Márcio com `npx playwright codegen --save-storage=…`, fora do
repositório, mesma origem do `BASE_URL`), também os paths de `NAV_ITEMS`/`ACCOUNT_NAV_ITEMS` e os `ROUTE_META` sem
parâmetro. Editores (`hideBottomNav`) ficam fora por padrão (`INCLUDE_EDITORS=1` inclui): abrir um editor pode
disparar autosave. O script não clica nem digita e bloqueia escrita REST (PATCH/PUT/DELETE e POST fora de
`/rest/v1/rpc/`). O Playwright não está no `package.json`: o script resolve `playwright`/`playwright-core` do projeto
ou `PLAYWRIGHT_MODULE`; para instalar sem mexer no lockfile, `npm i --no-save playwright-core@1.53.2` (usa o Chromium
do cache `~/Library/Caches/ms-playwright`).

### 25.6 PWA

| Item | Estado |
|---|---|
| Entrada do app instalado | `start_url: "/app?source=pwa"` (`public/manifest.webmanifest`) → `src/pages/AppEntry.tsx`: resolve a sessão e sai com `replace` (o `/app` nunca fica no histórico); logado vai à home (`getHomeRoute`; no celular, `/dashboard` "Hoje" quando há a feature); deslogado vai a `/login`. `?source=` vira a tag `app.launch_source` do Sentry |
| Fontes | Inter self-hosted em `public/fonts/*.woff2`, `@font-face` no `src/index.css` e `preload` do subset latin no `index.html` (sem Google Fonts no caminho crítico) |
| Status bar iOS | `black-translucent` (`index.html`): o conteúdo passa por baixo do notch e cada superfície usa `env(safe-area-inset-top)` (M3). O iOS congela as metas na instalação: quem já instalou precisa remover e adicionar o ícone de novo (texto ao cliente em `docs/mobile-redesign/PWA-PENDENTE-AVAL.md` §3.1) |
| Nova versão (D9) | build grava `dist/version.json` e `__APP_VERSION__` (plugin `kh-app-version` no `vite.config.ts`); `src/lib/pwa/app-version.ts` compara ao voltar ao app (máx. 1x/5 min) e a cada 30 min, e oferece "Atualizar" |
| Service worker | só push (`public/sw.js`, registrado por `registrarServiceWorker` no `main.tsx`); sem `fetch` e sem cache, então não interfere no stale-chunk recovery do `main.tsx` |
| **Pendente D8** | offline no SW (`fetch` só para `navigate`, rede primeiro + `offline.html`): patch pronto em `PWA-PENDENTE-AVAL.md` §1, aguarda aval |
| **Pendente D11** | host de produção (Vercel × Cloudflare Pages) e cabeçalhos de cache (`public/_headers` ou `vercel.json`): `PWA-PENDENTE-AVAL.md` §2 |

### 25.7 Validação com dados reais

O tenant "Escola do Fluxo" tem dados reais e é o mesmo usado na validação das telas. Regras para qualquer agente ou
pessoa validando no app:

1. **Exercitar até a confirmação e cancelar.** Não salvar/excluir/duplicar segmento, não exportar base, não
   recalcular RFM/cluster/lookalike/pipeline de analytics, não publicar post/grupo, não salvar conexão/agente do Chat,
   não disparar WhatsApp/e-mail, não mudar tag/transação/campanha/contato real. Registro "[TESTE]" só se
   imprescindível e removido no fim.
2. **Não abrir editor de registro real para "só olhar".** O builder de formulário grava ao abrir (autosave); o
   `check:overflow` exclui editores por isso.
3. **Um toque por ação irreversível.** Toque duplo em botão de disparo ("Recalcular agora") executou duas vezes;
   validação de botão de disparo para no diálogo de confirmação.
4. **Nenhum agente cria conta nem digita senha.** Sessão para automação só por `E2E_STORAGE_STATE` gerado pelo Márcio.
5. **Desktop é sentinela**: toda frente confere as sentinelas de 1280x800 (`PLANO-MESTRE.md` §5.3) além de 375x812.

Incidentes do programa (para não repetir): (a) contato regravado com os mesmos valores durante validação (só
`updated_at` mudou); (b) formulário real regravado pelo autosave ao abrir o builder; (c) "Recalcular agora" do
pipeline de analytics disparado por toque duplo. Detalhes em `docs/mobile-redesign/RESUMO-EXECUTIVO.md`.

---

# PARTE E — CHANGELOG

## [v7.28] — 2026-05-02
### Refactor Plan v1.1 — Cleanup pós-validação

**Cleanup dos documentos de refactor v1.0:**
- `docs/REFACTOR_PLAN.md` (duplicado na raiz) já tinha sido movido em `git mv` no commit `8b2e6f7` para `docs/refactor/PLAN.md`. Verificado no working tree: apenas 1 cópia existe.
- `docs/refactor/PLAN.md` §12: referência ARCHITECTURE.md v7.24 → v7.27.
- `docs/refactor/BACKLOG.md`:
  - R-1.0 nova (pré-requisito): definir whitelist antes da Fase 1.
  - R-1.4: naming de backup padronizado (`__pre_refactor_2026_05`).
  - R-3.4: critérios atualizados — `depends_on_fields` já existe e está populado em 52/78 segmentos (26 com array vazio recebem skip-filter).
- `docs/refactor/PROGRESS.md`:
  - D-2026-05-02-06: cadência ETL confirmada hourly (`5 * * * *`).
  - D-2026-05-02-07: `depends_on_fields` como fonte canônica.

**Nova seção §22.3 em ARCHITECTURE.md:**
- Whitelist canônica de dimensões que disparam `segmentation_input_version`
- Blacklist de dimensões que NÃO bumpam (predições, cosméticos, membership, timestamps internos)
- Categoria nova "Campos BULK_ONLY (R36)" para opportunity_* (não vivem em `contact_state`)
- Procedimento documentado de adição/remoção de dimensões + bump de `analytics_segmentation_contract_version`
- Snapshot de validação 2026-05-02: 78 segmentos ativos, 52 com depends_on_fields populado, 21 dimensões distintas em uso.

**Validações arquiteturais via Supabase MCP:**
- 21 dimensões em uso por segmentos ativos hoje (medido em `depends_on_fields`)
- `analytics.segments.depends_on_fields` populado em 52/78 segmentos ativos (26 com array vazio — sentinela ou pós D-SEG-10 sem re-derivar)
- Cron `etl-event-health-metrics` confirmado em `5 * * * *` (hourly)

**R-1.0 closes by this PR.** Próximo passo: iniciar Sprint 1 com R-1.1 (criar branch develop no Supabase) após aprovação explícita de Marcio. Sprint 1 unblocked.

---

## [v7.27] — 2026-05-02
### Refactor Plan v1.0 — Segmentation Pipeline

**Documentos de refactor publicados** em `docs/refactor/`:
- `PLAN.md` — visão arquitetural, 10 princípios não-negociáveis (P1-P10), 8 fases sequenciais (~6 sprints), critérios de sucesso mensuráveis via D-CRON-3 telemetry, plano de migração com gates de estabilidade.
- `PROGRESS.md` — savepoint operacional (estado, próximas 3 ações, bloqueadores, baseline metrics, decisões arquiteturais formalizadas D-2026-05-02-01..05, lições aprendidas L-001..L-003).
- `BACKLOG.md` — ~30 tarefas atômicas com IDs estáveis (R-1.1 a R-8.4), critérios de aceite, dependências entre tarefas. Granularidade micro nas Fases 1-3, macro nas 4-8.

**Mudança em ARCHITECTURE.md** (nova seção "Refactor em andamento (Maio 2026)" antes do §23 Roadmap):
- Link para os 3 documentos do refactor.
- Lista pública dos 10 princípios P1-P10.
- Regra: toda PR durante o refactor deve referenciar tarefa do BACKLOG (R-X.Y) e respeitar P1-P10.

**Sync workflow** atualizado (`.github/workflows/sync-architecture-docs.yml`):
- Trigger paths inclui `docs/refactor/**`.
- Constituição (ARCHITECTURE/SECURITY/DEPENDENCY_MAP) continua sincronizando para raiz de `kreatorshub-docs`.
- Documentos de refactor sincronizam para `kreatorshub-docs/refactor/` (subpasta espelhando D-2026-05-02-04).

**Status:** plano aprovado, Sprint 0 (setup) concluído, Sprint 1 inicia com R-1.1 (criar branch develop no Supabase) após aprovação de Marcio.

**Não inicia tarefas R-X.Y neste commit.** Apenas documentação publicada.

---

## [v7.26] — 2026-05-01
### Trigger enrichment — purchase_sequence_number

**`trigger_purchase_journey_event()`** enriquecido com `purchase_sequence_number` no `event_data` do bloco `purchase`. Valor computado via `COUNT(*) + 1` em `commerce.transactions` filtrado por `(tenant_id, party_id, status='paid')` — mesma transação SQL, zero delay, zero dependência do analytics pipeline.

**Design:** campo numérico genérico, não boolean. Suporta todos os operadores do Rule Engine: `=`, `>`, `<`, `>=`, `between`. Exemplos: primeira compra (`= 1`), recompra (`> 1`), terceira compra (`= 3`), entre N e M (`between 2 e 5`).

**Chave de agregação:** `party_id` (party-first, 100% cobertura em transactions pagas). Índice `idx_transactions_party_id` confirmado — 0.234ms por insert.

**Frontend:** campo `trigger.purchase_sequence_number` registrado em `TRIGGER_FIELD_CATEGORIES` (type: number, label: "Numero da compra (sequencia)").

**Padrão replicável (R53):** Qualquer trigger recorrente pode enriquecer `event_data` com sequence_number via COUNT na mesma transação. Candidatos futuros: `pending_sequence_number` (transaction_pending), `abandonment_sequence_number` (cart_abandonment), `submission_sequence_number` (form_submitted).

**Race condition:** READ COMMITTED permite que dois `paid` simultâneos do mesmo party vejam o mesmo COUNT — ambos recebem sequence=N. Aceito como best-effort (pagamento concorrente do mesmo party é raro).

---

## [v7.25] — 2026-05-01
### Platform Health Observability (D-CRON-3) + R52

**Contexto:** D-CRON-3 evoluiu de "alarme simples em `fallback_log`" para a primeira camada de observability arquitetural da plataforma. Princípio guia: instrumentação não pode ficar acoplada ao mecanismo observado — quando D-CRON-4 substituir o cron `segment-eval-fallback` por worker, view e handler permanecem inalterados.

**Migration `20260501100000_dcron3_observability.sql`:**
- `core.platform_health_whitelist` — whitelist auditável (reason CHECK, added_by, expires_at). Começa vazia. Tenant `00000000-...001` (Escola do Fluxo) NÃO entra: é tenant real.
- `analytics.event_health_metrics` — tabela particionada mensal (PK `tenant_id, metric_name, bucket_hour`), RLS service-only. Partições 2026-04 + 2026-05 criadas para warmup retroativo dos 7d iniciais de baseline.
- `analytics.emit_health_metric()` — RPC SECURITY DEFINER para UPSERT. Workers consomem.
- `analytics.platform_alert_state` — dedup state (last_tier, last_changed_at, last_alert_at).
- `analytics.v_event_driven_health` — tier classification per-tenant baseada em rolling 7d P50/P95/P99 do PRÓPRIO tenant, excluindo a hora atual. Tier `INSUFFICIENT_DATA` durante warmup (< 168 samples).
- `analytics.etl_event_health_metrics(window_hours)` — pulse banco-only (R50 categoria A). Agrega `fallback_log` + `contact_events` + `segment_eval_queue` em 5 métricas. Substituível por workers em D-CRON-4 sem mudar consumidores (R52).
- `analytics.event_health_metrics_partition_create_if_needed()` — DDL idempotente para rotação mensal.
- `analytics.event_health_metrics_purge_old(retention_days)` — DROP partition (90d retention).
- 3 novos pg_cron jobs categoria A registrados em §11: `etl-event-health-metrics` (5 * * * *), `event-health-metrics-partition-create` (30 2 25 * *), `event-health-metrics-purge-old` (30 4 * * *).

**Worker code:**
- `workers/shared/src/sentry.ts` — helper reutilizável. `initSentryWorker(opts)` no-op silencioso se `SENTRY_DSN_WORKERS` ausente — permite deploy do código antes do DSN. Adicionado `@sentry/node ^8.55.2` em deps de `@creator-hub/shared`. Re-exportado em `index.ts` + entrada em `exports` map.
- `workers/journeys/src/handlers/healthCheck.ts` — handler `runEventDrivenHealthCheck` (5min cadence). `shouldAlert` puro, testável. Lê apenas `v_event_driven_health` + `platform_alert_state`. Recovery silencioso (HEALTHY sustentado 30min).
- `workers/journeys/src/index.ts` — hook `lastHealthCheck` em `runSchedulerLoop` E `runCombinedLoop`. Sentry init no startup. Flush Sentry no shutdown.
- `workers/analytics/src/segment-eval-worker.ts` — `flushHealthMetricBatch` (60s) acumula `segment_membership_changes` em memória e emite via `emit_health_metric`. Custo zero no hot path.

**R52 nova:** Observability em camada própria desacoplada do mecanismo observado. View e handler nunca consultam tabelas operacionais — apenas `event_health_metrics`. Garante que substituir cron por worker (D-CRON-4) não invalide instrumentação. Precedente para futura observabilidade arquitetural.

**Testes (19 passing):**
- Pure fns: `isWorseTier`, `shouldAlert` (8 cenários — transições, dedup, intervalos).
- Integration: tier classification, multi-tenant isolation, recovery, WARN não emite, CRITICAL→ERROR não re-alerta.

**Baseline real coletado em Fase 1.b** (P50=8.613 hits/h, P95=26.420, P99=36.348 fallback_hits global) usado como sanity check da view; thresholds são per-tenant e bootstrap automático após 168h de coleta.

**Pré-requisito de produção:**
- Criar Sentry project `workers-railway` em org `creators-hub-ur`.
- Setar `SENTRY_DSN_WORKERS`, `SENTRY_ENVIRONMENT`, `RELEASE_SHA` em env vars Railway do `journeys-worker`.
- Sem isso: handler funciona normalmente, escreve em `platform_alert_state` + log, mas nada vai pro Sentry. **Não bloqueia deploy do código.**

**Pendências encadeadas:**
- D-CRON-4 (migrar `segment-eval-fallback` cron para worker) agora desbloqueado — D-CRON-3 forneceu observability + métrica `fallback_hits` que o substituto deve emitir.
- D-SEG-7 (event-driven audit) ganha visibility via `event_driven_hits` métrica.

---

## [v7.24] — 2026-04-08
### D28 Fase 1 — DROP customer_id em 6 tabelas zero-data

**Contexto:** diagnóstico de 2026-03-20 listou 17 tabelas/views ainda com coluna `customer_id`. Re-auditoria em 2026-04-08 identificou que 6 dessas tabelas tinham a coluna mas com **zero linhas populadas** — seguro dropar sem backfill nem migração de leitores.

**Tabelas afetadas (6 colunas dropadas):**
- `analytics.customer_cluster_assignments`
- `analytics.customer_cluster_subgroup_assignments`
- `analytics.customer_lookalike_scores`
- `core.eduzz_webhook_events`
- `eduzz.webhook_events`
- `crm.customer_badges`

**Procedimento:**
1. `SELECT COUNT(customer_id)` em cada tabela → todas retornaram 0.
2. `ALTER TABLE … DROP COLUMN IF EXISTS customer_id` nas 6 tabelas.
3. Re-query do `information_schema.columns` confirmou 0 ocorrências restantes nas 6.

**Inventário D28 restante (11 entradas):**
- **Base tables (8):** `commerce.transactions` (35K rows, 100% `party_id`), `commerce.archived_transactions` (7.9K), `journeys.journey_enrollments` (17, 99.8% `party_id`), `crm.opportunities` (22, 100% `party_id`), `crm.customer_tags` (7), `analytics.lifecycle_events` (9.4K, 5 órfãos), `analytics.segment_customers` (112), `eduzz.buyers` (46).
- **Views (3):** `analytics.v_segment_base`, `analytics.v_segment_leads_base`, `crm.vw_parties_unified`.
- **FKs ativas:** 5 → `crm.customers` (bridge table, 16.4K rows 100% party_id).
- **Funções:** 35 com "customer" no nome — auditoria obrigatória antes da Fase 3.

**Mudanças nos docs:**
- **D28** atualizada: Fase 1 ✅; restante e roadmap (Fase 2–4) detalhados.

**Próximas fases (não executadas nesta sprint):**
- **Fase 2:** tabelas com `party_id` 100% — `transactions`, `journey_enrollments`, `opportunities` ganham DROP de `customer_id` direto.
- **Fase 3:** audit das 35 funções com "customer" + drop das 5 FKs → `crm.customers`.
- **Fase 4:** DROP `crm.customers` + rebuild das 3 views (`v_segment_base`, `v_segment_leads_base`, `vw_parties_unified`).

---

## [v7.23] — 2026-05-01
### Sprint de Performance — RPCs, frontend e cache

**Contexto:** loading timeouts em tenants grandes (escola-do-fluxo, Sentry JAVASCRIPT-REACT-22/21/1S) escalaram para sprint dedicado de performance. Foco: matar full scans em RPCs quentes, eliminar polling agressivo, segmentar bundle, cachear o que é estável e arrumar fast-path do CRM condition. D29 era a dívida-mãe; ficou resolvida.

**Backend — RPCs novas:**
- `crm.get_parties_stats(p_tenant_id)` — stats globais separados da listagem paginada. Frontend cacheia com `staleTime` 5min, evita recalcular contagem a cada paginação.
- `analytics.get_transactions_chart_data(p_tenant_id, p_granularity, p_start_date, p_end_date)` — substitui query de 26K rows brutas por agregação por período (37 buckets, 22ms). Granularidade controlada server-side.

**Backend — RPCs otimizadas in-place** (rollbacks em `docs/rollback/`):
- `crm.list_parties` — subqueries correlacionadas → CTEs batch.
- `analytics.get_analytics_data` — 4 scans da tabela de transactions → 1 CTE `base_txn` reusada.
- `analytics.get_dashboard_data` — 3 lead CTEs colapsados em 1 `COUNT FILTER`. **534ms → 17ms**.

**Frontend:**
- React Query Persist com IndexedDB, TTL 24h, limpeza no logout.
- Connection heartbeat (4min, pausa quando tab inativa).
- Prefetch durante auth: dashboard, tags, features.
- Loading progressivo: Campaigns, Segmentation, Forms, Analytics.
- `AccessGate` com bypass em warm path (cache válido).
- Dynamic imports cortando **−924K** do bundle inicial.
- Vendor chunks dedicados: `@xyflow`, `@tiptap`, `zod`.
- `useTenantTags` — 6 query keys colapsados em 1.
- `usePartiesStats` — stats separados da listagem, cache 5min.
- Polling **1s → 30s** (redução -99.8% de requests em sessões idle).

**Worker / SQL — fix `has_open_opportunity`:**
- Worker in-memory não tem oportunidades carregadas → retorna optimistic (`true`).
- Filtragem definitiva no SQL path via subquery indexada em `eval_condition_v2`.
- **Não adicionou coluna em `contact_state`** (R36 + R51).

**Mudanças nos docs:**
- **R51** registrada: campos de domínio separado avaliam optimistic no worker, SQL path filtra de fato.
- **D29** marcada como ✅ Done com checklist completo do sprint.

**Pendências encadeadas:**
- D-CRON-3 (P2) segue como próximo fix calibrado (alarme sobre `analytics.fallback_log`).
- Padrão optimistic+subquery vira referência para futuros campos 1:N (tickets, custom relations).

---

## [v7.22] — 2026-05-01
### Auditoria pg_cron + R50 + D-CRON-1..4 + D-SEG-12

**Contexto:** investigação dupla pós-fix D-SEG-10. Q1 confirmou que NÃO existe rota oculta corrompendo `segment_parties` — a percepção de "BULK 4168 hoje" referia-se ao bulk de 30/04 19:10:52 UTC (Operação A pré-fix); pós-deploy, apenas 44 rows entraram via `apply_segment_membership_diff` per-party com `source` correto baseado em `contact_state.party_type`. Q2 auditou 14 pg_cron jobs ativos — 12 são housekeeping puro (categoria A), 2 violam §2.4 com loop bloqueante de tenants (`rfm-rolling-refresh`, `segment-eval-fallback`).

**Mudanças nos docs:**
- §11 expandida: tabela completa dos 14 crons com schedule, command, função SQL, categoria A/B/C/D, notas. Substitui a lista incompleta anterior.
- **R50** adicionada: pg_cron é APENAS housekeeping. Loop de tenants, lógica de negócio embutida, INSERT em tabelas de domínio, fanout dependente de volume — tudo proibido em SQL. Vai pra worker.
- **D-CRON-1** marcado como Done (auditoria + tabela já em §11).
- **D-CRON-2** (rfm-rolling-refresh → worker), **D-CRON-3** (instrumentar fallback_log), **D-CRON-4** (segment-eval-fallback → worker), **D-SEG-12-LOOKALIKE-DIRECT-WRITE** (`analyticsStorage.ts:527` migrar para SQL canonical) registrados.

**Pendências encadeadas:**
- D-CRON-3 (P2) é o próximo fix calibrado — alarme sobre `analytics.fallback_log` antes de qualquer migração.
- D-CRON-4 bloqueado por D-CRON-3 + D-SEG-7 (event-driven audit).
- D-SEG-12 é PR independente, pode ir junto com qualquer outro de cleanup do segmentation path.

**Próximas fases:**
- Fase 2 (backfill SQL canonicalizando shape A) e Fase 4 (trigger guardrail de R48) seguem planejadas mas não-iniciadas — aguardam 48h estável de v7.21 em produção.

---

## [v7.21] — 2026-04-30
### Hotfix segment builder + auditoria de membership

**Frontend (UI):**
- **Sintoma reportado:** segmentos antigos abriam vazios no builder; banner amarelo aparecia em loop.
- **Causa-raiz 1 (shape AST v2):** 46/52 segmentos manuais ativos tinham `rules_json_v2.root.children` com conditions diretas (shape B legado), enquanto o builder novo espera grupos (shape A "Blocos Paralelos"). 23 desses foram criados nativamente entre 10/03 e 01/04 — duas rotas de criação coexistiam.
- **Causa-raiz 2 (state-lock):** `SegmentBuilderV2` é uncontrolled — `useState(initialAST)` capturava o valor inicial vazio antes do useEffect de hidratação, ignorando o prop atualizado.
- **Fix 1 — adapter no load:** `normalizeAstV2ForUI` aplicado antes de `fromBackendAST`. Wraps loose conditions em grupos AND preservando `root.op`. Aliasing de fields/ops deprecated. Postura preserve > drop: fields desconhecidos viram conditions read-only com flag `__ui_locked` (strip antes de salvar). Apenas sentinelas (`all_leads`, `all_customers`) são dropadas.
- **Fix 2 — gate na page:** `SegmentationBuilder.tsx` não monta `<SegmentBuilderV2>` até `initialized === true`. Spinner inline durante hidratação. Banner de warnings no mesmo gate (não aparece antes do builder).
- **Fix 3 — banner só para warnings actionable:** apenas `field_dropped` e `field_locked` geram banner amarelo. `shape_wrapped`, `field_aliased`, `op_aliased`, `expanded_array_op` continuam logados internamente mas são silenciosos no UI (transformações cosméticas que preservam semântica).
- **Validação:** smoke test sobre 52 segmentos, 0 falhas. Suite de 30+ testes incluindo fixtures dos 7 segmentos críticos do banco. Build verde.

**Banco (operação A):**
- **Sintoma reportado:** segmento "Segmentação Amazônia" (3a487e5d) com 3 contatos no card mas 4.167 na lista; campanhas atingindo 3 errados em vez de 4.167 corretos.
- **Auditoria:** 22 de 52 segmentos manuais ativos com `segment_parties` divergente da avaliação fresh via `build_unified_where_clause_v2`. Top 3 com >700 contatos de excesso; "LEADS COM TELEFONE" sozinho com 9.153 contatos a mais do que a regra permite — violação direta de R37 (blast radius cap) e R38 (fail-closed).
- **Operação executada:** `refresh_segment_parties_unified` em massa nos 22 divergentes, ~13s total, zero erros. Snapshot pré-operação preservado em `analytics._segment_parties_audit_20260430` por 7 dias.
- **Resultado:** 22/22 com `segment_parties` = avaliação fresh = cache (`customer_count`/`lead_count`). Estado matematicamente correto.

**Pendências registradas:**
- D-SEG-3-CONTROLLED-COMPONENT (P3): `SegmentBuilderV2` deve virar controlled.
- D-SEG-4-INTEGRATION-TESTS (P3): cobertura de integração da page.
- D-SEG-5-MEMBERSHIP-DRIFT (P1): causa-raiz do drift entre segment_parties e regras não identificada. Operação A trata sintoma, não causa.
- D-SEG-6-LAST-CALC-LIES (P2): `last_calculated_at` é atualizado sem refresh real.
- D-SEG-7-EVENT-DRIVEN-AUDIT (P2): rota event-driven populou complemento semântico no Amazônia. Suspeita de regra invertida em path antigo.
- D-SEG-8-DAILY-AUDIT-JOB (P2): implementar job diário de auditoria + auto-refresh.
- D-SEG-9-LEGACY-TRIGGER (P2): `trg_refresh_segment_parties_on_change` chama `refresh_segment_parties` (legacy) em vez de `_unified` — possível parte da causa-raiz.
- D-RFM-LABELS (P3): RFM labels duplicados worker↔frontend.
- D-PRODUCT-FIRST-PURCHASE-UI (P3): campo válido no backend, exige UI nova.
- D-OPP-FIELDS-UI (P3): pickers UUID humanizados para campos de oportunidade.
- D-RECENCY-DAYS-UNIT (P4): adicionar suffix "dias" no input.
- D-CONVERSION-TEST-FIX (P4): 4 testes pré-existentes broken em `conversion.test.ts`.
- D-DATA-1-RFM-ACCENT (P4): acentuação inconsistente de labels rfm_segment em produção.
- D-SEG-2-BACKFILL (P2): canonicalizar todos rules_json_v2 para shape A no banco.

**Próximas fases planejadas:**
- Fase 2 (backfill SQL): após 48h estável em produção, canonicalizar shape A em todos os segmentos. Adapter `normalizeAstV2ForUI` vira no-op defensivo.
- Fase 3: eliminar `SegmentBuilder.tsx` antigo, `astV2ToBuilder.ts`, `conversion.test.ts`.
- Fase 4: guardrail estrutural — trigger BEFORE INSERT/UPDATE em `analytics.segments` validando shape canônico de `rules_json_v2`. R48 vira enforcement.
- Investigação D-SEG-5: diagnóstico do worker, do scheduler e do trigger event-driven para descobrir por que o drift acontece.

---

## [v7.20] — 2026-04-08
### Sprint 19 — Email Engagement Conditions (Segmentação + Jornadas)

**Handler fix:** `sendgrid:engagement_event` e `sendgrid:delivery_event` agora propagam `campaign_id`, `send_id`, `recipient_id` no INSERT de `email.email_events` via `resolveCampaignContext()` (1 query JOIN `campaign_recipients` → `campaign_sends`). Backfill aplicado: 75.2% opens e 68.0% clicks linkados (restante = transacionais/teste sem campaign — esperado).

**Novo resolver `email_engagement`:** Dual-mode (compileSQL para segmentação bulk + compileMemory para condition node). EXISTS subquery em `email.email_events` filtrado por `send_id` + `event_type`. Suporta operadores `is_true`, `is_false`, `url_contains`.

**Campos registrados:**
- `campaign_opened` — "Abriu campanha" (uuid[] send_ids, operator_family engagement)
- `campaign_clicked` — "Clicou em campanha" (uuid[] send_ids, + url_contains para filtro de URL)
- `last_email_opened_at` / `last_email_clicked_at` — expostos no frontend (já existiam no registry)

**Pre-enrichment no conditionEngine:** `extractEngagementSendIds()` scanneia AST, faz 1 query em `email.email_events`, injeta `__email_engagement` no contactState. Custo: 1 query extra apenas quando condição de engagement é usada.

**Frontend:** Categoria "Engajamento de email" no SegmentBuilderV2 com 4 campos. `useCampaignSends()` hook busca `campaign_sends` + `email_campaigns` JOIN. `CampaignSelectInput` via MultiSelectInput pattern (mesmo de bought_product). Operador `url_contains` renderiza input text adicional.

**Regra R47:** Campos de email engagement usam resolver `email_engagement` com EXISTS subquery — nunca materializar engagement data em `contact_state` (dados são per-campaign, não per-contact snapshot). Pre-enrichment no conditionEngine é o padrão para campos que dependem de dados relacionais no modo SINGLE.

## [v7.19] — 2026-03-30
### Cleanup sprint — G-010, D34, D37, D41
- **G-010:** revogados TRUNCATE/REFERENCES/TRIGGER de `authenticated` em 63 tabelas / 10 schemas
- **D34:** unique index `processing_jobs_coalesced_pending_uidx` agora exclui `project_traits` e `sync_field_definitions`
- **D37:** dead code v1 removido — `runEquivalenceGate.ts`, `equivalence.test.ts`, fallback legacy em `segmentEvaluator.ts`
- **D41:** `segment_eval_queue` TTL 24h→4h (projeção: 193MB→~50MB)

## [v7.17] — 2026-03-30
### D35 Done — Backfill de traits completo
- 8.816 traits projetados via SQL function temporária (text: 8.464, number: 402, date: 368, boolean: 3)
- 4 tenants processados: Escola do Fluxo (+8.694), Cone Island (+112), Thais Zacharias (+2), Instituto Socrates (+8)
- `segment_eval_queue`: 1.187 entries processados, pipeline drenou em minutos
- Function temporária `backfill_form_traits` dropada após uso
- Gap documentado: ~163K answers sem `field_definitions` (`syncFieldDefinitions` pendente)

## [v7.16] — 2026-03-26
### B-18 Done — Campos financeiros Eduzz
- Handler `eduzz:enrich_invoice` atualizado: popula `fee_value`, `net_gain`, `installments` em `commerce.transactions` a partir de `eduzz.invoices`
- Backfill aplicado: ~22K transações Eduzz existentes atualizadas (cobertura 99.9%)
- Edge function `integrations-worker` deployed (v86)
- B-18: ✅ Done

## [v7.18] — 2026-03-30
### Sprint 18 — Automações v2, Etapa 4: Novos Nodes (parcial)
- **If/Then/Else (Condition Engine no Canvas):** SegmentBuilderV2 integrado no NodeInspector via Dialog popup. 40+ campos de profile + 10 campos de trigger (`trigger.amount`, `trigger.product_name`, `trigger.status`, `trigger.utm_*`, `trigger.payment_method`, etc.) via `TRIGGER_FIELD_CATEGORIES`. Dual path: v2 (`rules_json_v2` via `toBackendAST()`) + legacy (`conditions[]`) com badge "Condição legada" e botão de upgrade. `FieldSelectV2` com Dialog + busca + grid 2 colunas por categoria. `configSummary` descritivo gerado do AST.
- **Cart abandonment trigger:** trigger SQL `on_cart_abandonment_journey_event` em `eduzz.cart_abandonments`. `event_data`: cart_id, product_ids, product_name, payment_method, UTMs, abandoned_at. Pipeline E2E: webhook → upsert cart → trigger → `journey_events` → `event_processing_jobs` → enrollment.
- **Transaction pending trigger:** `trigger_purchase_journey_event` estendido. `status='pending'` (e OLD != 'pending') gera `event_type='transaction_pending'`. INSERT direto como 'paid' gera apenas purchase. `event_data`: transaction_id, amount, product_id, product_name, payment_type.
- **Wait Until Date/Time:** nó wait expandido com 2 modos — `duration` (minutos/horas/dias) e `until_date` (data absoluta + timezone). Quando data já passou, 3 comportamentos via `if_passed`: `execute` (continua imediatamente), `skip_to_next_wait` (pula nós intermediários, registra em `journey_executions`), `exit_journey` (encerra enrollment). `convertLocalToUTC()` via Intl API nativa (Node 18+), sem dependência externa. `findNextWaitOrExit()` com safety limit 50 iterações.
- **Frontend:** EventEntryConfig expandido (6 event_types). WaitConfig com seletor de tipo, date/time picker, timezone selector (8 opções BR/US/EU), dropdown `if_passed`. ConditionConfig com Dialog popup + SegmentBuilderV2 compact.
- **Frequency guard default:** alterado de 16h para **0 (desabilitado)**. Opt-in via `EMAIL_FREQUENCY_GUARD_HOURS` env var no Railway.
- **semanticFieldRegistry fix:** 14 campos adicionados ao `workers/shared/src/segmentation/semanticFieldRegistry.ts` e `executionRegistry.ts` — campos financeiros (`total_revenue`, `frequency`, `monetary`) + 10 trigger fields (`trigger.amount`, `trigger.product_name`, `trigger.payment_type`, etc.). Sincronizado com `workers/analytics/src/segmentation/semanticFieldRegistry.ts` (8 campos financeiros). Fix identificado durante E2E testing — Condition Engine retornava `null` para campos não registrados.

## [v7.15] — 2026-03-26
### Sprint 17 — Automações v2, Etapa 3: Batch, Fairness & Performance
- **D20 ✅:** Batch claim com fairness — `claim_next_jobs(batch_size, job_types, tenant_cap)` com `ROW_NUMBER() OVER (PARTITION BY tenant_id)`. journeys-worker 10 jobs/poll, email-campaign 5 jobs/poll. Singular preservado.
- **Email paralelo:** `p-limit(10)` + rate limiter compartilhado. ~9→~50-90 emails/sec. `SEND_CONCURRENCY` configurável. Circuit breaker preservado.
- **R45 ✅:** Channel frequency guard — **default=0 (desabilitado, opt-in)** entre emails por contato (cross-journey). Skip com razão persistida em `journey_executions` (R42). `EMAIL_FREQUENCY_GUARD_HOURS` configurável. Index parcial `idx_campaign_recipients_frequency_guard`.
- **R46 ✅:** SLA separation verificada — zero overlap de job_types. journeys-worker (seconds SLA), email-campaign (minutes SLA).
- **D44 registrada:** Migrar template engine para SendGrid dynamic templates (P4).

## [v7.14] — 2026-03-26
### Automações v2, Etapa 2: Condition Engine + Versionamento
- **R40 ✅:** Condition Engine — Rule Engine v2 modo SINGLE. 8 arquivos extraídos para `@creator-hub/shared/segmentation/`. `evaluateConditionSingle()` com 40+ campos/operadores. Dual path: v2 (rules_json_v2) + legacy fallback.
- **R41 ✅:** Journey versions — `journey_versions` table, `publish_journey_version()` RPC, auto-publish on activation. `processJourneyNode` lê da versão congelada. Backfill: 8 jornadas + 39 enrollments vinculados.
- **R42 ✅:** Decisão persistida — `decision_result`, `decision_input_snapshot`, `resolved_edge_id` em `journey_executions`.
- **R43 ✅:** Trigger context — `context_json.trigger_payload` populado nos event handlers. `conditionEngine` flatten trigger fields como `trigger_<key>`.
- **Entry policy:** `entry_policy` (once/after_exit/always), `reentry_cooldown_hours`, `max_concurrent_enrollments`. `canEnterJourney()` v2 com fallback legacy.

## [v7.13] — 2026-03-25
### Sprint 14 — Automações v2, Etapa 1: Estabilizar Runtime
- **D18 ✅:** Fanout atômico — `sql.begin()` em 5 entry handlers (processEventJob, processBackfillJob, checkSegmentEntries, checkFormEntries, checkEventEntries). Zero enrollments órfãos possíveis.
- **D18b ✅:** send_email node — 3 INSERTs (campaign_sends → campaign_recipients → enqueue) atômicos via `sql.begin()`
- **D40 ✅:** create_opportunity guard — lookup parties → `crm.upsert_party_person` → skip gracioso. 10 jobs failed corrigidos.
- **D41 ✅ completo:** TTL cleanup em TODAS as filas de journeys: processing_jobs (pg_cron 4AM + scheduler 1h), event_processing_jobs (3AM), backfill_jobs (3AM)
- **Push-based segments:** `refresh_segment_parties_unified` detecta novos membros e enfileira `check_segment_entries` apenas para jornadas afetadas
- **Scheduler throttle:** `enqueue_journey_checks` skipa jornadas com check nos últimos 5 min. ~12.9K→~2K jobs/dia (85% redução)
- **Sweeper:** pg_cron `sweep-orphan-enrollments` (*/10 * * * *) — recuperou 3 órfãos históricos
- **R40-R46:** Regras arquiteturais para Automações v2 Etapa 2
- **D43 registrada:** `segment_entered` events não gerados pelo evaluator atual (P3)
- **Orphan sweeper:** `journeys.sweep_orphan_enrollments()` + pg_cron `sweep-orphan-enrollments` (*/10 * * * *) — recuperou 3 órfãos históricos
- **Push-based segments:** `refresh_segment_parties_unified` detecta novos membros e enfileira `check_segment_entries` apenas para jornadas afetadas. Zero push se zero novos.
- **Scheduler throttle:** `enqueue_journey_checks` skipa jornadas com check nos últimos 5 min. Projeção: ~12.9K/dia → ~2K/dia (fallback only) + push on-demand
- Wait node UI: métricas separadas em "Aguardando" (amber) + "Concluídos" (verde) no flow e na Performance dos Nós

## [v7.12] — 2026-03-24
### Correções baseadas em validação direta no banco
- **RPCs de lifecycle:** tabela na seção do ingestion-worker corrigida — `public.*` → `worker.*` (stubs confirmados removidos)
- **B-18:** colunas `fee_value`, `installments`, `net_gain` confirmadas em `commerce.transactions` — fix é no mapper, não no schema
- **D41:** `journeys.processing_jobs` atualizado — 90K rows / 66 MB (cresceu de 76K/55MB desde v7.11)

## [v7.11] — 2026-03-20

### Workers Railway — migração public.* → worker.* concluída
- `ingestion-worker`: já usava `worker.*` (confirmado, sem alteração)
- `historical-sync`: migrado — `public.claim_jobs/complete_job/fail_job/get_jobs_count_grouped/bulk_enqueue_jobs` → `worker.*` e `integrations.*`
- `email-sync`: migrado — `public.claim_jobs/complete_job/fail_job` → `worker.*`
- Funções de domínio eduzz: `public.eduzz_increment_sync_progress` → `eduzz.increment_sync_progress` (atualizada com lógica robusta), `eduzz.claim_invoices_for_enrichment` criada (SKIP LOCKED)
- Funções de domínio inbox: `public.upsert_message/batch_link_messages_to_crm/claim_active_accounts` → `inbox.*`
- `public.*` stubs de job management: **zero** — 6 stubs dropados + 4 stubs de domínio dropados
- `worker.*`: 10 funções canônicas (block_job, claim_jobs, claim_next_jobs, complete_job, enqueue_job, enqueue_webhook_job, fail_job, get_jobs_count_grouped, log_event, receive_webhook)

### TTL/Purge — integrations (R20 cumprida)
- `integrations.purge_old_jobs()` criada (retention: success=7d, terminal=30d; deleta `job_errors` dependentes)
- `integrations.purge_old_webhook_events()` criada (retention: processed=7d, failed=30d; NOT EXISTS guard para FK)
- pg_cron `purge-integrations-jobs` ativo (30 3 * * *)
- pg_cron `purge-integrations-webhook-events` ativo (45 3 * * *)
- Purge inicial: 10.722 jobs + 1.609 webhook_events removidos (11.480→758 jobs, 1.941→332 events)

### D5 — SendGrid pipeline unificado ✅
- `sendgrid-webhook` (inline, legado): URL migrada, não recebe mais tráfego
- `webhooks-sendgrid` Edge Fn ativa: O(1) — recebe → salva raw → enfileira job → 200 (verificação ECDSA P-256)
- `integrations-worker` v81: `registerSendgridHandlers` ativado
- Pipeline: SendGrid → `webhooks-sendgrid` → `integrations.jobs` (provider='sendgrid') → `integrations-worker` → `email.*`
- Handlers: `sendgrid:delivery_event` (bounce retry + supressão adaptativa), `sendgrid:engagement_event` (open/click tracking), `sendgrid:suppression_event` (unsubscribe + spam report)
- URL webhook migrada para `https://core.kreatorshub.com.br/functions/v1/webhooks-sendgrid` em 5 subusers via action `migrate_webhooks`
- `domain-delegation`: URL hardcoded atualizada + auth fix (JWT decode fallback para service_role)

### Itens auditados e fechados
- D11 (refreshFeatures roles revogados): ✅ zero órfãos em contact_state — refreshFeatures recalcula do zero
- D41 (queue audit): ✅ parcial — integrations.jobs e webhook_events com purge; journeys.processing_jobs pendente
- B-04 (webhooks eduzz orphaned 23–25/02): ✅ sem reprocessamento — transações já existiam via historical sync
- B-05 (segment_parties não populado): ✅ trigger `trg_queue_segment_refresh` já existia e funcionava

## [v7.9] — 2026-03-20
### Auditoria Fase 2 — D-series consolidação
- D26: confirmado completo — trigger `trg_record_email_engagement` + 4 colunas em `contact_state`
- D32: confirmado completo — `evaluation_mode` populado automaticamente, 70/70 = `event_driven`
- D28: atualizado com inventário factual — 17 tabelas/views com `customer_id` em 6 schemas
- D41: queue audit — 3 filas sem purge (`journeys` 76K, `integrations` 10K+1.9K), `segment_eval_queue` TTL 24h→4h
- D42: DROP tabelas Eduzz legadas (bloqueado por Sprint Security)
- Eduzz webhooks confirmados ativos (último evento: 2026-03-20 20:12)

## [v7.8] — 2026-03-20
### Auditoria da base de conhecimento
- D33: confirmado completo (Sprint 12). Tipo `rule_based` renomeado para `manual`. 42/42 com v2.
- D24: cancelado — `segment_customers` é ativa para clusters/lookalikes (~112 rows, 3 writers, 2 readers)
- D37: dead code confirmado — `build_segment_where_clause` dropada do banco, fallback v1 no evaluator chamaria função inexistente
- IC-5: seção 13 (CRM) atualizada — referências a `sync_opportunity_summary` e colunas dropadas removidas
- Seção 13 alinhada com R36 (`opportunity` via EXISTS, não denormalizado em `contact_state`)

## [v7.7] — 2026-03-19
### Sprint 12 — Documentação final + D40
- D40: party_id null em enrollments de formulário (P1 — investigar checkFormEntries)
- Fix: duplicate_campaign RPC — SECURITY DEFINER + assert_tenant_access + qualified extensions.digest()

## [v7.6] — 2026-03-20
### Journey Node Skip + Docker Cache Fix
- R39: send_email node skip deve retornar `success: false` + `waitUntil`, não `success: true`
- Status `automation` é válido para envio em contexto de journey
- BUILD_VERSION marker adicionado ao journey worker para deploy verification
- Lição: Docker layer cache pode servir código antigo — sempre ter marker de versão no entrypoint

## [v7.5] — 2026-03-19
### Campaign Safety Guards
- Guard 1: colunas de rastreabilidade em `campaign_sends` (`recipient_source`, `segment_ids`, `segment_parties_snapshot`, `recipient_count_at_send`)
- Guard 2: blast radius cap (ratio > 1.1 = abort), confirmação "Todos" com digitação obrigatória, hard cap 50K
- R38: implementação completa em 4 pontos (CampaignSend, generate-export, checkSegmentEntries worker, check-journey-entries Edge Function)
- D38: alerta pós-envio (P2 — pendente)
- D39: backend validation RPC (P2 — pendente)

## [v7.4] — 2026-03-19
### Segmentation Engine — Oportunidades
- Implementado resolver `opportunity_exists` com 10 campos via EXISTS subquery em `crm.opportunities`
- Removidas colunas órfãs `has_open_opportunity` e `open_opportunity_count` de `analytics.contact_state`
- Criado índice `idx_opportunities_party_tenant_status` em `crm.opportunities`
- R36 expandido: campos 1:N nunca denormalizados em contact_state, modo BULK_ONLY documentado, lista completa de 10 campos implementados


## Platform Billing — cobrança da plataforma aos tenants (v9.3)

A plataforma cobra seus tenants pelo MESMO Asaas usado pelo tenant especial **Kreators Platform** no módulo de checkout (`commerce.asaas_config` desse tenant; assinaturas criadas via painel de ofertas). O módulo Platform Billing NÃO cria cobranças — apenas exibe e permite pagar as cobranças existentes no Asaas.

**Contrato:**
- `core.tenant_billing` (tenant_id PK → asaas_customer_id): vínculo explícito tenant↔cliente Asaas. RLS ligada, zero policies/grants — acesso somente via edge fn com service role.
- Edge fn `supabase/functions/platform-billing/index.ts` (verify_jwt=false; auth por ação):
  - `admin_search_asaas_customer` / `admin_link_customer` / `admin_unlink_customer` — exige `core.system_admins`. Busca local-first em `commerce.checkout_orders` do tenant plataforma, fallback API Asaas.
  - `generate_share_link` — system admin OU owner/admin do tenant. Token HMAC-SHA256 `payloadB64.sigB64`, payload `{v:1, t:tenant_id, iat, exp}` (default 30d, máx 90d), secret `PLATFORM_BILLING_HMAC_SECRET` (lista rotacionável). Diferente do preference center, `exp` é OBRIGATÓRIO.
  - `get_billing` / `get_payment_artifacts` — token HMAC (página pública `/cobranca/:token`) ou JWT de owner/admin (página `/admin/account`). Dados live da API Asaas (`GET /v3/payments?customer=`, cap 300); PIX/boleto sob demanda; guard IDOR: `payment.customer === asaas_customer_id` vinculado.
- Env: `PLATFORM_TENANT_ID` (tenant Kreators Platform), `PLATFORM_BILLING_HMAC_SECRET`. Credencial Asaas vem do `asaas_config` do tenant plataforma — NUNCA de secret separado, NUNCA do asaas_config de outro tenant.

**Regra:** `commerce.asaas_config` dos demais tenants é do checkout DELES (eles cobrando os clientes deles). Platform Billing só lê o config do `PLATFORM_TENANT_ID`.

**Dívida técnica:** tokens de cobrança são stateless — sem revogação individual antes do `exp`; rotação do secret revoga todos. Desvincular o tenant (`is_active=false`) também invalida os links (o `get_billing` deixa de resolver).


### Minha conta — Visão geral (`api.get_account_overview`, 2026-10-28)

- `api.get_account_overview(p_tenant_id) → jsonb` (migration `20261028120000_account_overview_rpc`, aplicada) — `{tenant{id,name,slug,created_at}, plan{code,name,description,is_custom,started_at,expires_at}|null, limits{users,contacts,emails_monthly,emails_daily}, users{total, by_role{owner,admin,member,viewer}, pending_invitations}, emails{month{since,sends,sent,delivered}, last_30d{…}, total{…,first_send_at}}, features{active}}`. Consumidor: `src/lib/accountOverview.ts` → `AccountBilling.tsx` (aba Visão geral).
- Regras: plano = `core.tenant_plans` vigente mais recente por `started_at` (join `core.plans`); `limits.users`/`contacts` sempre `null` até existir tier com limites (ponto de plugar comentado no SQL); `emails_*` = `email.tenant_config.monthly_send_limit`/`daily_send_limit`; usuários sem system admins (mesmo critério de `core.get_tenant_team_members`); emails = `email.campaign_sends` com o filtro de `api.get_tenant_consent_summary` (sem teste, sem `cancelled/scheduled/pending/draft`, data `coalesce(started_at, created_at)`), mês civil em America/Sao_Paulo, uma passada só; features = `core.tenant_features` não expiradas.
- Segurança: `SECURITY DEFINER`, `search_path=''`, `app_auth.assert_tenant_access` (R56) + `core.is_tenant_admin` (owner/admin ou system admin; senão `42501`). EXECUTE só `authenticated`/`service_role`.

## Fila de jornadas — incidente de 22–28/10/2026

**D20 reaberto e mitigado parcialmente.** O fairness anterior não protegia candidatos de prioridade menor. A Fase 0 (29/10, só banco e compatível com o worker atual) garante:
- portão de pré-condições antes de enfileirar;
- espera como estado na inscrição;
- pausa real;
- anti-inanição no claim.

O isolamento completo entre tenants é a Fase 1, com o desenho aprovado e ainda não implementado:
- raias por classe de trabalho;
- revezamento justo por tenant;
- rota de excedente e contrapressão nos produtores;
- dedupe com índice UNIQUE;
- teto de tentativas com fila de falhas definitivas;
- alertas.

Contratos, testes, rollback e o plano completo estão em [JOURNEY_ISOLATION.md](JOURNEY_ISOLATION.md).

**Dívida operacional:** o Railway publica todos os workers a cada push em `main`. Um push de worker que dependa de um objeto de banco ainda não aplicado quebra produção; isso aconteceu em 29/10, entre 19:10 e 19:27 UTC. Antes de cada push de worker, confira que esses objetos já existem.
