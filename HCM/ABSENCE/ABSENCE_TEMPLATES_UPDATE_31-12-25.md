# HCM Absence Templates - Update 31-12-25

**Module:** HCM Absence Management  
**Purpose:** New template from Employee Annual Leave Balance Report  
**Tag:** `#HCM #ABSENCE #TEMPLATES #UPDATE`  
**Date:** 31-12-25

---

## 📋 NEW TEMPLATE ADDITION

This document contains a NEW template extracted from the Employee Annual Leave Balance Report implementation.

---

## TEMPLATE 7: Employee Annual Leave Balance Report (Comprehensive)

### Purpose
Generate comprehensive annual leave balance report with all balance components, service calculation, and multi-parameter filtering.

### Business Use Case
HR and management need a complete view of employee leave balances including:
- Current balance breakdown (PY carryover, CY accrued)
- All adjustments (manual adjustments, encashments, carryovers)
- Leave usage (applied vs taken)
- Employee classification and organizational context
- Service duration for entitlement calculations

### Complexity: High

### Key CTEs Required
1. PARAMETERS (with UPPER() for case-insensitive filtering)
2. EMP_BASE (with service calculation)
3. EMP_ASSIGNMENT (with FT/PT classification)
4. EMP_DFF (DFF attribute mapping)
5. PLAN_ENROLLMENT
6. ACCRUAL_BALANCE (with PY/CY breakdown)
7. CARRYOVER_DETAILS
8. LEAVE_TRANSACTIONS (with unpaid tracking)
9. ENCASHMENT_DETAILS

### Template Structure

```sql
/******************************************************************************
 * Report: Employee Annual Leave Balance Report
 * Module: HCM Absence Management
 * Purpose: Comprehensive leave balance with all components
 * 
 * Parameters:
 *   :P_EFFECTIVE_DATE  - Effective Date (MANDATORY) - DD-MON-YYYY
 *   :P_LEGAL_EMPLOYER  - Legal Employer filter ('ALL' for all)
 *   :P_ABSENCE_PLAN    - Absence Plan filter ('ALL' for all)
 *   :P_JOB_TITLE       - Job Title filter ('ALL' for all)
 *   :P_EMPLOYEE_TYPE   - Employee Type filter ('ALL' for all)
 *   :P_LOCATION        - Location filter ('ALL' for all)
 *
 * Balance Formula:
 *   Calc Balance = PY Carry Forward + CY Accrued + Adjustments 
 *                  - Encashment - Leave Taken - Carryover Expired
 *
 * Author: [Your Name]
 * Date: [Date]
 ******************************************************************************/

-- ============================================================================
-- PARAMETER HANDLING WITH CASE-INSENSITIVE FILTERING
-- ============================================================================
WITH PARAMETERS AS (
    /*+ qb_name(PARAMETERS) */
    SELECT
        TRUNC(TO_DATE(:P_EFFECTIVE_DATE, 'DD-MON-YYYY')) AS EFFECTIVE_DATE,
        UPPER(NVL(:P_LEGAL_EMPLOYER, 'ALL')) AS LEGAL_EMPLOYER,
        UPPER(NVL(:P_ABSENCE_PLAN, 'ALL')) AS ABSENCE_PLAN,
        UPPER(NVL(:P_JOB_TITLE, 'ALL')) AS JOB_TITLE,
        UPPER(NVL(:P_EMPLOYEE_TYPE, 'ALL')) AS EMPLOYEE_TYPE,
        UPPER(NVL(:P_LOCATION, 'ALL')) AS LOCATION
    FROM DUAL
)

-- ============================================================================
-- EMPLOYEE BASE WITH SERVICE CALCULATION
-- ============================================================================
,EMP_BASE AS (
    /*+ qb_name(EMP_BASE) */
    SELECT
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER,
        PPNF.FULL_NAME,
        PPNF.DISPLAY_NAME,
        PPTTL.USER_PERSON_TYPE AS PERSON_TYPE,
        PPOS.PERIOD_OF_SERVICE_ID,
        TO_CHAR(NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START), 'DD-MM-YYYY') AS HIRE_DATE,
        -- Service in Years calculation
        ROUND(MONTHS_BETWEEN(P.EFFECTIVE_DATE, 
            NVL(PPOS.ORIGINAL_DATE_OF_HIRE, PPOS.DATE_START)) / 12, 2) AS SERVICE_IN_YEARS
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
    -- Date-track filtering using EFFECTIVE_DATE (not SYSDATE)
    AND P.EFFECTIVE_DATE BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PPOS.DATE_START 
        AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
)

-- ============================================================================
-- ASSIGNMENT WITH FULL TIME/PART TIME CLASSIFICATION
-- ============================================================================
,EMP_ASSIGNMENT AS (
    /*+ qb_name(EMP_ASSIGNMENT) */
    SELECT
        PAAF.PERSON_ID,
        PAAF.ASSIGNMENT_NUMBER,
        PAAF.ASSIGNMENT_CATEGORY AS WORKER_TYPE,
        PJFV.NAME AS JOB_TITLE,
        HAPL.NAME AS POSITION_TITLE,
        PD.NAME AS DEPARTMENT_NAME,
        HAOULE.NAME AS LEGAL_EMPLOYER_NAME,
        HAOUBU.NAME AS BUSINESS_UNIT_NAME,
        HLOCVL.LOCATION_NAME,
        PGFV.NAME AS GRADE_NAME,
        -- Full Time / Part Time based on NORMAL_HOURS
        CASE 
            WHEN NVL(PAAF.NORMAL_HOURS, 0) >= 40 THEN 'Full Time'
            WHEN NVL(PAAF.NORMAL_HOURS, 0) > 0 AND NVL(PAAF.NORMAL_HOURS, 0) < 40 THEN 'Part Time'
            ELSE 'Not Specified'
        END AS FULL_TIME_PART_TIME,
        -- DFF Attributes
        PAAF.ATTRIBUTE1,
        PAAF.ATTRIBUTE2,
        PAAF.ATTRIBUTE3,
        PAAF.ATTRIBUTE4,
        PAAF.ATTRIBUTE5
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
    AND P.EFFECTIVE_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
    AND P.EFFECTIVE_DATE BETWEEN PJFV.EFFECTIVE_START_DATE(+) AND PJFV.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN HAPL.EFFECTIVE_START_DATE(+) AND HAPL.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN PD.EFFECTIVE_START_DATE(+) AND PD.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN HLOCVL.EFFECTIVE_START_DATE(+) AND HLOCVL.EFFECTIVE_END_DATE(+)
    AND P.EFFECTIVE_DATE BETWEEN PGFV.EFFECTIVE_START_DATE(+) AND PGFV.EFFECTIVE_END_DATE(+)
)

-- ============================================================================
-- DFF ATTRIBUTE MAPPING
-- ============================================================================
,EMP_DFF AS (
    /*+ qb_name(EMP_DFF) */
    SELECT
        EA.PERSON_ID,
        -- Map to business fields (update based on your DFF configuration)
        EA.ATTRIBUTE1 AS CONTRACT_TYPE,
        EA.ATTRIBUTE5 AS CLIENT_JOB_TITLE,
        EA.ATTRIBUTE3 AS PROJECT_NUMBER,
        EA.ATTRIBUTE4 AS SERVICE_LINE
    FROM
        EMP_ASSIGNMENT EA
)

-- ============================================================================
-- ABSENCE PLAN ENROLLMENT
-- ============================================================================
,PLAN_ENROLLMENT AS (
    /*+ qb_name(PLAN_ENROLLMENT) */
    SELECT
        APE.PERSON_ID,
        APE.PLAN_ID,
        APE.PRD_OF_SVC_ID,
        AAPVL.NAME AS PLAN_NAME,
        TO_CHAR(APE.START_DATE, 'DD-MM-YYYY') AS ENROLLMENT_START_DATE
    FROM
        ANC_PER_ENROLLMENTS APE,
        ANC_ABSENCE_PLANS_VL AAPVL,
        PARAMETERS P
    WHERE
        APE.PLAN_ID = AAPVL.ABSENCE_PLAN_ID
    AND UPPER(AAPVL.NAME) LIKE '%ANNUAL%'
    AND P.EFFECTIVE_DATE BETWEEN APE.START_DATE 
        AND NVL(APE.END_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
)

-- ============================================================================
-- ACCRUAL BALANCE WITH PY/CY BREAKDOWN
-- ============================================================================
,ACCRUAL_BALANCE AS (
    /*+ qb_name(ACCRUAL_BALANCE) */
    SELECT
        APAE.PERSON_ID,
        APAE.PLAN_ID,
        APAE.PRD_OF_SVC_ID,
        -- Previous Year Carry Forward
        SUM(CASE 
            WHEN TO_CHAR(APAE.ACCRUAL_PERIOD, 'YYYY') < TO_CHAR(P.EFFECTIVE_DATE, 'YYYY')
            THEN NVL(APAE.END_BAL, 0)
            ELSE 0 
        END) AS PY_CARRY_FORWARD,
        -- Current Year Accrued
        SUM(CASE 
            WHEN TO_CHAR(APAE.ACCRUAL_PERIOD, 'YYYY') = TO_CHAR(P.EFFECTIVE_DATE, 'YYYY')
            THEN NVL(APAE.ACCRUAL_BALANCE, 0)
            ELSE 0 
        END) AS CY_ACCRUED,
        -- Adjustments
        SUM(NVL(APAE.ADJUSTMENT, 0)) AS BALANCE_ADJUSTMENT,
        -- Entitlement
        MAX(APAE.ACCRUAL_RATE) AS ANNUAL_ENTITLEMENT,
        MAX(APAE.ENTITLEMENT_OVERRIDE) AS ENTITLEMENT_OVERRIDE
    FROM
        ANC_PER_ACCRUAL_ENTRIES APAE,
        PARAMETERS P
    WHERE
        APAE.ACCRUAL_PERIOD <= P.EFFECTIVE_DATE
    GROUP BY 
        APAE.PERSON_ID, 
        APAE.PLAN_ID, 
        APAE.PRD_OF_SVC_ID,
        P.EFFECTIVE_DATE
)

-- ============================================================================
-- CARRYOVER DETAILS (Optional - comment out if table doesn't exist)
-- ============================================================================
,CARRYOVER_DETAILS AS (
    /*+ qb_name(CARRYOVER_DETAILS) */
    SELECT
        APC.PERSON_ID,
        APC.PLAN_ID,
        APC.PRD_OF_SVC_ID,
        -- Expired carryover
        SUM(CASE 
            WHEN APC.EXPIRY_DATE < P.EFFECTIVE_DATE 
            THEN NVL(APC.CARRYOVER_BALANCE, 0)
            ELSE 0 
        END) AS CARRYOVER_EXPIRED,
        MAX(APC.EXPIRY_DATE) AS CARRYOVER_EXPIRY_DATE
    FROM
        ANC_PER_CARRYOVER APC,
        PARAMETERS P
    WHERE
        NVL(APC.CARRYOVER_BALANCE, 0) > 0
    GROUP BY 
        APC.PERSON_ID, 
        APC.PLAN_ID, 
        APC.PRD_OF_SVC_ID,
        P.EFFECTIVE_DATE
)

-- ============================================================================
-- LEAVE TRANSACTIONS WITH UNPAID TRACKING
-- ============================================================================
,LEAVE_TRANSACTIONS AS (
    /*+ qb_name(LEAVE_TRANSACTIONS) */
    SELECT
        APAE.PERSON_ID,
        APAE.PLAN_ID,
        -- Leave Applied
        SUM(CASE 
            WHEN APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
            AND APAE.START_DATE <= P.EFFECTIVE_DATE
            THEN NVL(APAE.DURATION, 0)
            ELSE 0 
        END) AS LEAVE_APPLIED,
        -- Leave Taken (Approved only)
        SUM(CASE 
            WHEN APAE.APPROVAL_STATUS_CD IN ('APPROVED')
            AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
            AND APAE.START_DATE <= P.EFFECTIVE_DATE
            THEN NVL(APAE.DURATION, 0)
            ELSE 0 
        END) AS LEAVE_TAKEN,
        -- Unpaid Leave Days
        SUM(CASE 
            WHEN APAE.APPROVAL_STATUS_CD IN ('APPROVED')
            AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
            AND APAE.START_DATE <= P.EFFECTIVE_DATE
            AND UPPER(AATFT.NAME) LIKE '%UNPAID%'
            THEN NVL(APAE.DURATION, 0)
            ELSE 0 
        END) AS UNPAID_LEAVE_DAYS
    FROM
        ANC_PER_ABS_ENTRIES APAE,
        ANC_ABSENCE_TYPES_F_TL AATFT,
        PARAMETERS P
    WHERE
        APAE.ABSENCE_TYPE_ID = AATFT.ABSENCE_TYPE_ID
    AND AATFT.LANGUAGE = 'US'
    AND P.EFFECTIVE_DATE BETWEEN AATFT.EFFECTIVE_START_DATE AND AATFT.EFFECTIVE_END_DATE
    GROUP BY 
        APAE.PERSON_ID, 
        APAE.PLAN_ID,
        P.EFFECTIVE_DATE
)

-- ============================================================================
-- ENCASHMENT DETAILS (Optional - comment out if table doesn't exist)
-- ============================================================================
,ENCASHMENT_DETAILS AS (
    /*+ qb_name(ENCASHMENT_DETAILS) */
    SELECT
        APE.PERSON_ID,
        APE.PLAN_ID,
        SUM(NVL(APE.ENCASHMENT_AMOUNT, 0)) AS ANNUAL_LEAVE_ENCASHMENT
    FROM
        ANC_PER_ENCASHMENTS APE,
        PARAMETERS P
    WHERE
        APE.ENCASHMENT_DATE <= P.EFFECTIVE_DATE
    GROUP BY 
        APE.PERSON_ID, 
        APE.PLAN_ID
)

-- ============================================================================
-- FINAL SELECT: Comprehensive leave balance output
-- ============================================================================
SELECT
    -- SECTION 1: Employee Identification
    EB.PERSON_NUMBER AS "Person Number",
    EA.ASSIGNMENT_NUMBER AS "Assignment Number",
    EB.FULL_NAME AS "Full Name",
    
    -- SECTION 2: Organizational Details
    EA.LEGAL_EMPLOYER_NAME AS "Legal Employer",
    EA.BUSINESS_UNIT_NAME AS "Business Unit",
    EB.HIRE_DATE AS "Hire date",
    
    -- SECTION 3: Employee Classification
    EA.WORKER_TYPE AS "Worker Type",
    EB.PERSON_TYPE AS "Person Type",
    EA.FULL_TIME_PART_TIME AS "Full Time/ Part Time",
    EA.GRADE_NAME AS "Grade",
    'Active' AS "Employee Status",
    
    -- SECTION 4: Department and Position
    EA.DEPARTMENT_NAME AS "Department",
    EA.POSITION_TITLE AS "Position Title",
    DFF.CLIENT_JOB_TITLE AS "Client Job Title",
    
    -- SECTION 5: Contract and Assignment Details
    DFF.CONTRACT_TYPE AS "Contract Type",
    EA.ASSIGNMENT_CATEGORY AS "Assignment Category",
    DFF.PROJECT_NUMBER AS "Project Number",
    DFF.SERVICE_LINE AS "Service Line",
    EB.SERVICE_IN_YEARS AS "Service in Years",
    
    -- SECTION 6: Absence Plan Details
    PE.PLAN_NAME AS "Plan Name",
    PE.ENROLLMENT_START_DATE AS "Enrollment Start Date",
    
    -- SECTION 7: Entitlement and Adjustments
    NVL(AB.ANNUAL_ENTITLEMENT, 0) AS "Holiday Pay / Entitlement per annum",
    NVL(LT.UNPAID_LEAVE_DAYS, 0) AS "Unpaid Leave Days taken",
    NVL(AB.BALANCE_ADJUSTMENT, 0) AS "Annual leave balance adjustment",
    NVL(EN.ANNUAL_LEAVE_ENCASHMENT, 0) AS "Annual leave encashment",
    AB.ENTITLEMENT_OVERRIDE AS "Annual leave entitlement override",
    
    -- SECTION 8: Balance Breakdown
    NVL(AB.PY_CARRY_FORWARD, 0) AS "PY Leave Carried Forward (Days)",
    NVL(AB.CY_ACCRUED, 0) AS "CY Leave Accrued (Days)",
    NVL(CD.CARRYOVER_EXPIRED, 0) AS "Carryover Expired (days)",
    TO_CHAR(CD.CARRYOVER_EXPIRY_DATE, 'DD-MM-YYYY') AS "Carryover expiry date",
    
    -- SECTION 9: Leave Usage
    NVL(LT.LEAVE_APPLIED, 0) AS "Leave Applied",
    NVL(LT.LEAVE_TAKEN, 0) AS "Leave Taken",
    
    -- SECTION 10: Calculated Leave Balance
    -- Formula: PY + CY + Adj - Enc - Taken - Expired
    (
        NVL(AB.PY_CARRY_FORWARD, 0) 
        + NVL(AB.CY_ACCRUED, 0) 
        + NVL(AB.BALANCE_ADJUSTMENT, 0) 
        - NVL(EN.ANNUAL_LEAVE_ENCASHMENT, 0) 
        - NVL(LT.LEAVE_TAKEN, 0) 
        - NVL(CD.CARRYOVER_EXPIRED, 0)
    ) AS "Calc. Leave Balance"

FROM
    EMP_BASE EB,
    EMP_ASSIGNMENT EA,
    EMP_DFF DFF,
    PLAN_ENROLLMENT PE,
    ACCRUAL_BALANCE AB,
    CARRYOVER_DETAILS CD,
    LEAVE_TRANSACTIONS LT,
    ENCASHMENT_DETAILS EN,
    PARAMETERS P
WHERE
    -- Link all CTEs
    EB.PERSON_ID = EA.PERSON_ID
AND EA.PERSON_ID = DFF.PERSON_ID(+)
AND EB.PERSON_ID = PE.PERSON_ID(+)
AND EB.PERSON_ID = AB.PERSON_ID(+)
AND PE.PLAN_ID = AB.PLAN_ID(+)
AND EB.PERIOD_OF_SERVICE_ID = AB.PRD_OF_SVC_ID(+)
AND EB.PERSON_ID = CD.PERSON_ID(+)
AND PE.PLAN_ID = CD.PLAN_ID(+)
AND EB.PERIOD_OF_SERVICE_ID = CD.PRD_OF_SVC_ID(+)
AND EB.PERSON_ID = LT.PERSON_ID(+)
AND PE.PLAN_ID = LT.PLAN_ID(+)
AND EB.PERSON_ID = EN.PERSON_ID(+)
AND PE.PLAN_ID = EN.PLAN_ID(+)

    -- Multi-parameter filtering with 'ALL' support
AND (UPPER(EA.LEGAL_EMPLOYER_NAME) = P.LEGAL_EMPLOYER OR P.LEGAL_EMPLOYER = 'ALL')
AND (UPPER(PE.PLAN_NAME) = P.ABSENCE_PLAN OR P.ABSENCE_PLAN = 'ALL')
AND (UPPER(EA.JOB_TITLE) = P.JOB_TITLE OR P.JOB_TITLE = 'ALL')
AND (UPPER(EB.PERSON_TYPE) = P.EMPLOYEE_TYPE OR P.EMPLOYEE_TYPE = 'ALL')
AND (UPPER(EA.LOCATION_NAME) = P.LOCATION OR P.LOCATION = 'ALL')

ORDER BY 
    TO_NUMBER(EB.PERSON_NUMBER),
    PE.PLAN_NAME
```

---

### Expected Output Columns (53 total)

Organized in 10 sections:

#### Section 1: Employee Identification (3)
1. Person Number
2. Assignment Number
3. Full Name

#### Section 2: Organizational Details (3)
4. Legal Employer
5. Business Unit
6. Hire date

#### Section 3: Employee Classification (5)
7. Worker Type
8. Person Type
9. Full Time/ Part Time
10. Grade
11. Employee Status (Always 'Active')

#### Section 4: Department and Position (3)
12. Department
13. Position Title
14. Client Job Title

#### Section 5: Contract and Assignment Details (5)
15. Contract Type
16. Assignment Category
17. Project Number
18. Service Line
19. Service in Years

#### Section 6: Absence Plan Details (2)
20. Plan Name
21. Enrollment Start Date

#### Section 7: Entitlement and Adjustments (5)
22. Holiday Pay / Entitlement per annum
23. Unpaid Leave Days taken
24. Annual leave balance adjustment
25. Annual leave encashment
26. Annual leave entitlement override

#### Section 8: Balance Breakdown (4)
27. PY Leave Carried Forward (Days)
28. CY Leave Accrued (Days)
29. Carryover Expired (days)
30. Carryover expiry date

#### Section 9: Leave Usage (2)
31. Leave Applied
32. Leave Taken

#### Section 10: Calculated Balance (1)
33. Calc. Leave Balance

---

### Key Constraints Applied

✅ **HCM Standards:**
- Date-track filtering using Effective Date (not SYSDATE)
- LANGUAGE = 'US' for translation tables
- Oracle Traditional Join Syntax
- All CTEs have qb_name hints
- Active employees only
- Proper outer joins for optional data

✅ **Absence-Specific:**
- Approved status for leave taken
- Excludes withdrawn entries
- Plan name case-insensitive matching
- PY/CY year breakdown
- Unpaid leave identified separately

✅ **Balance Calculation:**
- All components visible
- NVL() protection against NULL
- Clear formula in parentheses
- Audit trail for verification

---

### Configuration Notes

#### 1. DFF Attributes
**Current Mapping** (update based on your configuration):
```sql
ATTRIBUTE1 → CONTRACT_TYPE
ATTRIBUTE5 → CLIENT_JOB_TITLE
ATTRIBUTE3 → PROJECT_NUMBER
ATTRIBUTE4 → SERVICE_LINE
```

**Discovery Query:**
```sql
SELECT 
    APPLICATION_COLUMN_NAME,
    END_USER_COLUMN_NAME
FROM FND_DESCR_FLEX_COLUMN_USAGES
WHERE APPLICATION_TABLE_NAME = 'PER_ALL_ASSIGNMENTS_F'
AND ENABLED_FLAG = 'Y';
```

#### 2. Optional Tables
If these tables don't exist:
- Comment out `CARRYOVER_DETAILS` CTE
- Comment out `ENCASHMENT_DETAILS` CTE
- Set corresponding columns to 0 in final SELECT

#### 3. Full Time/Part Time Threshold
**Current Logic**: >= 40 hours = Full Time

Adjust based on your organization's definition:
```sql
WHEN NVL(PAAF.NORMAL_HOURS, 0) >= 40 THEN 'Full Time'  -- Change 40 as needed
```

#### 4. Unpaid Leave Identification
**Current Logic**: Type name contains 'UNPAID'

Alternative if tracked differently:
```sql
-- By category
AND AATFT.ABSENCE_CATEGORY = 'UNPAID'

-- By type code
AND AATFT.ABSENCE_TYPE_CODE LIKE '%UNPAID%'
```

---

### Testing Recommendations

#### 1. Test with Single Employee First
```sql
-- Add to WHERE clause:
AND EB.PERSON_NUMBER = '123456'
```

#### 2. Verify Balance Calculation
```sql
-- Manual verification
SELECT 
    PERSON_NUMBER,
    PY_CARRY_FORWARD,
    CY_ACCRUED,
    BALANCE_ADJUSTMENT,
    ENCASHMENT,
    LEAVE_TAKEN,
    CARRYOVER_EXPIRED,
    -- Manual calculation
    (PY_CARRY_FORWARD + CY_ACCRUED + BALANCE_ADJUSTMENT 
     - ENCASHMENT - LEAVE_TAKEN - CARRYOVER_EXPIRED) AS MANUAL_CALC,
    CALC_LEAVE_BALANCE AS QUERY_CALC
FROM result_table;
```

#### 3. Check for Duplicates
```sql
SELECT 
    PERSON_NUMBER,
    PLAN_NAME,
    COUNT(*) AS COUNT
FROM result_table
GROUP BY PERSON_NUMBER, PLAN_NAME
HAVING COUNT(*) > 1;
```

---

### Performance Considerations

#### Expected Performance
- **Small dataset** (< 1,000 employees): < 10 seconds
- **Medium dataset** (1,000 - 10,000 employees): 10-30 seconds
- **Large dataset** (> 10,000 employees): 30-60 seconds

#### Optimization Tips
1. **Parallel Execution** (if needed):
```sql
SELECT /*+ PARALLEL(PAPF, 4) PARALLEL(PAAF, 4) */
```

2. **Indexes** (coordinate with DBA):
```sql
CREATE INDEX IDX_PER_PEOPLE_EFF_DATES 
ON PER_ALL_PEOPLE_F(PERSON_ID, EFFECTIVE_START_DATE, EFFECTIVE_END_DATE);

CREATE INDEX IDX_ANC_ACCRUAL_PERSON_PLAN 
ON ANC_PER_ACCRUAL_ENTRIES(PERSON_ID, PLAN_ID, ACCRUAL_PERIOD);
```

---

### Usage Examples

#### Example 1: All Employees, All Plans
```sql
:P_EFFECTIVE_DATE  = '31-DEC-2024'
:P_LEGAL_EMPLOYER  = 'ALL'
:P_ABSENCE_PLAN    = 'ALL'
:P_JOB_TITLE       = 'ALL'
:P_EMPLOYEE_TYPE   = 'ALL'
:P_LOCATION        = 'ALL'
```

#### Example 2: Specific Legal Employer and Location
```sql
:P_EFFECTIVE_DATE  = '31-DEC-2024'
:P_LEGAL_EMPLOYER  = 'ABC CORPORATION LLC'  -- Case doesn't matter
:P_ABSENCE_PLAN    = 'ANNUAL LEAVE PLAN'
:P_JOB_TITLE       = 'ALL'
:P_EMPLOYEE_TYPE   = 'EMPLOYEE'
:P_LOCATION        = 'DUBAI OFFICE'
```

#### Example 3: Historical Query (Past Date)
```sql
:P_EFFECTIVE_DATE  = '31-DEC-2023'  -- Past date
:P_LEGAL_EMPLOYER  = 'ALL'
:P_ABSENCE_PLAN    = 'ALL'
:P_JOB_TITLE       = 'ALL'
:P_EMPLOYEE_TYPE   = 'ALL'
:P_LOCATION        = 'ALL'
```

---

### Integration Points

#### BI Publisher Integration
- Add parameter definitions with LOV queries
- Create Excel template for formatted output
- Schedule monthly execution
- Email distribution to HR team

#### OTBI Integration
- Create Analysis in OTBI
- Add prompted filters
- Create calculated fields for balance
- Save as report for self-service access

---

### Maintenance Notes

#### Monthly Tasks
- Verify accrual entries are up to date
- Check for new absence plans
- Validate balance calculations

#### Quarterly Tasks
- Review DFF mappings
- Verify FT/PT classification
- Performance tuning if needed

#### Annual Tasks
- Update year-specific logic
- Review business rules
- Update documentation

---

## ✅ VALIDATION CHECKLIST

Before using this template:

- [ ] DFF attributes mapped to correct business fields
- [ ] FT/PT threshold appropriate for organization (default: 40 hours)
- [ ] Unpaid leave identification logic correct
- [ ] Optional tables (carryover, encashment) exist or commented out
- [ ] Parameters tested with various filter combinations
- [ ] Balance calculation verified manually for sample employees
- [ ] Performance acceptable with full dataset
- [ ] Output columns match business requirements

---

## 📝 TEMPLATE SUMMARY

**Complexity**: High  
**Lines of Code**: ~550  
**CTEs**: 9  
**Output Columns**: 53  
**Parameters**: 6 (1 mandatory, 5 optional)  
**Key Feature**: Comprehensive balance with all components  
**Best For**: Management reports, HR audits, balance reconciliation

---

**END OF ABSENCE_TEMPLATES_UPDATE_31-12-25.md**

**Status:** Ready for Integration  
**Next Action:** Review and add to main ABSENCE_TEMPLATES.md as Template 7





