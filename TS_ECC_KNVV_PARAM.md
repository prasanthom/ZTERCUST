# Technical Specification — Externalize VKORG / VTWEG via ZAPOPARAM

## Document Control

| Item | Detail |
|------|--------|
| **Program** | `ZAPO_REG_MAP_UPLOAD` |
| **Systems** | AD1 (change), RD2 (no code change) |
| **Related docs** | [FS_ECC_KNVV_PARAM.md](./FS_ECC_KNVV_PARAM.md), [TEST_ECC_KNVV_PARAM.md](./TEST_ECC_KNVV_PARAM.md) |

---

## 1. Objects Changed

| System | Object | Change type |
|--------|--------|-------------|
| AD1 | `ZAPO_REG_MAP_UPLOAD` | Add filter helper; replace hardcoded VKORG/VTWEG |
| AD1 | `ZAPOPARAM` | New customizing rows (`ECC_KNVV`) |
| RD2 | — | No code change |

---

## 2. New Data Structures (AD1)

Add to program top / class `LCL_ADVPAY`:

```abap
TYPES: BEGIN OF ty_ecc_knvv_filter,
         bsg    TYPE zbsg,
         field  TYPE char10,  " VKORG | VTWEG | KNVV_KEY
         sign   TYPE c LENGTH 1,
         opt    TYPE c LENGTH 2,
         low    TYPE char30,
         high   TYPE char30,
       END OF ty_ecc_knvv_filter.

TYPES: BEGIN OF ty_knvv_key,
         vkorg TYPE vkorg,
         vtweg TYPE vtweg,
       END OF ty_knvv_key.

DATA: gt_ecc_filter TYPE STANDARD TABLE OF ty_ecc_knvv_filter,
      gt_knvv_key   TYPE STANDARD TABLE OF ty_knvv_key.
```

---

## 3. New Methods in `LCL_ADVPAY`

### Class definition additions

```abap
load_ecc_knvv_filters
  IMPORTING iv_bsg TYPE zbsg OPTIONAL,
build_selopt_from_filter
  IMPORTING iv_field TYPE char10
            iv_bsg    TYPE zbsg OPTIONAL
  EXPORTING et_selopt TYPE rsdsselopt_t,
split_values_to_selopt
  IMPORTING iv_sign   TYPE c
            iv_values TYPE char30
  EXPORTING et_selopt TYPE rsdsselopt_t.
```

### Form (or private static method)

```abap
FORM fill_default_ecc_knvv_filters USING iv_bsg TYPE zbsg.
```

---

## 4. ZAPOPARAM Read Logic

### SELECT

```abap
SELECT * FROM zapoparam INTO TABLE lt_param
  WHERE param1 = 'DP'
    AND param2 = 'ZAPO_REG_MAP_UPLOAD'
    AND param3 = 'ECC_KNVV'
    AND active_flag = 'X'
    AND ( param4 = iv_bsg OR param4 = '*' ).
```

### Merge order

1. Load rows with `PARAM4 = '*'`.
2. Overlay rows with `PARAM4 = iv_bsg` (delete existing entry with same `field`, then append).

### Field mapping

| PARAM5 | Internal `field` | Mapping |
|--------|------------------|---------|
| `VKORG` | `VKORG` | `sign = VALUE1`, split `VALUE2` by comma → selopt `EQ` rows |
| `VTWEG` | `VTWEG` | `sign = VALUE1`, split `VALUE2` by comma → selopt `EQ` rows |
| `KNVV_KEY` | `KNVV_KEY` | `low = VALUE1` (vkorg), `high = VALUE2` (vtweg) → `gt_knvv_key` |

**Explicitly excluded:** `PARAM5 = SPART` — not read or applied.

---

## 5. Method Implementations

### 5.1 `load_ecc_knvv_filters`

```abap
METHOD load_ecc_knvv_filters.
  DATA: lt_param TYPE TABLE OF zapoparam,
        ls_param TYPE zapoparam,
        ls_filt  TYPE ty_ecc_knvv_filter.

  CLEAR: gt_ecc_filter, gt_knvv_key.

  SELECT * FROM zapoparam INTO TABLE lt_param
    WHERE param1 = 'DP'
      AND param2 = 'ZAPO_REG_MAP_UPLOAD'
      AND param3 = 'ECC_KNVV'
      AND active_flag = 'X'
      AND ( param4 = iv_bsg OR param4 = '*' ).

  " Defaults first
  LOOP AT lt_param INTO ls_param WHERE param4 = '*'.
    PERFORM map_zapoparam_to_filter USING ls_param CHANGING ls_filt.
    APPEND ls_filt TO gt_ecc_filter.
  ENDLOOP.

  " BSG-specific overlay
  LOOP AT lt_param INTO ls_param WHERE param4 = iv_bsg AND param4 <> '*'.
    PERFORM map_zapoparam_to_filter USING ls_param CHANGING ls_filt.
    DELETE gt_ecc_filter WHERE field = ls_filt-field AND bsg = ls_filt-bsg.
    APPEND ls_filt TO gt_ecc_filter.
  ENDLOOP.

  " Build KNVV key table for PMR-style reads
  LOOP AT gt_ecc_filter INTO ls_filt WHERE field = 'KNVV_KEY'.
    APPEND VALUE #( vkorg = ls_filt-low vtweg = ls_filt-high ) TO gt_knvv_key.
  ENDLOOP.

  IF gt_ecc_filter IS INITIAL.
    PERFORM fill_default_ecc_knvv_filters USING iv_bsg.
  ENDIF.
ENDMETHOD.
```

### 5.2 `map_zapoparam_to_filter` (FORM)

```abap
FORM map_zapoparam_to_filter USING is_param TYPE zapoparam
                             CHANGING cs_filt TYPE ty_ecc_knvv_filter.
  CLEAR cs_filt.
  cs_filt-bsg   = is_param-param4.
  cs_filt-field = is_param-param5.
  cs_filt-sign  = is_param-value1.
  cs_filt-low   = is_param-value2.
  cs_filt-high  = is_param-value3.
ENDFORM.
```

### 5.3 `build_selopt_from_filter`

```abap
METHOD build_selopt_from_filter.
  DATA ls_filt TYPE ty_ecc_knvv_filter.
  CLEAR et_selopt.
  LOOP AT gt_ecc_filter INTO ls_filt WHERE field = iv_field.
    IF ls_filt-low CS ','.
      me->split_values_to_selopt(
        EXPORTING iv_sign = ls_filt-sign iv_values = ls_filt-low
        IMPORTING et_selopt = DATA(lt_part) ).
      APPEND LINES OF lt_part TO et_selopt.
    ELSE.
      APPEND VALUE #(
        sign   = ls_filt-sign
        option = COND #( WHEN ls_filt-high IS NOT INITIAL THEN 'BT' ELSE 'EQ' )
        low    = ls_filt-low
        high   = ls_filt-high
      ) TO et_selopt.
    ENDIF.
  ENDLOOP.
ENDMETHOD.
```

### 5.4 `split_values_to_selopt`

```abap
METHOD split_values_to_selopt.
  DATA: lt_vals TYPE TABLE OF string,
        lv_val  TYPE string.
  CLEAR et_selopt.
  SPLIT iv_values AT ',' INTO TABLE lt_vals.
  LOOP AT lt_vals INTO lv_val.
    CONDENSE lv_val.
    APPEND VALUE #( sign = iv_sign option = 'EQ' low = lv_val ) TO et_selopt.
  ENDLOOP.
ENDMETHOD.
```

### 5.5 `fill_default_ecc_knvv_filters`

```abap
FORM fill_default_ecc_knvv_filters USING iv_bsg TYPE zbsg.
  REFRESH gt_ecc_filter.
  APPEND VALUE #( field = 'VKORG' sign = 'I' low = '1010' ) TO gt_ecc_filter.
  APPEND VALUE #( field = 'VKORG' sign = 'I' low = '1020' ) TO gt_ecc_filter.
  APPEND VALUE #( field = 'VTWEG' sign = 'E' low = '30'  ) TO gt_ecc_filter.
  APPEND VALUE #( field = 'VTWEG' sign = 'E' low = '80'  ) TO gt_ecc_filter.
  IF iv_bsg = 'PMR'.
    APPEND VALUE #( field = 'KNVV_KEY' low = '1010' high = '20' ) TO gt_ecc_filter.
    APPEND VALUE #( field = 'KNVV_KEY' low = '1010' high = '25' ) TO gt_ecc_filter.
    APPEND VALUE #( field = 'KNVV_KEY' low = '1020' high = '20' ) TO gt_ecc_filter.
    APPEND VALUE #( field = 'KNVV_KEY' low = '1020' high = '25' ) TO gt_ecc_filter.
    REFRESH gt_knvv_key.
    APPEND VALUE #( vkorg = '1010' vtweg = '20' ) TO gt_knvv_key.
    APPEND VALUE #( vkorg = '1010' vtweg = '25' ) TO gt_knvv_key.
    APPEND VALUE #( vkorg = '1020' vtweg = '20' ) TO gt_knvv_key.
    APPEND VALUE #( vkorg = '1020' vtweg = '25' ) TO gt_knvv_key.
  ENDIF.
ENDFORM.
```

---

## 6. Code Change Points

### TS-01 — `fetch_ecc` (MDG / `update_history1`)

**Remove** (~lines 3705–3751):

- Hardcoded VKORG `1010`, `1020`
- Hardcoded VTWEG exclude `30`, `80`

**Add** before SPART `s_div_u` loop:

```abap
me->load_ecc_knvv_filters( iv_bsg = lw_bsg ).

DATA lt_vkorg TYPE rsdsselopt_t.
me->build_selopt_from_filter(
  EXPORTING iv_field = 'VKORG'
  IMPORTING et_selopt = lt_vkorg ).
IF lt_vkorg IS NOT INITIAL.
  APPEND VALUE #( fieldname = 'KNVV~VKORG' selopt_t = lt_vkorg ) TO lt_frange.
ENDIF.

DATA lt_vtweg TYPE rsdsselopt_t.
me->build_selopt_from_filter(
  EXPORTING iv_field = 'VTWEG'
  IMPORTING et_selopt = lt_vtweg ).
IF lt_vtweg IS NOT INITIAL.
  APPEND VALUE #( fieldname = 'KNVV~VTWEG' selopt_t = lt_vtweg ) TO lt_frange.
ENDIF.
```

**Unchanged:** SPART from `s_div_u` → `KNVV~SPART`.

---

### TS-02 — `update_history` (PMR KNVV key table)

**Remove** (~lines 3157–3206): six hardcoded `lt_knvv_tmp` APPEND blocks.

**Replace:**

```abap
IF lw_bsg = 'PMR'.
  me->load_ecc_knvv_filters( iv_bsg = lw_bsg ).
  IF gt_knvv_key IS NOT INITIAL.
    lt_knvv_tmp = gt_knvv_key.
  ELSE.
    PERFORM fill_default_ecc_knvv_filters USING lw_bsg.
    lt_knvv_tmp = gt_knvv_key.
  ENDIF.
ENDIF.
```

**Unchanged:** SPART-driven ODS read, `lt_ods` division 31/50 logic, `s_div_u` filters.

---

### TS-03 — `data_knvv_ecc` (ODS path)

**Unchanged:** SPART built from `s_div_u` / tercust (`lt_spart`).

**Add** (optional enhancement): after `load_ecc_knvv_filters`, append VKORG/VTWEG to RFC `lt_options` / `lt_tmp_options` using same selopt builder to reduce RD2 result volume.

---

### TS-04 — `val_shpdiv`

**No change.** SPART hardcoding (22/23/24) remains upload/PMR context driven per FS.

---

## 7. RD2 Technical Notes

| Item | Detail |
|------|--------|
| `Z_APO_FM_DYN_SELECT_QUERY` | Receives `IT_WHERE_COND` from AD1; no RD2 change |
| `RFC_READ_TABLE` on `KNVV` | Receives `OPTIONS` from AD1; no RD2 change |
| ZAPOPARAM on RD2 | Not required unless landscape policy copies customizing AD1 → RD2 |

---

## 8. Transport List

| # | Object | System | Description |
|---|--------|--------|-------------|
| 1 | `ZAPO_REG_MAP_UPLOAD` | AD1 | Program change |
| 2 | ZAPOPARAM entries (`ECC_KNVV`) | AD1 | Customizing data |

---

## 9. Error Handling

| Condition | Behaviour |
|-----------|-----------|
| No ZAPOPARAM entries | `fill_default_ecc_knvv_filters` |
| Invalid sign (not I/E) | Skip row in mapping |
| Empty VKORG and VTWEG after load | Use fallback |
| RFC returns no data | Existing messages unchanged |

---

## 10. Suggested PR Metadata

**Title:** `feat(ZAPO_REG_MAP_UPLOAD): externalize VKORG/VTWEG filters via ZAPOPARAM`

**Labels:** `enhancement`, `APO`, `ZAPOPARAM`, `regression-test-required`

**Files in PR:**

- `ZAPO_REG_MAP_UPLOAD` (AD1)
- `docs/ZAPO_REG_MAP_UPLOAD/*.md`
- ZAPOPARAM transport (DEV → QA → PRD)
