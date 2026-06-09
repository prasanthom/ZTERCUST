# Test Scenarios — Externalize VKORG / VTWEG via ZAPOPARAM

## Document Control

| Item | Detail |
|------|--------|
| **Program** | `ZAPO_REG_MAP_UPLOAD` |
| **Systems** | AD1 (execute), RD2 (KNVV source) |
| **Related docs** | [FS_ECC_KNVV_PARAM.md](./FS_ECC_KNVV_PARAM.md), [TS_ECC_KNVV_PARAM.md](./TS_ECC_KNVV_PARAM.md) |

---

## 1. Test Environment

| System | Role |
|--------|------|
| AD1 | Execute `ZAPO_REG_MAP_UPLOAD` |
| RD2 | Source `KNVV` data |
| ZAPOPARAM | AD1 customizing (`PARAM3 = ECC_KNVV`) |

### Prerequisites

- [ ] Known ship-to + division in `ZAPO_PC_TERCUST` with `zexception_flag <> 'X'`
- [ ] Known `VKBUR` in RD2 `KNVV` for that ship-to / division / VKORG / VTWEG
- [ ] RFC AD1 → RD2 working (`ZAPO_GET_RFC`, `iv_system = 'ECC'`)
- [ ] Tester has access to ZAPOPARAM maintenance (SM30 or equivalent)

### Test Data Checklist

| Data element | Where to verify |
|--------------|-----------------|
| Ship-to (`zpaposhp`) | `ZAPO_PC_TERCUST` |
| Division (`zpdi`) | Screen `s_div_u` / BSG setup |
| RD2 `VKBUR` | SE16 `KNVV` on RD2 |
| Target `P_REG_OFF` | `ZAPO_PC_TERCUST` after run |

---

## 2. Test Cases

### TC-01 — Baseline: no ZAPOPARAM entries (fallback)

| Field | Value |
|-------|-------|
| **Objective** | Confirm backward compatibility when no `ECC_KNVV` rows exist |
| **Precondition** | All `ECC_KNVV` rows deactivated or removed |
| **Steps** | 1. Run Update History (MDG) for a non-PMR BSG<br>2. Compare `P_REG_OFF` before and after |
| **Expected** | Behaviour identical to pre-change (VKORG 1010/1020, VTWEG excl 30/80) |
| **Priority** | P1 |

---

### TC-02 — VKORG include via ZAPOPARAM

| Field | Value |
|-------|-------|
| **Objective** | VKORG filter driven by param |
| **Precondition** | `ECC_KNVV \| * \| VKORG \| I \| 1010` only (deactivate 1020) |
| **Steps** | 1. Run Update History (MDG)<br>2. Verify only VKORG 1010 KNVV rows influence `P_REG_OFF` |
| **Expected** | VKORG 1020 records ignored |
| **Priority** | P1 |

---

### TC-03 — VTWEG exclusion via ZAPOPARAM

| Field | Value |
|-------|-------|
| **Objective** | VTWEG exclude driven by param |
| **Precondition** | `ECC_KNVV \| * \| VTWEG \| E \| 25` (in addition to or replacing default 30/80) |
| **Steps** | 1. Run Update History (ODS) for PMR BSG<br>2. Check `P_REG_OFF` for ship-to with VTWEG 25 in RD2 |
| **Expected** | KNVV rows with VTWEG 25 excluded from fetch |
| **Priority** | P1 |

---

### TC-04 — BSG-specific override

| Field | Value |
|-------|-------|
| **Objective** | BSG row overrides default `*` |
| **Precondition** | Default: `* \| VKORG \| I \| 1010,1020`<br>PMR: `PMR \| VKORG \| I \| 1010` |
| **Steps** | 1. Run MDG path for PMR<br>2. Run MDG path for non-PMR BSG |
| **Expected** | PMR uses 1010 only; other BSG uses 1010+1020 |
| **Priority** | P1 |

---

### TC-05 — PMR KNVV_KEY matrix from ZAPOPARAM

| Field | Value |
|-------|-------|
| **Objective** | PMR lookup combinations configurable |
| **Precondition** | Four `KNVV_KEY` rows for PMR (1010/20, 1010/25, 1020/20, 1020/25) |
| **Steps** | 1. Run Update History (ODS) for PMR<br>2. Verify `P_REG_OFF` = RD2 `VKBUR` for first matching key |
| **Expected** | Same result as legacy hardcoded matrix |
| **Priority** | P1 |

---

### TC-06 — SPART remains screen-driven (regression)

| Field | Value |
|-------|-------|
| **Objective** | SPART not driven by ZAPOPARAM |
| **Precondition** | BSG with divisions 22, 23 only (not 24) |
| **Steps** | 1. Run Update History<br>2. Confirm only divisions 22/23 processed |
| **Expected** | SPART filter follows `s_div_u` only |
| **Priority** | P1 |

---

### TC-07 — SPART not affected by VKORG param change

| Field | Value |
|-------|-------|
| **Objective** | VKORG param does not alter division scope |
| **Precondition** | VKORG param = `1010` only; screen includes division 24 |
| **Steps** | 1. Run Update History for PMR<br>2. Verify division 24 still in scope if selected on screen |
| **Expected** | Division scope unchanged; only VKORG filtering of KNVV changes |
| **Priority** | P2 |

---

### TC-08 — Division 31 / 50 (ODS path)

| Field | Value |
|-------|-------|
| **Objective** | ODS divisions still use BI source for P_REG_OFF |
| **Precondition** | ODS path with div 31 or 50 in selection |
| **Steps** | 1. Run ODS Update History<br>2. Verify `P_REG_OFF` source |
| **Expected** | `P_REG_OFF` from ODS `/bic/p_reg_off`, not RD2 `VKBUR` |
| **Priority** | P1 |

---

### TC-09 — `zexception_flag = X`

| Field | Value |
|-------|-------|
| **Objective** | Exception records excluded |
| **Precondition** | Test record with `zexception_flag = X` |
| **Steps** | 1. Run Update History |
| **Expected** | Record excluded from RD2-driven update |
| **Priority** | P2 |

---

### TC-10 — Order block in KNVV

| Field | Value |
|-------|-------|
| **Objective** | Blocked KNVV not updated |
| **Precondition** | Ship-to with `AUFSD`, `LIFSD`, `FAKSD`, or `CASSD` set in RD2 |
| **Steps** | 1. Run MDG update |
| **Expected** | `P_REG_OFF` not updated; existing block/delete logic applies |
| **Priority** | P2 |

---

### TC-11 — Excel upload (regression)

| Field | Value |
|-------|-------|
| **Objective** | Upload path unaffected |
| **Precondition** | Excel with `P_REG_OFF` in column 006 |
| **Steps** | 1. Run with `p_rupd` |
| **Expected** | No impact from `ECC_KNVV` param |
| **Priority** | P2 |

---

### TC-12 — Invalid / empty ZAPOPARAM

| Field | Value |
|-------|-------|
| **Objective** | Graceful handling of bad config |
| **Precondition** | `VKORG \| I \|` with empty VALUE2 |
| **Steps** | 1. Run program |
| **Expected** | Fallback defaults applied; no dump |
| **Priority** | P2 |

---

### TC-13 — P_REG_OFF end-to-end

| Field | Value |
|-------|-------|
| **Objective** | Full field update validation |
| **Precondition** | Ship-to where RD2 `VKBUR` ≠ current `P_REG_OFF`; param allows that KNVV row |
| **Steps** | 1. Note current `P_REG_OFF`<br>2. Run Update History<br>3. Check `ZAPO_PC_TERCUST` |
| **Expected** | `P_REG_OFF` = RD2 `VKBUR` (or `NONE` if blank) |
| **Priority** | P1 |

---

## 3. Test Execution Matrix

| TC ID | Path | BSG | ZAPOPARAM variant | P1/P2 |
|-------|------|-----|-------------------|-------|
| TC-01 | MDG | Any | None (fallback) | P1 |
| TC-02 | MDG | Any | VKORG = 1010 only | P1 |
| TC-03 | ODS | PMR | VTWEG excl 25 | P1 |
| TC-04 | MDG | PMR + other | BSG override | P1 |
| TC-05 | ODS | PMR | KNVV_KEY rows | P1 |
| TC-06 | ODS/MDG | Any | Default | P1 |
| TC-07 | ODS | PMR | VKORG narrowed | P2 |
| TC-08 | ODS | Any | Default | P1 |
| TC-09 | MDG | Any | Default | P2 |
| TC-10 | MDG | Any | Default | P2 |
| TC-11 | Excel | Any | Any | P2 |
| TC-12 | MDG | Any | Invalid row | P2 |
| TC-13 | MDG/ODS | PMR | Default | P1 |

---

## 4. Test Evidence Template

Copy into PR description or test ticket:

```markdown
## Test Run

| Field | Value |
|-------|-------|
| Date | YYYY-MM-DD |
| Environment | DEV / QA / PRD |
| Tester | <name> |
| Program version | <transport / package> |

### ZAPOPARAM active during test

| PARAM4 | PARAM5 | VALUE1 | VALUE2 | VALUE3 |
|--------|--------|--------|--------|--------|
| * | VKORG | I | 1010,1020 | |
| * | VTWEG | E | 30,80 | |
| PMR | KNVV_KEY | | 1010 | 20 |

### Results

| TC ID | Result | Evidence | Notes |
|-------|--------|----------|-------|
| TC-01 | Pass / Fail | | |
| TC-06 | Pass / Fail | | |
| TC-13 | Pass / Fail | SE16 screenshot | |

### Defects

| ID | TC | Description | Status |
|----|-----|-------------|--------|
| | | | |
```

---

## 5. Regression Scope

| Area | Must pass |
|------|-----------|
| Update History MDG (`update_history1`) | TC-01, TC-02, TC-04, TC-13 |
| Update History ODS (`update_history`) | TC-03, TC-05, TC-06, TC-08 |
| SPART screen logic | TC-06, TC-07 |
| Excel upload | TC-11 |
| Exception / block handling | TC-09, TC-10 |

---

## 6. Sign-off Criteria (PR merge gate)

- [ ] All **P1** test cases passed in QA
- [ ] TC-06 (SPART regression) passed
- [ ] TC-13 (`P_REG_OFF` E2E) passed
- [ ] ZAPOPARAM entries transported to AD1 QA
- [ ] No RD2 ABAP objects in transport
- [ ] FS / TS / TEST docs linked in PR

---

## 7. Suggested GitHub PR Checklist

```markdown
## PR Checklist

### Code
- [ ] `load_ecc_knvv_filters` implemented
- [ ] `fetch_ecc` uses param-driven VKORG/VTWEG
- [ ] `update_history` PMR keys from `KNVV_KEY` param
- [ ] SPART logic untouched (`s_div_u`)
- [ ] Fallback defaults in place

### Config
- [ ] ZAPOPARAM `ECC_KNVV` rows created in DEV
- [ ] Transport request created for AD1

### Test
- [ ] TC-01, TC-06, TC-13 executed
- [ ] Evidence attached to PR / ticket

### Docs
- [ ] FS_ECC_KNVV_PARAM.md
- [ ] TS_ECC_KNVV_PARAM.md
- [ ] TEST_ECC_KNVV_PARAM.md
```
