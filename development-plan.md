# Customer Support Ticket System — Phased Development Plan

> Project: 303-customer-support-ticket-system · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The database design adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB)** as the operational schema — chosen because it gives ~15 core tables for fast MVP iteration while retaining typed columns for the fields that drive indexing, SLA reporting, and routing. The **SLA policy → targets → instances** three-tier structure from Model 1 is layered in (Phase 5), and the **root-cause cluster read model** from Model 2 is adopted as a derived projection (Phase 11). Event sourcing (Model 2) and the property graph (Model 4) are explicitly *not* adopted as the base model to avoid over-engineering the MVP; an append-only `audit_logs` table provides the compliance trail instead.

---

## Core Requirements (Synthesis)

**What it does.** An AI-native, open-source, multi-tenant help desk: omnichannel ticket intake (email, chat, web form), ITIL-aligned ticket lifecycle, rules-based routing and SLA enforcement, an agent workspace with conversation history and internal notes, a knowledge base, analytics, and a first-class REST API with webhooks. The AI layer adds intent/sentiment triage, agent reply suggestions, autonomous tier-1 resolution, knowledge-base auto-generation, and root-cause clustering.

**Who uses it.** Support operations managers (config, SLA, reporting), support agents (ticket workspace), end customers (portal + email/chat), VPs of Customer Success (analytics), and integrators (REST API/webhooks).

**Differentiators (AI-native).** Autonomous tier-1 resolution with confidence-gated escalation; root-cause clustering surfacing engineering/docs work orders; dynamic SLA prioritisation from sentiment + customer health; KB auto-generation from resolved tickets. These ship in later phases on top of a solid ticketing core.

**Deployment model.** Self-hostable (Docker Compose, single command) and cloud-ready, following Chatwoot's precedent. Multi-tenant from day one.

**Standards the build must honour.** ITIL ticket lifecycle (`new→open→pending→on_hold→solved→closed`) and urgency×impact priority; SLA first-response/resolution targets with business hours; OpenAPI 3.1 REST API; OAuth 2.0 + API keys; idempotency keys (RFC 9110); HMAC-SHA256 signed webhooks; GDPR right-to-erasure + retention; TLS 1.2+ and AES-256 at rest.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **Python 3.12** | The product's centre of gravity is LLM orchestration (triage, autonomous resolution, KB generation, clustering). Python has the richest LLM/embedding ecosystem and keeps AI and API code in one language. |
| API framework | **FastAPI** | Native async (needed for streaming LLM calls and webhook fan-out), Pydantic v2 request/response models, and automatic **OpenAPI 3.1** generation — a hard requirement from `standards.md`. |
| ASGI server | **Uvicorn** behind **Gunicorn** | Standard production combo for FastAPI; Gunicorn manages worker lifecycle, Uvicorn provides the async loop. |
| Database | **PostgreSQL 16** | Hybrid model needs JSONB + GIN indexes, partial indexes, `pgvector` for KB/cluster embeddings, range partitioning for audit logs, and Row-Level Security for tenant isolation. SQLite cannot satisfy these. |
| Vector search | **pgvector** extension | Keeps embeddings in the same store as tickets/articles — no separate vector DB to operate for self-hosters. Used for KB semantic search and clustering. |
| ORM / migrations | **SQLAlchemy 2.0 (async)** + **Alembic** | Mature async ORM; Alembic gives versioned, reviewable migrations required for a schema that evolves across phases. |
| Task queue | **Celery** + **Redis** broker | Async workloads (inbound email polling, LLM triage, webhook delivery with retries, SLA breach sweeps, embedding generation) must run off the request path. Celery has mature retry/backoff and beat scheduling. |
| Scheduler | **Celery Beat** | Periodic SLA breach sweeps, retention enforcement, business-hours recomputation. |
| Cache / rate limiting | **Redis** | Reused as cache, rate-limit token buckets, and Celery broker/result backend. |
| LLM access | **LiteLLM** gateway | Provider-agnostic interface (OpenAI, Anthropic, local) so self-hosters can point at any model. Centralises retries, cost logging, and model routing. |
| Embeddings | **LiteLLM embeddings** → pgvector | Same gateway; default to a small embedding model, configurable. |
| Email | **aioimaplib** (inbound) + **aiosmtplib** (outbound) | Async IMAP poll and SMTP send for the email channel. |
| Frontend | **React 18 + TypeScript + Vite**, **TanStack Query**, **Tailwind**, **shadcn/ui** | Agent workspace and admin are dashboard-heavy SPAs; SPA + REST keeps a clean separation from the API. Customer portal is server-rendered minimal pages from the same build. |
| Realtime | **WebSocket (FastAPI)** + Redis pub/sub | Live ticket updates and chat require push; Redis pub/sub fans out across API workers. |
| Auth | **OAuth 2.0** (Authlib) + **API keys** + JWT sessions | `standards.md` mandates OAuth 2.0 for integrations and API keys for direct access. JWT for agent SPA sessions. |
| Containerisation | **Docker** + **docker-compose** | Self-hosted one-command deploy (api, worker, beat, postgres, redis, frontend). |
| Testing | **pytest** + **pytest-asyncio** + **httpx** + **testcontainers** | Async tests, real Postgres/Redis via testcontainers for integration, httpx ASGI client for API e2e. |
| Frontend testing | **Vitest** + **Playwright** | Unit/component tests and browser e2e for the agent workspace. |
| Lint / format / types | **Ruff** + **Black** + **mypy** (py); **ESLint** + **Prettier** + **tsc** (ts) | Standard, fast toolchain per ecosystem. |
| Package managers | **uv** (Python) / **pnpm** (frontend) | Fast, reproducible installs. |
| Config | **pydantic-settings** | Typed env config with validation and defaults. |
| Object storage | **S3-compatible** via **boto3** (MinIO for self-host) | Attachments stored out of Postgres; MinIO ships in compose for self-hosters. |

### Project Structure

```
customer-support-ticket-system/
├── pyproject.toml
├── uv.lock
├── alembic.ini
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── README.md
├── openapi.json                      # exported spec (CI artifact)
├── migrations/                       # Alembic
│   ├── env.py
│   └── versions/
├── src/
│   └── support/
│       ├── __init__.py
│       ├── main.py                   # FastAPI app factory, lifespan, router mounting
│       ├── config.py                 # pydantic-settings Settings
│       ├── db.py                     # async engine, session, RLS tenant setter
│       ├── deps.py                   # FastAPI dependencies (auth, tenant, pagination)
│       ├── errors.py                 # error model + exception handlers
│       ├── models/                   # SQLAlchemy ORM models
│       │   ├── base.py
│       │   ├── tenant.py  agent.py  team.py  contact.py  channel.py
│       │   ├── ticket.py  message.py  attachment.py
│       │   ├── sla.py  automation.py  kb.py  ai.py  audit.py  integration.py
│       ├── schemas/                  # Pydantic request/response models
│       ├── repositories/             # data-access layer (queries)
│       ├── services/                 # business logic
│       │   ├── tickets.py  routing.py  sla.py  automation.py
│       │   ├── kb.py  analytics.py  webhooks.py  erasure.py
│       ├── ai/                       # LLM-facing logic
│       │   ├── client.py             # LiteLLM wrapper, cost logging
│       │   ├── triage.py             # intent/sentiment/language
│       │   ├── suggest.py            # reply suggestions (RAG over KB)
│       │   ├── autonomous.py         # tier-1 resolver agent + tools
│       │   ├── kb_generation.py      # article drafting from resolved tickets
│       │   ├── clustering.py         # root-cause clustering
│       │   └── prompts/              # versioned prompt templates (*.md)
│       ├── channels/                 # intake adapters
│       │   ├── email.py  webform.py  chat.py
│       ├── api/                      # FastAPI routers
│       │   ├── v1/
│       │   │   ├── tickets.py  messages.py  contacts.py  agents.py
│       │   │   ├── sla.py  automation.py  kb.py  analytics.py
│       │   │   ├── webhooks.py  auth.py  ai.py  portal.py
│       │   └── ws.py                 # websocket endpoints
│       ├── workers/                  # Celery
│       │   ├── app.py  email_poll.py  triage_tasks.py
│       │   ├── webhook_delivery.py  sla_sweep.py  retention.py
│       │   ├── embedding_tasks.py  clustering_tasks.py
│       └── lib/                      # cross-cutting (hashing, hmac, paging, idempotency)
├── frontend/
│   ├── package.json  vite.config.ts  tailwind.config.ts
│   └── src/
│       ├── api/        # generated client from openapi.json
│       ├── components/ pages/ hooks/ stores/
│       └── portal/     # customer-facing pages
└── tests/
    ├── conftest.py     # testcontainers fixtures (pg, redis), app client
    ├── unit/  integration/  e2e/
    └── fixtures/       # sample emails, tickets, SARIF-free JSON payloads
```

The structure is grouped by concern (api / services / ai / channels / workers), so every phase adds files without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Config, DB, Tenancy

### Purpose
Establish the runnable spine: FastAPI app, typed config, async Postgres with Alembic migrations, multi-tenant Row-Level Security, Redis, Celery wiring, Docker Compose, and the CI quality gate. After this phase the app boots, a health check passes, and a migration creates the `tenants` and `agents` tables with tenant isolation enforced.

### Tasks

#### 1.1 — Project scaffolding & quality gate
**What**: Create the repo skeleton, dependency manifests, and lint/format/type/test tooling.

**Design**:
- `pyproject.toml` with dependencies from the tech table; tool sections for Ruff, Black, mypy (strict), pytest.
- `src/support/main.py` exposes `create_app() -> FastAPI` (app factory) used by both server and tests.
- `GET /healthz` returns `{"status": "ok", "version": <str>}`; `GET /readyz` checks DB + Redis connectivity.
- CI (GitHub Actions) runs: `ruff check`, `black --check`, `mypy`, `pytest`, frontend `tsc`/`vitest`, then exports `openapi.json` as an artifact.

**Testing**:
- `Unit: create_app() returns FastAPI instance with /healthz route registered`
- `Integration (real pg+redis via testcontainers): GET /readyz → 200 when both up`
- `Integration: GET /readyz → 503 when Redis unreachable (broker URL pointed at dead port)`
- `E2E: docker compose up; curl /healthz → 200`

#### 1.2 — Typed configuration
**What**: Centralised `Settings` loaded from environment.

**Design**:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str
    secret_key: str                       # JWT signing
    llm_provider: str = "openai"
    llm_model: str = "gpt-4o-mini"
    embedding_model: str = "text-embedding-3-small"
    s3_endpoint: str | None = None
    s3_bucket: str = "support-attachments"
    default_data_residency: Literal["us", "eu", "ap"] = "us"
    webhook_max_retries: int = 6
    rate_limit_per_minute: int = 300
    model_config = SettingsConfigDict(env_prefix="SUPPORT_", env_file=".env")
```
`.env.example` lists every variable with safe defaults. Settings is a cached singleton via `@lru_cache`.

**Testing**:
- `Unit: env with all required vars → Settings populated, defaults applied`
- `Unit: missing SUPPORT_DATABASE_URL → ValidationError naming the field`
- `Unit: invalid default_data_residency value → ValidationError`

#### 1.3 — Async DB layer, migrations, tenant RLS
**What**: SQLAlchemy async engine, session dependency, Alembic baseline, and RLS-based tenant isolation.

**Design**:
- `db.py`: `async_engine`, `async_session_maker`, `get_session()` dependency.
- Tenant context: dependency sets `SET LOCAL app.current_tenant_id = :tid` on the session per request; all tenant-scoped queries rely on RLS rather than manual `WHERE tenant_id=`.
- Baseline migration creates `tenants` and `agents` (from Model 3 / Model 1):
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL, slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free'
        CHECK (plan IN ('free','starter','professional','enterprise')),
    settings JSONB NOT NULL DEFAULT '{}',
    data_residency TEXT NOT NULL DEFAULT 'us'
        CHECK (data_residency IN ('us','eu','ap')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT NOT NULL, name TEXT NOT NULL,
    password_hash TEXT,
    role TEXT NOT NULL DEFAULT 'agent'
        CHECK (role IN ('owner','admin','agent','light_agent')),
    status TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active','inactive','suspended')),
    timezone TEXT NOT NULL DEFAULT 'UTC', signature TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
ALTER TABLE agents ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_agents ON agents
    USING (tenant_id = current_setting('app.current_tenant_id', true)::UUID);
```

**Testing**:
- `Integration: alembic upgrade head then downgrade base succeeds cleanly`
- `Integration: insert agents for tenant A and B; with app.current_tenant_id=A set, SELECT returns only A's rows`
- `Integration: query without tenant setting → zero rows (RLS denies)`

#### 1.4 — Celery & Redis wiring
**What**: Celery app with Redis broker/backend and a smoke task.

**Design**: `workers/app.py` builds the Celery app from `Settings`; `ping` task returns `"pong"`. Beat schedule registered (empty for now). Compose adds `worker` and `beat` services.

**Testing**:
- `Integration (real redis): apply ping task, .get() == "pong"`
- `Integration: beat schedule loads without error`

### Definition of Done
App boots, `/healthz` and `/readyz` pass, migrations up/down cleanly, RLS isolation proven, Celery ping works, all lint/type/test gates green, compose stack starts.

---

## Phase 2: Core Domain — Tickets, Messages, Contacts, Channels

### Purpose
Build the heart of the product: the ticket lifecycle, conversation messages (public replies + private notes), contacts/organisations, and channel records. This phase delivers full CRUD via the REST API with ITIL-aligned states, ticket numbering, and an audit log — the minimum to manage tickets manually.

### Tasks

#### 2.1 — Core schema (Hybrid Relational + JSONB, Model 3)
**What**: Migration adding contacts, organizations, channels, tickets, messages, attachments, and audit_logs.

**Design** (key tables, JSONB for the long tail):
```sql
CREATE TABLE customer_organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL, domain TEXT, external_id TEXT,
    plan_tier TEXT, health_score SMALLINT CHECK (health_score BETWEEN 0 AND 100),
    contract_value_cents BIGINT, attributes JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, domain)
);
CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT, phone TEXT, name TEXT, external_id TEXT,
    organization_id UUID REFERENCES customer_organizations(id),
    locale TEXT DEFAULT 'en', timezone TEXT,
    attributes JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    channel_type TEXT NOT NULL
        CHECK (channel_type IN ('email','chat','web_form','api')),
    configuration JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);
CREATE TABLE tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    ticket_number BIGINT NOT NULL,
    subject TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'new'
        CHECK (status IN ('new','open','pending','on_hold','solved','closed')),
    priority TEXT NOT NULL DEFAULT 'normal'
        CHECK (priority IN ('urgent','high','normal','low')),
    urgency TEXT CHECK (urgency IN ('high','medium','low')),
    impact  TEXT CHECK (impact  IN ('high','medium','low')),
    ticket_type TEXT NOT NULL DEFAULT 'incident'
        CHECK (ticket_type IN ('incident','question','problem','task','feature_request')),
    channel_id UUID REFERENCES channels(id),
    requester_id UUID NOT NULL REFERENCES contacts(id),
    assignee_id UUID REFERENCES agents(id),
    team_id UUID,                               -- FK added Phase 3
    organization_id UUID REFERENCES customer_organizations(id),
    tags TEXT[] NOT NULL DEFAULT '{}',
    custom_fields JSONB NOT NULL DEFAULT '{}',  -- Model 3: no EAV
    language TEXT DEFAULT 'en',
    first_responded_at TIMESTAMPTZ, resolved_at TIMESTAMPTZ, closed_at TIMESTAMPTZ,
    satisfaction_rating SMALLINT CHECK (satisfaction_rating BETWEEN 1 AND 5),
    is_spam BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ticket_number)
);
CREATE INDEX idx_tickets_tenant_status ON tickets(tenant_id, status);
CREATE INDEX idx_tickets_assignee ON tickets(assignee_id) WHERE assignee_id IS NOT NULL;
CREATE INDEX idx_tickets_custom_fields ON tickets USING GIN (custom_fields);
CREATE INDEX idx_tickets_tags ON tickets USING GIN (tags);
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    author_type TEXT NOT NULL CHECK (author_type IN ('contact','agent','system','ai_bot')),
    author_id UUID,
    message_type TEXT NOT NULL DEFAULT 'reply'
        CHECK (message_type IN ('reply','note','system','ai_suggestion')),
    body_text TEXT, body_html TEXT,
    is_private BOOLEAN NOT NULL DEFAULT FALSE,
    channel_metadata JSONB NOT NULL DEFAULT '{}',  -- email headers, chat session
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_ticket ON messages(ticket_id, created_at);
```
`message_attachments` and `audit_logs` (range-partitioned by `created_at`, `changes` JSONB diff) as per Model 1/3. RLS policies on tickets/messages/contacts. `ticket_number` allocated per-tenant via a `tenant_counters` row with `SELECT ... FOR UPDATE`.

**Testing**:
- `Integration: migration up/down clean; GIN indexes present`
- `Unit: ticket number allocator returns 1,2,3 sequentially per tenant; isolated across tenants`
- `Integration: RLS blocks cross-tenant ticket/message reads`

#### 2.2 — Ticket lifecycle service & state machine
**What**: Service enforcing valid ITIL status transitions and stamping lifecycle timestamps.

**Design**:
- Allowed transitions: `new→{open,solved,closed}`, `open→{pending,on_hold,solved}`, `pending→{open,solved}`, `on_hold→{open,solved}`, `solved→{open,closed}`, `closed→{open}` (reopen). Invalid transitions raise `InvalidTransition`.
- On first agent public reply, set `first_responded_at` if null. On `→solved` set `resolved_at`; on `→closed` set `closed_at`.
- Every mutation writes an `audit_logs` row: `{action, actor_type, actor_id, changes:{field:{from,to}}}`.

**Testing**:
- `Unit: new→open allowed; new→pending raises InvalidTransition`
- `Unit: first agent reply sets first_responded_at once, not on second reply`
- `Unit: →solved stamps resolved_at; audit row records status from/to`

#### 2.3 — Tickets, messages, contacts REST API (OpenAPI 3.1)
**What**: CRUD + list endpoints with cursor pagination and filtering.

**Design** (selected endpoints):
```
POST   /v1/tickets            body: TicketCreate            → 201 TicketOut
GET    /v1/tickets            ?status&priority&assignee_id&cursor&limit → TicketPage
GET    /v1/tickets/{id}                                     → TicketOut
PATCH  /v1/tickets/{id}       body: TicketUpdate            → TicketOut
POST   /v1/tickets/{id}/messages  body: MessageCreate       → 201 MessageOut
GET    /v1/tickets/{id}/messages                            → MessagePage
POST   /v1/contacts  GET /v1/contacts  GET/PATCH /v1/contacts/{id}
```
- `TicketCreate`: `subject, requester{email,name}|requester_id, channel_id?, priority?, body, tags?, custom_fields?`. Upserts requester contact by email.
- Cursor pagination: opaque base64 of `(created_at, id)`; `limit` default 25 max 100.
- All error responses use the shared model `{error:{code,message,details?}}`.

**Testing**:
- `Integration: POST /v1/tickets minimal body → 201, ticket_number=1, status=new, requester contact created`
- `Integration: POST with existing requester email → reuses contact`
- `Integration: GET list filtered by status=open → only open tickets, paginated`
- `Integration: PATCH status new→pending → 409 InvalidTransition`
- `E2E: create ticket, add agent note (is_private=true) then public reply; GET messages returns both ordered; first_responded_at set`

#### 2.4 — Idempotent ticket creation (RFC 9110)
**What**: `Idempotency-Key` header support on POST /v1/tickets.

**Design**: `idempotency_keys(key PK, tenant_id, response_status, response_body JSONB, expires_at default now()+24h)`. On request: if key seen, return stored response; else execute, store, return. Concurrent same-key requests serialise via unique-insert; loser returns stored result.

**Testing**:
- `Integration: two POSTs with same Idempotency-Key → one ticket created, identical responses`
- `Integration: same key different tenant → independent (no collision)`
- `Integration: expired key reused → new ticket created`

### Definition of Done
Tickets/messages/contacts CRUD works end-to-end; lifecycle transitions enforced and audited; idempotent creation proven; endpoints appear in generated `openapi.json`; RLS verified; all gates green.

---

## Phase 3: Agents, Teams, Auth & RBAC

### Purpose
Add identity and access control: agent login (JWT), API keys and OAuth 2.0 for integrations, teams and memberships, and role-based permissions. After this phase every endpoint is authenticated and authorised, enabling the agent workspace and external integrations.

### Tasks

#### 3.1 — Teams schema & ticket FK
**What**: `teams`, `team_memberships`, and the deferred `tickets.team_id` FK.

**Design**: tables from Model 1 (`teams`, `team_memberships` with role lead/member). Add `ALTER TABLE tickets ADD CONSTRAINT fk_tickets_team FOREIGN KEY (team_id) REFERENCES teams(id)`.

**Testing**: `Integration: create team, add agent membership, assign ticket.team_id → FK enforced; cross-tenant team assignment rejected.`

#### 3.2 — Agent authentication (JWT)
**What**: Password login issuing short-lived access + refresh JWTs.

**Design**:
```
POST /v1/auth/login    {email,password} → {access_token, refresh_token, expires_in}
POST /v1/auth/refresh  {refresh_token}  → {access_token, expires_in}
POST /v1/auth/logout   → 204 (refresh token revoked via Redis denylist)
```
Passwords hashed with Argon2id. Access token claims: `sub=agent_id, tid=tenant_id, role, exp`. `get_current_agent` dependency decodes token and sets tenant RLS context.

**Testing**:
- `Unit: Argon2 hash/verify round-trip; wrong password fails`
- `Integration: login valid → tokens; expired access token → 401`
- `Integration: refresh after logout (denylisted) → 401`

#### 3.3 — API keys & OAuth 2.0 (RFC 6749)
**What**: API key issuance/verification and OAuth 2.0 client-credentials flow for integrations.

**Design**:
- `api_keys(key_hash UNIQUE, key_prefix, scopes TEXT[], expires_at, ...)`. Key shown once as `sk_<prefix>_<secret>`; only SHA-256 hash stored. `Authorization: Bearer sk_...`.
- OAuth 2.0 client-credentials via Authlib: `oauth_applications(client_id, client_secret_hash, scopes)`; `POST /v1/oauth/token` issues scoped access tokens.
- Scopes: `tickets:read tickets:write contacts:write webhooks:manage kb:read analytics:read`.

**Testing**:
- `Unit: generated key verifies against its hash; tampered key fails`
- `Integration: request with key lacking tickets:write → 403 on POST /v1/tickets`
- `Integration: OAuth client-credentials → token usable; expired → 401`

#### 3.4 — RBAC enforcement
**What**: Role + scope checks across endpoints.

**Design**: `require(permission)` dependency. Matrix: `owner/admin` full config; `agent` ticket CRUD + own profile; `light_agent` read + private notes only. Light agents cannot send public replies or change SLA/automation config.

**Testing**:
- `Unit: permission matrix table-driven — each (role, action) → allow/deny as specified`
- `Integration: light_agent POST public reply → 403; private note → 201`

### Definition of Done
JWT login/refresh/logout, API keys, and OAuth all work; RBAC enforced and table-tested; teams wired to tickets; auth applied to all v1 routes; gates green.

---

## Phase 4: Routing & Automation Engine

### Purpose
Add rules-based ticket routing/assignment and a general automation engine (trigger → conditions → actions). This is the deterministic backbone the AI layer later augments — assignment, tagging, priority setting, and escalation run without human clicks.

### Tasks

#### 4.1 — Condition/action DSL & evaluator
**What**: JSONB rule format and a pure evaluator.

**Design**:
```json
{"all":[{"field":"priority","operator":"is","value":"urgent"},
        {"field":"organization.plan_tier","operator":"is","value":"enterprise"}]}
```
- Operators: `is, is_not, contains, not_contains, greater_than, less_than, changed, in`.
- Actions: `assign_team, assign_agent, set_priority, add_tag, set_status, send_webhook, notify_agent`.
- `evaluate(conditions, ticket_context) -> bool`; `apply_actions(actions, ticket) -> list[Change]`. Dotted field paths resolve into the ticket + joined org/contact context. Pure functions, no I/O.

**Testing**:
- `Unit: all/any nesting evaluates correctly; unknown field → False, logged`
- `Unit: greater_than on non-numeric → ValidationError at rule-save time`
- `Unit: apply_actions(set_priority high) returns Change(priority, from, to)`

#### 4.2 — Automation rules schema & runner
**What**: `automation_rules` table (Model 1) and event-driven runner.

**Design**: trigger events `ticket_created, ticket_updated, time_based, sla_breach`. On a domain event, load active rules for the tenant ordered by `position`, evaluate, apply matching actions, write audit rows tagged `source=automation`. Time-based rules executed by a Celery Beat sweep.

**Testing**:
- `Integration: rule (created & priority=urgent → assign_team T) fires on matching create; non-matching create untouched`
- `Integration: two rules by position apply in order; second sees first's changes`
- `Integration: time_based rule (status=open >24h → escalate) fires in sweep`

#### 4.3 — Round-robin & load-based assignment
**What**: Assignment strategies for `assign_team`/auto-assign.

**Design**: strategies `round_robin` (per-team cursor in Redis) and `least_loaded` (fewest open assigned tickets). Pluggable `AssignmentStrategy` interface; predictive strategy added in Phase 9.

**Testing**:
- `Unit: round_robin cycles A,B,C,A across 4 calls`
- `Integration: least_loaded picks agent with fewest open tickets; ties broken deterministically`

### Definition of Done
Rule DSL evaluator tested; automation runner fires on create/update/time/SLA events; round-robin and least-loaded assignment work; all actions audited; gates green.

---

## Phase 5: SLA Management & Business Hours

### Purpose
Implement SLA policies, per-priority targets, business-hours-aware deadline calculation, breach detection, and escalation. Delivers the contractual first-response/resolution tracking that `standards.md` and every competitor treat as table stakes.

### Tasks

#### 5.1 — SLA schema (three-tier, Model 1)
**What**: `sla_policies`, `sla_targets`, `business_hours`, `ticket_sla_instances`.

**Design**: exactly the Model 1 structure — policy (match conditions) → targets (per priority × metric `first_response|next_response|resolution`, `target_seconds`, `business_hours_only`) → instances (per ticket per metric: `target_at`, `achieved_at`, `breached`). `business_hours.schedule` JSONB weekly windows + `holidays` JSONB.

**Testing**: `Integration: migration up/down; partial index on unbreached instances present.`

#### 5.2 — Business-hours deadline calculator
**What**: Add N business seconds to a start timestamp respecting schedule, holidays, timezone.

**Design**: `add_business_seconds(start, seconds, schedule, holidays, tz) -> datetime`. Walks forward across open windows; skips closed days/holidays. Pure function.

**Testing**:
- `Unit: Fri 16:00 + 4h business (9–17) → Mon 12:00`
- `Unit: target spanning a holiday skips it`
- `Unit: business_hours_only=false → simple wall-clock add`
- `Unit: DST boundary handled (tz-aware)`

#### 5.3 — SLA application & breach sweep
**What**: Attach SLA on ticket create/update; detect breaches.

**Design**: on create, first matching policy by position creates instances with computed `target_at`. First agent public reply marks `first_response.achieved_at`; `→solved` marks `resolution.achieved_at`. Celery Beat sweep (every 60s) flags `breached=true` for past-due unachieved instances, emits `ticket.sla_breached` event (feeds automation Phase 4), and triggers escalation actions.

**Testing**:
- `Integration: urgent ticket gets 1h first-response target_at; reply within 1h → achieved, not breached`
- `Integration: no reply past target → sweep sets breached, emits event, escalation rule fires`
- `Integration (mocked clock): next_response target recalculated after each customer reply`

### Definition of Done
SLA policies/targets/business-hours configurable; deadlines correct incl. holidays/DST; breach sweep flags + escalates; SLA status queryable per ticket; gates green.

---

## Phase 6: Knowledge Base & Semantic Search

### Purpose
Add the knowledge base — categories, articles, public help-centre rendering, and pgvector-backed semantic search. This is both an end-customer self-service surface and the retrieval corpus for AI reply suggestions and autonomous resolution.

### Tasks

#### 6.1 — KB schema + embeddings
**What**: `kb_categories`, `kb_articles`, `ticket_article_links`, plus an embedding column.

**Design**: Model 1 KB tables. Add `kb_articles.embedding vector(1536)` and `CREATE INDEX ... USING ivfflat (embedding vector_cosine_ops)`. On publish, a Celery task computes the embedding via LiteLLM and stores it. `body_markdown` rendered to sanitised `body_html` on save.

**Testing**:
- `Integration: publish article → embedding task runs, vector stored (mocked embedding endpoint)`
- `Unit: markdown→html sanitises script tags`

#### 6.2 — KB REST API & search
**What**: Article CRUD, category tree, and hybrid search.

**Design**:
```
POST/GET/PATCH /v1/kb/articles     GET /v1/kb/articles/{slug}
GET /v1/kb/search?q=...&limit=     → ranked articles
```
Search = semantic (cosine over pgvector) blended with Postgres full-text `ts_rank`; weighted sum, configurable. Returns snippet + score.

**Testing**:
- `Integration: search "reset password" ranks the password article first (seeded fixtures, mocked query embedding)`
- `Integration: draft articles excluded from public search`
- `Integration: GET /v1/kb/articles/{slug} increments view_count`

#### 6.3 — Public help centre & portal
**What**: Server-rendered help-centre pages and a customer ticket-status portal.

**Design**: `GET /portal/{tenant_slug}/kb`, `/portal/{tenant_slug}/kb/{slug}`; `/portal/{tenant_slug}/tickets/{token}` shows status/history via signed token (no login). Article "helpful?" buttons increment `helpful_count`/`not_helpful_count`.

**Testing**:
- `E2E: GET help-centre lists published categories/articles`
- `E2E: portal ticket link with valid signed token → status shown; tampered token → 403`

### Definition of Done
KB CRUD + category tree; embeddings generated on publish; hybrid search ranks correctly; public help centre and portal render; ticket↔article linking works; gates green.

---

## Phase 7: Webhooks & REST API Hardening

### Purpose
Make the platform integrable: signed, retrying outbound webhooks; rate limiting; consistent pagination/errors; and a finalised, published OpenAPI 3.1 spec with a generated TypeScript client. After this phase external systems can subscribe to ticket events reliably.

### Tasks

#### 7.1 — Webhook subscriptions & signed delivery
**What**: `webhooks` table, event emission, and HMAC-signed delivery with retries.

**Design**:
- `webhooks(url, events TEXT[], secret_hash, is_active, failure_count)`. Events: `ticket.created/updated/status_changed/assigned/solved/closed`, `message.created`, `sla.breached`.
- Delivery payload is a CloudEvents-style envelope `{id,type,source,subject,time,data}`. Header `X-Signature: sha256=<hmac>` over the raw body using the per-webhook secret.
- Celery task delivers with exponential backoff (`webhook_max_retries`, default 6). After max failures, mark inactive and emit `webhook.disabled`.

**Testing**:
- `Integration (mock receiver): ticket.created → POST with valid HMAC header; receiver verifies signature`
- `Integration: receiver returns 500 → retried with backoff; succeeds on retry 3`
- `Integration: persistent 500 → disabled after max retries`

#### 7.2 — Rate limiting
**What**: Per-tenant/per-key token-bucket limiting.

**Design**: Redis sliding-window or token bucket keyed by tenant+key; default `rate_limit_per_minute`. Over limit → `429` with `Retry-After`. `X-RateLimit-Limit/Remaining/Reset` headers on all responses.

**Testing**:
- `Integration: N+1 requests in a minute → 429 with Retry-After; counter resets after window`

#### 7.3 — OpenAPI 3.1 finalisation & TS client
**What**: Polish the spec and generate the frontend client.

**Design**: tags, examples, security schemes (bearer JWT, api key, oauth2) declared. CI exports `openapi.json` and runs `openapi-typescript` to regenerate `frontend/src/api`; CI fails if the committed client is stale.

**Testing**:
- `Unit: openapi.json validates against OpenAPI 3.1 meta-schema`
- `Integration: every v1 route appears with a documented security scheme`
- `CI: regenerated client matches committed client (no drift)`

### Definition of Done
Webhooks deliver with valid HMAC, retry, and auto-disable; rate limiting enforced with headers; OpenAPI 3.1 validates; TS client generated and drift-checked; gates green.

---

## Phase 8: AI Triage & Agent Reply Suggestions

### Purpose
Introduce the first AI layer: automatic intent/sentiment/language classification on inbound tickets (feeding routing + dynamic prioritisation), and RAG-based reply suggestions for agents grounded in the knowledge base. Human-in-the-loop — AI proposes, agents decide — building the correction data the autonomous tier (Phase 10) depends on.

### Tasks

#### 8.1 — LLM client & cost logging
**What**: A single LiteLLM-backed client wrapping all model calls.

**Design**:
```python
async def complete(messages, *, model=None, temperature=0.2,
                   response_format=None, tenant_id) -> LLMResult
async def embed(texts: list[str], *, model=None) -> list[list[float]]
```
Every call logs `{tenant_id, model, prompt_tokens, completion_tokens, cost_cents, latency_ms, feature}` to `ai_usage`. Retries with backoff; structured-output calls validated against a Pydantic schema.

**Testing**:
- `Unit (mocked gateway): complete() returns parsed result; cost logged`
- `Unit: structured output failing schema → one retry then AIError`

#### 8.2 — Triage classifier
**What**: Classify intent, sentiment, language, suggested priority/team on ticket create.

**Design**: `ai_classifications` table (Model 1). Celery task on `ticket.created` calls `complete()` with the triage prompt; structured output:
```json
{"intent":"billing_question","intent_confidence":0.92,
 "sentiment":"frustrated","sentiment_confidence":0.81,
 "language":"en","suggested_priority":"high","suggested_team":"billing"}
```
Prompt template versioned in `ai/prompts/triage.md` (system: role + allowed enums + JSON schema; user: subject + body). Above a confidence threshold, results feed automation (auto-tag, suggested routing); always stored for agent review and feedback.

**Testing**:
- `Integration (mocked LLM): billing ticket → intent=billing_question stored; ai_usage logged`
- `Integration: low-confidence result does not auto-apply routing, only stores`
- `Unit: malformed LLM JSON → no crash, classification marked failed`

#### 8.3 — RAG reply suggestions
**What**: Suggest a draft reply citing KB articles.

**Design**: `POST /v1/tickets/{id}/ai/suggest` → retrieve top-k KB articles by embedding similarity to the ticket thread, prompt the model to draft a grounded reply, store in `ai_suggested_responses` with `knowledge_article_ids` and `confidence`. Agent can accept/edit/reject; feedback (`helpful|not_helpful|partially_helpful`) recorded.

**Testing**:
- `Integration (mocked LLM+real pgvector): suggest on password ticket → draft cites password article; suggestion stored`
- `Integration: agent marks not_helpful → feedback persisted`
- `Integration: no relevant articles → suggestion flagged low_confidence, no fabricated citations`

### Definition of Done
LLM client with cost logging; triage runs on create and stores classifications with confidence-gated routing; RAG suggestions grounded in KB with citations and feedback capture; prompts versioned; gates green.

---

## Phase 9: Agent Workspace, Realtime & Analytics Frontend

### Purpose
Ship the human-facing product: the React agent workspace (ticket queues, conversation view, reply composer with AI suggestions, internal notes), live updates over WebSocket, and the analytics dashboards (response/resolution time, CSAT, SLA compliance, agent performance) that VPs of Customer Success need.

### Tasks

#### 9.1 — Analytics aggregation API
**What**: Metrics endpoints over tickets/SLA/CSAT.

**Design**:
```
GET /v1/analytics/overview?from&to&team_id  → {avg_first_response_s, avg_resolution_s,
                                               csat, ticket_volume, sla_compliance_pct}
GET /v1/analytics/agents?from&to             → per-agent rows
GET /v1/analytics/trends?metric&interval     → time series
```
Computed via SQL aggregates; heavier rollups cached in Redis (5-min TTL) and a nightly `rm_agent_stats` (Model 2) materialisation by Celery Beat.

**Testing**:
- `Integration: seed tickets with known timings → overview returns expected averages & SLA %`
- `Integration: agent metrics exclude other tenants (RLS)`

#### 9.2 — Realtime updates (WebSocket + Redis pub/sub)
**What**: Push ticket/message changes to connected agents.

**Design**: `WS /v1/ws?token=` authenticated via JWT. On domain events, publish to Redis channel `tenant:{tid}:tickets`; each API worker relays to its subscribed sockets. Messages: `{type:"ticket.updated", ticket_id, fields}`.

**Testing**:
- `Integration: two WS clients in tenant A; ticket update → both receive; tenant B client does not`
- `Integration: WS with invalid token → closed with 4401`

#### 9.3 — Agent workspace SPA
**What**: React app: login, ticket list with filters/saved views, conversation thread, reply composer (public reply / private note), AI-suggestion panel, assignment + status + priority controls, SLA countdown.

**Design**: TanStack Query against the generated client; WebSocket hook invalidates queries on push; optimistic updates for status/assignment. shadcn/ui components; accessible (keyboard-navigable queue).

**Testing**:
- `Vitest: ticket-list filter state reducer`
- `Playwright e2e: login → open ticket → send reply → status auto open→pending; AI suggest panel inserts draft; SLA timer renders`

#### 9.4 — Analytics dashboard SPA
**What**: Charts for volume, response/resolution, CSAT, SLA compliance, agent leaderboard.

**Design**: date-range + team filters; charts from analytics endpoints; CSV export.

**Testing**: `Playwright e2e: dashboard loads, date filter changes numbers; CSV export downloads.`

#### 9.5 — Predictive agent assignment
**What**: Replace static round-robin with predicted-fastest-resolver routing.

**Design**: `PredictiveStrategy` scores candidate agents using historical `rm_agent_stats` (avg resolution for the ticket's intent/team) blended with current load; selects min predicted time. Falls back to least-loaded when history is sparse.

**Testing**:
- `Unit: agent with faster history on this intent chosen over idle-but-slower agent`
- `Unit: no history → falls back to least_loaded`

### Definition of Done
Analytics endpoints accurate and tenant-isolated; realtime push works and is isolated; agent workspace supports the full reply loop incl. AI suggestions; dashboards render and export; predictive routing live with fallback; frontend tests + Playwright e2e green.

---

## Phase 10: Autonomous Tier-1 Resolution

### Purpose
Deliver the flagship differentiator: an agentic loop that fully resolves routine tier-1 tickets (password resets, billing FAQs, how-to) end-to-end, executing safe backend actions via tools, and escalating to humans when confidence is low or the case is out of scope. This is the headcount-lever capability gated behind enterprise contracts at incumbents.

### Tasks

#### 10.1 — Tool framework & registry
**What**: Define callable tools the autonomous agent may invoke.

**Design**:
```python
class Tool(Protocol):
    name: str; description: str; params_schema: dict  # JSON Schema
    requires_scope: str
    async def run(self, args: dict, ctx: ToolContext) -> ToolResult
```
Built-in tools: `search_kb`, `get_customer_info`, `reset_password` (calls a configured tenant webhook/connector), `create_followup_task`, `escalate_to_human`. Each tenant enables/configures tools in `settings`. Side-effecting tools require explicit per-tenant enablement and are audited.

**Testing**:
- `Unit: tool args validated against params_schema; invalid args rejected`
- `Unit: tool requiring disabled scope → not offered to the agent`

#### 10.2 — Autonomous resolver loop
**What**: Confidence-gated agent loop with escalation.

**Design**: on `ticket.created`, if intent ∈ tenant's `autonomous_intents` and confidence ≥ threshold, run a bounded ReAct loop (max N steps): retrieve KB → optionally call tools → draft resolution. Guardrails: max steps, no fabricated actions, mandatory citation for factual claims. If the model emits `escalate_to_human` or confidence drops below threshold, assign to a team and post an internal note summarising what was attempted. Outcomes recorded in `ai_resolutions(ticket_id, outcome, steps JSONB, confidence, escalated, tokens, cost_cents)`.

**Testing**:
- `Integration (mocked LLM + tools): password-reset ticket → reset_password tool called, reply sent, ticket→solved, resolution recorded`
- `Integration: ambiguous ticket → escalate_to_human, assigned to team, internal note added, NOT solved`
- `Integration: tool failure → graceful escalation, no partial state left`
- `Unit: loop terminates at max steps even if model keeps requesting tools`

#### 10.3 — Resolution analytics & guardrail config
**What**: Measure autonomous resolution rate and expose tenant controls.

**Design**: `GET /v1/analytics/autonomous` → `{attempted, resolved, escalated, resolution_rate, csat_on_auto}`. Admin settings: enabled intents, confidence threshold, max steps, allowed tools, "shadow mode" (draft only, never send).

**Testing**:
- `Integration: shadow mode → resolution drafted but ticket stays new, no customer message sent`
- `Integration: analytics computes resolution_rate = resolved/attempted over range`

### Definition of Done
Tool framework with per-tenant enablement and audit; bounded resolver loop resolves routine cases and escalates safely; shadow mode works; resolution analytics accurate; no unsafe state on tool failure; gates green.

---

## Phase 11: Root-Cause Clustering, KB Auto-Generation & Compliance

### Purpose
Close the learning loop and finish compliance. Cluster related tickets to surface engineering/documentation work orders; auto-generate KB articles from resolved tickets; and complete GDPR/HIPAA obligations (right-to-erasure, retention enforcement, data residency). These turn the ticket stream into proactive product intelligence and make the platform deployable in regulated environments.

### Tasks

#### 11.1 — Root-cause clustering
**What**: Group semantically similar tickets into clusters with a type label.

**Design**: adopt Model 2's `rm_ticket_clusters` + `rm_ticket_cluster_members`. A Celery Beat job embeds recent ticket subjects/bodies, runs incremental clustering (cosine threshold / HDBSCAN over embeddings), assigns each cluster a type (`product_defect|documentation_gap|feature_request|recurring_question`) and an LLM-generated label + representative ticket. New tickets attach to existing clusters above similarity threshold.
```
GET /v1/clusters?type&status → clusters with ticket_count, trend, representative
POST /v1/clusters/{id}/work-order → export to Jira/GitHub via connector
```

**Testing**:
- `Integration: 20 seeded "billing double-charge" tickets cluster together; unrelated ticket excluded`
- `Integration: new matching ticket joins existing cluster, increments count`
- `Integration: work-order export posts to mocked Jira connector`

#### 11.2 — KB article auto-generation
**What**: Draft KB articles from resolved-ticket clusters.

**Design**: for `recurring_question` clusters above a size threshold, generate a draft article from the cluster's resolved conversations (prompt in `ai/prompts/kb_draft.md`), create a `draft` `kb_article` linked to source tickets via `ticket_article_links(link_type='auto_generated_from')`. Drafts require human publish (never auto-published).

**Testing**:
- `Integration (mocked LLM): cluster of 10 resolved how-to tickets → draft article created, status=draft, source links recorded`
- `Integration: cluster below threshold → no article`

#### 11.3 — GDPR/HIPAA: erasure, retention, residency
**What**: Right-to-erasure, retention sweeps, residency enforcement.

**Design**:
- `erasure_requests` + `data_retention_policies` (Model 1). `POST /v1/contacts/{id}/erasure` enqueues anonymisation: scrub PII in contacts/messages/attachments, record counts in `entities_erased`, keep audit structure. Retention Beat job applies per-tenant policies (`delete|anonymize|archive`).
- Residency: `tenants.data_residency` controls which storage/region config is used; attachments routed to the matching bucket.
- HIPAA: per-tenant `hipaa_enabled` enforces AES-256 at-rest checks and disables non-compliant channels.

**Testing**:
- `Integration: erasure request → contact PII nulled, messages scrubbed, audit row retained, entities_erased counts correct`
- `Integration: retention policy (anonymize tickets >365d) → old tickets anonymised, recent untouched`
- `Unit: residency=eu selects EU bucket config`

### Definition of Done
Clustering groups tickets and exports work orders; KB drafts auto-generated (human-publish gated); erasure/retention/residency implemented and tested; HIPAA toggle enforces encryption/channel rules; gates green.

---

## Phase 12: Channels — Email, Web Form, Live Chat

### Purpose
Complete omnichannel intake. Email is the dominant support channel and is intentionally built last as a robust adapter over the now-stable ticket core, alongside an embeddable web form and live chat widget. After this phase customers reach support through every channel with unified threading.

### Tasks

#### 12.1 — Inbound/outbound email
**What**: Ingest email into tickets and send replies as email.

**Design**: Celery Beat IMAP poll (`aioimaplib`) per email channel; parse MIME (subject, body text/html, attachments → S3). Thread matching: `In-Reply-To`/`References` headers and a `[#ticket_number]` subject token map replies onto existing tickets; otherwise create new. Outbound agent public replies sent via SMTP (`aiosmtplib`) with the ticket token injected. Loop/auto-reply detection prevents storms.

**Testing**:
- `Integration (fixture .eml files): new email → ticket created, requester upserted, attachment stored`
- `Integration: reply email with References header → appended to existing ticket, not a new one`
- `Integration: auto-reply ("Out of office") → flagged, no ticket/loop`
- `Integration (mock SMTP): agent reply → email sent with ticket token in subject`

#### 12.2 — Web form intake
**What**: Embeddable form that creates tickets.

**Design**: `POST /v1/channels/webform/{channel_id}` (public, captcha + per-IP rate limit) → ticket with `channel_type=web_form`. Configurable fields map to `custom_fields`.

**Testing**:
- `Integration: valid submission → ticket created with mapped custom_fields`
- `Integration: missing required field → 422; over rate limit → 429`

#### 12.3 — Live chat
**What**: WebSocket chat widget creating a chat-channel ticket with realtime two-way messaging.

**Design**: visitor opens `WS /v1/chat/{tenant_slug}` → creates/loads a chat ticket; messages persist as `messages` and stream to agents via the Phase 9 pub/sub. AI triage (Phase 8) and autonomous resolution (Phase 10) can answer before an agent joins; "talk to human" escalates.

**Testing**:
- `Integration: visitor opens chat, sends message → chat ticket created, agent WS receives it`
- `Integration: visitor message answered by autonomous agent; "talk to human" escalates to team`

### Definition of Done
Email round-trips with correct threading and attachments; loop detection works; web form creates tickets with validation/rate limiting; live chat creates tickets and streams both directions with AI-then-human handoff; gates green.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (config, DB, RLS, Celery)        ─── required by everything
   │
Phase 2: Core Domain (tickets/messages/contacts)     ─── requires 1
   │
Phase 3: Agents, Teams, Auth & RBAC                  ─── requires 2
   │
   ├── Phase 4: Routing & Automation Engine          ─── requires 3
   │      │
   │      └── Phase 5: SLA & Business Hours           ─── requires 4 (uses automation events)
   │
   ├── Phase 6: Knowledge Base & Semantic Search      ─── requires 3 (parallel with 4/5)
   │
   └── Phase 7: Webhooks & API Hardening              ─── requires 3 (parallel with 4/5/6)
              │
Phase 8: AI Triage & Reply Suggestions               ─── requires 5 (routing/SLA) + 6 (KB/RAG)
   │
   ├── Phase 9: Agent Workspace, Realtime, Analytics  ─── requires 7 (TS client) + 8
   │
   └── Phase 10: Autonomous Tier-1 Resolution         ─── requires 8 + 7 (tool webhooks); parallel with 9
              │
Phase 11: Clustering, KB Auto-Gen & Compliance       ─── requires 8 (embeddings) + 10 (resolutions)
   │
Phase 12: Channels (email, web form, live chat)      ─── requires 9 (realtime) + 10 (AI in chat)
```

**Parallelism opportunities**
- After Phase 3: **Phases 4, 6, and 7** can be developed concurrently (Phase 5 follows Phase 4).
- After Phase 8: **Phase 9 (frontend)** and **Phase 10 (autonomous)** can be developed concurrently.

**Estimated scope: large** — a multi-tenant, omnichannel, AI-native platform with frontend, async workers, and compliance. 12 phases.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented.
2. All unit and integration tests pass (`pytest`); frontend unit/component tests pass (`vitest`) where applicable.
3. Linting and formatting pass (`ruff check`, `black --check`; `eslint`, `prettier --check`).
4. Type checking passes (`mypy` strict; `tsc --noEmit`).
5. Alembic migrations created, and `upgrade head` / `downgrade` run cleanly.
6. The feature works end-to-end (relevant Playwright e2e green for UI phases).
7. New configuration options added to `.env.example` and documented.
8. New/changed API endpoints appear in the regenerated `openapi.json` with a security scheme, and the generated TS client is regenerated (no drift).
9. Multi-tenant isolation verified (RLS) for any new tenant-scoped tables.
10. New side-effecting or AI actions write `audit_logs` / `ai_usage` records.
11. Docker Compose stack builds and starts with the new components.
```
