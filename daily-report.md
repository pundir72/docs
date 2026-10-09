# Stance Daily Report

**9 October 2026, 4:47 PM IST**  
**Active patients: 7,864**

Includes patient accounts with `isActive = true`. All percentages use 7,864 active patients. Figures reflect the supplied 16:47:15 IST snapshot.

## Overall Coverage

| Service | Have data | % | Missing data | % |
|---|---:|---:|---:|---:|
| Prognosis | 4,589 | 58.4% | 3,275 | 41.6% |
| Summary | 3,323 | 42.3% | 4,541 | 57.7% |
| Phase analysis | 3,555 | 45.2% | 4,309 | 54.8% |
| VALD measurements | 3,793 | 48.2% | 4,071 | 51.8% |

Phase coverage uses the production collection, `new-patient-phases`.

## Prognosis

A complete first assessment contains chief complaint, clinical history, subjective assessment, and provisional diagnosis.

| Category | Patients | % |
|---|---:|---:|
| Complete assessment; prognosis generated | 3,137 | 39.9% |
| Complete assessment; prognosis missing | 46 | 0.6% |
| Assessment absent, empty, or incomplete | 4,681 | 59.5% |
| **Total** | **7,864** | **100%** |

Of the 46 patients with a complete assessment but no prognosis, 7 have multiple first assessments and 39 have no confirmed execution status.

The 4,681 patients without a complete assessment include 1,452 with saved prognosis. These are included in the overall prognosis count of 4,589.

## Summary and Phase Analysis

| Category | Summary | % | Phase | % |
|---|---:|---:|---:|---:|
| Generated output exists | 3,323 | 42.3% | 3,555 | 45.2% |
| Output missing | 4,541 | 57.7% | 4,309 | 54.8% |
| **Total active patients** | **7,864** | **100%** | **7,864** | **100%** |

### Additional condition breakdown

These rows split the generated and missing totals above; they do not replace them. All percentages use 7,864 active patients.

| Category | Summary | % | Phase | % |
|---|---:|---:|---:|---:|
| Saved-data condition matched; output generated | 1,727 | 22.0% | 1,725 | 21.9% |
| Saved-data condition matched; output missing | 203 | 2.6% | 205 | 2.6% |
| Condition not established from saved data; output generated | 1,596 | 20.3% | 1,830 | 23.3% |
| Condition not established from saved data; output missing | 4,338 | 55.2% | 4,104 | 52.2% |
| **Total active patients** | **7,864** | **100%** | **7,864** | **100%** |

**Condition matched:** At least 5 distinct saved reports with content, or 5 distinct saved VALD profile/document references, or a database-confirmed 15-day fallback with activity. This is a current saved-data check, not a reconstruction of past scheduler decisions.

**Condition not established:** The current records do not establish that rule. This does not mean the patient failed the generation conditions when their output was created. Completed processing can clear activity counters; direct requests and earlier fallback processing can also produce outputs. Individual historical triggers have not been verified.

**How the totals reconcile:** Summary generated is 1,727 + 1,596 = **3,323**. Phase generated is 1,725 + 1,830 = **3,555**. Missing totals are 203 + 4,338 = **4,541** and 205 + 4,104 = **4,309**, respectively.

**Earlier Phase count correction:** The previously reported 1,704 came from the legacy collection. Production writes to `new-patient-phases`, which gives **3,555** for this snapshot. The matched group of 1,725 is only part of that total.

## VALD Connection

| Category | Patients | % |
|---|---:|---:|
| Valid profile mapping | 6,659 | 84.7% |
| Not mapped | 1,205 | 15.3% |
| **Total** | **7,864** | **100%** |

## VALD Measurements

| Category | Patients | % |
|---|---:|---:|
| Saved measurements exist | 3,793 | 48.2% |
| Mapped; no saved measurements; reason unknown | 2,866 | 36.4% |
| **Total for these two categories** | **6,659** | **84.7%** |

The subtotal covers only the two categories shown. Overall missing-measurement coverage is shown in the first table.

## Yesterday — 8 October 2026, IST

| Service | Patients with first creation recorded | % |
|---|---:|---:|
| Prognosis | 4 | 0.1% |
| Summary | 8 | 0.1% |
| Phase analysis | 13 | 0.2% |

Counts cover 8 October, 00:00 IST to 9 October, 00:00 IST, using the earliest retained creation timestamps for currently active patients. They are not verified lifetime first-ever generations where older records may have been deleted or migrated.

## Pending Validation

| Item | Status |
|---|---|
| VALD freshness | Withheld pending reconciliation with the existing Slack report's population and date fields |
| First VALD measurement sync yesterday | Not available |

Missing data does not automatically indicate a service failure. Percentages may not total 100% due to rounding.
