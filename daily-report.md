# Stance Daily Report

Includes patient accounts with `isActive = true`. Percentages are based on all 7,810 active patients.

## Overall Coverage

| Service | Have data | % | Missing data | % |
|---|---:|---:|---:|---:|
| Prognosis | 4,578 | 58.6% | 3,232 | 41.4% |
| Summary | 3,317 | 42.5% | 4,493 | 57.5% |
| Phase analysis | 1,704 | 21.8% | 6,106 | 78.2% |
| VALD measurements | 3,789 | 48.5% | 4,021 | 51.5% |

## Prognosis

A complete first assessment contains chief complaint, clinical history, subjective assessment, and provisional diagnosis.

| Category | Patients | % |
|---|---:|---:|
| Complete assessment; prognosis generated | 3,126 | 40.0% |
| Complete assessment; prognosis missing | 46 | 0.6% |
| Assessment absent, empty, or incomplete | 4,638 | 59.4% |
| **Total** | **7,810** | **100%** |

Of the 46 patients with a complete assessment but no prognosis, 7 have multiple first assessments and 39 have no confirmed execution status.

The 4,638 patients without a complete assessment include 1,452 with saved prognosis. These are included in the overall prognosis count of 4,578.

## Summary and Phase Analysis

Automatic enrollment covers seven configured centers. Scheduling uses 5 queued report references, 5 queued VALD references, or the 15-day fallback with recorded activity.

| Category | Summary | % | Phase | % |
|---|---:|---:|---:|---:|
| Generated data exists | 3,317 | 42.5% | 1,704 | 21.8% |
| Missing; queued and not yet due | 310 | 4.0% | 1,286 | 16.5% |
| Missing; configured-center appointment exists, but no queue record | 1,804 | 23.1% | 2,431 | 31.1% |
| Missing; no configured-center appointment or queue record | 2,379 | 30.5% | 2,389 | 30.6% |
| **Total** | **7,810** | **100%** | **7,810** | **100%** |

For patients without a queue record, generation eligibility and the reason for missing enrollment are not established. Queue thresholds do not represent lifetime attended sessions.

## VALD Connection

| Category | Patients | % |
|---|---:|---:|
| Valid profile mapping | 6,638 | 85.0% |
| Not mapped | 1,172 | 15.0% |
| **Total** | **7,810** | **100%** |

## VALD Measurements

| Category | Patients | % |
|---|---:|---:|
| Saved measurements exist | 3,789 | 48.5% |
| Mapped; no saved measurements; reason unknown | 2,849 | 36.5% |
| No sync record or mapping found | 1,113 | 14.3% |
| Mapping missing or invalid; no saved measurements | 59 | 0.8% |
| **Total** | **7,810** | **100%** |

A profile mapping does not confirm that measurements have synced.

## VALD Force Changes Across Dates

| Category | Patients | % |
|---|---:|---:|
| Increases only, with or without unchanged values | 248 | 3.2% |
| Decreases only, with or without unchanged values | 24 | 0.3% |
| Mixed increases and decreases | 457 | 5.9% |
| Measurements exist; no supported comparison | 3,060 | 39.2% |
| No measurements | 4,021 | 51.5% |
| **Total** | **7,810** | **100%** |

729 patients have supported comparisons of positive average or maximum force in newtons across different IST dates, matching product, exercise, movement, metric, and side. Differing same-day attempts are excluded. These are numeric changes; clinical improvement requires clinician review.

## Yesterday — 7 October 2026, IST

| Service | Patients with first creation recorded | % |
|---|---:|---:|
| Prognosis | 14 | 0.2% |
| Summary | 4 | 0.1% |
| Phase analysis | 0 | 0.0% |

Counts cover 7 October, 00:00 IST to 8 October, 00:00 IST, using the earliest retained creation timestamps for currently active patients. They are not verified lifetime first-ever generations where older records may have been deleted or migrated.

## Pending Validation

| Item | Status |
|---|---|
| VALD freshness | Withheld pending reconciliation with the existing Slack report's population and date fields |
| First VALD measurement sync yesterday | Not available |

Figures reflect the stated snapshot. Missing data does not automatically indicate a service failure. Percentages may not total 100% due to rounding.
