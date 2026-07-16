# Rentwell — Plan

## Epics (in build order)

1. **Foundations** — auth (OIDC), multi-tenancy at the data layer (firm = tenant), configurable RBAC scaffold bound to authorization policies, OpenTelemetry via ServiceDefaults, append-only audit logging, API surface conventions (versioning, Problem Details, idempotency keys, per-tenant rate limiting), PII tagging + data-export/deletion procedure, dashboard shell + portal shells (operator console and resident/owner portal skeletons), chatbot panel, connector registry. Always epic 1; pulled from the enterprise guardrails.
2. **Portfolio & lease management** — the core data model and lease lifecycle. Capabilities (from SPEC): *Portfolio & lease management* (properties, units, ownership entities, people directory, document storage; lease drafting from templates, e-signature, renewals, security-deposit records). Depends on: Foundations.
3. **Rent collection & payments** — the monthly money machine. Capabilities (from SPEC): *Rent collection & payments* (rent schedules and charge generation from leases, ACH/card collection and autopay via the payment-partner abstraction, late-fee automation, outbound owner/vendor payment rails). Depends on: Portfolio & lease management.
4. **Resident & owner portals** — persona surfaces over what exists so far. Capabilities (from SPEC): *Resident & owner portals* (invite/registration flows; resident pay/autopay, documents, notices; owner documents, reports, approvals inbox). Depends on: Rent collection & payments.
5. **Trust accounting & owner finances** — compliance-grade books. Capabilities (from SPEC): *Trust accounting & owner finances* (double-entry GL with manager-vs-client fund separation, bank reconciliation from imported statements, owner statements + distributions into the owner portal, 1099-MISC/NEC preparation and e-file output). Depends on: Rent collection & payments; feeds Resident & owner portals.
6. **Maintenance & work orders** — the operational loop. Capabilities (from SPEC): *Maintenance & work orders* (resident intake with photos via portal, work orders, vendor directory and assignment, owner over-limit approvals, invoice capture). Depends on: Portfolio & lease management, Resident & owner portals.
7. **Leasing pipeline** — vacancy to signed lease. Capabilities (from SPEC): *Leasing pipeline* (vacancy listings + public listing pages + standards-based syndication feed, online applications with applicant-paid fees, FCRA screening via CRA partner with adverse-action records, applicant→lease conversion). Depends on: Portfolio & lease management, Rent collection & payments (application fees).
8. **Communication hub** — firm-wide messaging. Capabilities (from SPEC): *Communication hub* (template library, bulk email/SMS campaigns, threaded per-entity history unifying the transactional notifications shipped by earlier epics). Depends on: Foundations (channel connectors); enriches all prior epics.
9. **Audit-ready trust compliance** *(differentiator)* — depth on epic 5. Capabilities (from SPEC): *Audit-ready trust compliance* (three-way reconciliation reports, surfaced immutable audit trail, per-state security-deposit rules engine with deadline tracking). Depends on: Trust accounting & owner finances.
10. **Agentic AI** *(differentiator)* — the flagship. Capabilities (from SPEC): *Agentic AI across leasing, maintenance, and accounting* (24/7 leasing lead responder, conversational maintenance intake/triage, invoice→ledger extraction; draft-for-approval by default, per-firm autonomy opt-in, decision logging for Fair-Housing defensibility). Depends on: Leasing pipeline, Maintenance & work orders, Trust accounting & owner finances, Communication hub.
11. **Open platform (API + MCP)** *(differentiator)* — programmatic access. Capabilities (from SPEC): *Open API + MCP server* (versioned public API surface with API-key/OAuth client management; MCP server exposing firm data/tools to external LLMs under tenant RBAC). Depends on: all domain epics (exposes them).

Differentiators are epics 9–11, last by design so they can slip without blocking the v1 core.

## Module list

Module names use the `Rentwell.*` prefix (the legacy `The<Domain>` convention is retired with the brand-name rename); final project naming is pinned in design-architecture.

| Module (.NET project name) | Bounded context | Capabilities served | Skills used to build it |
|---|---|---|---|
| `Rentwell.Application.Foundations` | foundations | (cross-cutting: auth, tenancy, RBAC, audit, connector registry) | dotnet-aspire-base, pluggable-connectors, entity-framework-core |
| `Rentwell.Application.Portfolio` | portfolio | Portfolio & lease management (properties, units, owners, people, documents) | entity-framework-core, build-system |
| `Rentwell.Application.Leases` | leases | Portfolio & lease management (lease lifecycle, e-signature, deposits) | entity-framework-core, build-system |
| `Rentwell.Application.Payments` | payments | Rent collection & payments | entity-framework-core, build-system |
| `Rentwell.Application.Accounting` | accounting | Trust accounting & owner finances; Audit-ready trust compliance | entity-framework-core, build-system |
| `Rentwell.Application.Maintenance` | maintenance | Maintenance & work orders | entity-framework-core, build-system |
| `Rentwell.Application.Leasing` | leasing | Leasing pipeline | entity-framework-core, build-system |
| `Rentwell.Application.Communications` | communications | Communication hub | pluggable-connectors, build-system |
| `Rentwell.Application.Agents` | ai-agents | Agentic AI across leasing/maintenance/accounting | agent-framework-csharp |
| `Rentwell.Api` (host) | api | Serves all contexts incl. portal-scoped endpoints (Resident & owner portals are SPA surfaces + portal-scoped policies over existing contexts, not a domain module) | dotnet-aspire-base, verify-runtime |
| `Rentwell.OpenPlatform` | open-platform | Open API + MCP server | agent-framework-csharp, build-system |

Every SPEC capability appears above; the *Resident & owner portals* capability is delivered as SPA surfaces + portal-scoped authorization over the context modules (UI organization is design-architecture's concern).

## Data model sketch

Tenant boundary: **Firm** (the PM company) — every entity below carries the firm/tenant id, enforced by query filters. Audit events are append-only and stored outside the operational DB per the guardrails.

- **Firm** — the tenant; licensing info, trust-account config, fee schedules, per-state settings.
- **Property / Unit** — the managed asset tree; property → units; belongs to an OwnershipEntity.
- **OwnershipEntity / Owner** — the client (person or LLC); PII (tax id for 1099); linked to properties, statements, distributions.
- **Person** — shared directory for residents, applicants, owner contacts, staff; PII (name, contact, DOB where screening requires).
- **Lease / Tenancy** — unit + residents, term, rent schedule, deposit record (state-rule flags), renewal chain; signed documents attached.
- **Document** — files bound to firm/property/lease/person; PII flag per document class.
- **RentCharge / PaymentTransaction / Payout** — charges generated from lease schedules; transactions reference the payment-partner's ids (no card/bank primary data stored — PCI stays with the processor); payouts to owners/vendors.
- **LedgerAccount / JournalEntry** — double-entry books per firm with fund class (trust vs operating) on every account; journal entries immutable once posted.
- **ReconciliationRun / BankStatementImport** — monthly reconciliation state per bank account (three-way artifacts live in epic 9).
- **OwnerStatement / Distribution** — generated statement documents + the payout runs behind them.
- **Tax1099Record** — payee TIN (PII, encrypted), amounts by box, filing status, e-file batch reference.
- **MaintenanceRequest / WorkOrder / Vendor / VendorInvoice** — intake → triage → assignment → completion → invoice; vendor tax id is PII.
- **Listing / RentalApplication / ScreeningReport / AdverseActionRecord** — pipeline entities; application SSN and screening reports are restricted-access PII (FCRA); adverse-action records retained.
- **MessageThread / MessageTemplate / OutboundMessage** — comms bound to entities; delivery state per channel.
- **AgentRun / AgentDecisionLog** — every AI agent action with inputs, criteria, and outcome (Fair-Housing defensibility); approval state for draft-for-approval mode.
- **AuditEvent** — who/what/when/tenant/before-after for every domain mutation (append-only, external store).

## RBAC model (refined)

Policy names use `<Module>.<Action>` form; code references policies, never roles. Role→policy baselines are runtime-editable per firm (platform configurable-RBAC), seeded as:

| Role | Policies | Notes |
|---|---|---|
| broker-owner | `Firm.Configure`, `Rbac.Manage`, `Audit.View`, `Payouts.Release`, `Agents.Configure`, `OpenPlatform.Manage`, plus all property-manager and bookkeeper policies | Only role that changes compliance-relevant config or releases money |
| property-manager | `Portfolio.View/Edit`, `Leases.View/Edit/Execute`, `Maintenance.View/Edit/Dispatch`, `Leasing.View/Edit/Decide`, `Communications.Send/BulkSend`, `Payments.View`, `Agents.Review` | Scoped to assigned portfolios; cannot touch `Firm.*`, `Payouts.Release`, `Accounting.*` |
| bookkeeper | `Accounting.View/Post/Reconcile/File1099`, `Payments.View/Collect/Refund`, `Payouts.Prepare`, `Portfolio.View` | Prepares distributions; release stays with broker-owner |
| vendor | `WorkOrders.ViewAssigned/Update`, `VendorInvoices.Submit` | Sees only assigned work orders; no resident PII beyond the order |
| owner | `OwnerPortal.View` (statements, reports, documents), `Maintenance.Approve` | Read-only except over-limit maintenance approval; scoped to own properties |
| resident | `ResidentPortal.View`, `Payments.PaySelf`, `Maintenance.Submit`, `Documents.ViewOwn` | Scoped to own tenancy |
| screening-restricted (policy set, not a role) | `Leasing.Screen`, `ScreeningReports.View` | FCRA: grantable subset, off by default for everyone except property-manager with firm opt-in |

## Integration surface

`workflow.json.connectors` is empty today; email + SMS channel connectors are added at compose (Phase 6). Domain integrations (payments, screening) are core abstractions with one concrete provider each in v1 — provider choice is design-architecture's.

| Connector | Direction | Purpose | Webhook routes | Per-tenant config |
|---|---|---|---|---|
| payment-partner | outbound + inbound | ACH/card collection, payouts, trust sub-ledgering | `/api/connectors/payments/webhook/{events}` (payment status, payout status, chargeback) | partner account id, trust + operating account mapping, convenience-fee policy |
| screening-cra | outbound + inbound | FCRA credit/criminal/eviction reports | `/api/connectors/screening/webhook/{events}` (report ready) | CRA account credentials (vault ref), report package selection, applicant-pay toggle |
| email | outbound | transactional + bulk email | — (delivery-status webhook optional post-v1) | sender identity/domain, per-firm branding |
| sms | outbound + inbound | transactional + bulk SMS, resident replies | `/api/connectors/sms/webhook/inbound` | provisioned number, quiet hours |
| listing-feed | outbound (file/feed) | standards-based ILS/XML vacancy feed for aggregators | — | feed enable flag, firm branding fields |
| e-file-1099 | outbound | IRS 1099 e-filing (IRIS format v1: generate + track) | — | firm TIN (vault ref), transmitter config |

## Background work

| Job | Trigger | Cadence | Outbox required? |
|---|---|---|---|
| Rent charge generation | scheduled | daily (generates charges due per lease schedules) | yes |
| Autopay execution + payment status sync | scheduled + reactive (webhook) | daily run; webhook-driven updates | yes |
| Late-fee assessment | scheduled | daily, per-firm policy | yes |
| Security-deposit deadline monitor | scheduled | daily (state-rule deadlines → notifications) | yes |
| Owner distribution run | scheduled or manual | monthly, per-firm calendar; release gated on approval | yes |
| Owner statement generation | scheduled | monthly | yes (portal notification) |
| Bank statement import processing | reactive (upload) | on demand | no |
| Screening result processing | reactive (webhook) | on event | no |
| Listing feed export | scheduled | daily | no |
| 1099 season batch | scheduled + manual | annual window | yes (e-file submission) |
| Bulk message campaign send | reactive (user action) | on demand | yes |
| AI agent runs (lead response, maintenance triage, invoice extraction) | reactive (domain events) | on event; draft-for-approval queue | yes (any outbound side effect) |

## Open questions for design-architecture

1. **Payment partner selection** — needs ACH + card, payout rails, and trust-account sub-ledgering compatible with state trust rules (e.g. a for-benefit-of account model). Which provider, and does the trust ledger live with us (partner as pure rails) or partially with the partner?
2. **Screening CRA selection** — FCRA-compliant API with adverse-action support and applicant-paid flow (TransUnion-style). Which, and how is applicant PII (SSN) scoped so it never enters the general document store?
3. **E-signature: in-house or partner?** — ESIGN/UETA-compliant signing built on the platform's document infrastructure (Rentvine ships in-house) vs a partner integration. Cost-per-signature argues in-house; legal-audit features argue partner.
4. **Bank reconciliation input** — v1 assumes statement import (OFX/CSV). Is a bank-feed aggregator (Plaid-style) justified for v1, or post-v1?
5. **1099 e-file mechanics** — generate IRIS-format files with manual IRS-portal upload in v1, or integrate the IRIS API directly?
6. **Per-state rules engine representation** — security-deposit caps/interest/deadlines as versioned configuration data vs code; who maintains the state table and how is it updated mid-year?
7. **Scale/regional posture** — plan assumes single-region US, 100 firms × ≤2,000 doors (from SPEC answers). Confirm the data-residency and DR posture that implies.
8. **Portal identity** — residents/owners as external identities in the same OIDC tenant vs separate identity pool; affects invite/registration flows in epic 4.
