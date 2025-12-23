# Time & Labor Repositories

**Module:** Time & Labor (HWM)  
**Tag:** `#HCM #TL #HWM #Repositories`  
**Status:** Production-Ready  
**Last Updated:** 18-Dec-2025

---

## Critical Rules

1. **NEVER** write fresh timesheet joins from scratch
2. **ALWAYS** copy these CTEs as-is
3. **ALWAYS** include `/*+ qb_name(NAME) MATERIALIZE */` hints
4. **ALWAYS** use MAX version filtering for attributes
5. **ALWAYS** filter by STATUS for header CTEs

---

## 1. Timesheet Header CTEs

### 1.1 TL_TIMESHEET_HEADER

**Purpose:** Get timesheet group summary with status and dates

**When to Use:** For timesheet listings, approval reports

```sql
TL_TS_HEADER AS (
    SELECT /*+ qb_name(TL_TS_HDR) MATERIALIZE */
           TMH.TM_REC_GRP_ID
          ,TMH.PERSON_ID
          ,TO_CHAR(TMH.START_TIME, 'DD-MON-YYYY') GRP_START_TIME
          ,TO_CHAR(TMH.STOP_TIME, 'DD-MON-YYYY') GRP_STOP_TIME
          ,TMH.STATUS
          ,NVL(TMH.RECORDED_HOURS, 0) + NVL(TMH.ABSENCE_HOURS, 0) RECORDED_HOURS
          ,TMH.SUBMISSION_DATE
          ,CASE WHEN TMH.STATUS = 'Approved' THEN TMH.LAST_UPDATE_DATE END APPROVAL_DATE
    FROM   HWM_TM_REC_SUMM TMH
    WHERE  TMH.STATUS IN ('Approved', 'Submitted')
      AND  TMH.START_TIME >= :P_START_DATE
      AND  TMH.STOP_TIME <= :P_END_DATE
)
```

---

## 2. Timesheet Detail CTEs

### 2.1 TL_TIMESHEET_DETAIL

**Purpose:** Get timesheet line items with times and measures

**When to Use:** For detailed timesheet reports

```sql
TL_TS_DETAIL AS (
    SELECT /*+ qb_name(TL_TS_DTL) MATERIALIZE */
           TMD.TM_REC_ID
          ,TMD.TM_REC_GRP_ID
          ,TMD.TM_REC_GRP_VERSION
          ,TMD.PERSON_ID
          ,TMD.GRP_TYPE_NAME
          ,TMD.TM_REC_TYPE
          ,TO_CHAR(TMD.START_TIME, 'YYYY-MM-DD') START_TIME
          ,TO_CHAR(TMD.STOP_TIME, 'DD-MON-YYYY') STOP_TIME
          ,TMD.MEASURE
          ,TMD.UNIT_OF_MEASURE
    FROM   HWM_TM_REP_HIERARCHIES_VL TMD
    WHERE  TMD.START_TIME >= TO_DATE(:P_START_DATE, 'YYYY-MM-DD')
      AND  TMD.STOP_TIME <= TO_DATE(:P_END_DATE, 'YYYY-MM-DD')
)
```

### 2.2 TL_TIMESHEET_FULL

**Purpose:** Combined header + detail in single CTE

**When to Use:** When both header and detail needed together

```sql
TL_TS_FULL AS (
    SELECT /*+ qb_name(TL_TS_FULL) MATERIALIZE */
           TMH.TM_REC_GRP_ID
          ,TMH.PERSON_ID
          ,TO_CHAR(TMH.START_TIME, 'DD-MON-YYYY') GRP_START_TIME
          ,TO_CHAR(TMH.STOP_TIME, 'DD-MON-YYYY') GRP_STOP_TIME
          ,TMH.STATUS
          ,NVL(TMH.RECORDED_HOURS, 0) + NVL(TMH.ABSENCE_HOURS, 0) RECORDED_HOURS
          ,TMH.SUBMISSION_DATE
          ,CASE WHEN TMH.STATUS = 'Approved' THEN TMH.LAST_UPDATE_DATE END APPROVAL_DATE
          ,TMD.TM_REC_ID
          ,TMD.TM_REC_GRP_VERSION
          ,TMD.GRP_TYPE_NAME
          ,TMD.TM_REC_TYPE
          ,TO_CHAR(TMD.START_TIME, 'YYYY-MM-DD') START_TIME
          ,TO_CHAR(TMD.STOP_TIME, 'DD-MON-YYYY') STOP_TIME
          ,TMD.MEASURE
          ,TMD.UNIT_OF_MEASURE
    FROM   HWM_TM_REC_SUMM TMH
          ,HWM_TM_REP_HIERARCHIES_VL TMD
    WHERE  TMH.TM_REC_GRP_ID = TMD.TM_REC_GRP_ID
      AND  TMH.STATUS IN ('Approved', 'Submitted')
      AND  TMH.START_TIME >= :P_START_DATE
      AND  TMH.STOP_TIME <= :P_END_DATE
)
```

---

## 3. Project Attribute CTEs

### 3.1 TL_PROJECT_ATTR

**Purpose:** Get project number and name from timesheet attributes

**When to Use:** For project time reporting

```sql
TL_PROJ_ATTR AS (
    SELECT /*+ qb_name(TL_PROJ) MATERIALIZE */
           AUSG.USAGES_SOURCE_ID TM_REC_ID
          ,PJP.SEGMENT1 PROJECT_NUMBER
          ,PJP.NAME PROJECT_NAME
          ,PJP.PROJECT_ID
    FROM   HWM_TM_REP_ATRB_USAGES AUSG
          ,HWM_TM_REP_ATRBS ATR
          ,PJF_PROJECTS_ALL_VL PJP
    WHERE  AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
      AND  ATR.ATTRIBUTE_CATEGORY = 'Projects'
      AND  ATR.ATTRIBUTE_NUMBER1 = PJP.PROJECT_ID
      AND  AUSG.USAGES_SOURCE_VERSION = (
           SELECT MAX(A1.USAGES_SOURCE_VERSION)
           FROM HWM_TM_REP_ATRB_USAGES A1
           WHERE A1.USAGES_SOURCE_ID = AUSG.USAGES_SOURCE_ID
      )
)
```

### 3.2 TL_TASK_ATTR

**Purpose:** Get task number and name from timesheet attributes

**When to Use:** For task-level time reporting

```sql
TL_TASK_ATTR AS (
    SELECT /*+ qb_name(TL_TASK) MATERIALIZE */
           AUSG.USAGES_SOURCE_ID TM_REC_ID
          ,PJT.TASK_NUMBER
          ,PJT.TASK_NAME
          ,PJT.TASK_ID
    FROM   HWM_TM_REP_ATRB_USAGES AUSG
          ,HWM_TM_REP_ATRBS ATR
          ,PJF_TASKS_V PJT
    WHERE  AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
      AND  ATR.ATTRIBUTE_CATEGORY = 'Projects'
      AND  ATR.ATTRIBUTE_NUMBER2 = PJT.TASK_ID
      AND  AUSG.USAGES_SOURCE_VERSION = (
           SELECT MAX(A1.USAGES_SOURCE_VERSION)
           FROM HWM_TM_REP_ATRB_USAGES A1
           WHERE A1.USAGES_SOURCE_ID = AUSG.USAGES_SOURCE_ID
      )
)
```

### 3.3 TL_EXPEND_TYPE_ATTR

**Purpose:** Get expenditure type or absence type from attributes

**When to Use:** For project costing, absence time tracking

```sql
TL_EXP_TYPE_ATTR AS (
    SELECT /*+ qb_name(TL_EXP) MATERIALIZE */
           AUSG.USAGES_SOURCE_ID TM_REC_ID
          ,NVL(PET.EXPENDITURE_TYPE_NAME, ABT.NAME) EXPENDITURE_TYPE
    FROM   HWM_TM_REP_ATRB_USAGES AUSG
          ,HWM_TM_REP_ATRBS ATR
          ,PJF_EXP_TYPES_TL PET
          ,ANC_ABSENCE_TYPES_VL ABT
    WHERE  AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
      AND  ATR.ATTRIBUTE_CATEGORY = TO_CHAR(PET.EXPENDITURE_TYPE_ID(+))
      AND  ATR.ATTRIBUTE_CATEGORY = TO_CHAR(ABT.ABSENCE_TYPE_ID(+))
      AND  PET.LANGUAGE(+) = 'US'
      AND  ABT.LANGUAGE(+) = 'US'
      AND  AUSG.USAGES_SOURCE_VERSION = (
           SELECT MAX(A1.USAGES_SOURCE_VERSION)
           FROM HWM_TM_REP_ATRB_USAGES A1
           WHERE A1.USAGES_SOURCE_ID = AUSG.USAGES_SOURCE_ID
      )
)
```

### 3.4 TL_PROJECT_FULL

**Purpose:** Combined project, task, and expenditure type

**When to Use:** For comprehensive project time reports

```sql
TL_PROJ_FULL AS (
    SELECT /*+ qb_name(TL_PROJ_F) MATERIALIZE */
           AUSG.USAGES_SOURCE_ID TM_REC_ID
          ,PJP.SEGMENT1 PROJECT_NUMBER
          ,PJP.NAME PROJECT_NAME
          ,PJT.TASK_NUMBER
          ,PJT.TASK_NAME
          ,NVL(PET.EXPENDITURE_TYPE_NAME, ABT.NAME) EXPENDITURE_TYPE
    FROM   HWM_TM_REP_ATRB_USAGES AUSG
          ,HWM_TM_REP_ATRBS ATR
          ,PJF_PROJECTS_ALL_VL PJP
          ,PJF_TASKS_V PJT
          ,PJF_EXP_TYPES_TL PET
          ,ANC_ABSENCE_TYPES_VL ABT
    WHERE  AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
      AND  ATR.ATTRIBUTE_CATEGORY = 'Projects'
      AND  ATR.ATTRIBUTE_NUMBER1 = PJP.PROJECT_ID(+)
      AND  ATR.ATTRIBUTE_NUMBER2 = PJT.TASK_ID(+)
      AND  ATR.ATTRIBUTE_CATEGORY = TO_CHAR(PET.EXPENDITURE_TYPE_ID(+))
      AND  ATR.ATTRIBUTE_CATEGORY = TO_CHAR(ABT.ABSENCE_TYPE_ID(+))
      AND  PET.LANGUAGE(+) = 'US'
      AND  ABT.LANGUAGE(+) = 'US'
      AND  AUSG.USAGES_SOURCE_VERSION = (
           SELECT MAX(A1.USAGES_SOURCE_VERSION)
           FROM HWM_TM_REP_ATRB_USAGES A1
           WHERE A1.USAGES_SOURCE_ID = AUSG.USAGES_SOURCE_ID
      )
)
```

---

## 4. Approver CTEs

### 4.1 TL_LINE_MANAGER

**Purpose:** Get line manager name for timesheet approval

**When to Use:** For approval tracking, manager reports

```sql
TL_LINE_MGR AS (
    SELECT /*+ qb_name(TL_LN_MGR) MATERIALIZE */
           PAPF.PERSON_ID
          ,MGNAME.DISPLAY_NAME LINE_MANAGER_NAME
    FROM   PER_ALL_PEOPLE_F PAPF
          ,PER_ALL_ASSIGNMENTS_M ASG
          ,PER_ASSIGNMENT_SUPERVISORS_F PASF
          ,PER_ALL_ASSIGNMENTS_M MGASG
          ,PER_PERSON_NAMES_F MGNAME
    WHERE  PAPF.PERSON_ID = ASG.PERSON_ID
      AND  ASG.ASSIGNMENT_ID = PASF.ASSIGNMENT_ID
      AND  PASF.MANAGER_ASSIGNMENT_ID = MGASG.ASSIGNMENT_ID
      AND  MGASG.PERSON_ID = MGNAME.PERSON_ID
      AND  PASF.MANAGER_TYPE = 'LINE_MANAGER'
      AND  ASG.PRIMARY_FLAG = 'Y'
      AND  ASG.ASSIGNMENT_TYPE = 'E'
      AND  ASG.EFFECTIVE_LATEST_CHANGE = 'Y'
      AND  MGNAME.NAME_TYPE = 'GLOBAL'
      AND  TRUNC(SYSDATE) BETWEEN ASG.EFFECTIVE_START_DATE AND ASG.EFFECTIVE_END_DATE
      AND  TRUNC(SYSDATE) BETWEEN PASF.EFFECTIVE_START_DATE AND PASF.EFFECTIVE_END_DATE
      AND  TRUNC(SYSDATE) BETWEEN MGNAME.EFFECTIVE_START_DATE AND MGNAME.EFFECTIVE_END_DATE
)
```

### 4.2 TL_PROJECT_MANAGER

**Purpose:** Get project manager from task manager or project role

**When to Use:** For project-based approvals

```sql
TL_PROJ_MGR AS (
    SELECT /*+ qb_name(TL_PROJ_MGR) MATERIALIZE */
           AUSG.USAGES_SOURCE_ID TM_REC_ID
          ,NVL(TMGR.DISPLAY_NAME, PMGR.DISPLAY_NAME) PROJECT_MANAGER_NAME
    FROM   HWM_TM_REP_ATRB_USAGES AUSG
          ,HWM_TM_REP_ATRBS ATR
          ,PJF_TASKS_V PJT
          ,PER_PERSON_NAMES_F TMGR
          ,PJF_PROJECT_PARTIES PPP
          ,PJF_PROJ_ROLE_TYPES_TL PPR
          ,PER_PERSON_NAMES_F PMGR
    WHERE  AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
      AND  ATR.ATTRIBUTE_CATEGORY = 'Projects'
      AND  ATR.ATTRIBUTE_NUMBER2 = PJT.TASK_ID(+)
      AND  PJT.TASK_MANAGER_PERSON_ID = TMGR.PERSON_ID(+)
      AND  ATR.ATTRIBUTE_NUMBER1 = PPP.PROJECT_ID(+)
      AND  PPP.PROJECT_ROLE_ID = PPR.PROJECT_ROLE_ID(+)
      AND  PPP.RESOURCE_SOURCE_ID = PMGR.PERSON_ID(+)
      AND  PPR.PROJECT_ROLE_NAME(+) = 'Project Manager'
      AND  PPR.LANGUAGE(+) = 'US'
      AND  TMGR.NAME_TYPE(+) = 'GLOBAL'
      AND  PMGR.NAME_TYPE(+) = 'GLOBAL'
      AND  TRUNC(SYSDATE) BETWEEN TMGR.EFFECTIVE_START_DATE(+) AND TMGR.EFFECTIVE_END_DATE(+)
      AND  TRUNC(SYSDATE) BETWEEN PMGR.EFFECTIVE_START_DATE(+) AND PMGR.EFFECTIVE_END_DATE(+)
      AND  AUSG.USAGES_SOURCE_VERSION = (
           SELECT MAX(A1.USAGES_SOURCE_VERSION)
           FROM HWM_TM_REP_ATRB_USAGES A1
           WHERE A1.USAGES_SOURCE_ID = AUSG.USAGES_SOURCE_ID
      )
)
```

---

## 5. Absence Integration CTEs

### 5.1 TL_ABSENCE_STATUS

**Purpose:** Get absence status from timesheet date and expenditure type

**When to Use:** For absence validation, leave balance reconciliation

```sql
TL_ABS_STATUS AS (
    SELECT /*+ qb_name(TL_ABS_STAT) MATERIALIZE */
           TMD.TM_REC_ID
          ,TMD.PERSON_ID
          ,TMD.START_TIME
          ,CASE
               WHEN ABSE.ABSENCE_STATUS_CD = 'SUBMITTED' AND ABSE.APPROVAL_STATUS_CD = 'APPROVED' 
                   THEN 'Approved'
               WHEN ABSE.ABSENCE_STATUS_CD = 'ORA_WITHDRAWN' AND ABSE.APPROVAL_STATUS_CD = 'ORA_AWAIT_AWAIT' 
                   THEN 'Approved'
               WHEN ABSE.ABSENCE_STATUS_CD = 'SUBMITTED' AND ABSE.APPROVAL_STATUS_CD = 'AWAITING' 
                   THEN 'Submitted'
               WHEN ABSE.ABSENCE_STATUS_CD = 'SAVED' 
                   THEN 'Saved'
           END ABSENCE_STATUS
    FROM   HWM_TM_REP_HIERARCHIES_VL TMD
          ,ANC_PER_ABS_ENTRIES ABSE
          ,ANC_ABSENCE_TYPES_F_TL ABST
    WHERE  ABSE.PERSON_ID = TMD.PERSON_ID
      AND  ABSE.ABSENCE_TYPE_ID = ABST.ABSENCE_TYPE_ID
      AND  ABST.LANGUAGE = 'US'
      AND  TRUNC(TMD.START_TIME) BETWEEN TRUNC(ABST.EFFECTIVE_START_DATE) AND TRUNC(ABST.EFFECTIVE_END_DATE)
      AND  TRUNC(TMD.START_TIME) BETWEEN TRUNC(ABSE.START_DATE) AND TRUNC(ABSE.END_DATE)
      AND  ((ABSE.ABSENCE_STATUS_CD = 'SUBMITTED' AND ABSE.APPROVAL_STATUS_CD = 'APPROVED') OR
            (ABSE.ABSENCE_STATUS_CD = 'ORA_WITHDRAWN' AND ABSE.APPROVAL_STATUS_CD = 'ORA_AWAIT_AWAIT') OR
            (ABSE.ABSENCE_STATUS_CD = 'SUBMITTED' AND ABSE.APPROVAL_STATUS_CD = 'AWAITING') OR
            (ABSE.ABSENCE_STATUS_CD = 'SAVED'))
)
```

---

## 6. Public Holiday CTE

### 6.1 TL_PUBLIC_HOLIDAY

**Purpose:** Identify public holidays from time records

**When to Use:** For statutory holiday tracking

```sql
TL_PUB_HOLIDAY AS (
    SELECT /*+ qb_name(TL_PH) MATERIALIZE */
           AUSG.USAGES_SOURCE_ID TM_REC_ID
          ,'Public Holiday' PUBLIC_HOLIDAY
    FROM   HWM_TM_REP_ATRB_USAGES AUSG
          ,HWM_TM_REP_ATRBS ATR
    WHERE  AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
      AND  UPPER(ATR.ATTRIBUTE_CATEGORY) LIKE 'STAT%HOLIDAY%'
      AND  AUSG.USAGES_SOURCE_VERSION = (
           SELECT MAX(A1.USAGES_SOURCE_VERSION)
           FROM HWM_TM_REP_ATRB_USAGES A1
           WHERE A1.USAGES_SOURCE_ID = AUSG.USAGES_SOURCE_ID
      )
)
```

---

## 7. Leave Reference CTE

### 7.1 TL_LEAVE_REFERENCE

**Purpose:** Get leave reference from attributes (ORA_ANC_*)

**When to Use:** For linking timesheets to absence records

```sql
TL_LEAVE_REF AS (
    SELECT /*+ qb_name(TL_LV_REF) MATERIALIZE */
           AUSG.USAGES_SOURCE_ID TM_REC_ID
          ,ATR.ATTRIBUTE_CHAR1 LEAVE_PROJECT_REFERENCE
    FROM   HWM_TM_REP_ATRB_USAGES AUSG
          ,HWM_TM_REP_ATRBS ATR
    WHERE  AUSG.TM_REP_ATRB_ID = ATR.TM_REP_ATRB_ID
      AND  ATR.ATTRIBUTE_CATEGORY LIKE 'ORA_ANC_%'
      AND  AUSG.USAGES_SOURCE_VERSION = (
           SELECT MAX(A1.USAGES_SOURCE_VERSION)
           FROM HWM_TM_REP_ATRB_USAGES A1
           WHERE A1.USAGES_SOURCE_ID = AUSG.USAGES_SOURCE_ID
      )
)
```

---

## 8. Combined Master CTE

### 8.1 TL_MASTER

**Purpose:** Complete timesheet master with all attributes

**When to Use:** For comprehensive timesheet reports (one-stop CTE)

```sql
TL_MASTER AS (
    SELECT /*+ qb_name(TL_MSTR) MATERIALIZE */
           TSF.TM_REC_GRP_ID
          ,TSF.TM_REC_ID
          ,TSF.TM_REC_GRP_VERSION
          ,TSF.PERSON_ID
          ,TSF.GRP_START_TIME
          ,TSF.GRP_STOP_TIME
          ,TSF.STATUS
          ,TSF.RECORDED_HOURS
          ,TSF.SUBMISSION_DATE
          ,TSF.APPROVAL_DATE
          ,TSF.GRP_TYPE_NAME
          ,TSF.TM_REC_TYPE
          ,TSF.START_TIME
          ,TSF.STOP_TIME
          ,TSF.MEASURE
          ,TSF.UNIT_OF_MEASURE
          ,PF.PROJECT_NUMBER
          ,PF.PROJECT_NAME
          ,PF.TASK_NUMBER
          ,PF.TASK_NAME
          ,PF.EXPENDITURE_TYPE
          ,LM.LINE_MANAGER_NAME
          ,PM.PROJECT_MANAGER_NAME
          ,NVL(PM.PROJECT_MANAGER_NAME, LM.LINE_MANAGER_NAME) APPROVER
          ,ABS.ABSENCE_STATUS
          ,PH.PUBLIC_HOLIDAY
          ,LR.LEAVE_PROJECT_REFERENCE
    FROM   TL_TS_FULL TSF
          ,TL_PROJ_FULL PF
          ,TL_LINE_MGR LM
          ,TL_PROJ_MGR PM
          ,TL_ABS_STATUS ABS
          ,TL_PUB_HOLIDAY PH
          ,TL_LEAVE_REF LR
    WHERE  TSF.TM_REC_ID = PF.TM_REC_ID(+)
      AND  TSF.PERSON_ID = LM.PERSON_ID(+)
      AND  TSF.TM_REC_ID = PM.TM_REC_ID(+)
      AND  TSF.TM_REC_ID = ABS.TM_REC_ID(+)
      AND  TSF.TM_REC_ID = PH.TM_REC_ID(+)
      AND  TSF.TM_REC_ID = LR.TM_REC_ID(+)
)
```

**Note:** This is a convenience CTE that joins all the above CTEs. Use when you need comprehensive timesheet data with all attributes.

---

**Last Updated:** 18-Dec-2025  
**Status:** Production-Ready  
**Source:** Employee Timesheet Report (980 lines analyzed)

