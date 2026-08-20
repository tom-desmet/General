# REDCap Task 1 — Build Guide v3

**Supersedes v2.** Adds the instrument and navigation path to every item in §1, §2 and §4, plus a Form column on the Data Quality rules.

**Supersedes v1.** v1 was written blind, as a greenfield spec. This version is written against **your actual project** — six events on one arm, eight instruments, 67 fields, REDCap v16.0.15, status Development.

Everything below uses **your variable names**. Nothing here asks you to rebuild.

**Deadline:** Monday 17 August 2026, 23:00 CEST.

---

## 0. How to use this

Work top to bottom. The sections are ordered by consequence, not by form.

| Section | What it is | Time | Skippable? |
|---|---|---|---|
| §1 | Six things that are broken | 30 min | ❌ No |
| §2 | Structural gaps — repeating, designation, Delivery access | 35 min | ❌ No |
| §3 | Form-by-form remediation | 90 min | Partially |
| §4 | Cross-field warning library | 45 min | ⚠️ Yes if short |
| §5 | Query management + Data Quality rules | 30 min | ⚠️ Partially |
| §6 | Paper CRF | 25 min | ❌ No — it's a deliverable |
| §7 | Test records | 30 min | ❌ No |
| §8 | Submission pack + build note | 25 min | ❌ No |
| §9 | Defending the design | — | Read it |

**If you only have three hours: §1, §2, §6, §7, §8.** That produces a complete, defensible submission. §3 and §4 are what raise it from complete to strong.

### What changed from v1

| v1 said | v2 says | Why |
|---|---|---|
| Build two arms | **Keep one arm**, control Delivery with Form Display Logic | You've built one arm; it works, and v16 has the tool to close the gap |
| Use `[first-event-name]` | Use `[day_1_arm_1]` | With a single arm they're equivalent, and the explicit name is easier to read and explain |
| Prefix all variables by form | **Keep your names** | Renaming 67 fields the night before is pure downside risk |
| Add `dem_participant_type` | **Your `pregnant` field already does this** | It just needs hardening |
| Align code lists to the Task 2 export | **Withdrawn** | Their lists are worse than yours |

---

## 1. Six breaking fixes — do these first

### Where each one lives

| # | Field | Instrument | Event | Navigation |
|---|---|---|---|---|
| 1.1 | `age` | **Demographics** | Day 1 | Designer → Demographics → `age` → *Calculation* |
| 1.2 | `d1_date` | **Demographics** | Day 1 | Designer → Demographics → `d1_date` → *Text Validation Min* |
| 1.3 | `bmi_above_35` | **Screening and Eligibility** | Day 1 | Designer → Screening and Eligibility → `bmi_above_35` |
| 1.4 | `eligible` | **Screening and Eligibility** | Day 1 | Designer → Screening and Eligibility → `eligible` |
| 1.5 | `specify_other` | **Delivery Data** | Delivery | Designer → Delivery Data → `specify_other` |
| 1.6 | *section header* | **Vaccination** | Day 1, Day 28 | Designer → Vaccination → `vaccination_date` → *Section Header* |

Two instruments carry four of the six — Demographics and Screening. Open those two first and you clear most of this section without switching forms.

---

### 1.1 `age` calculates from `'today'` 🔴

**Form: Demographics** · Day 1 · `Designer → Demographics → age → Calculation`

**Current:** `rounddown(datediff([dob], 'today', 'y'))`

REDCap warns against `today`/`now` inside calculations: the value only refreshes when the form is re-saved, so stored age drifts away from displayed age, and **Data Quality Rule E (incorrect calculated values) flags every record permanently**. Age in a trial also means *age at enrolment* — a fixed fact, not a moving one.

**Change to:**
```
rounddown(datediff([dob],[d1_date],"y"))
```

Then use **Test this calculation** before leaving the field.

⚠️ If your version rejects that argument order, try `datediff([dob],[d1_date],"y","dmy",true)`.

### 1.2 `d1_date` has `min = today` 🔴

**Form: Demographics** · Day 1 · `Designer → Demographics → d1_date → Text Validation Min`

That forces Day 1 to be the exact date of data entry. A site typing up yesterday's visit cannot record the correct date.

**Delete the min. Keep `max = today`.**

### 1.3 `bmi_above_35` calculation is invalid syntax 🔴

**Form: Screening and Eligibility** · Day 1 · `Designer → Screening and Eligibility → bmi_above_35`

**Current:** `if([bmi]>35,0="Yes",1="No")`

`if()` takes `if(condition, value_if_true, value_if_false)` — equals-signs inside the arguments are not syntax. A standard `calc` field also returns numbers only; text output needs the `@CALCTEXT()` action tag in the **Field Annotation** column of a *text* field.

**Fix:** this field is being converted to a collected radio in §3.3 anyway. Change the field type to **radio**, choices `1, Yes | 0, No`, and delete the calculation.

### 1.4 `eligible` sums text values 🔴

**Form: Screening and Eligibility** · Day 1 · `Designer → Screening and Eligibility → eligible`

**Current:**
```
if([generally_healthy] + [age_18_or_older] + [icf_signed] + [previous_ebola_trial] + [previous_mpox_vaccination] + [bmi_above_35] = 6, "Yes", "No")
```

Two of those operands return strings, and you cannot add strings. A blank form would also evaluate to "No" rather than blank.

**Fix:** converted to a collected radio with a discrepancy warning in §3.3.

### 1.5 `specify_other` is a radio with no choices 🔴

**Form: Delivery Data** · Delivery event · `Designer → Delivery Data → specify_other`

A radio with an empty choice list renders as nothing at all.

**Change to:**

| Property | Value |
|---|---|
| Field type | **Text** |
| Label | Specify other delivery method |
| Branching logic | `[delivery_method] = '4'` |
| Required | Yes |

> This is exactly the P015 defect in the Task 2 export — method recorded as Other with the specification blank. Worth a line in your build note.

### 1.6 `[event_label]` on the Vaccination section header 🔴

**Form: Vaccination** · Day 1 and Day 28 · `Designer → Vaccination → vaccination_date → Section Header`

Underscore instead of hyphen. The smart variable is **`[event-label]`**. With the underscore REDCap pipes your descriptive field `event_label`, which holds no value, and renders blank.

**Change the Vaccination section header to:**
```
Vaccination information for [event-label]
```

While you're there: the standalone `event_label` descriptive field on Blood Sampling is redundant — the section header already pipes `[event-label]`. Delete it or leave it; no marks either way.

---

## 2. Structural gaps

### 2.1 Enable repeating instruments 🔴 — the biggest single fix

**Forms affected: Medical History · Adverse Events · Protocol Deviations** · `Project Setup → Main project settings`, then `Project Setup → Repeatable Instruments and Events → Modify`

Nothing in your project repeats. That fails **three explicit requirements**:

> *"It should be possible to record multiple medical history events for the same participant"*
> *"should allow multiple adverse events to be recorded for the same participant"*
> *"should allow multiple protocol deviations to be recorded for the same participant"*

**Steps:**

1. **Project Setup → Main project settings** → *"Use repeatable instruments and events?"* → **Enable**
2. **Project Setup → Repeatable Instruments and Events → Modify**

| Event | Instrument to set repeating | Custom instance label |
|---|---|---|
| Day 1 | Medical History | `[diagnosis]` |
| Adverse Event | Adverse Events | `[ae_term] ([ae_start_date])` |
| Protocol Deviation | Protocol Deviations | `[dev_date] – [dev_category]` |

**Use repeating *instrument*, not repeating event.** Day 1 holds four other forms, so that event cannot repeat. Instrument-level repeat produces a clean `redcap_repeat_instance` column in the export either way — which is what `MH_No`, `AE_No` and `PD_No` are in the Task 2 dataset.

The custom instance label is what turns "Instance 1, Instance 2, Instance 3" into a readable safety register. Two minutes, disproportionate payoff.

### 2.2 Verify Designate Instruments for My Events

**Forms affected: all eight** · `Designer → Designate Instruments for My Events`

**Designer → Designate Instruments for My Events.** Confirm the grid reads:

| Instrument | Day 1 | Day 28 | Day 42 | Delivery | Adverse Event | Protocol Deviation |
|---|:--:|:--:|:--:|:--:|:--:|:--:|
| Demographics | ✓ | | | | | |
| Medical History | ✓ | | | | | |
| Screening & Eligibility | ✓ | | | | | |
| Blood Sampling | ✓ | ✓ | ✓ | | | |
| Vaccination | ✓ | ✓ | | | | |
| Delivery Data | | | | ✓ | | |
| Adverse Events | | | | | ✓ | |
| Protocol Deviations | | | | | | ✓ |

If Blood Sampling is designated at Day 1 only, the three-visit sampling requirement is unmet **and** your `[event-label]` piping has nothing to vary across.

### 2.3 Close the Delivery form to non-pregnant participants 🟠

**Form: Delivery Data** · `Designer → Form Display Logic → Delivery Data`

> *"This form should only be applicable to pregnant women."*

With one arm, the Delivery event currently exists for all participants including adult men.

**Designer → Form Display Logic** (link above the instrument list). On **Delivery Data**:

```
[day_1_arm_1][pregnant] = '1'
```

The instrument then greys out for anyone not flagged pregnant at Day 1. Stronger than field-level branching, which would leave an empty-but-open form.

⚠️ *Verify placement in your instance* — Form Display Logic is per-instrument and applies across all events.

### 2.4 Harden the `pregnant` field 🟠

**Form: Demographics** · Day 1 · `Designer → Demographics → pregnant`

It now determines whether an entire CRF is available, so it cannot be optional.

| Property | Change to |
|---|---|
| Label | `Is the participant pregnant at enrolment?` |
| Required | **Yes** |
| Field note | `Determines whether the Delivery CRF becomes available. Must be completed for all female participants.` |
| Branching | Keep `[sex] = '2'` |

### 2.5 Participant ID 🟠

**Form: Demographics** · Day 1 · `Designer → Demographics → record_id` and `Project Setup → Additional customizations`

> *"Each participant must be uniquely identifiable using a Participant ID."*

`record_id` still carries the default label. Minimum fix: relabel it **Participant ID**.

Better, if you have five minutes: **Project Setup → Additional customizations → turn OFF auto-numbering**, and add a field note specifying the format (`P001`, `P002` — matching the Task 2 export convention).

Also set a **Custom Record Label**:
```
[sex] | [pregnant]
```
so the record picker shows something meaningful.

---

## 3. Form-by-form remediation

### 3.1 Demographics

| Variable | Change | Priority |
|---|---|---|
| `record_id` | Label → `Participant ID` | 🟠 |
| `dob` | Label → `Date of birth`; add `max = today`; set **Identifier? = y** | 🟠 |
| `age` | Fix calculation (§1.1); label → `Age at enrolment (years)`; field note: `Calculated automatically from date of birth and the Day 1 date.` | 🔴 |
| `sex` | Choices → `1, Male \| 2, Female` | 🟡 |
| `pregnant` | See §2.4 | 🟠 |
| `d1_date` | Remove `min`; label → `Date of Day 1 visit` | 🔴 |
| `height` | Label → `Height (cm)`; validation `number`, **min 100, max 220** | 🟠 |
| `weight` | Label → `Weight (kg)`; validation **`number`** (not `number_comma_decimal`), **min 30, max 200** | 🟠 |
| `bmi` | Label → `BMI (kg/m²)`; keep the calculation — it's correct | ✅ |

**Two things to fix beyond the table:**

**Decimal convention.** `height` uses `number` (period) and `weight` uses `number_comma_decimal` (comma). Mixed conventions feeding the same calculation is a real hazard. Standardise on `number`.

**Register.** Every label is second-person survey phrasing — *"What is your date of birth?"*, *"How tall are you?"*, *"What is you body weight?"*, *"Your BMI"*. This is a **CRF completed by study staff from source documents**, not a participant questionnaire. Nominal labels throughout, as in the table above. Also catches the `you body weight` typo.

### 3.2 Medical History

| Variable | Change | Priority |
|---|---|---|
| — | **Set repeating** (§2.1) | 🔴 |
| `diagnosis` | Required = y. Field note: `Record the diagnosis as documented in the source. One condition per record.` | 🟠 |
| `diagnosis` | ⚠️ **`BIOPORTAL:ICDO` is the wrong ontology** — ICD-O is oncology morphology/topography. Switch to ICD-10, or drop to plain text | 🟠 |
| `body_system` | Replace SNOMED lookup with a **radio list** (below) | 🟠 |
| `start_date` | `max = today`; Required = y | 🟠 |
| `end_date` | `max = today` | 🟠 |
| `ongoing` | Required = y | 🟠 |
| `medication` | Required = y | 🟠 |
| `medication_name` | Required = y (when shown) | 🟠 |

**On the ontology fields.** The ambition is good and I'd keep the idea if the instance supports it — but ICD-O is a clinical-coding error a reviewer with domain background spots instantly, and free SNOMED lookup on "body system" gives uncontrolled granularity (a site can pick *structure of left fourth toe*). ⚠️ Also confirm BioPortal has an API key configured on your instance, or the field silently degrades.

**`body_system` choices:**
```
1, Cardiovascular
2, Respiratory
3, Gastrointestinal / hepatic
4, Renal / genitourinary
5, Neurological
6, Musculoskeletal
7, Endocrine / metabolic
8, Haematological
9, Immune / allergic
10, Infectious disease
11, Dermatological
12, Psychiatric
99, Other
```
Add `body_system_other` (text, branching `[body_system] = '99'`, required when shown).

### 3.3 Screening and Eligibility — the highest-value form

The assignment says the form *"should **collect** Yes/No responses"* for all six criteria, and asks for consistency to be enforced *"with an appropriate warning or check"*. Your current build **derives** two criteria and the overall decision, which means:

- Two criteria are never collected, only computed
- The overall decision can never disagree with the criteria — so **there is no check to demonstrate**
- A participant enrolled in error (the P003 scenario) becomes **unrecordable**: the database would show `eligible = No` for someone who is, in fact, enrolled, and nothing captures the investigator's actual decision or triggers the deviation

**Restructure to: collect all six, derive nothing, warn on disagreement.**

| # | Variable | Field type | Choices | Required |
|---|---|---|---|---|
| — | *Section: Inclusion criteria* — `Answer Yes only if the criterion is met.` | | | |
| 1 | `generally_healthy` | yesno | — | y |
| 2 | `age_18_or_older` | **radio** *(was calc)* | `1, Yes \| 0, No` | y |
| 3 | `w_age_check` | descriptive | warning §4.1 | — |
| 4 | `icf_signed` | yesno | — | y |
| — | *Section: Exclusion criteria* — `Answer Yes if the criterion is PRESENT. A Yes here excludes the participant.` | | | |
| 5 | `previous_ebola_trial` | radio | **`1, Yes \| 0, No`** *(flip — see below)* | y |
| 6 | `previous_mpox_vaccination` | radio | **`1, Yes \| 0, No`** *(flip)* | y |
| 7 | `bmi_above_35` | **radio** *(was calc)* | `1, Yes \| 0, No` | y |
| 8 | `w_bmi_check` | descriptive | warning §4.2 | — |
| — | *Section: Eligibility assessment* | | | |
| 9 | `eligible` | **radio** *(was calc)* | `1, Yes \| 0, No` | y |
| 10 | `w_eligible` | descriptive | warning §4.3 | — |

#### Flip the exclusion coding

You currently have `0, Yes | 1, No` — a deliberate inversion so the eligibility sum worked. Clever, but it leaves a permanent trap: in any export, `previous_ebola_trial = 1` means **No**. Every downstream analyst will misread it, and it is invisible to whoever fills the form.

Since the summing calculation is going away, flip to natural coding: **`1, Yes | 0, No`**. All logic in §4 is written for natural coding.

> ⚠️ **If you decide to keep the inverted coding**, invert every exclusion term in the §4.3 logic: `= '1'` becomes `= '0'` and vice versa for the three exclusion fields.

#### Field notes to add here

- On each exclusion item: `Yes = criterion present (exclusionary).`
- On `eligible`: `Record the investigator's enrolment decision. If this differs from the criteria above, a protocol deviation must be documented.`

### 3.4 Blood Sampling

| Variable | Change | Priority |
|---|---|---|
| `sample_collected` | Required = y | 🟠 |
| `reason_not_collected` | Branching → `[sample_collected] = '0'` (quote it); Required = y | 🟡 |
| `sample_barcode` | Branching → `[sample_collected] = '1'`; **Required = y**; keep `@BARCODE-APP` | 🟠 |
| `sample_date` | Branching → `[sample_collected] = '1'`; `max = today`; Required = y | 🟠 |
| `sample_time` | Validation → **`time` (hh:mm)** not `hh:mm:ss`; Required = y | 🟡 |
| `w_sample_seq` | New descriptive — warning §4.4 | 🟡 |

**Required on `sample_barcode` is the important one.** In the Task 2 export, P017–P020 have `Sample_Collected = Yes` with a blank barcode — four consecutive participants, which points at a systematic site issue *and* at a field that was never required.

**Standardise the branching quotes.** You mix `[sample_collected]=1` here with `[ongoing] = '0'` on Medical History. Use `= '1'` consistently.

### 3.5 Vaccination

| Variable | Change | Priority |
|---|---|---|
| Section header | `[event_label]` → **`[event-label]`** | 🔴 |
| `vaccination_date` | `max = today`; Required = y | 🟠 |
| `vaccination_time` | Required = y | 🟠 |
| `vaccination_arm` | Required = y; field note: `Arm in which the study vaccine was administered.` | 🟡 |
| `adverse_reaction` | Required = y; field note: `30-minute observation period post-vaccination.` | 🟡 |
| `reaction_description` | Branching → `[adverse_reaction] = '1'`; Required = y | 🟡 |
| `w_vac_ae` | New descriptive — prompt §4.5 | 🟡 |
| `w_vac_date` | New descriptive — warning §4.6 | 🟡 |

**No `@TODAY` or `@NOW` defaults on any date or time field here.** In the Task 2 export, `Vaccination_Time` is `10:00` on 47 of 48 rows and `Collection_Time` is 09:00 / 09:10 / 09:15 on essentially every record — the signature of pre-filled defaults. Data that looks complete and is silently unverified is worse than data that looks incomplete. Say this out loud if asked; it is one of the strongest lines available to you.

### 3.6 Adverse Events

| Variable | Change | Priority |
|---|---|---|
| — | **Set repeating** (§2.1) | 🔴 |
| `ae_term` | Required = y; field note: `Record the diagnosis where available, otherwise the verbatim reported term. One event per record.` | 🟠 |
| `ae_start_date` | `max = today`; Required = y | 🟠 |
| `ae_ongoing` | Required = y | 🟠 |
| `ae_end_date` | `max = today`; Required = y (when shown) | 🟠 |
| `w_ae_dates` | New descriptive — warning §4.7 | 🟡 |
| `ae_severity` | Required = y; **field note below** | 🟠 |
| `ae_serious` | Required = y; **field note below** | 🟠 |
| `action_taken` | **text → radio**, choices below; Required = y | 🟠 |
| `action_taken_spec` | **New** text; branching `[action_taken] = '2' or [action_taken] = '99'` | 🟠 |
| `outcome` | **text → radio**, choices below; Required = y | 🟠 |
| `w_ae_outcome` | New descriptive — warning §4.8 | 🟡 |
| `sae_type` | **text → checkbox**, criteria below; **branching `[ae_serious] = '1'`** | 🔴 |
| `sae_date_investigator` | **Branching `[ae_serious] = '1'`**; `max = today`; Required = y | 🔴 |
| `sae_date_sponsor` | **Branching `[ae_serious] = '1'`**; `max = today`; Required = y | 🔴 |
| `w_sae_timing` | New descriptive — warning §4.9 | 🟠 |

**The branching on the three SAE fields is a direct requirement:**

> *"The form should behave appropriately depending on the information entered, for example whether the event is ongoing or serious."*

They currently display for every adverse event, serious or not.

**`action_taken` choices:**
```
1, None
2, Medication administered
3, Non-drug therapy administered
4, Study vaccination delayed
5, Study vaccination discontinued
6, Participant hospitalised
7, Surgical or other procedure
99, Other
```
`action_taken_spec` label: `Specify medication or other action`.

> The Task 2 export mixes drug names (`Paracetamol`, `Antihistamine`) with actions (`Observation`, `Hospitalized`, `Surgery`) in a single free-text column. Separating the coded category from the specification is the fix — and pointing at that as a *database configuration issue* in your Task 2 review is worth a finding.

**`outcome` choices (CIOMS-aligned):**
```
1, Recovered / resolved
2, Recovering / resolving
3, Not recovered / not resolved
4, Recovered / resolved with sequelae
5, Fatal
6, Unknown
```

> The Task 2 export only has *Recovered* and *Recovering* — meaning a fatal outcome could not be recorded at all. Another configuration finding.

**`sae_type` — checkbox, not radio:**
```
1, Results in death
2, Is life-threatening
3, Requires inpatient hospitalisation or prolongation of existing hospitalisation
4, Results in persistent or significant disability / incapacity
5, Is a congenital anomaly / birth defect
6, Other medically important event
```
The assignment says "Type of the SAE" in the singular; a checkbox is clinically correct because an SAE can meet several criteria simultaneously. Note the deliberate deviation in your build note — that reads as judgement, not as a missed requirement.

**The two field notes that matter most:**

On `ae_severity`:
> *Severity describes the intensity of the event as assessed by the investigator. It is independent of seriousness.*

On `ae_serious`:
> *Seriousness is a regulatory classification. Answer Yes only if the event meets at least one of the criteria below. **A severe event is not automatically a serious event.***

Severity-versus-seriousness is the commonest source of AE data error in any trial. Putting the distinction in the CRF is exactly what the assignment means by instructions *"sufficiently clear for study staff to complete the CRF consistently without requiring additional explanation from the Data Manager."*

### 3.7 Protocol Deviations

| Variable | Change | Priority |
|---|---|---|
| — | **Set repeating** (§2.1) | 🔴 |
| `dev_lassification` | **Typo** — rename to `dev_classification` | 🟡 |
| `dev_date` | `max = today`; Required = y | 🟠 |
| `dev_category` | **text → radio**, choices below; Required = y | 🟠 |
| `dev_category_other` | **New** text; branching `[dev_category] = '99'` | 🟠 |
| `dev_description` | **text → notes**; Required = y; field note: `Describe what happened, when it was identified, and which participants or procedures were affected.` | 🟠 |
| `dev_classification` | Required = y; **field note below** | 🟠 |
| `dev_corrective_action` | **text → notes**; Required = y | 🟡 |

**`dev_category` choices:**
```
1, Eligibility / inclusion or exclusion criteria
2, Informed consent
3, Visit schedule or visit window
4, Study procedure not performed or performed incorrectly
5, Study product administration
6, Sample collection, handling or storage
7, Safety reporting
8, Documentation or data collection
99, Other
```

Field note on `dev_classification`:
> *Major: a deviation that may affect participant safety, participant rights, or the integrity of the study data. Minor: all other deviations. If unsure, classify as Major and discuss with the sponsor.*

### 3.8 Delivery Data

| Variable | Change | Priority |
|---|---|---|
| — | **Form Display Logic** `[day_1_arm_1][pregnant] = '1'` (§2.3) | 🟠 |
| `specify_other` | **radio → text**; branching `[delivery_method] = '4'`; Required = y | 🔴 |
| `delivery_date` | `max = today`; Required = y | 🟠 |
| `delivery_time` | Validation → `time` (hh:mm); Required = y; **fix "Tim of delivery" typo** | 🟡 |
| `delivery_timing` | Required = y | 🟠 |
| `delivery_method` | Required = y | 🟠 |
| `apgar1_*` (5 fields) | Required = y | 🟡 |
| `apgar2_*` (5 fields) | Required = y; **consider renaming to `apgar5_*`** — it's the 5-minute score | 🟡 |
| `w_apgar` | New descriptive — warning §4.10 | 🟡 |
| `w_delivery_date` | New descriptive — warning §4.11 | 🟡 |

**Keep the Apgar decomposition.** Five components × two timepoints with calculated totals is the best thing in your dictionary — clinically correct and it removes the arithmetic error entirely. It is twelve fields where the spec asked for two, so be ready to justify it: *"the total is what the spec asked for, but a transcribed total cannot be verified against source, whereas the five components can."*

---

## 4. Cross-field warning library

### The pattern

REDCap has no native "warn but allow" rule, so build one:

1. Add a **Descriptive Text** field.
2. Field label = HTML:
```html
<div style="color:#B00020;font-weight:bold;padding:8px;border:1px solid #B00020;border-radius:4px;">
⚠ [warning text here]
</div>
```
3. Branching logic = the **inconsistent** condition.
4. Field annotation: **`@HIDDEN-PDF`** — keeps warnings off the paper CRF.

The field appears only when the data are inconsistent, is impossible to miss, and never blocks a legitimate entry. Each one is paired with a Data Quality rule in §5 as the backstop for anything arriving by import or later edit.

### Where the eleven warnings go

| Form | Warnings to add |
|---|---|
| Screening and Eligibility | `w_age_check` · `w_bmi_check` · `w_eligible` |
| Blood Sampling | `w_sample_seq` |
| Vaccination | `w_vac_ae` · `w_vac_date` |
| Adverse Events | `w_ae_dates` · `w_ae_outcome` · `w_sae_timing` |
| Delivery Data | `w_apgar` · `w_delivery_date` |
| Demographics · Medical History · Protocol Deviations | *none — validation and required fields are sufficient* |

---

### 4.1 `w_age_check` — Screening

**Add to: Screening and Eligibility**, immediately after `age_18_or_older`.

```
([age_18_or_older] = '1' and [age] <> "" and [age] < 18)
or ([age_18_or_older] = '0' and [age] <> "" and [age] >= 18)
```
> *The age criterion does not agree with the age calculated from date of birth ([age] years). Verify against source before saving.*

### 4.2 `w_bmi_check` — Screening

**Add to: Screening and Eligibility**, immediately after `bmi_above_35`.

```
([bmi_above_35] = '0' and [bmi] <> "" and [bmi] > 35)
or ([bmi_above_35] = '1' and [bmi] <> "" and [bmi] <= 35)
```
> *The BMI criterion does not agree with the BMI calculated from height and weight ([bmi] kg/m²). Verify measurements and criterion before saving.*

### 4.3 `w_eligible` — Screening

**Add to: Screening and Eligibility**, immediately after `eligible` — the last field on the form.

Written for **natural** exclusion coding (`1, Yes | 0, No`):

```
([eligible] = '1' and ([generally_healthy] = '0' or [age_18_or_older] = '0' or [icf_signed] = '0' or [previous_ebola_trial] = '1' or [previous_mpox_vaccination] = '1' or [bmi_above_35] = '1'))
or
([eligible] = '0' and [generally_healthy] = '1' and [age_18_or_older] = '1' and [icf_signed] = '1' and [previous_ebola_trial] = '0' and [previous_mpox_vaccination] = '0' and [bmi_above_35] = '0')
```
> *The overall eligibility decision does not agree with the individual criteria recorded above. If the participant is being enrolled regardless, a protocol deviation must be documented.*

### 4.4 `w_sample_seq` — Blood Sampling

**Add to: Blood Sampling**, after `sample_time`. Appears at Day 1, Day 28 and Day 42.

```
[sample_date] <> "" and [day_1_arm_1][d1_date] <> "" and [sample_date] < [day_1_arm_1][d1_date]
```
> *The sample collection date precedes the Day 1 visit date. Verify against source.*

Works because REDCap stores dates as `YYYY-MM-DD`, so a string comparison is also a chronological one.

### 4.5 `w_vac_ae` — Vaccination *(prompt, not error — style it blue)*

**Add to: Vaccination**, immediately after `reaction_description`.

```
[adverse_reaction] = '1'
```
> *An immediate reaction has been recorded. Ensure a corresponding entry is completed on the Adverse Events form.*

### 4.6 `w_vac_date` — Vaccination

**Add to: Vaccination**, after `w_vac_ae`.

```
[vaccination_date] <> "" and [sample_date] <> "" and [vaccination_date] <> [sample_date]
```
> *The vaccination date differs from the blood sampling date at this visit. Confirm whether the visit was split across two days; if so, document a protocol deviation.*

### 4.7 `w_ae_dates` — Adverse Events

**Add to: Adverse Events**, immediately after `ae_end_date`.

```
[ae_end_date] <> "" and [ae_start_date] <> "" and [ae_end_date] < [ae_start_date]
```
> *The end date precedes the start date. Correct before saving.*

### 4.8 `w_ae_outcome` — Adverse Events

**Add to: Adverse Events**, immediately after `outcome`.

```
[ae_ongoing] = '1' and ([outcome] = '1' or [outcome] = '4' or [outcome] = '5')
```
> *The event is recorded as ongoing but the outcome indicates it has resolved or is fatal. Review and correct.*

### 4.9 `w_sae_timing` — Adverse Events

**Add to: Adverse Events**, at the end of the SAE section, after `sae_date_sponsor`. Give it the same `[ae_serious] = '1'` condition as the rest of the block, combined with the logic below.

```
[sae_date_investigator] <> "" and [sae_date_sponsor] <> ""
and datediff([sae_date_investigator],[sae_date_sponsor],"d",true) > 1
```
> *The SAE was reported to the sponsor more than 24 hours after the investigator became aware. This is a reportable protocol deviation — complete the Protocol Deviations form.*

⚠️ *Verify `datediff` works in branching logic on your version.* If not, move this to a Data Quality rule only.

> This is the P007 case in the Task 2 export: aware 18 June, reported 22 June.

### 4.10 `w_apgar` — Delivery

**Add to: Delivery Data**, after `apgar_score_after_5_minutes`.

```
[apgar_score_after_1_minute] <> "" and [apgar_score_after_5_minutes] <> ""
and [apgar_score_after_5_minutes] < [apgar_score_after_1_minute]
```
> *The 5-minute Apgar score is lower than the 1-minute score. This is possible but uncommon — verify against source.*

### 4.11 `w_delivery_date` — Delivery

**Add to: Delivery Data**, after `delivery_time`.

```
[delivery_date] <> "" and [day_1_arm_1][d1_date] <> "" and [delivery_date] < [day_1_arm_1][d1_date]
```
> *The delivery date precedes the Day 1 visit date. Verify against source.*

---

## 5. Query management and data quality

### 5.1 Data Resolution Workflow — the answer to "query management"

> *"The database should provide a mechanism that allows data queries to be raised and followed up when data require clarification or correction."*

**Project Setup → Enable optional modules and customizations → Additional customizations → Data Resolution Workflow.**

Choose the **full Data Resolution Workflow**, not plain Field Comment Log:

| Field Comment Log | Data Resolution Workflow |
|---|---|
| Free-text notes on a field | A query object: Open → Responded → Closed |
| No status, no assignment | Assignable to a user, with response threads |
| No dashboard | **Resolve Issues** dashboard, filterable by status |

I can see **Field Comment Log** in your left sidebar but not a Resolve Issues entry — so this is very likely still off. It is one checkbox and it answers a named requirement.

Then set **User Rights → Data Resolution Workflow** per role: data managers open and close, site staff respond only. Worth a sentence in the build note.

### 5.2 Data Quality rules

**Data Quality → Rules → Add new rule.** Leave predefined rules A–H active — Rule E (incorrect calculated values) in particular catches calculation drift after design changes.

Data Quality rules are **project-level, not attached to a form** — you build them once under Data Quality, and REDCap evaluates them across every record and event. The *Form* column below is only there to tell you which instrument a discrepancy will point at when a rule fires.

Custom rules, all using your variable names:

| # | Name | Points at | Logic |
|---|---|---|---|
| 1 | Age criterion inconsistent | Screening | `[age] <> "" and [age] < 18 and [age_18_or_older] = '1'` |
| 2 | BMI criterion inconsistent | Screening | `[bmi] <> "" and [bmi] > 35 and [bmi_above_35] = '0'` |
| 3 | Eligible despite failing criterion | Screening | `[eligible] = '1' and ([previous_ebola_trial] = '1' or [previous_mpox_vaccination] = '1' or [bmi_above_35] = '1' or [generally_healthy] = '0' or [icf_signed] = '0' or [age_18_or_older] = '0')` |
| 4 | Sample collected, barcode missing | Blood Sampling | `[sample_collected] = '1' and [sample_barcode] = ""` |
| 5 | Sample not collected, no reason | Blood Sampling | `[sample_collected] = '0' and [reason_not_collected] = ""` |
| 6 | Vaccination and sampling dates differ | Vaccination | `[vaccination_date] <> "" and [sample_date] <> "" and [vaccination_date] <> [sample_date]` |
| 7 | AE end before start | Adverse Events | `[ae_end_date] <> "" and [ae_end_date] < [ae_start_date]` |
| 8 | AE ongoing but resolved outcome | Adverse Events | `[ae_ongoing] = '1' and ([outcome] = '1' or [outcome] = '5')` |
| 9 | SAE reported late | Adverse Events | `[sae_date_investigator] <> "" and [sae_date_sponsor] <> "" and datediff([sae_date_investigator],[sae_date_sponsor],"d",true) > 1` |
| 10 | Delivery before enrolment | Delivery Data | `[delivery_date] <> "" and [delivery_date] < [day_1_arm_1][d1_date]` |
| 11 | Delivery recorded, not flagged pregnant | Delivery Data | `[delivery_date] <> "" and [day_1_arm_1][pregnant] <> '1'` |
| 12 | Other method, no specification | Delivery Data | `[delivery_method] = '4' and [specify_other] = ""` |

Enable **real-time execution** on rules 1–5 so discrepancies surface at entry rather than at the next review cycle.

The overlap with §4 is deliberate: the inline warning catches errors at entry, the DQ rule catches everything arriving by import, from a user who ignored the warning, or through a later edit. That redundancy is the point, and it is worth stating.

---

## 6. Paper CRF

This is a named deliverable and easy marks are lost here.

**Generate:** Project Setup / Other Functionality → **Download PDF of all instruments → All instruments (blank)**. If your version offers a longitudinal variant showing which form belongs to which event, use it.

**Then check against these:**

| Problem | Fix |
|---|---|
| Warning fields print as red blocks and confuse a paper user | `@HIDDEN-PDF` on every `w_*` descriptive field |
| Branching-hidden fields missing from the PDF | Turn on the setting to include branching-logic fields, otherwise the paper CRF has holes |
| Calculated fields (`age`, `bmi`, Apgar totals) print as empty boxes with no explanation | Field note: `Calculated automatically in the database — leave blank on paper.` |
| Dropdown fields may print as a blank line without their choices ⚠️ *verify on your instance* | Your only dropdown is `ae_severity` — switch it to radio if the choices don't print |
| No room to write | Check `ae_term`, `dev_description`, `dev_corrective_action` — notes fields print a box, single-line text prints a line |
| No completion header | Add a section header on Demographics with Participant ID, site, initials of the person completing, and date completed |

---

## 7. Test records

Enter **two records**. This is not optional — it is how you prove the build works and it takes thirty minutes.

**Record 1 — `P001`, clean adult male**
- Day 1: full Demographics, one Medical History entry, all Screening = eligible, Blood Sampling collected with barcode, Vaccination
- Day 28, Day 42: Blood Sampling; Vaccination at Day 28
- One non-serious AE
- Confirm: the **Delivery form is greyed out** — this proves Form Display Logic works

**Record 2 — `P002`, pregnant, deliberately inconsistent**
- Demographics: female, pregnant = Yes, height/weight giving **BMI > 35**
- Screening: `bmi_above_35` = **No**, `eligible` = **Yes** → `w_bmi_check` and `w_eligible` must both fire
- **Three** Medical History entries → proves repeating
- **Two** AEs, one serious, aware 18 June, reported to sponsor 22 June → `w_sae_timing` must fire
- One Protocol Deviation
- Delivery: method = Other, and confirm it will not save without `specify_other`

Then **Data Quality → Run all rules** and confirm the discrepancies appear.

If you have five minutes more: **open one query through the Data Resolution Workflow and leave it open.** The reviewers then see the query mechanism working live rather than taking your word for it.

---

## 8. Submission pack

### 8.1 Checklist

- ☐ Project access for **`Ylarivi`** and **`mabuazoum`**, with **Project Design and Setup** rights — they need to inspect configuration, not just data
- ☐ PDF of all instruments (blank), checked against §6
- ☐ **Data Dictionary export (CSV)** — fastest way for a reviewer to audit every field and logic string
- ☐ **Codebook** printed to PDF — human-readable companion
- ☐ One-page build note (§8.2)
- ☐ Two test records present, Data Quality rules run
- ☐ Project left in **Development** — see below

### 8.2 The build note — content

One page. It converts design decisions a reviewer might miss into decisions they can see you made deliberately.

**1. Structure.** Single arm, six events: three scheduled visits, Delivery, and two unscheduled events for Adverse Events and Protocol Deviations. AEs and PDs are participant-level rather than visit-bound; a rolling event gives one chronological register per participant with continuous instance numbering. Delivery is restricted to pregnant participants via Form Display Logic on `[day_1_arm_1][pregnant]`.

**2. Derived versus collected.** Age and BMI are calculated to remove a class of transcription error at source. All six eligibility criteria and the overall decision are **collected**, with derived values used only to trigger discrepancy warnings — because the investigator's enrolment decision is a regulatory record, and auto-deriving it would make an erroneous enrolment unrecordable.

**3. No default dates or times.** Deliberately no `@TODAY` / `@NOW` on collection dates and times. Pre-filled timestamps produce data that appears complete and is silently unverified.

**4. Two-layer data quality.** Inline warnings at entry, Data Quality rules as the backstop for imported or later-edited data. Query management via the Data Resolution Workflow with role-differentiated rights.

**5. Deliberate deviations from the specification.**
- `sae_type` is a checkbox rather than a single selection, because an SAE can meet several seriousness criteria at once.
- Apgar is captured as five components at each timepoint with a calculated total, because a transcribed total cannot be verified against source.

**6. Observations on the specification.** Two gaps, raised rather than silently filled:
- **No confirmation of pregnancy or gestational age is collected anywhere**, yet the Delivery form asks for pre-term / term / post-term. That classification is unverifiable and unrecomputable at analysis. Recommend an estimated date of delivery or gestational age at enrolment.
- **No informed consent date is collected**, only a Yes/No that the ICF was signed. The database therefore cannot evidence that consent preceded any study procedure.

### 8.3 Project status

Leave it in **Development**. Reviewers can then inspect and modify freely, which is what they need.

Add one line to your email: *"The project remains in Development so the configuration can be reviewed and modified; in a live study it would move to Production only after user acceptance testing and Data Management Plan sign-off."* That earns the lifecycle credit with none of the locking risk.

---

## 9. Defending the design

Three challenges you should expect, with the answer prepared.

**"Why one arm rather than two?"**
> Arms require the participant group to be known at record creation. Here pregnancy status is captured on the Day 1 CRF, so the group is derived from collected data rather than assigned administratively. A single arm keeps one event schedule and one set of cross-event references to maintain, and Form Display Logic closes the Delivery form for non-pregnant participants. The trade-off I accepted is that the Delivery column appears greyed out on the Record Status Dashboard for every participant; two arms would keep that cleaner.

**"Why did you make the eligibility decision manual when you could calculate it?"**
> Because eligibility is a regulatory act by the investigator, not an arithmetic output. If I auto-derive it, a participant enrolled in error shows as ineligible in a database where they are in fact enrolled — I lose the record of the actual decision and nothing triggers the protocol deviation. The database's job is to make the disagreement impossible to ignore and point the site at the deviation form.

**"Why put Adverse Events in their own event?"**
> An adverse event has a start date and an end date; it does not have a visit. If I attach the form to Day 1, Day 28 and Day 42, I get three independent repeating containers per participant, instance numbering restarts in each, and an event spanning two visits has no obvious home. A single rolling event gives one chronological safety register, which is what the safety physician and the statistician both need — and it matches how the data are actually structured in the export.

---

## 10. Quick reference

### Your events

| Event | Unique name |
|---|---|
| Day 1 | `day_1_arm_1` |
| Day 28 | `day_28_arm_1` |
| Day 42 | `day_42_arm_1` |
| Delivery | `delivery_arm_1` |
| Adverse Event | `adverse_event_arm_1` |
| Protocol Deviation | `protocol_deviation_arm_1` |

### Logic syntax

| Need | Syntax |
|---|---|
| Same event | `[bmi]` |
| Cross-event | `[day_1_arm_1][d1_date]` |
| Checkbox option selected | `[sae_type(3)] = '1'` |
| Not equal | `<>` |
| Guard against blank | `[field] <> ""` |
| Date difference in days | `datediff([d1],[d2],"d",true)` |
| Conditional | `if([a] = '1' and [b] = '0', 1, 0)` |
| Round down | `rounddown(x)` · Round to n places `round(x, 1)` |

### Action tags used

| Tag | Where |
|---|---|
| `@BARCODE-APP` | `sample_barcode` |
| `@HIDDEN-PDF` | every `w_*` descriptive field |
| `@TODAY` / `@NOW` | **deliberately not used** |

### Smart variables used

| Variable | Where |
|---|---|
| `[event-label]` | Blood Sampling and Vaccination section headers — **hyphen, not underscore** |

---

*Version v3 — 16 August 2026. Supersedes REDCap_Task1_Build_Guide_v2.md.*
