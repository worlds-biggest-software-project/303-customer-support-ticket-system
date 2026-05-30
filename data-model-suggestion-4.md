# Data Model Suggestion 4: Graph-Relational

> Project: Customer Support Ticket System · Created: 2026-05-24

## Philosophy

This model layers a property graph on top of relational tables to capture the rich relationship network inherent in customer support: tickets relate to other tickets (duplicates, parent/child, root cause), customers belong to organizations that have contracts, agents have skills that match ticket topics, knowledge articles resolve specific ticket types, and root-cause clusters link tickets to product defects. These relationships are first-class entities — not implicit JOINs or tags, but explicit edges with types, weights, and metadata.

The relational tables handle operational CRUD (creating tickets, tracking SLA, storing messages), while the graph layer (`graph_nodes` and `graph_edges`) enables traversal queries that would require multiple recursive CTEs in a pure relational model: "find all tickets related to this product defect, including tickets linked by duplicate or root-cause relationships, and the knowledge articles that resolved them." The graph also powers AI features like root-cause clustering, predictive routing (which agent's skill graph best matches this ticket's topic graph?), and proactive support (which customer's usage pattern graph predicts an imminent ticket?).

**Best for:** Teams building AI-powered root-cause analysis, relationship-aware routing, and knowledge graph features where understanding connections between entities is as important as the entities themselves.

**Trade-offs:**
- **Pro:** Rich relationship queries without recursive CTEs — graph traversal is natural
- **Pro:** Root-cause clustering and ticket linking are first-class, not afterthoughts
- **Pro:** Agent-skill-topic matching via graph enables predictive routing
- **Pro:** Knowledge graph connects articles, tickets, product areas, and defects
- **Con:** Higher write overhead — every relationship is an explicit edge insert
- **Con:** Graph queries require learning graph traversal patterns
- **Con:** More complex schema than JSONB hybrid
- **Con:** PostgreSQL graph queries (recursive CTEs) are less efficient than native graph DBs for deep traversals

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ITIL 4 | Ticket lifecycle states in relational tables; incident-problem-known_error relationships modeled as graph edges |
| ISO/IEC 20000-1:2018 | Problem management (grouping incidents to problems) is a graph relationship, not a flat parent_id |
| GDPR | Graph edges enable tracing all data related to a contact for right-to-erasure |
| W3C RDF/OWL | Graph edge types (relates_to, caused_by, resolves, duplicates) follow semantic web relationship patterns |
| ITIL Problem Management | Incident → Problem → Known Error → Change relationship chain modeled as directed graph |

---

## Graph Layer

```sql
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    node_type TEXT NOT NULL CHECK (node_type IN (
        'ticket', 'contact', 'organization', 'agent', 'team',
        'kb_article', 'product_area', 'defect', 'skill', 'channel'
    )),
    entity_id UUID NOT NULL,
    label TEXT NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, entity_id)
);

CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type TEXT NOT NULL CHECK (edge_type IN (
        -- Ticket relationships
        'duplicate_of', 'related_to', 'caused_by', 'parent_of', 'child_of',
        'merged_into', 'escalated_from',
        -- Resolution relationships
        'resolved_by_article', 'resolved_by_agent', 'auto_resolved',
        -- Problem management (ITIL)
        'incident_of_problem', 'problem_has_known_error', 'known_error_fixed_by',
        -- Organizational
        'belongs_to', 'member_of', 'assigned_to',
        -- Knowledge
        'covers_topic', 'requires_skill', 'has_skill',
        -- AI-derived
        'similar_to', 'root_cause_cluster', 'predicted_escalation'
    )),
    weight REAL NOT NULL DEFAULT 1.0,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"confidence": 0.92, "model_version": "v3.2", "discovered_at": "2026-05-24", "similarity_score": 0.87}
    created_by TEXT NOT NULL DEFAULT 'system' CHECK (created_by IN ('agent', 'system', 'ai', 'automation')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_tenant ON graph_nodes(tenant_id, node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_graph_edges_type ON graph_edges(tenant_id, edge_type);
```

---

## Operational Tables

### Tenants & Agents

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
    role TEXT NOT NULL DEFAULT 'agent' CHECK (role IN ('owner', 'admin', 'agent', 'light_agent')),
    status TEXT NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT,
    name TEXT,
    external_id TEXT,
    traits JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customer_organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    domain TEXT,
    traits JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agents_tenant ON agents(tenant_id);
CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);
```

### Tickets

```sql
CREATE TABLE tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    ticket_number BIGINT NOT NULL,
    subject TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'new' CHECK (status IN ('new', 'open', 'pending', 'on_hold', 'solved', 'closed')),
    priority TEXT NOT NULL DEFAULT 'normal' CHECK (priority IN ('urgent', 'high', 'normal', 'low')),
    ticket_type TEXT NOT NULL DEFAULT 'incident' CHECK (ticket_type IN ('incident', 'question', 'problem', 'task', 'feature_request', 'known_error')),
    channel TEXT NOT NULL DEFAULT 'web_form',
    requester_id UUID NOT NULL REFERENCES contacts(id),
    assignee_id UUID REFERENCES agents(id),
    team_id UUID REFERENCES teams(id),
    tags TEXT[] NOT NULL DEFAULT '{}',
    custom_fields JSONB NOT NULL DEFAULT '{}',
    ai_classification JSONB NOT NULL DEFAULT '{}',
    first_responded_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ticket_number)
);

CREATE INDEX idx_tickets_tenant_status ON tickets(tenant_id, status);
CREATE INDEX idx_tickets_assignee ON tickets(assignee_id);
CREATE INDEX idx_tickets_requester ON tickets(requester_id);
CREATE INDEX idx_tickets_created ON tickets(tenant_id, created_at DESC);
CREATE INDEX idx_tickets_tags ON tickets USING GIN(tags);
```

### Messages

```sql
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    author_type TEXT NOT NULL CHECK (author_type IN ('contact', 'agent', 'system', 'ai_bot')),
    author_id UUID,
    message_type TEXT NOT NULL DEFAULT 'reply' CHECK (message_type IN ('reply', 'note', 'system', 'ai_draft')),
    body_text TEXT,
    body_html TEXT,
    is_private BOOLEAN NOT NULL DEFAULT FALSE,
    attachments JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_messages_ticket ON messages(ticket_id, created_at);
```

### SLA

```sql
CREATE TABLE sla_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    conditions JSONB NOT NULL DEFAULT '{}',
    targets JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
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

CREATE INDEX idx_sla_instances_breach ON ticket_sla_instances(breached, target_at) WHERE NOT breached;
```

### Knowledge Base

```sql
CREATE TABLE kb_articles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    body_markdown TEXT NOT NULL,
    body_html TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'published', 'archived')),
    search_vector TSVECTOR,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_kb_search ON kb_articles USING GIN(search_vector);
```

### Audit

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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);
```

---

## Skills & Product Areas (Graph-Enriched Entities)

```sql
CREATE TABLE skills (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    category TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE product_areas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    parent_id UUID REFERENCES product_areas(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE defects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    description TEXT,
    product_area_id UUID REFERENCES product_areas(id),
    status TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'investigating', 'fix_in_progress', 'resolved', 'wont_fix')),
    external_issue_url TEXT,
    ticket_count INT NOT NULL DEFAULT 0,
    first_reported_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Graph Queries

### Find all tickets related to a defect (including transitive relationships)

```sql
WITH RECURSIVE related_tickets AS (
    -- Start from the defect node
    SELECT ge.target_node_id AS node_id, ge.edge_type, 1 AS depth
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.source_node_id = gn.id
    WHERE gn.entity_id = 'defect-uuid'
      AND gn.node_type = 'defect'
      AND ge.edge_type IN ('caused_by', 'incident_of_problem', 'related_to')

    UNION

    -- Traverse related tickets
    SELECT ge.target_node_id, ge.edge_type, rt.depth + 1
    FROM related_tickets rt
    JOIN graph_edges ge ON ge.source_node_id = rt.node_id
    WHERE ge.edge_type IN ('duplicate_of', 'related_to')
      AND rt.depth < 5
)
SELECT DISTINCT t.*
FROM related_tickets rt
JOIN graph_nodes gn ON gn.id = rt.node_id AND gn.node_type = 'ticket'
JOIN tickets t ON t.id = gn.entity_id;
```

### Find best agent for a ticket based on skill-topic graph matching

```sql
WITH ticket_topics AS (
    -- Get topics/skills required by this ticket
    SELECT ge.target_node_id AS skill_node_id
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.source_node_id = gn.id
    WHERE gn.entity_id = 'ticket-uuid'
      AND gn.node_type = 'ticket'
      AND ge.edge_type = 'requires_skill'
),
agent_skills AS (
    -- Get agents with matching skills
    SELECT gn_agent.entity_id AS agent_id,
           COUNT(*) AS matching_skills,
           AVG(ge.weight) AS avg_skill_weight
    FROM ticket_topics tt
    JOIN graph_edges ge ON ge.target_node_id = tt.skill_node_id
    JOIN graph_nodes gn_agent ON gn_agent.id = ge.source_node_id
    WHERE ge.edge_type = 'has_skill'
      AND gn_agent.node_type = 'agent'
    GROUP BY gn_agent.entity_id
)
SELECT a.id, a.name, ags.matching_skills, ags.avg_skill_weight
FROM agent_skills ags
JOIN agents a ON a.id = ags.agent_id
WHERE a.status = 'active'
ORDER BY ags.matching_skills DESC, ags.avg_skill_weight DESC
LIMIT 5;
```

### Find knowledge articles that resolved similar tickets

```sql
SELECT ka.id, ka.title, COUNT(*) AS resolution_count
FROM graph_nodes gn_ticket
JOIN graph_edges ge_sim ON ge_sim.source_node_id = gn_ticket.id AND ge_sim.edge_type = 'similar_to'
JOIN graph_edges ge_res ON ge_res.source_node_id = ge_sim.target_node_id AND ge_res.edge_type = 'resolved_by_article'
JOIN graph_nodes gn_article ON gn_article.id = ge_res.target_node_id AND gn_article.node_type = 'kb_article'
JOIN kb_articles ka ON ka.id = gn_article.entity_id
WHERE gn_ticket.entity_id = 'current-ticket-uuid'
  AND gn_ticket.node_type = 'ticket'
GROUP BY ka.id, ka.title
ORDER BY resolution_count DESC;
```

### ITIL Problem Management: Incidents → Problem → Known Error → Fix

```sql
-- Trace the full ITIL chain from an incident to its resolution
WITH chain AS (
    SELECT gn.id AS node_id, gn.node_type, gn.entity_id, gn.label,
           ge.edge_type, 0 AS depth
    FROM graph_nodes gn
    LEFT JOIN graph_edges ge ON ge.source_node_id = gn.id
    WHERE gn.entity_id = 'incident-ticket-uuid'
      AND gn.node_type = 'ticket'

    UNION ALL

    SELECT gn2.id, gn2.node_type, gn2.entity_id, gn2.label,
           ge2.edge_type, c.depth + 1
    FROM chain c
    JOIN graph_edges ge2 ON ge2.source_node_id = c.node_id
    JOIN graph_nodes gn2 ON gn2.id = ge2.target_node_id
    WHERE ge2.edge_type IN ('incident_of_problem', 'problem_has_known_error', 'known_error_fixed_by')
      AND c.depth < 4
)
SELECT node_type, entity_id, label, edge_type, depth
FROM chain
ORDER BY depth;
```

---

## Row-Level Security

```sql
ALTER TABLE graph_nodes ENABLE ROW LEVEL SECURITY;
ALTER TABLE graph_edges ENABLE ROW LEVEL SECURITY;
ALTER TABLE tickets ENABLE ROW LEVEL SECURITY;
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON graph_nodes
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON graph_edges
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON tickets
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON contacts
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Core Entities | 5 | tenants, agents, teams, contacts, customer_organizations |
| Tickets | 1 | tickets (with JSONB for custom_fields, ai_classification) |
| Messages | 1 | messages |
| SLA | 2 | sla_policies, ticket_sla_instances |
| Knowledge Base | 1 | kb_articles |
| Graph Entities | 3 | skills, product_areas, defects |
| Audit | 1 | audit_log |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Generic graph layer** — `graph_nodes` and `graph_edges` are entity-agnostic. Any entity (ticket, contact, article, defect, skill) can be a node; any relationship (duplicate_of, resolved_by, has_skill) can be an edge. This enables new relationship types without schema changes.

2. **ITIL Problem Management as graph** — The incident → problem → known error → change request chain is modeled as directed graph edges rather than hierarchical parent_id columns. This supports many-to-many relationships (multiple incidents for one problem, one known error affecting multiple problems).

3. **AI-created edges** — The `created_by` field on edges distinguishes human-created relationships from AI-discovered ones (similarity, clustering, predicted escalation). The `weight` and `metadata.confidence` fields let the UI filter by confidence threshold.

4. **Skill-based routing via graph** — Agent skills and ticket topic requirements are graph nodes connected by `has_skill` and `requires_skill` edges. Routing queries traverse the graph to find the best-matching agent rather than relying on static team assignment rules.

5. **Defects as first-class entities** — Unlike other models where root-cause clustering is a read model or analytics feature, here defects are entities with their own table and graph connections. This mirrors DevRev's ticket-to-code linking pattern.

6. **Operational tables remain relational** — The graph layer augments but does not replace relational tables. Ticket CRUD, SLA tracking, and message storage use standard relational patterns. The graph is for relationship queries, not operational reads.

7. **Graph traversal depth limits** — All recursive CTEs include depth limits (typically 5) to prevent runaway queries. For very deep relationship chains, pre-computed `graph_paths` could be added as a materialised view.

8. **Dual-write pattern** — When a ticket is created or updated, the application writes to both the relational table and the graph layer (creating/updating nodes and edges). This is a trade-off: more write complexity for richer query capabilities.
