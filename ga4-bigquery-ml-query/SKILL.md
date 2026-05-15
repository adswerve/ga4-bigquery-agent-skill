---
name: ga4-bigquery-ml-query
description: >
  Build ML-ready datasets in BigQuery using SQL. Covers dataset architecture
  (lookback/lookahead windows, user pools, snapshot dates), feature engineering
  patterns (cumulative metrics, recency, site content, traffic/geo encoding),
  leakage prevention, deterministic splits, training vs inference modes.
  GA4-specific examples for propensity and predictive LTV.
---

# BigQuery ML Dataset Creation Skill

You are an expert at building ML-ready datasets in BigQuery using SQL. You correctly structure lookback/lookahead windows, prevent data leakage, create deterministic splits, and produce datasets that feed directly into training or inference pipelines.

## Required Context

Before writing a dataset query, confirm:
1. **ML objective** — what is predicted (purchase probability, future revenue, churn)
2. **Observation grain** — entity + moment (per-session, per-user per-week)
3. **Label definition** — positive outcome + future window duration
4. **Source tables** — BQ project, dataset, table names
5. **Window sizes** — lookback and lookahead durations

If unclear, ask before writing.

## Core Architecture

```
┌──────────────────────────────────────────────────┐
│  1. PARAMETERS   (windows, dates, mode)          │
│  2. ENTITY POOL  (eligible population filter)    │
│  3. BASE LAYER   (pre-aggregated raw data)       │
│  4. LABELS       (target from lookahead window)  │
│  5. FEATURES     (join + engineer features)      │
│  6. OUTPUT       (split + materialize)           │
└──────────────────────────────────────────────────┘
```

→ [references/dataset-architecture.md](./references/dataset-architecture.md)

## Lookback & Lookahead Windows

| Window | Purpose |
|--------|---------|
| **Lookback** | Gather features from historical behavior before observation point |
| **Lookahead** | Observe the outcome after observation point |

**CRITICAL**: Source data range must extend BOTH directions:

```sql
WHERE event_date BETWEEN date_start - LOOKBACK_DAYS AND date_end + LOOKAHEAD_DAYS
-- NOT just date_start to date_end (truncates edge users → systematic bias)
```

→ [references/dataset-architecture.md](./references/dataset-architecture.md)

## Entity Pool

Filters eligible entities — removes noise, ensures data quality:

```sql
user_pool AS (
  SELECT user_id, MAX(activity_date) as last_activity_date
  FROM source_table
  WHERE activity_date BETWEEN date_start - LOOKBACK_DAYS AND date_end
  GROUP BY user_id
  HAVING
    last_activity_date BETWEEN date_start AND date_end
    AND COUNT(*) > threshold
)
```

Typical criteria: minimum sessions/events, activity within range, exclude bots, business filters (e.g. prior purchase for LTV).

## Snapshot Approach (Multiple Observation Points)

For user-level predictions, use multiple snapshot dates (e.g. every Monday) instead of a single cutoff — captures every lifecycle phase:

```sql
SET snapshot_dates = ARRAY(
  SELECT d FROM UNNEST(GENERATE_DATE_ARRAY(date_start, date_end, INTERVAL 1 DAY)) d
  WHERE EXTRACT(DAYOFWEEK FROM d) = 2
);
```

**Correlated rows mitigation**: Sample one per user (`QUALIFY ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY FARM_FINGERPRINT(user_id || '/' || date)) = 1`) or cap at N snapshots.

→ [references/dataset-architecture.md](./references/dataset-architecture.md)

## Data Leakage Prevention

| Rule | Implementation |
|------|----------------|
| Features: lookback only | `WHERE date BETWEEN snapshot - LOOKBACK AND snapshot` |
| Label: lookahead only | `WHERE date BETWEEN snapshot + 1 AND snapshot + LOOKAHEAD` |
| Training: full lookahead | `WHERE date <= date_end - LOOKAHEAD_DAYS` |
| Inference: no label | `SELECT * EXCEPT (label)` |
| Split on entity, not row | `FARM_FINGERPRINT(user_id)` not `FARM_FINGERPRINT(session_id)` |

→ [references/dataset-architecture.md](./references/dataset-architecture.md)

## Feature Engineering Quick Reference

| Pattern | SQL |
|---------|-----|
| Cumulative metric | `SUM(x) OVER (PARTITION BY user ORDER BY session_start)` |
| Recency (hours) | `IFNULL(TIMESTAMP_DIFF(now, MAX(IF(event>0, ts, NULL)) OVER (...), HOUR), sentinel)` |
| Time-windowed agg | `SUM(IF(DATE_DIFF(d, date, DAY) BETWEEN 0 AND 28, x, 0))` |
| Site content feature | `COUNTIF(event_name='page_view' AND LOWER(page_location) LIKE '%pattern%')` |
| Geo/traffic one-hot | `MAX(IF(geo.region = 'California', 1, 0))` |
| Event params extraction | `MAX(IFNULL((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key='key'), 0))` |
| Ratio/intensity | `SAFE_DIVIDE(SUM(x), SUM(y))` |
| Derived score | `(pageviews + engagements + scrolls) as engagement_score` |

→ [references/feature-engineering.md](./references/feature-engineering.md)

## Deterministic Split

```sql
CASE
  WHEN MOD(ABS(FARM_FINGERPRINT(user_id)), 100) < 70 THEN 'TRAIN'
  WHEN MOD(ABS(FARM_FINGERPRINT(user_id)), 100) < 90 THEN 'EVAL'
  ELSE 'TEST'
END as data_split
```

Deterministic, entity-level, reproducible, no random seed.

→ [references/training-and-inference.md](./references/training-and-inference.md)

## Training vs Inference

| Aspect | Training | Inference |
|--------|----------|-----------|
| Label | Included | Excluded |
| data_split | Included | Not needed |
| Identifiers | Excluded from features | Included |
| Snapshots | Multiple (weekly) | Single (latest) |
| Expiration | 7-90 days | 1 day |

→ [references/training-and-inference.md](./references/training-and-inference.md)

## Observation Grain

- **Session-level** (one row per session): Real-time propensity, next-action. Each session is the observation point.
- **User-level snapshots** (one row per user per date): LTV, churn, periodic scoring.

Both follow: pool → base → label → features → output.

## GA4-Specific Patterns

- Entity: `user_pseudo_id` (or `user_id` if set)
- Session key: `user_pseudo_id || '/' || event_date || '/' || ga_session_id`
- Cost control: `_TABLE_SUFFIX` filtering
- Labels: purchase in next N days, premium subscription, revenue
- Site content features via `page_location` LIKE/REGEXP
- Traffic source via `session_traffic_source_last_click.cross_channel_campaign.medium`
- Geo/device/platform one-hot encoding
- Event params: `engaged_session_event`, `percent_scrolled`, custom params
- External enrichment: holiday proximity, user lookup tables

→ [references/ga4-examples.md](./references/ga4-examples.md)

## Key Warnings

1. **Extend source read range**: `date_start - LOOKBACK_DAYS` to `date_end + LOOKAHEAD_DAYS`
2. **Split on entity ID**, never on rows
3. **Filter training to full lookahead**: `WHERE date <= date_end - LOOKAHEAD_DAYS`
4. **SAFE_DIVIDE** for ratios, **IFNULL** for missing values
5. **Exclude identifiers/timestamps** from training features
6. **Correlated snapshots**: sample or cap per user
7. **Conditional labels**: define business-specific conversion precisely (e.g. purchase WHERE program='AUTO' AND product NOT LIKE '%accessory%')
