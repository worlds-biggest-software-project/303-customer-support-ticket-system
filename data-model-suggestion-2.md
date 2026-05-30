# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Customer Support Ticket System · Created: 2026-05-24

## Philosophy

Every change to a ticket — creation, assignment, status transition, message added, SLA breach, AI classification — is recorded as an immutable event in a single append-only event store. The event stream is the source of truth; all queryable state (ticket dashboards, agent queues, SLA reports) is derived by replaying events into materialised read models (CQRS pattern).

This approach is a natural fit for customer support, where audit trails are not optional — GDPR, HIPAA, SOC 2, and ITIL all demand a complete, tamper-proof record of every action taken on a case. Unlike the normalized model where audit logs are a secondary concern bolted onto mutable tables, here the audit trail *is* the data. Temporal queries ("what was the ticket status at 3pm yesterday?", "who was assigned when the SLA breached?") are answered by replaying events up to that timestamp rather than reconstructing from change logs.

The event-sourced model also enables AI-powered pattern analysis: the event stream is training data for predicting resolution times, identifying bottleneck agents, and detecting tickets likely to escalate.

**Best for:** Regulated environments requiring complete audit trails, teams building AI on top of ticket interaction patterns, and deployments where temporal queries ("what was true at time T?") are a core requirement.

**Trade-offs:**
- **Pro:** Complete, immutable audit history — every change is a first-class record
- **Pro:** Temporal queries are trivial (replay to any point in time)
- **Pro:** Event stream doubles as AI/ML training data
- **Pro:** GDPR erasure can anonymize events without deleting audit structure
- **Con:** Read model eventual consistency — slight lag between write and read
- **Con:** Schema evolution requires event versioning (upcasters)
- **Con:** More complex codebase (event handlers, projections, snapshots)
- **Con:** Debugging requires understanding event replay, not just SELECT

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ITIL 4 | Event types map to ITIL incident lifecycle transitions; priority changes tracked as events |
| ISO/IEC 20000-1:2018 | Event store provides complete incident management audit trail required by the standard |
| GDPR | Events anonymized (not deleted) on erasure request — audit structure preserved |
| HIPAA | PHI access logged as events; BAA compliance demonstrable through event replay |
| SOC 2 | Event store is the audit log — no separate audit table needed |
| CloudEvents 1.0 | Event envelope follows CloudEvents spec (type, source, subject, time, data) |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    stream_type TEXT NOT NULL CHECK (stream_type IN ('ticket', 'contact', 'agent', 'sla_policy', 'kb_article')),
    stream_id UUID NOT NULL,
    sequence_number BIGINT NOT NULL,
    event_type TEXT NOT NULL,
    -- CloudEvents-aligned envelope
    ce_source TEXT NOT NULL DEFAULT '/ticket-system',
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"actor_type": "agent", "actor_id": "uuid", "ip_address": "...", "user_agent": "..."}
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_event_store_stream ON event_store(stream_id, sequence_number);
CREATE INDEX idx_event_store_tenant ON event_store(tenant_id, occurred_at DESC);
CREATE INDEX idx_event_store_type ON event_store(event_type, occurred_at DESC);
```

### Event Type Registry

```sql
CREATE TABLE event_type_registry (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    data_schema JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Ticket events
INSERT INTO event_type_registry (event_type, stream_type, description, data_schema) VALUES
('ticket.created',       'ticket', 'Ticket created',                '{"subject": "text", "description": "text", "priority": "enum", "channel": "text", "requester_id": "uuid"}'),
('ticket.assigned',      'ticket', 'Ticket assigned to agent',      '{"assignee_id": "uuid", "team_id": "uuid"}'),
('ticket.status_changed','ticket', 'Ticket status changed',         '{"from": "enum", "to": "enum"}'),
('ticket.priority_changed','ticket','Priority changed',             '{"from": "enum", "to": "enum", "reason": "text"}'),
('ticket.message_added', 'ticket', 'Message added to ticket',       '{"message_id": "uuid", "author_type": "enum", "author_id": "uuid", "is_private": "bool", "channel": "text"}'),
('ticket.tagged',        'ticket', 'Tag added',                     '{"tag": "text", "source": "enum"}'),
('ticket.untagged',      'ticket', 'Tag removed',                   '{"tag": "text"}'),
('ticket.merged',        'ticket', 'Ticket merged into another',    '{"merged_into_id": "uuid"}'),
('ticket.sla_applied',   'ticket', 'SLA policy applied',            '{"sla_policy_id": "uuid", "targets": "jsonb"}'),
('ticket.sla_breached',  'ticket', 'SLA target breached',           '{"metric": "enum", "target_at": "timestamptz"}'),
('ticket.ai_classified', 'ticket', 'AI classification applied',     '{"intent": "text", "sentiment": "text", "confidence": "float", "model_version": "text"}'),
('ticket.ai_responded',  'ticket', 'AI auto-response sent',         '{"response_id": "uuid", "confidence": "float"}'),
('ticket.satisfaction_rated','ticket','Customer satisfaction rated', '{"rating": "int", "comment": "text"}'),
('ticket.escalated',     'ticket', 'Ticket escalated',              '{"from_team_id": "uuid", "to_team_id": "uuid", "reason": "text"}'),
('ticket.field_changed', 'ticket', 'Custom field changed',          '{"field_key": "text", "from": "any", "to": "any"}'),
('ticket.erased',        'ticket', 'PII erased (GDPR)',             '{"erasure_request_id": "uuid", "fields_erased": "text[]"}');
```

---

## Snapshots (Performance Optimization)

```sql
CREATE TABLE stream_snapshots (
    stream_id UUID NOT NULL,
    stream_type TEXT NOT NULL,
    sequence_number BIGINT NOT NULL,
    snapshot_data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);
```

---

## Materialised Read Models (CQRS Projections)

### Ticket Read Model

```sql
CREATE TABLE rm_tickets (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    ticket_number BIGINT NOT NULL,
    subject TEXT NOT NULL,
    status TEXT NOT NULL,
    priority TEXT NOT NULL,
    ticket_type TEXT NOT NULL,
    channel TEXT,
    requester_id UUID NOT NULL,
    assignee_id UUID,
    team_id UUID,
    organization_id UUID,
    tags TEXT[] NOT NULL DEFAULT '{}',
    custom_fields JSONB NOT NULL DEFAULT '{}',
    message_count INT NOT NULL DEFAULT 0,
    first_responded_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    satisfaction_rating SMALLINT,
    ai_intent TEXT,
    ai_sentiment TEXT,
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_tickets_tenant_status ON rm_tickets(tenant_id, status);
CREATE INDEX idx_rm_tickets_assignee ON rm_tickets(assignee_id);
CREATE INDEX idx_rm_tickets_priority ON rm_tickets(tenant_id, priority, status);
CREATE INDEX idx_rm_tickets_created ON rm_tickets(tenant_id, created_at DESC);
```

### SLA Read Model

```sql
CREATE TABLE rm_sla_status (
    ticket_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    first_response_target_at TIMESTAMPTZ,
    first_response_achieved_at TIMESTAMPTZ,
    first_response_breached BOOLEAN NOT NULL DEFAULT FALSE,
    resolution_target_at TIMESTAMPTZ,
    resolution_achieved_at TIMESTAMPTZ,
    resolution_breached BOOLEAN NOT NULL DEFAULT FALSE,
    next_response_target_at TIMESTAMPTZ,
    next_response_breached BOOLEAN NOT NULL DEFAULT FALSE,
    sla_policy_id UUID,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_sla_pending ON rm_sla_status(tenant_id, first_response_breached)
    WHERE NOT first_response_breached AND first_response_achieved_at IS NULL;
```

### Agent Performance Read Model

```sql
CREATE TABLE rm_agent_stats (
    agent_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    period_date DATE NOT NULL,
    tickets_assigned INT NOT NULL DEFAULT 0,
    tickets_resolved INT NOT NULL DEFAULT 0,
    avg_resolution_seconds BIGINT,
    avg_first_response_seconds BIGINT,
    sla_breaches INT NOT NULL DEFAULT 0,
    satisfaction_sum INT NOT NULL DEFAULT 0,
    satisfaction_count INT NOT NULL DEFAULT 0,
    ai_assists_used INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (agent_id, period_date)
);

CREATE INDEX idx_rm_agent_stats_tenant ON rm_agent_stats(tenant_id, period_date);
```

### Root-Cause Clustering Read Model

```sql
CREATE TABLE rm_ticket_clusters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    cluster_label TEXT NOT NULL,
    cluster_type TEXT NOT NULL CHECK (cluster_type IN ('product_defect', 'documentation_gap', 'feature_request', 'recurring_question')),
    ticket_count INT NOT NULL DEFAULT 0,
    representative_ticket_id UUID,
    first_seen_at TIMESTAMPTZ NOT NULL,
    last_seen_at TIMESTAMPTZ NOT NULL,
    status TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'acknowledged', 'resolved')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_ticket_cluster_members (
    cluster_id UUID NOT NULL REFERENCES rm_ticket_clusters(id) ON DELETE CASCADE,
    ticket_id UUID NOT NULL,
    similarity_score REAL NOT NULL,
    added_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (cluster_id, ticket_id)
);
```

---

## Reference Data (Non-Event Tables)

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'agent',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT,
    name TEXT,
    external_id TEXT,
    organization_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL,
    author_type TEXT NOT NULL,
    author_id UUID,
    channel TEXT,
    message_type TEXT NOT NULL DEFAULT 'reply',
    body_text TEXT,
    body_html TEXT,
    is_private BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
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

CREATE TABLE sla_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    conditions JSONB NOT NULL DEFAULT '{}',
    targets JSONB NOT NULL DEFAULT '{}',
    -- {"urgent": {"first_response": 3600, "resolution": 14400}, ...}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE kb_articles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    body_markdown TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'draft',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Projection Checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_sequence BIGINT NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status TEXT NOT NULL DEFAULT 'running' CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message TEXT
);
```

---

## GDPR Erasure via Event Anonymization

```sql
-- Example: anonymize a contact's data across all events
-- Events are NOT deleted — audit structure is preserved
UPDATE event_store
SET event_data = event_data
    - 'requester_email'
    - 'requester_name'
    || '{"erased": true, "erasure_request_id": "uuid..."}'::JSONB,
    metadata = metadata || '{"anonymized_at": "2026-05-24T00:00:00Z"}'::JSONB
WHERE stream_id IN (
    SELECT id FROM rm_tickets WHERE requester_id = 'contact-uuid-to-erase'
);
```

---

## Example: Temporal Query — Ticket State at Point in Time

```sql
-- Reconstruct ticket state as of a specific timestamp
SELECT
    e.stream_id AS ticket_id,
    (SELECT event_data->>'to' FROM event_store e2
     WHERE e2.stream_id = e.stream_id AND e2.event_type = 'ticket.status_changed'
       AND e2.occurred_at <= '2026-05-20 15:00:00+00'
     ORDER BY e2.sequence_number DESC LIMIT 1) AS status_at_time,
    (SELECT event_data->>'assignee_id' FROM event_store e2
     WHERE e2.stream_id = e.stream_id AND e2.event_type = 'ticket.assigned'
       AND e2.occurred_at <= '2026-05-20 15:00:00+00'
     ORDER BY e2.sequence_number DESC LIMIT 1) AS assignee_at_time
FROM event_store e
WHERE e.stream_id = 'ticket-uuid'
  AND e.event_type = 'ticket.created'
LIMIT 1;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store, event_type_registry, stream_snapshots |
| Read Models | 5 | rm_tickets, rm_sla_status, rm_agent_stats, rm_ticket_clusters, rm_ticket_cluster_members |
| Reference Data | 8 | tenants, agents, contacts, teams, messages, message_attachments, sla_policies, kb_articles |
| Projections | 1 | projection_checkpoints |
| **Total** | **17** | |

---

## Key Design Decisions

1. **Event store as source of truth** — All state changes are immutable events. The `rm_` prefixed tables are derived projections that can be rebuilt from scratch by replaying the event stream. This eliminates "audit log drift" where the audit trail disagrees with the current state.

2. **CloudEvents envelope** — Events include `ce_source` and `ce_specversion` fields aligning with the CloudEvents 1.0 spec, enabling future integration with event-driven architectures and external event consumers.

3. **Stream-scoped sequences** — Each event stream (ticket, contact) has its own monotonically increasing `sequence_number`, enabling optimistic concurrency control and ordered replay without global locking.

4. **Snapshots every N events** — `stream_snapshots` stores periodic state snapshots to avoid replaying hundreds of events for long-lived tickets. Snapshot interval is configurable per stream type.

5. **GDPR via anonymization, not deletion** — Events are never deleted. GDPR erasure anonymizes PII fields within events while preserving the audit structure. The `erased` flag marks anonymized events.

6. **Projection checkpoints** — Each read model tracks which event it last processed, enabling pause/resume/rebuild of individual projections without affecting the event store.

7. **Root-cause clustering as a projection** — The AI clustering model processes the event stream to group related tickets, storing results in `rm_ticket_clusters`. As new ticket events arrive, clusters update automatically.

8. **Simplified reference tables** — Non-event entities (tenants, agents, SLA policies, KB articles) remain as simple relational tables. Only ticket lifecycle and interaction data is event-sourced — avoiding over-engineering for slowly-changing reference data.
