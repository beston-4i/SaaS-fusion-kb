# PR Detail Report - Report Requirements

**Module:** SCM - Purchasing (PO)  
**Report Name:** PR Detail Report  
**Report Purpose:** Detailed listing of Purchase Requisitions with linked Purchase Orders, Requester information, Charge Accounts, and Supplier details  
**Created:** 19-01-2026

---

## Input Parameters

- **P_PR_FROM_DATE** (DATE, Required) - PR creation date from filter
- **P_PR_TO_DATE** (DATE, Required) - PR creation date to filter  
- **P_REQUESTER_NAME** (VARCHAR2, Optional) - Requester name filter (uses NVL for optional parameter)

---

## Output Columns (9 Total)

1. **Requisition Number** - `POR_REQUISITION_HEADERS_ALL.REQUISITION_NUMBER`
2. **Requester Name** - `PER_PERSON_NAMES_F.FIRST_NAME || ' ' || PER_PERSON_NAMES_F.LAST_NAME` (date-effective)
3. **Requisition Header Description** - `POR_REQUISITION_HEADERS_ALL.DESCRIPTION`
4. **Requisition Creation Date** - `POR_REQUISITION_HEADERS_ALL.CREATION_DATE` (formatted as DD-MON-YYYY)
5. **Requisition Status** - `FND_LOOKUP_VALUES_TL.MEANING` (from `POR_REQUISITION_HEADERS_ALL.DOCUMENT_STATUS`)
6. **Requisition Amount** - `POR_REQUISITION_LINES_ALL.ASSESSABLE_VALUE` with fallback calculation (rounded to 2 decimals)
7. **Purchase Order Number** - `PO_HEADERS_ALL.SEGMENT1` (via `POR_REQUISITION_LINES_ALL.PO_HEADER_ID`)
8. **Charge Account** - `GL_CODE_COMBINATIONS.CONCATENATED_SEGMENTS` (from `POR_REQ_DISTRIBUTIONS_ALL.CODE_COMBINATION_ID`)
9. **Supplier Name** - `POZ_SUPPLIERS_V.VENDOR_NAME` (via PO link)

---

## Business Rules

### Data Filters
- **PR Creation Date Range:** `TRUNC(POR_REQUISITION_HEADERS_ALL.CREATION_DATE) BETWEEN TRUNC(:P_PR_FROM_DATE) AND TRUNC(:P_PR_TO_DATE)`
- **Requester Name Filter:** `NVL(:P_REQUESTER_NAME, REQUESTER_NAME) = REQUESTER_NAME`
  - **CRITICAL:** Uses NVL pattern to make parameter optional - if parameter is NULL, all requesters are included; if provided, filters by requester name

### Row Structure
- **One row per PR Distribution** - Report shows distribution-level detail (one PR can have multiple distributions with different charge accounts)
- **PR to PO Link:** Uses `POR_REQUISITION_LINES_ALL.PO_HEADER_ID` for direct link to PO (if PR converted to PO)
- **Multiple Distributions:** If PR has multiple distributions, each distribution appears as separate row with its charge account

### Critical Calculations

#### PR Amount Calculation (CRITICAL)
- **Primary Source:** `POR_REQUISITION_LINES_ALL.ASSESSABLE_VALUE`
- **Fallback Logic:** `NVL(ASSESSABLE_VALUE, CASE WHEN CURRENCY_UNIT_PRICE IS NOT NULL AND QUANTITY IS NOT NULL THEN (CURRENCY_UNIT_PRICE * QUANTITY * NVL(RATE, 1)) ELSE NVL(CURRENCY_AMOUNT, 0) * NVL(RATE, 1) END)`
- **Rounding:** `ROUND(..., 2)` for 2 decimal places
- **Why:** `ASSESSABLE_VALUE` is the primary amount field, with fallback to calculated values when not available

#### Requester Name (CRITICAL)
- **Source:** `PER_PERSON_NAMES_F.FIRST_NAME || ' ' || PER_PERSON_NAMES_F.LAST_NAME`
- **Date-Effective Filter:** `TRUNC(SYSDATE) BETWEEN TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_START_DATE) AND TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_END_DATE)`
- **Name Type Filter:** `PER_PERSON_NAMES_F.NAME_TYPE = 'GLOBAL'`
- **Why:** Person names are date-effective, must filter by current effective date range

#### PR Status Lookup (CRITICAL)
- **Source:** `FND_LOOKUP_VALUES_TL.MEANING`
- **Lookup Type:** `'POR_DOCUMENT_STATUS'`
- **Filters:** `VIEW_APPLICATION_ID = 0`, `SET_ID = 0`, `LANGUAGE = USERENV('LANG')`
- **Join:** `POR_REQUISITION_HEADERS_ALL.DOCUMENT_STATUS = FND_LOOKUP_VALUES_TL.LOOKUP_CODE`
- **Why:** PR status meanings are stored in FND lookup, not directly in PR header table

### COA Segments Source
- **Charge Account:** `POR_REQ_DISTRIBUTIONS_ALL.CODE_COMBINATION_ID` links to `GL_CODE_COMBINATIONS.CODE_COMBINATION_ID`
- **Display:** Use `GL_CODE_COMBINATIONS.CONCATENATED_SEGMENTS` for full account string
- **Note:** Charge account is at distribution level, not header or line level

---

## COA Segment Mapping

Reference: `CONFIGURATION/ENV_METADATA.md`

| Segment | Business Name | Report Usage |
|---------|---------------|--------------|
| `SEGMENT1` | Entity | Part of Charge Account display |
| `SEGMENT2` | NaturalAccount | Part of Charge Account display |
| `SEGMENT3` | CostCenter | Part of Charge Account display |
| `SEGMENT4` | Project | Part of Charge Account display |
| `SEGMENT5` | ContractType | Part of Charge Account display |
| `SEGMENT6` | InterCompany | Part of Charge Account display |

**Display Format:** Use `GL_CODE_COMBINATIONS.CONCATENATED_SEGMENTS` for complete account string

---

## SQL Standards (MANDATORY)

1. **Join Syntax:** Oracle Traditional Syntax (comma-separated tables with WHERE clause)
2. **Outer Joins:** Use `(+)` operator on optional side
3. **CTE Structure:** All CTEs must have `/*+ qb_name(NAME) */` hint
4. **CTE Hints:** Use `/*+ MATERIALIZE */` for CTEs reused 2+ times or containing complex logic
5. **Parallel Hints:** Use `/*+ PARALLEL(2) */` for large table scans (>500K rows)
6. **Multi-Tenant:** Include `REQ_BU_ID` in joins where applicable
7. **Date Formatting:** Use `TO_CHAR(date, 'DD-MON-YYYY')` for date columns
8. **Ampersand Constraint:** NEVER use `&` symbol - use word "AND" instead

---

## Required CTE Structure

1. **PARAMS** - Input parameters (date range, requester name)
2. **PR_MASTER** - PR headers filtered by creation date range
3. **PR_LINES** - PR lines with amount calculation
4. **PR_DISTRIBUTIONS** - PR distributions with charge account
5. **REQUESTER_MASTER** - Requester names (date-effective)
6. **PR_STATUS_LOOKUP** - PR status meanings from FND lookup
7. **PO_MASTER** - Purchase order headers (for PO number and supplier)
8. **SUPPLIER_MASTER** - Supplier names
9. **CHARGE_ACCOUNT** - GL code combinations for charge accounts
10. **PR_DETAIL** - Final join of all CTEs with business logic

---

## Sorting

- **Primary:** Requisition Number (ascending)
- **Secondary:** Requisition Creation Date (descending)
- **Tertiary:** Distribution Number (ascending)

---

## Key Tables

| Table | Alias | Purpose |
|-------|-------|---------|
| `POR_REQUISITION_HEADERS_ALL` | PRHA | PR Header (Number, Description, Creation Date, Status) |
| `POR_REQUISITION_LINES_ALL` | PRLA | PR Lines (Amount, Requester ID, PO_HEADER_ID) |
| `POR_REQ_DISTRIBUTIONS_ALL` | PRDA | PR Distributions (Charge Account, Distribution Number) |
| `PER_PERSON_NAMES_F` | PPNF | Requester Names (date-effective) |
| `PO_HEADERS_ALL` | PHA | Purchase Order Headers (PO Number, Supplier) |
| `POZ_SUPPLIERS_V` | PSV | Supplier Master (Supplier Name) |
| `GL_CODE_COMBINATIONS` | GCC | Chart of Accounts (Charge Account segments) |
| `FND_LOOKUP_VALUES_TL` | FLVT | Lookup Values (PR Status meanings) |

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

### CRITICAL: Charge Account Column
- **Rule:** Use `POR_REQ_DISTRIBUTIONS_ALL.CODE_COMBINATION_ID` (NOT `CHARGE_ACCOUNT_ID`)
- **Display:** Use `GL_CODE_COMBINATIONS.CONCATENATED_SEGMENTS` for full account string

### CRITICAL: Requester Name Date-Effective
- **Rule:** Must filter `TRUNC(SYSDATE) BETWEEN TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_START_DATE) AND TRUNC(PER_PERSON_NAMES_F.EFFECTIVE_END_DATE)`
- **Filter:** `PER_PERSON_NAMES_F.NAME_TYPE = 'GLOBAL'`

### CRITICAL: Requester Name Parameter NVL
- **Rule:** Parameter filter must use NVL pattern to make parameter optional
- **Pattern:** `NVL(:P_REQUESTER_NAME, REQUESTER_NAME) = REQUESTER_NAME`
- **Logic:** 
  - If parameter is NULL: `REQUESTER_NAME = REQUESTER_NAME` (always true - shows all requesters)
  - If parameter is NOT NULL: `:P_REQUESTER_NAME = REQUESTER_NAME` (filters by requester name)
- **Why:** NVL pattern allows optional parameter filtering without using OR condition

---

## Notes

- **Multiple Distributions:** One PR can have multiple distributions with different charge accounts - each distribution appears as separate row
- **PO Link:** PO number and supplier are only populated if PR has been converted to PO (PO_HEADER_ID is not NULL)
- **Supplier Link:** Supplier is linked via PO, so if PR not converted to PO, supplier will be NULL
- **Currency:** PR amounts are in PR currency - no currency conversion applied unless specified in requirements

