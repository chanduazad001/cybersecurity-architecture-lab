# ADR-0002: Scanner Selection — Tenable Nessus (Authenticated Agent + Agentless)

## Status

Accepted

## Context

Meridian has no vulnerability scanning tooling in place today and needs authenticated scanning operating on a defined cadence within the 90-day compliance window (see [Charter §4](../00-charter-and-business-case.md#4-success-criteria-90-day-and-12-month)). The estate is a hybrid mix — an on-prem AD domain (~3,000 Windows endpoints, ~400 Linux/Windows servers), a growing Azure footprint hosting an AKS client portal, and a WFH workforce (~35% of headcount) connecting over VPN with variable bandwidth (see [Charter §1](../00-charter-and-business-case.md#1-company-context)).

Budget is an explicit constraint: a full enterprise platform (e.g., the complete Qualys suite) likely exceeds approved mid-market spend and must be justified against ROI/audit-risk avoidance (see [Charter §5](../00-charter-and-business-case.md#5-constraints)). Scan design must also account for WFH endpoints on variable-bandwidth VPN, and must not degrade agent desktop performance during business hours, since call-center SLAs are themselves contractual (**Low-friction** attribute, [Charter §3](../00-charter-and-business-case.md#3-business-attributes-conceptual-layer-sabsa)).

Per [ADR-0001](ADR-0001-composite-risk-prioritization-model.md), the scanner is treated as one interchangeable input to an independent scoring engine, not the owner of prioritization logic — so this decision is scoped narrowly to scan coverage, deployment fit, and API quality, not to risk scoring.

## Business Attributes

| Attribute | Relevance |
|---|---|
| **Compliant** | Must support authenticated scanning, the baseline evidence type auditors expect |
| **Timely** | Must be deployable within the 90-day window |
| **Low-friction** | Must not degrade WFH/office desktop performance during business hours |
| **Sustainable** | Licensing cost must fit a mid-market budget on an ongoing basis, not just for the initial rollout |
| **Prioritized** | Must expose a clean API so findings can feed the composite scoring engine ([ADR-0001](ADR-0001-composite-risk-prioritization-model.md)) without the scanner's own scoring getting in the way |

## Decision Drivers

- **Budget fit** against the mid-market constraint, versus full enterprise suite pricing
- **WFH/VPN compatibility** — variable-bandwidth, intermittently-connected endpoints need a scan mechanism that doesn't depend on a continuously live authenticated network path
- **API quality** for feeding findings into the independent scoring engine ([ADR-0001](ADR-0001-composite-risk-prioritization-model.md))
- **Coverage breadth** across Windows/Linux hosts, Azure resources, and (via a companion component) containers
- **Time-to-deploy** within the 90-day compliance window
- **Desktop performance impact** on call-center agent endpoints during business hours

## Options Considered

### Option A — Qualys (Full Enterprise Suite)

A single-vendor platform spanning vulnerability management, CSPM, and container scanning.

- **Pros:** broadest single-vendor coverage — one platform instead of several components; strong, long-established enterprise track record
- **Cons:** licensing cost likely exceeds the approved mid-market budget, the explicit constraint in [Charter §5](../00-charter-and-business-case.md#5-constraints); heavier agent footprint has a real risk of impacting desktop performance on call-center endpoints during business hours, conflicting with the **Low-friction** attribute; broader than what a 90-day MVP actually needs, when a narrower authenticated-scanning tool paired with a separate container scanner ([Architecture & Process §3](../01-architecture-and-process.md#3-physical-layer-component-choices-for-meridian)) covers the same ground at lower cost

### Option B — OpenVAS / Greenbone (Open Source)

A no-license-cost, self-hosted scanning platform.

- **Pros:** no licensing cost, which fits the tightest possible budget constraint
- **Cons:** its authenticated-scanning strength is primarily network-based, with a weaker native agent-based option for endpoints that are only intermittently reachable over variable-bandwidth VPN — a poor fit for the WFH portion of the estate; a less mature API/enrichment ecosystem makes clean integration with the independent scoring engine ([ADR-0001](ADR-0001-composite-risk-prioritization-model.md)) more work to build and maintain; self-hosted support burden falls entirely on Meridian's own small team, a real risk against a hard 90-day compliance deadline

### Option C — Tenable Nessus (Authenticated Agent + Agentless)

Authenticated scanning using a lightweight agent for endpoints (including WFH/VPN) and agentless network-based scanning for servers and on-prem infrastructure.

- **Pros:** the agent-based mode solves the WFH/variable-bandwidth-VPN problem directly — the agent reports on its own local schedule rather than requiring a continuously live authenticated network scan path; mid-market pricing fits the budget constraint far better than a full Qualys suite; a well-documented, mature API for pushing findings to the independent scoring engine, cleanly supporting the [ADR-0001](ADR-0001-composite-risk-prioritization-model.md) separation; fast to license and deploy within the 90-day window; established, well-understood authenticated-scanning support across Windows, Linux, and Azure-hosted resources
- **Cons:** narrower single-product scope than Qualys — no native CSPM or container-image scanning platform, requiring a separate component (Trivy, per [Architecture & Process §3](../01-architecture-and-process.md#3-physical-layer-component-choices-for-meridian)) to cover those; still a paid, licensed product, though materially cheaper than the full enterprise-suite alternative; agent lifecycle (distribution, updates) becomes a new operational responsibility for IT Infrastructure

## Decision

Adopt **Option C — Tenable Nessus**, using authenticated agent-based scanning for WFH/VPN endpoints and agentless authenticated scanning for on-prem servers and AD-joined infrastructure.

Option A is rejected primarily on the budget constraint from [Charter §5](../00-charter-and-business-case.md#5-constraints) and the desktop-performance risk to the **Low-friction** attribute; its broader single-vendor coverage isn't worth the cost when a narrower tool plus a dedicated container scanner covers the same functional ground. Option B is rejected because its scanning model doesn't fit the WFH/variable-bandwidth-VPN portion of the estate well, and its support burden is too high a risk against a hard 90-day deadline for a program with no existing team or tooling. Option C's agent-based mode is the deciding factor: it is the only option of the three that directly solves the "how do you authenticate-scan an endpoint that's only intermittently online over a slow VPN link" problem, while also fitting the budget and API-integration requirements.

## Consequences

### Positive

- Fits the mid-market budget constraint without the full-suite cost of Option A
- Agent-based scanning directly addresses the WFH/VPN connectivity constraint, rather than working around it
- Clean API decouples scan collection from the independent scoring engine ([ADR-0001](ADR-0001-composite-risk-prioritization-model.md)), preserving the intended architectural separation
- Fast enough to license and deploy within the 90-day compliance window

### Negative

- Does not natively cover container image or cloud-configuration (CSPM) scanning — these remain separate components in the architecture ([Architecture & Process §3](../01-architecture-and-process.md#3-physical-layer-component-choices-for-meridian)) rather than a single consolidated platform
- Agent distribution and lifecycle management (deployment, updates, health monitoring) becomes a new, ongoing operational responsibility for IT Infrastructure
- Licensing is an ongoing cost beyond the initial 90-day rollout, and must be sustained as a BAU budget line per the 12-month maturity target ([Charter §4](../00-charter-and-business-case.md#4-success-criteria-90-day-and-12-month))

## Follow-ups

- Procure/license Tenable Nessus and confirm final pricing against the approved mid-market budget
- Pilot agent deployment on a representative subset of WFH endpoints first, to validate the **Low-friction** attribute (desktop performance during business hours) before estate-wide rollout
- Define the agent distribution, update, and health-monitoring process with IT Infrastructure
- Confirm the Nessus findings API integrates cleanly with the composite scoring engine ([ADR-0001](ADR-0001-composite-risk-prioritization-model.md))
- Revisit tool choice at the 12-month maturity checkpoint if coverage gaps (cloud/container consolidation) make a broader single-vendor platform worth the added cost
