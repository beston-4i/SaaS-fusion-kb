# Time & Labor Master Instructions

**Module:** Time & Labor (HWM + Custom ZMM)  
**Tag:** `#HCM #TL #HWM #Timesheet #Production`  
**Status:** Production-Ready  
**Last Updated:** 29-Jan-2026  
**Version:** 2.0 - Enhanced with Production Schedule/Shift Patterns

---

## 1. 🚨 Critical Time & Labor Constraints

*Violating these rules breaks timesheet calculations.*

### 1.1 Version Handling
**Rule:** Always use the LATEST version for time record groups

```sql
AND AUSG.USAGES_SOURCE_VERSION = (
    SELECT MAX(A1.USAGES_SOURCE_VERSION) 
    FROM HWM_TM_REP_ATRB_USAGES A1 
    WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
)
```

**Why:** Timesheets can be edited multiple times; only the latest version is current.

### 1.2 Date-Track Filtering
**Rule:** Always filter Time & Labor tables by `EFFECTIVE_START_DATE` and `EFFECTIVE_END_DATE`

```sql
AND TRUNC(TMD.START_TIME) BETWEEN P1.EFFECTIVE_START_DATE AND P1.EFFECTIVE_END_DATE
```

**Why:** Time records span multiple effective dates; must align with person/assignment context.

### 1.3 Status Filtering
**Rule:** Filter by `STATUS` to exclude drafts and rejected timesheets

```sql
AND TMH.STATUS IN ('Approved', 'Submitted')
```

**Status Values:**
- `'Approved'` - Approved by manager
- `'Submitted'` - Submitted, awaiting approval
- `'Saved'` - Draft (not submitted)

### 1.4 Absence Integration
**Rule:** When timesheet has absence types, cross-check with `ANC_PER_ABS_ENTRIES`

```sql
AND ((ABSE.ABSENCE_STATUS_CD = 'SUBMITTED' AND ABSE.APPROVAL_STATUS_CD = 'APPROVED') OR
     (ABSE.ABSENCE_STATUS_CD = 'ORA_WITHDRAWN' AND ABSE.APPROVAL_STATUS_CD = 'ORA_AWAIT_AWAIT') OR
     (ABSE.ABSENCE_STATUS_CD = 'SUBMITTED' AND ABSE.APPROVAL_STATUS_CD = 'AWAITING'))
```

**Why:** Timesheet absence entries must align with official absence records.

### 1.5 Project Attribute Handling
**Rule:** Use `ATTRIBUTE_CATEGORY` and `ATTRIBUTE_NUMBER1/2` for project/task linkage

```sql
AND ATR.ATTRIBUTE_CATEGORY = 'Projects'
AND ATR.ATTRIBUTE_NUMBER1 = PJP.PROJECT_ID
AND ATR.ATTRIBUTE_NUMBER2 = PJT.TASK_ID
```

**Why:** Time attributes are stored generically; category determines the entity type.

---

## 2. 🗺️ Schema Map

### 2.1 Time Record Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **TMH** | `HWM_TM_REC_SUMM` | Timesheet Header (Group Summary) |
| **TMD** | `HWM_TM_REP_HIERARCHIES_VL` | Timesheet Detail (Line Items) |
| **AUSG** | `HWM_TM_REP_ATRB_USAGES` | Attribute Usages (Project, Task, Expenditure Type linkage) |
| **ATR** | `HWM_TM_REP_ATRBS` | Attribute Definitions |
| **TRT** | `HWM_TM_REC_TYPES_VL` | Time Record Types |

### 2.2 Project Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PJP** | `PJF_PROJECTS_ALL_VL` | Projects |
| **PJT** | `PJF_TASKS_V` | Project Tasks |
| **PET** | `PJF_EXP_TYPES_TL` | Expenditure Types |
| **PPP** | `PJF_PROJECT_PARTIES` | Project Team Members |
| **PPR** | `PJF_PROJ_ROLE_TYPES_TL` | Project Roles |

### 2.3 Absence Tables (Integration)

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **ABSE** | `ANC_PER_ABS_ENTRIES` | Absence Entries |
| **ABST** | `ANC_ABSENCE_TYPES_F_TL` | Absence Types |
| **ABV** | `ANC_ABSENCE_TYPES_VL` | Absence Types View |

### 2.4 Person & Assignment Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PAPF** | `PER_ALL_PEOPLE_F` | Person Master |
| **PPNF** | `PER_PERSON_NAMES_F` | Person Names |
| **PAAM** / **ASG** | `PER_ALL_ASSIGNMENTS_M` | Assignments (Managed) |
| **PAAF** | `PER_ALL_ASSIGNMENTS_F` | Assignments (Date-Tracked) |
| **PASF** | `PER_ASSIGNMENT_SUPERVISORS_F` | Line Managers |

---

## 3. 📋 Attribute Patterns

### 3.1 Project Scalar Subquery

**Pattern:**
```sql
(SELECT PJP.SEGMENT1
 FROM HWM_TM_REP_ATRB_USAGES AUSG,
      HWM_TM_REP_ATRBS ATR,
      PJF_PROJECTS_ALL_VL PJP
 WHERE TMD.TM_REC_ID = AUSG.USAGES_SOURCE_ID
 AND AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
 AND ATR.ATTRIBUTE_CATEGORY = 'Projects'
 AND ATR.ATTRIBUTE_NUMBER1 = PJP.PROJECT_ID
 AND AUSG.USAGES_SOURCE_VERSION = (
     SELECT MAX(A1.USAGES_SOURCE_VERSION)
     FROM HWM_TM_REP_ATRB_USAGES A1
     WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
 )
 AND ROWNUM = 1
) PROJECT_NUMBER
```

### 3.2 Task Scalar Subquery

**Pattern:**
```sql
(SELECT PJT.TASK_NUMBER
 FROM HWM_TM_REP_ATRB_USAGES AUSG,
      HWM_TM_REP_ATRBS ATR,
      PJF_TASKS_V PJT
 WHERE TMD.TM_REC_ID = AUSG.USAGES_SOURCE_ID
 AND AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
 AND ATR.ATTRIBUTE_CATEGORY = 'Projects'
 AND ATR.ATTRIBUTE_NUMBER2 = PJT.TASK_ID
 AND AUSG.USAGES_SOURCE_VERSION = (
     SELECT MAX(A1.USAGES_SOURCE_VERSION)
     FROM HWM_TM_REP_ATRB_USAGES A1
     WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
 )
 AND ROWNUM = 1
) TASK_NUMBER
```

### 3.3 Expenditure Type Scalar Subquery

**Pattern:**
```sql
NVL(
  (SELECT D.EXPENDITURE_TYPE_NAME
   FROM HWM_TM_REP_ATRB_USAGES B,
        HWM_TM_REP_ATRBS C,
        PJF_EXP_TYPES_TL D
   WHERE TMD.TM_REC_ID = B.USAGES_SOURCE_ID
   AND B.TM_REP_ATRB_ID = C.TM_REP_ATRB_ID
   AND C.ATTRIBUTE_CATEGORY = TO_CHAR(D.EXPENDITURE_TYPE_ID)
   AND D.LANGUAGE = 'US'
   AND B.USAGES_SOURCE_VERSION = (
       SELECT MAX(A1.USAGES_SOURCE_VERSION)
       FROM HWM_TM_REP_ATRB_USAGES A1
       WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
   )
   AND ROWNUM = 1
  ),
  (SELECT D.NAME
   FROM HWM_TM_REP_ATRB_USAGES B,
        HWM_TM_REP_ATRBS C,
        ANC_ABSENCE_TYPES_VL D
   WHERE TMD.TM_REC_ID = B.USAGES_SOURCE_ID
   AND B.TM_REP_ATRB_ID = C.TM_REP_ATRB_ID
   AND C.ATTRIBUTE_CATEGORY = TO_CHAR(D.ABSENCE_TYPE_ID)
   AND D.LANGUAGE = 'US'
   AND B.USAGES_SOURCE_VERSION = (
       SELECT MAX(A1.USAGES_SOURCE_VERSION)
       FROM HWM_TM_REP_ATRB_USAGES A1
       WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
   )
   AND ROWNUM = 1
  )
) EXPENDITURE_TYPE
```

**Note:** This pattern handles BOTH project expenditure types AND absence types.

### 3.4 Stat Holiday Pattern

**Use Case:** Identify public holidays from time records

**Pattern:**
```sql
(SELECT UPPER(MAX(C.ATTRIBUTE_CATEGORY))
 FROM HWM_TM_REP_ATRB_USAGES B,
      HWM_TM_REP_ATRBS C
 WHERE TMD.TM_REC_ID = B.USAGES_SOURCE_ID
 AND B.TM_REP_ATRB_ID = C.TM_REP_ATRB_ID
 AND UPPER(C.ATTRIBUTE_CATEGORY) LIKE 'STAT%HOLIDAY%'
 AND B.USAGES_SOURCE_VERSION = (
     SELECT MAX(A1.USAGES_SOURCE_VERSION)
     FROM HWM_TM_REP_ATRB_USAGES A1
     WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
 )
) STAT_HOLIDAY
```

### 3.5 Leave Project Reference Pattern

**Use Case:** Get reference from leave attributes

**Pattern:**
```sql
(SELECT C.ATTRIBUTE_CHAR1
 FROM HWM_TM_REP_ATRB_USAGES B,
      HWM_TM_REP_ATRBS C
 WHERE TMD.TM_REC_ID = B.USAGES_SOURCE_ID
 AND B.TM_REP_ATRB_ID = C.TM_REP_ATRB_ID
 AND C.ATTRIBUTE_CATEGORY LIKE 'ORA_ANC_%'
 AND B.USAGES_SOURCE_VERSION = (
     SELECT MAX(A1.USAGES_SOURCE_VERSION)
     FROM HWM_TM_REP_ATRB_USAGES A1
     WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
 )
 AND ROWNUM = 1
) LEAVE_PROJECT_REFERENCE
```

---

## 4. 🔗 Standard Joins (Copy-Paste Ready)

### 4.1 Timesheet Header to Detail Join

```sql
FROM HWM_TM_REC_SUMM TMH,
     HWM_TM_REP_HIERARCHIES_VL TMD
WHERE TMH.TM_REC_GRP_ID = TMD.TM_REC_GRP_ID
AND TMH.STATUS IN ('Approved', 'Submitted')
```

### 4.2 Timesheet to Project Join (via Attributes)

```sql
FROM HWM_TM_REP_HIERARCHIES_VL TMD,
     HWM_TM_REP_ATRB_USAGES AUSG,
     HWM_TM_REP_ATRBS ATR,
     PJF_PROJECTS_ALL_VL PJP
WHERE TMD.TM_REC_ID = AUSG.USAGES_SOURCE_ID
AND AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
AND ATR.ATTRIBUTE_CATEGORY = 'Projects'
AND ATR.ATTRIBUTE_NUMBER1 = PJP.PROJECT_ID
AND AUSG.USAGES_SOURCE_VERSION = (
    SELECT MAX(A1.USAGES_SOURCE_VERSION)
    FROM HWM_TM_REP_ATRB_USAGES A1
    WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
)
```

### 4.3 Timesheet to Task Join (via Attributes)

```sql
FROM HWM_TM_REP_HIERARCHIES_VL TMD,
     HWM_TM_REP_ATRB_USAGES AUSG,
     HWM_TM_REP_ATRBS ATR,
     PJF_TASKS_V PJT
WHERE TMD.TM_REC_ID = AUSG.USAGES_SOURCE_ID
AND AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
AND ATR.ATTRIBUTE_CATEGORY = 'Projects'
AND ATR.ATTRIBUTE_NUMBER2 = PJT.TASK_ID
AND AUSG.USAGES_SOURCE_VERSION = (
    SELECT MAX(A1.USAGES_SOURCE_VERSION)
    FROM HWM_TM_REP_ATRB_USAGES A1
    WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
)
```

### 4.4 Timesheet to Line Manager Join

**Using Assignment Supervisor:**
```sql
FROM HWM_TM_REP_HIERARCHIES_VL TMD,
     HWM_TM_REC_SUMM TMH,
     PER_ALL_PEOPLE_F PAPF,
     PER_ALL_ASSIGNMENTS_M ASG,
     PER_ASSIGNMENT_SUPERVISORS_F PASF,
     PER_ALL_ASSIGNMENTS_M MGASG,
     PER_PERSON_NAMES_F MGNAME
WHERE TMH.TM_REC_GRP_ID = TMD.TM_REC_GRP_ID
AND TMD.PERSON_ID = PAPF.PERSON_ID
AND PAPF.PERSON_ID = ASG.PERSON_ID
AND ASG.ASSIGNMENT_ID = PASF.ASSIGNMENT_ID
AND PASF.MANAGER_ASSIGNMENT_ID = MGASG.ASSIGNMENT_ID
AND MGASG.PERSON_ID = MGNAME.PERSON_ID
AND PASF.MANAGER_TYPE = 'LINE_MANAGER'
AND ASG.PRIMARY_FLAG = 'Y'
AND ASG.EFFECTIVE_LATEST_CHANGE = 'Y'
AND MGNAME.NAME_TYPE = 'GLOBAL'
AND TRUNC(TMD.START_TIME) BETWEEN ASG.EFFECTIVE_START_DATE AND ASG.EFFECTIVE_END_DATE
AND TRUNC(TMD.START_TIME) BETWEEN PASF.EFFECTIVE_START_DATE AND PASF.EFFECTIVE_END_DATE
AND TRUNC(TMD.START_TIME) BETWEEN MGNAME.EFFECTIVE_START_DATE AND MGNAME.EFFECTIVE_END_DATE
```

**Using Project Manager (Fallback):**
```sql
FROM HWM_TM_REP_ATRB_USAGES AUSG,
     HWM_TM_REP_ATRBS ATR,
     PJF_TASKS_V PJT,
     PER_PERSON_NAMES_F P1
WHERE TMD.TM_REC_ID = AUSG.USAGES_SOURCE_ID
AND AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
AND ATR.ATTRIBUTE_CATEGORY = 'Projects'
AND ATR.ATTRIBUTE_NUMBER2 = PJT.TASK_ID
AND PJT.TASK_MANAGER_PERSON_ID = P1.PERSON_ID
AND P1.NAME_TYPE = 'GLOBAL'
AND TRUNC(TMD.START_TIME) BETWEEN P1.EFFECTIVE_START_DATE AND P1.EFFECTIVE_END_DATE
```

### 4.5 Timesheet to Absence Join (Cross-Check)

```sql
FROM HWM_TM_REP_HIERARCHIES_VL TMD,
     ANC_PER_ABS_ENTRIES ABSE,
     ANC_ABSENCE_TYPES_F_TL ABST
WHERE ABSE.PERSON_ID = TMD.PERSON_ID
AND ABSE.ABSENCE_TYPE_ID = ABST.ABSENCE_TYPE_ID
AND ABST.LANGUAGE = 'US'
AND TRUNC(TMD.START_TIME) BETWEEN TRUNC(ABSE.START_DATE) AND TRUNC(ABSE.END_DATE)
AND TRUNC(TMD.START_TIME) BETWEEN TRUNC(ABST.EFFECTIVE_START_DATE) AND TRUNC(ABST.EFFECTIVE_END_DATE)
AND ((ABSE.ABSENCE_STATUS_CD = 'SUBMITTED' AND ABSE.APPROVAL_STATUS_CD = 'APPROVED') OR
     (ABSE.ABSENCE_STATUS_CD = 'ORA_WITHDRAWN' AND ABSE.APPROVAL_STATUS_CD = 'ORA_AWAIT_AWAIT') OR
     (ABSE.ABSENCE_STATUS_CD = 'SUBMITTED' AND ABSE.APPROVAL_STATUS_CD = 'AWAITING') OR
     (ABSE.ABSENCE_STATUS_CD = 'SAVED'))
AND UPPER(ABST.NAME) = UPPER(TMD.EXPENDITURE_TYPE)  -- Match by name
```

---

## 5. 📊 Standard Filters

### 5.1 Date Range Filtering

**Timesheet Group Level:**
```sql
AND TMH.START_TIME >= :P_START_DATE
AND TMH.STOP_TIME <= :P_END_DATE
```

**Timesheet Detail Level:**
```sql
AND TMD.START_TIME >= TO_DATE(:P_START_DATE, 'YYYY-MM-DD')
AND TMD.STOP_TIME <= TO_DATE(:P_END_DATE, 'YYYY-MM-DD')
```

### 5.2 Status Filtering

**Approved Only:**
```sql
AND TMH.STATUS = 'Approved'
```

**Submitted or Approved:**
```sql
AND TMH.STATUS IN ('Approved', 'Submitted')
```

### 5.3 Project Filtering

**By Project Number:**
```sql
AND PJP.SEGMENT1 = :P_PROJECT_NUMBER
```

**By Project Name:**
```sql
AND UPPER(PJP.NAME) LIKE UPPER('%' || :P_PROJECT_NAME || '%')
```

### 5.4 Hours Calculation

**Total Hours (Recorded + Absence):**
```sql
NVL(TMH.RECORDED_HOURS, 0) + NVL(TMH.ABSENCE_HOURS, 0) TOTAL_HOURS
```

---

## 6. ⚠️ Common Pitfalls

### 6.1 Using Old Versions
**Problem:** Getting incorrect project/task/expenditure type  
**Cause:** Not filtering for latest `USAGES_SOURCE_VERSION`

**Solution:**
```sql
AND AUSG.USAGES_SOURCE_VERSION = (
    SELECT MAX(A1.USAGES_SOURCE_VERSION)
    FROM HWM_TM_REP_ATRB_USAGES A1
    WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
)
```

### 6.2 Missing Date Context
**Problem:** Cartesian products, wrong person/manager names  
**Cause:** Not filtering date-tracked tables with `START_TIME`

**Solution:**
```sql
AND TRUNC(TMD.START_TIME) BETWEEN P1.EFFECTIVE_START_DATE AND P1.EFFECTIVE_END_DATE
```

### 6.3 Assuming Single Approver
**Problem:** NULLs for approver when multiple approvers exist  
**Cause:** Not using `ROWNUM = 1`

**Solution:**
```sql
AND ROWNUM = 1  -- Take first approver
```

### 6.4 Mixing Expenditure Types and Absence Types
**Problem:** Getting only one type  
**Cause:** Not using `NVL` fallback pattern

**Solution:** Use the combined pattern (see Section 3.3)

---

## 7. 💡 Calculation Patterns

### 7.1 Line Status Derivation

**Complex logic for combining timesheet status and absence status:**

```sql
CASE
    WHEN PROJECT IS NOT NULL THEN NVL(LINE_STATUS1, STATUS)
    WHEN ABS_COUNT > 0 THEN (
        SELECT CASE
            WHEN ABSENCE_STATUS_CD = 'SUBMITTED' AND APPROVAL_STATUS_CD = 'APPROVED' THEN 'Approved'
            WHEN ABSENCE_STATUS_CD = 'ORA_WITHDRAWN' AND APPROVAL_STATUS_CD = 'ORA_AWAIT_AWAIT' THEN 'Approved'
            WHEN ABSENCE_STATUS_CD = 'SUBMITTED' AND APPROVAL_STATUS_CD = 'AWAITING' THEN 'Submitted'
            WHEN ABSENCE_STATUS_CD = 'SAVED' THEN 'Saved'
        END
        FROM ANC_PER_ABS_ENTRIES ABSE
        WHERE ABSE.PERSON_ID = X.PERSON_ID
        AND TRUNC(X.SEQ_DATE) BETWEEN TRUNC(ABSE.START_DATE) AND TRUNC(ABSE.END_DATE)
        AND ROWNUM = 1
    )
END LINE_STATUS
```

### 7.2 Approval Date Logic

**Only show approval date when status is Approved:**
```sql
CASE WHEN TMH.STATUS = 'Approved' THEN TMH.LAST_UPDATE_DATE END APPROVAL_DATE
```

---

## 8. 📅 Parameters

| Parameter | Format | Description | Example |
|-----------|--------|-------------|---------|
| `:P_START_DATE` | Date | Start date | TO_DATE('01-12-2024','DD-MM-YYYY') |
| `:P_END_DATE` | Date | End date | TO_DATE('31-12-2024','DD-MM-YYYY') |
| `:P_EMP_NO` | String | Employee number | '12345' |
| `:P_PROJECT_NUMBER` | String | Project number | 'PRJ001' |
| `:P_STATUS` | String | Timesheet status | 'Approved' |
| `:P_DEPT` | String | Department name | 'Finance' |

---

**Last Updated:** 29-Jan-2026  
**Status:** Production-Ready  
**Source:** Employee Timesheet Report (980 lines) + OTL Production Queries (10 files) + Missing Timecard & Overtime Reports (3,500+ lines analyzed)  
**Version:** 2.0 - Enhanced with Production Schedule/Shift Patterns

---

## 9. 🚀 Advanced OTL Patterns (07-Jan-2026)

### 9.1 HWM_TM_REP_ATRB_USAGES Latest Version Pattern

**Problem:** Get latest attribute usage version for timecard

**Solution:**

```sql
SELECT
    TMD.TM_REC_ID,
    
    -- Project (latest version)
    (SELECT PJP.SEGMENT1
     FROM
         HWM_TM_REP_ATRB_USAGES AUSG,
         HWM_TM_REP_ATRBS ATR,
         PJF_PROJECTS_ALL_VL PJP
     WHERE
         TMD.TM_REC_ID = AUSG.USAGES_SOURCE_ID
         AND AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
         AND ATR.ATTRIBUTE_CATEGORY = 'Projects'
         AND ATR.ATTRIBUTE_NUMBER1 = PJP.PROJECT_ID
         
         -- CRITICAL: Get latest usage version
         AND AUSG.USAGES_SOURCE_VERSION = (
             SELECT MAX(A1.USAGES_SOURCE_VERSION)
             FROM HWM_TM_REP_ATRB_USAGES A1
             WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
         )
         AND ROWNUM = 1
    ) PROJECT_NUMBER
FROM HWM_EXT_TIMECARD_DETAIL_V TMD
```

**Why Critical:**
- Timecard attributes can be updated/versioned
- Must use `MAX(USAGES_SOURCE_VERSION)` to get latest
- Without this, may get old/deleted attribute values

**Attribute Categories:**
- `'Projects'` - Project attributes (ATTRIBUTE_NUMBER1 = PROJECT_ID, ATTRIBUTE_NUMBER2 = TASK_ID)
- `TO_CHAR(EXPENDITURE_TYPE_ID)` - Expenditure type
- `TO_CHAR(ABSENCE_TYPE_ID)` - Absence type
- `'ORA_CUSTOM'` - Custom attributes
- `'STAT HOLIDAY'` - Stat holiday attribute

### 9.2 Custom Attribute Detection (Leave Project Reference)

**Problem:** Detect if timecard is for leave project vs regular project

**Solution:**

```sql
-- Leave project reference
NVL(
    -- Check for custom attribute indicating leave/fringe
    (SELECT DECODE(C.ATTRIBUTE_VARCHAR1, '1', 'Fringe', C.ATTRIBUTE_VARCHAR1)
     FROM
         HWM_TM_REP_ATRB_USAGES B,
         HWM_TM_REP_ATRBS C
     WHERE
         TMD.TM_REC_ID = B.USAGES_SOURCE_ID
         AND B.TM_REP_ATRB_ID = C.TM_REP_ATRB_ID
         AND C.ATTRIBUTE_CATEGORY = 'ORA_CUSTOM'
         AND C.ATTRIBUTE_VARCHAR1 IN ('1', 'O003 | Regular Time', 'O009 | Regular Time')
         AND B.USAGES_SOURCE_VERSION = (
             SELECT MAX(A1.USAGES_SOURCE_VERSION)
             FROM HWM_TM_REP_ATRB_USAGES A1
             WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
         )
    ),
    -- Otherwise, show project | task reference
    (SELECT PJP.SEGMENT1 || ' | ' || PJT.TASK_NUMBER
     FROM
         HWM_TM_REP_ATRB_USAGES B,
         HWM_TM_REP_ATRBS C,
         PJF_PROJECTS_ALL_VL PJP,
         PJF_TASKS_V PJT
     WHERE
         TMD.TM_REC_ID = B.USAGES_SOURCE_ID
         AND B.TM_REP_ATRB_ID = C.TM_REP_ATRB_ID
         AND C.ATTRIBUTE_CATEGORY = 'ORA_CUSTOM'
         AND C.ATTRIBUTE_VARCHAR1 NOT IN ('1', 'O003 | Regular Time')
         AND C.ATTRIBUTE_VARCHAR1 = TO_CHAR(PJT.TASK_ID)
         AND PJT.PROJECT_ID = PJP.PROJECT_ID
         AND B.USAGES_SOURCE_VERSION = (
             SELECT MAX(A1.USAGES_SOURCE_VERSION)
             FROM HWM_TM_REP_ATRB_USAGES A1
             WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
         )
    )
) LEAVE_PROJECT_REFERENCE
```

**Purpose:** Distinguish between regular time (non-project, fringe), leave projects (absence tracking via projects), and actual project work

### 9.3 Payroll Time Type from Timecard

**Problem:** Get payroll time type assigned to timecard

**Solution:**

```sql
-- Payroll time type
(SELECT A.PAY_PAYROLL_TIME_TYPE
 FROM HWM_TM_REP_M_PTT_ATRBS_V A
 WHERE A.USAGES_SOURCE_ID = TMD.TM_REC_ID
 AND A.USAGES_SOURCE_VERSION = (
     SELECT MAX(A1.USAGES_SOURCE_VERSION)
     FROM HWM_TM_REP_ATRB_USAGES A1
     WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
 )
) PAYROLL_TIME_TYPE
```

**Table:** `HWM_TM_REP_M_PTT_ATRBS_V`  
**Purpose:** View that joins timecard attributes to payroll time types

### 9.4 Stat Holiday Detection

**Solution:**

```sql
-- Stat holiday detection
(SELECT UPPER(MAX(C.ATTRIBUTE_CATEGORY))
 FROM
     HWM_TM_REP_ATRB_USAGES B,
     HWM_TM_REP_ATRBS C
 WHERE
     TMD.TM_REC_ID = B.USAGES_SOURCE_ID
     AND B.TM_REP_ATRB_ID = C.TM_REP_ATRB_ID
     AND UPPER(C.ATTRIBUTE_CATEGORY) LIKE 'STAT%HOLIDAY%'
     AND B.USAGES_SOURCE_VERSION = (
         SELECT MAX(A1.USAGES_SOURCE_VERSION)
         FROM HWM_TM_REP_ATRB_USAGES A1
         WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
     )
) STAT_HOLIDAY
```

### 9.5 Project-Based Approval Hierarchy

**Problem:** Determine approver based on project assignment (task manager → project manager → line manager)

**Approval Hierarchy:**
```
1. Task Manager (if project task assigned)
   ↓ (if not found)
2. Project Manager (if project assigned)
   ↓ (if not found or not project time)
3. Line Manager (default)
```

**Solution:** See TL_OTL_COMPREHENSIVE_GUIDE for complete implementation

### 9.6 Project Status Integration

**Problem:** Get project costing status for timecard line

**Solution:**

```sql
-- Project status (from HWM_TM_A_APP_STATUS_PJC_V)
(SELECT DISTINCT INITCAP(PJCS.STATUS_VALUE)
 FROM HWM_TM_A_APP_STATUS_PJC_V PJCS
 WHERE PJCS.TM_BLDG_BLK_ID = TMD.TM_REC_ID
 AND PJCS.TM_BLDG_BLK_VERSION = TMD.TM_REC_VERSION
) LINE_STATUS_PROJECT
```

**Tables:**
- `HWM_TM_A_APP_STATUS_PJC_V` - Timecard-to-project costing approval status
- Links timecard to project costing approval workflow

### 9.7 Critical OTL Tables

#### HWM_TM_REP_ATRB_USAGES (Timecard Attribute Usages)

**Purpose:** Link timecard records to attributes (projects, tasks, expenditure types)

**Critical Columns:**
- `USAGES_SOURCE_ID` - TM_REC_ID (timecard line)
- `USAGES_SOURCE_VERSION` - Version number (CRITICAL: use MAX)
- `TM_REP_ATRB_ID` - Links to HWM_TM_REP_ATRBS

**Why Versioning Matters:**
- Timecards can be edited
- Each edit creates new version
- Old versions retained
- **MUST** use MAX(USAGES_SOURCE_VERSION)

#### HWM_TM_REP_ATRBS (Timecard Attributes)

**Purpose:** Define timecard attributes

**Critical Columns:**
- `TM_REP_ATRB_ID` - Primary key
- `ATTRIBUTE_CATEGORY` - Category/type
- `ATTRIBUTE_NUMBER1` - First numeric attribute (e.g., PROJECT_ID)
- `ATTRIBUTE_NUMBER2` - Second numeric attribute (e.g., TASK_ID)
- `ATTRIBUTE_VARCHAR1` - First character attribute

**Common Categories:**
- `'Projects'` - Project/task attributes
- `TO_CHAR(EXPENDITURE_TYPE_ID)` - Expenditure type
- `TO_CHAR(ABSENCE_TYPE_ID)` - Absence type
- `'ORA_CUSTOM'` - Custom attributes
- `'STAT HOLIDAY'` - Stat holiday

#### HWM_TM_REP_M_PTT_ATRBS_V (Payroll Time Type View)

**Purpose:** View linking timecard to payroll time types

**Columns:**
- `USAGES_SOURCE_ID` - TM_REC_ID
- `USAGES_SOURCE_VERSION` - Version
- `PAY_PAYROLL_TIME_TYPE` - Payroll time type name

#### HWM_TM_A_APP_STATUS_PJC_V (Project Costing Approval Status)

**Purpose:** Timecard-to-project costing approval status

**Columns:**
- `TM_BLDG_BLK_ID` - TM_REC_ID or TM_REC_GRP_ID
- `TM_BLDG_BLK_VERSION` - Version
- `STATUS_VALUE` - Status (e.g., 'Approved', 'Rejected')

---

## 9. 🏭 Production Patterns (Custom Schedule/Shift Reports)

**Source:** 5 Production Reports (3,500+ lines analyzed - 29-Jan-2026)  
**Status:** Production-Proven Patterns

### 9.1 🚨 CRITICAL: Custom Schedule Tables (ZMM Prefix)

**⚠️ Your environment uses CUSTOM schedule/shift tables. MUST be used for all schedule/shift-based reports:**

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| **ZMM_SR_SCHEDULES_VL** | Schedule Master | `SCHEDULE_ID`, `SCHEDULE_NAME` |
| **ZMM_SR_SCHEDULE_DTLS** | Shift Date/Time Details | `SCHEDULE_ID`, `SHIFT_ID`, `START_DATE_TIME`, `END_DATE_TIME` |
| **ZMM_SR_SHIFTS_VL** | Shift Definitions | `SHIFT_ID`, `SHIFT_NAME`, `DURATION_MS_NUM`, `START_TIME_MS_NUM`, `DELETED_FLAG` |
| ⭐ **ZMM_SR_AVAILABLE_DATES** | Working Dates Calendar | `SCHEDULE_ID`, `CALENDAR_DATE` |
| **PER_SCHEDULE_ASSIGNMENTS** | Employee-Schedule Link | `RESOURCE_ID`, `SCHEDULE_ID`, `RESOURCE_TYPE`, `PRIMARY_FLAG`, `START_DATE`, `END_DATE` |

### 9.2 Dual Schedule Assignment Pattern (CRITICAL)

**⚠️ CRITICAL:** Schedules assigned at TWO levels - YOU MUST USE UNION:

1. **Legal Entity Level** (`RESOURCE_TYPE = 'LEGALEMP'`)
2. **Assignment Level** (`RESOURCE_TYPE = 'ASSIGN'`) - **OVERRIDES Legal Entity**

**Pattern:**
```sql
-- ALWAYS use this UNION pattern for schedule assignments
PERSON_WITH_SCHEDULE AS (
    -- Part 1: Legal Entity Level Schedules
    SELECT
        PAPF.PERSON_ID,
        PAAF.ASSIGNMENT_ID,
        PSA.SCHEDULE_ID,
        ZSSV.SCHEDULE_NAME,
        CASE WHEN TRUNC(PAPF.START_DATE) > TRUNC(PSA.START_DATE) 
             THEN TRUNC(PAPF.START_DATE) 
             ELSE TRUNC(PSA.START_DATE) 
        END START_DATE_LE,
        TRUNC(PSA.END_DATE) END_DATE_LE,
        NULL START_DATE,
        NULL END_DATE
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_SCHEDULE_ASSIGNMENTS PSA,
        ZMM_SR_SCHEDULES_VL ZSSV
    WHERE PAAF.LEGAL_ENTITY_ID = PSA.RESOURCE_ID      -- LE link
      AND PSA.SCHEDULE_ID = ZSSV.SCHEDULE_ID
      AND PSA.RESOURCE_TYPE = 'LEGALEMP'
      AND PSA.PRIMARY_FLAG = 'Y'
      -- Exclude if assignment-level schedule exists
      AND NOT EXISTS (
          SELECT 1
          FROM PER_SCHEDULE_ASSIGNMENTS PSA2,
               ZMM_SR_AVAILABLE_DATES Z
          WHERE PAAF.ASSIGNMENT_ID = PSA2.RESOURCE_ID
            AND PSA2.RESOURCE_TYPE = 'ASSIGN'
            AND PSA2.PRIMARY_FLAG = 'Y'
            AND PSA2.SCHEDULE_ID = Z.SCHEDULE_ID
            AND TRUNC(Z.CALENDAR_DATE) = TRUNC(:P_DATE)
            AND TRUNC(Z.CALENDAR_DATE) BETWEEN TRUNC(PSA2.START_DATE) AND TRUNC(PSA2.END_DATE)
      )

    UNION ALL

    -- Part 2: Assignment Level Schedules (overrides LE)
    SELECT
        PAPF.PERSON_ID,
        PAAF.ASSIGNMENT_ID,
        PSA.SCHEDULE_ID,
        ZSSV.SCHEDULE_NAME,
        NULL START_DATE_LE,
        NULL END_DATE_LE,
        CASE WHEN TRUNC(PAPF.START_DATE) > TRUNC(PSA.START_DATE) 
             THEN TRUNC(PAPF.START_DATE) 
             ELSE TRUNC(PSA.START_DATE) 
        END START_DATE,
        TRUNC(PSA.END_DATE) END_DATE
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_SCHEDULE_ASSIGNMENTS PSA,
        ZMM_SR_SCHEDULES_VL ZSSV
    WHERE PAAF.ASSIGNMENT_ID = PSA.RESOURCE_ID        -- Assignment link
      AND PSA.SCHEDULE_ID = ZSSV.SCHEDULE_ID
      AND PSA.RESOURCE_TYPE = 'ASSIGN'
      AND PSA.PRIMARY_FLAG = 'Y'
)
```

### 9.3 HWM_TM_REC 3-Level Version Control (PRODUCTION)

**⚠️ CRITICAL:** HWM_TM_REC requires 3-level version filtering (not just LATEST_VERSION):

```sql
TIME_RECORDS AS (
    SELECT
        /*+ index(HTR, HWM_TM_REC_U1) */
        /*+ index(HTRGU, HWM_TM_REC_GRP_USAGES_U1) */
        /*+ index(DTRG, HWM_TM_REC_GRP_U1) */
        HTR.RESOURCE_ID,          -- PERSON_ID
        HTR.SUBRESOURCE_ID,       -- ASSIGNMENT_ID
        TRUNC(DTRG.START_TIME) DAY_START_TIME,
        HTR.MEASURE               -- Hours
    FROM 
        HWM_TM_REC HTR,
        HWM_TM_REC_GRP_USAGES HTRGU,
        HWM_TM_REC_GRP DTRG,
        HWM_TM_REC_GRP DTRG1,
        HWM_TM_D_TM_UI_STATUS_V HTDTUSV
    WHERE 
        -- Table joins
        HTRGU.TM_REC_GRP_ID = DTRG.TM_REC_GRP_ID
      AND HTRGU.TM_REC_GRP_VERSION = DTRG.TM_REC_GRP_VERSION
      AND DTRG.PARENT_TM_REC_GRP_ID = DTRG1.TM_REC_GRP_ID
      AND DTRG.PARENT_TM_REC_GRP_VERSION = DTRG1.TM_REC_GRP_VERSION
      AND HTR.TM_REC_ID = HTRGU.TM_REC_ID
      AND HTR.TM_REC_VERSION = HTRGU.TM_REC_VERSION
      AND DTRG1.TM_REC_GRP_ID = HTDTUSV.TM_BLDG_BLK_ID
      
      -- Standard filters (ALWAYS include)
      AND UPPER(HTR.UNIT_OF_MEASURE) IN ('UN', 'HR')
      AND DTRG1.GRP_TYPE_ID = 100              -- Daily group
      AND HTR.RESOURCE_TYPE = 'PERSON'
      AND HTR.DELETE_FLAG IS NULL
      AND HTR.LAYER_CODE = 'TIME_RPTD'
      AND TRUNC(HTR.STOP_TIME) IS NOT NULL
      
      -- Exclude midnight system entries
      AND (EXTRACT(HOUR FROM HTR.START_TIME) <> 00
        OR EXTRACT(MINUTE FROM HTR.START_TIME) <> 00)
      
      -- VERSION LEVEL 1: Latest version flag
      AND NVL(HTR.LATEST_VERSION,'Y') = 'Y'
      AND HTR.LATEST_VERSION = 'Y'
      
      -- VERSION LEVEL 2: Latest status version
      AND (HTDTUSV.TM_BLDG_BLK_VERSION, HTDTUSV.STATUS_ID) = (
          SELECT MAX(TM_BLDG_BLK_VERSION), MAX(STATUS_ID) 
          FROM HWM_TM_D_TM_UI_STATUS_V 
          WHERE TM_BLDG_BLK_ID = HTDTUSV.TM_BLDG_BLK_ID
      )
      
      -- VERSION LEVEL 3: Latest creation date (handles edits)
      AND HTR.CREATION_DATE = (
          SELECT MAX(HTR1.CREATION_DATE)
          FROM HWM_TM_REC HTR1,
               HWM_TM_REC_GRP_USAGES HTRGU1,
               HWM_TM_REC_GRP DTRG1,
               HWM_TM_REC_GRP DTRG2
          WHERE HTRGU1.TM_REC_GRP_ID = DTRG1.TM_REC_GRP_ID
            AND HTRGU1.TM_REC_GRP_VERSION = DTRG1.TM_REC_GRP_VERSION
            AND DTRG1.PARENT_TM_REC_GRP_ID = DTRG2.TM_REC_GRP_ID
            AND DTRG1.PARENT_TM_REC_GRP_VERSION = DTRG2.TM_REC_GRP_VERSION
            AND HTR1.TM_REC_ID = HTRGU1.TM_REC_ID
            AND HTR1.TM_REC_VERSION = HTRGU1.TM_REC_VERSION
            AND HTR1.LATEST_VERSION = 'Y'
            AND HTR1.DELETE_FLAG IS NULL
            AND HTR1.LAYER_CODE = 'TIME_RPTD'
            AND DTRG1.TM_REC_GRP_ID = DTRG.TM_REC_GRP_ID
            AND HTR1.RESOURCE_ID = HTR.RESOURCE_ID
      )
)
```

### 9.4 Missing Timecard Detection Pattern (NOT EXISTS)

**Pattern:**
```sql
-- Identify employees missing timecards
SELECT
    EMP.PERSON_NUMBER,
    WD.CALENDAR_DATE MISSING_DATE
FROM
    EMPLOYEES_WITH_SCHEDULE EMP,
    WORKING_DATES WD
WHERE EMP.SCHEDULE_ID = WD.SCHEDULE_ID
  -- NO timecard exists
  AND NOT EXISTS (
      SELECT 1
      FROM TIME_RECORDS TR
      WHERE TR.RESOURCE_ID = EMP.PERSON_ID
        AND TR.SUBRESOURCE_ID = EMP.ASSIGNMENT_ID
        AND TR.DAY_START_TIME = WD.CALENDAR_DATE
  )
  -- Exclude public holidays
  AND NOT EXISTS (
      SELECT 1
      FROM PUBLIC_HOLIDAYS PH
      WHERE PH.PERSON_ID = EMP.PERSON_ID
        AND WD.CALENDAR_DATE BETWEEN PH.START_DATE AND PH.END_DATE
  )
  -- Exclude absences
  AND NOT EXISTS (
      SELECT 1
      FROM ABSENCES ABS
      WHERE ABS.PERSON_ID = EMP.PERSON_ID
        AND WD.CALENDAR_DATE BETWEEN ABS.START_DATE AND ABS.END_DATE
  )
```

### 9.5 Working Dates Calendar Pattern

```sql
-- Get working dates from ZMM calendar
WORKING_DATES AS (
    SELECT 
        SCHEDULE_ID,
        TRUNC(CALENDAR_DATE) CALENDAR_DATE
    FROM ZMM_SR_AVAILABLE_DATES
    WHERE TRUNC(CALENDAR_DATE) BETWEEN :P_FROM_DATE AND :P_TO_DATE
)
```

### 9.6 Shift Details Pattern

```sql
-- Get shift details with time calculations
SHIFT_DETAILS AS (
    SELECT
        ZSST.SCHEDULE_ID,
        ZSST.SCHEDULE_NAME,
        TRUNC(ZSSTD.START_DATE_TIME) CALENDAR_DATE,
        ZSSTD.START_DATE_TIME,
        ZSSTD.END_DATE_TIME,
        ZSSV.SHIFT_NAME,
        (ZSSV.DURATION_MS_NUM/3600000) SHIFT_HRS,     -- MS to hours
        -- Day of week (1=Monday, 7=Sunday)
        DECODE(TRIM(TO_CHAR(ZSSTD.START_DATE_TIME, 'DAY', 'NLS_DATE_LANGUAGE = AMERICAN')), 
               'SUNDAY',7, 'MONDAY',1, 'TUESDAY',2, 'WEDNESDAY',3, 
               'THURSDAY',4, 'FRIDAY',5, 'SATURDAY',6) SHIFT_DAY_NUM
    FROM
        ZMM_SR_SCHEDULES_TL      ZSST,
        ZMM_SR_SCHEDULE_DTLS     ZSSTD,
        ZMM_SR_SHIFTS_VL         ZSSV
    WHERE ZSST.SCHEDULE_ID = ZSSTD.SCHEDULE_ID
      AND ZSST.LANGUAGE = 'US'
      AND ZSSTD.SHIFT_ID = ZSSV.SHIFT_ID  
      AND ZSSV.DELETED_FLAG = 'N'
      AND TRUNC(ZSSTD.START_DATE_TIME) BETWEEN :P_FROM_DATE AND :P_TO_DATE
)
```

### 9.7 Public Holiday by Location Pattern

```sql
-- Get holidays based on employee location
PUBLIC_HOLIDAY AS (
    SELECT DISTINCT 
        PAPF.PERSON_ID,
        TRUNC(PCE.START_DATE_TIME) START_DATE,
        TRUNC(PCE.END_DATE_TIME) END_DATE
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_CALENDAR_EVENTS PCE,
        PER_GEO_TREE_NODE_RF PNR,
        HR_LOCATIONS HR
    WHERE PCE.TREE_CODE = PNR.TREE_CODE
      AND PCE.TREE_STRUCTURE_CODE = PNR.TREE_STRUCTURE_CODE
      AND PAPF.PERSON_ID = PAAF.PERSON_ID
      AND PAAF.LOCATION_ID = HR.LOCATION_ID
      AND HR.COUNTRY = PNR.PK1_VALUE                  -- Geographic link
      AND PCE.CATEGORY = 'PH'                         -- Public Holiday
      AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
      AND TO_CHAR(PCE.START_DATE_TIME,'YYYY') = TO_CHAR(:P_FROM_DATE,'YYYY')
)
```

### 9.8 'ALL' Parameter Bypass Pattern

**Pattern for flexible filtering:**
```sql
WHERE 1=1
    -- Employee filter
    AND (PAPF.PERSON_ID IN (:P_PERSON_ID) OR 'ALL' IN (:P_PERSON_ID || 'ALL'))
    
    -- Department filter
    AND (HDORG.NAME IN (:P_DEPARTMENT) OR 'ALL' IN (:P_DEPARTMENT || 'ALL'))
    
    -- Status filter
    AND (HTDTUSV.STATUS_VALUE IN (:P_STATUS) OR 'ALL' IN (:P_STATUS || 'ALL'))
```

**Usage:** Pass 'ALL' to bypass filter, or specific value(s) to filter

### 9.9 Time Calculations (Milliseconds Conversion)

```sql
-- Shift duration in hours
(ZSSV.DURATION_MS_NUM / 3600000) SHIFT_HRS

-- Shift start time hour component
TRUNC(ROUND(ZSSV.START_TIME_MS_NUM/1000)/3600) SHIFT_HR

-- Shift start time minute component
TRUNC(MOD(ROUND(ZSSV.START_TIME_MS_NUM/1000), 3600)/60) SHIFT_MIN

-- Day of week (1=Monday, 7=Sunday)
DECODE(TRIM(TO_CHAR(date_column, 'DAY', 'NLS_DATE_LANGUAGE = AMERICAN')), 
       'SUNDAY',7, 'MONDAY',1, 'TUESDAY',2, 'WEDNESDAY',3, 
       'THURSDAY',4, 'FRIDAY',5, 'SATURDAY',6) DAY_NUM
```

### 9.10 Overtime Calculations

**Normal OT (Weekday - capped at 2 hours/day):**
```sql
CASE 
    WHEN (HOURS_WORKED - SHIFT_DURATION) > 2 THEN 2
    WHEN (HOURS_WORKED - SHIFT_DURATION) <= 2 THEN HOURS_WORKED - SHIFT_DURATION
    ELSE 0 
END NORMAL_OT
```

**Holiday OT (Weekend/PH - capped at 8 hours/day):**
```sql
CASE 
    WHEN HOURS_WORKED >= 8 THEN 8 
    ELSE HOURS_WORKED 
END HOLIDAY_OT
```

**Weekly OT Aggregation (45 hours regular, 35 hours Ramadan):**
```sql
CASE 
    WHEN TOTAL_WEEKLY_HOURS <= 45 THEN 0
    WHEN (TOTAL_WEEKLY_HOURS - 45) >= SUM_NORMAL_OT THEN SUM_NORMAL_OT
    ELSE TOTAL_WEEKLY_HOURS - 45
END WEEKLY_NORMAL_OT
```

### 9.11 Standard HWM_TM_REC Filters (ALWAYS Include)

```sql
WHERE 1=1
    AND UPPER(HTR.UNIT_OF_MEASURE) IN ('UN', 'HR')
    AND DTRG1.GRP_TYPE_ID = 100              -- Daily group
    AND HTR.RESOURCE_TYPE = 'PERSON'
    AND HTR.DELETE_FLAG IS NULL
    AND HTR.LAYER_CODE = 'TIME_RPTD'
    AND TRUNC(HTR.STOP_TIME) IS NOT NULL
    AND NVL(HTR.LATEST_VERSION,'Y') = 'Y'
    AND HTR.LATEST_VERSION = 'Y'
    -- Exclude midnight system entries
    AND (EXTRACT(HOUR FROM HTR.START_TIME) <> 00
      OR EXTRACT(MINUTE FROM HTR.START_TIME) <> 00)
```

### 9.12 Performance: Index Hints (CRITICAL)

**ALWAYS use index hints for HWM tables:**
```sql
SELECT
    /*+ index(HTR, HWM_TM_REC_U1) */
    /*+ index(HTRGU, HWM_TM_REC_GRP_USAGES_U1) */
    /*+ index(DTRG, HWM_TM_REC_GRP_U1) */
    /*+ index(PAAF, PER_ALL_ASSIGNMENTS_M_PK) */
    /*+ index(PSA, PER_SCHEDULE_ASSIGNMENTS_U1) */
    /*+ index(APAE, ANC_PER_ABS_ENTRIES_UK1) */
    ...
```

**Impact:** 10x-100x performance improvement on large datasets

### 9.13 Production Best Practices Summary

**For ALL Time & Labor Reports:**

✅ **ALWAYS use UNION** for schedule assignments (LEGALEMP + ASSIGN)  
✅ **Apply 3-level version control** for HWM_TM_REC (not just flag)  
✅ **Include index hints** for performance (6+ hints per query)  
✅ **Use ZMM_SR_AVAILABLE_DATES** for working dates calendar  
✅ **Exclude midnight entries** (system-generated, false positives)  
✅ **Exclude holidays by location** (geography tree)  
✅ **Exclude approved absences** (not withdrawn/denied)  
✅ **Use 'ALL' parameter bypass** for flexible filtering  
✅ **Apply standard HWM filters** (LAYER_CODE, DELETE_FLAG, etc.)  
✅ **Handle midnight crossover** in time calculations  

**Common Errors to Avoid:**

| ❌ WRONG | ✅ CORRECT | Impact |
|---------|-----------|--------|
| Only LEGALEMP or ASSIGN | UNION both types | Missing employees |
| Single version control | 3-level version control | Gets old/incorrect data |
| No index hints | Include 6+ index hints | 10x slower |
| Include midnight entries | Exclude 00:00:00 | False positives |
| No holiday exclusion | Exclude by location | False missing alerts |
| Direct joins to HWM_TM_REC | Use 4-table join pattern | Incomplete data |

**Reference:** See TL_REPOSITORIES.md Section 0 for production CTEs, TL_TEMPLATES.md Section 0 for complete missing timecard template.

---

## 10. 📚 Additional Resources

### OTL Knowledge Base Reference

For comprehensive Oracle Time & Labor (OTL) attendance and shift management patterns, refer to:
- **TL_OTL_COMPREHENSIVE_GUIDE** - Complete OTL implementation guide (50+ patterns)
- **TL_OTL_QUERY_TEMPLATES** - 8 ready-to-use OTL query templates
- **TL_OTL_KNOWLEDGE_SUMMARY** - Quick start guide and troubleshooting

**Note:** The OTL files focus on attendance tracking, shift management, and time clock operations, while TL_MASTER focuses on timesheet processing and project time tracking.

