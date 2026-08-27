# ADR-0003: External ASV-Certified Scanning Approach

## Status

Accepted

## Context

Meridian is pivoting from purely internal vulnerability management (SOC 2 / ISO 27001, see [Charter §2](../00-charter-and-business-case.md#2-why-this-is-a-business-problem-contextual-layer)) into offering Vulnerability-Management-as-a-Service to external clients pursuing PCI DSS compliance (see [Charter §2.1](../00-charter-and-business-case.md#21-new-business-driver-pci-dss-as-a-service-added-2026-08-23)).

PCI DSS Requirement 11.3.2 mandates that external-facing scans of the cardholder data environment (CDE) be performed by, or through, a PCI SSC–certified **Approved Scanning Vendor (ASV)**, and that scan reports carry ASV attestation a client's QSA can rely on as compliance evidence. Meridian's existing internal scanning stack — Tenable Nessus, authenticated agent + agentless (ADR-0002) — is not itself ASV-certified. An unaffiliated authenticated Nessus scan carries no PCI SSC standing for the external CDE-scanning portion of this new service line, regardless of scan quality.

## Business Attributes

| Attribute | Relevance |
|---|---|
| **ASV-certifiable** | Core driver — client QSAs will only accept ASV-attested scan evidence for external CDE scans |
| **Compliant** | Must map cleanly onto PCI DSS Req 11.3.2 evidence language |
| **Sustainable** | Must not require Meridian to carry certification/audit overhead disproportionate to the size of this new service line |
| **Timely** | Slow accreditation timelines would delay or block onboarding the first PCI-as-a-service client |

## Decision Drivers

- **Time-to-first-client** — how fast Meridian can start selling this service
- **Certification overhead** — PCI SSC ASV status requires annual re-certification plus quarterly scan-quality audits and specific infrastructure/process requirements
- **Credibility with client QSAs**, who will scrutinize scan provenance directly
- **Organizational fit** — Meridian is a BPO/ITES firm; scanning-tool vendorship is not its core business

## Options Considered

### Option A — Become an ASV

Meridian itself pursues PCI SSC ASV certification.

- **Pros:** full control of the scanning pipeline and roadmap; no revenue share to a third party; strongest long-term differentiation if VM-as-a-Service grows into a serious business line
- **Cons:** a 6–12 month certification runway before the first client can be onboarded, directly conflicting with capturing pipeline now; an ongoing PCI SSC compliance program (quarterly scan validation, annual recertification) becomes a standing operational cost unrelated to Meridian's core business; requires dedicated compliance/scanning staff Meridian does not currently have

### Option B — Partner With an Existing ASV

Contract an already-certified ASV to perform the external CDE scans; Meridian's platform orchestrates scheduling, ingests ASV results, and layers its own scoring/reporting/remediation workflow on top (see [Architecture & Process §1](../01-architecture-and-process.md#1-logical-layer-program-architecture)).

- **Pros:** fastest time-to-market — no certification runway; the ASV relationship is a commercial contract, not an operational build; Meridian stays focused on its actual differentiator (orchestration, prioritization, remediation workflow, client reporting), not scanner certification
- **Cons:** ongoing revenue share / per-scan cost to the ASV partner; less control over ASV scan scheduling and report format, requiring API/file-based integration; the client-facing story is slightly more complex ("Meridian orchestrates; Partner X is the ASV of record")

### Option C — Resell an ASV-Certified Tool Under an MSSP Model

License an ASV-certified scanning platform (e.g., a vendor offering ASV scanning as an API/white-label product) and resell/operate it under Meridian's existing MSSP relationship with the client.

- **Pros:** faster than Option A; tighter platform integration than Option B, since the tool is embedded rather than a separate vendor relationship; can be white-labeled into Meridian's existing client reporting
- **Cons:** still dependent on a third party's ASV status and product roadmap; licensing cost; requires due diligence to confirm the vendor's ASV certification actually covers the scan types needed (external CDE scanning specifically, not general vulnerability scanning)

## Decision

Adopt **Option B — partner with an existing ASV** for the initial service launch, with **Option C** (a white-labeled ASV tool) as an explicit 12–18 month evaluation checkpoint once client volume justifies tighter integration.

The same urgency pattern that shaped the internal program's tooling decisions (ADR-0001, ADR-0002) applies commercially here: Meridian needs to onboard its first PCI-as-a-service client without a 6–12 month certification runway, and ASV certification is not Meridian's core competency. Partnering lets the existing scoring engine, CMDB, and reporting pipeline — already built for the internal program — absorb ASV scan results as just another authenticated input, consistent with the Physical-vs-Logical separation established in ADR-0001.

## Consequences

### Positive

- No certification runway blocking first-client onboarding
- Meridian's differentiation stays on orchestration, prioritization, and reporting — the parts of the pipeline it already operates
- ASV partner risk is commercial/contractual, not an internal compliance program Meridian has to run

### Negative

- Ongoing per-scan or revenue-share cost tied to the ASV partner
- Client-facing materials must clearly attribute ASV-of-record status to the partner, not Meridian, to avoid misrepresenting certification status to a QSA
- Scan scheduling and format are constrained by the partner's API/integration capabilities

## Follow-ups

- Select and contract an ASV partner (due diligence: PCI SSC ASV list, API availability, scan turnaround SLA)
- Define the ingestion contract between the ASV partner's scan output and the Composite Scoring Engine (ADR-0001)
- Revisit Option C (white-label ASV tool) once client count/volume is known — track as a 12–18 month decision checkpoint
- Confirm with legal/contracts whether Meridian's MSSP agreement language needs to disclose the ASV partner relationship to clients
