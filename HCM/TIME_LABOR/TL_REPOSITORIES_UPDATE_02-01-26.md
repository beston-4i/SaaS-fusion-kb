# Time & Labor Repository Patterns - UPDATE 02-01-26

**Module:** Time & Labor  
**Update Type:** Cross-Module Knowledge Integration  
**Source:** Employee Annual Leave Balance Report  
**Date:** 02-01-26  
**Applicability:** MEDIUM - 3 patterns applicable

---

## 📋 EXECUTIVE SUMMARY

This update adds Time & Labor-applicable patterns from the Employee Annual Leave Balance Report. Focus on effective date filtering for timecard history and service calculations for time-off eligibility.

**Impact:** Medium - Enhances historical timecard queries  
**Priority:** Medium (integrate within 1 month)  
**Scope:** Timecard history and time-off balance queries

---

## 🆕 APPLICABLE PATTERNS FOR TIME & LABOR

### Pattern 1: Effective Date Filtering for Timecard History

**What's New:** Parameter-based date filtering for "as of" timecard queries

**Pattern:**
```sql
WITH PARAMETERS AS (
    /*+ qb_name(PARAMETERS) */
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        TO_DATE(:P_START_DATE, 'DD-MON-YYYY') AS START_DATE,
        TO_DATE(:P_END_DATE, 'DD-MON-YYYY') AS END_DATE
    FROM DUAL
)
,TL_TIMECARD_HISTORY AS (
    /*+ qb_name(TL_TC_HIST) MATERIALIZE */
    SELECT
        HTC.PERSON_ID,
        HTC.TIME_ENTRY_ID,
        HTC.TIME_BUILDING_BLOCK_ID,
        TO_CHAR(HTC.START_TIME, 'DD-MM-YYYY HH24:MI') AS START_TIME,
        TO_CHAR(HTC.STOP_TIME, 'DD-MM-YYYY HH24:MI') AS STOP_TIME,
        HTC.MEASURE AS HOURS,
        HTE.TIME_ENTRY_TYPE_NAME
    FROM
        HWM_TIME_CARDS HTC,
        HWM_TIME_ENTRY_TYPES HTE,
        PARAMETERS P
    WHERE
        HTC.TIME_ENTRY_TYPE_ID = HTE.TIME_ENTRY_TYPE_ID(+)
    -- Timecard entries within date range
    AND TRUNC(HTC.START_TIME) BETWEEN P.START_DATE AND P.END_DATE
    -- Use Effective Date for date-tracked lookups
    AND P.EFFECTIVE_DATE BETWEEN HTE.EFFECTIVE_START_DATE(+) AND HTE.EFFECTIVE_END_DATE(+)
)
```

**Benefits:**
- Query timecards for any historical period
- Historical hours analysis
- Audit compliance for past timecards

**Applicable To:** Timecard history, hours worked analysis queries

---

### Pattern 2: Service Calculation for Time-Off Eligibility

**What's New:** Service in Years for time-off accrual eligibility

**Pattern:**
```sql
,EMP_SERVICE AS (
    SELECT
        PAPF.PERSON_ID,
        ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
              NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) AS SERVICE_IN_YEARS
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERIODS_OF_SERVICE PPOS,
        PARAMETERS P
    WHERE
        PAPF.PERSON_ID = PPOS.PERSON_ID
    AND P.EFFECTIVE_DATE BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PPOS.DATE_START 
        AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
)

-- Use in time-off eligibility
SELECT
    ES.PERSON_ID,
    ES.SERVICE_IN_YEARS,
    CASE 
        WHEN ES.SERVICE_IN_YEARS < 1 THEN 10  -- 10 days for first year
        WHEN ES.SERVICE_IN_YEARS BETWEEN 1 AND 5 THEN 15  -- 15 days for 1-5 years
        WHEN ES.SERVICE_IN_YEARS BETWEEN 5 AND 10 THEN 20  -- 20 days for 5-10 years
        ELSE 25  -- 25 days for 10+ years
    END AS ANNUAL_LEAVE_ENTITLEMENT
FROM EMP_SERVICE ES;
```

**Benefits:**
- Service-based time-off eligibility
- Accrual rate determination
- Time-off policy compliance

**Applicable To:** Time-off balance, accrual calculation queries

---

### Pattern 3: Case-Insensitive Parameter Filtering

**What's New:** UPPER() for time entry type and project filtering

**Pattern:**
```sql
WITH PARAMETERS AS (
    SELECT
        UPPER(NVL(:P_TIME_ENTRY_TYPE, 'ALL')) AS TIME_ENTRY_TYPE,
        UPPER(NVL(:P_PROJECT_NAME, 'ALL')) AS PROJECT_NAME,
        UPPER(NVL(:P_TASK_NAME, 'ALL')) AS TASK_NAME
    FROM DUAL
)

-- Apply in WHERE
AND (UPPER(HTE.TIME_ENTRY_TYPE_NAME) = P.TIME_ENTRY_TYPE OR P.TIME_ENTRY_TYPE = 'ALL')
AND (UPPER(PROJ.PROJECT_NAME) = P.PROJECT_NAME OR P.PROJECT_NAME = 'ALL')
AND (UPPER(TASK.TASK_NAME) = P.TASK_NAME OR P.TASK_NAME = 'ALL')
```

**Benefits:**
- Flexible filtering (users can enter "regular" or "Regular" or "REGULAR")
- 'ALL' bypass for showing all records
- Consistent with other HCM modules

**Applicable To:** All time & labor queries with filters

---

## 🎯 INTEGRATION EXAMPLES

### Example 1: Historical Hours Query

```sql
WITH PARAMETERS AS (
    SELECT 
        TRUNC(TO_DATE('01-JAN-2024', 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        TO_DATE('01-JAN-2024', 'DD-MON-YYYY') AS START_DATE,
        TO_DATE('31-JAN-2024', 'DD-MON-YYYY') AS END_DATE
    FROM DUAL
)
,TL_TIMECARD_HISTORY AS (
    -- Enhanced pattern with Effective Date
    ...
)
SELECT
    PERSON_NUMBER,
    FULL_NAME,
    TIME_ENTRY_TYPE_NAME,
    SUM(HOURS) AS TOTAL_HOURS
FROM TL_TIMECARD_HISTORY
GROUP BY PERSON_NUMBER, FULL_NAME, TIME_ENTRY_TYPE_NAME
ORDER BY PERSON_NUMBER;

-- Result: Shows timecard hours for January 2024
```

### Example 2: Service-Based Time-Off Eligibility

```sql
SELECT
    PERSON_NUMBER,
    FULL_NAME,
    HIRE_DATE,
    SERVICE_IN_YEARS,
    CASE 
        WHEN SERVICE_IN_YEARS < 1 THEN 10
        WHEN SERVICE_IN_YEARS BETWEEN 1 AND 5 THEN 15
        WHEN SERVICE_IN_YEARS BETWEEN 5 AND 10 THEN 20
        ELSE 25
    END AS ANNUAL_LEAVE_DAYS,
    -- Calculate accrual rate per month
    ROUND(CASE 
        WHEN SERVICE_IN_YEARS < 1 THEN 10
        WHEN SERVICE_IN_YEARS BETWEEN 1 AND 5 THEN 15
        WHEN SERVICE_IN_YEARS BETWEEN 5 AND 10 THEN 20
        ELSE 25
    END / 12, 2) AS MONTHLY_ACCRUAL
FROM EMP_SERVICE
ORDER BY SERVICE_IN_YEARS;
```

---

## ⚠️ TIME & LABOR-SPECIFIC NOTES

### Critical Time & Labor Constraints (DO NOT CHANGE)
These patterns must remain as-is:
1. **Timecard Approval Status:** Filter by approval status as needed
2. **Time Entry Validation:** Check for validation errors
3. **Date Range:** Always define clear start/end dates for timecard queries

**The new patterns COMPLEMENT these, they do not replace them.**

---

## 📊 INTEGRATION CHECKLIST

### Priority 1: Consider Integration (Within 1 Month)
- [ ] Add PARAMETERS CTE with Effective Date to timecard history queries
- [ ] Add service calculation for time-off eligibility queries
- [ ] Add case-insensitive filtering for time entry types

### Priority 2: Enhancement (Optional)
- [ ] Add date range parameters for flexible timecard queries
- [ ] Document time-off eligibility rules by service years

---

**END OF TL_REPOSITORIES_UPDATE_02-01-26.md**

**Status:** Ready for Review  
**Priority:** MEDIUM  
**Next Action:** Review by Time & Labor Module Maintainers

**Author:** AI Assistant  
**Date:** 02-01-2026  
**Version:** 1.0
