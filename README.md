# Quality Control & Inspection

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source quality management platform combining digital inspection, real-time SPC, supplier quality, and CAPA workflows for discrete manufacturers.

Quality Control & Inspection is a candidate open-source project providing inspection checklists, defect tracking, SPC charts, and supplier quality management. It targets quality engineers, supplier quality managers, and production supervisors at discrete manufacturers who need enterprise-grade quality capabilities without enterprise-grade pricing.

---

## Why Quality Control & Inspection?

- Enterprise QMS platforms (MasterControl, ETQ Reliance, Siemens Opcenter) start between USD 500 and USD 800 per month and scale to tens of thousands monthly, putting them out of reach for most SMB manufacturers.
- Accessible inspection tools like SafetyCulture lack the depth needed for full QMS workflows: no document control, no supplier quality, no CAPA workflow, and no real SPC capability.
- Specialist SPC platforms (InfinityQS, Net-Inspect) deliver excellent statistical process control but are not full QMS systems and are priced for large enterprises.
- Existing open-source QMS projects (OpenQMS.net, Open EQMS) are minimal and lack production-ready SPC, PPAP, and supplier portal features.
- Computer vision defect detection is generally treated as a separate system from the QMS, forcing manufacturers to integrate disparate tools rather than work in one platform.

---

## Key Features

### Digital Inspection & Data Capture

- Drag-and-drop inspection form builder with conditional logic, photo capture, and pass/fail scoring
- Mobile-first offline data collection with automatic sync on reconnect
- Ballooned-drawing import for FAI/PPAP measurement plan setup
- Operator-facing shop floor entry screens designed for minimal clicks per measurement

### Statistical Process Control

- Real-time control charts (I-MR, X-bar R, X-bar S, P-chart, U-chart) with Nelson and WECO rule alerting
- Process capability analysis (Cp, Cpk, Pp, Ppk) calculated in real time
- Gage R&R and measurement system analysis (MSA) tooling
- Role-based alerts and escalation workflows for out-of-control conditions

### Non-Conformance, CAPA & Audits

- Non-conformance record creation with root cause analysis and corrective action linkage
- 8D and A3 problem-solving workflow templates
- ISO 9001-aligned audit management from scheduling through execution to closure
- Document control with version history, electronic approvals, and controlled distribution

### Supplier Quality

- Supplier portal for PPAP request workflow, submission tracking, and approval status
- AS9102-compliant First Article Inspection (FAI) report builder
- Supplier performance scorecards with automated risk-tier assignment
- Real-time incoming inspection data sharing between customer and supplier

### Compliance & Reporting

- ISO 9001, IATF 16949, AS9100, ISO 13485 compliance frameworks
- Guided record recommendations based on certification scope
- Role-based dashboards: escape rate, CAPA closure rate, supplier defect rate
- Export to PDF, CSV, and Excel for reporting

---

## AI-Native Advantage

AI is integrated directly into quality workflows rather than bolted on as a separate system. Computer vision classifies defects from operator-captured photos and supports edge-deployed inline inspection. AI-driven SPC interpretation identifies subtle process drift and recommends specific corrective actions before defects occur. Natural language CAPA drafting turns plain-text escape descriptions into structured 8D or A3 documents pre-populated with relevant historical defect data. Supplier risk scoring uses delivery performance, incoming inspection history, and PPAP status to predict non-conformance probability.

---

## Tech Stack & Deployment

Expected deployment modes include self-hosted and cloud variants. The platform is designed around open APIs (REST with webhook support) for integration with ERP, MES, PLM, and LIMS systems. Standards alignment includes ISO 9001:2015, IATF 16949:2016, AS9100D, ISO/IEC 17025, the AIAG SPC Reference Manual, and 21 CFR Part 820 / ISO 13485 for regulated environments. Automated gauge and CMM data collection is supported via OPC-UA.

---

## Market Context

The global quality management software market is estimated at approximately USD 14–18 billion in 2026, with the manufacturing-specific quality inspection and SPC segment growing at over 10% CAGR (research.md). Mid-market QMS platforms start at USD 500–750/month and enterprise platforms at USD 800/month and up. Primary buyers include Quality Managers and Directors at discrete manufacturers (automotive, aerospace, electronics, medical device), Supplier Quality Engineers, Regulatory Affairs managers, and ISO compliance leads.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
