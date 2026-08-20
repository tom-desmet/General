# REDCap Task 1 — Build Console

The build guide for the Data Manager assessment, Task 1, as an interactive working page.

## Files

| File | What it is |
|---|---|
| `build-console.html` | **The console.** Open in any browser — no build step, no dependencies, no network access. |
| `REDCap_Task1_Build_Guide_v3.md` | The source guide it was built from |
| `optional/bmi-integration-addendum.md` | A parked design note, not referenced by the guide — see below |

## What the console does

The v3 guide is ~800 lines of markdown you scroll and lose your place in. The console is the same
content, restructured around how you actually work: in REDCap Designer, one form at a time, with a
deadline.

- **38 build steps as checkboxes**, with progress tracked overall and per section. State persists in
  the browser, so you can close the tab and come back.
- **Copy button on all 42 logic strings**, choice lists, and field notes — nothing gets retyped into
  REDCap by hand.
- **Group by instrument.** One toggle turns the section-ordered guide into nine per-form lists
  (Demographics, Medical History, Screening, Blood Sampling, Vaccination, Adverse Events, Protocol
  Deviations, Delivery, and project-level), each with its Designer path and its own done-count. Open
  one form in REDCap, work its list, close it.
- **Filter** by instrument or by priority (critical / high / polish), hide what's already done, and
  free-text search every step for a variable name.
- **Section navigation** with scroll-spy and a per-section completion count.

Everything from the guide is carried over: the six breaking fixes, the structural gaps, the
form-by-form remediation, the eleven-warning library, the Data Resolution Workflow and twelve Data
Quality rules, the paper CRF checks, test records, the submission pack, and the design defences.

## The optional addendum

`optional/bmi-integration-addendum.md` proposes deriving the age and BMI screening criteria from the
values Form 1 already calculates, rather than having staff retype them as Yes/No. It was drafted
before the scope of this work was clarified and is kept only in case it's wanted later.

**It is not wired into the console or the guide.** Following the guide as written requires none of
it, and `build-console.html` is a faithful interactive version of v3.
