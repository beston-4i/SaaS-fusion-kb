# CRITICAL TABLE MAPPINGS - Oracle HCM Cloud
**Module:** HR Master Data  
**Date:** 05-01-2026  
**Source:** All Employee detail with salary Query Must use.sql  
**Priority:** CRITICAL - Must Follow

---

## ⚠️ PURPOSE
This document captures the **CORRECT** table relationships and join patterns from production queries. These are NOT assumptions - they are validated patterns that MUST be followed.

---

## 🔴 CRITICAL FIELD MAPPINGS

### 1. PERSONAL INFORMATION FIELDS

| Field | ❌ WRONG Table | ✅ CORRECT Table | Join Pattern |
|-------|---------------|------------------|--------------|
| **SEX (Gender)** | `PER_ALL_PEOPLE_F.SEX` | `PER_PEOPLE_LEGISLATIVE_F.SEX` | Direct via PERSON_ID |
| **DATE_OF_BIRTH** | `PER_ALL_PEOPLE_F.DATE_OF_BIRTH` | `PER_PERSONS.DATE_OF_BIRTH` | Direct via PERSON_ID (NOT date-tracked) |
| **MARITAL_STATUS** | `PER_ALL_PEOPLE_F.MARITAL_STATUS` | `PER_PEOPLE_LEGISLATIVE_F.MARITAL_STATUS` | Direct via PERSON_ID |

**CORRECT SQL Pattern:**
```sql
SELECT
    PPLF.SEX AS GENDER,
    PP.DATE_OF_BIRTH,
    PPLF.MARITAL_STATUS
FROM
    PER_ALL_PEOPLE_F PAPF,
    PER_PEOPLE_LEGISLATIVE_F PPLF,
    PER_PERSONS PP
WHERE
    PAPF.PERSON_ID = PPLF.PERSON_ID
AND PAPF.PERSON_ID = PP.PERSON_ID
AND :P_EFFECTIVE_DATE BETWEEN PPLF.EFFECTIVE_START_DATE AND PPLF.EFFECTIVE_END_DATE
-- Note: PER_PERSONS is NOT date-tracked
```

---

### 2. RELIGION

| Field | ❌ WRONG Approach | ✅ CORRECT Approach |
|-------|------------------|---------------------|
| **RELIGION** | `PER_ALL_PEOPLE_F.RELIGION_ID = PER_RELIGIONS.RELIGION_ID` | Subquery via PERSON_ID + Lookup |

**CORRECT SQL Pattern (SUBQUERY):**
```sql
(SELECT FLV.MEANING
 FROM PER_RELIGIONS PR, FND_LOOKUP_VALUES FLV
 WHERE PR.RELIGION = FLV.LOOKUP_CODE
   AND FLV.LOOKUP_TYPE = 'PER_RELIGION'
   AND FLV.LANGUAGE = 'US'
   AND PR.PERSON_ID = PAPF.PERSON_ID
   AND ROWNUM = 1
) AS RELIGION_NAME
```

**Key Points:**
- ❌ **NOT** joined via `PAPF.RELIGION_ID` (this column doesn't exist or isn't reliable)
- ✅ Join via `PR.PERSON_ID = PAPF.PERSON_ID`
- ✅ Use `FND_LOOKUP_VALUES` to get the meaning
- ✅ Lookup type = `'PER_RELIGION'`
- ✅ Implemented as **SUBQUERY**, not in FROM clause

---

### 3. NATIONAL IDENTIFIER (Emirates ID)

| Field | ❌ WRONG Approach | ✅ CORRECT Approach |
|-------|------------------|---------------------|
| **EMIRATES_ID** | Only `PRIMARY_NID_ID` join | **BOTH** `PRIMARY_NID_ID` + `PERSON_ID` |

**CORRECT SQL Pattern:**
```sql
SELECT
    PNI.NATIONAL_IDENTIFIER_NUMBER AS EMIRATES_ID
FROM
    PER_ALL_PEOPLE_F PAPF,
    PER_NATIONAL_IDENTIFIERS PNI
WHERE
    PAPF.PRIMARY_NID_ID = PNI.NATIONAL_IDENTIFIER_ID(+)
AND PNI.PERSON_ID(+) = PAPF.PERSON_ID
```

**Key Points:**
- ✅ Use **BOTH** join conditions
- ✅ `PRIMARY_NID_ID` links to `NATIONAL_IDENTIFIER_ID`
- ✅ `PERSON_ID` links to `PERSON_ID`

---

### 4. NATIONALITY & COUNTRY OF BIRTH

| Field | ❌ WRONG Table | ✅ CORRECT Table |
|-------|---------------|------------------|
| **NATIONALITY** | `PER_NATIONALITIES_TL` | `PER_CITIZENSHIPS` + `FND_COMMON_LOOKUPS` |
| **COUNTRY_OF_BIRTH** | `PER_NATIONALITIES_TL` | `PER_CITIZENSHIPS` + `FND_TERRITORIES_VL` |

**CORRECT SQL Pattern (SUBQUERIES):**

**Nationality:**
```sql
(SELECT LISTAGG(LKP.MEANING,', ') WITHIN GROUP (ORDER BY LKP.MEANING)
 FROM PER_CITIZENSHIPS PC, FND_COMMON_LOOKUPS LKP
 WHERE PC.PERSON_ID = PAPF.PERSON_ID
   AND PC.CITIZENSHIP_STATUS = 'A'
   AND LKP.LOOKUP_CODE = PC.LEGISLATION_CODE
   AND LKP.LOOKUP_TYPE = 'NATIONALITY'
) AS NATIONALITY
```

**Country of Birth:**
```sql
(SELECT LISTAGG(FTV.TERRITORY_SHORT_NAME,', ') WITHIN GROUP (ORDER BY FTV.TERRITORY_SHORT_NAME)
 FROM PER_CITIZENSHIPS PC, FND_TERRITORIES_VL FTV
 WHERE PC.PERSON_ID = PAPF.PERSON_ID
   AND PC.CITIZENSHIP_STATUS = 'A'
   AND PC.LEGISLATION_CODE = FTV.TERRITORY_CODE
) AS COUNTRY_OF_BIRTH
```

**Key Points:**
- ❌ `PER_NATIONALITIES_TL` table does NOT exist or is not the correct approach
- ✅ Use `PER_CITIZENSHIPS` for both fields
- ✅ Use `LISTAGG` to handle multiple citizenships
- ✅ Filter by `CITIZENSHIP_STATUS = 'A'` (Active)

---

### 5. PERSON TYPE

| Field | ❌ WRONG Approach | ✅ CORRECT Approach |
|-------|------------------|---------------------|
| **PERSON_TYPE** | Via `PER_ALL_PEOPLE_F` | Via `PER_ALL_ASSIGNMENTS_F` |

**CORRECT SQL Pattern:**
```sql
SELECT
    PPTTL.USER_PERSON_TYPE AS PERSON_TYPE
FROM
    PER_ALL_PEOPLE_F PAPF,
    PER_ALL_ASSIGNMENTS_F PAAF,
    PER_PERSON_TYPES_TL PPTTL
WHERE
    PAPF.PERSON_ID = PAAF.PERSON_ID
AND PAAF.PERSON_TYPE_ID = PPTTL.PERSON_TYPE_ID
AND PAAF.PRIMARY_FLAG = 'Y'
AND PAAF.ASSIGNMENT_TYPE = 'E'
AND PPTTL.LANGUAGE = 'US'
AND :P_EFFECTIVE_DATE BETWEEN PAAF.EFFECTIVE_START_DATE AND PAAF.EFFECTIVE_END_DATE
```

**Key Points:**
- ❌ **NOT** from `PER_ALL_PEOPLE_F.PERSON_TYPE_ID`
- ✅ Get via `PER_ALL_ASSIGNMENTS_F.PERSON_TYPE_ID`
- ✅ Must filter: `PRIMARY_FLAG = 'Y'` and `ASSIGNMENT_TYPE = 'E'`

---

### 6. DEPARTMENT

| Field | ❌ WRONG Join | ✅ CORRECT Join |
|-------|--------------|-----------------|
| **DEPARTMENT** | `PAAF.DEPARTMENT_ID = PD.DEPARTMENT_ID` | `PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID` |

**CORRECT SQL Pattern:**
```sql
SELECT
    PD.NAME AS DEPARTMENT_NAME
FROM
    PER_ALL_ASSIGNMENTS_F PAAF,
    PER_DEPARTMENTS PD
WHERE
    PAAF.ORGANIZATION_ID = PD.ORGANIZATION_ID(+)
```

**Key Points:**
- ❌ `PD.DEPARTMENT_ID` column does NOT exist in `PER_DEPARTMENTS`
- ✅ Join via `ORGANIZATION_ID`

---

## 📋 COMPLETE TABLE REFERENCE

### Tables with PERSON_ID Join
| Table | Date-Tracked? | Key Field | Notes |
|-------|---------------|-----------|-------|
| `PER_ALL_PEOPLE_F` | ✅ Yes | Base person record | Primary table |
| `PER_PEOPLE_LEGISLATIVE_F` | ✅ Yes | SEX, MARITAL_STATUS | Legislative fields |
| `PER_PERSONS` | ❌ No | DATE_OF_BIRTH | Base person data (not date-tracked) |
| `PER_CITIZENSHIPS` | ❌ No | NATIONALITY, COUNTRY | Via subquery |
| `PER_RELIGIONS` | ❌ No | RELIGION | Via subquery with lookup |
| `PER_NATIONAL_IDENTIFIERS` | ❌ No | EMIRATES_ID | Dual join (PRIMARY_NID_ID + PERSON_ID) |

---

## 🔍 VALIDATION CHECKLIST

Before generating any SQL query involving personal information:

- [ ] SEX from `PER_PEOPLE_LEGISLATIVE_F.SEX` (NOT PAPF)
- [ ] DATE_OF_BIRTH from `PER_PERSONS.DATE_OF_BIRTH` (NOT PAPF)
- [ ] MARITAL_STATUS from `PER_PEOPLE_LEGISLATIVE_F.MARITAL_STATUS` (NOT PAPF)
- [ ] RELIGION via subquery from `PER_RELIGIONS` + `FND_LOOKUP_VALUES`
- [ ] NATIONALITY via subquery from `PER_CITIZENSHIPS` + `FND_COMMON_LOOKUPS`
- [ ] COUNTRY_OF_BIRTH via subquery from `PER_CITIZENSHIPS` + `FND_TERRITORIES_VL`
- [ ] EMIRATES_ID with BOTH `PRIMARY_NID_ID` and `PERSON_ID` joins
- [ ] PERSON_TYPE via `PER_ALL_ASSIGNMENTS_F` (NOT from PAPF)
- [ ] DEPARTMENT via `ORGANIZATION_ID` (NOT DEPARTMENT_ID)
- [ ] `PER_PERSONS` has NO date-track filtering

---

## 🚨 COMMON MISTAKES TO AVOID

### Mistake 1: Assuming PAPF has all personal fields
```sql
-- ❌ WRONG
SELECT PAPF.SEX, PAPF.DATE_OF_BIRTH, PAPF.MARITAL_STATUS

-- ✅ CORRECT
SELECT PPLF.SEX, PP.DATE_OF_BIRTH, PPLF.MARITAL_STATUS
```

### Mistake 2: Using RELIGION_ID join
```sql
-- ❌ WRONG
AND PAPF.RELIGION_ID = PR.RELIGION_ID(+)

-- ✅ CORRECT
(SELECT FLV.MEANING FROM PER_RELIGIONS PR, FND_LOOKUP_VALUES FLV
 WHERE PR.PERSON_ID = PAPF.PERSON_ID ...)
```

### Mistake 3: Using PER_NATIONALITIES_TL
```sql
-- ❌ WRONG - This table doesn't exist or is wrong approach
PER_NATIONALITIES_TL

-- ✅ CORRECT
PER_CITIZENSHIPS + FND_COMMON_LOOKUPS
```

### Mistake 4: Single join for National Identifier
```sql
-- ❌ WRONG
AND PAPF.PRIMARY_NID_ID = PNI.NATIONAL_IDENTIFIER_ID(+)

-- ✅ CORRECT
AND PAPF.PRIMARY_NID_ID = PNI.NATIONAL_IDENTIFIER_ID(+)
AND PNI.PERSON_ID(+) = PAPF.PERSON_ID
```

---

## 📚 REFERENCE SOURCE

All patterns in this document are extracted from:
- **File:** `New SQL Code/All Employee detail with salary Query Must use.sql`
- **Status:** Production-validated
- **Authority:** These are working production patterns, NOT assumptions

---

## ⚠️ IMPORTANT NOTES

1. **ALWAYS** reference this document when generating HCM personal data queries
2. **DO NOT** assume field locations based on table names
3. **VALIDATE** against the reference query before finalizing SQL
4. **These patterns are MANDATORY** - they come from production code

---

**END OF CRITICAL TABLE MAPPINGS**
