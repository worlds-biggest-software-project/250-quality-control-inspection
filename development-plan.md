# Quality Control & Inspection — Phased Development Plan

> Project: 250-quality-control-inspection · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` proposals into a concrete, phased build. The product is an **AI-native, open-source Quality Management System (QMS)** for discrete manufacturers, delivering digital inspection, real-time SPC, non-conformance/CAPA, supplier quality (PPAP/FAI), audits, document control, and equipment calibration — with AI woven directly into the workflows (inspection-plan generation, SPC interpretation, CAPA drafting, defect classification, supplier risk).

---

## Core Requirements (Synthesis)

**What it does.** A single platform where quality engineers build digital inspection plans, operators capture measurements (online and offline), the system computes SPC control charts and capability indices in real time, out-of-control conditions raise non-conformances that flow into structured CAPA (8D/A3) workflows, suppliers submit PPAP/FAI packages through a portal, and auditors run ISO 9001 / IATF 16949 / AS9100 audits — all with a REST + webhook API for ERP/MES integration and AI assistance at each step.

**Primary personas.** Quality Manager/Director; Supplier Quality Engineer; Production supervisor/operator (shop-floor SPC entry); ISO/compliance lead/auditor; Supplier (external portal user).

**Key differentiators.** (1) Enterprise-grade SPC + PPAP + supplier portal at SMB-accessible price, open source; (2) AI-native: CV defect classification, AI SPC interpretation, NL CAPA drafting, AI inspection-plan generation, predictive supplier risk; (3) Open-standards interoperability (OpenAPI 3.1, OPC-UA, OSLC QM affinity) rather than proprietary lock-in.

**MVP scope (from features.md "Must-have").** Inspection form builder w/ conditional logic + photo + pass/fail; NCR creation + RCA + corrective action; basic SPC (I-MR, X-bar R) with Nelson-rule alerting; supplier PPAP request/tracking/approval; ISO 9001 audit management; mobile-first offline capture + sync; REST API + webhooks; role-based KPI dashboards.

**Post-MVP (v1.1 / backlog).** AI inspection-plan generation; advanced SPC (Cp/Cpk/Pp/Ppk, Gage R&R, MSA); CAPA root-cause assistant; AS9102 FAI builder + ballooning import; IATF/AS9100 compliance modes; document control; supplier scorecards w/ risk tiers; CV defect classification; predictive supplier risk ML; NL 8D/A3 drafting; 21 CFR Part 11 e-signatures; PLM integration; executive AI reporting.

**Deployment model.** Hybrid — self-hosted (Docker Compose) and cloud SaaS from one codebase; multi-tenant with row-level isolation. Offline-capable PWA for shop floor.

**Integration surface.** REST (OpenAPI 3.1) + outbound webhooks; OPC-UA gauge/CMM ingestion via a bridge agent; LLM providers (via an abstraction); object storage for attachments; SAML/OIDC SSO.

**Standards to implement.** ISO 9001:2015 (audit/CAPA/doc-control clause mapping), IATF 16949 (APQP/PPAP/MSA/SPC), AS9100D / AS9102B (FAI Forms 1/2/3), AIAG SPC manual (chart types, Nelson/WECO rules, Cp/Cpk/Pp/Ppk), AIAG MSA (Gage R&R), ISO 2859-1 / ISO 3951 / ANSI-ASQ Z1.4-Z1.9 (sampling), 21 CFR Part 11 (e-sig + audit trail), OpenAPI 3.1 + JSON Schema 2020-12, OAuth2/OIDC + JWT, OPC-UA (IEC 62541).

**Data model choice.** Primary = **Data Model Suggestion 3 (Hybrid Relational + JSONB)** — typed columns for SPC numerics, FK integrity, and compliance-critical fields; JSONB for inspection-form responses, industry-variant PPAP/FAI form data, sampling parameters, equipment connection config, and AI metadata. Audit-trail and electronic-signature tables are adopted from **Suggestion 1**. The schema is built additively over the phases below.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The product is AI-heavy (CV defect classification, LLM orchestration, SPC statistics, ML supplier risk). Python has the strongest ecosystem for NumPy/SciPy SPC math, scikit-learn, ONNX/torch inference, and LLM SDKs. |
| API framework | **FastAPI** | Async, first-class Pydantic v2 validation, and auto-generated **OpenAPI 3.1** (a stated standard). Native dependency-injection suits multi-tenant RBAC guards. |
| Data validation / schemas | **Pydantic v2** | Request/response models double as the OpenAPI 3.1 / JSON Schema 2020-12 source of truth and validate JSONB payloads at the app layer (required by the hybrid model). |
| Database | **PostgreSQL 16** | Hybrid model depends on JSONB + GIN indexes, arrays, `ltree`, RLS for tenant isolation, and `TIMESTAMPTZ`. Single ACID store for relational + flexible data. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Mature async ORM; Alembic gives the formal, reviewable migrations the compliance story needs. |
| Task queue | **Celery + Redis** | Async workloads: webhook delivery, SPC recompute, PPAP package assembly, CV inference, LLM calls, scorecard rollups, OPC-UA ingestion buffering. Redis doubles as cache + Celery broker/result backend. |
| Realtime | **WebSockets (FastAPI) + Redis pub/sub** | Live SPC charts and out-of-control alerts pushed to dashboards/shop-floor terminals. |
| Frontend | **Next.js 15 (React, TypeScript) as a PWA** | Mobile-first offline shop-floor capture requires a PWA with a service worker; SSR dashboards; one app for desktop + tablet. |
| Offline store (client) | **IndexedDB via Dexie.js** | Queues inspection results offline; background sync on reconnect (a hard MVP requirement). |
| UI components / charts | **shadcn/ui + Tailwind; uPlot for SPC charts** | uPlot renders large SPC time-series with control-limit/zone overlays at high performance; shadcn for forms/dashboards. |
| Auth | **OAuth2 / OIDC + JWT; SAML 2.0 (enterprise)** | Matches standards.md (RFC 6749, 7519; OIDC; SAML). Local password auth for self-hosted; SSO for enterprise. |
| LLM abstraction | **`llm` provider layer (Anthropic/OpenAI/local)** | AI-native features must not lock to one vendor; a thin interface allows self-hosted models (Ollama) for air-gapped manufacturers. |
| CV inference | **ONNX Runtime** | Run defect-classification models portably (server or edge); export from any training framework. |
| SPC / stats | **NumPy + SciPy** | Control-limit constants, capability indices, Gage R&R ANOVA, AQL lookups. |
| OPC-UA | **`asyncua`** | Pure-Python OPC-UA client for the gauge/CMM bridge agent (IEC 62541). |
| PDF/report generation | **WeasyPrint** | HTML→PDF for FAI/PPAP packages, inspection reports, audit reports, e-sig manifests. |
| Object storage | **S3-compatible (MinIO self-host / S3 cloud)** | Photos, documents, PPAP files, calibration certs. |
| Containerisation | **Docker + Docker Compose** | One-command self-host; same images deploy to cloud. |
| Testing | **pytest + pytest-asyncio + testcontainers; Playwright (frontend e2e)** | Unit + integration against ephemeral Postgres/Redis; e2e for offline-sync and shop-floor flows. |
| Code quality | **Ruff (lint+format), mypy (strict), pre-commit** | Consistent style + static types across a large codebase. |
| Package mgmt | **uv (Python), pnpm (frontend)** | Fast, reproducible installs. |
| API docs | **FastAPI auto OpenAPI + Redoc** | Published spec is itself a deliverable for ERP/MES integrators. |

### Project Structure

```
quality-control-inspection/
├── pyproject.toml
├── uv.lock
├── docker-compose.yml                # postgres, redis, minio, api, worker, web
├── Dockerfile.api
├── Dockerfile.worker
├── alembic.ini
├── .pre-commit-config.yaml
├── README.md
├── docs/
│   └── openapi.json                  # exported spec (CI artefact)
├── migrations/                       # Alembic
│   ├── env.py
│   └── versions/
├── src/qci/
│   ├── main.py                       # FastAPI app factory, router mounting
│   ├── config.py                     # Pydantic Settings (env)
│   ├── db.py                         # async engine, session, RLS helpers
│   ├── core/
│   │   ├── security.py               # JWT, password hashing, OIDC/SAML
│   │   ├── rbac.py                   # permission registry + dependency guards
│   │   ├── tenancy.py                # tenant context middleware, RLS GUC
│   │   ├── audit.py                  # audit_log writer, change diffing
│   │   ├── esignature.py            # 21 CFR Part 11 signing
│   │   ├── pagination.py
│   │   ├── errors.py                 # error envelope, exception handlers
│   │   └── events.py                 # domain event bus → webhooks + ws
│   ├── models/                       # SQLAlchemy ORM (one file per domain)
│   │   ├── base.py                   # Base, TenantMixin, TimestampMixin
│   │   ├── org.py  users.py  product.py  inspection.py  spc.py
│   │   ├── nc_capa.py  supplier.py  audit_mgmt.py  document.py
│   │   ├── equipment.py  attachment.py  webhook.py
│   ├── schemas/                      # Pydantic request/response + JSONB schemas
│   ├── services/                     # business logic (pure, testable)
│   │   ├── inspection_service.py  spc_engine.py  capability.py
│   │   ├── sampling.py  capa_service.py  ppap_service.py  fai_service.py
│   │   ├── audit_service.py  document_service.py  scorecard.py
│   │   ├── calibration.py  dashboard.py
│   ├── ai/
│   │   ├── llm.py                    # provider abstraction
│   │   ├── prompts/                  # prompt templates (jinja)
│   │   ├── plan_generation.py  spc_interpreter.py  capa_assistant.py
│   │   ├── defect_classifier.py  supplier_risk.py  report_narrative.py
│   ├── integrations/
│   │   ├── webhooks.py  opcua_bridge.py  storage.py  reports.py
│   ├── api/v1/                       # FastAPI routers (one per domain)
│   ├── workers/                      # Celery tasks
│   └── ws/                           # websocket endpoints + redis bridge
├── tests/
│   ├── conftest.py                   # testcontainers fixtures, factories
│   ├── unit/  integration/  e2e/  fixtures/   # sample drawings, CMM files
└── web/                              # Next.js PWA
    ├── package.json  next.config.js
    ├── public/manifest.json  sw.js
    ├── src/app/  src/components/  src/lib/api/  src/lib/offline/ (Dexie)
    └── tests/                        # Playwright
```

The structure is grouped by concern (models / schemas / services / ai / api), so every phase adds files without restructuring.

---

## Phase 1: Foundation — Platform, Tenancy, Auth, Audit Trail

### Purpose
Establish the multi-tenant skeleton every other feature depends on: configuration, database connectivity with row-level isolation, the user/RBAC model, JWT/OIDC auth, the polymorphic audit trail, error handling, and a running FastAPI app with auto OpenAPI. After this phase a developer can authenticate, the tenant context is enforced on every query, and every write is auditable.

### Tasks

#### 1.1 — Project scaffold, config, app factory
**What**: Bootable FastAPI app with settings, health check, structured logging, Docker Compose (postgres/redis/minio).

**Design**:
- `Settings` (pydantic-settings) reads env: `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `JWT_ALG=HS256`, `ACCESS_TOKEN_TTL=900`, `REFRESH_TOKEN_TTL=2592000`, `S3_ENDPOINT/KEY/SECRET/BUCKET`, `LLM_PROVIDER`, `LLM_API_KEY`, `ENV=dev|prod`.
- `create_app()` mounts routers, registers exception handlers, adds tenancy + audit middleware, exposes `/healthz` (returns `{status, db, redis}`) and `/openapi.json`.
- Error envelope: `{ "error": { "code": str, "message": str, "details": [...] } }`; HTTP 422 for validation, 401/403 for auth, 404, 409 conflict.

**Testing**:
- `Unit: Settings loads from env with defaults; missing JWT_SECRET in prod → startup error`.
- `Integration: GET /healthz with DB+Redis up → 200 {status:"ok"}`.
- `Integration: GET /healthz with DB down → 503, db:"down"`.

#### 1.2 — Database layer, base models, tenancy + RLS
**What**: Async SQLAlchemy engine/session, `Base`/`TenantMixin`/`TimestampMixin`, and Postgres RLS enforcing tenant isolation.

**Design**:
```python
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class TenantMixin:
    tenant_id: Mapped[UUID] = mapped_column(ForeignKey("tenant.id"), index=True)
```
- Tables `tenant`, `facility` per data-model-suggestion-1.
- RLS: each tenant-scoped table gets `ENABLE ROW LEVEL SECURITY` + policy `USING (tenant_id = current_setting('app.tenant_id')::uuid)`. `tenancy.py` middleware resolves tenant from the JWT and issues `SET LOCAL app.tenant_id = :tid` at session start.
- Session dependency `get_session()` yields an `AsyncSession` already bound to the request's tenant.

**Testing**:
- `Integration: insert facilities for tenant A and B; session as A → SELECT returns only A's rows`.
- `Integration: attempt cross-tenant fetch by id → 404 (RLS hides row)`.
- `Unit: TimestampMixin sets updated_at on update`.

#### 1.3 — Users, roles, permissions (RBAC)
**What**: `app_user`, `role`, `permission`, `role_permission`, `user_role` (facility-scoped), seeded system roles + permission registry.

**Design**:
- Tables per suggestion-1 (RBAC section). `user_role.facility_id NULL = all facilities`.
- Permission registry: `Permission(resource, action)` constants, e.g. `("inspection","create")`, `("capa","approve")`, `("spc","read")`, `("ppap","review")`, `("supplier_portal","submit")`.
- Seeded roles: `quality_manager` (all), `quality_engineer`, `inspector` (inspection create/read, spc read), `auditor` (audit/finding), `supplier` (portal only), `viewer` (read dashboards).
- RBAC guard dependency:
```python
def require(resource: str, action: str) -> Callable:
    async def dep(user: CurrentUser = Depends(get_current_user)) -> CurrentUser:
        if not user.has(resource, action): raise Forbidden(resource, action)
        return user
    return dep
```

**Testing**:
- `Unit: user with quality_engineer role → has("inspection","create") True, has("capa","approve") False`.
- `Integration: route guarded by require("capa","approve") called by inspector → 403`.
- `Integration: facility-scoped role → user limited to facility X cannot read facility Y inspection`.

#### 1.4 — Authentication (local + OIDC/JWT)
**What**: Login, refresh, logout; password hashing; JWT issuance with tenant + roles; OIDC code-flow hook; SAML stub.

**Design**:
- Endpoints: `POST /v1/auth/login {email,password}` → `{access_token, refresh_token}`; `POST /v1/auth/refresh`; `POST /v1/auth/logout`; `GET /v1/auth/me`.
- Passwords: `argon2`. JWT claims: `sub, tenant_id, roles[], facilities[], exp, jti`. Refresh tokens stored hashed in Redis (`jti`-keyed) for revocation.
- `auth_provider` per user: `local|oidc|saml`. OIDC: `GET /v1/auth/oidc/start`, `/callback` (PKCE), maps `external_id`→user (auto-provision optional).

**Testing**:
- `Integration: valid creds → 200 with tokens; me returns user`.
- `Integration: wrong password → 401, no token`.
- `Integration: expired access token → 401; valid refresh → new access token`.
- `Integration: revoked refresh jti → 401`.

#### 1.5 — Audit trail + electronic signatures + domain event bus
**What**: Automatic `audit_log` writes on mutations, 21 CFR Part 11 `electronic_signature`, and an in-process event bus that later feeds webhooks/websockets.

**Design**:
- `audit.py`: `record(action, entity_type, entity_id, field_changes)` writing the suggestion-1 `audit_log` (with `ip_address`, `user_agent`). A SQLAlchemy `after_flush` hook diffs dirty fields into `{field:{old,new}}`.
- `esignature.py`: `sign(entity_type, entity_id, meaning, user, password)` re-verifies password (Part 11 non-repudiation), stores `signature_hash = sha512(user_id|entity|meaning|timestamp|prev_hash)`.
- `events.py`: `EventBus.publish(DomainEvent(type, tenant_id, entity_type, entity_id, payload))`; Phase 2+ subscribers (webhooks, ws) attach here. States/enums centralised.

**Testing**:
- `Integration: update an NCR field → audit_log row with correct old/new diff, user, ip`.
- `Integration: sign with wrong password → 401, no signature row`.
- `Unit: signature_hash chains prev_hash (tamper-evident)`.

### Definition of Done
All 1.x tests pass; Ruff + mypy clean; Docker Compose boots api+db+redis+minio; `/openapi.json` generated; Alembic baseline migration created.

---

## Phase 2: Product Definition & Inspection Engine

### Purpose
Deliver the core inspection capability: products and their characteristics ("balloons"), inspection plans with conditional-logic items and sampling parameters, and inspection execution that records typed measurements + attribute results + photos. This is the data-capture heart the rest of the platform analyses.

### Tasks

#### 2.1 — Product & characteristics
**What**: `product` and `product_characteristic` CRUD with spec limits and GD&T metadata.

**Design**:
- Tables per suggestion-1 (`product`, `product_characteristic` with `nominal/usl/lsl`, `data_type variable|attribute`, `is_critical/is_significant`, `balloon_number`). `customer_id` references `supplier` (Phase 6) — nullable FK until then.
- Endpoints: `GET/POST /v1/products`, `GET/PATCH /v1/products/{id}`, `POST /v1/products/{id}/characteristics`, etc. Unique `(tenant, part_number, revision)`.

**Testing**:
- `Unit: characteristic with usl<lsl → ValidationError`.
- `Integration: create product + 3 characteristics → retrievable, balloon order preserved`.
- `Integration: duplicate part_number+revision → 409`.

#### 2.2 — Inspection plans with conditional logic (hybrid JSONB)
**What**: `inspection_plan` + `inspection_plan_item`; form structure and conditional branching stored as validated JSONB (per data-model-3).

**Design**:
- Relational columns per suggestion-1 plan/plan_item (plan_type, status lifecycle `draft→active→superseded→retired`, `sampling_standard`, `aql`, `inspection_level`). Add `form_schema JSONB` on the plan and `logic JSONB` on items.
- `logic` schema (Pydantic-validated): `{"show_if": {"item_id": "...", "op": "eq|neq|gt|lt", "value": ...}}`. `form_schema` defines field type (`number|select|bool|text|photo`), units, and acceptance criteria.
- A `PlanEvaluator` service resolves which items are visible given partial responses (used by the client and validated server-side).

**Testing**:
- `Unit: PlanEvaluator hides item B until item A == "fail"`.
- `Unit: invalid logic op → ValidationError naming the field`.
- `Integration: activate plan → status active, prior active revision auto-superseded`.

#### 2.3 — Inspection execution & results
**What**: `inspection` + `inspection_result`; record per-sample measured values / attribute pass-fail, defect codes, photo attachment refs; compute conformance against characteristic limits.

**Design**:
- Tables per suggestion-1. `is_conforming` computed server-side: variable → `lsl<=value<=usl`; attribute → `attribute_result=='pass'`.
- Endpoints: `POST /v1/inspections` (open against a plan), `POST /v1/inspections/{id}/results` (batch sample results), `POST /v1/inspections/{id}/complete` (sets disposition, quantities). Status `in_progress→completed→accepted|rejected|on_hold`.
- Publishes `inspection.completed` and, per non-conforming result, `result.nonconforming` events (consumed in Phase 4/5).

**Testing**:
- `Unit: value above usl → is_conforming False`.
- `Integration: submit mixed results → quantity_accepted/rejected computed; status completed`.
- `Integration: complete with disposition=reject → inspection rejected, nonconforming events emitted`.

#### 2.4 — Attachments & object storage
**What**: Polymorphic `attachment` table + S3/MinIO presigned upload/download; photo capture for results.

**Design**:
- Table per suggestion-1 (`entity_type`,`entity_id`,`mime_type`,`ai_classification JSONB`). `storage.py`: `presign_upload(key)`, `presign_get(key)`; keys namespaced `tenant/{tid}/{entity_type}/{id}/{uuid}`.
- Endpoints: `POST /v1/attachments/presign` → upload URL + attachment row (pending); `POST /v1/attachments/{id}/confirm`.

**Testing**:
- `Integration (MinIO container): presign → PUT file → confirm → presign_get downloads identical bytes`.
- `Integration: attach to inspection_result; result returns photo url`.

### Definition of Done
2.x tests pass; lint/type clean; new endpoints in OpenAPI; Alembic migration for product/inspection/attachment tables; an inspection can be opened, completed, and a photo round-tripped through MinIO.

---

## Phase 3: SPC Engine — Control Charts, Rules, Capability

### Purpose
Add the statistical core that differentiates this from simple checklist tools: feed measurements into SPC studies, compute control limits and capability, evaluate Nelson/WECO rules in real time, and record violations. This realises the "real-time SPC" must-have and underpins later AI interpretation.

### Tasks

#### 3.1 — SPC data model & ingestion
**What**: `spc_study`, `spc_subgroup`, `spc_data_point`, `spc_rule_violation`; auto-route conforming variable results into studies.

**Design**:
- Tables per suggestion-1. `chart_type` enum `i_mr|xbar_r|xbar_s|p|np|c|u`. A subscriber on `inspection_result` events appends `spc_data_point`s to the open study for that `product_characteristic_id` + facility, grouping into subgroups by `subgroup_size`.

**Testing**:
- `Unit: 5 data points with subgroup_size=5 → one subgroup, mean+range computed`.
- `Integration: variable result with active study → data point appended`.

#### 3.2 — Control-limit & capability calculation engine
**What**: `spc_engine.py` computing UCL/LCL/center for each chart type and `capability.py` computing Cp/Cpk/Pp/Ppk.

**Design** (per AIAG SPC manual constants A2,D3,D4,d2,B3,B4,c4):
```python
def xbar_r_limits(subgroups: list[Subgroup], n: int) -> ControlLimits:
    xbarbar = mean(s.mean for s in subgroups); rbar = mean(s.range for s in subgroups)
    A2,D3,D4 = CONSTANTS[n]
    return ControlLimits(cl=xbarbar, ucl=xbarbar+A2*rbar, lcl=xbarbar-A2*rbar,
                         cl_r=rbar, ucl_r=D4*rbar, lcl_r=D3*rbar, sigma=rbar/D2[n])
def capability(values, usl, lsl, sigma_within, sigma_overall) -> Capability:
    cp=(usl-lsl)/(6*sigma_within); cpk=min(usl-mean,mean-lsl)/(3*sigma_within)
    pp=(usl-lsl)/(6*sigma_overall); ppk=min(usl-mean,mean-lsl)/(3*sigma_overall)
```
- I-MR uses moving range; X-bar S uses c4/B3/B4; P/NP/C/U use attribute formulas. Constants table covers n=2..10. Recompute is a Celery task `recompute_study(study_id)` triggered on new subgroups.

**Testing**:
- `Unit: known AIAG worked example → UCL/LCL/Cpk match published values within 1e-4` (fixture).
- `Unit: subgroup_size out of constants range → error`.
- `Unit: I-MR moving range computed correctly`.

#### 3.3 — Nelson & WECO rule evaluation
**What**: `sampling`-independent rule engine flagging out-of-control patterns; persist `spc_rule_violation`.

**Design**:
- Implement Nelson rules 1–8 and WECO 1–4 as pure functions over the subgroup mean series + control limits/zones. Each returns the indices/subgroups violating. Severity: rule 1 (beyond 3σ) = `alarm`; trend/zone rules = `warning`.
- Engine runs after each `recompute_study`; new violations create rows and publish `spc.violation` events (→ alerts in Phase 5, NCR in Phase 4).

**Testing**:
- `Unit: point beyond UCL → nelson_1 violation`.
- `Unit: 9 consecutive points one side of center → nelson_2`.
- `Unit: in-control series → no violations`.
- `Unit: WECO 2-of-3 in zone A → weco rule fires`.

#### 3.4 — SPC API + realtime
**What**: Endpoints to read chart data and a WebSocket stream for live points/violations.

**Design**:
- `GET /v1/spc/studies/{id}` → limits + subgroups + violations + capability; `GET /v1/spc/studies?characteristic_id=` ; `POST /v1/spc/studies` (config).
- `WS /v1/ws/spc/{study_id}`: on each new subgroup/violation, push `{type, subgroup|violation}` via Redis pub/sub fan-out.
- `is_in_control` on study set false while unacknowledged alarm exists.

**Testing**:
- `Integration: post data crossing UCL → study read shows violation, is_in_control False`.
- `Integration (ws): subscribe, append subgroup → client receives subgroup message`.

### Definition of Done
3.x tests pass; AIAG worked-example fixtures validate the math; recompute runs as a Celery task; live SPC WebSocket verified; migration for SPC tables.

---

## Phase 4: Non-Conformance & CAPA Workflow

### Purpose
Turn detected problems into managed resolution: create NCRs (manually, from rejected inspections, or from SPC alarms), drive them through disposition, and run structured 8D/A3 CAPA with actions, root cause, and effectiveness verification. Delivers the NCR + CAPA must-haves and the ISO 9001 corrective-action backbone.

### Tasks

#### 4.1 — Non-conformance records
**What**: `non_conformance` CRUD with disposition workflow and links to inspection/supplier/product.

**Design**:
- Table per suggestion-1 (status `open→under_review→dispositioned→closed`; disposition `rework|use_as_is|scrap|return_to_supplier`; severity; `cost_of_quality`). Auto-create NCR option on inspection rejection and on SPC `alarm` violations (configurable per tenant).
- Endpoints: `GET/POST /v1/ncrs`, `PATCH /v1/ncrs/{id}`, `POST /v1/ncrs/{id}/disposition`, `POST /v1/ncrs/{id}/close`.

**Testing**:
- `Integration: rejected inspection with auto-NCR on → NCR created linked to inspection`.
- `Integration: close NCR without disposition → 409`.

#### 4.2 — CAPA workflow (8D / A3)
**What**: `capa`, `capa_action`, `capa_non_conformance` (m:n) with phase-gated lifecycle.

**Design**:
- Tables per suggestion-1. Lifecycle `open→containment→root_cause→action_plan→implementation→verification→closed`. `methodology 8d|a3|pdca|dmaic`; `root_cause_method five_why|fishbone|fault_tree|pareto`.
- Transition rules: cannot advance to `verification` until all `corrective` actions completed; closing requires `effectiveness_verified=true`.
- Endpoints: `POST /v1/capas` (optionally from NCR ids), `POST /v1/capas/{id}/actions`, `POST /v1/capas/{id}/transition {to}`, `POST /v1/capas/{id}/close`.

**Testing**:
- `Unit: transition to verification with open corrective action → error`.
- `Integration: one CAPA linked to two NCRs (m:n) persists`.
- `Integration: close with effectiveness_verified false → 409`.

#### 4.3 — Webhook delivery for quality events
**What**: Outbound webhooks for ERP/MES on key events (a stated MVP integration requirement).

**Design**:
- `webhook_subscription(tenant_id, url, event_types[], secret, is_active)`. Event bus subscriber enqueues Celery `deliver_webhook` with HMAC-SHA256 signature header `X-QCI-Signature`; retries with exponential backoff (max 5), dead-letter on exhaustion.
- Event types: `inspection.completed`, `ncr.created`, `ncr.dispositioned`, `capa.closed`, `spc.violation`, `ppap.approved`.

**Testing**:
- `Integration (mock server): ncr.created → POST with valid HMAC received`.
- `Integration: 500 from endpoint → retried; after max → dead-lettered`.

### Definition of Done
4.x tests pass; NCR/CAPA migrations; webhook signing verified; CAPA state machine enforced; events flow end-to-end (SPC alarm → NCR → CAPA).

---

## Phase 5: Frontend PWA — Shop-Floor Capture, Offline Sync, Dashboards

### Purpose
Provide the mobile-first, offline-capable UI that operators and quality managers actually use: a form-builder UI, shop-floor inspection entry that works offline and syncs on reconnect, live SPC charts, NCR/CAPA management, and role-based KPI dashboards. This makes the platform usable end-to-end. Can be developed in parallel with Phase 6 once Phases 2–4 APIs exist.

### Tasks

#### 5.1 — App shell, auth, API client, PWA setup
**What**: Next.js PWA with login, token refresh, typed API client generated from OpenAPI, service worker.

**Design**:
- `openapi-typescript` generates types from `docs/openapi.json`; `lib/api` wraps fetch with auth + refresh. `manifest.json` + `sw.js` (Workbox) cache the app shell. Role-aware navigation reads JWT claims.

**Testing**:
- `Playwright: login → dashboard; refresh on 401 transparently retries`.
- `Playwright: offline reload → app shell still renders (SW cache)`.

#### 5.2 — Inspection form builder + shop-floor capture (offline)
**What**: Drag-and-drop plan builder (conditional logic, photo, pass/fail) and an operator entry screen optimised for minimal clicks, queuing results offline in IndexedDB.

**Design**:
- Builder edits `form_schema`/`logic` JSONB; preview uses the same `PlanEvaluator` logic (shared TS port) for live branching.
- Capture screen: Dexie store `pending_results`; each submit writes locally + attempts POST; a background-sync handler flushes the queue on `online`, attaching photos via presigned upload. Conflict policy: server is source of truth; client retries idempotently using a client-generated `result_uuid`.

**Testing**:
- `Playwright: build plan with show-if rule → preview hides/show field correctly`.
- `Playwright (offline): enter 3 results offline → queued; go online → all synced, badge clears`.
- `Playwright: photo captured offline → uploaded after reconnect`.

#### 5.3 — Live SPC charts
**What**: Operator/manager SPC view with control-limit + zone overlays, violation markers, live updates.

**Design**:
- uPlot chart subscribing to `WS /v1/ws/spc/{id}`; overlays UCL/LCL/center and shaded zones A/B/C; violation points highlighted with rule tooltip. Worst-to-best capability ranking list (Net-Inspect pattern) on the dashboard.

**Testing**:
- `Playwright: append subgroup via API → chart point appears live without reload`.
- `Playwright: out-of-control point rendered red with rule name in tooltip`.

#### 5.4 — NCR/CAPA UI + role-based KPI dashboards
**What**: NCR list/detail + disposition, CAPA board with phase columns, and dashboards (escape rate, CAPA closure rate, supplier defect rate).

**Design**:
- `dashboard.py` service exposes `GET /v1/dashboard/kpis?period=` computing: escape rate = rejected/inspected; CAPA closure rate = closed/opened in period; supplier defect rate = supplier-attributed NCRs / lots received; open-NCR aging buckets. CAPA board reflects the Phase-4 state machine. Role determines which widgets render.

**Testing**:
- `Integration: KPI endpoint returns correct escape rate for seeded data`.
- `Playwright: inspector role sees capture + SPC but not CAPA-approve actions`.

### Definition of Done
5.x tests pass; Playwright offline-sync scenario green; Lighthouse PWA installable; dashboards render real KPI data; frontend builds in CI.

---

## Phase 6: Supplier Quality — Portal, PPAP, FAI, Scorecards

### Purpose
Add the supplier-facing capabilities that distinguish full QMS platforms: a no-install supplier portal for PPAP requests/submissions, AS9102-compliant FAI report building, and supplier performance scorecards with automated risk tiers. Delivers the supplier-portal must-have plus v1.1 FAI/scorecard items. Parallelisable with Phase 5.

### Tasks

#### 6.1 — Supplier model + portal access
**What**: `supplier` table, supplier-role users scoped to a supplier, and a restricted portal API.

**Design**:
- Table per suggestion-1 (`risk_tier`, `quality_rating`, `certification_scope TEXT[]`). Supplier users get role `supplier` and a `supplier_id` claim; RBAC + a portal middleware constrain them to their own PPAP/FAI/inspection-share records.

**Testing**:
- `Integration: supplier user cannot list other suppliers' PPAPs → 403/empty`.

#### 6.2 — PPAP submission workflow (18 elements)
**What**: `ppap_submission` + `ppap_element` (AIAG 18-element breakdown), request→submit→review→approve lifecycle.

**Design**:
- Tables per suggestion-1. Requesting a PPAP at `ppap_level 1..5` seeds required `ppap_element` rows per AIAG level matrix. Supplier uploads attachments per element; reviewer approves/rejects per element; PSW (element 18) gates overall approval. Status per suggestion-1. Emits `ppap.approved` webhook.

**Testing**:
- `Unit: level 3 request → correct required-element set seeded`.
- `Integration: approve all elements + PSW → submission approved; webhook fired`.
- `Integration: reject one required element → submission cannot be approved`.

#### 6.3 — AS9102B FAI report builder
**What**: `fai_report` + `fai_characteristic_result` (Forms 1/2/3) with PDF export.

**Design**:
- Tables per suggestion-1; `form1_data JSONB` (part accountability), characteristic results mapped to Form 3 with `is_conforming`. `fai_service` validates every product characteristic is accounted for. WeasyPrint renders an AS9102B-styled PDF.
- (v1.1) Ballooning import endpoint accepting CMM/DISCUS/Capvidia exports to pre-populate characteristics (stub parser + CSV path now).

**Testing**:
- `Unit: characteristic missing a result → FAI not submittable`.
- `Integration: complete FAI → PDF generated, all balloons present`.

#### 6.4 — Supplier scorecards & risk tiering
**What**: Periodic `supplier_scorecard` rollups + automated `risk_tier` assignment.

**Design**:
- Celery `compute_scorecards(period)` aggregates: quality_score from incoming-inspection defect rate, delivery_score, responsiveness (CAPA response time), ppap_score; overall weighted. Rule-based risk tier: `critical` if overall<60 or any critical NCR; `high` <75; `standard` <90; `low` otherwise. (ML upgrade in Phase 8.)

**Testing**:
- `Unit: defect rate 12% → quality_score and tier computed per thresholds`.
- `Integration: run rollup → scorecard rows for period; supplier.risk_tier updated`.

### Definition of Done
6.x tests pass; PPAP/FAI/scorecard migrations; AS9102B PDF renders; supplier portal isolation verified; `ppap.approved` webhook delivered.

---

## Phase 7: Audits, Document Control, Equipment & Calibration

### Purpose
Round out the QMS with the remaining ISO 9001 modules: audit management (schedule→execute→close) with findings linked to CAPA, controlled documents with versioning and e-signed approvals, and equipment/calibration tracking (with OPC-UA node metadata for later automated capture). Completes the table-stakes feature set.

### Tasks

#### 7.1 — Audit management
**What**: `audit`, `audit_team_member`, `audit_finding` with clause-referenced findings linked to CAPA.

**Design**:
- Tables per suggestion-1 (status `planned→in_progress→completed→closed`; finding_type `major_nc|minor_nc|observation|opportunity|positive`; `clause_reference`). A finding can spawn/link a CAPA (`audit_finding.capa_id`). `standard` field drives a checklist template (ISO 9001 clauses 4–10) loaded from JSONB config.
- Endpoints: schedule, add team/findings, close (blocks if open major_nc findings without linked CAPA).

**Testing**:
- `Integration: close audit with open major_nc lacking CAPA → 409`.
- `Integration: finding → create linked CAPA → finding references it`.

#### 7.2 — Document control with e-signed approvals
**What**: `document` + `document_version` with controlled lifecycle and 21 CFR Part 11 signatures.

**Design**:
- Tables per suggestion-1 (status `draft→in_review→approved→effective→superseded→obsolete`; `retention_years`, `review_due_date`). Approval transition requires `esignature.sign(..., meaning="approved")`; effective version supersedes prior. Versions stored in object storage.

**Testing**:
- `Integration: approve without signature → 409; with signature → status effective, prior superseded`.
- `Integration: signed approval recorded in electronic_signature + audit_log`.

#### 7.3 — Equipment & calibration
**What**: `equipment` + `calibration_record`; calibration-due alerting; OPC-UA node metadata.

**Design**:
- Tables per suggestion-1 (`opcua_node_id`, `next_calibration_due`, `calibration_interval_days`; status incl. `due_for_calibration`, `out_of_calibration`). Daily Celery `flag_calibration_due` sets statuses and emits alerts. `as_found`/`as_left` stored JSONB.
- Guard: results recorded with out-of-calibration equipment are flagged on the inspection.

**Testing**:
- `Unit: next_due in past → status out_of_calibration`.
- `Integration: record calibration pass → next_due recalculated from interval`.

### Definition of Done
7.x tests pass; migrations for audit/document/equipment; e-signed document approval verified; calibration-due job runs; all table-stakes modules from features.md present.

---

## Phase 8: AI-Native Capabilities

### Purpose
Layer the differentiating intelligence onto the now-complete data foundation: AI inspection-plan generation, SPC interpretation, CAPA root-cause drafting, computer-vision defect classification, predictive supplier risk, and executive report narratives. These are the features no incumbent open-source tool offers and the project's core thesis.

### Tasks

#### 8.1 — LLM abstraction + AI inspection-plan generation
**What**: Provider-agnostic `llm.py`; generate inspection plans from drawing text / spec via OCR+LLM.

**Design**:
```python
class LLMProvider(Protocol):
    async def complete(self, system: str, user: str, *, json_schema: dict|None) -> dict|str: ...
```
- Providers: Anthropic, OpenAI, Ollama (local/air-gapped). Prompt template (jinja) instructs extraction of characteristics → JSON matching the `product_characteristic` + plan-item schema (nominal, USL/LSL, method, balloon). OCR (Tesseract) for image drawings; output is a *draft* plan a human approves.
- Endpoint: `POST /v1/ai/plans/generate {drawing_text|attachment_id}` → draft plan.

**Testing**:
- `Integration (mocked LLM): spec text → draft plan with characteristics matching golden output`.
- `Unit: malformed LLM JSON → repaired/validated against schema or error`.

#### 8.2 — AI SPC interpretation
**What**: Explain which Nelson/WECO rule fired and recommend specific corrective actions.

**Design**:
- On `spc.violation`, `spc_interpreter` builds a prompt with chart type, violated rule(s), recent trend, characteristic context, and similar historical violations; returns `{explanation, likely_causes[], recommended_actions[]}` stored on the violation and surfaced in the SPC UI/alert.

**Testing**:
- `Integration (mocked LLM): nelson_2 trend → interpretation references drift + actionable recommendation`.

#### 8.3 — CAPA root-cause assistant + NL 8D/A3 drafting
**What**: Given an NCR description, retrieve similar past CAPAs and draft a structured 8D/A3.

**Design**:
- Embed NCR/CAPA descriptions (provider embeddings) into a `pgvector` column; kNN retrieve top-k similar CAPAs. Prompt drafts root-cause hypotheses + populated 8D sections, pre-filling `capa.root_cause`, `corrective_action`, etc., as editable suggestions (never auto-final).
- Endpoint: `POST /v1/ai/capa/draft {ncr_id|description}`.

**Testing**:
- `Integration (mocked embeddings+LLM): description → draft with similar-CAPA citations and 8D fields populated`.
- `Unit: pgvector kNN returns nearest historical CAPA`.

#### 8.4 — Computer-vision defect classification
**What**: Classify defect type/severity from inspection photos via ONNX, written to `attachment.ai_classification`.

**Design**:
- On photo confirm, Celery `classify_defect(attachment_id)` runs an ONNX model (pluggable; IPC-A-610 class taxonomy as default labels for electronics). Output `{label, severity, confidence, bbox?}` → `ai_classification` JSONB; low-confidence flagged for human review. Edge mode: same ONNX model runnable in the OPC-UA bridge for inline inspection.

**Testing**:
- `Integration (stub ONNX): photo → ai_classification populated`.
- `Unit: confidence < threshold → needs_review True`.

#### 8.5 — Predictive supplier risk + executive narrative
**What**: ML model forecasting supplier non-conformance probability; AI monthly KPI narrative.

**Design**:
- `supplier_risk.py`: scikit-learn gradient-boosted classifier trained on scorecard history, incoming-inspection defect rates, PPAP status, delivery performance → `risk_probability` feeding `risk_tier` (overrides Phase-6 rules when a model exists). Retrain Celery job; model versioned in storage.
- `report_narrative.py`: `POST /v1/ai/reports/narrative {period}` → LLM exec summary from `dashboard.kpis`.

**Testing**:
- `Unit: trained on fixture history → predict() returns calibrated probability; ranking matches expected order`.
- `Integration (mocked LLM): period KPIs → narrative mentions top movers`.

### Definition of Done
8.x tests pass with mocked providers; `LLM_PROVIDER=ollama` path works offline; pgvector migration added; AI outputs are always human-reviewable drafts; no AI feature blocks core workflows if the provider is unavailable (graceful degradation).

---

## Phase 9: Integrations, Compliance Modes & Hardening

### Purpose
Make the platform deployable in real manufacturing environments: OPC-UA gauge/CMM ingestion, sampling-standard enforcement, compliance modes (IATF 16949 / AS9100 / FDA QMSR guided records), SSO (SAML), and production hardening (rate limits, backups, security, observability).

### Tasks

#### 9.1 — OPC-UA bridge agent
**What**: Standalone agent that reads gauge/CMM values via OPC-UA and posts them as inspection results.

**Design**:
- `opcua_bridge.py` using `asyncua`: subscribes to nodes from `equipment.opcua_node_id`, maps readings to `(inspection_plan_item, characteristic)`, buffers offline, and POSTs to the inspection-results API with the result `uuid` idempotency key. Ships as a separate Docker image for on-prem deployment near the shop floor.

**Testing**:
- `Integration (asyncua test server): node value change → result posted with correct characteristic mapping`.
- `Integration: API unreachable → buffered, flushed on recovery`.

#### 9.2 — Sampling standards (ISO 2859-1 / 3951 / Z1.4-Z1.9)
**What**: `sampling.py` computing sample size and accept/reject numbers from AQL + inspection level + lot size.

**Design**:
- Encode AQL master tables (code letters by lot size + level; Ac/Re by code + AQL). Inspection plan with `sampling_standard` auto-derives `sample_size`; inspection enforces accept/reject against Ac/Re. Switching rules (normal/tightened/reduced) tracked per product-supplier.

**Testing**:
- `Unit: lot 500, level GII, AQL 1.0 → code letter + sample size + Ac/Re per ISO 2859-1 tables` (fixture).
- `Unit: rejects > Ac → lot rejected`.

#### 9.3 — Compliance modes & guided records
**What**: Tenant compliance profile (ISO 9001 / IATF 16949 / AS9100 / FDA QMSR) driving required-record recommendations and audit templates.

**Design**:
- `compliance_profile` config (JSONB) maps certification scope → required modules/records (e.g., IATF → PPAP+APQP+MSA+SPC mandatory; FDA QMSR → e-sig + design history + complaint handling). A `GET /v1/compliance/recommendations` returns gaps given current tenant data. Audit checklist templates per standard loaded from versioned JSON. Anticipates ISO 9001:2026 transition fields per standards.md note.

**Testing**:
- `Integration: IATF profile with no MSA studies → recommendation lists MSA gap`.
- `Unit: FDA QMSR profile requires e-signature on document approvals`.

#### 9.4 — SAML SSO, rate limiting, security & observability
**What**: Enterprise SAML, API rate limits, security headers, structured logs/metrics/traces, backup guidance.

**Design**:
- SAML 2.0 (python3-saml) ACS/metadata endpoints mapping assertions→users (matches standards.md). Redis token-bucket rate limiting per tenant/IP. Security: CORS allowlist, CSP, HSTS, argon2, JWT rotation, secrets via env. OpenTelemetry traces; Prometheus `/metrics`; Sentry hook. Document pg backup + object-storage lifecycle.

**Testing**:
- `Integration: SAML assertion → user authenticated, roles mapped`.
- `Integration: exceed rate limit → 429 with Retry-After`.
- `Security: cross-tenant API call rejected; audit_log immutable (no update/delete route)`.

### Definition of Done
9.x tests pass; OPC-UA bridge image builds and ingests against a test server; sampling math validated against published tables; SAML login works; rate limiting + security headers verified; metrics exposed; full `docker-compose up` brings up the complete stack and a smoke e2e (login → plan → inspect → SPC → NCR → CAPA → PPAP) passes.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (tenancy, auth, RBAC, audit)         ─── required by all
    │
Phase 2: Product & Inspection engine                     ─── requires 1
    │
Phase 3: SPC engine                                       ─── requires 2
    │
Phase 4: NCR & CAPA + webhooks                            ─── requires 2 (SPC link from 3)
    ├── Phase 5: Frontend PWA (capture, SPC, dashboards)  ─── requires 2,3,4 · parallel with 6
    └── Phase 6: Supplier quality (portal, PPAP, FAI)     ─── requires 2,4 · parallel with 5
         │
Phase 7: Audits, documents, equipment/calibration        ─── requires 1,4 (can start after 4)
    │
Phase 8: AI-native capabilities                           ─── requires 2,3,4,6 (data foundation)
    │
Phase 9: Integrations, compliance, hardening              ─── requires all; final hardening
```

**Parallelism opportunities:**
- Phases **5 and 6** can be built concurrently once Phases 2–4 land.
- **Phase 7** can begin in parallel with 5/6 once Phase 4 is complete (it only needs the foundation + CAPA linkage).
- Within Phase 8, tasks 8.1–8.5 are largely independent once 8.1 (LLM abstraction) exists; 8.4 (CV) and 8.5 (ML risk) can proceed in parallel with the LLM tasks.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented as specified.
2. All unit and integration tests pass (`pytest`), with mocked external providers; real-dependency tests (Postgres/Redis/MinIO) pass via testcontainers.
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes for `src/qci`.
5. `docker compose build` succeeds for affected images (api, worker, web, opcua-bridge as applicable).
6. The phase's feature works end-to-end (manual or e2e check described in the phase DoD).
7. New configuration options documented in README/`.env.example`.
8. New API endpoints appear in the auto-generated `docs/openapi.json` (CI exports and diffs it).
9. Alembic migration(s) created, and `alembic upgrade head` / `downgrade` round-trips cleanly.
10. New domain events registered and, where relevant, covered by webhook/websocket tests.
11. RLS/tenant isolation verified for any new tenant-scoped tables.
12. For compliance-touching phases (1, 4, 7, 9): audit-trail and (where applicable) e-signature behaviour verified.
