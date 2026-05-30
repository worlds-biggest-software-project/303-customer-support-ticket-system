# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Customer Support Ticket System · Created: 2026-05-24

## Philosophy

This model follows the classic normalized relational approach where every domain concept — tickets, messages, contacts, agents, SLA policies, knowledge articles, tags, custom fields — gets its own table with explicit foreign keys enforcing referential integrity. The design aligns with ITIL 4 incident management data requirements and mirrors the entity structure exposed by Zendesk's Ticketing API and Freshdesk's REST API.

The normalized approach makes complex cross-entity queries straightforward: "show me all P1 tickets for enterprise customers that breached SLA this month, grouped by assigned team" is a single SQL query with JOINs. Every relationship is explicit, every constraint is enforced at the database level, and reporting/analytics queries operate on clean, indexed columns rather than parsing JSONB.

**Best for:** Teams that need strong data integrity, complex SLA reporting, regulatory compliance (GDPR/HIPAA), and a well-understood schema that maps cleanly to ITIL processes.

**Trade-offs:**
- **Pro:** Full referential integrity, straightforward JOINs, clean reporting queries
- **Pro:** ITIL-aligned ticket lifecycle with enforced state transitions
- **Pro:** Custom fields via EAV pattern — no schema migration for new fields
- **Con:** Higher table count (35+) increases schema complexity
- **Con:** EAV custom fields require pivot queries for filtering/sorting
- **Con:** Adding new channel types requires new tables or columns
- **Con:** Schema migrations needed for structural changes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ITIL 4 | Ticket status lifecycle (new→open→pending→on_hold→solved→closed), priority matrix (urgency × impact), escalation paths |
| ISO/IEC 20000-1:2018 | Incident management fields, service request fulfillment tracking, SLA compliance reporting |
| GDPR | `data_retention_policies` table, `erasure_requests` table, PII flags on ticket fields |
| HIPAA | `baa_agreements` table, PHI detection flags, encryption_status tracking |
| SOC 2 | `audit_logs` table captures all changes for security/availability audit |
| OAuth 2.0 / RFC 6749 | `oauth_applications` and `api_keys` tables for integration authentication |
| RFC 9110 | `idempotency_keys` table prevents duplicate ticket creation |
| OpenAPI 3.1 | API resource structure mirrors table entities 1:1 |

---

## Multi-Tenancy & Authentication

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free', 'starter', 'professional', 'enterprise')),
    settings JSONB NOT NULL DEFAULT '{}',
    data_residency TEXT NOT NULL DEFAULT 'us' CHECK (data_residency IN ('us', 'eu', 'ap')),
    hipaa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'agent' CHECK (role IN ('owner', 'admin', 'agent', 'light_agent')),
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended')),
    avatar_url TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    signature TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE team_memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    agent_id UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('lead', 'member')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, agent_id)
);

CREATE INDEX idx_agents_tenant ON agents(tenant_id);
CREATE INDEX idx_teams_tenant ON teams(tenant_id);
CREATE INDEX idx_team_memberships_agent ON team_memberships(agent_id);
```

---

## Contacts & Organizations

```sql
CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT,
    phone TEXT,
    name TEXT,
    external_id TEXT,
    organization_id UUID REFERENCES customer_organizations(id),
    locale TEXT DEFAULT 'en',
    timezone TEXT,
    tags TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customer_organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    domain TEXT,
    external_id TEXT,
    plan_tier TEXT,
    health_score SMALLINT CHECK (health_score BETWEEN 0 AND 100),
    contract_value_cents BIGINT,
    tags TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, domain)
);

CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);
CREATE INDEX idx_contacts_org ON contacts(organization_id);
CREATE INDEX idx_customer_orgs_tenant ON customer_organizations(tenant_id);
```

---

## Channels & Inboxes

```sql
CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    channel_type TEXT NOT NULL CHECK (channel_type IN ('email', 'chat', 'web_form', 'social_twitter', 'social_facebook', 'social_instagram', 'phone', 'api', 'slack')),
    configuration JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"imap_host": "...", "smtp_host": "...", "from_address": "support@example.com"}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channels_tenant ON channels(tenant_id);
```

---

## Tickets

```sql
CREATE TABLE tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    ticket_number BIGINT NOT NULL,
    subject TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'new' CHECK (status IN ('new', 'open', 'pending', 'on_hold', 'solved', 'closed')),
    priority TEXT NOT NULL DEFAULT 'normal' CHECK (priority IN ('urgent', 'high', 'normal', 'low')),
    urgency TEXT CHECK (urgency IN ('high', 'medium', 'low')),
    impact TEXT CHECK (impact IN ('high', 'medium', 'low')),
    ticket_type TEXT NOT NULL DEFAULT 'incident' CHECK (ticket_type IN ('incident', 'question', 'problem', 'task', 'feature_request')),
    channel_id UUID REFERENCES channels(id),
    requester_id UUID NOT NULL REFERENCES contacts(id),
    assignee_id UUID REFERENCES agents(id),
    team_id UUID REFERENCES teams(id),
    organization_id UUID REFERENCES customer_organizations(id),
    parent_ticket_id UUID REFERENCES tickets(id),
    due_at TIMESTAMPTZ,
    first_responded_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    satisfaction_rating SMALLINT CHECK (satisfaction_rating BETWEEN 1 AND 5),
    satisfaction_comment TEXT,
    language TEXT DEFAULT 'en',
    is_spam BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ticket_number)
);

CREATE INDEX idx_tickets_tenant_status ON tickets(tenant_id, status);
CREATE INDEX idx_tickets_assignee ON tickets(assignee_id) WHERE assignee_id IS NOT NULL;
CREATE INDEX idx_tickets_team ON tickets(team_id) WHERE team_id IS NOT NULL;
CREATE INDEX idx_tickets_requester ON tickets(requester_id);
CREATE INDEX idx_tickets_org ON tickets(organization_id);
CREATE INDEX idx_tickets_priority ON tickets(tenant_id, priority, status);
CREATE INDEX idx_tickets_created ON tickets(tenant_id, created_at DESC);
CREATE INDEX idx_tickets_parent ON tickets(parent_ticket_id) WHERE parent_ticket_id IS NOT NULL;

CREATE TABLE ticket_tags (
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    tag TEXT NOT NULL,
    source TEXT NOT NULL DEFAULT 'manual' CHECK (source IN ('manual', 'ai', 'automation')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (ticket_id, tag)
);

CREATE INDEX idx_ticket_tags_tag ON ticket_tags(tag);

CREATE TABLE ticket_followers (
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    agent_id UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (ticket_id, agent_id)
);
```

---

## Messages & Conversations

```sql
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    author_type TEXT NOT NULL CHECK (author_type IN ('contact', 'agent', 'system', 'ai_bot')),
    author_id UUID,
    channel_id UUID REFERENCES channels(id),
    message_type TEXT NOT NULL DEFAULT 'reply' CHECK (message_type IN ('reply', 'note', 'system', 'ai_suggestion')),
    body_text TEXT,
    body_html TEXT,
    is_private BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE message_attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    file_name TEXT NOT NULL,
    content_type TEXT NOT NULL,
    size_bytes BIGINT NOT NULL,
    storage_key TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_messages_ticket ON messages(ticket_id, created_at);
CREATE INDEX idx_messages_author ON messages(author_type, author_id);
CREATE INDEX idx_message_attachments_message ON message_attachments(message_id);
```

---

## SLA Management

```sql
CREATE TABLE sla_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    description TEXT,
    position INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    conditions JSONB NOT NULL DEFAULT '{}',
    -- {"all": [{"field": "priority", "operator": "is", "value": "urgent"}]}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sla_targets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sla_policy_id UUID NOT NULL REFERENCES sla_policies(id) ON DELETE CASCADE,
    priority TEXT NOT NULL CHECK (priority IN ('urgent', 'high', 'normal', 'low')),
    metric TEXT NOT NULL CHECK (metric IN ('first_response', 'next_response', 'resolution')),
    target_seconds INT NOT NULL,
    business_hours_only BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (sla_policy_id, priority, metric)
);

CREATE TABLE business_hours (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL DEFAULT 'Default',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    schedule JSONB NOT NULL DEFAULT '{}',
    -- {"mon": {"start": "09:00", "end": "17:00"}, "tue": {...}, ...}
    holidays JSONB NOT NULL DEFAULT '[]',
    -- [{"date": "2026-12-25", "name": "Christmas"}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ticket_sla_instances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    sla_policy_id UUID NOT NULL REFERENCES sla_policies(id),
    metric TEXT NOT NULL CHECK (metric IN ('first_response', 'next_response', 'resolution')),
    target_at TIMESTAMPTZ NOT NULL,
    achieved_at TIMESTAMPTZ,
    breached BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (ticket_id, metric)
);

CREATE INDEX idx_sla_instances_ticket ON ticket_sla_instances(ticket_id);
CREATE INDEX idx_sla_instances_breach ON ticket_sla_instances(breached, target_at) WHERE NOT breached;
```

---

## Custom Fields (EAV Pattern)

```sql
CREATE TABLE custom_field_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    entity_type TEXT NOT NULL DEFAULT 'ticket' CHECK (entity_type IN ('ticket', 'contact', 'organization')),
    field_key TEXT NOT NULL,
    label TEXT NOT NULL,
    field_type TEXT NOT NULL CHECK (field_type IN ('text', 'textarea', 'number', 'decimal', 'boolean', 'date', 'datetime', 'dropdown', 'multi_select', 'regex')),
    options JSONB,
    -- for dropdown/multi_select: ["Option A", "Option B", "Option C"]
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    validation_regex TEXT,
    position INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entity_type, field_key)
);

CREATE TABLE custom_field_values (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_definition_id UUID NOT NULL REFERENCES custom_field_definitions(id) ON DELETE CASCADE,
    entity_id UUID NOT NULL,
    value_text TEXT,
    value_number NUMERIC,
    value_boolean BOOLEAN,
    value_date TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (field_definition_id, entity_id)
);

CREATE INDEX idx_custom_field_values_entity ON custom_field_values(entity_id);
```

---

## AI Triage & Classification

```sql
CREATE TABLE ai_classifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    model_version TEXT NOT NULL,
    intent TEXT,
    intent_confidence REAL CHECK (intent_confidence BETWEEN 0 AND 1),
    sentiment TEXT CHECK (sentiment IN ('positive', 'neutral', 'negative', 'frustrated')),
    sentiment_confidence REAL CHECK (sentiment_confidence BETWEEN 0 AND 1),
    detected_language TEXT,
    language_confidence REAL CHECK (language_confidence BETWEEN 0 AND 1),
    suggested_priority TEXT CHECK (suggested_priority IN ('urgent', 'high', 'normal', 'low')),
    suggested_team_id UUID REFERENCES teams(id),
    suggested_assignee_id UUID REFERENCES agents(id),
    human_corrected BOOLEAN NOT NULL DEFAULT FALSE,
    corrected_intent TEXT,
    corrected_at TIMESTAMPTZ,
    corrected_by UUID REFERENCES agents(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_suggested_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    message_id UUID REFERENCES messages(id),
    suggestion_text TEXT NOT NULL,
    knowledge_article_ids UUID[],
    confidence REAL CHECK (confidence BETWEEN 0 AND 1),
    was_used BOOLEAN NOT NULL DEFAULT FALSE,
    was_edited BOOLEAN NOT NULL DEFAULT FALSE,
    feedback TEXT CHECK (feedback IN ('helpful', 'not_helpful', 'partially_helpful')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_classifications_ticket ON ai_classifications(ticket_id);
CREATE INDEX idx_ai_suggested_responses_ticket ON ai_suggested_responses(ticket_id);
```

---

## Knowledge Base

```sql
CREATE TABLE kb_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    parent_id UUID REFERENCES kb_categories(id),
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    position INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE TABLE kb_articles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    category_id UUID REFERENCES kb_categories(id),
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    body_markdown TEXT NOT NULL,
    body_html TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'published', 'archived')),
    locale TEXT NOT NULL DEFAULT 'en',
    author_id UUID REFERENCES agents(id),
    view_count BIGINT NOT NULL DEFAULT 0,
    helpful_count INT NOT NULL DEFAULT 0,
    not_helpful_count INT NOT NULL DEFAULT 0,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug, locale)
);

CREATE TABLE ticket_article_links (
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    article_id UUID NOT NULL REFERENCES kb_articles(id) ON DELETE CASCADE,
    link_type TEXT NOT NULL CHECK (link_type IN ('suggested', 'used_in_resolution', 'auto_generated_from')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (ticket_id, article_id, link_type)
);

CREATE INDEX idx_kb_articles_tenant ON kb_articles(tenant_id, status);
CREATE INDEX idx_kb_articles_category ON kb_articles(category_id);
```

---

## Automation Rules

```sql
CREATE TABLE automation_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    description TEXT,
    trigger_event TEXT NOT NULL CHECK (trigger_event IN ('ticket_created', 'ticket_updated', 'time_based', 'sla_breach')),
    conditions JSONB NOT NULL DEFAULT '{}',
    -- {"all": [{"field": "priority", "operator": "is", "value": "urgent"}, {"field": "status", "operator": "is", "value": "new"}]}
    actions JSONB NOT NULL DEFAULT '[]',
    -- [{"type": "assign_team", "team_id": "..."}, {"type": "set_priority", "value": "high"}]
    position INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_automation_rules_tenant ON automation_rules(tenant_id, is_active);
```

---

## Audit & Compliance

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    action TEXT NOT NULL CHECK (action IN ('created', 'updated', 'deleted', 'status_changed', 'assigned', 'escalated', 'merged', 'sla_breached')),
    actor_type TEXT NOT NULL CHECK (actor_type IN ('agent', 'contact', 'system', 'ai', 'api')),
    actor_id UUID,
    changes JSONB NOT NULL DEFAULT '{}',
    -- {"status": {"from": "new", "to": "open"}, "assignee_id": {"from": null, "to": "uuid..."}}
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_tenant ON audit_logs(tenant_id, created_at DESC);

CREATE TABLE erasure_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    contact_id UUID NOT NULL REFERENCES contacts(id),
    requested_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'rejected')),
    reason TEXT,
    processed_by UUID REFERENCES agents(id),
    entities_erased JSONB
    -- {"tickets": 12, "messages": 45, "attachments": 3}
);

CREATE TABLE data_retention_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    entity_type TEXT NOT NULL,
    retention_days INT NOT NULL,
    action TEXT NOT NULL DEFAULT 'anonymize' CHECK (action IN ('delete', 'anonymize', 'archive')),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entity_type)
);
```

---

## API & Integrations

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_by UUID NOT NULL REFERENCES agents(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    url TEXT NOT NULL,
    events TEXT[] NOT NULL,
    secret_hash TEXT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_triggered_at TIMESTAMPTZ,
    failure_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    response_status INT NOT NULL,
    response_body JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '24 hours'
);

CREATE INDEX idx_idempotency_keys_expires ON idempotency_keys(expires_at);
```

---

## Row-Level Security

```sql
ALTER TABLE tickets ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE agents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_tickets ON tickets
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_messages ON messages
    USING (ticket_id IN (SELECT id FROM tickets WHERE tenant_id = current_setting('app.current_tenant_id')::UUID));

CREATE POLICY tenant_isolation_contacts ON contacts
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_agents ON agents
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Auth | 5 | tenants, agents, teams, team_memberships, api_keys |
| Contacts & Orgs | 2 | contacts, customer_organizations |
| Channels | 1 | channels |
| Tickets | 3 | tickets, ticket_tags, ticket_followers |
| Messages | 2 | messages, message_attachments |
| SLA | 4 | sla_policies, sla_targets, business_hours, ticket_sla_instances |
| Custom Fields | 2 | custom_field_definitions, custom_field_values |
| AI | 2 | ai_classifications, ai_suggested_responses |
| Knowledge Base | 3 | kb_categories, kb_articles, ticket_article_links |
| Automation | 1 | automation_rules |
| Audit & Compliance | 3 | audit_logs, erasure_requests, data_retention_policies |
| Integrations | 2 | webhooks, idempotency_keys |
| **Total** | **30** | |

---

## Key Design Decisions

1. **ITIL-aligned ticket lifecycle** — Status values (new, open, pending, on_hold, solved, closed) and priority levels (urgent, high, normal, low) follow ITIL 4 conventions. The urgency × impact matrix is supported via separate columns for ITIL incident prioritisation.

2. **EAV for custom fields** — Rather than JSONB, custom fields use the Entity-Attribute-Value pattern with typed value columns (`value_text`, `value_number`, `value_boolean`, `value_date`). This enables SQL-level filtering, sorting, and indexing on custom field values at the cost of pivot query complexity.

3. **Separate SLA policy → targets → instances** — SLA policies define conditions for matching tickets. Targets define per-priority time limits. Instances track actual SLA performance per ticket per metric. This three-tier model matches how Zendesk structures SLA management.

4. **AI classifications as separate table** — Predictions are stored alongside the ticket but in a dedicated table, enabling model version tracking, human correction feedback loops, and confidence-based filtering without polluting the core ticket schema.

5. **Partitioned audit logs** — `audit_logs` is range-partitioned by `created_at` for write performance and retention management. The `changes` JSONB column stores field-level diffs, mirroring Zendesk's ticket audits API structure.

6. **GDPR-ready erasure** — Dedicated `erasure_requests` and `data_retention_policies` tables enable right-to-erasure tracking and automated retention enforcement.

7. **Idempotency keys** — Following RFC 9110, the `idempotency_keys` table prevents duplicate ticket creation on API retries with a 24-hour TTL.

8. **Multi-tenant RLS** — Row-Level Security policies on all major tables enforce tenant isolation via `current_setting('app.current_tenant_id')`, eliminating the risk of cross-tenant data leaks.
