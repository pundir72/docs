# Stance Daily Patient Data Report

**Report snapshot:** 8 October 2026, 11:34 AM IST  
**Total active patients:** 7,810  
**Population:** Patient accounts with `isActive = true`, across all centers. All percentages use this population and are rounded to one decimal place.

## 1. Overall coverage

| Service | Have saved data | Missing saved data |
|---|---:|---:|
| Prognosis | 4,578 (58.6%) | 3,232 (41.4%) |
| Summary | 3,317 (42.5%) | 4,493 (57.5%) |
| Phase analysis | 1,704 (21.8%) | 6,106 (78.2%) |
| VALD measurements | 3,789 (48.5%) | 4,021 (51.5%) |

Saved-output coverage does not establish that an output is current or clinically correct. Missing output does not automatically mean generation failed.

## 2. Prognosis

A qualifying first assessment contains nonblank content in all four fields: chief complaint, clinical history, subjective assessment, and provisional diagnosis. This client reporting rule is stricter than the deployed agent's minimum-input rule.

| Assessment coverage | Patients | % of active patients |
|---|---:|---:|
| Qualifying first assessment; prognosis generated | 3,126 | 40.0% |
| Qualifying first assessment; prognosis missing | 46 | 0.6% |
| No qualifying first assessment: absent, empty, or incomplete | 4,638 | 59.4% |
| **Total** | **7,810** | **100.0%** |

Of the 4,638 patients without a qualifying assessment, **1,452 already have saved prognosis**. This is a subset, not an additional category. It explains the difference between the 3,126 qualifying patients with prognosis and the overall 4,578 patients with prognosis.

### Breakdown of all missing prognosis outputs

| Recorded condition | Patients |
|---|---:|
| No first-assessment report | 2,972 |
| First assessment empty or incomplete under the four-field reporting rule | 214 |
| Qualifying assessment exists; multiple first assessments make agent selection ambiguous | 7 |
| Qualifying assessment; prognosis missing; execution status unavailable | 39 |
| **Total missing** | **3,232** |

Assessment incompleteness is an observed condition, not proof of why the agent did not generate. The 7 ambiguous cases plus 39 with unavailable execution status make up the 46 qualifying patients missing prognosis.

## 3. Summary and Phase Analysis

Production startup logs confirm automatic enrollment is configured for seven centers, with report and VALD queue thresholds of 5 and a 15-day fallback. Coverage above includes all active patients, irrespective of center.

| Recorded condition among patients missing output | Summary | Phase analysis |
|---|---:|---:|
| Appointment at a configured center exists; no queue record | 1,804 | 2,431 |
| No appointment at a configured center and no queue record | 2,379 | 2,389 |
| Pending; thresholds not reached and fallback not due | 310 | 1,286 |
| **Total missing** | **4,493** | **6,106** |

The thresholds count distinct references accumulated in the queue, not five lifetime attended sessions. The fallback requires recorded activity. Summary and Phase share queue state, so shared processing or failure status cannot establish an individual service outcome.

No queue record does not establish the historical cause of missing enrollment. The seven-center expansion does not automatically backfill previously excluded patients.

## 4. One-view / VALD

### Mapping and measurement coverage

| Category | Patients |
|---|---:|
| Valid unambiguous One-view sync mapping | 6,638 |
| Not mapped | 1,172 |
| **Total active patients** | **7,810** |

Mapping and measurement coverage are separate measures: a mapped profile does not prove measurements have synced.

| Recorded condition among patients without measurements | Patients |
|---|---:|
| Mapped; no saved measurements; cause unknown | 2,849 |
| No One-view sync record or mapping found | 1,113 |
| One-view profile mapping invalid | 3 |
| One-view profile mapping missing | 56 |
| **Total without measurements** | **4,021** |

Absence of saved measurements does not prove that testing never occurred. Patient-specific sync evidence is needed to establish a failure cause.

### Measurement freshness relative to latest clinical report update

| Category | Patients | % of active patients |
|---|---:|---:|
| VALD measurement newer than report update | 422 | 5.4% |
| 0–15 days older | 1,732 | 22.2% |
| More than 15–30 days older | 403 | 5.2% |
| More than 30–60 days older | 496 | 6.4% |
| More than 60 days older | 590 | 7.6% |
| Measurements exist; supported measurement date unavailable | 25 | 0.3% |
| Measurements exist; clinical report update date unavailable | 121 | 1.5% |
| No saved measurements | 4,021 | 51.5% |
| **Total** | **7,810** | **100.0%** |

There are **3,643 patients with comparable dates** and **146 with measurements but unavailable comparison dates**. This compares measurement dates with report update timestamps; it is not sync latency or age relative to today. Rounded row percentages may not add to exactly 100%.

### Cross-day force changes

| Category | Patients | % of active patients |
|---|---:|---:|
| Increases only, or increases with unchanged values | 248 | 3.2% |
| Decreases only, or decreases with unchanged values | 24 | 0.3% |
| Mixed increases and decreases | 457 | 5.9% |
| Measurements exist; no supported cross-day force comparison | 3,060 | 39.2% |
| No saved measurements | 4,021 | 51.5% |
| **Total** | **7,810** | **100.0%** |

**729 patients** have supported cross-day comparisons. Comparisons use positive average/maximum force in newtons, matching product, exercise, movement, metric, and side across different IST dates. Differing same-day attempts, nonpositive readings, and unsupported metrics are excluded from these comparisons, not necessarily from measurement coverage.

These are numeric changes, **not confirmed clinical improvement**. Different dates do not establish identical testing protocols or separate clinical visits. Clinician confirmation is required before making an improvement claim.

## 5. Yesterday's recorded first creations

**Window:** 7 October 2026, 00:00 IST up to, but not including, 8 October 2026, 00:00 IST.

| Service | Unique currently active patients with first creation recorded yesterday |
|---|---:|
| Prognosis | 14 |
| Summary | 4 |
| Phase analysis | 0 |
| First VALD measurement sync | Not available: not implemented or verified |

These counts use the earliest retained output creation timestamp, rather than the latest update timestamp. Summary combines both summary collections and counts each patient once. No unusable stored creation timestamps were reported for the three AI services.

**Limitation:** These are recorded first creations, not certified lifetime first-ever generations. Deleted/recreated or migrated historical outputs cannot be ruled out. A zero Phase count does not prove there were no regenerations or processing attempts yesterday. Mapping creation time cannot establish first VALD measurement sync time.

## 6. Scope and evidence

This document reproduces the supplied report snapshot; no new database run was performed to prepare it. Coverage, missing-output breakdowns, and exclusive measurement categories reconcile to their stated totals. Unknown execution and sync causes remain explicitly unknown.

The center configuration is based on the supplied production startup logs from 7 October 2026. AI output freshness and clinical correctness are not assessed by this report. No patient identifiers or individual clinical narratives are included.
