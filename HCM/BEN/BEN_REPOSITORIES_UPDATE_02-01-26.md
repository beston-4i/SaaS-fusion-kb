# Benefits Repository Patterns - UPDATE 02-01-26

**Module:** Benefits  
**Update Type:** Cross-Module Knowledge Integration  
**Source:** Employee Annual Leave Balance Report  
**Date:** 02-01-26  
**Applicability:** MEDIUM - 3 patterns applicable

---

## 📋 EXECUTIVE SUMMARY

This update adds Benefits-applicable patterns from the Employee Annual Leave Balance Report. Focus on effective date filtering for enrollment queries and parameter handling for plan filtering.

**Impact:** Medium - Improves historical enrollment queries  
**Priority:** Medium (integrate within 1 month)  
**Scope:** Benefits enrollment and coverage queries

---

## 🆕 APPLICABLE PATTERNS FOR BENEFITS

### Pattern 1: Effective Date Filtering for Enrollments

**What's New:** Parameter-based date filtering for "as of" enrollment queries

**Current BEN_REPOSITORIES Pattern:**
```sql
BEN_ENROLL_MASTER AS (
    SELECT /*+ qb_name(BEN_ENR) MATERIALIZE */
           PEN.PERSON_ID
          ,PEN.PL_ID
          ,PEN.PRTT_ENRT_RSLT_ID
          ,PL.NAME AS PLAN_NAME
    FROM   BEN_PRTT_ENRT_RSLT_F PEN
          ,BEN_PL_F PL
    WHERE  TRUNC(SYSDATE) BETWEEN PEN.EFFECTIVE_START_DATE AND PEN.EFFECTIVE_END_DATE
      AND  PEN.PRTT_ENRT_RSLT_STAT_CD IS NULL
      AND  PEN.ENRT_CVG_THRU_DT = TO_DATE('4712-12-31', 'YYYY-MM-DD')
    -- Issue: Uses SYSDATE, can't query historical enrollments
)
```

**Enhanced Pattern:**
```sql
WITH PARAMETERS AS (
    /*+ qb_name(PARAMETERS) */
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        UPPER(NVL(:P_PLAN_TYPE, 'ALL')) AS PLAN_TYPE,
        UPPER(NVL(:P_PLAN_NAME, 'ALL')) AS PLAN_NAME
    FROM DUAL
)
,BEN_ENROLL_MASTER AS (
    /*+ qb_name(BEN_ENR) MATERIALIZE */
    SELECT
        PEN.PERSON_ID,
        PEN.PL_ID,
        PEN.OIPL_ID,
        PEN.PRTT_ENRT_RSLT_ID,
        PL.NAME AS PLAN_NAME,
        OPT.NAME AS OPTION_NAME,
        TO_CHAR(PEN.ENRT_CVG_STRT_DT, 'DD-MM-YYYY') AS COVERAGE_START_DATE,
        TO_CHAR(PEN.ENRT_CVG_THRU_DT, 'DD-MM-YYYY') AS COVERAGE_END_DATE
    FROM
        BEN_PRTT_ENRT_RSLT_F PEN,
        BEN_PL_F PL,
        BEN_OIPL_F OIPL,
        BEN_OPT_F OPT,
        PARAMETERS P
    WHERE
        PEN.PL_ID = PL.PL_ID
    AND PEN.OIPL_ID = OIPL.OIPL_ID(+)
    AND OIPL.OPT_ID = OPT.OPT_ID(+)
    -- Active enrollment status
    AND PEN.PRTT_ENRT_RSLT_STAT_CD IS NULL
    -- Coverage active as of Effective Date
    AND P.EFFECTIVE_DATE BETWEEN PEN.ENRT_CVG_STRT_DT 
        AND NVL(PEN.ENRT_CVG_THRU_DT, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
    -- Date-track filtering using Effective Date
    AND P.EFFECTIVE_DATE BETWEEN PEN.EFFECTIVE_START_DATE AND PEN.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PL.EFFECTIVE_START_DATE AND PL.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN OIPL.EFFECTIVE_START_DATE(+) AND OIPL.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN OPT.EFFECTIVE_START_DATE(+) AND OPT.EFFECTIVE_END_DATE(+)
)
```

**Benefits:**
- Query enrollments "as of" any past date
- Audit compliance for historical coverage verification
- Year-end enrollment snapshots

**Applicable To:** All benefits enrollment queries

---

### Pattern 2: Case-Insensitive Plan Filtering

**What's New:** UPPER() for plan name and type filtering

**Pattern:**
```sql
WITH PARAMETERS AS (
    SELECT
        UPPER(NVL(:P_PLAN_TYPE, 'ALL')) AS PLAN_TYPE,
        UPPER(NVL(:P_PLAN_NAME, 'ALL')) AS PLAN_NAME,
        UPPER(NVL(:P_OPTION_NAME, 'ALL')) AS OPTION_NAME
    FROM DUAL
)

-- Apply in WHERE
AND (UPPER(PT.NAME) = P.PLAN_TYPE OR P.PLAN_TYPE = 'ALL')
AND (UPPER(PL.NAME) = P.PLAN_NAME OR P.PLAN_NAME = 'ALL')
AND (UPPER(OPT.NAME) = P.OPTION_NAME OR P.OPTION_NAME = 'ALL')
```

**Benefits:**
- Users can enter "medical" or "Medical" or "MEDICAL"
- 'ALL' bypass for showing all plans
- Consistent with HR/Absence patterns

**Applicable To:** All benefits queries with plan filters

---

### Pattern 3: Optional Table Handling for Custom Benefits

**What's New:** Handling client-specific benefits tables

**Pattern:**
```sql
-- Define optional custom benefits CTE
BEN_CUSTOM_COVERAGE AS (
    SELECT
        PERSON_ID,
        CUSTOM_COVERAGE_AMOUNT,
        CUSTOM_COVERAGE_TYPE
    FROM CLIENT_CUSTOM_BENEFITS  -- May not exist
    WHERE ACTIVE_FLAG = 'Y'
)

-- Use outer join
FROM
    BEN_ENROLL_MASTER BEM,
    BEN_CUSTOM_COVERAGE BCC
WHERE
    BEM.PERSON_ID = BCC.PERSON_ID(+)

-- Handle NULL
SELECT
    BEM.PERSON_NUMBER,
    BEM.PLAN_NAME,
    NVL(BCC.CUSTOM_COVERAGE_AMOUNT, 0) AS CUSTOM_COVERAGE
FROM ...
```

**Documentation:**
```sql
/*
 * OPTIONAL BENEFITS TABLES
 * =========================
 * 1. CLIENT_CUSTOM_BENEFITS - Custom coverage tracking
 *    - If missing, comment out BEN_CUSTOM_COVERAGE CTE
 *    - Custom fields will show as 0 or NULL
 */
```

**Applicable To:** Flex benefits, client-specific benefit programs

---

## 🎯 INTEGRATION EXAMPLES

### Example 1: Historical Enrollment Query

```sql
WITH PARAMETERS AS (
    SELECT 
        TRUNC(TO_DATE('01-JAN-2023', 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        'ALL' AS PLAN_TYPE
    FROM DUAL
)
,BEN_ENROLL_MASTER AS (
    -- Enhanced pattern with Effective Date
    ...
)
SELECT
    PERSON_NUMBER,
    FULL_NAME,
    PLAN_NAME,
    OPTION_NAME,
    COVERAGE_START_DATE
FROM BEN_ENROLL_MASTER
WHERE PLAN_TYPE = 'Medical'
ORDER BY PERSON_NUMBER;

-- Result: Shows enrollments exactly as they were on 01-JAN-2023
```

### Example 2: Current Enrollments with Case-Insensitive Filter

```sql
WITH PARAMETERS AS (
    SELECT 
        TRUNC(SYSDATE) AS EFFECTIVE_DATE,
        UPPER(NVL(:P_PLAN_NAME, 'ALL')) AS PLAN_NAME
    FROM DUAL
)
,BEN_ENROLL_MASTER AS (
    -- Enhanced pattern
    ...
)
SELECT
    COUNT(*) AS ENROLLMENT_COUNT,
    PLAN_NAME
FROM BEN_ENROLL_MASTER, PARAMETERS P
WHERE (UPPER(PLAN_NAME) = P.PLAN_NAME OR P.PLAN_NAME = 'ALL')
GROUP BY PLAN_NAME;

-- User can enter :P_PLAN_NAME as 'medical plan' or 'Medical Plan' or 'MEDICAL PLAN'
```

---

## ⚠️ BENEFITS-SPECIFIC NOTES

### Critical Benefits Constraints (DO NOT CHANGE)
These patterns must remain as-is:
1. **Enrollment Status:** `AND PEN.PRTT_ENRT_RSLT_STAT_CD IS NULL` - REQUIRED for active enrollments
2. **Coverage Period:** Check `ENRT_CVG_STRT_DT` and `ENRT_CVG_THRU_DT` - REQUIRED
3. **Life Event Context:** Join to `BEN_PER_IN_LER` for enrollment reasons - RECOMMENDED

**The new patterns COMPLEMENT these, they do not replace them.**

---

## 📊 INTEGRATION CHECKLIST

### Priority 1: Consider Integration (Within 1 Month)
- [ ] Add PARAMETERS CTE with Effective Date to enrollment queries
- [ ] Replace SYSDATE with P.EFFECTIVE_DATE in enrollment queries
- [ ] Add case-insensitive plan filtering

### Priority 2: Enhancement (Optional)
- [ ] Document optional custom benefits tables
- [ ] Add coverage period validation using Effective Date

---

**END OF BEN_REPOSITORIES_UPDATE_02-01-26.md**

**Status:** Ready for Review  
**Priority:** MEDIUM  
**Next Action:** Review by Benefits Module Maintainers

**Author:** AI Assistant  
**Date:** 02-01-2026  
**Version:** 1.0
