# Stance Daily Report

## Prognosis

Complete assessment means all four fields contain data: chief complaint, clinical history, subjective assessment, and provisional diagnosis.

| Category | Patients | % |
|---|---:|---:|
| Complete first assessment; prognosis generated | 3,126 | 40.0% |
| Complete first assessment; prognosis missing | 46 | 0.6% |
| First assessment absent, empty, or incomplete | 4,638 | 59.4% |
| **Total** | **7,810** | **100%** |

Of the 46 missing, **7** have multiple first assessments; execution status is unknown for **39**. Overall, **4,578 patients have prognosis**, including 1,452 without a currently complete assessment. Those 1,452 are already included in the third row.

## Summary and Phase Analysis

Automatic generation uses the configured 5-report / 5-VALD-reference thresholds and 15-day fallback, with seven configured centers. These are queued references, not lifetime attended sessions.

| Category | Summary | % | Phase | % |
|---|---:|---:|---:|---:|
| Generated data exists | 3,317 | 42.5% | 1,704 | 21.8% |
| Missing; pending and not yet due | 310 | 4.0% | 1,286 | 16.5% |
| Missing; configured-center appointment exists, but no queue record—eligibility unknown | 1,804 | 23.1% | 2,431 | 31.1% |
| Missing; no configured-center appointment or queue record | 2,379 | 30.5% | 2,389 | 30.6% |
| **Total** | **7,810** | **100%** | **7,810** | **100%** |

**Total missing:** Summary **4,493 (57.5%)**; Phase **6,106 (78.2%)**. No queue record does not prove a generation failure or establish whether thresholds were met.

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
| Mapped; measurements missing—reason unknown | 2,849 | 36.5% |
| No sync record or mapping found | 1,113 | 14.3% |
| Mapping missing or invalid; measurements missing | 59 | 0.8% |
| **Total** | **7,810** | **100%** |

**Total without measurements: 4,021 (51.5%).** Mapping does not prove measurements have synced.

## VALD Freshness Compared with Latest Clinical Report Update

| Category | Patients | % |
|---|---:|---:|
| VALD newer than report | 422 | 5.4% |
| 0–15 days older | 1,732 | 22.2% |
| More than 15–30 days older | 403 | 5.2% |
| More than 30–60 days older | 496 | 6.4% |
| More than 60 days older | 590 | 7.6% |
| Measurements exist; comparison dates unavailable | 146 | 1.9% |
| No measurements | 4,021 | 51.5% |
| **Total** | **7,810** | **100%** |

This compares measurement dates with report update dates, not sync time or age relative to today.

## Patient Force-Measurement Changes

| Category | Patients | % |
|---|---:|---:|
| Increases only, possibly with unchanged values | 248 | 3.2% |
| Decreases only, possibly with unchanged values | 24 | 0.3% |
| Mixed increases and decreases | 457 | 5.9% |
| Measurements exist; no supported cross-day comparison | 3,060 | 39.2% |
| No measurements | 4,021 | 51.5% |
| **Total** | **7,810** | **100%** |

**729 patients** have supported comparisons of positive average/maximum force in newtons across different IST dates, matching exercise, movement, metric and side. Ambiguous same-day attempts are excluded. These are numeric changes; **clinical improvement requires clinician interpretation**.

## Yesterday — 7 October 2026, IST

| Service | First creations recorded | % |
|---|---:|---:|
| Prognosis | 14 | 0.2% |
| Summary | 4 | 0.1% |
| Phase analysis | 0 | 0.0% |

