# ADR-0004: Multi-Tenant Data Isolation Model for the CMDB and Scoring Engine

## Status

Accepted

## Context

The internal VM program's CMDB and Composite Scoring Engine (ADR-0001) were designed for a single-tenant estate — Meridian's own infrastructure. Extending VM-as-a-Service to external clients pursuing PCI DSS (see [Charter §2.1](../00-charter-and-business-case.md#21-new-business-driver-pci-dss-as-a-service-added-2026-08-23)) means these same components must now hold and process data belonging to multiple distinct client CDEs — some of whom may be competitors of each other within BFSI/retail.

PCI DSS treats any cross-client visibility into another client's CDE inventory, findings, or scan results as itself a reportable data exposure. This is not an ordinary application multi-tenancy problem — it is a compliance-boundary problem, and a QSA auditing one client's PCI evidence may ask Meridian to demonstrate the isolation model directly.

## Business Attributes

| Attribute | Relevance |
|---|---|
| **Multi-tenant-isolated** | Core driver — this ADR exists specifically to satisfy this attribute |
| **Compliant** | A cross-tenant leak is itself a PCI DSS violation and a reportable incident for the affected client |
| **Traceable** | Every finding must be attributable to exactly one client's CDE, with no ambiguity |
| **Segmentation-validated** | The isolation model must be independently verifiable, the same way network segmentation is validated for PCI scope |
| **Sustainable** | Must scale to additional clients without a re-architecture each time |

## Decision Drivers

- **Audit-defensibility** — Meridian must be able to show a QSA, on demand, exactly how Client A's data is prevented from being visible to Client B; a strong architectural boundary is easier to defend than a policy-only one
- **Blast radius of a breach** — how much data is exposed, and to whom, if isolation fails
- **Operational cost** — schema migrations, backups, and monitoring multiply with the number of isolated units
- **Time-to-onboard a new client** — how much provisioning work is required per client

## Options Considered

### Option A — Separate Database per Client

Each client gets a dedicated database (schema + data) within the CMDB and scoring engine, provisioned at onboarding.

- **Pros:** strongest audit story — "Client A's data physically cannot be queried by a connection scoped to Client B's database" is a simple, QSA-legible sentence; the blast radius of an application-layer bug (e.g., a missing tenant filter) is contained to one client's database rather than all of them; supports per-client backup/retention policies where contractually required
- **Cons:** migrations, monitoring, and backup jobs must run per-database and scale linearly with client count; connection pooling and ops tooling need to be tenant-aware; higher operational overhead than a shared schema

### Option B — Row-Level Tenancy With Strict Access Controls

A single shared database; every row carries a `client_id`, enforced via application-layer checks and database row-level security (RLS) policies.

- **Pros:** lowest operational overhead — one schema to migrate, monitor, and back up; fastest to provision a new client (insert a tenant record, no infrastructure change); easiest path for Meridian's own cross-tenant aggregate reporting
- **Cons:** isolation depends on every query correctly applying the tenant filter or RLS policy — a single missed filter (application bug or an ad hoc admin query) can leak cross-client data; the weakest audit story of the three, since isolation is logical/policy-enforced rather than structural; a bug or breach has full-database blast radius

### Option C — Fully Separate Deployed Instances per Client

Each client gets a dedicated, independently deployed instance of the CMDB and scoring engine (separate compute, separate database, no shared infrastructure).

- **Pros:** strongest possible isolation — no shared infrastructure at all between clients, even at the compute layer; a compromise of one client's instance has zero blast radius to any other client; simplest possible audit narrative
- **Cons:** highest operational cost by far — the full infrastructure stack is multiplied per client, which defeats much of the economic case for a shared VM-as-a-Service platform (this stops being "as-a-service" and becomes bespoke per-client deployments); slowest to onboard a new client; patching or upgrading the platform means N separate upgrades instead of one

## Decision

Adopt **Option A — separate database per client**, with shared application/compute infrastructure (scanning orchestration, scoring engine logic, reporting UI) but per-client database isolation.

This is weighed primarily on the two decision drivers the business context makes non-negotiable: audit-defensibility and blast radius. Option B's cross-tenant leak risk is exactly the failure mode PCI DSS treats as a reportable incident in its own right — "a missing `WHERE client_id = ?`" is not a risk Meridian should accept when the affected data belongs to a client's cardholder data environment. Option C's isolation is stronger still, but its operational cost defeats the commercial premise of offering this as a shared service rather than bespoke per-client deployments; that cost is not justified by the incremental risk reduction over Option A, which already provides structural database-level isolation. Option A is the point on this curve where the audit story is simple and defensible without paying for N fully separate infrastructure stacks.

## Consequences

### Positive

- Clear, QSA-legible isolation story: one client's data lives in one client's database
- Contained blast radius — an application bug affecting tenant-filtering logic cannot leak across a database boundary the way it could under Option B
- Per-client backup/retention policies are straightforward to support if a client's contract requires them

### Negative

- Operational tooling (migrations, monitoring, backups, connection management) must be built to be tenant-aware and scale per-database, not per-row
- New client onboarding requires a database provisioning step, not just a config/tenant-record insert — slower than Option B
- Internal cross-client reporting (e.g., Meridian's own aggregate service-health metrics across all PCI clients) requires an explicit aggregation layer rather than a simple query, since data is not held in one shared table

## Follow-ups

- Define the per-client database provisioning runbook as part of client onboarding, not a one-off manual step
- Confirm the scoring engine and scanning orchestration layer can be parameterized per-tenant database connection without code duplication
- Define the internal aggregate-reporting layer needed for Meridian's own cross-client service-health metrics without violating per-client isolation
- Re-evaluate at scale: if client count grows large enough that per-database operational overhead becomes the bottleneck, revisit Option C-style isolation for specifically high-value/high-risk clients, or invest in database-per-client provisioning automation
