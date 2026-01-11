# HR Repository Patterns - UPDATE 02-01-26

**Module:** Core HR  
**Update Type:** Cross-Module Knowledge Integration  
**Source:** Employee Annual Leave Balance Report (Absence Query)  
**Date:** 02-01-26  
**Applicability:** HIGH - 7 patterns directly applicable

---

## 📋 EXECUTIVE SUMMARY

This update adds HR-applicable patterns extracted from the Employee Annual Leave Balance Report. While the source query is from the Absence module, many patterns are fundamental to HR data extraction and apply across all employee-centric queries.

**Impact:** High - Critical patterns for historical queries and employee filtering  
**Priority:** Immediate integration recommended  
**Scope:** Core HR module (applicable to all HR queries)

---

## 🆕 NEW/ENHANCED CTE PATTERNS

### 1. PARAMETERS - Enhanced with Effective Date

**What's New:** Case-insensitive filtering with 'ALL' default and mandatory Effective Date parameter

**Current Pattern:**
```sql
-- No standardized parameter handling in current HR_REPOSITORIES
```

**Enhanced Pattern:**
```sql
WITH PARAMETERS AS (
    /*+ qb_name(PARAMETERS) */
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        UPPER(NVL(:P_LEGAL_EMPLOYER, 'ALL')) AS LEGAL_EMPLOYER,
        UPPER(NVL(:P_DEPARTMENT, 'ALL')) AS DEPARTMENT,
        UPPER(NVL(:P_JOB_TITLE, 'ALL')) AS JOB_TITLE,
        UPPER(NVL(:P_EMPLOYEE_TYPE, 'ALL')) AS EMPLOYEE_TYPE,
        UPPER(NVL(:P_LOCATION, 'ALL')) AS LOCATION
    FROM DUAL
)
```

**Key Improvements:**
1. **EFFECTIVE_DATE** parameter replaces SYSDATE for historical accuracy
2. **UPPER()** for case-insensitive matching
3. **NVL(..., 'ALL')** for simpler bypass logic
4. **TRUNC()** to normalize dates

**Usage in WHERE Clause:**
```sql
-- Old way (case-sensitive, complex NULL handling)
AND (:P_LEGAL_EMPLOYER IS NULL OR EA.LEGAL_EMPLOYER_NAME = :P_LEGAL_EMPLOYER)

-- New way (case-insensitive, simple)
AND (UPPER(EA.LEGAL_EMPLOYER_NAME) = P.LEGAL_EMPLOYER OR P.LEGAL_EMPLOYER = 'ALL')
```

**Applicable To:** ALL HR queries requiring parameter filtering

---

### 2. EMP_BASE - Enhanced with Service Calculation and Effective Date

**What's New:** Service calculation using Effective Date, not SYSDATE

**Current Pattern (from HR_REPOSITORIES.md):**
```sql
HR_WORKER_MASTER AS (
    SELECT /*+ qb_name(HR_WORKER) MATERIALIZE PARALLEL(2) */
           PAPF.PERSON_ID
          ,PAPF.PERSON_NUMBER
          ,PNAME.FULL_NAME
          ,PEMAIL.EMAIL_ADDRESS
          ,PPOS.ORIGINAL_DATE_OF_HIRE
    FROM   PER_ALL_PEOPLE_F PAPF
          ,PER_PERSON_NAMES_F PNAME
          ,PER_EMAIL_ADDRESSES PEMAIL
          ,PER_PERIODS_OF_SERVICE PPOS
    WHERE  TRUNC(SYSDATE) BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
      AND  TRUNC(SYSDATE) BETWEEN PNAME.EFFECTIVE_START_DATE AND PNAME.EFFECTIVE_END_DATE
    -- Issues: 
    -- 1. Uses SYSDATE (breaks historical queries)
    -- 2. No service calculation
)
```

**Enhanced Pattern:**
```sql
HR_WORKER_MASTER AS (
    /*+ qb_name(HR_WORKER) MATERIALIZE */
    SELECT
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER,
        PPNF.FULL_NAME,
        PPNF.DISPLAY_NAME,
        PPTTL.USER_PERSON_TYPE AS PERSON_TYPE,
        PPOS.PERIOD_OF_SERVICE_ID,
        TO_CHAR(NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START), 'DD-MM-YYYY') AS HIRE_DATE,
        -- NEW: Service in Years calculation
        ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) AS SERVICE_IN_YEARS,
        -- NEW: Store raw hire date for calculations
        NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START) AS HIRE_DATE_RAW
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
    -- CRITICAL: Use P.EFFECTIVE_DATE instead of SYSDATE
    AND P.EFFECTIVE_DATE BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PPOS.DATE_START 
        AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
)
```

**Key Improvements:**
1. **Service Calculation:** Accurate to 2 decimal places
2. **Effective Date Filtering:** Enables historical queries
3. **Raw Date Storage:** For further calculations
4. **Consistent Date Handling:** Uses parameter across all tables

**Business Value:**
- Enables "as of" reporting (e.g., "Show me employees as of 31-DEC-2023")
- Accurate service calculations for any point in time
- Audit compliance for historical data

**Applicable To:** All HR queries requiring employee base data

---

### 3. EMP_ASSIGNMENT - Enhanced with FT/PT Classification

**What's New:** Full Time / Part Time classification based on NORMAL_HOURS

**Current Pattern:**
```sql
HR_ASG_MASTER AS (
    SELECT /*+ qb_name(HR_ASG) MATERIALIZE */
           PAAM.ASSIGNMENT_ID
          ,PAAM.PERSON_ID
          ,PAAM.JOB_ID
          ,PAAM.ORGANIZATION_ID
          ,PAAM.LOCATION_ID
    FROM   PER_ALL_ASSIGNMENTS_M PAAM
    WHERE  TRUNC(SYSDATE) BETWEEN PAAM.EFFECTIVE_START_DATE AND PAAM.EFFECTIVE_END_DATE
    -- Issue: Missing FT/PT classification
)
```

**Enhanced Pattern:**
```sql
HR_ASG_MASTER AS (
    /*+ qb_name(HR_ASG) MATERIALIZE */
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
        -- NEW: Full Time / Part Time Classification
        CASE 
            WHEN NVL(PAAF.NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
            WHEN NVL(PAAF.NORMAL_HOURS, 0) > 0 AND NVL(PAAF.NORMAL_HOURS, 0) < 40 THEN 'Part Time'
            ELSE 'Not Specified'
        END AS FULL_TIME_PART_TIME
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
    -- Use Effective Date, not SYSDATE
    AND P.EFFECTIVE_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PJFV.EFFECTIVE_START_DATE(+) AND PJFV.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN HAPL.EFFECTIVE_START_DATE(+) AND HAPL.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN PD.EFFECTIVE_START_DATE(+) AND PD.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN HLOCVL.EFFECTIVE_START_DATE(+) AND HLOCVL.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN PGFV.EFFECTIVE_START_DATE(+) AND PGFV.EFFECTIVE_END_DATE(+)
)
```

**Configuration Note:** The 40-hour threshold is customizable. Some organizations use 37.5 or 35 hours.

**Applicable To:** HR, Payroll, Absence, Benefits queries requiring employment classification

---

### 4. EMP_DFF - NEW CTE for DFF Attributes

**What's New:** Standardized DFF attribute extraction pattern

**Pattern:**
```sql
EMP_DFF AS (
    /*+ qb_name(EMP_DFF) */
    SELECT
        EA.PERSON_ID,
        EA.ASSIGNMENT_ID,
        -- Map DFF attributes to business fields
        -- TODO: Update these mappings based on FND_DESCR_FLEX_COLUMN_USAGES query results
        EA.ATTR1 AS CONTRACT_TYPE,
        EA.ATTR5 AS CLIENT_JOB_TITLE,
        EA.ATTR3 AS PROJECT_NUMBER,
        EA.ATTR4 AS SERVICE_LINE,
        -- Add more DFF attributes as needed
        EA.ATTR2 AS CUSTOM_FIELD_2,
        EA.ATTR6 AS CUSTOM_FIELD_6
    FROM
        EMP_ASSIGNMENT EA
)
```

**Discovery Query (Run First):**
```sql
SELECT 
    DFC.APPLICATION_COLUMN_NAME,      -- e.g., ATTRIBUTE1, ATTRIBUTE2
    DFC.END_USER_COLUMN_NAME,         -- e.g., Contract Type, Client Job Title
    DFC.COLUMN_SEQ_NUM,
    DFC.ENABLED_FLAG
FROM FND_DESCR_FLEX_COLUMN_USAGES DFC
WHERE DFC.APPLICATION_TABLE_NAME = 'PER_ALL_ASSIGNMENTS_F'
AND DFC.ENABLED_FLAG = 'Y'
ORDER BY DFC.COLUMN_SEQ_NUM;
```

**Usage:** After running discovery query, update the DFF CTE with actual attribute mappings.

**Best Practice:** Document the mapping in comments for future maintenance.

**Applicable To:** All HR queries requiring custom DFF attributes

---

### 5. HR_ORG_MASTER - Enhanced with Full Organizational Context

**What's New:** Complete organizational hierarchy in single CTE

**Current Pattern:**
```sql
HR_ORG_MASTER AS (
    SELECT /*+ qb_name(HR_ORG) MATERIALIZE */
           HAOU.ORGANIZATION_ID
          ,HAOU.NAME AS DEPT_NAME
    FROM   HR_ALL_ORGANIZATION_UNITS_F HAOU
    WHERE  TRUNC(SYSDATE) BETWEEN HAOU.EFFECTIVE_START_DATE AND HAOU.EFFECTIVE_END_DATE
)
-- Issue: Incomplete, no legal entity, business unit, location
```

**Enhanced Pattern:**
```sql
HR_ORG_MASTER AS (
    /*+ qb_name(HR_ORG) MATERIALIZE */
    SELECT
        PD.ORGANIZATION_ID AS DEPT_ORG_ID,
        PD.NAME AS DEPARTMENT_NAME,
        HAOULE.ORGANIZATION_ID AS LEGAL_ENTITY_ID,
        HAOULE.NAME AS LEGAL_EMPLOYER_NAME,
        HAOUBU.ORGANIZATION_ID AS BUSINESS_UNIT_ID,
        HAOUBU.NAME AS BUSINESS_UNIT_NAME,
        HLOCVL.LOCATION_ID,
        HLOCVL.LOCATION_NAME,
        PGFV.GRADE_ID,
        PGFV.NAME AS GRADE_NAME
    FROM
        PER_DEPARTMENTS PD,
        HR_ALL_ORGANIZATION_UNITS HAOULE,
        HR_ALL_ORGANIZATION_UNITS HAOUBU,
        HR_LOCATIONS_ALL_F_VL HLOCVL,
        PER_GRADES_F_VL PGFV,
        PARAMETERS P
    WHERE
        -- Join conditions
        PD.ORGANIZATION_ID IS NOT NULL  -- Placeholder for actual joins
    AND PGFV.LANGUAGE(+) = 'US'
    -- Date-track filtering
    AND P.EFFECTIVE_DATE BETWEEN PD.EFFECTIVE_START_DATE(+) AND PD.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN HLOCVL.EFFECTIVE_START_DATE(+) AND HLOCVL.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN PGFV.EFFECTIVE_START_DATE(+) AND PGFV.EFFECTIVE_END_DATE(+)
)
```

**Applicable To:** All HR queries requiring organizational context

---

## 🎯 CRITICAL PATTERN: EFFECTIVE DATE vs SYSDATE

### ❌ NEVER DO THIS (Current HR_REPOSITORIES pattern):
```sql
WHERE TRUNC(SYSDATE) BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
```

### ✅ ALWAYS DO THIS (New pattern):
```sql
WHERE P.EFFECTIVE_DATE BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
```

**Why This Matters:**
- SYSDATE returns current data, breaking historical queries
- Effective Date parameter enables "as of" reporting
- Critical for audit compliance and historical analysis

**Example Impact:**
```sql
-- Query 1: Using SYSDATE (WRONG)
-- Question: "Show me employees as of 31-DEC-2023"
-- Result: Returns current employees (as of today), not 31-DEC-2023

-- Query 2: Using EFFECTIVE_DATE (CORRECT)
-- Question: "Show me employees as of 31-DEC-2023"
-- Result: Returns employees exactly as they were on 31-DEC-2023
```

---

## 📊 INTEGRATION CHECKLIST

### Priority 1: CRITICAL (Immediate Integration)
- [ ] Add PARAMETERS CTE with Effective Date to all HR queries
- [ ] Replace SYSDATE with P.EFFECTIVE_DATE in all date-track filters
- [ ] Update HR_WORKER_MASTER with service calculation
- [ ] Add case-insensitive parameter filtering pattern

### Priority 2: HIGH VALUE (Within 2 Weeks)
- [ ] Add FULL_TIME_PART_TIME classification to HR_ASG_MASTER
- [ ] Create EMP_DFF CTE template
- [ ] Enhance HR_ORG_MASTER with complete organizational context

### Priority 3: ENHANCEMENT (Within 1 Month)
- [ ] Document DFF discovery process
- [ ] Create validation queries for new patterns
- [ ] Update HR report templates

---

## 🔗 USAGE EXAMPLES

### Example 1: Historical Headcount Report
```sql
WITH PARAMETERS AS (
    SELECT TRUNC(TO_DATE('31-DEC-2023', 'DD-MON-YYYY')) AS EFFECTIVE_DATE
    FROM DUAL
)
,HR_WORKER_MASTER AS (
    -- Use enhanced pattern with service calculation
    ...
)
SELECT
    COUNT(*) AS HEADCOUNT,
    AVG(SERVICE_IN_YEARS) AS AVG_SERVICE
FROM HR_WORKER_MASTER;
```

### Example 2: Employee List with Filters
```sql
WITH PARAMETERS AS (
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        UPPER(NVL(:P_DEPARTMENT, 'ALL')) AS DEPARTMENT
    FROM DUAL
)
,HR_WORKER_MASTER AS (
    -- Enhanced pattern
    ...
)
,HR_ASG_MASTER AS (
    -- Enhanced pattern with FT/PT
    ...
)
SELECT
    HW.PERSON_NUMBER,
    HW.FULL_NAME,
    HA.DEPARTMENT_NAME,
    HA.FULL_TIME_PART_TIME,
    HW.SERVICE_IN_YEARS
FROM HR_WORKER_MASTER HW, HR_ASG_MASTER HA
WHERE HW.PERSON_ID = HA.PERSON_ID
AND (UPPER(HA.DEPARTMENT_NAME) = P.DEPARTMENT OR P.DEPARTMENT = 'ALL');
```

---

## ⚠️ BACKWARD COMPATIBILITY

### Breaking Changes: NONE
All new patterns are additive. Existing queries will continue to work.

### Migration Path:
1. **Phase 1:** Add PARAMETERS CTE to new queries
2. **Phase 2:** Gradually update existing queries during maintenance
3. **Phase 3:** Deprecate SYSDATE pattern in documentation

### Rollback Plan:
If issues arise, simply revert to SYSDATE pattern. However, loss of historical query capability.

---

## 📚 REFERENCE

### Source Query
- File: `Requirement\Employee_Annual_Leave_Balance_Query.sql`
- Lines: 42-96 (PARAMETERS, EMP_BASE)
- Lines: 101-171 (EMP_ASSIGNMENT)
- Lines: 177-190 (EMP_DFF)

### Validation Queries
```sql
-- Test 1: Verify Effective Date filtering works
SELECT COUNT(*) 
FROM PER_ALL_PEOPLE_F PAPF, PARAMETERS P
WHERE TO_DATE('31-DEC-2023', 'DD-MON-YYYY') BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE;

-- Test 2: Verify service calculation
SELECT 
    PERSON_NUMBER,
    HIRE_DATE,
    SERVICE_IN_YEARS
FROM HR_WORKER_MASTER
WHERE SERVICE_IN_YEARS > 5
ORDER BY SERVICE_IN_YEARS DESC;

-- Test 3: Verify case-insensitive filtering
SELECT COUNT(*)
FROM HR_ASG_MASTER, PARAMETERS P
WHERE UPPER(DEPARTMENT_NAME) = UPPER('human resources')  -- Should match 'Human Resources'
```

---

## 🎓 TRAINING NOTES

### For Developers
**Key Concepts to Learn:**
1. Why EFFECTIVE_DATE > SYSDATE for date-tracked tables
2. Service calculation formula
3. Case-insensitive parameter pattern
4. DFF discovery and mapping process

**Training Time:** 2 hours

### For Business Users
**What Changed:**
- Can now query employee data "as of" any past date
- New "Service in Years" field available
- Filter parameters now case-insensitive

**Training Time:** 30 minutes

---

**END OF HR_REPOSITORIES_UPDATE_02-01-26.md**

**Status:** Ready for Review and Integration  
**Priority:** HIGH  
**Next Action:** Review by HR Module Maintainers

**Author:** AI Assistant  
**Date:** 02-01-2026  
**Version:** 1.0
