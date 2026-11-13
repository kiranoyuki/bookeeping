# Vietnam dental clinic bookkeeping MVP — implementation specification v2

Version 2.0 · 13 November 2026 · Revised plan; no application is delivered by this specification.

## 1. Purpose and agreed scope

Build a practical, single-company bookkeeping and evidence tool for one clinic. The owner records purchases, supplier obligations, company cash and bank movements, actual investment funding, and payments made by authorized people. The accountant remains responsible for statutory accounts, legal capital treatment, VAT recovery, corporate income tax treatment, depreciation, and filings.

The application answers: What did the company receive? Where is its money held? What has it committed, been invoiced, and paid? What remains owed to suppliers and people? Which evidence or accountant decisions are missing?

### Decisions carried into v2

| Area | Implementation decision |
|---|---|
| Company funds | Track physical company cash and each company bank account independently. An investor may hand actual cash to the company or transfer funds to its bank. |
| Funding classification | Receipt of money is a fact; recognition as legally contributed equity is a separate reviewed classification. Support proposed equity, owner/investor loan, and unclassified funding. |
| Cash deposited at bank | One internal transfer, with cash reduced and bank increased. Never another contribution, purchase, or income. |
| Payment policy | For an invoice with gross value **at least 5,000,000 VND**, default to direct payment from a company bank account to the supplier. Exactly 5m is included. Exceptions require a recorded reason and accountant-review case. |
| People paying suppliers | Any authorized person, including owner, employee, or another approved payer. Track actual payment, person owed, and each partial reimbursement. |
| Exports | Simple `.xlsx` export of selected records or date-filtered records, containing latest saved values and stable IDs. No delta delivery protocol or immutable ZIP packages. |
| Retained controls | Contracts, deposits, installments, invoice documents, VAT review, opening positions, lightweight change history, backups, cash counts, and bank reconciliation. |

One authenticated owner operates the MVP. Payers and accountant reviewers are recorded parties/names, not additional application users. Accountant interaction uses exports and externally communicated review decisions. No email or automatic sharing is required.

VND only, whole đồng stored as signed 64-bit integers; never floating point. Dates are Vietnam business dates; timestamps are UTC, displayed in `Asia/Ho_Chi_Minh`. Keep invoice issue date, received date, recognition date, payment date/time, and entry timestamp separate. Foreign currency is rejected with an explanatory message.

Out of scope: statutory general ledger, patient billing or health records, payroll calculation, inventory valuation, depreciation engine, bank feeds, e-invoice issuance, tax filing, multicurrency, offline sync, and multi-company operation. Basic asset candidates and payroll-summary payables are allowed. OCR is optional assistance; manual entry must always work.

## 2. Internal policy and accountant review

### 2.1 Internal payment policy, not a legal conclusion

The 5m rule below is the company's chosen operational control. It does **not** state that every smaller cash payment is tax eligible, every larger personal payment is illegal, or every bank payment qualifies for a tax deduction. Recording a fact is not approval of its legal treatment.

| Situation | Planned-payment behavior | Actual-transaction behavior |
|---|---|---|
| Invoice gross <5m | Company bank, company cash, or authorized-person payment available; show evidence requirements. | Record actual payer, method, amount, date/time, evidence, and missing-evidence flags. |
| Invoice gross >=5m | Default to company bank → supplier. Other sources require exception reason and a review case before proceeding. | Always allow the factual record with mandatory exception reason and open accountant review; do not hide money already paid. |
| Contract deposit without invoice | Default company bank when known contract gross >=5m or value is unknown. Recheck when invoice arrives. | Preserve the actual source and date; create or update review flags when later information changes risk. |
| Partial installment of a >=5m invoice | Apply policy to the **whole invoice gross**, not the installment amount. | A 1m cash installment on a 10m invoice remains an exception. |
| Reimbursement | Record company cash/bank → original payer separately. | Reimbursement does not change the original supplier payment method or erase its flags. |

Policy uses the full positive charge invoice amount, including printed VAT and rounding, before payments or credits. Replacement uses the successor invoice; related adjustments trigger review without retroactively erasing earlier exceptions. Potential split purchases and same-supplier/same-day totals >=5m produce review warnings, not automatic legal determinations. Refunds, funding, reimbursements, and internal transfers are not tested as new purchase invoices.

Separate policy-exception status (`OPEN`, `REVIEWED`, `CLOSED`) from VAT/CIT decisions. Record external accountant name, decision date, evidence/basis, and follow-up. Reviewed exceptions never gain an automatic “tax eligible” label. Reopening is required after material changes in amount, payer, recipient, payment method, or evidence.

### 2.2 Legal and tax review boundaries

Do not hardcode a universal legal cash-payment prohibition. Applicable rules can depend on transaction date, company type, activity, taxpayer method, evidence, payment authorization, aggregation, and amendments. Owners, employees, and other payers are not automatically legally interchangeable. Neither physical cash receipt nor a later bank deposit establishes that capital was legally contributed.

Store printed VAT separately from accountant-approved recoverable VAT. Keep `ZERO`, `NON_TAXABLE`, `NOT_DECLARED`, and `UNKNOWN` distinct. Never assume all dentistry, cosmetic services, retail, or input purchases share the same tax treatment. VAT recovery and CIT deductibility are independent review decisions; `UNREVIEWED` amounts are NULL, not zero.

Official reference starting points checked during revision: [Government text of Decree 181/2025](https://xaydungchinhsach.chinhphu.vn/nghi-dinh-181-2025-nd-cp-quy-dinh-chi-tiet-thi-hanh-mot-so-dieu-cua-luat-thue-gia-tri-gia-tang-119250707172930626.htm) and [Ministry of Finance response concerning employee-authorized payment](https://baochinhphu.vn/khoan-uy-quyen-cho-nhan-vien-chi-ho-co-duoc-khau-tru-thue-102260224091838594.htm). These are review references, not a complete current-law opinion or proof of applicability to this clinic. Before legal configuration, the accountant must verify the applicable texts and amendments for the relevant date. The application must work with legal/tax classification pending.

## 3. Architecture and persistence

Use one supported Django application, server-rendered responsive forms, SQLite on local disk, private immutable document files, and a standard XLSX writer. Pin supported dependencies at implementation. No Redis, distributed event bus, full aggregate snapshots, or export-delivery engine is required.

A browser accesses one application server. Local deployment binds to loopback; remote deployment needs authenticated HTTPS and private access controls. Do not synchronize or share the live SQLite database through cloud folders or network filesystems.

Enable foreign keys on every connection, WAL, full synchronous durability, and bounded busy timeout. Financial commands run as short serialized write transactions with expected version checks and idempotency keys. Do not run uploads, OCR, file copying, export formatting, or external requests inside write transactions. The supported services enforce all cross-row financial constraints before commit.

Export reads all selected data in one consistent database read transaction, copies the rows to memory or a temporary private file, then formats the workbook outside that transaction. This is ordinary consistent reading, not a stored historical export model. Backup uses SQLite's supported consistent backup mechanism and matching documents.

## 4. Data contract

### 4.1 Common conventions

- UUID text primary keys; stable, unique display codes such as `PAY-000123`, never reused. Explicit indexed foreign keys with `ON DELETE RESTRICT`.
- Every root has `id`, `code`, `version >=1`, `created_at`, `created_by`, `updated_at`. A small `record` registry owns these fields and `kind`; each root subtype has `id PK/FK record`. Validate registry/subtype consistency in services. Documents and review issues link through this registry, avoiding unenforced polymorphic IDs.
- Financial roots use `DRAFT`, `POSTED`, `VOID`; invoices additionally have derived superseded state from replacement links. Contracts use `DRAFT`, `APPROVED`, `CLOSED`, `VOID`. Drafts may be incomplete; posting validates all mandatory facts and capacities.
- Enumerations use database CHECK constraints; money uses INTEGER, dates validated ISO `YYYY-MM-DD`, timestamps ISO UTC. `?` below means nullable; other fields are mandatory unless a draft allowance is stated.
- All child rows have UUIDs and one parent FK. A child edit increments its root version and writes a change log. Posted financial records cannot be hard-deleted.
- Individual amount input maximum: 999,999,999,999 VND. Validate sums with overflow-safe integer arithmetic. Never silently round amounts or truncate export cells.
- Reviews use `UNREVIEWED`, `ACCEPTED`, `QUERY` unless a more specific enumeration is shown. Acceptance records reviewer name, timestamp, and basis note. Owner entry of an external review is attributed to the owner in audit.

### 4.2 Settings, parties, company accounts, and opening positions

| Table | Required business columns / constraints |
|---|---|
| `company` | Singleton root; legal name, tax code, address, books_start, currency=`VND`, timezone, registered_charter_capital_vnd? >=0, registered-capital reference date?, setup status (`DRAFT`, `CONFIRMED`). Registered capital is a reference, never spendable money. |
| `party` | Name, roles (supplier/investor/owner/employee/other payer; multiple allowed), tax code?, address?, active. Preserve invoice party facts separately. |
| `payer_authorization` | Person FK, valid_from, valid_to?, scope note, authorization reference?, review status. Link supporting authorization documents. Expired/missing authorization opens review; do not discard an actual payment. |
| `account` | Name, kind (`COMPANY_CASH`, `COMPANY_BANK`), bank details? for bank, cash custodian? for cash, active. Only company-owned balances are tracked; no fictitious personal account balances. |
| `opening_account_balance` | One per account; opening date equals books_start, amount >=0, source/evidence note, review status. Represents balance immediately before the first day's movements. |
| `opening_item` | Kind (`SUPPLIER_PAYABLE`, `SUPPLIER_CREDIT`, `SUPPLIER_ADVANCE`, `PERSON_DUE`, `INVESTOR_LOAN`, `EQUITY_HISTORY`, `UNCLASSIFIED_FUNDING_HISTORY`), party FK, amount >0, original date?, source reference, review status, corresponding migrated record FK?. No company money movement. |
| `category`, `project` | Editable names, active flag; category default management treatment. Historical use survives archival. |
| `rule_profile` | Version label, scope (`INTERNAL_POLICY`, `TAX_REVIEW`), effective dates, config JSON, source links, external reviewer?, reviewed_at?. No overlapping active profiles of the same scope. Seed internal threshold 5m; tax profile starts unconfirmed. |

Opening obligations are materialized through the normal settlement model: unpaid legacy invoice/credit, advance lot, personal-payment entitlement, or funding position, each with `origin=OPENING` and a unique `opening_item_id`. Only the outstanding opening amount contributes to settlement capacity. Store original face value and previously settled amount as historical reference, not as current period spending. Opening loans/equity are balances/history, not new receipts. Do not put these amounts into a second set of additive opening totals.

### 4.3 Contracts, invoices, schedules, VAT, and assets

| Table | Required business columns / constraints |
|---|---|
| `contract` | Supplier FK, reference?, project?, signed_date?, gross ceiling >=0, description, status, review status. Changes to ceiling require reason/addendum reference. |
| `bill` | Supplier FK, contract FK?, kind (`VAT_INVOICE`, `SALES_INVOICE`, `RECEIPT`, `NO_INVOICE`, `OTHER`), effect (`CHARGE`, `CREDIT`, `MIXED_ADJUSTMENT`), form?, series?, number?, invoice date?, received date, recognition date, description, signed rounding, stated gross?, origin (`LIVE`, `OPENING`), opening_item_id?, status. |
| `bill_party_fact` | Bill FK, role (`ISSUER`, `RECIPIENT`), issued name/tax code/address. Unique bill+role. Master-party edits never rewrite issued facts. |
| `bill_line` | Bill FK, line_no >0, description, quantity text?, signed net, VAT code FK, signed printed VAT, category FK, treatment, project?, classification review. Unique bill+line_no. Authoritative money comes from invoice amounts, not quantity multiplication. |
| `vat_code` | Code, kind (`RATE`, `ZERO`, `NON_TAXABLE`, `NOT_DECLARED`, `UNKNOWN`), rate basis points?, valid dates, reference?. Dated selectable values, no supplier-based automatic legal assignment. |
| `tax_review` | One per line; activity (`TAXABLE`, `NON_TAXABLE`, `MIXED`, `UNKNOWN`), recovery status (`UNREVIEWED`, `NONE`, `FULL`, `PARTIAL`), approved recoverable VAT?, CIT status (`UNREVIEWED`, `DEDUCTIBLE`, `NONDEDUCTIBLE`, `PARTIAL`), nondeductible base amount?, payment evidence status, rule profile?, reviewer/date/basis. |
| `bill_relation` | New bill FK, prior bill FK, kind (`ADJUSTS`, `REPLACES`), reason. Same supplier; no cycles. One successor per predecessor and one predecessor per replacement in MVP. |
| `invoice_identity` | Bill FK, normalized issuer tax code, form, series, number, current flag; unique complete identity across all bills, including voided/superseded history. Missing identity components create warnings. Corrected identities remain reserved aliases. |
| `installment` | Exactly one parent: bill or contract; sequence, due date? or milestone trigger, amount >0, kind (deposit/delivery/acceptance/retention/other), active flag. Unique parent+sequence. |
| `installment_settlement` | Installment FK, settlement-source record FK, amount >0. Source is a validated positive bill allocation or advance application; use separate nullable FKs with exactly-one CHECK. No extra money movement. |
| `asset_candidate` | Description, kind, serial/location?, acceptance/in-service dates?, accountant asset code?, status, note. Cost links to bill lines with component amounts and provisional VAT flag. No depreciation engine. |

Management treatments: `CAPEX`, `OPEX`, `PREPAYMENT`, `REFUNDABLE_DEPOSIT`, `INVENTORY_REVIEW`, `UNCLASSIFIED`. Suggested categories retain equipment, fit-out, furniture/IT, premises, advance rent, opening costs, operations, clinical supplies, and people. Classification is not statutory recognition.

Invoice gross = sum(line net + printed VAT) + rounding. CHARGE lines nonnegative, CREDIT lines nonpositive; MIXED_ADJUSTMENT requires a related bill and review note. Net sign determines settlement direction; zero-total bills accept no settlement. Signed credit VAT remains signed. Approved recovery cannot exceed printed VAT magnitude and must follow its sign; unreviewed stays NULL. Posting requires at least one line, matching stated/calculated gross when stated, valid dates, supplier, and required issued facts or explicit missing-evidence flags.

### 4.4 Money movements and supplier settlement

Use **positive** amounts everywhere in settlement rows; direction comes from a typed operation. Do not store negative payment amounts or ambiguous signed allocations.

| Table | Required business columns / constraints |
|---|---|
| `movement` | Date, occurred_at? UTC, purpose, amount >0, from_account FK?, to_account FK?, party FK?, payer_person FK?, method (`CASH`, `BANK`, `CARD`, `OTHER`), reference?, description, origin=`LIVE`, status, reversal_of FK?. Purpose rules below. |
| `supplier_allocation` | Supplier-payment movement FK, exactly one of charge_bill FK / advance_lot FK, amount >0; unique movement+target. Company or personal payment can fund it. |
| `advance_lot` | Supplier FK, contract FK?, kind (`PURCHASE_ADVANCE`, `REFUNDABLE_SECURITY`), origin (`LIVE`, `OPENING`), original payer person FK? (NULL=company), opening_item FK?. Live lot has exactly one originating supplier allocation. |
| `advance_application` | Advance lot FK, positive bill FK, effective date, amount >0, reason. Noncash use of an existing advance, never another payment. Refundable security cannot be applied without explicit reviewed reclassification. |
| `credit_offset` | Negative bill FK, positive bill FK, date, amount >0, reason, status. Same supplier; no company money movement. |
| `supplier_refund` | Refund movement FK, exactly one of credit_bill FK / advance_lot FK, amount >0. Multiple rows allowed; sum equals movement. |
| `refund_origin` | Supplier-refund row FK, original supplier payment FK or opening advance FK, amount >0. Identifies whose payment was refunded; sum equals refund row. |
| `personal_entitlement` | Exactly one source: live personal supplier payment FK or opening_item FK; person FK, original reimbursable amount >0, authorization FK?, review status. Live amount equals full personal supplier-payment amount, including advances. |
| `reimbursement_allocation` | Reimbursement or reimbursement-return movement FK, personal entitlement FK, amount >0; unique pair. |

Purpose/account matrix:

| Purpose | Company account fields | Other required fields |
|---|---|---|
| `SUPPLIER_PAYMENT` by company | from only | Supplier; method compatible with source account. |
| `SUPPLIER_PAYMENT` by person | neither | Supplier, payer_person, actual method, entitlement; opens authorization review if needed. |
| `SUPPLIER_REFUND` to company | to only | Supplier; typed refund rows and origins. |
| `SUPPLIER_REFUND` to person | neither | Supplier, payer_person, origins belonging to same person. |
| `REIMBURSEMENT` | from only | Person; allocations to that person's entitlements; method cash/bank matches account. |
| `REIMBURSEMENT_RETURN` | to only | Person; allocations reversing prior reimbursed capacity. |
| `FUNDING_RECEIPT` | to only | Investor/source party, funding detail. |
| `FUNDING_RETURN` | from only | Investor, return allocations to funding positions, reviewed reason. |
| `TRANSFER` | both, distinct company accounts | Internal transfer detail; same amount on both legs. |
| `REVERSAL` | Exact opposite company legs of original; no company legs for a personal supplier-payment reversal | Original movement FK unique; full amount only; same supplier/person; actual reversal date and proof. See reversal constraints below. |
| `OTHER_RECEIPT` / `OTHER_DISBURSEMENT` | to only / from only | Counterparty where known, reason, accountant review. Never automatically classified as revenue or expense. |

Transfer method is `OTHER` with subtype (`CASH_DEPOSIT`, `CASH_WITHDRAWAL`, `BANK_TRANSFER`, `CASH_RELOCATION`); do not force one cash/bank method onto both legs. Transfers post when both legs are confirmed; unconfirmed deposits stay drafts. If funds leave cash custody before bank credit, support an explicit `COMPANY_CASH` account named “Deposit in transit” with two transfers and receipts; exclude it from physical drawer counts but include in recorded company funds. Bank fees are separate disbursements, not an unexplained difference between legs.

Reversal constraints: `reversal_of` is nonnull only for purpose `REVERSAL`, references one earlier POSTED non-reversal movement, and has a unique index for posted reversals. Amount equals the original amount. The original and reversal retain their real account legs in cash reports, so money nets on the respective dates. Original settlement rows and entitlement/funding effects cease contributing from the reversal effective date; they are retained, not deleted. As-of calculations before that date still include them. Store `reversed_effective_date` on the original as a validated derived field in the same command. A reversal itself has no new allocations or entitlement. Before reversal, require dependent reimbursements, refunds, advance applications, funding returns and schedule settlements to be unwound through explicit dated events or a valid atomic plan; block if any capacity would fail. Full-transfer reversal reverses both legs. Partial supplier money returns use normal credit/advance refunds, not this full-reversal command. Reversal of a reversal is outside MVP; retain the factual evidence and review issue for a supported correction plan.

### 4.5 Actual funding and legal classification

| Table | Required business columns / constraints |
|---|---|
| `funding_position` | Investor party FK, origin (`LIVE`, `OPENING`), receipt movement FK? unique, opening_item FK? unique, received amount >0, proposed class (`EQUITY`, `LOAN`, `UNCLASSIFIED`), confirmed class (`EQUITY`, `LOAN`, `UNCLASSIFIED`), review status, agreement reference?, legal effective date?, reviewer/date/basis?, evidence status. Exactly one live receipt or opening item. |
| `funding_return_allocation` | FUNDING_RETURN movement FK, funding position FK, amount >0; same investor, capacity cannot exceed position less prior returns. |
| `funding_classification_event` | Position FK, date, old/new class, amount (full remaining position in MVP), reason, external reviewer/date, evidence links. Noncash reclassification only; never another receipt. |

A live funding receipt can be company cash or bank. Record who delivered it, date/time, amount, receiver/custodian, receipt and source/agreement evidence. Missing documents create visible review issues; they do not change money physically received.

Default confirmed class is `UNCLASSIFIED` until reviewed. Proposed equity is shown as “received, proposed equity, pending review,” never legally verified contributed capital. A cash receipt classified as loan remains loan funding even if deposited in the bank. Classification changes do not change balances or income.

Registered charter capital, actual funding received, confirmed contributed equity, investor loan balance, and personal reimbursement due are separate displayed figures. Registered minus confirmed equity is an informational comparison, not a legal deadline or compliance determination. Opening equity history contributes to the historical comparison but not current receipts. Noncash conversion of personal dues to equity, loan-to-equity restructuring, and splitting one funding position among classifications are deferred; require an accountant-reviewed future extension rather than a fabricated transfer.

### 4.6 Documents, issues, change history, and operational records

| Table | Required business columns / constraints |
|---|---|
| `document_blob` | SHA-256 PK, private storage key unique, bytes >0, MIME, scan status, created_at. Immutable original bytes. |
| `document` | Root; blob FK, original filename, type, document date?, supersedes_document FK?, retention review date?, legal hold, note. Types include invoice XML/PDF, receipt, contract, addendum, authorization, funding agreement, cash receipt, deposit slip, bank proof, reimbursement proof, statement, cash count, other. |
| `document_link` | Document FK, target record FK, role, active flag; unique document+target+role. Removal deactivates link and is logged. |
| `review_issue` | Target record FK, code, severity, description, state, reason/decision?, external reviewer?, reviewed_at?. Stable fingerprint prevents duplicate automatic issues. |
| `policy_exception` | Target payment/bill FK, policy version, reason, requested_at, state, external reviewer/date?, outcome note?. Review decision is not tax acceptance. |
| `change_log` | Append-only ID, command ID, actor, timestamp, root FK, operation, field path, before/after JSON, reason. Log child additions/removals, posting, voiding, classification and document-link changes. No complete root snapshots or export watermarks. |
| `command_request` | Idempotency key PK, request hash, result/status, created_at. Same key+payload returns prior result; same key+different payload conflicts. |
| `reconciliation` | Account FK, date, observed statement/count amount, computed amount at review, difference at review, current recalculated difference, reviewer, note, state, last reviewed account version. Cash includes denomination counts/custodian; bank includes statement reference. |
| `export_run` | ID, created_at/by, selection mode/filter JSON, selected IDs, schema version, generated_at?, row counts, status/error, private path? with expiry. Informational log only; no delivery baseline. |
| `backup_run` | Started/completed dates, status, destination, manifest hash, verification/restore-test result; no secrets. |
| `extraction_job` | Document FK, XML/OCR method/version, state, suggested fields/provenance JSON, error. Suggestions are never authoritative transactions. |

Audit is lightweight, not tamper-proof against a machine administrator. Application and triggers prevent ordinary log editing. Critical edits always need a reason, even if nothing has ever been exported.

### 4.7 Invariants and balance formulas

All capacities below use active posted records only, including reviewed opening positions. Enforce in the same write transaction; drafts cannot consume capacity.

1. Company account balance at end of date D = opening balance + posted incoming legs through D − posted outgoing legs through D. Transfer contributes once on each account and zero to combined funds. Reject movements before books_start. Recheck all affected later balances on backdating/correction; negative physical cash is blocked. Negative bank balance needs explicit overdraft review.
2. Per supplier payment, total allocations <= payment amount. The remainder is an explicit unassigned supplier advance in reports; it is not lost or treated as expense. To later refund/apply it, create/fund an advance lot from that residual atomically. Contract tags do not create extra allocations.
3. Positive bill open = gross (or outstanding opening principal) − direct supplier allocations − advance applications − credit offsets. Cannot be negative. Negative bill open = signed gross (or negative opening principal) + credit refunds + credit offsets. Cannot be positive. Zero total has zero capacity. Cross-supplier settlement is blocked.
4. Advance remaining = original funded amount − advance applications − advance refunds, always >=0. A payment tagged as an advance is not also directly allocated to the same invoice. Using an advance changes classification, not cash or personal entitlement.
5. Refund row totals equal incoming movement; origin totals equal each row. Cumulative refunds of an origin cannot exceed its original supplier payment. Credit refund also consumes credit capacity; advance refund also consumes lot capacity. No double consumption or refund beyond net settlement. Original-payment links do not settle a charge bill a second time.
6. Person due per entitlement = original entitlement − supplier refunds received by that person − company reimbursements + reimbursement returns. Must stay >=0. A supplier refund received by the company leaves person due unchanged. A personal refund exceeding remaining entitlement is blocked until a reimbursement return is recorded; show the real-world recovery task clearly. Refund of company-funded money to a personal account needs a person-receivable extension and is outside MVP posting support; retain evidence and an unresolved issue rather than misclassifying it.
7. Reimbursement allocation totals equal movement amount, belong to the same person, and cannot exceed due. Returns cannot exceed earlier net reimbursements. Reimbursing a person settles company liability, never another purchase or supplier invoice payment.
8. Funding outstanding by position = receipt/opening funding principal − returns. None enters revenue or purchase totals. Legal classification affects funding reports only.
9. Installment settlement cannot exceed source amount or installment amount. A bill schedule covers its positive gross; installments already settled remain included. An approved contract schedule covers its ceiling. When an invoice replaces contract milestones, retire covered milestones and create bill installments in one command; never count both schedules as separate future obligations. Remaining unpaid bill installments plus unscheduled bill balance must equal bill open. Credits require explicit schedule amendment.
10. Contract invoiced = net active linked invoices; contract paid = linked direct payments plus funded advances, excluding returns and without adding advance applications again. Uninvoiced commitment estimate = max(ceiling − net invoiced, 0), flagged if invoicing exceeds ceiling. It is a forecast, not a payable. Exclude refundable security from purchase settlement and contract spending unless specifically included in contract terms.
11. Purchase reporting counts active invoice lines once; cash reporting counts account legs once; personal supplier payments do not reduce company funds. Committed, invoiced, and paid are parallel figures, never summed as total spending. Other receipts do not imply profitability.
12. An identical complete invoice identity is blocked; same supplier/date/amount or identical file hash is a duplicate warning, not automatic merging. Invoice numbers and tax codes remain strings with leading zeros.

Indexes: all FKs; bills on status/recognition_date and supplier/invoice_date; movements on status/date, from_account/date, to_account/date, payer/date; allocations on every target; installments on active/due_date; issues on state/target; change log on root/timestamp. Use partial unique indexes for nullable target pairs and CHECK exactly-one constraints. Cross-row sums, type compatibility, cycles, and capacities are service validations under serialized writes, with database checks/triggers for simple additional defense.

## 5. Operational workflows

### 5.1 Setup and opening positions

1. Set company facts, books_start, accounts/custodians, and internal policy. Collect accountant review questions without blocking draft setup.
2. Enter cash count and bank closing balances immediately before books_start. Enter outstanding invoices, credits, supplier deposits/security, personal dues, loans, and historical equity separately.
3. Each opening item has a source, remaining amount, review state, and exactly one migrated settlement record. Historical invoices may retain face value and prior-settled reference values, but contribute only remaining principal to opening obligations.
4. Preview company funds, supplier balances, personal dues, and funding history. Confirm setup with evidence and a change log. These are management opening positions, not a general-ledger balancing journal.
5. Reject live-period movements that duplicate opening history. If detailed earlier history is needed, move books_start and rebuild openings in a reviewed migration; do not enter both full historical flows and their opening balance.

### 5.2 Actual investment cash → deposit → supplier bank payment

1. Investor hands company 1,000,000,000 VND in cash. Record FUNDING_RECEIPT to company cash, investor, receipt date/time, custodian, evidence, proposed equity or loan, and pending legal classification.
2. Recorded company cash = 1bn; bank = 0; actual received funding = 1bn; confirmed equity remains 0 until accountant review. No invoice or revenue is created.
3. Deposit 400m cash into company bank. Record one TRANSFER with deposit slip and bank confirmation. Cash = 600m; bank = 400m; combined = 1bn; actual funding still = 1bn. If in transit, use the explicit transit account workflow in §4.4.
4. Pay a 100m invoice directly from company bank. Bank = 300m; cash = 600m; combined = 900m; invoice settled 100m. Invoice and payment remain separate records.
5. Accountant later confirms or queries funding classification. Update classification with evidence and audit only; no new cash, income, or contribution receipt is created.

### 5.3 Invoice entry and upload

Manual draft or upload XML/PDF/photo → compare original and suggested fields → verify issuer/recipient, dates, identity, line net/VAT/gross, category, and contract → save draft → review and post. XML is preferred when available; OCR optional. Neither route auto-posts. Missing invoice uses `NO_INVOICE`; no fabricated numbers. Preserve original XML and PDF when both are available.

Arithmetic, duplicate identity, foreign currency, and settlement-capacity errors block posting. Missing evidence or unresolved tax classification is visible and may remain under review. “Post and pay” is one atomic command; failure leaves no partial payment. All source documents are untrusted data and cannot authorize actions.

### 5.4 Authorized person and partial reimbursements

Employee A pays 3.8m to supplier: record a 3.8m invoice and personal supplier payment, person, actual method/date/time, authorization, and proof. Supplier payable = 0; person due = 3.8m; company funds unchanged.

Company reimburses 2m cash: one reimbursement with original entitlement allocation, date/time and proof. Company cash falls 2m; person due = 1.8m. Later 1.8m bank reimbursement closes due. There remains one 3.8m purchase. Show invoice face amount, amount paid by each person, reimbursed amount, refund/return amounts, and remaining due separately.

An 11m personally paid invoice can still be recorded, but opens a >=5m policy exception and accountant review. A 4m reimbursement leaves 7m due; it does not make the original payment compliant. Payer does not have to be the owner or a logged-in user. Mixed company/person payments and multiple people on one bill are supported through separate payments and entitlements.

### 5.5 Contracts, deposits, installments, and refundable security

For a 1.3bn equipment contract with 300m/700m/300m milestones:

1. Approve contract; pay 300m from bank to a purchase advance lot. Committed 1.3bn, paid 300m, advance 300m, invoiced 0. No purchase line yet.
2. Receive 1.3bn invoice, link contract, create asset candidate, and apply advance 300m without another cash movement. Retire corresponding contract milestones and build invoice schedule retaining settled deposit.
3. Pay 700m and 300m as separate bank payments with evidence. After 700m: bill open 300m; after final payment: zero. Contract paid totals 1.3bn, not 1.6bn.
4. If stages are separately invoiced, post stage invoices; flag a subsequent full invoice for potential duplicate billing. Do not post both as ordinary additional purchases without resolution.

Refundable lease security stays a refundable-security lot and not OPEX. Advance rent is prepayment pending accountant allocation; equipment is CAPEX candidate; clinical supplies remain reviewable. Security refund reduces the lot and increases company money without creating negative OPEX.

### 5.6 Supplier credit, refund, and corrections

- A settled 11m charge receives a separate −1m credit: retain charge; post related credit. Its open is −1m. Either refund 1m against that credit (open becomes 0) or offset 1m against another positive bill of the same supplier. Do not do both for the same capacity.
- A 300m advance returned before invoice uses an incoming supplier refund against the advance lot with origin links; no credit invoice is fabricated and no purchase is reversed.
- Supplier refund to a person reduces that person's entitlement. Refund to company cash/bank increases company funds and leaves entitlement unchanged. Both need origin evidence.
- Transcription/category correction updates current values with old/new field history and a reason. Recheck allocations and schedules; never silently drop excess settlement. Unapply excess direct allocations to explicit supplier advance or adjust advance applications in an atomic reviewed correction plan.
- Actual supplier adjustment is a separate signed bill linked `ADJUSTS`. Replacement is a new bill linked `REPLACES`; retain the predecessor as superseded history. Show a preview of every affected allocation, advance application, credit offset, refund and schedule. Require explicit confirmation; commit all changes atomically. If they cannot fit, block with an actionable resolution rather than automatic migration.
- Erroneous entries can be voided with reason after dependent settlement is resolved. A real full payment reversal uses `REVERSAL` with its actual date, original link and proof under §4.4. Partial economic returns use typed refund/return flows. Never void the original real bank transaction or silently erase its earlier cash effect.
- Noncash capital conversions and legally complex multiple-original replacements remain manual accountant issues; no invented cash movements. Filed-period consequences are accountant decisions, never automatic amendments.

### 5.7 Reconciliation and current historical views

Cash ledger: opening, receipts, disbursements, transfers, running balance, custodian, proof, and physical count comparison. Bank ledger: same running-balance structure plus statement references. Internal transfer legs link to one movement; consolidated funds exclude duplicated transfer turnover.

At least monthly, enter each bank statement balance and cash count on a date; show recorded amount and difference. Never create an automatic balancing expense. Investigate missing movements, fees, wrong dates, or opening errors, then post supported corrections. Any later backdated change invalidates the affected reviewed reconciliation and displays “review again.” Keep the original observed balance and reviewed comparison.

As-of views and exports use **current saved records** with effective dates through the selected date. Later corrections/backdated entries can change a past view. The MVP does not reconstruct “what was known at an earlier time.” Label this clearly. No future-dated actual payment posting; planned future payments belong in installments/drafts.

## 6. Screens and owner experience

Vietnamese UI labels, optional English export dictionary; `1.300.000.000 ₫`, `dd/mm/yyyy`, payment time when known. No patient information. Cards and warning counts open filtered lists.

| Screen | Required controls and information |
|---|---|
| Dashboard | Separate physical cash, in-transit cash, company bank, combined funds; supplier payables, supplier credits, advances/security, person due; actual funding received vs confirmed equity vs loans/unclassified; next due installments; evidence/tax/policy issues; backup and reconciliation status. No profit claim. |
| Cash & bank | Account/date filters, opening/running balances, from/to account, movement type, counterparty/person, proof, internal-transfer link, count/statement comparison. Actions: receipt, payment, deposit, withdrawal, reconcile. |
| Funding & capital | Registered capital reference; actual receipts by investor/account; proposed and confirmed class, outstanding loan/returns, evidence and review. “Record actual receipt” distinct from “Review classification.” |
| Purchases | Search; selectable rows; named date basis, supplier, treatment, payment/review filters; net, printed VAT, gross, settled, open. Detail tabs: lines/VAT, payments, installments, documents, history. Show issued recipient and source alongside suggestions. |
| Payment form | Purpose/date/time/amount, company account or person, actual method, supplier, allocations/open capacity, remainder, evidence, authorization; policy warning and exception reason; preview company funds and person due before posting. |
| People & reimbursements | Person → invoice gross, personal amount paid, reimbursements by date/time/method, refunds/returns, remaining due. Select entitlements and allocate a partial reimbursement; never a single unexplained owner balance. |
| Contracts & assets | Contract ceiling, net invoices, total paid once, advances, unpaid installments, uninvoiced estimate; schedule conversion preview. Basic asset candidates, line links, provisional cost and accountant classification. |
| Inbox & documents | Upload, safe preview/download, XML suggestions, manual fallback, evidence links/types, missing evidence; originals retained. |
| Review queue | Policy exceptions, legal funding classification, VAT/CIT, duplicates, missing proof, recipient mismatch, unscheduled balance, and reconciliation differences; record external accountant decisions. |
| Export | Selected rows OR date-filtered scope; date basis, record families, linked context option, preview counts, current-values notice, generate/download `.xlsx`. |
| Settings & recovery | Company/openings/accounts/payers/policies; backup destination/status; verified restore and retention controls. |

Dashboard and lists always distinguish purchase cost from money paid. “Paid” on an invoice means supplier settlement, not whether the company has reimbursed a person. Tax-reviewed and policy-reviewed badges are separate.

## 7. Simple Excel export

### 7.1 Selection and date rules

Two modes:

1. **Selected records:** export explicit stable IDs across pages, subject to confirmation of count. Existing list filters determine what is selectable; export does not silently add unrelated financial records. Date range is not reapplied after explicit selection.
2. **Date-filtered records:** inclusive start/end; select record families. Bills use chosen invoice/received/recognition date (default recognition); movements use effective date; contracts use signed date; funding uses receipt date; review/history uses review/change date. Invoice-date filtering reports how many undated records were excluded and offers their inclusion. Document-only exports use document date with the same missing-date choice.

Show each family's date basis before generating. Linked out-of-range bills/payments/contracts/parties/documents can be included as `CONTEXT`, visibly distinct from requested `PRIMARY` rows. Default include necessary context to interpret the selected records. A payment selected alone must still expose its invoice/person/account references. Only PRIMARY rows enter scope totals. Current balance sheets use separately labelled `BALANCE` rows and an explicit as-of date (default range end, or today for selected mode), independent of primary selection totals.

Read current saved values consistently. Default include posted active financial records, with options for drafts and voided/superseded history clearly labelled and excluded from active totals. Re-export may overlap prior exports and may contain corrected values. The accountant must match stable IDs and replace current values where appropriate, **not append all files as new spending**. No NEW/CORRECT/VOID delivery actions, export watermark, checksum chain, acknowledgement, or delivery cursor.

### 7.2 Workbook contract

Every data row includes stable record ID, parent/reference IDs where applicable, record code, version, status, `row_role` (`PRIMARY`, `CONTEXT`, `BALANCE`), and origin (`LIVE`, `OPENING`) where relevant. Never treat child allocation rows as additional cash movements. Precompute totals; formulas are optional convenience.

| Sheet | Contents |
|---|---|
| `README` | Company/tax code; version 2 schema; generation time; selection/filter/date bases; as-of date; row counts; current-values and overlap notices; opening/context/balance exclusions; column dictionary. |
| `Purchases`, `Purchase_Lines` | Issued facts/dates, supplier/contract, signed net/VAT/gross, category/treatment, current paid/open, replacement links and state. |
| `VAT_Review` | Printed and approved VAT separately; activity, recovery/CIT status, rule reference, external reviewer/date/basis, evidence issues. |
| `Payments` | One row per supplier payment/refund: actual date/time, company source/destination or person, method, amount, reference, exception/review. |
| `Allocations` | Direct bill/advance allocations, advance applications, refund targets/origins, credit offsets, installment settlements; typed rows and positive amounts. These are settlement detail, not additional payments. |
| `Cash_Movements` | All company account legs, including transfers, funding and reimbursements; movement ID and leg direction; per-account opening/closing section. Cross-sheet overlap is explicitly explained. |
| `Funding` | Investor, receipt account/date/amount, proposed/confirmed class, evidence, returns, remaining principal, registered-capital comparison in a separate reference section. |
| `Reimbursements` | Original person payment/entitlement, linked invoices and gross, personal amount, every reimbursement/return, date/time/method, supplier personal refunds, remaining due. |
| `Contracts`, `Installments` | Ceilings, invoice links, advances, paid totals, open/retired schedule, due/trigger and settlement. |
| `Balances` | Full-company company-account, supplier, advance/security, person and funding balances as of selected date; explicitly independent of selected transaction totals. |
| `Documents` | Document ID/type/date/name/hash, linked record IDs and availability; authenticated application reference, not a public or expiring URL presented as archival evidence. |
| `Issues`, `History` | Relevant review issues and optional field-level audit, reasons and timestamps. Not a delta feed. |
| `Reference_Data`, `Assets` | Relevant parties/accounts/categories and basic asset candidates/cost links. |

No macros. Freeze headers, add filters, preserve leading-zero IDs/tax codes as text, write numeric VND, and use ISO date columns. Treat all user text as literal text, including strings beginning `=`, `+`, `-`, or `@`; genuine negative amounts remain numeric. Split oversized sheets without dropping rows, warn before impractically large exports, and fail clearly rather than truncate text. Totals beyond Excel's 15-digit precision must be exported as labelled exact text or rejected with a precise message.

Supporting originals are obtained separately through authenticated document download or a user-selected bulk document download. Such a convenience download may be an ordinary ZIP, but it is not a required handover package, revision system, or backup. Missing evidence is listed; an unavailable stored blob is a technical issue, never a silently omitted file. No automatic send or fixed 14-day export cycle; the owner can export whenever needed.

## 8. Security, documents, retention, and backups

- Strong authenticated owner login, rate limiting, protected sessions, CSRF protection, and reauthentication for restore/credential changes. Use private HTTPS access remotely and encrypted device/storage. Keep secrets outside source, exports, and database backups unless separately protected.
- Store documents privately; authenticated handlers authorize every preview/download. Validate PDF/JPEG/PNG/XML signatures and parsing, default 25 MB/file configurable; reject executable files and uploaded archives in MVP. Disable XML external entities/network resolution; isolate PDF/image processing with size/page/time limits. Quarantine, hash, scan where available, then atomically store originals before linking metadata. Preserve originals; derivatives never replace them.
- No patient health information. Limit payroll and personal-bank details. Log necessary financial fields without credentials or file contents. Warn that exported files contain sensitive company/person information; owner chooses secure sharing.
- Use a configurable retention policy approved by the accountant for each record/document class. Until approved, disable automatic business-record deletion. Legal holds override cleanup; backup rotation must not delete the only retained original. Do not present one fixed retention duration as universally legally sufficient.

Recovery target: no more than 24 hours of data loss and restore within four hours, subject to a verified off-device backup and tested recovery environment.

1. Nightly and before migrations/restores, take a consistent SQLite backup while briefly coordinating writes/uploads. Never copy only a live database file with unhandled WAL state.
2. Capture document references and copy matching immutable blobs; verify hashes and database integrity. Include app/schema version and necessary configuration, excluding unprotected secrets.
3. Encrypt and copy to an off-device destination; mark success only after verified transfer. Keep recovery credentials available separately. Dashboard flags verified backup older than 24 hours.
4. Start with 30 daily and 12 monthly backups, adjusted to capacity and approved retention. Temporary generated exports may expire; original records and documents follow retention controls.
5. Monthly restore into an isolated directory/environment: integrity and FK checks, all financial invariants, document hashes, cash/bank balances, supplier advances/payables, personal dues, funding classifications, and a sample Excel export. Record outcome and elapsed time.
6. Production restore stops writes, first makes a pre-restore backup, restores matching database/documents and compatible app version, checks balances, then reopens. Reconcile any transactions entered and exports shared after the restored backup. Inform the accountant that regenerated current exports may replace previously shared values; there is no export-delivery baseline to rebuild.

Excel exports are not full system backups.

## 9. Command and correction contract

Authenticated service commands accept expected root versions and UUID idempotency keys; validate all affected roots under one transaction. Same key/payload returns the original result; conflicting reuse fails. Stale tabs receive a conflict and updated preview. Errors provide a stable code, field/row, and practical resolution. Never retry a money mutation with a new key merely because the response was lost.

| Command | Atomic effects |
|---|---|
| `ConfirmOpeningPositions` | Validate and link unique opening items; no live cash movements. |
| `PostBill` / `PostBillAndPayment` | Validate facts/totals/identity; create tax-review rows; optional movement/allocation; policy cases. |
| `RecordSupplierPayment` | Movement, allocations/advance funding, personal entitlement when applicable, schedule settlement and policy review. |
| `ApplyAdvance` / `ReallocateSettlement` | Validate old/new capacities and schedules; no money movement. |
| `RecordSupplierRefund` / `ApplyCreditOffset` | Refund money+targets+origins, or noncash offset; recheck person due. |
| `RecordFundingReceipt` / `ReviewFundingClass` | Actual company receipt and pending classification, or noncash reviewed class change. |
| `RecordFundingReturn` | Company outflow, reviewed position allocations, remaining-principal update. |
| `RecordTransfer` | Equal company account legs and proof; no funding/income effect. |
| `RecordReimbursement` / `RecordReimbursementReturn` | Company movement and original-entitlement allocations. |
| `CorrectRecord` / `PostReplacement` / `VoidError` / `ReverseActualMovement` | Reasoned preview; atomic affected settlements/schedules, audit and review reopening. No silent dependent deletion. |
| `AttachEvidence` / `RecordReview` | Document/review facts and target version/audit; no implied financial or tax result. |
| `ReconcileAccount` | Observed balance/count, compared recorded balance, differences; no balancing payment. |
| `GenerateExcel` | Consistent current selection, workbook, informational export log; no change to business balances or future selection. |
| `RunBackup` / `VerifyRestore` | Operational status, manifests and checks. |

A simple scheduled management command suffices for backups and optional extraction. XLSX generation may run inline for bounded datasets or as a single recoverable job for larger exports. No general background-job framework is an MVP prerequisite.

## 10. Implementation phases and release gates

| Phase | Deliverable | Gate |
|---|---|---|
| 0 — data examples | Confirm opening date, sample invoices/funding receipts/authorizations, policy wording, and example workbook. | Owner can explain cash vs bank vs funding vs personal due; accountant review inputs recorded without pretending they are verified law. |
| 1 — foundation | Authentication, migrations, manual bills, private documents, change log, opening positions, backup/restore. | One invoice survives round-trip and restored evidence matches; no partial write after failed posting. |
| 2 — money | Cash/bank ledgers, funding with pending classification, internal deposits, supplier allocations/advances, reimbursements, reconciliation. | Funding/deposit example and partial reimbursements reconcile exactly. |
| 3 — obligations and export | Contracts, installments, credits/refunds, explicit correction previews, selected/date-filtered Excel. | Equipment example settles once; accountant can interpret workbook without summing overlapping sheets. |
| 4 — pilot and hardening | XML review, optional OCR, asset candidates, security/concurrency tests, real opening migration and recovery drill. | All acceptance tests below pass; owner and accountant accept pilot balances/export and unresolved-review list. |

No fixed completion estimate is a release gate. Optional OCR, advanced asset analysis, elaborate dashboards, and complex legal restructuring must not delay reliable cash, settlement, evidence, export and recovery flows.

## 11. Required acceptance tests

Use automated service-level integration tests for money invariants, concurrency and failure behavior, plus owner/accountant review of representative screens and workbook output. Amounts below are VND.

| # | Scenario | Expected result |
|---|---|---|
| 1 | Receive 1bn investor cash, proposed equity pending review. | Cash +1bn; bank 0; actual receipts 1bn; confirmed equity 0; visible evidence/legal review; no revenue/purchase. |
| 2 | Deposit 400m of that cash at bank. | Cash 600m, bank 400m, combined 1bn; funding still 1bn. One transfer ID with two legs, no duplicate contribution. |
| 3 | Pay 100m invoice directly from that bank. | Cash 600m, bank 300m, combined 900m; bill open 0; one purchase, one supplier payment. |
| 4 | Deposit 400m via transit, then charge bank fee 20k. | Drawer −400m, transit +400m; later transit −400m, bank +400m. Fee separately bank −20k. No invented income; transit excluded from physical cash count. |
| 5 | Accountant confirms cash funding as loan instead of proposed equity. | Company balances unchanged; loan outstanding 1bn, equity 0; old/new class and basis logged. Deposit method does not override review. |
| 6 | Registered capital 2bn; actual confirmed equity 600m; loan 400m. | Three distinct figures; no extra 1bn or 2bn spendable cash; no claim of legal compliance. |
| 7 | Invoices 4,999,999; 5,000,000; 5,000,001. | First permits normal cash/person option with evidence flags. Last two default direct company bank and require exception case for other sources. |
| 8 | 1m cash installment against 10m invoice; two same-supplier/day 3m invoices. | First is >=5m policy exception despite installment size; second produces aggregation review warning. Neither auto-decides tax. |
| 9 | Authorized employee pays 3.8m; company reimburses 2m cash then 1.8m bank. | Supplier open 0; due 3.8m →1.8m →0; company outflows 3.8m total; purchase stays 3.8m. Name, dates/time and methods retained. |
| 10 | Person pays 11m despite policy; reimburse 4m. | Actual personal payment retained, exception open; supplier settled; person due 7m; company funds −4m; no legal eligibility implied. |
| 11 | One bill funded by company and two people; one payment covers two bills. | Capacities reconcile per payment/bill/person; no personal account balance invented; no duplicate reimbursement. |
| 12 | 1.3bn contract →300m bank advance →1.3bn invoice/application →700m →300m. | One 1.3bn purchase, three supplier outflows; advance 0 after application; open 300m then 0; no double schedule or 1.6bn paid total. |
| 13 | Refundable security returned; advance rent; chair; supplies. | Security refund reduces deposit only; treatment categories remain separate; no automatic full OPEX/depreciation. |
| 14 | Settled 11m bill, −1m credit, then refund or offset. | Credit open −1m →0, charge not settled twice; competing refund/offset cannot consume same credit twice. |
| 15 | 300m uninvoiced advance refunded in part 50m. | Advance 250m; company account +50m; origins match; no credit invoice or negative OPEX fabricated. |
| 16 | Person-paid bill refunded 1m to person vs company. | Personal receipt lowers person due 1m; company receipt leaves due unchanged and increases company funds. If personal receipt exceeds due, block pending reimbursement return resolution. |
| 17 | Seed openings: cash 20m, bank 100m, unpaid bill 10m, person due 3m, advance 5m, loan 50m. | Funds 120m; obligations/advance separately settleable; no current-period funding/purchase/cash receipts added. Re-entering migrated source is detected. |
| 18 | Replacement invoice lower than settled original; category typo correction. | Preview affected capacities; reject invalid migration or require explicit advance/reallocation plan; original retained as history. Typo edits logged without fabricated legal invoice. |
| 19 | Non-taxable, zero-rated, unknown VAT and partial credit recovery. | Distinct codes; unknown approved VAT blank; partial/signed approval validated; CIT independent of VAT/policy acceptance. |
| 20 | Selected export of one bill; date export with payment next month. | Selected bill PRIMARY; related rows CONTEXT; invoice/payment dates select independently; BALANCE sheets clearly separate. No unrelated PRIMARY additions. |
| 21 | Re-export after backdated correction while another edit occurs during generation. | Each workbook internally consistent; new export reflects current saved correction; README states overlap/current-values semantics; no delivery baseline or immutable ZIP dependency. |
| 22 | Formula-like supplier name, leading-zero invoice/tax code, large totals. | Text stays literal; identifiers preserved; whole VND exact within numeric precision or labelled text/rejection beyond it. No truncation. |
| 23 | Two tabs allocate last 10m; repeated same-key payment; conflicting key reuse. | Only one competing allocation succeeds; retry creates no duplicate; conflicting payload rejected; no partial child rows. |
| 24 | Backdated payment makes later physical cash negative; edit affects reviewed reconciliation. | Negative cash posting blocked; valid backdated edits mark affected reconciliations for review; no balancing expense. |
| 25 | Malicious XML/filename/upload; failed extraction; missing stored blob. | No external fetch/path escape; original safely quarantined; manual fallback; missing blob clearly flagged, never silently omitted. |
| 26 | Nightly backup with active WAL, then clean-environment restore. | Matching DB/documents, integrity/FKs/invariants/hashes and all balances pass; sample workbook regenerates; off-device success verified; recovery time recorded. |

Release requires all applicable financial/security/recovery tests to pass and unresolved legal/tax questions to remain visibly unreviewed. An accountant decision is not fabricated to satisfy a test.

## 12. Configuration and review inputs before live use

Confirm company legal form and investor/payer roles; registered-capital reference and evidence; books_start and opening positions; custody of physical cash; actual company bank accounts and bank beneficiary evidence; who may be authorized to pay; internal exception procedure; accountant-approved legal/tax reference sources and dates; VAT method/activity mix; classifications for CAPEX/prepayments/security; workbook labels/mapping; retention classes; secure sharing and backup destination.

These are implementation/configuration inputs. Routine recording can retain unresolved review states; the software must never silently convert missing review into a tax or legal assertion.

## 13. Changes from v1 and contradiction resolutions

- Replaced owner-only funding/payment assumptions with actual company cash/bank funding and any authorized payer; no tracked personal-bank pseudo balances.
- Separated funding received, proposed classification, confirmed legal classification, registered capital, investor loans, and reimbursement dues.
- Made cash-to-bank deposits equal-leg internal transfers; added transit handling and separate fees.
- Defined exactly 5m as included in internal direct-bank policy and evaluated whole invoice amounts; actual exceptions remain recordable and reviewable.
- Replaced ambiguous signed settlement rows with positive typed allocations, explicit advance applications, credit offsets, and refund origins.
- Kept contract advances, security deposits, invoice settlement, installments and personal reimbursement capacity independent, with explicit no-double-count formulas.
- Added migration sources and outstanding opening capacities so historical balances do not create duplicate live spending or funding.
- Replaced automatic replacement migration with explicit preview, confirmation, and transactional capacity checks.
- Removed immutable aggregate snapshots, delta ZIP manifests, sequence watermarks, delivery/acknowledgement chains, fixed handover interval, and export-driven edit restrictions. Retained stable IDs, current-value XLSX, field-level audit, ordinary consistent reads, and full recovery backups.
- Removed definitive tax/retention assertions from operational rules. Retained printed VAT, separate accountant decisions, dated references and visible unresolved issues.
