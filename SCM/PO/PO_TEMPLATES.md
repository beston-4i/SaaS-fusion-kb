# Purchasing Report Templates

**Purpose:** Ready-to-use SQL skeletons for PO reporting.

---

## 1. Open PO Report
*Detailed listing of POs with remaining open quantity.*

```sql
/*
TITLE: Open Purchase Orders
PURPOSE: Track POs not yet fully received
*/

WITH
-- 1. Get Headers
HEADERS AS (
    SELECT /*+ qb_name(H) MATERIALIZE */
           PHA.PO_HEADER_ID, PHA.PO_NUMBER, PHA.VENDOR_NAME
    FROM   PO_HEADERS_ALL PHA, POZ_SUPPLIERS_V POS
    WHERE  PHA.VENDOR_ID = POS.VENDOR_ID
      AND  PHA.CLOSED_CODE = 'OPEN'
),

-- 2. Get Open Lines
LINES AS (
    SELECT /*+ qb_name(L) MATERIALIZE */
           PLLA.PO_HEADER_ID, PLLA.QUANTITY, PLLA.QUANTITY_RECEIVED
          ,(PLLA.QUANTITY - NVL(PLLA.QUANTITY_RECEIVED, 0)) AS QTY_OPEN
           ,PLA.ITEM_DESCRIPTION
    FROM   PO_LINES_ALL PLA, PO_LINE_LOCATIONS_ALL PLLA
    WHERE  PLA.PO_LINE_ID = PLLA.PO_LINE_ID
      AND  NVL(PLLA.CANCEL_FLAG, 'N') = 'N'
      AND  PLLA.CLOSED_CODE != 'CLOSED'
)

-- 3. Final Select
SELECT H.PO_NUMBER
      ,H.VENDOR_NAME
      ,L.ITEM_DESCRIPTION
      ,L.QUANTITY AS QTY_ORDERED
      ,L.QTY_OPEN
FROM   HEADERS H
      ,LINES L
WHERE  H.PO_HEADER_ID = L.PO_HEADER_ID
  AND  L.QTY_OPEN > 0
ORDER BY H.PO_NUMBER
```

---

## 2. PO Detailed Report
*Comprehensive purchase order details with supplier, buyer, requisition, and line-level information.*

```sql
/*
TITLE: PO Detailed Report
PURPOSE: Comprehensive purchase order details with supplier, buyer, requisition, and line-level information
PARAMETERS: P_BU_NAME, P_SUPPLIER_NAME, P_BUYER_NAME, P_PO_NUMBER, P_CREATION_DATE_FROM, P_CREATION_DATE_TO
*/

WITH
-- 1. Parameters CTE
PARAMS AS (
    SELECT /*+ qb_name(PARAMS) */
           :P_BU_NAME AS BU_NAME
          ,:P_SUPPLIER_NAME AS SUPPLIER_NAME
          ,:P_BUYER_NAME AS BUYER_NAME
          ,:P_PO_NUMBER AS PO_NUMBER
          ,TRUNC(:P_CREATION_DATE_FROM) AS CREATION_DATE_FROM
          ,TRUNC(:P_CREATION_DATE_TO) AS CREATION_DATE_TO
    FROM   DUAL
),

-- 2. PO Header Master
PO_HEADER_MASTER AS (
    SELECT /*+ qb_name(PO_HDR) MATERIALIZE PARALLEL(2) */
           PHA.PO_HEADER_ID
          ,PHA.SEGMENT1 AS PO_NUMBER
          ,PHA.CREATION_DATE
          ,PHA.TYPE_LOOKUP_CODE AS PO_STATUS
          ,PHA.CURRENCY_CODE
          ,PHA.PRC_BU_ID
          ,PHA.VENDOR_ID
          ,PHA.VENDOR_SITE_ID
          ,PHA.AGENT_ID
          ,PHA.TERMS_ID
    FROM   PO_HEADERS_ALL PHA
          ,PARAMS P
    WHERE  1=1
      AND  (P.BU_NAME IS NULL OR EXISTS (
              SELECT 1 FROM HR_ALL_ORGANIZATION_UNITS HAOU
              WHERE HAOU.ORGANIZATION_ID = PHA.PRC_BU_ID
                AND UPPER(HAOU.NAME) = UPPER(P.BU_NAME)
            ))
      AND  (P.PO_NUMBER IS NULL OR PHA.SEGMENT1 = P.PO_NUMBER)
      AND  (P.CREATION_DATE_FROM IS NULL OR TRUNC(PHA.CREATION_DATE) >= P.CREATION_DATE_FROM)
      AND  (P.CREATION_DATE_TO IS NULL OR TRUNC(PHA.CREATION_DATE) <= P.CREATION_DATE_TO)
),

-- 3. Supplier Master (from PO_REPOSITORIES.md)
-- 4. Supplier Site Master (Address concatenation)
-- 5. Supplier Contact Master (from PO_REPOSITORIES.md)
-- 6. Payment Terms Master (AP_TERMS.NAME)
-- 7. Buyer Master (Date-Effective via AGENT_ID)
-- 8. Buyer Email Master
-- 9. PO Line Master
-- 10. PO Line Location Master
-- 11. PO Tax Aggregate (from PO_REPOSITORIES.md)
-- 12. PR Distribution Master (from PO_REPOSITORIES.md)
-- 13. PR Master (via distributions)
-- 14. PR Requester Master (Date-Effective)
-- 15. BU Master

-- Final Select
SELECT /*+ LEADING(PDF) */
       PDF.PO_NUMBER
      ,PDF.PO_CREATION_DATE
      ,PDF.PO_STATUS
      ,PDF.PO_CURRENCY_CODE
      ,PDF.PO_LINE_NUMBER
      ,PDF.PO_UNIT_PRICE
      ,PDF.PO_QUANTITY
      ,PDF.PO_LINE_AMOUNT
      ,PDF.PO_RECOVERABLE_TAX_AMOUNT
      ,PDF.NEED_BY_DATE
      ,PDF.ITEM_DESCRIPTION
      ,PDF.PAYMENT_TERM
      ,PDF.UNIT_OF_MEASUREMENT
      ,PDF.SUPPLIER_NAME
      ,PDF.SUPPLIER_NUMBER
      ,PDF.SUPPLIER_ADDRESS
      ,PDF.SUPPLIER_CONTACT_NAME
      ,PDF.SUPPLIER_EMAIL_ADDRESS
      ,PDF.PROCUREMENT_BU
      ,PDF.BUYER_NAME
      ,PDF.BUYER_EMAIL
      ,PDF.PR_NUMBER
      ,PDF.PR_REQUESTER_NAME
FROM   PO_DETAILED_FINAL PDF
ORDER BY PDF.PO_NUMBER
        ,PDF.PO_LINE_NUMBER
```

**Key Patterns:**
- Join PO to PR via: `PO_DISTRIBUTIONS_ALL.REQ_DISTRIBUTION_ID` = `POR_REQ_DISTRIBUTIONS_ALL.DISTRIBUTION_ID`
- Buyer via: `PO_HEADERS_ALL.AGENT_ID` (NOT `BUYER_ID`)
- Payment Terms: `AP_TERMS.NAME` (NOT `AP_TERMS_NAME.TERM_NAME`)
- Tax: Sum from `PO_DISTRIBUTIONS_ALL.RECOVERABLE_TAX` (NOT from `PO_LINE_LOCATIONS_ALL`)
- PR Number: `POR_REQUISITION_HEADERS_ALL.REQUISITION_NUMBER` (NOT `SEGMENT1`)
