# One HR Master Instructions: Core Human Resources

**Module:** Core HR
**Tag:** `#HCM #HR #CoreHR`
**Status:** Active

---

## 1. 🚨 Critical HR Constraints
*Violating these rules breaks the system.*

1.  **Date-Track Filtering (The "Golden Rule"):**
    *   **Rule:** `AND TRUNC(SYSDATE) BETWEEN [TABLE].EFFECTIVE_START_DATE AND [TABLE].EFFECTIVE_END_DATE`
    *   **Why:** Without this, you get duplicate rows for every historical change (Promotions, Salary Changes).
    *   **Scope:** Applied to all `_F` (DateTrack) and `_M` (DateTracked with Updates) tables.

2.  **Assignment Status:**
    *   **Rule:** `AND PAAM.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'` (unless reporting on Terminations).
    *   **Why:** Excludes 'Redundant' or 'Inactive' assignment rows.

3.  **Primary Assignment Flag:**
    *   **Rule:** `AND PAAM.PRIMARY_FLAG = 'Y'`
    *   **Why:** Using secondary assignments can duplicate headcount numbers.

4.  **Legislative Data Group (LDG):**
    *   **Rule:** `AND LEGISLATION_CODE = 'US'` (or relevant code).

---

## 2. ⚡ Performance Optimization

| Object | Optimal Access Path | Hint Syntax |
|--------|---------------------|-------------|
| **Person** | PERSON_ID | `/*+ INDEX(PAPF PER_PEOPLE_F_PK) */` |
| **Assignment (_F)** | ASSIGNMENT_ID | `/*+ INDEX(PAAF PER_ALL_ASSIGNMENTS_F_PK) */` |
| **Assignment (_M)** | ASSIGNMENT_ID | `/*+ INDEX(PAAM PER_ALL_ASSIGNMENTS_M_PK) */` |
| **Period of Service** | PERSON_ID | `/*+ INDEX(PPOS PER_PERIODS_OF_SERVICE_N1) */` |

---

## 2.1 🔥 Managed Tables (_M) vs Date-Tracked (_F)

**Critical Decision Point:** When to use `_M` vs `_F` tables

### Use `PER_ALL_ASSIGNMENTS_M` When:
- You only need the **latest/current** assignment record
- Report is for **current snapshot** (not historical)
- Performance is critical (single row per person)

**Pattern:**
```sql
FROM PER_ALL_ASSIGNMENTS_M PAAM
WHERE PAAM.EFFECTIVE_LATEST_CHANGE = 'Y'
AND PAAM.PRIMARY_FLAG = 'Y'
AND PAAM.ASSIGNMENT_TYPE = 'E'
```

### Use `PER_ALL_ASSIGNMENTS_F` When:
- You need **full date-track history**
- Report covers **historical changes** (promotions, transfers)
- Need to query **specific date range**

**Pattern:**
```sql
FROM PER_ALL_ASSIGNMENTS_F PAAF
WHERE PAAF.PRIMARY_FLAG = 'Y'
AND PAAF.ASSIGNMENT_TYPE = 'E'
AND [DATE] BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
```

**Performance Impact:** `_M` tables are **significantly faster** for current data queries.

---

## 3. 🗺️ Schema Map (Key Tables)

###  3.1 Core Person Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PAPF** | `PER_ALL_PEOPLE_F` | Person Header (Name, Hire Date) |
| **PPNF** | `PER_PERSON_NAMES_F` | Names (First, Last, Display, Title) |
| **PS** | `PER_PERSONS` | Person Core (DOB, Country of Birth) |
| **PPLF** | `PER_PEOPLE_LEGISLATIVE_F` | Legislative Data (Gender, Marital Status, SSN) |
| **PPTV** | `PER_PERSON_TYPES_VL` | Person Types (Employee, Contingent Worker) |
| **PPTTL** | `PER_PERSON_TYPES_TL` | Person Type Translations |
| **PPTU** | `PER_PERSON_TYPE_USAGES_M` | Person Type Usage History |

### 3.2 Assignment Tables

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PAAF** | `PER_ALL_ASSIGNMENTS_F` | Assignment (Full Date-Track History) |
| **PAAM** | `PER_ALL_ASSIGNMENTS_M` | Assignment (Managed - Latest Changes Only) |
| **PASF** | `PER_ASSIGNMENT_SUPERVISORS_F` | Manager/Supervisor Hierarchy |
| **PASTT** | `PER_ASSIGNMENT_STATUS_TYPES_TL` | Assignment Status Translations |

### 3.3 Identification & Documents

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PNI** | `PER_NATIONAL_IDENTIFIERS` | National IDs (Emirates ID, SSN, Tax IDs) |
| **PNIV** | `PER_NATIONAL_IDENTIFIERS_V` | National Identifiers View |
| **PP** | `PER_PASSPORTS` | Passport Details |
| **PC** | `PER_CITIZENSHIPS` | Citizenship/Nationality |

### 3.4 Employment & Service

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PPOS** | `PER_PERIODS_OF_SERVICE` | Employment Terms (Hire, Termination) |
| **PAC** | `PER_ACTION_OCCURRENCES` | Action Occurrences (Transfer, Termination) |
| **ACTN** | `PER_ACTIONS_VL` | Actions Master (Leaving, Promotion) |
| **PART** | `PER_ACTION_REASONS_TL` | Action Reasons (Resignation, etc.) |
| **PAR** | `PER_ACTION_REASONS_B` | Action Reasons Base |

### 3.5 Contact Information

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PEA** | `PER_EMAIL_ADDRESSES` | Email Addresses |
| **PR** | `PER_RELIGIONS` | Religion Information |

### 3.6 Organization & Job

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **PD** | `PER_DEPARTMENTS` | Department Details |
| **PJFV** | `PER_JOBS_F_VL` | Job Definitions (View with Translations) |
| **PG** | `PER_GRADES` | Grade Definitions |
| **PGFT** | `PER_GRADES_F_TL` | Grade Translations |
| **HAPFT** | `HR_ALL_POSITIONS_F_TL` | Position Translations |
| **PL** | `PER_LOCATION_DETAILS_F_VL` | Location Details View |
| **PPG** | `PER_PEOPLE_GROUPS` | People Groups (Payroll Grouping) |

### 3.7 Lookups & Reference

| Alias | Table Name | Purpose |
|-------|------------|---------|
| **HL** | `HCM_LOOKUPS` | HCM Lookup Values |
| **PVL** | `FND_TERRITORIES_VL` | Country/Territory Lookup |

---

## 4. 📋 Legislative Data Patterns

### 4.1 Gender Decode
```sql
DECODE(PPLF.SEX, 'M', 'Male', 'F', 'Female', PPLF.SEX) GENDER
```

### 4.2 Marital Status Decode
```sql
DECODE(PPLF.MARITAL_STATUS, 
    'M', 'Married', 
    'S', 'Single', 
    'W', 'Widowed', 
    'D', 'Divorced', 
    PPLF.MARITAL_STATUS
) MARITAL_STATUS
```

### 4.3 Nationality Lookup Pattern
```sql
SELECT H.MEANING
FROM PER_CITIZENSHIPS PC,
     HCM_LOOKUPS H
WHERE PAPF.PERSON_ID = PC.PERSON_ID
AND H.LOOKUP_CODE = PC.LEGISLATION_CODE
AND H.LOOKUP_TYPE = 'NATIONALITY'
```

### 4.4 Emirates ID Formatting (UAE Specific)
```sql
CASE WHEN PNI.NATIONAL_IDENTIFIER_NUMBER IS NOT NULL THEN 
    SUBSTR(PNI.NATIONAL_IDENTIFIER_NUMBER, 1, 3) || '-' || 
    SUBSTR(PNI.NATIONAL_IDENTIFIER_NUMBER, 4, 4) || '-' ||
    SUBSTR(PNI.NATIONAL_IDENTIFIER_NUMBER, 8, 7) || '-' ||
    SUBSTR(PNI.NATIONAL_IDENTIFIER_NUMBER, 15, 1)
ELSE NULL 
END EMIRATES_ID_FORMATTED
```

**Format:** 784-1234-1234567-1

### 4.5 Religion Lookup Pattern
```sql
SELECT HL.MEANING
FROM PER_RELIGIONS PR,
     HCM_LOOKUPS HL
WHERE HL.LOOKUP_TYPE = 'PER_RELIGION'
AND PR.RELIGION = HL.LOOKUP_CODE
AND PAPF.PERSON_ID = PR.PERSON_ID
```

---

## 5. 🔍 Common Lookup Types

| Lookup Type | Purpose | Table | Sample Values |
|-------------|---------|-------|---------------|
| `NATIONALITY` | Nationality/Citizenship | HCM_LOOKUPS | ARE, USA, IND, PAK |
| `PER_RELIGION` | Religion | HCM_LOOKUPS | MUSLIM, CHRISTIAN, HINDU |
| `EMPLOYEE_CATG` | Employee Category | HR_LOOKUPS | STAFF, MANAGER, EXECUTIVE |
| `TITLE` | Name Title | HR_LOOKUPS | MR., MRS., MS., MISS |
| `PER_MARITAL_STATUS` | Marital Status | HR_LOOKUPS | M, S, W, D |

---

## 6. 🔗 Standard Joins (Copy-Paste Ready)

### 6.1 Person with Legislative Data
```sql
FROM PER_ALL_PEOPLE_F PAPF,
     PER_PERSON_NAMES_F PPNF,
     PER_PEOPLE_LEGISLATIVE_F PPLF
WHERE PAPF.PERSON_ID = PPNF.PERSON_ID
AND PAPF.PERSON_ID = PPLF.PERSON_ID
AND PPNF.NAME_TYPE = 'GLOBAL'
AND TRUNC(SYSDATE) BETWEEN TRUNC(PAPF.EFFECTIVE_START_DATE) AND TRUNC(PAPF.EFFECTIVE_END_DATE)
AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF.EFFECTIVE_START_DATE) AND TRUNC(PPNF.EFFECTIVE_END_DATE)
AND TRUNC(SYSDATE) BETWEEN TRUNC(PPLF.EFFECTIVE_START_DATE) AND TRUNC(PPLF.EFFECTIVE_END_DATE)
```

### 6.2 Person with Citizenship
```sql
FROM PER_ALL_PEOPLE_F PAPF,
     PER_CITIZENSHIPS PC,
     HCM_LOOKUPS HL
WHERE PAPF.PERSON_ID = PC.PERSON_ID(+)
AND PC.LEGISLATION_CODE = HL.LOOKUP_CODE(+)
AND HL.LOOKUP_TYPE(+) = 'NATIONALITY'
```

### 6.3 Person with National Identifier
```sql
FROM PER_ALL_PEOPLE_F PAPF,
     PER_NATIONAL_IDENTIFIERS PNI
WHERE PAPF.PERSON_ID = PNI.PERSON_ID(+)
AND PNI.NATIONAL_IDENTIFIER_TYPE(+) = 'NATIONAL_IDENTIFIER'
```

### 6.4 Person with Passport
```sql
FROM PER_ALL_PEOPLE_F PAPF,
     PER_PASSPORTS PP
WHERE PAPF.PERSON_ID = PP.PERSON_ID(+)
AND PP.PASSPORT_TYPE(+) = 'PASSPORT'
```

### 6.5 Assignment with Manager
```sql
FROM PER_ALL_ASSIGNMENTS_F PAAF,
     PER_ASSIGNMENT_SUPERVISORS_F PASF,
     PER_PERSON_NAMES_F PPNF_MGR
WHERE PAAF.PERSON_ID = PASF.PERSON_ID(+)
AND PASF.MANAGER_ID = PPNF_MGR.PERSON_ID(+)
AND PPNF_MGR.NAME_TYPE(+) = 'GLOBAL'
AND TRUNC(SYSDATE) BETWEEN TRUNC(PASF.EFFECTIVE_START_DATE(+)) AND TRUNC(PASF.EFFECTIVE_END_DATE(+))
AND TRUNC(SYSDATE) BETWEEN TRUNC(PPNF_MGR.EFFECTIVE_START_DATE(+)) AND TRUNC(PPNF_MGR.EFFECTIVE_END_DATE(+))
```

### 6.6 Assignment with Termination Actions
```sql
FROM PER_ALL_ASSIGNMENTS_M PAAM,
     PER_ACTION_OCCURRENCES PAC,
     PER_ACTIONS_VL ACTN,
     PER_ACTION_REASONS_TL PART
WHERE PAC.ACTION_OCCURRENCE_ID = PAAM.ACTION_OCCURRENCE_ID(+)
AND ACTN.ACTION_ID = PAC.ACTION_ID(+)
AND PAAM.REASON_CODE = PART.ACTION_REASON_CODE(+)
AND PART.LANGUAGE(+) = 'US'
```

---

## 7. ⚠️ Common Pitfalls

### 7.1 Using Wrong Assignment Table
**Problem:** Query too slow or returns duplicate rows  
**Cause:** Using `_F` when `_M` would be sufficient

**Solution:**
- Use `_M` for current data reports
- Use `_F` only when history needed

### 7.2 Missing EFFECTIVE_LATEST_CHANGE Flag
**Problem:** Multiple rows per person from `_M` table  
**Cause:** Not filtering for latest change

**Solution:**
```sql
AND PAAM.EFFECTIVE_LATEST_CHANGE = 'Y'
```

### 7.3 Legislative Data Not Found
**Problem:** Gender, Marital Status returning NULL  
**Cause:** Not joining to `PER_PEOPLE_LEGISLATIVE_F`

**Solution:**
```sql
FROM PER_ALL_PEOPLE_F PAPF,
     PER_PEOPLE_LEGISLATIVE_F PPLF
WHERE PAPF.PERSON_ID = PPLF.PERSON_ID
AND TRUNC(SYSDATE) BETWEEN PPLF.EFFECTIVE_START_DATE AND PPLF.EFFECTIVE_END_DATE
```

### 7.4 Nationality Shows Code Instead of Meaning
**Problem:** Nationality column shows 'ARE', 'IND' instead of 'United Arab Emirates', 'India'

**Solution:** Join to HCM_LOOKUPS to get meaning
```sql
SELECT HL.MEANING
FROM PER_CITIZENSHIPS PC,
     HCM_LOOKUPS HL
WHERE PC.LEGISLATION_CODE = HL.LOOKUP_CODE
AND HL.LOOKUP_TYPE = 'NATIONALITY'
```

### 7.5 Multiple National Identifiers
**Problem:** Duplicate rows because person has multiple IDs (Emirates ID + Passport)

**Solution:** Filter by specific identifier type
```sql
AND PNI.NATIONAL_IDENTIFIER_TYPE = 'NATIONAL_IDENTIFIER'  -- Emirates ID only
-- OR
AND PNI.NATIONAL_IDENTIFIER_TYPE = 'SSN'  -- Social Security Number only
```

### 7.6 PERSON_TYPE_ID Location
**Problem:** Using `PAPF.PERSON_TYPE_ID` causes "column does not exist" error

**Cause:** `PERSON_TYPE_ID` exists in `PER_ALL_ASSIGNMENTS_F`, NOT in `PER_ALL_PEOPLE_F`

**Solution:**
```sql
FROM PER_ALL_PEOPLE_F PAPF,
     PER_ALL_ASSIGNMENTS_F PAAF,
     PER_PERSON_TYPES_TL PPTTL
WHERE PAPF.PERSON_ID = PAAF.PERSON_ID
AND PAAF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID(+)  -- Correct: Use PAAF, not PAPF
```

---

## 8. 📊 Standard Filters

### 8.1 Active Employees (Managed Table)
```sql
AND PAAM.EFFECTIVE_LATEST_CHANGE = 'Y'
AND PAAM.ASSIGNMENT_STATUS_TYPE = 'ACTIVE'
AND PAAM.PRIMARY_FLAG = 'Y'
AND PAAM.ASSIGNMENT_TYPE = 'E'
```

### 8.2 Current Employees with Termination Date Handling
```sql
AND TRUNC(SYSDATE) BETWEEN PPOS.DATE_START 
    AND NVL(PPOS.ACTUAL_TERMINATION_DATE, TO_DATE('4712-12-31', 'YYYY-MM-DD'))
```

### 8.3 Legislative Data Current Snapshot
```sql
AND TRUNC(SYSDATE) BETWEEN PPLF.EFFECTIVE_START_DATE AND PPLF.EFFECTIVE_END_DATE
```

---

## 10. 🚪 Leaving Actions & Reasons

### 10.1 Leaving Action Pattern

**Use Case:** Get termination action (Voluntary Resignation, Retirement, etc.)

```sql
(SELECT ACTN.ACTION_NAME
 FROM PER_ALL_ASSIGNMENTS_M ASG1,
      PER_ACTION_OCCURRENCES PAC,
      PER_ACTIONS_VL ACTN
 WHERE PAC.ACTION_OCCURRENCE_ID = ASG1.ACTION_OCCURRENCE_ID
 AND ACTN.ACTION_ID = PAC.ACTION_ID
 AND ASG1.ASSIGNMENT_ID = PAAM.ASSIGNMENT_ID
 AND ASG1.ASSIGNMENT_TYPE = 'E'
 AND ASG1.PRIMARY_FLAG = 'Y'
 AND ASG1.EFFECTIVE_LATEST_CHANGE = 'Y'
 AND TRUNC(PPOS.ACTUAL_TERMINATION_DATE + 1) BETWEEN ASG1.EFFECTIVE_START_DATE 
                                                  AND ASG1.EFFECTIVE_END_DATE
 AND ROWNUM = 1
) LEAVING_ACTION
```

### 10.2 Leaving Reason Pattern

**Use Case:** Get termination reason

```sql
(SELECT PART.ACTION_REASON
 FROM PER_ALL_ASSIGNMENTS_M ASG1,
      PER_ACTION_REASONS_TL PART,
      PER_ACTION_REASONS_B PAR
 WHERE ASG1.ASSIGNMENT_ID = PAAM.ASSIGNMENT_ID
 AND PART.ACTION_REASON_ID = PAR.ACTION_REASON_ID
 AND ASG1.REASON_CODE = PAR.ACTION_REASON_CODE
 AND ASG1.ASSIGNMENT_TYPE = 'E'
 AND ASG1.PRIMARY_FLAG = 'Y'
 AND ASG1.EFFECTIVE_LATEST_CHANGE = 'Y'
 AND PART.LANGUAGE = 'US'
 AND TRUNC(PPOS.ACTUAL_TERMINATION_DATE + 1) BETWEEN ASG1.EFFECTIVE_START_DATE 
                                                  AND ASG1.EFFECTIVE_END_DATE
 AND TRUNC(PPOS.ACTUAL_TERMINATION_DATE + 1) BETWEEN PAR.START_DATE AND PAR.END_DATE
 AND ROWNUM = 1
) LEAVING_REASON
```

---

## 11. 💼 Worker Category & Type

### 11.1 Worker Category Pattern

```sql
(SELECT MEANING 
 FROM HR_LOOKUPS 
 WHERE LOOKUP_TYPE = 'EMPLOYEE_CATG' 
 AND LOOKUP_CODE = PAAM.EMPLOYEE_CATEGORY 
 AND ROWNUM = 1
) WORKER_CATEGORY
```

### 11.2 Payroll Time Type Pattern

```sql
PAAM.ASS_ATTRIBUTE1 AS PAYROLL_TIME_TYPE
```

---

## 12. 🆔 Additional Identification Patterns

### 12.1 National Identifier

```sql
(SELECT PNI.NATIONAL_IDENTIFIER_NUMBER
 FROM PER_NATIONAL_IDENTIFIERS PNI,
      PER_NAT_IDENTIFIER_TYPES PNIT
 WHERE PAPF.PERSON_ID = PNI.PERSON_ID
 AND PNI.NATIONAL_IDENTIFIER_TYPE_ID = PNIT.NATIONAL_IDENTIFIER_TYPE_ID
 AND UPPER(PNIT.LEGISLATION_CODE) = 'US'
 AND UPPER(PNIT.NATIONAL_IDENTIFIER_TYPE) = 'SSN'
 AND ROWNUM = 1
) NATIONAL_ID
```

### 12.2 Email Address

```sql
FROM PER_EMAIL_ADDRESSES PEA
WHERE PAPF.PERSON_ID = PEA.PERSON_ID
AND PEA.EMAIL_TYPE = 'W1'  -- Work email
AND TRUNC(SYSDATE) BETWEEN PEA.DATE_FROM 
    AND NVL(PEA.DATE_TO, TO_DATE('4712-12-31', 'YYYY-MM-DD'))
```

---

## 13. 🏢 Organization Classification

```sql
FROM HR_ORG_UNIT_CLASSIFICATIONS_F HOUCF,
     HR_ALL_ORGANIZATION_UNITS_F HAOUF,
     HR_ORGANIZATION_UNITS_F_TL HAUFT
WHERE HAOUF.ORGANIZATION_ID = HOUCF.ORGANIZATION_ID
AND HAOUF.ORGANIZATION_ID = HAUFT.ORGANIZATION_ID
AND HAOUF.EFFECTIVE_START_DATE BETWEEN HOUCF.EFFECTIVE_START_DATE 
                                   AND HOUCF.EFFECTIVE_END_DATE
AND HAUFT.LANGUAGE = 'US'
AND HOUCF.CLASSIFICATION_CODE = 'DEPARTMENT'
AND TRUNC(SYSDATE) BETWEEN HAUFT.EFFECTIVE_START_DATE AND HAUFT.EFFECTIVE_END_DATE
```

---

**Last Updated:** 18-Dec-2025  
**Status:** Production-Ready  
**Module:** Core HR  
**Source:** 13 Production Queries Revalidation
