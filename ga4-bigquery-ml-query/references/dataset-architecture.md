# Dataset Architecture

Core architecture for ML datasets in BigQuery: temporal windows, entity pools, snapshot approach, and leakage prevention.

## Temporal Windows

### Lookback Window

Defines how far back to gather features relative to the observation point.

- Too short → misses long-term patterns (seasonal habits)
- Too long → stale data, increased cost
- Business-dependent: subscription service ~365d; fast-moving app ~28d

```sql
DECLARE LOOKBACK_DAYS INT64 DEFAULT 90;
-- Features: only data within lookback relative to observation point
WHERE date BETWEEN snapshot_date - LOOKBACK_DAYS AND snapshot_date
```

### Lookahead Window

Defines how far forward to observe the outcome (label).

- Too short → noisy labels (few conversions)
- Too long → delayed feedback; model becomes stale
- Match business cadence: revenue ~90d; propensity ~7-14d

```sql
DECLARE LOOKAHEAD_DAYS INT64 DEFAULT 7;
-- Label: only future data after observation point
WHERE date BETWEEN snapshot_date + 1 AND snapshot_date + LOOKAHEAD_DAYS
```

### The Date Boundary Rule (Most Common Mistake)

Source data range must account for BOTH windows:

```sql
WHERE date BETWEEN date_start - LOOKBACK_DAYS AND date_end + LOOKAHEAD_DAYS
```

Why: User B near `date_start` needs lookback into `date_start - LOOKBACK_DAYS`. User C near `date_end` needs lookahead into `date_end + LOOKAHEAD_DAYS`. Without extension → truncated windows → systematic bias.

```
  date_start                           date_end
       │                                  │
  ◄─ LOOKBACK ─►│◄── observation range ──►│◄─ LOOKAHEAD ─►
       │                                  │
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ▲                                                      ▲
  Read from: date_start - LOOKBACK      Read to: date_end + LOOKAHEAD
```

Training output: filter `WHERE date <= date_end - LOOKAHEAD_DAYS` (ensures complete lookahead).

---

## Entity Pool

Defines eligible population. Removes noise, ensures data quality.

**Design principles**:
1. Activity within range: `last_activity_date BETWEEN date_start AND date_end`
2. Minimum engagement: `pageviews > 1` or `sessions >= 2`
3. Business eligibility: domain-specific (e.g. prior purchase for LTV)
4. Non-null identity

```sql
user_pool AS (
  SELECT user_id, MAX(activity_date) as last_activity_date
  FROM source_events
  WHERE activity_date BETWEEN date_start - LOOKBACK_DAYS AND date_end
    AND user_id IS NOT NULL
  GROUP BY user_id
  HAVING
    last_activity_date BETWEEN date_start AND date_end
    AND COUNT(DISTINCT session_id) >= 2
)
```

Pool is joined downstream to filter all subsequent CTEs:

```sql
base_data AS (
  SELECT ...
  FROM source_table
  INNER JOIN user_pool USING (user_id)
  WHERE date BETWEEN date_start - LOOKBACK_DAYS AND date_end + LOOKAHEAD_DAYS
)
```

---

## The Snapshot Approach

### Problem with Single Prediction Points

Single cutoff → each entity appears once → wastes years of behavioral data from churned/dormant users. Model never learns from their active phase.

### Multiple Snapshot Dates

"Travel back in time" to each snapshot date. For each: features from `[snapshot - LOOKBACK, snapshot]`, label from `[snapshot + 1, snapshot + LOOKAHEAD]`.

```sql
SET snapshot_dates = ARRAY(
  SELECT d FROM UNNEST(GENERATE_DATE_ARRAY(date_start, date_end, INTERVAL 1 DAY)) d
  WHERE EXTRACT(DAYOFWEEK FROM d) = 2  -- Monday
  ORDER BY d DESC
);
```

Joining to data:

```sql
SELECT d as snapshot_date, user_id,
  SUM(IF(date BETWEEN d - LOOKBACK_DAYS AND d, revenue, 0)) as lookback_revenue,
  SUM(IF(date BETWEEN d + 1 AND d + LOOKAHEAD_DAYS, revenue, 0)) as label
FROM orders
INNER JOIN UNNEST(snapshot_dates) d ON date BETWEEN d - LOOKBACK_DAYS AND d + LOOKAHEAD_DAYS
GROUP BY 1, 2
```

### Correlated Snapshots Mitigation

Multiple snapshots per user → correlated rows, bias toward active users.

```sql
-- Option 1: Random one per user (recommended)
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY user_id ORDER BY FARM_FINGERPRINT(CONCAT(user_id, '/', CAST(date AS STRING)))
) = 1

-- Option 2: Cap at N per user
) <= 5

-- Option 3: Minimum spacing (M days apart)
```

| Scenario | Approach |
|----------|----------|
| User-level (LTV, churn) | Snapshots (weekly) |
| Session-level (propensity) | Each session IS the observation point |
| Small user base (<10K) | Snapshots increase volume significantly |
| Large user base (>1M) | Consider sampling to control cost |

---

## Data Leakage Prevention

### Types

| Type | Example |
|------|---------|
| **Target leakage** | "has_purchased" flag from same future window as label |
| **Feature leakage** | "total_lifetime_revenue" including post-snapshot data |
| **Temporal leakage** | Not filtering `date <= date_end - LOOKAHEAD_DAYS` |
| **Split leakage** | Splitting on session_id instead of user_id |

### Prevention Checklist

1. Features use ONLY `[snapshot - LOOKBACK, snapshot]`
2. Label uses ONLY `[snapshot + 1, snapshot + LOOKAHEAD]`
3. Training output: `WHERE date <= date_end - LOOKAHEAD_DAYS`
4. Split on entity ID: `FARM_FINGERPRINT(user_id)`
5. Exclude identifiers from feature columns in training
6. Join conditions enforce temporal ordering
7. Inference mirrors training feature logic exactly

---

## Base Layer

Pre-aggregates raw data at the working grain (per-session or daily-per-user).

**Principles**: one CTE per source domain, join to pool immediately, include full date range, flatten nested fields.

```sql
base_sessions AS (
  SELECT user_id, session_id,
    MIN(event_timestamp) as session_start,
    MAX(event_timestamp) as session_end,
    COUNTIF(event_name = 'page_view') as pageviews,
    COUNTIF(event_name = 'add_to_cart') as add_to_carts,
    COUNTIF(event_name = 'purchase') as purchases
  FROM events
  INNER JOIN user_pool USING (user_id)
  WHERE date BETWEEN date_start - LOOKBACK_DAYS AND date_end + LOOKAHEAD_DAYS
  GROUP BY 1, 2
)
```

### Label Generation

Separate CTE using ONLY lookahead data:

```sql
-- Binary (propensity): purchase in lookahead window?
conv AS (
  SELECT bs1.session_id, IF(MAX(bs2.purchases) > 0, 1, 0) as label
  FROM base_sessions bs1
  LEFT JOIN base_sessions bs2
    ON bs2.user_id = bs1.user_id
    AND bs2.session_start BETWEEN bs1.session_end
      AND TIMESTAMP_ADD(bs1.session_end, INTERVAL LOOKAHEAD_DAYS DAY)
  GROUP BY 1
)

-- Continuous (LTV): revenue in lookahead window
labels AS (
  SELECT snapshot_date, user_id, SUM(IFNULL(revenue, 0)) as label
  FROM orders
  INNER JOIN UNNEST(snapshot_dates) snapshot_date
    ON order_date BETWEEN snapshot_date + 1 AND snapshot_date + LOOKAHEAD_DAYS
  GROUP BY ALL
)
```
