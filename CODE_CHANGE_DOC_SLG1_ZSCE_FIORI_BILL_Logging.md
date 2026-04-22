# Code Change Document — SLG1 Application Log (`ZSCE_FIORI_BILL`)

**Audience:** ABAP developer / transport owner  
**Target system:** SAP ECC 6.0 / NetWeaver **7.31** (`abap_731`)  
**Function module:** `Z_FIORI_SWM_OB_INV_GENRATE` (function group per SE80)

**Related rules:** Project **`ABAP Rules - 02-04-2026`** — see **§7**.

**Change type:** Observability only — **no** change to billing creation, `YTTSTX0002` updates, Fiori return tables, or `Z_SCM_YTTS_UPDATE` call logic.

---

## 1. Executive summary

| Item | Detail |
|------|--------|
| **Purpose** | After `CALL FUNCTION 'Z_PTC_FIORI_BILLING'`, persist **SLG1** traces: **per delivery** when possible; **report-level** External ID **`area|report_no`** when there is no usable delivery or no output (see **§3**). |
| **Log object / subobject** | `ZSCE_EWMLOG` / `ZSCE_FIORI_BILL` |
| **Wrapper FM** | `Z_APPLICATION_LOG_CREATE` (`I_ASYNC = space`) → `Z_APPLICATION_LOG_CREATE_ASYNC` / BAL; requires **`ZAPPLOGACT`**. |
| **Deliverables** | Constant `LC_SUBOBJECT_FIORI_BILL`, one `PERFORM`, **`FORM LOG_PTC_FIORI_BILLING_TRACE`**, **`FORM BUILD_AREA_REPORT_KEY`**, **`FORM APPEND_ZST_FIORI_BILL_ROW`** (after `ENDFUNCTION.`). |

---

## 2. Insertion points in `Z_FIORI_SWM_OB_INV_GENRATE`

### 2.1 Constant

**Where:** With `LC_OBJECT`, `LC_SUBOBJECT`, `LC_LGORT` (export ~**249–252**).

```abap
  CONSTANTS : lc_object     TYPE balobj_d   VALUE 'ZSCE_EWMLOG',
              lc_subobject  TYPE balsubobj  VALUE 'ZSCE_PTC',
              lc_subobject_fiori_bill TYPE balsubobj VALUE 'ZSCE_FIORI_BILL',
              lc_lgort      TYPE lgort_d    VALUE '7906'.
```

### 2.2 Call site

**Where:** Immediately after `CALL FUNCTION 'Z_PTC_FIORI_BILLING'` / `EX_LOG = LT_LOG`, **before** `IF LT_LOG IS NOT INITIAL.` (export ~**533–545**).  
**Context:** `LW_ZLOG_PROG_TABLE IS NOT INITIAL` → RPU OK (`READ TABLE ET_RETURN` with `TYPE = GC_E` → `SY-SUBRC <> 0`).

```abap
* SLG1 ZSCE_EWMLOG/ZSCE_FIORI_BILL (see LOG_PTC_FIORI_BILLING_TRACE)
      PERFORM log_ptc_fiori_billing_trace USING i_area i_report_no lt_output lt_log
                                                lc_object lc_subobject_fiori_bill.
```

### 2.3 Forms

After `ENDFUNCTION.` of `Z_FIORI_SWM_OB_INV_GENRATE`, same include as existing `FORM`s. **No** new `DATA` in the main FM for this feature.

---

## 3. Behaviour matrix (only valid cases)

| Case | Condition | External ID (`I_OBJECTKEY`) | Message body (summary) |
|------|------------|-------------------------------|-------------------------|
| **A** | `IT_OUTPUT` not initial **and** ≥1 row has **non-initial** `DELIVERY` | **Delivery** (`CONVERSION_EXIT_ALPHA_OUTPUT`, len ≤ 50) | Per delivery row: INPUT + `APPEND_ZST_FIORI_BILL_ROW` + `--- EX_LOG ---` + full `IT_LOG` (or `EX_LOG: (empty)`). |
| **D** | `IT_OUTPUT` not initial **and** **every** row has **initial** `DELIVERY` | **`area|report_no`** only — **`BUILD_AREA_REPORT_KEY`** (**no** `ITEM_NO` in key) | **Single** log: INPUT + line that output exists but delivery is initial + **all** output rows via `APPEND_ZST_FIORI_BILL_ROW` (loop with row index) + `--- EX_LOG ---` + full `IT_LOG` (or empty). |
| **B** | `IT_OUTPUT` **initial**, `IT_LOG` **not** initial | **`area|report_no`** (`BUILD_AREA_REPORT_KEY`) | INPUT + `EX_OUTPUT: (empty)` + `--- EX_LOG ---` + all `IT_LOG` lines |
| **C** | **Both** initial | **`area|report_no`** | Minimal: INPUT + `EX_OUTPUT: (empty)` + `--- EX_LOG ---` + `EX_LOG: (empty)` |

**Control flow (required):**

1. **`IF IT_OUTPUT IS NOT INITIAL.`**
   - Scan `IT_OUTPUT` once: set **`LV_HAS_DELIVERY = 'X'`** if any **`LW_LINE-DELIVERY` is not initial** (use **`DATA LV_HAS_DELIVERY TYPE C LENGTH 1.`**).
   - **`IF LV_HAS_DELIVERY = 'X'.`** → **Case A** (`LOOP`, `CONTINUE` when delivery initial).
   - **`ELSE.`** → **Case D** (single log, **`BUILD_AREA_REPORT_KEY`**, no item in external id).
2. **`ELSE.`** → **`IF IT_LOG IS NOT INITIAL`** → **Case B**; **`ELSE`** → **Case C**.

**Note:** Item numbers appear in **message** lines from **`APPEND_ZST_FIORI_BILL_ROW`** where applicable; only **Case A** uses delivery in **External ID**. Cases **B/C/D** use **`area|report_no`** only (never append **`ITEM_NO`** to **`I_OBJECTKEY`**).

---

## 4. Data elements and fields

### 4.1 INPUT (logged in all cases)

| Source | Log prefix |
|--------|------------|
| `IM_AREA` | `Z_PTC_FIORI_BILLING trace \| INPUT \| AREA` + value |
| `IM_REPORT_NO` | `INPUT \| REPORT_NO` + value |

### 4.2 `ZST_FIORI_BILLING` (`APPEND_ZST_FIORI_BILL_ROW`)

| Component | Message pattern |
|-----------|-----------------|
| `AREA` | `EX_OUTPUT row n \| AREA` … |
| `REPORT_NO` … `ERZET` | One line each (see §6.3) |

### 4.3 `ZST_LOG` (`EX_LOG`)

| Field | Maps to `BAPIRET2` |
|-------|---------------------|
| `TYPE` | `TYPE` (default `'I'`) |
| `MESSAGE` | `MESSAGE` |

### 4.4 `Z_APPLICATION_LOG_CREATE`

| Case | `I_OBJECTKEY` |
|------|----------------|
| A | Delivery (after exit) |
| D, B, C | `BUILD_AREA_REPORT_KEY` → `area|report_no` (≤ 50 chars) |

---

## 5. SAP customizing

| # | T-code | Action |
|---|--------|--------|
| 1 | **SLG0** | Subobject **`ZSCE_FIORI_BILL`** under **`ZSCE_EWMLOG`**. |
| 2 | **ZAPPLOGACT** | **`ZSCE_EWMLOG`** / **`ZSCE_FIORI_BILL`**, **ACTIVE** = **`X`**. |

---

## 6. Complete ABAP source

### 6.1 `FORM LOG_PTC_FIORI_BILLING_TRACE`

```abap
*&---------------------------------------------------------------------*
*&      Form  LOG_PTC_FIORI_BILLING_TRACE
*&---------------------------------------------------------------------*
*  Case A: at least one delivery in EX_OUTPUT -> one SLG1 per delivery
*  Case D: EX_OUTPUT rows but all DELIVERY initial -> one SLG1, key area|report
*  Case B: empty EX_OUTPUT, EX_LOG filled -> one SLG1, key area|report
*  Case C: both empty -> minimal SLG1, key area|report
*&---------------------------------------------------------------------*
FORM log_ptc_fiori_billing_trace USING iv_area TYPE yarea
                                        iv_report_no TYPE yreport_no
                                        it_output TYPE ztt_fiori_billing
                                        it_log TYPE ztt_log
                                        iv_object TYPE balobj_d
                                        iv_subobject TYPE balsubobj.

  DATA: lt_appl TYPE bapiret2_t,
        lw_appl TYPE bapiret2,
        lw_line TYPE zst_fiori_billing,
        lw_log  TYPE zst_log,
        lv_key  TYPE balnrext,
        lv_len  TYPE i,
        lv_has_delivery TYPE c LENGTH 1,
        lv_row TYPE i.

  IF it_output IS NOT INITIAL.

    CLEAR lv_has_delivery.
    LOOP AT it_output INTO lw_line.
      IF lw_line-delivery IS NOT INITIAL.
        lv_has_delivery = 'X'.
        EXIT.
      ENDIF.
    ENDLOOP.

    IF lv_has_delivery = 'X'.

      LOOP AT it_output INTO lw_line.
        IF lw_line-delivery IS INITIAL.
          CONTINUE.
        ENDIF.

        REFRESH lt_appl.

        CLEAR lw_appl.
        lw_appl-type = 'I'.
        CONCATENATE 'Z_PTC_FIORI_BILLING trace | INPUT | AREA' iv_area INTO lw_appl-message SEPARATED BY space.
        APPEND lw_appl TO lt_appl.

        CLEAR lw_appl.
        lw_appl-type = 'I'.
        CONCATENATE 'INPUT | REPORT_NO' iv_report_no INTO lw_appl-message SEPARATED BY space.
        APPEND lw_appl TO lt_appl.

        PERFORM append_zst_fiori_bill_row USING sy-tabix lw_line CHANGING lt_appl.

        CLEAR lw_appl.
        lw_appl-type = 'I'.
        lw_appl-message = '--- EX_LOG ---'.
        APPEND lw_appl TO lt_appl.

        IF it_log IS INITIAL.
          CLEAR lw_appl.
          lw_appl-type = 'I'.
          lw_appl-message = 'EX_LOG: (empty)'.
          APPEND lw_appl TO lt_appl.
        ELSE.
          LOOP AT it_log INTO lw_log.
            CLEAR lw_appl.
            lw_appl-type = lw_log-type.
            IF lw_appl-type IS INITIAL.
              lw_appl-type = 'I'.
            ENDIF.
            lw_appl-message = lw_log-message.
            APPEND lw_appl TO lt_appl.
          ENDLOOP.
        ENDIF.

        CLEAR lv_key.
        lv_key = lw_line-delivery.
        CALL FUNCTION 'CONVERSION_EXIT_ALPHA_OUTPUT'
          EXPORTING
            input  = lv_key
          IMPORTING
            output = lv_key.
        lv_len = strlen( lv_key ).
        IF lv_len GT 50.
          lv_key = lv_key(50).
        ENDIF.

        CALL FUNCTION 'Z_APPLICATION_LOG_CREATE'
          EXPORTING
            i_objectkey = lv_key
            i_object    = iv_object
            i_subobject = iv_subobject
            it_return   = lt_appl
            i_async     = space.

      ENDLOOP.

    ELSE.

      REFRESH lt_appl.

      CLEAR lw_appl.
      lw_appl-type = 'I'.
      CONCATENATE 'Z_PTC_FIORI_BILLING trace | INPUT | AREA' iv_area INTO lw_appl-message SEPARATED BY space.
      APPEND lw_appl TO lt_appl.

      CLEAR lw_appl.
      lw_appl-type = 'I'.
      CONCATENATE 'INPUT | REPORT_NO' iv_report_no INTO lw_appl-message SEPARATED BY space.
      APPEND lw_appl TO lt_appl.

      CLEAR lw_appl.
      lw_appl-type = 'I'.
      lw_appl-message = 'EX_OUTPUT: rows present but DELIVERY initial'.
      APPEND lw_appl TO lt_appl.

      CLEAR lv_row.
      LOOP AT it_output INTO lw_line.
        lv_row = lv_row + 1.
        PERFORM append_zst_fiori_bill_row USING lv_row lw_line CHANGING lt_appl.
      ENDLOOP.

      CLEAR lw_appl.
      lw_appl-type = 'I'.
      lw_appl-message = '--- EX_LOG ---'.
      APPEND lw_appl TO lt_appl.

      IF it_log IS INITIAL.
        CLEAR lw_appl.
        lw_appl-type = 'I'.
        lw_appl-message = 'EX_LOG: (empty)'.
        APPEND lw_appl TO lt_appl.
      ELSE.
        LOOP AT it_log INTO lw_log.
          CLEAR lw_appl.
          lw_appl-type = lw_log-type.
          IF lw_appl-type IS INITIAL.
            lw_appl-type = 'I'.
          ENDIF.
          lw_appl-message = lw_log-message.
          APPEND lw_appl TO lt_appl.
        ENDLOOP.
      ENDIF.

      PERFORM build_area_report_key USING iv_area iv_report_no CHANGING lv_key.

      CALL FUNCTION 'Z_APPLICATION_LOG_CREATE'
        EXPORTING
          i_objectkey = lv_key
          i_object    = iv_object
          i_subobject = iv_subobject
          it_return   = lt_appl
          i_async     = space.

    ENDIF.

  ELSE.

    REFRESH lt_appl.

    CLEAR lw_appl.
    lw_appl-type = 'I'.
    CONCATENATE 'Z_PTC_FIORI_BILLING trace | INPUT | AREA' iv_area INTO lw_appl-message SEPARATED BY space.
    APPEND lw_appl TO lt_appl.

    CLEAR lw_appl.
    lw_appl-type = 'I'.
    CONCATENATE 'INPUT | REPORT_NO' iv_report_no INTO lw_appl-message SEPARATED BY space.
    APPEND lw_appl TO lt_appl.

    CLEAR lw_appl.
    lw_appl-type = 'I'.
    lw_appl-message = 'EX_OUTPUT: (empty)'.
    APPEND lw_appl TO lt_appl.

    CLEAR lw_appl.
    lw_appl-type = 'I'.
    lw_appl-message = '--- EX_LOG ---'.
    APPEND lw_appl TO lt_appl.

    IF it_log IS INITIAL.
      CLEAR lw_appl.
      lw_appl-type = 'I'.
      lw_appl-message = 'EX_LOG: (empty)'.
      APPEND lw_appl TO lt_appl.
    ELSE.
      LOOP AT it_log INTO lw_log.
        CLEAR lw_appl.
        lw_appl-type = lw_log-type.
        IF lw_appl-type IS INITIAL.
          lw_appl-type = 'I'.
        ENDIF.
        lw_appl-message = lw_log-message.
        APPEND lw_appl TO lt_appl.
      ENDLOOP.
    ENDIF.

    PERFORM build_area_report_key USING iv_area iv_report_no CHANGING lv_key.

    CALL FUNCTION 'Z_APPLICATION_LOG_CREATE'
      EXPORTING
        i_objectkey = lv_key
        i_object    = iv_object
        i_subobject = iv_subobject
        it_return   = lt_appl
        i_async     = space.

  ENDIF.

ENDFORM.
```

### 6.2 `FORM BUILD_AREA_REPORT_KEY`

```abap
FORM build_area_report_key USING iv_area TYPE yarea
                                 iv_report_no TYPE yreport_no
                            CHANGING cv_key TYPE balnrext.

  DATA: lv_len TYPE i.

  CLEAR cv_key.
  CONCATENATE iv_area '|' iv_report_no INTO cv_key.
  lv_len = strlen( cv_key ).
  IF lv_len GT 50.
    cv_key = cv_key(50).
  ENDIF.

ENDFORM.
```

### 6.3 `FORM APPEND_ZST_FIORI_BILL_ROW`

*(Unchanged — one informational line per structure field; `ITEM_NO` appears only in **messages**, not in External ID.)*

```abap
FORM append_zst_fiori_bill_row USING pv_row TYPE i
                                    pw TYPE zst_fiori_billing
                               CHANGING pt_appl TYPE bapiret2_t.

  DATA: lw_appl TYPE bapiret2,
        lv_rowc(10) TYPE c.

  WRITE pv_row TO lv_rowc.
  CONDENSE lv_rowc.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'EX_OUTPUT row' lv_rowc '| AREA' pw-area INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'REPORT_NO' pw-report_no INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'ITEM_NO' pw-item_no INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'DELIVERY' pw-delivery INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'BILLNO' pw-billno INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'LR_NO' pw-lr_no INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'LR_DT' pw-lr_dt INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'GTABUKRS' pw-gtabukrs INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'CN_DUELISTSTAT' pw-cn_dueliststat INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'FKDAT' pw-fkdat INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

  CLEAR lw_appl.
  lw_appl-type = 'I'.
  CONCATENATE 'ERZET' pw-erzet INTO lw_appl-message SEPARATED BY space.
  APPEND lw_appl TO pt_appl.

ENDFORM.
```

---

## 7. Compliance with **ABAP Rules - 02-04-2026**

| Rule | Application |
|------|-------------|
| **7.31** (`01-compatibility.mdc`) | No inline `DATA(...)`; declarations at top of `FORM`. |
| **Naming** (`02-naming.mdc`) | `IV_`/`IT_`, `LW_`/`LT_`/`LV_`, `PW_`/`PT_`. |
| **FORM** (`00-main.mdc`) | Legacy-consistent with existing function group `FORM`s. |
| **Code Inspector** | Run before transport. |

---

## 8. Transport

| Object | Change |
|--------|--------|
| `Z_FIORI_SWM_OB_INV_GENRATE` | Constant, `PERFORM`, three `FORM`s |
| Customizing | SLG0 / ZAPPLOGACT |

---

## 9. Test checklist

| # | Scenario | Expected |
|---|----------|----------|
| 1 | Two output rows, **non-initial** deliveries, `EX_LOG` filled | Two SLG1 entries, External ID = **each delivery** |
| 2 | Output rows **all** **initial** `DELIVERY`, `EX_LOG` filled | **One** SLG1, External ID = **`area|report`** only; message **`EX_OUTPUT: rows present but DELIVERY initial`** + all rows + `EX_LOG` |
| 3 | Empty `EX_OUTPUT`, non-empty `EX_LOG` | One SLG1, **`area|report`**; `EX_OUTPUT: (empty)` |
| 4 | Both empty | One minimal SLG1, **`area|report`** |
| 5 | `ZAPPLOGACT` inactive | No persisted log; process unchanged |

---

## 10. Rollback

1. Remove `PERFORM` and `LC_SUBOBJECT_FIORI_BILL`.  
2. Delete `LOG_PTC_FIORI_BILLING_TRACE`, `BUILD_AREA_REPORT_KEY`, `APPEND_ZST_FIORI_BILL_ROW`.  
3. Optionally deactivate ZAPPLOGACT for `ZSCE_FIORI_BILL`.

---

*End of document.*
