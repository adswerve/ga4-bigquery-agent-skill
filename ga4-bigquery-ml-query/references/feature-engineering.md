# Feature Engineering Patterns

Common patterns for BigQuery ML datasets: time-windowed aggregations, cumulative metrics, recency, categorical encodings, ratios, and GA4-specific patterns (site content, traffic source, geo, event params).

## Time-Windowed Aggregations

Break lookback into slices to capture behavioral trends (increasing/decreasing activity):

```sql
SUM(IF(DATE_DIFF(d, date, DAY) BETWEEN 0 AND 28, revenue, 0)) as spend_last_28d,
SUM(IF(DATE_DIFF(d, date, DAY) BETWEEN 29 AND 70, revenue, 0)) as spend_29d_to_70d,
SUM(IF(DATE_DIFF(d, date, DAY) BETWEEN 71 AND 140, revenue, 0)) as spend_71d_to_140d,
SUM(IF(DATE_DIFF(d, date, DAY) > 140, revenue, 0)) as spend_over_140d_ago,
```

Best practices: `IFNULL(..., 0)` for empty windows, non-overlapping windows, descriptive names `{metric}_{window}`.

---

## Cumulative (Running) Metrics

Running totals up to each observation point via window functions:

```sql
COUNT(*) OVER (PARTITION BY user_id ORDER BY session_start ASC) as sessions_to_date,
SUM(pageviews) OVER (PARTITION BY user_id ORDER BY session_start ASC) as ttl_pageviews,
SUM(add_to_carts) OVER (PARTITION BY user_id ORDER BY session_start ASC) as ttl_add_to_carts,
SUM(purchases) OVER (PARTITION BY user_id ORDER BY session_start ASC) as ttl_purchases,
```

`ORDER BY session_start ASC` creates running sums — each row sees a different cumulative total.

---

## Recency Features

Often among the most predictive features.

```sql
-- Hours since last event (session-level, using window function)
IFNULL(
  TIMESTAMP_DIFF(
    session_end,
    MAX(IF(purchases > 0, session_end, NULL)) OVER (PARTITION BY user_id ORDER BY session_start ASC),
    HOUR
  ),
  LOOKBACK_DAYS * 24 * 2  -- sentinel when no prior event
) as hours_since_last_purchase,

-- Days since last event (snapshot-level)
IFNULL(DATE_DIFF(d, MAX(purchase_date), DAY), LOOKBACK_DAYS + 1) as days_since_last_purchase,
```

**Multiple recency points** — track hours_since for any high-value action:

```sql
IFNULL(TIMESTAMP_DIFF(session_end, MAX(IF(checkout_starts>0, session_end, NULL)) OVER (...), HOUR), sentinel) as hours_since_checkout,
IFNULL(TIMESTAMP_DIFF(session_end, MAX(IF(key_action_events>0, session_end, NULL)) OVER (...), HOUR), sentinel) as hours_since_key_action,
```

### N Most Recent Transactions

```sql
ARRAY_AGG(STRUCT(DATE_DIFF(d, date, DAY) as days_ago, revenue as amount)
  IGNORE NULLS ORDER BY date DESC LIMIT 5) as prev_orders
-- Unpack: IFNULL(prev_orders[SAFE_OFFSET(0)].days_ago, LOOKBACK_DAYS + 1) as order_1_days_ago
```

---

## Categorical & Proportion Features

```sql
-- Category counts
COUNTIF(category = 'Electronics') as count_electronics,
-- Revenue by category
SUM(IF(department = 'Socks', revenue, 0)) as revenue_socks,
-- Proportions (SAFE_DIVIDE always)
IFNULL(SAFE_DIVIDE(SUM(revenue_socks), SUM(total_revenue)), 0) as pct_spend_socks,
```

---

## Ratio & Intensity Features

```sql
SAFE_DIVIDE(SUM(revenue_net), SUM(revenue_gross)) as avg_discount_ratio,
-- Spend intensity
SAFE_DIVIDE(SUM(daily_revenue), COUNT(DISTINCT active_date)) as avg_daily_spend,
-- Session frequency
COUNT(*) OVER (...) / LOOKBACK_DAYS as sessions_per_day,
-- Engagement per page
SAFE_DIVIDE(time_on_site, pageviews) as avg_time_per_page,
-- Composite scores
(pageviews + user_engagements + scrolls) as engagement_score,
```

---

## Temporal Context Features

```sql
EXTRACT(DAYOFYEAR FROM snapshot_date) as day_of_year,
EXTRACT(MONTH FROM snapshot_date) - 1 as month_of_snapshot,
-- Seasonal proportion
IFNULL(SAFE_DIVIDE(
  SUM(IF(EXTRACT(MONTH FROM date) IN (11, 12), revenue, 0)), SUM(revenue)
), 0) as pct_spend_gifting_season,
```

---

## GA4-Specific: Site Content Features (page_location)

Count pageviews for specific URL patterns to signal product/content interest:

```sql
-- LIKE pattern (simple substring match)
COUNTIF(event_name = 'page_view' AND
  LOWER((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location'))
  LIKE '%auto-delivery%') AS auto_delivery_pageviews,

-- REGEXP pattern (query parameter extraction)
COUNTIF(event_name = 'page_view' AND
  REGEXP_CONTAINS(
    LOWER((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location')),
    r'[?&]vertical=child_care([%&]|$)')
) AS vertical_child_care_page_views,

-- Inclusion OR / exclusion AND NOT pattern
COUNTIF(event_name = 'page_view' AND
  (LOWER(page_location) LIKE '%category-a%' OR LOWER(page_location) LIKE '%category-a-alt%')
  AND NOT LOWER(page_location) LIKE '%excluded-section%'
) AS category_a_pageviews,
```

These get both session-level values AND cumulative totals via window functions:

```sql
-- Session level (in base_sessions)
COUNTIF(...) AS auto_delivery_pageviews,
-- Cumulative (in features_and_label)
SUM(bs.auto_delivery_pageviews) OVER (PARTITION BY user_pseudo_id ORDER BY session_start ASC) as ttl_auto_delivery_pageviews,
```

---

## GA4-Specific: Traffic Source Encoding

One-hot encode traffic medium from `session_traffic_source_last_click`:

```sql
MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'cpc', 1, 0)) as is_medium_cpc,
MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'organic', 1, 0)) as is_medium_organic,
MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'email', 1, 0)) as is_medium_email,
MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'referral', 1, 0)) as is_medium_referral,
MAX(IF(LOWER(COALESCE(medium, '')) IN ('(none)', '(not set)', ''), 1, 0)) as is_medium_none,
```

Cumulative: use `MAX(...) OVER (... ORDER BY session_start ASC)` to get "ever seen this medium" flags:

```sql
MAX(bs.is_medium_cpc) OVER (PARTITION BY user_pseudo_id ORDER BY session_start ASC) as ever_medium_cpc,
```

---

## GA4-Specific: Geo & Device Encoding

One-hot encode top N regions/countries and device types:

```sql
-- Geo (top regions by volume)
MAX(IF(geo.region = 'California', 1, 0)) as is_region_california,
MAX(IF(geo.region = 'New York', 1, 0)) as is_region_new_york,
MAX(IF(geo.country = 'United States', 1, 0)) as united_states,

-- Device category
MAX(IF(device.category = 'desktop', 1, 0)) as desktop,
MAX(IF(device.category = 'mobile', 1, 0)) as mobile,
MAX(IF(device.category = 'tablet', 1, 0)) as tablet,

-- Platform
MAX(IF(platform = 'WEB', 1, 0)) as is_web,
MAX(IF(platform = 'IOS', 1, 0)) as is_ios,
MAX(IF(platform = 'ANDROID', 1, 0)) as is_android,
```

---

## GA4-Specific: Event Params Extraction

Extract numeric/string values from the `event_params` repeated field:

```sql
-- Integer params (engagement, scroll depth, custom counters)
MAX(IFNULL((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'engaged_session_event' LIMIT 1), 0)) as engaged_session,
MAX(IFNULL((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'percent_scrolled' LIMIT 1), 0)) as max_percent_scrolled,
COALESCE(MAX((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'logged_in_status')), 0) as logged_in_status,

-- String params (count events where a param matches a target value)
COUNTIF((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'param_key' LIMIT 1) = 'target_value') as param_match_events,

-- Conditional conversion (purchase + event_param conditions to narrow the conversion definition)
COUNTIF(event_name = 'purchase' AND
  (SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'subscription_type') = 'auto'
  AND LOWER((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'product_category')) NOT LIKE '%excluded%'
) as conversion,
```

---

## Handling NULLs

| Scenario | Strategy |
|----------|----------|
| Count/sum with no data | `IFNULL(SUM(...), 0)` |
| Ratio with zero denominator | `IFNULL(SAFE_DIVIDE(...), 0)` |
| Recency with no prior event | `IFNULL(..., LOOKBACK_DAYS * 24 * 2)` |
| Boolean flag | `IF(condition, 1, 0)` |

Every feature column must produce a valid numeric value. No NULLs in final output.

---

## Naming Conventions

| Prefix/Suffix | Meaning | Example |
|---------------|---------|---------|
| `ttl_` | Cumulative total | `ttl_pageviews` |
| `pct_` | Proportion | `pct_spend_socks` |
| `_last_Nd` | Last N days | `spend_last_28d` |
| `_Nd_to_Md` | Between N-M days ago | `spend_29d_to_70d` |
| `_hours_ago` / `_days_ago` | Recency | `last_purchase_hours_ago` |
| `is_` / `ever_` | One-hot flag | `is_medium_cpc`, `ever_medium_cpc` |

---

## Pre-Aggregation (Rollup) Pattern

Two-stage approach for complex multi-source datasets:

```sql
-- Stage 1: Daily grain
base_orders AS (
  SELECT user_id, DATE(created_at) as date, COUNT(*) as daily_orders, SUM(revenue) as daily_revenue
  FROM orders INNER JOIN user_pool USING (user_id)
  GROUP BY ALL
),
-- Stage 2: Rollup per snapshot
rollup AS (
  SELECT d as snapshot_date, user_id,
    SUM(daily_orders) as total_orders,
    SUM(IF(DATE_DIFF(d, date, DAY) BETWEEN 0 AND 28, daily_revenue, 0)) as spend_last_28d
  FROM base_orders INNER JOIN UNNEST(snapshot_dates) d ON date BETWEEN d - LOOKBACK_DAYS AND d
  GROUP BY ALL
)
```
