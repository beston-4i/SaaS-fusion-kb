# Compensation Repository Patterns - UPDATE 02-01-26

**Module:** Compensation  
**Update Type:** Cross-Module Knowledge Integration  
**Source:** Employee Annual Leave Balance Report  
**Date:** 02-01-26  
**Applicability:** LOW-MEDIUM - 3 patterns applicable

---

## 📋 EXECUTIVE SUMMARY

This update adds Compensation-applicable patterns from the Employee Annual Leave Balance Report. Focus on effective date filtering for salary history queries and service-based calculations.

**Impact:** Low-Medium - Enhances historical compensation queries  
**Priority:** Low (integrate as needed)  
**Scope:** Compensation history and salary progression queries

---

## 🆕 APPLICABLE PATTERNS FOR COMPENSATION

### Pattern 1: Effective Date Filtering for Salary History

**What's New:** Parameter-based date filtering for "as of" salary queries

**Pattern:**
```sql
WITH PARAMETERS AS (
    /*+ qb_name(PARAMETERS) */
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        UPPER(NVL(:P_SALARY_BASIS, 'ALL')) AS SALARY_BASIS
    FROM DUAL
)
,CMP_SALARY_HISTORY AS (
    /*+ qb_name(CMP_SAL_HIST) MATERIALIZE */
    SELECT
        CSAL.PERSON_ID,
        CSAL.SALARY_AMOUNT,
        CSAL.SALARY_BASIS_ID,
        CSB.NAME AS SALARY_BASIS_NAME,
        TO_CHAR(CSAL.DATE_FROM, 'DD-MM-YYYY') AS EFFECTIVE_FROM,
        TO_CHAR(CSAL.DATE_TO, 'DD-MM-YYYY') AS EFFECTIVE_TO
    FROM
        CMP_SALARY CSAL,
        CMP_SALARY_BASIS CSB,
        PARAMETERS P
    WHERE
        CSAL.SALARY_BASIS_ID = CSB.SALARY_BASIS_ID(+)
    -- Salary active as of Effective Date
    AND P.EFFECTIVE_DATE BETWEEN CSAL.DATE_FROM 
        AND NVL(CSAL.DATE_TO, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
)
```

**Benefits:**
- Query salaries "as of" any past date
- Salary progression analysis
- Historical compensation snapshots

**Applicable To:** Salary history, compensation analysis queries

---

### Pattern 2: Service-Based Compensation Analysis

**What's New:** Service in Years for compensation planning

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

-- Use in compensation analysis
SELECT
    ES.PERSON_ID,
    ES.SERVICE_IN_YEARS,
    CS.SALARY_AMOUNT,
    CASE 
        WHEN ES.SERVICE_IN_YEARS < 2 THEN 'Entry Level'
        WHEN ES.SERVICE_IN_YEARS BETWEEN 2 AND 5 THEN 'Mid Level'
        WHEN ES.SERVICE_IN_YEARS BETWEEN 5 AND 10 THEN 'Senior Level'
        ELSE 'Expert Level'
    END AS EXPERIENCE_BAND,
    -- Salary per year of service
    ROUND(CS.SALARY_AMOUNT / GREATEST(ES.SERVICE_IN_YEARS, 1), 2) AS SALARY_PER_SERVICE_YEAR
FROM
    EMP_SERVICE ES,
    CMP_SALARY_HISTORY CS
WHERE
    ES.PERSON_ID = CS.PERSON_ID;
```

**Benefits:**
- Service-based compensation analysis
- Salary progression tracking
- Compensation planning by experience level

**Applicable To:** Compensation planning, salary benchmarking

---

### Pattern 3: Case-Insensitive Parameter Filtering

**What's New:** UPPER() for salary basis and grade filtering

**Pattern:**
```sql
WITH PARAMETERS AS (
    SELECT
        UPPER(NVL(:P_SALARY_BASIS, 'ALL')) AS SALARY_BASIS,
        UPPER(NVL(:P_GRADE, 'ALL')) AS GRADE,
        UPPER(NVL(:P_PAY_FREQUENCY, 'ALL')) AS PAY_FREQUENCY
    FROM DUAL
)

-- Apply in WHERE
AND (UPPER(CSB.NAME) = P.SALARY_BASIS OR P.SALARY_BASIS = 'ALL')
AND (UPPER(PG.NAME) = P.GRADE OR P.GRADE = 'ALL')
AND (UPPER(CSB.PAY_FREQUENCY) = P.PAY_FREQUENCY OR P.PAY_FREQUENCY = 'ALL')
```

**Benefits:**
- Flexible filtering (users can enter "monthly" or "Monthly" or "MONTHLY")
- 'ALL' bypass for showing all records
- Consistent with other HCM modules

**Applicable To:** All compensation queries with filters

---

## 🎯 INTEGRATION EXAMPLES

### Example 1: Historical Salary Query

```sql
WITH PARAMETERS AS (
    SELECT 
        TRUNC(TO_DATE('01-JAN-2023', 'DD-MON-YYYY')) AS EFFECTIVE_DATE
    FROM DUAL
)
,CMP_SALARY_HISTORY AS (
    -- Enhanced pattern with Effective Date
    ...
)
SELECT
    PERSON_NUMBER,
    FULL_NAME,
    SALARY_AMOUNT,
    SALARY_BASIS_NAME,
    EFFECTIVE_FROM
FROM CMP_SALARY_HISTORY
WHERE SALARY_AMOUNT > 50000
ORDER BY SALARY_AMOUNT DESC;

-- Result: Shows salaries exactly as they were on 01-JAN-2023
```

### Example 2: Service-Based Compensation Analysis

```sql
SELECT
    EXPERIENCE_BAND,
    COUNT(*) AS EMPLOYEE_COUNT,
    AVG(SALARY_AMOUNT) AS AVG_SALARY,
    MIN(SALARY_AMOUNT) AS MIN_SALARY,
    MAX(SALARY_AMOUNT) AS MAX_SALARY,
    AVG(SALARY_PER_SERVICE_YEAR) AS AVG_SALARY_PER_YEAR
FROM (
    SELECT
        ES.SERVICE_IN_YEARS,
        CS.SALARY_AMOUNT,
        CASE 
            WHEN ES.SERVICE_IN_YEARS < 2 THEN 'Entry Level'
            WHEN ES.SERVICE_IN_YEARS BETWEEN 2 AND 5 THEN 'Mid Level'
            WHEN ES.SERVICE_IN_YEARS BETWEEN 5 AND 10 THEN 'Senior Level'
            ELSE 'Expert Level'
        END AS EXPERIENCE_BAND,
        ROUND(CS.SALARY_AMOUNT / GREATEST(ES.SERVICE_IN_YEARS, 1), 2) AS SALARY_PER_SERVICE_YEAR
    FROM EMP_SERVICE ES, CMP_SALARY_HISTORY CS
    WHERE ES.PERSON_ID = CS.PERSON_ID
)
GROUP BY EXPERIENCE_BAND
ORDER BY AVG_SALARY;
```

---

## 📊 INTEGRATION CHECKLIST

### Priority 1: Consider Integration (As Needed)
- [ ] Add PARAMETERS CTE with Effective Date to salary history queries
- [ ] Add service calculation for compensation planning queries
- [ ] Add case-insensitive filtering for salary basis/grade

---

**END OF CMP_REPOSITORIES_UPDATE_02-01-26.md**

**Status:** Ready for Review  
**Priority:** LOW-MEDIUM  
**Next Action:** Review by Compensation Module Maintainers

**Author:** AI Assistant  
**Date:** 02-01-2026  
**Version:** 1.0
