# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Quality Control & Inspection · Created: 2026-05-22

## Philosophy

This model follows a fully normalized relational approach where every domain concept maps to its own table with rigorous foreign key constraints, check constraints, and referential integrity. The design mirrors the structure of the domain itself: inspection plans contain inspection items, which produce measurement results, which feed SPC calculations, which may trigger non-conformances, which spawn CAPAs. Every relationship is explicit, every identifier is typed, and every reference is enforceable at the database level.

Real-world systems that use this pattern include enterprise QMS platforms like MasterControl and ETQ Reliance, as well as ERP-native quality modules like SAP QM and Plex QMS. The approach is also aligned with the QIF (Quality Information Framework, ISO 23952:2020) data model which defines separate, explicitly-linked entities for measurement plans, measurement results, measurement resources, and statistical analysis.

This model is best suited for regulated manufacturing environments (medical device under ISO 13485/FDA QMSR, aerospace under AS9100D, automotive under IATF 16949) where every data relationship must be auditable, every record must have a clear provenance chain, and schema changes go through formal change control. It prioritises data integrity and query flexibility over write throughput or schema agility.

**Best for:** Regulated manufacturers needing auditable, standards-aligned data with complex cross-entity queries and compliance reporting.

**Trade-offs:**
- (+) Maximum data integrity through foreign keys and constraints
- (+) Complex cross-entity queries are straightforward with JOINs
- (+) Schema self-documents the domain; new developers understand relationships immediately
- (+) Aligns naturally with QIF, OSLC QM, and AIAG Core Tools data structures
- (-) High table count increases migration complexity
- (-) Schema changes require formal migrations; adding a new inspection type or compliance field means DDL
- (-) Write-heavy SPC data collection may encounter contention on heavily-indexed tables
- (-) Junction tables add complexity to simple CRUD operations

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 23952:2020 (QIF) | Entity separation mirrors QIF's six application schemas: Plans, Results, Resources, Rules, Statistics, Product |
| ISO 9001:2015 | Audit, CAPA, document control, and management review tables map to ISO 9001 clause structure |
| IATF 16949:2016 | APQP, PPAP, FMEA, MSA, and SPC tables implement AIAG Core Tools data requirements |
| AS9100D / AS9102B | FAI report tables implement AS9102B form structure (Forms 1, 2, 3) |
| ISO 13485:2016 / FDA QMSR | Design history, complaint handling, and electronic signature tables support medical device compliance |
| ISO/IEC 17025:2017 | Calibration and measurement resource tables align with laboratory competence requirements |
| ISO 2859-1 / ISO 3951 | Sampling plan tables encode AQL-based attribute and variable sampling schemes |
| OASIS OSLC QM 2.1 | Test plan / test case / test result entity pattern informs inspection plan / item / result structure |
| OPC-UA (IEC 62541) | Gauge and equipment tables include OPC-UA node identifiers for automated data collection |
| 21 CFR Part 11 | Electronic signature and audit trail tables enforce FDA e-signature requirements |

---

## Core Platform Tables

### Tenant & Organisation

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

CREATE TABLE facility (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL, -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_facility_tenant ON facility(tenant_id);
```

### Users & RBAC

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local', -- local, saml, oidc
    external_id     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
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
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        VARCHAR(100) NOT NULL, -- e.g. 'inspection', 'capa', 'spc'
    action          VARCHAR(50) NOT NULL,  -- e.g. 'create', 'read', 'approve'
    description     TEXT,
    UNIQUE (resource, action)
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    facility_id     UUID REFERENCES facility(id), -- NULL = all facilities
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, facility_id)
);

CREATE INDEX idx_user_role_user ON user_role(user_id);
```

### Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(50) NOT NULL,  -- CREATE, UPDATE, DELETE, APPROVE, SIGN
    entity_type     VARCHAR(100) NOT NULL, -- e.g. 'inspection', 'capa', 'ncr'
    entity_id       UUID NOT NULL,
    field_changes   JSONB,  -- {"field": {"old": ..., "new": ...}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, created_at DESC);
```

### Electronic Signatures (21 CFR Part 11)

```sql
CREATE TABLE electronic_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    signer_id       UUID NOT NULL REFERENCES app_user(id),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    signature_meaning VARCHAR(100) NOT NULL, -- 'approved', 'reviewed', 'authored'
    signer_name     VARCHAR(255) NOT NULL,
    signer_title    VARCHAR(255),
    signature_hash  VARCHAR(512) NOT NULL,
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_valid        BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_esig_entity ON electronic_signature(entity_type, entity_id);
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
    product_type    VARCHAR(50) NOT NULL, -- 'component', 'assembly', 'raw_material'
    uom             VARCHAR(20) NOT NULL DEFAULT 'EA', -- unit of measure
    drawing_number  VARCHAR(100),
    material_spec   VARCHAR(255),
    customer_id     UUID REFERENCES supplier(id), -- reuse supplier table for customers
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, part_number, revision)
);

CREATE TABLE product_characteristic (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    balloon_number  INTEGER,
    name            VARCHAR(255) NOT NULL,
    characteristic_type VARCHAR(50) NOT NULL, -- 'dimensional', 'visual', 'material', 'functional'
    data_type       VARCHAR(20) NOT NULL, -- 'variable', 'attribute'
    nominal         NUMERIC(18,6),
    usl             NUMERIC(18,6), -- upper specification limit
    lsl             NUMERIC(18,6), -- lower specification limit
    uom             VARCHAR(20),
    gdt_symbol      VARCHAR(50),   -- GD&T symbol if applicable
    is_critical     BOOLEAN NOT NULL DEFAULT false, -- safety/critical characteristic
    is_significant  BOOLEAN NOT NULL DEFAULT false, -- significant characteristic
    sort_order      INTEGER NOT NULL DEFAULT 0,
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
    plan_type       VARCHAR(50) NOT NULL, -- 'receiving', 'in_process', 'final', 'fai', 'audit'
    status          VARCHAR(30) NOT NULL DEFAULT 'draft', -- draft, active, superseded, retired
    sampling_standard VARCHAR(50), -- 'ISO_2859_1', 'ISO_3951', 'ANSI_Z1_4', '100_percent'
    aql             NUMERIC(5,2),  -- acceptable quality level
    inspection_level VARCHAR(10),  -- GI, GII, GIII, S1-S4
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    effective_from  DATE,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, plan_number, revision)
);

CREATE TABLE inspection_plan_item (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_plan_id      UUID NOT NULL REFERENCES inspection_plan(id) ON DELETE CASCADE,
    product_characteristic_id UUID REFERENCES product_characteristic(id),
    item_number             INTEGER NOT NULL,
    description             TEXT NOT NULL,
    inspection_method       VARCHAR(100), -- 'caliper', 'cmm', 'visual', 'gauge', 'go_no_go'
    equipment_type          VARCHAR(100),
    frequency               VARCHAR(100), -- 'every_part', 'first_5', 'hourly', 'per_lot'
    sample_size             INTEGER,
    acceptance_criteria     TEXT,
    is_required             BOOLEAN NOT NULL DEFAULT true,
    sort_order              INTEGER NOT NULL DEFAULT 0,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plan_item_plan ON inspection_plan_item(inspection_plan_id);

CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    inspection_plan_id UUID NOT NULL REFERENCES inspection_plan(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    inspector_id    UUID NOT NULL REFERENCES app_user(id),
    inspection_number VARCHAR(100) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress',
        -- in_progress, completed, accepted, rejected, on_hold
    work_order      VARCHAR(100),
    lot_number      VARCHAR(100),
    serial_number   VARCHAR(100),
    quantity_inspected INTEGER,
    quantity_accepted  INTEGER,
    quantity_rejected  INTEGER,
    disposition     VARCHAR(50), -- 'accept', 'reject', 'rework', 'use_as_is', 'scrap'
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, inspection_number)
);

CREATE INDEX idx_inspection_tenant_status ON inspection(tenant_id, status);
CREATE INDEX idx_inspection_plan ON inspection(inspection_plan_id);
CREATE INDEX idx_inspection_lot ON inspection(tenant_id, lot_number) WHERE lot_number IS NOT NULL;

CREATE TABLE inspection_result (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id           UUID NOT NULL REFERENCES inspection(id) ON DELETE CASCADE,
    inspection_plan_item_id UUID NOT NULL REFERENCES inspection_plan_item(id),
    sample_number           INTEGER NOT NULL DEFAULT 1,
    measured_value          NUMERIC(18,6),
    attribute_result        VARCHAR(20), -- 'pass', 'fail', 'na' for attribute data
    is_conforming           BOOLEAN NOT NULL,
    defect_code             VARCHAR(50),
    defect_severity         VARCHAR(20), -- 'critical', 'major', 'minor'
    equipment_id            UUID REFERENCES equipment(id),
    photo_attachment_id     UUID REFERENCES attachment(id),
    notes                   TEXT,
    measured_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    measured_by             UUID NOT NULL REFERENCES app_user(id),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_result_inspection ON inspection_result(inspection_id);
CREATE INDEX idx_result_item ON inspection_result(inspection_plan_item_id);
CREATE INDEX idx_result_conforming ON inspection_result(is_conforming) WHERE NOT is_conforming;
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
        -- 'i_mr', 'xbar_r', 'xbar_s', 'p', 'np', 'c', 'u'
    subgroup_size           INTEGER NOT NULL DEFAULT 1,
    ucl                     NUMERIC(18,6), -- upper control limit
    lcl                     NUMERIC(18,6), -- lower control limit
    center_line             NUMERIC(18,6),
    ucl_range               NUMERIC(18,6), -- for range chart
    lcl_range               NUMERIC(18,6),
    cl_range                NUMERIC(18,6),
    cp                      NUMERIC(8,4),
    cpk                     NUMERIC(8,4),
    pp                      NUMERIC(8,4),
    ppk                     NUMERIC(8,4),
    is_in_control           BOOLEAN NOT NULL DEFAULT true,
    last_calculated_at      TIMESTAMPTZ,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spc_study_char ON spc_study(product_characteristic_id);

CREATE TABLE spc_subgroup (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spc_study_id    UUID NOT NULL REFERENCES spc_study(id) ON DELETE CASCADE,
    subgroup_number INTEGER NOT NULL,
    mean_value      NUMERIC(18,6),
    range_value     NUMERIC(18,6),
    std_dev         NUMERIC(18,6),
    sample_size     INTEGER NOT NULL,
    is_excluded     BOOLEAN NOT NULL DEFAULT false,
    collected_at    TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spc_subgroup_study ON spc_subgroup(spc_study_id, collected_at);

CREATE TABLE spc_data_point (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spc_subgroup_id UUID NOT NULL REFERENCES spc_subgroup(id) ON DELETE CASCADE,
    sample_index    INTEGER NOT NULL, -- position within subgroup
    measured_value  NUMERIC(18,6) NOT NULL,
    inspection_result_id UUID REFERENCES inspection_result(id),
    equipment_id    UUID REFERENCES equipment(id),
    operator_id     UUID REFERENCES app_user(id),
    machine_id      UUID REFERENCES equipment(id),
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spc_dp_subgroup ON spc_data_point(spc_subgroup_id);

CREATE TABLE spc_rule_violation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spc_study_id    UUID NOT NULL REFERENCES spc_study(id),
    spc_subgroup_id UUID NOT NULL REFERENCES spc_subgroup(id),
    rule_name       VARCHAR(50) NOT NULL,
        -- 'nelson_1' through 'nelson_8', 'weco_1' through 'weco_4'
    rule_description TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning', -- warning, alarm
    acknowledged_by UUID REFERENCES app_user(id),
    acknowledged_at TIMESTAMPTZ,
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
        -- 'incoming', 'in_process', 'final', 'customer_complaint', 'audit_finding'
    severity        VARCHAR(20) NOT NULL, -- 'critical', 'major', 'minor'
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    quantity_affected INTEGER,
    lot_number      VARCHAR(100),
    serial_numbers  TEXT[],
    disposition     VARCHAR(50), -- 'rework', 'use_as_is', 'scrap', 'return_to_supplier', 'pending'
    disposition_by  UUID REFERENCES app_user(id),
    disposition_at  TIMESTAMPTZ,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
        -- open, under_review, dispositioned, closed
    reported_by     UUID NOT NULL REFERENCES app_user(id),
    assigned_to     UUID REFERENCES app_user(id),
    cost_of_quality NUMERIC(12,2),
    cost_currency   CHAR(3) DEFAULT 'USD', -- ISO 4217
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ncr_number)
);

CREATE INDEX idx_ncr_tenant_status ON non_conformance(tenant_id, status);
CREATE INDEX idx_ncr_product ON non_conformance(product_id);
CREATE INDEX idx_ncr_supplier ON non_conformance(supplier_id) WHERE supplier_id IS NOT NULL;

CREATE TABLE capa (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    capa_number     VARCHAR(100) NOT NULL,
    capa_type       VARCHAR(20) NOT NULL, -- 'corrective', 'preventive'
    methodology     VARCHAR(20) NOT NULL DEFAULT '8d', -- '8d', 'a3', 'pdca', 'dmaic'
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium', -- low, medium, high, critical
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
        -- open, containment, root_cause, action_plan, implementation,
        -- verification, closed, cancelled
    owner_id        UUID NOT NULL REFERENCES app_user(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    root_cause      TEXT,
    root_cause_method VARCHAR(50), -- 'five_why', 'fishbone', 'fault_tree', 'pareto'
    containment_action TEXT,
    corrective_action  TEXT,
    preventive_action  TEXT,
    verification_method TEXT,
    verification_result TEXT,
    effectiveness_check_due DATE,
    effectiveness_verified BOOLEAN,
    due_date        DATE,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, capa_number)
);

CREATE INDEX idx_capa_tenant_status ON capa(tenant_id, status);
CREATE INDEX idx_capa_owner ON capa(owner_id);

CREATE TABLE capa_non_conformance (
    capa_id         UUID NOT NULL REFERENCES capa(id) ON DELETE CASCADE,
    non_conformance_id UUID NOT NULL REFERENCES non_conformance(id),
    PRIMARY KEY (capa_id, non_conformance_id)
);

CREATE TABLE capa_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capa_id         UUID NOT NULL REFERENCES capa(id) ON DELETE CASCADE,
    action_type     VARCHAR(30) NOT NULL, -- 'containment', 'corrective', 'preventive', 'verification'
    description     TEXT NOT NULL,
    assigned_to     UUID NOT NULL REFERENCES app_user(id),
    due_date        DATE NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open', -- open, in_progress, completed, overdue
    completed_at    TIMESTAMPTZ,
    evidence_notes  TEXT,
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
    duns_number     VARCHAR(9),  -- D-U-N-S identifier
    contact_name    VARCHAR(255),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2), -- ISO 3166-1
    risk_tier       VARCHAR(20) NOT NULL DEFAULT 'standard', -- low, standard, high, critical
    quality_rating  NUMERIC(5,2), -- 0-100 score
    is_approved     BOOLEAN NOT NULL DEFAULT false,
    approved_at     TIMESTAMPTZ,
    certification_scope TEXT[], -- e.g. {'ISO_9001', 'IATF_16949', 'AS9100D'}
    is_active       BOOLEAN NOT NULL DEFAULT true,
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
        -- requested, in_progress, submitted, under_review, approved,
        -- approved_interim, rejected
    requested_by    UUID NOT NULL REFERENCES app_user(id),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ,
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ,
    disposition     VARCHAR(30), -- 'approved', 'interim_approval', 'rejected'
    disposition_notes TEXT,
    psw_signed      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, submission_number)
);

CREATE INDEX idx_ppap_supplier ON ppap_submission(supplier_id);
CREATE INDEX idx_ppap_product ON ppap_submission(product_id);

CREATE TABLE ppap_element (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ppap_submission_id UUID NOT NULL REFERENCES ppap_submission(id) ON DELETE CASCADE,
    element_number  INTEGER NOT NULL CHECK (element_number BETWEEN 1 AND 18),
    element_name    VARCHAR(255) NOT NULL,
        -- 1: Design Records, 2: Engineering Change Documents, 3: Customer Approval,
        -- 4: Design FMEA, 5: Process Flow Diagram, 6: Process FMEA,
        -- 7: Control Plan, 8: MSA Studies, 9: Dimensional Results,
        -- 10: Material/Performance Test Results, 11: Initial Process Studies (SPC),
        -- 12: Qualified Lab Documentation, 13: Appearance Approval Report,
        -- 14: Sample Production Parts, 15: Master Sample, 16: Checking Aids,
        -- 17: Customer-Specific Requirements, 18: Part Submission Warrant
    is_required     BOOLEAN NOT NULL DEFAULT true,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending, uploaded, approved, rejected, not_applicable
    attachment_id   UUID REFERENCES attachment(id),
    notes           TEXT,
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ppap_element_sub ON ppap_element(ppap_submission_id);

CREATE TABLE fai_report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    supplier_id     UUID REFERENCES supplier(id),
    ppap_submission_id UUID REFERENCES ppap_submission(id),
    report_number   VARCHAR(100) NOT NULL,
    fai_type        VARCHAR(20) NOT NULL, -- 'full', 'partial', 'delta'
    standard        VARCHAR(30) NOT NULL DEFAULT 'AS9102B',
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, submitted, approved, rejected
    form1_data      JSONB, -- Part Number Accountability (AS9102 Form 1)
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, report_number)
);

CREATE TABLE fai_characteristic_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fai_report_id   UUID NOT NULL REFERENCES fai_report(id) ON DELETE CASCADE,
    product_characteristic_id UUID NOT NULL REFERENCES product_characteristic(id),
    char_number     INTEGER NOT NULL, -- AS9102 Form 3 characteristic number
    requirement     TEXT NOT NULL,
    result          TEXT NOT NULL,
    is_conforming   BOOLEAN NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fai_char_report ON fai_characteristic_result(fai_report_id);

CREATE TABLE supplier_scorecard (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    quality_score   NUMERIC(5,2), -- based on incoming inspection defect rate
    delivery_score  NUMERIC(5,2),
    responsiveness_score NUMERIC(5,2), -- CAPA response time
    ppap_score      NUMERIC(5,2),
    overall_score   NUMERIC(5,2),
    total_lots_received INTEGER,
    total_lots_rejected INTEGER,
    total_ncrs      INTEGER,
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
        -- 'internal', 'external', 'supplier', 'process', 'product', 'system'
    standard        VARCHAR(50), -- ISO_9001, IATF_16949, AS9100D, ISO_13485
    facility_id     UUID NOT NULL REFERENCES facility(id),
    supplier_id     UUID REFERENCES supplier(id), -- for supplier audits
    title           VARCHAR(500) NOT NULL,
    scope           TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'planned',
        -- planned, in_progress, completed, closed, cancelled
    lead_auditor_id UUID NOT NULL REFERENCES app_user(id),
    scheduled_start DATE,
    scheduled_end   DATE,
    actual_start    DATE,
    actual_end      DATE,
    summary         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, audit_number)
);

CREATE INDEX idx_audit_tenant_status ON audit(tenant_id, status);

CREATE TABLE audit_team_member (
    audit_id        UUID NOT NULL REFERENCES audit(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role            VARCHAR(50) NOT NULL, -- 'lead', 'auditor', 'observer', 'auditee'
    PRIMARY KEY (audit_id, user_id)
);

CREATE TABLE audit_finding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id        UUID NOT NULL REFERENCES audit(id) ON DELETE CASCADE,
    finding_number  INTEGER NOT NULL,
    finding_type    VARCHAR(30) NOT NULL,
        -- 'major_nc', 'minor_nc', 'observation', 'opportunity', 'positive'
    clause_reference VARCHAR(50), -- e.g. '8.5.1' for ISO 9001 clause
    description     TEXT NOT NULL,
    evidence        TEXT,
    capa_id         UUID REFERENCES capa(id), -- linked corrective action
    status          VARCHAR(30) NOT NULL DEFAULT 'open', -- open, in_progress, closed
    due_date        DATE,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_finding_audit ON audit_finding(audit_id);
```

---

## Document Control

```sql
CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    document_number VARCHAR(100) NOT NULL,
    revision        VARCHAR(20) NOT NULL DEFAULT '1',
    title           VARCHAR(500) NOT NULL,
    doc_type        VARCHAR(50) NOT NULL,
        -- 'procedure', 'work_instruction', 'form', 'specification',
        -- 'drawing', 'control_plan', 'fmea', 'policy'
    category        VARCHAR(100),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, in_review, approved, effective, superseded, obsolete
    owner_id        UUID NOT NULL REFERENCES app_user(id),
    effective_date  DATE,
    review_due_date DATE,
    retention_years INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, document_number, revision)
);

CREATE INDEX idx_document_tenant_status ON document(tenant_id, status);

CREATE TABLE document_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    file_path       VARCHAR(1000) NOT NULL,
    file_size       BIGINT,
    mime_type       VARCHAR(100),
    change_summary  TEXT,
    uploaded_by     UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_version ON document_version(document_id);
```

---

## Equipment & Calibration

```sql
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    equipment_number VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    equipment_type  VARCHAR(50) NOT NULL,
        -- 'gauge', 'cmm', 'fixture', 'caliper', 'micrometer', 'go_no_go'
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    serial_number   VARCHAR(100),
    opcua_node_id   VARCHAR(255), -- OPC-UA node for automated data collection
    accuracy        NUMERIC(18,6),
    resolution      NUMERIC(18,6),
    measurement_range_min NUMERIC(18,6),
    measurement_range_max NUMERIC(18,6),
    uom             VARCHAR(20),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
        -- active, due_for_calibration, out_of_calibration, retired
    location        VARCHAR(255),
    last_calibration_date DATE,
    next_calibration_due  DATE,
    calibration_interval_days INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, equipment_number)
);

CREATE INDEX idx_equipment_cal_due ON equipment(next_calibration_due)
    WHERE status != 'retired';

CREATE TABLE calibration_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    calibration_date DATE NOT NULL,
    next_due_date   DATE NOT NULL,
    performed_by    UUID REFERENCES app_user(id),
    external_lab    VARCHAR(255), -- if outsourced calibration
    certificate_number VARCHAR(100),
    result          VARCHAR(20) NOT NULL, -- 'pass', 'fail', 'adjusted'
    as_found        JSONB, -- measurement results before adjustment
    as_left         JSONB, -- measurement results after adjustment
    uncertainty     NUMERIC(18,6),
    attachment_id   UUID REFERENCES attachment(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cal_equipment ON calibration_record(equipment_id, calibration_date DESC);
```

---

## Attachments

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
    ai_classification JSONB, -- AI defect classification results
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_entity ON attachment(entity_type, entity_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Platform (tenant, users, RBAC, audit) | 8 | Tenant, facility, user, role, permission, role_permission, user_role, audit_log |
| Electronic Signatures | 1 | 21 CFR Part 11 compliance |
| Product Definition | 2 | Product, product_characteristic |
| Inspection Planning & Execution | 4 | Plan, plan_item, inspection, inspection_result |
| Statistical Process Control | 4 | Study, subgroup, data_point, rule_violation |
| Non-Conformance & CAPA | 4 | NCR, CAPA, capa_non_conformance junction, capa_action |
| Supplier Quality & PPAP | 6 | Supplier, ppap_submission, ppap_element, fai_report, fai_characteristic_result, supplier_scorecard |
| Audit Management | 3 | Audit, audit_team_member, audit_finding |
| Document Control | 2 | Document, document_version |
| Equipment & Calibration | 2 | Equipment, calibration_record |
| Attachments | 1 | Polymorphic attachment storage |
| **Total** | **37** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — Supports multi-tenant distributed deployments, avoids sequential ID leakage between tenants, and aligns with modern SaaS practices.

2. **Tenant ID on every root entity** — Enables PostgreSQL Row-Level Security (RLS) policies for data isolation. Index-leading with tenant_id ensures efficient per-tenant queries.

3. **Separate product_characteristic table** — Models the "balloon" concept from engineering drawings as first-class entities. Each characteristic has its own specification limits, enabling per-characteristic SPC tracking aligned with QIF's characteristic model.

4. **Normalised PPAP with 18-element breakdown** — Each of the AIAG PPAP elements is a separate row rather than 18 columns, making it easy to track status per element and adapt to different PPAP levels without schema changes.

5. **SPC three-level hierarchy (study → subgroup → data_point)** — Mirrors the mathematical structure of control charts. Subgroups aggregate to chart points; data points are individual measurements. This supports all AIAG SPC manual chart types.

6. **Polymorphic audit_log table** — Uses entity_type + entity_id pattern to record changes across all domain entities in a single table. field_changes JSONB column captures before/after values. This is the one controlled use of JSONB in an otherwise fully normalised model.

7. **Junction table for CAPA-to-NCR** — A single CAPA may address multiple non-conformances (pattern from systemic issues), and a single NCR may spawn multiple CAPAs (immediate containment + long-term fix). Many-to-many is the correct cardinality.

8. **Equipment with OPC-UA node ID** — Enables automated measurement data collection from shop floor gauges and CMMs without manual entry, aligned with IEC 62541.

9. **AS9102B FAI report structure** — Separate fai_report and fai_characteristic_result tables map to the Form 1 (part accountability) and Form 3 (characteristic accountability) structure of the AS9102B standard.

10. **Supplier certification_scope as TEXT array** — A supplier may hold multiple certifications (ISO 9001 + IATF 16949 + Nadcap). Using a PostgreSQL array avoids a junction table for this simple many-valued attribute while remaining queryable with array operators.
