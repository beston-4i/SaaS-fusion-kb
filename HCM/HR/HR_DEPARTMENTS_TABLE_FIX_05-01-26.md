# HR Master Instructions - PER_DEPARTMENTS Table Join Fix

**Update Date:** 05-01-2026  
**Priority:** HIGH  
**Update Type:** Data Model Correction  
**Affected Module:** All HCM queries using Departments

---

## 🚨 CRITICAL CORRECTION - PER_DEPARTMENTS Join

### Issue Identified
**WRONG JOIN COLUMN:**
```sql
-- ❌ INCORRECT - DEPARTMENT_ID does NOT exist in PER_DEPARTMENTS
AND PAAF.DEPARTMENT_ID = PD.DEPARTMENT_ID(+)
```

**Error:** `PER_DEPARTMENTS` table does NOT have a `DEPARTMENT_ID` column

---

## ✅ CORRECT PATTERN (MANDATORY)

### Table Structure
- **PER_DEPARTMENTS**: Uses `ORGANIZATION_ID` as primary key (NOT `DEPARTMENT_ID`)
- **PER_ALL_ASSIGNMENTS_F**: Contains `ORGANIZATION_ID` for department link
- **Join Column**: `ORGANIZATION_ID` (not `DEPARTMENT_ID`)

### Correct Implementation
```sql
-- ✅ CORRECT - Join using ORGANIZATION_ID
FROM
    PER_ALL_ASSIGNMENTS_F PAAF,
    PER_DEPARTMENTS PD
WHERE
    PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
```

---

## 📋 TABLE STRUCTURE FACTS

### PER_DEPARTMENTS Table
- **Primary Key Column**: `ORGANIZATION_ID`
- **Key Columns**:
  - `ORGANIZATION_ID` - Primary identifier
  - `NAME` - Department name
  - `MANAGER_ID` - Department manager
  - `LOCATION_ID` - Location reference
  - `STATUS` - Active/Inactive status

**Important:** NO `DEPARTMENT_ID` column exists in this table!

---

## 🔍 CORRECT JOIN PATTERN

### Pattern 1: Single Department Join
```sql
SELECT
    PAPF.PERSON_NUMBER,
    PD.NAME AS DEPARTMENT_NAME
FROM
    PER_ALL_PEOPLE_F PAPF,
    PER_ALL_ASSIGNMENTS_F PAAF,
    PER_DEPARTMENTS PD
WHERE
    PAPF.PERSON_ID = PAAF.PERSON_ID
AND PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)  -- Correct join
AND PAAF.PRIMARY_FLAG = 'Y'
AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
```

### Pattern 2: Department AND Organization Unit
```sql
-- When you need both Department and Org Unit
FROM
    PER_ALL_ASSIGNMENTS_F PAAF,
    PER_DEPARTMENTS PD,
    HR_ALL_ORGANIZATION_UNITS ORG
WHERE
    PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)      -- Department
AND PAAF.ORGANIZATION_ID = ORG.ORGANIZATION_ID(+)    -- Org Unit
```

**Note:** Both join to `PAAF.ORGANIZATION_ID` - this is correct!

---

## ⚠️ COMMON CONFUSION

### Why Two Tables Join to Same Column?

**PER_DEPARTMENTS** and **HR_ALL_ORGANIZATION_UNITS** both represent organizational structures:

- **PER_DEPARTMENTS**: Specific department information
- **HR_ALL_ORGANIZATION_UNITS**: General organization unit (can be dept, business unit, etc.)

Both use `ORGANIZATION_ID` as their primary key, and both can be joined to `PAAF.ORGANIZATION_ID`.

**Typical Usage:**
- Use `PER_DEPARTMENTS` when you specifically need department details
- Use `HR_ALL_ORGANIZATION_UNITS` for general organizational hierarchy

---

## 📊 COMPLETE CTE EXAMPLE

```sql
EMP_ASSIGNMENT AS (
    /*+ qb_name(EMP_ASSIGNMENT) MATERIALIZE */
    SELECT
        EB.PERSON_ID,
        EB.PERSON_NUMBER,
        PAAF.ASSIGNMENT_ID,
        -- Legal Entity
        LE.NAME AS LEGAL_ENTITY_NAME,
        -- Department
        PD.NAME AS DEPARTMENT_NAME,
        -- Organization Unit
        ORG.NAME AS ORG_UNIT_NAME,
        -- Grade
        PGFV.NAME AS GRADE_NAME,
        -- Position
        HAPL.NAME AS POSITION_NAME,
        -- Job
        PJFV.NAME AS JOB_NAME
    FROM
        EMP_BASE EB,
        PER_ALL_ASSIGNMENTS_F PAAF,
        HR_ALL_ORGANIZATION_UNITS LE,
        PER_DEPARTMENTS PD,
        HR_ALL_ORGANIZATION_UNITS ORG,
        PER_GRADES_F_VL PGFV,
        HR_ALL_POSITIONS_F_TL HAPL,
        PER_JOBS_F_VL PJFV,
        PARAMETERS P
    WHERE
        -- Join person to assignment
        EB.PERSON_ID = PAAF.PERSON_ID
        -- Legal Entity (different ID)
    AND PAAF.LEGAL_ENTITY_ID = LE.ORGANIZATION_ID(+)
        -- Department (uses ORGANIZATION_ID, not DEPARTMENT_ID)
    AND PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
        -- Org Unit (also uses ORGANIZATION_ID)
    AND PAAF.ORGANIZATION_ID = ORG.ORGANIZATION_ID(+)
        -- Other joins...
    AND PAAF.GRADE_ID = PGFV.GRADE_ID(+)
    AND PAAF.POSITION_ID = HAPL.POSITION_ID(+)
    AND PAAF.JOB_ID = PJFV.JOB_ID(+)
        -- Filters
    AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PAAF.PRIMARY_FLAG = 'Y'
    AND PAAF.ASSIGNMENT_TYPE = 'E'
    AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
)
```

---

## ✅ VALIDATION CHECKLIST

Before running any query with departments, verify:

- [ ] Department join uses `PD.ORGANIZATION_ID` (NOT `PD.DEPARTMENT_ID`)
- [ ] Join to `PAAF.ORGANIZATION_ID` (assignment table)
- [ ] Outer join `(+)` used if department is optional
- [ ] No reference to non-existent `DEPARTMENT_ID` column

---

## 🚨 AFFECTED QUERIES TO REVIEW

This fix must be applied to:

1. **Employee Leave Count Report** ✅ FIXED
2. **Employee Absence Report** - CHECK NEEDED
3. **Annual Leave Balance Report** - CHECK NEEDED
4. **Any HR reports joining departments**

---

## 📝 MASTER RULE UPDATE

**OLD RULE (WRONG):**
> Join PER_ALL_ASSIGNMENTS_F to PER_DEPARTMENTS using DEPARTMENT_ID

**NEW RULE (CORRECT):**
> Join PER_ALL_ASSIGNMENTS_F to PER_DEPARTMENTS using ORGANIZATION_ID
> Column: PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)

---

## 🔍 QUICK REFERENCE

### Assignment Table Links:

| Assignment Column | Target Table | Target Column | Description |
|-------------------|--------------|---------------|-------------|
| `LEGAL_ENTITY_ID` | `HR_ALL_ORGANIZATION_UNITS` | `ORGANIZATION_ID` | Legal entity |
| `ORGANIZATION_ID` | `PER_DEPARTMENTS` | `ORGANIZATION_ID` | Department |
| `ORGANIZATION_ID` | `HR_ALL_ORGANIZATION_UNITS` | `ORGANIZATION_ID` | Org unit |
| `GRADE_ID` | `PER_GRADES_F_VL` | `GRADE_ID` | Grade |
| `POSITION_ID` | `HR_ALL_POSITIONS_F_TL` | `POSITION_ID` | Position |
| `JOB_ID` | `PER_JOBS_F_VL` | `JOB_ID` | Job |

---

## 🚨 VALIDATION QUERY

Test if department join is correct:
```sql
-- This should return data:
SELECT 
    PAAF.PERSON_ID,
    PAAF.ORGANIZATION_ID,
    PD.NAME AS DEPARTMENT_NAME
FROM 
    PER_ALL_ASSIGNMENTS_F PAAF,
    PER_DEPARTMENTS PD
WHERE 
    PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID
AND PAAF.PRIMARY_FLAG = 'Y'
AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
AND ROWNUM = 1;
```

---

**IMPORTANT:** This is a critical data model correction that affects all queries joining to department information.

**Last Updated:** 05-01-2026  
**Status:** ACTIVE - Apply to all queries immediately  
**Related Fix:** CRITICAL_PERSON_TYPE_FIX_05-01-26.md
