# Stance Daily Report

**9 October 2026, 4:25 PM IST**  
**Active patients: 7,863**

Includes patient accounts with `isActive = true`. All percentages use 7,863 active patients. Figures reflect the supplied 16:25:33 IST snapshot.

## Overall Coverage

| Service | Have data | % | Missing data | % |
|---|---:|---:|---:|---:|
| Prognosis | 4,589 | 58.4% | 3,274 | 41.6% |
| Summary | 3,323 | 42.3% | 4,540 | 57.7% |
| Phase analysis | 3,555 | 45.2% | 4,308 | 54.8% |
| VALD measurements | 3,793 | 48.2% | 4,070 | 51.8% |

Phase coverage uses the production collection, `new-patient-phases`.

## Prognosis

A complete first assessment contains chief complaint, clinical history, subjective assessment, and provisional diagnosis.

| Category | Patients | % |
|---|---:|---:|
| Complete assessment; prognosis generated | 3,137 | 39.9% |
| Complete assessment; prognosis missing | 46 | 0.6% |
| Assessment absent, empty, or incomplete | 4,680 | 59.5% |
| **Total** | **7,863** | **100%** |

Of the 46 patients with a complete assessment but no prognosis, 7 have multiple first assessments and 39 have no confirmed execution status.

The 4,680 patients without a complete assessment include 1,452 with saved prognosis. These are included in the overall prognosis count of 4,589.

## Summary and Phase Analysis

Coverage across all 7,863 active patients:

| Category | Summary | % | Phase | % |
|---|---:|---:|---:|---:|
| Generated data exists | 3,323 | 42.3% | 3,555 | 45.2% |
| Generated data missing | 4,540 | 57.7% | 4,308 | 54.8% |
| **Total** | **7,863** | **100%** | **7,863** | **100%** |

The requested condition-met breakdown for the full population is not yet established. The available historical evidence below covers only 25 patients and must not be used as the complete eligibility report.

### Additional eligibility evidence — 25 patients only

| Category | Summary | % | Phase | % |
|---|---:|---:|---:|---:|
| Generation condition met; output generated | 22 | 0.3% | 22 | 0.3% |
| Generation condition met; output missing | 3 | 0.0% | 3 | 0.0% |

**Partial history: these are confirmed minimum counts, not totals for all historically eligible patients.** Qualification is based on retained queue and dispatch evidence for the report/VALD thresholds or fallback. Saved output is checked separately and may predate the qualifying event. Three patients represent 0.038% of active patients, displayed as 0.0% after rounding.

## VALD Connection

| Category | Patients | % |
|---|---:|---:|
| Valid profile mapping | 6,656 | 84.6% |
| Not mapped | 1,207 | 15.4% |
| **Total** | **7,863** | **100%** |

## VALD Measurements

| Category | Patients | % |
|---|---:|---:|
| Saved measurements exist | 3,793 | 48.2% |
| Mapped; no saved measurements; reason unknown | 2,863 | 36.4% |
| **Total for these two categories** | **6,656** | **84.6%** |

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
| Full Summary/Phase historical eligibility | Available evidence supports only the minimum counts shown above |
| VALD freshness | Withheld pending reconciliation with the existing Slack report's population and date fields |
| First VALD measurement sync yesterday | Not available |

Missing data does not automatically indicate a service failure. Percentages may not total 100% due to rounding.
