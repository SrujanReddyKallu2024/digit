# Signoff Stats - Complete Documentation

## Overview

The Signoff Stats module validates the Digital Taxonomy output before delivery to clients. It:
1. Computes statistics for every column
2. Compares with previous month's stats
3. Flags significant changes
4. Blocks output if validation fails
5. Sends email notifications

---

## Function Call Chain

```
main_emr.py
    └── validate_output()                    ← Entry point
            ├── compute_column_stats()       ← Compute stats for ALL columns
            ├── get_previous_month()         ← Calculate previous month (202601 → 202512)
            ├── load_previous_stats()        ← Load from S3
            ├── compare_stats()              ← Calculate diffs
            ├── add_change_flags()           ← Add breach flags
            ├── validate_thresholds()        ← Check thresholds, create reasons
            └── save_stats()                 ← Save to S3 (only if passed)
```

---

## Thresholds

| Threshold | Value | Meaning |
|-----------|-------|---------|
| `pct_cutoff` | 20.0% | Flag if null count % change > 20% |
| `abs_count_cutoff` | 50 | Flag if null count change > 50 |
| `abs_pct_point_cutoff` | 5.0 | Flag if null percentage points change > 5 |
| `max_row_count_change` | 5.0% | Flag if total row count change > 5% |

---

## Complete Example

### Input Data: 5 Rows, 4 Columns

**Current Month (January 2026)**:
```
cb_key_db_person|C000063|C000064|p_age_coarse
ABC001|Y|Y|25-34
ABC002|Y|N|35-44
ABC003||Y|45-54
ABC004|Y|Y|
ABC005|N|Y|55-64
```

**Previous Month (December 2025)**:
```
cb_key_db_person|C000063|C000064|p_age_coarse
ABC001|Y|Y|25-34
ABC002|Y|N|35-44
ABC003|Y|Y|45-54
ABC004|Y|Y|55-64
ABC005|N|Y|65+
```

---

## Step 1: `compute_column_stats()`

Counts stats for each column:

| Column | Nulls | Non-Null | Null % |
|--------|-------|----------|--------|
| cb_key_db_person | 0 | 5 | 0% |
| C000063 | 1 | 4 | 20% |
| C000064 | 0 | 5 | 0% |
| p_age_coarse | 1 | 4 | 20% |

### Output: `current_stats`

```json
{
  "total_rows": 5,
  "computed_at": "2026-01-22 09:00:00",
  "columns": {
    "cb_key_db_person": {
      "distinct_values": 5,
      "min_value": "ABC001",
      "max_value": "ABC005",
      "data_type": "string",
      "non_null_values": 5,
      "null_count": 0,
      "null_percentage": 0.0,
      "min_length": 6,
      "max_length": 6
    },
    "C000063": {
      "distinct_values": 2,
      "min_value": "N",
      "max_value": "Y",
      "data_type": "string",
      "non_null_values": 4,
      "null_count": 1,
      "null_percentage": 20.0,
      "min_length": 1,
      "max_length": 1
    },
    "C000064": {
      "distinct_values": 2,
      "min_value": "N",
      "max_value": "Y",
      "data_type": "string",
      "non_null_values": 5,
      "null_count": 0,
      "null_percentage": 0.0,
      "min_length": 1,
      "max_length": 1
    },
    "p_age_coarse": {
      "distinct_values": 4,
      "min_value": "25-34",
      "max_value": "55-64",
      "data_type": "string",
      "non_null_values": 4,
      "null_count": 1,
      "null_percentage": 20.0,
      "min_length": 5,
      "max_length": 5
    }
  }
}
```

**Saved to**: `s3://stats-bucket/taxonomy_signoff/202601/output_stats.json` (only if validation passes)

---

## Step 2: `load_previous_stats()`

Loads from: `s3://stats-bucket/taxonomy_signoff/202512/output_stats.json`

```json
{
  "total_rows": 5,
  "computed_at": "2025-12-15 09:00:00",
  "columns": {
    "cb_key_db_person": {
      "distinct_values": 5,
      "non_null_values": 5,
      "null_count": 0,
      "null_percentage": 0.0
    },
    "C000063": {
      "distinct_values": 2,
      "non_null_values": 5,
      "null_count": 0,
      "null_percentage": 0.0
    },
    "C000064": {
      "distinct_values": 2,
      "non_null_values": 5,
      "null_count": 0,
      "null_percentage": 0.0
    },
    "p_age_coarse": {
      "distinct_values": 5,
      "non_null_values": 5,
      "null_count": 0,
      "null_percentage": 0.0
    }
  }
}
```

---

## Step 3: `compare_stats()`

Calculates diff for every column:

| Column | Dec Nulls | Jan Nulls | Diff | % Diff |
|--------|-----------|-----------|------|--------|
| cb_key_db_person | 0 | 0 | 0 | 0 |
| **C000063** | 0 | 1 | **+1** | **+20** |
| C000064 | 0 | 0 | 0 | 0 |
| **p_age_coarse** | 0 | 1 | **+1** | **+20** |

### Output: `comparison`

```json
{
  "row_count": {
    "previous": 5,
    "current": 5,
    "diff": 0,
    "pct_change": 0.0
  },
  "columns": {
    "cb_key_db_person": {
      "status": "compared",
      "new_field": false,
      "old_field": false,
      "total_change": 0.0,
      "prev_null_values": 0,
      "curr_null_values": 0,
      "diff_null_values": 0,
      "pct_change_null_values": 0.0,
      "prev_non_null_values": 5,
      "curr_non_null_values": 5,
      "diff_non_null_values": 0,
      "prev_distinct_values": 5,
      "curr_distinct_values": 5,
      "diff_distinct_values": 0,
      "pct_change_distinct_values": 0.0,
      "null_pct": {
        "previous": 0.0,
        "current": 0.0,
        "diff": 0.0
      }
    },
    "C000063": {
      "status": "compared",
      "new_field": false,
      "old_field": false,
      "total_change": 0.0,
      "prev_null_values": 0,
      "curr_null_values": 1,
      "diff_null_values": 1,
      "pct_change_null_values": 100.0,
      "prev_non_null_values": 5,
      "curr_non_null_values": 4,
      "diff_non_null_values": -1,
      "prev_distinct_values": 2,
      "curr_distinct_values": 2,
      "diff_distinct_values": 0,
      "pct_change_distinct_values": 0.0,
      "null_pct": {
        "previous": 0.0,
        "current": 20.0,
        "diff": 20.0
      }
    },
    "C000064": {
      "status": "compared",
      "prev_null_values": 0,
      "curr_null_values": 0,
      "diff_null_values": 0,
      "null_pct": {
        "previous": 0.0,
        "current": 0.0,
        "diff": 0.0
      }
    },
    "p_age_coarse": {
      "status": "compared",
      "prev_null_values": 0,
      "curr_null_values": 1,
      "diff_null_values": 1,
      "pct_change_null_values": 100.0,
      "null_pct": {
        "previous": 0.0,
        "current": 20.0,
        "diff": 20.0
      }
    }
  }
}
```

---

## Step 4: `validate_thresholds()`

Checks each column against thresholds:

| Column | Check | Result |
|--------|-------|--------|
| cb_key_db_person | Critical (0% nulls required) | ✅ PASS |
| C000063 | prev=0, curr=1 → zero_to_nonzero | ❌ FAIL |
| C000064 | No change | ✅ PASS |
| p_age_coarse | prev=0, curr=1 → zero_to_nonzero | ❌ FAIL |

### Output: `reasons` list

```python
reasons = [
    "🔴 C000063: Nulls went from 0 to 1",
    "🔴 p_age_coarse: Nulls went from 0 to 1"
]
```

### Output: `ValidationResult`

```python
ValidationResult(
    passed = False,
    reasons = [
        "🔴 C000063: Nulls went from 0 to 1",
        "🔴 p_age_coarse: Nulls went from 0 to 1"
    ],
    current_stats = {...},
    previous_stats = {...},
    comparison = {...}
)
```

---

## Step 5: `save_validation_result()`

Saves to: `s3://stats-bucket/taxonomy_signoff/202601/validation_result.json`

```json
{
  "passed": false,
  "reasons": [
    "🔴 C000063: Nulls went from 0 to 1",
    "🔴 p_age_coarse: Nulls went from 0 to 1"
  ],
  "total_rows": 5,
  "computed_at": "2026-01-22 09:00:00",
  "month": "202601",
  "email": {
    "subject": "❌ Digital Taxonomy 202601 - VALIDATION FAILED",
    "body": "⚠️ Digital Taxonomy for 202601 has FAILED validation...",
    "passed": false
  },
  "row_count_comparison": {
    "previous": 5,
    "current": 5,
    "diff": 0,
    "pct_change": 0.0
  },
  "flagged_columns": [
    {
      "column_name": "C000063",
      "prev_null_values": 0,
      "curr_null_values": 1,
      "diff_null_values": 1,
      "pct_change_null_values": 100.0,
      "null_pct": {"previous": 0.0, "current": 20.0, "diff": 20.0}
    },
    {
      "column_name": "p_age_coarse",
      "prev_null_values": 0,
      "curr_null_values": 1,
      "diff_null_values": 1,
      "pct_change_null_values": 100.0,
      "null_pct": {"previous": 0.0, "current": 20.0, "diff": 20.0}
    }
  ]
}
```

---

## Step 6: Exception Raised

```python
raise Exception(
    "Signoff validation failed with 2 issue(s). "
    "File NOT delivered. Reasons: "
    "🔴 C000063: Nulls went from 0 to 1; "
    "🔴 p_age_coarse: Nulls went from 0 to 1"
)
```

**OUTPUT FILE IS NOT WRITTEN!**

---

## Step 7: Email Sent

The DAG reads `validation_result.json` and sends:

```
Subject: ❌ Digital Taxonomy 202601 - VALIDATION FAILED

*** Digital Taxonomy Processing FAILED ***

Processing Month: 2026-01
Failed Tasks: run_main_processing

⚠️ Signoff Validation Failed
❌ FILE NOT DELIVERED TO CLIENT
Total Rows: 5

Validation Issues (2):
• 🔴 C000063: Nulls went from 0 to 1
• 🔴 p_age_coarse: Nulls went from 0 to 1

Flagged Columns Detail:
┌───────────────┬────────────┬────────────┬────────┬──────────┐
│ Column        │ Prev Nulls │ Curr Nulls │ Change │ % Change │
├───────────────┼────────────┼────────────┼────────┼──────────┤
│ C000063       │ 0          │ 1          │ +1     │ +100.0%  │
│ p_age_coarse  │ 0          │ 1          │ +1     │ +100.0%  │
└───────────────┴────────────┴────────────┴────────┴──────────┘
```

---

## Files Saved Summary

| File | Path | When Saved |
|------|------|------------|
| `output_stats.json` | `s3://.../202601/output_stats.json` | Only if PASSED |
| `validation_result.json` | `s3://.../202601/validation_result.json` | Always |
| Output .txt | `s3://.../digital_taxonomy_202601.txt` | Only if PASSED |

---

## Threshold Breach Conditions

| Condition | Description | Example |
|-----------|-------------|---------|
| `zero_to_nonzero` | prev=0, curr>0 | 0 → 1 null |
| `drop_to_zero` | prev>0, curr=0 | 100 → 0 nulls |
| `abs_count_exceeds` | abs(diff) >= 50 | +100 nulls |
| `relative_pct_exceeds` | abs(% change) >= 20% | +25% change |
| `abs_pct_points_exceeds` | abs(% points diff) >= 5 | 10% → 20% |

---

## Critical Columns

These columns MUST have 0% nulls:
- `cb_key_db_person` (primary key)

If these have any nulls, validation fails immediately.
