# Rentwell — Architecture

Consumes: [PLAN.md](PLAN.md) (epics, modules, RBAC, integration surface), [SPEC.md](SPEC.md) (capabilities, personas, metrics). Non-default choices carry an ADR in [DECISIONS.md](DECISIONS.md). Project naming uses the `Rentwell.*` prefix per PLAN (the legacy `The<Domain>` convention is retired).

## Context (C4 L1)

External actors: PM firm staff (broker-owner, property manager, bookkeeper), residents, property owners, vendors, external LLM clients (via MCP), and four external systems — payment rails (Stripe Connect, ADR-0002), screening CRA (TransUnion SmartMove, ADR-0003), email/SMS providers (channel connectors), and listing aggregators (ILS feed consumers). The IRS (IRIS 1099 files) is a manual-export actor in v1 (ADR-0006).
Diagram: `docs/diagrams/c1-context.md`

## Containers (C4 L2)

Each is an Aspire resource composed by `Rentwell.AppHost`:

- **Rentwell.Api** — the modular-monolith backend: minimal APIs per bounded context, application modules, MAF agent runtime, in-process Quartz scheduler (ADR-0009), outbox dispatcher.
- **Rentwell.OpenPlatform** — MCP server + API-client credential management (differentiator epic 11); enforces the same RBAC policies as the API.
- **Rentwell.Web** — Vite + React + TypeScript + shadcn/ui SPA serving three shells: operator console, resident portal, owner portal.
- **PostgreSQL** (+ pgvector) — operational store, one logical DB, tenant-filtered; pgvector for agent memory/knowledge.
- **Redis** — distributed cache: sessions, idempotency replay, rate-limit windows.
- **Blob storage** — documents (leases, invoices, statements); metadata + PII classification live in Postgres.
- **Audit store** — append-only audit events outside the operational DB (dedicated Postgres schema on a separate database in v1's single server, table partitioned, INSERT-only role; export to blob archive).
Diagram: `docs/diagrams/c2-containers.md`

## Components (C4 L3) — Rentwell.Api

Endpoint groups (one per bounded context) → command/query handlers per module (CQRS-lite, ADR-0010) → EF Core via `Rentwell.Infrastructure` (tenancy filters, outbox) → adapters (`Infrastructure.Stripe`, `Infrastructure.SmartMove`, `Infrastructure.Email`, `Infrastructure.Sms`, `Infrastructure.Azure`). MAF agents (Assistant, Leasing, MaintenanceIntake, AccountingExtraction) run in-process with tools that call the same application handlers under the caller's RBAC context.
Diagram: `docs/diagrams/c3-components-api.md`

## Solution layout

One `Application.<Context>` per PLAN epic; connectors and cloud specifics behind `Infrastructure.*` interfaces.

```
src/
  Rentwell.AppHost/                    ← Aspire AppHost (composes everything above)
  Rentwell.ServiceDefaults/            ← OTel + health checks + resilience defaults
  Rentwell.Api/                        ← minimal APIs grouped by bounded context; hosts agents + scheduler
  Rentwell.Application/                ← shared app services: audit, idempotency, outbox, tenancy context
  Rentwell.Application.Portfolio/      ← epic 2 (properties, units, owners, people, documents)
  Rentwell.Application.Leases/         ← epic 2 (lease lifecycle, in-house e-sign ADR-0004, deposits)
  Rentwell.Application.Payments/       ← epic 3 (charges, autopay, late fees, payouts)
  Rentwell.Application.Accounting/     ← epics 5+9 (GL, trust funds, reconciliation, statements, 1099)
  Rentwell.Application.Maintenance/    ← epic 6 (requests, work orders, vendors, approvals)
  Rentwell.Application.Leasing/        ← epic 7 (listings, feed, applications, screening)
  Rentwell.Application.Communications/ ← epic 8 (templates, bulk, threads)
  Rentwell.Application.Agents/         ← epic 10 (MAF agent definitions, tools, decision log)
  Rentwell.Domain/                     ← entities, value objects, domain events (all contexts)
  Rentwell.Infrastructure/             ← EF Core, multi-tenancy filters, outbox, Quartz, [Pii] pipeline
  Rentwell.Infrastructure.Azure/       ← Key Vault, Blob, Entra External ID glue (ADR-0008)
  Rentwell.Infrastructure.Stripe/      ← IPaymentRails adapter (ADR-0002)
  Rentwell.Infrastructure.SmartMove/   ← IScreeningProvider adapter (ADR-0003)
  Rentwell.Infrastructure.Email/       ← email channel connector
  Rentwell.Infrastructure.Sms/         ← sms channel connector
  Rentwell.OpenPlatform/               ← epic 11: MCP server host + API-client management
  Rentwell.Web/                        ← Vite SPA (console + resident portal + owner portal)
tests/
  one test project per source project; Rentwell.IntegrationTests uses Testcontainers (Postgres, Redis);
  Playwright E2E added by verify-runtime when the SPA lands
```

Epic→module traceability matches PLAN's module list; portals are SPA route areas + portal-scoped policies, not a domain module.

## Cross-cutting wiring

- **AuthN**: Microsoft Entra External ID (CIAM) via OIDC — one authority for staff and portal users; identity gives *who*, Rentwell RBAC gives *what* (ADR-0008).
- **RBAC**: PLAN's `<Module>.<Action>` policies as named ASP.NET Core authorization policies; role→policy baselines runtime-editable per firm (platform configurable-RBAC pattern); UI gates on the same policy names.
- **Multi-tenancy**: firm id resolved from the authenticated principal's firm membership (never from client input), flowed via `ITenantContext`, enforced with EF Core global query filters; tenant id column on every domain table.
- **Observability**: OTel traces/metrics/logs via `ServiceDefaults` → Azure Monitor (OTLP); health endpoints on every resource; audit events (who/what/when/tenant/before-after) written through the outbox to the audit store.
- **Resilience**: Polly standard handlers on all outbound calls (Stripe, SmartMove, email/SMS, blob); circuit-breaker + retry-with-jitter; webhook processing is idempotent by event id.
- **Caching**: Redis — session state, idempotency replay records (24h TTL), rate-limit windows (sliding, per tenant+endpoint), short-TTL (60s) read-through for portal dashboards.
- **Background work**: Quartz.NET in-process (ADR-0009) running PLAN's job inventory (charge generation, autopay, late fees, deposit deadlines, distributions, statements, feed export, 1099 batch); every external side effect goes through the transactional outbox.
- **Idempotency**: `Idempotency-Key` header on all non-GET writes; replay records in Redis (24h), fallback table in Postgres for money-movement endpoints.

## Cloud topology

- **Provider**: Azure (workflow.json `cloud: azure`); Terraform is the IaC tool; per-customer dedicated infra follows the platform SaaS model.
- **Compute**: Azure Container Apps environment — API, OpenPlatform, and Web containers (ADR-0001).
- **Data**: Azure Database for PostgreSQL Flexible Server (zone-redundant HA), pgvector enabled.
- **Vector**: pgvector (guardrail default; no connector brings an alternative).
- **Secrets**: Azure Key Vault via managed identity; `IOptions<T>` validated at startup; no secrets in appsettings.
- **Identity**: Entra External ID (CIAM) tenant (ADR-0008).
- **Storage**: Azure Blob Storage (documents, audit archive, IRIS export files).
- **CDN / Edge**: none in v1 (single region, SPA served from Container Apps); revisit post-v1.
- **Networking**: single region (East US 2), VNet-integrated ACA environment, private endpoints for Postgres/Redis/Key Vault/Blob, public ingress only on Api, OpenPlatform, and Web. Data residency: US only (SPEC).

## Data model (concrete)

EF Core entities per context (migrations per context schema, `rentwell_<context>`); every table carries `FirmId` (tenant); `[Pii]` marks fields flowing into audit/export. Key entities and relations (full sketch in PLAN):

- Portfolio: `Firm`, `Property` 1—n `Unit`, `OwnershipEntity` (tax id `[Pii]`, encrypted), `Person` (`[Pii]` contact/DOB), `Document` (blob ref + PII class).
- Leases: `Lease` (unit + parties, term, rent schedule, renewal chain), `DepositRecord` (state-rule flags), `SignatureEnvelope` (in-house e-sign: intent, consent, hash, audit trail — ADR-0004).
- Payments: `RentCharge`, `PaymentTransaction` (partner ids only, no PANs/bank numbers), `PayoutRun` → `Payout`.
- Accounting: `LedgerAccount` (fund class: trust|operating), `JournalEntry` (immutable after post; period-locked), `BankStatementImport`, `ReconciliationRun`, `OwnerStatement`, `Distribution`, `Tax1099Record` (TIN `[Pii]`, encrypted).
- Maintenance: `MaintenanceRequest`, `WorkOrder`, `Vendor` (tax id `[Pii]`), `VendorInvoice`, `ApprovalRequest`.
- Leasing: `Listing`, `RentalApplication` (SSN `[Pii]`, encrypted, restricted policy), `ScreeningReport` (restricted, retention-limited), `AdverseActionRecord`.
- Communications: `MessageThread`, `MessageTemplate`, `OutboundMessage` (channel, delivery state).
- Agents: `AgentRun`, `AgentDecisionLog` (inputs, criteria, outcome, approval state).
- Compliance data: `StateComplianceRuleSet` — versioned, effective-dated config data (ADR-0007).

Migrations strategy: EF Core migrations per context, applied by the AppHost in dev and by CI migration bundles in deployment; no `EnsureCreated`.

## API surface (concrete)

- Groups: `/api/v1/portfolio`, `/api/v1/leases`, `/api/v1/payments`, `/api/v1/accounting`, `/api/v1/maintenance`, `/api/v1/leasing`, `/api/v1/communications`, `/api/v1/portal/resident`, `/api/v1/portal/owner`, `/api/v1/admin` (firm config + RBAC editor), `/api/connectors/<name>/webhook/...` (payments, screening, sms inbound).
- Versioning: URL segment. Errors: Problem Details (RFC 7807). Writes: `Idempotency-Key` required on non-GET (24h replay). CORS: explicit allow-list per environment (SPA origin + registered OpenPlatform clients); no wildcards anywhere.
- Rate limits: per tenant 100 req/10s default; portal endpoints 30 req/10s per user; webhook routes exempt from tenant limits but signature-verified; OpenPlatform per-client quotas.
- Public API (epic 11): same `/api/v1` surface exposed to API-key/OAuth clients managed in OpenPlatform; MCP tools map 1:1 to public endpoints.

## MAF agents

All agents run on MAF (no custom orchestration loops), persist conversations, write `AgentDecisionLog` entries, and default to draft-for-approval for resident- or money-facing actions (per-firm, per-agent autonomy opt-in). No protected-class proxies in prompts (Fair Housing).

- **Rentwell Assistant** — operator chatbot in the console slide-over. Tools: domain queries (portfolio, ledger, work orders, pipeline) + drafting actions, all under the caller's RBAC policies. Memory: pgvector per firm; conversations persisted per user.
- **Leasing Agent** — responds to listing inquiries 24/7, proposes showings. Tools: listing/availability lookup, comms draft/send (send only when firm opted in), applicant invite. Triggered by inbound lead events.
- **Maintenance Intake Agent** — conversational intake in the resident portal: gathers details/photos, suggests triage + vendor, flags self-resolvable issues. Tools: tenancy lookup, work-order draft, vendor suggestion, knowledge base (pgvector).
- **Accounting Extraction Agent** — vendor invoice/receipt → draft bill + GL coding for bookkeeper approval. Tools: platform document reader, chart-of-accounts lookup, draft journal entry.

## SPA architecture

- Routing: React Router; three route areas — `/app` (operator console), `/resident`, `/owner` — one SPA, persona decided by role.
- State: TanStack Query for server state; local UI state in components/Zustand only where needed.
- Components: shadcn primitives (owned/copied) + shared `DataTable` (TanStack Table) + shared `Form` (react-hook-form + zod) + the slide-over chatbot panel (platform chrome: sidebar nav, top bar with firm switch + user menu).
- Feature folders per bounded context (`features/portfolio`, `features/leases`, …); portal areas compose the same feature components with portal-scoped queries.

## Diagrams checked into the repo

- `docs/diagrams/c1-context.md`
- `docs/diagrams/c2-containers.md`
- `docs/diagrams/c3-components-api.md`
