# Data Model Suggestion 4: Graph-Relational (Supply Chain Traceability)

> Project: Quality Control & Inspection · Created: 2026-05-22

## Philosophy

This model combines a conventional relational core for operational CRUD (inspections, measurements, CAPAs) with a property graph layer that models the complex relationship networks inherent in manufacturing quality: supplier-to-part-to-inspection-to-defect-to-CAPA chains, lot traceability through multi-tier supply networks, conflict-of-interest detection for audit assignments, and root cause propagation analysis. The relational tables handle day-to-day quality operations while the graph layer answers relationship-heavy questions that would require recursive CTEs or multiple self-JOINs in a purely relational model.

Manufacturing quality management is fundamentally a graph problem. A finished product traces back through an assembly hierarchy to sub-components, each sourced from different suppliers, each inspected at different facilities, each with their own quality histories. When a field failure occurs, the manufacturer needs to traverse this graph to identify all affected lots, all inspection records for those lots, all other products using the same supplier and material, and all historical CAPAs for similar defects. In a normalised relational model, this requires complex multi-table JOINs and recursive queries. In a graph model, it is a simple traversal.

The graph layer is implemented using PostgreSQL's relational tables (`graph_node` and `graph_edge`) with `ltree` for hierarchical paths, rather than requiring a separate graph database like Neo4j. This keeps the entire system in one database with full ACID transactions, unified backup, and no synchronisation between separate stores. For organisations that outgrow the relational graph layer, the schema maps directly to a dedicated graph database without structural changes.

**Best for:** Multi-tier supply chain manufacturers (automotive, aerospace, defence) needing deep traceability, lot genealogy, supplier network analysis, and root cause impact assessment across complex product hierarchies.

**Trade-offs:**
- (+) Lot-to-lot traceability and supplier network analysis are native graph traversals, not complex JOINs
- (+) Impact analysis ("what else is affected?") is fast and intuitive
- (+) Product hierarchy (BOM) navigation is efficient with graph or ltree patterns
- (+) Root cause propagation tracking: follow a defect from supplier through assembly to customer
- (+) Conflict-of-interest detection for audit assignments via relationship traversal
- (+) Graph layer maps directly to Neo4j/Neptune if relational graph outgrows capacity
- (-) Two conceptual models (relational + graph) increase developer cognitive load
- (-) Graph queries require learning Cypher-like patterns or recursive CTEs
- (-) Edge table grows large in complex supply chains; requires careful indexing and pruning
- (-) Graph consistency must be maintained alongside relational consistency
- (-) Overhead of maintaining graph edges for every relationship change

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 9001:2015 (Clause 8.5.2) | Product traceability requirements implemented via graph traversal from finished goods to raw material lots |
| IATF 16949:2016 | Supply chain traceability for automotive recall scenarios modelled as graph paths from VIN to component lots |
| AS9100D | Configuration management and traceability for aerospace parts modelled as hierarchical graph with serialised units |
| ISO 13485 / FDA QMSR | UDI-based device traceability through the supply chain as a directed graph |
| ISO 22095 (Chain of Custody) | Material chain of custody modelled as directed edges between supply chain nodes |
| AIAG Core Tools | PFMEA cause-effect relationships modelled as graph edges between process steps and failure modes |
| ISO 23952:2020 (QIF) | QIF Plans → Results → Statistics → Rules mapped as graph relationships between quality artefacts |
| OASIS OSLC QM 2.1 | OSLC resource links (test plan → test case → test result → defect) as graph edges |
| OPC-UA (IEC 62541) | Equipment → measurement → product graph for automated data provenance |

---

## Graph Layer

The graph layer provides a flexible, traversable representation of relationships between quality entities. It is implemented using two PostgreSQL tables with ltree support for hierarchical paths.

```sql
-- Enable ltree extension for hierarchical path queries
CREATE EXTENSION IF NOT EXISTS ltree;

-- ============================================================
-- GRAPH NODE: Represents any entity in the quality graph
-- ============================================================

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
        -- 'product', 'component', 'lot', 'supplier', 'facility',
        -- 'inspection', 'ncr', 'capa', 'equipment', 'process_step',
        -- 'failure_mode', 'material', 'customer', 'work_order'
    entity_id       UUID NOT NULL,          -- FK to the relational table
    label           VARCHAR(500) NOT NULL,   -- human-readable label
    hierarchy_path  LTREE,                  -- e.g. 'product.assy_100.sub_200.part_300'
    properties      JSONB NOT NULL DEFAULT '{}',
        -- Node-specific metadata for graph queries:
        -- Product: {"part_number": "PN-1234", "revision": "C"}
        -- Lot: {"lot_number": "LOT-2026-0042", "quantity": 500}
        -- Supplier: {"supplier_code": "SUP-001", "risk_tier": "high"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_tenant_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gn_entity ON graph_node(entity_id);
CREATE INDEX idx_gn_hierarchy ON graph_node USING gist(hierarchy_path);
CREATE INDEX idx_gn_properties ON graph_node USING gin(properties);

-- ============================================================
-- GRAPH EDGE: Directed relationship between two nodes
-- ============================================================

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
        -- Supply chain:
        --   'supplies', 'manufactured_at', 'assembled_from', 'contains_component'
        -- Quality events:
        --   'inspected_by', 'found_defect_in', 'raised_ncr_for', 'capa_addresses',
        --   'used_equipment', 'measured_characteristic'
        -- Traceability:
        --   'lot_used_in', 'lot_sourced_from', 'serial_traced_to'
        -- Process:
        --   'follows_step', 'has_failure_mode', 'causes', 'mitigated_by'
        -- Organisational:
        --   'works_at', 'reports_to', 'audited_by', 'approved_by'
    weight          NUMERIC(8,4) DEFAULT 1.0,  -- for weighted graph algorithms
    properties      JSONB NOT NULL DEFAULT '{}',
        -- {"effective_from": "2026-01-01", "effective_to": null,
        --  "quantity": 4, "notes": "4 units per assembly"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_source ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_ge_target ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_ge_tenant_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_ge_properties ON graph_edge USING gin(properties);
```

---

## Relational Core: Operational Tables

### Tenant, Users & RBAC

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
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    is_system       BOOLEAN NOT NULL DEFAULT false,
    UNIQUE (tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    facility_id     UUID REFERENCES facility(id),
    PRIMARY KEY (user_id, role_id, facility_id)
);
```

### Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    field_changes   JSONB,
    context         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
```

### Product & BOM Hierarchy

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
    hierarchy_path  LTREE,
        -- e.g. 'assy_widget_a.sub_housing.part_bearing_insert'
        -- enables: SELECT * FROM product WHERE hierarchy_path <@ 'assy_widget_a'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, part_number, revision)
);

CREATE INDEX idx_product_hierarchy ON product USING gist(hierarchy_path);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_char ON product_characteristic(product_id);

-- BOM relationships are modelled as graph edges (contains_component)
-- rather than a separate bom_line table, enabling multi-level traversal
```

### Supplier

```sql
CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    supplier_code   VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    duns_number     VARCHAR(9),
    country_code    CHAR(2),
    risk_tier       VARCHAR(20) NOT NULL DEFAULT 'standard',
    quality_rating  NUMERIC(5,2),
    is_approved     BOOLEAN NOT NULL DEFAULT false,
    certification_scope TEXT[],
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, supplier_code)
);

CREATE INDEX idx_supplier_tenant ON supplier(tenant_id);
```

### Lot & Serial Tracking

```sql
-- Lot tracking is central to the graph model: lots are graph nodes
-- with edges connecting them to suppliers, products, inspections, and NCRs

CREATE TABLE lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    lot_number      VARCHAR(100) NOT NULL,
    product_id      UUID NOT NULL REFERENCES product(id),
    supplier_id     UUID REFERENCES supplier(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    quantity         INTEGER,
    received_at     TIMESTAMPTZ,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending_inspection',
        -- pending_inspection, accepted, rejected, quarantined, consumed
    work_order      VARCHAR(100),
    purchase_order  VARCHAR(100),
    material_cert   JSONB DEFAULT '{}',
        -- {"heat_number": "H-9876", "mill": "XYZ Steel",
        --  "material_spec": "ASTM A36", "test_results": {...}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, lot_number)
);

CREATE INDEX idx_lot_tenant ON lot(tenant_id, status);
CREATE INDEX idx_lot_product ON lot(product_id);
CREATE INDEX idx_lot_supplier ON lot(supplier_id) WHERE supplier_id IS NOT NULL;
```

### Inspection

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
    sampling_config JSONB DEFAULT '{}',
    items           JSONB NOT NULL DEFAULT '[]',
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, plan_number, revision)
);

CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    inspection_plan_id UUID NOT NULL REFERENCES inspection_plan(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    inspector_id    UUID NOT NULL REFERENCES app_user(id),
    inspection_number VARCHAR(100) NOT NULL,
    lot_id          UUID REFERENCES lot(id),  -- graph-aware: links to lot node
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress',
    work_order      VARCHAR(100),
    lot_number      VARCHAR(100),
    quantity_inspected INTEGER,
    quantity_accepted  INTEGER,
    quantity_rejected  INTEGER,
    disposition     VARCHAR(50),
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
CREATE INDEX idx_inspection_lot ON inspection(lot_id) WHERE lot_id IS NOT NULL;

CREATE TABLE inspection_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspection(id) ON DELETE CASCADE,
    plan_item_id    UUID NOT NULL,
    characteristic_id UUID REFERENCES product_characteristic(id),
    sample_number   INTEGER NOT NULL DEFAULT 1,
    measured_value  NUMERIC(18,6),
    attribute_result VARCHAR(20),
    is_conforming   BOOLEAN NOT NULL,
    defect_code     VARCHAR(50),
    defect_severity VARCHAR(20),
    equipment_id    UUID REFERENCES equipment(id),
    measured_by     UUID NOT NULL REFERENCES app_user(id),
    measured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_result_inspection ON inspection_result(inspection_id);
```

### SPC

```sql
CREATE TABLE spc_study (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_characteristic_id UUID NOT NULL REFERENCES product_characteristic(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    chart_type      VARCHAR(20) NOT NULL,
    subgroup_size   INTEGER NOT NULL DEFAULT 1,
    ucl             NUMERIC(18,6),
    lcl             NUMERIC(18,6),
    center_line     NUMERIC(18,6),
    ucl_range       NUMERIC(18,6),
    lcl_range       NUMERIC(18,6),
    cl_range        NUMERIC(18,6),
    cp              NUMERIC(8,4),
    cpk             NUMERIC(8,4),
    pp              NUMERIC(8,4),
    ppk             NUMERIC(8,4),
    is_in_control   BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    last_calculated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spc_char ON spc_study(product_characteristic_id);

CREATE TABLE spc_data_point (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spc_study_id    UUID NOT NULL REFERENCES spc_study(id) ON DELETE CASCADE,
    subgroup_number INTEGER NOT NULL,
    sample_index    INTEGER NOT NULL,
    measured_value  NUMERIC(18,6) NOT NULL,
    inspection_result_id UUID REFERENCES inspection_result(id),
    equipment_id    UUID REFERENCES equipment(id),
    operator_id     UUID REFERENCES app_user(id),
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spc_dp ON spc_data_point(spc_study_id, collected_at);

CREATE TABLE spc_rule_violation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spc_study_id    UUID NOT NULL REFERENCES spc_study(id),
    subgroup_number INTEGER NOT NULL,
    rule_name       VARCHAR(50) NOT NULL,
    rule_description TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    acknowledged_by UUID REFERENCES app_user(id),
    acknowledged_at TIMESTAMPTZ,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spc_violation ON spc_rule_violation(spc_study_id, detected_at DESC);
```

### Non-Conformance & CAPA

```sql
CREATE TABLE non_conformance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    ncr_number      VARCHAR(100) NOT NULL,
    facility_id     UUID NOT NULL REFERENCES facility(id),
    product_id      UUID REFERENCES product(id),
    inspection_id   UUID REFERENCES inspection(id),
    supplier_id     UUID REFERENCES supplier(id),
    lot_id          UUID REFERENCES lot(id),
    nc_type         VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    quantity_affected INTEGER,
    disposition     VARCHAR(50),
    disposition_by  UUID REFERENCES app_user(id),
    disposition_at  TIMESTAMPTZ,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    reported_by     UUID NOT NULL REFERENCES app_user(id),
    assigned_to     UUID REFERENCES app_user(id),
    cost_of_quality NUMERIC(12,2),
    cost_currency   CHAR(3) DEFAULT 'USD',
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ncr_number)
);

CREATE INDEX idx_ncr_tenant_status ON non_conformance(tenant_id, status);
CREATE INDEX idx_ncr_lot ON non_conformance(lot_id) WHERE lot_id IS NOT NULL;

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
    root_cause      TEXT,
    root_cause_method VARCHAR(50),
    due_date        DATE,
    effectiveness_check_due DATE,
    effectiveness_verified BOOLEAN,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, capa_number)
);

CREATE INDEX idx_capa_tenant_status ON capa(tenant_id, status);

CREATE TABLE capa_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capa_id         UUID NOT NULL REFERENCES capa(id) ON DELETE CASCADE,
    action_type     VARCHAR(30) NOT NULL,
    description     TEXT NOT NULL,
    assigned_to     UUID NOT NULL REFERENCES app_user(id),
    due_date        DATE NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    completed_at    TIMESTAMPTZ,
    evidence_notes  TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_capa_action ON capa_action(capa_id);
```

### PPAP & FAI

```sql
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
    fai_data        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, submission_number)
);

CREATE INDEX idx_ppap_supplier ON ppap_submission(supplier_id);
```

### Equipment & Documents

```sql
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    equipment_number VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    equipment_type  VARCHAR(50) NOT NULL,
    opcua_node_id   VARCHAR(255),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    next_calibration_due DATE,
    calibration_interval_days INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, equipment_number)
);

CREATE INDEX idx_equipment_cal ON equipment(next_calibration_due)
    WHERE status != 'retired';

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    document_number VARCHAR(100) NOT NULL,
    revision        VARCHAR(20) NOT NULL DEFAULT '1',
    title           VARCHAR(500) NOT NULL,
    doc_type        VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    owner_id        UUID NOT NULL REFERENCES app_user(id),
    effective_date  DATE,
    versions        JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, document_number, revision)
);

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
    metadata        JSONB DEFAULT '{}',
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
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_esig_entity ON electronic_signature(entity_type, entity_id);
```

---

## Graph Query Examples

### 1. Full Lot Traceability: From Raw Material to Finished Goods

```sql
-- Trace all products and assemblies that contain material from a specific lot
-- (e.g., for recall impact assessment)

WITH RECURSIVE lot_trace AS (
    -- Start from the lot node
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        gn.properties,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.tenant_id = 'tenant-uuid'
      AND gn.node_type = 'lot'
      AND gn.properties->>'lot_number' = 'LOT-2026-0042'

    UNION ALL

    -- Follow 'lot_used_in' and 'contains_component' edges forward
    SELECT
        target.id,
        target.node_type,
        target.label,
        target.properties,
        lt.depth + 1,
        lt.path || target.id
    FROM lot_trace lt
    JOIN graph_edge ge ON ge.source_node_id = lt.node_id
        AND ge.edge_type IN ('lot_used_in', 'contains_component', 'assembled_from')
        AND ge.is_active = true
    JOIN graph_node target ON target.id = ge.target_node_id
    WHERE target.id != ALL(lt.path)  -- prevent cycles
      AND lt.depth < 10             -- max traversal depth
)
SELECT node_type, label, properties, depth
FROM lot_trace
ORDER BY depth, node_type;

-- Returns:
-- depth 0: lot LOT-2026-0042 (bearing insert, 500 pcs)
-- depth 1: product Bearing Insert PN-300 Rev C
-- depth 2: product Sub-Assembly Housing PN-200 Rev B
-- depth 3: product Widget A Assembly PN-100 Rev A
-- depth 3: lot LOT-FINISHED-2026-0099 (finished goods)
```

### 2. Supplier Impact Analysis: Find All Products Affected by a Supplier

```sql
-- When a supplier's quality degrades, find all products they supply
-- and all downstream assemblies that depend on those products

WITH RECURSIVE supplier_impact AS (
    -- Start from the supplier node
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        0 AS depth
    FROM graph_node gn
    WHERE gn.tenant_id = 'tenant-uuid'
      AND gn.node_type = 'supplier'
      AND gn.entity_id = 'supplier-uuid'

    UNION ALL

    -- Follow supply chain edges forward
    SELECT
        target.id,
        target.node_type,
        target.label,
        si.depth + 1
    FROM supplier_impact si
    JOIN graph_edge ge ON ge.source_node_id = si.node_id
        AND ge.edge_type IN ('supplies', 'contains_component')
        AND ge.is_active = true
    JOIN graph_node target ON target.id = ge.target_node_id
    WHERE si.depth < 5
)
SELECT DISTINCT node_type, label, depth
FROM supplier_impact
WHERE node_type IN ('product', 'component')
ORDER BY depth, label;
```

### 3. Root Cause Propagation: NCR to CAPA to Prevention

```sql
-- Trace the complete quality event chain for an NCR:
-- lot -> inspection -> defect -> NCR -> CAPA -> actions

SELECT
    source.node_type AS from_type,
    source.label AS from_label,
    ge.edge_type AS relationship,
    target.node_type AS to_type,
    target.label AS to_label,
    ge.created_at
FROM graph_edge ge
JOIN graph_node source ON source.id = ge.source_node_id
JOIN graph_node target ON target.id = ge.target_node_id
WHERE ge.tenant_id = 'tenant-uuid'
  AND (
    source.entity_id = 'ncr-uuid'
    OR target.entity_id = 'ncr-uuid'
  )
ORDER BY ge.created_at;

-- Returns:
-- lot LOT-2026-0042 --[found_defect_in]--> inspection INS-2026-01234
-- inspection INS-2026-01234 --[raised_ncr_for]--> ncr NCR-2026-0089
-- ncr NCR-2026-0089 --[capa_addresses]--> capa CAPA-2026-0023
-- supplier SUP-001 --[supplies]--> product PN-1234 Rev C
```

### 4. Conflict-of-Interest Detection for Audits

```sql
-- Before assigning an auditor, check if they have any direct relationships
-- with the facility or supplier being audited

SELECT
    ge.edge_type,
    target.node_type,
    target.label
FROM graph_node auditor_node
JOIN graph_edge ge ON ge.source_node_id = auditor_node.id
JOIN graph_node target ON target.id = ge.target_node_id
WHERE auditor_node.node_type = 'user'
  AND auditor_node.entity_id = 'proposed-auditor-uuid'
  AND target.entity_id IN ('facility-being-audited-uuid', 'supplier-being-audited-uuid')
  AND ge.edge_type IN ('works_at', 'reports_to', 'approved_by', 'supplies');
```

### 5. Product Hierarchy Navigation with ltree

```sql
-- Find all sub-components of an assembly using ltree
SELECT p.part_number, p.name, p.hierarchy_path
FROM product p
WHERE p.tenant_id = 'tenant-uuid'
  AND p.hierarchy_path <@ 'assy_widget_a'
ORDER BY p.hierarchy_path;

-- Returns:
-- assy_widget_a                          Widget A Assembly
-- assy_widget_a.sub_housing              Housing Sub-Assembly
-- assy_widget_a.sub_housing.part_bearing Bearing Insert
-- assy_widget_a.sub_housing.part_seal    Seal Ring
-- assy_widget_a.sub_electronics          Electronics Module
-- assy_widget_a.sub_electronics.part_pcb PCB Assembly
```

### 6. Similar Defect Pattern Analysis (AI Input)

```sql
-- Find all NCRs linked to similar products/characteristics via graph
-- Used as input for AI root cause suggestion

WITH ncr_context AS (
    SELECT
        target.node_type,
        target.entity_id,
        target.properties
    FROM graph_node ncr_node
    JOIN graph_edge ge ON ge.source_node_id = ncr_node.id
        OR ge.target_node_id = ncr_node.id
    JOIN graph_node target ON target.id = CASE
        WHEN ge.source_node_id = ncr_node.id THEN ge.target_node_id
        ELSE ge.source_node_id
    END
    WHERE ncr_node.entity_id = 'current-ncr-uuid'
)
SELECT nc.ncr_number, nc.title, nc.severity, nc.root_cause,
       nc.disposition, nc.created_at
FROM non_conformance nc
JOIN graph_node other_ncr ON other_ncr.entity_id = nc.id
    AND other_ncr.node_type = 'ncr'
JOIN graph_edge ge2 ON (ge2.source_node_id = other_ncr.id
    OR ge2.target_node_id = other_ncr.id)
JOIN graph_node related ON related.id = CASE
    WHEN ge2.source_node_id = other_ncr.id THEN ge2.target_node_id
    ELSE ge2.source_node_id
END
WHERE related.entity_id IN (SELECT entity_id FROM ncr_context)
  AND nc.id != 'current-ncr-uuid'
ORDER BY nc.created_at DESC
LIMIT 20;
```

---

## Graph Maintenance

### Automatic Graph Node/Edge Creation

When relational records are created or updated, corresponding graph nodes and edges are maintained via application-level hooks or database triggers.

```sql
-- Example trigger: when an inspection is created, create a graph node
-- and edges linking it to the lot, product, inspector, and facility

CREATE OR REPLACE FUNCTION maintain_inspection_graph()
RETURNS TRIGGER AS $$
DECLARE
    v_node_id UUID;
    v_lot_node_id UUID;
    v_product_node_id UUID;
    v_facility_node_id UUID;
    v_inspector_node_id UUID;
BEGIN
    -- Create or find the inspection node
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.tenant_id, 'inspection', NEW.id,
        'Inspection ' || NEW.inspection_number,
        jsonb_build_object(
            'inspection_number', NEW.inspection_number,
            'status', NEW.status,
            'plan_type', (SELECT plan_type FROM inspection_plan WHERE id = NEW.inspection_plan_id)
        )
    )
    ON CONFLICT (entity_id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties
    RETURNING id INTO v_node_id;

    -- Link to lot (if present)
    IF NEW.lot_id IS NOT NULL THEN
        SELECT id INTO v_lot_node_id FROM graph_node
        WHERE entity_id = NEW.lot_id AND node_type = 'lot';

        IF v_lot_node_id IS NOT NULL THEN
            INSERT INTO graph_edge (tenant_id, source_node_id, target_node_id, edge_type)
            VALUES (NEW.tenant_id, v_lot_node_id, v_node_id, 'inspected_by')
            ON CONFLICT DO NOTHING;
        END IF;
    END IF;

    -- Link to inspector
    SELECT id INTO v_inspector_node_id FROM graph_node
    WHERE entity_id = NEW.inspector_id AND node_type = 'user';

    IF v_inspector_node_id IS NOT NULL THEN
        INSERT INTO graph_edge (tenant_id, source_node_id, target_node_id, edge_type)
        VALUES (NEW.tenant_id, v_inspector_node_id, v_node_id, 'performed_inspection')
        ON CONFLICT DO NOTHING;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_inspection_graph
    AFTER INSERT OR UPDATE ON inspection
    FOR EACH ROW EXECUTE FUNCTION maintain_inspection_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge |
| Core Platform (tenant, users, RBAC, audit) | 6 | tenant, facility, app_user, role, user_role, audit_log |
| Product & BOM | 2 | product (with ltree), product_characteristic |
| Lot Tracking | 1 | lot (central to graph model) |
| Supplier & PPAP | 2 | supplier, ppap_submission |
| Inspection | 3 | inspection_plan, inspection, inspection_result |
| SPC | 3 | spc_study, spc_data_point, spc_rule_violation |
| Non-Conformance & CAPA | 3 | non_conformance, capa, capa_action |
| Document & Equipment | 2 | document, equipment |
| Attachments & Signatures | 2 | attachment, electronic_signature |
| **Total** | **26** | Plus graph_node/edge rows scale with relationships |

---

## Key Design Decisions

1. **PostgreSQL-native graph layer** — Using `graph_node` and `graph_edge` tables within PostgreSQL rather than a separate graph database (Neo4j, Amazon Neptune) keeps the entire system in one database with full ACID transactions. Recursive CTEs provide graph traversal capability. If the graph workload outgrows PostgreSQL, the schema maps directly to a property graph database without structural changes.

2. **ltree for product hierarchy** — PostgreSQL's `ltree` extension provides efficient hierarchical queries (ancestors, descendants, path matching) for product BOM structures. Combined with graph edges for non-hierarchical relationships (supplier links, quality events), this covers both tree and graph query patterns.

3. **Lot as a first-class entity with graph node** — Unlike the normalised model where lot_number is just a string on an inspection, this model promotes lots to their own table and graph node. This enables lot-level traceability: every lot has edges connecting it to its supplier, product, facility, inspections, and any NCRs. Lot genealogy (sub-lot splits, lot merges) is modelled as graph edges.

4. **Graph edges are typed and weighted** — Edge types form a controlled vocabulary (`supplies`, `lot_used_in`, `found_defect_in`, `capa_addresses`). Weights enable shortest-path and importance ranking algorithms. Properties on edges carry temporal and quantitative metadata (effective dates, quantities per assembly).

5. **Dual query paths** — Simple operational queries (list open inspections, get CAPA details) use the relational tables directly. Complex relationship queries (lot traceability, impact analysis, supplier network) use the graph layer. Developers choose the right tool for the query type.

6. **Graph maintenance via triggers** — Database triggers automatically maintain graph nodes and edges when relational records are created or updated. This keeps the graph synchronised without requiring application code to explicitly manage both layers for every operation.

7. **BOM modelled as graph edges, not a separate table** — Bill of materials relationships (`contains_component`) are graph edges between product nodes. This means BOM traversal uses the same graph infrastructure as lot traceability and supplier analysis, enabling cross-domain queries like "find all assemblies containing components from supplier X that have had NCRs in the last 90 days."

8. **Conflict-of-interest detection** — Organisational relationships (works_at, reports_to) are graph edges. Before assigning an auditor, a simple graph query checks for relationships between the proposed auditor and the audit target. This would require multiple joins and union queries in a purely relational model.

9. **AI-ready relationship data** — The graph structure is natural input for graph neural networks and knowledge graph embeddings. Supplier risk scoring, defect pattern similarity, and root cause prediction all benefit from explicit relationship data that would be implicit (and harder to extract) in a normalised relational model.

10. **26 tables total** — Fewer than the normalised model (37) but more than the event-sourced model (20). The graph layer adds only 2 tables but the graph_edge table can grow large. The lot table is new compared to other models, reflecting the graph model's emphasis on traceability as a first-class concern.
