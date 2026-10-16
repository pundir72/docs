# Stance Daily Report

**16 October 2026, 1:52 PM IST**  
**Active patients: 8,184**

Includes patient accounts with `isActive = true`. All percentages use 8,184 active patients. Figures reflect the 13:52:04 IST snapshot.

## Overall Coverage

| Service | Have data | % | Missing data | % |
|---|---:|---:|---:|---:|
| Prognosis | 4,667 | 57.0% | 3,517 | 43.0% |
| Summary | 3,344 | 40.9% | 4,840 | 59.1% |
| Phase analysis | 3,593 | 43.9% | 4,591 | 56.1% |
| VALD measurements | 3,863 | 47.2% | 4,321 | 52.8% |

Phase coverage uses the production collection, `new-patient-phases`.

## Prognosis

A complete first assessment contains chief complaint, clinical history, subjective assessment, and provisional diagnosis.

| Category | Patients | % |
|---|---:|---:|
| Complete assessment; prognosis generated | 3,216 | 39.3% |
| Complete assessment; prognosis missing | 45 | 0.5% |
| Assessment absent, empty, or incomplete | 4,923 | 60.2% |
| **Total** | **8,184** | **100%** |

Of the 45 patients with a complete assessment but no prognosis, 8 have multiple first assessments and 37 have no confirmed execution status.

The 4,923 patients without a complete assessment include 1,451 with saved prognosis. These are included in the overall prognosis count of 4,667.

## Summary and Phase Analysis

| Category | Summary | % | Phase | % |
|---|---:|---:|---:|---:|
| Generated output exists | 3,344 | 40.9% | 3,593 | 43.9% |
| Output missing | 4,840 | 59.1% | 4,591 | 56.1% |
| **Total active patients** | **8,184** | **100%** | **8,184** | **100%** |

### Additional condition breakdown

These rows split the generated and missing totals above; they do not replace them. All percentages use 8,184 active patients.

| Category | Summary | % | Phase | % |
|---|---:|---:|---:|---:|
| Saved-data condition matched; output generated | 1,752 | 21.4% | 1,750 | 21.4% |
| Saved-data condition matched; output missing | 224 | 2.7% | 226 | 2.8% |
| Condition not established from saved data; output generated | 1,592 | 19.5% | 1,843 | 22.5% |
| Condition not established from saved data; output missing | 4,616 | 56.4% | 4,365 | 53.3% |
| **Total active patients** | **8,184** | **100%** | **8,184** | **100%** |

**Condition matched:** At least 5 distinct saved reports with content, or 5 distinct saved VALD profile/document references, or a database-confirmed 15-day fallback with activity. This is a current saved-data check, not a reconstruction of past scheduler decisions.

**Condition not established:** The current records do not establish that rule. This does not mean the patient failed the generation conditions when their output was created. Completed processing can clear activity counters; direct requests and earlier fallback processing can also produce outputs. Individual historical triggers have not been verified.

**How the totals reconcile:** Summary generated is 1,752 + 1,592 = **3,344**. Phase generated is 1,750 + 1,843 = **3,593**. Missing totals are 224 + 4,616 = **4,840** and 226 + 4,365 = **4,591**, respectively.

## VALD Connection

| Category | Patients | % |
|---|---:|---:|
| Valid profile mapping | 6,754 | 82.5% |
| Not mapped | 1,430 | 17.5% |
| **Total** | **8,184** | **100%** |

## VALD Measurements

| Category | Patients | % |
|---|---:|---:|
| Saved measurements exist | 3,863 | 47.2% |
| Mapped; no saved measurements; reason unknown | 2,891 | 35.3% |
| **Total for these two categories** | **6,754** | **82.5%** |

The subtotal covers only the two categories shown. Overall missing-measurement coverage is shown in the first table.

## Yesterday — 15 October 2026, IST

| Service | Patients with first creation recorded | % |
|---|---:|---:|
| Prognosis | 9 | 0.1% |
| Summary | 3 | 0.0% |
| Phase analysis | 3 | 0.0% |

Counts cover 15 October, 00:00 IST to 16 October, 00:00 IST, using the earliest retained creation timestamps for currently active patients. They are not verified lifetime first-ever generations where older records may have been deleted or migrated.

## Pending Validation

| Item | Status |
|---|---|
| VALD freshness | Withheld pending reconciliation with the existing Slack report's population and date fields |
| First VALD measurement sync yesterday | Not available |

Missing data does not automatically indicate a service failure. Percentages may not total 100% due to rounding.
