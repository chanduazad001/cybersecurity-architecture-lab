# ADR-0001: Composite Risk-Based Prioritization Model (Build, Not Buy)

## Status

Accepted

## Context

Meridian has no existing vulnerability management program; patching today is ad hoc, driven by Patch Tuesday and occasional client audit findings (see [Charter §1](../00-charter-and-business-case.md#1-company-context)). The 90-day BFSI contract remediation clause requires a *documented, risk-based prioritization model*, not just a scan report (see [Charter §4](../00-charter-and-business-case.md#4-success-criteria-90-day-and-12-month)), and the **Prioritized** business attribute explicitly calls for effort concentrated on client-facing/BFSI-in-scope assets first, not spread flat across the estate (see [Charter §3](../00-charter-and-business-case.md#3-business-attributes-conceptual-layer-sabsa)).

Vulnerability scanners typically ship a built-in risk score (e.g., a vendor-proprietary severity/priority rating) alongside raw CVSS. Using that built-in score directly is the path of least resistance, but it couples Meridian's prioritization logic to whichever scanner is selected in [ADR-0002](ADR-0002-scanner-selection-tenable-nessus.md) — which conflicts with the SABSA Logical-vs-Physical separation the rest of the architecture is built around (see [Architecture & Process §1](../01-architecture-and-process.md#1-logical-layer-program-architecture)): a scanner swap should never force a prioritization-model swap, and vice versa.

## Business Attributes

| Attribute | Relevance |
|---|---|
| **Prioritized** | Core driver — effort must concentrate on client-facing/BFSI-in-scope assets, not a flat severity sort |
| **Compliant** | The prioritization model itself is part of the auditor-facing evidence for the 90-day clause |
| **Traceable** | Every finding's priority must be explainable — "why was this ranked here" needs a documented answer, not a vendor black box |
| **Sustainable** | The model must survive a future scanner change without being rebuilt from scratch |
| **Timely** | Must be operational within the 90-day compliance window |

## Decision Drivers

- **Quality of prioritization** — whether the model can combine severity, real-world exploitability, and Meridian's own asset/client context, or only reflects what one vendor's scanner happens to expose
- **Vendor lock-in / architectural coupling** — whether swapping the scanner (a Physical-layer decision) forces rework of the prioritization logic (a Logical-layer decision)
- **Auditability** — whether the scoring rationale can be explained plainly to a CISO, client, or auditor on demand
- **Time-to-operate** — must be standing up findings prioritization inside the 90-day window, not a multi-quarter build
- **Ability to weight client/BFSI-in-scope criticality** — a factor no scanner vendor can know without Meridian's own CMDB context

## Options Considered

### Option A — Use the Scanner Vendor's Built-In Risk Score

Adopt whatever proprietary priority/risk rating the selected scanner ships with, unmodified.

- **Pros:** zero build effort; available from day one; vendor-maintained as CVEs are published
- **Cons:** ties prioritization logic directly to scanner choice — a future scanner change (or a second scan source, e.g. container/CSPM findings) forces the prioritization model to change too, violating the intended Physical-vs-Logical separation; the scoring methodology is typically a vendor black box, hard to explain confidently to auditors or clients when asked why a specific finding was prioritized as it was; cannot incorporate Meridian's own CMDB asset-criticality or BFSI-in-scope weighting, since the vendor has no visibility into Meridian's client contracts

### Option B — CVSS Base Score Only

Sort findings by CVSS Base Score, descending, with no additional weighting.

- **Pros:** simplest possible model; universally available from any scan source; trivially explainable
- **Cons:** measures theoretical severity only — routinely ranks a high-CVSS finding on a low-value internal asset above a lower-CVSS finding under active exploitation on internet-facing, client-facing infrastructure; does not satisfy the **Prioritized** attribute's requirement to concentrate effort on BFSI-in-scope assets specifically; ignores real-world exploitation likelihood entirely

### Option C — Build a Custom Composite Scoring Engine

Build a scanner-independent scoring service that combines CVSS severity, EPSS exploitation probability, and CMDB-sourced asset criticality/exposure into a single composite score, consuming scan findings via API rather than owning the scan itself (see [Architecture & Process §1](../01-architecture-and-process.md#1-logical-layer-program-architecture)).

- **Pros:** decouples prioritization logic from scanner choice entirely — the scanner is just one of several API inputs, so [ADR-0002](ADR-0002-scanner-selection-tenable-nessus.md) can change independently in the future; can directly encode the **Prioritized** attribute by weighting BFSI/client-facing asset criticality from the CMDB; scoring rationale is fully owned and documented by Meridian, so it is auditable and explainable on demand; extensible to new inputs later (e.g., KEV status, the PCI-DSS-as-a-service segmentation findings in [ADR-0004](ADR-0004-multi-tenant-data-isolation-model.md))
- **Cons:** requires engineering effort to build and maintain during the same 90-day window as everything else — a real opportunity cost; more complex to explain in one sentence than a single vendor number, though more explainable in substance once documented; quality of the output is bounded by the accuracy of CMDB criticality tagging, which is itself still being bootstrapped (see [Charter §5](../00-charter-and-business-case.md#5-constraints))

## Decision

Adopt **Option C — build a custom composite scoring engine** (CVSS × Criticality × EPSS × Exposure), implemented as a standalone service that ingests findings and asset context via API rather than being embedded in the scanner.

Options A and B both fail the **Prioritized** attribute in the same way: neither can incorporate Meridian's own client-contract context (BFSI-in-scope priority), because that context lives in Meridian's CMDB, not in any scanner's data model. Option A additionally creates the exact coupling the architecture is designed to avoid — a scanner decision (Physical layer) should not dictate a prioritization decision (Logical layer). The build effort in Option C is accepted as a necessary cost within the 90-day window; the first version can ship as a minimum-viable composite (CVSS + Criticality, with EPSS added once the feed integration is stable) rather than waiting for every input to be perfect before producing prioritized output.

## Consequences

### Positive

- Remediation effort concentrates on findings that are both severe and business-critical (BFSI/client-facing), not just high-CVSS anywhere in the estate
- Scanner choice ([ADR-0002](ADR-0002-scanner-selection-tenable-nessus.md)) can change in the future without forcing a rebuild of the prioritization model
- Scoring rationale is fully documented and explainable to auditors, satisfying the **Traceable** attribute directly
- New inputs (KEV status, PCI-DSS-as-a-service segmentation findings) can be added later without an architectural rework

### Negative

- Requires dedicated engineering effort to build and operate, competing for time against other 90-day-window work
- More complex to explain than "the scanner says X," though the added complexity is the point — it makes the rationale explicit rather than hidden in a vendor formula
- Score quality is bounded by CMDB completeness and accuracy, which is a parallel, still-in-progress workstream (see [Charter §5](../00-charter-and-business-case.md#5-constraints))
- Requires ongoing integration maintenance with the EPSS feed and, longer-term, KEV

## Follow-ups

- Define the exact composite formula and initial weighting (CVSS band, criticality tier, EPSS score, exposure) before the first 90-day closure-rate report is due
- Ship a minimum-viable version (CVSS + CMDB criticality) first; add EPSS once the feed integration is validated, rather than blocking the 90-day deadline on a fully-weighted model
- Revisit whether CISA KEV status should be added as an explicit override (a known-exploited finding should never be silently outranked by a higher-CVSS-but-unexploited one)
- Confirm the scoring engine's API contract with [ADR-0002](ADR-0002-scanner-selection-tenable-nessus.md)'s selected scanner, and design it to accept additional scan sources (container, CSPM) without a schema rework
