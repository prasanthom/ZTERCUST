# ZAPO_REG_MAP_UPLOAD — ECC KNVV Parameterization

Documentation for externalizing **VKORG** and **VTWEG** include/exclude rules into **ZAPOPARAM** for program `ZAPO_REG_MAP_UPLOAD`.

**SPART remains screen-driven** (`s_div_u`) and is out of scope for parameterization.

## Documents

| Document | Description |
|----------|-------------|
| [FS_ECC_KNVV_PARAM.md](./FS_ECC_KNVV_PARAM.md) | Functional specification |
| [TS_ECC_KNVV_PARAM.md](./TS_ECC_KNVV_PARAM.md) | Technical specification and ABAP change points |
| [TEST_ECC_KNVV_PARAM.md](./TEST_ECC_KNVV_PARAM.md) | Test scenarios and PR sign-off criteria |

## Quick reference — ZAPOPARAM keys

```
PARAM1 = DP
PARAM2 = ZAPO_REG_MAP_UPLOAD
PARAM3 = ECC_KNVV
PARAM4 = * | <BSG>
PARAM5 = VKORG | VTWEG | KNVV_KEY
```

## Suggested PR title

```
feat(ZAPO_REG_MAP_UPLOAD): externalize VKORG/VTWEG filters via ZAPOPARAM
```

## Systems

| System | Change |
|--------|--------|
| AD1 | `ZAPO_REG_MAP_UPLOAD` + ZAPOPARAM entries |
| RD2 | No ABAP change |
