# Negotiation Detailed Report - Report Requirements

**Module:** SCM - Sourcing/Negotiations (SOURCING)  
**Report Name:** Negotiation Detailed Report  
**Report Purpose:** Comprehensive negotiation details with requisition, line-level, pricing, and purchase order information  
**Created:** 19-01-2026

---

## Input Parameters

- **P_REQ_BU_NAME** (VARCHAR2, Optional) - Requisition Business Unit name filter
- **P_PO_NUMBER** (VARCHAR2, Optional) - Purchase Order Number filter (via PR to PO link)
- **P_NEGOTIATION_NUMBER** (VARCHAR2, Optional) - Negotiation Number filter
- **P_OPEN_DATE_FROM** (DATE, Optional) - Negotiation open date creation date from filter
- **P_OPEN_DATE_TO** (DATE, Optional) - Negotiation open date creation date to filter

---

## Output Columns (15 Total)

1. **Negotiation Number** - `PON_AUCTION_HEADERS_ALL.DOCUMENT_NUMBER`
2. **Negotiation Title** - `PON_AUCTION_HEADERS_ALL.AUCTION_TITLE`
3. **Negotiation Status** - `PON_AUCTION_HEADERS_ALL.AUCTION_STATUS`
4. **Negotiation Open Date** - `PON_AUCTION_HEADERS_ALL.OPEN_BIDDING_DATE`
5. **Negotiation Closed Date** - `PON_AUCTION_HEADERS_ALL.CLOSE_BIDDING_DATE`
6. **Negotiation Line Number** - `PON_AUCTION_ITEM_PRICES_ALL.LINE_NUMBER`
7. **Negotiation Line Description** - `PON_AUCTION_ITEM_PRICES_ALL.ITEM_DESCRIPTION`
8. **Requisition BU** - `HR_ALL_ORGANIZATION_UNITS.NAME` (via `POR_REQUISITION_HEADERS_ALL.REQ_BU_ID`)
9. **Line Type** - `PON_AUCTION_ITEM_PRICES_ALL.ORDER_TYPE_LOOKUP_CODE`
10. **Category Name** - `EGP_CATEGORIES_VL.CATEGORY_NAME`
11. **Requested Delivery Date** - `PON_AUCTION_ITEM_PRICES_ALL.REQUESTED_DELIVERY_DATE`
12. **Current Price** - `PON_AUCTION_ITEM_PRICES_ALL.CURRENT_PRICE`
13. **PR Number** - `PON_AUCTION_ITEM_PRICES_ALL.REQUISITION_NUMBER` (direct column, no join required)
14. **PR Line Number** - `POR_REQUISITION_LINES_ALL.LINE_NUMBER` (via REQUISITION_NUMBER join)
15. **PO Number** - `PO_HEADERS_ALL.SEGMENT1` (via PR to PO link: `POR_REQUISITION_LINES_ALL.PO_HEADER_ID`)

---

## Business Rules

### Data Filters
- **Negotiation Number Filter:** `P_NEGOTIATION_NUMBER IS NULL OR PON_AUCTION_HEADERS_ALL.DOCUMENT_NUMBER = P_NEGOTIATION_NUMBER`
- **Negotiation Open Date Range:** `P_OPEN_DATE_FROM IS NULL OR TRUNC(PON_AUCTION_HEADERS_ALL.OPEN_BIDDING_DATE) >= TRUNC(P_OPEN_DATE_FROM)` AND `P_OPEN_DATE_TO IS NULL OR TRUNC(PON_AUCTION_HEADERS_ALL.OPEN_BIDDING_DATE) <= TRUNC(P_OPEN_DATE_TO)`
- **Requisition BU Filter:** `P_REQ_BU_NAME IS NULL OR UPPER(HR_ALL_ORGANIZATION_UNITS.NAME) = UPPER(P_REQ_BU_NAME)`
- **PO Number Filter:** `P_PO_NUMBER IS NULL OR PO_HEADERS_ALL.SEGMENT1 = P_PO_NUMBER`

### Row Structure
- **One row per Negotiation Line** - Report shows line-level detail for each negotiation line
- **PR Link:** PR Number is directly available from `PON_AUCTION_ITEM_PRICES_ALL.REQUISITION_NUMBER` (no join required for PR Number)
- **PR Line Link:** Join to `POR_REQUISITION_LINES_ALL` via `PON_AUCTION_ITEM_PRICES_ALL.REQUISITION_NUMBER = POR_REQUISITION_HEADERS_ALL.REQUISITION_NUMBER` then `POR_REQUISITION_HEADERS_ALL.REQUISITION_HEADER_ID = POR_REQUISITION_LINES_ALL.REQUISITION_HEADER_ID` to get PR Line Number
- **PO Link:** Join via PR lines: `POR_REQUISITION_LINES_ALL.PO_HEADER_ID = PO_HEADERS_ALL.PO_HEADER_ID` (if PR converted to PO)

### Critical Calculations

#### Current Price
- **Source:** `PON_AUCTION_ITEM_PRICES_ALL.CURRENT_PRICE` (direct column, no calculation needed)
- **CRITICAL:** Use `CURRENT_PRICE` (NOT `UNIT_PRICE`) - `UNIT_PRICE` column does not exist

---

## Critical Constraints

### CRITICAL: Negotiation Line Table
- **Rule:** Use `PON_AUCTION_ITEM_PRICES_ALL` (NOT `PON_AUCTION_LINES_ALL`)
- **Why:** `PON_AUCTION_LINES_ALL` does not exist. All negotiation line details are in `PON_AUCTION_ITEM_PRICES_ALL`.

### CRITICAL: Negotiation Date Columns
- **Rule:** Use `PON_AUCTION_HEADERS_ALL.OPEN_BIDDING_DATE` and `CLOSE_BIDDING_DATE` (NOT `OPEN_DATE` and `CLOSE_DATE`)
- **Why:** The correct column names for negotiation open and close dates are `OPEN_BIDDING_DATE` and `CLOSE_BIDDING_DATE`.

### CRITICAL: PR Number Direct Access
- **Rule:** `REQUISITION_NUMBER` is available directly in `PON_AUCTION_ITEM_PRICES_ALL` (no join required)
- **Why:** No need to join through `POR_REQUISITION_LINES_ALL` to get PR Number. It's a direct column.

### CRITICAL: Negotiation Line Description
- **Rule:** Use `PON_AUCTION_ITEM_PRICES_ALL.ITEM_DESCRIPTION` (NOT `DESCRIPTION`)
- **Why:** The column name is `ITEM_DESCRIPTION`, not `DESCRIPTION`.

### CRITICAL: Line Type Column
- **Rule:** Use `PON_AUCTION_ITEM_PRICES_ALL.ORDER_TYPE_LOOKUP_CODE` for line type (NOT `LINE_TYPE`)
- **Why:** The actual column name is `ORDER_TYPE_LOOKUP_CODE`.

### CRITICAL: Requested Delivery Date
- **Rule:** Use `PON_AUCTION_ITEM_PRICES_ALL.REQUESTED_DELIVERY_DATE` (NOT `NEED_BY_DATE`)
- **Why:** The column name is `REQUESTED_DELIVERY_DATE`, not `NEED_BY_DATE`.

### CRITICAL: Current Price
- **Rule:** Use `PON_AUCTION_ITEM_PRICES_ALL.CURRENT_PRICE` directly (NOT `UNIT_PRICE`)
- **Why:** `UNIT_PRICE` column does not exist in this table. Use `CURRENT_PRICE` only.

### CRITICAL: PR Line Number Link
- **Rule:** Join to `POR_REQUISITION_LINES_ALL` via PR Header: `PON_AUCTION_ITEM_PRICES_ALL.REQUISITION_NUMBER = POR_REQUISITION_HEADERS_ALL.REQUISITION_NUMBER` then `POR_REQUISITION_HEADERS_ALL.REQUISITION_HEADER_ID = POR_REQUISITION_LINES_ALL.REQUISITION_HEADER_ID`
- **Note:** Multiple PR lines may exist per PR Number. Since there is no direct link between negotiation lines and specific PR lines in `PON_AUCTION_ITEM_PRICES_ALL`, the query uses the first PR line (by LINE_NUMBER) as a default. This is a limitation - if a PR has multiple lines, only the first line number will be shown.
- **Implementation:** Uses `ROW_NUMBER() OVER (PARTITION BY REQUISITION_HEADER_ID ORDER BY LINE_NUMBER)` to select the first PR line per PR header.

### CRITICAL: PO Number Link
- **Rule:** Link via PR lines: `POR_REQUISITION_LINES_ALL.PO_HEADER_ID = PO_HEADERS_ALL.PO_HEADER_ID` (if PR converted to PO)
- **Note:** Not all PR lines convert to PO. Use outer join to handle cases where PO may not exist.

---

## SQL Standards (MANDATORY)

1. **Join Syntax:** Oracle Traditional Syntax (NOT ANSI)
   - Use `FROM A, B WHERE A.ID = B.ID(+)` format
   - **NEVER** use `INNER JOIN` or `LEFT JOIN`

2. **Outer Join Operator:** Use `(+)` operator (NOT `LEFT JOIN`)

3. **CTE Structure:** 
   - **ALWAYS** use `/*+ qb_name(NAME) */` hint for every CTE
   - **ALWAYS** use `/*+ MATERIALIZE */` for CTEs reused 2+ times or containing complex logic
   - **ALWAYS** use `/*+ PARALLEL(2) */` for large table scans (>500K rows)

4. **Multi-Tenant:** 
   - Include `ORG_ID` or `BU_ID` in joins where applicable
   - **NEVER** assume Single-Org environment

5. **Ampersand Constraint (CRITICAL):**
   - **NEVER** use ampersand (&) symbol **ANYWHERE** in SQL queries
   - **NOT EVEN IN COMMENTS** - Use word "AND" instead of "&"

---

## Required CTE Structure

1. **PARAMS** - Input parameters (P_REQ_BU_NAME, P_PO_NUMBER, P_NEGOTIATION_NUMBER, P_OPEN_DATE_FROM, P_OPEN_DATE_TO)
2. **NEGOTIATION_HEADER_MASTER** - Negotiation headers with filters (Negotiation Number, Open Date Range)
3. **NEGOTIATION_LINE_MASTER** - Negotiation lines with all line details
4. **PR_HEADER_MASTER** - PR headers for BU filtering and PR Line Number link
5. **PR_LINE_MASTER** - PR lines for PR Line Number and PO link
6. **PO_HEADER_MASTER** - PO headers for PO Number (filtered by P_PO_NUMBER)
7. **CATEGORY_MASTER** - Category names lookup
8. **BU_MASTER** - Business Unit names lookup
9. **NEGOTIATION_DETAILED_FINAL** - Final join of all CTEs with business logic

---

## Sorting

- **Primary:** Negotiation Number (ascending)
- **Secondary:** Negotiation Line Number (ascending)
- **Tertiary:** PR Number (ascending)

---

## Key Tables

| Table | Alias | Purpose |
|-------|-------|---------|
| `PON_AUCTION_HEADERS_ALL` | PAH | Negotiation Header (Number, Title, Status, OPEN_BIDDING_DATE, CLOSE_BIDDING_DATE) |
| `PON_AUCTION_ITEM_PRICES_ALL` | PAIP | Negotiation Line (LINE_NUMBER, ITEM_DESCRIPTION, ORDER_TYPE_LOOKUP_CODE, REQUESTED_DELIVERY_DATE, CURRENT_PRICE, REQUISITION_NUMBER, CATEGORY_ID) |
| `POR_REQUISITION_HEADERS_ALL` | PRHA | Requisition Header (REQUISITION_NUMBER, REQ_BU_ID) - joined via REQUISITION_NUMBER |
| `POR_REQUISITION_LINES_ALL` | PRLA | Requisition Lines (LINE_NUMBER, PO_HEADER_ID) - for PR Line Number and PO link |
| `PO_HEADERS_ALL` | PHA | Purchase Order Headers (SEGMENT1 as PO Number) - joined via PR lines |
| `EGP_CATEGORIES_VL` | ECV | Category Names (CATEGORY_NAME) |
| `HR_ALL_ORGANIZATION_UNITS` | HAOU | Business Unit Names (NAME) |

---

## COA Segment Mapping

**Not Applicable** - This report does not require Chart of Accounts segments.

---

