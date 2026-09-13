# Synthea Source Profiling Summary

## Scope

This document summarizes the initial, read-only profiling of the raw Synthea CSV files. It describes the source data as received and identifies issues that should be addressed before graph transformation.

This is not a graph schema. The findings establish source grain, identifiers, foreign-key-like references, temporal behavior, and data-quality risks.

## Source inventory

| Dataset | Rows | Columns | Explicit source ID | Exact duplicate rows |
|---|---:|---:|---|---:|
| `patients.csv` | 108 | 28 | `Id` | 0 |
| `encounters.csv` | 5,571 | 15 | `Id` | 0 |
| `conditions.csv` | 3,517 | 7 | None | 0 |
| `medications.csv` | 3,850 | 13 | None | 0 |
| `procedures.csv` | 15,884 | 10 | None | 0 |
| `observations.csv` | 68,648 | 9 | None | 18 |
| `careplans.csv` | 349 | 9 | `Id` | 0 |

## Identifiers and row identity

- `patients.Id` is complete and unique across all 108 patient rows.
- `encounters.Id` is complete and unique across all 5,571 encounter rows.
- `careplans.Id` is complete and unique across all 349 care-plan rows.
- Conditions, medications, procedures, and observations do not contain explicit event identifiers.
- No composite event key should be accepted until its uniqueness and business meaning are tested.
- File row position is useful for source lineage within one extract but is not necessarily stable across regenerated exports.
- A full-row hash alone cannot distinguish two genuinely separate but identical source events.

## Null and missing-value patterns

### Patients

Important optional fields contain blanks, including:

- `DEATHDATE`: 99 rows
- `DRIVERS`: 19 rows
- `PASSPORT`: 28 rows
- `PREFIX`: 22 rows
- `MIDDLE`: 23 rows
- `SUFFIX`: 107 rows
- `MAIDEN`: 88 rows
- `MARITAL`: 39 rows
- `FIPS`: 29 rows

These blanks are not automatically data-quality defects. Many represent attributes that are optional or not applicable.

### Clinical records

- Conditions without `STOP`: 889 rows (25.3%).
- Medications without `STOP`: 268 rows (7.0%).
- Care plans without `STOP`: 150 rows (43.0%).
- Observations without `ENCOUNTER`: 3,060 rows (4.5%).
- Observations without `UNITS`: 18,874 rows (27.5%). This is expected for many text or categorical values.
- Encounter reason code and description are both blank in 2,044 rows (36.7%). The encounter still exists but has no stated clinical reason.

A missing stop date means that no stop was supplied in the extract. It should not automatically be interpreted as proof that an event remains active.

## Referential integrity

The following checks were performed:

1. Every `encounters.PATIENT` was checked against `patients.Id`.
2. Every clinical record's `PATIENT` was checked against `patients.Id`.
3. Every nonblank clinical `ENCOUNTER` was checked against `encounters.Id`.
4. For each resolved encounter, the clinical record's patient was compared with the patient assigned to that encounter.

### Results

| Dataset | Missing patient | Orphan patient | Missing encounter | Orphan encounter | Encounter/patient mismatch |
|---|---:|---:|---:|---:|---:|
| Encounters | 0 | 0 | Not applicable | Not applicable | Not applicable |
| Conditions | 0 | 0 | 0 | 0 | 0 |
| Medications | 0 | 0 | 0 | 0 | 0 |
| Procedures | 0 | 0 | 0 | 0 | 0 |
| Observations | 0 | 0 | 3,060 | 0 | 0 |
| Care plans | 0 | 0 | 0 | 0 | 0 |

The explicit references are trustworthy in this extract.

## Encounterless observations

The 3,060 observations with blank `ENCOUNTER` are not observations for patients who lack encounters. All 108 patients have encounter records, with at least two encounters per patient.

The encounterless observations belong to 103 patients and consist exclusively of:

| Code | Description | Rows |
|---|---|---:|
| `QALY` | QALY | 1,020 |
| `DALY` | DALY | 1,020 |
| `QOLS` | QOLS | 1,020 |

These appear to be periodic patient-level measures rather than encounter-level measurements. The source therefore supports a mandatory patient reference and an optional encounter reference for observations.

## Duplicate records

`observations.csv` contains 18 exact duplicate rows, representing nine duplicated pairs. The duplicates include laboratory and vital-sign measurements with identical patient, encounter, timestamp, code, description, value, units, category, and type.

Possible explanations include duplicate export output, overlapping Synthea modules, or genuinely separate measurements that cannot be distinguished using the available columns.

These rows should be reported as duplicate candidates rather than silently deleted. Any later deduplication policy should preserve source lineage and document its rule version.

## Code and description consistency

Most code-to-description mappings are consistent. Three codes have description capitalization differences:

- Medication code `1000126`: `medroxyPROGESTERone` versus `medroxyprogesterone`.
- Medication code `856987`: `HYDROcodone` versus `Hydrocodone`.
- Observation code `6299-2`: `Urea Nitrogen` versus `Urea nitrogen`.

These appear to be label variations rather than distinct concepts. Codes should be used as concept identifiers only together with their code system. Original descriptions should remain available for provenance.

## Observation value semantics

Observation values are heterogeneous:

- Numeric observations: 42,531
- Text observations: 26,117

Observation categories include laboratory, survey, vital signs, social history, examination, imaging, procedure, and therapy. Encounterless QALY, DALY, and QOLS rows have a blank category.

Several observation codes appear with more than one type or unit. Examples include:

- Urinalysis results represented as text/nominal or numeric/presence.
- Estimated glomerular filtration rate using `mL/min/{1.73_m2}` and `mL/min`.
- Red-cell distribution width using `fL` and `%`.
- High-sensitivity troponin using `pg/mL` and `ng/L`.
- Housing status with blank or nominal units.

An observation code alone does not establish comparable value semantics. Profiling and later normalization must retain the original type and unit.

## Temporal quality

All populated date and timestamp values parsed successfully. No dataset contained a populated `START` later than its populated `STOP`.

### Temporal precision

- Conditions and care plans use date-only values.
- Encounters, medications, procedures, and observations use UTC timestamps.

Parsing a date-only condition or care-plan start as midnight makes it appear earlier than an encounter occurring later that day. That is a precision difference, not reliable event ordering. Original precision should be preserved.

### Event and encounter alignment

Some event timestamps fall outside their referenced encounter intervals:

- 65 medication starts occur after encounter stop.
- 3,394 procedures start after encounter stop.
- 2,543 encounter-linked observations occur after encounter stop.

Many occur later on the same calendar day, while some occur on different dates. This suggests that `ENCOUNTER` can represent episode association rather than strict containment within the encounter's start and stop timestamps.

These relationships should not be rejected solely because an event timestamp is outside the encounter interval. A useful profiling classification is:

- `WITHIN_INTERVAL`
- `SAME_DAY_OUTSIDE_INTERVAL`
- `OUTSIDE_INTERVAL_DIFFERENT_DAY`
- `DATE_PRECISION_ONLY`

### Patient lifespan

No profiled event begins before its patient's birth date.

Seven encounters occur after the recorded death date. All seven are `Death Certification` encounters. Corresponding cause-of-death observations are administrative records rather than evidence of clinical treatment after death.

One patient also has encounterless QALY, DALY, and QOLS observations generated months after death. Those records should be reported separately for investigation.

## Patient coverage

| Dataset | Patients represented | Minimum rows per represented patient | Median | Maximum |
|---|---:|---:|---:|---:|
| Encounters | 108 | 2 | 36 | 565 |
| Conditions | 108 | 1 | 26 | 273 |
| Medications | 105 | 1 | 11 | 1,113 |
| Procedures | 107 | 4 | 110 | 1,421 |
| Observations | 108 | 36 | 275 | 12,118 |
| Care plans | 101 | 1 | 3 | 10 |

Absence from a clinical dataset should mean that no corresponding source rows were supplied. It should not automatically be converted into a clinical assertion that the patient never had that treatment or condition.

## Key conclusions before graph transformation

The following decisions should guide staging and normalization before any graph nodes or relationships are created:

1. **Keep raw data unchanged.** All cleaning and standardization should produce new outputs. The original CSV values must remain available for comparison, troubleshooting, and reproducibility.

2. **Preserve source lineage.** Each staged record should identify its source file, source row, and ingestion run. This is especially important for event tables without IDs and for records that appear more than once.

3. **Define stable event identifiers.** Conditions, medications, procedures, and observations do not provide explicit event IDs. They require a documented deterministic identifier strategy that remains stable across reruns without accidentally merging identical source rows.

4. **Do not silently remove duplicates.** Exact duplicate observations should first be retained and marked as duplicate candidates. Any later deduplication rule must preserve lineage and explain which records were kept or excluded.

5. **Allow valid observations without encounters.** QALY, DALY, and QOLS observations are linked to patients but not to encounters. Observation-to-patient association is required, while observation-to-encounter association is optional.

6. **Validate patient and encounter agreement.** When a clinical row contains both identifiers, its patient must match the patient assigned to the referenced encounter. A mismatch must not produce a graph relationship until it is investigated.

7. **Preserve original and normalized values.** Standardized descriptions, units, or codes may be added, but they should not overwrite the values supplied by the source. This allows normalized results to remain traceable.

8. **Identify concepts using code and system, not description alone.** Descriptions can vary in capitalization or wording. Code-system values inferred from dataset knowledge must be distinguished from systems explicitly supplied in the CSV.

9. **Preserve temporal precision.** A date-only value must remain distinguishable from a full timestamp. The pipeline should not invent midnight or imply an event ordering that the source does not provide.

10. **Separate encounter association from temporal containment.** An event can explicitly reference an encounter even when its timestamp falls outside the encounter's start and stop interval. Preserve the relationship and classify the temporal difference separately rather than changing or rejecting the source record automatically.

11. **Treat missing stop dates as unknown.** A blank stop date does not prove that a condition, medication, or care plan is currently active. It only means that the source did not provide an end date.

12. **Keep observation value semantics together.** Observation code, value, datatype, and unit must remain connected. Measurements using different units or value types should not be treated as directly comparable without a documented conversion rule.

13. **Distinguish post-death administration from clinical activity.** Death-certification encounters and cause-of-death observations may validly occur after the recorded death date. Other post-death activity should remain visible and be flagged for investigation.

14. **Report unsafe records instead of silently dropping them.** Validation findings should identify the source row, failed rule, severity, and processing decision. Records can then be accepted, accepted with a warning, quarantined, or rejected using explicit rules.


