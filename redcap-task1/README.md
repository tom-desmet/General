# REDCap Task 1 — Build Console

Working materials for the Data Manager assessment, Task 1 (REDCap database development).

## Files

| File | What it is |
|---|---|
| `build-console.html` | **The interactive guide.** Open in any browser — no build step, no dependencies, no network access. |
| `REDCap_Task1_Build_Guide_v4.md` | Section 0 in full (the BMI/age integration) plus the amendments it makes to v3 |
| `REDCap_Task1_Build_Guide_v3.md` | The previous guide, carried forward for reference |

## What changed in v4

v3 converted `bmi_above_35` and `age_18_or_older` into plain collected radios. That fixes the broken calculation syntax but leaves study staff retyping two numbers the database already holds — which is the failure visible in the Task 2 review file.

v4 wires Form 1 into Form 3:

- Two hidden calculated flags on **Demographics** (`bmi_gt35_calc`, `age_ge18_calc`) turn the computed BMI and age into criterion answers
- `@DEFAULT` on the two Screening radios seeds them from those flags, so the common case takes zero decisions
- The source numbers are piped into the field labels, so the answer is verifiable without leaving the form
- `w_demog_missing` blocks the top of Screening when Demographics is incomplete, so nothing is guessed
- The existing discrepancy warnings and Data Quality rules stay as the backstop for overrides, imports, and later edits

The fields stay **collected and editable** on purpose. The assignment requires Yes/No responses to be collected, and an override has to remain possible — otherwise a participant enrolled against criteria becomes unrecordable and the protocol deviation is never triggered.

The overall `eligible` decision is deliberately **not** pre-filled. The form shows what the six criteria imply; the investigator records the actual decision.

## The console

- All 47 build steps as checkboxes, with progress tracked per section (persists in the browser)
- Copy button on every logic string, choice list, and field note — 42 of them
- Filter by instrument, by priority, or by what's still outstanding; free-text search across every step
- A **live simulator** of the Form 1 → Form 3 derivation: change height, weight or date of birth and watch the criteria pre-fill, then override one and watch the real warning text fire

Four presets in the simulator: `P003 — enrolled in error`, `Clean adult`, `Demographics not done`, and `BMI 35.0 exactly` (the boundary — the criterion is *above* 35, so 35.0 passes).
