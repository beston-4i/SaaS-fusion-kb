# Sourcing/Negotiations Templates

**Purpose:** Standard report patterns for Negotiation/Sourcing queries.

---

## 1. Negotiation Detailed Report
*Comprehensive negotiation details with requisition, line-level, and pricing information.*

```sql
/*
TITLE: Negotiation Detailed Report
PURPOSE: Comprehensive negotiation details with requisition, line-level, and pricing information
PARAMETERS: P_REQ_BU_NAME, P_NEGOTIATION_NUMBER, P_OPEN_DATE_FROM, P_OPEN_DATE_TO
*/

WITH
-- 1. Parameters CTE
PARAMS AS (
    SELECT /*+ qb_name(PARAMS) */
           :P_REQ_BU_NAME AS REQ_BU_NAME
          ,:P_NEGOTIATION_NUMBER AS NEGOTIATION_NUMBER
          ,TRUNC(:P_OPEN_DATE_FROM) AS OPEN_DATE_FROM
          ,TRUNC(:P_OPEN_DATE_TO) AS OPEN_DATE_TO
    FROM   DUAL
),

-- 2. Negotiation Header Master (from SOURCING_REPOSITORIES.md)
-- 3. Negotiation Line Master (from SOURCING_REPOSITORIES.md)
-- 4. Category Master (from SOURCING_REPOSITORIES.md)
-- 5. Business Unit Master
-- 6. Final Detailed Report

-- Final Select
SELECT /*+ LEADING(NDF) */
       NDF.NEGOTIATION_NUMBER
      ,NDF.NEGOTIATION_TITLE
      ,NDF.NEGOTIATION_STATUS
      ,NDF.NEGOTIATION_OPEN_DATE
      ,NDF.NEGOTIATION_CLOSED_DATE
      ,NDF.NEGOTIATION_LINE_NUMBER
      ,NDF.NEGOTIATION_LINE_DESCRIPTION
      ,NDF.REQUISITION_BU
      ,NDF.LINE_TYPE
      ,NDF.CATEGORY_NAME
      ,NDF.REQUESTED_DELIVERY_DATE
      ,NDF.CURRENT_PRICE
      ,NDF.PR_NUMBER
FROM   NEGOTIATION_DETAILED_FINAL NDF
ORDER BY NDF.NEGOTIATION_NUMBER
        ,NDF.NEGOTIATION_LINE_NUMBER
```

**Key Patterns:**
- Use `PON_AUCTION_ITEM_PRICES_ALL` (NOT `PON_AUCTION_LINES_ALL`)
- Use `OPEN_BIDDING_DATE` and `CLOSE_BIDDING_DATE` (NOT `OPEN_DATE` and `CLOSE_DATE`)
- PR Number: Direct from `PON_AUCTION_ITEM_PRICES_ALL.REQUISITION_NUMBER` (no join required)
- Join to `POR_REQUISITION_HEADERS_ALL` only for BU filtering/display
- Use `ITEM_DESCRIPTION`, `ORDER_TYPE_LOOKUP_CODE`, `REQUESTED_DELIVERY_DATE`, `CURRENT_PRICE`

---

