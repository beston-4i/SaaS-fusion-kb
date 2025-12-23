# HCM Module Instructions

**Domain:** Oracle Fusion Human Capital Management
**Location:** `FUSION_SAAS/HCM/`

---

## 1. 📂 Module Navigation (Routes)

| Sub-Module | Instruction File | Repository File | Template File |
|------------|------------------|-----------------|---------------|
| **Core HR** | [HR_MASTER](HR/HR_MASTER.md) | [HR_REPOS](HR/HR_REPOSITORIES.md) | [HR_TMPL](HR/HR_TEMPLATES.md) |
| **Payroll** | [PAY_MASTER](PAY/PAY_MASTER.md) | [PAY_REPOS](PAY/PAY_REPOSITORIES.md) | [PAY_TMPL](PAY/PAY_TEMPLATES.md) |
| **Benefits** | [BEN_MASTER](BEN/BEN_MASTER.md) | [BEN_REPOS](BEN/BEN_REPOSITORIES.md) | [BEN_TMPL](BEN/BEN_TEMPLATES.md) |
| **Absence** | [ABSENCE_MASTER](ABSENCE/ABSENCE_MASTER.md) | [ABSENCE_REPOS](ABSENCE/ABSENCE_REPOSITORIES.md) | [ABSENCE_TMPL](ABSENCE/ABSENCE_TEMPLATES.md) |
| **Time & Labor** | [TL_MASTER](TIME_LABOR/TL_MASTER.md) | [TL_REPOS](TIME_LABOR/TL_REPOSITORIES.md) | [TL_TMPL](TIME_LABOR/TL_TEMPLATES.md) |
| **Compensation** | [CMP_MASTER](COMPENSATION/CMP_MASTER.md) | [CMP_REPOS](COMPENSATION/CMP_REPOSITORIES.md) | [CMP_TMPL](COMPENSATION/CMP_TEMPLATES.md) |

---

## 2. 🔗 Shared Integration Rules (Cross-Module)

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
