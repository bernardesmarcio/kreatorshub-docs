# KreatorsHub — Security Architecture & Compliance Roadmap

## Estado: Sprint 1 — Completo ✅
## Última atualização: 2026-03-10
## Versão: 0.8

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

**Status:** ❌ Não existe — Sprint 1

**Schema alvo:** `app_auth`

```sql
-- Funções helper para policies
app_auth.active_tenant_id() → uuid
app_auth.current_role() → text
app_auth.is_owner() → boolean
app_auth.is_admin_or_above() → boolean
app_auth.is_aal2() → boolean
```

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

**Status:** ⚠️ Parcial — Sprint 2

**Padrão de policy:**
```sql
-- Tabelas tenant-scoped
CREATE POLICY "tenant_isolation" ON schema.table
  USING (tenant_id = app_auth.active_tenant_id())
  WITH CHECK (tenant_id = app_auth.active_tenant_id());
```

**Coverage atual:** Ver seção 11 (Inventário)

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

**Status:** ⚠️ A auditar — Sprint 0

**Alvo:**
- `api` → único schema exposto (views SECURITY INVOKER)
- Todos os outros → schemas privados

**Estado atual:** Ver seção 10 (Inventário)

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

**Status:** ⚠️ A auditar — Sprint 0

| Worker | Credencial atual | Credencial alvo |
|--------|------------------|-----------------|
| ingestion-worker | ? | Dedicada |
| analytics-worker | ? | Dedicada |
| journeys-worker | ? | Dedicada |
| journeys-event-worker | ? | Dedicada |
| email-campaign-worker | ? | Dedicada |
| eduzz-enrichment | ? | Dedicada |

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

**Status:** ❌ Não existe — Sprint 5

**Estrutura alvo:**
```sql
CREATE SCHEMA audit;

CREATE TABLE audit.row_changes (
  id uuid PRIMARY KEY,
  table_schema text NOT NULL,
  table_name text NOT NULL,
  operation text NOT NULL,  -- INSERT, UPDATE, DELETE
  row_id uuid,
  old_data jsonb,
  new_data jsonb,
  changed_by uuid,
  tenant_id uuid,
  changed_at timestamptz DEFAULT now()
);

CREATE TABLE audit.sensitive_actions (
  id uuid PRIMARY KEY,
  action_type text NOT NULL,
  actor_id uuid,
  tenant_id uuid,
  target_type text,
  target_id uuid,
  metadata jsonb,
  ip_address inet,
  user_agent text,
  created_at timestamptz DEFAULT now()
);
```

### 9.2 Row-level Changes

**Tabelas que precisam de audit trigger:**
- `crm.parties` (PII)
- `crm.party_person` (PII)
- `commerce.transactions` (financeiro)
- `core.tenant_members` (permissões)
- `integrations.accounts` (credenciais)

### 9.3 Admin Activity Logs

**Ações a logar:**
- Login/logout
- Mudança de permissão
- Export de dados
- Acesso a dados de outro tenant (support)
- Alteração de configuração sensível

### 9.4 SIEM Integration

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

**SEVERIDADE: CRITICA**

O role `anon` (usuários NÃO autenticados) tem acesso direto a tabelas sensíveis:

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

### 10.5 Resumo de Risco — Diagnóstico 1

| Achado | Severidade | Ação |
|--------|------------|------|
| `anon` com FULL ACCESS em `core.tenants`, `core.system_admins` | **CRITICA** | Revogar grants imediatamente ou garantir RLS + deny-all para anon |
| `anon` com FULL ACCESS em `commerce.transactions`, `crm.customers` | **CRITICA** | Revogar grants desnecessários |
| `public` schema com 242 functions | **ALTA** | Auditar quais são callable via `/rpc/` e restringir |
| 11 schemas com grants para `anon`/`authenticated` | **ALTA** | Avaliar se todos precisam ser expostos |
| `authenticated` sem RLS = cross-tenant leakage | **CRITICA** | Depende do Diagnóstico 2 (RLS coverage) |

---

## 11. RLS Coverage

> **Diagnóstico executado:** 2026-03-09 — diagnostic-2.sql

### 11.1 Resumo por Schema

| Schema | Total Tables | Com RLS | Sem RLS | Coverage |
|--------|-------------|---------|---------|----------|
| `commerce` | 14 | 14 | 0 | **100%** |
| `community` | 5 | 5 | 0 | **100%** |
| `content` | 4 | 4 | 0 | **100%** |
| `core` | 19 | 19 | 0 | **100%** |
| `cron` | 2 | 2 | 0 | **100%** |
| `doare` | 2 | 2 | 0 | **100%** |
| `eduzz` | 7 | 7 | 0 | **100%** |
| `email` | 17 | 17 | 0 | **100%** |
| `forms` | 7 | 7 | 0 | **100%** |
| `imports` | 2 | 2 | 0 | **100%** |
| `inbox` | 4 | 4 | 0 | **100%** |
| `smart_forms` | 6 | 6 | 0 | **100%** |
| `whatsapp` | 3 | 3 | 0 | **100%** |
| `crm` | 29 | 28 | **1** | 96.6% |
| `integrations` | 8 | 7 | **1** | 87.5% |
| `journeys` | 7 | 4 | **3** | 57.1% |
| `analytics` | 78 | 21 | **57** | **26.9%** |
| `sympla` | 4 | 0 | **4** | **0%** |

**Total geral:** 149 tabelas COM RLS, 66 tabelas SEM RLS.

### 11.2 Tabelas COM RLS (149 tabelas)

RLS habilitado em todos os schemas principais: `commerce` (14), `core` (19), `crm` (28/29), `email` (17), `forms` (7), `smart_forms` (6), `integrations` (7/8), `community` (5), `content` (4), `eduzz` (7), `journeys` (4/7), `inbox` (4), `imports` (2), `whatsapp` (3), `doare` (2), `cron` (2), `analytics` (21/78).

### 11.3 Tabelas SEM RLS (Gaps)

#### CRITICO — Schemas potencialmente expostos (62 tabelas)

**analytics (57 tabelas sem RLS):**
- `contact_attribution`
- `contact_events` (parent table + 43 partitions: `contact_events_2024_01` a `contact_events_2027_03`, `contact_events_default`, `contact_events_pre2024`)
- `contact_product_stats` (parent + 8 partitions: `_p0` a `_p7`)
- `fallback_log`, `product_journey_analysis`, `segment_eval_queue`, `tenant_distribution`, `visitor_sessions`

**crm (1 tabela):**
- `opportunity_activities_archive`

**integrations (1 tabela):**
- `sync_schedules`

**journeys (3 tabelas):**
- `backfill_jobs`, `event_processing_jobs`, `processing_jobs`

#### MEDIO — Schemas internos (4 tabelas)

**sympla (4 tabelas):**
- `events`, `orders`, `participants`, `sync_cursors`

### 11.4 Tabelas Multi-tenant SEM RLS (CRITICO)

67 tabelas possuem coluna `tenant_id` mas NÃO têm RLS habilitado:

| Schema | Tabelas |
|--------|---------|
| `analytics` | `contact_attribution`, `contact_events` + 43 partitions, `contact_product_stats` + 8 partitions, `fallback_log`, `product_journey_analysis`, `segment_eval_queue`, `tenant_distribution`, `visitor_sessions` |
| `crm` | `opportunity_activities_archive` |
| `integrations` | `sync_schedules` |
| `journeys` | `backfill_jobs`, `event_processing_jobs`, `processing_jobs` |
| `sympla` | `events`, `orders`, `participants`, `sync_cursors` |

**Impacto:** Estas tabelas contêm dados de múltiplos tenants e podem ser acessadas sem filtro de tenant via API REST (se o schema tiver grants). O `analytics` schema é o mais crítico com 57 tabelas expostas contendo dados de eventos e comportamento de contatos.

### 11.5 Policies Existentes (498 policies em 149 tabelas)

#### Resumo por schema

| Schema | Tabelas c/ policies | Total policies |
|--------|-------------------|----------------|
| `analytics` | 17 | 51 |
| `commerce` | 14 | 27 |
| `community` | 5 | 16 |
| `content` | 4 | 8 |
| `core` | 19 | 57 |
| `crm` | 28 | 129 |
| `cron` | 2 | 2 |
| `doare` | 2 | 4 |
| `eduzz` | 7 | 9 |
| `email` | 17 | 76 |
| `forms` | 7 | 20 |
| `imports` | 2 | 8 |
| `inbox` | 4 | 12 |
| `integrations` | 7 | 24 |
| `journeys` | 4 | 11 |
| `smart_forms` | 6 | 15 |
| `whatsapp` | 3 | 14 |

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

### 11.6 Resumo de Risco — Diagnóstico 2

| Achado | Severidade | Ação |
|--------|------------|------|
| `analytics` com 57 tabelas multi-tenant sem RLS (26.9% coverage) | **CRITICA** | Habilitar RLS + policies em todas tabelas com `tenant_id` |
| `contact_events` (43 partitions) sem RLS | **CRITICA** | Contém histórico comportamental de contatos de todos tenants |
| `contact_product_stats` (9 tabelas) sem RLS | **ALTA** | Dados de produto/compra por contato |
| `journeys` com 3 tabelas de jobs sem RLS | **ALTA** | Tabelas internas, mas expostas se schema tem grants |
| `crm.opportunity_activities_archive` sem RLS | **MEDIA** | Archive table, provavelmente read-only |
| `sympla` (4 tabelas) sem RLS | **MEDIA** | Schema interno, menor risco se não exposto |
| 498 policies existentes — boa cobertura nos schemas principais | **INFO** | Padrão de tenant isolation consistente |

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

### 12.3 SECURITY DEFINER sem search_path seguro

**SEVERIDADE: CRITICA — 26 funções vulneráveis a search_path injection**

Funções SECURITY DEFINER que **não** definem `search_path` explícito, permitindo potencial privilege escalation:

| Schema | Função | Config |
|--------|--------|--------|
| `analytics` | `cleanup_stale_segment_eval_jobs` | — |
| `analytics` | `count_segment_customers` | `statement_timeout=30s` |
| `analytics` | `enqueue_refresh_all_tenants` (×2) | — |
| `analytics` | `get_leads_count` | — |
| `analytics` | `get_segment_by_id` | — |
| `analytics` | `get_segment_preview` | `statement_timeout=120s` |
| `analytics` | `get_segments_with_live_count` | — |
| `analytics` | `get_unique_segmented_customers_count` | — |
| `analytics` | `purge_processed_eval_queue` | — |
| `analytics` | `refresh_segment_customers` | — |
| `analytics` | `refresh_segment_parties` | — |
| `content` | `get_landing_page_by_slug` | — |
| `content` | `get_next_landing_page_version` | — |
| `content` | `publish_landing_page` | — |
| `forms` | `get_public_form` | — |
| `forms` | `save_public_answer` | — |
| `forms` | `trg_enqueue_link_submission` | — |
| `integrations` | `bulk_enqueue_jobs` | — |
| `integrations` | `invoke_apidozap` | — |
| `integrations` | `invoke_doare_sync` | — |
| `integrations` | `invoke_eduzz_webhook` | — |
| `integrations` | `invoke_nylas_sync` | — |
| `integrations` | `invoke_worker` | — |
| `public` | `get_customer_tag_counts` | `statement_timeout=15s` |
| `whatsapp` | `upsert_chat` | — |

**Impacto:** Um atacante poderia criar objetos em `public` schema com nomes que collide com funções/tabelas referenciadas nestas funções, potencialmente escalando privilégios quando a função é executada com privilégios do owner (`postgres`).

**Funções de maior risco (acessíveis via anon/forms públicos):**
- `forms.get_public_form` — chamada por formulários públicos sem autenticação
- `forms.save_public_answer` — aceita input de usuários anônimos
- `content.get_landing_page_by_slug` — chamada por landing pages públicas
- `public.get_customer_tag_counts` — callable via `/rpc/`

### 12.4 Views sem security_invoker

**3 views executam como owner (postgres), não como caller:**

| Schema | View | Owner | Risco |
|--------|------|-------|-------|
| `analytics` | `v_segment_base` | `postgres` | **ALTO** — base para segmentação de clientes, bypassa RLS |
| `analytics` | `v_segment_leads_base` | `postgres` | **ALTO** — base para segmentação de leads, bypassa RLS |
| `crm` | `vw_parties_unified` | `postgres` | **CRITICO** — view principal do CRM, define `is_customer`/`is_lead` |

**Impacto:** Estas views executam queries com privilégios de `postgres`, que bypassa RLS. Se acessadas diretamente via PostgREST, podem expor dados cross-tenant. Atualmente mitigado parcialmente porque:
1. `v_segment_base` e `v_segment_leads_base` são usadas internamente por funções SECURITY DEFINER que filtram por `tenant_id`
2. `vw_parties_unified` tem RLS na tabela que a referencia

**Ação:** Adicionar `security_invoker = true` a todas as views, ou garantir que não são acessíveis diretamente via API.

### 12.5 Resumo de Risco — Diagnóstico 3

| Achado | Severidade | Ação |
|--------|------------|------|
| 319 funções SECURITY DEFINER (66% do total) | **CRITICA** | Migrar para INVOKER onde possível (Sprint 3) |
| 26 funções DEFINER sem `search_path` seguro | **CRITICA** | Adicionar `SET search_path = ''` imediatamente |
| `forms.get_public_form` e `save_public_answer` sem search_path (acessíveis por anon) | **CRITICA** | Fix prioritário — superfície de ataque direta |
| `public` schema com 113 funções DEFINER callable via `/rpc/` | **ALTA** | Auditar quais devem ser chamáveis externamente |
| `core`, `email`, `eduzz` são 100% DEFINER | **ALTA** | Avaliar necessidade real por função |
| 3 views sem `security_invoker` (dados cross-tenant) | **ALTA** | Adicionar `security_invoker = true` |

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

### 13.5 Credenciais de Terceiros

**SEVERIDADE: CRITICA — 20 colunas com credenciais em texto**

| Schema | Tabela | Coluna | Tipo |
|--------|--------|--------|------|
| `commerce` | `asaas_config` | `api_key_encrypted` | API Key (criptografada) |
| `core` | `eduzz_integrations` | `api_key` | API Key **texto plano** |
| `core` | `eduzz_integrations` | `webhook_secret` | Webhook Secret **texto plano** |
| `eduzz` | `integrations` | `api_key` | API Key **texto plano** |
| `eduzz` | `integrations` | `webhook_secret` | Webhook Secret **texto plano** |
| `eduzz` | `integrations` | `client_secret` | Client Secret **texto plano** |
| `eduzz` | `integrations` | `access_token` | OAuth Token **texto plano** |
| `eduzz` | `integrations` | `refresh_token` | OAuth Refresh **texto plano** |
| `email` | `tenant_config` | `sendgrid_subuser_api_key` | API Key **texto plano** |
| `email` | `tenant_config` | `webhook_secret` | Webhook Secret **texto plano** |
| `integrations` | `accounts` | `access_token` | OAuth Token **texto plano** |
| `integrations` | `accounts` | `refresh_token` | OAuth Refresh **texto plano** |
| `integrations` | `accounts` | `client_secret` | Client Secret **texto plano** |
| `whatsapp` | `instances` | `webhook_secret` | Webhook Secret **texto plano** |

**Impacto:** Se um atacante conseguir ler estas tabelas (via falha de RLS ou SQL injection), obtém acesso direto a:
- APIs da Eduzz de todos os tenants
- APIs do SendGrid de todos os tenants
- OAuth tokens de integrações (Nylas, Doaré, etc.)
- Webhook secrets (permitindo forjar eventos)

**Única exceção positiva:** `commerce.asaas_config.api_key_encrypted` — já usa criptografia.

### 13.6 Resumo de Risco — Diagnóstico 4

| Achado | Severidade | Ação |
|--------|------------|------|
| 13+ credenciais em texto plano nas tabelas | **CRITICA** | Migrar para Vault com urgência (Sprint 4) |
| `eduzz.integrations` com 7 colunas de credencial | **CRITICA** | Maior concentração de secrets expostos |
| CPF/CNPJ em 8+ tabelas sem field-level encryption | **ALTA** | Avaliar encryption at field level |
| ~160 colunas JSONB com PII potencialmente não mapeado | **ALTA** | Auditar `raw_payload`, `raw_source`, `config` |
| `integrations.accounts.config` pode ter secrets em JSONB | **CRITICA** | Auditar conteúdo e migrar |
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
| Apenas 4 secrets no Vault vs 13+ em texto plano | **CRITICA** | Migrar credenciais para Vault |
| `service_role_key` está no Vault | **INFO** | Positivo — usado por Edge Functions via pg_net |

---

## 15. Auditoria e Logging

> **Diagnóstico executado:** 2026-03-09 — diagnostic-6.sql

### 15.1 Schema audit existe?

**Resultado: ❌ NÃO EXISTE**

O schema `audit` não foi criado. Não há infraestrutura dedicada de auditoria no banco.

### 15.2 Tabelas de Auditoria Existentes

Tabelas que funcionam como log/histórico (por convenção de nome):

| Schema | Tabela | Colunas | Propósito |
|--------|--------|---------|-----------|
| `auth` | `audit_log_entries` | 5 | Auth audit (Supabase) — **VAZIA (0 registros)** |
| `core` | `permission_audit_log` | 7 | Log de mudanças de permissão |
| `commerce` | `merge_history` | 14 | Histórico de merges de produtos |
| `crm` | `lead_import_history` | 6 | Histórico de imports de leads |
| `crm` | `opportunity_stage_history` | 7 | Histórico de stages de oportunidades |
| `analytics` | `fallback_log` | 5 | Log de fallback de analytics |
| `integrations` | `logs` | 10 | Logs de integrações |

**Observação:** `auth.audit_log_entries` está **vazia** — Supabase Auth não está gerando registros de auditoria.

### 15.3 Triggers de Auditoria

**Resultado: ❌ NENHUM trigger de auditoria encontrado**

Nenhum trigger com nome contendo `audit`, `log`, `history` ou `track` foi identificado. As tabelas de histórico existentes (15.2) são populadas por lógica de aplicação, não por triggers automáticos.

### 15.4 Tabelas Sensíveis SEM Audit Trigger

**24 tabelas com dados sensíveis (PII, financeiro, credenciais) sem qualquer trigger de auditoria:**

| Schema | Tabelas |
|--------|---------|
| `analytics` | `contact_state`, `segment_parties` |
| `commerce` | `archived_transactions`, `asaas_config`, `checkout_orders`, `transaction_import_errors`, `transactions` |
| `core` | `eduzz_integrations`, `system_admins`, `team_invitations` |
| `crm` | `customers`, `opportunities`, `parties`, `party_person` |
| `email` | `campaign_recipients`, `campaign_sends`, `email_events`, `sender_identities`, `suppression_list`, `tenant_config`, `tenant_send_stats`, `unsubscribe_feedback`, `unsubscribes` |
| `integrations` | `accounts` |

**Impacto:** Alterações em dados de clientes, transações financeiras, credenciais de integrações e configurações de email não são rastreadas. Impossível saber quem alterou o quê e quando.

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

| Achado | Severidade | Ação |
|--------|------------|------|
| Schema `audit` não existe | **CRITICA** | Criar schema + tabelas (Sprint 5) |
| Zero triggers de auditoria em tabelas sensíveis | **CRITICA** | Implementar triggers em 24 tabelas prioritárias |
| `pgaudit.log = none` — efetivamente desabilitado | **ALTA** | Configurar `pgaudit.log = 'write, ddl'` |
| `auth.audit_log_entries` vazia | **ALTA** | Investigar configuração do Supabase Auth |
| `core.permission_audit_log` é o único log de audit real | **MEDIA** | Expandir padrão para outros schemas |
| 13 tabelas de eventos existem mas sem padrão formal | **MEDIA** | Formalizar como parte do audit trail |
| `pgaudit.log_parameter = off` | **MEDIA** | Habilitar para capturar valores |

---

# PARTE D — CONTROLES E COMPLIANCE

## 16. Control Matrix

| ID | Controle | Framework | Implementação | Owner | Status | Evidência |
|----|----------|-----------|---------------|-------|--------|-----------|
| C-001 | RLS em tabelas expostas | SOC 2, ISO | Policies SQL | Eng | ⚠️ | Query 2.4 |
| C-002 | MFA para admins | SOC 2, ISO | Supabase Auth | Eng | ❌ | - |
| C-003 | Audit trail | SOC 2, HIPAA, ISO | Schema audit | Eng | ❌ | - |
| C-004 | Encryption at rest | SOC 2, HIPAA | Supabase | Infra | ✅ | Supabase docs |
| C-005 | Encryption in transit | SOC 2, HIPAA | TLS | Infra | ✅ | Supabase docs |
| C-006 | Access reviews | SOC 2, ISO | Manual | Ops | ❌ | - |
| C-007 | Incident response | SOC 2, ISO | Runbook | Ops | ❌ | - |
| C-008 | Backup/restore | SOC 2, ISO | PITR | Infra | ⚠️ | Verificar |
| C-009 | Vulnerability scanning | SOC 2, ISO | CI/CD | Eng | ❌ | - |
| C-010 | Least privilege | SOC 2, ISO | RLS, grants | Eng | ⚠️ | Sprint 3 |

## 17. Gaps Conhecidos (Ordenados por Severidade)

### 🔴 Crítico

| Gap | Descrição | Sprint |
|-----|-----------|--------|
| G-001 | Tabelas multi-tenant sem RLS | Sprint 2 |
| G-002 | SECURITY DEFINER sem search_path | Sprint 3 |
| G-003 | Credenciais de terceiros em texto | Sprint 4 |

### 🟠 Alto

| Gap | Descrição | Sprint |
|-----|-----------|--------|
| G-004 | MFA não enforced | Sprint 8 |
| G-005 | Audit trail não existe | Sprint 5 |
| G-006 | Sem testes de cross-tenant | Sprint 2 |

### 🟡 Médio

| Gap | Descrição | Sprint |
|-----|-----------|--------|
| G-007 | Views sem security_invoker | Sprint 3 |
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

### Sprint 2 — RLS Sistemático
- [ ] Habilitar RLS em todas tabelas expostas
- [ ] Aplicar policy padrão tenant-scoped usando `app_auth.active_tenant_id()`
- [ ] Adicionar índices para performance
- [ ] Criar suite de testes cross-tenant

### Sprint 3 — Workers e Credenciais
- [ ] Inventariar credenciais por worker
- [ ] Avaliar necessidade real de service_role
- [ ] Implementar logging de operações privilegiadas
- [ ] Mover secrets críticos para Vault

### Sprint 4 — Auditoria Básica
- [ ] Implementar triggers em tabelas críticas (schema `audit` já criado)
- [ ] Configurar retention
- [ ] Dashboard básico de auditoria

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
