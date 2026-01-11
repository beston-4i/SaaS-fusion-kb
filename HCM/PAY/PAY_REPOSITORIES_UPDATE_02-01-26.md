# Payroll Repository Patterns - UPDATE 02-01-26

**Module:** Global Payroll  
**Update Type:** Cross-Module Knowledge Integration  
**Source:** Employee Annual Leave Balance Report  
**Date:** 02-01-26  
**Applicability:** MEDIUM - 4 patterns applicable

---

## 📋 EXECUTIVE SUMMARY

This update adds Payroll-applicable patterns from the Employee Annual Leave Balance Report. Focus is on effective date filtering, parameter handling, and optional table patterns that improve payroll query flexibility and accuracy.

**Impact:** Medium - Enhances historical payroll queries and parameter handling  
**Priority:** Medium (integrate within 1 month)  
**Scope:** Payroll queries requiring historical accuracy

---

## 🆕 APPLICABLE PATTERNS FOR PAYROLL

### Pattern 1: Effective Date Filtering for Date-Tracked Elements

**What's New:** Parameter-based date filtering instead of hardcoded period dates

**Current Payroll Pattern:**
```sql
AND TRUNC(LAST_DAY(PPA.EFFECTIVE_DATE)) = :P_PERIOD_END
AND TRUNC(LAST_DAY(PPA.EFFECTIVE_DATE)) BETWEEN TRUNC(PETF.EFFECTIVE_START_DATE) 
    AND TRUNC(PETF.EFFECTIVE_END_DATE)
```

**Enhanced Pattern:**
```sql
WITH PARAMETERS AS (
    SELECT 
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        TRUNC(LAST_DAY(TO_DATE(:P_PERIOD_END, 'DD-MON-YYYY'))) AS PERIOD_END
    FROM DUAL
)

-- Apply consistently
AND P.EFFECTIVE_DATE BETWEEN PETF.EFFECTIVE_START_DATE AND PETF.EFFECTIVE_END_DATE
AND P.EFFECTIVE_DATE BETWEEN PIVF.EFFECTIVE_START_DATE AND PIVF.EFFECTIVE_END_DATE
```

**Benefits:**
- More flexible date handling
- Can query element definitions as of any past date
- Consistent with HR/Absence date patterns

**Applicable To:** Element entry queries, element type lookups

---

### Pattern 2: Case-Insensitive Parameter Filtering

**What's New:** UPPER() for element name and payroll name filtering

**Current Pattern:**
```sql
AND PET F.BASE_ELEMENT_NAME = :P_ELEMENT_NAME
AND PAP.PAYROLL_NAME = :P_PAYROLL_NAME
```

**Enhanced Pattern:**
```sql
WITH PARAMETERS AS (
    SELECT
        UPPER(NVL(:P_ELEMENT_NAME, 'ALL')) AS ELEMENT_NAME,
        UPPER(NVL(:P_PAYROLL_NAME, 'ALL')) AS PAYROLL_NAME,
        UPPER(NVL(:P_CLASSIFICATION, 'ALL')) AS CLASSIFICATION
    FROM DUAL
)

-- Apply in WHERE
AND (UPPER(PETF.BASE_ELEMENT_NAME) = P.ELEMENT_NAME OR P.ELEMENT_NAME = 'ALL')
AND (UPPER(PAP.PAYROLL_NAME) = P.PAYROLL_NAME OR P.PAYROLL_NAME = 'ALL')
AND (UPPER(PEC.BASE_CLASSIFICATION_NAME) = P.CLASSIFICATION OR P.CLASSIFICATION = 'ALL')
```

**Benefits:**
- Users can enter "basic salary", "Basic Salary", or "BASIC SALARY"
- 'ALL' bypass for showing all elements/payrolls
- Simpler than NULL handling

**Applicable To:** All payroll queries with filters

---

### Pattern 3: Optional Table Handling for Accruals

**What's New:** Handling tables that may not exist in all environments

**Pattern:**
```sql
-- Define optional accrual CTE
PAY_ACCRUAL_CUSTOM AS (
    SELECT
        PERSON_ID,
        ACCRUAL_AMOUNT
    FROM CUSTOM_ACCRUAL_TABLE  -- May not exist in all environments
    WHERE ACTIVE_FLAG = 'Y'
)

-- Use outer join in main query
FROM
    PAY_RESULTS_MASTER PRM,
    PAY_ACCRUAL_CUSTOM PAC
WHERE
    PRM.PERSON_ID = PAC.PERSON_ID(+)

-- Handle NULL in SELECT
SELECT
    PRM.PERSON_NUMBER,
    NVL(PAC.ACCRUAL_AMOUNT, 0) AS CUSTOM_ACCRUAL
FROM ...
```

**Documentation:**
```sql
/*
 * OPTIONAL TABLES FOR PAYROLL
 * ============================
 * 1. CUSTOM_ACCRUAL_TABLE - Custom accrual tracking
 *    - If missing, comment out PAY_ACCRUAL_CUSTOM CTE (lines XX-XX)
 *    - Accrual fields will show as 0
 */
```

**Applicable To:** Custom element tables, client-specific accrual tables

---

### Pattern 4: Component Breakdown Pattern (from Balance Calculation)

**What's New:** Showing balance components for transparency

**Pattern Application for Payroll:**
```sql
SELECT
    PERSON_NUMBER,
    FULL_NAME,
    -- Show components
    NVL(BASIC_SALARY, 0) AS BASIC_SALARY,
    NVL(HOUSING_ALLOWANCE, 0) AS HOUSING_ALLOWANCE,
    NVL(TRANSPORT_ALLOWANCE, 0) AS TRANSPORT_ALLOWANCE,
    NVL(OTHER_ALLOWANCES, 0) AS OTHER_ALLOWANCES,
    -- Show calculated gross
    (NVL(BASIC_SALARY, 0) + NVL(HOUSING_ALLOWANCE, 0) + 
     NVL(TRANSPORT_ALLOWANCE, 0) + NVL(OTHER_ALLOWANCES, 0)) AS CALCULATED_GROSS_SALARY,
    -- Show deductions
    NVL(TOTAL_DEDUCTIONS, 0) AS TOTAL_DEDUCTIONS,
    -- Show net
    ((NVL(BASIC_SALARY, 0) + NVL(HOUSING_ALLOWANCE, 0) + 
      NVL(TRANSPORT_ALLOWANCE, 0) + NVL(OTHER_ALLOWANCES, 0)) - 
     NVL(TOTAL_DEDUCTIONS, 0)) AS CALCULATED_NET_PAY
FROM PAY_RESULTS;
```

**Benefits:**
- Users can verify calculations
- Transparent breakdown of gross/net
- Easier troubleshooting

**Applicable To:** Payslips, salary statements, earning/deduction reports

---

## 🎯 INTEGRATION EXAMPLES

### Example 1: Enhanced Element Entry Query

```sql
WITH PARAMETERS AS (
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        UPPER(NVL(:P_ELEMENT_NAME, 'ALL')) AS ELEMENT_NAME
    FROM DUAL
)
,PAY_ENTRY_ELEMENTS AS (
    SELECT
        PEEF.PERSON_ID,
        PETV.BASE_ELEMENT_NAME,
        TO_NUMBER(PEEV.SCREEN_ENTRY_VALUE) AS AMOUNT
    FROM
        PAY_ELEMENT_TYPES_VL PETV,
        PAY_ELEMENT_ENTRIES_F PEEF,
        PAY_ELEMENT_ENTRY_VALUES_F PEEV,
        PAY_INPUT_VALUES_VL PIVL,
        PARAMETERS P
    WHERE
        PETV.ELEMENT_TYPE_ID = PEEF.ELEMENT_TYPE_ID
    AND PEEF.ELEMENT_ENTRY_ID = PEEV.ELEMENT_ENTRY_ID
    AND PEEV.INPUT_VALUE_ID = PIVL.INPUT_VALUE_ID
    AND UPPER(TRIM(PIVL.NAME)) = 'AMOUNT'
    -- Use Effective Date
    AND P.EFFECTIVE_DATE BETWEEN PEEF.EFFECTIVE_START_DATE AND PEEF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PEEV.EFFECTIVE_START_DATE AND PEEV.EFFECTIVE_END_DATE
    -- Case-insensitive filter
    AND (UPPER(PETV.BASE_ELEMENT_NAME) = P.ELEMENT_NAME OR P.ELEMENT_NAME = 'ALL')
)
SELECT * FROM PAY_ENTRY_ELEMENTS;
```

### Example 2: Payslip with Component Breakdown

```sql
SELECT
    PERSON_NUMBER,
    FULL_NAME,
    PAYROLL_NAME,
    PERIOD_NAME,
    -- Earnings breakdown
    SUM(CASE WHEN ELEMENT_NAME = 'Basic Salary' THEN AMOUNT ELSE 0 END) AS BASIC,
    SUM(CASE WHEN ELEMENT_NAME = 'Housing Allowance' THEN AMOUNT ELSE 0 END) AS HOUSING,
    SUM(CASE WHEN ELEMENT_NAME = 'Transport Allowance' THEN AMOUNT ELSE 0 END) AS TRANSPORT,
    -- Calculated gross
    SUM(CASE WHEN CLASSIFICATION = 'Standard Earnings' THEN AMOUNT ELSE 0 END) AS CALC_GROSS,
    -- Deductions
    SUM(CASE WHEN CLASSIFICATION = 'Voluntary Deductions' THEN AMOUNT ELSE 0 END) AS DEDUCTIONS,
    -- Calculated net
    (SUM(CASE WHEN CLASSIFICATION = 'Standard Earnings' THEN AMOUNT ELSE 0 END) -
     SUM(CASE WHEN CLASSIFICATION = 'Voluntary Deductions' THEN AMOUNT ELSE 0 END)) AS CALC_NET_PAY
FROM PAY_RESULTS_MASTER
GROUP BY PERSON_NUMBER, FULL_NAME, PAYROLL_NAME, PERIOD_NAME;
```

---

## ⚠️ PAYROLL-SPECIFIC NOTES

### Critical Payroll Constraints (DO NOT CHANGE)
These patterns must remain as-is:
1. **ACTION_TYPE filtering:** `AND PPA.ACTION_TYPE IN ('Q', 'R')` - REQUIRED
2. **ACTION_STATUS filtering:** `AND PPA.ACTION_STATUS IN ('C')` - REQUIRED
3. **RETRO_COMPONENT exclusion:** `AND PPRA.RETRO_COMPONENT_ID IS NULL` - REQUIRED
4. **INPUT_VALUE selection:** `AND PIVF.BASE_NAME = 'Pay Value'` - REQUIRED

**The new patterns COMPLEMENT these, they do not replace them.**

---

## 📊 INTEGRATION CHECKLIST

### Priority 1: Consider Integration (Within 1 Month)
- [ ] Add PARAMETERS CTE with Effective Date to element entry queries
- [ ] Add case-insensitive filtering for element names
- [ ] Document optional tables in custom queries

### Priority 2: Enhancement (Optional)
- [ ] Add component breakdown pattern to payslip queries
- [ ] Standardize 'ALL' bypass pattern for filters

---

**END OF PAY_REPOSITORIES_UPDATE_02-01-26.md**

**Status:** Ready for Review  
**Priority:** MEDIUM  
**Next Action:** Review by Payroll Module Maintainers

**Author:** AI Assistant  
**Date:** 02-01-2026  
**Version:** 1.0
