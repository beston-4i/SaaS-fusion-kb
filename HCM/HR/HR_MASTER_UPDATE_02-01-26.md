# HR Master Instructions - UPDATE 02-01-26

**Module:** Core HR  
**Update Type:** Best Practices and Patterns from Cross-Module Learning  
**Source:** Employee Annual Leave Balance Report  
**Date:** 02-01-26  
**Applicability:** HIGH

---

## 📋 EXECUTIVE SUMMARY

This update adds 8 new best practices and patterns to the Core HR Master Instructions based on learnings from the Employee Annual Leave Balance Report. These patterns enhance historical query accuracy, improve user experience, and establish consistent standards across HCM modules.

---

## 🆕 NEW PATTERNS TO ADD

### Pattern 1: Effective Date Filtering (CRITICAL - ADD TO SECTION 1)

**Location in HR_MASTER.md:** Add as **Rule 1.5** under "Critical HR Constraints"

```markdown
### 1.5 **Effective Date Parameter (The "Historical Rule"):**
*   **Rule:** `AND :P_EFFECTIVE_DATE BETWEEN [TABLE].EFFECTIVE_START_DATE AND [TABLE].EFFECTIVE_END_DATE`
*   **Why:** SYSDATE always returns current data, breaking historical/"as of" queries.
*   **Impact:** Critical for audit compliance, historical reporting, point-in-time analysis.
*   **Scope:** ALL date-tracked tables (`_F`, `_TL` tables).

**Pattern:**
```sql
-- Define parameter first
WITH PARAMETERS AS (
    SELECT TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE
    FROM DUAL
)

-- Apply consistently across ALL date-tracked tables
WHERE P.EFFECTIVE_DATE BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
  AND P.EFFECTIVE_DATE BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
  AND P.EFFECTIVE_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
```

**When to Use:**
- Historical reports (e.g., "Show me employees as of 31-DEC-2023")
- Audit queries requiring specific date accuracy
- Compliance reports for past periods
- Year-end snapshots

**When to Use SYSDATE:**
- Real-time dashboards requiring current data
- Operational reports always showing "now"
- Queries explicitly scoped to current state only
```

---

### Pattern 2: Service in Years Calculation (ADD TO SECTION 5)

**Location in HR_MASTER.md:** Add to **Section 5** (new section or existing calculations section)

```markdown
## 5.5 Service in Years Calculation

**Formula:**
```sql
ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) AS SERVICE_IN_YEARS
```

**Key Points:**
- Use `ORIGINAL_DATE_OF_HIRE` if available, fallback to `DATE_START`
- Use `EFFECTIVE_DATE` parameter (not SYSDATE) for historical accuracy
- `MONTHS_BETWEEN` provides precise calculation
- Round to 2 decimals for readability
- Result accurate to day-level precision

**Source Table:** `PER_PERIODS_OF_SERVICE`

**Date Filter:**
```sql
AND P.EFFECTIVE_DATE BETWEEN PPOS.DATE_START 
    AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
```

**Usage Example:**
```sql
SELECT
    PERSON_NUMBER,
    FULL_NAME,
    TO_CHAR(HIRE_DATE, 'DD-MM-YYYY') AS HIRE_DATE,
    ROUND(MONTHS_BETWEEN(TRUNC(SYSDATE), HIRE_DATE) / 12, 2) AS SERVICE_YEARS,
    CASE 
        WHEN ROUND(MONTHS_BETWEEN(TRUNC(SYSDATE), HIRE_DATE) / 12, 2) >= 10 THEN 'Senior'
        WHEN ROUND(MONTHS_BETWEEN(TRUNC(SYSDATE), HIRE_DATE) / 12, 2) >= 5 THEN 'Mid-Level'
        ELSE 'Junior'
    END AS SENIORITY_BAND
FROM HR_WORKER_MASTER;
```
```

---

### Pattern 3: Case-Insensitive Parameter Filtering (ADD TO SECTION 8)

**Location in HR_MASTER.md:** Add as **Section 8.4** under "Standard Filters"

```markdown
### 8.4 Case-Insensitive Parameter Filtering with 'ALL' Default

**Problem:** Users struggle with exact case matching ("DUBAI" vs "Dubai" vs "dubai")

**Solution:**
```sql
-- Step 1: In PARAMETERS CTE
WITH PARAMETERS AS (
    SELECT
        UPPER(NVL(:P_LEGAL_EMPLOYER, 'ALL')) AS LEGAL_EMPLOYER,
        UPPER(NVL(:P_DEPARTMENT, 'ALL')) AS DEPARTMENT,
        UPPER(NVL(:P_LOCATION, 'ALL')) AS LOCATION
    FROM DUAL
)

-- Step 2: In WHERE clause
AND (UPPER(EA.LEGAL_EMPLOYER_NAME) = P.LEGAL_EMPLOYER OR P.LEGAL_EMPLOYER = 'ALL')
AND (UPPER(EA.DEPARTMENT_NAME) = P.DEPARTMENT OR P.DEPARTMENT = 'ALL')
AND (UPPER(EA.LOCATION_NAME) = P.LOCATION OR P.LOCATION = 'ALL')
```

**Benefits:**
- Users can enter "DUBAI", "Dubai", or "dubai" - all work
- 'ALL' bypasses filter without complex NULL handling
- Simplified logic, no DECODE/CASE needed
- Self-documenting code
- Consistent pattern across all parameters

**Old Way (Complex NULL Handling):**
```sql
-- Complex and error-prone
AND (:P_DEPARTMENT IS NULL OR UPPER(DEPT_NAME) = UPPER(:P_DEPARTMENT))
AND (DECODE(:P_LOCATION, NULL, 1, DECODE(UPPER(LOC_NAME), UPPER(:P_LOCATION), 1, 0)) = 1)
```

**New Way (Simple and Clear):**
```sql
-- Simple and maintainable
AND (UPPER(DEPT_NAME) = P.DEPARTMENT OR P.DEPARTMENT = 'ALL')
AND (UPPER(LOC_NAME) = P.LOCATION OR P.LOCATION = 'ALL')
```
```

---

### Pattern 4: Full Time / Part Time Classification (ADD TO SECTION 11)

**Location in HR_MASTER.md:** Add as **Section 11.3** under "Worker Category & Type"

```markdown
### 11.3 Full Time / Part Time Classification

**Pattern:**
```sql
CASE 
    WHEN NVL(PAAF.NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
    WHEN NVL(PAAF.NORMAL_HOURS, 0) > 0 AND NVL(PAAF.NORMAL_HOURS, 0) < 40 THEN 'Part Time'
    ELSE 'Not Specified'
END AS FULL_TIME_PART_TIME
```

**Source Field:** `PER_ALL_ASSIGNMENTS_F.NORMAL_HOURS`

**Configuration Note:** The 40-hour threshold is customizable based on:
- Country legislation (some use 37.5 or 35 hours)
- Organization policy
- Industry standards

**Enhanced Version with Frequency Check:**
```sql
CASE 
    WHEN PAAF.FREQUENCY = 'W' AND NVL(PAAF.NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
    WHEN PAAF.FREQUENCY = 'W' AND NVL(PAAF.NORMAL_HOURS, 0) > 0 AND NVL(PAAF.NORMAL_HOURS, 0) < 40 THEN 'Part Time'
    WHEN PAAF.FREQUENCY = 'M' AND NVL(PAAF.NORMAL_HOURS, 0) >= 160 THEN 'Full Time'  -- 40 hours * 4 weeks
    WHEN PAAF.FREQUENCY = 'M' AND NVL(PAAF.NORMAL_HOURS, 0) > 0 AND NVL(PAAF.NORMAL_HOURS, 0) < 160 THEN 'Part Time'
    ELSE 'Not Specified'
END AS FULL_TIME_PART_TIME
```

**Frequency Values:**
- `'W'` - Weekly
- `'M'` - Monthly
- `'Y'` - Yearly

**Usage Example:**
```sql
SELECT
    PERSON_NUMBER,
    FULL_NAME,
    NORMAL_HOURS,
    FREQUENCY,
    CASE 
        WHEN NVL(NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
        WHEN NVL(NORMAL_HOURS, 0) > 0 AND NVL(NORMAL_HOURS, 0) < 40 THEN 'Part Time'
        ELSE 'Not Specified'
    END AS EMPLOYMENT_TYPE
FROM HR_ASG_MASTER;
```
```

---

### Pattern 5: DFF Attribute Discovery and Handling (ADD AS NEW SECTION 14)

**Location in HR_MASTER.md:** Add as new **Section 14**

```markdown
## 14. 🔍 DFF (Descriptive Flexfield) Attributes

### 14.1 DFF Discovery Query

**Purpose:** Identify which ATTRIBUTE columns map to which business fields

**Discovery Query:**
```sql
SELECT 
    DFC.APPLICATION_COLUMN_NAME,      -- Physical column (ATTRIBUTE1, ATTRIBUTE2, etc.)
    DFC.END_USER_COLUMN_NAME,         -- Business name (Contract Type, Client Job Title, etc.)
    DFC.COLUMN_SEQ_NUM,               -- Display sequence
    DFC.DESCRIPTIVE_FLEXFIELD_NAME,   -- Flexfield name
    DFC.ENABLED_FLAG                  -- Whether enabled
FROM FND_DESCR_FLEX_COLUMN_USAGES DFC
WHERE DFC.APPLICATION_TABLE_NAME = 'PER_ALL_ASSIGNMENTS_F'
AND DFC.ENABLED_FLAG = 'Y'
AND DFC.DESCRIPTIVE_FLEXFIELD_NAME = 'Assignment Developer DF'  -- Common DFF name
ORDER BY DFC.COLUMN_SEQ_NUM;
```

**Sample Output:**
| APPLICATION_COLUMN_NAME | END_USER_COLUMN_NAME | COLUMN_SEQ_NUM |
|-------------------------|----------------------|----------------|
| ATTRIBUTE1              | Contract Type        | 1              |
| ATTRIBUTE2              | Employee Grade       | 2              |
| ATTRIBUTE3              | Project Number       | 3              |
| ATTRIBUTE4              | Service Line         | 4              |
| ATTRIBUTE5              | Client Job Title     | 5              |

### 14.2 DFF Usage Pattern

**After running discovery query, create mapping CTE:**
```sql
EMP_DFF AS (
    SELECT
        PAAF.PERSON_ID,
        PAAF.ASSIGNMENT_ID,
        -- Map based on discovery query results
        PAAF.ATTRIBUTE1 AS CONTRACT_TYPE,        -- Discovered mapping
        PAAF.ATTRIBUTE2 AS EMPLOYEE_GRADE_DFF,   -- Discovered mapping
        PAAF.ATTRIBUTE3 AS PROJECT_NUMBER,       -- Discovered mapping
        PAAF.ATTRIBUTE4 AS SERVICE_LINE,         -- Discovered mapping
        PAAF.ATTRIBUTE5 AS CLIENT_JOB_TITLE      -- Discovered mapping
    FROM PER_ALL_ASSIGNMENTS_F PAAF, PARAMETERS P
    WHERE PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PAAF.PRIMARY_FLAG = 'Y'
    AND PAAF.ASSIGNMENT_TYPE = 'E'
    AND P.EFFECTIVE_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
)
```

### 14.3 DFF Best Practices

1. **Always run discovery query first** - Don't guess attribute mappings
2. **Document mappings in comments** - Future developers will thank you
3. **Handle NULL values** - DFF fields are often optional
4. **Use meaningful aliases** - `CONTRACT_TYPE` not `ATTR1`
5. **Date-track filter** - DFF attributes can change over time

### 14.4 Common DFF Contexts

**For PER_ALL_ASSIGNMENTS_F:**
- Assignment Developer DF
- Assignment DFF
- ASG_DF (varies by implementation)

**For PER_ALL_PEOPLE_F:**
- Person Developer DF
- Person DFF
- PER_PEOPLE_DFF

**Discovery Query for All Contexts:**
```sql
SELECT DISTINCT
    DFC.DESCRIPTIVE_FLEXFIELD_NAME,
    DFC.APPLICATION_TABLE_NAME,
    COUNT(*) AS ATTRIBUTE_COUNT
FROM FND_DESCR_FLEX_COLUMN_USAGES DFC
WHERE DFC.ENABLED_FLAG = 'Y'
AND DFC.APPLICATION_TABLE_NAME IN ('PER_ALL_ASSIGNMENTS_F', 'PER_ALL_PEOPLE_F')
GROUP BY DFC.DESCRIPTIVE_FLEXFIELD_NAME, DFC.APPLICATION_TABLE_NAME
ORDER BY DFC.APPLICATION_TABLE_NAME, DFC.DESCRIPTIVE_FLEXFIELD_NAME;
```
```

---

### Pattern 6: Optional Table Handling (ADD TO SECTION 7)

**Location in HR_MASTER.md:** Add as **Section 7.7** under "Common Pitfalls"

```markdown
### 7.7 Handling Optional Tables

**Problem:** Query fails when table doesn't exist in all environments  
**Cause:** Hardcoded table joins to environment-specific tables

**Solution:** Use outer joins and provide documentation

**Pattern:**
```sql
-- Define optional CTE
OPTIONAL_DATA AS (
    SELECT
        PERSON_ID,
        OPTIONAL_FIELD
    FROM OPTIONAL_TABLE  -- May not exist in all environments
    WHERE conditions
)

-- In final SELECT, use outer join
FROM
    REQUIRED_TABLE RT,
    OPTIONAL_DATA OD
WHERE
    RT.PERSON_ID = OD.PERSON_ID(+)  -- Outer join allows NULL when table missing or empty

-- Use NVL in SELECT to handle NULL
SELECT
    RT.PERSON_NUMBER,
    NVL(OD.OPTIONAL_FIELD, 'Not Available') AS OPTIONAL_FIELD
FROM ...
```

**Deployment Documentation:**
```sql
/*
 * OPTIONAL TABLES - Environment-Specific
 * =====================================
 *
 * The following CTEs query tables that may not exist in all environments:
 * 
 * 1. EXTENDED_INFO_CTE - Queries CUSTOM_EMPLOYEE_INFO table
 *    - If table doesn't exist, comment out lines 45-60
 *    - Fields will show as NULL in output
 *
 * 2. PROJECT_ASSIGNMENTS_CTE - Queries PA_PROJECT_ASSIGNMENTS
 *    - If table doesn't exist, comment out lines 75-90
 *    - Project fields will show as 'Not Available'
 */
```

**Example: Multiple Optional Tables:**
```sql
-- CTE 1: Optional project data
EMP_PROJECTS AS (
    SELECT
        PERSON_ID,
        PROJECT_NAME,
        PROJECT_START_DATE
    FROM PA_PROJECT_ASSIGNMENTS  -- May not exist
    WHERE PROJECT_STATUS = 'ACTIVE'
)

-- CTE 2: Optional custom fields
EMP_CUSTOM AS (
    SELECT
        PERSON_ID,
        CUSTOM_FIELD_1,
        CUSTOM_FIELD_2
    FROM CUSTOM_EMPLOYEE_INFO  -- May not exist
    WHERE ACTIVE_FLAG = 'Y'
)

-- Final SELECT with outer joins
SELECT
    EB.PERSON_NUMBER,
    EB.FULL_NAME,
    NVL(EP.PROJECT_NAME, 'No Project') AS PROJECT_NAME,
    NVL(EC.CUSTOM_FIELD_1, 'N/A') AS CUSTOM_FIELD_1
FROM
    EMP_BASE EB,
    EMP_PROJECTS EP,
    EMP_CUSTOM EC
WHERE
    EB.PERSON_ID = EP.PERSON_ID(+)
AND EB.PERSON_ID = EC.PERSON_ID(+);
```
```

---

### Pattern 7: Multi-Parameter Filtering (UPDATE SECTION 8)

**Location in HR_MASTER.md:** Enhance **Section 8** with comprehensive parameter pattern

```markdown
### 8.5 Multi-Parameter Filtering Pattern

**Complete Parameter Handling Pattern:**

```sql
-- Step 1: Define ALL parameters in single CTE
WITH PARAMETERS AS (
    /*+ qb_name(PARAMETERS) */
    SELECT
        -- Mandatory parameters
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        
        -- Optional parameters with UPPER and 'ALL' default
        UPPER(NVL(:P_LEGAL_EMPLOYER, 'ALL')) AS LEGAL_EMPLOYER,
        UPPER(NVL(:P_DEPARTMENT, 'ALL')) AS DEPARTMENT,
        UPPER(NVL(:P_JOB_TITLE, 'ALL')) AS JOB_TITLE,
        UPPER(NVL(:P_EMPLOYEE_TYPE, 'ALL')) AS EMPLOYEE_TYPE,
        UPPER(NVL(:P_LOCATION, 'ALL')) AS LOCATION,
        UPPER(NVL(:P_GRADE, 'ALL')) AS GRADE
    FROM DUAL
)

-- Step 2: Join PARAMETERS to all CTEs
,EMP_BASE AS (
    SELECT ...
    FROM ..., PARAMETERS P
    WHERE P.EFFECTIVE_DATE BETWEEN ...
)

-- Step 3: Apply filters in WHERE clause
WHERE
    -- Legal Employer filter
    (UPPER(EA.LEGAL_EMPLOYER_NAME) = P.LEGAL_EMPLOYER OR P.LEGAL_EMPLOYER = 'ALL')
    -- Department filter
AND (UPPER(EA.DEPARTMENT_NAME) = P.DEPARTMENT OR P.DEPARTMENT = 'ALL')
    -- Job Title filter
AND (UPPER(EA.JOB_TITLE) = P.JOB_TITLE OR P.JOB_TITLE = 'ALL')
    -- Employee Type filter
AND (UPPER(EB.PERSON_TYPE) = P.EMPLOYEE_TYPE OR P.EMPLOYEE_TYPE = 'ALL')
    -- Location filter
AND (UPPER(EA.LOCATION_NAME) = P.LOCATION OR P.LOCATION = 'ALL')
    -- Grade filter
AND (UPPER(EA.GRADE_NAME) = P.GRADE OR P.GRADE = 'ALL')
```

**Benefits:**
- All parameters defined in one place
- Consistent case-insensitive matching
- Simple bypass logic with 'ALL'
- No complex NULL handling
- Easy to add new parameters

**Testing Parameters:**
```sql
-- Test 1: All employees (no filters)
:P_EFFECTIVE_DATE = '31-DEC-2024'
:P_LEGAL_EMPLOYER = 'ALL'
:P_DEPARTMENT = 'ALL'

-- Test 2: Specific department (case-insensitive)
:P_EFFECTIVE_DATE = '31-DEC-2024'
:P_LEGAL_EMPLOYER = 'ALL'
:P_DEPARTMENT = 'human resources'  -- Will match 'Human Resources', 'HUMAN RESOURCES', etc.

-- Test 3: Multiple filters
:P_EFFECTIVE_DATE = '31-DEC-2024'
:P_LEGAL_EMPLOYER = 'ABC CORPORATION LLC'
:P_DEPARTMENT = 'IT'
:P_LOCATION = 'Dubai'
```
```

---

### Pattern 8: Date Handling for Period of Service (UPDATE SECTION 8)

**Location in HR_MASTER.md:** Update **Section 8.2**

```markdown
### 8.2 Current Employees with Termination Date Handling (UPDATED)

**Old Pattern:**
```sql
AND TRUNC(SYSDATE) BETWEEN PPOS.DATE_START 
    AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('4712-12-31', 'YYYY-MM-DD'))
```

**New Pattern (with Effective Date):**
```sql
AND P.EFFECTIVE_DATE BETWEEN PPOS.DATE_START 
    AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
```

**Key Changes:**
1. Use `P.EFFECTIVE_DATE` instead of `SYSDATE`
2. Consistent date format: `'DD/MM/YYYY'`
3. End of time date: `31/12/4712`

**Why This Matters:**
- Enables historical queries ("Show terminated employees as of 31-DEC-2023")
- Consistent with date-track filtering pattern
- Handles active employees (ACTUAL_TERMINATION_DATE is NULL)

**Complete Pattern:**
```sql
WITH PARAMETERS AS (
    SELECT TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE
    FROM DUAL
)
,EMP_BASE AS (
    SELECT
        PAPF.PERSON_ID,
        PPOS.DATE_START AS HIRE_DATE,
        PPOS.ACTUAL_TERMINATION_DATE AS TERM_DATE,
        CASE 
            WHEN PPOS.ACTUAL_TERMINATION_DATE IS NULL THEN 'Active'
            WHEN PPOS.ACTUAL_TERMINATION_DATE > P.EFFECTIVE_DATE THEN 'Active'
            ELSE 'Terminated'
        END AS EMPLOYMENT_STATUS
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERIODS_OF_SERVICE PPOS,
        PARAMETERS P
    WHERE
        PAPF.PERSON_ID = PPOS.PERSON_ID
    -- Include employees who were active on Effective Date
    AND P.EFFECTIVE_DATE BETWEEN PPOS.DATE_START 
        AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
)
```
```

---

## 📊 INTEGRATION PRIORITY

### Priority 1: CRITICAL (Add Immediately)
1. **Pattern 1:** Effective Date Filtering → Section 1.5
2. **Pattern 3:** Case-Insensitive Parameter Filtering → Section 8.4

### Priority 2: HIGH VALUE (Within 2 Weeks)
3. **Pattern 2:** Service in Years Calculation → Section 5.5
4. **Pattern 4:** FT/PT Classification → Section 11.3
5. **Pattern 7:** Multi-Parameter Filtering → Section 8.5

### Priority 3: ENHANCEMENT (Within 1 Month)
6. **Pattern 5:** DFF Discovery → New Section 14
7. **Pattern 6:** Optional Table Handling → Section 7.7
8. **Pattern 8:** Period of Service Date Handling → Section 8.2 (update)

---

## ✅ VALIDATION CHECKLIST

Before integrating these patterns:

- [ ] Review all 8 new patterns
- [ ] Test Effective Date pattern with historical data
- [ ] Verify case-insensitive filtering with sample data
- [ ] Test service calculation formula accuracy
- [ ] Run DFF discovery query in target environment
- [ ] Document any environment-specific customizations
- [ ] Update training materials
- [ ] Create example queries for each pattern

---

## 📚 REFERENCE

### Source Documentation
- File: `Requirement\Employee_Annual_Leave_Balance_Query.sql` (lines 1-549)
- Summary: `Requirement\Query_Summary.md`
- ABSENCE Update: `SaaS-main\HCM\ABSENCE\ABSENCE_MASTER_UPDATE_31-12-25.md`

### Related Updates
- `HR_REPOSITORIES_UPDATE_02-01-26.md` - CTE implementations
- `HCM_KB_UPDATE_SUMMARY_31-12-25.md` - Original cross-module analysis

---

**END OF HR_MASTER_UPDATE_02-01-26.md**

**Status:** Ready for Integration  
**Priority:** HIGH  
**Next Action:** Add patterns to HR_MASTER.md in specified sections

**Author:** AI Assistant  
**Date:** 02-01-2026  
**Version:** 1.0
