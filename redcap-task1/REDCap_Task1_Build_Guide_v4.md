# REDCap Task 1 — Build Guide v4

**Supersedes v3.** v3 told you to convert `bmi_above_35` and `age_18_or_older` into plain collected radios and left it there — which fixes the broken syntax but leaves a human retyping two numbers the database already knows. That is the exact failure mode visible in the Task 2 review file.

v4 adds **Section 0**: derive both criteria from Form 1, pre-fill them, and keep a loud warning on the override path. Sections 1–10 are unchanged from v3 except for the amendments listed in §0.9.

**Interactive version:** `build-console.html` in this directory — the full guide as a checklist with copy-to-clipboard logic strings, filters by instrument and priority, and a live simulator of the Form 1 → Form 3 derivation.

**Deadline:** Monday 17 August 2026, 23:00 CEST.

---

## 0. Integrate the calculated BMI and age into Screening

### 0.0 Why this is the right shape

The assignment states two things that have to hold at once:

> The form *"should **collect** Yes/No responses"* for the six inclusion and exclusion criteria.

> *"A participant with a BMI above 35 kg/m² should not be recorded as satisfying the corresponding eligibility requirement **without an appropriate warning or check**."*

The second clause only means anything if the field *can* disagree with the derived value. So the answer is not to lock the field to a calculation — it is to **derive the answer, pre-fill it, and make any disagreement loud and auditable**.

| Option | Verdict | Why |
|---|---|---|
| **A** — keep it a pure `calc` field | ❌ Rejected | Fails "should collect Yes/No responses". No warning left to demonstrate. An erroneously enrolled participant becomes unrecordable. |
| **B** — derive → pre-fill → warn | ✅ **Built** | Collected, editable, auditable radio seeded from a derived flag. Satisfies both clauses. |
| **C** — pre-fill then `@READONLY` | ❌ Rejected | Removes the last-resort override. A site facing a genuine measurement error has no path except a database change request. |

### 0.1 `bmi_gt35_calc` — new calculated field on **Demographics**

`Designer → Demographics → add field immediately after bmi`

```
if([bmi] = "", "", if([bmi] > 35, 1, 0))
```

- Label: `BMI > 35 kg/m² (derived for screening)`
- Field Annotation: `@HIDDEN-PDF`
- Leave it visible on screen — it costs one line and makes the derivation auditable to the reviewers without opening the data dictionary.

**Placement is the trick.** A calc field only stores its value when *its own* instrument is saved. Putting this on Demographics means the flag is already in the database before Screening is ever opened, which is what allows the `@DEFAULT` in §0.3 to fire on first load. On Screening it would be computed too late to seed anything.

> ⚠️ **The blank guard is not optional.** Without `if([bmi] = "", "", …)` an empty BMI evaluates as `0`, and the flag would confidently pre-fill *"No, BMI is not above 35"* for a participant who has never been measured. That is a worse failure than the one being fixed.

### 0.2 `age_ge18_calc` — new calculated field on **Demographics**

`Designer → Demographics → add field immediately after age`

```
if([age] = "", "", if([age] >= 18, 1, 0))
```

- Label: `Age ≥ 18 at enrolment (derived for screening)`
- Field Annotation: `@HIDDEN-PDF`

Depends on **§1.1** being done first — `age` must calculate off `[d1_date]`, not `'today'`, or this flag drifts on every re-save.

### 0.3 Seed the two criteria with `@DEFAULT`

`Designer → Screening and Eligibility → age_18_or_older / bmi_above_35 → Field Annotation`

Both become **radio**, choices `1, Yes | 0, No`, Required = y, each carrying:

| Field | Annotation |
|---|---|
| `age_18_or_older` | `@DEFAULT="[age_ge18_calc]"` |
| `bmi_above_35` | `@DEFAULT="[bmi_gt35_calc]"` |

`@DEFAULT` writes only into an **empty** field, so it can never silently overwrite an entered value.

Both instruments are in the same event (`day_1_arm_1`), so the plain field name resolves. If Screening ever moves to another event this becomes `@DEFAULT="[day_1_arm_1][bmi_gt35_calc]"`.

> ⚠️ **The one ordering caveat.** `@DEFAULT` is evaluated when the form is *loaded*. Open Screening before Demographics has been saved and there is nothing to copy, so the fields stay blank. That is correct behaviour — blank beats guessed — and §0.5 puts a visible block on the form for exactly that case. Test it as record P003 (§7.3).

### 0.4 Pipe the source numbers into the labels

Pre-filling is half of it; the person confirming should be able to *see* the number without navigating back to Form 1.

**`age_18_or_older` label:**
```
Is the participant at least 18 years old?
<span style="font-weight:normal;color:#0A5F73">Age calculated from date of birth: <b>[age]</b> years</span>
```

**`bmi_above_35` label:**
```
Is the BMI above 35 kg/m²?
<span style="font-weight:normal;color:#0A5F73">BMI calculated from Day 1 measurements: <b>[bmi]</b> kg/m² ([height] cm, [weight] kg)</span>
```

**Field note on both:**
```
Pre-filled from the value calculated on the Demographics form. Change it only if the Day 1 measurements are themselves wrong — in that case correct Demographics first, then return here.
```

### 0.5 `w_demog_missing` — the gate at the top of Screening

`Designer → Screening and Eligibility → new Descriptive Text field, first field on the form`

This is the field that actually prevents the wrong click. If Demographics is not complete, neither criterion can be pre-filled or checked — so say so, before anything else is answered.

Branching logic:
```
[bmi] = "" or [age] = ""
```

Label:
```html
<div style="color:#8A5800;font-weight:bold;padding:8px;border:1px solid #8A5800;border-radius:4px;">
⚠ Demographics is not complete for this participant. The age and BMI criteria below cannot be pre-filled or checked against source until height, weight and date of birth are entered on Form 1. Complete Demographics first.
</div>
```

Annotation: `@HIDDEN-PDF`.

### 0.6 Keep `w_age_check` and `w_bmi_check` — they are the "check"

Pre-filling handles the honest mistake. These handle the override, the late correction to Demographics, and anything arriving by data import — none of which `@DEFAULT` touches. **Do not drop them because the field is now pre-filled.** Logic unchanged from v3 §4.1 and §4.2.

> **The behaviour worth defending.** If someone fixes a weight typo on Demographics a week later, the stored `bmi_above_35` does *not* silently change — it is a collected value, a record of what was assessed at the time. What happens instead is that `w_bmi_check` starts firing and Data Quality rule 2 flags the record. The disagreement becomes visible and gets resolved through a query rather than overwritten with no trace.

### 0.7 Show the expected decision beside `eligible` — never fill it

Extends the same idea to the overall decision without crossing the line: the form computes what the six criteria imply and *says* it; the investigator still records the actual decision.

**`elig_calc`** — calculated field, `@HIDDEN-PDF`:
```
if([generally_healthy] = '1' and [age_18_or_older] = '1' and [icf_signed] = '1'
   and [previous_ebola_trial] = '0' and [previous_mpox_vaccination] = '0'
   and [bmi_above_35] = '0', 1, 0)
```

**`elig_hint_yes`** — descriptive, branching `[elig_calc] = '1'`:
```html
<div style="color:#2A6B4A;padding:8px;border:1px solid #2A6B4A;border-radius:4px;">
All six criteria as recorded above are satisfied. The expected enrolment decision is <b>Yes</b>.
</div>
```

**`elig_hint_no`** — descriptive, branching `[elig_calc] = '0'`:
```html
<div style="color:#8A5800;padding:8px;border:1px solid #8A5800;border-radius:4px;">
At least one criterion above is not satisfied. The expected enrolment decision is <b>No</b>. If the participant is being enrolled regardless, complete the Protocol Deviations form.
</div>
```

Keep `w_eligible` (v3 §4.3) below these as the hard discrepancy warning.

### 0.8 Build-note text

Add to §8.2 section 2, "Derived versus collected":

> The age and BMI eligibility criteria are pre-filled from the values calculated on the Demographics form (`@DEFAULT` seeded from hidden calculated flags), rather than being typed independently. They remain collected, editable fields: the specification requires Yes/No responses to be collected, and an override must stay possible so that a participant enrolled against criteria is recordable and the corresponding protocol deviation is triggered.
>
> Overriding a pre-filled value, or later correcting the Demographics measurements, raises an inline discrepancy warning and a Data Quality rule rather than silently rewriting the stored answer.
>
> The overall eligibility decision is deliberately not pre-filled. The form displays what the six recorded criteria imply, but the decision itself is the investigator's regulatory act and is recorded as entered.

### 0.9 Amendments to the v3 sections

| v3 section | Amendment |
|---|---|
| §1.3 `bmi_above_35` | After converting to radio, apply §0.3 and §0.4 so it arrives pre-filled rather than empty |
| §3.1 Demographics | The `height`/`weight` decimal-convention fix (`number`, not `number_comma_decimal`) is now **critical**, not a nicety — everything in §0 is downstream of `[bmi]` being right |
| §3.3 Screening | Field order gains `w_demog_missing` first, and `elig_calc` / `elig_hint_yes` / `elig_hint_no` after `eligible` |
| §3.5 Vaccination | The "no `@TODAY`/`@NOW`" rule stands and does not contradict §0: pre-filling a **derived** value the database already holds is verification; pre-filling an **observed** value it cannot know is fabrication. Worth stating explicitly if challenged |
| §5.2 DQ rules | Rules 1 and 2 are now the backstop for §0 — they catch the case where Demographics is corrected after Screening was signed off |
| §6 Paper CRF | Add `@HIDDEN-PDF` to `bmi_gt35_calc`, `age_ge18_calc`, `elig_calc`. The piped labels from §0.4 correctly print as `Age calculated from date of birth: ___ years` for hand transcription |
| §7 Test records | Record P001 must confirm the pre-fill fires; P002 must confirm the override still saves; new record **P003** tests the blank-Demographics path (§7.3 below) |
| §10 Action tags | `@DEFAULT` added — used on `age_18_or_older` and `bmi_above_35` only |

### 0.10 New test record — P003, the out-of-order case

Two minutes, and it demonstrates the one caveat in the design. Create a record and open **Screening first**, before touching Demographics.

- `w_demog_missing` must be showing at the top of the form
- Both criteria must be **blank**, not pre-filled with a guess
- Now complete Demographics, return to Screening — the pre-fill takes effect on this fresh load

If a reviewer asks *"what happens if the forms are completed out of order?"*, you have already tested it and the answer is on screen.

### 0.11 New defence, for §9

**"You pre-filled a field the specification told you to collect. Isn't that the same as calculating it?"**

> No — and the distinction is why the requirement has two clauses. The value is stored as a collected response, it is editable, it appears in the audit trail as a user entry, and it can disagree with the derived value. What the pre-fill removes is the transcription step, which is where the error in the Task 2 data actually came from. What it keeps is the human confirmation and the ability to override, which is what makes an erroneous enrolment recordable and the protocol deviation traceable.

---

## 1–10. Unchanged from v3

See `REDCap_Task1_Build_Guide_v3.md` for the full text of sections 1 through 10, or work from `build-console.html`, which carries all of it as a checklist with the §0.9 amendments already applied.

Summary of what those sections cover:

| Section | What it is | Time |
|---|---|---|
| §1 | Six breaking fixes — `age`, `d1_date`, `bmi_above_35`, `eligible`, `specify_other`, `[event-label]` | 30 min |
| §2 | Structural gaps — repeating instruments, event designation, Delivery access | 35 min |
| §3 | Form-by-form remediation across all eight instruments | 90 min |
| §4 | Cross-field warning library — eleven descriptive-field warnings | 45 min |
| §5 | Data Resolution Workflow + twelve custom Data Quality rules | 30 min |
| §6 | Paper CRF generation and checks | 25 min |
| §7 | Test records | 30 min |
| §8 | Submission pack and build note | 25 min |
| §9 | Defending the design | — |
| §10 | Quick reference — events, logic syntax, action tags | — |

---

*Version v4 — 16 August 2026. Supersedes REDCap_Task1_Build_Guide_v3.md.*
