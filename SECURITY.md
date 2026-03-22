# KreatorsHub — Security Architecture & Compliance Roadmap

## Estado: Sprint 5 — Completo ✅ | Auditoria factual: 2026-03-20
## Última atualização: 2026-03-20
## Versão: 1.5

---

# ESTADO ATUAL E PRÓXIMOS PASSOS

> **Última atualização:** 2026-03-20 | **Versão:** 1.5
>
> Esta seção é o ponto de entrada para continuidade entre sessões. Leia aqui primeiro.

### Sprints Concluídos

| Sprint | Escopo | Métricas-chave |
|--------|--------|----------------|
| Sprint 0 | Diagnóstico completo | 6 diagnósticos, inventário de 19 schemas |
| Sprint 1 | Hardening fundacional | 469 grants revogados, 27 funções corrigidas, schema `app_auth` criado |
| Sprint 2 | RLS sistemático | 66 tabelas protegidas, 267 políticas, 13 índices |
| Sprint 3 | SECURITY DEFINER audit | 69 funções migradas DEFINER→INVOKER, 0 sem search_path |
| Sprint 5 | Audit trail | 11 triggers, 7 funções, tabela `audit.trail` com RLS |

### Sprint em Andamento

**Sprint 4 — Secrets Management** (⏸️ pausado — requer mudanças em código)

| Fase | Status | Descrição |
|------|--------|-----------|
| Fase 1 | ✅ | Diagnóstico: 20 colunas, 7 tabelas, 4 tenants |
| Fase 2 | ⏸️ | Infraestrutura: schema `secrets`, funções encrypt/decrypt |
| Fase 3a | ⏸️ | Migrar `email.tenant_config` |
| Fase 3b | ⏸️ | Migrar `integrations.accounts` |
| Fase 3c | ⏸️ | Consolidar JSONB config |
| Fase 3d | ⏸️ | DROP tabelas legadas |
| Fase 4 | ⏸️ | Validação |

**Bloqueadores:**
1. Requer mudanças em código (workers, Edge Functions)
2. Decisão pendente: campo criptografado (pgcrypto) vs Vault direto vs híbrido

**Dependências de código:**
- `email-campaign-worker` — usar `secrets.decrypt_value()` para SendGrid API key
- `historical-sync-worker` — migrar de `eduzz.integrations` para `integrations.accounts`
- Edge Functions — `sendgrid-webhook`, `eduzz-receiver-webhook`

### Decisões Pendentes

| # | Decisão | Opções | Impacto |
|---|---------|--------|---------|
| D-SEC-1 | Estratégia de criptografia | A) Vault direto, B) Campo criptografado, C) Híbrido | Sprint 4 |
| D-SEC-2 | Ordem de migração | `email.tenant_config` primeiro vs `integrations.accounts` | Sprint 4 |
| D-SEC-3 | Timeline para DROP tabelas legadas | `eduzz.integrations`, `core.eduzz_integrations` | Sprint 4 |

### Inventário de Credenciais (Sprint 4 Fase 1)

| Tabela | Colunas sensíveis | Rows | Status |
|--------|-------------------|------|--------|
| `commerce.asaas_config` | api_key_encrypted, webhook_token | 1 | ✅ api_key ok |
| `core.eduzz_integrations` | api_key, webhook_secret | 1 | ❌ LEGADO — planejar DROP |
| `eduzz.integrations` | 6 colunas | 4 | ❌ LEGADO — planejar DROP |
| `email.tenant_config` | sendgrid_subuser_api_key, webhook_secret | 4 | ❌ Texto plano |
| `integrations.accounts` | access_token, refresh_token, client_secret | 8 | ❌ Texto plano |
| `whatsapp.instances` | webhook_secret | 1 | ⚠️ Parcial |

**Descoberta:** `integrations.accounts.config` (JSONB) também contém secrets duplicados.

### Gaps Resolvidos

| Gap | Descrição | Sprint |
|-----|-----------|--------|
| G-001 | Tabelas multi-tenant sem RLS | ✅ Sprint 2 |
| G-002 | DEFINER sem search_path | ✅ Sprint 1 + 3 |
| G-005 | Audit trail não existe | ✅ Sprint 5 |
| G-006 | Sem testes cross-tenant | ✅ Sprint 2 |
| G-007 | Views sem security_invoker | ✅ Sprint 1 |

### Gaps Pendentes

| Gap | Descrição | Sprint Planejado |
|-----|-----------|------------------|
| G-003 | Credenciais em texto plano (20 colunas) | Sprint 4 (pausado) |
| G-004 | MFA não enforced | Sprint 8 |
| G-008 | Sem SIEM integration | Sprint 12 |
| G-009 | Sem incident runbook | Sprint 16 |
| G-012 | `authenticated` com TRUNCATE em schemas de produção (commerce, community, content, core, journeys). TRUNCATE bypassa RLS. Resquício de `GRANT ALL` pré-Sprint 1. | Sprint 6 (P1) |
| G-013 | 10 funções SECURITY DEFINER sem `search_path` — todas de segmentação (analytics e api): `get_segments_with_live_count`, `process_segment_refresh_queue`, `queue_segment_for_refresh`, `queue_stale_segments`, `trg_queue_segment_refresh`, `trg_refresh_segment_parties_on_change`, `detect_segment_entity_type`, `get_segment_preview`, `refresh_all_segments`. Criadas pós-Sprint 3. | Sprint 6 (P3) |

### Snapshot de Segurança (2026-03-20)

| Métrica | Valor | Tendência |
|---------|-------|-----------|
| Tabelas com RLS | 227 | ✅ Estável |
| Tabelas multi-tenant sem RLS | 2 (fila + cache) | ✅ Aceitas |
| SECURITY DEFINER functions | 364 | ⚠️ 10 sem search_path (G-013) |
| Audit triggers (tabelas × operações) | 33 (11 tabelas) | ⚠️ Faltam: checkout_offers, checkout_orders, opportunities, eduzz.integrations |
| Workers com credencial compartilhada | 8/8 | ❌ Todos mesma DATABASE_URL |
| Credenciais em texto plano | ~20 colunas / 7 tabelas | ❌ Sprint 4 pausado |

### Roadmap Futuro

| Sprint | Escopo | Complexidade | Pré-requisito |
|--------|--------|--------------|---------------|
| Sprint 4 completo | Migrar credenciais para Vault/encrypted | 🟠 SQL + código | Decisões D-SEC-1/2/3 |
| Sprint 6 | Revogar grants excessivos (G-012) + Schema `api` audit + DEFINER search_path fix (G-013) | 🟡 Baixo-Médio | — |
| Sprint 7 | Hardening final Fase 1 | 🟡 Baixo | — |
| Sprint 8-9 | SSO/SAML, MFA enforcement | 🔴 Enterprise | — |
| Sprint 10-11 | Field encryption, Vault centralizado | 🔴 Enterprise | Sprint 4 |
| Sprint 12-13 | Audit exportável, SIEM | 🟠 Médio | Sprint 5 |
| Sprint 14-17 | Rate limiting, compliance program | 🟠 Médio | — |
| Sprint 18+ | Pentest, SOC 2 Type II | 🔴 Externo | Todos anteriores |

### Para Continuar

1. Ler `docs/SECURITY.md` no repo (versão canônica)
2. Esta seção tem o estado atual — começar aqui
3. Sprint 4 é o próximo — decidir estratégia de criptografia primeiro (D-SEC-1)
4. Migrations de sprints anteriores estão em `supabase/migrations/`

---

# PARTE A — VISÃO E PRINCÍPIOS

## 1. Objetivo

Este documento define a arquitetura de segurança, controles de compliance, e roadmap de hardening do KreatorsHub. É o guia de referência para:

- Decisões de segurança técnica
- Preparação para SOC 2 Type II, ISO 27001, HIPAA
- Auditoria e due diligence de clientes enterprise
- Onboarding de novos desenvolvedores

**Escopo:** Plataforma SaaS multi-tenant para infoprodutores brasileiros. Stack: Supabase (PostgreSQL), Edge Functions, Railway workers (Node.js/TypeScript), React frontend.

---

## 2. Princípios de Segurança (Não Negociáveis)

### 2.1 Isolamento Multi-tenant por Design

> **Regra:** Nenhum usuário do tenant A pode ver, modificar ou inferir dados do tenant B.

- `tenant_id` é coluna obrigatória em toda tabela multi-tenant
- RLS é habilitado em toda tabela exposta via PostgREST
- JWT carrega `active_tenant_id` — policies validam contra ele
- Testes automatizados de cross-tenant leakage são obrigatórios

### 2.2 Mínimo Privilégio

> **Regra:** Cada componente tem apenas os privilégios necessários para sua função.

- Frontend usa credencial `anon` + RLS (nunca `service_role`)
- Workers usam credenciais específicas, não compartilhadas
- SECURITY DEFINER é exceção aprovada, não default
- Schemas expostos são minimizados

### 2.3 Defense in Depth

> **Regra:** Segurança não depende de uma única camada.

- RLS no banco (última linha de defesa)
- Validação no backend (business logic)
- Validação no frontend (UX)
- Rate limiting em todas as camadas
- Auditoria independente

### 2.4 Auditabilidade Total

> **Regra:** Toda ação relevante pode ser rastreada até origem, autor e timestamp.

- Mudanças em dados sensíveis geram registro de auditoria
- Acessos administrativos são logados
- Logs são imutáveis e têm retention definida
- Correlação possível entre camadas (request_id, tenant_id, actor_id)

### 2.5 Fail Secure

> **Regra:** Em caso de dúvida ou erro, negar acesso.

- Policy ausente = acesso negado (não acesso liberado)
- Token inválido = request rejeitado
- Erro de autorização = não mostrar dados parciais

---

## 3. Modelo de Ameaças

### 3.1 Ameaças Externas

| Ameaça | Mitigação | Status |
|--------|-----------|--------|
| SQL Injection | Prepared statements, PostgREST, RLS | ✅ |
| XSS | React escaping, CSP headers | ⚠️ Verificar CSP |
| CSRF | SameSite cookies, CORS | ⚠️ Verificar |
| Credential stuffing | Rate limiting, MFA | ❌ MFA não enforced |
| Account takeover | MFA, session management | ❌ MFA não enforced |
| DDoS | Cloudflare, rate limiting | ⚠️ Parcial |

### 3.2 Ameaças Internas

| Ameaça | Mitigação | Status |
|--------|-----------|--------|
| Dev acessa dados de tenant | RLS, audit logs | ⚠️ Audit parcial |
| Credencial vazada | Rotação, Vault | ❌ Não estruturado |
| Insider malicioso | Least privilege, audit | ⚠️ Parcial |
| Erro de código expõe dados | Code review, testes | ⚠️ Sem testes de segurança |

### 3.3 Ameaças de Supply Chain

| Ameaça | Mitigação | Status |
|--------|-----------|--------|
| Dependência comprometida | Dependency scanning | ❌ Não existe |
| Provider comprometido | Vendor assessment | ❌ Não existe |
| Supabase breach | Encryption, backups | ⚠️ Backup existe |

---

## 4. Frameworks de Compliance

### 4.1 SOC 2 Type II

**Status:** Não iniciado
**Target:** Q4 2027

| Trust Service Criteria | Relevância | Status |
|------------------------|------------|--------|
| Security | Alta | Sprint 1-5 |
| Availability | Alta | Sprint 5+ |
| Processing Integrity | Média | Sprint 6+ |
| Confidentiality | Alta | Sprint 1-5 |
| Privacy | Alta | Sprint 6+ |

### 4.2 ISO 27001

**Status:** Não iniciado
**Target:** 2027

| Área | Status |
|------|--------|
| ISMS (Information Security Management System) | ❌ |
| Risk Assessment | ❌ |
| Access Control (A.9) | ⚠️ Parcial |
| Cryptography (A.10) | ⚠️ Parcial |
| Operations Security (A.12) | ❌ |
| Communications Security (A.13) | ⚠️ Parcial |

### 4.3 HIPAA (se aplicável)

**Status:** Não planejado (avaliar se necessário para mercado US)

---

# PARTE B — ARQUITETURA DE SEGURANÇA

## 5. Identity & Access Management

### 5.1 Modelo de JWT e Claims

**Status:** ⚠️ A definir formalmente

**Estrutura alvo:**
```json
{
  "sub": "user_uuid",
  "aud": "authenticated",
  "role": "authenticated",
  "app_metadata": {
    "active_tenant_id": "tenant_uuid",
    "membership_id": "tenant_members_uuid",
    "role": "owner|admin|member|viewer",
    "perm_version": 1
  }
}
```

**Regras:**
- `active_tenant_id` é obrigatório para operações multi-tenant
- `role` define permissões base
- `perm_version` incrementa quando permissões mudam (força refresh)
- Claims grandes (lista de tenants, permissões granulares) NÃO vão no JWT

### 5.2 Helpers Centralizados

**Status:** ✅ Implementado — Sprint 1 (2026-03-10)

**Schema:** `app_auth` — 11 funções (todas SECURITY INVOKER + `search_path = ''`)

```sql
-- Funções helper para policies (Sprint 2 usa em todas as novas policies)
app_auth.active_tenant_id() → uuid        -- tenant_id do JWT
app_auth.current_user_id() → uuid         -- auth.uid() wrapper
app_auth.current_role() → text            -- owner/admin/member/viewer
app_auth.membership_id() → uuid           -- membership_id do JWT
app_auth.is_owner() → boolean
app_auth.is_admin_or_above() → boolean
app_auth.is_member_or_above() → boolean
app_auth.is_authenticated() → boolean
app_auth.is_system_admin() → boolean      -- CUIDADO: cross-tenant
app_auth.is_aal2() → boolean             -- MFA step-up
app_auth.perm_version() → integer         -- cache busting
```

**Grants:** `authenticated` (all), `anon` (só `is_authenticated`), `service_role` (all)

### 5.3 Roles e Permissões

| Role | Descrição | Capabilities |
|------|-----------|--------------|
| `owner` | Dono do tenant | Tudo |
| `admin` | Administrador | Tudo exceto billing/delete tenant |
| `member` | Membro padrão | CRUD operacional |
| `viewer` | Somente leitura | Visualização |

### 5.4 Step-up Auth / MFA

**Status:** ❌ Não implementado

**Ações que exigirão MFA/step-up:**
- Alteração de billing
- Export massivo de dados
- Acesso a audit logs
- Impersonation/support mode
- Alteração de permissões

### 5.5 SSO/SAML

**Status:** ❌ Não implementado — Fase 2

---

## 6. Data Protection

### 6.1 Isolamento Multi-tenant (RLS)

**Status:** ✅ Auditado 2026-03-20 — 227 tabelas com RLS, 2 exceções aceitas.

**Exceções aceitas:**
- `analytics.segment_refresh_queue` — fila interna, sem dados sensíveis
- `typeform.themes_cache` — cache de temas, read-only

Ambas não são expostas via PostgREST.

**Padrão de policy (Sprint 2):**
```sql
-- Tabelas tenant-scoped — authenticated
CREATE POLICY "tenant_isolation_select" ON schema.table
  FOR SELECT USING (tenant_id = app_auth.active_tenant_id());

CREATE POLICY "tenant_isolation_insert" ON schema.table
  FOR INSERT WITH CHECK (tenant_id = app_auth.active_tenant_id());

CREATE POLICY "tenant_isolation_update" ON schema.table
  FOR UPDATE USING (tenant_id = app_auth.active_tenant_id());

CREATE POLICY "tenant_isolation_delete" ON schema.table
  FOR DELETE USING (tenant_id = app_auth.active_tenant_id());

-- service_role bypass (workers)
CREATE POLICY "service_role_all" ON schema.table
  TO service_role USING (true) WITH CHECK (true);
```

**Coverage atual:** Ver seção 11 (Inventário)

**Sprint 2 — Tabelas alvo (66 sem RLS):**

| Schema | Tabelas | Severidade |
|--------|---------|------------|
| analytics | 57 (9 principais + 43 partições contact_events + 8 partições contact_product_stats) | 🔴 CRÍTICO |
| journeys | 3 (backfill_jobs, event_processing_jobs, processing_jobs) | 🟠 ALTO |
| crm | 1 (opportunity_activities_archive) | 🟠 ALTO |
| integrations | 1 (sync_schedules) | 🟠 ALTO |
| sympla | 4 (events, orders, participants, sync_cursors) | 🟡 MÉDIO |

**Nota sobre partições:**
- **Policies** na tabela pai **são herdadas** pelas partições automaticamente (não precisam ser criadas em cada partição)
- **`relrowsecurity` NÃO propaga** — cada partição precisa de `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` explícito
- `contact_events` (41 partições) e `contact_product_stats` (8 partições) foram habilitadas individualmente na migration `analytics_partitions_rls`

### 6.2 Classificação de Dados

| Classificação | Exemplos | Controles |
|---------------|----------|-----------|
| **Crítico** | Credenciais, tokens OAuth | Vault, encryption, audit |
| **Sensível** | CPF, email, telefone, financeiro | RLS, audit, retention |
| **Interno** | Configurações, metadata | RLS |
| **Público** | Documentação, termos | Nenhum especial |

### 6.3 Criptografia

| Camada | Status | Método |
|--------|--------|--------|
| Transit | ✅ | TLS 1.2+ (Supabase) |
| At rest | ✅ | AES-256 (Supabase) |
| Field-level | ❌ | Não implementado |
| Customer-managed keys | ❌ | Não implementado |

### 6.4 Secrets Management

**Status:** ❌ Não estruturado

**Estado atual:**
- Env vars em Railway (não rotacionadas)
- Supabase secrets em Edge Functions
- Algumas credenciais em `integrations.accounts.config` (JSONB)

**Alvo:**
- Supabase Vault para secrets críticos
- Rotação automática
- Inventário centralizado

### 6.5 Data Retention & Deletion

**Status:** ❌ Não definido formalmente

**Alvo:**
| Tipo de dado | Retention | Deletion |
|--------------|-----------|----------|
| Audit logs | 7 anos | Automático |
| Transações | Indefinido | Soft-delete |
| PII | Por solicitação | Hard-delete com prova |
| Logs operacionais | 90 dias | Automático |

---

## 7. API & Database Security

### 7.1 Schemas Expostos vs Internos

**Status:** ⚠️ Auditado 2026-03-20 — grants excessivos detectados

**Estado factual (2026-03-20):**
- `anon`: SELECT em `analytics` (3), `content` (3), `crm` (7), `forms` (5 SELECT + 2 INSERT)
- `authenticated`: CRUD amplo em 18 schemas — commerce, community, content, core, crm, doare, eduzz, email, forms, imports, inbox, integrations, journeys, smart_forms, whatsapp
- ⚠️ **RISCO G-012:** `authenticated` tem TRUNCATE, REFERENCES, TRIGGER em schemas de produção (commerce, community, content, core, journeys). TRUNCATE bypassa RLS. Resquício de `GRANT ALL` pré-hardening.

**Alvo:**
- `api` → único schema exposto (views SECURITY INVOKER)
- Revogar TRUNCATE, REFERENCES, TRIGGER de `authenticated` em todos os schemas (G-012)
- Restringir grants ao mínimo necessário por schema

### 7.2 SECURITY DEFINER vs INVOKER

**Padrão:**
- Default: `SECURITY INVOKER`
- Views: `security_invoker = true`
- `SECURITY DEFINER`: apenas com aprovação + justificativa

**Inventário atual:** Ver seção 12

### 7.3 RLS Policies por Tipo de Tabela

| Tipo | Exemplo | Policy |
|------|---------|--------|
| Tenant-scoped | `crm.parties` | `tenant_id = active_tenant_id()` |
| User-scoped | `preferences` | `user_id = auth.uid()` |
| Membership-scoped | `notifications` | `tenant_id + user_id` |
| System/internal | `integrations.jobs` | Não expor |
| Audit | `audit.*` | Somente leitura restrita |

### 7.4 Índices para Performance de RLS

**Regra:** Toda coluna usada em policy RLS deve ter índice.

```sql
-- Exemplo
CREATE INDEX idx_table_tenant_id ON schema.table(tenant_id);
CREATE INDEX idx_table_tenant_deleted ON schema.table(tenant_id) WHERE deleted_at IS NULL;
```

---

## 8. Workers & Privileged Access

### 8.1 Credenciais por Worker

**Status:** ⚠️ Auditado 2026-03-20 — credencial compartilhada

**Estado factual:** Todos os 8 workers usam a mesma `DATABASE_URL` (conexão direta via `postgres` library). Nenhum usa Supabase JS client. Nenhum tem role dedicado.

| Worker | Credencial atual | Risco |
|--------|------------------|-------|
| ingestion-worker | DATABASE_URL compartilhada | Comprometimento de 1 = acesso total |
| analytics-worker | DATABASE_URL compartilhada | Idem |
| journeys-worker | DATABASE_URL compartilhada | Idem |
| journeys-event-worker | DATABASE_URL compartilhada | Idem |
| email-campaign-worker | DATABASE_URL compartilhada | Idem |
| email-sync-worker | DATABASE_URL compartilhada | Idem |
| historical-sync-worker | DATABASE_URL compartilhada | Idem |
| pull-sync-worker | DATABASE_URL compartilhada | Idem |

**Alvo (Sprint 8+):** Roles PostgreSQL dedicados por worker com grants mínimos. Ex: `analytics_worker_role` com SELECT/INSERT em `analytics.*`, sem acesso a `email.*` ou `commerce.*`.

### 8.2 service_role Scope

**Regra:** `service_role` deve ser usado apenas quando:
- Operação precisa bypassar RLS por design (ex: batch cross-tenant)
- Operação é administrativa (ex: migrations)

**Não usar quando:**
- Operação é em nome de usuário específico
- Lazy "porque é mais fácil"

### 8.3 Logging de Operações Privilegiadas

**Status:** ❌ Não existe

**Alvo:**
- Toda operação com `service_role` logada
- Worker ID + operation + affected records
- Alertas para operações anômalas

---

## 9. Auditoria & Logging

### 9.1 Audit Schema

**Status:** ✅ Implementado — Sprint 5 (2026-03-10)

**Tabela:** `audit.trail` — registro imutável de mudanças em tabelas sensíveis.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `id` | uuid | PK |
| `created_at` | timestamptz | Timestamp imutável |
| `user_id` | uuid | auth.uid() quando disponível |
| `user_email` | text | Email para referência |
| `role_name` | text | authenticated, service_role, anon |
| `schema_name` | text | Schema da tabela |
| `table_name` | text | Nome da tabela |
| `operation` | text | INSERT, UPDATE, DELETE |
| `record_id` | text | PK do registro |
| `tenant_id` | uuid | Para isolamento |
| `old_data` | jsonb | Estado anterior |
| `new_data` | jsonb | Novo estado |
| `changed_fields` | text[] | Campos alterados (UPDATE) |
| `ip_address` | inet | IP do request |
| `user_agent` | text | User agent |
| `request_id` | text | Correlação com logs |
| `transaction_id` | bigint | txid_current() |

**Índices:** 7 (PK, tenant+created_at, table+created_at, user+created_at, record, operation, created_at)

**RLS:** Habilitado — `service_role` full access, `authenticated` isolado por tenant (`app_auth.active_tenant_id()`)

**Funções trigger:** `audit.log_change()` (completa, com changed_fields) e `audit.log_change_simple()` (fallback). Ambas SECURITY DEFINER + `search_path = ''`.

### 9.2 Tabelas Auditadas (11 triggers)

| Prioridade | Schema | Tabela | Trigger |
|------------|--------|--------|---------|
| P0 | core | tenants | audit_tenants |
| P0 | core | tenant_users | audit_tenant_users |
| P0 | core | system_admins | audit_system_admins |
| P1 | email | tenant_config | audit_tenant_config |
| P1 | integrations | accounts | audit_integrations_accounts |
| P1 | commerce | asaas_config | audit_asaas_config |
| P2 | analytics | segments | audit_segments |
| P2 | journeys | journeys | audit_journeys |
| P2 | forms | forms | audit_forms |
| P2 | smart_forms | forms | audit_smart_forms |
| P3 | whatsapp | instances | audit_whatsapp_instances |

Todos: INSERT + UPDATE + DELETE → `audit.log_change()`

### 9.3 Funções Helper

| Função | Descrição | Acesso |
|--------|-----------|--------|
| `audit.get_record_history(schema, table, record_id)` | Histórico de um registro | authenticated |
| `audit.get_tenant_changes(tenant_id, since)` | Mudanças recentes do tenant | authenticated |
| `audit.get_user_changes(user_id, since)` | Mudanças feitas por usuário | authenticated |
| `audit.get_stats(since)` | Estatísticas por tabela | service_role |
| `audit.export_trail(tenant_id, from, to)` | Export para compliance | service_role |

### 9.4 Admin Activity Logs

**Ações logadas automaticamente (via triggers):**
- Mudança de permissão (core.tenant_users)
- Alteração de configuração sensível (email.tenant_config, integrations.accounts)
- Mudanças em tenants e admins

**Ações a logar (futuro):**
- Login/logout (depende de `auth.audit_log_entries` — vazio atualmente)
- Export de dados
- Acesso a dados de outro tenant (support)

### 9.5 SIEM Integration

**Status:** ❌ Não existe

**Alvo:** Datadog ou Sumo Logic

### 9.5 Retention

| Log type | Retention | Justificativa |
|----------|-----------|---------------|
| Audit (row changes) | 7 anos | Compliance |
| Audit (actions) | 7 anos | Compliance |
| Application logs | 90 dias | Debugging |
| Access logs | 1 ano | Security |

---

# PARTE C — INVENTÁRIO ATUAL (Sprint 0)

> **NOTA:** Esta seção será populada com os resultados dos diagnósticos SQL.

## 10. Schemas e Exposição

> **Diagnóstico executado:** 2026-03-09 — diagnostic-1.sql

### 10.1 Schemas em exposed_schemas

**Resultado:** `pg_settings` não retorna configs `pgrst.*` no Supabase managed. Verificar via Dashboard > API Settings.

**Schemas potencialmente expostos via PostgREST (por convenção Supabase, `public` + schemas com grants):**

| Schema | Classificação |
|--------|---------------|
| `public` | Exposto (default PostgREST) |
| `analytics` | Potencialmente exposto (grants existem) |
| `commerce` | Potencialmente exposto (grants existem) |
| `content` | Potencialmente exposto (grants existem) |
| `core` | Potencialmente exposto (grants existem) |
| `crm` | Potencialmente exposto (grants existem) |
| `email` | Potencialmente exposto (grants existem) |
| `forms` | Potencialmente exposto (grants existem) |
| `integrations` | Potencialmente exposto (grants existem) |
| `journeys` | Potencialmente exposto (grants existem) |
| `smart_forms` | Potencialmente exposto (grants existem) |

**Schemas custom/interno (sem exposição direta):**
`community`, `cron`, `doare`, `eduzz`, `imports`, `inbox`, `sympla`, `tags`, `whatsapp`

**Schemas Supabase gerenciados:**
`auth`, `extensions`, `graphql`, `realtime`, `storage`, `vault`

### 10.2 Tabelas por Schema

| Schema | Tables | Views | Mat. Views | Functions |
|--------|--------|-------|------------|-----------|
| `analytics` | 76 | 2 | 0 | 68 |
| `crm` | 29 | 1 | 0 | 30 |
| `core` | 19 | 0 | 0 | 30 |
| `email` | 17 | 0 | 0 | 14 |
| `commerce` | 14 | 0 | 0 | 11 |
| `integrations` | 8 | 0 | 0 | 20 |
| `journeys` | 7 | 0 | 0 | 12 |
| `forms` | 7 | 0 | 0 | 10 |
| `eduzz` | 7 | 0 | 0 | 5 |
| `smart_forms` | 6 | 0 | 0 | 1 |
| `community` | 5 | 0 | 0 | 0 |
| `inbox` | 4 | 0 | 0 | 15 |
| `content` | 4 | 0 | 0 | 4 |
| `sympla` | 4 | 0 | 0 | 0 |
| `whatsapp` | 3 | 0 | 0 | 8 |
| `imports` | 2 | 0 | 0 | 8 |
| `doare` | 2 | 0 | 0 | 4 |
| `public` | 0 | 1 | 0 | **242** |

**Alerta:** `public` tem **242 funções** — principal superfície de ataque via PostgREST `/rpc/`.

### 10.3 Grants para `anon`

> **ATUALIZAÇÃO Sprint 1 (2026-03-10):** 469 grants de `anon` foram revogados. Restam apenas 10 grants (7 em `forms` + 3 em `content`), todos com RLS habilitado. O inventário abaixo é o estado **PRÉ-Sprint 1** mantido para referência histórica. Backup completo em `audit.grant_backup_sprint1`.

~~**SEVERIDADE: CRITICA**~~  **RESOLVIDO** ✅

~~O role `anon` (usuários NÃO autenticados) tem acesso direto a tabelas sensíveis:~~

#### FULL ACCESS (DELETE, INSERT, SELECT, UPDATE, TRUNCATE, REFERENCES, TRIGGER)

| Schema | Tabelas |
|--------|---------|
| `analytics` | `base_rfm`, `rfm_segments` |
| `commerce` | `archived_transactions`, `courses`, `merge_history`, `product_tags`, `products`, `tags`, `transaction_import_errors`, `transaction_imports`, `transaction_product_names`, `transactions` |
| `community` | `comments`, `group_members`, `groups`, `posts`, `reactions` |
| `content` | `home_page_settings`, `landing_page_assets`, `landing_page_versions`, `landing_pages` |
| `core` | `activation_code_uses`, `activation_codes`, `custom_domains`, `delegated_subdomains`, `eduzz_integrations`, `eduzz_webhook_events`, `export_jobs`, `features`, `permission_audit_log`, `plan_features`, `plans`, `system_admins`, `tags`, `team_invitations`, `tenant_features`, `tenant_plans`, `tenant_users`, `tenants`, `user_features` |
| `crm` | `badges`, `customer_badges`, `customer_tags`, `customers`, `lead_form_submissions`, `lead_import_history`, `lead_notes`, `lead_tags`, `lost_reasons`, `opportunity_products` |
| `doare` | `donations`, `sync_state` |
| `email` | `asset_folders`, `campaign_recipients`, `campaign_sends`, `email_assets`, `email_campaigns`, `email_templates` |
| `journeys` | `journey_enrollments`, `journey_events`, `journey_executions`, `journeys` |
| `public` | `v_feed_posts` |

#### CRUD (DELETE, INSERT, SELECT, UPDATE)

| Schema | Tabelas |
|--------|---------|
| `analytics` | `rfm_score_mapping`, `rfm_segment_labels`, `segment_customers`, `segment_tags`, `segments` |
| `commerce` | `asaas_config`, `checkout_offers` |
| `crm` | `activity_types`, `lead_extra_info`, `lead_imports`, `legacy_party_map`, `opportunities`, `opportunity_activities`, `opportunity_notes`, `opportunity_stage_history`, `opportunity_tags`, `parties`, `party_organization`, `party_person`, `party_relationships`, `party_role_assignments`, `party_roles`, `pipeline_stages`, `pipelines`, `sales_goals` |
| `eduzz` | `buyers`, `cart_abandonments`, `contracts`, `integrations`, `invoices`, `webhook_events` |
| `email` | `authenticated_domains`, `branded_links`, `campaign_tags`, `sender_identities`, `tenant_config` |
| `forms` | `form_answers`, `form_blocks`, `form_submissions`, `form_tags`, `form_versions`, `forms`, `tag_definitions` |
| `inbox` | `accounts`, `attachment_cache`, `folders`, `messages` |
| `integrations` | `accounts`, `capabilities` |
| `smart_forms` | `form_answers`, `form_blocks`, `form_submissions`, `form_tags`, `form_versions`, `forms` |
| `whatsapp` | `instances` |

#### SELECT + partial

| Schema | Tabelas | Privileges |
|--------|---------|------------|
| `analytics` | `contact_attribution`, `contact_events_*` (44 partitions), `contact_product_stats_*`, `contact_state`, `customer_cluster_*`, `fallback_log`, `lifecycle_events`, `processing_jobs`, etc. | SELECT |
| `analytics` | `growth_formula_inputs` | INSERT, SELECT, UPDATE |
| `commerce` | `checkout_orders` | SELECT |
| `crm` | `opportunity_activities_archive`, `vw_parties_unified` | SELECT |
| `email` | `unsubscribe_feedback`, `unsubscribes` | SELECT |
| `email` | `unsubscribe_groups` | INSERT, SELECT, UPDATE |
| `imports` | `import_rows`, `import_runs` | INSERT, SELECT, UPDATE |
| `integrations` | `jobs`, `logs`, `providers`, `sync_schedules`, `webhook_events` | SELECT |
| `sympla` | `events`, `orders`, `participants`, `sync_cursors` | SELECT |
| `whatsapp` | `chats` | SELECT, UPDATE |
| `whatsapp` | `messages` | INSERT, SELECT |

**Impacto:** Qualquer pessoa pode acessar `https://core.kreatorshub.com.br/rest/v1/` com a anon key (pública no frontend) e ler/escrever dados de `core.tenants`, `core.system_admins`, `commerce.transactions`, `crm.customers`, etc. — a menos que RLS esteja habilitado e bloqueie efetivamente.

### 10.4 Grants para `authenticated`

Padrão similar ao `anon` — quase todas as tabelas recebem os mesmos grants. Exemplos relevantes:

| Schema | Tabelas | Privileges |
|--------|---------|------------|
| `core` | `tenants`, `system_admins`, `activation_codes` | FULL (7 privileges) |
| `commerce` | `transactions`, `products` | FULL (7 privileges) |
| `crm` | `customers`, `parties`, `opportunities` | DELETE, INSERT, SELECT, UPDATE |
| `email` | `email_campaigns`, `email_templates` | FULL (7 privileges) |
| `analytics` | `contact_state`, `contact_events_*` | SELECT |
| `integrations` | `accounts`, `capabilities` | DELETE, INSERT, SELECT, UPDATE |

**Impacto:** A segurança multi-tenant depende 100% de RLS. Se uma tabela não tem RLS habilitado, qualquer usuário autenticado pode acessar dados de QUALQUER tenant.

### 10.5 Resumo de Risco — Diagnóstico 1 — ✅ PARCIALMENTE RESOLVIDO

| Achado | Severidade | Status |
|--------|------------|--------|
| ~~`anon` com FULL ACCESS em `core.tenants`, `core.system_admins`~~ | ~~**CRITICA**~~ | ✅ Sprint 1 — 469 grants revogados |
| ~~`anon` com FULL ACCESS em `commerce.transactions`, `crm.customers`~~ | ~~**CRITICA**~~ | ✅ Sprint 1 — grants revogados |
| `public` schema com 242 functions | **ALTA** | Pendente — auditar callable via `/rpc/` |
| ~~11 schemas com grants para `anon`/`authenticated`~~ | ~~**ALTA**~~ | ✅ Sprint 1 — anon reduzido a 2 schemas (forms + content) |
| ~~`authenticated` sem RLS = cross-tenant leakage~~ | ~~**CRITICA**~~ | ✅ Sprint 2 — 0 tabelas multi-tenant sem RLS |

---

## 11. RLS Coverage

> **Diagnóstico inicial:** 2026-03-09 — diagnostic-2.sql
> **Atualizado:** 2026-03-10 — Sprint 2 completo

### 11.1 Resumo por Schema (Pós-Sprint 2)

| Schema | Total Tables | Com RLS | Sem RLS | Coverage |
|--------|-------------|---------|---------|----------|
| `analytics` | 78 | **78** | 0 | **100%** ✅ |
| `commerce` | 14 | 14 | 0 | **100%** |
| `community` | 5 | 5 | 0 | **100%** |
| `content` | 4 | 4 | 0 | **100%** |
| `core` | 19 | 19 | 0 | **100%** |
| `crm` | 29 | **29** | 0 | **100%** ✅ |
| `doare` | 2 | 2 | 0 | **100%** |
| `eduzz` | 7 | 7 | 0 | **100%** |
| `email` | 17 | 17 | 0 | **100%** |
| `forms` | 7 | 7 | 0 | **100%** |
| `imports` | 2 | 2 | 0 | **100%** |
| `inbox` | 4 | 4 | 0 | **100%** |
| `integrations` | 8 | **8** | 0 | **100%** ✅ |
| `journeys` | 7 | **7** | 0 | **100%** ✅ |
| `smart_forms` | 6 | 6 | 0 | **100%** |
| `sympla` | 4 | **4** | 0 | **100%** ✅ |
| `whatsapp` | 3 | 3 | 0 | **100%** |

**Total geral:** 219 tabelas COM RLS, **0 tabelas SEM RLS**.

### 11.2 Tabelas COM RLS (219 tabelas)

RLS habilitado em **todos** os 17 schemas: `analytics` (78), `commerce` (14), `community` (5), `content` (4), `core` (19), `crm` (29), `doare` (2), `eduzz` (7), `email` (17), `forms` (7), `imports` (2), `inbox` (4), `integrations` (8), `journeys` (7), `smart_forms` (6), `sympla` (4), `whatsapp` (3).

### 11.3 Tabelas SEM RLS (Gaps) — ✅ RESOLVIDO

~~66 tabelas sem RLS~~ — **Todas resolvidas no Sprint 2** (2026-03-10).

Tabelas que foram corrigidas:
- **analytics** (57): 9 tabelas principais + 41 partições `contact_events` + 8 partições `contact_product_stats` — RLS habilitado explicitamente em cada partição
- **journeys** (3): `backfill_jobs`, `event_processing_jobs`, `processing_jobs`
- **crm** (1): `opportunity_activities_archive`
- **integrations** (1): `sync_schedules`
- **sympla** (4): `events`, `orders`, `participants`, `sync_cursors`

Adicionalmente, 2 tabelas pré-existentes com RLS mas sem policies foram corrigidas:
- `analytics.product_link_jobs` — adicionadas policies `tenant_isolation_select` + `service_role_all`
- `integrations.job_errors` — adicionadas policies `tenant_isolation_select` + `service_role_all`

### 11.4 Tabelas Multi-tenant SEM RLS — ✅ ZERO

**Nenhuma tabela com `tenant_id` existe sem RLS habilitado.** Validado em 2026-03-10.

### 11.5 Policies Existentes (Pós-Sprint 2)

#### Resumo por schema

| Schema | Tabelas c/ policies | Total policies | Novas Sprint 2 |
|--------|-------------------|----------------|----------------|
| `analytics` | 28 | 78+ | +27 |
| `commerce` | 14 | 27 | — |
| `community` | 5 | 16 | — |
| `content` | 4 | 8 | — |
| `core` | 19 | 57 | — |
| `crm` | 29 | 131 | +2 |
| `doare` | 2 | 4 | — |
| `eduzz` | 7 | 9 | — |
| `email` | 17 | 76 | — |
| `forms` | 7 | 20 | — |
| `imports` | 2 | 8 | — |
| `inbox` | 4 | 12 | — |
| `integrations` | 8 | 29 | +5 |
| `journeys` | 7 | 17 | +6 |
| `smart_forms` | 6 | 15 | — |
| `sympla` | 4 | 8 | +8 |
| `whatsapp` | 3 | 14 | — |

#### Padrões de policy observados

| Padrão | Exemplo | Frequência |
|--------|---------|------------|
| `tenant_isolation` (ALL) | `commerce.archived_transactions` | Comum em tabelas simples |
| Per-operation (SELECT/INSERT/UPDATE/DELETE) | `crm.opportunities` | Comum em CRM |
| `service_role_bypass` (ALL) | `integrations.jobs` | Presente em tabelas de workers |
| `system_admins_*` (ALL/SELECT) | `core.tenants` | Presente em tabelas admin |
| `public_read` (SELECT) | `forms.forms` | Forms/landing pages públicos |
| Roles: `{authenticated}` | Maioria das policies | Padrão |
| Roles: `{anon}` | `content.home_page_settings` | Acesso público |

#### Observações de qualidade das policies

1. **Policies duplicadas:** Algumas tabelas têm policies redundantes (ex: `email.email_templates` tem 8 policies com overlap)
2. **`service_role` bypass:** Presente em workers, mas precisa auditoria de quem realmente precisa
3. **`system_admins` check:** Geralmente via subquery em `core.system_admins` — adequado
4. **Tenant isolation:** Maioria usa `tenant_id = (SELECT active_tenant_id FROM ...)` ou similar

### 11.6 Resumo de Risco — Diagnóstico 2 — ✅ RESOLVIDO

| Achado | Severidade | Status |
|--------|------------|--------|
| ~~`analytics` com 57 tabelas sem RLS~~ | ~~**CRITICA**~~ | ✅ Sprint 2 — 78/78 com RLS |
| ~~`contact_events` (43 partitions) sem RLS~~ | ~~**CRITICA**~~ | ✅ Sprint 2 — RLS explícito em cada partição |
| ~~`contact_product_stats` (9 tabelas) sem RLS~~ | ~~**ALTA**~~ | ✅ Sprint 2 — RLS explícito em cada partição |
| ~~`journeys` com 3 tabelas de jobs sem RLS~~ | ~~**ALTA**~~ | ✅ Sprint 2 — 7/7 com RLS |
| ~~`crm.opportunity_activities_archive` sem RLS~~ | ~~**MEDIA**~~ | ✅ Sprint 2 — 29/29 com RLS |
| ~~`sympla` (4 tabelas) sem RLS~~ | ~~**MEDIA**~~ | ✅ Sprint 2 — 4/4 com RLS |
| Policies — cobertura consistente | **INFO** | 219 tabelas, 500+ policies |

**Todos os achados do diagnóstico 2 foram resolvidos no Sprint 2.**

---

## 12. Funções e Privilégios

> **Diagnóstico executado:** 2026-03-09 — diagnostic-3.sql

### 12.1 SECURITY DEFINER — Inventário Completo

**Total: 319 funções SECURITY DEFINER** em 16 schemas.

#### Schemas expostos (205 funções DEFINER)

**`analytics` (55 funções DEFINER):**
`apply_segment_membership_diff`, `auto_seed_rfm_mapping`, `build_lead_where_clause`, `claim_next_jobs`, `cleanup_stale_jobs`, `cleanup_stale_segment_eval_jobs`, `complete_job`, `count_segment_customers`, `count_segment_leads_unique`, `count_segment_parties`, `count_segment_parties_leads`, `count_segment_parties_total`, `detect_segment_entity_type`, `enqueue_analytics_refresh`, `enqueue_coalesced`, `enqueue_job`, `enqueue_job_internal`, `enqueue_product_link_job`, `enqueue_refresh_all_tenants` (×2), `export_lookalike_scores`, `fail_job`, `get_customer_features_by_ids`, `get_leads_count`, `get_lookalike_members`, `get_lookalike_source_customers`, `get_lookalike_source_options`, `get_markov_transitions`, `get_pareto_analysis`, `get_reactivation_snapshot`, `get_segment_by_id`, `get_segment_cached_counts`, `get_segment_cached_leads`, `get_segment_parties_leads_preview`, `get_segment_parties_preview` (×2), `get_segment_preview`, `get_segments_with_live_count`, `get_unique_segmented_customers_count`, `initialize_tenant_rfm`, `process_purchase_analytics`, `purge_processed_eval_queue`, `record_contact_event`, `refresh_segment_customers`, `refresh_segment_parties`, `release_jobs`, `seed_rfm_score_mapping`, `sync_cluster_segment_customers`, `trg_enqueue_on_customer_change`, `trg_enqueue_on_form_submission`, `trg_enqueue_on_lead_import`, `trg_enqueue_on_transaction_change`, `upsert_cluster_segment` (×2), `upsert_contact_state_on_purchase`

**`core` (30 funções DEFINER — 100% do schema):**
`accept_team_invitation`, `add_existing_user_to_team`, `cancel_team_invitation`, `check_email_exists`, `create_team_invitation`, `create_tenant_from_signup`, `generate_invitation_token`, `get_invitation_by_token`, `get_member_features`, `get_pending_invitations`, `get_permission_audit_log`, `get_tenant_members_list`, `get_tenant_team_members`, `get_user_features`, `get_user_tenant_memberships`, `grant_feature_to_member`, `has_feature_access`, `is_system_admin`, `is_tenant_admin`, `list_all_users`, `on_tenant_created`, `remove_team_member`, `resend_team_invitation`, `revoke_feature_from_member`, `switch_tenant`, `sync_tenant_to_jwt`, `update_member_features`, `update_team_member_role`, `use_activation_code`, `validate_activation_code`

**`crm` (25 funções DEFINER):**
`assign_customer_role_on_paid_transaction`, `backfill_parties`, `backfill_party_ids`, `batch_import_rows`, `ensure_customer_for_party`, `ensure_party_for_customer`, `fn_aging_per_stage`, `fn_forecast_base`, `fn_pipeline_value_open`, `fn_sales_cycle_duration`, `fn_stalled_deals`, `fn_win_rate`, `get_party_complete`, `get_party_timeline`, `list_parties`, `resolve_legacy_id`, `resolve_party_id`, `search_parties`, `seed_default_activity_types`, `sync_lead_update_to_party`, `sync_opportunity_summary`, `trg_update_last_activity_at`, `trg_update_last_activity_on_note`, `trg_update_next_activity_at`, `upsert_party_person`

**`integrations` (15 funções DEFINER):**
`bulk_enqueue_jobs`, `cancel_job`, `complete_job`, `enqueue_webhook_job`, `fail_job`, `get_account_by_tenant_provider`, `get_job_stats`, `invoke_apidozap`, `invoke_doare_sync`, `invoke_eduzz_webhook`, `invoke_nylas_sync`, `invoke_worker`, `log_event`, `pending_jobs_count`, `receive_webhook`

**`email` (14 funções DEFINER — 100% do schema):**
`atomic_check_and_complete_send`, `cancel_scheduled_send`, `claim_pending_recipients`, `enqueue_campaign_send`, `expire_soft_bounce_suppressions`, `get_preference_center_data`, `increment_campaign_send_counter`, `increment_campaign_send_counts`, `increment_campaign_send_delivered`, `increment_recipient_open_count`, `increment_send_stat`, `record_email_engagement_event`, `reset_all_stale_processing`, `reset_stale_processing_recipients`

**`commerce` (10 funções DEFINER):**
`archive_duplicate_transactions`, `find_duplicate_transaction_products`, `get_checkout_data`, `get_customer_billing_data`, `process_transactions_batch`, `sync_orphan_transaction_customers`, `sync_transaction_customer`, `upsert_provider_transaction` (×3 overloads)

**`journeys` (10 funções DEFINER):**
`claim_next_job`, `cleanup_old_jobs`, `complete_job`, `enqueue_event_processing_job`, `enqueue_job`, `enqueue_journey_checks`, `enqueue_ready_waits`, `enqueue_segment_checks`, `fail_job`, `release_stale_claims`

**`forms` (8 funções DEFINER):**
`abandon_public_submission`, `complete_public_submission`, `create_public_submission`, `get_public_form`, `link_public_submission_to_lead`, `save_public_answer`, `trg_enqueue_link_submission`, `update_submission_progress`

**`content` (3 funções DEFINER):**
`get_landing_page_by_slug`, `get_next_landing_page_version`, `publish_landing_page`

#### `public` schema (113 funções DEFINER)

**Principal superfície de ataque via `/rpc/`.**

Categorias:
- **Eduzz** (16): `eduzz_create_customer_from_lead`, `eduzz_find_customer`, `eduzz_find_or_create_lead`, `eduzz_get_integration`, `eduzz_increment_sync_progress`, `eduzz_link_buyer`, `eduzz_link_cart_lead`, `eduzz_link_invoice_transaction`, `eduzz_log_event`, `eduzz_update_event`, `eduzz_update_integration_stats`, `eduzz_upsert_buyer`, `eduzz_upsert_cart_abandonment`, `eduzz_upsert_contract`, `eduzz_upsert_invoice`
- **Analytics/RFM** (15): `claim_next_jobs`, `cleanup_stale_jobs`, `complete_job`, `enqueue_analytics_refresh`, `enqueue_compute_clusters_stale`, `enqueue_job_internal`, `enqueue_refresh_stale_segments`, `fail_job`, `get_analytics_data`, `get_customer_features_by_ids`, `get_customer_ids_paginated`, `get_customer_rfm`, `get_customers_by_segment`, `get_dashboard_data`, `get_rfm_snapshot`, etc.
- **CRM/Customers** (10): `get_aggregated_customers`, `get_customer_complete`, `get_customer_tag_counts`, `get_filtered_customers`, `get_leads_with_stats`, `get_product_customers`, etc.
- **Inbox/Email** (12): `batch_link_messages_to_crm`, `claim_active_accounts`, `create_nylas_account`, `disconnect_nylas_account`, `get_folder_counts`, `get_inbox_messages`, `get_messages_for_contact`, `get_tenant_account`, `increment_campaign_send_counter`, `update_message_flags`, `upsert_message` (×2)
- **WhatsApp** (5): `wa_get_chats`, `wa_get_instances`, `wa_get_messages`, `wa_increment_unread`, `wa_mark_read`
- **Imports** (7): `create_import_run`, `finalize_import_run`, `insert_import_rows_batch`, `invoke_process_import`, `process_import_batch`, `process_import_run`, `upsert_import_row` (×2)
- **Commerce** (5): `execute_product_merge`, `execute_product_merge_v2`, `get_merge_preview`, `rollback_product_merge`, `upsert_provider_transaction` (×3)
- **Segmentation** (4): `count_segment_parties_leads`, `count_segment_parties_total`, `get_segment_parties_leads_preview`, `refresh_segment_customers`
- **Auth/Tenant** (11): `check_is_system_admin`, `get_tenant_id`, `get_user_id_by_email`, `get_user_role`, `handle_new_user`, `is_tenant_admin`, `is_tenant_member`, `resolve_tenant_by_domain`, `update_tenant_default_sender`, etc.
- **Integrations** (8): `bulk_enqueue_jobs`, `cancel_job`, `claim_jobs`, `enqueue_webhook_job`, `get_integration_config`, `get_job_stats`, `get_jobs_count_grouped`, `log_event`, `receive_webhook`
- **Secrets** (2): `create_secret`, `read_secret`
- **Doações** (3): `doare_bulk_upsert_donations`, `doare_process_unprocessed_donations`, `doare_upsert_sync_state`
- **Outros** (6): `get_form_og_data`, `get_monthly_revenue_by_product` (×2), `get_pareto_analysis`, `get_reactivation_snapshot`, `toggle_reaction` (×2), `trigger_email_engagement_journey_event`, `trigger_purchase_journey_event`, `upsert_cluster_segment`

#### Schemas internos (114 funções DEFINER)

**`inbox` (14):** `batch_link_messages_to_crm`, `batch_upsert_messages`, `claim_active_accounts`, `create_nylas_account`, `disconnect_nylas_account`, `get_folder_counts`, `get_messages_for_contact`, `get_tenant_account`, `get_user_tenant_ids`, `link_message_to_crm`, `mark_inbox_active`, `update_message_flags`, `upsert_message` (×2)

**`imports` (7):** `create_import_run`, `finalize_import_run`, `insert_import_rows_batch`, `invoke_process_import`, `process_import_row`, `process_import_run`, `validate_import_row`

**`whatsapp` (7):** `get_chats_for_instance`, `get_messages_for_chat`, `get_user_tenant_ids`, `increment_unread`, `mark_chat_read`, `upsert_chat`, `upsert_message`

**`eduzz` (5 — 100% do schema):** `enqueue_pending_invoices`, `fail_sync`, `increment_sync_progress`, `init_sync`, `reset_sync`

**`doare` (2):** `enqueue_sync_jobs`, `process_unprocessed_donations`

**`pgbouncer` (1):** `get_auth`

### 12.2 Contagem por Schema

| Schema | DEFINER | INVOKER | Total | % DEFINER |
|--------|---------|---------|-------|-----------|
| `public` | **113** | 129 | 242 | 46.7% |
| `analytics` | **55** | 14 | 69 | 79.7% |
| `core` | **30** | 0 | 30 | **100%** |
| `crm` | **25** | 5 | 30 | 83.3% |
| `integrations` | **15** | 5 | 20 | 75.0% |
| `inbox` | **14** | 1 | 15 | 93.3% |
| `email` | **14** | 0 | 14 | **100%** |
| `commerce` | **10** | 1 | 11 | 90.9% |
| `journeys` | **10** | 2 | 12 | 83.3% |
| `forms` | **8** | 2 | 10 | 80.0% |
| `whatsapp` | **7** | 1 | 8 | 87.5% |
| `imports` | **7** | 1 | 8 | 87.5% |
| `eduzz` | **5** | 0 | 5 | **100%** |
| `content` | **3** | 1 | 4 | 75.0% |
| `doare` | **2** | 2 | 4 | 50.0% |
| `pgbouncer` | **1** | 0 | 1 | **100%** |
| **TOTAL** | **319** | **164** | **483** | **66.0%** |

**Alerta:** 66% das funções são SECURITY DEFINER — deveria ser exceção, não regra. Schemas `core`, `email`, `eduzz` são 100% DEFINER.

### 12.3 SECURITY DEFINER sem search_path seguro — ✅ RESOLVIDO

~~**SEVERIDADE: CRITICA — 26 funções vulneráveis a search_path injection**~~

**Corrigido no Sprint 1** (2026-03-10): Todas as 27 funções SECURITY DEFINER em schemas expostos receberam `SET search_path = ''`.

Funções corrigidas: analytics (12), content (3), forms (3), integrations (6), public (2), whatsapp (1).

Backup em `audit.function_backup_sprint1`.

### 12.4 Views sem security_invoker

**3 views corrigidas com `security_invoker = true` (Sprint 1):** ✅

| Schema | View | security_invoker | Status |
|--------|------|-----------------|--------|
| `analytics` | `v_segment_base` | `true` | ✅ Corrigido Sprint 1 |
| `analytics` | `v_segment_leads_base` | `true` | ✅ Corrigido Sprint 1 |
| `crm` | `vw_parties_unified` | `true` | ✅ Corrigido Sprint 1 |

Views agora executam com privilégios do **caller** (authenticated), respeitando RLS das tabelas subjacentes.

### 12.5 Resumo de Risco — Diagnóstico 3

| Achado | Severidade | Ação | Status |
|--------|------------|------|--------|
| ~~319 funções SECURITY DEFINER (66% do total)~~ | ~~**CRITICA**~~ | ~~Auditar e migrar para INVOKER~~ | ✅ Sprint 3 — reduzido para 250 (51%) |
| ~~26 funções DEFINER sem `search_path` seguro~~ | ~~**CRITICA**~~ | ~~Adicionar `SET search_path = ''`~~ | ✅ Sprint 1 |
| ~~`forms.get_public_form` e `save_public_answer` sem search_path~~ | ~~**CRITICA**~~ | ~~Fix prioritário~~ | ✅ Sprint 1 |
| ~~`public` schema com 113 funções DEFINER callable via `/rpc/`~~ | ~~**ALTA**~~ | ~~Auditar quais devem ser chamáveis externamente~~ | ✅ Sprint 3 — reduzido para 82 |
| `core`, `eduzz`, `pgbouncer` são 100% DEFINER | **BAIXA** | Justificado: funções de sistema/infraestrutura | ✅ Sprint 3 — documentado |
| ~~3 views sem `security_invoker` (dados cross-tenant)~~ | ~~**ALTA**~~ | ~~Adicionar `security_invoker = true`~~ | ✅ Sprint 1 |

---

## 13. Dados Sensíveis (PII)

> **Diagnóstico executado:** 2026-03-09 — diagnostic-4.sql

### 13.1 Colunas com PII

**Total: 400 colunas com PII potencial** identificadas por heurística de nome.

#### CRITICO — Credenciais e Documentos

| Schema | Tabela | Colunas | Tipo |
|--------|--------|---------|------|
| `commerce` | `asaas_config` | `api_key_encrypted`, `webhook_token` | Credencial |
| `core` | `eduzz_integrations` | `api_key`, `public_key`, `webhook_secret` | Credencial |
| `core` | `team_invitations` | `token` | Credencial |
| `eduzz` | `integrations` | `access_token`, `api_key`, `client_secret`, `public_key`, `refresh_token`, `token_expires_at`, `webhook_secret` | Credencial |
| `email` | `tenant_config` | `sendgrid_subuser_api_key`, `webhook_secret` | Credencial |
| `integrations` | `accounts` | `access_token`, `client_secret`, `refresh_token`, `secrets_ref`, `token_expires_at` | Credencial |
| `whatsapp` | `instances` | `webhook_secret` | Credencial |
| `commerce` | `archived_transactions`, `checkout_orders`, `transactions` | `customer_cpf` | Documento |
| `crm` | `customers`, `party_person` | `cpf` | Documento |
| `crm` | `party_organization` | `cnpj` | Documento |
| `imports` | `import_rows` | `parsed_cpf` | Documento |
| `sympla` | `orders`, `participants` | `cpf` | Documento |

#### ALTO — Email e Telefone

| Schema | Tabelas com email | Tabelas com telefone |
|--------|-------------------|----------------------|
| `analytics` | `contact_state`, `segment_parties` | — |
| `commerce` | `archived_transactions`, `checkout_orders`, `transaction_import_errors`, `transactions` | `archived_transactions`, `checkout_orders`, `transactions` |
| `core` | `system_admins`, `team_invitations` | — |
| `crm` | `customers`, `opportunities`, `parties` | `customers`, `opportunities`, `parties` |
| `doare` | `donations` | — |
| `eduzz` | `buyers`, `cart_abandonments`, `contracts` (×3), `integrations`, `invoices` (×4) | `buyers` (×4), `cart_abandonments`, `contracts` (×2), `invoices` |
| `email` | `campaign_recipients`, `campaign_sends` (×2), `email_events`, `sender_identities` (×2), `suppression_list`, `tenant_config`, `unsubscribe_feedback`, `unsubscribes` | — |
| `forms` | `form_submissions` | — |
| `imports` | `import_rows` | `import_rows` |
| `inbox` | `accounts`, `messages` (×5) | — |
| `integrations` | `accounts` | — |
| `journeys` | `journey_enrollments`, `journey_events` | — |
| `smart_forms` | `form_submissions` | — |
| `sympla` | `orders`, `participants` | `participants` |
| `whatsapp` | — | `chats`, `instances` |

### 13.2 Resumo por Schema

| Schema | Tabelas c/ PII | Colunas PII | Tipos de PII |
|--------|---------------|-------------|--------------|
| `eduzz` | 6 | 38 | credential, document, email, name, phone |
| `crm` | 14 | 37 | document, email, name, phone |
| `email` | 14 | 30 | credential, email, name |
| `analytics` | 13 | 29 | email, name, phone |
| `commerce` | 12 | 29 | credential, document, email, name, phone |
| `sympla` | 3 | 13 | document, email, name, phone |
| `core` | 9 | 12 | credential, email, name |
| `whatsapp` | 3 | 10 | credential, name, phone |
| `imports` | 2 | 8 | document, email, name, phone |
| `integrations` | 3 | 8 | credential, email, name |
| `inbox` | 3 | 8 | email, name |
| `cron` | 2 | 4 | name |
| `journeys` | 3 | 3 | email, name |
| `forms` | 3 | 3 | email, name |
| `doare` | 1 | 2 | document, email |
| `content` | 2 | 2 | name |
| `smart_forms` | 2 | 2 | email, name |
| `public` | 1 | 1 | name |
| `community` | 1 | 1 | name |
| **TOTAL** | **~97** | **~240** | |

### 13.3 Dados Financeiros

**33 tabelas com colunas financeiras** em 10 schemas:

| Schema | Tabela | Colunas financeiras |
|--------|--------|---------------------|
| `analytics` | `base_rfm` | `monetary_score` |
| `analytics` | `contact_attribution` | `revenue`, `revenue_first_touch`, `revenue_last_touch`, `revenue_linear`, `revenue_time_decay`, `transaction_id` |
| `analytics` | `contact_state` | `monetary`, `first_touch_revenue`, `total_attributed_revenue`, `total_revenue` |
| `analytics` | `customer_cluster_subgroups` | `avg_revenue` |
| `analytics` | `customer_clusters` | `avg_revenue` |
| `analytics` | `product_journey_analysis` | `avg_revenue` |
| `analytics` | `tenant_distribution` | `monetary_*` (×7), `revenue_*` (×3), `transactions_last_30d` |
| `commerce` | `archived_transactions` | `amount`, `payment_type`, `external_transaction_id` |
| `commerce` | `checkout_offers` | `price`, `original_price`, `payment_methods` |
| `commerce` | `checkout_orders` | `amount`, `asaas_payment_id` |
| `commerce` | `courses` | `price`, `original_price` |
| `commerce` | `products` | `price`, `cost_price` |
| `commerce` | `transactions` | `amount`, `payment_type`, `external_transaction_id`, `transaction_date` |
| `core` | `plans` | `price_monthly`, `price_yearly` |
| `crm` | `opportunities` | `amount` |
| `crm` | `opportunity_products` | `unit_price` |
| `doare` | `donations` | `valor_bruto`, `valor_bruto_brl`, `valor_liquido`, `valor_liquido_brl`, `valor_taxa` |
| `eduzz` | `contracts` | `recurrence_price`, `payment_method`, `payment_installments` |
| `eduzz` | `invoices` | `amount`, `amount_paid`, `payment_method` |
| `eduzz` | `products` | `price` |
| `imports` | `import_rows` | `parsed_amount`, `parsed_payment_type`, `external_transaction_id` |
| `sympla` | `orders` | `order_total_sale_price`, `transaction_type` |
| `sympla` | `participants` | `ticket_sale_price` |

### 13.4 Colunas JSONB (risco de PII não mapeado)

**~160 colunas JSONB** em 19 schemas. Colunas de maior risco (podem conter PII não indexado):

| Schema | Tabela | Coluna JSONB | Risco PII |
|--------|--------|-------------|-----------|
| `analytics` | `contact_events_*` (45 partitions) | `payload`, `state_snapshot` | **ALTO** — eventos comportamentais com possível PII |
| `commerce` | `archived_transactions`, `transactions` | `raw_source` | **ALTO** — dados brutos do provider |
| `commerce` | `transactions` | `affiliate_data` | MEDIO — pode ter nomes/emails |
| `core` | `eduzz_webhook_events` | `raw_payload` | **ALTO** — payload completo do webhook |
| `core` | `tenants` | `settings` | MEDIO — configurações do tenant |
| `crm` | `customers`, `parties` | `metadata` | MEDIO — metadata livre |
| `eduzz` | `invoices` | `items`, `fees`, `gains`, `chargeback_data` | MEDIO — dados financeiros detalhados |
| `eduzz` | `webhook_events` | `raw_payload` | **ALTO** — payload completo |
| `email` | `email_events` | `raw_payload` | **ALTO** — eventos de email com possível PII |
| `imports` | `import_rows` | `raw_json`, `parsed_address_json` | **ALTO** — dados originais do import |
| `integrations` | `accounts` | `config` | **CRITICO** — pode conter credenciais |
| `integrations` | `webhook_events` | `payload`, `headers` | **ALTO** — payloads de webhooks |
| `journeys` | `journey_executions` | `input_json`, `output_json` | MEDIO — dados de execução |
| `sympla` | `events`, `orders`, `participants` | `raw` | **ALTO** — dados brutos completos |

### 13.5 Credenciais de Terceiros — Inventário Atualizado (Sprint 4)

**Diagnóstico executado:** 2026-03-10 — Sprint 4 Fase 1

**Total: 20 colunas com credenciais em 7 tabelas**

| Tabela | Colunas | Rows | Credenciais Populadas | Status |
|--------|---------|------|-----------------------|--------|
| `commerce.asaas_config` | `api_key_encrypted`, `webhook_token` | 1 | api_key: 1, webhook_token: 0 | ✅ api_key criptografado |
| `core.eduzz_integrations` | `api_key`, `webhook_secret` | 1 | api_key: 0, webhook_secret: 1 | ❌ LEGADO — dropar |
| `eduzz.integrations` | `api_key`, `access_token`, `refresh_token`, `client_secret`, `webhook_secret`, `token_expires_at` | 4 | api_key: 1, access_token: 4, webhook_secret: 4 | ❌ LEGADO — migrar |
| `email.tenant_config` | `sendgrid_subuser_api_key`, `webhook_secret` | 4 | sendgrid_key: 4, webhook_secret: 0 | ❌ Texto plano |
| `integrations.accounts` | `access_token`, `refresh_token`, `client_secret`, `secrets_ref`, `token_expires_at` | 8 | access_token: 4, refresh_token: 0, client_secret: 2 | ❌ Texto plano |
| `whatsapp.instances` | `webhook_secret`, `vault_secret_name` | 1 | webhook_secret: 1 | ⚠️ Parcial |
| `core.team_invitations` | `token` | — | — | ➖ Invite token (não credencial) |

**Descoberta crítica:** `integrations.accounts.config` (JSONB) também contém secrets — duplicação:

| Provider | Chaves no config JSONB |
|----------|------------------------|
| eduzz | `api_key`, `public_key`, `webhook_secret` |
| sendgrid | `sendgrid_subuser_api_key`, `webhook_secret` |
| sympla | `s_token` |

**Funções que acessam credenciais:**

| Função | Tabela | Campos |
|--------|--------|--------|
| `public.eduzz_get_integration` | `integrations.accounts` | tokens |
| `public.get_tenant_by_sendgrid_message` | `email.tenant_config` | sendgrid config |
| `public.get_tenant_email_config` | `email.tenant_config` | sendgrid config |

**Tenants afetados:** 4 tenants (eduzz, sendgrid, integrations), 1 tenant (whatsapp)

**Vault atual (4 secrets):**
- `supabase_url` — infra
- `service_role_key` — infra
- `whatsapp_*` — 1 instância
- `DOARE_API_URL` — Doaré

**Impacto:** Se um atacante conseguir ler estas tabelas (via falha de RLS ou SQL injection), obtém acesso direto a APIs de todos os tenants.

**Única exceção positiva:** `commerce.asaas_config.api_key_encrypted` — já usa criptografia.

### 13.6 Resumo de Risco — Diagnóstico 4

| Achado | Severidade | Ação |
|--------|------------|------|
| 20 colunas com credenciais em texto (4-5 tenants, 7 tabelas) | **ALTA** | Sprint 4 — diagnóstico completo, migração incremental planejada |
| `eduzz.integrations` com 6 colunas de credencial | **ALTA** | LEGADO — dropar após migração para `integrations.accounts` |
| CPF/CNPJ em 8+ tabelas sem field-level encryption | **ALTA** | Avaliar encryption at field level |
| ~160 colunas JSONB com PII potencialmente não mapeado | **ALTA** | Auditar `raw_payload`, `raw_source`, `config` |
| `integrations.accounts.config` JSONB contém secrets (eduzz, sendgrid, sympla) | **ALTA** | Incluir na migração de Sprint 4 |
| 400 colunas PII em 19 schemas | **MEDIO** | Classificação formal de dados necessária |
| `analytics.contact_events` com payload JSONB sem RLS | **CRITICA** | Combinar com gap do Diagnóstico 2 |

---

## 14. Workers e Credenciais

> **Diagnóstico executado:** 2026-03-09 — diagnostic-5.sql

### 14.1 Roles e Privilégios no Banco

#### Roles custom (além dos padrão Supabase)

| Role | Superuser | Create Roles | Login | Bypass RLS | Conn Limit | Validade |
|------|-----------|-------------|-------|------------|------------|----------|
| `cli_login_postgres` | ❌ | ❌ | ✅ | ❌ | -1 | 2026-03-03 (EXPIRADO) |
| `supabase_etl_admin` | ❌ | ❌ | ✅ | **✅** | -1 | — |
| `supabase_functions_admin` | ❌ | **✅** | ✅ | ❌ | -1 | — |

**Observações:**
- `cli_login_postgres` — expirado em 2026-03-03, pode ser removido
- `supabase_etl_admin` — tem BYPASSRLS (ver 14.2)
- `supabase_functions_admin` — pode criar roles (risco de privilege escalation)

### 14.2 Roles com BYPASSRLS

**5 roles podem ignorar TODAS as RLS policies:**

| Role | Superuser | Login |
|------|-----------|-------|
| `postgres` | ❌* | ✅ |
| `service_role` | ❌ | ❌ |
| `supabase_admin` | **✅** | ✅ |
| `supabase_etl_admin` | ❌ | ✅ |
| `supabase_read_only_user` | ❌ | ✅ |

*`postgres` não é superuser no Supabase managed, mas tem BYPASSRLS.

**Alerta:** `supabase_read_only_user` tem BYPASSRLS — se comprometido, pode ler dados de TODOS os tenants ignorando RLS.

### 14.3 Grants do service_role

`service_role` tem **FULL ACCESS** (7 privileges: DELETE, INSERT, REFERENCES, SELECT, TRIGGER, TRUNCATE, UPDATE) em todos os schemas:

| Schema | Grants (tabelas × privileges) |
|--------|-------------------------------|
| `crm` | 164 |
| `analytics` | 146 |
| `core` | 133 |
| `email` | 119 |
| `commerce` | 77 |
| `forms` | 49 |
| `smart_forms` | 42 |
| `eduzz` | 42 |
| `community` | 35 |
| `journeys` | 31 |
| `integrations` | 30 |
| `content` | 28 |
| `inbox` | 19 |
| `sympla` | 16 |
| `doare` | 14 |
| `imports` | 14 |
| `whatsapp` | 12 |
| `public` | 7 |

**Impacto:** `service_role` + BYPASSRLS = acesso total sem restrição. Qualquer worker que use `service_role` key tem poder absoluto sobre o banco.

### 14.4 Funções executáveis pelo service_role

`service_role` pode executar **488 funções** em 17 schemas:

| Schema | Funções executáveis |
|--------|--------------------|
| `public` | 242 |
| `analytics` | 69 |
| `crm` | 30 |
| `core` | 30 |
| `integrations` | 20 |
| `inbox` | 15 |
| `email` | 14 |
| `journeys` | 12 |
| `commerce` | 11 |
| `forms` | 10 |
| `imports` | 8 |
| `whatsapp` | 8 |
| `cron` | 5 |
| `eduzz` | 5 |
| `content` | 4 |
| `doare` | 4 |
| `smart_forms` | 1 |

### 14.5 Secrets no Vault

**4 secrets configurados:**

| Nome | Descrição | Criado em |
|------|-----------|-----------|
| `DOARE_API_URL` | Doaré API URL for proxy edge function | 2026-02-20 |
| `service_role_key` | Service role key for pg_net Edge Function calls | 2026-02-12 |
| `supabase_url` | Supabase project URL for pg_net calls | 2026-02-12 |
| `whatsapp_00000000-...` | (sem descrição) | 2026-02-20 |

**Alerta:** Apenas 4 secrets no Vault, mas o Diagnóstico 4 identificou **13+ credenciais em texto plano** nas tabelas. A maioria das credenciais NÃO está no Vault.

### 14.6 Mapeamento de Workers e Credenciais

| Worker | Plataforma | Credencial provável | Bypass RLS |
|--------|-----------|---------------------|------------|
| `ingestion-worker` | Railway | `service_role` key | ✅ |
| `analytics-worker` | Railway | `service_role` key | ✅ |
| `journeys-worker` | Railway | `service_role` key | ✅ |
| `journeys-event-worker` | Railway | `service_role` key | ✅ |
| `email-campaign-worker` | Railway | `service_role` key | ✅ |
| `eduzz-enrichment` | Railway | `service_role` key | ✅ |
| `eduzz-receiver-webhook` | Supabase Edge | `service_role` key (via Vault) | ✅ |
| `eduzz-processor-worker` | Supabase Edge | `service_role` key (via Vault) | ✅ |
| `integrations-worker` | Supabase Edge | `service_role` key (via Vault) | ✅ |

**NOTA:** Env vars do Railway e secrets de Edge Functions precisam ser levantados manualmente via dashboard para confirmar.

### 14.7 Resumo de Risco — Diagnóstico 5

| Achado | Severidade | Ação |
|--------|------------|------|
| Todos os workers usam `service_role` (BYPASSRLS) | **CRITICA** | Criar roles dedicados por worker com grants mínimos |
| `supabase_read_only_user` tem BYPASSRLS | **ALTA** | Verificar quem usa este role |
| `supabase_etl_admin` tem BYPASSRLS | **ALTA** | Verificar se é necessário |
| `supabase_functions_admin` pode criar roles | **MEDIA** | Verificar se é necessário |
| `cli_login_postgres` expirado | **BAIXA** | Remover role |
| 20 colunas com credenciais em texto (4-5 tenants) | **ALTA** | Sprint 4 — migração incremental planejada |
| `service_role_key` está no Vault | **INFO** | Positivo — usado por Edge Functions via pg_net |

---

## 15. Auditoria e Logging

> **Diagnóstico executado:** 2026-03-09 — diagnostic-6.sql
> **Sprint 5 executado:** 2026-03-10 — audit.trail + 11 triggers

### 15.1 Schema audit

**Resultado: ✅ IMPLEMENTADO (Sprint 5)**

O schema `audit` contém:
- `audit.trail` — tabela centralizada de auditoria (17 colunas, 7 índices, RLS habilitado)
- 5 tabelas de backup (Sprints 1-3): `grant_backup_sprint1`, `function_backup_sprint1`, `view_backup_sprint1`, `definer_classification`, `function_backup_sprint3`
- 7 funções: 2 triggers (`log_change`, `log_change_simple`) + 5 helpers

### 15.2 Tabelas de Auditoria e Log

| Schema | Tabela | Tipo | Status |
|--------|--------|------|--------|
| `audit` | `trail` | **Auditoria centralizada (Sprint 5)** | ✅ Ativo — 11 triggers |
| `auth` | `audit_log_entries` | Auth audit (Supabase) | ⚠️ VAZIA (0 registros) |
| `core` | `permission_audit_log` | Log de mudanças de permissão | ✅ Pré-existente |
| `commerce` | `merge_history` | Histórico de merges | ✅ Pré-existente |
| `crm` | `lead_import_history` | Histórico de imports | ✅ Pré-existente |
| `crm` | `opportunity_stage_history` | Histórico de stages | ✅ Pré-existente |
| `analytics` | `fallback_log` | Log de fallback | ✅ Pré-existente |
| `integrations` | `logs` | Logs de integrações | ✅ Pré-existente |

### 15.3 Triggers de Auditoria

**Resultado: ✅ 11 triggers ativos (Sprint 5)**

| Prioridade | Schema | Tabela | Trigger | Operações |
|------------|--------|--------|---------|-----------|
| P0 | core | tenants | audit_tenants | INSERT, UPDATE, DELETE |
| P0 | core | tenant_users | audit_tenant_users | INSERT, UPDATE, DELETE |
| P0 | core | system_admins | audit_system_admins | INSERT, UPDATE, DELETE |
| P1 | email | tenant_config | audit_tenant_config | INSERT, UPDATE, DELETE |
| P1 | integrations | accounts | audit_integrations_accounts | INSERT, UPDATE, DELETE |
| P1 | commerce | asaas_config | audit_asaas_config | INSERT, UPDATE, DELETE |
| P2 | analytics | segments | audit_segments | INSERT, UPDATE, DELETE |
| P2 | journeys | journeys | audit_journeys | INSERT, UPDATE, DELETE |
| P2 | forms | forms | audit_forms | INSERT, UPDATE, DELETE |
| P2 | smart_forms | forms | audit_smart_forms | INSERT, UPDATE, DELETE |
| P3 | whatsapp | instances | audit_whatsapp_instances | INSERT, UPDATE, DELETE |

**Correção:** `core.tenant_users` (não `team_members`) é o nome correto da tabela de membros de equipe.

### 15.4 Tabelas Sensíveis SEM Audit Trigger (remanescentes)

**13 tabelas com dados sensíveis ainda sem trigger de auditoria (candidatas para Sprint futuro):**

| Schema | Tabelas | Motivo para futuro |
|--------|---------|-------------------|
| `analytics` | `contact_state`, `segment_parties` | Alto volume — avaliar impacto |
| `commerce` | `archived_transactions`, `checkout_orders`, `transactions` | Alto volume |
| `commerce` | `transaction_import_errors` | Baixa prioridade |
| `core` | `eduzz_integrations`, `team_invitations` | Legada / baixo risco |
| `crm` | `customers`, `opportunities`, `parties`, `party_person` | Alto volume — avaliar |
| `email` | `sender_identities` | Médio risco |

**Tabelas que GANHARAM audit trigger no Sprint 5:**
- `core.system_admins` (antes: nenhum trigger)
- `email.tenant_config`, `integrations.accounts`, `commerce.asaas_config` (credenciais)
- `analytics.segments`, `journeys.journeys`, `forms.forms`, `smart_forms.forms` (soft-delete tracking)
- `whatsapp.instances` (webhook_secret)

### 15.5 pgAudit Status

**pgAudit está INSTALADO mas NÃO CONFIGURADO:**

| Setting | Valor | Status |
|---------|-------|--------|
| `pgaudit.log` | `none` | ❌ Nenhum statement é auditado |
| `pgaudit.log_catalog` | `on` | — |
| `pgaudit.log_client` | `off` | — |
| `pgaudit.log_level` | `log` | — |
| `pgaudit.log_parameter` | `off` | ❌ Parâmetros não logados |
| `pgaudit.log_relation` | `off` | — |
| `pgaudit.log_rows` | `off` | — |
| `pgaudit.log_statement` | `on` | — |
| `pgaudit.role` | (vazio) | ❌ Sem role de audit configurado |

**Ação:** Configurar `pgaudit.log = 'write, ddl'` no mínimo para capturar todas as operações de escrita e alterações de schema.

### 15.6 Extensões de Segurança

| Extensão | Versão | Propósito |
|----------|--------|-----------|
| `pgcrypto` | 1.3 | ✅ Funções criptográficas |
| `supabase_vault` | 0.3.1 | ✅ Gerenciamento de secrets |
| `pg_stat_statements` | 1.11 | ✅ Performance monitoring |
| `pg_cron` | 1.6.4 | Agendamento de jobs |
| `pg_net` | 0.19.5 | HTTP requests do banco |
| `pg_trgm` | 1.6 | Fuzzy search |
| `plpgsql` | 1.0 | PL/pgSQL |
| `uuid-ossp` | 1.1 | Geração de UUIDs |

**Ausentes:**
- ❌ `pgaudit` — instalado mas `log = none` (efetivamente desabilitado)
- ❌ `pgsodium` — não listada (criptografia avançada)

### 15.7 Tabelas de Histórico/Eventos Existentes

**13 tabelas funcionam como log de eventos (sem ser auditoria formal):**

| Schema | Tabela | Tipo |
|--------|--------|------|
| `analytics` | `contact_events` | Eventos comportamentais de contatos |
| `analytics` | `fallback_log` | Log de fallback |
| `analytics` | `lifecycle_events` | Eventos de ciclo de vida |
| `commerce` | `asaas_webhook_events` | Webhooks do Asaas |
| `commerce` | `merge_history` | Histórico de merges |
| `core` | `eduzz_webhook_events` | Webhooks da Eduzz (legado) |
| `core` | `permission_audit_log` | **Único log de auditoria real** |
| `crm` | `lead_import_history` | Histórico de imports |
| `crm` | `opportunity_stage_history` | Mudanças de stage |
| `eduzz` | `webhook_events` | Webhooks da Eduzz |
| `email` | `email_events` | Eventos de email (opens, clicks) |
| `integrations` | `webhook_events` | Webhooks genéricos |
| `journeys` | `journey_events` | Eventos de journeys |

### 15.8 Auth Audit Log

**`auth.audit_log_entries` está VAZIA (0 registros).**

Supabase Auth não está gerando registros de auditoria de login/logout/signup. Possível causa: configuração padrão do Supabase ou limpeza automática.

### 15.9 Resumo de Risco — Diagnóstico 6

| Achado | Severidade | Ação | Status |
|--------|------------|------|--------|
| ~~Schema `audit` não existe~~ | ~~CRITICA~~ | ~~Criar schema + tabelas~~ | ✅ Sprint 5 |
| ~~Zero triggers de auditoria~~ | ~~CRITICA~~ | ~~Implementar triggers~~ | ✅ Sprint 5 (11 triggers) |
| `pgaudit.log = none` — efetivamente desabilitado | **ALTA** | Configurar `pgaudit.log = 'write, ddl'` | ⏸️ Avaliar necessidade |
| `auth.audit_log_entries` vazia | **ALTA** | Investigar configuração do Supabase Auth | ⏸️ Pendente |
| ~~`core.permission_audit_log` único log~~ | ~~MEDIA~~ | ~~Expandir padrão~~ | ✅ Sprint 5 (padrão expandido) |
| 13 tabelas de eventos sem padrão formal | **MEDIA** | Formalizar como parte do audit trail | ⏸️ Baixa prioridade |
| `pgaudit.log_parameter = off` | **MEDIA** | Habilitar para capturar valores | ⏸️ Avaliar |

---

# PARTE D — CONTROLES E COMPLIANCE

## 16. Control Matrix

| ID | Controle | Framework | Implementação | Owner | Status | Evidência |
|----|----------|-----------|---------------|-------|--------|-----------|
| C-001 | RLS em tabelas expostas | SOC 2, ISO | Policies SQL | Eng | ⚠️ | Query 2.4 |
| C-002 | MFA para admins | SOC 2, ISO | Supabase Auth | Eng | ❌ | - |
| C-003 | Audit trail | SOC 2, HIPAA, ISO | Schema audit | Eng | ✅ | Sprint 5 — `audit.trail` + 11 triggers |
| C-004 | Encryption at rest | SOC 2, HIPAA | Supabase | Infra | ✅ | Supabase docs |
| C-005 | Encryption in transit | SOC 2, HIPAA | TLS | Infra | ✅ | Supabase docs |
| C-006 | Access reviews | SOC 2, ISO | Manual | Ops | ❌ | - |
| C-007 | Incident response | SOC 2, ISO | Runbook | Ops | ❌ | - |
| C-008 | Backup/restore | SOC 2, ISO | PITR | Infra | ⚠️ | Verificar |
| C-009 | Vulnerability scanning | SOC 2, ISO | CI/CD | Eng | ❌ | - |
| C-010 | Least privilege | SOC 2, ISO | RLS, grants, INVOKER | Eng | ✅ | Sprint 1-3 |

## 17. Gaps Conhecidos (Ordenados por Severidade)

### 🔴 Crítico

| Gap | Descrição | Sprint |
|-----|-----------|--------|
| G-001 | ~~Tabelas multi-tenant sem RLS~~ | ~~Sprint 2~~ ✅ Sprint 2 |
| G-002 | ~~SECURITY DEFINER sem search_path~~ | ~~Sprint 3~~ ✅ Sprint 1 |
| G-003 | Credenciais de terceiros em texto (20 colunas, 4 tenants) | Sprint 4 — Fase 1 completa, decisões pendentes |

### 🟠 Alto

| Gap | Descrição | Sprint |
|-----|-----------|--------|
| G-004 | MFA não enforced | Sprint 8 |
| G-005 | ~~Audit trail não existe~~ | ~~Sprint 5~~ ✅ Sprint 5 — `audit.trail` + 11 triggers em tabelas sensíveis |
| G-006 | ~~Sem testes de cross-tenant~~ | ~~Sprint 2~~ ✅ Sprint 2 (SQL validated) |

### 🟡 Médio

| Gap | Descrição | Sprint |
|-----|-----------|--------|
| G-007 | ~~Views sem security_invoker~~ | ~~Sprint 3~~ ✅ Sprint 1 |
| G-008 | Sem SIEM integration | Sprint 12 |
| G-009 | Sem incident runbook | Sprint 16 |

### ⚪ Baixo

| Gap | Descrição | Sprint |
|-----|-----------|--------|
| G-010 | Sem bug bounty | Fase 3 |
| G-011 | Sem pentest formal | Sprint 18 |

---

## 18. Exceções Aprovadas

| ID | Exceção | Justificativa | Aprovador | Data | Revisão |
|----|---------|---------------|-----------|------|---------|
| E-001 | - | - | - | - | - |

---

# PARTE E — ROADMAP

## 19. Fase 1 — Fundação (Sprints 0-7)

### Sprint 0 — Diagnóstico ✅ Completo
- [x] Executar diagnóstico 1 (Schemas e PostgREST)
- [x] Executar diagnóstico 2 (RLS Coverage)
- [x] Executar diagnóstico 3 (SECURITY DEFINER)
- [x] Executar diagnóstico 4 (PII / Dados Sensíveis)
- [x] Executar diagnóstico 5 (Workers e Credenciais)
- [x] Executar diagnóstico 6 (Auditoria e Logging)
- [x] Popular inventário completo (Parte C)
- [ ] Priorizar gaps
- [ ] Levantar env vars Railway
- [ ] Verificar configuração Supabase Auth

### Sprint 1 — Hardening Fundacional ✅ Completo (2026-03-10)

**Phase 1 — Revoke Anon Grants:**
- [x] Backup de 479 grants em `audit.grant_backup_sprint1`
- [x] Revogar ALL de `anon` em 12 schemas (core, commerce, crm, analytics, email, journeys, doare, community, public, inbox, smart_forms)
- [x] Manter SELECT em `content` (landing pages) e SELECT+INSERT em `forms` (submissões públicas)
- [x] Verificar RLS ativo nas 8 tabelas com grants mantidos
- **Resultado:** 469 grants revogados, 10 mantidos (todos com RLS)

**Phase 2 — Fix SECURITY DEFINER search_path:**
- [x] Backup de 27 funções em `audit.function_backup_sprint1`
- [x] `ALTER FUNCTION ... SET search_path = ''` em 27 funções (analytics: 12, content: 3, forms: 3, integrations: 6, public: 2, whatsapp: 1)
- [x] Incluído `public.handle_new_user` (não listado originalmente)
- **Resultado:** 0 funções DEFINER sem search_path seguro

**Phase 3 — Criar schema app_auth:**
- [x] Criar schema `app_auth` com 11 helper functions
- [x] `active_tenant_id()`, `current_user_id()`, `current_role()`, `membership_id()`
- [x] `is_owner()`, `is_admin_or_above()`, `is_member_or_above()`
- [x] `is_authenticated()`, `is_system_admin()`, `is_aal2()`, `perm_version()`
- [x] Todas SECURITY INVOKER + `search_path = ''`
- [x] Grants: `authenticated` (all), `anon` (só `is_authenticated`), `service_role` (all)

**Phase 4 — Fix Views + Cleanup:**
- [x] Backup de views em `audit.view_backup_sprint1`
- [x] Recriar `analytics.v_segment_base` com `security_invoker = true`
- [x] Recriar `analytics.v_segment_leads_base` com `security_invoker = true`
- [x] Recriar `crm.vw_parties_unified` com `security_invoker = true`
- [x] ~~DROP cli_login_postgres~~ — Supabase impede (role expirado, bypassrls=false, sem risco)

**Phase 5 — Validação:**
- [x] Check 1: Grants anon — PASS (8 app tables, todas com RLS)
- [x] Check 2: DEFINER search_path — PASS (0 vulneráveis)
- [x] Check 3: Schema app_auth — PASS (11 funções)
- [x] Check 4: Views security_invoker — PASS (3/3)
- [x] Check 5: Roles expirados — WARN (Supabase impede DROP, sem risco)
- [x] Check 6: Tabelas críticas — PASS (admin tables blocked from anon)

### Sprint 2 — RLS Sistemático ✅ Completo (2026-03-10)

**Objetivo:** Habilitar RLS em 66 tabelas multi-tenant sem proteção.

**Resultado:** 0 tabelas multi-tenant sem RLS. 17/17 schemas com 100% coverage.

**Fases executadas:**

| Fase | Descrição | Severidade | Status |
|------|-----------|------------|--------|
| 1 | Diagnóstico atualizado | — | ✅ |
| 2 | RLS no schema analytics (57 tabelas: 9 principais + 49 partições) | 🔴 CRÍTICO | ✅ |
| 3 | RLS em journeys (3), crm (1), integrations (1) | 🟠 ALTO | ✅ |
| 4 | RLS no schema sympla (4 tabelas) | 🟡 MÉDIO | ✅ |
| 5 | Índices para performance de RLS | — | ✅ |
| 6 | Validação e fix de policies órfãs | — | ✅ |

**Checklist:**
- [x] Fase 1: Diagnóstico confirmou 66 tabelas sem RLS (57 analytics, 3 journeys, 1 crm, 1 integrations, 4 sympla)
- [x] Fase 2: RLS + policies em 9 tabelas analytics principais (4 migrations)
- [x] Fase 2: RLS habilitado explicitamente em 49 partições (PostgreSQL NÃO propaga `relrowsecurity` para partições existentes)
- [x] Fase 3: RLS + policies em journeys (3), crm (1), integrations (1)
- [x] Fase 4: RLS + policies em sympla (4)
- [x] Fase 5: 5 índices simples faltantes + 5 compostos + 3 parciais + ANALYZE em 11 tabelas
- [x] Fase 6: Validação — 0 tabelas multi-tenant sem RLS, 0 tabelas com RLS sem policies
- [x] Fase 6: Fix de 2 tabelas pré-existentes com RLS mas sem policies (`product_link_jobs`, `job_errors`)

**Lições aprendidas:**
- PostgreSQL **NÃO** propaga `relrowsecurity` para partições existentes — cada partição precisa de `ENABLE ROW LEVEL SECURITY` explícito
- Policies na tabela pai **SÃO** herdadas pelas partições (não precisam ser criadas em cada partição)
- `contact_events` usa `event_at` (não `created_at`) — verificar nomes de colunas antes de criar índices
- Validação encontrou 2 tabelas órfãs (`product_link_jobs`, `job_errors`) com RLS pré-Sprint 2 mas sem policies

**Números finais:**
- 219 tabelas com RLS (total no banco)
- 126 tabelas protegidas nos 5 schemas Sprint 2
- 267 policies nos 5 schemas Sprint 2
- 0 tabelas multi-tenant sem RLS
- [ ] Monitorar performance 24-48h (pg_stat_statements, sequential scans)

**Rollback:** `ALTER TABLE schema.table DISABLE ROW LEVEL SECURITY;` + `DROP POLICY IF EXISTS ...`

**Migration files (10 no total):**
- `20260310164733_security_sprint2_phase2_analytics_indexes.sql`
- `20260310164746_security_sprint2_phase2_analytics_enable_rls.sql`
- `20260310164802_security_sprint2_phase2_analytics_policies.sql`
- `20260310164850_security_sprint2_phase2_analytics_partitions_rls.sql`
- `20260310170653_security_sprint2_phase3_journeys_crm_integrations.sql`
- `20260310170812_security_sprint2_phase4_sympla_rls.sql`
- `20260310171122_security_sprint2_phase5_indexes_v2.sql`
- `20260310171324_security_sprint2_phase6_fix_missing_policies.sql`

### Sprint 3 — SECURITY DEFINER Audit ✅

**Objetivo:** Auditar e reduzir funções SECURITY DEFINER. INVOKER deve ser o padrão; DEFINER é exceção com justificativa.

**Resultado:** 319 DEFINER (66%) → **250 DEFINER (51%)** — 69 funções migradas para INVOKER, 0 erros.

**Fases executadas:**

| Fase | Descrição | Resultado |
|------|-----------|-----------|
| 1 | Diagnóstico e categorização automática | 319 DEFINER em 16 schemas |
| 2 | Classificação em `audit.definer_classification` | 70 migrate, 80 keep, 169 evaluate |
| 2.5 | Correções manuais (16 funções reclassificadas) | anon-callable, workers, sistema |
| 3 | Backup + migração para INVOKER | 70 migradas, 0 erros |
| 4 | Validação | 6/6 checks PASS |

**Classificação final:**

| Decisão | Quantidade | % |
|---------|-----------|---|
| evaluate (futuro) | 169 | 53.0% |
| keep_definer (justificado) | 80 | 25.1% |
| migrate_invoker (migrado) | 70 | 21.9% |

**Schemas 100% DEFINER restantes (justificados):**
- `core` (30) — funções de sistema/tenant management
- `eduzz` (5) — provider legado, acesso interno
- `pgbouncer` (1) — infraestrutura

**Correções manuais aplicadas (16 funções):**

| Função | Decisão | Motivo |
|--------|---------|--------|
| `forms.get_public_form` | keep_definer | Chamada por `anon` |
| `content.get_landing_page_by_slug` | keep_definer | Chamada por `anon` |
| `commerce.get_checkout_data` | keep_definer | Chamada por `anon` |
| `commerce.get_customer_billing_data` | keep_definer | Chamada por `anon` |
| `email.increment_*` (5 funções) | keep_definer | Workers/webhooks |
| `public.get_tenant_id` | keep_definer | Dados de sistema |
| `public.get_user_id_by_email` | keep_definer | Dados de sistema |
| `public.get_tenant_by_sendgrid_message` | keep_definer | Dados de sistema |
| `public.get_customer_ids_paginated` | keep_definer | Worker cross-tenant |
| `public.get_jobs_count_grouped` | keep_definer | Admin cross-tenant |
| `inbox.create_nylas_account` | evaluate | Categorização incorreta |
| `inbox.disconnect_nylas_account` | evaluate | Categorização incorreta |

**Rollback:** `audit.function_backup_sprint3` (74 rows) permite reverter qualquer função via `ALTER FUNCTION ... SECURITY DEFINER`.

**Migrations:**
- `20260310194050_security_sprint3_phase2_classification.sql`
- `20260310195754_security_sprint3_phase3_backup.sql`
- `20260310195808_security_sprint3_phase3_migrate_to_invoker.sql`

- [x] Fase 1 — Diagnóstico e categorização
- [x] Fase 2 — Classificação em `audit.definer_classification`
- [x] Review manual de funções `migrate_invoker`
- [x] Fase 3 — Backup + migração para INVOKER
- [x] Fase 4 — Validação
- [ ] Testes funcionais (login, dashboard, segmentos, jornadas, CRM)
- [x] Atualizar SECURITY.md com métricas finais

### Sprint 4 — Secrets Management

**Objetivo:** Migrar credenciais de texto plano para Supabase Vault / campo criptografado. Complexidade: 🔴 ALTA — requer mudanças em workers e Edge Functions.

**Contexto (Sprint 0):** 20 colunas com credenciais em 7 tabelas, apenas 4 secrets no Vault. Se DB comprometido, atacante acessa APIs Eduzz/SendGrid/Nylas/webhooks de todos os tenants (4-5 tenants afetados).

**Estratégia recomendada: Híbrida**
- **Secrets globais** (client_id, client_secret da plataforma) → Vault direto
- **Secrets por tenant** (access_token, webhook_secret) → Campo criptografado com chave mestra no Vault

**Tabelas afetadas:**

| Tabela | Colunas | Criticidade |
|--------|---------|-------------|
| `eduzz.integrations` | api_key, access_token, refresh_token, client_secret, webhook_secret, public_key, token_expires_at | 🔴 CRÍTICA — NÃO MIGRAR (legada, planejar DROP) |
| `core.eduzz_integrations` | api_key, webhook_secret, public_key | 🔴 CRÍTICA — legada (duplicada) |
| `email.tenant_config` | sendgrid_subuser_api_key, webhook_secret | 🔴 CRÍTICA |
| `integrations.accounts` | access_token, refresh_token, client_secret | 🟠 ALTA |
| `whatsapp.instances` | webhook_secret | 🟡 MÉDIA |
| `commerce.asaas_config` | webhook_token | 🟡 MÉDIA (api_key já criptografado) |

**Fases:**

| Fase | Descrição | Tempo est. |
|------|-----------|-----------|
| 1 | Diagnóstico: mapear quem usa cada credencial | 1-2h |
| 2 | Infraestrutura: schema `secrets`, funções encrypt/decrypt, chave mestra no Vault | 2-4h |
| 3a | Migração: `email.tenant_config` (menor risco, maior impacto) | 4-8h |
| 3b | Migração: `integrations.accounts` (modelo correto) | 4-8h |
| 3c | Limpeza: tabelas legadas `eduzz.integrations`, `core.eduzz_integrations` | 2-4h |
| 4 | Validação: testar integrações, confirmar criptografia, dropar colunas antigas | 2-4h |

**Infraestrutura: funções helper**
- `secrets.encrypt(p_plaintext, p_key_name)` → bytea (SECURITY DEFINER, search_path = '')
- `secrets.decrypt(p_ciphertext, p_key_name)` → text (SECURITY DEFINER, search_path = '')
- Chave mestra: AES-256-CBC via pgcrypto, armazenada no Vault

**Código afetado:**
- `email-campaign-worker` — `tenantConfig.sendgrid_subuser_api_key` → `decryptSecret()`
- `historical-sync-worker` — `integration.access_token` → `getDecryptedToken()`
- Edge Functions — `integration.webhook_secret` → `decryptSecret()`

**Riscos:**

| Risco | Mitigação |
|-------|-----------|
| Quebrar integrações ativas | Migração incremental, manter colunas antigas em paralelo |
| Perder credenciais | Backup antes de dropar colunas |
| Performance decrypt | Cache em memória nos workers |

**Recomendação:** Dado a complexidade, executar Sprint 4 em fases incrementais (4a: email, 4b: integrations, 4c: cleanup). Considerar priorizar Sprint 5 (Audit) se compliance for mais urgente.

**Status:** ⏳ Fase 1 (diagnóstico) completa. Execução adiada — requer mudanças em workers/Edge Functions.

- [x] Fase 1 — Diagnóstico de credenciais (20 colunas, 7 tabelas, 4 tenants)
- [ ] Decisão arquitetural: estratégia de criptografia
- [ ] Fase 2 — Infraestrutura (schema `secrets`, funções encrypt/decrypt, chave mestra no Vault)
- [ ] Fase 3a — Migração `email.tenant_config` (sendgrid_subuser_api_key)
- [ ] Fase 3b — Migração `integrations.accounts` (access_token, refresh_token, client_secret)
- [ ] Fase 3c — Consolidar `integrations.accounts.config` JSONB (remover duplicação)
- [ ] Fase 3d — DROP tabelas legadas (`eduzz.integrations`, `core.eduzz_integrations`)
- [ ] Fase 4 — Validação e cleanup de colunas antigas

**Decisões pendentes:**
1. Estratégia: campo criptografado (pgcrypto + chave no Vault) vs Vault direto vs híbrido
2. Ordem: `email.tenant_config` primeiro (4 tenants, menor risco) ou `integrations.accounts` (modelo correto)
3. Consolidar JSONB config ou manter duplicação temporária
4. Timeline para DROP de tabelas legadas

**Dependências:**
- Atualização de workers: `email-campaign-worker`, `historical-sync-worker`
- Atualização de Edge Functions: `sendgrid-webhook`, `eduzz-receiver-webhook`
- Testes de integração antes de dropar colunas antigas

### Sprint 5 — Audit Schema ✅ Completo (2026-03-10)

**Objetivo:** Criar infraestrutura de auditoria para compliance — registro imutável de mudanças em tabelas sensíveis.

**Contexto:** Sprint 0 identificou: schema `audit` não existia, `pgaudit.log = none`, 0 triggers de auditoria, 24 tabelas sensíveis sem rastreamento. Sprint 1 criou schema `audit` para backups temporários.

**Resultado:** 11 triggers de auditoria em tabelas sensíveis, tabela `audit.trail` centralizada, 7 funções, RLS habilitado.

**Fases executadas:**

| Fase | Descrição | Resultado |
|------|-----------|-----------|
| 1 | Diagnóstico | pgaudit não instalado, 0 triggers existentes, `core.tenant_users` (não `team_members`) |
| 2 | Infraestrutura | `audit.trail` (17 colunas, 7 índices, RLS), `log_change()` + `log_change_simple()` |
| 3 | Triggers P0/P1/P2/P3 | 11 triggers em tabelas sensíveis (INSERT + UPDATE + DELETE) |
| 4 | Helpers + validação | 5 funções helper, 6/6 checks PASS |

**Métricas finais:**

| Componente | Quantidade |
|------------|-----------|
| Triggers de auditoria | 11 |
| Índices em audit.trail | 7 |
| Políticas RLS | 2 (service_role full, authenticated por tenant) |
| Funções trigger | 2 (log_change, log_change_simple) |
| Funções helper | 5 (get_record_history, get_tenant_changes, get_user_changes, get_stats, export_trail) |

**Descobertas:**
- `core.tenant_users` é o nome correto (não `team_members` como documentado no Sprint 0)
- `core.permission_audit_log` já existia como log pré-existente
- `whatsapp.instances` existe e foi incluída (P3)
- pgaudit: instalado no Supabase mas `log = none` (efetivamente desabilitado)

**Retenção:**
- Atual: sem cleanup automático
- Recomendação futura: particionamento mensal após 6 meses, archiving após 1 ano

**pgaudit:** Mais granular (cobre SELECTs) mas gera muito log. Recomendação: começar com triggers, avaliar pgaudit depois.

- [x] Fase 1 — Diagnóstico (pgaudit não instalado, 0 triggers existentes)
- [x] Fase 2 — Infraestrutura (`audit.trail`, funções, índices, RLS)
- [x] Fase 3 — Triggers em 11 tabelas sensíveis (P0/P1/P2/P3)
- [x] Fase 4 — Helpers + validação (6/6 checks PASS)
- [x] Atualizar SECURITY.md para v1.3

### Sprint 6 — Schema Strategy (Opcional)
- [ ] Criar schema `api`
- [ ] Migrar para views SECURITY INVOKER
- [ ] Reduzir exposed_schemas

### Sprint 7 — Hardening Final Fase 1
- [ ] Revisão de gaps remanescentes
- [ ] Documentação atualizada
- [ ] Checklist de PR finalizado

---

## 20. Fase 2 — Enterprise Readiness (Sprints 8-17)

### Sprint 8-9 — Identity Enterprise
- [ ] SSO/SAML integration
- [ ] MFA enforcement por tenant
- [ ] Session policies
- [ ] IP allowlisting
- [ ] Step-up auth

### Sprint 10-11 — Criptografia & Secrets
- [ ] Field-level encryption para PII
- [ ] Vault centralizado
- [ ] Key rotation
- [ ] Avaliar BYOK

### Sprint 12-13 — Auditoria Enterprise
- [ ] Audit logs exportáveis por tenant
- [ ] Retention configurável
- [ ] SIEM integration
- [ ] Immutable audit trail

### Sprint 14-15 — Isolamento Avançado
- [ ] Rate limiting granular
- [ ] Resource quotas
- [ ] Noisy neighbor protection

### Sprint 16-17 — Compliance Program
- [ ] Control matrix completa
- [ ] Incident response runbook
- [ ] DR/restore drill
- [ ] Access review process

---

## 21. Fase 3 — Certification (Sprints 18+)

### Sprint 18 — Penetration Test
- [ ] Contratar empresa terceirizada
- [ ] Executar pentest
- [ ] Remediar findings
- [ ] Retest

### Sprint 19+ — SOC 2 Type II
- [ ] Engajar auditor
- [ ] Período de observação (6 meses)
- [ ] Coleta de evidências
- [ ] Audit
- [ ] Certification

### Sprint 20+ — ISO 27001
- [ ] Gap assessment
- [ ] ISMS implementation
- [ ] Certification audit

---

# PARTE F — OPERAÇÃO

## 22. Guardrails de Engenharia

### G1 — Toda tabela exposta nasce com checklist

Ao criar tabela em schema exposto, PR deve incluir:
- [ ] RLS habilitado
- [ ] Policy definida
- [ ] Grants mínimos
- [ ] Índices compatíveis com policy
- [ ] Teste de isolamento
- [ ] Classificação de dados (PII? Financeiro?)

### G2 — PR de banco exige security review

Template obrigatório:
```
## Security Checklist
- [ ] Tabela é exposta via API?
- [ ] RLS está habilitado?
- [ ] Policy usa `app_auth.active_tenant_id()`?
- [ ] SECURITY DEFINER justificado?
- [ ] Contém PII? Qual?
- [ ] Precisa de audit trigger?
- [ ] Índice para policy existe?
```

### G3 — SECURITY DEFINER requer aprovação

Processo:
1. Justificativa escrita
2. Aprovação de security owner
3. search_path explícito obrigatório
4. Teste dedicado
5. Owner restrito

### G4 — Chaves e secrets

- `anon` → frontend apenas
- `service_role` → backend interno apenas
- Credenciais de terceiros → Vault ou secret manager
- Nunca em código, logs ou tabelas comuns

### G5 — Acesso privilegiado é rastreável

Ações que geram audit log obrigatório:
- Support mode / impersonation
- Export massivo
- Alteração de billing
- Alteração de permissões
- Leitura de audit logs
- Operações cross-tenant

---

## 23. PR Checklist para Banco

```markdown
## Database Change Checklist

### Basics
- [ ] Migration tem rollback definido
- [ ] Testado em ambiente de staging

### Security
- [ ] RLS habilitado se tabela é exposta
- [ ] Policy usa helpers centralizados
- [ ] SECURITY INVOKER é o default
- [ ] SECURITY DEFINER tem aprovação + search_path
- [ ] Grants são mínimos necessários

### Data Classification
- [ ] Não adiciona PII sem aprovação
- [ ] Credenciais vão para Vault
- [ ] JSONB não contém PII não mapeado

### Performance
- [ ] Índices para colunas em policies
- [ ] EXPLAIN ANALYZE para queries críticas

### Audit
- [ ] Trigger de audit se tabela é sensível
- [ ] Retention definida se aplicável
```

---

## 24. Incident Response (Draft)

### Severidades

| Severidade | Descrição | Response Time | Exemplos |
|------------|-----------|---------------|----------|
| SEV-1 | Breach confirmado | Imediato | Dados vazados, acesso não autorizado |
| SEV-2 | Vulnerabilidade ativa | < 1h | Exploit em produção |
| SEV-3 | Vulnerabilidade identificada | < 24h | Falha de segurança sem exploit |
| SEV-4 | Melhoria de segurança | < 1 semana | Gap identificado em review |

### Processo SEV-1/SEV-2

1. **Containment** (imediato)
   - Isolar sistema afetado
   - Revogar credenciais comprometidas
   - Preservar evidências

2. **Assessment** (< 1h)
   - Escopo do incidente
   - Dados afetados
   - Tenants afetados

3. **Communication** (< 4h)
   - Notificar stakeholders internos
   - Preparar comunicação para clientes

4. **Remediation**
   - Fix técnico
   - Verificação
   - Deploy

5. **Post-mortem** (< 1 semana)
   - Timeline
   - Root cause
   - Lessons learned
   - Action items

---

## 25. Access Review Process (Draft)

### Frequência

| Tipo | Frequência | Owner |
|------|------------|-------|
| Produção (banco) | Trimestral | CTO |
| Railway | Trimestral | CTO |
| Supabase dashboard | Trimestral | CTO |
| GitHub | Trimestral | CTO |
| Third-party services | Semestral | CTO |

### Checklist

- [ ] Listar todos os acessos atuais
- [ ] Verificar se cada acesso ainda é necessário
- [ ] Remover acessos desnecessários
- [ ] Documentar justificativa para acessos mantidos
- [ ] Atualizar matriz de acesso

---

# PARTE G — HISTÓRICO

## 26. Changelog

### 2026-03-10 — v1.4
- Adicionada seção "Estado Atual e Próximos Passos" no topo para continuidade entre sessões
- Resumo executivo: sprints completos, pendentes, bloqueadores
- Inventário de credenciais consolidado com status por tabela
- Decisões pendentes documentadas (D-SEC-1/2/3)
- Roadmap futuro até Sprint 18+
- Seção 15.9 atualizada com status dos achados resolvidos pelo Sprint 5

### 2026-03-10 — v1.3
- **Sprint 5 — Audit Schema COMPLETO** ✅
- Tabela `audit.trail` criada (17 colunas, 7 índices, RLS habilitado)
- 11 triggers de auditoria em tabelas sensíveis (P0/P1/P2/P3)
- 7 funções: 2 triggers (`log_change`, `log_change_simple`) + 5 helpers
- Funções helper: `get_record_history`, `get_tenant_changes`, `get_user_changes`, `get_stats`, `export_trail`
- RLS: service_role full access, authenticated isolado por tenant
- Correção: `core.tenant_users` (não `team_members`)
- Descoberta: `core.permission_audit_log` já existia (log de permissões)
- Gap G-005 resolvido, Compliance C-003 marcado ✅

### 2026-03-10 — v1.2
- **Sprint 4 Fase 1 — Diagnóstico de Credenciais COMPLETO** ✅
- 20 colunas com credenciais identificadas em 7 tabelas
- Escala real: apenas 4 tenants afetados (menor do que estimado no Sprint 0)
- Descoberta: `integrations.accounts.config` JSONB também contém secrets (duplicação com colunas)
- Tabelas legadas confirmadas para DROP: `core.eduzz_integrations`, `eduzz.integrations`
- Vault atual: apenas 4 secrets (supabase_url, service_role_key, whatsapp_*, DOARE_API_URL)
- Funções que acessam credenciais: `eduzz_get_integration`, `get_tenant_by_sendgrid_message`, `get_tenant_email_config`
- Sprint 4 completo adiado — requer mudanças em workers + decisões arquiteturais

### 2026-03-10 — v1.1
- **Sprint 3 — SECURITY DEFINER Audit COMPLETO** ✅
- 319 DEFINER (66%) → 250 DEFINER (51%) — 69 funções migradas para INVOKER
- Classificação formal de todas as 319 funções em `audit.definer_classification`
- 80 funções keep_definer com justificativa documentada
- 16 correções manuais (anon-callable, workers, sistema, cross-tenant)
- 0 erros na migração, 0 funções DEFINER sem search_path
- Schemas 100% DEFINER restantes justificados: core (30), eduzz (5), pgbouncer (1)
- 169 funções em `evaluate` para revisão futura
- 3 migrations aplicadas + versionadas no repositório
- Compliance C-010 (Least privilege) marcado como ✅

### 2026-03-10 — v1.0
- **Sprint 2 — RLS Sistemático COMPLETO** ✅
- 66 tabelas multi-tenant protegidas com RLS (57 analytics, 3 journeys, 1 crm, 1 integrations, 4 sympla)
- 49 partições com RLS habilitado explicitamente (PostgreSQL não propaga `relrowsecurity`)
- 2 tabelas órfãs corrigidas (`product_link_jobs`, `job_errors` — RLS pré-existente sem policies)
- 13 índices novos (5 simples + 5 compostos + 3 parciais) + ANALYZE em 11 tabelas
- 17/17 schemas com 100% RLS coverage. 219 tabelas com RLS. 267 policies nos schemas Sprint 2.
- Gaps G-001 e G-006 resolvidos
- 10 migrations aplicadas + versionadas no repositório
- 18 migration files do Sprint 1 versionados no repositório
- Gaps G-002 e G-007 marcados como resolvidos no Sprint 1

### 2026-03-10 — v0.8
- **Sprint 1 — Hardening Fundacional COMPLETO** ✅
- Phase 1: 469 anon grants revogados (de 479 → 10 mantidos com RLS)
- Phase 2: 27 SECURITY DEFINER functions corrigidas com `search_path = ''`
- Phase 3: Schema `app_auth` criado com 11 helper functions (INVOKER + safe search_path)
- Phase 4: 3 views recriadas com `security_invoker = true`
- Phase 5: 5/6 checks PASS, 1 WARN (cli_login_postgres: Supabase impede DROP)
- Backups em `audit.grant_backup_sprint1`, `audit.function_backup_sprint1`, `audit.view_backup_sprint1`
- 15 migrations aplicadas no total

### 2026-03-09 — v0.7
- Diagnóstico 6 executado e documentado (Auditoria e Logging)
- Parte C seção 15 populada: schema `audit` não existe, zero triggers
- pgAudit instalado mas `log = none` (efetivamente desabilitado)
- `auth.audit_log_entries` vazia — sem registro de login/logout
- 24 tabelas sensíveis sem qualquer trigger de auditoria
- **Sprint 0 — Parte C (Inventário) COMPLETA** ✅

### 2026-03-09 — v0.6
- Diagnóstico 5 executado e documentado (Workers e Credenciais)
- Parte C seção 14 populada: 5 roles com BYPASSRLS, todos workers usam service_role
- Apenas 4 secrets no Vault vs 13+ credenciais em texto plano
- `cli_login_postgres` expirado identificado para remoção

### 2026-03-09 — v0.5
- Diagnóstico 4 executado e documentado (PII / Dados Sensíveis)
- Parte C seção 13 populada: 400 colunas PII em 19 schemas
- 13+ credenciais em texto plano identificadas (CRITICO)
- ~160 colunas JSONB com PII potencialmente não mapeado
- CPF/CNPJ em 8+ tabelas sem encryption

### 2026-03-09 — v0.4
- Diagnóstico 3 executado e documentado (SECURITY DEFINER)
- Parte C seção 12 populada: 319 funções DEFINER (66% do total)
- 26 funções DEFINER sem `search_path` seguro identificadas (CRITICO)
- 3 views sem `security_invoker` identificadas
- `public` schema com 113 funções DEFINER — maior superfície de ataque

### 2026-03-09 — v0.3
- Diagnóstico 2 executado e documentado (RLS Coverage)
- Parte C seção 11 populada: 149 tabelas COM RLS, 66 SEM RLS
- `analytics` é o maior gap: 57 tabelas multi-tenant sem RLS (26.9% coverage)
- 498 policies existentes auditadas e categorizadas

### 2026-03-09 — v0.2
- Diagnóstico 1 executado e documentado (Schemas, Grants, PostgREST)
- Parte C seção 10 populada com resultados reais
- Identificados achados CRÍTICOS: `anon` com FULL ACCESS em tabelas sensíveis

### 2026-03-09 — v0.1
- Criação inicial do documento
- Estrutura definida
- Sprint 0 iniciado
- Diagnósticos SQL criados

---

# APÊNDICES

## A. Glossário

| Termo | Definição |
|-------|-----------|
| RLS | Row-Level Security — políticas que filtram dados por linha |
| SECURITY DEFINER | Função executada com privilégios do owner, não do caller |
| SECURITY INVOKER | Função executada com privilégios do caller |
| PostgREST | API REST automática do Supabase sobre PostgreSQL |
| service_role | Role Supabase que bypassa RLS |
| PII | Personally Identifiable Information — dados que identificam indivíduo |

## B. Referências

- [Supabase Security Best Practices](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase Hardening Guide](https://supabase.com/docs/guides/auth/hardening)
- [SOC 2 Trust Services Criteria](https://www.aicpa.org/soc)
- [ISO 27001 Controls](https://www.iso.org/isoiec-27001-information-security.html)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
