# Code Change Document — SLG1 trace for `ZSCM_POST_CHK_BILLING` (`Z_PTC_FIORI_BILLING`)

**Document version:** 1.0 — **standalone** (complete specification; no dependency on other code-change files).

**Audience:** ABAP developer / transport owner  
**Target system:** SAP ECC 6.0 / NetWeaver **7.31** (`abap_731`)  
**Function module:** `ZSCM_POST_CHK_BILLING` (function group per SE80)

**ABAP standards:** Project **`ABAP Rules - 02-04-2026`** — compliance summarized in **§8**.

**Change type:** Observability only — **no** change to `ET_RETURN` / `ET_OUTPUT` population logic, RPU posting check (`Z_SCM_RPU_POSTING_CHK`), GI validation, or control flow except **one** new `PERFORM` immediately after `Z_PTC_FIORI_BILLING`.

---

## 1. Executive summary

| Item | Detail |
|------|--------|
| **Purpose** | When `ZSCM_POST_CHK_BILLING` calls **`Z_PTC_FIORI_BILLING`**, write **SLG1** (BAL) traces via **`Z_APPLICATION_LOG_CREATE`**: full **`EX_OUTPUT`** as returned into **`LT_BILL`** (`ZTT_FIORI_BILLING` / `ZST_FIORI_BILLING` lines) and full **`EX_LOG`** as returned into **`LT_MESSAGE`** (`ZTT_LOG` / `ZST_LOG` lines). |
| **External ID (`I_OBJECTKEY`)** | **Delivery** when Case **A** applies; otherwise **`area\|report_no`** (built by **`FORM BUILD_AREA_REPORT_KEY`**, max 50 characters). Never use free-text trace sentences as **`I_OBJECTKEY`**. |
| **Log object / subobject** | **`ZSCE_EWMLOG`** / **`ZSCE_FIORI_BILL`** |
| **Wrapper FM** | **`Z_APPLICATION_LOG_CREATE`** with **`I_ASYNC = space`** (synchronous save). Persistence requires active row in **`ZAPPLOGACT`** for **`ZSCE_EWMLOG`** / **`ZSCE_FIORI_BILL`**. |
| **Inputs mapped from this FM** | Area = **`LV_AREA`** (from `YTTSTX0001` for **`IM_REPORT_NO`**; may be initial if not found). Report = **`IM_REPORT_NO`**. Output table = **`LT_BILL`**. Log table = **`LT_MESSAGE`**. |

---

## 2. Functional scope — when logging runs

`Z_PTC_FIORI_BILLING` is invoked only inside:

```abap
IF lt_post[] IS INITIAL.
  CALL FUNCTION 'Z_PTC_FIORI_BILLING'
    EXPORTING
      im_area      = lv_area
      im_report_no = im_report_no
    IMPORTING
      ex_output    = lt_bill
      ex_log       = lt_message.
ENDIF.
```

| Situation | `Z_PTC_FIORI_BILLING` | SLG1 trace (this change) |
|-----------|------------------------|---------------------------|
| **`LT_POST[]` not initial** (after `DELETE` where `rpu_posting_flag` is initial) | **Not** called | **Not** created by this change |
| **`LT_POST[]` initial** | Called | **`PERFORM log_ptc_fiori_billing_trace`** runs **immediately** after the `CALL FUNCTION` returns |

**Note:** If **`LV_AREA`** is initial (e.g. **`IM_REPORT_NO`** initial or `YTTSTX0001` read failed), PTC may still run; Cases **B/C/D** still build **`I_OBJECTKEY`** from **`BUILD_AREA_REPORT_KEY`** using whatever **`IV_AREA`** / **`IV_REPORT_NO`** values are passed (concatenate `area|report_no`, truncate to 50).

---

## 3. Insertion points in `ZSCM_POST_CHK_BILLING`

### 3.1 Constants

**Where:** In the main source of **`ZSCM_POST_CHK_BILLING`**, immediately after the existing GI-validation constants block (the block that ends with **`LC_RFC_PARAM_CHK`**).

```abap
  CONSTANTS: lc_gi_param_chk TYPE rvari_vnam VALUE 'ZSCM_GI_STATUS_CHECK',
             lc_rfc_param_chk TYPE zparam1 VALUE 'ZEWM_DEST',
             lc_bal_object     TYPE balobj_d  VALUE 'ZSCE_EWMLOG',
             lc_bal_subobj_fiori TYPE balsubobj VALUE 'ZSCE_FIORI_BILL'.
```

Keep existing **`CONSTANTS`** entries unchanged; **append** the two BAL lines (or merge into one **`CONSTANTS`** statement per your style guide).

*(If local naming prefers `lc_object` / `lc_subobject_fiori_bill`, use those names but keep values **`ZSCE_EWMLOG`** / **`ZSCE_FIORI_BILL`**.)*

### 3.2 Call site (order is mandatory)

**Insert** the **`PERFORM`**:

- **After** the closing of `CALL FUNCTION 'Z_PTC_FIORI_BILLING'` (after **`ex_log = lt_message`**).
- **Before** `IF lt_message[] IS NOT INITIAL.`

```abap
* SLG1 ZSCE_EWMLOG / ZSCE_FIORI_BILL — PTC billing trace (EX_OUTPUT + EX_LOG)
      PERFORM log_ptc_fiori_billing_trace USING lv_area im_report_no lt_bill lt_message
                                                lc_bal_object lc_bal_subobj_fiori.
```

Match **`lc_bal_object`** / **`lc_bal_subobj_fiori`** to the identifiers declared in **§3.1**.

### 3.3 Mandatory rules and anti-patterns

| Required | Detail |
|----------|--------|
| **Single logging entry point** | All BAL lines for this PTC call are built inside **`FORM LOG_PTC_FIORI_BILLING_TRACE`** into **`LT_APPL`** (`BAPIRET2_T`), then passed as **`IT_RETURN`** to **`Z_APPLICATION_LOG_CREATE`**. |
| **Order** | **`PERFORM`** must run **before** `IF lt_message[]` so that when **both** `LT_MESSAGE` and `LT_BILL` are filled, the log still contains **full `EX_OUTPUT`** expansion (**`APPEND_ZST_FIORI_BILL_ROW`**) and **full `EX_LOG`** — not log-table rows only. |

| Do **not** | Reason |
|-------------|--------|
| Assign **`I_OBJECTKEY`** from `CONCATENATE` of text such as `Z_PTC_FIORI_BILLING trace…` + area + report | That is **message** text, not a BAL external number; produces unreadable External ID and can exceed or misuse **`BALNREXT`**. |
| Call **`Z_APPLICATION_LOG_CREATE`** only inside `IF lt_message[]` / `IF lt_bill[]` for PTC | Wrong branch priority: when `LT_MESSAGE` has rows (even with blank `MESSAGE`), the **`ELSEIF`/output** path may never run; SLG1 can show **header only** and hide **`LT_BILL`**. |
| Log only **`BILLNO`** (or one field) from output | Insufficient for support; full structure lines are required via **`APPEND_ZST_FIORI_BILL_ROW`**. |
| Mix **`LT_MESSAGE`** into a table reused for both **`ET_RETURN`** and BAL in a way that skips building **`LT_APPL`** | Keep **`ET_RETURN`** / **`ET_OUTPUT`** logic unchanged; SLG1 content is **only** from **`LT_APPL`** inside the `FORM`. |

---

## 4. External ID and behaviour matrix (Cases A–D)

### 4.1 External ID rules (`I_OBJECTKEY`)

| Case | When | `I_OBJECTKEY` value |
|------|------|---------------------|
| **A** | **`LT_BILL`** is **not** initial **and** at least one line has **non-initial** `DELIVERY` | For **each** `LOOP` iteration with non-initial `DELIVERY`: that delivery, after **`CONVERSION_EXIT_ALPHA_OUTPUT`**, truncated to **50** characters. **One SLG1 document per such line** (same delivery appearing twice → two saves with the same external id is acceptable unless you add de-duplication later). |
| **D** | **`LT_BILL`** is **not** initial **and every** line has **initial** `DELIVERY` | **`FORM BUILD_AREA_REPORT_KEY`**: `CONCATENATE iv_area '|' iv_report_no` into key, **`STRLEN` > 50** → **`cv_key(50)`**. **Single** SLG1. |
| **B** | **`LT_BILL`** is **initial**, **`LT_MESSAGE`** is **not** initial | Same as **D**: **`area|report_no`** only. |
| **C** | **Both** **`LT_BILL`** and **`LT_MESSAGE`** are **initial** | Same as **D/B** key; minimal message body (see **§7.1** `ELSE` branch). |

**Item numbers** and other fields from **`ZST_FIORI_BILLING`** appear **only** in **`BAPIRET2-MESSAGE`** lines from **`APPEND_ZST_FIORI_BILL_ROW`**, **not** concatenated into **`I_OBJECTKEY`**.

### 4.2 Message body summary (by case)

| Case | Message body (high level) |
|------|---------------------------|
| **A** | Two INPUT lines (area, report_no) + **`PERFORM append_zst_fiori_bill_row`** for **that** output line + separator `--- EX_LOG ---` + every `ZST_LOG` line from **`LT_MESSAGE`** (type + message; default type `'I'` if blank) or line `EX_LOG: (empty)` if log table is initial. |
| **D** | INPUT lines + informational line `EX_OUTPUT: rows present but DELIVERY initial` + **all** `LT_BILL` rows via **`APPEND_ZST_FIORI_BILL_ROW`** with row counter + `--- EX_LOG ---` + full **`LT_MESSAGE`** or empty placeholder. |
| **B** | INPUT lines + `EX_OUTPUT: (empty)` + `--- EX_LOG ---` + all **`LT_MESSAGE`** lines (or handled in shared `ELSE` branch of `FORM`). |
| **C** | INPUT lines + `EX_OUTPUT: (empty)` + `--- EX_LOG ---` + `EX_LOG: (empty)`. |

### 4.3 Control flow inside `FORM LOG_PTC_FIORI_BILLING_TRACE`

1. If **`IT_OUTPUT`** (`LT_BILL`) **is not initial**: scan once for any non-initial **`DELIVERY`** → set flag **`LV_HAS_DELIVERY = 'X'`** (`DATA LV_HAS_DELIVERY TYPE C LENGTH 1`).
2. If **`LV_HAS_DELIVERY = 'X'`** → **Case A**: `LOOP` at `IT_OUTPUT`; **`CONTINUE`** when `DELIVERY` initial; for each non-initial delivery build **`LT_APPL`** and call **`Z_APPLICATION_LOG_CREATE`** with delivery key.
3. Else (output not initial but no usable delivery) → **Case D**: single **`LT_APPL`**, then **`BUILD_AREA_REPORT_KEY`**, one **`Z_APPLICATION_LOG_CREATE`**.
4. Else (**`IT_OUTPUT`** initial) → **Case B/C**: build **`LT_APpl`** with empty-output markers; append all **`IT_LOG`** or `EX_LOG: (empty)`; **`BUILD_AREA_REPORT_KEY`**; one **`Z_APPLICATION_LOG_CREATE`**.

---

## 5. Code reuse strategy

`ZSCM_POST_CHK_BILLING` lives in its own function group. The three **`FORM`** routines in **§7** must exist in a compilation unit visible to this FM. Choose **one** approach:

| Option | Description |
|--------|-------------|
| **Recommended** | **Shared include** (project-specific name, e.g. `LZ…F00`) containing **`LOG_PTC_FIORI_BILLING_TRACE`**, **`BUILD_AREA_REPORT_KEY`**, **`APPEND_ZST_FIORI_BILL_ROW`**, included in the function group main program(s) that need PTC tracing. **One** physical copy of the source. |
| **Alternative** | New **function module** (e.g. `Z_LOG_PTC_FIORI_BILLING_TRACE`) that receives area, report_no, tables, object, subobject and performs the same logic internally; **`ZSCM_POST_CHK_BILLING`** calls it with **`CALL FUNCTION`**. |
| **Not recommended** | Identical **`FORM`** source pasted into multiple function groups (drift risk). |

The **authoritative** ABAP for the routines is **§7** of **this** document.

---

## 6. SAP customizing (`SLG0` / `ZAPPLOGACT`)

| # | T-code | Action |
|---|--------|--------|
| 1 | **SLG0** | Define subobject **`ZSCE_FIORI_BILL`** under application log object **`ZSCE_EWMLOG`** (if not already present). |
| 2 | **ZAPPLOGACT** (or your activation table for `Z_APPLICATION_LOG_CREATE`) | Row for **`ZSCE_EWMLOG`** / **`ZSCE_FIORI_BILL`** with **ACTIVE** = **`X`** so logs are persisted. |

---

## 7. Complete ABAP source (`FORM` routines)

Place routines in the **shared include** chosen in **§5**, or in the function group include after **`ENDFUNCTION.`** of **`ZSCM_POST_CHK_BILLING`** if this group is the sole owner.

**Parameter mapping from `ZSCM_POST_CHK_BILLING`:** `IV_AREA` ← `LV_AREA`, `IV_REPORT_NO` ← `IM_REPORT_NO`, `IT_OUTPUT` ← `LT_BILL`, `IT_LOG` ← `LT_MESSAGE`.

### 7.1 `FORM LOG_PTC_FIORI_BILLING_TRACE`

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

### 7.2 `FORM BUILD_AREA_REPORT_KEY`

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

### 7.3 `FORM APPEND_ZST_FIORI_BILL_ROW`

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

**DDIC:** If **`ZST_FIORI_BILLING`** differs in your system, adjust **only** **`APPEND_ZST_FIORI_BILL_ROW`**.

---

## 8. Compliance with **ABAP Rules - 02-04-2026**

| Rule file | Application |
|-----------|-------------|
| **`01-compatibility.mdc`** (7.31) | No inline `DATA(...)`, no string templates, no `VALUE` / `NEW` / table expressions; declare all **`DATA`** at the start of each **`FORM`**. |
| **`02-naming.mdc`** | Constants `lc_*`; `FORM` parameters `iv_*` / `it_*`; locals `lw_*` / `lt_*` / `lv_*` as in **§7**. |
| **`00-main.mdc`** | Project prefers OO for **new** logic; this change extends a **legacy function module** with **`FORM`** routines — acceptable as **function group / procedural maintenance** if kept in a shared include or small wrapper FM per **§5**. |
| **Code Inspector** | Run and clear findings before transport. |

---

## 9. Transport objects

| Object | Change |
|--------|--------|
| **`ZSCM_POST_CHK_BILLING`** | BAL constants + one **`PERFORM`** after **`Z_PTC_FIORI_BILLING`** |
| **Shared include** or **wrapper FM** | Per **§5** — contains or implements **§7** |
| **Customizing** | None if **SLG0** / **ZAPPLOGACT** already correct for **`ZSCE_FIORI_BILL`** |

---

## 10. Test checklist

| # | Scenario | Expected |
|---|----------|----------|
| 1 | **`LT_POST`** not initial → PTC not called | No new SLG1 from this change |
| 2 | **`LT_POST`** initial, **`LT_BILL`** has ≥1 non-initial **`DELIVERY`** | One or more SLG1 headers; External ID = **delivery** (alpha, ≤50); message list includes **`APPEND_ZST_…`** fields and **`LT_MESSAGE`** lines |
| 3 | **`LT_POST`** initial, **`LT_BILL`** has rows but **all** **`DELIVERY`** initial | One SLG1; External ID = **`area|report`**; line `EX_OUTPUT: rows present but DELIVERY initial` + all rows + **`LT_MESSAGE`** |
| 4 | **`LT_POST`** initial, **`LT_BILL`** empty, **`LT_MESSAGE`** not empty | One SLG1; **`area|report`**; `EX_OUTPUT: (empty)` + all log lines |
| 5 | **`LT_POST`** initial, both tables empty | One minimal SLG1; **`area|report`**; empty markers per **§7.1** `ELSE` |
| 6 | **Both** `LT_MESSAGE` and `LT_BILL` filled, with delivery on bill lines | SLG1 shows **full bill expansion** and **full log** — verify **`PERFORM`** is **before** `IF lt_message` |
| 7 | **`ZAPPLOGACT`** (or equivalent) inactive | No persisted BAL; **`ET_RETURN`** / **`ET_OUTPUT`** unchanged |
| 8 | **Regression** | External ID must **not** be a long concatenation of the literal `Z_PTC_FIORI_BILLING trace` used **as** `I_OBJECTKEY` |

---

## 11. Rollback

1. Remove the **`PERFORM log_ptc_fiori_billing_trace`** call and the BAL **`CONSTANTS`** from **`ZSCM_POST_CHK_BILLING`** if they are unused elsewhere.  
2. If the **`FORM`** routines live in a **shared include** still required by **other** function modules, remove only the **`INCLUDE`** from **this** function group’s main program when rolling back **this** FM only — do **not** delete the shared source unless **no** other consumer remains.  
3. Do **not** reintroduce inline **`Z_APPLICATION_LOG_CREATE`** patterns listed in **§3.3** as a substitute.

---

*End of standalone document (v1.0).*
