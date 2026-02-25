# Time & Labor Repositories

**Module:** Time & Labor (HWM + Custom ZMM)  
**Tag:** `#HCM #TL #HWM #Repositories #Production`  
**Status:** Production-Ready  
**Last Updated:** 29-Jan-2026

---

## Critical Rules

1. **NEVER** write fresh timesheet joins from scratch
2. **ALWAYS** copy these CTEs as-is
3. **ALWAYS** include `/*+ qb_name(NAME) MATERIALIZE */` hints OR index hints for HWM tables
4. **ALWAYS** use MAX version filtering for attributes
5. **ALWAYS** filter by STATUS for header CTEs
6. ⭐ **ALWAYS** use UNION for schedule assignments (LEGALEMP + ASSIGN types)
7. ⭐ **ALWAYS** apply 3-level version control for HWM_TM_REC (LATEST_VERSION + STATUS + CREATION_DATE)

---

## 🚨 PRODUCTION PATTERNS (From Missing Timecard & Overtime Reports)

### ⭐ Custom Schedule Tables (ZMM Prefix - CRITICAL)

**These are custom tables. MUST be used for all schedule/shift-based reports:**

| Table | Purpose |
|-------|---------|
| `ZMM_SR_SCHEDULES_VL` | Schedule master |
| `ZMM_SR_SCHEDULE_DTLS` | Shift date/time details |
| `ZMM_SR_SHIFTS_VL` | Shift definitions |
| ⭐ `ZMM_SR_AVAILABLE_DATES` | Working dates calendar |
| `PER_SCHEDULE_ASSIGNMENTS` | Employee-schedule link |

---

## 0. PRODUCTION CTEs (From Missing Timecard & Overtime Reports)

### 0.1 TL_PERSON_SCHEDULE (CRITICAL - Dual Resource Type)

**Purpose:** Get employees with schedule assignments (BOTH Legal Entity and Assignment levels)

**When to Use:** ALL Time & Labor reports requiring schedule information

**⚠️ CRITICAL:** Must use UNION to capture both schedule assignment types

```sql
TL_PERSON_SCHEDULE AS (
    -- Part 1: Legal Entity Level Schedules
    SELECT /*+ qb_name(TL_PSA_LE) MATERIALIZE */
        PAPF.PERSON_ID,
        PPNF.DISPLAY_NAME,
        PAAF.ASSIGNMENT_ID,
        PSA.SCHEDULE_ID,
        ZSSV.SCHEDULE_NAME,
        CASE WHEN TRUNC(PAPF.START_DATE) > TRUNC(PSA.START_DATE) 
             THEN TRUNC(PAPF.START_DATE) 
             ELSE TRUNC(PSA.START_DATE) 
        END START_DATE_LE,
        TRUNC(PSA.END_DATE) END_DATE_LE,
        NULL START_DATE,
        NULL END_DATE,
        'LEGALEMP' RESOURCE_TYPE,
        PAPF.PERSON_NUMBER,
        PAAF.LOCATION_ID,
        PD.NAME DEPT_NAME,
        HAP.NAME POSITION_CODE,
        PG.NAME GRADE_NAME
    FROM
        PER_ALL_PEOPLE_F          PAPF,
        PER_PERSON_NAMES_F       PPNF,
        PER_ALL_ASSIGNMENTS_F    PAAF,
        PER_SCHEDULE_ASSIGNMENTS PSA,
        ZMM_SR_SCHEDULES_VL      ZSSV,
        PER_DEPARTMENTS          PD,
        HR_ALL_POSITIONS         HAP,
        PER_GRADES               PG
    WHERE PAPF.PERSON_ID       = PPNF.PERSON_ID
      AND PAPF.PERSON_ID       = PAAF.PERSON_ID
      AND PAAF.LEGAL_ENTITY_ID = PSA.RESOURCE_ID      -- Legal Entity link
      AND PSA.SCHEDULE_ID      = ZSSV.SCHEDULE_ID
      AND PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
      AND PAAF.POSITION_ID     = HAP.POSITION_ID(+)
      AND PAAF.GRADE_ID        = PG.GRADE_ID(+)
      AND PPNF.NAME_TYPE       = 'GLOBAL'
      AND PSA.PRIMARY_FLAG     = 'Y'
      AND PAAF.ASSIGNMENT_STATUS_TYPE  = 'ACTIVE'
      AND PAAF.ASSIGNMENT_TYPE         = 'E'
      AND PAAF.PRIMARY_ASSIGNMENT_FLAG = 'Y'
      AND PSA.RESOURCE_TYPE            = 'LEGALEMP'
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE) AND TRUNC(PPNF.EFFECTIVE_END_DATE)
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PG.EFFECTIVE_START_DATE(+)) AND TRUNC(PG.EFFECTIVE_END_DATE(+))
      AND TRUNC(SYSDATE) BETWEEN TRUNC(HAP.EFFECTIVE_START_DATE(+)) AND TRUNC(HAP.EFFECTIVE_END_DATE(+))
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PD.EFFECTIVE_START_DATE(+)) AND TRUNC(PD.EFFECTIVE_END_DATE(+))
      -- Exclude if assignment-level schedule exists (assignment overrides LE)
      AND NOT EXISTS (
          SELECT 1
          FROM PER_SCHEDULE_ASSIGNMENTS PSA2,
               ZMM_SR_AVAILABLE_DATES Z
          WHERE PAAF.ASSIGNMENT_ID = PSA2.RESOURCE_ID
            AND PSA2.PRIMARY_FLAG = 'Y'
            AND PSA2.RESOURCE_TYPE = 'ASSIGN'
            AND PSA2.SCHEDULE_ID = Z.SCHEDULE_ID
            AND TRUNC(Z.CALENDAR_DATE) BETWEEN TRUNC(SYSDATE) AND TRUNC(SYSDATE)
            AND TRUNC(Z.CALENDAR_DATE) BETWEEN TRUNC(PSA2.START_DATE) AND TRUNC(PSA2.END_DATE)
      )

    UNION ALL

    -- Part 2: Assignment Level Schedules (overrides Legal Entity)
    SELECT /*+ qb_name(TL_PSA_ASG) MATERIALIZE */
        PAPF.PERSON_ID,
        PPNF.DISPLAY_NAME,
        PAAF.ASSIGNMENT_ID,
        PSA.SCHEDULE_ID,
        ZSSV.SCHEDULE_NAME,
        NULL START_DATE_LE,
        NULL END_DATE_LE,
        CASE WHEN TRUNC(PAPF.START_DATE) > TRUNC(PSA.START_DATE) 
             THEN TRUNC(PAPF.START_DATE) 
             ELSE TRUNC(PSA.START_DATE) 
        END START_DATE,
        TRUNC(PSA.END_DATE) END_DATE,
        'ASSIGN' RESOURCE_TYPE,
        PAPF.PERSON_NUMBER,
        PAAF.LOCATION_ID,
        PD.NAME DEPT_NAME,
        HAP.NAME POSITION_CODE,
        PG.NAME GRADE_NAME
    FROM
        PER_ALL_PEOPLE_F          PAPF,
        PER_PERSON_NAMES_F       PPNF,
        PER_ALL_ASSIGNMENTS_F    PAAF,
        PER_SCHEDULE_ASSIGNMENTS PSA,
        ZMM_SR_SCHEDULES_VL      ZSSV,
        PER_DEPARTMENTS          PD,
        HR_ALL_POSITIONS         HAP,
        PER_GRADES               PG
    WHERE PAPF.PERSON_ID      = PPNF.PERSON_ID
      AND PAPF.PERSON_ID      = PAAF.PERSON_ID
      AND PAAF.ASSIGNMENT_ID  = PSA.RESOURCE_ID       -- Assignment link
      AND PSA.SCHEDULE_ID     = ZSSV.SCHEDULE_ID
      AND PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
      AND PAAF.POSITION_ID     = HAP.POSITION_ID(+)
      AND PAAF.GRADE_ID        = PG.GRADE_ID(+)
      AND PPNF.NAME_TYPE      = 'GLOBAL'
      AND PSA.PRIMARY_FLAG    = 'Y'
      AND PAAF.ASSIGNMENT_STATUS_TYPE  = 'ACTIVE'
      AND PAAF.ASSIGNMENT_TYPE         = 'E'
      AND PAAF.PRIMARY_ASSIGNMENT_FLAG = 'Y'
      AND PSA.RESOURCE_TYPE            = 'ASSIGN'
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE) AND TRUNC(PPNF.EFFECTIVE_END_DATE)
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PG.EFFECTIVE_START_DATE(+)) AND TRUNC(PG.EFFECTIVE_END_DATE(+))
      AND TRUNC(SYSDATE) BETWEEN TRUNC(HAP.EFFECTIVE_START_DATE(+)) AND TRUNC(HAP.EFFECTIVE_END_DATE(+))
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PD.EFFECTIVE_START_DATE(+)) AND TRUNC(PD.EFFECTIVE_END_DATE(+))
)
```

**Why UNION?** Assignment-level schedule overrides Legal Entity-level schedule. Must capture both!

### 0.2 TL_WORKING_DATES (Working Calendar)

**Purpose:** Get working dates from ZMM_SR_AVAILABLE_DATES calendar

**When to Use:** Missing timecard reports, attendance tracking

```sql
TL_WORKING_DATES AS (
    SELECT /*+ qb_name(TL_WORK_DATES) MATERIALIZE */
        SCHEDULE_ID,
        TRUNC(CALENDAR_DATE) CALENDAR_DATE
    FROM ZMM_SR_AVAILABLE_DATES
    WHERE TRUNC(CALENDAR_DATE) BETWEEN :P_FROM_DATE AND :P_TO_DATE
)
```

### 0.3 TL_SHIFT_DETAILS (Shift Schedule Details)

**Purpose:** Get shift date/time details with duration calculations

**When to Use:** Shift-based reports, overtime calculations

```sql
TL_SHIFT_DETAILS AS (
    SELECT /*+ qb_name(TL_SHIFT_DTL) MATERIALIZE */
        ZSST.SCHEDULE_ID,
        ZSST.SCHEDULE_NAME,
        TRUNC(ZSSTD.START_DATE_TIME) CALENDAR_DATE,
        ZSSTD.START_DATE_TIME,
        ZSSTD.END_DATE_TIME,
        ZSSV.SHIFT_NAME,
        ZSSV.DURATION_MS_NUM,
        (ZSSV.DURATION_MS_NUM/3600000) SHIFT_HRS,     -- Duration in hours
        -- Day of week (1=Monday, 7=Sunday)
        DECODE(TRIM(TO_CHAR(ZSSTD.START_DATE_TIME, 'DAY', 'NLS_DATE_LANGUAGE = AMERICAN')), 
               'SUNDAY',7, 'MONDAY',1, 'TUESDAY',2, 'WEDNESDAY',3, 
               'THURSDAY',4, 'FRIDAY',5, 'SATURDAY',6) SHIFT_DAY_NUM,
        -- Shift start time components
        LPAD(TRUNC(ROUND(ZSSV.START_TIME_MS_NUM/1000)/3600), 2, '0') SHIFT_HR,
        LPAD(TRUNC(MOD(ROUND(ZSSV.START_TIME_MS_NUM/1000), 3600)/60), 2, '0') SHIFT_MIN
    FROM
        ZMM_SR_SCHEDULES_TL      ZSST,
        ZMM_SR_SCHEDULE_DTLS     ZSSTD,
        ZMM_SR_SHIFTS_VL         ZSSV
    WHERE ZSST.SCHEDULE_ID    = ZSSTD.SCHEDULE_ID
      AND ZSST.LANGUAGE       = 'US'
      AND ZSSTD.SHIFT_ID      = ZSSV.SHIFT_ID  
      AND ZSSV.DELETED_FLAG   = 'N'
      AND TRUNC(ZSSTD.START_DATE_TIME) BETWEEN :P_FROM_DATE AND :P_TO_DATE
    ORDER BY ZSSTD.START_DATE_TIME
)
```

### 0.4 TL_TIME_RECORDS (Production HWM Pattern with 3-Level Version Control)

**Purpose:** Get latest time records with proper version filtering

**When to Use:** ALL reports using HWM_TM_REC table

**⚠️ CRITICAL:** This CTE includes 3-level version control (production-proven)

```sql
TL_TIME_RECORDS AS (
    SELECT /*+ qb_name(TL_TM_REC) MATERIALIZE */
        /*+ index(HTR, HWM_TM_REC_U1) */
        /*+ index(HTRGU, HWM_TM_REC_GRP_USAGES_U1) */
        /*+ index(DTRG, HWM_TM_REC_GRP_U1) */
        /*+ index(DTRG1, HWM_TM_REC_GRP_U1) */
        HTR.RESOURCE_ID,                          -- PERSON_ID
        HTR.SUBRESOURCE_ID,                       -- ASSIGNMENT_ID
        TRUNC(HTR.START_TIME) DAY_START_TIME,
        HTR.START_TIME,
        HTR.STOP_TIME,
        HTR.MEASURE,                              -- Hours worked
        HTDTUSV.STATUS_VALUE,                     -- Status
        TRUNC(DTRG1.START_TIME) START_WEEK,
        TRUNC(DTRG1.STOP_TIME) END_WEEK
    FROM 
        HWM_TM_REC HTR,
        HWM_TM_REC_GRP_USAGES HTRGU,
        HWM_TM_REC_GRP DTRG,
        HWM_TM_REC_GRP DTRG1,
        HWM_TM_D_TM_UI_STATUS_V HTDTUSV
    WHERE HTRGU.TM_REC_GRP_ID      = DTRG.TM_REC_GRP_ID
      AND HTRGU.TM_REC_GRP_VERSION = DTRG.TM_REC_GRP_VERSION
      AND DTRG.PARENT_TM_REC_GRP_ID = DTRG1.TM_REC_GRP_ID
      AND DTRG.PARENT_TM_REC_GRP_VERSION = DTRG1.TM_REC_GRP_VERSION
      AND HTR.TM_REC_ID          = HTRGU.TM_REC_ID
      AND HTR.TM_REC_VERSION     = HTRGU.TM_REC_VERSION
      AND DTRG1.TM_REC_GRP_ID    = HTDTUSV.TM_BLDG_BLK_ID
      -- Critical filters (ALWAYS include these)
      AND UPPER(HTR.UNIT_OF_MEASURE) IN ('UN', 'HR')
      AND DTRG1.GRP_TYPE_ID      = 100              -- Daily group
      AND HTR.RESOURCE_TYPE      = 'PERSON'
      AND HTR.DELETE_FLAG        IS NULL
      AND HTR.LAYER_CODE         = 'TIME_RPTD'      -- Reported time layer
      AND TRUNC(HTR.STOP_TIME)   IS NOT NULL
      -- Exclude midnight system entries
      AND (EXTRACT(HOUR FROM HTR.START_TIME) <> 00
        OR EXTRACT(MINUTE FROM HTR.START_TIME) <> 00 
        OR EXTRACT(SECOND FROM HTR.START_TIME) <> 00)
      -- VERSION CONTROL LEVEL 1: Latest version flag
      AND NVL(HTR.LATEST_VERSION,'Y') = 'Y'
      AND HTR.LATEST_VERSION     = 'Y'
      -- VERSION CONTROL LEVEL 2: Latest status version
      AND (HTDTUSV.TM_BLDG_BLK_VERSION, HTDTUSV.STATUS_ID) = (
          SELECT MAX(TM_BLDG_BLK_VERSION), MAX(STATUS_ID) 
          FROM HWM_TM_D_TM_UI_STATUS_V 
          WHERE HTDTUSV.TM_BLDG_BLK_ID = TM_BLDG_BLK_ID
      )
      -- VERSION CONTROL LEVEL 3: Latest creation date (handles edits)
      AND HTR.CREATION_DATE = (
          SELECT MAX(HTR1.CREATION_DATE)
          FROM HWM_TM_REC HTR1,
               HWM_TM_REC_GRP_USAGES HTRGU1,
               HWM_TM_REC_GRP DTRG1,
               HWM_TM_REC_GRP DTRG2
          WHERE HTRGU1.TM_REC_GRP_ID    = DTRG1.TM_REC_GRP_ID
            AND HTRGU1.TM_REC_GRP_VERSION = DTRG1.TM_REC_GRP_VERSION
            AND DTRG1.PARENT_TM_REC_GRP_ID= DTRG2.TM_REC_GRP_ID
            AND DTRG1.PARENT_TM_REC_GRP_VERSION = DTRG2.TM_REC_GRP_VERSION
            AND HTR1.TM_REC_ID      = HTRGU1.TM_REC_ID
            AND HTR1.TM_REC_VERSION = HTRGU1.TM_REC_VERSION
            AND UPPER(HTR1.UNIT_OF_MEASURE) IN ('UN', 'HR')
            AND DTRG2.GRP_TYPE_ID = 100
            AND NVL(HTR1.LATEST_VERSION,'Y') = 'Y'
            AND HTR1.LATEST_VERSION = 'Y'
            AND HTR1.RESOURCE_TYPE  = 'PERSON'
            AND HTR1.DELETE_FLAG    IS NULL
            AND HTR1.LAYER_CODE     = 'TIME_RPTD'
            AND DTRG1.TM_REC_GRP_ID = DTRG.TM_REC_GRP_ID
            AND HTR1.RESOURCE_ID    = HTR.RESOURCE_ID
      )
      -- Date range filter
      AND TRUNC(HTR.START_TIME) BETWEEN :P_FROM_DATE AND :P_TO_DATE
)
```

**Why 3 Levels?**
- Level 1: `LATEST_VERSION = 'Y'` - Marks current version
- Level 2: `MAX(STATUS_ID)` - Latest status update
- Level 3: `MAX(CREATION_DATE)` - Handles timecard edits/resubmissions

### 0.5 TL_PUBLIC_HOLIDAYS_BY_LOCATION (Production Pattern)

**Purpose:** Get public holidays by employee location

**When to Use:** Missing timecard reports, overtime reports (holiday OT)

```sql
TL_PUBLIC_HOLIDAYS AS (
    SELECT /*+ qb_name(TL_PUB_HOL) MATERIALIZE */
        DISTINCT 
        PAPF.PERSON_ID,
        TRUNC(PCE.START_DATE_TIME) START_DATE,
        TRUNC(PCE.END_DATE_TIME) END_DATE
    FROM
        PER_ALL_PEOPLE_F      PAPF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_CALENDAR_EVENTS   PCE,
        PER_GEO_TREE_NODE_RF  PNR,
        HR_LOCATIONS          HR
    WHERE PCE.TREE_CODE           = PNR.TREE_CODE
      AND PCE.TREE_STRUCTURE_CODE = PNR.TREE_STRUCTURE_CODE
      AND PAPF.PERSON_ID          = PAAF.PERSON_ID
      AND PAAF.ASSIGNMENT_STATUS_TYPE  = 'ACTIVE'
      AND PAAF.ASSIGNMENT_TYPE         = 'E'
      AND PAAF.PRIMARY_ASSIGNMENT_FLAG = 'Y'
      AND PAAF.LOCATION_ID        = HR.LOCATION_ID
      AND HR.COUNTRY              = PNR.PK1_VALUE     -- Link to country
      AND PCE.CATEGORY            = 'PH'              -- Public Holiday
      AND TO_CHAR(PCE.START_DATE_TIME,'YYYY') = TO_CHAR(:P_FROM_DATE,'YYYY')
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
      AND TRUNC(SYSDATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
)
```

### 0.6 TL_ABSENCE_DATES (For Exclusion from Missing Checks)

**Purpose:** Get absence dates to exclude from missing timecard checks

**When to Use:** Missing timecard reports

```sql
TL_ABSENCE_DATES AS (
    SELECT /*+ qb_name(TL_ABS_DATES) MATERIALIZE */
        APAE.PERSON_ID,
        AATV.NAME ABSENCE_TYPE,
        TRUNC(APAE.START_DATE) START_DATE,
        TRUNC(APAE.END_DATE) END_DATE,
        APAE.DURATION
    FROM
        ANC_PER_ABS_ENTRIES APAE,
        ANC_ABSENCE_TYPES_VL AATV
    WHERE APAE.ABSENCE_TYPE_ID   = AATV.ABSENCE_TYPE_ID
      AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
      AND APAE.APPROVAL_STATUS_CD <> 'DENIED'
      -- Date overlap with parameter range
      AND (APAE.START_DATE BETWEEN TRUNC(:P_FROM_DATE) AND TRUNC(:P_TO_DATE)
        OR APAE.END_DATE BETWEEN TRUNC(:P_FROM_DATE) AND TRUNC(:P_TO_DATE)
        OR TRUNC(APAE.END_DATE) > TRUNC(:P_TO_DATE))
      AND TRUNC(SYSDATE) BETWEEN TRUNC(AATV.EFFECTIVE_START_DATE) AND TRUNC(AATV.EFFECTIVE_END_DATE)
)
```

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

**Last Updated:** 29-Jan-2026  
**Status:** Production-Ready  
**Source:** Employee Timesheet Report (980 lines) + Missing Timecard/Overtime Reports (3,500+ lines analyzed) + Cross-Module Updates (02-Jan-2026)

---

## 9. Enhanced Date Filtering CTEs (02-Jan-2026 Update)

### 9.1 TL_PARAMETERS (Parameter Management)

**Purpose:** Centralized parameter handling with effective date support

**When to Use:** For historical timecard queries, "as of" date reports

```sql
TL_PARAMETERS AS (
    SELECT /*+ qb_name(TL_PARAMS) MATERIALIZE */
           TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
           TO_DATE(:P_START_DATE, 'DD-MON-YYYY') AS START_DATE,
           TO_DATE(:P_END_DATE, 'DD-MON-YYYY') AS END_DATE,
           UPPER(NVL(:P_TIME_ENTRY_TYPE, 'ALL')) AS TIME_ENTRY_TYPE,
           UPPER(NVL(:P_PROJECT_NAME, 'ALL')) AS PROJECT_NAME,
           UPPER(NVL(:P_STATUS, 'ALL')) AS STATUS
    FROM DUAL
)
```

**Benefits:**
- Single source of parameters
- Case-insensitive filtering
- 'ALL' bypass for flexible filtering
- Effective date for historical queries

### 9.2 TL_TIMECARD_HISTORY (Historical Timecard CTE)

**Purpose:** Query timecards for any historical period with effective date filtering

**When to Use:** Historical hours analysis, audit compliance, past timecard reports

```sql
TL_TIMECARD_HISTORY AS (
    SELECT /*+ qb_name(TL_TC_HIST) MATERIALIZE */
           TMH.TM_REC_GRP_ID,
           TMH.PERSON_ID,
           TMD.TM_REC_ID,
           TO_CHAR(TMD.START_TIME, 'YYYY-MM-DD') START_TIME,
           TO_CHAR(TMD.STOP_TIME, 'DD-MON-YYYY') STOP_TIME,
           TMD.MEASURE,
           TMD.TM_REC_TYPE,
           TMH.STATUS,
           P.EFFECTIVE_DATE
    FROM   HWM_TM_REC_SUMM TMH,
           HWM_TM_REP_HIERARCHIES_VL TMD,
           TL_PARAMETERS P
    WHERE  TMH.TM_REC_GRP_ID = TMD.TM_REC_GRP_ID
      AND  TMH.STATUS IN ('Approved', 'Submitted')
      -- Timecard entries within date range
      AND  TMD.START_TIME >= P.START_DATE
      AND  TMD.STOP_TIME <= P.END_DATE
      -- Use Effective Date for date-tracked lookups
      AND  TMH.START_TIME >= P.START_DATE
      AND  TMH.STOP_TIME <= P.END_DATE
)
```

**Key Features:**
- Parameter-based date filtering
- Effective date support for "as of" queries
- Historical audit trail
- Flexible status filtering

### 9.3 TL_EMP_SERVICE (Service Calculation)

**Purpose:** Calculate employee service years for time-off eligibility

**When to Use:** Time-off accrual calculation, eligibility determination, service-based benefits

```sql
TL_EMP_SERVICE AS (
    SELECT /*+ qb_name(TL_EMP_SVC) MATERIALIZE */
           PAPF.PERSON_ID,
           PAPF.PERSON_NUMBER,
           PPOS.DATE_START HIRE_DATE,
           PPOS.ORIGINAL_DATE_OF_HIRE,
           ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
                 NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) AS SERVICE_IN_YEARS,
           
           -- Service-based entitlement
           CASE 
               WHEN ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
                          NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) < 1
               THEN 10  -- 10 days for first year
               
               WHEN ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
                          NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) BETWEEN 1 AND 5
               THEN 15  -- 15 days for 1-5 years
               
               WHEN ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
                          NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) BETWEEN 5 AND 10
               THEN 20  -- 20 days for 5-10 years
               
               ELSE 25  -- 25 days for 10+ years
           END AS ANNUAL_LEAVE_ENTITLEMENT,
           
           -- Monthly accrual rate
           ROUND(
               CASE 
                   WHEN ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
                              NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) < 1
                   THEN 10
                   WHEN ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
                              NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) BETWEEN 1 AND 5
                   THEN 15
                   WHEN ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
                              NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) BETWEEN 5 AND 10
                   THEN 20
                   ELSE 25
               END / 12, 2
           ) AS MONTHLY_ACCRUAL
           
    FROM   PER_ALL_PEOPLE_F PAPF,
           PER_PERIODS_OF_SERVICE PPOS,
           TL_PARAMETERS P
    WHERE  PAPF.PERSON_ID = PPOS.PERSON_ID
      AND  P.EFFECTIVE_DATE BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
      AND  P.EFFECTIVE_DATE BETWEEN PPOS.DATE_START 
           AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
)
```

**Business Rules:**
- < 1 year: 10 days
- 1-5 years: 15 days
- 5-10 years: 20 days
- 10+ years: 25 days

### 9.4 TL_FLEXIBLE_FILTER (Case-Insensitive Filtering)

**Purpose:** Apply flexible, case-insensitive filters with 'ALL' bypass

**When to Use:** All time & labor queries with user-entered filters

**Pattern:**
```sql
-- In main query WHERE clause
WHERE
    (UPPER(TMD.TM_REC_TYPE) = P.TIME_ENTRY_TYPE OR P.TIME_ENTRY_TYPE = 'ALL')
    AND (UPPER(PJP.NAME) = P.PROJECT_NAME OR P.PROJECT_NAME = 'ALL')
    AND (UPPER(TMH.STATUS) = P.STATUS OR P.STATUS = 'ALL')
```

**Benefits:**
- Users can enter "regular" or "Regular" or "REGULAR"
- 'ALL' shows all records
- Consistent with other HCM modules
- Reduces filter errors

---

## 10. Integration Examples

### Example 1: Historical Timecard Report with Service Calculation

```sql
WITH
TL_PARAMETERS AS (/* parameter CTE */),
TL_TIMECARD_HISTORY AS (/* history CTE */),
TL_EMP_SERVICE AS (/* service CTE */)

SELECT
    SVC.PERSON_NUMBER,
    PPNF.DISPLAY_NAME,
    SVC.HIRE_DATE,
    SVC.SERVICE_IN_YEARS,
    SVC.ANNUAL_LEAVE_ENTITLEMENT,
    TCH.START_TIME,
    TCH.STOP_TIME,
    TCH.MEASURE,
    TCH.STATUS
FROM
    TL_TIMECARD_HISTORY TCH,
    TL_EMP_SERVICE SVC,
    PER_PERSON_NAMES_F PPNF
WHERE
    TCH.PERSON_ID = SVC.PERSON_ID
    AND TCH.PERSON_ID = PPNF.PERSON_ID
    AND PPNF.NAME_TYPE = 'GLOBAL'
ORDER BY SVC.PERSON_NUMBER, TCH.START_TIME;
```

### Example 2: Time-Off Eligibility with Flexible Filtering

```sql
WITH
TL_PARAMETERS AS (/* parameter CTE */),
TL_EMP_SERVICE AS (/* service CTE */)

SELECT
    SVC.PERSON_NUMBER,
    PPNF.DISPLAY_NAME,
    HDORG.NAME DEPARTMENT,
    SVC.SERVICE_IN_YEARS,
    SVC.ANNUAL_LEAVE_ENTITLEMENT,
    SVC.MONTHLY_ACCRUAL
FROM
    TL_EMP_SERVICE SVC,
    PER_PERSON_NAMES_F PPNF,
    PER_ALL_ASSIGNMENTS_M PAAM,
    HR_ORGANIZATION_UNITS_F_TL HDORG,
    TL_PARAMETERS P
WHERE
    SVC.PERSON_ID = PPNF.PERSON_ID
    AND SVC.PERSON_ID = PAAM.PERSON_ID
    AND PAAM.ORGANIZATION_ID = HDORG.ORGANIZATION_ID
    AND PPNF.NAME_TYPE = 'GLOBAL'
    AND PAAM.PRIMARY_FLAG = 'Y'
    AND PAAM.EFFECTIVE_LATEST_CHANGE = 'Y'
    AND HDORG.LANGUAGE = 'US'
    AND TRUNC(SYSDATE) BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
    AND TRUNC(SYSDATE) BETWEEN HDORG.EFFECTIVE_START_DATE AND HDORG.EFFECTIVE_END_DATE
ORDER BY SVC.SERVICE_IN_YEARS DESC;
```

---

**Note:** These enhanced CTEs complement existing Time & Labor patterns and do not replace critical timecard constraints (version handling, status filtering, date-track filtering).

