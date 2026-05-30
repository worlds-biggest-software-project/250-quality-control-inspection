# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Quality Control & Inspection · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The current state of any entity (inspection, CAPA, non-conformance, SPC study) is derived by replaying its event stream. Materialised read models (views and projections) provide fast query access for dashboards, reports, and SPC charts, but the event store is the single source of truth. This is a CQRS (Command Query Responsibility Segregation) architecture: writes go to the event store, reads come from projections.

This pattern is used in financial trading systems, healthcare record systems, and audit-critical government platforms where the ability to answer "what was the state of this record at 3:47 PM on March 12th?" is a regulatory requirement, not a nice-to-have. In quality management, this is directly relevant to 21 CFR Part 11 (FDA electronic records), AS9100D traceability requirements, and ISO 9001 management review processes where historical state must be provably reconstructable.

The event-sourced model is best suited for organisations where complete audit traceability is the primary design driver — medical device manufacturers under FDA QMSR, aerospace suppliers under AS9100D DCMA oversight, or any environment where a regulator may demand proof of what a quality engineer saw, decided, and documented at any historical point. It also provides a natural foundation for AI-driven pattern analysis, since the full history of every quality event is available for machine learning.

**Best for:** Regulated environments requiring provable temporal state reconstruction, full change traceability, and AI-ready historical event streams.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is permanently recorded with timestamp, user, and context
- (+) Temporal queries are trivial: replay events to any point in time to reconstruct past state
- (+) Natural fit for 21 CFR Part 11, AS9100D, and ISO 13485 traceability requirements
- (+) Event streams are ideal training data for AI anomaly detection and predictive models
- (+) Schema evolution is easier: new event types are additive, old events remain unchanged
- (-) Higher storage requirements — every change is stored, not just current state
- (-) Read model complexity: projections must be maintained and rebuilt when business logic changes
- (-) Eventually consistent reads unless synchronous projections are used (adds latency)
- (-) Steeper learning curve for developers unfamiliar with event sourcing
- (-) Debugging requires understanding event replay rather than simple SELECT queries

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| 21 CFR Part 11 | Every event is an immutable electronic record with user identity, timestamp, and action — inherently satisfies audit trail requirements |
| FDA QMSR (ISO 13485) | Event replay provides provable state reconstruction for any quality record at any point in time |
| AS9100D | Complete traceability chain from design through inspection through non-conformance through corrective action, stored as linked event sequences |
| ISO 9001:2015 | Management review data is reconstructable for any historical period by replaying event streams |
| AIAG SPC Manual | SPC measurement events form a time-series naturally suited to control chart generation and capability recalculation |
| IATF 16949 | PPAP submission lifecycle events provide a complete approval audit trail for automotive customer audits |
| OASIS OSLC QM 2.1 | Event types map to OSLC QM resource lifecycle transitions |
| ISO 23952:2020 (QIF) | QIF Results maps to measurement_recorded events; QIF Plans maps to inspection_plan_created events |

---

## Event Store Core

### Event Store Table

```sql
-- The single source of truth for the entire system
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_id       UUID NOT NULL,          -- aggregate root ID
    stream_type     VARCHAR(100) NOT NULL,   -- 'inspection', 'capa', 'ncr', 'spc_study', etc.
    event_type      VARCHAR(200) NOT NULL,   -- 'inspection.started', 'measurement.recorded', etc.
    event_version   INTEGER NOT NULL,        -- position within stream (for ordering)
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- {"user_id": "...", "ip": "...", "correlation_id": "...", "causation_id": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Optimistic concurrency: no two events can have the same version in the same stream
    UNIQUE (stream_id, event_version)
);

-- Primary query path: get all events for a stream in order
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);

-- Tenant-scoped queries
CREATE INDEX idx_event_tenant_type ON event_store(tenant_id, stream_type, created_at DESC);

-- Global event ordering for projections
CREATE INDEX idx_event_global_order ON event_store(created_at, event_id);

-- Event type filtering for subscribers
CREATE INDEX idx_event_type ON event_store(event_type, created_at DESC);

-- Partition by month for large-scale deployments
-- CREATE TABLE event_store ... PARTITION BY RANGE (created_at);
```

### Snapshot Table (Performance Optimisation)

```sql
-- Periodic snapshots to avoid replaying entire event streams
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    snapshot_version INTEGER NOT NULL,       -- event_version at snapshot time
    state           JSONB NOT NULL,          -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Event Subscriptions (for projection rebuilds)

```sql
CREATE TABLE event_subscription (
    subscription_id VARCHAR(100) PRIMARY KEY,  -- 'projection:inspection_summary', 'webhook:erp_sync'
    last_event_id   UUID,
    last_processed_at TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- active, paused, failed
    checkpoint      JSONB DEFAULT '{}',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Type Catalogue

The following event types define the vocabulary of the system. Each event type has a defined payload schema.

### Inspection Events

```sql
-- Example event payloads (stored in event_store.payload JSONB column)

-- event_type: 'inspection_plan.created'
-- {
--   "plan_number": "IP-2026-0042",
--   "product_id": "uuid",
--   "facility_id": "uuid",
--   "plan_type": "receiving",
--   "name": "Receiving Inspection - Widget A Rev C",
--   "sampling_standard": "ISO_2859_1",
--   "aql": 1.0,
--   "items": [
--     {
--       "item_id": "uuid",
--       "balloon_number": 1,
--       "characteristic_name": "Overall Length",
--       "data_type": "variable",
--       "nominal": 25.400,
--       "usl": 25.450,
--       "lsl": 25.350,
--       "uom": "mm"
--     }
--   ]
-- }

-- event_type: 'inspection.started'
-- {
--   "inspection_number": "INS-2026-01234",
--   "inspection_plan_id": "uuid",
--   "inspector_id": "uuid",
--   "work_order": "WO-5678",
--   "lot_number": "LOT-2026-03-15-A",
--   "quantity_to_inspect": 50
-- }

-- event_type: 'measurement.recorded'
-- {
--   "inspection_id": "uuid",
--   "plan_item_id": "uuid",
--   "sample_number": 1,
--   "measured_value": 25.412,
--   "equipment_id": "uuid",
--   "is_conforming": true,
--   "measured_by": "uuid"
-- }

-- event_type: 'inspection.completed'
-- {
--   "inspection_id": "uuid",
--   "quantity_inspected": 50,
--   "quantity_accepted": 48,
--   "quantity_rejected": 2,
--   "disposition": "accept_with_deviation",
--   "completed_by": "uuid"
-- }

-- event_type: 'inspection.approved'
-- {
--   "inspection_id": "uuid",
--   "approved_by": "uuid",
--   "electronic_signature": {
--     "signer_name": "Jane Smith",
--     "signer_title": "Quality Manager",
--     "meaning": "reviewed_and_approved",
--     "signature_hash": "sha256:..."
--   }
-- }
```

### Non-Conformance Events

```sql
-- event_type: 'ncr.opened'
-- {
--   "ncr_number": "NCR-2026-0089",
--   "nc_type": "incoming",
--   "severity": "major",
--   "title": "Dimension out of tolerance on Widget A",
--   "description": "Balloon #3 measured 25.48mm, USL is 25.45mm",
--   "product_id": "uuid",
--   "supplier_id": "uuid",
--   "inspection_id": "uuid",
--   "quantity_affected": 2,
--   "lot_number": "LOT-2026-03-15-A",
--   "reported_by": "uuid"
-- }

-- event_type: 'ncr.dispositioned'
-- {
--   "ncr_id": "uuid",
--   "disposition": "rework",
--   "disposition_by": "uuid",
--   "cost_of_quality": 450.00,
--   "cost_currency": "USD",
--   "rework_instructions": "Re-machine feature #3 to nominal"
-- }

-- event_type: 'ncr.closed'
-- {
--   "ncr_id": "uuid",
--   "closed_by": "uuid",
--   "closure_notes": "Rework completed and re-inspected. All features conforming."
-- }
```

### CAPA Events

```sql
-- event_type: 'capa.initiated'
-- {
--   "capa_number": "CAPA-2026-0023",
--   "capa_type": "corrective",
--   "methodology": "8d",
--   "title": "Recurring dimension failure on Widget A from Supplier X",
--   "linked_ncr_ids": ["uuid1", "uuid2"],
--   "owner_id": "uuid",
--   "priority": "high"
-- }

-- event_type: 'capa.root_cause_identified'
-- {
--   "capa_id": "uuid",
--   "root_cause_method": "five_why",
--   "root_cause": "Supplier tooling wear not detected — no SPC on supplier side",
--   "identified_by": "uuid"
-- }

-- event_type: 'capa.action_assigned'
-- {
--   "capa_id": "uuid",
--   "action_id": "uuid",
--   "action_type": "corrective",
--   "description": "Require supplier to implement SPC on critical dimensions",
--   "assigned_to": "uuid",
--   "due_date": "2026-04-30"
-- }

-- event_type: 'capa.effectiveness_verified'
-- {
--   "capa_id": "uuid",
--   "verified_by": "uuid",
--   "verification_method": "30-day incoming inspection trend review",
--   "is_effective": true,
--   "evidence_notes": "Zero NCRs on Widget A in 30-day window"
-- }
```

### SPC Events

```sql
-- event_type: 'spc_study.created'
-- {
--   "characteristic_id": "uuid",
--   "chart_type": "xbar_r",
--   "subgroup_size": 5,
--   "facility_id": "uuid"
-- }

-- event_type: 'spc.measurement_collected'
-- {
--   "study_id": "uuid",
--   "subgroup_number": 42,
--   "sample_index": 3,
--   "measured_value": 25.408,
--   "equipment_id": "uuid",
--   "operator_id": "uuid",
--   "machine_id": "uuid"
-- }

-- event_type: 'spc.control_limits_recalculated'
-- {
--   "study_id": "uuid",
--   "ucl": 25.442,
--   "lcl": 25.358,
--   "center_line": 25.400,
--   "cp": 1.33,
--   "cpk": 1.21,
--   "data_points_used": 125
-- }

-- event_type: 'spc.rule_violation_detected'
-- {
--   "study_id": "uuid",
--   "subgroup_number": 42,
--   "rule_name": "nelson_1",
--   "rule_description": "One point beyond 3-sigma",
--   "severity": "alarm",
--   "measured_value": 25.455
-- }
```

### PPAP & Supplier Events

```sql
-- event_type: 'ppap.requested'
-- {
--   "submission_number": "PPAP-2026-0015",
--   "supplier_id": "uuid",
--   "product_id": "uuid",
--   "ppap_level": 3,
--   "requested_by": "uuid",
--   "required_elements": [1, 4, 5, 6, 7, 8, 9, 10, 11, 18]
-- }

-- event_type: 'ppap.element_submitted'
-- {
--   "ppap_id": "uuid",
--   "element_number": 9,
--   "element_name": "Dimensional Results",
--   "attachment_id": "uuid",
--   "submitted_by": "uuid"
-- }

-- event_type: 'ppap.approved'
-- {
--   "ppap_id": "uuid",
--   "disposition": "approved",
--   "reviewed_by": "uuid",
--   "psw_signed": true
-- }

-- event_type: 'supplier.risk_tier_changed'
-- {
--   "supplier_id": "uuid",
--   "previous_tier": "standard",
--   "new_tier": "high",
--   "reason": "3 major NCRs in 90-day window",
--   "changed_by": "uuid"
-- }
```

---

## Materialised Read Models (Projections)

These tables are rebuilt from the event store. They can be dropped and reconstructed at any time.

### Inspection Summary Projection

```sql
CREATE TABLE proj_inspection (
    inspection_id   UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    inspection_number VARCHAR(100) NOT NULL,
    inspection_plan_id UUID NOT NULL,
    facility_id     UUID NOT NULL,
    inspector_id    UUID NOT NULL,
    product_id      UUID,
    product_part_number VARCHAR(100),
    status          VARCHAR(30) NOT NULL,
    work_order      VARCHAR(100),
    lot_number      VARCHAR(100),
    quantity_inspected INTEGER,
    quantity_accepted  INTEGER,
    quantity_rejected  INTEGER,
    disposition     VARCHAR(50),
    total_measurements INTEGER NOT NULL DEFAULT 0,
    conforming_measurements INTEGER NOT NULL DEFAULT 0,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    approved_by     UUID,
    approved_at     TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_insp_tenant ON proj_inspection(tenant_id, status);
CREATE INDEX idx_proj_insp_lot ON proj_inspection(tenant_id, lot_number);
```

### Non-Conformance Summary Projection

```sql
CREATE TABLE proj_non_conformance (
    ncr_id          UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    ncr_number      VARCHAR(100) NOT NULL,
    nc_type         VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    product_id      UUID,
    supplier_id     UUID,
    inspection_id   UUID,
    status          VARCHAR(30) NOT NULL,
    disposition     VARCHAR(50),
    cost_of_quality NUMERIC(12,2),
    reported_by     UUID NOT NULL,
    assigned_to     UUID,
    capa_ids        UUID[],
    opened_at       TIMESTAMPTZ NOT NULL,
    closed_at       TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_ncr_tenant ON proj_non_conformance(tenant_id, status);
CREATE INDEX idx_proj_ncr_supplier ON proj_non_conformance(supplier_id);
```

### CAPA Summary Projection

```sql
CREATE TABLE proj_capa (
    capa_id         UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    capa_number     VARCHAR(100) NOT NULL,
    capa_type       VARCHAR(20) NOT NULL,
    methodology     VARCHAR(20) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL,
    owner_id        UUID NOT NULL,
    linked_ncr_count INTEGER NOT NULL DEFAULT 0,
    total_actions   INTEGER NOT NULL DEFAULT 0,
    completed_actions INTEGER NOT NULL DEFAULT 0,
    root_cause_identified BOOLEAN NOT NULL DEFAULT false,
    is_effective    BOOLEAN,
    due_date        DATE,
    opened_at       TIMESTAMPTZ NOT NULL,
    closed_at       TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_capa_tenant ON proj_capa(tenant_id, status);
CREATE INDEX idx_proj_capa_owner ON proj_capa(owner_id);
```

### SPC Real-Time Projection

```sql
CREATE TABLE proj_spc_study (
    study_id        UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    characteristic_id UUID NOT NULL,
    characteristic_name VARCHAR(255),
    product_id      UUID,
    facility_id     UUID NOT NULL,
    chart_type      VARCHAR(20) NOT NULL,
    subgroup_size   INTEGER NOT NULL,
    ucl             NUMERIC(18,6),
    lcl             NUMERIC(18,6),
    center_line     NUMERIC(18,6),
    cp              NUMERIC(8,4),
    cpk             NUMERIC(8,4),
    pp              NUMERIC(8,4),
    ppk             NUMERIC(8,4),
    is_in_control   BOOLEAN NOT NULL DEFAULT true,
    total_subgroups INTEGER NOT NULL DEFAULT 0,
    active_violations INTEGER NOT NULL DEFAULT 0,
    last_measurement_at TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_spc_char ON proj_spc_study(characteristic_id);

-- SPC measurement time-series for chart rendering
CREATE TABLE proj_spc_chart_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    study_id        UUID NOT NULL,
    subgroup_number INTEGER NOT NULL,
    mean_value      NUMERIC(18,6),
    range_value     NUMERIC(18,6),
    std_dev         NUMERIC(18,6),
    sample_count    INTEGER NOT NULL,
    has_violation   BOOLEAN NOT NULL DEFAULT false,
    violation_rules VARCHAR(50)[],
    collected_at    TIMESTAMPTZ NOT NULL,
    UNIQUE (study_id, subgroup_number)
);

CREATE INDEX idx_proj_spc_chart ON proj_spc_chart_data(study_id, collected_at);
```

### Supplier Scorecard Projection

```sql
CREATE TABLE proj_supplier (
    supplier_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    supplier_code   VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    risk_tier       VARCHAR(20) NOT NULL,
    is_approved     BOOLEAN NOT NULL,
    total_ncrs_ytd  INTEGER NOT NULL DEFAULT 0,
    total_ppaps     INTEGER NOT NULL DEFAULT 0,
    approved_ppaps  INTEGER NOT NULL DEFAULT 0,
    quality_score   NUMERIC(5,2),
    last_ncr_at     TIMESTAMPTZ,
    last_ppap_at    TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_supplier_tenant ON proj_supplier(tenant_id);
```

---

## Platform Reference Tables

These are conventional relational tables for reference data that does not change through events.

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE facility (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    UNIQUE (tenant_id, code)
);
```

---

## Example Queries

### Reconstruct inspection state at a specific point in time

```sql
-- Get all events for an inspection up to a specific timestamp
SELECT event_type, payload, metadata, created_at
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'inspection'
  AND created_at <= '2026-03-15 15:47:00+00'
ORDER BY event_version ASC;

-- Application code replays these events to reconstruct state
```

### Find all quality events for a specific lot (cross-stream)

```sql
SELECT event_type, stream_type, stream_id, payload, created_at
FROM event_store
WHERE tenant_id = 'tenant-uuid'
  AND payload->>'lot_number' = 'LOT-2026-03-15-A'
ORDER BY created_at ASC;
```

### AI training data: all measurement events for a product

```sql
SELECT 
    payload->>'measured_value' AS value,
    payload->>'characteristic_name' AS characteristic,
    payload->>'equipment_id' AS equipment,
    payload->>'operator_id' AS operator,
    created_at
FROM event_store
WHERE tenant_id = 'tenant-uuid'
  AND event_type = 'spc.measurement_collected'
  AND payload->>'product_id' = 'product-uuid'
ORDER BY created_at ASC;
```

### Dashboard query using projection (fast read path)

```sql
-- Current CAPA summary by status
SELECT status, priority, COUNT(*) as count,
       AVG(EXTRACT(EPOCH FROM (COALESCE(closed_at, now()) - opened_at)) / 86400)::INTEGER AS avg_days
FROM proj_capa
WHERE tenant_id = 'tenant-uuid'
GROUP BY status, priority
ORDER BY priority, status;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store Core | 3 | event_store, event_snapshot, event_subscription |
| Reference Data | 3 | tenant, app_user, facility |
| Projections — Inspection | 1 | proj_inspection |
| Projections — NCR | 1 | proj_non_conformance |
| Projections — CAPA | 1 | proj_capa |
| Projections — SPC | 2 | proj_spc_study, proj_spc_chart_data |
| Projections — Supplier | 1 | proj_supplier |
| **Total** | **12** | Plus projections added as needed |

---

## Key Design Decisions

1. **Single event_store table as source of truth** — All domain events across all aggregate types flow into one table. This simplifies backup, replication, and compliance archival. Partitioning by created_at handles scale.

2. **Stream-based event ordering with optimistic concurrency** — The UNIQUE constraint on (stream_id, event_version) prevents conflicting concurrent writes to the same aggregate. Application code retries on conflict.

3. **JSONB payloads with typed event_type** — Payloads are schema-flexible, but each event_type has a documented payload schema enforced at the application layer. This avoids rigid DDL changes when adding new event fields while maintaining structural consistency.

4. **Snapshots for performance** — Long-lived aggregates (a CAPA that runs for months accumulating dozens of events) snapshot periodically to avoid full replay on every read.

5. **Projections are disposable** — Every projection table (proj_*) can be dropped and rebuilt from the event store. This means read model schema changes are zero-risk: add the new projection, rebuild from events, drop the old one.

6. **Electronic signatures embedded in events** — Rather than a separate e-signature table, signatures are part of approval/sign events. Since events are immutable, signatures cannot be retroactively altered — directly satisfying 21 CFR Part 11 non-repudiation requirements.

7. **SPC measurements as events** — Each measurement is an individual event, creating a natural time-series. This enables arbitrary time-window SPC calculations, historical capability recalculation, and ML model training without ETL.

8. **Lot-level traceability via payload indexing** — GIN indexes on event_store.payload enable efficient cross-stream queries like "show me every quality event related to lot X" — a common regulatory inquiry.

9. **Event subscriptions for integration** — External systems (ERP, MES) subscribe to event types via the event_subscription table, receiving a reliable stream of quality events. This replaces point-to-point API polling with event-driven integration.

10. **Lower table count, higher conceptual complexity** — Only 12 tables vs. 37 in the normalised model, but developers must understand event replay, projection maintenance, and eventual consistency. The trade-off is operational simplicity (fewer migrations, schema changes) for conceptual complexity (event-driven thinking).
