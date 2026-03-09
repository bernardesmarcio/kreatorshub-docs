# KreatorsHub — Security Architecture & Compliance Roadmap

## Estado: Sprint 0 — Diagnóstico
## Última atualização: 2026-03-09
## Versão: 0.1

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

### 10.1 Schemas em exposed_schemas

```
[PENDENTE: Executar diagnostic-1.sql query 1.1]
```

### 10.2 Tabelas por Schema

```
[PENDENTE: Executar diagnostic-1.sql query 1.3]
```

### 10.3 Grants para `anon`

```
[PENDENTE: Executar diagnostic-1.sql query 1.4]
```

### 10.4 Grants para `authenticated`

```
[PENDENTE: Executar diagnostic-1.sql query 1.5]
```

---

## 11. RLS Coverage

### 11.1 Resumo por Schema

```
[PENDENTE: Executar diagnostic-2.sql query 2.4]
```

### 11.2 Tabelas COM RLS

```
[PENDENTE: Executar diagnostic-2.sql query 2.1]
```

### 11.3 Tabelas SEM RLS (Gaps)

```
[PENDENTE: Executar diagnostic-2.sql query 2.2]
```

### 11.4 Tabelas Multi-tenant SEM RLS (CRÍTICO)

```
[PENDENTE: Executar diagnostic-2.sql query 2.5]
```

### 11.5 Policies Existentes

```
[PENDENTE: Executar diagnostic-2.sql query 2.3]
```

---

## 12. Funções e Privilégios

### 12.1 SECURITY DEFINER — Inventário Completo

```
[PENDENTE: Executar diagnostic-3.sql query 3.1]
```

### 12.2 Contagem por Schema

```
[PENDENTE: Executar diagnostic-3.sql query 3.2]
```

### 12.3 SECURITY DEFINER sem search_path seguro

```
[PENDENTE: Executar diagnostic-3.sql query 3.3]
```

### 12.4 Views sem security_invoker

```
[PENDENTE: Executar diagnostic-3.sql query 3.4]
```

---

## 13. Dados Sensíveis (PII)

### 13.1 Colunas com PII

```
[PENDENTE: Executar diagnostic-4.sql query 4.1]
```

### 13.2 Resumo por Schema

```
[PENDENTE: Executar diagnostic-4.sql query 4.2]
```

### 13.3 Dados Financeiros

```
[PENDENTE: Executar diagnostic-4.sql query 4.3]
```

### 13.4 Colunas JSONB (risco de PII não mapeado)

```
[PENDENTE: Executar diagnostic-4.sql query 4.4]
```

### 13.5 Credenciais de Terceiros

```
[PENDENTE: Executar diagnostic-4.sql query 4.5]
```

---

## 14. Workers e Credenciais

### 14.1 Roles e Privilégios no Banco

```
[PENDENTE: Executar diagnostic-5.sql query 5.1]
```

### 14.2 Roles com BYPASSRLS

```
[PENDENTE: Executar diagnostic-5.sql query 5.2]
```

### 14.3 Env Vars por Worker (Railway)

```
[PENDENTE: Levantar manualmente do Railway dashboard]
```

### 14.4 Secrets em Edge Functions

```
[PENDENTE: Levantar manualmente do Supabase dashboard]
```

---

## 15. Auditoria e Logging

### 15.1 Schema audit existe?

```
[PENDENTE: Executar diagnostic-6.sql query 6.1]
```

### 15.2 Tabelas de Auditoria Existentes

```
[PENDENTE: Executar diagnostic-6.sql query 6.2]
```

### 15.3 Triggers de Auditoria

```
[PENDENTE: Executar diagnostic-6.sql query 6.3]
```

### 15.4 pgAudit Status

```
[PENDENTE: Executar diagnostic-6.sql query 6.5]
```

### 15.5 Extensões de Segurança

```
[PENDENTE: Executar diagnostic-6.sql query 6.6]
```

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

### Sprint 0 — Diagnóstico ✅ Em andamento
- [ ] Executar diagnósticos 1-6
- [ ] Popular inventário (Parte C)
- [ ] Priorizar gaps
- [ ] Levantar env vars Railway
- [ ] Verificar configuração Supabase Auth

### Sprint 1 — Helpers Centralizados
- [ ] Criar schema `app_auth`
- [ ] Implementar `active_tenant_id()`, `current_role()`, etc.
- [ ] Documentar contrato de JWT
- [ ] Configurar Custom Access Token Hook (se necessário)

### Sprint 2 — RLS Sistemático
- [ ] Habilitar RLS em todas tabelas expostas
- [ ] Aplicar policy padrão tenant-scoped
- [ ] Adicionar índices para performance
- [ ] Criar suite de testes cross-tenant

### Sprint 3 — SECURITY DEFINER Audit
- [ ] Revisar todas as 57+ funções DEFINER
- [ ] Migrar para INVOKER onde possível
- [ ] Adicionar search_path seguro nas restantes
- [ ] Criar guardrail de PR para novas funções

### Sprint 4 — Workers e Credenciais
- [ ] Inventariar credenciais por worker
- [ ] Avaliar necessidade real de service_role
- [ ] Implementar logging de operações privilegiadas
- [ ] Mover secrets críticos para Vault

### Sprint 5 — Auditoria Básica
- [ ] Criar schema `audit`
- [ ] Implementar triggers em tabelas críticas
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
