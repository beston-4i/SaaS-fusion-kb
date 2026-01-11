# HR Master Instructions - CRITICAL PERSON_TYPE_ID Fix

**Update Date:** 05-01-2026  
**Priority:** CRITICAL  
**Update Type:** Data Model Correction  
**Affected Module:** All HCM queries using Person Type

---

## 🚨 CRITICAL CORRECTION

### Issue Identified
**WRONG PATTERN (Used in previous queries):**
```sql
-- ❌ INCORRECT - This column does NOT exist in PER_ALL_PEOPLE_F
FROM PER_ALL_PEOPLE_F PAPF,
     PER_PERSON_TYPES_TL PPTTL
WHERE PAPF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID  -- ERROR: Column does not exist
```

**Root Cause:** `PERSON_TYPE_ID` column does **NOT** exist in `PER_ALL_PEOPLE_F` table

---

## ✅ CORRECT PATTERN (MANDATORY)

### Table Structure
- **PER_ALL_PEOPLE_F**: Does NOT contain `PERSON_TYPE_ID`
- **PER_ALL_ASSIGNMENTS_F**: DOES contain `PERSON_TYPE_ID`
- **Join Path**: PAPF → PAAF → PPTTL

### Correct Implementation
```sql
-- ✅ CORRECT - Person Type via Assignment Table
SELECT
    PAPF.PERSON_ID,
    PAPF.PERSON_NUMBER,
    PPTTL.USER_PERSON_TYPE AS PERSON_TYPE
FROM
    PER_ALL_PEOPLE_F PAPF,
    PER_ALL_ASSIGNMENTS_F PAAF,
    PER_PERSON_TYPES_TL PPTTL
WHERE
    PAPF.PERSON_ID = PAAF.PERSON_ID
AND PAAF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID  -- Via assignment table
AND PAAF.PRIMARY_FLAG = 'Y'
AND PAAF.ASSIGNMENT_TYPE = 'E'
AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
AND PPTTL.LANGUAGE = 'US'
```

---

## 📋 CTE IMPLEMENTATION PATTERNS

### Pattern 1: Early CTE (Before Assignment Data Needed)
When you need person type BEFORE retrieving full assignment details:

```sql
EMP_BASE AS (
    /*+ qb_name(EMP_BASE) MATERIALIZE */
    SELECT
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER,
        PPNF.FULL_NAME AS EMPLOYEE_NAME,
        PPTTL.USER_PERSON_TYPE AS PERSON_TYPE
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF,
        PER_ALL_ASSIGNMENTS_F PAAF_TYPE,  -- Separate alias for type lookup
        PER_PERSON_TYPES_TL PPTTL,
        PARAMETERS P
    WHERE
        -- Join person to names
        PAPF.PERSON_ID = PPNF.PERSON_ID
        -- Join person to assignment for type
    AND PAPF.PERSON_ID = PAAF_TYPE.PERSON_ID
        -- Join assignment to person type
    AND PAAF_TYPE.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID
        -- Name type filter
    AND PPNF.NAME_TYPE = 'GLOBAL'
        -- Language filter
    AND PPTTL.LANGUAGE = 'US'
        -- Assignment filters for type lookup
    AND PAAF_TYPE.PRIMARY_FLAG = 'Y'
    AND PAAF_TYPE.ASSIGNMENT_TYPE = 'E'
        -- Date-track filtering
    AND TRUNC(SYSDATE) BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
    AND TRUNC(SYSDATE) BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
    AND TRUNC(SYSDATE) BETWEEN PAAF_TYPE.EFFECTIVE_START_DATE AND PAAF_TYPE.EFFECTIVE_END_DATE
        -- Parameter filters
    AND (UPPER(PAPF.PERSON_NUMBER) = P.EMPLOYEE_NUMBER OR P.EMPLOYEE_NUMBER = 'ALL')
)
```

### Pattern 2: Later CTE (After Assignment CTE Exists)
When assignment data already retrieved in earlier CTE:

```sql
EMP_ASSIGNMENT AS (
    /*+ qb_name(EMP_ASSIGNMENT) MATERIALIZE */
    SELECT
        EB.PERSON_ID,
        EB.PERSON_NUMBER,
        EB.EMPLOYEE_NAME,
        -- Can use from EMP_BASE if already retrieved
        EB.PERSON_TYPE,
        -- OR retrieve fresh from assignment:
        PPTTL.USER_PERSON_TYPE AS PERSON_TYPE_CURRENT,
        PAAF.ASSIGNMENT_ID,
        -- ... other assignment fields
    FROM
        EMP_BASE EB,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_PERSON_TYPES_TL PPTTL
    WHERE
        EB.PERSON_ID = PAAF.PERSON_ID
    AND PAAF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID
    AND PPTTL.LANGUAGE = 'US'
    AND PAAF.PRIMARY_FLAG = 'Y'
    AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
)
```

---

## 🔄 MANDATORY FILTERS FOR PERSON TYPE LOOKUP

When joining to get person type, ALWAYS include:

1. **Primary Flag**: `PAAF.PRIMARY_FLAG = 'Y'`
2. **Assignment Type**: `PAAF.ASSIGNMENT_TYPE = 'E'` (for employees)
3. **Date Track**: `TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE`
4. **Language**: `PPTTL.LANGUAGE = 'US'`

---

## 📊 AFFECTED REPORTS TO UPDATE

This fix must be applied to:

1. **Employee Leave Count Report** ✅ FIXED
2. **Employee Absence Report** - CHECK NEEDED
3. **Annual Leave Balance Report** - CHECK NEEDED
4. **Any future reports needing person type**

---

## 🎯 MASTER RULE UPDATE

**OLD RULE (WRONG):**
> Join PER_ALL_PEOPLE_F directly to PER_PERSON_TYPES_TL using PERSON_TYPE_ID

**NEW RULE (CORRECT):**
> Person Type MUST be retrieved via PER_ALL_ASSIGNMENTS_F table
> Join path: PER_ALL_PEOPLE_F → PER_ALL_ASSIGNMENTS_F → PER_PERSON_TYPES_TL
> Required filters: PRIMARY_FLAG='Y', ASSIGNMENT_TYPE='E', date-track, language

---

## ⚠️ VALIDATION CHECKLIST

Before running any query with person type, verify:

- [ ] Person type NOT joined directly from PER_ALL_PEOPLE_F
- [ ] PER_ALL_ASSIGNMENTS_F included in join path
- [ ] PRIMARY_FLAG = 'Y' filter present
- [ ] ASSIGNMENT_TYPE = 'E' filter present
- [ ] Date-track filter on assignment present
- [ ] LANGUAGE = 'US' filter on types table present

---

## 📝 EXAMPLE COMPLETE CTE STRUCTURE

```sql
WITH 
PARAMETERS AS (
    SELECT
        UPPER(NVL(:P_EMPLOYEE_NUMBER, 'ALL')) AS EMPLOYEE_NUMBER
    FROM DUAL
),
-- Retrieve person type early via assignment
EMP_BASE AS (
    SELECT
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER,
        PPNF.FULL_NAME,
        PPTTL.USER_PERSON_TYPE AS PERSON_TYPE  -- Via PAAF join
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF,
        PER_ALL_ASSIGNMENTS_F PAAF,  -- REQUIRED for person type
        PER_PERSON_TYPES_TL PPTTL,
        PARAMETERS P
    WHERE
        PAPF.PERSON_ID = PPNF.PERSON_ID
    AND PAPF.PERSON_ID = PAAF.PERSON_ID
    AND PAAF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID  -- Correct join
    AND PPNF.NAME_TYPE = 'GLOBAL'
    AND PPTTL.LANGUAGE = 'US'
    AND PAAF.PRIMARY_FLAG = 'Y'
    AND PAAF.ASSIGNMENT_TYPE = 'E'
    AND TRUNC(SYSDATE) BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
    AND TRUNC(SYSDATE) BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
    AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
    AND (UPPER(PAPF.PERSON_NUMBER) = P.EMPLOYEE_NUMBER OR P.EMPLOYEE_NUMBER = 'ALL')
)
-- Continue with other CTEs...
```

---

## 🚨 IMPORTANCE LEVEL

**CRITICAL - This is the 3rd time reported by user**

- Immediate action required on all new queries
- Review all existing queries for this issue
- Add to pre-deployment checklist
- Include in code review standards

---

## 📚 RELATED DOCUMENTATION

- Main Fix Document: `CRITICAL_PERSON_TYPE_FIX.md`
- HR Master Instructions: `HR_MASTER.md`
- HR Repositories: `HR_REPOSITORIES.md`

---

**Last Updated:** 05-01-2026  
**Status:** ACTIVE - Apply to all queries immediately  
**User Feedback:** Reported 3 times - Critical priority fix
