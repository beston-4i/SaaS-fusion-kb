# HCM Absence Repositories: Pre-Validated CTEs

**Module:** HCM Absence Management  
**Purpose:** Ready-to-use CTE components for absence/leave queries  
**Tag:** `#HCM #ABSENCE #REPOSITORIES #CTEs`  
**Date:** 18-12-25

---

## 📋 Repository Index

| CTE Name | Purpose | Source Tables | Use Case |
|----------|---------|---------------|----------|
| **PERIOD** | Date range & employee filter | Parameters | All reports |
| **EMP_MASTER** | Employee base details | PER_ALL_PEOPLE_F, PER_PERSON_NAMES_F | All employee queries |
| **EMP_ASSIGNMENT** | Assignment & org details | PER_ALL_ASSIGNMENTS_F, departments | All queries |
| **DEPARTMENTS** | Organization/Department hierarchy | HR_ALL_ORGANIZATION_UNITS_F | Org structure queries |
| **LEAVE_TRANSACTIONS** | Absence entries (approved) | ANC_PER_ABS_ENTRIES | Leave history reports |
| **ABSENCE_TYPES** | Leave type master | ANC_ABSENCE_TYPES_F_TL | Type classification |
| **ANNUAL_LEAVE_BALANCE** | Annual leave balance | ANC_PER_ACCRUAL_ENTRIES | Balance reports |
| **LEAVE_BALANCES** | All leave plan balances | ANC_PER_ACCRUAL_ENTRIES | Multi-plan balance |
| **SUPERVISOR** | Manager hierarchy | PER_ASSIGNMENT_SUPERVISORS_F | Manager approval queries |
| **WORKFLOW_APPROVAL** | Approval workflow history | FA_FUSION_SOAINFRA.WFTASK | Approval tracking |
| **ABSENCE_REASONS** | Absence reason lookup | ANC_ABSENCE_REASONS_F_TL | Reason classification |
| **LEAVE_HISTORY** | Historical leave summary | ANC_PER_ABS_ENTRIES | Historical comparison |
| **WORKFLOW_APPROVAL_DETAIL** | Detailed workflow with names | FA_FUSION_SOAINFRA.WFTASK | Enhanced approval tracking |
| **SALARY_ADVANCE_RECOVERY** | Salary advance from payroll | PAY_ELEMENT_ENTRIES_F | Payroll integration |
| **LEAVE_DETAILS_ENHANCED** | Enhanced leave with workflow | ANC_PER_ABS_ENTRIES, WFTASK | Comprehensive leave details |
| **ANNUAL_LEAVE_BALANCE_ADVANCED** | Case-insensitive balance | ANC_PER_ACCRUAL_ENTRIES | Balance with UPPER matching |
| **CURRENT_ABSENCE_ENTRIES** | Current/ongoing absences | ANC_PER_ABS_ENTRIES | Current absence activity |
| **TERMINATED_EMP_FILTER** | Employee with termination handling | PER_ALL_ASSIGNMENTS_F, PPOS | Terminated employee queries |
| **ACCRUAL_BALANCE_CURRENT_YEAR** | Current year balance only | ANC_PER_ACCRUAL_ENTRIES | Year-specific balance |

---

## 1. PERIOD CTE
**Purpose:** Parameter-driven date range and employee filter  
**Usage:** Copy at the start of every query

```sql
WITH PERIOD AS (
    /*+ qb_name(PERIOD) */
    SELECT
        TRUNC(CAST(NVL(:P_START_DATE, ADD_MONTHS(TRUNC(SYSDATE), -12)) AS DATE)) AS START_DATE,
        TRUNC(CAST(NVL(:P_END_DATE, LAST_DAY(SYSDATE)) AS DATE)) AS END_DATE,
        :P_EMP_NO AS EMPLOYEE_NUMBER,
        :P_EMP_NAME AS EMPLOYEE_NAME
    FROM DUAL
)
```

**Parameters:**
- `:P_START_DATE` - Start date (defaults to 12 months ago)
- `:P_END_DATE` - End date (defaults to end of current month)
- `:P_EMP_NO` - Employee number (optional, use 'ALL' for all employees)
- `:P_EMP_NAME` - Employee name (optional, use 'ALL' for all employees)

---

## 2. DEPARTMENTS CTE
**Purpose:** Organization/Department hierarchy with proper classification  
**Usage:** Standard department lookup for all queries

```sql
,DEPARTMENTS AS (
    /*+ qb_name(DEPARTMENTS) */
    SELECT
        HAUFT.ORGANIZATION_ID,
        HAUFT.NAME DEPARTMENT
    FROM
        HR_ORG_UNIT_CLASSIFICATIONS_F HOUCF,
        HR_ALL_ORGANIZATION_UNITS_F HAOUF,
        HR_ORGANIZATION_UNITS_F_TL HAUFT
    WHERE
        HAOUF.ORGANIZATION_ID = HOUCF.ORGANIZATION_ID
    AND HAOUF.ORGANIZATION_ID = HAUFT.ORGANIZATION_ID
    AND HAOUF.EFFECTIVE_START_DATE BETWEEN HOUCF.EFFECTIVE_START_DATE AND HOUCF.EFFECTIVE_END_DATE
    AND HAUFT.LANGUAGE = 'US'
    AND HAUFT.EFFECTIVE_START_DATE = HAOUF.EFFECTIVE_START_DATE
    AND HAUFT.EFFECTIVE_END_DATE = HAOUF.EFFECTIVE_END_DATE
    AND HOUCF.CLASSIFICATION_CODE = 'DEPARTMENT'
    AND SYSDATE BETWEEN HAUFT.EFFECTIVE_START_DATE AND HAUFT.EFFECTIVE_END_DATE
)
```

**Key Filters:**
- `CLASSIFICATION_CODE = 'DEPARTMENT'` - Only department-level orgs
- `LANGUAGE = 'US'` - English translation
- Date-tracked on SYSDATE

---

## 3. EMP_MASTER CTE
**Purpose:** Employee base details (person and names)  
**Usage:** Foundation for all employee queries

```sql
,EMP_MASTER AS (
    /*+ qb_name(EMP_MASTER) */
    SELECT
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER,
        PPNF.DISPLAY_NAME,
        PPNF.FULL_NAME,
        PPTTL.USER_PERSON_TYPE EMPLOYEE_TYPE
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_PERSON_TYPES_TL PPTTL
    WHERE
        PAPF.PERSON_ID = PPNF.PERSON_ID
    AND PAPF.PERSON_ID = PAAF.PERSON_ID
    AND PAAF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID(+)
    AND PPNF.NAME_TYPE = 'GLOBAL'
    AND PPTTL.LANGUAGE(+) = 'US'
    AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PAAF.PRIMARY_FLAG = 'Y'
    AND PAAF.ASSIGNMENT_TYPE = 'E'
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE) AND TRUNC(PPNF.EFFECTIVE_END_DATE)
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
)
```

**Key Filters:**
- `NAME_TYPE = 'GLOBAL'` - Standard display name
- Date-tracked on SYSDATE
- Outer join to person type (optional)

---

## 4. EMP_ASSIGNMENT CTE
**Purpose:** Assignment details with job and department  
**Usage:** For organizational context in reports

```sql
,EMP_ASSIGNMENT AS (
    /*+ qb_name(EMP_ASSIGNMENT) */
    SELECT
        PAAF.PERSON_ID,
        PAAF.ASSIGNMENT_ID,
        PAAF.ASSIGNMENT_NUMBER,
        PAAF.ORGANIZATION_ID,
        PAAF.BUSINESS_UNIT_ID,
        PAAF.PERIOD_OF_SERVICE_ID,
        PJFV.NAME AS JOB_NAME,
        PD.NAME AS DEPARTMENT_NAME,
        HAOU.NAME AS ORG_NAME
    FROM
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_JOBS_F_VL PJFV,
        PER_DEPARTMENTS PD,
        HR_ALL_ORGANIZATION_UNITS HAOU
    WHERE
        PAAF.JOB_ID = PJFV.JOB_ID(+)
    AND PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
    AND PAAF.ORGANIZATION_ID = HAOU.ORGANIZATION_ID(+)
    AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PAAF.PRIMARY_FLAG = 'Y'
    AND PAAF.ASSIGNMENT_TYPE = 'E'
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
)
```

**Key Filters:**
- `ASSIGNMENT_STATUS_TYPE = 'ACTIVE'` - Active assignments only
- `PRIMARY_FLAG = 'Y'` - Primary assignment only
- `ASSIGNMENT_TYPE = 'E'` - Employee assignments (not contingent worker)
- Date-tracked on SYSDATE

---

## 5. SUPERVISOR CTE
**Purpose:** Manager/Supervisor hierarchy with job details  
**Usage:** For manager approval queries and reporting structure

```sql
,SUPERVISOR AS (
    /*+ qb_name(SUPERVISOR) */
    SELECT
        PASF.PERSON_ID,
        PASF.MANAGER_ID,
        PPN.DISPLAY_NAME SUPERVISOR_NAME,
        PJF.NAME SUPERVISOR_JOB
    FROM
        PER_ASSIGNMENT_SUPERVISORS_F PASF,
        PER_PERSON_NAMES_F PPN,
        PER_ALL_ASSIGNMENTS_F PAA,
        PER_JOBS_F_TL PJF
    WHERE
        PASF.MANAGER_ID = PPN.PERSON_ID
    AND PPN.PERSON_ID = PAA.PERSON_ID
    AND PAA.JOB_ID = PJF.JOB_ID
    AND TRUNC(SYSDATE) BETWEEN TRUNC(NVL(PASF.EFFECTIVE_START_DATE, SYSDATE)) AND TRUNC(NVL(PASF.EFFECTIVE_END_DATE, SYSDATE))
    AND TRUNC(SYSDATE) BETWEEN TRUNC(NVL(PPN.EFFECTIVE_START_DATE, SYSDATE)) AND TRUNC(NVL(PPN.EFFECTIVE_END_DATE, SYSDATE))
    AND TRUNC(SYSDATE) BETWEEN TRUNC(NVL(PAA.EFFECTIVE_START_DATE, SYSDATE)) AND TRUNC(NVL(PAA.EFFECTIVE_END_DATE, SYSDATE))
    AND PPN.NAME_TYPE = 'GLOBAL'
    AND PAA.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PAA.PRIMARY_FLAG = 'Y'
    AND PAA.ASSIGNMENT_TYPE = 'E'
    AND PJF.LANGUAGE = 'US'
)
```

**Key Filters:**
- Links supervisor through MANAGER_ID
- Gets supervisor's job title
- Date-tracked on SYSDATE
- Active managers only

---

## 6. ANNUAL_LEAVE_BALANCE CTE
**Purpose:** Annual leave balance (latest accrual period)  
**Usage:** For annual leave balance queries

```sql
,ANNUAL_LEAVE_BALANCE AS (
    /*+ qb_name(ANNUAL_LEAVE_BALANCE) */
    SELECT
        APAE.END_BAL ANU_BAL,
        APAE.PERSON_ID,
        APAE.PLAN_ID
    FROM
        ANC_PER_ACCRUAL_ENTRIES APAE,
        ANC_ABSENCE_PLANS_VL AAPV
    WHERE
        APAE.PLAN_ID = AAPV.ABSENCE_PLAN_ID(+)
    AND UPPER(AAPV.NAME) = 'ANNUAL LEAVE'
    AND APAE.ACCRUAL_PERIOD = (
        SELECT MAX(APA.ACCRUAL_PERIOD)
        FROM ANC_PER_ACCRUAL_ENTRIES APA
        WHERE APAE.PERSON_ID = APA.PERSON_ID
        AND APA.PLAN_ID = APAE.PLAN_ID
        AND TO_CHAR(APA.ACCRUAL_PERIOD, 'YYYY') <= TO_CHAR(SYSDATE, 'YYYY')
    )
)
```

**Key Logic:**
- MAX(ACCRUAL_PERIOD) for current year
- Filter by plan name 'ANNUAL LEAVE'
- END_BAL is the current balance

---

## 7. LEAVE_BALANCES CTE (Generic - All Plans)
**Purpose:** Current leave balances for all absence plans  
**Usage:** For multi-plan balance reports

```sql
,LEAVE_BALANCES AS (
    /*+ qb_name(LEAVE_BALANCES) */
    SELECT
        APAE.PERSON_ID,
        APAE.PLAN_ID,
        AAPFTL.NAME PLAN_NAME,
        SUM(APAE.END_BAL) ACCRUAL_BALANCE,
        PPOS.PERIOD_OF_SERVICE_ID
    FROM
        ANC_PER_ACCRUAL_ENTRIES APAE,
        ANC_ABSENCE_PLANS_F_TL AAPFTL,
        PER_PERIODS_OF_SERVICE PPOS
    WHERE
        APAE.PLAN_ID = AAPFTL.ABSENCE_PLAN_ID
    AND APAE.PRD_OF_SVC_ID = PPOS.PERIOD_OF_SERVICE_ID
    AND AAPFTL.SOURCE_LANG = 'US'
    AND AAPFTL.LANGUAGE = 'US'
    AND TRUNC(SYSDATE) BETWEEN AAPFTL.EFFECTIVE_START_DATE AND AAPFTL.EFFECTIVE_END_DATE
    AND APAE.ACCRUAL_PERIOD = (
        SELECT MAX(ACC.ACCRUAL_PERIOD)
        FROM ANC_PER_ACCRUAL_ENTRIES ACC
        WHERE ACC.PERSON_ID = APAE.PERSON_ID
        AND APAE.PLAN_ID = ACC.PLAN_ID
        AND APAE.PRD_OF_SVC_ID = ACC.PRD_OF_SVC_ID
        AND ACC.ACCRUAL_PERIOD BETWEEN 
            TO_DATE('01/01/' || TO_CHAR(TRUNC(SYSDATE), 'YYYY'), 'DD/MM/YYYY') 
            AND TO_DATE('31/12/' || TO_CHAR(TRUNC(SYSDATE), 'YYYY'), 'DD/MM/YYYY')
    )
    GROUP BY APAE.PERSON_ID, APAE.PLAN_ID, AAPFTL.NAME, PPOS.PERIOD_OF_SERVICE_ID
)
```

**Key Logic:**
- SUM of all balances for each plan
- Current year accrual period
- Grouped by person and plan
- Links to period of service

---

## 8. LEAVE_TRANSACTIONS CTE (Approved Leaves)
**Purpose:** Absence entries with approval status (approved leaves only)  
**Usage:** For leave history and detail reports

```sql
,LEAVE_TRANSACTIONS AS (
    /*+ qb_name(LEAVE_TRANSACTIONS) */
    SELECT DISTINCT
        APAE.PERSON_ID,
        APAE.PER_ABSENCE_ENTRY_ID,
        APAE.ABSENCE_TYPE_ID,
        AATFT.NAME LEAVE_TYPE,
        INITCAP(TO_CHAR(APAE.START_DATE, 'DD-fmMON-YYYY', 'NLS_DATE_LANGUAGE = AMERICAN')) AS START_DATE_DISPLAY,
        INITCAP(TO_CHAR(APAE.END_DATE, 'DD-fmMON-YYYY', 'NLS_DATE_LANGUAGE = AMERICAN')) AS END_DATE_DISPLAY,
        APAE.START_DATE,
        APAE.END_DATE,
        APAE.DURATION,
        INITCAP(TO_CHAR(APAE.SUBMITTED_DATE, 'DD-fmMON-YYYY', 'NLS_DATE_LANGUAGE = AMERICAN')) AS SUBMITTED_DATE,
        TO_CHAR(APAE.CREATION_DATE, 'DD-MM-YYYY') AS CREATION_DATE,
        TO_CHAR(APAE.LAST_UPDATE_DATE, 'DD-MM-YYYY') AS LAST_UPDATE_DATE,
        APAE.ABSENCE_TYPE_REASON_ID,
        APAE.APPROVAL_STATUS_CD,
        APAE.ABSENCE_STATUS_CD
    FROM
        ANC_PER_ABS_ENTRIES APAE,
        ANC_ABSENCE_TYPES_F_TL AATFT,
        PERIOD P
    WHERE
        AATFT.LANGUAGE = 'US'
    AND AATFT.ABSENCE_TYPE_ID = APAE.ABSENCE_TYPE_ID
    AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
    AND APAE.APPROVAL_STATUS_CD NOT IN ('DENIED', 'ORA_WITHDRAWN', 'ORA_AWAIT_AWAIT')
    AND TRUNC(APAE.START_DATE) BETWEEN P.START_DATE AND P.END_DATE
)
```

**Key Filters:**
- Approved leaves only (excludes denied/withdrawn)
- Date range from PERIOD CTE
- Formatted dates for display
- DISTINCT to avoid workflow duplicates

---

## 9. ABSENCE_TYPES CTE
**Purpose:** Absence type master with translations  
**Usage:** For type classification and filtering

```sql
,ABSENCE_TYPES AS (
    /*+ qb_name(ABSENCE_TYPES) */
    SELECT
        AATFT.ABSENCE_TYPE_ID,
        AATFT.NAME ABSENCE_TYPE_NAME,
        AATFT.DESCRIPTION
    FROM
        ANC_ABSENCE_TYPES_F_TL AATFT
    WHERE
        AATFT.LANGUAGE = 'US'
    AND SYSDATE BETWEEN AATFT.EFFECTIVE_START_DATE AND AATFT.EFFECTIVE_END_DATE
)
```

**Key Filters:**
- `LANGUAGE = 'US'` - English translation
- Date-tracked on SYSDATE

---

## 10. ABSENCE_REASONS CTE
**Purpose:** Absence reason lookup with type relationship  
**Usage:** For reason classification in detail reports

```sql
,ABSENCE_REASONS AS (
    /*+ qb_name(ABSENCE_REASONS) */
    SELECT
        AARF.NAME REASON_NAME,
        AARF.ABSENCE_REASON_ID,
        AAT.ABSENCE_TYPE_REASON_ID
    FROM
        ANC_ABSENCE_REASONS_F AAR,
        ANC_ABSENCE_REASONS_F_TL AARF,
        ANC_ABSENCE_TYPE_REASONS_F AAT
    WHERE
        AAR.ABSENCE_REASON_ID = AARF.ABSENCE_REASON_ID
    AND AAR.ABSENCE_REASON_ID = AAT.ABSENCE_REASON_ID
    AND AARF.LANGUAGE = 'US'
    AND SYSDATE BETWEEN AAR.EFFECTIVE_START_DATE AND AAR.EFFECTIVE_END_DATE
    AND SYSDATE BETWEEN AARF.EFFECTIVE_START_DATE AND AARF.EFFECTIVE_END_DATE
    AND SYSDATE BETWEEN AAT.EFFECTIVE_START_DATE AND AAT.EFFECTIVE_END_DATE
)
```

**Key Filters:**
- Links reason to absence type through ABSENCE_TYPE_REASON_ID
- `LANGUAGE = 'US'`
- Date-tracked on SYSDATE

---

## 11. WORKFLOW_APPROVAL CTE
**Purpose:** Workflow approval history from WFTASK  
**Usage:** For approver name and approval tracking

```sql
,WORKFLOW_APPROVAL AS (
    /*+ qb_name(WORKFLOW_APPROVAL) */
    SELECT
        WF.IDENTIFICATIONKEY,
        WF.ASSIGNEESDISPLAYNAME APPROVER_NAME,
        WF.FROMUSERDISPLAYNAME REQUESTOR_NAME,
        WF.OUTCOME,
        WF.ASSIGNMENTDATE,
        WF.APPROVEDDATE
    FROM
        FA_FUSION_SOAINFRA.WFTASK WF
    WHERE
        WF.OUTCOME IN ('APPROVE')
    AND WF.ASSIGNEES IS NOT NULL
    AND WF.WORKFLOWPATTERN NOT IN ('AGGREGATION', 'FYI')
)
```

**Key Filters:**
- `OUTCOME = 'APPROVE'` - Approved tasks only
- Excludes aggregation and FYI workflows
- IDENTIFICATIONKEY links to PER_ABSENCE_ENTRY_ID

**Note:** This table requires cross-schema access to FA_FUSION_SOAINFRA. May not be available in all environments.

---

## 12. LEAVE_HISTORY CTE (Historical Comparison)
**Purpose:** Historical leave summary for comparison  
**Usage:** For "All Previous Leave Taken Records" column

```sql
,LEAVE_HISTORY AS (
    /*+ qb_name(LEAVE_HISTORY) */
    SELECT
        APAE.PERSON_ID,
        APAE.ABSENCE_TYPE_ID,
        COUNT(*) PREVIOUS_COUNT,
        SUM(APAE.DURATION) TOTAL_DAYS_TAKEN,
        MAX(APAE.END_DATE) LAST_LEAVE_DATE,
        LISTAGG(
            INITCAP(TO_CHAR(APAE.START_DATE, 'DD-MON-YY')) || ' (' || APAE.DURATION || ' days)',
            ', '
        ) WITHIN GROUP (ORDER BY APAE.START_DATE DESC) AS PREVIOUS_DETAILS
    FROM
        ANC_PER_ABS_ENTRIES APAE
    WHERE
        APAE.APPROVAL_STATUS_CD = 'APPROVED'
    AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
    GROUP BY APAE.PERSON_ID, APAE.ABSENCE_TYPE_ID
)
```

**Key Logic:**
- COUNT of previous approved leaves by type
- SUM of total days taken
- LISTAGG for comma-separated history
- Approved leaves only

**Usage Example:**
```sql
LEFT JOIN LEAVE_HISTORY LH
    ON LT.PERSON_ID = LH.PERSON_ID
    AND LT.ABSENCE_TYPE_ID = LH.ABSENCE_TYPE_ID
    AND LT.PER_ABSENCE_ENTRY_ID <> [exclude current entry if needed]
```

---

## 13. PERIOD_OF_SERVICE CTE
**Purpose:** Employment term validation  
**Usage:** For hire date and termination date context

```sql
,PERIOD_OF_SERVICE AS (
    /*+ qb_name(PERIOD_OF_SERVICE) */
    SELECT
        PPOS.PERSON_ID,
        PPOS.PERIOD_OF_SERVICE_ID,
        TO_CHAR(NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START), 'DD-MM-YYYY') HIRE_DATE,
        PPOS.DATE_START,
        PPOS.ACTUAL_TERMINATION_DATE
    FROM
        PER_PERIODS_OF_SERVICE PPOS
    WHERE
        TRUNC(SYSDATE) BETWEEN PPOS.DATE_START 
            AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
)
```

**Key Filters:**
- Active employment periods only (SYSDATE between start and termination)
- Uses ORIGINAL_DATE_OF_HIRE or DATE_START for hire date

---

## 14. BUSINESS_UNIT CTE
**Purpose:** Business unit lookup  
**Usage:** For business unit name in reports

```sql
,BUSINESS_UNIT AS (
    /*+ qb_name(BUSINESS_UNIT) */
    SELECT
        HAOU.ORGANIZATION_ID BUSINESS_UNIT_ID,
        HAOU.NAME BUSINESS_UNIT_NAME
    FROM
        HR_ALL_ORGANIZATION_UNITS HAOU
)
```

**Simple lookup:** Links using BUSINESS_UNIT_ID from assignment

---

## 15. ABSENCE_STATUS_LOOKUP CTE
**Purpose:** Decode absence status codes to meanings  
**Usage:** For user-friendly status display

```sql
,ABSENCE_STATUS_LOOKUP AS (
    /*+ qb_name(ABSENCE_STATUS_LOOKUP) */
    SELECT
        HL.LOOKUP_CODE,
        HL.MEANING STATUS_MEANING
    FROM
        HR_LOOKUPS HL
    WHERE
        HL.LOOKUP_TYPE = 'ANC_PER_ABS_ENT_STATUS'
)
```

**Status Types:**
- `'ANC_PER_ABS_ENT_STATUS'` - Absence status
- `'ANC_PER_ABS_ENT_APROVAL_STATUS'` - Approval status

---

## 16. APPROVAL_STATUS_LOOKUP CTE
**Purpose:** Decode approval status codes to meanings  
**Usage:** For user-friendly approval status display

```sql
,APPROVAL_STATUS_LOOKUP AS (
    /*+ qb_name(APPROVAL_STATUS_LOOKUP) */
    SELECT
        HL.LOOKUP_CODE,
        HL.MEANING APPROVAL_MEANING
    FROM
        HR_LOOKUPS HL
    WHERE
        HL.LOOKUP_TYPE = 'ANC_PER_ABS_ENT_APROVAL_STATUS'
)
```

---

## 17. FULL_EMPLOYEE_DETAILS CTE (Composite)
**Purpose:** Complete employee record with all context  
**Usage:** For comprehensive employee reports (combines multiple CTEs)

```sql
,FULL_EMPLOYEE_DETAILS AS (
    /*+ qb_name(FULL_EMPLOYEE_DETAILS) */
    SELECT
        EM.PERSON_ID,
        EM.PERSON_NUMBER,
        EM.DISPLAY_NAME,
        EM.FULL_NAME,
        EM.EMPLOYEE_TYPE,
        EA.ASSIGNMENT_ID,
        EA.ASSIGNMENT_NUMBER,
        EA.JOB_NAME,
        EA.DEPARTMENT_NAME,
        EA.ORG_NAME,
        EA.BUSINESS_UNIT_ID,
        BU.BUSINESS_UNIT_NAME,
        SP.SUPERVISOR_NAME,
        SP.SUPERVISOR_JOB,
        POS.HIRE_DATE,
        POS.PERIOD_OF_SERVICE_ID
    FROM
        EMP_MASTER EM,
        EMP_ASSIGNMENT EA,
        BUSINESS_UNIT BU,
        SUPERVISOR SP,
        PERIOD_OF_SERVICE POS
    WHERE
        EM.PERSON_ID = EA.PERSON_ID
    AND EA.BUSINESS_UNIT_ID = BU.BUSINESS_UNIT_ID(+)
    AND EM.PERSON_ID = SP.PERSON_ID(+)
    AND EM.PERSON_ID = POS.PERSON_ID
)
```

**Composite CTE:** Combines employee, assignment, supervisor, and period of service data

---

## 18. WORKFLOW_APPROVAL_DETAIL CTE (Enhanced)
**Purpose:** Detailed workflow approval with approver and requestor  
**Usage:** For comprehensive approval tracking with names

```sql
,WORKFLOW_APPROVAL_DETAIL AS (
    /*+ qb_name(WORKFLOW_APPROVAL_DETAIL) */
    SELECT
        WF.ASSIGNEESDISPLAYNAME APPR_NAME,
        WF.FROMUSERDISPLAYNAME REPL_NAME,
        WF.IDENTIFICATIONKEY
    FROM
        FA_FUSION_SOAINFRA.WFTASK WF
    WHERE
        WF.OUTCOME IN ('APPROVE')
    AND WF.ASSIGNEES IS NOT NULL
    AND WF.WORKFLOWPATTERN NOT IN ('AGGREGATION', 'FYI')
)
```

**Key Features:**
- ASSIGNEESDISPLAYNAME - Approver name
- FROMUSERDISPLAYNAME - Requestor/Submitter name
- Excludes aggregation and FYI patterns

---

## 19. SALARY_ADVANCE_RECOVERY CTE
**Purpose:** Salary advance recovery from payroll elements  
**Usage:** For linking salary advance to leave entries

```sql
,SALARY_ADVANCE_RECOVERY AS (
    /*+ qb_name(SALARY_ADVANCE_RECOVERY) */
    SELECT
        PEEF.PERSON_ID,
        PEEV.SCREEN_ENTRY_VALUE ADVANCE_SALARY
    FROM
        PAY_ELEMENT_ENTRIES_F PEEF,
        PAY_ELEMENT_ENTRY_VALUES_F PEEV,
        PAY_INPUT_VALUES_F PIVF,
        PAY_ELEMENT_TYPES_F PETF,
        PER_ALL_PEOPLE_F PAPF
    WHERE
        PEEF.ELEMENT_ENTRY_ID = PEEV.ELEMENT_ENTRY_ID
    AND PIVF.ELEMENT_TYPE_ID = PETF.ELEMENT_TYPE_ID
    AND PIVF.INPUT_VALUE_ID = PEEV.INPUT_VALUE_ID
    AND PIVF.ELEMENT_TYPE_ID = PEEF.ELEMENT_TYPE_ID
    AND PIVF.BASE_NAME IN ('AMOUNT', 'PAY VALUE')
    AND PETF.BASE_ELEMENT_NAME = 'SALARY ADVANCE RECOVERY RETRO'
    AND PEEF.PERSON_ID = PAPF.PERSON_ID
)
```

**Key Features:**
- Links to payroll element entries
- Filters by element type: 'SALARY ADVANCE RECOVERY RETRO'
- Gets screen entry value for advance amount

---

## 20. LEAVE_DETAILS_ENHANCED CTE
**Purpose:** Enhanced leave details with workflow and reasons  
**Usage:** For comprehensive leave detail reports

```sql
,LEAVE_DETAILS_ENHANCED AS (
    /*+ qb_name(LEAVE_DETAILS_ENHANCED) */
    SELECT DISTINCT
        APAPAE.PERSON_ID,
        AATFT.NAME L_TYPE,
        APAPAE.PER_ABSENCE_ENTRY_ID,
        WF.APPR_NAME,
        WF.REPL_NAME,
        INITCAP(TO_CHAR(APAPAE.START_DATE, 'DD-fmMON-YYYY', 'NLS_DATE_LANGUAGE = AMERICAN')) AS ACTUAL_LEAVE_START_DATE,
        INITCAP(TO_CHAR(APAPAE.END_DATE, 'DD-fmMON-YYYY', 'NLS_DATE_LANGUAGE = AMERICAN')) AS ACTUAL_LEAVE_END_DATE,
        APAPAE.DURATION,
        INITCAP(TO_CHAR(APAPAE.SUBMITTED_DATE, 'DD-fmMON-YYYY', 'NLS_DATE_LANGUAGE = AMERICAN')) AS DATE_OF_LEAVE_APPLY,
        REA.NAME ABSENCE_REASON_NAME,
        PERIOD P
    FROM
        ANC_ABSENCE_TYPES_F_TL AATFT,
        ANC_PER_ABS_ENTRIES APAPAE,
        WORKFLOW_APPROVAL_DETAIL WF,
        ABSENCE_REASONS REA,
        PERIOD P
    WHERE
        AATFT.LANGUAGE = 'US'
    AND AATFT.NAME IN ('Annual Leave')
    AND AATFT.ABSENCE_TYPE_ID = APAPAE.ABSENCE_TYPE_ID(+)
    AND TO_CHAR(APAPAE.PER_ABSENCE_ENTRY_ID) = WF.IDENTIFICATIONKEY(+)
    AND APAPAE.ABSENCE_STATUS_CD <> ('ORA_WITHDRAWN')
    AND TRUNC(APAPAE.START_DATE) BETWEEN 
        TRUNC(CAST(NVL((:P_START_DATE), (SELECT MIN(START_DATE) FROM ANC_PER_ABS_ENTRIES APAPAE1 WHERE APAPAE1.PER_ABSENCE_ENTRY_ID = APAPAE.PER_ABSENCE_ENTRY_ID)) AS DATE))
        AND TRUNC(CAST(NVL((:P_END_DATE), (LAST_DAY(SYSDATE))) AS DATE))
    AND APAPAE.ABSENCE_TYPE_REASON_ID = REA.ABSENCE_TYPE_REASON_ID(+)
    AND APAPAE.APPROVAL_STATUS_CD <> ('DENIED')
)
```

**Key Features:**
- Includes workflow approver/requestor names
- Includes absence reason
- Date range filtering with dynamic defaults
- INITCAP for proper name casing
- Formatted dates for display

---

## 21. ANNUAL_LEAVE_BALANCE_ADVANCED CTE
**Purpose:** Annual leave balance with UPPER case-insensitive matching  
**Usage:** For annual leave balance queries (case-insensitive)

```sql
,ANNUAL_LEAVE_BALANCE_ADVANCED AS (
    /*+ qb_name(ANNUAL_LEAVE_BALANCE_ADVANCED) */
    SELECT
        APAE.END_BAL ANU_BAL,
        APAE.PERSON_ID,
        APAE.PLAN_ID
    FROM
        ANC_PER_ACCRUAL_ENTRIES APAE,
        ANC_ABSENCE_PLANS_VL AAPV
    WHERE
        APAE.PLAN_ID = AAPV.ABSENCE_PLAN_ID(+)
    AND UPPER(AAPV.NAME) = 'ANNUAL LEAVE'
    AND APAE.ACCRUAL_PERIOD = (
        SELECT MAX(APA.ACCRUAL_PERIOD)
        FROM ANC_PER_ACCRUAL_ENTRIES APA
        WHERE APAE.PERSON_ID = APA.PERSON_ID
        AND APA.PLAN_ID = APAE.PLAN_ID
        AND TO_CHAR(APA.ACCRUAL_PERIOD, 'YYYY') <= TO_CHAR(SYSDATE, 'YYYY')
    )
)
```

**Key Features:**
- UPPER function for case-insensitive plan name matching
- Handles variations in plan name casing

---

## 22. CURRENT_ABSENCE_ENTRIES CTE
**Purpose:** Current/ongoing absence entries only  
**Usage:** For current absence activity reports

```sql
,CURRENT_ABSENCE_ENTRIES AS (
    /*+ qb_name(CURRENT_ABSENCE_ENTRIES) */
    SELECT
        ABS_ENTR.PERSON_ID,
        ABS_TYPE.NAME ABSENCE_TYPE,
        TO_CHAR(ABS_ENTR.START_DATE, 'DD-MM-YYYY') ABSENCE_START,
        TO_CHAR(ABS_ENTR.END_DATE, 'DD-MM-YYYY') ABSENCE_END,
        ABS_ENTR.START_DATE,
        ABS_ENTR.DURATION,
        ABS_ENTR.ABSENCE_TYPE_REASON_ID,
        TO_CHAR(ABS_ENTR.CREATION_DATE, 'DD-MM-YYYY') SUBMITTED_DATE,
        TO_CHAR(ABS_ENTR.LAST_UPDATE_DATE, 'DD-MM-YYYY') CONFIRMED_DATE,
        ABS_ENTR.APPROVAL_STATUS_CD,
        ABS_ENTR.ABSENCE_STATUS_CD
    FROM
        ANC_PER_ABS_ENTRIES ABS_ENTR,
        ANC_ABSENCE_TYPES_F_TL ABS_TYPE
    WHERE
        ABS_ENTR.ABSENCE_TYPE_ID = ABS_TYPE.ABSENCE_TYPE_ID
    AND ABS_TYPE.LANGUAGE = 'US'
    AND ABS_TYPE.SOURCE_LANG = 'US'
    AND ABS_ENTR.APPROVAL_STATUS_CD NOT IN ('DENIED', 'ORA_WITHDRAN', 'ORA_AWAIT_AWAIT')
    AND TRUNC(SYSDATE) BETWEEN ABS_ENTR.START_DATE AND ABS_ENTR.END_DATE
)
```

**Key Features:**
- Filters for current absences only (SYSDATE BETWEEN START_DATE AND END_DATE)
- SOURCE_LANG filter to prevent duplicates
- Formatted dates for display

---

## 23. TERMINATED_EMP_FILTER CTE
**Purpose:** Employee filter with termination date handling  
**Usage:** For queries including terminated employees

```sql
,TERMINATED_EMP_FILTER AS (
    /*+ qb_name(TERMINATED_EMP_FILTER) */
    SELECT
        PER.PERSON_ID,
        PER.FULL_NAME,
        ASSI_NEW.ASSIGNMENT_NUMBER,
        ASSI_NEW.ORGANIZATION_ID,
        ASSI_NEW.BUSINESS_UNIT_ID,
        ASSI_NEW.PERIOD_OF_SERVICE_ID,
        PPOS.ACTUAL_TERMINATION_DATE
    FROM
        PER_PERSON_NAMES_F PER,
        PER_ALL_ASSIGNMENTS_F ASSI_NEW,
        PER_ALL_PEOPLE_F PAPF,
        PER_PERIODS_OF_SERVICE PPOS,
        PER_PERSON_TYPES_TL PPTTL,
        PER_PEOPLE_LEGISLATIVE_F PPLF
    WHERE
        PER.PERSON_ID = ASSI_NEW.PERSON_ID
    AND PAPF.PERSON_ID = ASSI_NEW.PERSON_ID
    AND ASSI_NEW.PRIMARY_FLAG = 'Y'
    AND ASSI_NEW.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PER.NAME_TYPE = 'GLOBAL'
    AND PPOS.PERSON_ID = PER.PERSON_ID
    AND PPOS.PERIOD_OF_SERVICE_ID = ASSI_NEW.PERIOD_OF_SERVICE_ID
    AND ASSI_NEW.ASSIGNMENT_TYPE = 'E'
    AND PPLF.PERSON_ID = PAPF.PERSON_ID
    AND PPTTL.PERSON_TYPE_ID = ASSI_NEW.PERSON_TYPE_ID
    AND PPTTL.LANGUAGE = 'US'
    AND PPTTL.SOURCE_LANG = 'US'
    AND TRUNC(SYSDATE) BETWEEN PPOS.DATE_START AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
    AND LEAST(NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY')), TRUNC(SYSDATE))
        BETWEEN PER.EFFECTIVE_START_DATE AND PER.EFFECTIVE_END_DATE
    AND LEAST(NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY')), TRUNC(SYSDATE))
        BETWEEN ASSI_NEW.EFFECTIVE_START_DATE AND ASSI_NEW.EFFECTIVE_END_DATE
    AND LEAST(NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY')), TRUNC(SYSDATE))
        BETWEEN PPLF.EFFECTIVE_START_DATE AND PPLF.EFFECTIVE_END_DATE
    AND LEAST(NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY')), TRUNC(SYSDATE))
        BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
)
```

**Key Features:**
- Uses LEAST function for date-track filtering with termination dates
- Handles both active and terminated employees
- Includes period of service validation
- Includes legislative data filter

---

## 24. ACCRUAL_BALANCE_CURRENT_YEAR CTE
**Purpose:** Accrual balance for current year only  
**Usage:** For year-specific balance queries

```sql
,ACCRUAL_BALANCE_CURRENT_YEAR AS (
    /*+ qb_name(ACCRUAL_BALANCE_CURRENT_YEAR) */
    SELECT
        PAPF.PERSON_ID,
        SUM(APAE.END_BAL) ACCRUAL_BAL
    FROM
        PER_ALL_PEOPLE_F PAPF,
        ANC_PER_ACCRUAL_ENTRIES APAE,
        ANC_ABSENCE_PLANS_F_TL AAPFTL,
        PER_PERIODS_OF_SERVICE PPOS
    WHERE
        PAPF.PERSON_ID = APAE.PERSON_ID
    AND APAE.PRD_OF_SVC_ID = PPOS.PERIOD_OF_SERVICE_ID
    AND APAE.PLAN_ID = AAPFTL.ABSENCE_PLAN_ID
    AND AAPFTL.SOURCE_LANG = 'US'
    AND AAPFTL.LANGUAGE = 'US'
    AND AAPFTL.NAME = 'Annual Leave Plan'
    AND TRUNC(SYSDATE) BETWEEN AAPFTL.EFFECTIVE_START_DATE AND AAPFTL.EFFECTIVE_END_DATE
    AND APAE.ACCRUAL_PERIOD = (
        SELECT MAX(ACC.ACCRUAL_PERIOD)
        FROM ANC_PER_ACCRUAL_ENTRIES ACC
        WHERE ACC.PERSON_ID = APAE.PERSON_ID
        AND APAE.PLAN_ID = ACC.PLAN_ID
        AND APAE.PRD_OF_SVC_ID = ACC.PRD_OF_SVC_ID
        AND ACC.ACCRUAL_PERIOD BETWEEN 
            TO_DATE('01/01/' || TO_CHAR(TRUNC(SYSDATE), 'YYYY'), 'DD/MM/YYYY') 
            AND TO_DATE('31/12/' || TO_CHAR(TRUNC(SYSDATE), 'YYYY'), 'DD/MM/YYYY')
    )
    GROUP BY PAPF.PERSON_ID
)
```

**Key Features:**
- Current year filter on accrual period
- SUM aggregation for total balance
- SOURCE_LANG and LANGUAGE filters
- Period of service validation

---

## 🎯 Usage Patterns

### Pattern 1: Simple Leave History Query
```sql
WITH PERIOD AS (...),
     EMP_MASTER AS (...),
     LEAVE_TRANSACTIONS AS (...)
SELECT
    EM.PERSON_NUMBER,
    EM.DISPLAY_NAME,
    LT.LEAVE_TYPE,
    LT.START_DATE_DISPLAY,
    LT.END_DATE_DISPLAY,
    LT.DURATION
FROM
    EMP_MASTER EM,
    LEAVE_TRANSACTIONS LT
WHERE
    EM.PERSON_ID = LT.PERSON_ID
```

### Pattern 2: Leave Balance Query
```sql
WITH PERIOD AS (...),
     FULL_EMPLOYEE_DETAILS AS (...),
     ANNUAL_LEAVE_BALANCE AS (...)
SELECT
    FED.PERSON_NUMBER,
    FED.DISPLAY_NAME,
    FED.DEPARTMENT_NAME,
    NVL(ROUND(ALB.ANU_BAL, 2), 0) ANNUAL_LEAVE_BALANCE
FROM
    FULL_EMPLOYEE_DETAILS FED,
    ANNUAL_LEAVE_BALANCE ALB
WHERE
    FED.PERSON_ID = ALB.PERSON_ID(+)
```

### Pattern 3: Leave History with Comparison
```sql
WITH PERIOD AS (...),
     EMP_MASTER AS (...),
     LEAVE_TRANSACTIONS AS (...),
     LEAVE_HISTORY AS (...)
SELECT
    EM.PERSON_NUMBER,
    LT.LEAVE_TYPE,
    LT.START_DATE_DISPLAY,
    LT.DURATION,
    CASE 
        WHEN LH.PREVIOUS_COUNT > 0 THEN 'Full Leave History'
        ELSE 'New Leave Request'
    END AS HISTORY_TYPE,
    NVL(LH.PREVIOUS_DETAILS, 'None') PREVIOUS_LEAVE_RECORDS
FROM
    EMP_MASTER EM,
    LEAVE_TRANSACTIONS LT,
    LEAVE_HISTORY LH
WHERE
    EM.PERSON_ID = LT.PERSON_ID
AND LT.PERSON_ID = LH.PERSON_ID(+)
AND LT.ABSENCE_TYPE_ID = LH.ABSENCE_TYPE_ID(+)
```

---

## ✅ Validation Checklist

Before using any CTE from this repository:

- [ ] CTE has `/*+ qb_name(NAME) */` hint ✓
- [ ] All joins use Oracle Traditional Syntax ✓
- [ ] Date-track filters applied to `_F` tables ✓
- [ ] `LANGUAGE = 'US'` applied to `_TL` tables ✓
- [ ] Status filters applied (approved, not withdrawn) ✓
- [ ] Outer joins (+) used where appropriate ✓

---

## 📝 Notes

1. **DO NOT modify these CTEs** without understanding the constraints from ABSENCE_MASTER.md
2. **ALWAYS copy complete CTEs** - do not write fresh joins
3. **Test workflow CTEs** in your environment - FA_FUSION_SOAINFRA may require special access
4. **Combine CTEs efficiently** - use composite CTEs like FULL_EMPLOYEE_DETAILS to reduce redundancy

---

**END OF ABSENCE_REPOSITORIES.md**

