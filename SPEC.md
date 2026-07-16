# Rentwell — Product specification

## In one sentence

Rentwell is an AI-native property management platform for US third-party property management companies (~50–2,000 doors) that runs the monthly machine — rent, maintenance, leasing, and audit-proof trust accounting — with agentic AI handling the routine so a small team can manage a big portfolio.

## Primary jobs to be done

- When rent comes due each month, I want collection, late fees, and owner payouts to run themselves, so that my team only touches exceptions.
- When a resident reports an issue at 2am, I want intake, triage, and vendor dispatch to start immediately, so that small problems don't become expensive ones.
- When a unit goes vacant, I want it listed, applicants screened, and the lease signed with minimal manual work, so that vacancy days stay low.
- When a state auditor or an owner asks about money, I want trust-accounting books that prove where every dollar is, so that my broker's license is never at risk.
- When tax season arrives, I want owner and vendor 1099s generated and e-filed from the year's ledger, so that January isn't a fire drill.
- When owners ask "how is my property doing," I want them self-served through a portal and automated statements, so that I'm not writing the same email forty times.

## Target personas

- **Broker-owner (firm admin)** — owns the PM company, its broker license, and its trust accounts; configures the firm on the platform. Top 3 tasks:
  1. Configure trust accounts, fee schedules, and per-state deposit rules
  2. Assign roles/permissions to staff and review the audit log
  3. Review and release owner distributions
- **Property manager** — runs day-to-day operations for a portfolio of doors. Top 3 tasks:
  1. Work the maintenance queue: review AI-triaged requests, approve vendor dispatch
  2. Drive the leasing pipeline: listings, applicant review, screening decisions, lease signing
  3. Send and answer resident/owner communications from the shared hub
- **Bookkeeper** — keeps the firm's books compliant. Top 3 tasks:
  1. Run monthly bank reconciliation across trust and operating accounts
  2. Generate owner statements and distributions
  3. Prepare and e-file year-end 1099s for owners and vendors
- **Property owner (client)** — the PM firm's customer; landlord of record. Top 3 tasks:
  1. Read monthly statements and financial reports in the owner portal
  2. Approve maintenance work above their spend limit
  3. Download tax documents (1099, year-end statement)
- **Resident** — the tenant living in a managed unit. Top 3 tasks:
  1. Pay rent / set up autopay
  2. Submit a maintenance request with photos and track its status
  3. View lease, documents, and notices

## Capabilities

### Must have (v1)

| Capability | One-line description | Personas |
|---|---|---|
| Portfolio & lease management | Properties, units, tenancies, and ownership entities as the core data model; lease lifecycle with templates, e-signature, renewals, and per-entity document storage. | property manager, broker-owner |
| Rent collection & payments | ACH/card autopay with configurable late-fee automation; outbound owner distributions and vendor payments — all money movement on licensed payment-partner rails. | resident, bookkeeper, property manager |
| Trust accounting & owner finances | Double-entry GL with manager-vs-client fund separation, bank reconciliation, automated owner statements/distributions, and 1099-MISC/NEC e-filing. | bookkeeper, broker-owner, property owner |
| Leasing pipeline | Vacancy listings with syndication feed, online applications with applicant-paid fees, and FCRA-compliant screening (credit/criminal/eviction) with adverse-action records. | property manager |
| Maintenance & work orders | Resident intake with photos, triage, vendor assignment and approval workflow, status tracking through completion and invoice capture. | resident, property manager, property owner |
| Resident & owner portals | Portal-per-persona: residents pay and file requests; owners read statements, reports, and approvals — each seeing only their own slice. | resident, property owner |
| Communication hub | Templated email/SMS, bulk messaging, and threaded history bound to properties, leases, and people. | property manager, broker-owner |

(The research artifact's 14 table-stakes capabilities all map into these seven; nothing was dropped, only folded.)

### Differentiators (v1)

| Capability | Why it matters | Personas |
|---|---|---|
| Agentic AI across leasing, maintenance, and accounting | 24/7 lead response, conversational maintenance intake/triage, invoice→ledger extraction. Only AppFolio ships this end-to-end and gates it to its top tier; Rentwell ships it natively, un-gated, from day one. | property manager, resident, bookkeeper |
| Audit-ready trust compliance | Beyond fund separation: immutable audit log, three-way reconciliation reports, per-state deposit rules with deadline tracking. Verified-deep in only one player (Rentvine) — it's what PM firms switch vendors over. | broker-owner, bookkeeper |
| Open API + MCP server | Programmatic and external-LLM access to the firm's data under tenant RBAC. Only Rentvine has a beta; incumbents gate APIs to top tiers. Plenipo's agent/MCP infrastructure makes this nearly free to ship. | broker-owner |

### Explicitly out of scope (v1)

- **Renters insurance program** — requires licensed insurance sellers/partnerships; not software-only.
- **PM-company website builder** — separate product line; target customers already have websites.
- **Inspections module** — market accepts an integration path (zInspector/RentCheck pattern); strong post-v1 AI candidate.
- **Maintenance contact center (staffed 24/7 service)** — a service business; the AI intake agent covers the job.
- **Utility billing / convergent billing** — niche adjacent business, deep in only one player.
- **HOA / community-association management** — distinct vertical (boards, violations, dues).
- **Commercial property / CAM reconciliation** — distinct vertical; v1 is long-term residential only.
- **Affordable housing / Section 8 compliance** — heavy specialized compliance, enterprise-tier territory.
- **Tenant benefits / ancillary revenue packages** — partner-dependent monetization.
- **DIY-landlord freemium tier** — different buyer and economics; do not chase down-market.
- **Non-US markets** — the compliance spine (trust rules, FCRA, 1099) is US-specific.

## RBAC model (initial)

- **broker-owner** — full firm administration: trust-account and fee configuration, staff roles/permissions, distribution release, audit-log access. The only role that can change compliance-relevant settings.
- **property-manager** — day-to-day operations on assigned portfolios: leasing pipeline, maintenance dispatch, communications, lease execution. Cannot alter trust-account config, firm settings, or release distributions.
- **bookkeeper** — accounting surface: reconciliation, statements, payment runs, 1099 preparation. Cannot modify leases, screening decisions, or firm configuration.
- **vendor** — sees only work orders assigned to them; uploads invoices and completion evidence. No access to resident PII beyond the work order.
- **owner** (client) — read-only portal over their own properties: statements, reports, documents; single write action: approve/decline over-limit maintenance.
- **resident** — self-service on their own tenancy: pay, autopay, maintenance requests, documents, messages.

Roles bind to capabilities, not screens; baselines are runtime-editable per tenant per the platform's configurable-RBAC model.

## Regulatory constraints

- **State trust-accounting rules (state real-estate commissions)** — client funds segregated from operating funds, full audit trail, monthly (three-way) reconciliation. Implies: fund-separated ledger design, immutable audit log, per-state configuration, reconciliation reports as first-class artifacts.
- **FCRA §604/§615 (tenant screening)** — permissible purpose, applicant consent, adverse-action notices. Implies: screening only via a CRA partner integration, stored adverse-action records, strict access control on reports.
- **Fair Housing Act + HUD guidance on algorithmic screening** — no discriminatory criteria in advertising, screening, or AI lead handling; disparate-impact risk applies to models. Implies: AI agent decisions logged with criteria, human-in-the-loop on adverse outcomes, no protected-class proxies in prompts or models.
- **IRS 1099-MISC/NEC (IRIS/FIRE e-file)** — annual reporting for owner/vendor payments over the threshold ($600 as of 2025 filings). Implies: payee TIN capture, payment categorization in the ledger, an e-file workflow.
- **Money-transmission licensing** — in-house movement of rent between tenants, trust accounts, and owners would trigger licensing. Implies: all money movement through a licensed processor partner; NACHA/PCI scope stays with the processor.
- **State security-deposit statutes** — per-state caps, interest, and return deadlines. Implies: state-configurable deposit rules engine with deadline tracking.
- **CCPA/state privacy + SOC 2 baseline** — tenant PII (application SSNs, financial data) encrypted at rest, tenant-isolated, access-audited; breach process documented.

## Success metrics

- ≥95% of rent payments collected electronically without staff touch, measured in payments telemetry (baseline: industry cites ~80% e-payment adoption).
- ≥60% of maintenance requests fully triaged by the AI agent without staff edits, and median request→vendor-assignment time < 4 business hours (work-order telemetry).
- 100% of trust accounts reconciled within 5 days of month end (reconciliation events in accounting telemetry).
- Median vacancy→lease-signed ≤ 21 days for units run through the leasing pipeline (pipeline timestamps).
- Week-4 activation: ≥70% of onboarded firms have ≥1 owner registered on the owner portal and ≥50% of their residents registered (portal registration telemetry).

## Open questions for plan-system

All decisions delegated to the agent for this run; answers recorded in-place.

1. **Scale target for v1?**
   **Answer (delegated decision):** Design for 100 PM-firm tenants × up to 2,000 doors each; single-region US deployment. Multi-region and enterprise scale are post-v1.
2. **Listing syndication: direct network partnerships or feed-based?**
   **Answer (delegated decision):** v1 ships a standards-based listing feed (ILS/XML export consumable by Zillow-network and similar aggregators) plus a public listing page per vacancy; direct API partnerships are a post-v1 integration, since they require commercial agreements.
3. **AI autonomy default: act autonomously or draft-for-approval?**
   **Answer (delegated decision):** Draft-for-approval by default for anything resident- or money-facing (Fair-Housing and trust exposure); per-firm opt-in to autonomous mode per agent, consistent with the platform's approval-first agent posture.
4. **Data migration from incumbents in v1?**
   **Answer (delegated decision):** CSV/spreadsheet import for properties, units, leases, people, and opening ledger balances — enough to switch a firm off Buildium/AppFolio manually. No automated vendor-specific migrations in v1.
5. **Which payment processor and screening CRA?**
   **Answer (delegated decision):** Both are provider-agnostic abstractions with one concrete integration each in v1; concrete vendor selection belongs to design-architecture (payment partner must support ACH + trust-account sub-ledgering; CRA must be FCRA-compliant with adverse-action support).
