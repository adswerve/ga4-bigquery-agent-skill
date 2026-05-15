# Training and Inference Modes

Deterministic splits, output contracts, stored procedure pattern, and table materialization.

## Deterministic Split (FARM_FINGERPRINT)

```sql
CASE
  WHEN MOD(ABS(FARM_FINGERPRINT(user_id)), 100) < 70 THEN 'TRAIN'
  WHEN MOD(ABS(FARM_FINGERPRINT(user_id)), 100) < 90 THEN 'EVAL'
  ELSE 'TEST'
END as data_split
```

**Properties**: Deterministic (no random seed), entity-level (all rows for a user land in same split), reproducible across runs.

**Standard ratio**: 70/20/10. Adjust for small datasets (60/20/20) or very large (80/10/10).

**CRITICAL**: Split on entity ID, not row ID:

```sql
FARM_FINGERPRINT(user_pseudo_id)   -- CORRECT
FARM_FINGERPRINT(session_id)        -- WRONG: same user in train AND test → leakage
```

---

## Training Output Contract

| Requirement | Details |
|-------------|---------|
| `data_split` | STRING: 'TRAIN', 'EVAL', 'TEST' |
| `label` | Target (INT64 classification, FLOAT64 regression) |
| Features only | Exclude identifiers, timestamps, dates |
| No NULLs | All features must be non-NULL numeric |
| Full lookahead | `WHERE date <= date_end - LOOKAHEAD_DAYS` |

```sql
EXECUTE IMMEDIATE FORMAT("""
  CREATE OR REPLACE TABLE %s
  OPTIONS(
    expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY),
    labels = [("vai-mlops", "training")]
  ) AS
  SELECT
    CASE
      WHEN MOD(ABS(FARM_FINGERPRINT(user_pseudo_id)), 100) < 70 THEN 'TRAIN'
      WHEN MOD(ABS(FARM_FINGERPRINT(user_pseudo_id)), 100) < 90 THEN 'EVAL'
      ELSE 'TEST'
    END as data_split,
    * EXCEPT (user_pseudo_id, session_id, date, session_start_tstamp, session_end_tstamp)
  FROM dataset
  WHERE date <= @de;
""", table_name)
USING date_end - LOOKAHEAD_DAYS as de;
```

Identifiers excluded because: model memorizes users if included. Encode temporal signals as features (day_of_year, recency) instead of raw dates.

---

## Inference Output Contract

| Requirement | Details |
|-------------|---------|
| Entity ID(s) | MUST include (maps predictions back to users) |
| `date` column | DATE, for partitioning/versioning |
| Features | Identical to training (same names, types, logic) |
| No label | Excluded (this is what we predict) |
| No data_split | Not needed |

```sql
EXECUTE IMMEDIATE FORMAT("""
  CREATE OR REPLACE TABLE %s
  OPTIONS(
    expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 1 DAY),
    labels = [("vai-mlops", "inference")]
  ) AS
  SELECT * EXCEPT (label)
  FROM dataset
  WHERE date BETWEEN @ds AND @de;
""", table_name)
USING date_start as ds, date_end as de;
```

---

## Stored Procedure Pattern (Recommended)

Single code path for training + inference, parameterized, schedulable, auditable.

```sql
CREATE OR REPLACE PROCEDURE `project.dataset.create_ml_dataset`(
  table_name STRING, date_start DATE, date_end DATE, mode STRING
)
BEGIN
  DECLARE LOOKBACK_DAYS INT64 DEFAULT 90;
  DECLARE LOOKAHEAD_DAYS INT64 DEFAULT 7;

  CREATE OR REPLACE TEMP TABLE dataset AS ( ... );

  IF mode = 'TRAINING' THEN
    EXECUTE IMMEDIATE FORMAT("""
      CREATE OR REPLACE TABLE %s OPTIONS(...) AS
      SELECT CASE ... END as data_split, * EXCEPT(identifiers)
      FROM dataset WHERE date <= @de;
    """, table_name) USING date_end - LOOKAHEAD_DAYS as de;
  END IF;

  IF mode = 'INFERENCE' THEN
    EXECUTE IMMEDIATE FORMAT("""
      CREATE OR REPLACE TABLE %s OPTIONS(...) AS
      SELECT * EXCEPT(label) FROM dataset WHERE date BETWEEN @ds AND @de;
    """, table_name) USING date_start as ds, date_end as de;
  END IF;

  DROP TABLE dataset;
END;
```

### Calling

```sql
CALL `project.dataset.create_ml_dataset`('project.dataset.training_v1', '2023-01-01', '2024-12-31', 'TRAINING');
CALL `project.dataset.create_ml_dataset`('project.dataset.inference', CURRENT_DATE()-90, CURRENT_DATE(), 'INFERENCE');
```

---

## Standalone Alternative (No Procedure)

```sql
DECLARE LOOKBACK_DAYS INT64 DEFAULT 90;
DECLARE LOOKAHEAD_DAYS INT64 DEFAULT 7;
DECLARE date_start DATE DEFAULT '2023-01-01';
DECLARE date_end DATE DEFAULT '2024-12-31';

CREATE OR REPLACE TABLE `project.dataset.training_table` AS
WITH user_pool AS (...), base_data AS (...), labels AS (...), features AS (...)
SELECT
  CASE WHEN MOD(ABS(FARM_FINGERPRINT(user_id)), 100) < 70 THEN 'TRAIN' ... END as data_split,
  features.* EXCEPT (user_id, date), labels.label
FROM features LEFT JOIN labels USING (user_id, snapshot_date)
WHERE features.date <= date_end - LOOKAHEAD_DAYS;
```

---

## Table Expiration & Labels

```sql
OPTIONS(
  expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY),  -- training
  -- or INTERVAL 1 DAY for inference (regenerated frequently)
  labels = [("vai-mlops", "training"), ("model", "propensity-v2")]
)
```

---

## Validation Checklist

- [ ] Row count ~70/20/10 across splits
- [ ] Label distribution similar across TRAIN/EVAL/TEST
- [ ] No NULLs in feature columns
- [ ] Training rows: `date <= date_end - LOOKAHEAD_DAYS`
- [ ] Same user always in same split: `SELECT user_id, COUNT(DISTINCT data_split) ... HAVING count > 1` = 0 rows
- [ ] Feature parity: training and inference produce identical columns
