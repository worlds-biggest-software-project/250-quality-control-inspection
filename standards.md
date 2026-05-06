# Standards & API Reference

> Project: Quality Control & Inspection · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 9001:2015 (revision to ISO 9001:2026 in progress)**
- **URL:** https://www.iso.org/standard/62085.html
- Foundation quality management system standard covering leadership, risk-based thinking, process approach, and continual improvement. Baseline compliance requirement for virtually all manufacturing quality software. A revised edition (ISO 9001:2026) is expected September–November 2026, focusing on sustainability, knowledge management, and enhanced risk; a 3-year transition period will follow publication.

**ISO 13485:2016 — Medical devices: Quality management systems**
- **URL:** https://www.iso.org/standard/59752.html
- QMS standard specific to medical device design, development, and manufacturing. Mandates design history files, complaint handling, CAPA processes, and audit trails. The FDA has aligned its QMSR (Quality Management System Regulation) to this standard with full enforcement from February 2026, replacing the legacy 21 CFR Part 820.

**IATF 16949:2016 — Automotive QMS**
- **URL:** https://www.iatfglobaloversight.org/
- Automotive-specific QMS standard building on ISO 9001, mandating APQP, PPAP, FMEA, MSA, and SPC (AIAG Core Tools) in automotive supply chain software. A revised IATF 16949:2027 is anticipated to follow ISO 9001:2026 by 12–18 months.

**AS9100D / IA9100 (formerly AS9100D) — Aerospace & Defence QMS**
- **URL:** https://www.sae.org/standards/content/as9100d/
- QMS standard for aerospace and defence manufacturers mandating rigorous inspection records, product/process traceability, first-article inspection, and configuration management. Currently rebranding from "AS" to "IA" series (IA9100, IA9110, IA9120), estimated 2026–2027 release.

**ISO/IEC 17025:2017 — Competence of testing and calibration laboratories**
- **URL:** https://www.iso.org/standard/66912.html
- Specifies requirements for laboratory competence, impartiality, and consistent operation. Directly relevant to metrology and calibration management modules in quality inspection software, including equipment calibration records, uncertainty of measurement, and laboratory audit trails.

**ISO 2859-1:1999 — Sampling procedures for inspection by attributes (AQL)**
- **URL:** https://www.iso.org/standard/1141.html
- Defines acceptance sampling plans using the Acceptable Quality Level (AQL) concept for lot-by-lot inspection. Specifies single, double, and multiple sampling plans across three general inspection levels (GI, GII, GIII) and four special levels (S1–S4). Widely implemented in receiving inspection and supplier quality modules.

**ISO 3951 — Sampling procedures for inspection by variables**
- **URL:** https://www.iso.org/standard/57490.html
- Companion to ISO 2859 for variable (measured) data sampling plans, providing procedures for determining whether process average meets an acceptable quality level based on measured characteristics rather than attribute counts.

**ISO/TS 16949 (superseded by IATF 16949) / AIAG Core Tools**
- **URL:** https://www.aiag.org/expertise-areas/quality/quality-core-tools
- The AIAG Quality Core Tools suite (APQP, CP, PPAP, FMEA, MSA, SPC manuals) defines the data models and workflows implemented in automotive quality management systems. The AIAG & VDA FMEA Handbook is the current joint reference for failure mode analysis in automotive supply chains.

---

### W3C & IETF Standards

**OASIS OSLC Quality Management Version 2.1**
- **URL:** https://docs.oasis-open-projects.org/oslc-op/qm/v2.1/os/quality-management-spec.html
- RESTful web services specification defining a standardised interface for managing quality management domain artefacts: test plans, test cases, test execution records, and test results. Applies Linked Data principles (W3C LDP) to enable interoperability between quality tools, ALM platforms, and requirements management systems without duplicating data. Relevant for traceability chains linking requirements, test results, non-conformances, and corrective actions.

**W3C Linked Data Platform (LDP) 1.0**
- **URL:** https://www.w3.org/TR/ldp/
- Specifies HTTP-based patterns for reading and writing Linked Data resources and containers. Underpins OASIS OSLC specifications including OSLC Quality Management; enables audit-grade traceability chains across distributed quality systems.

**RFC 7230–7235 — HTTP/1.1**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7230
- Core HTTP protocol RFCs governing REST API communication. All quality management REST APIs are expected to conform to these specifications for request/response semantics, authentication, and caching behaviour.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- **URL:** https://datatracker.ietf.org/doc/html/rfc6749
- Standard for delegated API authorisation. All major QMS platforms (ETQ, MasterControl, SafetyCulture) use OAuth 2.0 for API access control and third-party integrations.

**RFC 7519 — JSON Web Token (JWT)**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7519
- Standard for compact, URL-safe claim assertions used as bearer tokens in OAuth 2.0 API authentication flows across quality management APIs.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.1**
- **URL:** https://spec.openapis.org/oas/v3.1.1.html
- Standard for describing REST APIs in a programming language–agnostic format. OAS 3.1 is fully aligned with JSON Schema Draft 2020-12. SafetyCulture, MasterControl, and ETQ Reliance publish OpenAPI-compatible API descriptions for their REST endpoints.

**JSON Schema Draft 2020-12**
- **URL:** https://json-schema.org/draft/2020-12/schema
- Standard for validating and documenting JSON data structures. Used in conjunction with OpenAPI to define request/response payloads for quality event records (non-conformances, CAPA, inspection results, SPC measurements).

**OPC Unified Architecture (OPC-UA) — IEC 62541**
- **URL:** https://opcfoundation.org/about/opc-technologies/opc-ua/
- Industry-standard machine-to-machine communication protocol for industrial automation. Used by InfinityQS Enact, Siemens Opcenter, and Plex to collect measurement data directly from gauges, CMMs, and production equipment without manual entry. Defines a secure, platform-independent data exchange model for shop-floor quality data.

**ANSI/ASQ Z1.4 & Z1.9 — Sampling Inspection Standards**
- **URL:** https://asq.org/quality-resources/z14-z19
- US equivalents to ISO 2859 and ISO 3951 respectively. Z1.4 covers attribute sampling; Z1.9 covers variable sampling. Widely referenced in US government and defence quality programmes alongside the ISO series.

---

### Security & Authentication Standards

**SAML 2.0 — Security Assertion Markup Language**
- **URL:** https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- XML-based standard for exchanging authentication and authorisation data between identity providers and service providers. Used by ETQ Reliance, MasterControl, and Intelex for enterprise SSO integration with corporate identity platforms (Azure AD, Okta, Ping).

**OpenID Connect 1.0 (built on OAuth 2.0)**
- **URL:** https://openid.net/connect/
- Identity layer on top of OAuth 2.0 providing standardised user authentication for QMS API consumers. Increasingly adopted alongside SAML for modern cloud QMS deployments.

**21 CFR Part 11 — Electronic Records and Electronic Signatures (FDA)**
- **URL:** https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11
- FDA regulation defining requirements for electronic records and signatures in regulated industries (pharma, biotech, medical device). Mandates time-stamped audit trails, identity verification, and non-repudiation — directly implemented by MasterControl and QT9 document control and approval modules.

**FDA Quality Management System Regulation (QMSR) — 21 CFR Part 820 (2024 revision)**
- **URL:** https://www.federalregister.gov/documents/2024/02/02/2024-01647/quality-management-system-regulation
- Replaces legacy QSR with requirements harmonised to ISO 13485:2016. Full enforcement from February 2026. Quality software targeting medical device manufacturers must align document control, CAPA, and complaint handling workflows to this updated regulation.

**ISO 27001:2022 — Information Security Management**
- **URL:** https://www.iso.org/standard/27001
- International standard for information security management systems. SafetyCulture is ISO 27001 certified; all enterprise QMS platforms (ETQ, MasterControl, Intelex) cite ISO 27001 or SOC 2 Type II certification as baseline security credentials.

**SOC 2 Type II — Service Organisation Control**
- **URL:** https://www.aicpa-cima.com/resources/landing/soc-2
- AICPA audit standard verifying that a SaaS provider's security, availability, processing integrity, confidentiality, and privacy controls operate effectively over a defined period. SafetyCulture, ETQ, and MasterControl publish SOC 2 Type II reports.

---

### Domain-Specific Standards

**IPC-A-610J:2024 — Acceptability of Electronic Assemblies**
- **URL:** https://www.ipc.org/ipc-610
- The globally dominant visual acceptance standard for inspecting finished electronic assemblies (PCB soldering, component placement, cleanliness, marking). Defines three quality classes (Class 1: general, Class 2: dedicated service, Class 3: high-performance) with progressively stricter acceptance criteria. Version J (2024) is the current release. Relevant for electronics manufacturing quality inspection software targeting PCB assembly.

**AS9102B — First Article Inspection Requirements**
- **URL:** https://www.sae.org/standards/content/as9102b/
- Aerospace standard defining documentation requirements for first article inspection (FAI) reports. 1factory and Net-Inspect implement AS9102-compliant FAI report builders. Specifies design characteristic accountability, part number traceability, and measurement data requirements.

**AIAG MSA 4th Edition — Measurement System Analysis**
- **URL:** https://www.aiag.org/training-and-resources/manuals/details/MSA-4
- Defines methodology for Gage R&R studies, linearity, bias, and stability analysis to validate that measurement systems are fit for purpose. Implemented in InfinityQS Enact, 1factory, and Plex QMS for gauge validation workflows.

**AIAG SPC 2nd Edition — Statistical Process Control Reference Manual**
- **URL:** https://www.aiag.org/training-and-resources/manuals/details/SPC-4
- Defines control chart types (X-bar R, X-bar S, I-MR, P-chart, C-chart, U-chart), Nelson and Western Electric (WECO) out-of-control rules, and process capability indices (Cp, Cpk, Pp, Ppk). The data models and calculations implemented in all SPC modules reference this manual.

---

## Similar Products — Developer Documentation & APIs

### SafetyCulture (iAuditor)

- **Description:** Mobile-first digital inspection and audit platform enabling organisations to digitise paper checklists and conduct quality, safety, and compliance inspections.
- **API Documentation:** https://developer.safetyculture.com/reference/introduction
- **Developer Portal:** https://developer.safetyculture.com/
- **SDKs/Libraries:** No official SDK; REST API with standard HTTP client libraries; Pipedream and n8n connectors available.
- **Developer Guide:** https://developer.safetyculture.com/ (getting started guides for inspections, sites, assets, and webhook subscriptions)
- **Standards:** REST/JSON; webhooks for event-driven integrations; OpenAPI-compatible documentation.
- **Authentication:** OAuth 2.0 bearer tokens; Premium or Enterprise plan required for API access.

---

### MasterControl Quality Excellence

- **Description:** Regulated-industry QMS covering document control, CAPA, audits, training, and change management for FDA and ISO-regulated manufacturers.
- **API Documentation:** https://currentcloud.onlinehelp.mastercontrol.com/2024.1/en_us/Content/Appendix/Access_and_Use_MasterControl_APIs.htm
- **API Toolkit Overview:** https://www.mastercontrol.com/solutions/toolkit/
- **Getting Started Guide:** https://www.mastercontrol.com/resource-center/documents/mastercontrol-api-toolkit-getting-started-guide/
- **SDKs/Libraries:** No public SDK; REST API + legacy Web Service API layer; MuleSoft connectors for enterprise iPaaS.
- **Standards:** RESTful HTTP/JSON; Web Service API layer for non-REST operations; OpenAPI format not publicly confirmed.
- **Authentication:** API key + OAuth 2.0 bearer token; full documentation behind login.

---

### ETQ Reliance (Hexagon)

- **Description:** Workflow-based enterprise eQMS with 40+ configurable applications for document control, CAPA, supplier quality, audit, and risk management across manufacturing and life sciences.
- **API Documentation:** https://www.etq.com/product-overview/ (integration overview); detailed documentation via Hexagon Nexus portal — https://nexus.hexagon.com/home/product/etq-reliance-nxg/
- **SDKs/Libraries:** No public SDK; REST API with OAuth 2.0; Alumio and MuleSoft iPaaS connectors.
- **Developer Guide:** https://www.etq.com/blog/establishing-qms-enterprise-integrations/
- **Standards:** Secure REST APIs; SAML 2.0 and OAuth 2.0 for authentication and SSO; AWS-hosted infrastructure.
- **Authentication:** OAuth 2.0 / SAML 2.0 enterprise SSO.

---

### InfinityQS Enact

- **Description:** Cloud SPC platform centralising quality data from multiple manufacturing facilities for real-time process monitoring, capability analysis, and enterprise quality intelligence.
- **API Documentation:** https://enacthelp.infinityqs.com/ (Enact Online Help; API details within configuration section)
- **SDKs/Libraries:** No public SDK; Enterprise Integration Service (EIS) and DCS API for data collection; OPC-UA interfaces for gauge and CMM connectivity.
- **Developer Guide:** https://enacthelp.infinityqs.com/en-us/Configuration/Documentation.htm
- **Standards:** OPC-UA for machine data; REST/JSON for application integration; internal DCS API for SPC data streaming.
- **Authentication:** Role-based access control; SSO available for enterprise deployments.

---

### Net-Inspect

- **Description:** Web-based aerospace and defence supplier quality management platform with patented real-time SPC capability scoring, FAI/PPAP workflows, and FedRAMP-equivalent cloud compliance.
- **API Documentation:** Available via vendor engagement; REST API and webhook capabilities confirmed.
- **SDKs/Libraries:** No public SDK; REST API + webhooks for third-party system integration; direct CMM output file import.
- **Standards:** REST/JSON; Microsoft Azure Government cloud (FedRAMP-equivalent); AS9102B-compliant FAI documentation.
- **Authentication:** Role-based access control; Azure AD integration available.

---

### 1factory

- **Description:** Cloud QMS for discrete manufacturing covering inspection planning, SPC, PPAP/FAI, supplier quality, and CAPA with real-time supplier data sharing.
- **API Documentation:** Available via vendor engagement.
- **SDKs/Libraries:** No public SDK; REST API; direct import from DISCUS and Capvidia ballooning tools; CMM output file import.
- **Developer Guide:** https://www.1factory.com/qms.html (product overview); technical API details via vendor engagement.
- **Standards:** REST/JSON; AS9102B-compliant FAI; IATF 16949–aligned PPAP/APQP toolset.
- **Authentication:** Role-based access; SSO available on enterprise plans.

---

### Siemens Opcenter Quality (Opcenter X Quality)

- **Description:** Enterprise digital inspection, SPC, and complaint management platform for manufacturers, with deep PLM integration via Siemens Teamcenter and native MES connectivity.
- **API Documentation:** Available via Siemens Support Center (login required); https://plm.sw.siemens.com/en-US/opcenter/quality/
- **SDKs/Libraries:** No public SDK; REST APIs; OPC-UA for automated gauge data; Xcelerator platform integration APIs.
- **Developer Guide:** Siemens Xcelerator Developer Hub (partner/customer access required).
- **Standards:** OPC-UA; REST/JSON; SAP integration adapters; ISO 9001/IATF 16949/AS9100 workflow templates.
- **Authentication:** SAML 2.0 SSO; role-based access control integrated with Siemens Xcelerator identity.

---

### Plex QMS (Rockwell Automation)

- **Description:** Automotive ERP-native QMS providing in-process SPC, supplier quality management, IATF 16949 toolset, and quality-production integration within the Plex cloud platform.
- **API Documentation:** https://plex.rockwellautomation.com/en-us/products/quality-management-system.html (product overview); REST API documentation via Rockwell Automation partner portal.
- **SDKs/Libraries:** No public SDK; REST API; OPC-UA and machine interfaces for automated data collection; EDI connectors.
- **Standards:** REST/JSON; OPC-UA; IATF 16949 and ISO 9001 workflow support; EDI (ANSI X12 / EDIFACT) for automotive B2B.
- **Authentication:** OAuth 2.0; SAML 2.0 SSO for enterprise deployments.

---

## Notes

- **OASIS OSLC QM as an interoperability opportunity:** The OSLC Quality Management 2.1 standard provides a vendor-neutral REST interface for quality artefacts. No major commercial QMS platform fully implements OSLC QM, representing an opportunity for an AI-native tool to build on open standards for cross-tool traceability rather than relying on proprietary integration points.

- **OPC-UA gap in cloud SPC:** While OPC-UA is the de facto standard for shop-floor machine communication, most cloud SPC platforms require a gateway agent to translate OPC-UA data to their proprietary ingest API. An open-source bridge or native OPC-UA adapter would reduce integration friction significantly.

- **ISO 9001:2026 transition:** The forthcoming revision introduces updated requirements around knowledge management, sustainability, and digital readiness. Quality software built now should anticipate these changes in its compliance mapping and workflow templates, giving early adopters a certification-readiness advantage.

- **FDA QMSR enforcement (February 2026):** Medical device QMS software must now align to ISO 13485:2016 requirements rather than the legacy 21 CFR Part 820 structure. Tools built for the FDA market need to map their data models and workflows to the updated QMSR structure to remain compliant.

- **IPC-A-610 in AI visual inspection:** The IPC-A-610 Class 1/2/3 acceptance criteria are a natural training target for computer vision defect classification models in electronics manufacturing. Annotated inspection datasets aligned to IPC-A-610 criteria would enable training of AI models with industry-standard acceptance thresholds.
