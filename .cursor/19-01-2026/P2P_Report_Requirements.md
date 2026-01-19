# P2P Report - Report Requirements

**Module:** SCM - Cross-Module (PO → INV → AP)  
**Report Name:** P2P Report (Procure to Pay)  
**Report Purpose:** End-to-end tracking of procurement cycle from Requisition creation through Payment, capturing all stages: Requisition → Purchase Order → Receipt → Invoice → Payment  
**Created:** 19-01-2026

---

## Input Parameters

- **P_BUSINESS_UNIT_NAME** (VARCHAR2, Optional) - Business Unit Name filter (uses NVL for optional parameter)
- **P_PR_FROM_DATE** (DATE, Required) - PR creation date from filter
- **P_PR_TO_DATE** (DATE, Required) - PR creation date to filter

---

## Output Columns (23 Total)

1. **Requisition Number** - `POR_REQUISITION_HEADERS_ALL.REQUISITION_NUMBER`
2. **Requester Name** - `PER_PERSON_NAMES_F.FIRST_NAME || ' ' || PER_PERSON_NAMES_F.LAST_NAME` (date-effective)
3. **Requisition Creation Date** - `POR_REQUISITION_HEADERS_ALL.CREATION_DATE` (formatted as DD-MON-YYYY)
4. **Requisition Status** - `FND_LOOKUP_VALUES_TL.MEANING` (from `POR_REQUISITION_HEADERS_ALL.DOCUMENT_STATUS`)
5. **Requisition Amount** - `POR_REQUISITION_LINES_ALL.ASSESSABLE_VALUE` with fallback calculation (rounded to 2 decimals)
6. **PO Number** - `PO_HEADERS_ALL.SEGMENT1` (via `POR_REQUISITION_LINES_ALL.PO_HEADER_ID`)
7. **Supplier Name** - `POZ_SUPPLIERS_V.VENDOR_NAME` (via PO link)
8. **Business Unit Name** - `FUN_ALL_BUSINESS_UNITS_V.BU_NAME` (from PO or PR)
9. **PO Status** - `FND_LOOKUP_VALUES_TL.MEANING` (from `PO_HEADERS_ALL.DOCUMENT_STATUS`)
10. **PO Currency** - `PO_HEADERS_ALL.CURRENCY_CODE`
11. **PO Amount** - `PO_DISTRIBUTIONS_ALL.TAX_EXCLUSIVE_AMOUNT + PO_DISTRIBUTIONS_ALL.NONRECOVERABLE_TAX` multiplied by `PO_HEADERS_ALL.RATE` (rounded to 2 decimals)
12. **Buyer Name** - `PER_PERSON_NAMES_F.DISPLAY_NAME` (from `PO_HEADERS_ALL.AGENT_ID`, date-effective)
13. **Receipt Number** - `RCV_SHIPMENT_HEADERS.RECEIPT_NUM` (via `RCV_TRANSACTIONS.PO_HEADER_ID`)
14. **Receipt Date** - `RCV_TRANSACTIONS.TRANSACTION_DATE` (formatted as DD-MON-YYYY)
15. **Receipt Amount** - `RCV_TRANSACTIONS.QUANTITY * RCV_TRANSACTIONS.UNIT_PRICE` or `RCV_SHIPMENT_LINES.AMOUNT` (rounded to 2 decimals)
16. **Invoice Number** - `AP_INVOICES_ALL.INVOICE_NUM` (via `AP_INVOICE_LINES_ALL.RCV_TRANSACTION_ID`)
17. **Invoice Date** - `AP_INVOICES_ALL.INVOICE_DATE` (formatted as DD-MON-YYYY)
18. **Invoice Status** - `FND_LOOKUP_VALUES_TL.MEANING` (from `AP_INVOICES_ALL.APPROVAL_STATUS`)
19. **Invoice Amount** - `AP_INVOICES_ALL.INVOICE_AMOUNT` (rounded to 2 decimals)
20. **Invoice Currency** - `AP_INVOICES_ALL.INVOICE_CURRENCY_CODE`
21. **Payment Number** - `AP_CHECKS_ALL.CHECK_NUMBER` (via `AP_INVOICE_PAYMENTS_ALL.INVOICE_ID`)
22. **Payment Date** - `AP_CHECKS_ALL.CHECK_DATE` (formatted as DD-MON-YYYY)
23. **Payment Amount** - `AP_INVOICE_PAYMENTS_ALL.AMOUNT` (rounded to 2 decimals)

---

## Business Rules

### Data Filters
- **PR Creation Date Range:** `TRUNC(POR_REQUISITION_HEADERS_ALL.CREATION_DATE) BETWEEN TRUNC(:P_PR_FROM_DATE) AND TRUNC(:P_PR_TO_DATE)`
- **Business Unit Name Filter:** `NVL(:P_BUSINESS_UNIT_NAME, BU_NAME) = BU_NAME`
  - **CRITICAL:** Uses NVL pattern to make parameter optional - if parameter is NULL, all business units are included; if provided, filters by business unit name
- **Invoice Filter:** `AP_INVOICES_ALL.CANCELLED_DATE IS NULL` (exclude cancelled invoices)
- **Payment Filter:** `AP_CHECKS_ALL.VOID_DATE IS NULL` (exclude voided payments)
- **Receipt Transaction Type:** `RCV_TRANSACTIONS.TRANSACTION_TYPE = 'RECEIVE'` (only receive transactions)

### Row Structure
- **One row per PR Distribution** - Report shows distribution-level detail (one PR can have multiple distributions)
- **Left Outer Joins for Optional Stages:** 
  - PR may not have PO (if not converted)
  - PO may not have Receipt (if not received)
  - Receipt may not have Invoice (if not invoiced)
  - Invoice may not have Payment (if not paid)
- **Multiple Payments per Invoice:** If invoice has multiple payments, each payment appears as separate row

### Critical Calculations

#### Requisition Amount Calculation (CRITICAL)
- **Primary Source:** `POR_REQUISITION_LINES_ALL.ASSESSABLE_VALUE`
- **Fallback Logic:** `NVL(ASSESSABLE_VALUE, CASE WHEN CURRENCY_UNIT_PRICE IS NOT NULL AND QUANTITY IS NOT NULL THEN (CURRENCY_UNIT_PRICE * QUANTITY * NVL(RATE, 1)) ELSE NVL(CURRENCY_AMOUNT, 0) * NVL(RATE, 1) END)`
- **Rounding:** `ROUND(..., 2)` for 2 decimal places
- **Why:** `ASSESSABLE_VALUE` is the primary amount field, with fallback to calculated values when not available

#### PO Amount Calculation (CRITICAL)
- **Source:** `(NVL(PO_DISTRIBUTIONS_ALL.TAX_EXCLUSIVE_AMOUNT, 0) + NVL(PO_DISTRIBUTIONS_ALL.NONRECOVERABLE_TAX, 0)) * NVL(PO_HEADERS_ALL.RATE, 1)`
- **Aggregation:** Sum by `PO_HEADER_ID` and `PO_LINE_ID` if multiple distributions exist
- **Rounding:** `ROUND(..., 2)` for 2 decimal places
- **Why:** Distribution-level calculation provides accurate amounts when tax is involved, multiplied by exchange rate for currency conversion

#### Receipt Amount Calculation (CRITICAL)
- **Primary Source:** `RCV_SHIPMENT_LINES.AMOUNT`
- **Alternative:** `RCV_TRANSACTIONS.QUANTITY * RCV_TRANSACTIONS.UNIT_PRICE` (if amount not available)
- **Aggregation:** Sum by `SHIPMENT_HEADER_ID` if multiple receipt lines exist
- **Rounding:** `ROUND(..., 2)` for 2 decimal places
- **Why:** Receipt amount may be stored at shipment line level or calculated from quantity and price

#### Invoice Amount (CRITICAL)
- **Source:** `AP_INVOICES_ALL.INVOICE_AMOUNT`
- **Filter:** `AP_INVOICES_ALL.CANCELLED_DATE IS NULL`
- **Rounding:** `ROUND(..., 2)` for 2 decimal places
- **Why:** Invoice amount is stored at header level, must exclude cancelled invoices

#### Payment Amount (CRITICAL)
- **Source:** `AP_INVOICE_PAYMENTS_ALL.AMOUNT`
- **Filter:** `AP_CHECKS_ALL.VOID_DATE IS NULL`
- **Aggregation:** Sum by `INVOICE_ID` if multiple payments exist
- **Rounding:** `ROUND(..., 2)` for 2 decimal places
- **Why:** Payment amount is at invoice payment level, must exclude voided payments

#### Requester Name (CRITICAL)
- **Source:** `PER_PERSON_NAMES_F.FIRST_NAME || ' ' || PER_PERSON_NAMES_F.LAST_NAME`
- **Date-Effective Filter:** `TRUNC(SYSDATE) BETWEEN TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_START_DATE) AND TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_END_DATE)`
- **Name Type Filter:** `PER_PERSON_NAMES_F.NAME_TYPE = 'GLOBAL'`
- **Why:** Person names are date-effective, must filter by current effective date range

#### Buyer Name (CRITICAL)
- **Source:** `PER_PERSON_NAMES_F.DISPLAY_NAME` (alternative: `FULL_NAME` or `FIRST_NAME || ' ' || LAST_NAME`)
- **Link:** `PO_HEADERS_ALL.AGENT_ID = PER_ALL_PEOPLE_F.PERSON_ID` (NOT `BUYER_ID`)
- **Date-Effective Filter:** `TRUNC(SYSDATE) BETWEEN TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_START_DATE) AND TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_END_DATE)`
- **Name Type Filter:** `PER_PERSON_NAMES_F.NAME_TYPE = 'GLOBAL'`
- **Why:** Buyer is stored as `AGENT_ID` in PO header, not `BUYER_ID`

#### PR Status Lookup (CRITICAL)
- **Source:** `FND_LOOKUP_VALUES_TL.MEANING`
- **Lookup Type:** `'POR_DOCUMENT_STATUS'`
- **Filters:** `VIEW_APPLICATION_ID = 0`, `SET_ID = 0`, `LANGUAGE = USERENV('LANG')`
- **Join:** `POR_REQUISITION_HEADERS_ALL.DOCUMENT_STATUS = FND_LOOKUP_VALUES_TL.LOOKUP_CODE`
- **Why:** PR status meanings are stored in FND lookup, not directly in PR header table

#### PO Status Lookup (CRITICAL)
- **Source:** `FND_LOOKUP_VALUES_TL.MEANING`
- **Lookup Type:** `'DOCUMENT_STATUS'` or `'PO_STATUS'` (validate against system)
- **Filters:** `VIEW_APPLICATION_ID = 0`, `SET_ID = 0`, `LANGUAGE = USERENV('LANG')`
- **Join:** `PO_HEADERS_ALL.DOCUMENT_STATUS = FND_LOOKUP_VALUES_TL.LOOKUP_CODE`
- **Why:** PO status meanings are stored in FND lookup

#### Invoice Status Lookup (CRITICAL)
- **Source:** `FND_LOOKUP_VALUES_TL.MEANING`
- **Lookup Type:** `'INVOICE APPROVAL STATUS'` or decode from `AP_INVOICES_ALL.APPROVAL_STATUS`
- **Alternative:** Use decode pattern from AP_MASTER.md:
  ```sql
  DECODE(AIA.APPROVAL_STATUS,
      'FULL', 'FULLY APPLIED',
      'NEVER APPROVED', 'NEVER VALIDATED',
      'NEEDS REAPPROVAL', 'NEEDS REVALIDATION',
      'CANCELLED', 'CANCELLED',
      'UNPAID', 'UNPAID',
      'AVAILABLE', 'AVAILABLE',
      'UNAPPROVED', 'UNVALIDATED',
      'APPROVED', 'VALIDATED',
      'PERMANENT', 'PERMANENT PREPAYMENT',
      NULL
  ) INVOICE_STATUS
  ```

### COA Segments Source
- **Not Required for This Report:** This report does not display COA segments, but charge accounts are available at PR distribution level if needed for future enhancements

---

## COA Segment Mapping

Reference: `CONFIGURATION/ENV_METADATA.md`

**Note:** This report does not currently display COA segments, but the following segments are available if needed:
- `SEGMENT1` - Entity
- `SEGMENT2` - NaturalAccount
- `SEGMENT3` - CostCenter
- `SEGMENT4` - Project
- `SEGMENT5` - ContractType
- `SEGMENT6` - InterCompany

---

## SQL Standards (MANDATORY)

1. **Join Syntax:** Oracle Traditional Syntax (comma-separated tables with WHERE clause)
2. **Outer Joins:** Use `(+)` operator on optional side
3. **CTE Structure:** All CTEs must have `/*+ qb_name(NAME) */` hint
4. **CTE Hints:** Use `/*+ MATERIALIZE */` for CTEs reused 2+ times or containing complex logic
5. **Parallel Hints:** Use `/*+ PARALLEL(2) */` for large table scans (>500K rows)
6. **Multi-Tenant:** Include `ORG_ID` or `BU_ID` in joins where applicable
7. **Date Formatting:** Use `TO_CHAR(date, 'DD-MON-YYYY')` for date columns
8. **Ampersand Constraint:** NEVER use `&` symbol - use word "AND" instead
9. **XLA Application ID:** Filter `XLA_AE_LINES` by `APPLICATION_ID = 200` for AP data (if used)

---

## Required CTE Structure

1. **PARAMS** - Input parameters (date range, business unit name)
2. **PR_MASTER** - PR headers filtered by creation date range
3. **PR_LINES** - PR lines with amount calculation
4. **PR_DISTRIBUTIONS** - PR distributions (one row per distribution)
5. **REQUESTER_MASTER** - Requester names (date-effective)
6. **PR_STATUS_LOOKUP** - PR status meanings from FND lookup
7. **PO_MASTER** - Purchase order headers (for PO number, supplier, buyer, currency, status)
8. **PO_DISTRIBUTIONS** - PO distributions with amount calculation
9. **BUYER_MASTER** - Buyer names (date-effective, from AGENT_ID)
10. **PO_STATUS_LOOKUP** - PO status meanings from FND lookup
11. **SUPPLIER_MASTER** - Supplier names
12. **BUSINESS_UNIT_MASTER** - Business unit names
13. **RECEIPT_MASTER** - Receipt headers and transactions (linked to PO)
14. **RECEIPT_AMOUNTS** - Receipt amounts aggregated by shipment
15. **INVOICE_MASTER** - Invoice headers (linked to receipts via invoice lines)
16. **INVOICE_STATUS_LOOKUP** - Invoice status meanings (decode or FND lookup)
17. **PAYMENT_MASTER** - Payment checks (linked to invoices via invoice payments)
18. **PAYMENT_AMOUNTS** - Payment amounts aggregated by invoice
19. **P2P_DETAIL** - Final join of all CTEs with business logic and left outer joins for optional stages

---

## Sorting

- **Primary:** Requisition Number (ascending)
- **Secondary:** Requisition Creation Date (descending)
- **Tertiary:** PO Number (ascending, if exists)
- **Quaternary:** Receipt Number (ascending, if exists)
- **Quinary:** Invoice Number (ascending, if exists)
- **Senary:** Payment Number (ascending, if exists)

---

## Key Tables

| Table | Alias | Purpose |
|-------|-------|---------|
| `POR_REQUISITION_HEADERS_ALL` | PRHA | PR Header (Number, Creation Date, Status, Requester ID) |
| `POR_REQUISITION_LINES_ALL` | PRLA | PR Lines (Amount, PO_HEADER_ID link) |
| `POR_REQ_DISTRIBUTIONS_ALL` | PRDA | PR Distributions (Distribution Number) |
| `PER_PERSON_NAMES_F` | PPNF | Requester Names (date-effective) |
| `PO_HEADERS_ALL` | PHA | Purchase Order Headers (PO Number, Supplier, Buyer/Agent, Currency, Status) |
| `PO_LINES_ALL` | PLA | Purchase Order Lines |
| `PO_DISTRIBUTIONS_ALL` | PDA | PO Distributions (Amount, Tax) |
| `POZ_SUPPLIERS_V` | PSV | Supplier Master (Supplier Name) |
| `POZ_SUPPLIER_SITES_V` | PSSV | Supplier Sites (Business Unit) |
| `FUN_ALL_BUSINESS_UNITS_V` | FABUV | Business Unit Master (BU Name) |
| `PER_ALL_PEOPLE_F` | PAPF | People Master (for Buyer/Agent link) |
| `RCV_SHIPMENT_HEADERS` | RSH | Receipt Headers (Receipt Number) |
| `RCV_SHIPMENT_LINES` | RSL | Receipt Lines (Amount) |
| `RCV_TRANSACTIONS` | RT | Receipt Transactions (Transaction Date, Quantity, Price) |
| `AP_INVOICES_ALL` | AIA | Invoice Headers (Invoice Number, Date, Amount, Currency, Status) |
| `AP_INVOICE_LINES_ALL` | AILA | Invoice Lines (RCV_TRANSACTION_ID link) |
| `AP_INVOICE_PAYMENTS_ALL` | AIPA | Invoice Payments (Payment Amount) |
| `AP_CHECKS_ALL` | ACA | Payment Checks (Payment Number, Date) |
| `FND_LOOKUP_VALUES_TL` | FLVT | Lookup Values (PR Status, PO Status, Invoice Status meanings) |

---

## Critical Constraints

### CRITICAL: PR Table Names
- **Rule:** Use `POR_REQUISITION_HEADERS_ALL` (NOT `PO_REQUISITION_HEADERS_ALL`)
- **PR Number:** `POR_REQUISITION_HEADERS_ALL.REQUISITION_NUMBER` (NOT `SEGMENT1`)
- **PR Status:** `POR_REQUISITION_HEADERS_ALL.DOCUMENT_STATUS` (NOT `STATUS_CODE`)

### CRITICAL: PR to PO Link
- **Rule:** `POR_REQUISITION_LINES_ALL.PO_HEADER_ID` provides direct link to PO (if PR converted to PO)
- **Alternative Path:** Can also link via distributions: `PO_DISTRIBUTIONS_ALL.REQ_DISTRIBUTION_ID = POR_REQ_DISTRIBUTIONS_ALL.DISTRIBUTION_ID`
- **Why:** PR lines can have direct `PO_HEADER_ID` reference when converted to PO

### CRITICAL: PR Amount Calculation
- **Rule:** Use `POR_REQUISITION_LINES_ALL.ASSESSABLE_VALUE` with fallback logic
- **Calculation:** `NVL(ASSESSABLE_VALUE, CASE WHEN CURRENCY_UNIT_PRICE IS NOT NULL AND QUANTITY IS NOT NULL THEN (CURRENCY_UNIT_PRICE * QUANTITY * NVL(RATE, 1)) ELSE NVL(CURRENCY_AMOUNT, 0) * NVL(RATE, 1) END)`

### CRITICAL: Buyer/Agent Column
- **Rule:** Use `PO_HEADERS_ALL.AGENT_ID` (NOT `BUYER_ID`)
- **Why:** `BUYER_ID` does not exist in `PO_HEADERS_ALL`. `AGENT_ID` links to `PER_ALL_PEOPLE_F.PERSON_ID` for buyer information

### CRITICAL: PO Amount Calculation
- **Rule:** Calculate PO amount from distributions: `(NONRECOVERABLE_TAX + TAX_EXCLUSIVE_AMOUNT) * RATE`
- **Source:** `PO_DISTRIBUTIONS_ALL` aggregated by `PO_HEADER_ID` and `PO_LINE_ID`
- **Why:** Distribution-level calculation provides more accurate amounts when tax is involved

### CRITICAL: PO to Receipt Link
- **Rule:** Link via `RCV_TRANSACTIONS.PO_HEADER_ID = PO_HEADERS_ALL.PO_HEADER_ID`
- **Filter:** `RCV_TRANSACTIONS.TRANSACTION_TYPE = 'RECEIVE'` (only receive transactions)
- **Why:** Receipt transactions link directly to PO header

### CRITICAL: Receipt to Invoice Link
- **Rule:** Link via `AP_INVOICE_LINES_ALL.RCV_TRANSACTION_ID = RCV_TRANSACTIONS.TRANSACTION_ID`
- **Then:** `AP_INVOICE_LINES_ALL.INVOICE_ID = AP_INVOICES_ALL.INVOICE_ID`
- **Why:** Invoice lines contain receipt transaction ID for 3-way matching

### CRITICAL: Invoice to Payment Link
- **Rule:** Link via `AP_INVOICE_PAYMENTS_ALL.INVOICE_ID = AP_INVOICES_ALL.INVOICE_ID`
- **Then:** `AP_INVOICE_PAYMENTS_ALL.CHECK_ID = AP_CHECKS_ALL.CHECK_ID`
- **Filter:** `AP_CHECKS_ALL.VOID_DATE IS NULL` (exclude voided payments)
- **Why:** Invoice payments table links invoices to payment checks

### CRITICAL: Invoice Filter
- **Rule:** `AP_INVOICES_ALL.CANCELLED_DATE IS NULL` (exclude cancelled invoices)
- **Why:** Cancelled invoices should not appear in standard P2P reports

### CRITICAL: Business Unit Source
- **Rule:** Use `FUN_ALL_BUSINESS_UNITS_V` (Financial BU view) for P2P reports
- **Join:** `PO_HEADERS_ALL.PRC_BU_ID = FUN_ALL_BUSINESS_UNITS_V.BU_ID` OR `POR_REQUISITION_HEADERS_ALL.REQ_BU_ID = FUN_ALL_BUSINESS_UNITS_V.BU_ID`
- **Why:** Financial BU view aligns with AP/AR modules and provides better integration for P2P reports

### CRITICAL: Requester Name Date-Effective
- **Rule:** Must filter `TRUNC(SYSDATE) BETWEEN TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_START_DATE) AND TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_END_DATE)`
- **Filter:** `PER_PERSON_NAMES_F.NAME_TYPE = 'GLOBAL'`

### CRITICAL: Buyer Name Date-Effective
- **Rule:** Must filter `TRUNC(SYSDATE) BETWEEN TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_START_DATE) AND TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_END_DATE)`
- **Filter:** `PER_PERSON_NAMES_F.NAME_TYPE = 'GLOBAL'`
- **Link:** `PO_HEADERS_ALL.AGENT_ID = PER_ALL_PEOPLE_F.PERSON_ID` then `PER_ALL_PEOPLE_F.PERSON_ID = PER_PERSON_NAMES_F.PERSON_ID`

### CRITICAL: Business Unit Name Parameter NVL
- **Rule:** Parameter filter must use NVL pattern to make parameter optional
- **Pattern:** `NVL(:P_BUSINESS_UNIT_NAME, BU_NAME) = BU_NAME`
- **Logic:** 
  - If parameter is NULL: `BU_NAME = BU_NAME` (always true - shows all business units)
  - If parameter is NOT NULL: `:P_BUSINESS_UNIT_NAME = BU_NAME` (filters by business unit name)
- **Why:** NVL pattern allows optional parameter filtering without using OR condition

### CRITICAL: Left Outer Joins for Optional Stages
- **Rule:** All stages after PR must use left outer joins `(+)` because:
  - PR may not have PO (if not converted)
  - PO may not have Receipt (if not received)
  - Receipt may not have Invoice (if not invoiced)
  - Invoice may not have Payment (if not paid)
- **Why:** Report must show complete PR lifecycle even if later stages are missing

### CRITICAL: Multiple Payments per Invoice
- **Rule:** If invoice has multiple payments, each payment appears as separate row
- **Aggregation:** Do NOT aggregate payments at invoice level - show payment-level detail
- **Why:** Report must show all payments for each invoice

---

## Notes

- **Multiple Distributions:** One PR can have multiple distributions - each distribution appears as separate row
- **PO Link:** PO number, supplier, buyer, and business unit are only populated if PR has been converted to PO (PO_HEADER_ID is not NULL)
- **Receipt Link:** Receipt number and date are only populated if PO has been received (RCV_TRANSACTION_ID exists)
- **Invoice Link:** Invoice number, date, and amount are only populated if receipt has been invoiced (AP_INVOICE_LINES_ALL.RCV_TRANSACTION_ID exists)
- **Payment Link:** Payment number, date, and amount are only populated if invoice has been paid (AP_INVOICE_PAYMENTS_ALL exists)
- **Currency:** Each stage may have different currency (PR currency, PO currency, Invoice currency) - report shows currency for each stage
- **Amount Aggregation:** If multiple distributions, receipts, or payments exist, amounts should be aggregated appropriately at each stage
- **Date Formatting:** All dates formatted as `DD-MON-YYYY` for consistency
- **Amount Rounding:** All amounts rounded to 2 decimal places

---

## Integration Flow Diagram

```
PR (POR_REQUISITION_HEADERS_ALL)
  ↓ (POR_REQUISITION_LINES_ALL.PO_HEADER_ID)
PO (PO_HEADERS_ALL)
  ↓ (RCV_TRANSACTIONS.PO_HEADER_ID)
Receipt (RCV_SHIPMENT_HEADERS)
  ↓ (AP_INVOICE_LINES_ALL.RCV_TRANSACTION_ID)
Invoice (AP_INVOICES_ALL)
  ↓ (AP_INVOICE_PAYMENTS_ALL.INVOICE_ID)
Payment (AP_CHECKS_ALL)
```

**Key Join Points:**
1. PR → PO: `POR_REQUISITION_LINES_ALL.PO_HEADER_ID`
2. PO → Receipt: `RCV_TRANSACTIONS.PO_HEADER_ID`
3. Receipt → Invoice: `AP_INVOICE_LINES_ALL.RCV_TRANSACTION_ID`
4. Invoice → Payment: `AP_INVOICE_PAYMENTS_ALL.INVOICE_ID`

---

**Last Updated:** 19-01-2026  
**Validation Status:** ✅ Validated against PO_MASTER.md, AP_MASTER.md, INV_MASTER.md, and CROSS_MODULE_MASTER.md

