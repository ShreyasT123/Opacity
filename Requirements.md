# Opacity — Requirements Document

*AI-assisted invoice and expense approval platform with a built-in analytics layer. Version 0.1, October 19, 2026. Author: Shreyas.*

## 1. Why this project

Opacity is one project that covers three resume lines at once: **full stack** (Next.js + Django REST + Postgres), **workflow automation** (a real approval state machine plus LLM-powered document intake), and **analyst work** (SQL-driven KPIs, anomaly detection, dashboards). It also sits in the finance domain, which lines up with fintech experience and gives every bullet a business outcome to quote.

**One-line pitch:** Upload an invoice PDF, get structured data extracted and validated, route it through a configurable approval chain, and see spend analytics and anomaly flags on a live dashboard.

## 2. Goals and non-goals

**Goals**

- Ship a deployed, demo-able product with seed data in 4 weeks.
- Show end-to-end ownership: schema design, API, UI, async jobs, analytics, tests, CI, deployment.
- Produce measurable outcomes (extraction accuracy, approval turnaround, anomalies caught) that can be quoted on a resume.

**Non-goals (v1)**

- Real payment execution or bank integrations.
- Multi-tenant billing, SSO, mobile apps.
- Training a custom OCR or vision model.

## 3. Users and roles

| Role | Needs |
| --- | --- |
| Submitter (employee/vendor clerk) | Upload invoices, fix extraction mistakes, track status |
| Approver (manager/finance) | Review queue, approve/reject/comment, delegate |
| Finance analyst | Dashboards, drill-downs, exports, anomaly review |
| Admin | Configure approval rules, manage users and vendors |

## 4. Verified technology baseline (checked October 19, 2026)

- **Next.js 16** is the Active LTS line (released October 21, 2025). Next.js 15 leaves Maintenance LTS on October 21, 2026, so start on 16 directly. Create the app with `npx create-next-app@latest`.
- **Claude structured outputs** are generally available on the Claude API and are requested through `output_config.format` with `type: "json_schema"`; the response is valid JSON matching your schema in `response.content[0].text`. The older beta header and `output_format` parameter still work during a transition period, so do not use them in new code. Pick the model from the current models page and keep it in an environment variable.
- Confirm exact package versions (Django, DRF, Celery, Postgres) against their release pages on day one and pin them.

## 4a. Architecture summary

| Layer | Choice | Reason |
| --- | --- | --- |
| Frontend | Next.js 16 (App Router), TypeScript, Tailwind, Recharts | Familiar stack, strong hiring signal |
| Backend | Django + Django REST Framework | Fast to build admin, auth, ORM, state machine |
| Database | PostgreSQL | Window functions and materialized views for analytics |
| Async jobs | Celery + Redis | Extraction and anomaly scoring off the request path |
| Extraction | Claude API with PDF input and JSON-schema structured output | Typed, validated fields instead of free text |
| Storage | S3-compatible bucket (MinIO locally) | Original documents kept immutable |
| Deployment | Docker Compose locally, one cloud host for the demo | Reproducible setup |

## 5. Functional requirements

### 5.1 Intake and extraction

- **FR-1** Users can upload PDF or image invoices (max 10 MB each) singly or in batches of up to 20.
- **FR-2** On upload, a background job sends the document to the Claude API and requests output against a fixed invoice JSON schema: vendor name, GSTIN, invoice number, invoice date, due date, currency, line items (description, quantity, unit price, tax rate), subtotal, tax total, grand total, PO number.
- **FR-3** The system validates extraction results: totals must reconcile with line items within a tolerance, dates must parse, GSTIN must match the 15-character format. Failed checks set status `NEEDS_REVIEW` with the failing rule recorded.
- **FR-4** Each extracted field stores a confidence flag (valid, suspect, failed) so the UI can highlight what to check first.
- **FR-5** Submitters can correct any field in a side-by-side view (document left, form right). Corrections are saved separately from the original extraction so accuracy can be measured later.
- **FR-6** Extraction failures (unreadable scan, refusal, API error) are stored on the record and shown in the UI, never silently dropped. Re-extraction can be triggered manually.

### 5.2 Approval workflow

- **FR-7** Invoice states: `UPLOADED → EXTRACTING → NEEDS_REVIEW | READY → PENDING_APPROVAL → APPROVED | REJECTED → PAID_MARKED`. Only listed transitions are allowed and each is logged.
- **FR-8** Admins define approval rules as data, not code: for example, amount under ₹50,000 needs one manager; ₹50,000 to ₹5,00,000 needs manager plus finance; above that adds a director. Rules can also key on vendor or cost center.
- **FR-9** Approvers get a queue with filters, can approve, reject (reason required), request changes, or delegate to another approver.
- **FR-10** SLA tracking: each approval step has a due time; overdue items are flagged and trigger a reminder email.
- **FR-11** Full audit trail: who did what, when, from which state to which state, with before/after values for edits. The audit log is append-only.

### 5.3 Controls and fraud signals

- **FR-12** Duplicate detection: flag invoices with the same vendor and invoice number, or the same vendor, amount, and date within a configurable window.
- **FR-13** Optional PO matching: if a PO number is present, compare invoice totals and line items to the stored PO and flag mismatches.
- **FR-14** Statistical anomaly scoring per vendor: flag amounts far outside the vendor's history using a robust z-score first, then an Isolation Forest over features (amount, day of month, line count, tax ratio) as a second stage. Each flag stores the reason in plain language.
- **FR-15** Analysts can mark an anomaly as true positive or false positive; this feedback is stored to evaluate the detector.

### 5.4 Analytics and dashboards

- **FR-16** Overview dashboard: total spend, invoice count, average approval time, approval rate, overdue count, with date-range and cost-center filters.
- **FR-17** Spend analysis: by vendor (top 10 plus Pareto chart), by category, by month with trend line.
- **FR-18** Process analysis: funnel from uploaded to paid, bottleneck step by median time, approver workload.
- **FR-19** Risk view: anomaly list, duplicate list, and a precision tracker built from FR-15 feedback.
- **FR-20** CSV export of any table; a scheduled weekly summary email to finance.
- **FR-21** Analytics queries run on SQL views or materialized views refreshed on a schedule, so the logic is visible and testable in SQL.

### 5.5 Administration

- **FR-22** Email/password auth with role-based access control enforced in the API, not just the UI.
- **FR-23** Vendor master with merge tool for near-duplicate vendor names.
- **FR-24** Seed script that loads at least 1,500 synthetic invoices across 40 vendors, with injected duplicates and outliers, so the dashboards and detector have something real to show.

## 6. Core data model

| Entity | Key fields |
| --- | --- |
| User | id, email, role, cost\_center |
| Vendor | id, name, gstin, normalized\_name |
| Invoice | id, vendor, number, dates, currency, totals, status, source\_file, extraction\_json, corrected\_json, error |
| InvoiceLine | invoice, description, qty, unit\_price, tax\_rate |
| ApprovalRule | id, conditions (JSON), steps (ordered list of roles) |
| ApprovalStep | invoice, approver, state, due\_at, decided\_at, comment |
| AuditEvent | id, invoice, actor, from\_state, to\_state, diff, timestamp |
| AnomalyFlag | invoice, type, score, reason, review\_label |
| PurchaseOrder | id, vendor, lines, total |

## 7. API outline

| Method and path | Purpose |
| --- | --- |
| POST /api/invoices/ | Upload one or many documents |
| GET /api/invoices/?status=&vendor=&from=&to= | List with filters and pagination |
| GET, PATCH /api/invoices/{id}/ | Detail and field correction |
| POST /api/invoices/{id}/reextract/ | Re-run extraction |
| POST /api/invoices/{id}/transition/ | Move state with a reason |
| GET /api/approvals/queue/ | Current user's pending items |
| POST /api/approvals/{id}/decide/ | Approve, reject, delegate |
| GET /api/analytics/{kpi}/ | Dashboard data endpoints |
| GET, POST /api/rules/ | Approval rule management |

Publish the OpenAPI schema and link it from the README.

## 8. Non-functional requirements

- **Performance:** list and dashboard endpoints respond in under 500 ms at 10,000 invoices; extraction completes in under 30 seconds for a 5-page PDF.
- **Security:** secrets only in environment variables; uploaded files served through signed URLs; per-role permission tests; no document content written to application logs.
- **Privacy:** a visible note that documents are sent to an external LLM API once a real key is configured, plus a demo mode with canned extraction results so the app runs without a key.
- **Reliability:** idempotent Celery tasks with retry and backoff; a failed job never leaves an invoice stuck in `EXTRACTING`.
- **Quality:** at least 70% backend test coverage on workflow, validation, and rule engine; Playwright tests for the upload-to-approval flow; CI runs lint, tests, and type checks on every push.
- **Accessibility:** keyboard-navigable queue and forms; chart data also available as tables.

## 9. Evaluation plan (this is what makes it credible)

| Metric | How measured | Target |
| --- | --- | --- |
| Field extraction accuracy | Compare extraction\_json to corrected\_json on a labeled set of 100 invoices | 90%+ on header fields, 85%+ on line items |
| Validation catch rate | Share of injected bad invoices caught by FR-3 | 95%+ |
| Duplicate detection | Precision and recall on injected duplicates | 95%+ recall |
| Anomaly detector | Precision at top 20 flags against injected outliers | Report honestly, even if lower |
| Approval turnaround | Median time in demo simulation, with and without rule-based routing | Report the delta |

Publish the results table in the README with the method, not just the numbers.

## 10. Milestones (4 weeks)

| Week | Deliverable |
| --- | --- |
| 1 | Repo, Docker Compose, auth and roles, data model, upload and storage, seed script |
| 2 | Extraction pipeline with schema, validation rules, review UI, demo mode |
| 3 | State machine, approval rules engine, queue UI, audit trail, SLA reminders |
| 4 | Duplicate and anomaly detection, dashboards and SQL views, evaluation run, deployment, README and demo video |

## 11. Acceptance criteria for "done"

- A stranger can open the live demo, log in as each role from a login hint, and complete upload, review, approve, and dashboard steps without help.
- `docker compose up` brings the whole system up locally with seed data.
- The README has an architecture diagram, the evaluation table, a 2-minute demo GIF, and a section on design decisions and tradeoffs.
- CI is green and the test suite covers the workflow engine and the rule evaluator.

## 12. Resume bullets this project should support

Fill the numbers in only after you measure them.

- Built Opacity, a full-stack invoice approval platform (Next.js, Django REST, PostgreSQL, Celery) with LLM-based document extraction validated against a JSON schema, reaching X% field accuracy on 100 labeled invoices.
- Designed a data-driven approval rules engine and audit trail, cutting simulated approval turnaround by X% versus fixed routing.
- Implemented duplicate and anomaly detection (robust z-score plus Isolation Forest) and SQL-view-backed dashboards for spend, process bottlenecks, and risk.

## 13. Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Scope creep into a full ERP | Hold to the FR list; park everything else in a backlog file |
| No real invoices to test on | Generate synthetic invoices with varied layouts, plus a handful of public sample invoices |
| LLM API cost or outage | Demo mode, caching by file hash, a small model for first-pass extraction |
| Over-claiming accuracy | Measure on a held-out labeled set and publish the method |

## 14. Sources checked

- Next.js support policy and release status: endoflife.ai/article-nextjs-15-eol and it-server-room.com/en/next-js-version-support-policy-lts-eol-eol
- Claude structured outputs: platform.claude.com/docs/en/build-with-claude/structured-outputs
