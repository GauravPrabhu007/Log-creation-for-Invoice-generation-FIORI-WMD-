# Code Change Document — `Z_SCM_YTTS_UPDATE` (SLG1 Input Log + Lock Retry)

**Audience:** ABAP development, functional reviewer, QA, support  
**Target system:** SAP ECC 6.0 / NetWeaver 7.31 (`abap_731`)  
**Date:** 08-May-2026  
**Change type:** Reliability + observability enhancement for `YTTSTX0002` update path

---

## 1) Objective

Strengthen `Z_SCM_YTTS_UPDATE` to reduce random missed updates in `YTTSTX0002` by:

1. writing SLG1 input trace at FM start,
2. introducing explicit enqueue/dequeue around each table update,
3. retrying lock acquisition using configurable retry count from `ZLOG_EXEC_VAR`.

This change is intended to address concurrency-related misses and improve root-cause diagnostics.

---

## 2) Scope and Existing Behavior

Current FM behavior (as of `07.05.2026` export):

- Direct `UPDATE yttstx0002 ... WHERE area/report_no/item_no`.
- `COMMIT WORK` on success, `ROLLBACK WORK` on failure.
- No explicit lock object usage before update.
- No retry loop for lock conflicts.
- No start-of-FM SLG1 trace of inbound payload.

---

## 3) Proposed Functional Changes

### 3.1 Start-of-FM SLG1 Log

At start of `Z_SCM_YTTS_UPDATE`, create one SLG1 log entry with:

- object: `ZSCE_EWMLOG`
- subobject: `ZSCE_YTTS2UPD`
- external ID (header-level): `AREA|REPORT_NO` (respective inbound values)

The header log must capture:

- FM name (`Z_SCM_YTTS_UPDATE`)
- record count in `IT_FIORI_BILLING`
- key input fields summary for the request
- effective lock retry count used for this execution

In message text, include full item-level details from `IT_FIORI_BILLING` for each row (not only `BILLNO`).

Use existing application log framework `Z_APPLICATION_LOG_CREATE`.

### 3.2 Configurable Retry Parameter

Read retry count from table `ZLOG_EXEC_VAR`:

- `NAME = 'ZSCE_YTTS2UPD_RETRY'`
- `ACTIVE = 'X'`
- source field for numeric value: `NUMB`

Validation rules:

- if no active entry exists -> fallback to default `3`
- if value non-numeric or less than `1` -> fallback to default `3`
- if value greater than cap `10` -> use capped value `10`

Log configured value and effective value in SLG1.

### 3.3 Enqueue / Dequeue Around `YTTSTX0002` Update

Before each `UPDATE yttstx0002`, call enqueue FM for lock object built on table key:

- lock key fields: `MANDT`, `AREA`, `REPORT_NO`, `ITEM_NO`

After update attempt (success/failure), always call dequeue FM for same key.

> Note: Use generated enqueue/dequeue FM names from lock object in your system (example naming pattern: `ENQUEUE_E<lockobj>`, `DEQUEUE_E<lockobj>`).  
> If lock object does not exist, create one for `YTTSTX0002` with the key fields above before implementing this FM change.

### 3.4 Retry Logic for Lock Conflict

For each billing row:

1. Attempt enqueue.
2. If lock success (`sy-subrc = 0`), perform update and exit retry loop.
3. If `foreign_lock`, wait and retry until `lv_retry_max` reached.
4. If max retries exhausted, append business error to `ET_RETURN` and continue next row.
5. If non-lock enqueue error (`system_failure` or others), append technical error and continue next row.

Recommended wait:

- fixed wait: `WAIT UP TO 1 SECONDS` between retries.

### 3.5 Foreign Lock Diagnostic Log (Owner/User/Time)

When enqueue returns `foreign_lock`, capture lock owner diagnostics and write them to SLG1 under the same header key (`AREA|REPORT_NO`).

Diagnostic source:

- Read current lock entries using standard enqueue read API (for example `ENQUEUE_READ`) filtered by relevant lock object/key.

Mandatory lock detail fields to log:

- lock object name
- lock argument/key string
- lock owner user
- lock owner client
- lock owner date
- lock owner time
- lock mode
- lock server (if available in returned structure)

If lock-owner details cannot be read, write fallback message:

- `FOREIGN_LOCK detected but lock owner details not available from ENQUEUE_READ`.

---

## 4) Technical Design Details

## 4.1 New/Updated Constants

Add constants in FM declaration section:

- log object: `lc_log_object TYPE balobj_d VALUE 'ZSCE_EWMLOG'`
- log subobject: `lc_log_subobject TYPE balsubobj VALUE 'ZSCE_YTTS2UPD'`
- retry param name: `lc_retry_param TYPE rvari_vnam VALUE 'ZSCE_YTTS2UPD_RETRY'`
- default retry: `lc_retry_default TYPE i VALUE 3`
- max retry cap: `lc_retry_cap TYPE i VALUE 10`

## 4.2 New Local Variables

Example variable set:

- `lv_retry_max TYPE i`
- `lv_retry_curr TYPE i`
- `lv_retry_raw  TYPE zlog_exec_var-numb`
- `lv_lock_ok TYPE c LENGTH 1`
- `lt_lock_entries TYPE TABLE OF seqg3` (or system-specific enqueue read output type)
- `lw_lock_entry TYPE seqg3`
- work/message variables for logging return details

## 4.3 Read Retry Configuration

Add one `SELECT` from `ZLOG_EXEC_VAR` at FM start for `ZSCE_YTTS2UPD_RETRY`.
Validate and derive `lv_retry_max` before processing loop.

## 4.4 Update Loop Changes

Inside existing loop on `IT_FIORI_BILLING`:

- replace direct update-only section with:
  - retry loop
  - enqueue call
  - update + commit/rollback
  - dequeue call
  - attempt-level message population

Commit model remains row-wise as existing logic currently commits each successful row.

## 4.5 Return Messages (`ET_RETURN`)

Keep existing success/error semantics and extend with clear lock messages:

- lock retry attempt info (optional info type `I`)
- lock timeout after max retries (`E`)
- enqueue technical failure (`E`)
- update success (`S`) unchanged
- update failure (`E`) unchanged

---

## 5) ABAP Rules Compliance (`ABAP Rules - 02-04-2026`)

Implementation must follow these rules:

1. **7.31 compatibility** (`00-main.mdc`, `01-compatibility.mdc`)
   - no inline declarations (`DATA(...)`)
   - no constructor operators (`VALUE`, `NEW`, `CORRESPONDING`)
   - no string templates (`|...|`) in newly added code
   - declarations at top of routine

2. **Naming conventions** (`02-naming.mdc`)
   - local vars `lv_*`, tables `lt_*`, work areas `lw_*`, constants `lc_*`, local types `lty_*`

3. **Database/performance** (`03-database.mdc`)
   - no `SELECT *`
   - no select-in-loop for retry param read
   - immediate `sy-subrc` check after DB/FM operations

4. **SY-SUBRC discipline** (`13-sy-subrc.mdc`)
   - mandatory immediate check after:
     - `SELECT`
     - `CALL FUNCTION` (enqueue/dequeue/logging API)
     - `UPDATE`
     - `READ TABLE`

5. **Transaction safety** (`16-transactions-data.mdc`)
   - explicit locking for updates is mandatory and covered in this change
   - rollback on failed update path

This change is legacy FM maintenance; procedural style is acceptable within existing function module context.

---

## 6) Pseudocode Blueprint (Reference Only)

```abap
" At FM start
PERFORM/FORM or inline block:
  read retry param from zlog_exec_var (name = 'ZSCE_YTTS2UPD_RETRY', active = 'X')
  validate and derive lv_retry_max (default 3, cap 10)
  build header key lv_objectkey = area|report_no
  build lt_appl with:
    - input count and retry config
    - one section per IT_FIORI_BILLING row with full row details
  call Z_APPLICATION_LOG_CREATE with:
    i_object = 'ZSCE_EWMLOG'
    i_subobject = 'ZSCE_YTTS2UPD'
    i_objectkey = lv_objectkey
    it_return = lt_appl

LOOP AT it_fiori_billing INTO lw_fiori_billing.
  CLEAR: lv_retry_curr, lv_lock_ok.

  DO lv_retry_max TIMES.
    lv_retry_curr = sy-index.

    CALL FUNCTION '<ENQUEUE_FM>'
      EXPORTING mandt = sy-mandt
                area = lw_fiori_billing-area
                report_no = lw_fiori_billing-report_no
                item_no = lw_fiori_billing-item_no
      EXCEPTIONS foreign_lock = 1
                 system_failure = 2
                 OTHERS = 3.

    IF sy-subrc = 0.
      lv_lock_ok = 'X'.
      EXIT.
    ELSEIF sy-subrc = 1.
      call ENQUEUE_READ and log lock owner details in SLG1
      IF lv_retry_curr < lv_retry_max.
        WAIT UP TO 1 SECONDS.
      ENDIF.
    ELSE.
      " technical enqueue error -> append ET_RETURN and EXIT DO
      EXIT.
    ENDIF.
  ENDDO.

  IF lv_lock_ok <> 'X'.
    " lock timeout -> append ET_RETURN and CONTINUE
    CONTINUE.
  ENDIF.

  UPDATE yttstx0002 SET ...
    WHERE area = lw_fiori_billing-area
      AND report_no = lw_fiori_billing-report_no
      AND item_no = lw_fiori_billing-item_no.
  IF sy-subrc = 0.
    COMMIT WORK.
  ELSE.
    ROLLBACK WORK.
  ENDIF.

  CALL FUNCTION '<DEQUEUE_FM>'
    EXPORTING mandt = sy-mandt
              area = lw_fiori_billing-area
              report_no = lw_fiori_billing-report_no
              item_no = lw_fiori_billing-item_no.
  " check sy-subrc and log warning if needed
ENDLOOP.
```

---

## 7) Required Technical Dependencies

1. **Application log customizing**
   - SLG0 object/subobject must exist and be active:
     - object: `ZSCE_EWMLOG`
     - subobject: `ZSCE_YTTS2UPD`
2. **Lock object**
   - enqueue object for `YTTSTX0002` key (`MANDT, AREA, REPORT_NO, ITEM_NO`) must be available.
3. **Retry parameter customizing**
   - maintain active entry in `ZLOG_EXEC_VAR`:
     - `NAME = ZSCE_YTTS2UPD_RETRY`
     - `NUMB = <retry_count>` (recommended `3`)
     - `ACTIVE = X`

---

## 8) SLG1 Message Format Template (Mandatory)

Use a consistent SLG1 body format for every FM call.

### 8.1 Header

- `OBJECT` = `ZSCE_EWMLOG`
- `SUBOBJECT` = `ZSCE_YTTS2UPD`
- `EXTERNAL ID` = `<AREA>|<REPORT_NO>` (trim/cap to `BALNREXT` length as required)

### 8.2 Message Body Layout (`IT_RETURN` passed to `Z_APPLICATION_LOG_CREATE`)

Recommended ordered lines:

1. `Z_SCM_YTTS_UPDATE trace | INPUT HEADER`
2. `AREA <value>`
3. `REPORT_NO <value>`
4. `INPUT_ROW_COUNT <n>`
5. `RETRY_PARAM ZSCE_YTTS2UPD_RETRY | CONFIGURED <raw> | EFFECTIVE <n>`
6. `--- IT_FIORI_BILLING ---`
7. One block per row (all rows must be logged):
   - `ROW <n> | AREA <value>`
   - `REPORT_NO <value>`
   - `ITEM_NO <value>`
   - `BILLNO <value>`
   - `LR_NO <value>`
   - `LR_DT <value>`
   - `FKDAT <value>`
   - `ERZET <value>`
   - `GTABUKRS <value>`
   - `CN_DUELISTSTAT <value>`
8. `--- LOCK/UPDATE RESULT ---`
9. Per processed row result:
   - `ROW <n> | LOCK_ATTEMPT <m>/<retry_max> | STATUS <LOCK_OK|FOREIGN_LOCK|TIMEOUT|TECH_ERROR>`
   - for `FOREIGN_LOCK`, add owner details lines:
     - `LOCK_OWNER_USER <value>`
     - `LOCK_OWNER_CLIENT <value>`
     - `LOCK_OWNER_DATE <value>`
     - `LOCK_OWNER_TIME <value>`
     - `LOCK_OBJECT <value>`
     - `LOCK_ARGUMENT <value>`
   - `ROW <n> | UPDATE_STATUS <SUCCESS|FAILED> | SUBRC <sy-subrc>`

### 8.3 Message Type Conventions

- Use `I` for trace/input/attempt details.
- Use `S` for successful update status summary.
- Use `W` for recoverable retry events (optional).
- Use `E` for lock timeout, enqueue technical failure, or update failure.

### 8.4 Rules

- Do not put long narrative text in external ID; external ID must remain key-based (`AREA|REPORT_NO`).
- Do not log only `BILLNO`; all required `IT_FIORI_BILLING` fields above must be present.
- Keep one header log per FM call (avoid duplicate start logs in same execution).

---

## 9) Test Checklist

1. **Baseline success (no lock contention)**
   - input rows update successfully
   - start-of-FM SLG1 log is created with payload and retry config

2. **Concurrent lock contention**
   - first attempts fail with lock
   - subsequent retry succeeds within configured retry count
   - row is eventually updated
   - SLG1 contains lock owner details for each foreign-lock attempt

3. **Lock timeout**
   - lock held beyond max retries
   - ET_RETURN contains clear lock timeout error
   - processing continues for subsequent rows
   - SLG1 shows lock owner trail for all attempts before timeout

4. **Invalid retry parameter**
   - set `NUMB` blank/non-numeric/zero
   - FM falls back to default retry value and logs fallback

5. **Retry cap behavior**
   - set `NUMB` > 10
   - FM uses capped value 10

6. **Update failure (non-lock)**
   - force update miss (`WHERE` no row match or DB issue simulation)
   - ET_RETURN shows update error; lock release still executed

7. **Regression**
   - existing tracking API (`Z_SCM_TRACKING_STATUS_SAVE_API`) path remains unaffected
8. **ENQUEUE_READ fallback**
   - simulate case where lock owner read is unavailable
   - SLG1 contains fallback diagnostic message without short dump

---

## 10) Production-Conclusive Controls (Mandatory)

To make root-cause analysis conclusive in Production, implement all controls below.

### 10.1 Deterministic Row Final Status

Every processed row in `IT_FIORI_BILLING` must end with exactly one final status in SLG1:

- `FINAL_STATUS SUCCESS`
- `FINAL_STATUS LOCK_TIMEOUT`
- `FINAL_STATUS ENQUEUE_TECH_ERROR`
- `FINAL_STATUS UPDATE_FAILED`
- `FINAL_STATUS ROW_KEY_NOT_FOUND` (if applicable in your logic)

No row should exit processing without one final status line.

### 10.2 Mandatory Per-Row Diagnostic Fields

For each row, SLG1 must include:

- `AREA`
- `REPORT_NO`
- `ITEM_NO`
- `BILLNO`
- `LOCK_ATTEMPTS_USED <n>`
- `LAST_ENQUEUE_SUBRC <n>`
- `UPDATE_SUBRC <n>`
- `FINAL_STATUS <value>`
- `FM_TIMESTAMP_DATE <sy-datum>`
- `FM_TIMESTAMP_TIME <sy-uzeit>`

### 10.3 Correlation Key Rules

- Header `EXTERNAL ID`: `AREA|REPORT_NO`
- Message-text correlation key per row: `AREA|REPORT_NO|ITEM_NO`

This allows support to correlate:

1. inbound payload,
2. lock attempts/owner details,
3. update result,
4. final outcome for each item.

### 10.4 Lock Owner Evidence (Foreign Lock)

For `foreign_lock`, log owner details from enqueue read plus:

- `LOCK_ATTEMPT_NO <m>`
- `LOCK_RETRY_MAX <n>`
- `LOCK_WAIT_SECONDS <n>`

If owner fields unavailable in target system, log placeholder text:

- `LOCK_OWNER_<field> NOT_AVAILABLE`

### 10.5 ET_RETURN and SLG1 Alignment

For any non-success final status, append aligned `ET_RETURN` message with same row key (`AREA|REPORT_NO|ITEM_NO`) so API caller output and SLG1 can be matched one-to-one.

### 10.6 Production Support Runbook

Support must check in this order:

1. Find SLG1 by Object/Subobject + External ID (`AREA|REPORT_NO`).
2. Identify row block for impacted `ITEM_NO`.
3. Read `FINAL_STATUS`.
4. If `LOCK_*`: check owner info and validate in `SM12`.
5. If `UPDATE_FAILED`: inspect `UPDATE_SUBRC` and row key existence in `YTTSTX0002`.

### 10.7 Go-Live Validation Gate

Do not close transport testing until at least one tested case is captured for each status:

- `SUCCESS`
- `LOCK_TIMEOUT` (simulated contention)
- `UPDATE_FAILED` (negative path)

---

## 11) Risks and Mitigations

### Risk 1: Longer runtime due to retries

**Mitigation:** cap retry count (`10`), keep wait short (`1` second), monitor volume.

### Risk 2: Missing or wrong lock object

**Mitigation:** verify/generated lock object before transport release; include in transport if newly created.

### Risk 3: Over-logging volume in SLG1

**Mitigation:** one start log per FM call; keep per-attempt messages concise; avoid duplicate payload dumps.

### Risk 4: Enqueue-read structure differences across systems

**Mitigation:** use system-available output structure for `ENQUEUE_READ` and map only available fields; keep fallback message when a field is not available.

---

## 12) Transport and Rollback

### Transport Objects

- FM include/object containing `Z_SCM_YTTS_UPDATE`
- lock object (only if newly created)
- any related message class updates (if introduced)

### Rollback

1. Remove enqueue/dequeue and retry loop block.
2. Revert to prior direct update behavior.
3. Remove start-of-FM SLG1 block for this enhancement.
4. Keep `ZLOG_EXEC_VAR` param entry harmless (or deactivate).

---

## 13) Implementation Confirmation Checklist

Before coding, confirm:

1. external ID formatting (`AREA|REPORT_NO`) and max length handling,
2. exact enqueue/dequeue FM names from lock object,
3. retry wait strategy (fixed 1 sec vs incremental backoff),
4. final message class/number for lock timeout and technical errors.
5. final `ENQUEUE_READ` output structure/field mapping in the target ECC system.
6. final list of `FINAL_STATUS` constants used in SLG1 and ET_RETURN.

---

*End of document.*
