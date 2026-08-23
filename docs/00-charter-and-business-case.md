# Vulnerability Management Program — Charter & Business Case

**Organization:** Sreelaxmi Global Services (fictional BPO/ITES company)
**Author:** Security Architecture
**Status:** Draft for CISO / Exec sign-off
**Date:** 2026-08-23

---

## 1. Company Context

Sreelaxmi Global Services is a mid-size BPO/ITES provider (~8,000 employees) with delivery centers in Hyderabad, Pune, and Manila, serving clients in BFSI, healthcare, and retail verticals. Infrastructure is a hybrid mix:

- Legacy on-prem AD domain, ~3,000 Windows endpoints (agent desktops), ~400 Linux/Windows servers
- Growing Azure footprint (landing zone stood up 18 months ago) hosting a client-facing portal on AKS
- WFH agents connecting via VPN (post-pandemic hybrid workforce, ~35% of headcount)
- No centralized asset inventory / CMDB — asset lists live in spreadsheets per delivery center
- No formal vulnerability management program: patching is ad hoc, driven by Patch Tuesday and occasional client audit findings

## 2. Why This Is a Business Problem (Contextual Layer)

Three client contracts (two BFSI, one healthcare) contractually require evidence of a functioning vulnerability management program as part of SOC 2 Type II and ISO 27001 surveillance audits. A recent client security questionnaire flagged Sreelaxmi as **non-compliant** on this control, placing a $4.2M annual contract under a 90-day remediation clause.

This reframes the problem from "we should patch more" to a business-critical, time-boxed compliance exposure with named financial consequence. That is the lens every downstream architecture decision must be traceable back to.

### Stakeholders

| Stakeholder | Interest |
|---|---|
| CEO / Board | Contract retention, audit posture, reputational risk |
| CISO | Owns the control, answers to auditors and clients |
| Delivery Center Ops Heads (Hyd/Pune/Manila) | Operational disruption from patching, SLA impact |
| Client Security teams | Evidence of control operation (reports, SLAs, closure rates) |
| IT Infrastructure team | Actually executes remediation |
| Employees/Agents | Endpoint performance impact from scanning/patch agents |
| Client QSAs (Qualified Security Assessors) *(added 2026-08-23 — PCI-DSS-as-a-Service pivot, see §2.1)* | Rely on Sreelaxmi's scan/reporting output as PCI DSS compliance evidence for the client they're assessing |
| Client CDE (Cardholder Data Environment) Owners *(added 2026-08-23 — PCI-DSS-as-a-Service pivot, see §2.1)* | Own the in-scope cardholder data environment being scanned; accountable to their own QSA for the accuracy and completeness of Sreelaxmi's evidence |

### 2.1 New Business Driver: PCI-DSS-as-a-Service (Added 2026-08-23)

Sreelaxmi is expanding beyond running vulnerability management for its own internal compliance (SOC 2 / ISO 27001, §2 above) into **offering Vulnerability-Management-as-a-Service to external clients pursuing PCI DSS compliance**. In this model, Sreelaxmi becomes a service provider whose scanning and reporting output a client's own QSA will rely on directly as PCI DSS compliance evidence — not just internal hygiene, but third-party-relied-upon attestation.

This is **additive** to the internal program described above, not a replacement: the same underlying platform (CMDB, scanner, scoring engine, reporting pipeline) now serves two distinct audiences — Sreelaxmi's own auditors (internal program) and external clients' QSAs (new service line) — and must keep those two evidence trails cleanly separated.

The business case mirrors the internal one in shape but not in mechanism: instead of a contractual remediation clause forcing internal compliance, this is a **new revenue opportunity** gated by Sreelaxmi's ability to produce evidence a QSA will actually accept — which is a materially stricter bar than internal audit evidence (see ADR-0003, ADR-0004).

## 3. Business Attributes (Conceptual Layer — SABSA)

Before any tool or process is chosen, we name what "good" looks like, so every later decision (tool, cadence, scoring model) is traceable back to intent rather than justified after the fact.

| Attribute | Meaning here |
|---|---|
| **Compliant** | Satisfies SOC 2 / ISO 27001 vulnerability management control language, produces auditor-ready evidence |
| **Timely** | Remediation SLAs met, especially for the two BFSI clients' contractual patch windows |
| **Prioritized** | Effort concentrated on client-facing/BFSI-in-scope assets first, not spread flat |
| **Traceable** | Every finding has an owner, a due date, and a closure record |
| **Low-friction** | Doesn't degrade agent desktop performance during business hours (call-center SLAs are themselves contractual) |
| **Sustainable** | Program survives beyond the 90-day audit deadline — this is not a one-time fire drill |
| **ASV-certifiable** *(new — PCI-DSS-as-a-Service, §2.1)* | External CDE scanning is performed by, or through, a PCI SSC Approved Scanning Vendor, so scan output carries the attestation a client QSA requires — see [ADR-0003](adr/ADR-0003-external-asv-scanning-approach.md) |
| **Segmentation-validated** *(new — PCI-DSS-as-a-Service, §2.1)* | The boundary of each client's cardholder data environment (CDE) is independently tested and confirmed, not assumed from network diagrams — see [Architecture & Process §1](01-architecture-and-process.md#1-logical-layer-program-architecture) |
| **Multi-tenant-isolated** *(new — PCI-DSS-as-a-Service, §2.1)* | One client's CDE data, findings, and scan results are structurally inaccessible to any other client — a cross-tenant leak is itself a reportable PCI DSS incident — see [ADR-0004](adr/ADR-0004-multi-tenant-data-isolation-model.md) |

## 4. Success Criteria (90-day and 12-month)

**90-day (audit remediation window):**

- Asset inventory covering ≥95% of in-scope (client-contracted) infrastructure
- Authenticated vulnerability scanning operating on a defined cadence
- A documented, risk-based prioritization model (see [ADR-0001](adr/ADR-0001-composite-risk-prioritization-model.md))
- First closure-rate report produced and reviewed by CISO

**12-month (maturity target):**

- Full estate coverage (all delivery centers, cloud, containers)
- SLA adherence ≥90% for Critical/High findings on in-scope assets
- CMDB integration live, asset-criticality tagging ≥95% coverage
- Program operating as BAU with quarterly executive reporting, not project mode

## 5. Constraints

- Budget: mid-market — full enterprise platform (Qualys full suite) likely exceeds approved spend; must be justified against ROI/audit-risk avoidance
- No existing CMDB — must be built or bootstrapped as part of this program, not assumed
- WFH agent endpoints on variable bandwidth VPN — scan/agent design must account for this
- 90-day hard deadline for BFSI contract remediation clause
