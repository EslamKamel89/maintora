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
