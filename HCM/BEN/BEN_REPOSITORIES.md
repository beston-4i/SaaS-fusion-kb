# Benefits Repository Patterns

**Purpose:** Standardized CTEs for extracting Benefits data.

---

## 1. Active Enrollments
*Retrieves currently active benefit enrollments.*

```sql
BEN_ENROLL_MASTER AS (
    SELECT /*+ qb_name(BEN_ENR) MATERIALIZE */
           PEN.PERSON_ID
          ,PEN.PL_ID
          ,PEN.OIPL_ID
          ,PEN.PRTT_ENRT_RSLT_ID
          ,PL.NAME AS PLAN_NAME
          ,OPT.NAME AS OPTION_NAME
    FROM   BEN_PRTT_ENRT_RSLT_F PEN
          ,BEN_PL_F PL
          ,BEN_OIPL_F OIPL
          ,BEN_OPT_F OPT
    WHERE  TRUNC(SYSDATE) BETWEEN PEN.EFFECTIVE_START_DATE AND PEN.EFFECTIVE_END_DATE
      AND  PEN.PRTT_ENRT_RSLT_STAT_CD IS NULL -- Active
      AND  PEN.ENRT_CVG_THRU_DT = TO_DATE('4712-12-31', 'YYYY-MM-DD') -- Through end of time
      AND  PEN.PL_ID = PL.PL_ID
      AND  TRUNC(SYSDATE) BETWEEN PL.EFFECTIVE_START_DATE AND PL.EFFECTIVE_END_DATE
      AND  PEN.OIPL_ID = OIPL.OIPL_ID(+)
      AND  OIPL.OPT_ID = OPT.OPT_ID(+)
)
```
