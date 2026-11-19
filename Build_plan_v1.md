# Dental clinic bookkeeping — combined seven-day build plan

Version 2.0 · 13 November 2026 · Implementation backlog for a locally hosted web app.

## 1. Delivery agreement

Build a browser-based bookkeeping app running on the owner's computer. Deliver proper input forms, a working dashboard, financial detail views, selected/date-filtered Excel exports, and tested backup/restore.

This plan combines the concrete stack and local-operation details of an earlier draft plan with the financial workflows and smaller task structure of the previous one-week plan. The financial source of truth is **[Specification_v2.md](Specification_v2.md)**, subject to the explicit week-one deferrals below. Older v1.1 export machinery and owner-only payment assumptions are not carried forward.

Target: seven focused working days for one experienced developer with AI assistance. Budget **56 hours: 42 planned hours + 14 contingency hours**. Each day has six planned hours and two hours reserved for integration and defects. Task estimates include focused checks; they are timeboxes, not guarantees. Days are relative to kickoff.

A release is complete only when its supported workflows pass the release gates. If required balances or recovery remain unreliable, deliver a labelled pilot and extend the schedule. Do not claim full v2 implementation after one week.

## 2. Week-one scope

| Area | Required working behavior |
|---|---|
| Setup | Single owner login; company; company cash/bank accounts; suppliers, investors and authorized people; categories; dated VAT codes; opening balances and obligations. |
| Purchases | Manual multi-line draft/post; issued supplier/buyer facts; invoice/received/recognition dates; printed VAT; management treatment; documents; search/filter/detail. |
| Funding | Actual cash/bank receipt; investor and custodian; proposed vs confirmed equity/loan/unclassified treatment; evidence/review; funding returns. Registered capital stays a separate reference. |
| Money | Company supplier payments; internal deposits, withdrawals and transfers; transit account when needed; fees and other receipts/disbursements with explicit review. |
| Settlement | Partial payments; one payment across multiple invoices; purchase advances; application to invoices; refundable security; capacity checks. |
| People | Any authorized person can pay a supplier; original entitlement; partial reimbursement/return; date/time, method, proof and remaining due. |
| Contracts | Commitment ceiling with VAT-included toggle and same-basis comparison, linked invoices, deposits, dated/milestone installments, explicit retirement of invoice-covered contract milestones. |
| Basic credits/refunds | Single-sign linked credit, same-supplier offset, credit/advance refund to company or original person; origin/capacity checks. |
| Review | >=5m internal payment exceptions; missing evidence; separate funding, VAT and CIT reviews; reviewer/date/basis and persistent follow-up. |
| Views | Dashboard, cash/bank ledgers, purchase/payment detail, person due, advances/security, commitments and upcoming obligations. |
| Operations | Lightweight change history; safe corrections; reconciliation; simple Excel; private evidence; nightly verified encrypted off-device backup; clean restore drill. |

### Deferred, with visible boundaries

- OCR/XML field extraction. XML originals may be stored/downloaded; manual entry works beside a safe document view.
- Asset register, depreciation and asset-cost splitting. Keep CAPEX treatment and serial/location notes.
- Mixed-sign legal adjustments, automatic replacement settlement migration, generic full-payment reversal automation, capital restructuring and personal receivables from misdirected company refunds. Preserve supporting documents and record an unresolved review case; block unsupported posting rather than inventing an ordinary transaction.
- Bulk document ZIP downloads, immutable export packages, delivery baselines, delta corrections, acknowledgement chains and export watermarks. The first is optional later; the remaining export machinery is intentionally excluded.
- Payroll in any form (payroll-summary payables, staff-cost categories, salary payments): a later stage.
- Extra user logins, LAN/cloud access, bank feeds, patient records, inventory valuation and automated tax conclusions.
- Budgets, configurable dashboards, advanced charts, large asynchronous exports and general historical import tooling.

If a deferred operation appears in the actual opening data or pilot, resolve it before live use or extend the schedule. A review case preserves the problem but is not a substitute for recording the correct financial outcome.

## 3. Concrete implementation stack

| Concern | Choice |
|---|---|
| Environment | Python 3.12 baseline, `uv`, committed lockfile. Verify supported compatible versions at kickoff. |
| Framework | Django 5.2 LTS baseline; pin a supported security-patched release after verification. |
| Database | SQLite, local disk, foreign keys enabled, WAL, full synchronous durability, bounded busy timeout, serialized financial writes. Verify actual connection settings in tests. |
| UI | Django templates + HTMX for invoice rows/allocation previews + Pico.css with small custom styling. Bundle assets in `static/vendor`; no CDN/runtime internet dependency. |
| Excel | XlsxWriter, explicit cell types, no macros. |
| Local server | Waitress, loopback only at `http://127.0.0.1:8000`; debug disabled for daily operation. |
| Tests | pytest + pytest-django for financial services; a small browser smoke suite or recorded manual browser checks for owner journeys. |
| Scheduling | macOS LaunchAgent templates for local server and nightly backup; show last success/failure in app. |
| Backup | SQLite backup API + matching document manifest; maintained authenticated-encryption tool selected and pinned on Day 1. No custom encryption. Verified off-device copy required for release. |

Initial host target: macOS. The app works offline after installation, with no separate frontend build pipeline or container runtime required. Local HTTP is restricted to loopback; exposing it to another device requires a separate access/security design.

**UI language:** Vietnamese primary labels, matching v2, with framework translation strings and optional English column dictionary. Use a single approved label list; no language-switcher feature is required in week one. Amounts display `1.300.000 ₫`; dates `dd/mm/yyyy`; stored/export machine dates ISO. Financial values remain whole-VND integers.

### Source and runtime layout

```text
clinicbooks/
  pyproject.toml, uv.lock, run.sh, stop.sh, README.md
  config/                 settings, URLs, server entry
  apps/
    core/                 registry, codes, commands, audit, parties, accounts
    documents/            private originals, links, upload/download
    bookkeeping/          bills, movements, funding, entitlements, settlement
    contracts/            commitments, installments
    dashboard/            views using shared queries
    exports/              current-value selection and workbook
    ops/                  backup, restore, integrity and scheduling
  templates/
  static/vendor/
  tests/
  data/                   ignored by version control
    db.sqlite3
    documents/
    exports/              temporary private outputs
    logs/
```

Keep runtime data out of source control; installation/update must preserve it. Keep secrets and recovery credentials in protected OS/deployment storage, separately recoverable. The backup destination must be off the host's primary storage; choose an actual external/off-device destination during setup.

## 4. Shared engineering rules

1. **One financial write path:** command service → idempotency check → write transaction → expected-version checks → validation/mutation → audit → commit. Same key/payload returns the first result; conflicting reuse fails. Thin views call services.
2. **One financial read implementation:** shared queries calculate balances for detail pages, dashboard and Excel. Templates and browser previews are not authoritative arithmetic.
3. **Lightweight history:** stable UUIDs/display codes, root versions, append-only field-level before/after change log with actor/time/reason. No full aggregate snapshots or export sequences.
4. **Money validation:** positive typed settlement amounts, purpose/account compatibility, supplier/person consistency, capacity limits, safe integer sums and negative-physical-cash prevention, including later effects of backdating.
5. **Documents:** private immutable originals, authenticated downloads, quarantine/type/size validation, safe paths and escaped XML. No file contents can authorize commands. Upload processing stays outside financial transactions.
6. **Review separation:** actual cash received ≠ confirmed legal equity; paid supplier ≠ reimbursed person; policy review ≠ VAT/CIT acceptance. Unreviewed recovery is blank, not zero.
7. **Opening positions:** immediate pre-start cash/bank balances plus distinct unpaid obligations, advances, person due and funding history, linked once into normal settlement records. No duplicate current-period receipts or purchases.
8. **Corrections:** reason required for posted edits; revalidate dependent capacity/schedules and reopen materially affected reviews/reconciliations. Block unsupported corrections with a clear next step.
9. **Test as built:** money/capacity tests accompany services, not just end-of-week testing. Avoid exhaustive cosmetic snapshots.

### Internal >=5m payment policy

Default to company bank → supplier for a positive invoice's full gross amount **>=5,000,000 VND**, including exactly 5m. A small installment does not bypass the rule. Other actual sources remain recordable with a mandatory reason and persistent accountant-review case. Personal bank payment still falls outside this chosen direct-company-bank default. Reimbursement does not rewrite original supplier payment method.

For uninvoiced deposits, default to bank where known contract gross is >=5m or unknown; recheck on invoice arrival. Potential split purchases/same-supplier-day totals trigger review warnings. This is internal policy, not a claim that smaller cash payments are tax eligible or all larger personal payments are illegal. No new legal interpretation is introduced by this build plan.

### Contract comparison basis

Each contract has a **"VAT included" toggle** (`ceiling_vat_included`) next to its ceiling amount. To avoid mixed-basis totals, the ceiling is only compared with invoices on the same basis (Specification_v2 §4.3):

- **Toggle on:** ceiling vs linked invoices' gross.
- **Toggle off:** ceiling vs linked invoices' line net, labelled "excl. VAT" on screen and in Excel.
- **Paid** is always the actual VND paid, shown beside the comparison, never subtracted from an excl. VAT ceiling.
- **Dashboard commitments** show VAT-included and VAT-excluded contracts as two separate figures.
- **Contract milestones** use the ceiling's basis; invoice installments are always gross.

## 5. Owner screens

| Area | Input | Viewing |
|---|---|---|
| Dashboard | Quick actions: invoice, payment, funding, transfer | Physical cash, transit, bank, combined funds; supplier open, advances/security, person due; funding classes; upcoming installments; actionable review/backup alerts. |
| Purchases | Invoice lines, posting, simple correction, credit | Filtered selectable list; invoice detail with VAT, payments, documents and history. |
| Money | Funding, supplier payment/refund, internal transfer, fee/other movement | Cash/bank running ledgers; payments; actual funding and returns. |
| People | Authorization, personal payment, reimbursement/return | Original payer/payment/invoice, each reimbursement, remaining due. |
| Contracts | Commitment, milestones, advance application | Invoiced, paid once, advance, uninvoiced estimate and unpaid schedule. |
| Reviews & exports | Tax/funding/policy decisions, cash count/bank statement comparison, export | Persistent review queue; reconciliation differences; Excel download. |
| Settings | Company, accounts, parties, openings, backup destination | Setup completeness, recovery status, local-operation guidance. |

Forms show field-level errors, a draft path where applicable, evidence links, and a short financial-effect confirmation before posting. Posted data must survive restart. Lists need usable empty states, filtering and pagination. No essential owner action may require framework admin.

## 6. Backlog and timeboxes

**58 tasks, each 15–60 minutes.** Each task has one concrete completion check. Daily subtotals are six hours, excluding the two-hour contingency. End each day with the listed demo and a small committed working increment.

### Day 1 — Foundation, local startup and setup (360 minutes)

| ID | Min | Task | Done when |
|---|---:|---|---|
| 1.1 | 30 | Initialize environment, pinned dependencies, project and runtime paths. | App starts against the intended local database. |
| 1.2 | 45 | Local server/start-stop scripts, offline layout and owner login. | Loopback-only login works; debug off; private pages reject logged-out access. |
| 1.3 | 45 | Registry, stable codes, money/date helpers and root versions. | Codes unique; integer/date validation and overflow checks pass. |
| 1.4 | 60 | Transaction/idempotency command helper and field-level audit. | Retry creates no duplicate; stale version/conflicting key fails; injected failure rolls back. |
| 1.5 | 45 | Company/account/party/category setup forms. | Owner creates cash, bank, supplier, investor and payer in UI. |
| 1.6 | 60 | Opening-account balances and typed opening-obligation sources. | Sources linked uniquely; no live receipt created; opening duties separated from cash. |
| 1.7 | 30 | Shared navigation, deterministic fixture and empty dashboard. | Click through core pages without placeholders posing as balances. |
| 1.8 | 45 | Choose backup encryption/destination approach; consistent DB+file backup spike. | One encrypted sample backup decrypts/restores in isolation; secrets excluded from output. |

**Gate/demo:** Log in, configure company/openings, restart and verify persistence. Demonstrate atomic rollback and idempotent retry. Encryption feasibility is resolved before the last days.

### Day 2 — Actual funding, transfers and recorded balances (360 minutes)

| ID | Min | Task | Done when |
|---|---:|---|---|
| 2.1 | 60 | Movement model, purpose/account matrix and validated posting. | Wrong account/person combinations fail atomically. |
| 2.2 | 45 | Shared account-balance query with backdated cash validation. | All affected dates stay nonnegative for physical cash. |
| 2.3 | 45 | Actual funding receipt service and form. | 1bn cash receipt creates one pending funding position, no purchase/revenue. |
| 2.4 | 30 | Funding-classification review form/history. | Confirmed class changes without another receipt or bank leg. |
| 2.5 | 45 | Internal transfer/deposit service and form. | Equal legs preserve combined funds and funding totals. |
| 2.6 | 30 | Transit account, fees and other reviewed movements. | Transit excluded from drawer count; bank fees entered separately. |
| 2.7 | 30 | Funding return capacity and input. | Return reduces account and position once; excess blocked. |
| 2.8 | 45 | Cash/bank ledger views and dashboard money cards. | Cards and running balances use shared queries. |
| 2.9 | 30 | Funding/transfer integration checks. | 1bn cash →400m deposit gives cash 600m, bank 400m, funding 1bn. |

**Gate/demo:** Receive investment cash, deposit part, view ledgers and review classification. Registered capital remains a separate reference.

### Day 3 — Invoice input, evidence and company payments (360 minutes)

| ID | Min | Task | Done when |
|---|---:|---|---|
| 3.1 | 60 | Bill/line/fact/tax-review models and invoice identity reservation. | Totals/signs/duplicates enforced; unreviewed VAT NULL. |
| 3.2 | 60 | Multi-line draft/post form, list and invoice detail. | Mixed chair/installation/supplies invoice entered entirely through UI. |
| 3.3 | 45 | Quarantined private document upload and linking. | Signature/size/path checks pass; original bytes retained. |
| 3.4 | 30 | Authenticated document view/download beside entry form. | PDF/image safely viewed; XML escaped/downloaded; no public file path. |
| 3.5 | 60 | Supplier payment/direct allocation service and capacity tests. | Partial/multi-bill payment valid; wrong supplier/over-allocation fails. |
| 3.6 | 45 | Payment allocation form and invoice payment history. | Effect preview and supplier open agree after posting. |
| 3.7 | 30 | >=5m exception capture and duplicate/aggregation warning. | Exactly 5m, personal payment and small installment cases handled correctly. |
| 3.8 | 30 | Payables dashboard and end-to-end payment checks. | 100m bank payment from Day 2 leaves bank 300m and invoice open zero. |

**Gate/demo:** Upload, enter, post and partially pay an invoice, inspect evidence and remaining balance. Missing evidence is reviewable; invalid arithmetic is blocked.

### Day 4 — People, partial reimbursement and review (360 minutes)

| ID | Min | Task | Done when |
|---|---:|---|---|
| 4.1 | 45 | Authorization and personal entitlement models/services. | Personal payment creates due without reducing company funds. |
| 4.2 | 45 | Personal supplier-payment UI extending existing form. | Payer/date/time/method/proof captured; missing authorization flagged. |
| 4.3 | 60 | Reimbursement/return allocation commands and capacity tests. | Cannot over-reimburse or over-return; same-key retry safe. |
| 4.4 | 45 | Reimbursement input and person detail views. | Original invoice gross, person paid, reimbursed and remaining due visible. |
| 4.5 | 45 | Persistent review queue and policy/funding decisions. | Reviewer/date/basis retained; resolved cases reopen after material changes. |
| 4.6 | 45 | Per-line VAT/CIT review panel. | Printed vs approved VAT separate; CIT independently saved; unknown stays blank. |
| 4.7 | 45 | Opening person/funding positions integration and mixed-payer checks. | Openings settle without new historical cash; two people on one bill remain distinct. |
| 4.8 | 30 | Person-due dashboard and reimbursement walkthrough. | 3.8m personal →2m cash →1.8m bank reimbursement closes due once. |

**Gate/demo:** Any authorized person can pay and receive partial reimbursements. Policy-exception review and tax acceptance remain visibly independent.

### Day 5 — Contracts, advances and basic return flows (360 minutes)

The financial services below should reuse the common command, allocation, document and review components. Any overrun uses contingency, not weaker settlement checks.

| ID | Min | Task | Done when |
|---|---:|---|---|
| 5.1 | 45 | Contract input/detail, ceiling + VAT-included toggle, same-basis commitment query. | Committed/invoiced/paid shown separately; toggle off compares with invoice net and is labelled "excl. VAT"; dashboard keeps the two bases apart. |
| 5.2 | 45 | Purchase advance/security lots and deposit allocation. | Funding an advance reduces money once; security not OPEX. |
| 5.3 | 45 | Advance-application service and compact form. | Apply to invoice without another payment; supplier/capacity validated. |
| 5.4 | 45 | Installment editor, settlement and schedule conversion. | Covered contract milestones retired; no duplicate future obligation. |
| 5.5 | 45 | Single-sign credit and same-supplier offset. | Credit consumed once; positive bill capacity cannot be exceeded. |
| 5.6 | 45 | Credit/advance refund with origin and recipient checks. | Company vs original-person refund has correct due/cash effect. |
| 5.7 | 30 | Shared compact credit/refund/offset forms and simple correction/history. | Owner can use supported flows; invalid dependent correction blocked. |
| 5.8 | 30 | Opening supplier payable/credit/advance integration. | Outstanding opening amounts settle once, without new purchases/receipts. |
| 5.9 | 30 | Equipment and return-flow regression checks. | 1.3bn contract, 300m/700m/300m payments count once; over-refund blocked. |

**Gate/demo:** Deposit →invoice →advance application →installments. Show one credit/refund and one refused capacity violation. Required review cases capture unsupported replacements; they do not silently change balances.

### Day 6 — Simple Excel, reconciliation and completed recovery (360 minutes)

| ID | Min | Task | Done when |
|---|---:|---|---|
| 6.1 | 45 | Selected/date-filtered query and consistent export read. | Named date bases and PRIMARY/CONTEXT/BALANCE roles correct. |
| 6.2 | 45 | Core XLSX workbook sheets and controls. | Purchases/lines, payments, allocations and cash legs reconcile. |
| 6.3 | 45 | Remaining supported sheets, selection/download UI and export checks. | Funding/people/contracts/reviews/documents/openings included; literal text/IDs safe. |
| 6.4 | 45 | Bank-statement/cash-count comparison and invalidation. | Difference visible; no balancing expense; backdated change requires review again. |
| 6.5 | 45 | Complete consistent DB+document backup manifest and integrity verification. | Restored source set is complete; missing blob or hash mismatch fails visibly. |
| 6.6 | 45 | Encrypt, copy off-device and verify successful recovery path. | Success only after verified encrypted off-device copy; recovery credential available separately. |
| 6.7 | 30 | Nightly scheduler, backup status and actionable dashboard alerts. | Scheduled test run logged; failure/stale backup visible; sleep limitations documented. |
| 6.8 | 30 | Full restore command and isolated restore smoke test. | Matching database/documents load with expected balances. |
| 6.9 | 30 | Operating guide and export/backup failure checks. | Failed output never offered as complete; guide identifies data and backup paths. |

**Gate/demo:** Select invoices, download Excel, compare cash/bank statement values, then restore a verified backup to an isolated installation.

### Day 7 — Feature freeze, real workflow verification and release (360 minutes)

| ID | Min | Task | Done when |
|---|---:|---|---|
| 7.1 | 60 | Run financial regression and scoped v2 scenario checklist. | Required scope green; deferred portions explicitly identified. |
| 7.2 | 60 | Browser walkthrough of inputs, dashboard, files and Excel. | All essential owner actions accessible; no dead controls or financial placeholders. |
| 7.3 | 45 | Race, retry, rollback and document-access checks. | Competing last-capacity allocation has one winner; unauthorized/unsafe access rejected. |
| 7.4 | 60 | Clean install and full off-device restore drill. | Integrity/FKs/hashes/balances pass; workbook regenerates; elapsed recovery time recorded. |
| 7.5 | 60 | Pilot representative records and reconcile hand-calculated figures. | Known cash/bank, supplier open, advances and person due all match. |
| 7.6 | 45 | Fix/recheck critical pilot defects using remaining contingency as needed. | No unresolved release blocker; otherwise label build pilot and document extension. |
| 7.7 | 30 | Version release, finalize operator guide and limitations. | Guide starts app reliably; checklist/results/sample Excel delivered. |

**Gate/demo:** Owner performs startup →invoice/payment →dashboard →export →backup, and verifies restored records. No automatic sending of pilot files; owner shares the sample with the accountant through the chosen channel.

## 7. Excel scope contract

Export **explicitly selected records** or records within an **inclusive date range**. Bills default to recognition date with selectable invoice/received basis; money uses effective date; contracts use signed date. Explain missing dates and context inclusion. Selected IDs are not silently filtered again by date.

Read a consistent copy of current saved values, then format outside the read transaction. Export run is informational only. Repeated files can overlap; stable IDs/versions let the accountant replace values rather than append duplicate purchases. No export delivery state affects later selection.

Required sheet families for supported entities:

- README/dictionary: company, generation time, scope/date bases, as-of date, overlap/current-value notice and control totals.
- Purchases/lines and VAT/CIT review.
- Supplier payments/refunds and typed allocations/advance applications/credit offsets.
- Company cash movements: one row per account leg; internal transfer legs identified.
- Funding and reimbursements with original references and remaining balances.
- Contracts/installments, balances, documents, issues and reference data; optional relevant field-history sheet.

Every row has stable IDs, version/status and scope role. PRIMARY selection totals exclude CONTEXT. BALANCE rows are separately labelled full-company as-of values, not totals of the selected subset. Live vs opening origin explicit. Do not sum overlapping payment, cash-leg and reimbursement sheets as extra spending.

Freeze headers, add filters, write identifiers/text literally, write VND as exact numeric integers within Excel precision, handle larger values explicitly, and never silently truncate. Originals remain separately available through private document download.

## 8. Required demonstrations and tests

| Group | Evidence before release |
|---|---|
| Funding | 1bn received as cash, 400m deposited, 100m paid from bank →cash 600m, bank 300m, combined 900m; funding receipt remains 1bn. Pending vs confirmed class distinct. |
| Policy | 4,999,999 / 5,000,000 / 5,000,001 and 1m installment on 10m invoice; exception captured in input, not only dashboard. |
| People | 3.8m personal payment, partial cash/bank reimbursements; 11m exceptional personal payment; mixed company/two-person invoice payments. |
| Contracts | 1.3bn equipment, 300m advance, invoice application, 700m/300m payments; one purchase and correct schedule, never 1.6bn paid. Same flow with the VAT toggle off: no false over-ceiling warning, basis labelled. |
| Returns | Credit refund vs offset capacity; partial advance refund; personal vs company refund recipient; excess personal refund blocked pending reimbursement return. |
| Openings | Cash, bank, unpaid bill, credit, advance, person due, loan/equity history all enter once; no duplicate current-period spending/funding. |
| Reviews | VAT codes distinct; unknown recovery blank; approved signed credit amounts validated; independent CIT/funding/policy decisions and history. |
| Integrity | Double submission, conflicting idempotency payload, stale tab, race for last allocation capacity, and injected failure produce no invalid partial state. |
| Reconciliation | Physical cash cannot go negative; bank/count differences shown; backdated edits invalidate affected reviews. |
| Export | Explicit selections/date range, context exclusions, current-value consistency, literal formula-like text, leading-zero identifiers and exact VND controls. |
| Files | Unsafe filenames/unsupported bytes rejected, XML inert, unauthenticated download blocked, missing blobs surfaced. |
| Recovery | Consistent active-WAL backup, matching immutable documents, encrypted off-device retrieval, clean installation restore and sample workbook regeneration. |

Use v2's 26 scenarios as the detailed checklist. Mark advanced asset, replacement and XML-extraction portions deferred; do not call these implemented. All supported portions and required controls must pass. A legal/tax decision remains unreviewed if the accountant has not supplied it.

## 9. Release and operating checklist

- [ ] Owner input, dashboard and detail flows work through normal navigation.
- [ ] Shared queries agree across UI, Excel and fixture/pilot totals.
- [ ] No duplicate capital/income from transfers; no duplicate purchase from reimbursements/advance applications.
- [ ] No open financial integrity, document-access or recovery blocker.
- [ ] Unsupported workflows visibly blocked with preserved evidence/review route.
- [ ] Local server binds only to loopback; debug off; static assets available offline.
- [ ] Data survives restart/update; secrets/runtime data excluded from version control.
- [ ] Actual off-device destination and encryption credential verified; nightly schedule tested.
- [ ] Restore into isolated clean installation passes and recovery time recorded.
- [ ] Operator guide covers setup/start/stop, entering records, review, export, backup/restore and host-sleep behavior.
- [ ] Pilot sample workbook ready for owner/accountant review; external sharing remains owner's action.
- [ ] Remaining limitations and post-week-one backlog delivered explicitly.

## 10. Progress and overrun rules

Track each task as Todo →Doing →Check →Done. Done includes the completion check and a small commit. Each daily checkpoint records task IDs, failed checks, actual hours, contingency remaining and the next blocker.

Dependency order: foundation/openings →money →invoices/payments →people →contracts/returns →export/recovery →pilot. Build UI alongside each service. Do not postpone all input screens or all testing to the final days.

If progress slips, use contingency for correctness and integration first. Cut optional charting, extra convenience controls, preview polish and optional history export before touching core flows. Keep documents downloadable if inline preview is delayed. Never cut transaction/idempotency checks, settlement capacities, basic reconciliation, private access or verified recovery.

Day 7 remains feature-frozen. If a required flow is still unreliable, extend the schedule or restrict the build to an explicitly labelled pilot; do not silently change the acceptance contract. Maintain a visible list of unresolved real transactions that fall outside supported flows.

## 11. Deliverables and first step

Deliver the versioned app with pinned requirements; local startup/scheduler helpers; dedicated owner UI; sample Excel and export implementation; backup/restore tools and drill results; focused financial tests and browser-check results; operating guide; release checklist and explicit deferred backlog.

Start with **1.1–1.2: a local app the owner can open and log into**. By the end of Day 1, the setup path and durable data foundation must work before adding financial workflows.
