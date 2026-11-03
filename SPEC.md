# Maintenance Contract Management SaaS — Product & Technical Specification

**Version:** 1.0  
**Date:** 2026-11-03  
**Status:** Approved architectural direction; implementation specification  
**Primary market:** Gulf-region small and medium maintenance contractors  
**Product type:** Industry-agnostic, multi-tenant B2B SaaS  
**Implementation:** Django modular monolith

## 1. Vision and goals

Build a configurable maintenance contract and service management platform that supports multiple equipment categories and industries without separate codebases. Companies manage customers, sites, assets, service contracts, preventive maintenance schedules, work orders, technician assignments, and inspection evidence.

**Product goals:** Reduce missed preventive visits and contract renewals; provide traceable service history; enable onboarding of different maintenance trades through configuration; charge recurring subscriptions.

**Engineering goals:** Exercise Django ORM, forms, templates, class-based views, authentication, authorization, migrations, constraints, transactions, background tasks, and testing. Prefer server-rendered UI with HTMX; do not build a SPA solely for this project.

**Non-goals for MVP:** Full ERP/accounting, payroll, inventory purchasing, route optimization, native mobile apps, IoT, predictive maintenance, arbitrary workflow builders, customer-defined SQL schemas, country-specific legal certification, and regulated e-invoicing integration.

## 2. Users, tenancy, and terminology

- **Platform operator:** Runs the SaaS, handles plans, billing support, and operational oversight. Access to customer data must be explicitly controlled and audited.
- **Tenant/company:** Maintenance service provider subscribing to the platform. A tenant may serve multiple industries.
- **Owner/Admin:** Manages company configuration, staff, customers, contracts, and subscription.
- **Dispatcher/Manager:** Manages schedules, work orders, assignments, and review.
- **Technician:** Sees assigned work, updates permitted states, completes checklists, uploads evidence.
- **Viewer:** Read-only access to authorized company records.
- **Customer:** Organization receiving maintenance service; customer portal is deferred.
- **Site:** Customer location containing equipment.
- **Asset:** Maintainable equipment or unit at a site.

All tenant-owned records must belong to exactly one company. A user may have memberships in multiple companies and must explicitly select the active company. Role applies to membership, not globally to user.

## 3. Commercial assumptions and validation

Initial target: small-to-medium Gulf maintenance contractors, without restricting the software to a trade. Arabic RTL first, English-ready. Currency, time zone, locale, and week-start are company settings. Suggested pricing hypothesis: USD 25–60 equivalent monthly, with tiering by active assets/users; validate before implementation of paid plans. This is not a proven market price.

Before expanding scope, interview at least 5–10 potential contractors from at least two trades. Validate the current process, renewal failures, work-order tracking, willingness to pay, and onboarding friction. Seek at least two serious pilot commitments. Track activation, weekly active companies, completed work orders, renewal alerts acted upon, churn, and support burden.

## 4. MVP functional requirements

### FR-01 — Accounts and company setup
- Email/password login, password reset, verified email where practical, secure session management.
- Create company; configure name, country, default currency, time zone, language, contact details.
- Invite users by email with expiring, single-use invitations; assign roles; deactivate membership without deleting history.
- Company switcher for users with multiple memberships; active tenant resolved on every request.
- A company can have multiple asset categories, including custom ones.

### FR-02 — Customers and sites
- Customer CRUD: name, organization/contact details, optional external reference, active status, notes.
- Site CRUD: customer, label, structured address, country, location notes, contact, status.
- Customer and site lists support search, pagination, filters, and safe archive (not destructive deletion).
- Prevent linking sites or assets to records belonging to another company.

### FR-03 — Asset categories and dynamic attributes
- Category CRUD with company ownership, name, description, and active status.
- Define typed attributes per category: key, localized label, type (text, integer, decimal, boolean, date, choice), required flag, choices, display order, optional constraints.
- Store core asset fields relationally: company, site, category, name, optional serial number, status, installation date, notes.
- Store category-specific attribute values in JSON, validated against the category's active attribute definitions at form and service boundaries.
- Changing a definition must not silently invalidate historical asset data. In MVP, block destructive changes to definitions already used by assets; allow additive fields and safe label/order changes. Record schema version if future migrations require it.
- Asset status: active, inactive, retired. Retired assets remain in history and cannot receive new preventive plans.
- Support asset lookup by customer/site/category/status and show full service history.

### FR-04 — Maintenance contracts
- Contract: tenant, customer, identifier, title, start/end dates, status, coverage notes, optional price/currency, renewal reminder lead days.
- Contracts can cover multiple assets; covered assets must belong to sites owned by the contract's customer.
- Status: draft, active, expired, terminated. Activate only when dates and coverage are valid. Expiration is determined by dates and reconciled by scheduled jobs.
- Explicit contract renewal creates a new term or linked successor record; preserve prior term and audit history. Never silently extend dates.
- Alerts for upcoming expiration and expired contracts. No claim that the platform provides legal compliance guarantees.
- Decide contract overlap policy per business: MVP permits overlapping contracts but warns on potential duplicate coverage; it must not silently create duplicate work orders.

### FR-05 — Maintenance plans and recurring scheduling
- Plan belongs to tenant, asset, and optionally contract; defines title, checklist template, frequency, start date, next due date, assigned team/technician optional, active state.
- MVP frequency: every N days, weeks, or months; company-local scheduling date, not complex cron expressions.
- Monthly recurrence must define end-of-month behavior consistently (clamp to last valid day); use calendar arithmetic, not fixed 30-day offsets.
- Plans can be paused/resumed; historical work orders remain unchanged.
- A periodic background task generates work orders for a configurable lookahead window.
- Generation must be idempotent under retries/concurrency using a deterministic plan+occurrence key with a database uniqueness constraint.
- Do not generate work orders beyond contract coverage when a plan requires an active contract. Manual corrective work orders may exist without a plan.
- Store due date, generation source, and immutable occurrence identity on each generated order.

### FR-06 — Work orders
- Work order fields: tenant, number, site, asset, optional contract/plan, type (preventive/corrective), priority, description, scheduled/due dates, assignee, status, timestamps, notes.
- Lifecycle: scheduled → assigned → in_progress → completed → closed; support cancelled and explicit reopened transitions.
- Transition authorization and required data are enforced in domain services, not only in templates.
- Technician can begin assigned work, record notes, complete checklist, and submit completion. Manager can assign, review, close, cancel, and reopen.
- Completion requires configured mandatory checklist responses (if a template is attached); closing requires completion and manager review.
- Preserve assignment history, state transitions, and timestamps in append-only audit/event records.
- Work orders must not reference assets/sites/contracts outside their tenant or inconsistent customer coverage.

### FR-07 — Configurable inspection checklists
- Company-owned inspection templates with name, category optional, active status, and version.
- Ordered questions: text, boolean, number, choice, short text, optional photo/evidence requirement, required flag, validation bounds/options.
- Template publishing creates an immutable version; edits create a new version rather than altering questions for existing work orders.
- Work orders snapshot/reference the published template version assigned at generation or creation.
- Record responses with respondent, timestamp, typed value, and evidence references; validate type and required answers server-side.
- Support a clear 'not applicable' response only where the question explicitly allows it.

### FR-08 — Technician and manager views
- Technician 'My Work' page: assigned jobs, due dates, priorities, status and detail/action screens; mobile-responsive web layout.
- Manager dashboard: upcoming/overdue work orders, active/expiring contracts, counts by status, technician workload.
- All list screens paginate server-side and filter by tenant.

### FR-09 — Notifications and auditability
- In-app notifications for approaching contract renewals and overdue work orders; email notifications configurable per company and user.
- Background jobs must be idempotent, retryable, and observable; deduplicate notifications per recipient/event/window.
- Audit important mutations: membership changes, contract activation/renewal, plan changes, work-order transitions, and inspection submissions.
- Do not store passwords, tokens, or sensitive attachment contents in audit payloads.

### FR-10 — Attachments
- Support optional photos/documents for assets, work orders, and inspection answers, using private object storage and short-lived authorized download URLs or authenticated streaming.
- Validate extension, MIME/content, maximum size, and access scope; malware scanning may be added before public launch based on deployment risk.
- Attachments inherit tenant and parent-record permissions; never expose public bucket paths.

### FR-11 — Subscription readiness
- Plan catalog, subscription state, trial start/end, seat/asset limits, and entitlement checks.
- MVP can onboard pilots manually; production paid launch requires a supported payment provider, webhook signature validation, idempotent event handling, reconciliation, and grace-period policy.
- Do not promise a specific Gulf payment gateway until merchant eligibility and country coverage are verified.
- Subscription lapse: make access read-only after grace period; preserve data and provide an export path under published retention policy.

## 5. Authorization matrix

| Capability | Owner/Admin | Manager | Technician | Viewer |
|---|---|---|---|---|
| Company settings and billing | Yes | No | No | No |
| Manage members/roles | Yes | No | No | No |
| Customers/sites/assets | CRUD | CRUD | Read assigned context | Read |
| Category/attribute templates | CRUD | CRUD | Read | Read |
| Contracts and plans | CRUD | CRUD | Read assigned context | Read |
| Assign work orders | Yes | Yes | No | No |
| Execute assigned work | Yes | Yes | Yes | No |
| Close/reopen work orders | Yes | Yes | No | No |
| Reports/dashboard | All | All | Own assignments | Read |

'CRUD' includes archive subject to business invariants. Owner/Admin can perform operational actions but should still be recorded as actor. Permissions are object-level and tenant-scoped; UI visibility is not a security boundary.

## 6. Data model and invariants

Suggested Django apps:

- `accounts`: custom User, authentication, invitations.
- `tenants`: Company, Membership, CompanySettings.
- `customers`: Customer, Site.
- `assets`: AssetCategory, AttributeDefinition, Asset.
- `contracts`: MaintenanceContract, ContractAsset, ContractRenewalLink.
- `maintenance`: MaintenancePlan, WorkOrder, WorkOrderAssignmentHistory, WorkOrderEvent.
- `inspections`: InspectionTemplate, InspectionTemplateVersion, InspectionQuestion, InspectionSubmission, InspectionAnswer.
- `notifications`: Notification, NotificationDelivery.
- `billing`: Plan, Subscription, BillingEvent.
- `audit`: AuditEvent.

**Required invariants:**
1. Every tenant-owned model has non-null `company_id` and appropriate indexes.
2. Every service accepts explicit tenant context; never trust `company_id` from client payload.
3. All foreign keys across tenant-owned models are checked for same-tenant ownership, including on update and background jobs. Django ForeignKey alone does not enforce matching tenant IDs; add service validation and database enforcement for critical cross-tenant relations where feasible (e.g. composite foreign keys via custom SQL/migrations), with adversarial tests.
4. A site's customer and an asset's site cannot be changed to cross-tenant records.
5. Contract-covered assets belong to the contract customer.
6. Work-order occurrence uniqueness: `(company_id, plan_id, occurrence_date)` for generated preventive jobs, with an explicit source/occurrence key for any non-date recurrence extension.
7. Human-readable identifiers are unique within a company, not globally; use immutable UUID primary keys internally where practical.
8. Never hard-delete records referenced by completed work orders; archive/retire instead.
9. Published inspection versions and completed answers are immutable except audited correction workflow (deferred).
10. State transitions are validated atomically with concurrency controls.

Use PostgreSQL `JSONField` only for genuinely category-specific attributes. Index relational fields used for tenant filtering, due dates, status, and foreign keys. Add JSON expression/GIN indexes only after measuring real query needs.

## 7. Key workflow specifications

### 7.1 Onboarding
Register → verify email → create company → configure locale/time zone → invite staff → create first customer/site → configure asset category → add asset → create contract/plan → first scheduled work order.

### 7.2 Preventive scheduling
Scheduler runs → finds active due occurrences within lookahead → validates asset/contract coverage → transactionally inserts work order with uniqueness guard → updates scheduling cursor consistently → emits notification via transaction-on-commit → logs outcome. Re-running the task must not duplicate jobs.

### 7.3 Work-order execution
Manager schedules/assigns → technician accepts/starts → records checklist answers and evidence → completes → manager reviews/closes. Invalid transitions return a user-safe error and do not partially persist changes.

### 7.4 Renewal
Reminder generated at configured lead time → manager reviews expiring contract → explicitly creates successor term → updates affected plans intentionally → historical term remains accessible. Do not assume auto-renewal.

## 8. Technical architecture

**Backend:** Supported stable Django version at implementation time, Python, PostgreSQL.  
**Frontend:** Django Templates, HTMX, Tailwind CSS, accessible responsive design.  
**Background work:** Celery + Redis, Celery Beat or equivalent scheduler.  
**Tests:** pytest, pytest-django, factories, PostgreSQL integration tests.  
**Deployment:** Docker, production WSGI/ASGI server as appropriate, reverse proxy, TLS, private object storage, structured logs, health checks, backups.

Prefer a modular monolith with domain services for transitions, recurrence, and contract renewal. Use Django forms/model forms for input and validation, class-based views where they improve reuse, explicit queryset scoping, and Django admin for trusted internal operations (not as the main customer UI). Avoid signals for critical business workflows; prefer explicit service calls and `transaction.on_commit` for async side effects.

### Request and tenant lifecycle
1. Authenticate request.
2. Resolve selected company from a trusted session/URL context.
3. Confirm active membership.
4. Apply role and object permissions.
5. Query strictly within company scope.
6. Validate input and cross-tenant references.
7. Execute domain service in transaction where needed.
8. Render tenant-scoped response; queue external side effects after commit.

For Celery tasks, pass company and object identifiers, re-fetch and revalidate tenant context, and avoid carrying mutable ORM instances between processes.

## 9. Security, privacy, and reliability

- Django security defaults: CSRF, secure cookies, HTTPOnly, SameSite, HTTPS/HSTS, rate limiting for login and invitations, secure password reset.
- Strict server-side role and object authorization; test IDOR and cross-tenant access for list/detail/create/update/delete/download endpoints.
- Use tenant-scoped object lookup; never rely on hiding IDs or client-side checks.
- Protect file uploads, enforce size limits and private storage, sanitize displayed user content.
- Record security-relevant actions and failed administrative operations; restrict platform support access.
- Backups with documented restore tests, migration rollback strategy, error monitoring, and alerting for failed jobs.
- Data retention, deletion/export, cross-border hosting, and privacy notices must be reviewed for each launch country; do not assume all Gulf jurisdictions have identical requirements.
- Dates: store instants in UTC; compute due dates and recurrence using tenant time zone. Distinguish local calendar dates from timestamps.

## 10. Localization and UX

- Arabic RTL is the primary experience; design all forms, tables, dialogs, and printable views for RTL.
- Use Django i18n/gettext for static UI strings; store multilingual display labels where configured.
- English-ready with proper LTR switching; avoid hardcoded direction in shared components.
- Company-specific currency formatting, time zone, date presentation, and locale; do not hardcode Saudi currency or VAT assumptions.
- Responsive technician screens usable on mobile browsers, keyboard-accessible forms, validation messages, and empty states.

## 11. Reporting and exports

MVP dashboard: overdue and upcoming work orders, expiring contracts, open work by technician, assets by category. CSV export for customers, assets, contracts, and work orders with authorization and tenant filtering. Large exports should run asynchronously. Advanced analytics and custom report builder are deferred.

## 12. Nonfunctional requirements and acceptance targets

- **Isolation:** Automated negative tests show no cross-company read/write via HTTP, services, tasks, and file downloads.
- **Scheduling:** Repeated or concurrent generation produces exactly one preventive work order per plan occurrence.
- **Integrity:** Invalid lifecycle transitions and inconsistent foreign keys fail without partial writes.
- **Performance:** Typical paginated list pages target p95 server response under 500 ms in a documented pilot load profile, excluding external services; measure rather than assume.
- **Reliability:** Background job retries are safe; failed tasks observable; backup restore rehearsed before paid launch.
- **Accessibility:** Forms usable by keyboard, clear labels/errors, adequate contrast; mobile technician workflow tested.
- **Testability:** Domain services covered by unit tests; workflows and tenancy by integration tests; key UI paths by browser tests.
- **Audit:** Actor, company, event, target, and timestamp retained for important operations.

## 13. Delivery phases and exit criteria

### Phase 0 — Discovery and domain design
- Interview target contractors; document two trade workflows; finalize core terminology and pilot success metrics.
- Deliver ERD, role matrix, key state diagrams, tenancy threat model.
- **Exit:** No industry-specific hardcoded model needed for the two example trades.

### Phase 1 — Platform foundation
- Django project, CI, PostgreSQL, Docker, auth, company memberships, tenant scoping, roles, company settings, test factories.
- **Exit:** Cross-tenant isolation tests pass for all existing endpoints.

### Phase 2 — Customer and asset configuration
- Customer/site CRUD, categories, typed attribute definitions, asset CRUD, validation and filters.
- **Exit:** Configure elevator and HVAC assets without schema changes.

### Phase 3 — Contracts and maintenance plans
- Contracts, covered assets, renewal records, recurrence rules, due dates, generation idempotency.
- **Exit:** Simulated scheduler retries/concurrency do not duplicate work orders.

### Phase 4 — Work orders and inspections
- Assignment, state machine, immutable inspection template versions, responses, evidence, manager review.
- **Exit:** Complete a preventive maintenance cycle with correct permissions and audit history.

### Phase 5 — Operations and pilot
- Dashboards, notifications, CSV exports, Arabic RTL polish, error monitoring, backup/restore tests, pilot onboarding.
- **Exit:** Pilot companies independently complete onboarding and recurring service workflows.

### Phase 6 — Paid launch
- Plan entitlements, payment integration appropriate to merchant jurisdiction, webhook reconciliation, legal/privacy review, documentation and support processes.
- **Exit:** Verified subscription lifecycle, tested downgrade/grace behavior, first paying customers.

## 14. Explicitly deferred enhancements

Inventory and spare parts; purchase orders; invoicing/e-invoicing; customer self-service portal; technician geolocation and routing; offline-first mobile app; QR/barcode asset scanning; SLA penalty calculations; custom workflow engine; IoT integrations; automated contract generation; advanced analytics; AI diagnostics; external API and webhooks; multi-tenant database-per-company deployment. Revisit based on pilot evidence.

## 15. Open decisions before implementation

1. Initial pilot country and payment merchant jurisdiction.
2. First two maintenance trades for workflow validation (not architectural restrictions).
3. Whether plans can cover a group of assets or only one asset in MVP (default: one asset per plan).
4. Contract pricing and invoice integration boundaries (default: record contract price only, no accounting).
5. Technician assignment cardinality (default: one primary assignee; teams later).
6. Reminder channels and lead times (default: in-app plus optional email).
7. Exact subscription entitlements, trial duration, and retention policy.
8. Required pilot data import format and attachment size limits.

## 16. Change control / source of truth

This document is the authoritative baseline for product scope. Implementation should preserve domain invariants even when UI changes. For each new requirement: record motivation, affected models/workflows, tenant/security impact, migration plan, test cases, and whether it is MVP or deferred. Do not expand MVP solely because a feature is technically interesting.
