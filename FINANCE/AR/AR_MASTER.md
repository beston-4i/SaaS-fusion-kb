# AR Master Instructions: Oracle Fusion Receivables

**Module:** Accounts Receivable (AR)
**Tag:** `#Finance #AR #Receivables`
**Status:** Active

---

## 1. 🚨 Critical AR Constraints
*Violating these rules results in incorrect financial data.*

1.  **Currency & Exchange Rates (CRITICAL):**
    *   **Problem:** Functional currency transactions often have `NULL` exchange rates.
    *   **Rule:** `NVL(EXCHANGE_RATE, 1)`
    *   **Formula:** `Functional_Amt = Entered_Amt * NVL(Rate, 1)`
    
2.  **Transaction Sign Reversal:**
    *   **Problem:** In the DB, Credit Memos and Payments are stored as positive numbers, but logically they reduce balance.
    *   **Rule:** If `CLASS IN ('CM', 'PMT')` then `Multiply by -1`.
    *   **Application:** Always apply this when summing `AMOUNT_DUE_ORIGINAL` or `AMOUNT_DUE_REMAINING`.

3.  **Reverse Receipts:**
    *   **Rule:** Exclude receipts where `STATUS = 'REVERSED'` in History.
    *   **Pattern:** `AND NOT EXISTS (SELECT 1 FROM AR_CASH_RECEIPT_HISTORY_ALL WHERE STATUS='REVERSED'...)`

4.  **XLA Application ID:**
    *   **Rule:** `AND APPLICATION_ID = 222` (Receivables)

5.  **Transaction Type Join:**
    *   **CRITICAL:** Use `CUST_TRX_TYPE_SEQ_ID` NOT `CUST_TRX_TYPE_ID`
    *   **Correct:** `RCTA.CUST_TRX_TYPE_SEQ_ID = RCTT.CUST_TRX_TYPE_SEQ_ID`
    *   **Wrong:** `RCTA.CUST_TRX_TYPE_ID = RCTT.CUST_TRX_TYPE_ID` (This will give incorrect results)

6.  **Customer Site Use:**
    *   **Rule:** Filter by `SITE_USE_CODE = 'BILL_TO'` for billing address
    *   **Pattern:** `AND HCSU.SITE_USE_CODE = 'BILL_TO'`

7.  **Receipt Status:**
    *   **Valid Statuses:** 'APP' (Applied), 'ACC' (Accepted), 'CLEARED' (Cleared)
    *   **Check Reversed:** Use `NOT EXISTS` check against `AR_CASH_RECEIPT_HISTORY_ALL` with `STATUS = 'REVERSED'`

8.  **Transaction Completion:**
    *   **Rule:** `AND COMPLETE_FLAG = 'Y'` for posted transactions only
    *   **Application:** Filter on `RA_CUSTOMER_TRX_ALL.COMPLETE_FLAG = 'Y'`

---

## 2. ⚡ Performance Optimization

| Object | Optimal Access Path | Hint Syntax |
|--------|---------------------|-------------|
| **Payment Schedule** | CUSTOMER_TRX_ID | `/*+ INDEX(APS AR_PAYMENT_SCHEDULES_N1) */` |
| **Transaction** | TRX_NUMBER + ORG | `/*+ INDEX(RCTA RA_CUSTOMER_TRX_U1) */` |
| **Receipts** | RECEIPT_NUMBER | `/*+ INDEX(ACRA AR_CASH_RECEIPTS_U2) */` |
| **Applications** | RECEIVABLE_APP_ID | `/*+ INDEX(ARA AR_RECEIVABLE_APPLICATIONS_U1) */` |

---

## 3. 🗺️ Schema Map (Key Tables)

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **RCTA** | `RA_CUSTOMER_TRX_ALL` | Transaction Header (Invoice/Memo) |
| **RCTL** | `RA_CUSTOMER_TRX_LINES_ALL` | Transaction Lines |
| **RCTLA** | `RA_CUSTOMER_TRX_LINE_GL_DIST_ALL` | Line-level GL distributions |
| **RCTT** | `RA_CUST_TRX_TYPES_ALL` | Transaction Type definitions (Invoice, CM, DM) |
| **PSA** | `AR_PAYMENT_SCHEDULES_ALL` | Balances, Due Dates, Aging Source |
| **ACRA** | `AR_CASH_RECEIPTS_ALL` | Receipt Header |
| **ACRH** | `AR_CASH_RECEIPT_HISTORY_ALL` | Receipt History & Status Tracking |
| **ARA** | `AR_RECEIVABLE_APPLICATIONS_ALL` | Links Receipt to Invoice |
| **ARM** | `AR_RECEIPT_METHODS` | Receipt Method definitions |
| **ARAA** | `AR_ADJUSTMENTS_ALL` | Invoice adjustments |
| **HCA** | `HZ_CUST_ACCOUNTS` | Customer Account Master |
| **HP** | `HZ_PARTIES` | Party Master (Name) |
| **HCAS** | `HZ_CUST_ACCT_SITES_ALL` | Customer Account Sites |
| **HCSU** | `HZ_CUST_SITE_USES_ALL` | Customer Site Uses (Bill-To, Ship-To) |
| **HPS** | `HZ_PARTY_SITES` | Party Site information |
| **HL** | `HZ_LOCATIONS` | Physical Location details |
| **RBS** | `RA_BATCH_SOURCES_ALL` | Batch Source definitions |
| **FBU** | `FUN_ALL_BUSINESS_UNITS_V` | Business Unit Master |
| **XTE** | `XLA_TRANSACTION_ENTITIES` | Subledger Accounting Entities |
| **XE** | `XLA_EVENTS` | Subledger Accounting Events |
| **XAH** | `XLA_AE_HEADERS` | Accounting Entry Headers |
| **XAL** | `XLA_AE_LINES` | Accounting Entry Lines |
| **XDL** | `XLA_DISTRIBUTION_LINKS` | Links between source and SLA |

---
