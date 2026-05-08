# Code Change Document (Clean) — FM1 Internal `YTTSTX0002` Update

## Purpose

In some executions, billing is created successfully but `YTTSTX0002` is not updated because `Z_SCM_YTTS_UPDATE` is triggered from the UI/caller path instead of being owned by FM1.

This change makes `ZSCM_POST_CHK_BILLING` the single owner of `YTTSTX0002` update for FM1 flow, so update execution is not skipped due to caller-side branching.

## Code Changes To Be Done

### 1) Function Module: `ZSCM_POST_CHK_BILLING`

- Add local table for FM3 return handling:
  - `lt_ytts_return TYPE bapiret2_t`
- Keep existing FM2 (`Z_PTC_FIORI_BILLING`) logic unchanged.
- In block where FM2 output is available (`IF lt_bill[] IS NOT INITIAL.`):
  - Keep existing `lt_bill -> et_output` mapping.
  - Call `Z_SCM_YTTS_UPDATE` internally:
    - `it_fiori_billing = lt_bill`
    - `it_log           = lt_message`
    - `et_return        = lt_ytts_return`
  - Append FM3 return messages to FM1 return:
    - `APPEND LINES OF lt_ytts_return TO et_return`
- Add call-level error handling after FM3 call:
  - If `sy-subrc <> 0`, append one technical error message to `et_return`.

### 2) Caller/UI Adjustment for FM1 Flow

- Remove (or guard) separate UI/caller invocation of `Z_SCM_YTTS_UPDATE` when FM1 (`ZSCM_POST_CHK_BILLING`) is used.
- FM1 flow should not execute FM3 in two places.

## Final Expected Behavior

- When FM1 produces billing output, FM1 itself triggers `Z_SCM_YTTS_UPDATE`.
- `YTTSTX0002` update no longer depends on UI-side branch conditions.
- `et_return` from FM1 includes both billing-related and YTTS-update-related messages.

---

*End of clean code change document.*
