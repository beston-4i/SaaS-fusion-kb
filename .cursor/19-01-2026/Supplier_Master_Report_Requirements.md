# Supplier Master Report - Report Requirements

**Module:** SCM - Supplier Management  
**Report Name:** Supplier Master Report  
**Report Purpose:** Comprehensive supplier master data report capturing supplier details, sites, addresses, bank accounts, tax information, and business classifications  
**Created:** 19-01-2026

---

## Input Parameters

- **P_BUSINESS_UNIT_NAME** (VARCHAR2, Optional) - Business Unit Name filter (uses NVL for optional parameter)
- **P_SUPPLIER_STATUS** (VARCHAR2, Optional) - Supplier Status filter ('Active' or 'Inactive', uses NVL for optional parameter)
- **P_SUPPLIER_TYPE** (VARCHAR2, Optional) - Supplier Type/Vendor Type filter (uses NVL for optional parameter)
- **P_SUPPLIER_CREATION_DATE_FROM** (DATE, Optional) - Supplier creation date from filter
- **P_SUPPLIER_CREATION_DATE_TO** (DATE, Optional) - Supplier creation date to filter

---

## Output Columns (22 Total)

1. **Supplier Name** - `POZ_SUPPLIERS_V.VENDOR_NAME`
2. **Supplier Number** - `POZ_SUPPLIERS_V.SEGMENT1`
3. **Vendor Type** - `FND_LOOKUP_VALUES_TL.MEANING` (from `POZ_SUPPLIERS_V.VENDOR_TYPE_LOOKUP_CODE` via lookup type `'ORA_POZ_VENDOR_TYPE'`)
4. **Supplier Category** - **CRITICAL:** May require validation - check if available in `POZ_SUPPLIERS_V` DFF attributes or supplier category assignment tables
5. **Supplier Creation Date** - `POZ_SUPPLIERS_V.CREATION_DATE` (formatted as DD-MON-YYYY)
6. **Supplier Site Name** - `POZ_SUPPLIER_SITES_V.PARTY_SITE_NAME`
7. **Address Name** - `POZ_SUPPLIER_SITES_V.PARTY_SITE_NAME` (same as site name, or use address name if available)
8. **Address Line 1** - `POZ_SUPPLIER_ADDRESS_V.ADDRESS1`
9. **Address Line 2** - `POZ_SUPPLIER_ADDRESS_V.ADDRESS2`
10. **City** - `POZ_SUPPLIER_ADDRESS_V.CITY`
11. **Phone Number** - `POZ_SUPPLIER_ADDRESS_V.PHONE_NUMBER`
12. **Vendor Status** - `DECODE(HZ_PARTIES.STATUS, 'A', 'Active', 'I', 'Inactive', 'Inactive')`
13. **Bank Account Number** - `IBY_EXT_BANK_ACCOUNTS.BANK_ACCOUNT_NUM` (via complex IBY join, primary account only)
14. **Bank Account Name** - `IBY_EXT_BANK_ACCOUNTS.BANK_ACCOUNT_NAME`
15. **Bank Name** - `CE_BANKS_V.BANK_NAME`
16. **Bank Number** - `CE_BANKS_V.BANK_NUMBER`
17. **Business Unit Name** - `FUN_ALL_BUSINESS_UNITS_V.BU_NAME` (from `POZ_SUPPLIER_SITES_V.PRC_BU_ID`)
18. **Tax Registration Number** - `ZX_PARTY_TAX_PROFILE.REP_REGISTRATION_NUMBER`
19. **Tax Classification** - `ZX_PARTY_TAX_PROFILE.TAX_CLASSIFICATION_CODE`
20. **Payment Term** - `AP_TERMS_TL.NAME` (from `POZ_SUPPLIER_SITES_V.TERMS_ID`, translated table)
21. **Trade License Number** - `POZ_BUSINESS_CLASSIFICATIONS_V.CERTIFICATE_NUMBER` (filtered by trade license pattern match)

---

## Business Rules

### Data Filters
- **Supplier Creation Date Range:** `TRUNC(POZ_SUPPLIERS_V.CREATION_DATE) BETWEEN TRUNC(:P_SUPPLIER_CREATION_DATE_FROM) AND TRUNC(:P_SUPPLIER_CREATION_DATE_TO)` (if both parameters provided)
- **Business Unit Name Filter:** `NVL(:P_BUSINESS_UNIT_NAME, BU_NAME) = BU_NAME`
  - **CRITICAL:** Uses NVL pattern to make parameter optional - if parameter is NULL, all business units are included; if provided, filters by business unit name
- **Supplier Status Filter:** `NVL(:P_SUPPLIER_STATUS, VENDOR_STATUS) = VENDOR_STATUS`
  - **CRITICAL:** Uses NVL pattern to make parameter optional - if parameter is NULL, all statuses are included; if provided, filters by supplier status ('Active' or 'Inactive')
- **Supplier Type Filter:** `NVL(:P_SUPPLIER_TYPE, VENDOR_TYPE) = VENDOR_TYPE`
  - **CRITICAL:** Uses NVL pattern to make parameter optional - if parameter is NULL, all types are included; if provided, filters by vendor type
- **Bank Account Filter:** `IBY_ACCOUNT_OWNERS.PRIMARY_FLAG = 'Y'` (only primary bank account per supplier)
- **Business Classifications Filter:** `POZ_BUSINESS_CLASSIFICATIONS_V.STATUS = 'A'` (only active classifications)

### Row Structure
- **One row per Supplier Site** - Report shows site-level detail (one supplier can have multiple sites)
- **Left Outer Joins for Optional Data:** 
  - Supplier may not have sites (if no sites created)
  - Site may not have address (if address not entered)
  - Supplier may not have bank account (if not configured)
  - Supplier may not have tax profile (if not configured)
  - Supplier may not have payment terms (if not assigned to site)
  - Supplier may not have trade license (if not configured)

### Critical Calculations

#### Vendor Status (CRITICAL)
- **Source:** `HZ_PARTIES.STATUS` (NOT `POZ_SUPPLIERS_V.ENABLED_FLAG`)
- **Join:** `POZ_SUPPLIERS_V.PARTY_ID = HZ_PARTIES.PARTY_ID`
- **Decode:** `DECODE(HP.STATUS, 'A', 'Active', 'I', 'Inactive', 'Inactive')`
- **Why:** Party-level status provides more accurate vendor status than supplier-level enabled flag

#### Vendor Type Lookup (CRITICAL)
- **Source:** `FND_LOOKUP_VALUES_TL.MEANING`
- **Lookup Type:** `'ORA_POZ_VENDOR_TYPE'` (NOT `'VENDOR_TYPE'`)
- **Filters:** `VIEW_APPLICATION_ID = 0`, `SET_ID = 0`, `LANGUAGE = USERENV('LANG')`
- **Join:** `POZ_SUPPLIERS_V.VENDOR_TYPE_LOOKUP_CODE = FND_LOOKUP_VALUES_TL.LOOKUP_CODE`
- **Why:** Supplier-specific vendor type lookup uses different lookup type than standard vendor type

#### Supplier Bank Account Join Pattern (CRITICAL)
- **Rule:** Join `IBY_EXT_BANK_ACCOUNTS` via `IBY_ACCOUNT_OWNERS`, `IBY_EXTERNAL_PAYEES_ALL`, and `IBY_PMT_INSTR_USES_ALL`
- **Path:** 
  1. `IBY_EXT_BANK_ACCOUNTS.EXT_BANK_ACCOUNT_ID = IBY_ACCOUNT_OWNERS.EXT_BANK_ACCOUNT_ID`
  2. `IBY_ACCOUNT_OWNERS.ACCOUNT_OWNER_PARTY_ID = IBY_EXTERNAL_PAYEES_ALL.PAYEE_PARTY_ID`
  3. `IBY_ACCOUNT_OWNERS.EXT_BANK_ACCOUNT_ID = IBY_PMT_INSTR_USES_ALL.INSTRUMENT_ID`
  4. `IBY_EXTERNAL_PAYEES_ALL.EXT_PAYEE_ID = IBY_PMT_INSTR_USES_ALL.EXT_PMT_PARTY_ID`
  5. `IBY_EXTERNAL_PAYEES_ALL.PAYEE_PARTY_ID = POZ_SUPPLIERS_V.PARTY_ID` (via party link)
- **Filter:** `IBY_ACCOUNT_OWNERS.PRIMARY_FLAG = 'Y'` for primary bank account
- **Bank Details:** Join `IBY_EXT_BANK_ACCOUNTS.BANK_ID = CE_BANKS_V.BANK_PARTY_ID` for bank name and number
- **Why:** Supplier bank accounts require complex join through multiple IBY tables to link to supplier party

#### Trade License Number (CRITICAL)
- **Source:** `POZ_BUSINESS_CLASSIFICATIONS_V.CERTIFICATE_NUMBER`
- **Filter:** `STATUS = 'A'` for active classifications
- **Pattern Match:** Search for classifications where `DISPLAYED_FIELD` or `CERTIFYING_AGENCY` contains 'TRADE LICENSE' or 'LICENSE'
- **Logic:** `MAX(CASE WHEN UPPER(DISPLAYED_FIELD) LIKE '%TRADE%LICENSE%' OR UPPER(DISPLAYED_FIELD) LIKE '%LICENSE%' OR UPPER(CERTIFYING_AGENCY) LIKE '%TRADE%LICENSE%' OR UPPER(CERTIFYING_AGENCY) LIKE '%LICENSE%' THEN CERTIFICATE_NUMBER ELSE NULL END)`
- **Why:** Trade license is stored as a business classification, not as a direct attribute

#### Payment Terms (CRITICAL)
- **Source:** `AP_TERMS_TL.NAME` (translated table, NOT `AP_TERMS`)
- **Join:** `POZ_SUPPLIER_SITES_V.TERMS_ID = AP_TERMS_TL.TERM_ID`
- **Filter:** `AP_TERMS_TL.LANGUAGE = USERENV('LANG')`
- **Why:** Translated table provides payment term names in user's language

#### Supplier Address (CRITICAL)
- **Rule:** Use `POZ_SUPPLIER_ADDRESS_V` (separate from `POZ_SUPPLIER_SITES_V`) for address details
- **Join:** `POZ_SUPPLIER_SITES_V.PARTY_SITE_ID = POZ_SUPPLIER_ADDRESS_V.PARTY_SITE_ID`
- **Why:** Address information is stored in a separate view from site information

#### Tax Profile (CRITICAL)
- **Rule:** Use `ZX_PARTY_TAX_PROFILE` for tax registration and classification (NOT `ZX_REGISTRATIONS`)
- **Join:** `POZ_SUPPLIERS_V.PARTY_ID = ZX_PARTY_TAX_PROFILE.PARTY_ID`
- **Why:** Party tax profile is the primary source for supplier tax information

#### Business Unit Source (CRITICAL)
- **Rule:** Use `FUN_ALL_BUSINESS_UNITS_V` (Financial BU view) for supplier reports
- **Join:** `POZ_SUPPLIER_SITES_V.PRC_BU_ID = FUN_ALL_BUSINESS_UNITS_V.BU_ID`
- **Why:** Financial BU view aligns with AP/AR modules and provides better integration for supplier reports

### COA Segments Source
- **Not Required for This Report:** This report does not display COA segments

---

## COA Segment Mapping

Reference: `CONFIGURATION/ENV_METADATA.md`

**Note:** This report does not currently display COA segments.

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
9. **Date Parameter Handling:** Use `TRUNC()` for date comparisons to remove time component

---

## Required CTE Structure

1. **PARAMS** - Input parameters (business unit name, supplier status, supplier type, creation date range)
2. **SUPPLIER_MASTER** - Supplier headers filtered by creation date range and supplier type
3. **VENDOR_TYPE_LOOKUP** - Vendor type meanings from FND lookup
4. **VENDOR_STATUS_LOOKUP** - Vendor status from HZ_PARTIES (Active/Inactive)
5. **SUPPLIER_SITES** - Supplier sites with business unit information
6. **SUPPLIER_ADDRESS** - Supplier address details (separate view)
7. **BUSINESS_UNIT_MASTER** - Business unit names from financial BU view
8. **TAX_PROFILE** - Tax registration and classification from party tax profile
9. **BANK_MASTER** - Primary supplier bank accounts with bank details (complex IBY join)
10. **PAYMENT_TERMS_MASTER** - Payment terms from translated table
11. **BUSINESS_CLASSIFICATIONS** - Trade license number from business classifications
12. **SUPPLIER_MASTER_DETAIL** - Final join of all CTEs with business logic and left outer joins for optional data

---

## Sorting

- **Primary:** Supplier Number (ascending)
- **Secondary:** Supplier Name (ascending)
- **Tertiary:** Supplier Site Name (ascending)
- **Quaternary:** Business Unit Name (ascending)

---

## Key Tables

| Table | Alias | Purpose |
|-------|-------|---------|
| `POZ_SUPPLIERS_V` | PSV | Supplier Master (VENDOR_NAME, SEGMENT1 as Supplier Number, PARTY_ID, VENDOR_TYPE_LOOKUP_CODE, CREATION_DATE) |
| `POZ_SUPPLIER_SITES_V` | PSSV | Supplier Site (PARTY_SITE_NAME, PARTY_SITE_ID, TERMS_ID, PRC_BU_ID) |
| `POZ_SUPPLIER_ADDRESS_V` | PSAV | Supplier Address (ADDRESS1, ADDRESS2, CITY, PHONE_NUMBER) |
| `HZ_PARTIES` | HP | Party Master (STATUS: 'A' = Active, 'I' = Inactive) |
| `ZX_PARTY_TAX_PROFILE` | ZPTP | Tax Profile (REP_REGISTRATION_NUMBER, TAX_CLASSIFICATION_CODE) |
| `IBY_EXT_BANK_ACCOUNTS` | IEBA | External Bank Accounts (BANK_ACCOUNT_NUM, BANK_ACCOUNT_NAME, BANK_ID) |
| `IBY_ACCOUNT_OWNERS` | IAO | Account Owners (PRIMARY_FLAG, ACCOUNT_OWNER_PARTY_ID) |
| `IBY_EXTERNAL_PAYEES_ALL` | IEPA | External Payees (PAYEE_PARTY_ID, SUPPLIER_SITE_ID, EXT_PAYEE_ID) |
| `IBY_PMT_INSTR_USES_ALL` | IPIUA | Payment Instrument Uses (INSTRUMENT_ID, EXT_PMT_PARTY_ID) |
| `CE_BANKS_V` | CE | Banks (BANK_NAME, BANK_NUMBER, BANK_PARTY_ID) |
| `AP_TERMS_TL` | APT | Payment Terms Translated (TERM_ID, NAME, LANGUAGE) |
| `POZ_BUSINESS_CLASSIFICATIONS_V` | PBCV | Business Classifications (CERTIFICATE_NUMBER, DISPLAYED_FIELD, CERTIFYING_AGENCY, STATUS) |
| `FUN_ALL_BUSINESS_UNITS_V` | FABUV | Business Unit Master (BU_ID, BU_NAME) |
| `FND_LOOKUP_VALUES_TL` | FLVT | Lookups (ORA_POZ_VENDOR_TYPE) |

---

## Critical Constraints

### CRITICAL: Supplier Category
- **Rule:** Supplier Category field may require validation - check if available in:
  - `POZ_SUPPLIERS_V` DFF attributes (`ATTRIBUTE1` through `ATTRIBUTE15`)
  - Supplier category assignment tables (if available in your environment)
  - Alternative: May need to use vendor type as category if category is not separately maintained
- **Action Required:** Validate with business users or check DFF configuration for supplier category field

### CRITICAL: Vendor Status Source
- **Rule:** Use `HZ_PARTIES.STATUS` (NOT `POZ_SUPPLIERS_V.ENABLED_FLAG`)
- **Join:** `POZ_SUPPLIERS_V.PARTY_ID = HZ_PARTIES.PARTY_ID`
- **Why:** Party-level status provides more accurate vendor status

### CRITICAL: Vendor Type Lookup
- **Rule:** Use `FND_LOOKUP_VALUES_TL` with `LOOKUP_TYPE = 'ORA_POZ_VENDOR_TYPE'` (NOT `'VENDOR_TYPE'`)
- **Filter:** `VIEW_APPLICATION_ID = 0`, `SET_ID = 0`, `LANGUAGE = USERENV('LANG')`

### CRITICAL: Supplier Bank Account Join Pattern
- **Rule:** Complex join required through multiple IBY tables:
  1. `IBY_EXT_BANK_ACCOUNTS` → `IBY_ACCOUNT_OWNERS` (via `EXT_BANK_ACCOUNT_ID`)
  2. `IBY_ACCOUNT_OWNERS` → `IBY_EXTERNAL_PAYEES_ALL` (via `ACCOUNT_OWNER_PARTY_ID = PAYEE_PARTY_ID`)
  3. `IBY_ACCOUNT_OWNERS` → `IBY_PMT_INSTR_USES_ALL` (via `EXT_BANK_ACCOUNT_ID = INSTRUMENT_ID`)
  4. `IBY_EXTERNAL_PAYEES_ALL` → `IBY_PMT_INSTR_USES_ALL` (via `EXT_PAYEE_ID = EXT_PMT_PARTY_ID`)
  5. `IBY_EXTERNAL_PAYEES_ALL` → Supplier Party (via `PAYEE_PARTY_ID = POZ_SUPPLIERS_V.PARTY_ID`)
- **Filter:** `IBY_ACCOUNT_OWNERS.PRIMARY_FLAG = 'Y'` for primary bank account only
- **Bank Details:** Join `CE_BANKS_V` via `IBY_EXT_BANK_ACCOUNTS.BANK_ID = CE_BANKS_V.BANK_PARTY_ID`

### CRITICAL: Tax Profile Table
- **Rule:** Use `ZX_PARTY_TAX_PROFILE` for tax registration and classification (NOT `ZX_REGISTRATIONS`)
- **Join:** `POZ_SUPPLIERS_V.PARTY_ID = ZX_PARTY_TAX_PROFILE.PARTY_ID`

### CRITICAL: Business Classifications for Trade License
- **Rule:** Use `POZ_BUSINESS_CLASSIFICATIONS_V` to retrieve trade license number
- **Filter:** `STATUS = 'A'` for active classifications
- **Pattern:** Search for classifications where `DISPLAYED_FIELD` or `CERTIFYING_AGENCY` contains 'TRADE LICENSE' or 'LICENSE'
- **Column:** `CERTIFICATE_NUMBER` contains the trade license number

### CRITICAL: Payment Terms Translated Table
- **Rule:** Use `AP_TERMS_TL` (translated table) instead of `AP_TERMS` for multilingual support
- **Filter:** `AP_TERMS_TL.LANGUAGE = USERENV('LANG')`
- **Join:** `POZ_SUPPLIER_SITES_V.TERMS_ID = AP_TERMS_TL.TERM_ID`

### CRITICAL: Supplier Address Table
- **Rule:** Use `POZ_SUPPLIER_ADDRESS_V` (separate from `POZ_SUPPLIER_SITES_V`) for address details
- **Join:** `POZ_SUPPLIER_SITES_V.PARTY_SITE_ID = POZ_SUPPLIER_ADDRESS_V.PARTY_SITE_ID`

### CRITICAL: Business Unit Source
- **Rule:** Use `FUN_ALL_BUSINESS_UNITS_V` (Financial BU view) for supplier reports
- **Join:** `POZ_SUPPLIER_SITES_V.PRC_BU_ID = FUN_ALL_BUSINESS_UNITS_V.BU_ID`

### CRITICAL: Optional Parameter Filters
- **Rule:** All optional parameters must use NVL pattern to make them optional
- **Pattern:** `NVL(:P_PARAMETER, COLUMN_VALUE) = COLUMN_VALUE`
- **Logic:** 
  - If parameter is NULL: `COLUMN_VALUE = COLUMN_VALUE` (always true - shows all values)
  - If parameter is NOT NULL: `:P_PARAMETER = COLUMN_VALUE` (filters by parameter value)
- **Why:** NVL pattern allows optional parameter filtering without using OR condition

### CRITICAL: Left Outer Joins for Optional Data
- **Rule:** All optional data must use left outer joins `(+)` because:
  - Supplier may not have sites (if no sites created)
  - Site may not have address (if address not entered)
  - Supplier may not have bank account (if not configured)
  - Supplier may not have tax profile (if not configured)
  - Supplier may not have payment terms (if not assigned to site)
  - Supplier may not have trade license (if not configured)
- **Why:** Report must show complete supplier information even if optional data is missing

---

## Notes

- **Multiple Sites:** One supplier can have multiple sites - each site appears as separate row
- **Address Link:** Address information is stored in separate view from site information - must join via `PARTY_SITE_ID`
- **Bank Account:** Only primary bank account is shown per supplier (filtered by `PRIMARY_FLAG = 'Y'`)
- **Tax Profile:** Tax information is at party level, not site level
- **Payment Terms:** Payment terms are assigned at site level
- **Trade License:** Trade license is stored as business classification - pattern matching required to identify trade license
- **Date Formatting:** All dates formatted as `DD-MON-YYYY` for consistency
- **Supplier Category:** May require validation - check DFF attributes or category assignment tables

---

## Integration Flow Diagram

```
Supplier (POZ_SUPPLIERS_V)
  ↓ (VENDOR_ID)
Supplier Sites (POZ_SUPPLIER_SITES_V)
  ↓ (PARTY_SITE_ID)
Supplier Address (POZ_SUPPLIER_ADDRESS_V)
  ↓ (PRC_BU_ID)
Business Unit (FUN_ALL_BUSINESS_UNITS_V)
  ↓ (TERMS_ID)
Payment Terms (AP_TERMS_TL)

Supplier (POZ_SUPPLIERS_V)
  ↓ (PARTY_ID)
Party Status (HZ_PARTIES)
  ↓ (PARTY_ID)
Tax Profile (ZX_PARTY_TAX_PROFILE)
  ↓ (PARTY_ID)
Bank Accounts (IBY_EXT_BANK_ACCOUNTS via IBY_ACCOUNT_OWNERS, IBY_EXTERNAL_PAYEES_ALL, IBY_PMT_INSTR_USES_ALL)
  ↓ (BANK_ID)
Banks (CE_BANKS_V)

Supplier (POZ_SUPPLIERS_V)
  ↓ (VENDOR_ID)
Business Classifications (POZ_BUSINESS_CLASSIFICATIONS_V)
```

**Key Join Points:**
1. Supplier → Sites: `POZ_SUPPLIERS_V.VENDOR_ID = POZ_SUPPLIER_SITES_V.VENDOR_ID`
2. Sites → Address: `POZ_SUPPLIER_SITES_V.PARTY_SITE_ID = POZ_SUPPLIER_ADDRESS_V.PARTY_SITE_ID`
3. Supplier → Party Status: `POZ_SUPPLIERS_V.PARTY_ID = HZ_PARTIES.PARTY_ID`
4. Supplier → Tax Profile: `POZ_SUPPLIERS_V.PARTY_ID = ZX_PARTY_TAX_PROFILE.PARTY_ID`
5. Supplier → Bank Accounts: Complex join through IBY tables via `PARTY_ID`
6. Sites → Payment Terms: `POZ_SUPPLIER_SITES_V.TERMS_ID = AP_TERMS_TL.TERM_ID`
7. Sites → Business Unit: `POZ_SUPPLIER_SITES_V.PRC_BU_ID = FUN_ALL_BUSINESS_UNITS_V.BU_ID`

---

**Last Updated:** 19-01-2026  
**Validation Status:** ✅ Validated against PO_MASTER.md and PO_REPOSITORIES.md  
**Note:** Supplier Category field requires validation - check DFF attributes or category assignment tables in your environment

