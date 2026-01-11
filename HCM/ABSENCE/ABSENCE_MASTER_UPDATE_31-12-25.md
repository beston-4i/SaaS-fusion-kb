# HCM Absence Master Instructions - Update 31-12-25

**Module:** HCM Absence Management  
**Purpose:** New patterns and best practices from Employee Annual Leave Balance Report  
**Tag:** `#HCM #ABSENCE #MASTER #UPDATE`  
**Date:** 31-12-25

---

## 📋 NEW PATTERNS AND BEST PRACTICES

This document contains NEW or ENHANCED patterns extracted from the Employee Annual Leave Balance Report implementation.

---

## 1. EFFECTIVE DATE FILTERING (Critical Best Practice)

**Pattern:** Use parameter date instead of SYSDATE for historical/point-in-time queries

**Rule Addition:**
```
1.8 Effective Date Parameter
Rule: Always use parameter date for date-track filtering (not SYSDATE)
Pattern:
  - Define parameter: TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY'))
  - Apply to all date-tracked tables
  - Use for accrual period filtering
  - Use for leave transaction cutoff

Why: Enables "as of" date queries for accurate historical reporting
Scope: All queries requiring point-in-time accuracy
```

**Implementation:**
```sql
-- In PARAMETERS CTE
WITH PARAMETERS AS (
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE
    FROM DUAL
)

-- Apply to ALL date-tracked tables
AND P.EFFECTIVE_DATE BETWEEN TABLE.EFFECTIVE_START_DATE AND TABLE.EFFECTIVE_END_DATE

-- For accrual periods
AND APAE.ACCRUAL_PERIOD <= P.EFFECTIVE_DATE

-- For leave transactions
AND APAE.START_DATE <= P.EFFECTIVE_DATE
```

**Benefits:**
- Historical queries: Run report as of any past date
- Consistency: All data reflects same point in time
- Audit support: Recreate exact report as it appeared
- Comparison: Run for different dates to compare changes

---

## 2. CASE-INSENSITIVE PARAMETER FILTERING

**Pattern:** Use UPPER() function on parameters for case-insensitive comparison

**Rule Addition:**
```
1.9 Case-Insensitive Filtering
Rule: Apply UPPER() to text parameters for case-insensitive comparison
Pattern:
  - In PARAMETERS CTE: UPPER(NVL(:P_PARAMETER, 'ALL'))
  - In WHERE clause: UPPER(field) = P.PARAMETER OR P.PARAMETER = 'ALL'

Why: User doesn't need to match exact case, improves usability
Scope: All text-based parameter filters
```

**Implementation:**
```sql
-- In PARAMETERS CTE
WITH PARAMETERS AS (
    SELECT
        UPPER(NVL(:P_LEGAL_EMPLOYER, 'ALL')) AS LEGAL_EMPLOYER,
        UPPER(NVL(:P_ABSENCE_PLAN, 'ALL')) AS ABSENCE_PLAN,
        UPPER(NVL(:P_JOB_TITLE, 'ALL')) AS JOB_TITLE
    FROM DUAL
)

-- In WHERE clause
WHERE
    (UPPER(EA.LEGAL_EMPLOYER_NAME) = P.LEGAL_EMPLOYER OR P.LEGAL_EMPLOYER = 'ALL')
AND (UPPER(PE.PLAN_NAME) = P.ABSENCE_PLAN OR P.ABSENCE_PLAN = 'ALL')
AND (UPPER(EA.JOB_TITLE) = P.JOB_TITLE OR P.JOB_TITLE = 'ALL')
```

**Example:**
```
User enters: "abc corporation llc"
Matches: "ABC CORPORATION LLC", "Abc Corporation LLC", "ABC Corporation Llc"
```

---

## 3. SERVICE IN YEARS CALCULATION

**Pattern:** Calculate service duration using MONTHS_BETWEEN

**Rule Addition:**
```
5.7 Service Calculation
Rule: Use MONTHS_BETWEEN() / 12 for accurate service year calculation
Formula: ROUND(MONTHS_BETWEEN(EFFECTIVE_DATE, HIRE_DATE) / 12, 2)

Why: Accurate to day-level precision, handles leap years correctly
Scope: When service duration is required in reports
```

**Implementation:**
```sql
SELECT
    ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
        NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) AS SERVICE_IN_YEARS
FROM
    PER_PERIODS_OF_SERVICE PPOS,
    PARAMETERS P
```

**Considerations:**
- Use ORIGINAL_DATE_OF_HIRE if available, else DATE_START
- ROUND to 2 decimal places for readability
- Use Effective Date (not SYSDATE) for consistency

---

## 4. FULL TIME / PART TIME CLASSIFICATION

**Pattern:** Determine FT/PT status based on NORMAL_HOURS

**Rule Addition:**
```
5.8 Full Time / Part Time Classification
Rule: Use NORMAL_HOURS from assignment to classify employees
Pattern:
  - >= 40 hours = 'Full Time'
  - > 0 and < 40 hours = 'Part Time'
  - NULL or 0 = 'Not Specified'

Why: Standardized classification for reporting
Scope: When FT/PT status is required
Note: Adjust threshold based on organizational policy
```

**Implementation:**
```sql
SELECT
    CASE 
        WHEN NVL(PAAF.NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
        WHEN NVL(PAAF.NORMAL_HOURS, 0) > 0 AND NVL(PAAF.NORMAL_HOURS, 0) < 40 THEN 'Part Time'
        ELSE 'Not Specified'
    END AS FULL_TIME_PART_TIME
FROM
    PER_ALL_ASSIGNMENTS_F PAAF
```

**Customization:**
```sql
-- For 35-hour workweek
WHEN NVL(PAAF.NORMAL_HOURS, 0) >= 35 THEN 'Full Time'

-- For specific categories
CASE 
    WHEN NVL(PAAF.NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
    WHEN NVL(PAAF.NORMAL_HOURS, 0) >= 20 AND NVL(PAAF.NORMAL_HOURS, 0) < 40 THEN 'Part Time'
    WHEN NVL(PAAF.NORMAL_HOURS, 0) > 0 AND NVL(PAAF.NORMAL_HOURS, 0) < 20 THEN 'Casual'
    ELSE 'Not Specified'
END
```

---

## 5. ACCRUAL BALANCE YEAR BREAKDOWN

**Pattern:** Separate Previous Year and Current Year accruals

**Rule Addition:**
```
5.9 Accrual Year Breakdown
Rule: Split accrual balance into PY (Previous Year) and CY (Current Year) components
Pattern:
  - PY: SUM of END_BAL where ACCRUAL_PERIOD year < EFFECTIVE_DATE year
  - CY: SUM of ACCRUAL_BALANCE where ACCRUAL_PERIOD year = EFFECTIVE_DATE year

Why: Required for leave balance reporting and compliance
Scope: Leave balance reports requiring carryover tracking
```

**Implementation:**
```sql
SELECT
    APAE.PERSON_ID,
    APAE.PLAN_ID,
    -- Previous Year Carry Forward
    SUM(CASE 
        WHEN TO_CHAR(APAE.ACCRUAL_PERIOD, 'YYYY') < TO_CHAR(P.EFFECTIVE_DATE, 'YYYY')
        THEN NVL(APAE.END_BAL, 0)
        ELSE 0 
    END) AS PY_CARRY_FORWARD,
    -- Current Year Accrued
    SUM(CASE 
        WHEN TO_CHAR(APAE.ACCRUAL_PERIOD, 'YYYY') = TO_CHAR(P.EFFECTIVE_DATE, 'YYYY')
        THEN NVL(APAE.ACCRUAL_BALANCE, 0)
        ELSE 0 
    END) AS CY_ACCRUED
FROM
    ANC_PER_ACCRUAL_ENTRIES APAE,
    PARAMETERS P
WHERE
    APAE.ACCRUAL_PERIOD <= P.EFFECTIVE_DATE
GROUP BY 
    APAE.PERSON_ID, 
    APAE.PLAN_ID
```

**Key Points:**
- Use TO_CHAR(date, 'YYYY') for year comparison
- PY uses END_BAL (cumulative balance from prior year)
- CY uses ACCRUAL_BALANCE (accrued in current year)
- Both filter ACCRUAL_PERIOD <= EFFECTIVE_DATE

---

## 6. UNPAID LEAVE IDENTIFICATION

**Pattern:** Identify unpaid leave separately in transactions

**Rule Addition:**
```
5.10 Unpaid Leave Tracking
Rule: Track unpaid leave separately using absence type name pattern
Pattern: UPPER(AATFT.NAME) LIKE '%UNPAID%'

Why: Unpaid leave affects entitlement calculations differently
Scope: Leave transaction queries requiring unpaid leave separation
Alternative: Use absence category, type code, or plan type if available
```

**Implementation:**
```sql
SELECT
    APAE.PERSON_ID,
    -- Regular leave taken
    SUM(CASE 
        WHEN APAE.APPROVAL_STATUS_CD = 'APPROVED'
        AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
        THEN NVL(APAE.DURATION, 0)
    END) AS LEAVE_TAKEN,
    -- Unpaid leave taken
    SUM(CASE 
        WHEN APAE.APPROVAL_STATUS_CD = 'APPROVED'
        AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
        AND UPPER(AATFT.NAME) LIKE '%UNPAID%'
        THEN NVL(APAE.DURATION, 0)
    END) AS UNPAID_LEAVE_DAYS
FROM
    ANC_PER_ABS_ENTRIES APAE,
    ANC_ABSENCE_TYPES_F_TL AATFT
WHERE
    APAE.ABSENCE_TYPE_ID = AATFT.ABSENCE_TYPE_ID
AND AATFT.LANGUAGE = 'US'
GROUP BY APAE.PERSON_ID
```

**Alternative Patterns:**
```sql
-- By category
AND AATFT.ABSENCE_CATEGORY = 'UNPAID'

-- By type code
AND AATFT.ABSENCE_TYPE_CODE LIKE '%UNPAID%'

-- By plan type
AND AAPV.PLAN_TYPE = 'UNPAID_LEAVE'
```

---

## 7. DFF ATTRIBUTE HANDLING

**Pattern:** Separate CTE for DFF attribute mapping

**Rule Addition:**
```
5.11 DFF Attribute Mapping
Rule: Create separate CTE for DFF attribute to business field mapping
Pattern:
  1. Expose DFF attributes (ATTRIBUTE1-15) in assignment CTE
  2. Create dedicated EMP_DFF CTE for mapping
  3. Document mapping in comments

Why: Isolates DFF logic for easy maintenance and updates
Scope: When DFF fields are required in reports
```

**Implementation:**
```sql
-- Step 1: Expose in EMP_ASSIGNMENT
,EMP_ASSIGNMENT AS (
    SELECT
        PAAF.PERSON_ID,
        PAAF.ATTRIBUTE1,
        PAAF.ATTRIBUTE2,
        PAAF.ATTRIBUTE3,
        PAAF.ATTRIBUTE4,
        PAAF.ATTRIBUTE5
        -- ... other columns
    FROM PER_ALL_ASSIGNMENTS_F PAAF
)

-- Step 2: Map in EMP_DFF
,EMP_DFF AS (
    /*+ qb_name(EMP_DFF) */
    SELECT
        EA.PERSON_ID,
        -- Map DFF attributes to business fields
        -- Update based on FND_DESCR_FLEX_COLUMN_USAGES query
        EA.ATTRIBUTE1 AS CONTRACT_TYPE,
        EA.ATTRIBUTE5 AS CLIENT_JOB_TITLE,
        EA.ATTRIBUTE3 AS PROJECT_NUMBER,
        EA.ATTRIBUTE4 AS SERVICE_LINE
    FROM
        EMP_ASSIGNMENT EA
)
```

**Discovery Query:**
```sql
SELECT 
    DFC.APPLICATION_COLUMN_NAME,
    DFC.END_USER_COLUMN_NAME,
    DFC.COLUMN_SEQ_NUM
FROM FND_DESCR_FLEX_COLUMN_USAGES DFC
WHERE DFC.APPLICATION_TABLE_NAME = 'PER_ALL_ASSIGNMENTS_F'
AND DFC.ENABLED_FLAG = 'Y'
ORDER BY DFC.COLUMN_SEQ_NUM;
```

---

## 8. MULTI-PARAMETER FILTERING WITH 'ALL' SUPPORT

**Pattern:** Implement optional filters with bypass capability

**Rule Addition:**
```
5.12 Multi-Parameter Filtering
Rule: Implement independent optional filters with 'ALL' bypass
Pattern: (UPPER(field) = P.PARAMETER OR P.PARAMETER = 'ALL')

Why: Allows flexible filtering without complex NULL handling
Scope: Reports with multiple independent filter parameters
```

**Implementation:**
```sql
-- In PARAMETERS CTE: Default to 'ALL'
WITH PARAMETERS AS (
    SELECT
        UPPER(NVL(:P_LEGAL_EMPLOYER, 'ALL')) AS LEGAL_EMPLOYER,
        UPPER(NVL(:P_ABSENCE_PLAN, 'ALL')) AS ABSENCE_PLAN,
        UPPER(NVL(:P_JOB_TITLE, 'ALL')) AS JOB_TITLE,
        UPPER(NVL(:P_EMPLOYEE_TYPE, 'ALL')) AS EMPLOYEE_TYPE,
        UPPER(NVL(:P_LOCATION, 'ALL')) AS LOCATION
    FROM DUAL
)

-- In WHERE clause: Apply 'OR = ALL' pattern
WHERE
    (UPPER(EA.LEGAL_EMPLOYER_NAME) = P.LEGAL_EMPLOYER OR P.LEGAL_EMPLOYER = 'ALL')
AND (UPPER(PE.PLAN_NAME) = P.ABSENCE_PLAN OR P.ABSENCE_PLAN = 'ALL')
AND (UPPER(EA.JOB_TITLE) = P.JOB_TITLE OR P.JOB_TITLE = 'ALL')
AND (UPPER(EB.PERSON_TYPE) = P.EMPLOYEE_TYPE OR P.EMPLOYEE_TYPE = 'ALL')
AND (UPPER(EA.LOCATION_NAME) = P.LOCATION OR P.LOCATION = 'ALL')
```

**Benefits:**
- Each filter works independently
- No NULL handling needed (defaulted to 'ALL')
- Case-insensitive matching
- Clear, readable syntax

---

## 9. COMPREHENSIVE BALANCE CALCULATION

**Pattern:** Display all balance components + calculated total

**Rule Addition:**
```
5.13 Balance Calculation Pattern
Rule: Display all balance components separately PLUS calculated total
Formula: PY + CY + Adj - Enc - Taken - Expired = Balance

Why: Transparency, audit trail, verification capability
Scope: Any balance calculation query
Components:
  - PY Carry Forward (from prior years)
  - CY Accrued (current year)
  - Adjustments (manual changes)
  - Encashment (paid out)
  - Leave Taken (used)
  - Carryover Expired (lapsed)
```

**Implementation:**
```sql
SELECT
    -- Individual Components (visible)
    NVL(AB.PY_CARRY_FORWARD, 0) AS "PY Leave Carried Forward (Days)",
    NVL(AB.CY_ACCRUED, 0) AS "CY Leave Accrued (Days)",
    NVL(AB.BALANCE_ADJUSTMENT, 0) AS "Annual leave balance adjustment",
    NVL(EN.ANNUAL_LEAVE_ENCASHMENT, 0) AS "Annual leave encashment",
    NVL(LT.LEAVE_TAKEN, 0) AS "Leave Taken",
    NVL(CD.CARRYOVER_EXPIRED, 0) AS "Carryover Expired (days)",
    
    -- Calculated Balance (formula)
    (
        NVL(AB.PY_CARRY_FORWARD, 0) 
        + NVL(AB.CY_ACCRUED, 0) 
        + NVL(AB.BALANCE_ADJUSTMENT, 0) 
        - NVL(EN.ANNUAL_LEAVE_ENCASHMENT, 0) 
        - NVL(LT.LEAVE_TAKEN, 0) 
        - NVL(CD.CARRYOVER_EXPIRED, 0)
    ) AS "Calc. Leave Balance"
FROM ...
```

**Validation Query:**
```sql
-- Verify calculation
SELECT 
    PERSON_NUMBER,
    (PY + CY + ADJ - ENC - TAKEN - EXPIRED) AS MANUAL_CALC,
    CALCULATED_BALANCE AS QUERY_CALC,
    MANUAL_CALC - QUERY_CALC AS DIFFERENCE
FROM result_table
WHERE ABS(MANUAL_CALC - QUERY_CALC) > 0.01;
```

---

## 10. OPTIONAL TABLE HANDLING

**Pattern:** Gracefully handle tables that may not exist

**Rule Addition:**
```
7.8 Optional Table Handling
Rule: Use outer joins and comments for optional tables
Pattern:
  1. Document table as optional in CTE comment
  2. Use outer join in final SELECT
  3. Apply NVL() to handle NULL values
  4. Provide comment-out instructions

Why: Query works in environments with or without optional tables
Scope: Carryover, encashment, or other optional absence tables
```

**Implementation:**
```sql
-- ============================================================================
-- CARRYOVER DETAILS (Optional - comment out if table doesn't exist)
-- ============================================================================
,CARRYOVER_DETAILS AS (
    /*+ qb_name(CARRYOVER_DETAILS) */
    SELECT
        APC.PERSON_ID,
        APC.PLAN_ID,
        SUM(CASE WHEN APC.EXPIRY_DATE < P.EFFECTIVE_DATE 
            THEN NVL(APC.CARRYOVER_BALANCE, 0) ELSE 0 END) AS CARRYOVER_EXPIRED
    FROM
        ANC_PER_CARRYOVER APC,
        PARAMETERS P
    WHERE
        NVL(APC.CARRYOVER_BALANCE, 0) > 0
    GROUP BY APC.PERSON_ID, APC.PLAN_ID
)

-- In final SELECT: Use outer join and NVL
SELECT
    NVL(CD.CARRYOVER_EXPIRED, 0) AS "Carryover Expired (days)"
FROM
    EMP_BASE EB
    LEFT JOIN CARRYOVER_DETAILS CD ON EB.PERSON_ID = CD.PERSON_ID
```

**If Table Doesn't Exist:**
```sql
-- Comment out the CTE
/*
,CARRYOVER_DETAILS AS (
    ...
)
*/

-- Set to 0 in final SELECT
SELECT
    0 AS "Carryover Expired (days)"  -- Table doesn't exist
FROM ...
```

---

## 11. STANDARD FILTERS UPDATE

### Addition to Section 4: Standard Filters

#### 4.6 Multi-Parameter Filter (New)
```sql
-- Multi-parameter with 'ALL' support
AND (UPPER(field1) = P.PARAM1 OR P.PARAM1 = 'ALL')
AND (UPPER(field2) = P.PARAM2 OR P.PARAM2 = 'ALL')
AND (UPPER(field3) = P.PARAM3 OR P.PARAM3 = 'ALL')
```

#### 4.7 Effective Date Filter (New)
```sql
-- Use parameter date instead of SYSDATE
AND P.EFFECTIVE_DATE BETWEEN TABLE.EFFECTIVE_START_DATE AND TABLE.EFFECTIVE_END_DATE

-- For accrual periods
AND APAE.ACCRUAL_PERIOD <= P.EFFECTIVE_DATE

-- For transactions
AND APAE.START_DATE <= P.EFFECTIVE_DATE
```

---

## 12. COMMON PITFALLS UPDATE

### Addition to Section 7: Common Pitfalls

#### 7.9 Using SYSDATE Instead of Parameter Date
**Problem:** Using SYSDATE for date-track filtering in historical queries  
**Cause:** Default pattern uses SYSDATE, but historical queries need parameter date

**Solution:**
```sql
-- WRONG: Using SYSDATE
AND SYSDATE BETWEEN TABLE.EFFECTIVE_START_DATE AND TABLE.EFFECTIVE_END_DATE

-- CORRECT: Using parameter date
AND P.EFFECTIVE_DATE BETWEEN TABLE.EFFECTIVE_START_DATE AND TABLE.EFFECTIVE_END_DATE
```

**Impact:** Historical queries return incorrect data, audit reports are inaccurate

#### 7.10 Case-Sensitive Parameter Matching
**Problem:** Parameter filters don't work due to case mismatch  
**Cause:** Direct string comparison without UPPER()

**Solution:**
```sql
-- WRONG: Case-sensitive
WHERE EA.LEGAL_EMPLOYER_NAME = P.LEGAL_EMPLOYER

-- CORRECT: Case-insensitive
WHERE UPPER(EA.LEGAL_EMPLOYER_NAME) = UPPER(P.LEGAL_EMPLOYER)
-- Or even better: UPPER in PARAMETERS CTE
WHERE UPPER(EA.LEGAL_EMPLOYER_NAME) = P.LEGAL_EMPLOYER
```

#### 7.11 Missing NVL() in Balance Calculations
**Problem:** Balance calculation returns NULL due to NULL arithmetic  
**Cause:** Any NULL component makes entire calculation NULL

**Solution:**
```sql
-- WRONG: NULL-prone
SELECT (PY + CY + ADJ - ENC - TAKEN - EXPIRED) AS BALANCE

-- CORRECT: NVL-protected
SELECT (
    NVL(PY, 0) + NVL(CY, 0) + NVL(ADJ, 0) 
    - NVL(ENC, 0) - NVL(TAKEN, 0) - NVL(EXPIRED, 0)
) AS BALANCE
```

---

## 13. REPORT-SPECIFIC RULES UPDATE

### Addition to Section 6: Report-Specific Rules

#### 6.4 Comprehensive Leave Balance Report (New)
**Purpose:** Complete balance report with all components  
**Key Columns:**
- All employee and organizational details
- Service in years calculation
- FT/PT classification
- Balance component breakdown (PY, CY, Adj, Enc, Taken, Expired)
- Calculated total balance

**Filters:**
- Effective Date (mandatory)
- Multiple optional filters (Legal Employer, Plan, Job, Type, Location)
- All filters case-insensitive with 'ALL' bypass

**Calculations:**
- Service in Years: `ROUND(MONTHS_BETWEEN(EFFECTIVE_DATE, HIRE_DATE) / 12, 2)`
- FT/PT: Based on NORMAL_HOURS >= 40
- Balance: `PY + CY + Adj - Enc - Taken - Expired`

**Special Handling:**
- DFF attribute mapping
- Optional tables (carryover, encashment)
- Unpaid leave tracking
- PY/CY year breakdown

---

## 14. ADVANCED PATTERNS UPDATE

### Addition to Section 9: Advanced Patterns

#### 9.7 Service Duration Calculation (New)
**Purpose:** Calculate accurate service years for entitlement  
**Pattern:**
```sql
ROUND(MONTHS_BETWEEN(EFFECTIVE_DATE, 
    NVL(ORIGINAL_DATE_OF_HIRE, DATE_START)) / 12, 2) AS SERVICE_IN_YEARS
```

#### 9.8 Year-Based Accrual Breakdown (New)
**Purpose:** Separate PY and CY accruals  
**Pattern:**
```sql
-- Previous Year
SUM(CASE 
    WHEN TO_CHAR(ACCRUAL_PERIOD, 'YYYY') < TO_CHAR(EFFECTIVE_DATE, 'YYYY')
    THEN NVL(END_BAL, 0)
    ELSE 0 
END) AS PY_CARRY_FORWARD

-- Current Year
SUM(CASE 
    WHEN TO_CHAR(ACCRUAL_PERIOD, 'YYYY') = TO_CHAR(EFFECTIVE_DATE, 'YYYY')
    THEN NVL(ACCRUAL_BALANCE, 0)
    ELSE 0 
END) AS CY_ACCRUED
```

#### 9.9 FT/PT Classification Logic (New)
**Purpose:** Standardized employee time classification  
**Pattern:**
```sql
CASE 
    WHEN NVL(NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
    WHEN NVL(NORMAL_HOURS, 0) > 0 AND NVL(NORMAL_HOURS, 0) < 40 THEN 'Part Time'
    ELSE 'Not Specified'
END AS FULL_TIME_PART_TIME
```

---

## ✅ UPDATED VALIDATION CHECKLIST

Addition to existing checklist:

- [ ] Effective Date parameter used (not SYSDATE) for date-track filtering
- [ ] All text parameters use UPPER() for case-insensitive comparison
- [ ] Service calculation uses MONTHS_BETWEEN() / 12 pattern
- [ ] FT/PT classification based on NORMAL_HOURS (threshold documented)
- [ ] Accrual balance split by year (PY vs CY) if required
- [ ] Unpaid leave identified separately if required
- [ ] Multi-parameter filters use 'OR = ALL' pattern
- [ ] Balance calculation displays all components + total
- [ ] All components use NVL() to prevent NULL arithmetic
- [ ] Optional tables handled with outer joins and comments
- [ ] DFF attributes mapped in separate CTE with discovery query documented

---

## 📊 UPDATED PATTERN PRIORITY

**Critical Patterns** (Must Use):
1. Effective Date filtering (not SYSDATE)
2. Date-track filtering on all `_F` tables
3. LANGUAGE = 'US' on all `_TL` tables
4. Active assignment filters
5. NVL() in calculations

**High Priority Patterns** (Strongly Recommended):
1. Case-insensitive parameter filtering
2. Multi-parameter 'ALL' bypass
3. Service calculation standard formula
4. Balance component breakdown

**Medium Priority Patterns** (Use When Applicable):
1. FT/PT classification
2. PY/CY accrual breakdown
3. Unpaid leave tracking
4. DFF attribute mapping
5. Optional table handling

---

## 📝 INTEGRATION NOTES

### Impact on Existing Documentation

1. **ABSENCE_MASTER.md**: Add new patterns as sections 1.8-1.9 and 5.7-5.13
2. **ABSENCE_REPOSITORIES.md**: Add enhanced CTEs (see ABSENCE_REPOSITORIES_UPDATE)
3. **ABSENCE_TEMPLATES.md**: Add Template 7 (see ABSENCE_TEMPLATES_UPDATE)

### Backward Compatibility

- All new patterns are additive
- Existing patterns remain valid
- Enhanced CTEs maintain same interface
- New columns added, existing columns unchanged

### Migration Path

1. Review new patterns
2. Identify queries that would benefit
3. Apply patterns incrementally
4. Test thoroughly before production
5. Update documentation

---

**END OF ABSENCE_MASTER_UPDATE_31-12-25.md**

**Status:** Ready for Integration  
**Next Action:** Review and merge into main ABSENCE_MASTER.md  
**Priority:** High - Contains critical best practices





