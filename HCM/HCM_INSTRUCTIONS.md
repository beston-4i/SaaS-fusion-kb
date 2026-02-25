# HCM Module Instructions

**Domain:** Oracle Fusion Human Capital Management
**Location:** `FUSION_SAAS/HCM/`

---

## 1. Ã°Å¸â€œâ€š Module Navigation (Routes)

| Sub-Module | Instruction File | Repository File | Template File |
|------------|------------------|-----------------|---------------|
| **Core HR** | [HR_MASTER](HR/HR_MASTER.md) | [HR_REPOS](HR/HR_REPOSITORIES.md) | [HR_TMPL](HR/HR_TEMPLATES.md) |
| **Payroll** | [PAY_MASTER](PAY/PAY_MASTER.md) | [PAY_REPOS](PAY/PAY_REPOSITORIES.md) | [PAY_TMPL](PAY/PAY_TEMPLATES.md) |
| **Benefits** | [BEN_MASTER](BEN/BEN_MASTER.md) | [BEN_REPOS](BEN/BEN_REPOSITORIES.md) | [BEN_TMPL](BEN/BEN_TEMPLATES.md) |
| **Absence** | [ABSENCE_MASTER](ABSENCE/ABSENCE_MASTER.md) | [ABSENCE_REPOS](ABSENCE/ABSENCE_REPOSITORIES.md) | [ABSENCE_TMPL](ABSENCE/ABSENCE_TEMPLATES.md) |
| **Time and Labor** | [TL_MASTER](TIME_LABOR/TL_MASTER.md) | [TL_REPOS](TIME_LABOR/TL_REPOSITORIES.md) | [TL_TMPL](TIME_LABOR/TL_TEMPLATES.md) |
| **Compensation** | [CMP_MASTER](COMPENSATION/CMP_MASTER.md) | [CMP_REPOS](COMPENSATION/CMP_REPOSITORIES.md) | [CMP_TMPL](COMPENSATION/CMP_TEMPLATES.md) |
| **Recruiting** | [ORC_MASTER](RECRUITING/ORC_MASTER.md) | [ORC_REPOS](RECRUITING/ORC_REPOSITORIES.md) | [ORC_TMPL](RECRUITING/ORC_TEMPLATES.md) |
| **Performance** | [PMS_MASTER](Performation/PMS_MASTER.md) | [PMS_REPOS](Performation/PMS_REPOSITORIES.md) | [PMS_TMPL](Performation/PMS_TEMPLATES.md) |

---

## 2. Ã°Å¸â€â€” Shared Integration Rules (Cross-Module)

### A. Date-Effective Records (The #1 Rule)
*   **Concept:** HCM tables track history. A person has multiple rows (one per change).
*   **Rule:** ALWAYS filter `TRUNC(SYSDATE) BETWEEN EFFECTIVE_START_DATE AND EFFECTIVE_END_DATE` (for current snapshots).
*   **Tables:** `PER_ALL_PEOPLE_F`, `PER_ALL_ASSIGNMENTS_M`, `PAY_ELEMENT_ENTRIES_F` (Any table ending in `_F` or `_M`).

### B. Enterprise Structures
*   **Business Unit:** `ORG_ID` (HR) vs `BU_ID` (Finance). Be careful with naming.
*   **Legal Entity:** `LEGAL_ENTITY_ID`.
*   **Legislative Data Group (LDG):** Crucial for Payroll partitioning.

### C. Security
*   **Assignment-Based Security:** Most data is secured at the Assignment level, not just Person level.

### D. SQL Coding Standards

#### D.1 Never Use '&' Character
*   **NEVER use '&' character:** Do not use '&' anywhere in SQL files (code, comments, or documentation)
*   **Reason:** '&' is a substitution variable marker in SQL*Plus/SQL Developer and causes prompts
*   **Rule:** Always use 'and' instead of '&' (e.g., "Time and Labor" not "Time & Labor")
*   **Applies to:** SQL code, SQL comments, and inline documentation

**Examples:**
```sql
-- WRONG: Time & Labor report
-- CORRECT: Time and Labor report

-- WRONG: Probation & Notice Period
-- CORRECT: Probation and Notice Period

-- WRONG: SELECT * FROM table WHERE col1 = 'A' & col2 = 'B'
-- CORRECT: SELECT * FROM table WHERE col1 = 'A' AND col2 = 'B'
```

#### D.2 Always Use NLS_DATE_LANGUAGE for Date Formatting
*   **ALWAYS use NLS_DATE_LANGUAGE for date formatting:** Include 'NLS_DATE_LANGUAGE=ENGLISH' in TO_CHAR date formatting
*   **Reason:** Ensures consistent English month/day names regardless of database language settings
*   **Rule:** Add NLS parameter to all TO_CHAR functions with date format masks containing month/day names
*   **Applies to:** Any format mask with Mon, Month, Day, or similar text-based date elements

**Examples:**
```sql
-- WRONG: TO_CHAR(date_column, 'DD-Mon-YYYY')
-- CORRECT: TO_CHAR(date_column, 'DD-Mon-YYYY', 'NLS_DATE_LANGUAGE=ENGLISH')

-- WRONG: TO_CHAR(date_column, 'DD-MON-YYYY')
-- CORRECT: TO_CHAR(date_column, 'DD-MON-YYYY', 'NLS_DATE_LANGUAGE=ENGLISH')

-- WRONG: TO_CHAR(date_column, 'Day')
-- CORRECT: TO_CHAR(date_column, 'Day', 'NLS_DATE_LANGUAGE=ENGLISH')

-- WRONG: TO_CHAR(date_column, 'DD-Mon-YYYY HH24:MI:SS')
-- CORRECT: TO_CHAR(date_column, 'DD-Mon-YYYY HH24:MI:SS', 'NLS_DATE_LANGUAGE=ENGLISH')

-- NOT NEEDED: TO_CHAR(date_column, 'YYYY-MM-DD')  -- No month/day names
-- NOT NEEDED: TO_CHAR(date_column, 'DD-MM-YYYY')  -- No month/day names
-- NOT NEEDED: TO_CHAR(date_column, 'HH24:MI:SS')  -- No date component
```

**When NLS is Required:**
- Format masks with: Mon, MON, Month, MONTH, Day, DAY, Dy, DY
- Ensures output like "01-Jan-2023" not "01-يناير-2023" (Arabic) or "01-Janv-2023" (French)

**When NLS is NOT Required:**
- Numeric-only formats: YYYY-MM-DD, DD-MM-YYYY, YYYYMMDD
- Time-only formats: HH24:MI:SS
- Numeric day/month: DD, MM, YYYY

### E. Critical Table Mappings (Common Mistakes)
*   **DATE_OF_BIRTH:** Located in `PER_PERSONS`, **NOT** `PER_ALL_PEOPLE_F`
*   **SEX (Gender):** Located in `PER_PEOPLE_LEGISLATIVE_F`, **NOT** `PER_ALL_PEOPLE_F`
*   **MARITAL_STATUS:** Located in `PER_PEOPLE_LEGISLATIVE_F`, **NOT** `PER_ALL_PEOPLE_F`
*   **PERSON_TYPE_ID:** Located in `PER_ALL_ASSIGNMENTS_F`, **NOT** `PER_ALL_PEOPLE_F`
*   **EMAIL_ADDRESS:** Located in `PER_EMAIL_ADDRESSES`, **NOT** `PER_ALL_PEOPLE_F`
*   **PHONE_NUMBER (Primary):** Join `PER_PHONES` using `PH.PHONE_ID = PAPF.PRIMARY_PHONE_ID`, **NOT** `PERSON_ID` + `PRIMARY_FLAG` + `PHONE_TYPE`
*   **NATIONALITY (Primary):** Filter using `PC.CITIZENSHIP_STATUS = 'A'`, **NOT** `PC.PRIMARY_FLAG = 'Y'` (column doesn't exist)
*   **MANAGER_ID:** Located in `PER_ASSIGNMENT_SUPERVISORS_F`, **NOT** `PER_ALL_ASSIGNMENTS_F` (column doesn't exist)
*   **EMPLOYEE_CATEGORY:** **ALWAYS** use `PAAF.EMPLOYEE_CATEGORY` (from `PER_ALL_ASSIGNMENTS_F`) for worker category, **NOT** `ASSIGNMENT_CATEGORY` (different purpose). Use with lookup 'EMPLOYEE_CATG' to decode values like 'Employee', 'Contingent Worker', etc.
*   **LEAVING_REASON:** Located in `PER_ACTION_REASONS_TL` (via `PER_ALL_ASSIGNMENTS_M` â†’ `PER_ACTION_REASONS_B`), **NOT** in `PER_PERIODS_OF_SERVICE` (column doesn't exist)
*   **PROBATION_PERIOD:** Standard columns in `PER_ALL_ASSIGNMENTS_F` (DATE_PROBATION_END, PROBATION_PERIOD, PROBATION_UNIT) - **NOT DFF**
*   **NOTICE_PERIOD:** Standard columns in `PER_ALL_ASSIGNMENTS_F` (NOTICE_PERIOD, NOTICE_PERIOD_UOM) - **NOT DFF**
*   **JOB TITLE:** Use `PER_JOBS_F_VL.NAME` (broader classification) - Use `_F_VL` view, not separate `_F` + `_TL` joins
*   **GRADE:** Use `PER_GRADES_F_VL.NAME` - Use `_F_VL` view, not separate `_F` + `_TL` joins
*   **DESIGNATION (Position):** Use `HR_ALL_POSITIONS_F_TL.NAME` (specific role/title)
*   **RECRUITER_ID (ORC):** Located in `IRC_REQUISITIONS_VL`, **NOT** `IRC_OFFERS` (recruiters are assigned to requisitions, not individual offers)
*   **CANDIDATE_ID (ORC):** Column **DOES NOT EXIST** in `IRC_CANDIDATES` table. Use `CANDIDATE_NUMBER` (for identification) and `PERSON_ID` (for joins)
*   **OFFER ASSIGNMENTS (ORC):** Link `IRC_OFFERS` to `PER_ALL_ASSIGNMENTS_M` using **PERSON_ID**, **NOT** `ASSIGNMENT_OFFER_ID`. Filter with `ASG.ASSIGNMENT_TYPE = 'O'` for offer assignments
*   **ABSENCE_ID (Absence):** Column **DOES NOT EXIST** in `ANC_PER_ABS_ENTRIES` table. Primary key is combination of `PERSON_ID` + `ABSENCE_TYPE_ID` + `START_DATE`
*   **ABSENCE PLAN_ID (Absence):** `PLAN_ID` **DOES NOT EXIST** in `ANC_PER_ABS_ENTRIES`. Link absence entries to `ANC_PER_ACCRUAL_ENTRIES` using `PERSON_ID` + `PRD_OF_SVC_ID` (Period of Service ID), not PLAN_ID
*   **PUBLIC HOLIDAYS (PER_CALENDAR_EVENTS):** **CRITICAL - Verified Columns**
    - **Available Columns:** `CALENDAR_EVENT_ID`, `START_DATE_TIME`, `END_DATE_TIME`, `CATEGORY`, `TREE_CODE`, `TREE_STRUCTURE_CODE`, `LAST_UPDATE_DATE`, `SHORT_CODE`
    - **CALENDAR_EVENT_ID:** Primary key for calendar events (use for sync and unique identification)
    - **SHORT_CODE:** Holiday name/description (e.g., "New Year", "Independence Day") - Use this for holiday description
    - **NOT Available:** `EVENT_NAME`, `DESCRIPTION` (columns do not exist or not accessible)
    - **Filter:** Use `CATEGORY = 'PH'` for Public Holidays
    - **Geography Mapping:** Join to `PER_GEO_TREE_NODE_RF` using `TREE_CODE` and `TREE_STRUCTURE_CODE` (use OUTER JOIN +)
    - **PER_GEO_TREE_NODE_RF Available Columns:** `PK1_VALUE` (country code), `TREE_CODE`, `TREE_STRUCTURE_CODE`

**Correct Pattern:**
```sql
FROM PER_ALL_PEOPLE_F PAPF,
     PER_PERSONS PS,                      -- For DATE_OF_BIRTH (NOT date-tracked)
     PER_PEOPLE_LEGISLATIVE_F PPLF,       -- For SEX, MARITAL_STATUS (date-tracked)
     PER_ALL_ASSIGNMENTS_F PAAF,          -- For PERSON_TYPE_ID, EMPLOYEE_CATEGORY
     PER_EMAIL_ADDRESSES PEA,             -- For EMAIL_ADDRESS (date-tracked)
     PER_PHONES PH,                       -- For PHONE_NUMBER (use PRIMARY_PHONE_ID)
     PER_JOBS_F_VL JOB,                   -- For JOB_TITLE (_F_VL view, not _F + _TL)
     PER_GRADES_F_VL GRADE,               -- For GRADE (_F_VL view, not _F + _TL)
     HR_ALL_POSITIONS_F_TL POS            -- For DESIGNATION (Position)
WHERE PAPF.PERSON_ID = PS.PERSON_ID
  AND PAPF.PERSON_ID = PPLF.PERSON_ID
  AND PAPF.PERSON_ID = PAAF.PERSON_ID
  AND PAPF.PERSON_ID = PEA.PERSON_ID(+)
  AND PAPF.PRIMARY_PHONE_ID = PH.PHONE_ID(+)  -- Primary phone using PRIMARY_PHONE_ID
  -- Assignment to Job, Grade, and Position
  AND PAAF.JOB_ID = JOB.JOB_ID(+)
  AND PAAF.GRADE_ID = GRADE.GRADE_ID(+)
  AND PAAF.POSITION_ID = POS.POSITION_ID(+)
  AND POS.LANGUAGE(+) = 'US'
  -- Note: PER_PERSONS is NOT date-tracked, no EFFECTIVE_START/END_DATE
  AND :P_DATE BETWEEN PPLF.EFFECTIVE_START_DATE AND PPLF.EFFECTIVE_END_DATE
  AND :P_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
  AND (:P_DATE BETWEEN JOB.EFFECTIVE_START_DATE(+) AND JOB.EFFECTIVE_END_DATE(+) OR JOB.JOB_ID IS NULL)
  AND (:P_DATE BETWEEN GRADE.EFFECTIVE_START_DATE(+) AND GRADE.EFFECTIVE_END_DATE(+) OR GRADE.GRADE_ID IS NULL)
  AND (:P_DATE BETWEEN POS.EFFECTIVE_START_DATE(+) AND POS.EFFECTIVE_END_DATE(+) OR POS.POSITION_ID IS NULL)
  -- Email filters
  AND PEA.EMAIL_TYPE(+) = 'W1'  -- Work email
  AND :P_DATE BETWEEN PEA.DATE_FROM(+) AND NVL(PEA.DATE_TO(+), TO_DATE('4712-12-31', 'YYYY-MM-DD'))
```

**Key Distinctions:**
- **JOB (Job Title)**: Broader classification (e.g., "Manager", "Developer") - Use `PER_JOBS_F_VL` (not `_F` + `_TL`)
- **GRADE**: Employee grade/level (e.g., "Grade 5", "Senior Level") - Use `PER_GRADES_F_VL` (not `_F` + `_TL`)
- **DESIGNATION (Position)**: Specific role/title (e.g., "Senior Project Manager", "Lead Frontend Developer")
- **EMPLOYEE_CATEGORY**: Worker category (e.g., "Employee", "Contingent Worker") - **ALWAYS** use `PAAF.EMPLOYEE_CATEGORY`, **NOT** `ASSIGNMENT_CATEGORY`

**Note:** Use `_F_VL` views (e.g., `PER_JOBS_F_VL`, `PER_GRADES_F_VL`) instead of joining `_F` and `_TL` tables separately. The `_F_VL` views combine base table and translations, reducing join complexity and improving performance.

**ORC/Recruiting Pattern (Complete Offer-to-Salary Flow):**
```sql
-- CRITICAL RULES FOR ORC:
-- 1. Recruiter is assigned at REQUISITION level, not OFFER level
-- 2. IRC_CANDIDATES: Use CANDIDATE_NUMBER (identifier) and PERSON_ID (for joins)
-- 3. DO NOT use CANDIDATE_ID - this column does not exist in IRC_CANDIDATES
-- 4. Link OFFER to ASSIGNMENT using PERSON_ID, NOT ASSIGNMENT_OFFER_ID
-- 5. Filter offer assignments with ASSIGNMENT_TYPE = 'O'

SELECT CAND.CANDIDATE_NUMBER,           -- For identification/display
       CAND.PERSON_ID,                  -- For joins to other tables
       PPNF.DISPLAY_NAME,               -- Candidate name
       REQ.RECRUITER_ID,                -- Recruiter ID from requisition
       REC_NAME.DISPLAY_NAME,           -- Recruiter name
       CSSC.COMPONENT_CODE,             -- Salary component code
       CSSC.AMOUNT                      -- Component amount
FROM IRC_CANDIDATES CAND,
     IRC_SUBMISSIONS SUB,
     IRC_OFFERS OFFER,
     IRC_REQUISITIONS_VL REQ,           -- For RECRUITER_ID and requisition details
     PER_PERSON_NAMES_F PPNF,           -- For candidate name
     PER_PERSON_NAMES_F REC_NAME,       -- For recruiter name
     PER_ALL_ASSIGNMENTS_M ASG,         -- For offer assignment (CRITICAL: join via PERSON_ID)
     CMP_SALARY CSA,                    -- For salary header
     CMP_SALARY_SIMPLE_COMPNTS CSSC     -- For salary components
WHERE CAND.PERSON_ID = SUB.PERSON_ID
  AND SUB.SUBMISSION_ID = OFFER.SUBMISSION_ID
  AND SUB.REQUISITION_ID = REQ.REQUISITION_ID
  -- Candidate name
  AND CAND.PERSON_ID = PPNF.PERSON_ID
  AND PPNF.NAME_TYPE = 'GLOBAL'
  AND TRUNC(SYSDATE) BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
  -- Recruiter information from requisition
  AND REQ.RECRUITER_ID = REC_NAME.PERSON_ID(+)
  AND REC_NAME.NAME_TYPE(+) = 'GLOBAL'
  AND TRUNC(SYSDATE) BETWEEN REC_NAME.EFFECTIVE_START_DATE(+) AND REC_NAME.EFFECTIVE_END_DATE(+)
  -- Offer to Assignment (CRITICAL: use PERSON_ID, not ASSIGNMENT_OFFER_ID)
  AND CAND.PERSON_ID = ASG.PERSON_ID
  AND ASG.ASSIGNMENT_TYPE = 'O'         -- 'O' = Offer Assignment
  AND ASG.EFFECTIVE_LATEST_CHANGE = 'Y'
  AND TRUNC(SYSDATE) BETWEEN ASG.EFFECTIVE_START_DATE AND ASG.EFFECTIVE_END_DATE
  -- Assignment to Salary
  AND ASG.ASSIGNMENT_ID = CSA.ASSIGNMENT_ID
  AND TRUNC(SYSDATE) BETWEEN CSA.DATE_FROM AND CSA.DATE_TO
  -- Salary to Components
  AND CSA.SALARY_ID = CSSC.SALARY_ID
  AND CSSC.ASSIGNMENT_ID = ASG.ASSIGNMENT_ID
  AND TRUNC(SYSDATE) BETWEEN CSSC.SALARY_DATE_FROM AND CSSC.SALARY_DATE_TO
```

**Reference:** 
- HCM/HR/HR_MASTER.md Sections 12.2 and 15.1 for complete field mappings
- HCM/HR/HR_REPOSITORIES.md Section 4 (HR_ASG_MASTER) for Job/Position pattern
- HCM/RECRUITING/ORC_REPOSITORIES.md Section 1 (IRC_REQUISITIONS_VL) for Recruiter pattern

**Absence/Leave Balance Pattern (Absence Entries to Accrual Balance):**
```sql
-- CRITICAL RULES FOR ABSENCE BALANCE:
-- 1. PLAN_ID does NOT exist in ANC_PER_ABS_ENTRIES
-- 2. Link absence entries to accrual balance using PERSON_ID + PRD_OF_SVC_ID
-- 3. Use SUM(END_BAL) to aggregate all plan balances for the person
-- 4. Date range filter: START_DATE >= FROM_DATE AND END_DATE <= TO_DATE (within range, not overlap)

-- Get Period of Service ID from assignment
WITH EMP_BASE AS (
    SELECT
        PAPF.PERSON_ID,
        PAAF.PERIOD_OF_SERVICE_ID
    FROM PER_ALL_PEOPLE_F PAPF,
         PER_ALL_ASSIGNMENTS_F PAAF
    WHERE PAPF.PERSON_ID = PAAF.PERSON_ID
      AND PAAF.PRIMARY_FLAG = 'Y'
      AND PAAF.ASSIGNMENT_TYPE = 'E'
      AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
)
,LEAVE_BALANCE AS (
    SELECT
        EB.PERSON_ID,
        APACE.PRD_OF_SVC_ID,
        SUM(APACE.END_BAL) AS CURRENT_BALANCE
    FROM EMP_BASE EB,
         ANC_PER_ACCRUAL_ENTRIES APACE
    WHERE EB.PERSON_ID = APACE.PERSON_ID
      AND EB.PERIOD_OF_SERVICE_ID = APACE.PRD_OF_SVC_ID
      -- Get latest accrual period
      AND APACE.ACCRUAL_PERIOD = (
          SELECT MAX(APACE2.ACCRUAL_PERIOD)
          FROM ANC_PER_ACCRUAL_ENTRIES APACE2
          WHERE APACE2.PERSON_ID = APACE.PERSON_ID
            AND APACE2.PRD_OF_SVC_ID = APACE.PRD_OF_SVC_ID
            AND TO_CHAR(APACE2.ACCRUAL_PERIOD, 'YYYY') <= TO_CHAR(SYSDATE, 'YYYY')
      )
    GROUP BY EB.PERSON_ID, APACE.PRD_OF_SVC_ID
)
-- Join absence entries to leave balance
SELECT APAE.PERSON_ID,
       APAE.START_DATE,
       APAE.END_DATE,
       LB.CURRENT_BALANCE
FROM ANC_PER_ABS_ENTRIES APAE,
     EMP_BASE EB,
     LEAVE_BALANCE LB
WHERE APAE.PERSON_ID = EB.PERSON_ID
  -- Link via PERSON_ID + PRD_OF_SVC_ID (NOT PLAN_ID)
  AND EB.PERSON_ID = LB.PERSON_ID(+)
  AND EB.PERIOD_OF_SERVICE_ID = LB.PRD_OF_SVC_ID(+)
  -- Date range filter: absences WITHIN the parameter range
  AND TRUNC(APAE.START_DATE) >= TRUNC(:P_FROM_DATE)
  AND TRUNC(APAE.END_DATE) <= TRUNC(:P_TO_DATE)
  -- Standard absence filters
  AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
  AND APAE.APPROVAL_STATUS_CD = 'APPROVED'
```

**Absence Adjustment Pattern (Detailed Balance Components):**
```sql
-- CRITICAL RULES FOR ABSENCE ADJUSTMENTS:
-- 1. Use ANC_PER_ACRL_ENTRY_DTLS (detail table) for adjustments, NOT ANC_PER_ACCRUAL_ENTRIES
-- 2. Link via PER_PLAN_ENRT_ID from ANC_PER_PLAN_ENROLLMENT
-- 3. Filter by PLAN_PERIOD_TYPE for correct period boundaries
-- 4. Balance = PY_CARRYOVER + CY_ACCRUAL + ADJUSTMENTS - TAKEN - ENCASHMENT - EXPIRY

WITH PLAN_ENROLLMENT_ACTIVE AS (
    SELECT
        APPE.PER_PLAN_ENRT_ID,
        APPE.PERSON_ID,
        APPE.PLAN_ID,
        AAPV.NAME AS PLAN_NAME,
        AAPV.PLAN_PERIOD_TYPE  -- 'C' = Calendar Year, Other = Anniversary
    FROM ANC_PER_PLAN_ENROLLMENT APPE,
         ANC_ABSENCE_PLANS_VL AAPV
    WHERE APPE.PLAN_ID = AAPV.ABSENCE_PLAN_ID
      AND APPE.STATUS = 'A'
      AND AAPV.PLAN_STATUS = 'A'
)
,ACCRUAL_DETAILS AS (
    SELECT
        PEA.PERSON_ID,
        -- Carryover (TYPE='COVR' OR TYPE='ADJOTH' with ADJUSTMENT_REASON='CARRYOVER')
        NVL(SUM(CASE WHEN (APACD.TYPE = 'COVR' 
                       OR (APACD.TYPE = 'ADJOTH' AND APACD.ADJUSTMENT_REASON = 'CARRYOVER'))
                     THEN APACD.VALUE ELSE 0 END), 0) AS PY_CARRYOVER,
        -- Current Year Accrual (TYPE IN 'ACRL', 'ORA_ANC_COMPTME', 'FLDR')
        NVL(SUM(CASE WHEN APACD.TYPE IN ('ACRL', 'ORA_ANC_COMPTME', 'FLDR')
                     THEN APACD.VALUE ELSE 0 END), 0) AS CY_ACCRUAL,
        -- Taken Leave (TYPE='ABS' with PROCD_DATE <= effective date)
        ABS(NVL(SUM(CASE WHEN APACD.TYPE = 'ABS' AND APACD.PROCD_DATE <= TRUNC(SYSDATE)
                         THEN APACD.VALUE ELSE 0 END), 0)) AS TAKEN_LEAVE,
        -- Adjustments (TYPE IN 'ADJOTH', 'INIT', exclude CARRYOVER)
        NVL(SUM(CASE WHEN APACD.TYPE IN ('ADJOTH', 'INIT')
                     AND NVL(APACD.ADJUSTMENT_REASON, 'X') <> 'CARRYOVER'
                     THEN APACD.VALUE ELSE 0 END), 0) AS ADJUSTMENT,
        -- Encashment (TYPE='CSH')
        NVL(SUM(CASE WHEN APACD.TYPE = 'CSH'
                     THEN APACD.VALUE ELSE 0 END), 0) AS ENCASHMENT,
        -- Carryover Expiry (TYPE='COVREX')
        NVL(SUM(CASE WHEN APACD.TYPE = 'COVREX'
                     THEN APACD.VALUE ELSE 0 END), 0) AS CARRYOVER_EXPIRY
    FROM PLAN_ENROLLMENT_ACTIVE PEA,
         ANC_PER_ACRL_ENTRY_DTLS APACD  -- Detail transaction table
    WHERE PEA.PER_PLAN_ENRT_ID = APACD.PER_PLAN_ENRT_ID
      AND PEA.PLAN_ID = APACD.PL_ID
      -- Filter by plan period (Calendar vs Anniversary)
      AND APACD.PROCD_DATE BETWEEN 
          CASE WHEN PEA.PLAN_PERIOD_TYPE = 'C' 
               THEN TO_DATE('0101' || TO_CHAR(TRUNC(SYSDATE),'YYYY'),'DDMMYYYY')
               ELSE TRUNC(TRUNC(SYSDATE), 'YYYY')
          END
          AND TRUNC(SYSDATE)
    GROUP BY PEA.PERSON_ID
)
SELECT 
    PERSON_ID,
    PY_CARRYOVER,
    CY_ACCRUAL,
    ADJUSTMENT,
    TAKEN_LEAVE,
    ENCASHMENT,
    CARRYOVER_EXPIRY,
    -- Total Balance Formula
    (PY_CARRYOVER + CY_ACCRUAL + ADJUSTMENT - TAKEN_LEAVE - ENCASHMENT - CARRYOVER_EXPIRY) AS TOTAL_BALANCE
FROM ACCRUAL_DETAILS
```

**ANC_PER_ACRL_ENTRY_DTLS TYPE Codes:**
| TYPE | Meaning | VALUE Sign | Example |
|------|---------|------------|---------|
| ACRL | Accrual | Positive | Regular monthly accrual |
| ABS | Absence | Negative | Leave taken |
| COVR | Carryover | Positive | PY balance carried forward |
| ADJOTH | Adjustment | Ã‚Â±Both | Manual corrections |
| INIT | Initial | Positive | Opening balance |
| CSH | Encashment | Negative | Cash-out |
| COVREX | Carryover Expiry | Negative | Expired carryover |

**Reference:**
- HCM/ABSENCE/ABSENCE_MASTER.md for complete absence patterns
- HCM/ABSENCE/ABSENCE_REPOSITORIES.md for standard CTEs
- HCM/ABSENCE/ABSENCE_MASTER.md Section 3.6 for ANC_PER_ACRL_ENTRY_DTLS details
- HCM/ABSENCE/ABSENCE_MASTER.md Section 5.15 for balance component breakdown

**Time and Labor Pattern (Production - Missing Timecard/Overtime Reports):**
```sql
-- CRITICAL RULES FOR TIME and LABOR:
-- 1. SCHEDULES assigned at TWO levels: Legal Entity (LEGALEMP) + Assignment (ASSIGN)
-- 2. MUST use UNION to get both schedule assignment types
-- 3. Use ZMM_SR_AVAILABLE_DATES for working dates calendar
-- 4. PERIOD OF SERVICE filter: Use MAX(DATE_START) <= SYSDATE (gets latest hire/rehire)
-- 5. 3-level version control: LATEST_VERSION + STATUS + CREATION_DATE
-- 6. NOT EXISTS pattern for missing timecard detection
-- 7. Exclude public holidays and absences from missing checks
-- 8. Index hints CRITICAL for performance (HWM tables are large)

-- CTE 1: Get employees with schedules (BOTH resource types - UNION required)
WITH PERSON_DETAILS AS (
    -- Part 1: Legal Entity Level Schedules
    SELECT
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
        NULL END_DATE
    FROM
        PER_ALL_PEOPLE_F          PAPF,
        PER_PERSON_NAMES_F       PPNF,
        PER_ALL_ASSIGNMENTS_F    PAAF,
        PER_SCHEDULE_ASSIGNMENTS PSA,
        ZMM_SR_SCHEDULES_VL      ZSSV
    WHERE PAPF.PERSON_ID       = PPNF.PERSON_ID
      AND PAPF.PERSON_ID       = PAAF.PERSON_ID
      AND PAAF.LEGAL_ENTITY_ID = PSA.RESOURCE_ID      -- Legal Entity link
      AND PSA.SCHEDULE_ID      = ZSSV.SCHEDULE_ID
      AND PPNF.NAME_TYPE       = 'GLOBAL'
      AND PSA.PRIMARY_FLAG     = 'Y'
      AND PAAF.ASSIGNMENT_STATUS_TYPE  = 'ACTIVE'
      AND PAAF.ASSIGNMENT_TYPE         = 'E'
      AND PAAF.PRIMARY_ASSIGNMENT_FLAG = 'Y'
      AND PSA.RESOURCE_TYPE            = 'LEGALEMP'   -- Legal Entity type
      AND TRUNC(:P_DATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
      -- Exclude if assignment-level schedule exists (assignment overrides LE)
      AND NOT EXISTS (
          SELECT 1
          FROM PER_SCHEDULE_ASSIGNMENTS PSA2,
               ZMM_SR_AVAILABLE_DATES Z
          WHERE PAAF.ASSIGNMENT_ID = PSA2.RESOURCE_ID
            AND PSA2.PRIMARY_FLAG = 'Y'
            AND PSA2.RESOURCE_TYPE = 'ASSIGN'
            AND PSA2.SCHEDULE_ID = Z.SCHEDULE_ID
            AND TRUNC(Z.CALENDAR_DATE) = TRUNC(:P_DATE)
            AND TRUNC(Z.CALENDAR_DATE) BETWEEN TRUNC(PSA2.START_DATE) AND TRUNC(PSA2.END_DATE)
      )

    UNION ALL

    -- Part 2: Assignment Level Schedules (overrides Legal Entity)
    SELECT
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
        TRUNC(PSA.END_DATE) END_DATE
    FROM
        PER_ALL_PEOPLE_F          PAPF,
        PER_PERSON_NAMES_F       PPNF,
        PER_ALL_ASSIGNMENTS_F    PAAF,
        PER_SCHEDULE_ASSIGNMENTS PSA,
        ZMM_SR_SCHEDULES_VL      ZSSV
    WHERE PAPF.PERSON_ID      = PPNF.PERSON_ID
      AND PAPF.PERSON_ID      = PAAF.PERSON_ID
      AND PAAF.ASSIGNMENT_ID  = PSA.RESOURCE_ID       -- Assignment link
      AND PSA.SCHEDULE_ID     = ZSSV.SCHEDULE_ID
      AND PPNF.NAME_TYPE      = 'GLOBAL'
      AND PSA.PRIMARY_FLAG    = 'Y'
      AND PAAF.ASSIGNMENT_STATUS_TYPE  = 'ACTIVE'
      AND PAAF.ASSIGNMENT_TYPE         = 'E'
      AND PAAF.PRIMARY_ASSIGNMENT_FLAG = 'Y'
      AND PSA.RESOURCE_TYPE            = 'ASSIGN'     -- Assignment type
      AND TRUNC(:P_DATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
      -- CRITICAL: Period of Service Filter (add to both UNION parts)
      AND PPOS.DATE_START = (
          SELECT MAX(PPOS1.DATE_START)
          FROM PER_PERIODS_OF_SERVICE PPOS1
          WHERE PPOS1.PERSON_ID = PPOS.PERSON_ID
            AND PPOS1.DATE_START <= TRUNC(SYSDATE)
      )
)
,
-- CTE 2: Get working dates from calendar
WORKING_DATES AS (
    SELECT 
        SCHEDULE_ID,
        CALENDAR_DATE
    FROM ZMM_SR_AVAILABLE_DATES
    WHERE TRUNC(CALENDAR_DATE) BETWEEN :P_FROM_DATE AND :P_TO_DATE
)
,
-- CTE 3: Get time records with version control (CRITICAL: 3 levels)
TIME_REPORTED AS (
    SELECT 
        /*+ index(HTR, HWM_TM_REC_U1) */
        /*+ index(HTRGU, HWM_TM_REC_GRP_USAGES_U1) */
        /*+ index(DTRG, HWM_TM_REC_GRP_U1) */
        HTR.RESOURCE_ID,          -- PERSON_ID
        HTR.SUBRESOURCE_ID,       -- ASSIGNMENT_ID
        TRUNC(HTR.START_TIME) DAY_START_TIME,
        HTR.START_TIME,
        HTR.STOP_TIME,
        HTR.MEASURE               -- Hours
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
      -- Critical filters
      AND UPPER(HTR.UNIT_OF_MEASURE) IN ('UN', 'HR')
      AND DTRG1.GRP_TYPE_ID      = 100              -- Daily group
      AND HTR.LATEST_VERSION     = 'Y'
      AND HTR.DELETE_FLAG        IS NULL
      AND HTR.LAYER_CODE         = 'TIME_RPTD'      -- Reported time layer
      AND HTR.RESOURCE_TYPE      = 'PERSON'
      AND TRUNC(HTR.STOP_TIME)   IS NOT NULL
      -- Version control level 1: Latest version flag
      AND NVL(HTR.LATEST_VERSION,'Y') = 'Y'
      -- Version control level 2: Latest status version
      AND (HTDTUSV.TM_BLDG_BLK_VERSION, HTDTUSV.STATUS_ID) = (
          SELECT MAX(TM_BLDG_BLK_VERSION), MAX(STATUS_ID) 
          FROM HWM_TM_D_TM_UI_STATUS_V 
          WHERE HTDTUSV.TM_BLDG_BLK_ID = TM_BLDG_BLK_ID
      )
      -- Version control level 3: Latest creation date (handles edits)
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
            AND HTR1.LATEST_VERSION = 'Y'
            AND HTR1.DELETE_FLAG    IS NULL
            AND HTR1.LAYER_CODE     = 'TIME_RPTD'
            AND DTRG1.TM_REC_GRP_ID = DTRG.TM_REC_GRP_ID
            AND HTR1.RESOURCE_ID    = HTR.RESOURCE_ID
      )
      -- Date range filter
      AND TRUNC(HTR.START_TIME) BETWEEN :P_FROM_DATE AND :P_TO_DATE
)
,
-- CTE 4: Public Holidays (exclude from missing checks)
PUBLIC_HOLIDAY AS (
    SELECT DISTINCT 
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
      AND PAAF.LOCATION_ID        = HR.LOCATION_ID
      AND HR.COUNTRY              = PNR.PK1_VALUE
      AND PCE.CATEGORY            = 'PH'              -- Public Holiday
      AND PAAF.ASSIGNMENT_STATUS_TYPE  = 'ACTIVE'
      AND PAAF.ASSIGNMENT_TYPE         = 'E'
      AND PAAF.PRIMARY_ASSIGNMENT_FLAG = 'Y'
      AND TO_CHAR(PCE.START_DATE_TIME,'YYYY') = TO_CHAR(:P_FROM_DATE,'YYYY')
      AND TRUNC(:P_DATE) BETWEEN TRUNC(PAAF.EFFECTIVE_START_DATE) AND TRUNC(PAAF.EFFECTIVE_END_DATE)
)
-- Missing Timecard Detection (NOT EXISTS pattern)
SELECT
    PD.PERSON_NUMBER,
    PD.DISPLAY_NAME,
    WD.CALENDAR_DATE AS MISSING_DATE
FROM
    PERSON_DETAILS PD,
    WORKING_DATES WD
WHERE PD.SCHEDULE_ID = WD.SCHEDULE_ID
  -- No timecard exists for this date
  AND NOT EXISTS (
      SELECT 1
      FROM TIME_REPORTED TR
      WHERE TR.RESOURCE_ID = PD.PERSON_ID
        AND TR.SUBRESOURCE_ID = PD.ASSIGNMENT_ID
        AND TR.DAY_START_TIME = WD.CALENDAR_DATE
  )
  -- Exclude public holidays
  AND NOT EXISTS (
      SELECT 1
      FROM PUBLIC_HOLIDAY PH
      WHERE PH.PERSON_ID = PD.PERSON_ID
        AND WD.CALENDAR_DATE BETWEEN PH.START_DATE AND PH.END_DATE
  )
  -- Exclude approved absences
  AND NOT EXISTS (
      SELECT 1
      FROM ANC_PER_ABS_ENTRIES APAE
      WHERE APAE.PERSON_ID = PD.PERSON_ID
        AND APAE.ABSENCE_STATUS_CD <> 'ORA_WITHDRAWN'
        AND APAE.APPROVAL_STATUS_CD <> 'DENIED'
        AND WD.CALENDAR_DATE BETWEEN TRUNC(APAE.START_DATE) AND TRUNC(APAE.END_DATE)
  )
```

**Key Time and Labor Tables:**
| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `ZMM_SR_SCHEDULES_VL` | Schedule Master (Custom) | SCHEDULE_ID, SCHEDULE_NAME |
| `ZMM_SR_SCHEDULE_DTLS` | Shift Details (Custom) | SCHEDULE_ID, SHIFT_ID, START_DATE_TIME, END_DATE_TIME |
| `ZMM_SR_SHIFTS_VL` | Shift Definitions (Custom) | SHIFT_ID, SHIFT_NAME, DURATION_MS_NUM, DELETED_FLAG |
| `ZMM_SR_AVAILABLE_DATES` | Working Dates Calendar (Custom) | SCHEDULE_ID, CALENDAR_DATE |
| `PER_SCHEDULE_ASSIGNMENTS` | Employee-Schedule Link | RESOURCE_ID, SCHEDULE_ID, RESOURCE_TYPE, PRIMARY_FLAG |
| `HWM_TM_REC` | Time Records | RESOURCE_ID (Person), SUBRESOURCE_ID (Asg), START_TIME, MEASURE |
| `HWM_TM_REC_GRP` | Time Record Groups | TM_REC_GRP_ID, GRP_TYPE_ID |
| `HWM_TM_REC_GRP_USAGES` | Record-Group Link | TM_REC_ID, TM_REC_GRP_ID, TM_REC_VERSION |
| `HWM_TM_D_TM_UI_STATUS_V` | Timecard Status | TM_BLDG_BLK_ID, STATUS_VALUE |
| `PER_CALENDAR_EVENTS` | Public Holidays | CATEGORY='PH', START_DATE_TIME |

**Critical HWM_TM_REC Filters (ALWAYS Include):**
```sql
WHERE UPPER(HTR.UNIT_OF_MEASURE) IN ('UN', 'HR')  -- Units or Hours
  AND HTR.LATEST_VERSION     = 'Y'                -- Latest version
  AND HTR.DELETE_FLAG        IS NULL              -- Not deleted
  AND HTR.LAYER_CODE         = 'TIME_RPTD'        -- Reported time layer
  AND HTR.RESOURCE_TYPE      = 'PERSON'           -- Person records
  AND TRUNC(HTR.STOP_TIME)   IS NOT NULL          -- Completed entries
  AND DTRG1.GRP_TYPE_ID      = 100                -- Daily group type
  -- Exclude midnight system entries
  AND (EXTRACT(HOUR FROM HTR.START_TIME) <> 00
    OR EXTRACT(MINUTE FROM HTR.START_TIME) <> 00 
    OR EXTRACT(SECOND FROM HTR.START_TIME) <> 00)
```

**Period of Service Filtering Pattern (CRITICAL for Employee Data):**
```sql
-- RULE: Always filter by latest period of service to get current employment
-- Purpose: Gets only the latest hire/rehire record, excludes terminated/future

-- Pattern to include in PERSON_DETAILS/EMPLOYEE CTEs:
AND PPOS.DATE_START = (
    SELECT MAX(PPOS1.DATE_START)
    FROM PER_PERIODS_OF_SERVICE PPOS1
    WHERE PPOS1.PERSON_ID = PPOS.PERSON_ID
      AND PPOS1.DATE_START <= TRUNC(SYSDATE)
)
```

**Why Use Period of Service Filter:**
1. Ã¢Å“â€¦ **Gets latest hire/rehire date** - MAX(DATE_START) retrieves most recent employment
2. Ã¢Å“â€¦ **Handles rehires correctly** - If employee was terminated and rehired, gets new period
3. Ã¢Å“â€¦ **Excludes future records** - DATE_START <= SYSDATE excludes future-dated periods
4. Ã¢Å“â€¦ **Simpler than assignment logic** - One clear condition vs multiple assignment filters
5. Ã¢Å“â€¦ **Independent of parameter dates** - Not affected by report date range

**Comparison:**

| Ã¢ÂÅ’ Assignment Effective Date | Ã¢Å“â€¦ Period of Service Date |
|------------------------------|---------------------------|
| `PAAF.EFFECTIVE_START_DATE = MAX(...)` | `PPOS.DATE_START = MAX(...)` |
| Complex conditions (status, type, flag) | Simple: person_id + date filter |
| Dependent on parameter date range | Independent of parameters |
| May miss employees if dates don't overlap | Captures all current employees |
| Tracks assignment changes | Tracks employment periods |

**Example:**
```
Employee hired: 01-JAN-2020
Terminated: 31-DEC-2022
Rehired: 01-JUN-2023

Period of Service Records:
- DATE_START = 01-JAN-2020 (terminated)
- DATE_START = 01-JUN-2023 (current) Ã¢Å“â€¦ This one is selected
```

**Note:** Always join PER_PERIODS_OF_SERVICE to PER_ALL_ASSIGNMENTS_F via `PERIOD_OF_SERVICE_ID` when using this pattern.

**'ALL' Parameter Bypass Pattern:**
```sql
-- Allow 'ALL' to bypass parameter filtering
AND (PAPF.PERSON_ID IN (:P_PERSON_ID) OR 'ALL' IN (:P_PERSON_ID || 'ALL'))
AND (HDORG.NAME IN (:P_DEPARTMENT) OR 'ALL' IN (:P_DEPARTMENT || 'ALL'))
```

**Ã¢Å¡Â¡ CRITICAL PERFORMANCE OPTIMIZATION: Apply PERSON_ID Filter in ALL CTEs**

**Rule:** When using PERSON_ID (or RESOURCE_ID) parameters, apply the filter in EVERY CTE, not just the final SELECT.

**Why:** Filtering early in each CTE dramatically reduces dataset size before complex operations like JOINs, version control checks, and aggregations.

**Performance Impact:**
- **10x-100x faster** when filtering by specific PERSON_ID(s)
- Reduced memory usage (smaller CTEs)
- Better index utilization
- Faster CTE materialization

**Pattern - Apply in ALL CTEs:**
```sql
-- PARAMETERS CTE
PARAMETERS AS (
    SELECT
        TRUNC(:P_FROM_DATE) FROM_DATE,
        TRUNC(:P_TO_DATE) TO_DATE,
        NVL(:P_PERSON_ID, 'ALL') EMP_FILTER
    FROM DUAL
),

-- Apply filter in PERSON CTE
PERSON_DETAILS AS (
    SELECT ...
    FROM PER_ALL_PEOPLE_F PAPF, ...
    WHERE ...
      -- Apply EMP_FILTER here
      AND (PAPF.PERSON_ID IN (P.EMP_FILTER) OR P.EMP_FILTER = 'ALL')
),

-- Apply filter in TIME_RECORDS CTE
TIME_RECORDS AS (
    SELECT ...
    FROM HWM_TM_REC HTR, ...
    WHERE ...
      -- Apply EMP_FILTER here (RESOURCE_ID = PERSON_ID)
      AND (HTR.RESOURCE_ID IN (P.EMP_FILTER) OR P.EMP_FILTER = 'ALL')
),

-- Apply filter in PUBLIC_HOLIDAY CTE
PUBLIC_HOLIDAY AS (
    SELECT ...
    FROM PER_ALL_PEOPLE_F PAPF, ...
    WHERE ...
      -- Apply EMP_FILTER here
      AND (PAPF.PERSON_ID IN (P.EMP_FILTER) OR P.EMP_FILTER = 'ALL')
),

-- Apply filter in ABSENCE_DATES CTE
ABSENCE_DATES AS (
    SELECT ...
    FROM ANC_PER_ABS_ENTRIES APAE, ...
    WHERE ...
      -- Apply EMP_FILTER here
      AND (APAE.PERSON_ID IN (P.EMP_FILTER) OR P.EMP_FILTER = 'ALL')
)
```

**Before vs After:**
```
Ã¢ÂÅ’ BEFORE (Filter only in Final SELECT):
   - Retrieve ALL 10,000 employees
   - Retrieve ALL 500,000 time records
   - Retrieve ALL 50,000 holidays
   - Process massive JOINs
   - Filter to 1 employee at end

Ã¢Å“â€¦ AFTER (Filter in ALL CTEs):
   - Retrieve 1 employee only
   - Retrieve 50 time records for that employee
   - Retrieve 5 holidays for that employee
   - Process minimal JOINs
   - Already filtered throughout
```

**Key Tables to Apply Filter:**
- `PER_ALL_PEOPLE_F` - Filter by `PERSON_ID`
- `HWM_TM_REC` - Filter by `RESOURCE_ID` (equals PERSON_ID)
- `ANC_PER_ABS_ENTRIES` - Filter by `PERSON_ID`
- `PER_CALENDAR_EVENTS` - Join through filtered PERSON_ID
- Any CTE working with employee data

**Note:** This optimization is especially critical for:
- Time and Labor reports (large HWM tables)
- Real-time/ad-hoc queries
- Single employee lookups
- Manager self-service reports

**Reference:**
- HCM/TIME_LABOR/TL_MASTER.md Section 9 for complete production patterns
- HCM/TIME_LABOR/TL_REPOSITORIES.md for Time and Labor CTEs
- HCM/TIME_LABOR/TL_TEMPLATES.md for ready-to-use query templates

---

## 3. Ã°Å¸â€œÂ SQL Coding Standards

### A. Query Termination
*   **Rule:** SQL queries should **NOT** end with a semicolon (;)
*   **Reason:** Compatibility with Oracle BI Publisher and OTBI reporting tools
*   **Example:**
    ```sql
    SELECT * FROM PER_ALL_PEOPLE_F
    WHERE PERSON_ID = 123
    ORDER BY PERSON_NUMBER
    -- NO semicolon here
    ```

### B. Join Syntax (Oracle Old-Style Joins REQUIRED)
*   **Rule:** ALWAYS use Oracle old-style joins (comma-separated tables with WHERE clause)
*   **Never Use:** ANSI JOIN syntax (INNER JOIN, LEFT JOIN, RIGHT JOIN, LEFT OUTER JOIN, RIGHT OUTER JOIN, FULL OUTER JOIN)
*   **Reason:** Standard practice for Oracle Fusion HCM queries, better compatibility, mandatory for all HCM SQL development
*   **CRITICAL:** This is a STRICT requirement - ANY use of ANSI JOIN syntax (JOIN, LEFT JOIN, etc.) is prohibited
*   **Examples:**
    
    **Inner Join (Required table):**
    ```sql
    FROM TABLE1 T1, TABLE2 T2
    WHERE T1.ID = T2.ID
    ```
    
    **Outer Join (Optional table):**
    ```sql
    FROM TABLE1 T1, TABLE2 T2
    WHERE T1.ID = T2.ID(+)  -- (+) on optional side
    ```
    
    **Multiple Joins:**
    ```sql
    FROM PERSON_BASE PB,
         EMPLOYMENT_INFO EI,
         ASSIGNMENT_BASE AB,
         COMPENSATION COMP
    WHERE PB.PERSON_ID = EI.PERSON_ID          -- Inner join
      AND PB.PERSON_ID = AB.PERSON_ID(+)       -- Left join (AB optional)
      AND PB.PERSON_ID = COMP.PERSON_ID(+)     -- Left join (COMP optional)
    ```
    
    **WRONG Examples (NEVER USE):**
    ```sql
    -- ❌ WRONG - ANSI JOIN syntax
    FROM TABLE1 T1
    INNER JOIN TABLE2 T2 ON T1.ID = T2.ID
    
    -- ❌ WRONG - LEFT JOIN syntax
    FROM TABLE1 T1
    LEFT JOIN TABLE2 T2 ON T1.ID = T2.ID
    
    -- ❌ WRONG - LEFT OUTER JOIN syntax
    FROM TABLE1 T1
    LEFT OUTER JOIN TABLE2 T2 ON T1.ID = T2.ID
    
    -- ✅ CORRECT - Oracle old-style
    FROM TABLE1 T1, TABLE2 T2
    WHERE T1.ID = T2.ID(+)
    ```

### C. CTE Optimization
*   **Rule:** Use `/*+ qb_name(NAME) MATERIALIZE */` hints for all CTEs
*   **Reason:** Improves performance by caching intermediate results

### D. Date Formatting
*   **Standard Format:** `DD-MON-YYYY` for date outputs
*   **Example:** `TO_CHAR(DATE_COLUMN, 'DD-MON-YYYY')`
*   **ISO 8601:** Use `YYYY-MM-DDTHH24:MI:SS.000+00:00` when ISO format required

### D.1 Hardcoded String Values (CRITICAL)
*   **Rule:** Always use **Initcap** (proper case) for hardcoded string values
*   **Standard:** First letter uppercase, rest lowercase
*   **Examples:**
    - ✅ CORRECT: `'Yes'`, `'No'`, `'Active'`, `'Inactive'`
    - ❌ WRONG: `'yes'`, `'no'`, `'YES'`, `'NO'`, `'active'`, `'ACTIVE'`

**Pattern:**
```sql
-- ✅ CORRECT - Initcap format
CASE WHEN FLAG = 'Y' THEN 'Yes' ELSE 'No' END AS IS_ACTIVE
CASE WHEN STATUS = 'A' THEN 'Active' ELSE 'Inactive' END AS STATUS_DESC

-- ❌ WRONG - Lowercase
CASE WHEN FLAG = 'Y' THEN 'yes' ELSE 'no' END AS IS_ACTIVE

-- ❌ WRONG - Uppercase
CASE WHEN FLAG = 'Y' THEN 'YES' ELSE 'NO' END AS IS_ACTIVE
```

### E. NULL Handling
*   **Numeric Fields:** Use `NVL(column, 0)` for numeric calculations
*   **Text Fields:** Use outer joins `(+)` for optional relationships
*   **Rounding:** Always `ROUND(amount, 2)` for monetary values

### E.1 ORA-01719: Outer Join with OR/IN (CRITICAL)
*   **Error:** `ORA-01719: outer join operator (+) not allowed in operand of OR or IN`
*   **Cause:** Using outer join operator (+) in combination with OR or IN clauses
*   **Solution:** Put NULL check FIRST, then date filter WITHOUT (+)

**❌ WRONG Pattern (Causes ORA-01719):**
```sql
-- Outer join operator with OR - NOT ALLOWED
WHERE TRUNC(SYSDATE) BETWEEN PJ.EFFECTIVE_START_DATE(+) AND PJ.EFFECTIVE_END_DATE(+) 
   OR PJ.JOB_ID IS NULL
```

**✅ CORRECT Pattern:**
```sql
-- NULL check first, then date filter without (+)
WHERE (PJ.JOB_ID IS NULL OR TRUNC(SYSDATE) BETWEEN PJ.EFFECTIVE_START_DATE AND PJ.EFFECTIVE_END_DATE)
```

**Explanation:**
- When NULL check comes first, Oracle evaluates it before the date comparison
- If `JOB_ID IS NULL`, the OR short-circuits and skips the date check
- If `JOB_ID` exists, the date filter runs without outer join operator
- This avoids the outer join conflict with OR

**Common Scenarios:**
```sql
-- For optional date-tracked tables (Job, Grade, Position, Payroll)
AND (PJ.JOB_ID IS NULL OR TRUNC(SYSDATE) BETWEEN PJ.EFFECTIVE_START_DATE AND PJ.EFFECTIVE_END_DATE)
AND (PG.GRADE_ID IS NULL OR TRUNC(SYSDATE) BETWEEN PG.EFFECTIVE_START_DATE AND PG.EFFECTIVE_END_DATE)
AND (POS.POSITION_ID IS NULL OR TRUNC(SYSDATE) BETWEEN POS.EFFECTIVE_START_DATE AND POS.EFFECTIVE_END_DATE)
AND (PP.PAYROLL_ID IS NULL OR TRUNC(SYSDATE) BETWEEN PP.EFFECTIVE_START_DATE AND PP.EFFECTIVE_END_DATE)
```

### F. Performance Best Practices
*   Use `_M` tables (e.g., `PER_ALL_ASSIGNMENTS_M`) for current snapshot queries
*   Filter with `EFFECTIVE_LATEST_CHANGE = 'Y'` when using `_M` tables
*   Apply `PRIMARY_FLAG = 'Y'` for primary assignments
*   Use `CAST(column AS NUMBER DEFAULT 9999999999 ON CONVERSION ERROR)` for safe sorting

### G. Duplicate Prevention
*   **Rule:** Use subqueries with `ROWNUM = 1` for one-to-many relationships
*   **Common Causes:** Payroll assignments, email addresses, national identifiers
*   **Pattern:**
    ```sql
    -- BAD: Direct join can cause duplicates
    FROM PER_ALL_PEOPLE_F PAPF,
         PAY_ALL_PAYROLLS_F PAP
    WHERE PAPF.PERSON_ID = PAP.PERSON_ID(+)  -- Person can have multiple payrolls!
    
    -- GOOD: Subquery with ROWNUM = 1
    SELECT
        PAPF.PERSON_ID,
        (SELECT PAP.PAYROLL_NAME
         FROM PAY_ALL_PAYROLLS_F PAP
         WHERE PAP.PERSON_ID = PAPF.PERSON_ID
         AND ROWNUM = 1
        ) AS PAYROLL
    FROM PER_ALL_PEOPLE_F PAPF
    ```

### H. Column Naming Convention
*   **Rule:** Output column aliases should be UPPERCASE with UNDERSCORES (no spaces)
*   **Reason:** Consistency, easier to reference in downstream applications, no need for double quotes
*   **Examples:**
    ```sql
    -- Ã¢ÂÅ’ WRONG: Using quoted strings with spaces and mixed case
    SELECT
        PAPF.PERSON_NUMBER AS "Person Number",
        PPNF.FULL_NAME AS "Employee Name",
        PAAF.ASSIGNMENT_NUMBER AS "Assignment Number"
    FROM ...
    
    -- Ã¢Å“â€¦ CORRECT: Using uppercase with underscores
    SELECT
        PAPF.PERSON_NUMBER AS PERSON_NUMBER,
        PPNF.FULL_NAME AS EMPLOYEE_NAME,
        PAAF.ASSIGNMENT_NUMBER AS ASSIGNMENT_NUMBER
    FROM ...
    ```

---

## 4. Ã°Å¸â€œâ€¹ Quick Reference - Common Field Locations

| Field Name | Correct Table | Key Points |
|------------|---------------|------------|
| **PERSON_ID** | PER_ALL_PEOPLE_F | Primary key for person |
| **PERSON_NUMBER** | PER_ALL_PEOPLE_F | Unique employee identifier |
| **PERSON_TYPE_ID** | PER_ALL_ASSIGNMENTS_F | Ã¢ÂÅ’ NOT in PER_ALL_PEOPLE_F |
| **DATE_OF_BIRTH** | PER_PERSONS | Ã¢ÂÅ’ NOT in PER_ALL_PEOPLE_F, NOT date-tracked |
| **SEX (Gender)** | PER_PEOPLE_LEGISLATIVE_F | Ã¢ÂÅ’ NOT in PER_ALL_PEOPLE_F |
| **MARITAL_STATUS** | PER_PEOPLE_LEGISLATIVE_F | Ã¢ÂÅ’ NOT in PER_ALL_PEOPLE_F |
| **EMAIL_ADDRESS** | PER_EMAIL_ADDRESSES | Filter: EMAIL_TYPE='W1' for work email |
|| **PHONE_NUMBER (Primary)** | PER_PHONES | Join: PH.PHONE_ID = PAPF.PRIMARY_PHONE_ID (Ã¢ÂÅ’ NOT PERSON_ID) |
| **MANAGER_ID** | PER_ASSIGNMENT_SUPERVISORS_F | âŒ NOT in PER_ALL_ASSIGNMENTS_F, Date-tracked (_F) |
| **EMPLOYEE_CATEGORY** | PER_ALL_ASSIGNMENTS_F | **ALWAYS** use PAAF.EMPLOYEE_CATEGORY for worker category (NOT ASSIGNMENT_CATEGORY) - Decode with lookup EMPLOYEE_CATG |
| **FULL_NAME** | PER_PERSON_NAMES_F | Filter: NAME_TYPE='GLOBAL' |
| **ASSIGNMENT_STATUS** | PER_ASSIGNMENT_STATUS_TYPES_TL | Via ASSIGNMENT_STATUS_TYPE_ID |
| **DEPARTMENT** | PER_DEPARTMENTS | Join on ORGANIZATION_ID |
| **JOB (Job Title)** | PER_JOBS_F_VL | .NAME column, Date-tracked, Use _F_VL view (not _F + _TL) |
| **DESIGNATION (Position)** | HR_ALL_POSITIONS_F_TL | .NAME column, Date-tracked, LANGUAGE='US' |
| **GRADE** | PER_GRADES_F_VL | .NAME column, Date-tracked, Use _F_VL view (not _F + _TL) |
| **LOCATION** | PER_LOCATION_DETAILS_F_VL | Date-tracked |
| **PAYROLL_NAME** | PAY_ALL_PAYROLLS_F | Via relationship tables |
| **NATIONALITY** | PER_CITIZENSHIPS | Use subquery with FND_TERRITORIES_VL, filter: CITIZENSHIP_STATUS='A' |
| **EMIRATES_ID** | PER_NATIONAL_IDENTIFIERS | Join on PRIMARY_NID_ID or PERSON_ID |
| **RELIGION** | PER_RELIGIONS | Use subquery with FND_LOOKUP_VALUES |
| **HIRE_DATE** | PER_PERIODS_OF_SERVICE | DATE_START column |
| **TERMINATION_DATE** | PER_PERIODS_OF_SERVICE | ACTUAL_TERMINATION_DATE |
|| **LEAVING_REASON** | PER_ACTION_REASONS_TL | Ã¢Å’ NOT in PER_PERIODS_OF_SERVICE - via PAAF Ã¢â€ ' PAR_B Ã¢â€ ' PAR_TL |
| **PROBATION_PERIOD** | PER_ALL_ASSIGNMENTS_F | DATE_PROBATION_END, PROBATION_PERIOD, PROBATION_UNIT (standard columns) |
| **NOTICE_PERIOD** | PER_ALL_ASSIGNMENTS_F | NOTICE_PERIOD, NOTICE_PERIOD_UOM (standard columns) |
| **RECRUITER_ID (ORC)** | IRC_REQUISITIONS_VL | Ã¢ÂÅ’ NOT in IRC_OFFERS - assigned at requisition level |
| **RECRUITER_NAME (ORC)** | PER_PERSON_NAMES_F | Join via IRC_REQUISITIONS_VL.RECRUITER_ID |

---

## 5. Ã¢Å¡Â Ã¯Â¸Â Common Errors to Avoid

| Ã¢ÂÅ’ WRONG | Ã¢Å“â€¦ CORRECT | Why |
|---------|-----------|-----|
| `SELECT * FROM ...;` | `SELECT * FROM ...` | No semicolon for BI Publisher/OTBI |
| `INNER JOIN TABLE2 ON ...` | `FROM TABLE1, TABLE2 WHERE ...` | Use Oracle old-style joins |
| Direct join to payroll tables | Use subquery with `ROWNUM = 1` | Prevents duplicate rows |
| `PAPF.DATE_OF_BIRTH` | `PS.DATE_OF_BIRTH` | Wrong table |
| `PAPF.EMAIL_ADDRESS` | `PEA.EMAIL_ADDRESS` | Wrong table |
|| `PH.PERSON_ID = PAPF.PERSON_ID` (phone) | `PH.PHONE_ID = PAPF.PRIMARY_PHONE_ID` | Use PRIMARY_PHONE_ID, not PERSON_ID |
| `PAPF.PERSON_TYPE_ID` | `PAAF.PERSON_TYPE_ID` | Wrong table |
|| `PPOS.LEAVING_REASON` | Via `PER_ACTION_REASONS_TL` subquery | Column doesn't exist in PER_PERIODS_OF_SERVICE |
| `PAAF.MANAGER_ID` | `PASF.MANAGER_ID` (PER_ASSIGNMENT_SUPERVISORS_F) | Column doesn't exist in PER_ALL_ASSIGNMENTS_F |
| `PAAF.ASSIGNMENT_CATEGORY` (for worker category) | `PAAF.EMPLOYEE_CATEGORY` | **ALWAYS** use PAAF.EMPLOYEE_CATEGORY for worker category (decode with EMPLOYEE_CATG lookup) |
| `JOB.DESCRIPTION` for designation | `POS.NAME` (HR_ALL_POSITIONS_F_TL) | Use Position, not Job description |
| Joining `PER_JOBS_F` + `PER_JOBS_F_TL` separately | Use `PER_JOBS_F_VL` (single view) | _F_VL views combine _F and _TL, simpler and faster |
| Joining `PER_GRADES_F` + `PER_GRADES_F_TL` separately | Use `PER_GRADES_F_VL` (single view) | _F_VL views combine _F and _TL, simpler and faster |
| `PC.PRIMARY_FLAG = 'Y'` | `PC.CITIZENSHIP_STATUS = 'A'` | PRIMARY_FLAG column doesn't exist |
| `(SYSDATE BETWEEN COL(+) AND COL(+) OR ID IS NULL)` | `(ID IS NULL OR SYSDATE BETWEEN COL AND COL)` | ORA-01719: Outer join (+) not allowed with OR - put NULL check first |
| `PPOS.PROBATION_PERIOD` (or using DFF) | `PAAF.DATE_PROBATION_END`, `PAAF.NOTICE_PERIOD` | Standard columns exist in PER_ALL_ASSIGNMENTS_F |
| `OFFER.RECRUITER_ID` (ORC) | `REQ.RECRUITER_ID` via IRC_REQUISITIONS_VL | Recruiter assigned at requisition level, not offer |
| `APAE.ABSENCE_ID` (Absence) | Use `PERSON_ID` + `ABSENCE_TYPE_ID` + `START_DATE` | ABSENCE_ID column doesn't exist |
|| `APAE.PLAN_ID` (Absence) | Link via `PERSON_ID` + `PRD_OF_SVC_ID` | PLAN_ID doesn't exist in ANC_PER_ABS_ENTRIES |
|| Absence overlap filter | `START_DATE >= FROM_DATE AND END_DATE <= TO_DATE` | Use within-range filter, not overlap |
| `TRUNC(SYSDATE)` everywhere | `:P_EFFECTIVE_DATE` | Historical accuracy |
|| `COL(+) BETWEEN TAB1.COL1(+) AND TAB1.COL2(+)` | `(COL IS NULL OR COL BETWEEN TAB1.COL1 AND TAB1.COL2)` | ORA-01468: Multiple outer joins in predicate |
| Missing `MATERIALIZE` | `/*+ MATERIALIZE */` | Performance |
| `TO_CHAR(date, 'MM/DD/YYYY')` | `TO_CHAR(date, 'DD-MON-YYYY')` | Standard format |

---

**Last Updated:** 09-Feb-2026  
**Version:** 3.0 (Merged HCM_PRODUCTION_PATTERNS, HCM_QUERY_ANALYSIS, CRITICAL_PERSON_TYPE_FIX, HCM_CROSS_MODULE_PATTERNS)  
**Status:** Production Standards Active

---

## 6. 📊 HCM PRODUCTION PATTERNS & TABLE INVENTORY

**Source:** Analysis of 13+ Production Queries (5,880+ lines)  
**Purpose:** Comprehensive pattern extraction for complete Knowledge Base coverage

### 6.1 Complete Table Inventory

#### Core HR Tables (from production queries)

| Table | Alias | Usage | Purpose |
|-------|-------|-------|---------|
| **PER_ALL_PEOPLE_F** | PAPF | All queries | Person master |
| **PER_PERSON_NAMES_F** | PPNF | All queries | Person names |
| **PER_ALL_ASSIGNMENTS_F** | PAAF | Most queries | Assignments (date-track) |
| **PER_ALL_ASSIGNMENTS_M** | PAAM | Several | Assignments (managed) |
| **PER_PERSON_TYPES_TL** | PPTT/PPTTL | Most | Person type translations |
| **PER_PERSON_TYPES_VL** | PPTV | Several | Person types view |
| **PER_PEOPLE_LEGISLATIVE_F** | PPLF | Most | Legislative data (gender, marital) |
| **PER_PERSONS** | PP/PS | Several | Person core (DOB) |
| **PER_NATIONAL_IDENTIFIERS** | PNI | Several | National IDs |
| **PER_PASSPORTS** | PP | Several | Passport details |
| **PER_CITIZENSHIPS** | PC | Most | Citizenship/nationality |
| **PER_RELIGIONS** | PR | Several | Religion |
| **PER_EMAIL_ADDRESSES** | PEA | Several | Email addresses |
| **PER_PHONES** | PH | Few | Phone numbers |
| **PER_ADDRESSES_F** | PAF | Few | Addresses |
| **PER_PEOPLE_GROUPS** | PPG | Several | People groups |
| **PER_DEPARTMENTS** | PD | Most | Department master |
| **PER_GRADES** | PG | Most | Grade master |
| **PER_GRADES_F_TL** | PGFT | Few | Grade translations |
| **PER_GRADES_F_VL** | - | Several | Grade view |
| **PER_JOBS** | PJ | Few | Job master |
| **PER_JOBS_F_TL** | - | Few | Job translations |
| **PER_JOBS_F_VL** | PJFV | Most | Job view |
| **PER_LOCATION_DETAILS_F_VL** | PL/PLDTL | Most | Location details |
| **PER_PERIODS_OF_SERVICE** | PPOS | Several | Period of service |
| **PER_ACTION_OCCURRENCES** | PAC | Few | Action occurrences |
| **PER_ACTIONS_VL** | ACTN | Few | Actions master |
| **PER_ACTION_REASONS_TL** | PART | Few | Action reasons |
| **PER_ASSIGNMENT_SUPERVISORS_F** | PASF | Several | Supervisor hierarchy |
| **PER_ASSIGNMENT_STATUS_TYPES_VL** | PASTV/PAST | Several | Assignment status |
| **PER_CONTACT_RELSHIPS_F** | PCRF | Several | Contact relationships |

#### Payroll Tables

| Table | Alias | Usage | Purpose |
|-------|-------|-------|---------|
| **PAY_PAYROLL_ACTIONS** | PPA | Payroll queries | Payroll actions |
| **PAY_PAYROLL_REL_ACTIONS** | PPRA | Payroll queries | Payroll relationship actions |
| **PAY_PAY_RELATIONSHIPS_DN** | PPRD | Payroll queries | Pay relationships |
| **PAY_RUN_RESULTS** | PRR | Payroll queries | Run results |
| **PAY_RUN_RESULT_VALUES** | PRRV | Payroll queries | Run result values |
| **PAY_ELEMENT_TYPES_F** | PETF | Payroll queries | Element types |
| **PAY_ELEMENT_TYPES_TL** | PETT | Dynamic payroll | Element type translations |
| **PAY_INPUT_VALUES_F** | PIVF | Payroll queries | Input values |
| **PAY_ELEMENT_ENTRIES_F** | PEEF | Hardcoded payroll | Element entries |
| **PAY_ELEMENT_ENTRIES_VL** | PEEV | Compensation | Element entries view |
| **PAY_ELEMENT_ENTRY_VALUES_F** | PEEV/PEEVF | Hardcoded/Comp | Element entry values |
| **PAY_ELE_CLASSIFICATIONS** | PEC | Payroll queries | Element classifications |
| **PAY_ALL_PAYROLLS_F** | PAP/PAPP | Most payroll | Payroll master |
| **PAY_TIME_PERIODS** | PTP | Payroll queries | Time periods |
| **PAY_CONSOLIDATION_SETS** | PCS | Dynamic payroll | Consolidation sets |
| **PAY_REQUESTS** | PRQ | Dynamic payroll | Pay requests |
| **PAY_FLOW_INSTANCES** | PFI | Dynamic payroll | Flow instances |
| **PAY_PERSONAL_PAYMENT_METHODS_F** | PPPMF | Payslip/Banking | Personal payment methods |
| **PAY_BANK_ACCOUNTS** | PBA | Payslip/Banking | Bank accounts |
| **PAY_ORG_PAY_METHODS_VL** | POPM | Payslip | Org payment methods |
| **PAY_PAYMENT_TYPES_VL** | PPT | Payslip | Payment types |
| **PAY_ASSIGNED_PAYROLLS_DN** | AP/PAPD | Payroll | Assigned payrolls |
| **PAY_REL_GROUPS_DN** | PRG | Emp Master | Relationship groups |
| **PAY_COST_ALLOCATIONS_F** | CCPCA | New Joiner | Cost allocations |
| **PAY_COST_ALLOC_ACCOUNTS** | CCPCAA | New Joiner | Cost alloc accounts |

#### Compensation Tables

| Table | Alias | Usage | Purpose |
|-------|-------|-------|---------|
| **CMP_SALARY_SIMPLE_COMPNTS** | CSSC | Emp Master | Salary simple components |
| **CMP_ATTRIBUTE_ELEMENTS** | CAE | Compensation | Attribute elements |
| **CMP_PLAN_ATTRIBUTES** | CPA | Compensation | Plan attributes |
| **CMP_PLANS_VL** | CPVL | Compensation | Compensation plans |
| **CMP_COMPONENTS_VL** | CCVL | Compensation | Compensation components |
| **CMP_ASG_SALARY_RATE_COMPTS_V** | CASR | Compensation | Assignment salary rate components |

#### Time and Labor Tables

| Table | Alias | Usage | Purpose |
|-------|-------|-------|---------|
| **HWM_TM_REC** | HTR | Time reports | Time records |
| **HWM_TM_REC_GRP** | DTRG/DTRG1 | Time reports | Time record groups |
| **HWM_TM_REC_GRP_USAGES** | HTRGU | Time reports | Record-group link |
| **HWM_TM_D_TM_UI_STATUS_V** | HTDTUSV | Time reports | Timecard status |
| **HWM_TM_REC_GRP_SUM_V** | TMH | Missing Timesheet | Timesheet summary view |
| **HWM_TM_REP_ATRB_USAGES** | AUSG | Timesheet | Time reporting attribute usages |
| **HWM_TM_REP_ATRBS** | ATR | Timesheet | Time reporting attributes |
| **HWM_TM_REP_M_PTT_ATRBS_V** | HTRMPTTAV | Overtime | Payroll time type attributes |
| **PER_SCHEDULE_ASSIGNMENTS** | PSA | Missing In/Out | Schedule assignments |
| **PER_SCHEDULE_EXCEPTIONS** | PSE | Missing Timesheet | Schedule exceptions |
| **ZMM_SR_SCHEDULES_VL** | ZSSV/ZSSV1 | Missing In/Out | Schedule view (Custom) |
| **ZMM_SR_AVAILABLE_DATES** | Z | Missing In/Out | Available dates (Custom) |
| **ZMM_SR_SHIFTS_VL** | ZSSV | Shift reports | Shift definitions (Custom) |
| **ZMM_SR_SCHEDULE_DTLS** | - | Shift reports | Shift details (Custom) |

#### Absence Tables

| Table | Alias | Usage | Purpose |
|-------|-------|-------|---------|
| **ANC_PER_ACCRUAL_ENTRIES** | APAE/APACE | Accrual | Accrual entries |
| **ANC_ABSENCE_PLANS_VL** | AAPV | Accrual | Absence plans |
| **ANC_PER_ABS_ENTRIES** | APAE | Absence | Absence entries |
| **ANC_ABSENCE_TYPES_F_TL** | AATFT | Absence | Absence type translations |
| **ANC_ABSENCE_REASONS_F** | AARF | Absence | Absence reasons |
| **ANC_PER_PLAN_ENROLLMENT** | APPE | Balance | Plan enrollment |
| **ANC_PER_ACRL_ENTRY_DTLS** | APACD | Adjustments | Accrual entry details |

#### Workflow Tables

| Table | Alias | Usage | Purpose |
|-------|-------|-------|---------|
| **FA_FUSION_SOAINFRA.WFTASK** | WF | Workflow | Workflow tasks |
| **HRC_TXN_HEADER** | TXNH | Transactions | Transaction header |
| **HRC_TXN_DATA** | TXND | Transactions | Transaction data |

### 6.2 Critical Pattern: Managed vs Date-Tracked Tables

**Use `_M` (Managed) When:**
- Only need current/latest record
- Using `EFFECTIVE_LATEST_CHANGE = 'Y'`
- Performance critical

**Use `_F` (Date-Tracked) When:**
- Need historical records
- Querying specific date range
- Need full audit trail

**Pattern:**
```sql
-- Managed Table Pattern
FROM PER_ALL_ASSIGNMENTS_M PAAM
WHERE PAAM.EFFECTIVE_LATEST_CHANGE = 'Y'
  AND PAAM.PRIMARY_FLAG = 'Y'
  AND PAAM.ASSIGNMENT_TYPE = 'E'

-- Date-Tracked Pattern
FROM PER_ALL_ASSIGNMENTS_F PAAF
WHERE TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
  AND PAAF.PRIMARY_FLAG = 'Y'
  AND PAAF.ASSIGNMENT_TYPE = 'E'
```

### 6.3 Element Entry Pattern (Payroll Elements)

**Standard Pattern for Getting Element Values:**
```sql
SELECT SUM(PEEV.SCREEN_ENTRY_VALUE)
FROM PAY_ELEMENT_TYPES_VL PETF,
     PAY_INPUT_VALUES_VL PIVF,
     PAY_ELEMENT_ENTRIES_F PEEF,
     PAY_ELEMENT_ENTRY_VALUES_F PEEVF
WHERE PETF.ELEMENT_TYPE_ID = PIVF.ELEMENT_TYPE_ID
AND PETF.ELEMENT_TYPE_ID = PEEF.ELEMENT_TYPE_ID
AND PEEF.ELEMENT_ENTRY_ID = PEEVF.ELEMENT_ENTRY_ID
AND PIVF.INPUT_VALUE_ID = PEEVF.INPUT_VALUE_ID
AND PETF.REPORTING_NAME = 'Basic Salary'  -- Element name
AND PIVF.NAME = 'Amount'                   -- Input value name
AND [DATE] BETWEEN PETF.EFFECTIVE_START_DATE AND PETF.EFFECTIVE_END_DATE
AND [DATE] BETWEEN PIVF.EFFECTIVE_START_DATE AND PIVF.EFFECTIVE_END_DATE
AND [DATE] BETWEEN PEEF.EFFECTIVE_START_DATE AND PEEF.EFFECTIVE_END_DATE
AND [DATE] BETWEEN PEEVF.EFFECTIVE_START_DATE AND PEEVF.EFFECTIVE_END_DATE
AND PEEF.PERSON_ID = [PERSON_ID]
```

### 6.4 Payroll Run Results Pattern

**Standard Pattern for Getting Payroll Results:**
```sql
SELECT SUM(TO_NUMBER(PRRV.RESULT_VALUE))
FROM PAY_RUN_RESULT_VALUES PRRV,
     PAY_RUN_RESULTS PRR,
     PAY_PAYROLL_REL_ACTIONS PPRA,
     PAY_PAYROLL_ACTIONS PPA,
     PAY_TIME_PERIODS PTP,
     PAY_ELEMENT_TYPES_F PETF,
     PAY_INPUT_VALUES_F PIVF
WHERE PRRV.RUN_RESULT_ID = PRR.RUN_RESULT_ID
AND PRR.PAYROLL_REL_ACTION_ID = PPRA.PAYROLL_REL_ACTION_ID
AND PPRA.PAYROLL_ACTION_ID = PPA.PAYROLL_ACTION_ID
AND PPA.ACTION_TYPE IN ('Q', 'R')      -- QuickPay or Regular
AND PPA.ACTION_STATUS = 'C'            -- Complete
AND PPA.EARN_TIME_PERIOD_ID = PTP.TIME_PERIOD_ID
AND PETF.ELEMENT_TYPE_ID = PRR.ELEMENT_TYPE_ID
AND PIVF.INPUT_VALUE_ID = PRRV.INPUT_VALUE_ID
AND PPRA.RETRO_COMPONENT_ID IS NULL    -- Exclude retro
```

**Key Filters:**
- `ACTION_TYPE IN ('Q', 'R')` - QuickPay or Regular payroll
- `ACTION_STATUS = 'C'` - Complete status only
- `RETRO_COMPONENT_ID IS NULL` - Exclude retro adjustments

### 6.5 Compensation Component Pattern

**Getting Compensation Components:**
```sql
SELECT CSSC.COMPONENT_CODE,
       CSSC.AMOUNT,
       SUBSTR(CSSC.COMPONENT_CODE, 5, 50) COMPONENT_CODE1  -- Remove 'ORA_' prefix
FROM CMP_SALARY_SIMPLE_COMPNTS CSSC
WHERE CSSC.PERSON_ID = [PERSON_ID]
AND TRUNC(SYSDATE) BETWEEN TRUNC(CSSC.SALARY_DATE_FROM) AND TRUNC(CSSC.SALARY_DATE_TO)
AND TRUNC(CSSC.LAST_UPDATE_DATE) = (
    SELECT MAX(TRUNC(LAST_UPDATE_DATE))
    FROM CMP_SALARY_SIMPLE_COMPNTS
    WHERE PERSON_ID = CSSC.PERSON_ID
)
```

**Component Code Pattern:**
- Format: `ORA_BASIC`, `ORA_HOUSING_ALLOWANCE`, `ORA_TRANSPORT_ALLOWANCE`
- Use `SUBSTR(COMPONENT_CODE, 5, 50)` to remove 'ORA_' prefix
- Filter by latest `LAST_UPDATE_DATE`

---

## 7. 🔧 HCM CROSS-MODULE PATTERNS

**Source:** Analysis of 40+ production queries  
**Purpose:** Advanced cross-module integration patterns

### 7.1 Advanced Date-Track Filtering (LEAST Pattern)

**Problem:** Handle Terminated Employees

**Solution: LEAST Pattern**
```sql
-- CRITICAL PATTERN: Use LEAST to handle termination date
LEAST(NVL(PPOS.ACTUAL_TERMINATION_DATE, :P_EFFECTIVE_DATE), :P_EFFECTIVE_DATE)

-- Full Pattern:
WHERE
    -- For person
    LEAST(NVL(PPOS.ACTUAL_TERMINATION_DATE, :P_EFFECTIVE_DATE), :P_EFFECTIVE_DATE)
        BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
    
    -- For assignment
    AND LEAST(NVL(PPOS.ACTUAL_TERMINATION_DATE, :P_EFFECTIVE_DATE), :P_EFFECTIVE_DATE)
        BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
```

**Why This Works:**
- If employee is **active** (ACTUAL_TERMINATION_DATE is NULL):
  - Uses report date for filtering
- If employee is **terminated**:
  - Uses termination date for filtering (gets last known state)

**When to Use:**
- ✅ Employee master reports (active + terminated)
- ✅ Historical data extraction
- ✅ End-of-service reports
- ✅ Salary extraction for terminated employees
- ✅ Any report requiring "as of" date functionality

### 7.2 Latest Version Patterns

**Pattern 1: Latest Accrual Period**
```sql
WHERE APAE.ACCRUAL_PERIOD = (
    SELECT MAX(ACCRUAL_PERIOD)
    FROM ANC_PER_ACCRUAL_ENTRIES A1
    WHERE A1.PERSON_ID = APAE.PERSON_ID
    AND A1.PLAN_ID = APAE.PLAN_ID
    AND A1.ACCRUAL_PERIOD <= LAST_DAY(:P_DATE)
)
```

**Pattern 2: Latest Salary Component Update**
```sql
WHERE TRUNC(CSSC.LAST_UPDATE_DATE) = (
    SELECT MAX(TRUNC(LAST_UPDATE_DATE))
    FROM CMP_SALARY_SIMPLE_COMPNTS
    WHERE PERSON_ID = CSSC.PERSON_ID
)
```

**Pattern 3: Latest Timecard Attribute Version**
```sql
WHERE AUSG.USAGES_SOURCE_VERSION = (
    SELECT MAX(A1.USAGES_SOURCE_VERSION)
    FROM HWM_TM_REP_ATRB_USAGES A1
    WHERE A1.USAGES_SOURCE_ID = TMD.TM_REC_ID
)
```

### 7.3 Parameter Handling Patterns

**Pattern 1: Multi-Value with NULL Handling**
```sql
WHERE
    (PAAF.LEGAL_ENTITY_ID IN (:P_LEGAL_EMPLOYER) OR LEAST(:P_LEGAL_EMPLOYER) IS NULL)
    AND (PAPF.PERSON_NUMBER IN (:P_EMP) OR LEAST(:P_EMP) IS NULL)
```

**Pattern 2: 'ALL' Pattern**
```sql
WHERE
    (PAPF.PERSON_NUMBER IN (:P_PERSON_NO) OR 'ALL' IN (:P_PERSON_NO || 'ALL'))
    AND (PD.NAME IN (:P_DEPT) OR 'ALL' IN (:P_DEPT || 'ALL'))
```

### 7.4 Dynamic Element Extraction

```sql
SELECT
    PPRD.PERSON_ID,
    PEC.BASE_CLASSIFICATION_NAME,
    SUM(CASE WHEN PEC.BASE_CLASSIFICATION_NAME = 'Standard Earnings'
             THEN TO_NUMBER(PRRV.RESULT_VALUE) ELSE 0 END) TOTAL_EARNINGS,
    SUM(CASE WHEN PEC.BASE_CLASSIFICATION_NAME IN ('Voluntary Deductions', 
                                                   'Social Insurance Deductions')
             THEN TO_NUMBER(PRRV.RESULT_VALUE) ELSE 0 END) TOTAL_DEDUCTIONS
FROM
    PAY_RUN_RESULT_VALUES PRRV,
    PAY_RUN_RESULTS PRR,
    PAY_PAYROLL_REL_ACTIONS PPRA,
    PAY_PAYROLL_ACTIONS PPA,
    PAY_ELE_CLASSIFICATIONS PEC
WHERE
    PRRV.RUN_RESULT_ID = PRR.RUN_RESULT_ID
    AND PRR.PAYROLL_REL_ACTION_ID = PPRA.PAYROLL_REL_ACTION_ID
    AND PPRA.PAYROLL_ACTION_ID = PPA.PAYROLL_ACTION_ID
    AND PPA.ACTION_TYPE IN ('Q', 'R')  -- QuickPay or Regular
    AND PPA.ACTION_STATUS = 'C'  -- Complete
    AND PPRA.RETRO_COMPONENT_ID IS NULL  -- Exclude retro
GROUP BY PPRD.PERSON_ID, PEC.BASE_CLASSIFICATION_NAME
```

### 7.5 FTE Calculation by Legislation

```sql
SELECT
    PAAF.LEGISLATION_CODE,
    -- FTE Calculation
    LEAST(ROUND(
        CASE
            WHEN PAAF.LEGISLATION_CODE = 'AU' THEN (WEEKLY_HOURS / 38)
            WHEN PAAF.LEGISLATION_CODE IN ('AZ', 'CA', 'CN', 'IN', 'US') THEN (WEEKLY_HOURS / 40)
            WHEN PAAF.LEGISLATION_CODE = 'AE' THEN (WEEKLY_HOURS / 42.5)
            WHEN PAAF.LEGISLATION_CODE = 'KZ' THEN (WEEKLY_HOURS / 45)
            WHEN PAAF.LEGISLATION_CODE IN ('IQ', 'SA', 'QA') THEN (WEEKLY_HOURS / 48)
            ELSE 1
        END, 2), 1) FTE
FROM PER_ALL_ASSIGNMENTS_F PAAF
```

**Standard Hours by Legislation:**
- 38 hours: Australia (AU)
- 40 hours: AZ, CA, CN, IN, US
- 42.5 hours: UAE specific entities
- 45 hours: Kazakhstan (KZ)
- 48 hours: IQ, SA, QA

### 7.6 Public Holiday Integration

```sql
SELECT
    PAPF.PERSON_NUMBER,
    -- Public holiday check
    (SELECT SCH.OBJECT_NAME
     FROM TABLE(PER_AVAILABILITY_DETAILS.GET_SCHEDULE_DETAILS(
         P_RESOURCE_TYPE => 'ASSIGN',
         P_RESOURCE_ID => ASG.ASSIGNMENT_ID,
         P_PERIOD_START => TRUNC(:P_DATE),
         P_PERIOD_END => TRUNC(:P_DATE) + 1
     )) SCH
     WHERE SCH.OBJECT_CATEGORY = 'PH'
     AND ROWNUM = 1
    ) PUBLIC_HOLIDAY_NAME
FROM
    PER_ALL_PEOPLE_F PAPF,
    PER_ALL_ASSIGNMENTS_M ASG
WHERE PAPF.PERSON_ID = ASG.PERSON_ID
```

**Key Parameters:**
- `P_RESOURCE_TYPE`: 'ASSIGN' or 'LEGALEMP'
- `P_RESOURCE_ID`: ASSIGNMENT_ID or LEGAL_ENTITY_ID
- `P_PERIOD_START` / `P_PERIOD_END`: Date range

**Object Categories:**
- `'PH'` - Public Holiday
- `'WO'` - Week-off
- `'SHIFT'` - Shift details

### 7.7 Workflow Approval Tracking

```sql
SELECT
    TXNH.TRANSACTION_ID,
    -- Current pending approvers
    (SELECT LISTAGG(REPLACE(ASSIGNEES, ',user', ''), '; ')
     FROM FA_FUSION_SOAINFRA.WFTASK WF
     WHERE WF.STATE IN ('ASSIGNED', 'INFO_REQUESTED')
     AND WF.IDENTIFICATIONKEY = TO_CHAR(TXNH.OBJECT_ID)
    ) PENDING_APPROVERS
FROM
    HRC_TXN_HEADER TXNH,
    HRC_TXN_DATA TXND
WHERE
    TXNH.TRANSACTION_ID = TXND.TRANSACTION_ID
    AND TXND.STATUS = 'PENDING'
```

**Key Tables:**
- `HRC_TXN_HEADER` - Transaction header
- `HRC_TXN_DATA` - Transaction data
- `FA_FUSION_SOAINFRA.WFTASK` - Active workflow tasks
- `FND_BPM_TASK_HISTORY_B` - Workflow history

---

## 8. BI PUBLISHER BURSTING QUERIES

### 8.1 Bursting Query Structure

**Purpose:** Bursting queries enable automated report distribution via email with dynamic recipients, formatting, and content based on data grouping.

**Critical Pattern:** Bursting queries use a **two-level SELECT structure**:
1. **Outer SELECT:** Defines bursting parameters (email, format, template)
2. **Inner SELECT:** Contains the actual data query

### 8.2 Bursting Query Template

```sql
SELECT DISTINCT
    -- REQUIRED: Bursting Key (Split/Group By)
    KEYS AS "KEY",                                    -- Column to split reports by
    
    -- REQUIRED: Report Configuration
    'Template_Name' TEMPLATE,                         -- BI Publisher template name
    'en-US' LOCALE,                                   -- Language/locale
    'xlsx' OUTPUT_FORMAT,                             -- pdf, xlsx, html, csv, rtf
    'Report_Output_Name' OUTPUT_NAME,                 -- Output file name
    
    -- REQUIRED: Delivery Configuration
    'EMAIL' DEL_CHANNEL,                              -- EMAIL, FAX, FTP, FILE, SFTP, PRINT, WEBDAV
    
    -- EMAIL PARAMETERS (when DEL_CHANNEL = 'EMAIL')
    'recipient@example.com' PARAMETER1,               -- TO: Primary recipient(s) (comma-separated)
    'cc@example.com' PARAMETER2,                      -- CC: Carbon copy recipient(s) (comma-separated)
    'Sender Name <sender@example.com>' PARAMETER3,   -- FROM: Sender email with display name
    'Email Subject Line' PARAMETER4,                  -- SUBJECT: Email subject
    
    -- EMAIL BODY (HTML supported)
    '<PRE>'
    || 'Dear ' || 'Recipient Name' || ',' || '<br>'
    || '<br>' || 'Email body content here.' || '<br>'
    || '<br>'
    || 'Thank you.' || '<br>'
    || '</PRE>' PARAMETER5,                           -- BODY: Email body (HTML format)
    
    'true' PARAMETER6,                                -- ATTACHMENT: 'true' or 'false'
    'reply-to@example.com' PARAMETER7,                -- REPLY-TO: (optional)
    'bcc@example.com' PARAMETER8                      -- BCC: Blind carbon copy (optional)

FROM
(
    -- ============================================================================
    -- INNER QUERY: ACTUAL DATA QUERY
    -- ============================================================================
    SELECT
        'All' KEYS,                                   -- Bursting key value (e.g., 'All', PERSON_ID, DEPT_ID)
        -- Add your data columns here
        PAPF.PERSON_ID,
        PAPF.PERSON_NUMBER,
        PPNF.DISPLAY_NAME,
        -- ... other columns ...
        TO_CHAR(SYSDATE, 'DD-Mon-YYYY') CURRENT_DATE
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF,
        PER_ALL_ASSIGNMENTS_F PAAF
    WHERE PAPF.PERSON_ID = PPNF.PERSON_ID
      AND PAPF.PERSON_ID = PAAF.PERSON_ID
      AND PAAF.PRIMARY_FLAG = 'Y'
      AND PAAF.ASSIGNMENT_TYPE = 'E'
      AND PPNF.NAME_TYPE = 'GLOBAL'
      AND TRUNC(SYSDATE) BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
      AND TRUNC(SYSDATE) BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
      AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
      -- Add your filters here
)
```

### 8.3 Bursting Parameters Reference

| Parameter | Purpose | Example Values | Required |
|-----------|---------|----------------|----------|
| **KEY** | Split/group reports by this value | 'All', PERSON_ID, DEPARTMENT_ID | Yes |
| **TEMPLATE** | BI Publisher template name | 'Passport_expiry', 'Visa_Expiry' | Yes |
| **LOCALE** | Language/locale code | 'en-US', 'ar-AE' | Yes |
| **OUTPUT_FORMAT** | File format | 'pdf', 'xlsx', 'html', 'csv', 'rtf' | Yes |
| **OUTPUT_NAME** | Output file name | 'Passport-Expiration-Report' | Yes |
| **DEL_CHANNEL** | Delivery method | 'EMAIL', 'FAX', 'FTP', 'FILE', 'PRINT' | Yes |
| **PARAMETER1** | TO: Email recipients | 'user1@example.com,user2@example.com' | Yes (for EMAIL) |
| **PARAMETER2** | CC: Carbon copy | 'cc@example.com' | No |
| **PARAMETER3** | FROM: Sender | 'Company Name <sender@example.com>' | Yes (for EMAIL) |
| **PARAMETER4** | SUBJECT: Email subject | 'Passport Expiration Alert' | Yes (for EMAIL) |
| **PARAMETER5** | BODY: Email body (HTML) | HTML formatted message | No |
| **PARAMETER6** | ATTACHMENT: Include file | 'true', 'false' | No |
| **PARAMETER7** | REPLY-TO: Reply address | 'reply@example.com' | No |
| **PARAMETER8** | BCC: Blind carbon copy | 'bcc@example.com' | No |

### 8.4 Common Bursting Use Cases

#### A. Alert/Notification Reports (Single Recipient Group)

**Use Case:** Send expiry alerts (passport, visa, contract) to HR team

```sql
SELECT DISTINCT
    KEYS AS "KEY",
    'Passport_expiry' TEMPLATE,
    'en-US' LOCALE,
    'xlsx' OUTPUT_FORMAT,
    'Passport-Expiration Notifications' OUTPUT_NAME,
    'EMAIL' DEL_CHANNEL,
    'hr.team@company.com' PARAMETER1,                 -- TO
    'manager@company.com' PARAMETER2,                 -- CC
    'HR System <noreply@company.com>' PARAMETER3,     -- FROM
    'Passport Expiration Notifications' PARAMETER4,   -- SUBJECT
    '<PRE>'
    || 'Dear HR Team,' || '<br>'
    || '<br>' || 'Below employees have passports expiring soon.' || '<br>'
    || '<br>' || 'Please renew and update in Oracle.' || '<br>'
    || '<br>' || 'Thank you.' || '<br>'
    || '</PRE>' PARAMETER5,
    'true' PARAMETER6
FROM
(
    SELECT
        'All' KEYS,                                   -- Single group for all records
        PAPF.PERSON_NUMBER,
        PPNF.DISPLAY_NAME,
        PP.PASSPORT_NUMBER,
        TO_CHAR(PP.EXPIRATION_DATE, 'DD-MON-YYYY') EXPIRATION_DATE
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_PASSPORTS PP
    WHERE PAPF.PERSON_ID = PPNF.PERSON_ID
      AND PAPF.PERSON_ID = PAAF.PERSON_ID
      AND PAPF.PERSON_ID = PP.PERSON_ID(+)
      AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
      AND PAAF.PRIMARY_FLAG = 'Y'
      AND PPNF.NAME_TYPE = 'GLOBAL'
      AND TRUNC(SYSDATE) BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
      AND TRUNC(SYSDATE) BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
      AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
      -- Expiry filter: Next 30 days
      AND PP.EXPIRATION_DATE BETWEEN TRUNC(SYSDATE) AND TRUNC(SYSDATE) + 30
)
```

#### B. Split by Department/Manager (Multiple Recipients)

**Use Case:** Send department-specific reports to each department manager

```sql
SELECT DISTINCT
    KEYS AS "KEY",
    'Department_Report' TEMPLATE,
    'en-US' LOCALE,
    'pdf' OUTPUT_FORMAT,
    'Department-Report-' || KEYS OUTPUT_NAME,         -- Dynamic file name
    'EMAIL' DEL_CHANNEL,
    MANAGER_EMAIL PARAMETER1,                         -- Dynamic TO based on department
    'hr@company.com' PARAMETER2,                      -- CC to HR
    'Reports <reports@company.com>' PARAMETER3,
    'Department Report - ' || DEPARTMENT_NAME PARAMETER4,  -- Dynamic subject
    '<PRE>'
    || 'Dear ' || MANAGER_NAME || ',' || '<br>'
    || '<br>' || 'Please find attached your department report.' || '<br>'
    || '</PRE>' PARAMETER5,
    'true' PARAMETER6
FROM
(
    SELECT
        DEPT.ORGANIZATION_ID KEYS,                    -- Split by department
        DEPT.NAME DEPARTMENT_NAME,
        MGR.EMAIL_ADDRESS MANAGER_EMAIL,
        MGR_NAME.FULL_NAME MANAGER_NAME,
        PAPF.PERSON_NUMBER,
        PPNF.FULL_NAME
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_DEPARTMENTS DEPT,
        PER_ASSIGNMENT_SUPERVISORS_F PASF,
        PER_ALL_PEOPLE_F MGR,
        PER_PERSON_NAMES_F MGR_NAME,
        PER_EMAIL_ADDRESSES MGR_EMAIL
    WHERE PAPF.PERSON_ID = PPNF.PERSON_ID
      AND PAPF.PERSON_ID = PAAF.PERSON_ID
      AND PAAF.ORGANIZATION_ID = DEPT.ORGANIZATION_ID
      AND PAAF.ASSIGNMENT_ID = PASF.ASSIGNMENT_ID(+)
      AND PASF.MANAGER_ID = MGR.PERSON_ID(+)
      AND MGR.PERSON_ID = MGR_NAME.PERSON_ID(+)
      AND MGR.PERSON_ID = MGR_EMAIL.PERSON_ID(+)
      AND MGR_EMAIL.EMAIL_TYPE(+) = 'W1'
      -- Add date-effective filters
)
```

#### C. Time and Labor Alerts (Monthly Aggregation)

**Use Case:** Send monthly time deficit alerts for employees exceeding threshold

**Example: Monthly Time Deficit Exceeding 6 Hours Alert**

```sql
SELECT DISTINCT
    KEYS AS "KEY",
    'Output' TEMPLATE,
    'en-US' LOCALE,
    'xlsx' OUTPUT_FORMAT,
    'Monthly Time Deficit Alert - ' || TO_CHAR(ADD_MONTHS(TRUNC(SYSDATE, 'MM'), -1), 'MON-YYYY') OUTPUT_NAME,
    'EMAIL' DEL_CHANNEL,
    'hr.team@company.com' PARAMETER1,                 -- TO: Primary recipients
    '' PARAMETER2,                                     -- CC: (empty)
    'HR System <noreply@company.com>' PARAMETER3,     -- FROM: Sender
    'Alert: Monthly Time Deficit Exceeding 6 Hours' PARAMETER4,  -- SUBJECT
    '<PRE>'
    || 'Dear HOD / Incharge,' || '<br>'
    || '<br>'
    || 'This is to bring to your attention that the following employee(s) under your team have accumulated a time deficit of more than 6 hours during the current month.' || '<br>'
    || '<br>'
    || 'Kindly discuss the matter with the concerned employee(s) and ensure corrective action is taken to avoid further discrepancies.' || '<br>'
    || '<br>'
    || 'Timely correction is essential to maintain accurate attendance and payroll records.' || '<br>'
    || '<br>'
    || 'Thank you for your support in maintaining attendance compliance.' || '<br>'
    || '</PRE>' PARAMETER5,
    'true' PARAMETER6
FROM
(
    -- ============================================================================
    -- INNER QUERY: MONTHLY DEFICIT SUMMARY
    -- ============================================================================
    WITH
    PARAMETERS AS (
        SELECT
            TRUNC(ADD_MONTHS(TRUNC(SYSDATE, 'MM'), -1)) AS FROM_DATE,  -- First day of last month
            TRUNC(LAST_DAY(ADD_MONTHS(TRUNC(SYSDATE, 'MM'), -1))) AS TO_DATE  -- Last day of last month
        FROM DUAL
    ),
    
    TIME_REPORTED AS (
        SELECT
            HTR.RESOURCE_ID,
            HTR.SUBRESOURCE_ID,
            -- Deficit Hours (shown as positive value)
            SUM(CASE WHEN HTRMPTTAV.PAY_PAYROLL_TIME_TYPE = 'Deficit Hours' 
                     THEN ABS(HTR.MEASURE) ELSE 0 END) DEFICIT_HOURS
        FROM 
            HWM_TM_REC HTR,
            HWM_TM_REC_GRP_USAGES HTRGU,
            HWM_TM_REC_GRP DTRG,
            HWM_TM_REC_GRP DTRG1,
            HWM_TM_REP_M_PTT_ATRBS_V HTRMPTTAV,
            PARAMETERS P
        WHERE HTRGU.TM_REC_GRP_ID = DTRG.TM_REC_GRP_ID
          AND DTRG.PARENT_TM_REC_GRP_ID = DTRG1.TM_REC_GRP_ID
          AND HTR.TM_REC_ID = HTRGU.TM_REC_ID
          AND HTR.TM_REC_ID = HTRMPTTAV.USAGES_SOURCE_ID(+)
          AND HTR.LATEST_VERSION = 'Y'
          AND HTR.RESOURCE_TYPE = 'PERSON'
          AND HTR.DELETE_FLAG IS NULL
          AND TRUNC(DTRG.START_TIME) BETWEEN P.FROM_DATE AND P.TO_DATE
        GROUP BY HTR.RESOURCE_ID, HTR.SUBRESOURCE_ID
    ),
    
    MONTHLY_DEFICIT_SUMMARY AS (
        SELECT
            TR.RESOURCE_ID,
            TR.SUBRESOURCE_ID,
            SUM(TR.DEFICIT_HOURS) AS MONTHLY_DEFICIT_HOURS
        FROM TIME_REPORTED TR
        GROUP BY TR.RESOURCE_ID, TR.SUBRESOURCE_ID
        HAVING SUM(TR.DEFICIT_HOURS) >= 6  -- Filter: Only employees with deficit >= 6 hours
    )
    
    SELECT
        'All' KEYS,
        PAPF.PERSON_NUMBER EMPLOYEE_NUMBER,
        PPNF.DISPLAY_NAME EMPLOYEE_NAME,
        PD.NAME DEPARTMENT,
        ROUND(MDS.MONTHLY_DEFICIT_HOURS, 2) MONTHLY_DEFICIT_HOURS
    FROM
        PER_ALL_PEOPLE_F PAPF,
        PER_PERSON_NAMES_F PPNF,
        PER_ALL_ASSIGNMENTS_F PAAF,
        PER_DEPARTMENTS PD,
        MONTHLY_DEFICIT_SUMMARY MDS,
        PARAMETERS P
    WHERE PAPF.PERSON_ID = MDS.RESOURCE_ID
      AND PAPF.PERSON_ID = PPNF.PERSON_ID
      AND PAPF.PERSON_ID = PAAF.PERSON_ID
      AND PAAF.ASSIGNMENT_ID = MDS.SUBRESOURCE_ID
      AND PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
      AND PAAF.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
      AND PAAF.PRIMARY_FLAG = 'Y'
      AND PPNF.NAME_TYPE = 'GLOBAL'
      AND TRUNC(SYSDATE) BETWEEN PAPF.EFFECTIVE_START_DATE AND PAPF.EFFECTIVE_END_DATE
      AND TRUNC(SYSDATE) BETWEEN PPNF.EFFECTIVE_START_DATE AND PPNF.EFFECTIVE_END_DATE
      AND TRUNC(SYSDATE) BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
)
```

**Key Features:**
- **Monthly Aggregation:** One row per employee (not daily breakdown)
- **Threshold Filter:** Only employees with monthly deficit >= 6 hours
- **Date Range:** Last complete month (previous month)
- **Deficit Calculation:** Sum of daily deficit hours from `HWM_TM_REP_M_PTT_ATRBS_V.PAY_PAYROLL_TIME_TYPE = 'Deficit Hours'`
- **Version Control:** Uses `LATEST_VERSION = 'Y'` to prevent duplicate counting
- **Dynamic Output Name:** Includes month/year in file name

**Reference:** `HCM/Requirement/Alert Report/Monthly Time Deficit Exceeding 6 Hours/Monthly_Time_Deficit_6Hours_Alert_Bursting.sql`

### 8.5 Bursting Best Practices

1. **Always use DISTINCT in outer SELECT** - Prevents duplicate emails
2. **KEYS column must be aliased as "KEY"** - Required by BI Publisher
3. **Use 'All' for single-group reports** - When sending to one recipient group
4. **HTML formatting in PARAMETER5** - Use `<br>` for line breaks, `<PRE>` tags for formatting
5. **Multiple recipients** - Comma-separated in PARAMETER1/PARAMETER2/PARAMETER8
6. **Dynamic content** - Use concatenation (`||`) for dynamic subjects/bodies
7. **Test with small dataset first** - Use PERSON_ID filter during development
8. **Date filters for alerts** - Use `BETWEEN TRUNC(SYSDATE) AND TRUNC(SYSDATE) + N` for expiry alerts

### 8.6 Common Bursting Patterns

#### Pattern 1: Expiry Alerts (Passport/Visa/Contract)
```sql
-- Filter: Documents expiring in next N days
AND DOCUMENT.EXPIRATION_DATE BETWEEN TRUNC(SYSDATE) AND TRUNC(SYSDATE) + 30
```

#### Pattern 2: Lookup Decoding in Subquery
```sql
-- Use inline subquery for lookup values
(SELECT MEANING FROM FND_LOOKUP_VALUES_TL FLV
 WHERE FLV.LOOKUP_CODE = PP.ISSUING_COUNTRY
   AND FLV.LOOKUP_TYPE = 'JEES_EURO_COUNTRY_CODES'
   AND FLV.LANGUAGE = 'US') AS ISSUING_COUNTRY
```

#### Pattern 3: HTML Email Body
```sql
'<PRE>'
|| 'Dear ' || RECIPIENT_NAME || ',' || '<br>'
|| '<br>' || 'Message line 1.' || '<br>'
|| '<br>' || 'Message line 2.' || '<br>'
|| '<br>' || 'Thank you.' || '<br>'
|| '</PRE>' PARAMETER5
```

### 8.7 Troubleshooting Bursting Queries

| Issue | Cause | Solution |
|-------|-------|----------|
| Duplicate emails sent | Missing DISTINCT | Add DISTINCT to outer SELECT |
| No emails sent | KEYS returns NULL | Ensure KEYS column has non-NULL values |
| Wrong recipients | Incorrect PARAMETER1 | Verify email addresses in data |
| Template not found | Wrong TEMPLATE name | Match exact template name in BI Publisher |
| Email body not formatted | Missing HTML tags | Use `<br>` for breaks, `<PRE>` tags |
| Multiple reports per recipient | Wrong KEYS grouping | Review KEYS column logic |

---

**Last Updated:** 16-Feb-2026  
**Version:** 3.2 (Added Monthly Time Deficit Alert bursting query example)  
**Status:** Production Standards Active
