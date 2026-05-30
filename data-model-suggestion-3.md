# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Quality Control & Inspection · Created: 2026-05-22

## Philosophy

This model uses a pragmatic hybrid approach: core entities with stable, well-known fields are modelled as typed relational columns with foreign keys and constraints, while variable, jurisdiction-specific, or rapidly-evolving fields are stored in JSONB columns. The result is a schema that is rigid where rigidity matters (referential integrity between inspections and products, SPC calculations on numeric columns, compliance-critical audit fields) and flexible where flexibility matters (inspection form responses, industry-specific PPAP variations, customer-specific quality requirements, AI-generated metadata).

This pattern is increasingly common in modern SaaS platforms that serve diverse customer segments. A medical device manufacturer under FDA QMSR needs different quality record fields than an automotive Tier 1 supplier under IATF 16949, which needs different fields than an electronics assembler inspecting against IPC-A-610. Rather than creating separate tables or adding dozens of nullable columns for each industry variant, the hybrid model places the common denominator in typed columns and the variable portion in validated JSONB.

PostgreSQL's JSONB implementation is particularly well-suited to this pattern: GIN indexes enable efficient containment queries, JSON Schema validation can enforce structure within JSONB columns at the application or database level, and JSONB fields participate fully in PostgreSQL's MVCC, WAL, and backup infrastructure. The hybrid approach delivers 80% of the query performance of a fully normalised model with 50% fewer tables, and dramatically faster time-to-market for new feature variants.

**Best for:** Multi-industry SaaS platforms serving diverse manufacturer types (automotive, aerospace, medical device, electronics) where inspection forms, compliance requirements, and customer-specific fields vary significantly across tenants but the core quality workflow remains consistent.

**Trade-offs:**
- (+) Dramatically fewer tables than a fully normalised model (22 vs. 37)
- (+) New inspection types, form fields, or compliance variants require no DDL changes
- (+) Faster MVP development: flexible fields can be added without migrations
- (+) Per-tenant and per-industry customisation without schema-per-tenant complexity
- (+) JSONB GIN indexes provide efficient querying within flexible fields
- (-) JSONB fields lack database-level type enforcement; validation shifts to application layer
- (-) Complex JSONB queries can be slower than equivalent normalised JOINs
- (-) Reporting tools (Power BI, Tableau) may struggle with nested JSONB structures
- (-) Referential integrity within JSONB is not enforceable at the database level
- (-) Risk of JSONB becoming a "junk drawer" without disciplined schema governance

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 9001:2015 | Core audit, CAPA, and document control workflows are relational; clause-specific evidence fields are JSONB |
| IATF 16949:2016 | PPAP element metadata, APQP phase gates, and customer-specific requirements stored in JSONB for automotive flexibility |
| AS9100D / AS9102B | FAI form data (Forms 1, 2, 3) stored as structured JSONB matching AS9102B field layout |
| ISO 13485 / FDA QMSR | Design history file entries and complaint fields in JSONB; core record linkage relational |
| ISO/IEC 17025 | Calibration certificate data and uncertainty budgets in JSONB with typed calibration record wrapper |
| ISO 2859-1 / ISO 3951 | Sampling plan parameters (AQL, inspection level, switching rules) in JSONB on inspection_plan |
| AIAG SPC Manual | SPC measurement values are typed NUMERIC columns; chart configuration and rule sets in JSONB |
| IPC-A-610 | Visual inspection acceptance criteria and class definitions stored in JSONB checklist items |
| OPC-UA (IEC 62541) | Equipment connection configuration (node IDs, namespaces, security) in JSONB |

---

## Core Platform Tables

### Tenant, Users & RBAC

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    industry        VARCHAR(50), -- 'automotive', 'aerospace', 'medical_device', 'electronics', 'general'
    compliance_frameworks TEXT[] DEFAULT '{}',
        -- e.g. {'ISO_9001', 'IATF_16949', 'AS9100D'}
    settings        JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "default_currency": "USD",
        --   "fiscal_year_start_month": 1,
        --   "spc_default_chart_type": "xbar_r",
        --   "ncr_auto_numbering_prefix": "NCR",
        --   "ppap_required_elements_by_level": {
        --     "1": [18],
        --     "2": [9, 11, 18],
        --     "3": [1,2,3,4,5,6,7,8,9,10,11,12,13,14,18],
        --     "4": "all",
        --     "5": "all_plus_samples"
        --   }
        -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    external_id     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    preferences     JSONB NOT NULL DEFAULT '{}',
        -- {"locale": "en-US", "timezone": "America/Chicago", "dashboard_layout": [...]}
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    permissions     JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {"resource": "inspection", "actions": ["create", "read", "update"]},
        --   {"resource": "capa", "actions": ["read"]},
        --   {"resource": "spc", "actions": ["read", "configure"]}
        -- ]
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    facility_id     UUID REFERENCES facility(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, facility_id)
);

CREATE TABLE facility (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    address         JSONB NOT NULL DEFAULT '{}',
        -- {"line1": "...", "line2": "...", "city": "...", "state": "...",
        --  "postal_code": "...", "country_code": "US"}
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_facility_tenant ON facility(tenant_id);
```

### Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
        -- {"field_name": {"old": "value1", "new": "value2"}, ...}
    context         JSONB DEFAULT '{}',
        -- {"ip": "...", "user_agent": "...", "source": "web|mobile|api"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
```

---

## Product & Part Definition

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    part_number     VARCHAR(100) NOT NULL,
    revision        VARCHAR(20) NOT NULL DEFAULT 'A',
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    product_type    VARCHAR(50) NOT NULL,
    uom             VARCHAR(20) NOT NULL DEFAULT 'EA',
    drawing_number  VARCHAR(100),
    customer_id     UUID REFERENCES supplier(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    specifications  JSONB NOT NULL DEFAULT '{}',
        -- Industry-specific product metadata:
        -- Automotive: {"material_spec": "ASTM A36", "heat_treat": "HRC 58-62",
        --              "surface_finish": "Ra 0.8", "customer_part_number": "C-1234"}
        -- Aerospace: {"cage_code": "1ABC2", "nsn": "5340-01-234-5678",
        --             "flight_safety": true, "fracture_critical": false}
        -- Medical:   {"device_class": "II", "biocompatibility": "ISO 10993",
        --             "sterilization": "EtO", "udi_di": "08717648200274"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, part_number, revision)
);

CREATE TABLE product_characteristic (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    balloon_number  INTEGER,
    name            VARCHAR(255) NOT NULL,
    characteristic_type VARCHAR(50) NOT NULL,
    data_type       VARCHAR(20) NOT NULL,
    nominal         NUMERIC(18,6),
    usl             NUMERIC(18,6),
    lsl             NUMERIC(18,6),
    uom             VARCHAR(20),
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    is_significant  BOOLEAN NOT NULL DEFAULT false,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    extra           JSONB NOT NULL DEFAULT '{}',
        -- {"gdt_symbol": "⌀", "datum_reference": "A|B",
        --  "inspection_method": "CMM", "gauge_id": "uuid",
        --  "ipc_class": 2, "acceptance_criteria": "No bridging per IPC-A-610 Class 2"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_char_product ON product_characteristic(product_id);
```

---

## Inspection Planning & Execution

```sql
CREATE TABLE inspection_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    plan_number     VARCHAR(100) NOT NULL,
    revision        VARCHAR(20) NOT NULL DEFAULT '1',
    name            VARCHAR(255) NOT NULL,
    plan_type       VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    sampling_config JSONB NOT NULL DEFAULT '{}',
        -- {"standard": "ISO_2859_1", "aql": 1.0, "inspection_level": "GII",
        --  "lot_size_range": "151-280", "sample_size": 32,
        --  "accept_number": 1, "reject_number": 2,
        --  "switching_rules": {"tightened_after": 2, "reduced_after": 5}}
    items           JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {
        --     "item_id": "uuid",
        --     "item_number": 1,
        --     "description": "Check overall length",
        --     "characteristic_id": "uuid",
        --     "inspection_method": "caliper",
        --     "equipment_type": "digital_caliper",
        --     "frequency": "every_part",
        --     "sample_size": 5,
        --     "acceptance_criteria": "25.400 ± 0.050 mm",
        --     "is_required": true
        --   }
        -- ]
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    effective_from  DATE,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, plan_number, revision)
);

CREATE INDEX idx_plan_tenant ON inspection_plan(tenant_id, status);
CREATE INDEX idx_plan_product ON inspection_plan(product_id);

CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    inspection_plan_id UUID NOT NULL REFERENCES inspection_plan(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    inspector_id    UUID NOT NULL REFERENCES app_user(id),
    inspection_number VARCHAR(100) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress',
    work_order      VARCHAR(100),
    lot_number      VARCHAR(100),
    serial_number   VARCHAR(100),
    quantity_inspected INTEGER,
    quantity_accepted  INTEGER,
    quantity_rejected  INTEGER,
    disposition     VARCHAR(50),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ,
    notes           TEXT,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
        -- Tenant/customer-specific fields:
        -- {"purchase_order": "PO-12345", "heat_number": "H-9876",
        --  "customer_job_number": "CJ-5555", "shift": "A",
        --  "temperature_f": 72, "humidity_pct": 45}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, inspection_number)
);

CREATE INDEX idx_inspection_tenant_status ON inspection(tenant_id, status);
CREATE INDEX idx_inspection_lot ON inspection(tenant_id, lot_number)
    WHERE lot_number IS NOT NULL;

CREATE TABLE inspection_result (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id           UUID NOT NULL REFERENCES inspection(id) ON DELETE CASCADE,
    plan_item_id            UUID NOT NULL,  -- references items[].item_id in inspection_plan.items JSONB
    characteristic_id       UUID REFERENCES product_characteristic(id),
    sample_number           INTEGER NOT NULL DEFAULT 1,
    measured_value          NUMERIC(18,6),
    attribute_result        VARCHAR(20),
    is_conforming           BOOLEAN NOT NULL,
    defect_code             VARCHAR(50),
    defect_severity         VARCHAR(20),
    equipment_id            UUID REFERENCES equipment(id),
    measured_by             UUID NOT NULL REFERENCES app_user(id),
    measured_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    extra                   JSONB NOT NULL DEFAULT '{}',
        -- {"photo_ids": ["uuid1"], "ai_classification": {"defect_type": "scratch",
        --  "confidence": 0.94, "model_version": "v2.3"},
        --  "cmm_program": "Widget_A_v3.prg", "temperature_compensation": true}
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_result_inspection ON inspection_result(inspection_id);
CREATE INDEX idx_result_conforming ON inspection_result(is_conforming)
    WHERE NOT is_conforming;
```

---

## Statistical Process Control (SPC)

```sql
CREATE TABLE spc_study (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenant(id),
    product_characteristic_id UUID NOT NULL REFERENCES product_characteristic(id),
    facility_id             UUID NOT NULL REFERENCES facility(id),
    chart_type              VARCHAR(20) NOT NULL,
    subgroup_size           INTEGER NOT NULL DEFAULT 1,
    -- Control limits as typed columns for fast SPC calculations
    ucl                     NUMERIC(18,6),
    lcl                     NUMERIC(18,6),
    center_line             NUMERIC(18,6),
    ucl_range               NUMERIC(18,6),
    lcl_range               NUMERIC(18,6),
    cl_range                NUMERIC(18,6),
    cp                      NUMERIC(8,4),
    cpk                     NUMERIC(8,4),
    pp                      NUMERIC(8,4),
    ppk                     NUMERIC(8,4),
    is_in_control           BOOLEAN NOT NULL DEFAULT true,
    last_calculated_at      TIMESTAMPTZ,
    config                  JSONB NOT NULL DEFAULT '{}',
        -- {"rules_enabled": ["nelson_1","nelson_2","nelson_5","weco_1"],
        --  "auto_recalculate": true, "recalc_after_n_subgroups": 25,
        --  "exclude_out_of_control": false,
        --  "alert_recipients": ["uuid1", "uuid2"],
        --  "alert_channels": ["email", "sms"]}
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spc_study_char ON spc_study(product_characteristic_id);

CREATE TABLE spc_data_point (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spc_study_id    UUID NOT NULL REFERENCES spc_study(id) ON DELETE CASCADE,
    subgroup_number INTEGER NOT NULL,
    sample_index    INTEGER NOT NULL,
    measured_value  NUMERIC(18,6) NOT NULL,
    inspection_result_id UUID REFERENCES inspection_result(id),
    equipment_id    UUID REFERENCES equipment(id),
    operator_id     UUID REFERENCES app_user(id),
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    extra           JSONB NOT NULL DEFAULT '{}',
        -- {"machine_id": "uuid", "tool_id": "T-42",
        --  "coolant_temperature": 22.5, "spindle_rpm": 8000}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spc_dp_study ON spc_data_point(spc_study_id, collected_at);
CREATE INDEX idx_spc_dp_subgroup ON spc_data_point(spc_study_id, subgroup_number);

CREATE TABLE spc_rule_violation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spc_study_id    UUID NOT NULL REFERENCES spc_study(id),
    subgroup_number INTEGER NOT NULL,
    rule_name       VARCHAR(50) NOT NULL,
    rule_description TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    acknowledged_by UUID REFERENCES app_user(id),
    acknowledged_at TIMESTAMPTZ,
    ncr_id          UUID REFERENCES non_conformance(id),
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spc_violation_study ON spc_rule_violation(spc_study_id, detected_at DESC);
```

---

## Non-Conformance & CAPA

```sql
CREATE TABLE non_conformance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    ncr_number      VARCHAR(100) NOT NULL,
    facility_id     UUID NOT NULL REFERENCES facility(id),
    product_id      UUID REFERENCES product(id),
    inspection_id   UUID REFERENCES inspection(id),
    supplier_id     UUID REFERENCES supplier(id),
    nc_type         VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    quantity_affected INTEGER,
    lot_number      VARCHAR(100),
    serial_numbers  TEXT[],
    disposition     VARCHAR(50),
    disposition_by  UUID REFERENCES app_user(id),
    disposition_at  TIMESTAMPTZ,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    reported_by     UUID NOT NULL REFERENCES app_user(id),
    assigned_to     UUID REFERENCES app_user(id),
    cost_of_quality NUMERIC(12,2),
    cost_currency   CHAR(3) DEFAULT 'USD',
    closed_at       TIMESTAMPTZ,
    details         JSONB NOT NULL DEFAULT '{}',
        -- Industry-specific NCR details:
        -- Automotive: {"customer_complaint_number": "CC-5678",
        --   "8d_team": ["uuid1","uuid2"], "containment_actions": [...],
        --   "sorted_quantity": 500, "sort_result": "3 additional defects found"}
        -- Aerospace: {"cage_code": "1ABC2", "contract_number": "F33657-99-C-0001",
        --   "dar_number": "DAR-2026-042", "mrbr_required": true}
        -- Medical: {"complaint_source": "field", "device_udi": "08717648200274",
        --   "mdr_reportable": true, "mdr_report_number": "MDR-2026-123"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ncr_number)
);

CREATE INDEX idx_ncr_tenant_status ON non_conformance(tenant_id, status);
CREATE INDEX idx_ncr_product ON non_conformance(product_id);
CREATE INDEX idx_ncr_supplier ON non_conformance(supplier_id)
    WHERE supplier_id IS NOT NULL;
-- GIN index for searching within JSONB details
CREATE INDEX idx_ncr_details ON non_conformance USING gin(details);

CREATE TABLE capa (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    capa_number     VARCHAR(100) NOT NULL,
    capa_type       VARCHAR(20) NOT NULL,
    methodology     VARCHAR(20) NOT NULL DEFAULT '8d',
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    owner_id        UUID NOT NULL REFERENCES app_user(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    linked_ncr_ids  UUID[] DEFAULT '{}',
    root_cause      TEXT,
    root_cause_method VARCHAR(50),
    due_date        DATE,
    effectiveness_check_due DATE,
    effectiveness_verified BOOLEAN,
    closed_at       TIMESTAMPTZ,
    methodology_data JSONB NOT NULL DEFAULT '{}',
        -- 8D methodology fields:
        -- {"d1_team": ["uuid1","uuid2"],
        --  "d2_problem_statement": "...",
        --  "d3_containment": "...",
        --  "d4_root_cause_analysis": {"method": "fishbone", "factors": [...]},
        --  "d5_corrective_actions": [...],
        --  "d6_implementation": {"verified_by": "uuid", "verified_at": "..."},
        --  "d7_preventive_actions": [...],
        --  "d8_team_recognition": "..."}
        -- A3 methodology fields:
        -- {"background": "...", "current_condition": "...",
        --  "goal": "...", "root_cause_analysis": "...",
        --  "countermeasures": [...], "implementation_plan": [...],
        --  "follow_up": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, capa_number)
);

CREATE INDEX idx_capa_tenant_status ON capa(tenant_id, status);
CREATE INDEX idx_capa_owner ON capa(owner_id);

CREATE TABLE capa_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capa_id         UUID NOT NULL REFERENCES capa(id) ON DELETE CASCADE,
    action_type     VARCHAR(30) NOT NULL,
    description     TEXT NOT NULL,
    assigned_to     UUID NOT NULL REFERENCES app_user(id),
    due_date        DATE NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    completed_at    TIMESTAMPTZ,
    evidence        JSONB NOT NULL DEFAULT '{}',
        -- {"notes": "...", "attachment_ids": ["uuid1"],
        --  "verification_method": "re-inspection",
        --  "verified_by": "uuid", "verified_at": "2026-04-15T10:30:00Z"}
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_capa_action_capa ON capa_action(capa_id);
CREATE INDEX idx_capa_action_assignee ON capa_action(assigned_to, status);
```

---

## Supplier Quality & PPAP

```sql
CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    supplier_code   VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    duns_number     VARCHAR(9),
    risk_tier       VARCHAR(20) NOT NULL DEFAULT 'standard',
    quality_rating  NUMERIC(5,2),
    is_approved     BOOLEAN NOT NULL DEFAULT false,
    approved_at     TIMESTAMPTZ,
    certification_scope TEXT[],
    is_active       BOOLEAN NOT NULL DEFAULT true,
    contact         JSONB NOT NULL DEFAULT '{}',
        -- {"primary_name": "...", "email": "...", "phone": "...",
        --  "quality_contact_name": "...", "quality_contact_email": "..."}
    address         JSONB NOT NULL DEFAULT '{}',
        -- {"line1": "...", "city": "...", "state": "...",
        --  "postal_code": "...", "country_code": "CN"}
    qualification   JSONB NOT NULL DEFAULT '{}',
        -- {"iso_9001_cert_number": "QMS-12345", "iso_9001_expiry": "2027-06-30",
        --  "iatf_16949_cert_number": "IATF-67890", "nadcap_accreditations": ["HT", "NDT"],
        --  "approved_processes": ["machining", "grinding", "heat_treat"],
        --  "last_audit_date": "2026-01-15", "next_audit_due": "2027-01-15"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, supplier_code)
);

CREATE INDEX idx_supplier_tenant ON supplier(tenant_id);

CREATE TABLE ppap_submission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    submission_number VARCHAR(100) NOT NULL,
    ppap_level      INTEGER NOT NULL CHECK (ppap_level BETWEEN 1 AND 5),
    status          VARCHAR(30) NOT NULL DEFAULT 'requested',
    requested_by    UUID NOT NULL REFERENCES app_user(id),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ,
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ,
    disposition     VARCHAR(30),
    psw_signed      BOOLEAN NOT NULL DEFAULT false,
    elements        JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {"element_number": 1, "element_name": "Design Records",
        --    "is_required": true, "status": "approved",
        --    "attachment_id": "uuid", "reviewed_by": "uuid",
        --    "reviewed_at": "2026-03-20T14:30:00Z", "notes": "Rev C drawing"},
        --   {"element_number": 9, "element_name": "Dimensional Results",
        --    "is_required": true, "status": "submitted",
        --    "attachment_id": "uuid"}
        -- ]
    fai_data        JSONB NOT NULL DEFAULT '{}',
        -- AS9102B FAI report data when applicable:
        -- {"report_number": "FAI-2026-0042", "fai_type": "full",
        --  "standard": "AS9102B",
        --  "form1": {"part_number": "...", "part_name": "...", ...},
        --  "form3_results": [
        --    {"char_number": 1, "requirement": "25.400 ± 0.050",
        --     "result": "25.412", "is_conforming": true}
        --  ]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, submission_number)
);

CREATE INDEX idx_ppap_supplier ON ppap_submission(supplier_id);
CREATE INDEX idx_ppap_product ON ppap_submission(product_id);

CREATE TABLE supplier_scorecard (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    scores          JSONB NOT NULL,
        -- {"quality": 92.5, "delivery": 88.0, "responsiveness": 95.0,
        --  "ppap": 100.0, "overall": 93.9,
        --  "metrics": {
        --    "total_lots_received": 45, "lots_rejected": 3,
        --    "total_ncrs": 4, "ppm_defective": 1250,
        --    "on_time_delivery_pct": 88.0,
        --    "avg_capa_response_days": 3.2
        --  }}
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scorecard_supplier ON supplier_scorecard(supplier_id, period_start DESC);
```

---

## Audit Management

```sql
CREATE TABLE audit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    audit_number    VARCHAR(100) NOT NULL,
    audit_type      VARCHAR(50) NOT NULL,
    standard        VARCHAR(50),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    supplier_id     UUID REFERENCES supplier(id),
    title           VARCHAR(500) NOT NULL,
    scope           TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'planned',
    lead_auditor_id UUID NOT NULL REFERENCES app_user(id),
    scheduled_start DATE,
    scheduled_end   DATE,
    actual_start    DATE,
    actual_end      DATE,
    summary         TEXT,
    team_and_findings JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "team": [
        --     {"user_id": "uuid", "role": "lead"},
        --     {"user_id": "uuid", "role": "auditor"}
        --   ],
        --   "findings": [
        --     {
        --       "finding_id": "uuid",
        --       "finding_number": 1,
        --       "finding_type": "minor_nc",
        --       "clause_reference": "8.5.1",
        --       "description": "Work instructions not current at workstation",
        --       "evidence": "WI-042 Rev A posted, Rev C is current",
        --       "capa_id": "uuid",
        --       "status": "closed",
        --       "due_date": "2026-04-30",
        --       "closed_at": "2026-04-22T16:00:00Z"
        --     }
        --   ]
        -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, audit_number)
);

CREATE INDEX idx_audit_tenant_status ON audit(tenant_id, status);
```

---

## Document Control & Equipment

```sql
CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    document_number VARCHAR(100) NOT NULL,
    revision        VARCHAR(20) NOT NULL DEFAULT '1',
    title           VARCHAR(500) NOT NULL,
    doc_type        VARCHAR(50) NOT NULL,
    category        VARCHAR(100),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    owner_id        UUID NOT NULL REFERENCES app_user(id),
    effective_date  DATE,
    review_due_date DATE,
    versions        JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {"version": 1, "file_path": "/docs/...", "file_size": 245000,
        --    "mime_type": "application/pdf", "change_summary": "Initial release",
        --    "uploaded_by": "uuid", "uploaded_at": "2026-01-15T10:00:00Z"}
        -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, document_number, revision)
);

CREATE INDEX idx_document_tenant_status ON document(tenant_id, status);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    equipment_number VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    equipment_type  VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    next_calibration_due DATE,
    calibration_interval_days INTEGER,
    specs           JSONB NOT NULL DEFAULT '{}',
        -- {"manufacturer": "Mitutoyo", "model": "500-196-30",
        --  "serial_number": "SN-12345", "accuracy": 0.01,
        --  "resolution": 0.001, "range_min": 0, "range_max": 150,
        --  "uom": "mm", "opcua_node_id": "ns=2;s=CMM.CH1"}
    calibration_history JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {"date": "2026-01-10", "next_due": "2026-07-10",
        --    "performed_by": "uuid", "result": "pass",
        --    "certificate_number": "CAL-2026-0042",
        --    "as_found": {"reading": 25.001, "std": 25.000, "error": 0.001},
        --    "as_left": {"reading": 25.000, "std": 25.000, "error": 0.000},
        --    "attachment_id": "uuid"}
        -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, equipment_number)
);

CREATE INDEX idx_equipment_cal_due ON equipment(next_calibration_due)
    WHERE status != 'retired';
```

---

## Attachments & Electronic Signatures

```sql
CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    file_name       VARCHAR(500) NOT NULL,
    file_path       VARCHAR(1000) NOT NULL,
    file_size       BIGINT NOT NULL,
    mime_type       VARCHAR(100) NOT NULL,
    uploaded_by     UUID NOT NULL REFERENCES app_user(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- {"ai_classification": {"defect_type": "scratch", "confidence": 0.94},
        --  "exif": {"camera": "iPhone 15", "timestamp": "..."}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_entity ON attachment(entity_type, entity_id);

CREATE TABLE electronic_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    signer_id       UUID NOT NULL REFERENCES app_user(id),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    signature_meaning VARCHAR(100) NOT NULL,
    signer_name     VARCHAR(255) NOT NULL,
    signer_title    VARCHAR(255),
    signature_hash  VARCHAR(512) NOT NULL,
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_valid        BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_esig_entity ON electronic_signature(entity_type, entity_id);
```

---

## Example Queries

### Query inspection results with JSONB custom fields

```sql
-- Find all inspections for a specific customer job number
SELECT i.inspection_number, i.status, i.lot_number,
       i.custom_fields->>'customer_job_number' AS job_number,
       i.custom_fields->>'heat_number' AS heat_number
FROM inspection i
WHERE i.tenant_id = 'tenant-uuid'
  AND i.custom_fields @> '{"customer_job_number": "CJ-5555"}';
```

### Query supplier qualifications

```sql
-- Find suppliers with Nadcap heat treat accreditation expiring in 90 days
SELECT s.name, s.supplier_code,
       s.qualification->>'nadcap_accreditations' AS accreditations,
       s.qualification->>'next_audit_due' AS next_audit
FROM supplier s
WHERE s.tenant_id = 'tenant-uuid'
  AND s.qualification @> '{"nadcap_accreditations": ["HT"]}'
  AND (s.qualification->>'next_audit_due')::DATE <= CURRENT_DATE + INTERVAL '90 days';
```

### Query PPAP element status

```sql
-- Get PPAP submissions with any rejected elements
SELECT p.submission_number, p.status, s.name AS supplier_name,
       jsonb_array_elements(p.elements) ->> 'element_name' AS element,
       jsonb_array_elements(p.elements) ->> 'status' AS element_status
FROM ppap_submission p
JOIN supplier s ON p.supplier_id = s.id
WHERE p.tenant_id = 'tenant-uuid'
  AND p.elements @> '[{"status": "rejected"}]';
```

### Query SPC with process conditions from JSONB

```sql
-- Correlate SPC violations with process conditions
SELECT rv.rule_name, rv.detected_at,
       dp.measured_value,
       dp.extra->>'spindle_rpm' AS spindle_rpm,
       dp.extra->>'coolant_temperature' AS coolant_temp,
       dp.extra->>'tool_id' AS tool_id
FROM spc_rule_violation rv
JOIN spc_data_point dp ON dp.spc_study_id = rv.spc_study_id
    AND dp.subgroup_number = rv.subgroup_number
WHERE rv.spc_study_id = 'study-uuid'
ORDER BY rv.detected_at DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Platform (tenant, users, RBAC) | 5 | tenant, app_user, role, user_role, facility |
| Audit Trail & Signatures | 2 | audit_log, electronic_signature |
| Product Definition | 2 | product (with specs JSONB), product_characteristic (with extra JSONB) |
| Inspection | 3 | inspection_plan (items in JSONB), inspection, inspection_result |
| SPC | 3 | spc_study, spc_data_point, spc_rule_violation |
| Non-Conformance & CAPA | 3 | non_conformance, capa (methodology_data JSONB), capa_action |
| Supplier Quality | 3 | supplier, ppap_submission (elements JSONB), supplier_scorecard |
| Audit Management | 1 | audit (team_and_findings JSONB) |
| Document Control & Equipment | 2 | document (versions JSONB), equipment (calibration_history JSONB) |
| Attachments | 1 | attachment |
| **Total** | **25** | vs. 37 in normalised model |

---

## Key Design Decisions

1. **JSONB for variable fields, typed columns for queryable/calculable fields** — SPC measurements, inspection dispositions, and compliance-critical statuses are typed columns. Industry-specific metadata, form configurations, and customer-specific fields are JSONB. The rule: if you GROUP BY it, ORDER BY it, or calculate with it, it is a column; if you display it or filter it occasionally, it is JSONB.

2. **Inspection plan items as JSONB array** — Eliminates the inspection_plan_item junction table. Each plan's items are a self-contained JSONB array. This matches the UX pattern (a form builder editing items as a list) and reduces JOINs when loading a complete plan. Trade-off: individual item updates require rewriting the array.

3. **PPAP elements as JSONB array** — The 18 PPAP elements and their per-submission status are stored as a JSONB array rather than 18 rows in a junction table. This simplifies PPAP display and reduces table count. The element structure is well-defined and unlikely to change (AIAG PPAP has been stable for decades).

4. **Audit findings embedded in audit record** — For most organisations, an audit has 5-15 findings. Embedding them as JSONB in the audit table eliminates a junction table and simplifies audit report generation (one query returns the complete audit with all findings).

5. **Calibration history as JSONB array on equipment** — Each equipment record carries its own calibration history as an embedded array. This avoids a separate calibration_record table and makes equipment history self-contained. For equipment with very long histories (hundreds of calibrations), the array is manageable at typical calibration intervals (semi-annual = 20 records over 10 years).

6. **GIN indexes on key JSONB columns** — The non_conformance.details column has a GIN index to support containment queries for industry-specific fields. This ensures that queries like "find all NCRs where MDR reporting was required" perform well even at scale.

7. **Methodology-specific CAPA data in JSONB** — An 8D CAPA has different fields than an A3 or PDCA CAPA. Rather than nullable columns for each methodology, the methodology_data JSONB column holds the methodology-specific fields. The capa.methodology column tells the application which schema to apply.

8. **Supplier qualification data in JSONB** — Different suppliers hold different certifications, and qualification requirements vary by industry. JSONB avoids a supplier_certification junction table and allows per-tenant customisation of tracked qualifications.

9. **Document versions as JSONB array** — Eliminates the document_version table. Version history is an embedded array on the document record. This works well for the typical document lifecycle (5-20 versions over its life) and simplifies version comparison queries.

10. **25 tables vs. 37 in normalised model** — The hybrid approach eliminates 12 tables (inspection_plan_item, ppap_element, fai_report, fai_characteristic_result, audit_team_member, audit_finding, calibration_record, document_version, permission, role_permission, capa_non_conformance, supplier_scorecard metrics) by embedding their data in JSONB columns on parent tables. Each elimination trades referential integrity for reduced schema complexity.
