# HCM Absence Repositories - Update 31-12-25

**Module:** HCM Absence Management  
**Purpose:** New CTE patterns from Employee Annual Leave Balance Report  
**Tag:** `#HCM #ABSENCE #REPOSITORIES #UPDATE`  
**Date:** 31-12-25

---

## 📋 NEW REPOSITORY ADDITIONS

This document contains NEW or ENHANCED CTEs extracted from the Employee Annual Leave Balance Report implementation.

---

## 1. PARAMETERS CTE (Enhanced with UPPER for Case-Insensitive Filtering)

**Purpose:** Parameter handling with automatic case normalization  
**Usage:** Use when parameters need case-insensitive comparison

```sql
WITH PARAMETERS AS (
    /*+ qb_name(PARAMETERS) */
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        UPPER(NVL(:P_LEGAL_EMPLOYER, 'ALL')) AS LEGAL_EMPLOYER,
        UPPER(NVL(:P_ABSENCE_PLAN, 'ALL')) AS ABSENCE_PLAN,
        UPPER(NVL(:P_JOB_TITLE, 'ALL')) AS JOB_TITLE,
        UPPER(NVL(:P_EMPLOYEE_TYPE, 'ALL')) AS EMPLOYEE_TYPE,
        UPPER(NVL(:P_LOCATION, 'ALL')) AS LOCATION
    FROM DUAL
)
```

**Key Features:**
- `UPPER()` function on all text parameters for case-insensitive comparison
- `NVL()` with 'ALL' default for optional parameters
- `TRUNC(TO_DATE())` for date parameter with explicit format
- All parameters accessible throughout query via cross join

**Benefits:**
- User doesn't need to match exact case
- Consistent handling of NULL parameters
- Single source of truth for parameter values

---

## 2. EMP_BASE WITH SERVICE CALCULATION (Enhanced)

**Purpose:** Employee base with service years calculation  
**Usage:** When service duration is required in reports

```sql
,EMP_BASE AS (
    /*+ qb_name(EMP_BASE) */
    SELECT
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER,
        PPNF.FULL_NAME,
        PPNF.DISPLAY_NAME,
        PPTTL.USER_PERSON_TYPE AS PERSON_TYPE,
        PPOS.PERIOD_OF_SERVICE_ID,
        TO_CHAR(NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START), 'DD-MM-YYYY') AS HIRE_DATE,
        NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START) AS HIRE_DATE_RAW,
        PPOS.DATE_START,
        PPOS.ACTUAL_TERMINATION_DATE,
        -- Calculate Service in Years as of Effective Date
        ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) AS SERVICE_IN_YEARS
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_PERSON_TYPES_TL PPTTL,
        PER_PERIODS_OF_SERVICE PPOS,
        PARAMETERS P
    WHERE
        PAPF.PERSON_ID = PPNF.PERSON_ID
    AND PAPF.PERSON_ID = PAAF.PERSON_ID
    AND PAAF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID(+)
    AND PAPF.PERSON_ID = PPOS.PERSON_ID
    AND PPNF.NAME_TYPE = 'GLOBAL'
    AND PPTTL.LANGUAGE(+) = 'US'
    AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PAAF.PRIMARY_FLAG = 'Y'
    AND PAAF.ASSIGNMENT_TYPE = 'E'
    -- Date-track filtering using P.EFFECTIVE_DATE (NOT SYSDATE)
    AND P.EFFECTIVE_DATE BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PPOS.DATE_START 
        AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
)
```

**Key Features:**
- **Service calculation**: `ROUND(MONTHS_BETWEEN(...) / 12, 2)` gives years of service
- **Dual date formats**: Both formatted (DD-MM-YYYY) and raw date for different uses
- **Effective Date filtering**: Uses parameter date instead of SYSDATE
- **Termination handling**: NVL with far future date for active employees

**Service Calculation Formula:**
```sql
ROUND(MONTHS_BETWEEN(EFFECTIVE_DATE, HIRE_DATE) / 12, 2)
```

---

## 3. EMP_ASSIGNMENT WITH FULL TIME/PART TIME CLASSIFICATION (New)

**Purpose:** Assignment details with employee classification  
**Usage:** When FT/PT status is required

```sql
,EMP_ASSIGNMENT AS (
    /*+ qb_name(EMP_ASSIGNMENT) */
    SELECT
        PAAF.PERSON_ID,
        PAAF.ASSIGNMENT_ID,
        PAAF.ASSIGNMENT_NUMBER,
        PAAF.ORGANIZATION_ID,
        PAAF.BUSINESS_UNIT_ID,
        PAAF.LOCATION_ID,
        PAAF.GRADE_ID,
        PAAF.ASSIGNMENT_CATEGORY,
        PAAF.POSITION_ID,
        PAAF.JOB_ID,
        PAAF.LEGAL_ENTITY_ID,
        PAAF.NORMAL_HOURS,
        PAAF.FREQUENCY,
        -- Job and Position
        PJFV.NAME AS JOB_TITLE,
        HAPL.NAME AS POSITION_TITLE,
        -- Organization
        PD.NAME AS DEPARTMENT_NAME,
        HAOULE.NAME AS LEGAL_EMPLOYER_NAME,
        HAOUBU.NAME AS BUSINESS_UNIT_NAME,
        HLOCVL.LOCATION_NAME,
        -- Grade
        PGFV.NAME AS GRADE_NAME,
        -- Worker Type
        PAAF.ASSIGNMENT_CATEGORY AS WORKER_TYPE,
        -- Full Time / Part Time determination
        CASE 
            WHEN NVL(PAAF.NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
            WHEN NVL(PAAF.NORMAL_HOURS, 0) > 0 AND NVL(PAAF.NORMAL_HOURS, 0) < 40 THEN 'Part Time'
            ELSE 'Not Specified'
        END AS FULL_TIME_PART_TIME,
        -- DFF Attributes (for reference)
        PAAF.ATTRIBUTE1 AS ATTR1,
        PAAF.ATTRIBUTE2 AS ATTR2,
        PAAF.ATTRIBUTE3 AS ATTR3,
        PAAF.ATTRIBUTE4 AS ATTR4,
        PAAF.ATTRIBUTE5 AS ATTR5
    FROM
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_JOBS_F_VL PJFV,
        HR_ALL_POSITIONS_F_TL HAPL,
        PER_DEPARTMENTS PD,
        HR_ALL_ORGANIZATION_UNITS HAOULE,
        HR_ALL_ORGANIZATION_UNITS HAOUBU,
        HR_LOCATIONS_ALL_F_VL HLOCVL,
        PER_GRADES_F_VL PGFV,
        PARAMETERS P
    WHERE
        PAAF.JOB_ID = PJFV.JOB_ID(+)
    AND PAAF.POSITION_ID = HAPL.POSITION_ID(+)
    AND PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
    AND PAAF.LEGAL_ENTITY_ID = HAOULE.ORGANIZATION_ID(+)
    AND PAAF.BUSINESS_UNIT_ID = HAOUBU.ORGANIZATION_ID(+)
    AND PAAF.LOCATION_ID = HLOCVL.LOCATION_ID(+)
    AND PAAF.GRADE_ID = PGFV.GRADE_ID(+)
    AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PAAF.PRIMARY_FLAG = 'Y'
    AND PAAF.ASSIGNMENT_TYPE = 'E'
    AND HAPL.LANGUAGE(+) = 'US'
    AND PGFV.LANGUAGE(+) = 'US'
    AND P.EFFECTIVE_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PJFV.EFFECTIVE_START_DATE(+) AND PJFV.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN HAPL.EFFECTIVE_START_DATE(+) AND HAPL.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN PD.EFFECTIVE_START_DATE(+) AND PD.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN HLOCVL.EFFECTIVE_START_DATE(+) AND HLOCVL.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN PGFV.EFFECTIVE_START_DATE(+) AND PGFV.EFFECTIVE_END_DATE(+)
)
```

**Key Features:**
- **FT/PT Classification**: Based on NORMAL_HOURS (>= 40 = Full Time, < 40 = Part Time)
- **Comprehensive Org Data**: Legal employer, business unit, location, department
- **DFF Attributes**: Exposed for further processing
- **Multiple Org Units**: Separate aliases for different organization types

**FT/PT Logic:**
```sql
CASE 
    WHEN NVL(NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
    WHEN NVL(NORMAL_HOURS, 0) > 0 AND NVL(NORMAL_HOURS, 0) < 40 THEN 'Part Time'
    ELSE 'Not Specified'
END
```

---

## 4. DFF ATTRIBUTE HANDLING CTE (New Pattern)

**Purpose:** Extract and map Descriptive Flexfield attributes  
**Usage:** When DFF fields need to be mapped to business fields

```sql
,EMP_DFF AS (
    /*+ qb_name(EMP_DFF) */
    SELECT
        EA.PERSON_ID,
        EA.ASSIGNMENT_ID,
        -- Map DFF attributes to business fields
        -- Update these mappings based on FND_DESCR_FLEX_COLUMN_USAGES query
        EA.ATTR1 AS CONTRACT_TYPE,
        EA.ATTR5 AS CLIENT_JOB_TITLE,
        EA.ATTR3 AS PROJECT_NUMBER,
        EA.ATTR4 AS SERVICE_LINE
    FROM
        EMP_ASSIGNMENT EA
)
```

**Key Features:**
- **Separate CTE**: Isolates DFF logic for easy maintenance
- **Business Field Mapping**: Maps technical attributes to business names
- **Documentation**: Comments indicate where to find correct mappings

**Discovery Query for DFF Mapping:**
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

## 5. ACCRUAL_BALANCE WITH YEAR BREAKDOWN (Enhanced)

**Purpose:** Accrual balance with Previous Year vs Current Year breakdown  
**Usage:** When PY carryover and CY accrued need to be separated

```sql
,ACCRUAL_BALANCE AS (
    /*+ qb_name(ACCRUAL_BALANCE) */
    SELECT
        APAE.PERSON_ID,
        APAE.PLAN_ID,
        APAE.PRD_OF_SVC_ID,
        -- Previous Year Carry Forward (balance from prior years)
        SUM(CASE 
            WHEN TO_CHAR(APAE.ACCRUAL_PERIOD, 'YYYY') < TO_CHAR(P.EFFECTIVE_DATE, 'YYYY')
            THEN NVL(APAE.END_BAL, 0)
            ELSE 0 
        END) AS PY_CARRY_FORWARD,
        -- Current Year Accrued (accrued in current year)
        SUM(CASE 
            WHEN TO_CHAR(APAE.ACCRUAL_PERIOD, 'YYYY') = TO_CHAR(P.EFFECTIVE_DATE, 'YYYY')
            THEN NVL(APAE.ACCRUAL_BALANCE, 0)
            ELSE 0 
        END) AS CY_ACCRUED,
        -- Current Balance (latest period balance)
        MAX(CASE 
            WHEN APAE.ACCRUAL_PERIOD = (
                SELECT MAX(APA.ACCRUAL_PERIOD)
                FROM ANC_PER_ACCRUAL_ENTRIES APA
                WHERE APA.PERSON_ID = APAE.PERSON_ID
                AND APA.PLAN_ID = APAE.PLAN_ID
                AND APA.PRD_OF_SVC_ID = APAE.PRD_OF_SVC_ID
                AND APA.ACCRUAL_PERIOD <= P.EFFECTIVE_DATE
            )
            THEN NVL(APAE.END_BAL, 0)
            ELSE 0
        END) AS CURRENT_BALANCE,
        -- Adjustments (sum of manual adjustments)
        SUM(NVL(APAE.ADJUSTMENT, 0)) AS BALANCE_ADJUSTMENT,
        -- Entitlement Override (latest value)
        MAX(APAE.ENTITLEMENT_OVERRIDE) AS ENTITLEMENT_OVERRIDE,
        -- Annual Entitlement (accrual rate per annum)
        MAX(APAE.ACCRUAL_RATE) AS ANNUAL_ENTITLEMENT
    FROM
        ANC_PER_ACCRUAL_ENTRIES APAE,
        PARAMETERS P
    WHERE
        APAE.ACCRUAL_PERIOD <= P.EFFECTIVE_DATE
    GROUP BY 
        APAE.PERSON_ID, 
        APAE.PLAN_ID, 
        APAE.PRD_OF_SVC_ID,
        P.EFFECTIVE_DATE
)
```

**Key Features:**
- **Year Comparison**: `TO_CHAR(ACCRUAL_PERIOD, 'YYYY')` vs `TO_CHAR(EFFECTIVE_DATE, 'YYYY')`
- **PY Logic**: Sum of END_BAL from years prior to effective year
- **CY Logic**: Sum of ACCRUAL_BALANCE from current year only
- **Latest Balance**: Uses correlated subquery with MAX(ACCRUAL_PERIOD)

**Balance Component Breakdown:**
```
PY Carry Forward = Sum of END_BAL where year < current year
CY Accrued       = Sum of ACCRUAL_BALANCE where year = current year
Current Balance  = END_BAL from latest accrual period
```

---

## 6. LEAVE_TRANSACTIONS WITH UNPAID IDENTIFICATION (Enhanced)

**Purpose:** Leave transactions with separate unpaid leave tracking  
**Usage:** When unpaid leave needs to be reported separately

```sql
,LEAVE_TRANSACTIONS AS (
    /*+ qb_name(LEAVE_TRANSACTIONS) */
    SELECT
        APAE.PERSON_ID,
        APAE.PLAN_ID,
        -- Leave Applied (all non-withdrawn leaves)
        SUM(CASE 
            WHEN APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
            AND APAE.START_DATE <= P.EFFECTIVE_DATE
            THEN NVL(APAE.DURATION, 0)
            ELSE 0 
        END) AS LEAVE_APPLIED,
        -- Leave Taken (approved leaves only)
        SUM(CASE 
            WHEN APAE.APPROVAL_STATUS_CD IN ('APPROVED')
            AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
            AND APAE.START_DATE <= P.EFFECTIVE_DATE
            THEN NVL(APAE.DURATION, 0)
            ELSE 0 
        END) AS LEAVE_TAKEN,
        -- Unpaid Leave Days (identify by absence type name containing 'UNPAID')
        SUM(CASE 
            WHEN APAE.APPROVAL_STATUS_CD IN ('APPROVED')
            AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
            AND APAE.START_DATE <= P.EFFECTIVE_DATE
            AND UPPER(AATFT.NAME) LIKE '%UNPAID%'
            THEN NVL(APAE.DURATION, 0)
            ELSE 0 
        END) AS UNPAID_LEAVE_DAYS
    FROM
        ANC_PER_ABS_ENTRIES APAE,
        ANC_ABSENCE_TYPES_F_TL AATFT,
        PARAMETERS P
    WHERE
        APAE.ABSENCE_TYPE_ID = AATFT.ABSENCE_TYPE_ID
    AND AATFT.LANGUAGE = 'US'
    AND P.EFFECTIVE_DATE BETWEEN AATFT.EFFECTIVE_START_DATE AND AATFT.EFFECTIVE_END_DATE
    GROUP BY 
        APAE.PERSON_ID, 
        APAE.PLAN_ID,
        P.EFFECTIVE_DATE
)
```

**Key Features:**
- **Three Metrics**: Applied, Taken (approved), and Unpaid separately
- **Unpaid Identification**: `UPPER(NAME) LIKE '%UNPAID%'` for flexibility
- **Date Cutoff**: `START_DATE <= EFFECTIVE_DATE` for as-of-date accuracy
- **Status Filtering**: Proper approval and absence status handling

**Unpaid Leave Pattern:**
```sql
AND UPPER(AATFT.NAME) LIKE '%UNPAID%'
```
Alternative patterns if tracked differently:
```sql
-- By absence category
AND AATFT.ABSENCE_CATEGORY = 'UNPAID'

-- By absence type code
AND AATFT.ABSENCE_TYPE_CODE LIKE '%UNPAID%'

-- By plan type
AND AAPV.PLAN_TYPE = 'UNPAID_LEAVE'
```

---

## 7. MULTI-PARAMETER FILTERING PATTERN (New)

**Purpose:** Implement multiple optional filters with 'ALL' support  
**Usage:** When report needs multiple independent filters

```sql
WHERE
    EB.PERSON_ID = EA.PERSON_ID
-- ... other joins ...

-- Parameter Filters with 'ALL' support
AND (UPPER(EA.LEGAL_EMPLOYER_NAME) = P.LEGAL_EMPLOYER OR P.LEGAL_EMPLOYER = 'ALL')
AND (UPPER(PE.PLAN_NAME) = P.ABSENCE_PLAN OR P.ABSENCE_PLAN = 'ALL')
AND (UPPER(EA.JOB_TITLE) = P.JOB_TITLE OR P.JOB_TITLE = 'ALL')
AND (UPPER(EB.PERSON_TYPE) = P.EMPLOYEE_TYPE OR P.EMPLOYEE_TYPE = 'ALL')
AND (UPPER(EA.LOCATION_NAME) = P.LOCATION OR P.LOCATION = 'ALL')

ORDER BY 
    TO_NUMBER(EB.PERSON_NUMBER),
    PE.PLAN_NAME
```

**Key Features:**
- **Case-Insensitive**: UPPER() on both sides of comparison
- **'ALL' Bypass**: `OR PARAMETER = 'ALL'` allows filter bypass
- **Independent Filters**: Each filter works independently
- **No NULL Issues**: Parameters defaulted to 'ALL' in PARAMETERS CTE

**Pattern Template:**
```sql
AND (UPPER(field_name) = P.PARAMETER_NAME OR P.PARAMETER_NAME = 'ALL')
```

---

## 8. COMPREHENSIVE BALANCE CALCULATION PATTERN (New)

**Purpose:** Complete leave balance calculation with all components  
**Formula:** `PY + CY + Adj - Enc - Taken - Expired`

```sql
SELECT
    -- All other columns...
    
    -- Individual Components
    NVL(AB.PY_CARRY_FORWARD, 0) AS "PY Leave Carried Forward (Days)",
    NVL(AB.CY_ACCRUED, 0) AS "CY Leave Accrued (Days)",
    NVL(AB.BALANCE_ADJUSTMENT, 0) AS "Annual leave balance adjustment",
    NVL(EN.ANNUAL_LEAVE_ENCASHMENT, 0) AS "Annual leave encashment",
    NVL(LT.LEAVE_TAKEN, 0) AS "Leave Taken",
    NVL(CD.CARRYOVER_EXPIRED, 0) AS "Carryover Expired (days)",
    
    -- Calculated Balance
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

**Key Features:**
- **All Components Visible**: Each component shown separately
- **NVL() Protection**: All values default to 0 to prevent NULL arithmetic
- **Clear Formula**: Parentheses group additions and subtractions
- **Audit Trail**: Users can verify calculation by seeing all components

**Validation Query:**
```sql
-- Verify calculation is correct
SELECT 
    PERSON_NUMBER,
    PY + CY + ADJ - ENC - TAKEN - EXPIRED AS MANUAL_CALC,
    CALCULATED_BALANCE AS QUERY_CALC,
    MANUAL_CALC - QUERY_CALC AS DIFFERENCE
FROM result_table
WHERE ABS(MANUAL_CALC - QUERY_CALC) > 0.01;
```

---

## 9. EFFECTIVE DATE FILTERING PATTERN (Critical Best Practice)

**Purpose:** Use parameter date instead of SYSDATE for historical/point-in-time queries  
**Why Critical:** Allows "as of" date queries for accurate historical reporting

**Standard Pattern:**
```sql
-- In PARAMETERS CTE
WITH PARAMETERS AS (
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE
    FROM DUAL
)

-- In every date-tracked table join
AND P.EFFECTIVE_DATE BETWEEN TABLE.EFFECTIVE_START_DATE AND TABLE.EFFECTIVE_END_DATE

-- For accrual periods
AND APAE.ACCRUAL_PERIOD <= P.EFFECTIVE_DATE

-- For leave transactions
AND APAE.START_DATE <= P.EFFECTIVE_DATE

-- For period of service
AND P.EFFECTIVE_DATE BETWEEN PPOS.DATE_START 
    AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
```

**Benefits:**
- **Historical Queries**: Can run report as of any past date
- **Consistency**: All data reflects same point in time
- **Audit Support**: Recreate exact report as it appeared on specific date
- **Comparison**: Run for different dates to compare changes

**Common Mistake to Avoid:**
```sql
-- WRONG: Using SYSDATE
AND SYSDATE BETWEEN TABLE.EFFECTIVE_START_DATE AND TABLE.EFFECTIVE_END_DATE

-- CORRECT: Using parameter date
AND P.EFFECTIVE_DATE BETWEEN TABLE.EFFECTIVE_START_DATE AND TABLE.EFFECTIVE_END_DATE
```

---

## 📊 USAGE EXAMPLES

### Example 1: Complete Balance Report Query Structure
```sql
WITH PARAMETERS AS (...),
     EMP_BASE AS (...),
     EMP_ASSIGNMENT AS (...),
     EMP_DFF AS (...),
     PLAN_ENROLLMENT AS (...),
     ACCRUAL_BALANCE AS (...),
     CARRYOVER_DETAILS AS (...),
     LEAVE_TRANSACTIONS AS (...),
     ENCASHMENT_DETAILS AS (...)
SELECT
    -- Employee identification
    EB.PERSON_NUMBER,
    EB.FULL_NAME,
    -- Balance components
    AB.PY_CARRY_FORWARD,
    AB.CY_ACCRUED,
    AB.BALANCE_ADJUSTMENT,
    LT.LEAVE_TAKEN,
    EN.ANNUAL_LEAVE_ENCASHMENT,
    CD.CARRYOVER_EXPIRED,
    -- Calculated balance
    (AB.PY_CARRY_FORWARD + AB.CY_ACCRUED + AB.BALANCE_ADJUSTMENT 
     - LT.LEAVE_TAKEN - EN.ANNUAL_LEAVE_ENCASHMENT - CD.CARRYOVER_EXPIRED) AS BALANCE
FROM EMP_BASE EB
    JOIN EMP_ASSIGNMENT EA ON EB.PERSON_ID = EA.PERSON_ID
    LEFT JOIN PLAN_ENROLLMENT PE ON EB.PERSON_ID = PE.PERSON_ID
    LEFT JOIN ACCRUAL_BALANCE AB ON EB.PERSON_ID = AB.PERSON_ID AND PE.PLAN_ID = AB.PLAN_ID
    LEFT JOIN LEAVE_TRANSACTIONS LT ON EB.PERSON_ID = LT.PERSON_ID AND PE.PLAN_ID = LT.PLAN_ID
    LEFT JOIN CARRYOVER_DETAILS CD ON EB.PERSON_ID = CD.PERSON_ID AND PE.PLAN_ID = CD.PLAN_ID
    LEFT JOIN ENCASHMENT_DETAILS EN ON EB.PERSON_ID = EN.PERSON_ID AND PE.PLAN_ID = EN.PLAN_ID;
```

---

## ✅ VALIDATION CHECKLIST

Before using these new patterns:

- [ ] PARAMETERS CTE uses UPPER() for case-insensitive comparison
- [ ] Date filtering uses P.EFFECTIVE_DATE (not SYSDATE)
- [ ] Service calculation uses MONTHS_BETWEEN() / 12
- [ ] FT/PT classification based on NORMAL_HOURS
- [ ] DFF attributes mapped to business fields
- [ ] Accrual balance split by year (PY vs CY)
- [ ] Unpaid leave identified correctly
- [ ] Multi-parameter filters use 'OR = ALL' pattern
- [ ] Balance calculation includes all components
- [ ] All NVL() applied to prevent NULL arithmetic

---

## 📝 INTEGRATION NOTES

### Merging with Existing Repositories

1. **Enhanced CTEs**: Replace existing versions with enhanced versions
2. **New CTEs**: Add to repository with appropriate documentation
3. **Patterns**: Apply to existing queries where applicable
4. **Testing**: Validate all changes with known test data

### Backward Compatibility

- Enhanced CTEs maintain same column names
- New columns added, existing columns unchanged
- Optional components (carryover, encashment) use outer joins

---

**END OF ABSENCE_REPOSITORIES_UPDATE_31-12-25.md**

**Status:** Ready for Integration  
**Next Action:** Review and merge into main ABSENCE_REPOSITORIES.md





