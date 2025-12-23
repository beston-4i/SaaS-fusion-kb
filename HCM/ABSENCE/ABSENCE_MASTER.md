# HCM Absence Master Instructions: Leave & Absence Management

**Module:** HCM Absence Management  
**Tag:** `#HCM #ABSENCE #LEAVE #TIMEOFF`  
**Status:** Active  
**Date:** 18-12-25

---

## 1. 🚨 Critical Absence Constraints
*Violating these rules breaks the system.*

### 1.1 Date-Track Filtering (The "Golden Rule")
**Rule:** `AND TRUNC(SYSDATE) BETWEEN [TABLE].EFFECTIVE_START_DATE AND [TABLE].EFFECTIVE_END_DATE`  
**Why:** Absence Type definitions and configurations change over time. Without this filter, you get historical versions.  
**Scope:** Applied to all `_F` tables (ANC_ABSENCE_TYPES_F_TL, ANC_ABSENCE_REASONS_F_TL, etc.)

### 1.2 Approval Status Filtering
**Rule:** `AND ABSENCE_ENTRY.APPROVAL_STATUS_CD NOT IN ('DENIED', 'ORA_WITHDRAWN', 'ORA_AWAIT_AWAIT')`  
**Why:** Prevents inclusion of rejected or withdrawn leave requests.  
**Scope:** All queries against `ANC_PER_ABS_ENTRIES`

**Approved Status Values:**
- `'APPROVED'` - Fully approved leaves
- Exclude: `'DENIED'`, `'ORA_WITHDRAWN'`, `'ORA_AWAIT_AWAIT'`, `'PENDING'`

### 1.3 Absence Status Filtering
**Rule:** `AND ABSENCE_ENTRY.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'`  
**Why:** Withdrawn entries should not appear in standard reports.  
**Scope:** All queries against `ANC_PER_ABS_ENTRIES`

**Valid Absence Status Lookups:**
- Lookup Type: `'ANC_PER_ABS_ENT_STATUS'`
- Exclude: `'Withdrawn'`, `'Denied'`, `'Awaiting'` (meanings)

### 1.4 Absence Type Classification
**Rule:** Always join to `ANC_ABSENCE_TYPES_F_TL` with `LANGUAGE = 'US'`  
**Why:** Absence types are translated. Without language filter, you get duplicate rows.  
**Scope:** All absence type lookups

**Common Absence Types:**
- `'Annual Leave'` - Standard annual/vacation leave
- `'Annual Leave Staff'` - Staff-specific annual leave
- `'Sick Leave'` - Medical/illness leave
- `'Vacation Leave'` - Holiday leave

### 1.5 Accrual Balance Filtering
**Rule:** Always get MAX accrual period for current year  
**Pattern:**
```sql
AND APAE.ACCRUAL_PERIOD = (
    SELECT MAX(APA.ACCRUAL_PERIOD)
    FROM ANC_PER_ACCRUAL_ENTRIES APA
    WHERE APAE.PERSON_ID = APA.PERSON_ID
    AND APA.PLAN_ID = APAE.PLAN_ID
    AND TO_CHAR(APA.ACCRUAL_PERIOD, 'YYYY') <= TO_CHAR(SYSDATE, 'YYYY')
)
```
**Why:** Accrual entries are incremental. Latest period has cumulative balance.  
**Scope:** All balance queries against `ANC_PER_ACCRUAL_ENTRIES`

### 1.6 Assignment Context
**Rule:** Link absence entries to active primary assignments  
**Pattern:**
```sql
AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
AND PAAF.PRIMARY_FLAG = 'Y'
AND PAAF.ASSIGNMENT_TYPE = 'E'
```
**Why:** Ensures absence is linked to current employment record.  
**Scope:** All queries joining `ANC_PER_ABS_ENTRIES` to `PER_ALL_ASSIGNMENTS_F`

### 1.7 Period of Service Context
**Rule:** Filter active employment periods  
**Pattern:**
```sql
AND TRUNC(SYSDATE) BETWEEN PPOS.DATE_START 
    AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY'))
```
**Why:** Links absence to valid employment term.  
**Scope:** Queries involving employment history

---

## 2. ⚡ Performance Optimization

### 2.1 Index Hints

| Object | Optimal Access Path | Hint Syntax |
|--------|---------------------|-------------|
| **ANC_PER_ABS_ENTRIES** | PERSON_ID | `/*+ INDEX(APAE ANC_PER_ABS_ENTRIES_N1) */` |
| **ANC_PER_ACCRUAL_ENTRIES** | PERSON_ID + PLAN_ID | `/*+ INDEX(APAE ANC_PER_ACCRUAL_ENTRIES_N1) */` |
| **PER_ALL_PEOPLE_F** | PERSON_ID | `/*+ INDEX(PAPF PER_PEOPLE_F_PK) */` |

### 2.2 CTE Naming Convention
**Rule:** All CTEs MUST have `/*+ qb_name(NAME) */` hint  
**Pattern:**
```sql
WITH PERIOD AS (
    /*+ qb_name(PERIOD) */
    SELECT ...
)
```
**Why:** Mandatory performance standard per system instructions.  
**Scope:** ALL CTE definitions

### 2.3 Join Order Optimization
**Recommended Order:**
1. Start with Person/Assignment base (smallest set after date filtering)
2. Join to Absence Entries (filtered by date range)
3. Join to Absence Types/Reasons (lookup tables)
4. Join to Accrual Balances (aggregated data)
5. Join to Organizational/Manager hierarchy (denormalized data)

---

## 3. 🗺️ Schema Map (Key Tables)

### 3.1 Core Absence Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **APAE** | `ANC_PER_ABS_ENTRIES` | Absence Entry Header (Leave requests) |
| **AATFT** | `ANC_ABSENCE_TYPES_F_TL` | Absence Type Master (Annual, Sick, etc.) |
| **APACE** | `ANC_PER_ACCRUAL_ENTRIES` | Accrual Balance History |
| **AAPV** | `ANC_ABSENCE_PLANS_VL` | Absence Plan Definitions |
| **AATRF** | `ANC_ABSENCE_TYPE_REASONS_F` | Absence Reason Link |
| **AARF** | `ANC_ABSENCE_REASONS_F_TL` | Absence Reason Master |

### 3.2 Integration Tables (HR Core)

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PAPF** | `PER_ALL_PEOPLE_F` | Person Header |
| **PPNF** | `PER_PERSON_NAMES_F` | Person Names |
| **PAAF** | `PER_ALL_ASSIGNMENTS_F` | Assignment Details (contains PERSON_TYPE_ID) |
| **PPTTL** | `PER_PERSON_TYPES_TL` | Person Type Translations |
| **PPOS** | `PER_PERIODS_OF_SERVICE` | Employment Terms |
| **PASF** | `PER_ASSIGNMENT_SUPERVISORS_F` | Manager Hierarchy |

### 3.3 Organizational Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **HAOU** | `HR_ALL_ORGANIZATION_UNITS` | Organization Unit Master |
| **HAOUF** | `HR_ALL_ORGANIZATION_UNITS_F` | Date-tracked Org Units |
| **HAUFT** | `HR_ORGANIZATION_UNITS_F_TL` | Org Unit Translations |
| **HOUCF** | `HR_ORG_UNIT_CLASSIFICATIONS_F` | Org Classifications (DEPARTMENT, etc.) |

### 3.4 Workflow/Approval Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **WF** | `FA_FUSION_SOAINFRA.WFTASK` | Workflow Task History |

---

## 4. 📋 Standard Filters (Copy-Paste Ready)

### 4.1 Employee Base Filter
```sql
AND PAPF.PERSON_ID = PPNF.PERSON_ID
AND PAPF.PERSON_ID = PAAF.PERSON_ID
AND PPNF.NAME_TYPE = 'GLOBAL'
AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
AND PAAF.PRIMARY_FLAG = 'Y'
AND PAAF.ASSIGNMENT_TYPE = 'E'
AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE) AND TRUNC(PPNF.EFFECTIVE_END_DATE)
AND TRUNC(SYSDATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
```

### 4.2 Absence Entry Filter (Approved Only)
```sql
AND AATFT.LANGUAGE = 'US'
AND AATFT.ABSENCE_TYPE_ID = APAE.ABSENCE_TYPE_ID
AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
AND APAE.APPROVAL_STATUS_CD NOT IN ('DENIED', 'ORA_WITHDRAWN', 'ORA_AWAIT_AWAIT')
```

### 4.3 Date Range Filter (Parameters)
```sql
AND TRUNC(APAE.START_DATE) BETWEEN 
    TRUNC(CAST(NVL(:P_START_DATE, (SELECT MIN(START_DATE) FROM ANC_PER_ABS_ENTRIES)) AS DATE))
    AND TRUNC(CAST(NVL(:P_END_DATE, LAST_DAY(SYSDATE)) AS DATE))
```

### 4.4 Organization/Department Filter
```sql
AND HAOUF.ORGANIZATION_ID = HOUCF.ORGANIZATION_ID
AND HAOUF.ORGANIZATION_ID = HAUFT.ORGANIZATION_ID
AND HAOUF.EFFECTIVE_START_DATE BETWEEN HOUCF.EFFECTIVE_START_DATE AND HOUCF.EFFECTIVE_END_DATE
AND HAUFT.LANGUAGE = 'US'
AND HAUFT.EFFECTIVE_START_DATE = HAOUF.EFFECTIVE_START_DATE
AND HAUFT.EFFECTIVE_END_DATE = HAOUF.EFFECTIVE_END_DATE
AND HOUCF.CLASSIFICATION_CODE = 'DEPARTMENT'
AND SYSDATE BETWEEN HAUFT.EFFECTIVE_START_DATE AND HAUFT.EFFECTIVE_END_DATE
```

### 4.5 Workflow Approval Filter
```sql
AND WF.OUTCOME IN ('APPROVE')
AND WF.ASSIGNEES IS NOT NULL
AND WF.WORKFLOWPATTERN NOT IN ('AGGREGATION', 'FYI')
AND TO_CHAR(APAE.PER_ABSENCE_ENTRY_ID) = WF.IDENTIFICATIONKEY(+)
```

---

## 5. 🔐 Business Rules

### 5.1 Leave Balance Calculation
**Rule:** Always use END_BAL from latest accrual period  
**Formula:** Current Balance = `APAE.END_BAL` from MAX(ACCRUAL_PERIOD)  
**Projected Balance:** Not stored - must be calculated as:
```
Projected = Current Balance - Pending Approved Leave Days
```

### 5.2 Duration Calculation
**Rule:** Duration is stored in `ANC_PER_ABS_ENTRIES.DURATION`  
**Unit:** Days (decimal, e.g., 0.5 for half-day)  
**Conversion:** Already calculated by Oracle - DO NOT recalculate from dates

### 5.3 Absence Type vs Plan Relationship
**Rule:** 
- `ANC_ABSENCE_TYPES_F_TL.NAME` = Display name (e.g., "Annual Leave")
- `ANC_ABSENCE_PLANS_VL.NAME` = Plan name (e.g., "Annual Leave Plan")
- Join: `PLAN_ID` links entries to plans
- Join: `ABSENCE_TYPE_ID` links entries to types

### 5.4 Absence Reason Handling
**Rule:** Absence reasons are OPTIONAL  
**Pattern:**
```sql
LEFT JOIN to ANC_ABSENCE_TYPE_REASONS_F using ABSENCE_TYPE_REASON_ID
```
**Why:** Not all absence types require reasons

### 5.5 Manager/Supervisor Lookup
**Rule:** Use `PER_ASSIGNMENT_SUPERVISORS_F` for current manager  
**Pattern:**
```sql
AND PASF.PERSON_ID = [Employee.PERSON_ID]
AND PASF.MANAGER_ID = [Manager.PERSON_ID]
AND TRUNC(SYSDATE) BETWEEN TRUNC(PASF.EFFECTIVE_START_DATE) AND TRUNC(PASF.EFFECTIVE_END_DATE)
```

### 5.6 Historical Leave Comparison
**Rule:** To find "All Previous Leave Taken Records":
```sql
SELECT COUNT(*) or details
FROM ANC_PER_ABS_ENTRIES APAE_HIST
WHERE APAE_HIST.PERSON_ID = [Current.PERSON_ID]
AND APAE_HIST.ABSENCE_TYPE_ID = [Current.ABSENCE_TYPE_ID]
AND APAE_HIST.PER_ABSENCE_ENTRY_ID <> [Current.PER_ABSENCE_ENTRY_ID]
AND APAE_HIST.APPROVAL_STATUS_CD = 'APPROVED'
AND APAE_HIST.START_DATE < [Current.START_DATE]
```

---

## 6. 🎯 Report-Specific Rules

### 6.1 Leave History Report
**Purpose:** Show all approved leaves with historical comparison  
**Key Columns:**
- Start Date, End Date, Type of Leave, Duration
- Historical comparison indicator
- Previous leave records summary
- Remarks/Details

**Filters:**
- Approved status only
- Date range (parameter-driven)
- Employee filter (parameter-driven)

### 6.2 Leave Balance Report
**Purpose:** Show current leave balances and active absences  
**Key Columns:**
- Employee details
- Absence type, start/end dates, duration
- Current accrual balance
- Absence reason, submission date
- Approval and absence status

**Filters:**
- Active absences (currently ongoing or future)
- Current year accrual balance
- Approved or submitted status

### 6.3 Leave Record Form
**Purpose:** Auto-generated form for single leave transaction  
**Key Fields:**
- Employee details (auto-populated)
- Leave information from timesheet
- Timesheet entry reference number
- Leave balance (before and after)
- Manager approval fields
- HR remarks

**Unique Requirements:**
- Single row per leave transaction
- Include SYSDATE for record generation
- Employee Login ID for signature
- Manager name and details

---

## 7. ⚠️ Common Pitfalls

### 7.1 Duplicate Rows
**Problem:** Getting multiple rows per absence entry  
**Causes:**
- Missing date-track filter on `_F` tables
- Missing `LANGUAGE = 'US'` on translation tables
- Joining without proper period of service context

**Solution:** Apply all date-track and language filters

### 7.2 PERSON_TYPE_ID Column Location
**Problem:** Column not found error for PERSON_TYPE_ID  
**Cause:**
- `PERSON_TYPE_ID` is NOT in `PER_ALL_PEOPLE_F`
- `PERSON_TYPE_ID` is in `PER_ALL_ASSIGNMENTS_F`

**Solution:** 
Always join through `PER_ALL_ASSIGNMENTS_F` to access `PERSON_TYPE_ID`:
```sql
FROM PER_ALL_PEOPLE_F PAPF,
     PER_ALL_ASSIGNMENTS_F PAAF,
     PER_PERSON_TYPES_TL PPTTL
WHERE PAPF.PERSON_ID = PAAF.PERSON_ID
AND PAAF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID(+)
```

### 7.2 Missing Balances
**Problem:** NULL or zero balances  
**Causes:**
- Not filtering for MAX(ACCRUAL_PERIOD)
- Wrong plan name or plan ID
- Not filtering for current year

**Solution:** Use MAX subquery pattern from Section 1.5

### 7.3 Workflow Data Issues
**Problem:** Missing approver names  
**Causes:**
- `FA_FUSION_SOAINFRA.WFTASK` is a cross-schema view
- May require special permissions
- Workflow data may not be available in all environments

**Solution:** Use outer join and handle NULLs gracefully

### 7.4 Organization Hierarchy
**Problem:** Missing department names  
**Causes:**
- Complex date-tracked joins across 3+ tables
- Missing classification code filter

**Solution:** Use pre-validated DEPARTMENTS CTE from repository

### 7.5 Plan Name Case Sensitivity
**Problem:** Leave balances not found due to case sensitivity  
**Causes:**
- Plan names may be stored with different casing
- Direct string comparison fails

**Solution:** Use `UPPER()` function for case-insensitive matching:
```sql
AND UPPER(AAPV.NAME) = 'ANNUAL LEAVE'
```

### 7.6 Terminated Employee Date Filtering
**Problem:** Incorrect date-track filtering for terminated employees  
**Causes:**
- Standard SYSDATE filter doesn't work for terminated employees
- Need to handle both active and terminated employees

**Solution:** Use `LEAST()` function with termination date:
```sql
AND LEAST(NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('31/12/4712', 'DD/MM/YYYY')), TRUNC(SYSDATE))
    BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
```

### 7.7 Source Language vs Language
**Problem:** Duplicate rows when using `_F_TL` tables  
**Causes:**
- Missing `SOURCE_LANG` filter
- `LANGUAGE` filter alone is insufficient

**Solution:** Always use both filters for `_F_TL` tables:
```sql
AND AAPFTL.SOURCE_LANG = 'US'
AND AAPFTL.LANGUAGE = 'US'
```

---

## 8. 📊 Standard Columns (Always Include)

### 8.1 Employee Identification
- `PERSON_NUMBER` - Employee number
- `DISPLAY_NAME` / `FULL_NAME` - Employee name
- `EMPLOYEE_TYPE` - Person type

### 8.2 Organizational Context
- `DEPARTMENT` - Department name
- `BUSINESS_UNIT` - Business unit name
- `JOB` - Job title

### 8.3 Absence Details
- `ABSENCE_TYPE` - Leave type name
- `START_DATE` - Leave start date (formatted)
- `END_DATE` - Leave end date (formatted)
- `DURATION` - Number of days

### 8.4 Status Fields
- `APPROVAL_STATUS` - Approval status (meaning)
- `ABSENCE_STATUS` - Absence status (meaning)
- `SUBMITTED_DATE` - Date submitted
- `CONFIRMED_DATE` - Date approved/confirmed

### 8.5 Audit Columns (Optional but Recommended)
- `CREATION_DATE` - Record creation date
- `LAST_UPDATE_DATE` - Last modified date
- `CREATED_BY` - Created by user
- `LAST_UPDATED_BY` - Last updated by user

---

## 9. 🔧 Advanced Patterns

### 9.1 Workflow Approval Integration
**Purpose:** Link absence entries to workflow approval history  
**Table:** `FA_FUSION_SOAINFRA.WFTASK`

**Pattern:**
```sql
,WORKFLOW_APPROVAL AS (
    SELECT
        WF.IDENTIFICATIONKEY,
        WF.ASSIGNEESDISPLAYNAME APPROVER_NAME,
        WF.FROMUSERDISPLAYNAME REQUESTOR_NAME,
        WF.OUTCOME
    FROM
        FA_FUSION_SOAINFRA.WFTASK WF
    WHERE
        WF.OUTCOME IN ('APPROVE')
    AND WF.ASSIGNEES IS NOT NULL
    AND WF.WORKFLOWPATTERN NOT IN ('AGGREGATION', 'FYI')
)
```

**Join Pattern:**
```sql
AND TO_CHAR(APAE.PER_ABSENCE_ENTRY_ID) = WF.IDENTIFICATIONKEY(+)
```

**Notes:**
- Cross-schema access required (FA_FUSION_SOAINFRA)
- May not be available in all environments
- Always use outer join
- IDENTIFICATIONKEY is character-based, convert PER_ABSENCE_ENTRY_ID to CHAR

### 9.2 Payroll Integration (Salary Advance Recovery)
**Purpose:** Link salary advance recovery to absence entries  
**Tables:** `PAY_ELEMENT_ENTRIES_F`, `PAY_ELEMENT_ENTRY_VALUES_F`, `PAY_INPUT_VALUES_F`, `PAY_ELEMENT_TYPES_F`

**Pattern:**
```sql
,SALARY_ADVANCE AS (
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

**Join Pattern:**
```sql
LEFT JOIN SALARY_ADVANCE SA ON PAAF.PERSON_ID = SA.PERSON_ID
```

### 9.3 Specific Absence Type Filtering
**Rule:** Different absence types may require different handling  

**Annual Leave Staff Pattern:**
```sql
AND ((ABS_TYPE.NAME = 'Annual Leave Staff'
      OR ABS_TYPE.NAME <> 'Annual Leave Staff')
    AND TRUNC(SYSDATE) BETWEEN ABS_ENTR.START_DATE AND ABS_ENTR.END_DATE)
```

**Multiple Type Filter:**
```sql
AND AATFT.NAME IN ('Annual Leave', 'Annual Leave Staff', 'Sick Leave')
```

**Case-Insensitive Filter:**
```sql
AND UPPER(AAPV.NAME) = 'ANNUAL LEAVE'
```

### 9.4 Current Year Accrual Balance Filter
**Purpose:** Get accrual balance for specific year only  

**Pattern:**
```sql
AND APAE.ACCRUAL_PERIOD BETWEEN 
    TO_DATE('01/01/' || TO_CHAR(TRUNC(SYSDATE), 'YYYY'), 'DD/MM/YYYY') 
    AND TO_DATE('31/12/' || TO_CHAR(TRUNC(SYSDATE), 'YYYY'), 'DD/MM/YYYY')
```

**Why:** Filters accrual periods to current calendar year only

### 9.5 Status Decoding via HR_LOOKUPS
**Purpose:** Convert status codes to user-friendly meanings  

**Approval Status Pattern:**
```sql
SELECT HL.MEANING
FROM HR_LOOKUPS HL
WHERE HL.LOOKUP_CODE = ABS_ENTR.APPROVAL_STATUS_CD
AND HL.LOOKUP_TYPE = 'ANC_PER_ABS_ENT_APROVAL_STATUS'
```

**Absence Status Pattern:**
```sql
SELECT HL.MEANING
FROM HR_LOOKUPS HL
WHERE HL.LOOKUP_CODE = ABS_ENTR.ABSENCE_STATUS_CD
AND HL.LOOKUP_TYPE = 'ANC_PER_ABS_ENT_STATUS'
```

**Filter by Meaning:**
```sql
AND (SELECT HL.MEANING
     FROM HR_LOOKUPS HL
     WHERE HL.LOOKUP_CODE = ABS_ENTR.ABSENCE_STATUS_CD
     AND HL.LOOKUP_TYPE = 'ANC_PER_ABS_ENT_STATUS') 
    NOT IN ('Withdrawn', 'Denied', 'Awaiting')
```

### 9.6 Complex Absence Reason Lookup
**Purpose:** Get absence reason name through type relationship  

**Pattern:**
```sql
,ABSENCE_REASONS AS (
    SELECT
        AARF.NAME,
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

**Join Pattern:**
```sql
AND APAE.ABSENCE_TYPE_REASON_ID = REA.ABSENCE_TYPE_REASON_ID(+)
```

---

## 10. 🔄 Integration Points

### 10.1 Payroll Integration
**Link:** Absence entries feed into payroll for leave deductions  
**Key Fields:** `PERSON_ID`, `ABSENCE_TYPE_ID`, `START_DATE`, `END_DATE`, `DURATION`

### 10.2 Time and Labor Integration
**Link:** Absence entries may originate from timesheet  
**Key Fields:** Timesheet entry reference (if available)

### 10.3 Workflow Integration
**Link:** Approval workflow tracks in `FA_FUSION_SOAINFRA.WFTASK`  
**Key Fields:** `IDENTIFICATIONKEY` = `PER_ABSENCE_ENTRY_ID`

### 10.4 Salary Advance Recovery Integration
**Link:** Leave may impact salary advance recovery in payroll  
**Tables:** `PAY_ELEMENT_ENTRIES_F`, `PAY_ELEMENT_ENTRY_VALUES_F`  
**Key Fields:** `PERSON_ID`, Element type: 'SALARY ADVANCE RECOVERY RETRO'

---

## 11. ✅ Self-Validation Checklist

Before finalizing any Absence Management query, verify:

- [ ] Date-track filter applied to all `_F` tables
- [ ] `LANGUAGE = 'US'` applied to all `_TL` tables
- [ ] Approval status filter applied (exclude DENIED, WITHDRAWN)
- [ ] Absence status filter applied (exclude Withdrawn)
- [ ] MAX accrual period logic for balances
- [ ] Active assignment filters applied
- [ ] Oracle Traditional Join Syntax used (no ANSI joins)
- [ ] All CTEs have `/*+ qb_name(NAME) */` hints
- [ ] Date formatting uses `INITCAP(TO_CHAR(..., 'DD-fmMON-YYYY'))`
- [ ] Parameters handle NULL with NVL or default values
- [ ] All joins use outer join (+) where appropriate

---

## Summary

This ABSENCE_MASTER document contains:
- Critical constraints for absence/leave queries
- Approval and status filtering rules
- Performance optimization patterns
- Schema map and table relationships
- Standard filters (copy-paste ready)
- Business rules for calculations
- Report-specific requirements
- Common pitfalls and solutions
- Self-validation checklist

**Usage:** Review this document BEFORE generating any absence/leave SQL. Apply all constraints and patterns.

**Update Frequency:** When new absence types, plans, or business rules are added.

---

**END OF ABSENCE_MASTER.md**

