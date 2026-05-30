# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Customer Support Ticket System · Created: 2026-05-24

## Philosophy

This model keeps core ticket fields (status, priority, assignee) as typed relational columns for indexing and querying, while storing variable or domain-specific data — custom fields, channel metadata, AI classifications, automation rule definitions — in JSONB columns. The result is roughly half the table count of the fully normalized model while retaining the ability to evolve the schema without migrations.

The hybrid approach mirrors how Freshdesk and Help Scout structure their APIs: a well-defined set of top-level ticket fields with a `custom_fields` JSONB bag for tenant-specific extensions. It also aligns with how modern SaaS products ship quickly — core functionality works out of the box, and the JSONB columns absorb the long tail of customer-specific requirements without schema changes.

GIN indexes on JSONB columns enable containment queries (`@>`) for filtering tickets by custom field values, channel metadata, or AI classification results. PostgreSQL's JSONB path operators make these queries efficient enough for dashboards and reports.

**Best for:** Teams prioritizing rapid iteration, MVP-first deployment, and the flexibility to support diverse customer configurations without per-tenant schema migrations.

**Trade-offs:**
- **Pro:** ~15 tables vs. 30 in the normalized model — simpler schema
- **Pro:** Custom fields are a JSONB column, not an EAV pattern — no pivot queries
- **Pro:** Channel-specific metadata (email headers, chat session data) stored inline
- **Pro:** Schema evolution via JSONB — no ALTER TABLE for new field types
- **Con:** JSONB queries are slower than typed column queries for complex aggregations
- **Con:** No foreign key constraints within JSONB — referential integrity is application-level
- **Con:** Harder to enforce NOT NULL or CHECK constraints on JSONB subfields
- **Con:** JSONB columns can grow unbounded without application-level validation

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ITIL 4 | Ticket status and priority as typed columns with CHECK constraints matching ITIL lifecycle |
| ISO/IEC 20000-1:2018 | SLA targets stored as JSONB per policy — flexible enough for varied SLA structures |
| GDPR | `contacts.consent` JSONB tracks consent records; `erasure_log` table for right-to-erasure |
| HIPAA | `tenant.settings` JSONB includes `hipaa_enabled` flag; PHI detection stored in `tickets.ai_classification` |
| RFC 9110 | Idempotency keys table for duplicate-safe API requests |
| JSON Schema 2020-12 | Custom field definitions include JSON Schema for validation |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free', 'starter', 'professional', 'enterprise')),
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"hipaa_enabled": false, "data_residency": "us", "business_hours": {"mon": ...}, "branding": {...}}
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
    team_ids UUID[] NOT NULL DEFAULT '{}',
    skills TEXT[] NOT NULL DEFAULT '{}',
    -- ["billing", "technical", "enterprise", "spanish"]
    preferences JSONB NOT NULL DEFAULT '{}',
    -- {"timezone": "America/New_York", "signature": "...", "notification_channels": ["email", "slack"]}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    description TEXT,
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"auto_assign": true, "round_robin": true, "max_tickets_per_agent": 25}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_agents_tenant ON agents(tenant_id);
CREATE INDEX idx_agents_skills ON agents USING GIN(skills);
CREATE INDEX idx_agents_team_ids ON agents USING GIN(team_ids);
```

---

## Contacts

```sql
CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT,
    phone TEXT,
    name TEXT,
    external_id TEXT,
    organization_name TEXT,
    organization_domain TEXT,
    traits JSONB NOT NULL DEFAULT '{}',
    -- {"plan": "enterprise", "company_size": "500+", "industry": "healthcare", "contract_value_cents": 120000, "health_score": 85}
    consent JSONB NOT NULL DEFAULT '{}',
    -- {"marketing": {"granted": true, "at": "2026-01-15T..."}, "tracking": {"granted": true, "at": "..."}}
    channel_identities JSONB NOT NULL DEFAULT '{}',
    -- {"twitter": "@handle", "slack": "U12345", "facebook": "page_scoped_id"}
    tags TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);
CREATE INDEX idx_contacts_traits ON contacts USING GIN(traits);
CREATE INDEX idx_contacts_tags ON contacts USING GIN(tags);
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
    ticket_type TEXT NOT NULL DEFAULT 'incident' CHECK (ticket_type IN ('incident', 'question', 'problem', 'task', 'feature_request')),
    channel TEXT NOT NULL DEFAULT 'web_form' CHECK (channel IN ('email', 'chat', 'web_form', 'social', 'phone', 'api', 'slack')),
    requester_id UUID NOT NULL REFERENCES contacts(id),
    assignee_id UUID REFERENCES agents(id),
    team_id UUID REFERENCES teams(id),
    parent_ticket_id UUID REFERENCES tickets(id),
    tags TEXT[] NOT NULL DEFAULT '{}',
    custom_fields JSONB NOT NULL DEFAULT '{}',
    -- {"product_area": "billing", "browser": "Chrome 125", "os": "macOS", "error_code": "ERR_402"}
    channel_metadata JSONB NOT NULL DEFAULT '{}',
    -- email: {"from": "user@example.com", "to": "support@...", "message_id": "<msg-id>", "in_reply_to": "...", "cc": [...]}
    -- chat: {"session_id": "...", "page_url": "...", "referrer": "...", "user_agent": "..."}
    -- social: {"platform": "twitter", "post_id": "...", "dm": true}
    ai_classification JSONB NOT NULL DEFAULT '{}',
    -- {"intent": "billing_question", "intent_confidence": 0.92, "sentiment": "frustrated", "sentiment_confidence": 0.87, "language": "en", "suggested_priority": "high", "model_version": "v3.2"}
    sla_status JSONB NOT NULL DEFAULT '{}',
    -- {"first_response": {"target_at": "...", "achieved_at": "...", "breached": false}, "resolution": {"target_at": "...", "breached": false}}
    satisfaction_rating SMALLINT CHECK (satisfaction_rating BETWEEN 1 AND 5),
    satisfaction_comment TEXT,
    first_responded_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ticket_number)
);

CREATE INDEX idx_tickets_tenant_status ON tickets(tenant_id, status);
CREATE INDEX idx_tickets_assignee ON tickets(assignee_id) WHERE assignee_id IS NOT NULL;
CREATE INDEX idx_tickets_team ON tickets(team_id) WHERE team_id IS NOT NULL;
CREATE INDEX idx_tickets_requester ON tickets(requester_id);
CREATE INDEX idx_tickets_priority ON tickets(tenant_id, priority, status);
CREATE INDEX idx_tickets_created ON tickets(tenant_id, created_at DESC);
CREATE INDEX idx_tickets_tags ON tickets USING GIN(tags);
CREATE INDEX idx_tickets_custom_fields ON tickets USING GIN(custom_fields);
CREATE INDEX idx_tickets_ai ON tickets USING GIN(ai_classification);
```

---

## Messages

```sql
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    author_type TEXT NOT NULL CHECK (author_type IN ('contact', 'agent', 'system', 'ai_bot')),
    author_id UUID,
    channel TEXT,
    message_type TEXT NOT NULL DEFAULT 'reply' CHECK (message_type IN ('reply', 'note', 'system', 'ai_draft')),
    body_text TEXT,
    body_html TEXT,
    is_private BOOLEAN NOT NULL DEFAULT FALSE,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"email_headers": {...}, "ai_confidence": 0.95, "suggested_articles": ["uuid1", "uuid2"]}
    attachments JSONB NOT NULL DEFAULT '[]',
    -- [{"id": "uuid", "file_name": "screenshot.png", "content_type": "image/png", "size_bytes": 45000, "storage_key": "s3://..."}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_messages_ticket ON messages(ticket_id, created_at);
```

---

## SLA Policies

```sql
CREATE TABLE sla_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    position INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    conditions JSONB NOT NULL DEFAULT '{}',
    -- {"all": [{"field": "priority", "op": "is", "value": "urgent"}, {"field": "custom_fields.product_area", "op": "is", "value": "billing"}]}
    targets JSONB NOT NULL DEFAULT '{}',
    -- {"urgent": {"first_response": 900, "resolution": 14400}, "high": {"first_response": 3600, "resolution": 28800}}
    business_hours JSONB NOT NULL DEFAULT '{}',
    -- {"timezone": "America/New_York", "schedule": {"mon": {"start": "09:00", "end": "17:00"}, ...}, "holidays": [...]}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sla_policies_tenant ON sla_policies(tenant_id, is_active, position);
```

---

## Knowledge Base

```sql
CREATE TABLE kb_articles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    body_markdown TEXT NOT NULL,
    body_html TEXT NOT NULL,
    category_path TEXT NOT NULL DEFAULT '/',
    -- ltree-style path: "/getting-started/billing" — no separate categories table
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'published', 'archived')),
    locale TEXT NOT NULL DEFAULT 'en',
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"author_id": "uuid", "view_count": 1234, "helpful": 89, "not_helpful": 12, "auto_generated_from_ticket": "uuid", "related_articles": ["uuid1", "uuid2"]}
    search_vector TSVECTOR,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug, locale)
);

CREATE INDEX idx_kb_articles_search ON kb_articles USING GIN(search_vector);
CREATE INDEX idx_kb_articles_category ON kb_articles(tenant_id, category_path);
CREATE INDEX idx_kb_articles_status ON kb_articles(tenant_id, status);
```

---

## Automation Rules

```sql
CREATE TABLE automation_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    trigger_event TEXT NOT NULL CHECK (trigger_event IN ('ticket_created', 'ticket_updated', 'time_based', 'sla_approaching', 'sla_breached')),
    conditions JSONB NOT NULL DEFAULT '{}',
    -- {"all": [{"field": "priority", "op": "is", "value": "urgent"}, {"field": "ai_classification.sentiment", "op": "is", "value": "frustrated"}]}
    actions JSONB NOT NULL DEFAULT '[]',
    -- [{"type": "assign_team", "team_id": "..."}, {"type": "add_tag", "tag": "escalated"}, {"type": "send_notification", "channel": "slack", "message": "..."}]
    position INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    run_count BIGINT NOT NULL DEFAULT 0,
    last_run_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_automation_rules_tenant ON automation_rules(tenant_id, is_active, trigger_event);
```

---

## Audit & Compliance

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    action TEXT NOT NULL,
    actor_type TEXT NOT NULL,
    actor_id UUID,
    changes JSONB NOT NULL DEFAULT '{}',
    -- {"status": {"from": "new", "to": "open"}, "assignee_id": {"from": null, "to": "uuid"}}
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);

CREATE TABLE erasure_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    contact_id UUID NOT NULL,
    requested_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed')),
    entities_affected JSONB NOT NULL DEFAULT '{}'
    -- {"tickets_anonymized": 5, "messages_anonymized": 23, "contact_deleted": true}
);
```

---

## Integrations

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
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
    failure_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    tenant_id UUID NOT NULL,
    response_status INT NOT NULL,
    response_body JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '24 hours'
);
```

---

## Example Queries

### Filter tickets by custom field value

```sql
SELECT id, subject, status, custom_fields->>'product_area' AS product_area
FROM tickets
WHERE tenant_id = 'tenant-uuid'
  AND custom_fields @> '{"product_area": "billing"}'
  AND status IN ('new', 'open');
```

### Find frustrated customers with high-value contracts

```sql
SELECT t.id, t.subject, t.ai_classification->>'sentiment' AS sentiment,
       c.traits->>'contract_value_cents' AS contract_value
FROM tickets t
JOIN contacts c ON t.requester_id = c.id
WHERE t.tenant_id = 'tenant-uuid'
  AND t.ai_classification @> '{"sentiment": "frustrated"}'
  AND (c.traits->>'contract_value_cents')::BIGINT > 10000000
  AND t.status IN ('new', 'open', 'pending');
```

### SLA breach report with custom field breakdown

```sql
SELECT custom_fields->>'product_area' AS area,
       COUNT(*) FILTER (WHERE (sla_status->'first_response'->>'breached')::BOOLEAN) AS fr_breaches,
       COUNT(*) FILTER (WHERE (sla_status->'resolution'->>'breached')::BOOLEAN) AS res_breaches
FROM tickets
WHERE tenant_id = 'tenant-uuid'
  AND created_at >= now() - INTERVAL '30 days'
GROUP BY custom_fields->>'product_area';
```

---

## Row-Level Security

```sql
ALTER TABLE tickets ENABLE ROW LEVEL SECURITY;
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE agents ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON tickets
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON contacts
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON agents
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core | 3 | tenants, agents, teams |
| Contacts | 1 | contacts (with traits JSONB) |
| Tickets | 1 | tickets (with custom_fields, channel_metadata, ai_classification, sla_status JSONB) |
| Messages | 1 | messages (with attachments JSONB array) |
| SLA | 1 | sla_policies (conditions + targets + business_hours all JSONB) |
| Knowledge Base | 1 | kb_articles (with metadata JSONB, ltree-style category_path) |
| Automation | 1 | automation_rules (conditions + actions JSONB) |
| Audit | 2 | audit_log, erasure_log |
| Integrations | 3 | api_keys, webhooks, idempotency_keys |
| **Total** | **14** | |

---

## Key Design Decisions

1. **Custom fields as JSONB, not EAV** — The `tickets.custom_fields` column replaces the normalized model's 2-table EAV pattern. GIN index enables containment queries. Trade-off: no DB-level type enforcement on custom field values — validation must happen at the application layer or via JSON Schema.

2. **Channel metadata inline** — Email headers, chat session data, and social media post IDs are stored in `tickets.channel_metadata` JSONB rather than separate per-channel tables. Adding a new channel type requires zero schema changes.

3. **AI classification inline** — Instead of a separate `ai_classifications` table, predictions are stored in `tickets.ai_classification` JSONB. This simplifies queries ("show frustrated tickets") but loses the ability to track multiple classification versions per ticket at the DB level.

4. **SLA status inline** — Each ticket carries its own `sla_status` JSONB with target times and breach flags. SLA recalculation updates this field directly. No separate SLA instance table.

5. **Attachments as JSONB array** — Message attachments are a JSONB array on the `messages` table rather than a separate join table. Simpler reads but no foreign key to a file storage table.

6. **Agent team membership as UUID array** — `agents.team_ids` is a `UUID[]` column rather than a junction table. GIN-indexed for containment queries. Trade-off: no cascading deletes or foreign key enforcement on team membership.

7. **KB categories as path strings** — `kb_articles.category_path` uses a forward-slash-delimited path ("/getting-started/billing") instead of a separate categories table with adjacency lists. Supports prefix queries without recursive CTEs.

8. **14 tables total** — Less than half the normalized model. Every JSONB column is a deliberate trade of referential integrity for schema flexibility and query simplicity.
