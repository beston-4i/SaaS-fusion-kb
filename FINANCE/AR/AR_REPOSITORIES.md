# AR Repository Patterns

**Purpose:** Standardized CTEs for extracting AR data.
**Critical Rule:** Copy-paste these blocks exactly. Do NOT attempt to rewrite the Gain/Loss logic.

---

## 1. Transaction Master (Heavily Used)
*Retrieves Invoices, Credit Memos, Debit Memos with correct Sign Logic.*
> [!WARNING]
> Always use `CUST_TRX_TYPE_SEQ_ID` when joining `RA_CUSTOMER_TRX_ALL` to `RA_CUST_TRX_TYPES_ALL`. Never use `CUST_TRX_TYPE_ID`.


```sql
AR_TRX_MASTER AS (
    SELECT /*+ qb_name(AR_TRX) MATERIALIZE */
           RCTA.CUSTOMER_TRX_ID
          ,RCTA.TRX_NUMBER
          ,RCTA.TRX_DATE
          ,RCTA.ORG_ID
           -- Currency Logic
          ,RCTA.INVOICE_CURRENCY_CODE
          ,NVL(RCTA.EXCHANGE_RATE, 1) AS EXCH_RATE
           -- Sign Logic
          ,CASE WHEN PSA.CLASS IN ('CM', 'PMT') THEN -1 ELSE 1 END AS SIGN_FACTOR
          ,PSA.AMOUNT_DUE_ORIGINAL
          ,PSA.AMOUNT_DUE_REMAINING
    FROM   RA_CUSTOMER_TRX_ALL RCTA
          ,AR_PAYMENT_SCHEDULES_ALL PSA
    WHERE  RCTA.CUSTOMER_TRX_ID = PSA.CUSTOMER_TRX_ID
      AND  RCTA.COMPLETE_FLAG = 'Y'
      AND  RCTA.ORG_ID = PSA.ORG_ID
)
```

---

## 2. Customer Master
*Retrieves Party and Account details.*

```sql
AR_CUST_MASTER AS (
    SELECT /*+ qb_name(CUST) MATERIALIZE */
           HCA.CUST_ACCOUNT_ID
          ,HCA.ACCOUNT_NUMBER
          ,HP.PARTY_NAME AS CUSTOMER_NAME
          ,HP.PARTY_ID
    FROM   HZ_CUST_ACCOUNTS HCA
          ,HZ_PARTIES HP
    WHERE  HCA.PARTY_ID = HP.PARTY_ID
      AND  HCA.STATUS = 'A'
)
```

---

## 3. Receipt Master
*Retrieves valid, non-reversed receipts.*

```sql
AR_RCPT_MASTER AS (
    SELECT /*+ qb_name(RCPT) MATERIALIZE */
           ACRA.CASH_RECEIPT_ID
          ,ACRA.RECEIPT_NUMBER
          ,ACRA.RECEIPT_DATE
          ,ACRA.AMOUNT
    FROM   AR_CASH_RECEIPTS_ALL ACRA
    WHERE  ACRA.ORG_ID IN (:P_ORG_ID)
      AND  NOT EXISTS (
           SELECT 1 FROM AR_CASH_RECEIPT_HISTORY_ALL H 
           WHERE H.CASH_RECEIPT_ID = ACRA.CASH_RECEIPT_ID 
           AND H.STATUS = 'REVERSED'
      )
)
```

---

## 4. Receipt Applications (Complex Logic)
*Handles Exchange Gain/Loss, Refunds, and Standard Applications.*

```sql
AR_APP_MASTER AS (
    SELECT /*+ qb_name(AR_APP) MATERIALIZE */
           APP.CASH_RECEIPT_ID
          ,APP.APPLIED_CUSTOMER_TRX_ID
          ,APP.AMOUNT_APPLIED
          ,APP.GL_DATE
          ,APP.ORG_ID
           -- Exchange Gain/Loss Logic
          ,CASE 
             WHEN APP.ACCTD_AMOUNT_APPLIED_FROM != APP.ACCTD_AMOUNT_APPLIED_TO THEN 'Y' 
             ELSE 'N' 
           END AS HAS_GAIN_LOSS
           -- Refund Logic
          ,CASE 
             WHEN APP.STATUS = 'ACTIVITY' AND APP.CASH_RECEIPT_ID IS NULL THEN 'REFUND'
             ELSE APP.STATUS 
           END AS APP_TYPE
    FROM   AR_RECEIVABLE_APPLICATIONS_ALL APP
    WHERE  APP.STATUS IN ('APP', 'ACTIVITY')
      AND  NVL(APP.DISPLAY, 'Y') = 'Y'
)
```

---

## 5. Exchange Gain/Loss (Detailed)
*Specific extraction for Gain/Loss Reconciliation.*

```sql
AR_EXCH_GAIN_LOSS AS (
    SELECT /*+ qb_name(AR_EXCH) */
           RCTA.TRX_NUMBER
          ,RCTA.INVOICE_CURRENCY_CODE
          ,APP.AMOUNT_APPLIED
          ,DECODE(SIGN(APP.ACCTD_AMOUNT_APPLIED_FROM - APP.ACCTD_AMOUNT_APPLIED_TO), 
                  1, 'Exchange Gain', 'Exchange Loss') AS TYPE
          ,ABS(APP.ACCTD_AMOUNT_APPLIED_FROM - APP.ACCTD_AMOUNT_APPLIED_TO) AS GL_IMPACT
    FROM   AR_RECEIVABLE_APPLICATIONS_ALL APP
          ,RA_CUSTOMER_TRX_ALL RCTA
    WHERE  APP.APPLIED_CUSTOMER_TRX_ID = RCTA.CUSTOMER_TRX_ID
      AND  APP.ACCTD_AMOUNT_APPLIED_FROM <> APP.ACCTD_AMOUNT_APPLIED_TO
)
```

---

## 6. Transaction Master (Complete with Sites and Types)
*Enhanced transaction master with customer site details and transaction types.*

```sql
AR_TRX_MASTER_FULL AS (
    SELECT /*+ qb_name(AR_TRX_FULL) MATERIALIZE */
           RCTA.CUSTOMER_TRX_ID
          ,RCTA.TRX_NUMBER
          ,RCTA.TRX_DATE
          ,RCTA.GL_DATE
          ,RCTA.ORG_ID
          ,FBU.BU_NAME
          -- Transaction Type Details
          ,RCTT.NAME TRX_TYPE_NAME
          ,RCTT.TYPE TRX_CLASS
          ,RBS.NAME BATCH_SOURCE_NAME
          -- Customer Details
          ,HCA.ACCOUNT_NUMBER
          ,HP.PARTY_NAME CUSTOMER_NAME
          -- Customer Site Details
          ,HCSU.LOCATION BILL_TO_LOCATION
          ,HL.ADDRESS1
          ,HL.CITY
          ,HL.STATE
          ,HL.POSTAL_CODE
          ,HL.COUNTRY
          -- Currency Logic
          ,RCTA.INVOICE_CURRENCY_CODE
          ,NVL(RCTA.EXCHANGE_RATE, 1) EXCH_RATE
          -- Sign Logic for CM/DM/PMT
          ,CASE WHEN PSA.CLASS IN ('CM', 'PMT') THEN -1 ELSE 1 END SIGN_FACTOR
          -- Amounts
          ,PSA.AMOUNT_DUE_ORIGINAL
          ,PSA.AMOUNT_DUE_REMAINING
          ,PSA.AMOUNT_DUE_ORIGINAL * NVL(RCTA.EXCHANGE_RATE, 1) FUNC_AMOUNT_ORIGINAL
          ,PSA.AMOUNT_DUE_REMAINING * NVL(RCTA.EXCHANGE_RATE, 1) FUNC_AMOUNT_REMAINING
          -- Due Date & Aging
          ,PSA.DUE_DATE
          ,PSA.CLASS
          ,PSA.STATUS
          -- Audit Fields
          ,PPNF.DISPLAY_NAME CREATED_BY_NAME
          ,RCTA.CREATION_DATE
    FROM   RA_CUSTOMER_TRX_ALL RCTA
          ,AR_PAYMENT_SCHEDULES_ALL PSA
          ,RA_CUST_TRX_TYPES_ALL RCTT
          ,RA_BATCH_SOURCES_ALL RBS
          ,HZ_CUST_ACCOUNTS HCA
          ,HZ_PARTIES HP
          ,HZ_CUST_SITE_USES_ALL HCSU
          ,HZ_CUST_ACCT_SITES_ALL HCAS
          ,HZ_PARTY_SITES HPS
          ,HZ_LOCATIONS HL
          ,FUN_ALL_BUSINESS_UNITS_V FBU
          ,PER_USERS PU
          ,PER_PERSON_NAMES_F PPNF
    WHERE  RCTA.CUSTOMER_TRX_ID = PSA.CUSTOMER_TRX_ID
      AND  RCTA.CUST_TRX_TYPE_SEQ_ID = RCTT.CUST_TRX_TYPE_SEQ_ID
      AND  RCTA.BATCH_SOURCE_SEQ_ID = RBS.BATCH_SOURCE_SEQ_ID(+)
      AND  RCTA.BILL_TO_CUSTOMER_ID = HCA.CUST_ACCOUNT_ID
      AND  HCA.PARTY_ID = HP.PARTY_ID
      AND  RCTA.BILL_TO_SITE_USE_ID = HCSU.SITE_USE_ID(+)
      AND  HCSU.CUST_ACCT_SITE_ID = HCAS.CUST_ACCT_SITE_ID(+)
      AND  HCAS.PARTY_SITE_ID = HPS.PARTY_SITE_ID(+)
      AND  HPS.LOCATION_ID = HL.LOCATION_ID(+)
      AND  RCTA.ORG_ID = FBU.BU_ID
      AND  RCTA.ORG_ID = PSA.ORG_ID
      AND  RCTA.COMPLETE_FLAG = 'Y'
      AND  RCTA.CREATED_BY = PU.USER_ID(+)
      AND  PU.PERSON_ID = PPNF.PERSON_ID(+)
      AND  PPNF.NAME_TYPE(+) = 'GLOBAL'
      AND  TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE(+)) 
                              AND TRUNC(PPNF.EFFECTIVE_END_DATE(+))
)
```

---

## 7. Receipt Master (Complete with Bank Details)
*Enhanced receipt master with bank account, deposit, and application details.*

```sql
AR_RCPT_MASTER_FULL AS (
    SELECT /*+ qb_name(RCPT_FULL) MATERIALIZE */
           ACRA.CASH_RECEIPT_ID
          ,ACRA.RECEIPT_NUMBER
          ,ACRA.RECEIPT_DATE
          ,ACRA.GL_DATE
          ,ACRA.AMOUNT
          ,ACRA.CURRENCY_CODE
          ,NVL(ACRA.EXCHANGE_RATE, 1) EXCH_RATE
          ,ACRA.AMOUNT * NVL(ACRA.EXCHANGE_RATE, 1) FUNC_AMOUNT
          -- Customer Details
          ,HP.PARTY_NAME CUSTOMER_NAME
          ,HCA.ACCOUNT_NUMBER
          -- Receipt Method
          ,ARM.NAME RECEIPT_METHOD
          -- Bank Details (if available)
          ,ACRA.CUSTOMER_BANK_ACCOUNT_ID
          -- Deposit Information
          ,ACRA.DEPOSIT_DATE
          -- Status Details
          ,ACRH.STATUS RECEIPT_STATUS
          ,ACRH.STATUS_TRX_NUMBER
          -- Business Unit
          ,FBU.BU_NAME
          ,ACRA.ORG_ID
          -- Audit Fields
          ,PPNF.DISPLAY_NAME CREATED_BY_NAME
          ,ACRA.CREATION_DATE
    FROM   AR_CASH_RECEIPTS_ALL ACRA
          ,HZ_CUST_ACCOUNTS HCA
          ,HZ_PARTIES HP
          ,AR_RECEIPT_METHODS ARM
          ,FUN_ALL_BUSINESS_UNITS_V FBU
          ,AR_CASH_RECEIPT_HISTORY_ALL ACRH
          ,PER_USERS PU
          ,PER_PERSON_NAMES_F PPNF
    WHERE  ACRA.PAY_FROM_CUSTOMER = HCA.CUST_ACCOUNT_ID(+)
      AND  HCA.PARTY_ID = HP.PARTY_ID(+)
      AND  ACRA.RECEIPT_METHOD_ID = ARM.RECEIPT_METHOD_ID(+)
      AND  ACRA.ORG_ID = FBU.BU_ID
      AND  ACRA.CASH_RECEIPT_ID = ACRH.CASH_RECEIPT_ID
      AND  ACRH.CURRENT_RECORD_FLAG = 'Y'
      AND  ACRA.CREATED_BY = PU.USER_ID(+)
      AND  PU.PERSON_ID = PPNF.PERSON_ID(+)
      AND  PPNF.NAME_TYPE(+) = 'GLOBAL'
      AND  TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE(+)) 
                              AND TRUNC(PPNF.EFFECTIVE_END_DATE(+))
      -- Exclude Reversed Receipts
      AND  NOT EXISTS (
           SELECT 1 
           FROM   AR_CASH_RECEIPT_HISTORY_ALL H
           WHERE  H.CASH_RECEIPT_ID = ACRA.CASH_RECEIPT_ID
             AND  H.STATUS = 'REVERSED'
      )
)
```

---

## 8. Aging Calculation (Multi-Bucket)
*Flexible aging calculation with configurable buckets.*

```sql
AR_AGING_CALC AS (
    SELECT /*+ qb_name(AR_AGING) */
           PSA.CUSTOMER_TRX_ID
          ,PSA.PAYMENT_SCHEDULE_ID
          ,PSA.DUE_DATE
          ,PSA.AMOUNT_DUE_REMAINING * NVL(RCTA.EXCHANGE_RATE, 1) FUNC_REMAINING
          ,TRUNC(:P_AS_OF_DATE) - TRUNC(PSA.DUE_DATE) DAYS_OVERDUE
          -- Bucket Classification
          ,CASE 
             WHEN (TRUNC(:P_AS_OF_DATE) - TRUNC(PSA.DUE_DATE)) < 0 THEN 'Not Yet Due'
             WHEN (TRUNC(:P_AS_OF_DATE) - TRUNC(PSA.DUE_DATE)) BETWEEN 0 AND 30 THEN 'Bucket 0-30'
             WHEN (TRUNC(:P_AS_OF_DATE) - TRUNC(PSA.DUE_DATE)) BETWEEN 31 AND 60 THEN 'Bucket 31-60'
             WHEN (TRUNC(:P_AS_OF_DATE) - TRUNC(PSA.DUE_DATE)) BETWEEN 61 AND 90 THEN 'Bucket 61-90'
             WHEN (TRUNC(:P_AS_OF_DATE) - TRUNC(PSA.DUE_DATE)) BETWEEN 91 AND 120 THEN 'Bucket 91-120'
             WHEN (TRUNC(:P_AS_OF_DATE) - TRUNC(PSA.DUE_DATE)) BETWEEN 121 AND 180 THEN 'Bucket 121-180'
             WHEN (TRUNC(:P_AS_OF_DATE) - TRUNC(PSA.DUE_DATE)) BETWEEN 181 AND 360 THEN 'Bucket 181-360'
             WHEN (TRUNC(:P_AS_OF_DATE) - TRUNC(PSA.DUE_DATE)) > 360 THEN 'Above 360'
             ELSE 'Unknown'
           END AGING_BUCKET
    FROM   AR_PAYMENT_SCHEDULES_ALL PSA
          ,RA_CUSTOMER_TRX_ALL RCTA
    WHERE  PSA.CUSTOMER_TRX_ID = RCTA.CUSTOMER_TRX_ID
      AND  PSA.STATUS = 'OP'
      AND  PSA.AMOUNT_DUE_REMAINING <> 0
      AND  RCTA.COMPLETE_FLAG = 'Y'
      AND  TRUNC(RCTA.TRX_DATE) <= :P_AS_OF_DATE
)
```

---

## 9. AR Transaction Union (Ledger Pattern)
*Union of all AR transaction types for ledger reports.*

```sql
AR_TRX_UNION AS (
    -- Invoices and Credit Memos
    SELECT 1 SORT_ORDER
          ,RCTA.CUSTOMER_TRX_ID TRX_ID
          ,RCTA.TRX_NUMBER
          ,RCTA.TRX_DATE TRX_DATE
          ,RCTA.GL_DATE
          ,RCTT.NAME TRX_TYPE
          ,RCTA.COMMENTS TRX_DESC
          ,RCTA.INVOICE_CURRENCY_CODE
          -- Debit/Credit Logic
          ,CASE WHEN PSA.CLASS = 'CM' THEN NULL 
                ELSE PSA.AMOUNT_DUE_ORIGINAL END AMOUNT_DR
          ,CASE WHEN PSA.CLASS = 'CM' THEN PSA.AMOUNT_DUE_ORIGINAL 
                ELSE NULL END AMOUNT_CR
          ,CASE WHEN PSA.CLASS = 'CM' THEN NULL 
                ELSE PSA.AMOUNT_DUE_ORIGINAL * NVL(RCTA.EXCHANGE_RATE, 1) END FUNC_AMOUNT_DR
          ,CASE WHEN PSA.CLASS = 'CM' THEN PSA.AMOUNT_DUE_ORIGINAL * NVL(RCTA.EXCHANGE_RATE, 1) 
                ELSE NULL END FUNC_AMOUNT_CR
          ,RCTA.BILL_TO_CUSTOMER_ID CUSTOMER_ID
          ,RCTA.BILL_TO_SITE_USE_ID SITE_USE_ID
          ,RCTA.ORG_ID
    FROM   RA_CUSTOMER_TRX_ALL RCTA
          ,AR_PAYMENT_SCHEDULES_ALL PSA
          ,RA_CUST_TRX_TYPES_ALL RCTT
    WHERE  RCTA.CUSTOMER_TRX_ID = PSA.CUSTOMER_TRX_ID
      AND  RCTA.CUST_TRX_TYPE_SEQ_ID = RCTT.CUST_TRX_TYPE_SEQ_ID
      AND  RCTA.COMPLETE_FLAG = 'Y'
      AND  RCTA.ORG_ID = PSA.ORG_ID
      AND  TRUNC(RCTA.GL_DATE) BETWEEN :P_FROM_DATE AND :P_TO_DATE
    
    UNION ALL
    
    -- Receipts (Applications)
    SELECT 2 SORT_ORDER
          ,ACRA.CASH_RECEIPT_ID
          ,ACRA.RECEIPT_NUMBER
          ,ACRA.RECEIPT_DATE
          ,ARA.GL_DATE
          ,'Receipt' TRX_TYPE
          ,ACRA.COMMENTS
          ,ACRA.CURRENCY_CODE
          ,NULL AMOUNT_DR
          ,ARA.AMOUNT_APPLIED AMOUNT_CR
          ,NULL FUNC_AMOUNT_DR
          ,ARA.AMOUNT_APPLIED * NVL(ACRA.EXCHANGE_RATE, 1) FUNC_AMOUNT_CR
          ,RCTA.BILL_TO_CUSTOMER_ID
          ,RCTA.BILL_TO_SITE_USE_ID
          ,ACRA.ORG_ID
    FROM   AR_CASH_RECEIPTS_ALL ACRA
          ,AR_RECEIVABLE_APPLICATIONS_ALL ARA
          ,RA_CUSTOMER_TRX_ALL RCTA
    WHERE  ACRA.CASH_RECEIPT_ID = ARA.CASH_RECEIPT_ID
      AND  ARA.APPLIED_CUSTOMER_TRX_ID = RCTA.CUSTOMER_TRX_ID(+)
      AND  ARA.STATUS = 'APP'
      AND  NVL(ARA.DISPLAY, 'Y') = 'Y'
      AND  TRUNC(ARA.GL_DATE) BETWEEN :P_FROM_DATE AND :P_TO_DATE
      -- Exclude Reversed Receipts
      AND  NOT EXISTS (
           SELECT 1 
           FROM   AR_CASH_RECEIPT_HISTORY_ALL H
           WHERE  H.CASH_RECEIPT_ID = ACRA.CASH_RECEIPT_ID
             AND  H.STATUS = 'REVERSED'
      )
    
    UNION ALL
    
    -- Adjustments
    SELECT 3 SORT_ORDER
          ,ARAA.ADJUSTMENT_ID
          ,TO_CHAR(ARAA.ADJUSTMENT_NUMBER)
          ,ARAA.APPLY_DATE
          ,ARAA.GL_DATE
          ,'Adjustment' TRX_TYPE
          ,ARAA.COMMENTS
          ,RCTA.INVOICE_CURRENCY_CODE
          ,CASE WHEN SIGN(ARAA.AMOUNT) = 1 THEN ARAA.AMOUNT ELSE NULL END
          ,CASE WHEN SIGN(ARAA.AMOUNT) = -1 THEN ABS(ARAA.AMOUNT) ELSE NULL END
          ,CASE WHEN SIGN(ARAA.AMOUNT) = 1 THEN ARAA.AMOUNT * NVL(RCTA.EXCHANGE_RATE, 1) ELSE NULL END
          ,CASE WHEN SIGN(ARAA.AMOUNT) = -1 THEN ABS(ARAA.AMOUNT) * NVL(RCTA.EXCHANGE_RATE, 1) ELSE NULL END
          ,RCTA.BILL_TO_CUSTOMER_ID
          ,RCTA.BILL_TO_SITE_USE_ID
          ,ARAA.ORG_ID
    FROM   AR_ADJUSTMENTS_ALL ARAA
          ,RA_CUSTOMER_TRX_ALL RCTA
    WHERE  ARAA.CUSTOMER_TRX_ID = RCTA.CUSTOMER_TRX_ID
      AND  TRUNC(ARAA.GL_DATE) BETWEEN :P_FROM_DATE AND :P_TO_DATE
)
```

---

## 10. SLA Distribution Details
*XLA Subledger Accounting distribution extraction for AR.*

```sql
AR_SLA_DIST AS (
    SELECT /*+ qb_name(AR_SLA) */
           XTE.SOURCE_ID_INT_1 CUSTOMER_TRX_ID
          ,RCTA.TRX_NUMBER
          ,XE.EVENT_TYPE_CODE
          ,XAH.ACCOUNTING_DATE
          ,XAH.PERIOD_NAME
          ,XAL.ACCOUNTING_CLASS_CODE
          ,GCC.CONCATENATED_SEGMENTS GL_ACCOUNT
          ,XAL.ENTERED_DR
          ,XAL.ENTERED_CR
          ,XAL.ACCOUNTED_DR
          ,XAL.ACCOUNTED_CR
          ,XAL.DESCRIPTION
    FROM   XLA_TRANSACTION_ENTITIES XTE
          ,XLA_EVENTS XE
          ,XLA_AE_HEADERS XAH
          ,XLA_AE_LINES XAL
          ,GL_CODE_COMBINATIONS_KFV GCC
          ,RA_CUSTOMER_TRX_ALL RCTA
    WHERE  XTE.APPLICATION_ID = 222
      AND  XTE.ENTITY_CODE = 'TRANSACTIONS'
      AND  XTE.APPLICATION_ID = XE.APPLICATION_ID
      AND  XTE.ENTITY_ID = XE.ENTITY_ID
      AND  XE.APPLICATION_ID = XAH.APPLICATION_ID
      AND  XE.EVENT_ID = XAH.EVENT_ID
      AND  XAH.AE_HEADER_ID = XAL.AE_HEADER_ID
      AND  XAL.CODE_COMBINATION_ID = GCC.CODE_COMBINATION_ID
      AND  XTE.SOURCE_ID_INT_1 = RCTA.CUSTOMER_TRX_ID(+)
      AND  XAH.ACCOUNTING_DATE BETWEEN :P_FROM_DATE AND :P_TO_DATE
)
```
