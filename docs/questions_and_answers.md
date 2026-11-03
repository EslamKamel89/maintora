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
