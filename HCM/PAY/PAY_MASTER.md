# Payroll Master Instructions

**Module:** Global Payroll  
**Tag:** `#HCM #PAY #Payroll`  
**Status:** Production-Ready  
**Last Updated:** 18-Dec-2025

---

## 1. 🚨 Critical Payroll Constraints

*Violating these rules breaks payroll calculations.*

### 1.1 Action Type Filtering
**Rule:** Always filter by `ACTION_TYPE` to get correct payroll results

```sql
AND PPA.ACTION_TYPE IN ('Q', 'R')  -- QuickPay or Run
-- OR specific type:
AND PPA.ACTION_TYPE = 'R'  -- Regular Run only
```

**Action Types:**
- `'R'` - Regular Payroll Run
- `'Q'` - QuickPay (off-cycle)
- `'V'` - Void
- `'P'` - Prepayment

**Why:** Without this filter, you get ALL actions including voids, prepayments, reversals.

### 1.2 Action Status Filtering
**Rule:** Only include COMPLETED payroll actions

```sql
AND PPA.ACTION_STATUS IN ('C')  -- Completed only
```

**Why:** Prevents including in-progress or failed runs.

### 1.3 Retro Component Exclusion
**Rule:** Exclude retroactive adjustments unless specifically needed

```sql
AND PPRA.RETRO_COMPONENT_ID IS NULL
```

**Why:** Retro entries duplicate amounts and create incorrect totals.

### 1.4 Time Period Context
**Rule:** Always join to `PAY_TIME_PERIODS` for date context

```sql
AND PPA.EARN_TIME_PERIOD_ID = PTP.TIME_PERIOD_ID
```

**Why:** Payroll results exist within specific processing windows.

### 1.5 Element Classification
**Rule:** Filter by `CLASSIFICATION_NAME` to separate earnings, deductions, information

```sql
AND PETF.CLASSIFICATION_ID = PEC.CLASSIFICATION_ID
AND PEC.BASE_CLASSIFICATION_NAME IN ('Standard Earnings')
-- OR
AND PEC.BASE_CLASSIFICATION_NAME IN ('Voluntary Deductions')
-- OR
AND PEC.BASE_CLASSIFICATION_NAME IN ('Information')
```

**Classifications:**
- `'Standard Earnings'` - Salary, allowances, bonuses
- `'Voluntary Deductions'` - Deductions that reduce net pay
- `'Social Insurance Deductions'` - Social security, insurance
- `'Involuntary Deductions'` - Court orders, garnishments
- `'Information'` - Informational elements (no payment impact)

**Why:** Without this, you mix earnings and deductions incorrectly.

### 1.6 Input Value Selection
**Rule:** Use `BASE_NAME = 'Pay Value'` for monetary amounts

```sql
AND PIVF.BASE_NAME = 'Pay Value'
-- OR
AND UPPER(TRIM(PIVF.BASE_NAME)) = 'PAY VALUE'
```

**Common Input Values:**
- `'Pay Value'` - The monetary result
- `'Amount'` - Entry amount (for element entries)
- `'Rate'` - Rate value
- `'Hours'` - Hour value
- `'Percentage'` - Percentage value

---

## 2. ⚡ Performance Optimization

| Object | Optimal Access Path | Hint Syntax |
|--------|---------------------|-------------|
| **Payroll Action** | PAYROLL_ACTION_ID | `/*+ INDEX(PPA PAY_PAYROLL_ACTIONS_PK) */` |
| **Run Results** | ASSIGNMENT_ACTION_ID | `/*+ INDEX(PRR PAY_RUN_RESULTS_N1) */` |
| **Element Entry** | ASSIGNMENT_ID | `/*+ INDEX(PEE PAY_ELEMENT_ENTRIES_F_N50) */` |
| **Element Type** | ELEMENT_TYPE_ID | `/*+ INDEX(PETF PAY_ELEMENT_TYPES_F_PK) */` |

---

## 3. 🗺️ Schema Map

### 3.1 Payroll Run Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PPA** | `PAY_PAYROLL_ACTIONS` | Payroll Run Header (Batch, Date, Status) |
| **PAA** | `PAY_ASSIGNMENT_ACTIONS` | Assignment-level Actions |
| **PPRA** | `PAY_PAYROLL_REL_ACTIONS` | Payroll Relationship Actions |
| **PRR** | `PAY_RUN_RESULTS` | Run Results Header |
| **PRRV** | `PAY_RUN_RESULT_VALUES` | Individual Result Values ($$) |
| **PPRD** | `PAY_PAY_RELATIONSHIPS_DN` | Pay Relationships |
| **PAC** | `PAY_ACTION_CLASSES` | Action Classifications |

### 3.2 Element Definition Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PETF** | `PAY_ELEMENT_TYPES_F` | Element Type Definitions (Date-Tracked) |
| **PETT** | `PAY_ELEMENT_TYPES_TL` | Element Type Translations |
| **PETV** | `PAY_ELEMENT_TYPES_VL` | Element Types View |
| **PIVF** | `PAY_INPUT_VALUES_F` | Input Value Definitions (Date-Tracked) |
| **PIVT** | `PAY_INPUT_VALUES_TL` | Input Value Translations |
| **PIVL** | `PAY_INPUT_VALUES_VL` | Input Values View |
| **PEC** | `PAY_ELE_CLASSIFICATIONS` | Element Classifications |
| **PECT** | `PAY_ELE_CLASSIFICATIONS_TL` | Element Classification Translations |

### 3.3 Element Entry Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PEEF** | `PAY_ELEMENT_ENTRIES_F` | Element Entries (Ongoing Assignments) |
| **PEEV** | `PAY_ELEMENT_ENTRIES_VL` | Element Entries View |
| **PEEVF** | `PAY_ELEMENT_ENTRY_VALUES_F` | Entry Values (Date-Tracked) |

### 3.4 Time Period Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PTP** | `PAY_TIME_PERIODS` | Payroll Time Periods |
| **PAP** / **PAPP** | `PAY_ALL_PAYROLLS_F` | Payroll Definitions |

### 3.5 Payment & Bank Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PPA** | `PAY_PAYROLL_ASSIGNMENTS` | Payroll Assignments |
| **PPPMF** | `PAY_PERSONAL_PAYMENT_METHODS_F` | Personal Payment Methods |
| **PBA** | `PAY_BANK_ACCOUNTS` | Bank Account Details |
| **POPM** | `PAY_ORG_PAY_METHODS_VL` | Organization Payment Methods |
| **PPT** | `PAY_PAYMENT_TYPES_VL` | Payment Types |
| **PCPV** | `PAY_CARD_PAYSLIPS_V` | Card Payslips View |

### 3.6 Payroll Relationship Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PPE** / **PPRD** | `PAY_PAY_RELATIONSHIPS_DN` | Pay Relationships |
| **PRG** | `PAY_REL_GROUPS_DN` | Relationship Groups |
| **PAPD** / **AP** | `PAY_ASSIGNED_PAYROLLS_DN` | Assigned Payrolls |
| **PT** | `PAY_PAYROLL_TERMS` | Payroll Terms |

### 3.7 Advanced Payroll Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PCS** | `PAY_CONSOLIDATION_SETS` | Consolidation Sets (Grouping) |
| **PRQ** | `PAY_REQUESTS` | Payroll Requests |
| **PFI** | `PAY_FLOW_INSTANCES` | Payroll Flow Instances (Processing Tracking) |
| **PBT** | `PAY_BALANCE_TYPES_VL` | Balance Type Definitions |
| **PDU** | `PAY_DIMENSION_USAGES_VL` | Dimension Usages |
| **LDG** | `PER_LEGISLATIVE_DATA_GROUPS_VL` | Legislative Data Groups |

---

## 4. 📋 Element Patterns

### 4.1 Hardcoded Element Names Pattern

**Use Case:** When you know exact element names

**Pattern:**
```sql
SUM(CASE WHEN PETF.BASE_ELEMENT_NAME = 'OGH Basic Salary' 
         THEN TO_NUMBER(PEEV.SCREEN_ENTRY_VALUE) END) BASIC
,SUM(CASE WHEN PETF.BASE_ELEMENT_NAME = 'OGH Housing Allowance' 
         THEN TO_NUMBER(PEEV.SCREEN_ENTRY_VALUE) END) HOUSING
```

**Common Prefixes:**
- `'OGH'` - Organization-specific elements (e.g., 'OGH Basic Salary')
- Element names vary by implementation

**Hardcoded Elements (Example from Production):**
- Basic: `'OGH Basic Salary'`, `'Basic'`, `'Basic Salary'`
- Housing: `'OGH Housing Allowance'`, `'Housing Allowance'`, `'HRA'`
- Transport: `'OGH Transportation Allowance'`, `'Transport Allowance'`
- Overtime: `'Holidays Overtime'`, `'Normal Overtime'`, `'Weekend Overtime'`
- Deductions: Element names ending with `'Results'` (e.g., `'Housing Deduction Results'`)

### 4.2 Dynamic Element Loading Pattern

**Use Case:** When element names are not known in advance

**Pattern:**
```sql
,PETT.ELEMENT_NAME  -- Use translations, not BASE
,SUM(TO_NUMBER(PRRV.RESULT_VALUE)) VALUE
,CASE
    WHEN PEC.BASE_CLASSIFICATION_NAME = 'Standard Earnings' THEN 1
    WHEN PEC.BASE_CLASSIFICATION_NAME IN ('Voluntary Deductions', ...) THEN 2
    ELSE 3
END AS CUSTOM_ORDER  -- For sorting
```

**Key Difference:** Use `ELEMENT_NAME` (translated) instead of `BASE_ELEMENT_NAME`

### 4.3 Classification-Based Aggregation

**Pattern:**
```sql
SUM(CASE WHEN PEC.BASE_CLASSIFICATION_NAME IN ('Standard Earnings')
         THEN TO_NUMBER(PRRV.RESULT_VALUE) END) TOTAL_EARNINGS
,SUM(CASE WHEN PEC.BASE_CLASSIFICATION_NAME IN ('Voluntary Deductions')
         THEN TO_NUMBER(PRRV.RESULT_VALUE) END) TOTAL_DEDUCTIONS
,(SUM(CASE WHEN PEC.BASE_CLASSIFICATION_NAME IN ('Standard Earnings')
           THEN TO_NUMBER(PRRV.RESULT_VALUE) END) -
  SUM(CASE WHEN PEC.BASE_CLASSIFICATION_NAME IN ('Voluntary Deductions')
           THEN TO_NUMBER(PRRV.RESULT_VALUE) END)) NET_PAY
```

### 4.4 Reporting Name Pattern (for Accruals)

**Use Case:** For accrual calculations

**Pattern:**
```sql
AND PETF.REPORTING_NAME = 'Gratuity Accrual'
AND PIVTFB.NAME = 'Monthly Amount'
-- NOT BASE_ELEMENT_NAME
```

**Common Reporting Names:**
- `'Gratuity Accrual'`
- `'Gratuity Accrual Payment'`
- `'Gratuity Gain'`
- `'Basic Salary'`
- `'Annual Leave Accrual'`

---

## 5. 🔗 Standard Joins (Copy-Paste Ready)

### 5.1 Payroll Run Results Join

**Full chain from Action to Values:**
```sql
FROM PAY_PAYROLL_ACTIONS PPA,
     PAY_PAYROLL_REL_ACTIONS PPRA,
     PAY_RUN_RESULTS PRR,
     PAY_RUN_RESULT_VALUES PRRV,
     PAY_PAY_RELATIONSHIPS_DN PPRD,
     PAY_TIME_PERIODS PTP,
     PAY_ELEMENT_TYPES_F PETF,
     PAY_INPUT_VALUES_F PIVF,
     PAY_ALL_PAYROLLS_F PAP,
     PAY_ELE_CLASSIFICATIONS PEC
WHERE PPRA.PAYROLL_ACTION_ID = PPA.PAYROLL_ACTION_ID
AND PRR.PAYROLL_REL_ACTION_ID = PPRA.PAYROLL_REL_ACTION_ID(+)
AND PRRV.RUN_RESULT_ID = PRR.RUN_RESULT_ID
AND PPRA.PAYROLL_RELATIONSHIP_ID = PPRD.PAYROLL_RELATIONSHIP_ID
AND PPA.ACTION_TYPE IN ('Q', 'R')
AND PPA.ACTION_STATUS IN ('C')
AND PPA.EARN_TIME_PERIOD_ID = PTP.TIME_PERIOD_ID
AND PETF.ELEMENT_TYPE_ID = PRR.ELEMENT_TYPE_ID
AND PIVF.ELEMENT_TYPE_ID = PETF.ELEMENT_TYPE_ID
AND PIVF.INPUT_VALUE_ID = PRRV.INPUT_VALUE_ID
AND PPA.PAYROLL_ID = PAP.PAYROLL_ID
AND PPRA.RETRO_COMPONENT_ID IS NULL
AND PETF.CLASSIFICATION_ID = PEC.CLASSIFICATION_ID
AND PIVF.BASE_NAME = 'Pay Value'
```

### 5.2 Element Entry Join

**For ongoing element assignments:**
```sql
FROM PAY_ELEMENT_ENTRIES_F PEEF,
     PAY_ELEMENT_ENTRY_VALUES_F PEEV,
     PAY_INPUT_VALUES_F PIVF,
     PAY_ELEMENT_TYPES_F PETF,
     PAY_ELE_CLASSIFICATIONS PEC
WHERE PEEF.ELEMENT_ENTRY_ID = PEEV.ELEMENT_ENTRY_ID(+)
AND PEEF.ELEMENT_TYPE_ID = PETF.ELEMENT_TYPE_ID(+)
AND PEEV.INPUT_VALUE_ID = PIVF.INPUT_VALUE_ID(+)
AND PEEF.ELEMENT_TYPE_ID = PIVF.ELEMENT_TYPE_ID(+)
AND PETF.CLASSIFICATION_ID = PEC.CLASSIFICATION_ID(+)
AND PEC.BASE_CLASSIFICATION_NAME IN ('Standard Earnings')
```

### 5.3 Bank Account Join

**Person → Payroll → Payment Method → Bank:**
```sql
FROM PAY_PAYROLL_ASSIGNMENTS PPA,
     PAY_PERSONAL_PAYMENT_METHODS_F PPPMF,
     PAY_BANK_ACCOUNTS PBA,
     PAY_ORG_PAY_METHODS_VL POPM,
     PAY_PAYMENT_TYPES_VL PPT
WHERE PPA.PAYROLL_RELATIONSHIP_ID = PPPMF.PAYROLL_RELATIONSHIP_ID(+)
AND PPPMF.BANK_ACCOUNT_ID = PBA.BANK_ACCOUNT_ID(+)
AND PPPMF.ORG_PAYMENT_METHOD_ID = POPM.ORG_PAYMENT_METHOD_ID(+)
AND POPM.PAYMENT_TYPE_ID = PPT.PAYMENT_TYPE_ID(+)
AND TO_DATE(:P_PERIOD, 'DDMMYYYY') BETWEEN TRUNC(PPPMF.EFFECTIVE_START_DATE) 
    AND TRUNC(PPPMF.EFFECTIVE_END_DATE)
```

**Payment Method Hierarchy:**
```sql
NVL(PPT.BASE_PAYMENT_TYPE_NAME,
    NVL(PPPMF.NAME,
        NVL(POPM.ORG_PAYMENT_METHOD_NAME,
            PPT.PAYMENT_TYPE_NAME))) PAY_METHOD_NAME
```

### 5.4 Payroll Assignment Join

**Person → Payroll Name:**
```sql
FROM PAY_PAY_RELATIONSHIPS_DN PPE,
     PAY_REL_GROUPS_DN PRG,
     PAY_ASSIGNED_PAYROLLS_DN PAPD,
     PAY_ALL_PAYROLLS_F PAP
WHERE PPE.PAYROLL_RELATIONSHIP_ID = PRG.PAYROLL_RELATIONSHIP_ID
AND PRG.RELATIONSHIP_GROUP_ID = PAPD.PAYROLL_TERM_ID
AND PAPD.PAYROLL_ID = PAP.PAYROLL_ID
AND TRUNC(SYSDATE) BETWEEN TRUNC(PAP.EFFECTIVE_START_DATE) 
    AND TRUNC(PAP.EFFECTIVE_END_DATE)
```

### 5.5 Balance Dimensions Join

**Using TABLE function for balances:**
```sql
FROM PAY_PAYROLL_REL_ACTIONS PRA,
     PAY_PAYROLL_ACTIONS PPA,
     PAY_BALANCE_TYPES_VL PBT,
     TABLE(PAY_BALANCE_VIEW_PKG.GET_BALANCE_DIMENSIONS(
         P_BALANCE_TYPE_ID => PBT.BALANCE_TYPE_ID,
         P_PAYROLL_REL_ACTION_ID => PRA.PAYROLL_REL_ACTION_ID,
         P_PAYROLL_TERM_ID => NULL,
         P_PAYROLL_ASSIGNMENT_ID => NULL
     )) BAL,
     PAY_DIMENSION_USAGES_VL PDU
WHERE PPA.PAYROLL_ACTION_ID = PRA.PAYROLL_ACTION_ID
AND PBT.BALANCE_NAME IN ('Air Fare Provision', 'DYR_Gratuity_Provision')
AND PDU.DIMENSION_NAME = 'Assignment Inception to Date'
AND PDU.BALANCE_DIMENSION_ID = BAL.BALANCE_DIMENSION_ID
AND BAL.BALANCE_VALUE != '0'
```

### 5.6 Flow Instance Join (Dynamic Payroll)

**For payroll processing tracking:**
```sql
FROM PAY_PAYROLL_ACTIONS PPA,
     PAY_REQUESTS PRQ,
     PAY_FLOW_INSTANCES PFI,
     PAY_CONSOLIDATION_SETS PCS
WHERE PPA.PAY_REQUEST_ID = PRQ.PAY_REQUEST_ID
AND PRQ.FLOW_INSTANCE_ID = PFI.FLOW_INSTANCE_ID
AND PPA.CONSOLIDATION_SET_ID = PCS.CONSOLIDATION_SET_ID
AND PFI.INSTANCE_NAME = NVL(:P_FLOW_NAME, PFI.INSTANCE_NAME)
AND PCS.CONSOLIDATION_SET_NAME = NVL(:P_CONSOLIDATE_SET, PCS.CONSOLIDATION_SET_NAME)
```

---

## 6. 📊 Standard Filters

### 6.1 Period Filtering

**Last Day of Month Pattern:**
```sql
AND TRUNC(LAST_DAY(PPA.EFFECTIVE_DATE)) = 
    NVL(TO_DATE(:P_PERIOD, 'DDMMYYYY'), TRUNC(LAST_DAY(PPA.EFFECTIVE_DATE)))
```

**Date Range Pattern:**
```sql
AND PPA.DATE_EARNED BETWEEN :P_START_DATE AND :P_END_DATE
```

**Effective Date with Element Types:**
```sql
AND TRUNC(LAST_DAY(PPA.EFFECTIVE_DATE)) BETWEEN TRUNC(PETF.EFFECTIVE_START_DATE) 
    AND TRUNC(PETF.EFFECTIVE_END_DATE)
AND TRUNC(LAST_DAY(PPA.EFFECTIVE_DATE)) BETWEEN TRUNC(PIVF.EFFECTIVE_START_DATE) 
    AND TRUNC(PIVF.EFFECTIVE_END_DATE)
```

### 6.2 Element Name Filters

**For Earnings (Exclude Results):**
```sql
AND PETF.BASE_ELEMENT_NAME NOT LIKE '%Results'
```

**For Deductions (Include Results):**
```sql
AND PETF.BASE_ELEMENT_NAME LIKE '%Results'
```

### 6.3 Processing Type Filter

**For Element Entries:**
```sql
AND PETF.PROCESSING_TYPE = 'R'  -- Recurring
```

**Processing Types:**
- `'R'` - Recurring (ongoing)
- `'N'` - Nonrecurring (one-time)

### 6.4 Termination Date Handling

**For Element Entries with Terminated Employees:**
```sql
AND LEAST(NVL(POPS.ACTUAL_TERMINATION_DATE, TRUNC(:P_DATE)), TRUNC(:P_DATE)) 
    BETWEEN TRUNC(PEEF.EFFECTIVE_START_DATE) 
    AND TRUNC(PEEF.EFFECTIVE_END_DATE)
```

### 6.5 Legislative Data Group Filter

**For Multi-Legislation Environments:**
```sql
AND NVL(PBT.LEGISLATION_CODE, LDG.LEGISLATION_CODE) = LDG.LEGISLATION_CODE
AND NVL(PBT.LEGISLATIVE_DATA_GROUP_ID, LDG.LEGISLATIVE_DATA_GROUP_ID) = 
    LDG.LEGISLATIVE_DATA_GROUP_ID
```

---

## 7. ⚠️ Common Pitfalls

### 7.1 Including Voided Runs
**Problem:** Totals are incorrect  
**Cause:** Not filtering `ACTION_STATUS`

**Solution:**
```sql
AND PPA.ACTION_STATUS IN ('C')  -- Completed only
```

### 7.2 Including Retro Adjustments
**Problem:** Duplicate amounts  
**Cause:** Not excluding `RETRO_COMPONENT_ID`

**Solution:**
```sql
AND PPRA.RETRO_COMPONENT_ID IS NULL
```

### 7.3 Missing Time Period Context
**Problem:** Results span multiple periods  
**Cause:** Not filtering by period

**Solution:**
```sql
AND PPA.EARN_TIME_PERIOD_ID = PTP.TIME_PERIOD_ID
AND TRUNC(LAST_DAY(PPA.EFFECTIVE_DATE)) = :P_PERIOD_END
```

### 7.4 Wrong Input Value
**Problem:** Getting non-monetary values  
**Cause:** Not specifying `BASE_NAME`

**Solution:**
```sql
AND PIVF.BASE_NAME = 'Pay Value'  -- For run results
-- OR
AND PIVF.NAME = 'Amount'  -- For element entries
```

### 7.5 Mixing Element Entry and Run Results
**Problem:** Seeing unexpected values  
**Cause:** Element entries show setup, run results show calculations

**Solution:**
- Use **Element Entries** (`PAY_ELEMENT_ENTRIES_F`) for current assignments/setup
- Use **Run Results** (`PAY_RUN_RESULTS`) for actual payroll calculations

### 7.6 Using BASE_ELEMENT_NAME for Accruals
**Problem:** Accrual elements not found  
**Cause:** Accruals use `REPORTING_NAME`, not `BASE_ELEMENT_NAME`

**Solution:**
```sql
AND PETF.REPORTING_NAME = 'Gratuity Accrual'  -- Not BASE_ELEMENT_NAME
```

---

## 8. 💡 Calculation Patterns

### 8.1 Net Pay Calculation

```sql
(SUM(CASE WHEN PEC.BASE_CLASSIFICATION_NAME IN ('Standard Earnings')
          THEN TO_NUMBER(PRRV.RESULT_VALUE) END) -
 SUM(CASE WHEN PEC.BASE_CLASSIFICATION_NAME IN ('Voluntary Deductions')
          THEN TO_NUMBER(PRRV.RESULT_VALUE) END)) NET_PAY
```

### 8.2 Gross Salary (from Element Entries)

```sql
(NVL(BASIC, 0) + 
 NVL(HOUSING, 0) + 
 NVL(TRANSPORT, 0) +
 ... all allowances ...) GROSS_SALARY
```

### 8.3 Monthly Basic from Annual

```sql
ROUND(PEEV.SCREEN_ENTRY_VALUE / 12, 2) MONTHLY_BASIC
```

---

## 9. 📅 Parameters

| Parameter | Format | Description | Example |
|-----------|--------|-------------|---------|
| `:P_PERIOD` | DDMMYYYY | Period end date | 31122024 |
| `:P_PERIOD_NAME` | DD-MM-YYYY | Period name | 31-12-2024 |
| `:P_PAYROLL` | Numeric | Payroll ID | 300000123456 |
| `:P_PAYROLLNAME` | String | Payroll name | 'UAE Monthly' |
| `:P_FLOW_NAME` | String | Flow instance | 'Monthly Payroll Dec 2024' |
| `:P_CONSOLIDATE_SET` | String | Consolidation set | 'Standard' |
| `:P_START_DATE` | Date | Start date | TO_DATE('01-12-2024','DD-MM-YYYY') |
| `:P_END_DATE` | Date | End date | TO_DATE('31-12-2024','DD-MM-YYYY') |
| `:P_DATE` | Date | Effective date | TRUNC(SYSDATE) |

---

## 10. 🔍 Advanced Patterns

### 10.1 Scalar Subquery for Single Element

**For Basic Salary:**
```sql
(SELECT ROUND(PEEV.SCREEN_ENTRY_VALUE / 12, 2)
 FROM PAY_ELEMENT_TYPES_VL PETV,
      PAY_ELEMENT_ENTRIES_F PEEF,
      PAY_ELEMENT_ENTRY_VALUES_F PEEV,
      PAY_INPUT_VALUES_VL PIVL
 WHERE PETV.ELEMENT_TYPE_ID = PEEF.ELEMENT_TYPE_ID
 AND PEEF.ELEMENT_ENTRY_ID = PEEV.ELEMENT_ENTRY_ID
 AND PEEV.INPUT_VALUE_ID = PIVL.INPUT_VALUE_ID
 AND PETV.BASE_ELEMENT_NAME = 'Basic'
 AND UPPER(TRIM(PIVL.NAME)) = 'AMOUNT'
 AND PEEF.PERSON_ID = PAPF.PERSON_ID
 AND :P_DATE BETWEEN PEEF.EFFECTIVE_START_DATE AND PEEF.EFFECTIVE_END_DATE
 AND ROWNUM = 1) BASIC_SALARY
```

### 10.2 Cost Center Decode (Client-Specific)

**Hardcoded cost center mapping:**
```sql
DECODE(DEPARTMENT_NAME,
    'Business Support', '502',
    'CEO Off', '510',
    'Human Resources', '512',
    'Finance', '501',
    'IT', '508',
    ...) COST_CENTER
```

---

**Last Updated:** 18-Dec-2025  
**Status:** Production-Ready  
**Source:** 4 Production Payroll Queries (1,252 lines analyzed)
