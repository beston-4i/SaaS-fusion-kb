# HCM Absence Templates: Report Skeletons

**Module:** HCM Absence Management  
**Purpose:** Ready-to-use templates for absence/leave reports  
**Tag:** `#HCM #ABSENCE #TEMPLATES #REPORTS`  
**Date:** 18-12-25

---

## 📋 Template Index

| Template Name | Purpose | Complexity | Key CTEs Required |
|---------------|---------|------------|-------------------|
| **T1: Leave History Report** | All approved leaves with historical comparison | Medium | PERIOD, EMP_MASTER, LEAVE_TRANSACTIONS, LEAVE_HISTORY |
| **T2: Leave Balance Report** | Current absence activity with balances | Medium | PERIOD, FULL_EMPLOYEE_DETAILS, LEAVE_BALANCES |
| **T3: Leave Record Form** | Single leave transaction detail | Simple | PERIOD, FULL_EMPLOYEE_DETAILS, LEAVE_TRANSACTIONS |
| **T4: Leave Details Report** | Comprehensive leave with workflow and supervisor | Complex | DEPARTMENTS, ANNUAL_LEAVE_BALANCE, SUPERVISOR, LEAVE_DETAILS (with workflow) |
| **T5: Leave Summary Report** | Aggregated leave statistics by type | Medium | PERIOD, EMP_MASTER, LEAVE_TRANSACTIONS |
| **T6: Absence Register** | Comprehensive absence ledger | Complex | ALL CTEs |

---

## TEMPLATE 1: Leave History Report (Approved Leaves Only)

### Purpose
Display all approved leaves with historical comparison and previous leave records.

### Business Use Case
HR needs to review approved leave history and compare current leave requests against historical patterns.

### Template Structure

```sql
/******************************************************************************
 * Report: Full Leave History - Approved Leaves Only
 * Module: HCM Absence Management
 * Purpose: Display approved leaves with historical comparison
 * 
 * Parameters:
 *   :P_START_DATE    - Start date (default: 12 months ago)
 *   :P_END_DATE      - End date (default: current month end)
 *   :P_EMP_NO        - Employee number (or 'ALL')
 *   :P_EMP_NAME      - Employee name (or 'ALL')
 *
 * Author: [Your Name]
 * Date: [SYSDATE]
 ******************************************************************************/

WITH PERIOD AS (
    /*+ qb_name(PERIOD) */
    SELECT
        TRUNC(CAST(NVL(:P_START_DATE, ADD_MONTHS(TRUNC(SYSDATE), -12)) AS DATE)) AS START_DATE,
        TRUNC(CAST(NVL(:P_END_DATE, LAST_DAY(SYSDATE)) AS DATE)) AS END_DATE,
        :P_EMP_NO AS EMPLOYEE_NUMBER,
        :P_EMP_NAME AS EMPLOYEE_NAME
    FROM DUAL
)
-- Copy EMP_MASTER CTE from ABSENCE_REPOSITORIES.md
,EMP_MASTER AS (
    /*+ qb_name(EMP_MASTER) */
    SELECT
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER,
        PPNF.DISPLAY_NAME,
        PPNF.FULL_NAME
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF
    WHERE
        PAPF.PERSON_ID = PPNF.PERSON_ID
    AND PPNF.NAME_TYPE = 'GLOBAL'
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE) AND TRUNC(PPNF.EFFECTIVE_END_DATE)
)
-- Copy LEAVE_TRANSACTIONS CTE from ABSENCE_REPOSITORIES.md
,LEAVE_TRANSACTIONS AS (
    /*+ qb_name(LEAVE_TRANSACTIONS) */
    SELECT DISTINCT
        APAE.PERSON_ID,
        APAE.PER_ABSENCE_ENTRY_ID,
        APAE.ABSENCE_TYPE_ID,
        AATFT.NAME LEAVE_TYPE,
        APAE.START_DATE,
        APAE.END_DATE,
        INITCAP(TO_CHAR(APAE.START_DATE, 'DD-fmMON-YYYY', 'NLS_DATE_LANGUAGE = AMERICAN')) AS START_DATE_DISPLAY,
        INITCAP(TO_CHAR(APAE.END_DATE, 'DD-fmMON-YYYY', 'NLS_DATE_LANGUAGE = AMERICAN')) AS END_DATE_DISPLAY,
        APAE.DURATION,
        APAE.COMMENTS REMARKS
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
-- Copy LEAVE_HISTORY CTE from ABSENCE_REPOSITORIES.md (for historical comparison)
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
        ANC_PER_ABS_ENTRIES APAE,
        PERIOD P
    WHERE
        APAE.APPROVAL_STATUS_CD = 'APPROVED'
    AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
    AND APAE.START_DATE < P.START_DATE
    GROUP BY APAE.PERSON_ID, APAE.ABSENCE_TYPE_ID
)
-- FINAL SELECT: Organized columns (IDs -> Dates -> Amounts -> Status -> Desc)
SELECT
    -- Employee Identification
    EM.PERSON_NUMBER AS EMPLOYEE_NUMBER,
    EM.DISPLAY_NAME AS EMPLOYEE_NAME,
    
    -- Leave Dates
    LT.START_DATE_DISPLAY AS "Start Date",
    LT.END_DATE_DISPLAY AS "End Date",
    
    -- Leave Classification
    LT.LEAVE_TYPE AS "Type of Leave",
    
    -- Duration
    NVL(LT.DURATION, 0) AS "Number of Days Taken",
    
    -- Historical Comparison
    CASE 
        WHEN LH.PREVIOUS_COUNT > 0 THEN 'Full Leave History'
        ELSE 'New Leave Request'
    END AS "Full Leave History vs New Leave Request",
    
    -- Previous Leave Records
    NVL(LH.PREVIOUS_DETAILS, 'None') AS "All Previous Leave Taken Records",
    
    -- Remarks
    NVL(LT.REMARKS, 'Approved leave') AS "Remarks/Leave Details"
    
FROM
    EMP_MASTER EM,
    LEAVE_TRANSACTIONS LT,
    LEAVE_HISTORY LH,
    PERIOD P
WHERE
    EM.PERSON_ID = LT.PERSON_ID
AND LT.PERSON_ID = LH.PERSON_ID(+)
AND LT.ABSENCE_TYPE_ID = LH.ABSENCE_TYPE_ID(+)
-- Parameter Filters
AND (EM.PERSON_NUMBER IN (P.EMPLOYEE_NUMBER) OR 'ALL' IN (P.EMPLOYEE_NUMBER || 'ALL'))
AND (EM.DISPLAY_NAME IN (P.EMPLOYEE_NAME) OR 'ALL' IN (P.EMPLOYEE_NAME || 'ALL'))

ORDER BY 
    TO_NUMBER(EM.PERSON_NUMBER),
    LT.START_DATE DESC
```

### Expected Output Columns
1. Employee Number
2. Employee Name
3. Start Date (formatted)
4. End Date (formatted)
5. Type of Leave
6. Number of Days Taken
7. Full Leave History vs New Leave Request
8. All Previous Leave Taken Records
9. Remarks/Leave Details

### Key Constraints Applied
✓ Approved leaves only  
✓ Date range filtering  
✓ Historical comparison logic  
✓ Oracle Traditional Join Syntax  
✓ CTE performance hints  

---

## TEMPLATE 2: Leave Balance Report (Current Absences)

### Purpose
Display current/future absence activity with accrual balances.

### Business Use Case
Managers need to see who is currently on leave or has upcoming approved leave, along with their remaining leave balance.

### Template Structure

```sql
/******************************************************************************
 * Report: Absence Balance Details - Current and Future Leaves
 * Module: HCM Absence Management
 * Purpose: Show current absence activity with leave balances
 * 
 * Parameters:
 *   (No parameters - shows current/future absences)
 *
 * Author: [Your Name]
 * Date: [SYSDATE]
 ******************************************************************************/

WITH EMP_BASE AS (
    /*+ qb_name(EMP_BASE) */
    SELECT
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER,
        PER.FULL_NAME,
        ASSI_NEW.ASSIGNMENT_NUMBER,
        ASSI_NEW.BUSINESS_UNIT_ID,
        ASSI_NEW.PERIOD_OF_SERVICE_ID,
        ASSI_NEW.ORGANIZATION_ID,
        PPTTL.USER_PERSON_TYPE
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PER,
        PER_ALL_ASSIGNMENTS_F ASSI_NEW,
        PER_PERSON_TYPES_TL PPTTL
    WHERE
        PER.PERSON_ID = ASSI_NEW.PERSON_ID
    AND PAPF.PERSON_ID = ASSI_NEW.PERSON_ID
    AND ASSI_NEW.PRIMARY_FLAG = 'Y'
    AND ASSI_NEW.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PER.NAME_TYPE = 'GLOBAL'
    AND PPTTL.PERSON_TYPE_ID = ASSI_NEW.PERSON_TYPE_ID
    AND PPTTL.LANGUAGE = 'US'
    AND PPTTL.SOURCE_LANG = 'US'
    AND ASSI_NEW.ASSIGNMENT_TYPE = 'E'
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PER.EFFECTIVE_START_DATE) AND TRUNC(PER.EFFECTIVE_END_DATE)
    AND TRUNC(SYSDATE) BETWEEN TRUNC(ASSI_NEW.EFFECTIVE_START_DATE) AND TRUNC(ASSI_NEW.EFFECTIVE_END_DATE)
)
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
,ORG_UNITS AS (
    /*+ qb_name(ORG_UNITS) */
    SELECT
        HAOU.ORGANIZATION_ID,
        HAOU.NAME DEPARTMENT
    FROM
        HR_ALL_ORGANIZATION_UNITS HAOU
)
,BUSINESS_UNITS AS (
    /*+ qb_name(BUSINESS_UNITS) */
    SELECT
        HAOU.ORGANIZATION_ID BUSINESS_UNIT_ID,
        HAOU.NAME BUSINESS_UNIT
    FROM
        HR_ALL_ORGANIZATION_UNITS HAOU
)
,LEAVE_BALANCES AS (
    /*+ qb_name(LEAVE_BALANCES) */
    SELECT
        APAE.PERSON_ID,
        APAE.PRD_OF_SVC_ID,
        SUM(APAE.END_BAL) ACCRUAL_BAL
    FROM
        ANC_PER_ACCRUAL_ENTRIES APAE,
        ANC_ABSENCE_PLANS_F_TL AAPFTL
    WHERE
        APAE.PLAN_ID = AAPFTL.ABSENCE_PLAN_ID
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
    GROUP BY APAE.PERSON_ID, APAE.PRD_OF_SVC_ID
)
,ABSENCE_ENTRIES AS (
    /*+ qb_name(ABSENCE_ENTRIES) */
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
    -- Current or Future Absences
    AND TRUNC(SYSDATE) BETWEEN ABS_ENTR.START_DATE AND ABS_ENTR.END_DATE
)
,ABSENCE_REASONS AS (
    /*+ qb_name(ABSENCE_REASONS) */
    SELECT
        REASON_TL.NAME REASON_NAME,
        REASON_F.ABSENCE_TYPE_REASON_ID
    FROM
        ANC_ABSENCE_TYPE_REASONS_F REASON_F,
        ANC_ABSENCE_REASONS_F_TL REASON_TL
    WHERE
        REASON_F.ABSENCE_REASON_ID = REASON_TL.ABSENCE_REASON_ID
    AND REASON_TL.LANGUAGE = 'US'
    AND REASON_TL.SOURCE_LANG = 'US'
    AND SYSDATE BETWEEN REASON_F.EFFECTIVE_START_DATE AND REASON_F.EFFECTIVE_END_DATE
    AND SYSDATE BETWEEN REASON_TL.EFFECTIVE_START_DATE AND REASON_TL.EFFECTIVE_END_DATE
)
,APPROVAL_STATUS AS (
    /*+ qb_name(APPROVAL_STATUS) */
    SELECT
        HL.LOOKUP_CODE,
        HL.MEANING APPROVAL_STATUS
    FROM
        HR_LOOKUPS HL
    WHERE
        HL.LOOKUP_TYPE = 'ANC_PER_ABS_ENT_APROVAL_STATUS'
)
,ABSENCE_STATUS AS (
    /*+ qb_name(ABSENCE_STATUS) */
    SELECT
        HL.LOOKUP_CODE,
        HL.MEANING ABSENCE_STATUS
    FROM
        HR_LOOKUPS HL
    WHERE
        HL.LOOKUP_TYPE = 'ANC_PER_ABS_ENT_STATUS'
)
-- FINAL SELECT
SELECT DISTINCT
    -- Employee Identification
    EB.PERSON_NUMBER PERSON_NUMBER,
    EB.ASSIGNMENT_NUMBER,
    EB.FULL_NAME,
    
    -- Absence Details
    AE.ABSENCE_TYPE,
    AE.ABSENCE_START,
    AE.ABSENCE_END,
    AE.START_DATE,
    AE.DURATION,
    
    -- Organizational Context
    OU.DEPARTMENT,
    NULL NULL_VAL,
    BU.BUSINESS_UNIT,
    
    -- Employment Context
    POS.HIRE_DATE,
    TO_CHAR(TRUNC(SYSDATE), 'YYYY') CALENDAR_YEAR,
    
    -- Leave Balance
    LB.ACCRUAL_BAL,
    
    -- Reason and Status
    AR.REASON_NAME ABS_REASON,
    AE.SUBMITTED_DATE,
    AE.CONFIRMED_DATE,
    APPR.APPROVAL_STATUS,
    ABS.ABSENCE_STATUS
    
FROM
    EMP_BASE EB,
    ABSENCE_ENTRIES AE,
    ORG_UNITS OU,
    BUSINESS_UNITS BU,
    PERIOD_OF_SERVICE POS,
    LEAVE_BALANCES LB,
    ABSENCE_REASONS AR,
    APPROVAL_STATUS APPR,
    ABSENCE_STATUS ABS
WHERE
    EB.PERSON_ID = AE.PERSON_ID
AND EB.ORGANIZATION_ID = OU.ORGANIZATION_ID(+)
AND EB.BUSINESS_UNIT_ID = BU.BUSINESS_UNIT_ID(+)
AND EB.PERSON_ID = POS.PERSON_ID
AND EB.PERSON_ID = LB.PERSON_ID(+)
AND EB.PERIOD_OF_SERVICE_ID = LB.PRD_OF_SVC_ID(+)
AND AE.ABSENCE_TYPE_REASON_ID = AR.ABSENCE_TYPE_REASON_ID(+)
AND AE.APPROVAL_STATUS_CD = APPR.LOOKUP_CODE(+)
AND AE.ABSENCE_STATUS_CD = ABS.LOOKUP_CODE(+)

ORDER BY EB.PERSON_NUMBER, AE.ABSENCE_TYPE, AE.START_DATE
```

### Expected Output Columns
1. Person Number
2. Assignment Number
3. Full Name
4. Absence Type
5. Absence Start
6. Absence End
7. Duration
8. Department
9. Business Unit
10. Hire Date
11. Calendar Year
12. Accrual Balance
13. Absence Reason
14. Submitted Date
15. Confirmed Date
16. Approval Status
17. Absence Status

### Key Constraints Applied
✓ Current/Future absences only (SYSDATE BETWEEN START_DATE AND END_DATE)  
✓ Approved/Submitted status (excludes denied/withdrawn)  
✓ Current year accrual balance  
✓ Status lookups for user-friendly display  

---

## TEMPLATE 3: Leave Record Form (Single Transaction)

### Purpose
Auto-generated form data for single leave transaction.

### Business Use Case
HR needs to generate a leave record form with all details pre-populated from Oracle for employee/manager signatures.

### Template Structure

```sql
/******************************************************************************
 * Report: Leave Record Form - System Auto-Generated
 * Module: HCM Absence Management
 * Purpose: Auto-populate leave record form for single transaction
 * 
 * Parameters:
 *   :P_EMP_NO          - Employee number (mandatory)
 *   :P_ABSENCE_ENTRY_ID - Absence entry ID (optional - for specific leave)
 *
 * Author: [Your Name]
 * Date: [SYSDATE]
 ******************************************************************************/

WITH PERIOD AS (
    /*+ qb_name(PERIOD) */
    SELECT
        :P_EMP_NO AS EMPLOYEE_NUMBER,
        :P_ABSENCE_ENTRY_ID AS ABSENCE_ENTRY_ID
    FROM DUAL
)
,EMPLOYEE_DETAILS AS (
    /*+ qb_name(EMPLOYEE_DETAILS) */
    SELECT
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER EMPLOYEE_NUMBER,
        PPNF.DISPLAY_NAME EMPLOYEE_NAME,
        PAAF.ASSIGNMENT_ID,
        PJFV.NAME JOB_POSITION,
        PD.NAME DEPARTMENT,
        PAPF.EMAIL_ADDRESS EMPLOYEE_LOGIN_ID
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_JOBS_F_VL PJFV,
        PER_DEPARTMENTS PD,
        PERIOD P
    WHERE
        PAPF.PERSON_ID = PPNF.PERSON_ID
    AND PAPF.PERSON_ID = PAAF.PERSON_ID
    AND PAAF.JOB_ID = PJFV.JOB_ID(+)
    AND PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
    AND PAPF.PERSON_NUMBER = P.EMPLOYEE_NUMBER
    AND PPNF.NAME_TYPE = 'GLOBAL'
    AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
    AND PAAF.PRIMARY_FLAG = 'Y'
    AND PAAF.ASSIGNMENT_TYPE = 'E'
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE) AND TRUNC(PPNF.EFFECTIVE_END_DATE)
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
)
,MANAGER_DETAILS AS (
    /*+ qb_name(MANAGER_DETAILS) */
    SELECT
        PASF.PERSON_ID,
        PPN.DISPLAY_NAME MANAGER_NAME
    FROM
        PER_ASSIGNMENT_SUPERVISORS_F PASF,
        PER_PERSON_NAMES_F PPN
    WHERE
        PASF.MANAGER_ID = PPN.PERSON_ID
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PASF.EFFECTIVE_START_DATE) AND TRUNC(PASF.EFFECTIVE_END_DATE)
    AND TRUNC(SYSDATE) BETWEEN TRUNC(PPN.EFFECTIVE_START_DATE) AND TRUNC(PPN.EFFECTIVE_END_DATE)
    AND PPN.NAME_TYPE = 'GLOBAL'
)
,LEAVE_INFORMATION AS (
    /*+ qb_name(LEAVE_INFORMATION) */
    SELECT
        APAE.PERSON_ID,
        APAE.PER_ABSENCE_ENTRY_ID,
        AATFT.NAME LEAVE_TYPE,
        TO_CHAR(APAE.START_DATE, 'DD-MON-YYYY') START_DATE,
        TO_CHAR(APAE.END_DATE, 'DD-MON-YYYY') END_DATE,
        APAE.DURATION NUMBER_OF_DAYS,
        APAE.PER_ABSENCE_ENTRY_ID TIMESHEET_REF_NO,
        APAE.PLAN_ID
    FROM
        ANC_PER_ABS_ENTRIES APAE,
        ANC_ABSENCE_TYPES_F_TL AATFT,
        PERIOD P
    WHERE
        AATFT.LANGUAGE = 'US'
    AND AATFT.ABSENCE_TYPE_ID = APAE.ABSENCE_TYPE_ID
    AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
    AND APAE.APPROVAL_STATUS_CD NOT IN ('DENIED')
    AND (APAE.PER_ABSENCE_ENTRY_ID = P.ABSENCE_ENTRY_ID 
         OR P.ABSENCE_ENTRY_ID IS NULL)
)
,LEAVE_BALANCE AS (
    /*+ qb_name(LEAVE_BALANCE) */
    SELECT
        APAE.PERSON_ID,
        APAE.PLAN_ID,
        APAE.END_BAL CURRENT_BALANCE
    FROM
        ANC_PER_ACCRUAL_ENTRIES APAE
    WHERE
        APAE.ACCRUAL_PERIOD = (
            SELECT MAX(APA.ACCRUAL_PERIOD)
            FROM ANC_PER_ACCRUAL_ENTRIES APA
            WHERE APAE.PERSON_ID = APA.PERSON_ID
            AND APA.PLAN_ID = APAE.PLAN_ID
            AND TO_CHAR(APA.ACCRUAL_PERIOD, 'YYYY') <= TO_CHAR(SYSDATE, 'YYYY')
        )
)
-- FINAL SELECT: Form-Formatted Output
SELECT
    -- SECTION 1: Employee Details
    ED.EMPLOYEE_NAME AS "Employee Name",
    ED.EMPLOYEE_NUMBER AS "Employee ID",
    ED.DEPARTMENT AS "Department",
    ED.JOB_POSITION AS "Position",
    MD.MANAGER_NAME AS "Line Manager",
    TO_CHAR(SYSDATE, 'DD-MON-YYYY') AS "Record Generated Date",
    
    -- SECTION 2: Leave Information (Captured from Oracle Timesheet)
    LI.LEAVE_TYPE AS "Leave Type",
    LI.START_DATE AS "Start Date",
    LI.END_DATE AS "End Date",
    LI.NUMBER_OF_DAYS AS "Number of Days",
    LI.TIMESHEET_REF_NO AS "Timesheet Entry Reference No.",
    
    -- SECTION 3: Leave Balance Calculation
    NVL(ROUND(LB.CURRENT_BALANCE, 2), 0) AS "Leave Balance (Current)",
    NVL(ROUND(LB.CURRENT_BALANCE, 2), 0) - NVL(LI.NUMBER_OF_DAYS, 0) AS "Leave Balance (After Applying)",
    
    -- SECTION 4: Employee Confirmation
    ED.EMPLOYEE_LOGIN_ID AS "Employee Wet Signature (Login ID)",
    TO_CHAR(SYSDATE, 'DD-MON-YYYY') AS "Employee Signature Date",
    
    -- SECTION 5: Manager Approval (To be filled)
    MD.MANAGER_NAME AS "Manager Name",
    '___________________________' AS "Manager Signature",
    '___________________________' AS "Manager Approval Date",
    
    -- SECTION 6: Remarks
    '___________________________' AS "HR Remarks"

FROM
    EMPLOYEE_DETAILS ED,
    MANAGER_DETAILS MD,
    LEAVE_INFORMATION LI,
    LEAVE_BALANCE LB
WHERE
    ED.PERSON_ID = MD.PERSON_ID(+)
AND ED.PERSON_ID = LI.PERSON_ID
AND LI.PERSON_ID = LB.PERSON_ID(+)
AND LI.PLAN_ID = LB.PLAN_ID(+)
```

### Expected Output (Form Format)
**Employee Details:**
- Employee Name, Employee ID, Department, Position
- Line Manager, Record Generated Date (SYSDATE)

**Leave Information:**
- Leave Type, Start Date, End Date, Number of Days
- Timesheet Entry Reference No.
- Leave Balance (current and after applying)

**Employee Confirmation:**
- Employee Login ID (for wet signature)
- Date (SYSDATE)

**Manager Approval:**
- Manager Name (pre-filled)
- Signature (blank for physical signature)
- Date (blank for approval date)

**HR Remarks:**
- Blank field for HR comments

### Key Constraints Applied
✓ Single employee filter  
✓ Optional absence entry ID filter  
✓ Current balance calculation  
✓ Projected balance after applying  
✓ SYSDATE for system-generated fields  

---

## TEMPLATE 4: Leave Details Report (Comprehensive with Workflow)

### Purpose
Comprehensive leave detail report with workflow approval, supervisor, and balance information.

### Business Use Case
HR needs to track annual leave details with approver information, supervisor context, organizational hierarchy, and current leave balances for audit and reporting purposes.

### Template Structure

```sql
/******************************************************************************
 * Report: Leave Details Balance - Comprehensive Annual Leave Report
 * Module: HCM Absence Management
 * Purpose: Display leave details with workflow, supervisor, and balance
 * 
 * Parameters:
 *   :P_START_DATE - Start date (optional, default: MIN leave start date)
 *   :P_END_DATE   - End date (optional, default: LAST_DAY(SYSDATE))
 *   :P_EMP_NO     - Employee number (optional, use 'ALL' for all employees)
 *   :P_EMP_NAME   - Employee name (optional, use 'ALL' for all employees)
 *
 * Author: [Your Name]
 * Date: [SYSDATE]
 ******************************************************************************/

WITH DEPARTMENTS AS (
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
,SUPERVISOR AS (
    /*+ qb_name(SUPERVISOR) */
    SELECT
        PASF.PERSON_ID,
        PPN.DISPLAY_NAME SUPERVISOR_NAME,
        PJF.NAME JOB
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
,LEAVE_DETAILS AS (
    /*+ qb_name(LEAVE_DETAILS) */
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
        REA.NAME ABSENCE_REASON
    FROM
        ANC_ABSENCE_TYPES_F_TL AATFT,
        ANC_PER_ABS_ENTRIES APAPAE,
        (SELECT WF.ASSIGNEESDISPLAYNAME APPR_NAME,
                WF.FROMUSERDISPLAYNAME REPL_NAME,
                WF.IDENTIFICATIONKEY
         FROM FA_FUSION_SOAINFRA.WFTASK WF
         WHERE WF.OUTCOME IN ('APPROVE')
         AND WF.ASSIGNEES IS NOT NULL
         AND WF.WORKFLOWPATTERN NOT IN ('AGGREGATION', 'FYI')) WF,
        (SELECT AARF.NAME,
                AARF.ABSENCE_REASON_ID,
                AAT.ABSENCE_TYPE_REASON_ID
         FROM ANC_ABSENCE_REASONS_F AAR,
              ANC_ABSENCE_REASONS_F_TL AARF,
              ANC_ABSENCE_TYPE_REASONS_F AAT
         WHERE AAR.ABSENCE_REASON_ID = AARF.ABSENCE_REASON_ID
         AND AAR.ABSENCE_REASON_ID = AAT.ABSENCE_REASON_ID
         AND AARF.LANGUAGE = 'US'
         AND SYSDATE BETWEEN AAR.EFFECTIVE_START_DATE AND AAR.EFFECTIVE_END_DATE
         AND SYSDATE BETWEEN AARF.EFFECTIVE_START_DATE AND AARF.EFFECTIVE_END_DATE
         AND SYSDATE BETWEEN AAT.EFFECTIVE_START_DATE AND AAT.EFFECTIVE_END_DATE) REA
    WHERE
        AATFT.LANGUAGE = 'US'
    AND AATFT.NAME IN ('Annual Leave')
    AND AATFT.ABSENCE_TYPE_ID = APAPAE.ABSENCE_TYPE_ID(+)
    AND TO_CHAR(APAPAE.PER_ABSENCE_ENTRY_ID) = WF.IDENTIFICATIONKEY(+)
    AND APAPAE.ABSENCE_STATUS_CD <> ('ORA_WITHDRAWN')
    AND TRUNC(APAPAE.START_DATE) BETWEEN 
        TRUNC(CAST(NVL(:P_START_DATE, (SELECT MIN(START_DATE) FROM ANC_PER_ABS_ENTRIES)) AS DATE))
        AND TRUNC(CAST(NVL(:P_END_DATE, LAST_DAY(SYSDATE)) AS DATE))
    AND APAPAE.ABSENCE_TYPE_REASON_ID = REA.ABSENCE_TYPE_REASON_ID(+)
    AND APAPAE.APPROVAL_STATUS_CD <> ('DENIED')
)
-- FINAL SELECT
SELECT
    -- Employee Identification
    PAPF.PERSON_NUMBER EMPLOYEE_NUMBER,
    PPNF.DISPLAY_NAME EMPLOYEE_NAME,
    PPTTL.USER_PERSON_TYPE EMPLOYEE_TYPE,
    
    -- Job and Organization
    PJFV.NAME AS JOB,
    PD.DEPARTMENT PARENT_FUNCTION_NAME,
    PD.DEPARTMENT DIRECTORATE,
    PDS.NAME ORG_NAME,
    
    -- Supervisor
    SP.SUPERVISOR_NAME,
    
    -- Leave Details
    LEAVE.ACTUAL_LEAVE_START_DATE,
    LEAVE.ACTUAL_LEAVE_END_DATE,
    NVL(LEAVE.DURATION, 0) DURATION,
    NVL(LEAVE.L_TYPE, 'Annual Leave') LEAVE_TYPE,
    LEAVE.ABSENCE_REASON,
    LEAVE.DATE_OF_LEAVE_APPLY,
    
    -- Workflow Approval
    LEAVE.APPR_NAME APPROVER_NAME,
    LEAVE.REPL_NAME REQUESTOR_NAME,
    
    -- Leave Balance
    NVL(ROUND(AL.ANU_BAL, 2), 0) ANNUAL_LEAVE_BAL
    
FROM
    PER_ALL_PEOPLE_F PAPF,
    PER_PERSON_NAMES_F PPNF,
    PER_ALL_ASSIGNMENTS_F PAAF,
    PER_PERSON_TYPES_TL PPTTL,
    PER_JOBS_F_VL PJFV,
    PER_DEPARTMENTS PDS,
    HR_ALL_ORGANIZATION_UNITS HAO,
    DEPARTMENTS PD,
    ANNUAL_LEAVE_BALANCE AL,
    SUPERVISOR SP,
    LEAVE_DETAILS LEAVE
WHERE
    PAPF.PERSON_ID = PPNF.PERSON_ID
AND PAPF.PERSON_ID = PAAF.PERSON_ID
AND PAAF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID(+)
AND PAAF.JOB_ID = PJFV.JOB_ID(+)
AND PAAF.ORGANIZATION_ID = PDS.ORGANIZATION_ID(+)
AND PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
AND PAAF.ORGANIZATION_ID = HAO.ORGANIZATION_ID(+)
AND PAPF.PERSON_ID = LEAVE.PERSON_ID(+)
AND PAPF.PERSON_ID = AL.PERSON_ID(+)
AND PAAF.PERSON_ID = SP.PERSON_ID(+)
AND PPNF.NAME_TYPE = 'GLOBAL'
AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
AND PAAF.PRIMARY_FLAG = 'Y'
AND PAAF.ASSIGNMENT_TYPE = 'E'
AND PPTTL.LANGUAGE = 'US'
AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE) AND TRUNC(PPNF.EFFECTIVE_END_DATE)
AND TRUNC(SYSDATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
AND (PAPF.PERSON_NUMBER IN (:P_EMP_NO) OR 'ALL' IN (:P_EMP_NO || 'ALL'))
AND (PPNF.DISPLAY_NAME IN (:P_EMP_NAME) OR 'ALL' IN (:P_EMP_NAME || 'ALL'))

ORDER BY TO_NUMBER(PAPF.PERSON_NUMBER)
```

### Expected Output Columns
1. Employee Number
2. Employee Name
3. Employee Type
4. Job
5. Parent Function Name
6. Directorate
7. Org Name
8. Supervisor Name
9. Actual Leave Start Date
10. Actual Leave End Date
11. Duration
12. Leave Type
13. Absence Reason
14. Date of Leave Apply
15. Approver Name
16. Requestor Name
17. Annual Leave Balance

### Key Features Applied
✓ Workflow approval integration (FA_FUSION_SOAINFRA.WFTASK)  
✓ Absence reason lookup (3-table join)  
✓ Department hierarchy with classification  
✓ Supervisor hierarchy  
✓ Annual leave balance with UPPER matching  
✓ Date range parameters with dynamic defaults  
✓ INITCAP date formatting  
✓ Oracle Traditional Join Syntax  
✓ CTE performance hints  

### Integration Points
- **Workflow System:** Approver and requestor names from WFTASK
- **Department Hierarchy:** Classification-based org structure
- **Supervisor Hierarchy:** Manager name and job
- **Leave Balance:** Current year accrual balance

---

## TEMPLATE 5: Leave Summary Report (Aggregated)

### Purpose
Aggregated leave statistics by employee and leave type.

### Template Structure

```sql
/******************************************************************************
 * Report: Leave Summary - Aggregated Statistics
 * Module: HCM Absence Management
 * Purpose: Summarize leave usage by employee and type
 * 
 * Parameters:
 *   :P_START_DATE - Start date (default: start of year)
 *   :P_END_DATE   - End date (default: current date)
 *
 * Author: [Your Name]
 * Date: [SYSDATE]
 ******************************************************************************/

WITH PERIOD AS (
    /*+ qb_name(PERIOD) */
    SELECT
        TRUNC(CAST(NVL(:P_START_DATE, TRUNC(SYSDATE, 'YEAR')) AS DATE)) AS START_DATE,
        TRUNC(CAST(NVL(:P_END_DATE, SYSDATE) AS DATE)) AS END_DATE
    FROM DUAL
)
-- Copy EMP_MASTER and LEAVE_TRANSACTIONS from repositories
-- Add aggregation CTE
,LEAVE_SUMMARY AS (
    /*+ qb_name(LEAVE_SUMMARY) */
    SELECT
        LT.PERSON_ID,
        LT.LEAVE_TYPE,
        COUNT(*) LEAVE_COUNT,
        SUM(LT.DURATION) TOTAL_DAYS,
        MIN(LT.START_DATE) FIRST_LEAVE_DATE,
        MAX(LT.END_DATE) LAST_LEAVE_DATE
    FROM
        LEAVE_TRANSACTIONS LT
    GROUP BY LT.PERSON_ID, LT.LEAVE_TYPE
)
SELECT
    EM.PERSON_NUMBER,
    EM.DISPLAY_NAME,
    LS.LEAVE_TYPE,
    LS.LEAVE_COUNT AS "Number of Leaves Taken",
    NVL(LS.TOTAL_DAYS, 0) AS "Total Days Taken",
    TO_CHAR(LS.FIRST_LEAVE_DATE, 'DD-MON-YYYY') AS "First Leave Date",
    TO_CHAR(LS.LAST_LEAVE_DATE, 'DD-MON-YYYY') AS "Last Leave Date"
FROM
    EMP_MASTER EM,
    LEAVE_SUMMARY LS
WHERE
    EM.PERSON_ID = LS.PERSON_ID
ORDER BY EM.PERSON_NUMBER, LS.LEAVE_TYPE
```

---

## 📝 Template Usage Instructions

### Step 1: Select Appropriate Template
Choose the template that matches your report requirement:
- **T1** for leave history with comparison
- **T2** for current absence activity
- **T3** for single leave form
- **T4** for aggregated statistics

### Step 2: Copy Template Skeleton
Copy the entire SQL structure including:
- Header comment block
- CTE definitions
- Final SELECT

### Step 3: Verify CTE Dependencies
Cross-check that all required CTEs are copied from ABSENCE_REPOSITORIES.md:
- Check the "Key CTEs Required" column in Template Index
- Copy complete CTEs (do NOT write fresh joins)

### Step 4: Apply Constraints
Verify all constraints from ABSENCE_MASTER.md are applied:
- [ ] Date-track filtering on `_F` tables
- [ ] `LANGUAGE = 'US'` on `_TL` tables
- [ ] Approval status filtering
- [ ] Absence status filtering
- [ ] Oracle Traditional Join Syntax
- [ ] CTE qb_name hints

### Step 5: Customize (If Needed)
- Add/remove columns as per requirement
- Adjust date formatting
- Modify parameter logic

### Step 6: Test & Validate
- Run with sample parameters
- Verify output matches expected format
- Check performance (execution time)
- Validate data accuracy

---

## ✅ Validation Checklist (Before Finalizing)

- [ ] Header comment block complete with parameters documented
- [ ] All CTEs copied from ABSENCE_REPOSITORIES.md (no fresh joins)
- [ ] All CTEs have `/*+ qb_name(NAME) */` hints
- [ ] Oracle Traditional Join Syntax used throughout
- [ ] Date-track filters applied to all `_F` tables
- [ ] `LANGUAGE = 'US'` applied to all `_TL` tables
- [ ] Approval/Absence status filters applied
- [ ] Parameters handle NULL with NVL/defaults
- [ ] Final SELECT organized: IDs → Dates → Amounts → Status → Desc
- [ ] Output column names match requirement specification

---

## 📊 Expected Performance

| Template | Complexity | Expected Runtime | Records Returned |
|----------|------------|------------------|------------------|
| T1: Leave History | Medium | < 10 seconds | 100-5,000 |
| T2: Leave Balance | Medium | < 15 seconds | 50-500 |
| T3: Leave Record Form | Simple | < 3 seconds | 1 |
| T4: Leave Details | Complex | < 20 seconds | 100-1,000 |
| T5: Leave Summary | Medium | < 10 seconds | 100-1,000 |

*Performance estimates based on typical HCM environments with 1,000-10,000 employees.*

**Note:** T4 (Leave Details) includes workflow and payroll integration which may require additional time for cross-schema access.

---

**END OF ABSENCE_TEMPLATES.md**


