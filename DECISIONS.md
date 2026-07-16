# Rentwell — Architecture decision records

ADRs accumulate append-only; superseding adds a new ADR with a back-reference. Guardrail-mandated choices (Postgres, EF Core, Redis, Polly, OTel, MAF, Vite/React/shadcn, Terraform, GitHub Actions) are constraints, not decisions, and carry no ADR.

## ADR-0001: Deploy to Azure Container Apps rather than App Service

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

The dotnet-architecture skill's default compute is Azure App Service (Linux). Rentwell is an Aspire-composed topology (Api, OpenPlatform, Web + data resources), and the platform SaaS model provisions dedicated infra per customer via Terraform. Affects ARCH.md *Cloud topology*.

### Decision

We will run Rentwell's containers in an Azure Container Apps environment, VNet-integrated, deployed via Terraform.

### Consequences

- **Positive**: native fit for Aspire multi-container apps; per-container scale-to-zero for OpenPlatform; consistent with Aspire deployment tooling.
- **Negative**: deviates from the family default (App Service), so the dotnet-architecture Terraform scaffold needs the ACA variant; slightly more networking config.
- **Neutral**: both options are Linux containers behind managed ingress; the app model doesn't change.

### Alternatives considered

- **Azure App Service (Linux)** — family default, simplest ops. Rejected because: multi-container Aspire topology maps awkwardly (one plan per app, no scale-to-zero for the MCP host).
- **AKS** — full Kubernetes control. Rejected because: operational overhead unjustified at v1 scale (100 firms, single region).

## ADR-0002: Payment rails via Stripe Connect behind IPaymentRails; ledger-of-record stays internal

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

SPEC mandates all money movement through a licensed processor (no in-house money transmission). PLAN leaves the provider open. Trust accounting requires knowing where every dollar sits — either we keep the authoritative ledger or lease the partner's. Affects ARCH.md *Cross-cutting wiring*, *Data model* (Payments, Accounting).

### Decision

We will integrate one processor in v1 — Stripe Connect (ACH debits + card collection, transfers for owner/vendor payouts) — behind an `IPaymentRails` interface, and keep the authoritative trust ledger inside Rentwell's Accounting context; the processor is rails only.

### Consequences

- **Positive**: mature APIs/webhooks and sandbox; ledger sovereignty keeps trust compliance (ADR-0007, epic 9) in our control; provider swap is a finite adapter project.
- **Negative**: Stripe's fit for FBO-style trust flows is weaker than ACH-specialists; fund-flow design must be validated against state trust rules during epic 3; processing fees are pass-through to firms.
- **Neutral**: no PANs/bank numbers stored locally either way (PCI stays with the processor).

### Alternatives considered

- **Dwolla** — ACH-first with FBO account model, common in PM fintech. Rejected for v1 because: weaker card support and smaller ecosystem; remains the natural second adapter if Stripe's trust-flow fit fails validation.
- **Moov** — modern money-movement platform. Rejected because: younger platform, less operational track record for this use case.

## ADR-0003: Tenant screening via TransUnion SmartMove behind IScreeningProvider

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

FCRA requires screening through a consumer reporting agency with permissible purpose, consent, and adverse-action support. SPEC requires an applicant-paid flow. Affects ARCH.md *Data model* (Leasing) and the leasing endpoint group.

### Decision

We will integrate TransUnion SmartMove as the v1 screening provider behind an `IScreeningProvider` interface, with consent capture, restricted-access reports, and adverse-action records as first-class domain records.

### Consequences

- **Positive**: SmartMove is purpose-built for landlord/PM screening with native applicant-paid flow and FCRA tooling; three of the five researched competitors screen through TransUnion products.
- **Negative**: single-bureau data in v1; commercial onboarding (credentialing) is a real-world dependency the build can only stub.
- **Neutral**: reports render from partner data; we store references + decision records, not raw bureau files.

### Alternatives considered

- **Experian Connect** — comparable bureau product. Rejected because: weaker PM-segment tooling for applicant-paid flows.
- **CRS/aggregator (multi-bureau)** — broader data. Rejected for v1 because: heavier integration and credentialing for marginal v1 value.

## ADR-0004: Build e-signature in-house (ESIGN/UETA-compliant)

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

Lease execution is core to epic 2 and runs at high volume (every lease, renewal, and addendum). Per-envelope pricing from e-sign vendors is a recurring per-lease cost competitors monetize (Buildium charges $1–5/doc); Rentvine ships in-house signing (RentSign) as a selling point. Affects ARCH.md *Data model* (Leases: `SignatureEnvelope`).

### Decision

We will implement e-signature in-house in the Leases context: intent-to-sign capture, ESIGN/UETA consent, document hash sealing, signer authentication via portal identity, and a tamper-evident audit trail on the envelope.

### Consequences

- **Positive**: zero marginal cost per signature; UX stays in-product; matches the research's "un-gated" positioning.
- **Negative**: we own legal-defensibility features (audit trail, retention) ourselves; no third-party evidentiary brand behind signatures.
- **Neutral**: scope is lease-shaped signing, not a general-purpose e-sign product.

### Alternatives considered

- **DocuSign** — strongest evidentiary brand. Rejected because: per-envelope cost at rent-roll volume and an external UX break in the core loop.
- **Dropbox Sign (embedded)** — cheaper embedded option. Rejected because: still per-envelope pricing; embedding effort comparable to building lease-scoped signing.

## ADR-0005: Bank reconciliation from OFX/CSV imports in v1; bank-feed aggregator deferred

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

The reconciliation metric (100% of trust accounts within 5 days of month end) needs bank data. Live feeds (Plaid-style) add a vendor, consent UX, and cost; monthly statement files are how many PM bookkeepers already work. Affects ARCH.md *Data model* (Accounting: `BankStatementImport`).

### Decision

We will ship OFX/CSV statement import with a matching workbench in v1; a bank-feed aggregator is a post-v1 connector behind the same import abstraction.

### Consequences

- **Positive**: no third-party dependency on the compliance-critical path; import format is testable and deterministic.
- **Negative**: reconciliation freshness is monthly-statement-bound; competitive parity with "bank feeds" marketing waits for post-v1.
- **Neutral**: the matching engine is identical either way; only the ingestion changes.

### Alternatives considered

- **Plaid transactions feed** — daily freshness. Rejected for v1 because: extra vendor + consent flows for a metric monthly imports already satisfy.
- **Direct bank OFX endpoints (per-bank credentials)** — no aggregator. Rejected because: per-bank fragility and credential handling we don't want.

## ADR-0006: 1099 e-filing via generated IRIS files with manual IRS-portal submission in v1

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

SPEC requires 1099-MISC/NEC preparation and e-filing. The IRS IRIS system accepts both API transmission (requires TCC credentialing) and portal-uploaded files. Filing is an annual, low-frequency workflow. Affects ARCH.md *Data model* (Accounting: `Tax1099Record`) and the C1 diagram's IRS actor.

### Decision

We will generate validated IRIS-format files from the ledger with a guided review flow, and the firm uploads them to the IRS portal in v1; direct IRIS API transmission is post-v1.

### Consequences

- **Positive**: full data pipeline (TIN capture, categorization, box mapping) ships in v1 without credentialing lead time; annual cadence makes manual upload tolerable.
- **Negative**: "e-filing" is generate-and-upload, not one-click transmit, until the API lands.
- **Neutral**: the record model and validation are identical for both transmission modes.

### Alternatives considered

- **IRIS API transmission in v1** — one-click filing. Rejected because: TCC credentialing and API certification are external dependencies that would gate an annual feature.
- **Third-party filing service (Avalara-style, as DoorLoop uses)** — outsourced filing. Rejected because: per-form fees and an extra vendor for a flow we can generate ourselves.

## ADR-0007: State compliance rules as versioned, effective-dated configuration data

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

Security-deposit caps, interest, and return deadlines vary by state and change over time; trust-accounting expectations vary by state commission. Hardcoding rules makes legal updates a deploy; a full rules engine is over-architecture for v1. Affects ARCH.md *Data model* (`StateComplianceRuleSet`) and epic 9.

### Decision

We will represent state compliance rules as versioned, effective-dated configuration data (one ruleset per state, shipped and updated as reviewed data releases), evaluated by a small deterministic evaluator; firms cannot override rules, only firm-level parameters (e.g. their management-agreement thresholds).

### Consequences

- **Positive**: legal updates are data changes with review + effective dates; evaluators are unit-testable against known state fixtures.
- **Negative**: someone must curate the 50-state table — v1 seeds the states of pilot firms and flags unseeded states in-product.
- **Neutral**: the evaluator interface leaves room for a real rules engine if complexity ever demands it.

### Alternatives considered

- **Hardcoded per-state logic** — fastest start. Rejected because: every legal change becomes a code deploy and audit risk.
- **General rules engine (e.g. expression-based)** — maximum flexibility. Rejected because: premature abstraction; three rule families don't justify an engine.

## ADR-0008: One Entra External ID (CIAM) authority for staff and portal users; authorization stays in Rentwell RBAC

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

Rentwell serves PM-firm staff plus external residents/owners/vendors across many firm tenants. Staff are not members of the operator's workforce directory, so the guardrail's "Entra ID" maps to the CIAM flavor. PLAN open question 8 asked whether portal users get a separate identity pool. Affects ARCH.md *Cross-cutting wiring*, *Cloud topology*.

### Decision

We will use a single Entra External ID (CIAM) tenant as the OIDC authority for all human users; identity establishes *who*, and all authorization (role→policy baselines per firm, portal scoping) resolves from Rentwell's RBAC store — no authorization claims in tokens.

### Consequences

- **Positive**: one sign-in stack, invite flows, and MFA story for every persona; role edits take effect without re-issuing tokens; a person can hold roles at multiple firms.
- **Negative**: every request pays an authorization lookup (mitigated by the Redis-cached policy baseline).
- **Neutral**: aligns with the platform's configurable-RBAC model (authorization is tenant data, not directory data).

### Alternatives considered

- **Separate identity pools (workforce Entra for staff, CIAM for portals)** — cleaner separation. Rejected because: PM-firm staff aren't in any workforce tenant we control; doubles the auth surface.
- **Roles as token claims** — no per-request lookup. Rejected because: contradicts runtime-editable per-firm RBAC; stale-claim revocation problems.

## ADR-0009: Quartz.NET as the single in-process background scheduler

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

The guardrails mandate a single in-process scheduler plus the outbox pattern, but don't name one. PLAN inventories 12 jobs (cron-like cadences plus reactive outbox dispatch). Quartz.NET is not in the guardrail technology list, so naming it requires an ADR. Affects ARCH.md *Cross-cutting wiring*.

### Decision

We will use Quartz.NET hosted in Rentwell.Api (persistent job store in Postgres, clustered mode off until multi-instance) to run the scheduled job inventory; outbox dispatch runs as a Quartz-triggered drain.

### Consequences

- **Positive**: boring, battle-tested cron semantics; persistent store survives restarts; misfire policies cover deploy windows.
- **Negative**: one more library to own; clustered locking must be revisited if Api scales beyond one replica for scheduled work.
- **Neutral**: jobs stay thin — they invoke application handlers, so the scheduler is swappable.

### Alternatives considered

- **Hangfire** — dashboard included. Rejected because: dashboard duplicates the Aspire/OTel surface; storage schema heavier.
- **Plain `BackgroundService` timers** — zero dependencies. Rejected because: hand-rolling cron, misfire, and persistence for 12 jobs re-implements Quartz badly.

## ADR-0010: CQRS-lite per context; no event sourcing

- **Status**: accepted
- **Date**: 2026-07-16
- **Deciders**: Abraham Fernandez (delegated to agent)

### Context

Contexts differ: Accounting needs immutable journal semantics and period locks; Portfolio is mostly CRUD. A uniform heavyweight CQRS/event-sourcing approach would over-architect v1; pure CRUD would under-serve the ledger. Affects ARCH.md *Components*.

### Decision

We will use CQRS-lite everywhere — explicit command/query handlers per context, EF Core as the single write model — with immutability enforced in the Accounting domain model (post-only journals, period locks, reversal entries); no event sourcing anywhere in v1.

### Consequences

- **Positive**: uniform handler shape for build-system generation and RBAC/idempotency middleware; ledger correctness comes from domain rules, not infrastructure novelty.
- **Negative**: audit "before-after" is captured by the audit pipeline rather than derivable from an event stream.
- **Neutral**: domain events (in-process, via outbox for side effects) still exist for integration; they're just not the source of truth.

### Alternatives considered

- **Event sourcing for Accounting** — perfect audit derivation. Rejected because: operational and tooling cost across migrations/reporting outweighs benefit when journals are already append-only by rule.
- **Plain CRUD services** — least ceremony. Rejected because: money-touching flows need explicit commands for idempotency, approval gates, and decision logging.
