## Batch 1 — Market and Core Domain Decisions

**Date:** 2026-11-03  
**Source:** Interactive requirements clarification, Batch 1  
**Status:** Confirmed by product owner; `SPEC.md` changes proposed, not yet applied.

### Q1 — Initial target market

**Question:** Which market should drive the first pilot's terminology, currency defaults, and operational assumptions?

**Answer (D):** Gulf-wide from the beginning.

**Implementation implications:** Support configurable country, currency, time zone, locale, and week-start per company; avoid hardcoded Saudi/UAE assumptions. Design data formats and country-specific configuration for Gulf markets. Commercial launch sequence, merchant jurisdiction, and local regulatory integrations remain undecided. Do not equate Gulf-wide product support with immediate regulatory compliance in every country.

**SPEC.md updates:** Section 3 should replace the single-country pilot decision with Gulf-wide product compatibility. Section 15 item 1 should separately request the first commercial launch/merchant jurisdiction. Retain section 9's country-by-country legal review.

### Q2 — First maintenance industries

**Question:** Which two trades should be used to validate industry flexibility?

**Answer (E, custom):** The project must adapt to any industry through flexible design.

**Implementation implications:** Keep a common relational core (customer, site, asset, contract, plan, work order), configurable asset categories and typed attributes, and versioned inspection templates. Avoid trade-specific tables and hardcoded workflows unless justified by a general domain invariant. The product should be _configurable for diverse maintenance industries_, not promise that every conceivable industry is supported without additional requirements. Validate the design with representative workflows from at least two distinct trades, still to be selected.

**SPEC.md updates:** Reinforce sections 1, 4 FR-03/FR-07, and 6 with the universal configuration objective. Section 13 Phase 0 should retain cross-industry validation; section 15 item 2 should ask which example workflows to test rather than which trades the product is restricted to.

### Q3 — Maintenance plan scope

**Question:** Should one preventive maintenance plan apply to one asset or multiple assets?

**Answer (C):** Support both from the beginning.

**Implementation implications:** Model plans with a company-scoped asset membership/association instead of requiring only one `asset` foreign key. Define whether group plans create one work order per asset or one work order covering multiple assets; decide how checklist responses, due dates, contract coverage, and idempotency apply to group occurrences. Do not implement a uniqueness key that assumes a single asset without resolving generation semantics.

**SPEC.md updates:** Revise FR-05, FR-06, section 6 occurrence uniqueness, section 7.2 scheduling, and section 15 item 3. Add tests for cross-tenant asset membership, mixed customer/contract coverage, and concurrent generation.

### Q4 — Technician assignments

**Question:** How should technicians be assigned to work orders in the MVP?

**Answer (B):** Multiple technicians per work order.

**Implementation implications:** Use an explicit company-scoped assignment relation, recording technician, assignment state, actor, and timestamps. Preserve assignment history and enforce company membership. Define whether any assigned technician may start/complete a work order, whether a lead technician is required, and whether individual acceptance is needed. Avoid a single-assignee-only schema.

**SPEC.md updates:** Revise FR-06, FR-08, section 5 authorization matrix where needed, section 6 suggested models, section 7.3 workflow, and section 15 item 5. Add tests for multiple assignees, unassigned technician access, and membership deactivation.

### Q5 — Contract pricing and billing

**Question:** How much financial functionality should the first version include?

**Answer (C):** Contract value, invoices, and payment tracking.

**Implementation implications:** Distinguish invoices issued by maintenance companies to their customers from SaaS subscription billing. Define invoice status, numbering, currency, partial payments, allocations, and financial history. Keep accounting ledger, regulated e-invoicing, and tax integrations outside scope until explicitly decided. Financial mutations require tenant isolation, transaction-safe updates, and immutable/auditable posted records.

**SPEC.md updates:** Revise section 1 MVP non-goals to permit limited operational invoicing/payment tracking; add a functional requirement for customer invoicing and receipts/payments; update sections 6, 11, 13, 14, and 15 item 4. Keep subscription billing (FR-11) separate. Country-specific legal/tax compliance remains uncommitted.

### Cross-cutting decisions and unresolved details

- Confirmed: Gulf-wide configurable product; maintenance-industry flexibility; single- and multi-asset plans; multiple work-order technicians; customer invoicing and payment tracking.
- Unresolved: group-plan work-order cardinality; group asset/customer boundaries; team completion authority; invoice/payment lifecycle; tax treatment; initial commercial jurisdiction.
- **Change control:** These decisions amend the intended requirements but do not themselves modify `SPEC.md`, migrations, or application code. Update `SPEC.md` before implementing affected features, preserving existing tenant isolation, audit, and idempotency requirements.

## Batch 2 — Maintenance Workflows and Financial Rules

**Date:** 2026-11-03  
**Source:** Interactive requirements clarification, Batch 2  
**Status:** Confirmed by product owner; `SPEC.md` updates proposed, not yet applied.  
**Baseline checked:** `main` branch, `SPEC.md` (blob `52892cf089d74c1561dc40913dad0d4c691bd2c9`); existing `docs/questions_and_answers.md` includes Batch 1.

### Q6 — Multi-asset maintenance plans

**Question:** When a maintenance plan covers 10 assets and becomes due, how should the system generate work orders?

**Answer (C):** Support both, configurable per plan.

**Implementation implications:** Introduce an explicit generation mode (`per_asset` or `combined`) on the plan. Per-asset mode generates one work order per covered asset per occurrence; combined mode generates one work order with multiple covered assets. Work-order/asset associations must support multiple assets, while preserving site/customer consistency and inspection evidence attributable to the correct asset. Define deterministic idempotency keys: `(company, plan, occurrence, asset)` for per-asset orders and `(company, plan, occurrence)` for combined orders, or equivalent explicit source keys with database constraints. Snapshot planned coverage at generation to avoid changing historical work orders when plan membership changes. Plan edits must not silently duplicate or remove already generated occurrences.

**SPEC.md updates:** Revise FR-05 and FR-06; replace section 6 invariant 6's single uniqueness assumption with generation-mode-aware constraints; revise sections 7.2 and 13 Phase 3; replace section 15 item 3 with the confirmed policy. Clarify checklist response ownership and mixed-site combined orders in a future batch.

### Q7 — Assets belonging to different customers

**Question:** Can one maintenance plan contain assets from multiple customers?

**Answer (A):** No. All assets must belong to the same customer.

**Implementation implications:** Each plan has one owning customer; every included asset must be at a site belonging to that customer and the same tenant. Validate both initial selection and subsequent changes to asset/site/customer associations. Enforce through tenant-scoped services, transactional workflows, and database protections where feasible. A plan may cover multiple sites of the same customer unless a later decision restricts it. Contract-backed plans must additionally validate contract customer and coverage for all included assets.

**SPEC.md updates:** Add the same-customer plan invariant to FR-05 and section 6; extend FR-02/FR-04 relationship validation, section 7.2 generation checks, and tenant/cross-customer negative tests.

### Q8 — Multiple technician responsibilities

**Question:** How should work-order execution work when several technicians are assigned?

**Answer (B):** One lead technician controls completion; others contribute.

**Implementation implications:** Model multiple tenant-scoped work-order assignments with a designated lead; ensure no more than one active lead per work order through appropriate constraints. Assigned contributors may record authorized notes/checklist/evidence; only the lead may submit technician completion, subject to checklist requirements. Manager review/closure remains a separate permission per FR-06. Preserve assignment history and audit lead reassignment. Define how the system behaves if the lead is deactivated or unavailable; do not implicitly grant all contributors completion rights.

**SPEC.md updates:** Revise FR-06, FR-08, authorization matrix in section 5, section 6 assignment models/invariants, section 7.3, and section 15 item 5. Test lead-only completion, contributor edits, reassignment, and inactive memberships.

### Q9 — Invoice and payment management

**Question:** Which invoice/payment workflow should Maintora support?

**Answer (B):** Invoices, partial payments, credit notes, and refunds.

**Implementation implications:** Add a bounded customer-billing domain distinct from SaaS subscription billing. Track invoice numbering, currency, status, line items/taxes, amounts due, partial payment allocation, credit notes referencing issued invoices, and refunds referencing eligible payments. Use decimal money amounts and transaction-safe balance updates; audit financial actions and avoid destructive edits to issued documents. Define whether refunds represent recorded external transactions or actual payment-provider disbursements; no external money movement is implied by this answer. Full general ledger/accounting and regulated e-invoicing remain excluded unless explicitly approved.

**SPEC.md updates:** Add a customer invoicing/payment functional requirement, and update section 1 non-goals, sections 6 (models/invariants), 7 (financial workflows), 9 (audit/security), 11 (exports), 13 (phases), 14 (deferred features), and 15 item 4. Keep FR-11 SaaS subscriptions separate.

### Q10 — Multi-country taxation

**Question:** How should Maintora handle different tax requirements across Gulf countries?

**Answer (B):** Configurable tax rules per country/company, without government integrations.

**Implementation implications:** Store company/country tax configuration with effective dates where appropriate; calculate invoice taxes from configured rules and snapshot applied rates/amounts on issued documents. Support exemptions or zero rates when configured, and maintain an auditable record of overrides. Configuration is not a guarantee of legal compliance. Do not hardcode Gulf-wide rates or promise government e-invoicing submissions. Define whether invoices can carry multiple tax components and how tax-inclusive pricing and rounding behave in a later batch.

**SPEC.md updates:** Add taxation requirements to the new customer-billing section, revise section 10 localization and section 9 jurisdiction-specific legal review, and retain regulated e-invoicing as deferred in section 14.

### Cross-cutting decisions and remaining questions

- **Confirmed:** Configurable per-asset/combined work-order generation; same-customer asset membership in each plan; one lead technician plus contributors; partial payments, credit notes, and refunds; configurable tax rules without government integrations.
- **Still open:** Whether combined work orders may span sites; inspection response granularity; contract coverage for multi-asset plans; lead replacement/acceptance; invoice issuance/immutability and payment allocation; tax rounding/inclusivity; refund processing boundaries; subscription/trial and retention details.
- **Change control:** These clarifications extend the intended MVP and conflict with portions of `SPEC.md` v1.0 (especially single-asset work orders, single assignee, and invoicing as deferred). Update `SPEC.md` explicitly before implementation. No code, migration, or GitHub modification is implied by this report.

## Batch 3 — Execution Boundaries, Inspections, and Financial Documents

**Date:** 2026-11-03  
**Source:** Interactive requirements clarification, Batch 3  
**Status:** Confirmed by product owner; `SPEC.md` revisions proposed, not applied.  
**Repository baseline:** `EslamKamel89/maintora`, branch `main`; `SPEC.md` blob `52892cf089d74c1561dc40913dad0d4c691bd2c9`. Existing Q&A contains Batches 1 and 2.

### Q11 — Combined work orders across sites

**Question:** Can a single combined work order include assets at different sites belonging to the same customer?

**Answer (A):** Yes, across multiple sites of the same customer.

**Implementation implications:** A combined work order cannot rely on a single `site` and `asset` foreign key as its complete coverage representation. Use a work-order-to-asset association, retaining each asset's site and validating one customer and one tenant across all covered assets. Any primary site field must be clearly informational rather than an exclusive scope restriction. Define site-level scheduling, technician visibility, and completion/evidence tracking for multi-site work. Snapshot covered assets/sites at generation so later changes do not rewrite history. Do not infer that cross-customer work orders are allowed.

**SPEC.md changes required:** Update FR-05 and FR-06 (multi-site work-order coverage), section 6 (work-order relationships and cross-customer invariants), section 7.2 (generation), section 7.3 (execution), and Phase 4 acceptance tests. Replace assumptions that a work order always has exactly one site and asset.

### Q12 — Inspection checklist granularity

**Question:** When one work order covers multiple assets, how should inspection responses be recorded?

**Answer (C):** Support both asset-specific and shared checklists.

**Implementation implications:** Distinguish inspection instances scoped to a work order from instances scoped to a particular covered asset. A shared checklist applies to the overall job; asset-specific checklists apply independently to each selected asset. Preserve the immutable published template version for every instance, respondent and timestamps for answers, and tenant/asset authorization for attachments. Define which checklists are mandatory before the lead technician can submit completion. Prevent answers from being attached to assets outside the work order. Versioned checklist templates remain industry-configurable.

**SPEC.md changes required:** Extend FR-07 with explicit checklist scope and completion rules; update FR-06 completion validation, section 6 inspection associations/invariants, section 7.3 execution, and Phase 4 verification. Add tests for mixed shared/asset-specific inspections and required-answer enforcement.

### Q13 — Contract coverage for maintenance plans

**Question:** How should maintenance plans interact with contracts?

**Answer (C):** Plans may exist independently, but contract-linked plans must only generate work within valid coverage.

**Implementation implications:** Keep the contract relationship optional. For contract-linked plans, validate tenant, customer, covered assets, contract status, and applicable coverage dates before generating each occurrence. A multi-asset plan with partial contract coverage must not silently produce unauthorized work; define whether to reject the plan, skip uncovered assets, or split occurrences in a later decision. Standalone plans remain allowed. Preserve historical work orders after expiration/termination and make generation idempotent under retries.

**SPEC.md changes required:** Clarify FR-04 and FR-05 optional contract linkage and coverage rules, FR-06 order consistency, section 6 cross-entity invariants, and section 7.2 generation behavior. This largely confirms the existing optional-contract direction but requires explicit multi-asset behavior.

### Q14 — Invoice issuance and editing

**Question:** What should happen after an invoice is officially issued?

**Answer (B):** Lock issued invoices; corrections use credit notes and replacement invoices.

**Implementation implications:** Introduce a draft-to-issued transition. Issued invoice details, tax snapshots, totals, customer identity, and invoice number must be immutable through normal edit paths. Corrections require traceable credit notes and, where needed, replacement invoices linked to the original. Enforce status transitions in transactional services with authorization and audit records. Payments and refunds are separate append-only financial events or auditable transactions; do not modify historical invoice amounts to reflect them. Define partial credit notes and numbering policies in later batches.

**SPEC.md changes required:** Add explicit issuance, immutability, correction, and audit rules to a new customer-billing functional requirement and section 6 invariants. Extend sections 7, 9, 11, and 13 with billing workflows, exports, and tests. Remove customer invoicing from the blanket deferred list in section 14 while keeping regulated e-invoicing and full accounting deferred.

### Q15 — Refund processing

**Question:** Should Maintora actually transfer money to customers when processing refunds?

**Answer (A):** Record refunds performed outside Maintora (bank transfer, cash, etc.).

**Implementation implications:** Maintainora records a refund's amount, currency, date, method, reference, actor, reason, and links to the relevant payment/customer; it does not initiate a bank transfer or payment-provider refund. Validate refundable balances, prevent double recording with appropriate unique external references or idempotency controls, and preserve audit history. Marking a refund recorded must not imply that money moved through the platform. Distinguish credit notes (invoice adjustment) from refunds (money returned).

**SPEC.md changes required:** Clarify the new customer-billing requirement and financial workflow in section 7; add refund authorization, integrity, and audit rules in sections 6 and 9. Keep payment disbursement integrations deferred in section 14; FR-11 SaaS subscription billing remains a separate domain.

### Consolidated decisions and remaining questions

- **Confirmed:** Combined work orders may span multiple sites of one customer; shared and asset-specific inspections both supported; contract linkage optional but linked work strictly constrained by coverage; issued invoices immutable with credit-note/replacement corrections; refunds recorded externally only.
- **Unresolved:** Multi-site work-order scheduling/closure semantics; mandatory checklist selection and whether asset-specific checklists are required for every asset; behavior when only some assets have valid contract coverage; lead reassignment; invoice credit-note/partial-payment allocation, tax inclusivity/rounding, and numbering; subscription limits and retention.
- **Change control:** These decisions are requirements clarifications, not evidence of implemented code. Update `SPEC.md` explicitly before implementation; preserve tenant isolation, append-only audit, concurrency safety, and idempotent scheduling. This report is an independent Markdown section intended to be appended, not to replace earlier batches.

## Batch 4 — Scheduling, Contract Coverage, and Completion Rules

**Date:** 2026-11-03  
**Source:** Interactive requirements clarification, Batch 4  
**Status:** Product-owner decisions confirmed; `SPEC.md` revisions proposed, not applied.  
**Repository baseline:** `EslamKamel89/maintora`, `main`; `SPEC.md` blob `52892cf089d74c1561dc40913dad0d4c691bd2c9`. Existing Q&A through Batch 3 verified.

### Q16 — Partial contract coverage

**Question:** A maintenance plan covers 10 assets, but the linked contract covers only 8. What should Maintora do?

**Answer (B):** Generate work orders only for the 8 covered assets.

**Implementation implications:** Preserve the full plan asset membership but compute eligible assets per scheduled occurrence based on contract/customer/tenant ownership, contract status, effective dates, and asset coverage. In per-asset generation mode, create orders only for eligible assets; in combined mode, create a combined order containing only the eligible subset. If no assets qualify, generate no work order. Record which assets were excluded and why in scheduler results so silent omission does not conceal missed service. Snapshot the eligible asset set on generated orders. Idempotency must prevent duplicate generation under retries even when coverage changes; define an explicit policy for newly eligible assets after an occurrence has already been generated (still open). Validate cross-tenant and cross-customer relationships regardless of coverage.

**SPEC.md revisions:** FR-04 contract coverage; FR-05 scheduling and partial eligibility; FR-06 multi-asset work orders; section 6 uniqueness/invariants; section 7.2 scheduler workflow; section 12 scheduling tests. Replace any all-assets-or-nothing assumption with the confirmed skip-uncovered policy.

### Q17 — Multi-site work-order scheduling

**Question:** A combined work order includes assets at several sites. How should scheduling work?

**Answer (A):** One overall scheduled date and due date for the entire work order.

**Implementation implications:** Store a single scheduled date and due date on the work order, while retaining separate site/asset associations for execution, inspection evidence, and access checks. A multi-site work order does not imply individual site-level due dates in MVP. Clarify whether site-level progress is tracked separately from the overall work-order status in a later decision. Due-date calculations must use the tenant's configured time zone.

**SPEC.md revisions:** FR-06 work-order fields must no longer require exactly one site/asset for combined orders; section 6 relationships and section 7.3 execution must support multi-site associations. Clarify shared scheduling fields and add multi-site validation tests.

### Q18 — Mandatory inspections

**Question:** For a work order covering multiple assets, which inspections must be completed before the lead technician can submit completion?

**Answer (C):** Manager configures which inspections are mandatory for each work order.

**Implementation implications:** Allow managers to select applicable shared and asset-specific published checklist versions, and mark which assigned inspections are mandatory. The completion service must verify every _configured mandatory_ inspection is satisfied before allowing lead-technician completion; optional inspections may remain incomplete. Snapshot the checklist version, asset/work-order association, and required status so later template changes do not alter existing obligations. Do not interpret this choice as permission to bypass requirements once the work order is in progress; authorization and timing for modifying mandatory assignments require a later decision. Record all modifications in the audit trail.

**SPEC.md revisions:** FR-06 completion requirements; FR-07 inspection assignments/versioning; section 5 manager versus technician permissions; section 6 inspection associations and invariants; section 7.3 completion workflow. Add tests for optional versus mandatory shared/asset inspections, permissions, and immutability of published versions.

### Q19 — Lead technician availability

**Question:** What happens if the lead technician is unavailable or their membership is deactivated?

**Answer (A):** A manager must explicitly assign a new lead.

**Implementation implications:** Never automatically promote contributors or grant them lead-only completion authority. Block lead-only transitions while no eligible active lead exists; allow authorized manager/owner reassignment, preserving previous assignment history and audit actor/time. Validate that the new lead has active membership in the correct company and is assigned to the work order. Decide later whether an owner/admin can directly submit completion when no lead is available; existing role permissions should not be silently expanded.

**SPEC.md revisions:** FR-06 and FR-08 assignments/actions; section 5 authorization matrix; section 6 assignment invariants; section 7.3 lead handover; section 9 audit. Test deactivated lead, cross-tenant reassignment, and contributor completion denial.

### Q20 — Recurring maintenance plan modifications

**Question:** If a manager changes the assets, frequency, or schedule of an active maintenance plan, what should happen to already generated work orders?

**Answer (A):** Preserve generated work orders; changes affect future occurrences only.

**Implementation implications:** Treat generated work orders as historical snapshots of the plan and covered assets at creation. Editing a plan must not mutate or delete existing scheduled, assigned, in-progress, or completed orders. Version or audit meaningful plan changes, and recalculate only future ungenerated occurrences. Preserve deterministic occurrence identity and use transactions/locking so concurrent edits and scheduler runs cannot create duplicates or skip due work. Explicit manual edits to an existing work order, if allowed, are separate authorized/audited actions rather than implicit plan synchronization.

**SPEC.md revisions:** FR-05 recurrence/editing; FR-06 historical order snapshot; section 6 immutable occurrence identity; section 7.2 scheduler cursor and edit concurrency; section 9 audit; section 12 concurrency/integrity tests.

### Cross-batch consistency and outstanding decisions

- **Confirmed:** Partial contract coverage filters eligible assets rather than rejecting a plan; combined multi-site orders share one schedule; managers choose mandatory inspections per order; unavailable leads require explicit manager reassignment; plan edits apply only to future ungenerated occurrences.
- **Still open:** What happens if coverage expands after a partially generated occurrence; whether combined work orders track progress by site; when mandatory inspection assignments may change; how to handle contract expiration during an in-progress order; invoice/payment allocation, tax calculation details, subscription limits, and retention.
- **Change control:** This section is append-only and does not modify `SPEC.md` or code. Before implementing affected workflows, revise `SPEC.md` to reconcile its original single-asset/single-site assumptions and retain tenant isolation, transaction safety, idempotency, and auditability.
