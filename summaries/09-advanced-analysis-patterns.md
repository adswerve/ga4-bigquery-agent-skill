# Advanced Analysis Patterns: Funnels, Cohorts, Paths & Attribution

## 1. Checkout abandonment & recovery analysis
Identifies users who started checkout but didn't purchase, then measures how long until they return.
- Use `begin_checkout` and `purchase` events
- Group by user + session to create session-level flags (has_begin_checkout, has_purchase)
- Use `ROW_NUMBER()` to find first abandonment per user
- Join with subsequent purchase sessions
- Calculate `DATE_DIFF(purchase_date, abandon_date, DAY)` for recovery time
- **Tip**: `QUALIFY ROW_NUMBER() OVER (...) = 1` saves an extra CTE

## 2. Days-to-key-action by traffic source
Measures average days from first touch to add_to_cart and purchase, segmented by traffic source.
- CTE 1: first_touch_users — `MIN(event_timestamp)` per user + `traffic_source.source/medium`
- CTE 2: first_add_to_cart — `MIN(event_timestamp)` for `event_name = 'add_to_cart'` per user
- CTE 3: first_purchase — `MIN(event_timestamp)` for `event_name = 'purchase'` per user
- INNER JOIN all three; calculate `TIMESTAMP_DIFF(..., DAY)` and `STDDEV()`
- `HAVING COUNT(DISTINCT user) >= 10` for statistical significance
- Alternative: Use `user_first_touch_timestamp` if full historical data is available

## 3. Market basket analysis (product pair co-occurrence)
See ecommerce summary (07). Self-join on transaction_id with `a.item_name < b.item_name` to avoid duplicate pairs.

## 4. Multi-item purchase patterns (supporting products)
Identifies products that frequently appear as companions in multi-item checkouts:
- Filter `begin_checkout` events with `COUNT(DISTINCT item_name) >= 2`
- `ARRAY_AGG(DISTINCT item_name)` to collect items per checkout
- UNNEST and count frequency of each item across multi-item checkouts

## 5. Cohort revenue analysis (month-by-month)
Tracks revenue over time for user cohorts defined by acquisition month:
- CTE: first_touch — `MIN(event_timestamp)` per user = cohort_day
- CTE: purchase_events — all purchases with revenue
- CTE: cohort_revenue — join and calculate `months_after_acquisition`
- Final: GROUP BY cohort_month + months_after_acquisition, SUM revenue, COUNT users

## 6. First-touch product analysis
Identifies which first-viewed products create the most valuable customers:
- `ARRAY_AGG(item_name ORDER BY event_timestamp ASC)[OFFSET(0)]` for first viewed product per user
- Join with all purchase transactions
- Check if first-viewed product appears in subsequent purchases using `IN UNNEST(items_array)`
- Calculate conversion rates per first-viewed product

## 7. User path / journey analysis (STRING_AGG)
Maps complete page-by-page user journeys to identify common conversion paths:
```sql
STRING_AGG(
  CONCAT(CAST(step_number AS STRING), '. ', page_location), 
  ' > ' ORDER BY step_number
) AS full_path
```
- Use `ROW_NUMBER()` to sequence pageviews within sessions
- Filter for sessions containing target events (e.g., purchase)
- Group paths and count frequency
- **Tip**: Deduplicate consecutive same-page views for cleaner paths

## 8. Attribution modeling (rule-based)
BigQuery enables custom attribution models on GA4 data:

### Base: Build session table with 30-day lookback
For each conversion, get all sessions within 30 days prior, with interaction numbers.

### Models:
- **Last-touch**: `WHERE interaction_number_desc = 1` — full credit to last session
- **First-touch**: `WHERE interaction_number = 1` — full credit to first session
- **Linear**: `1 / total_interactions` per session — equal credit
- **Position-based (40-20-40)**: 40% first, 40% last, 20% split among middle
- **Time decay (7-day halving)**: `POW(0.5, minutes_before_conversion / (7*24*60))`

### Lookback window considerations
- Static 30 days is common
- Consider dynamic windows: max time between interactions, seasonal, product-specific

## 9. Pivot tables
```sql
SELECT * FROM prep
PIVOT (
  COUNT(event_name)
  FOR event_name IN ('session_start', 'page_view', 'scroll', 'purchase')
)
```
- `PIVOT()` only accepts static values in `IN()` — no dynamic subqueries
- Useful for: events by page, funnel metrics by product, revenue by week/source

## Useful SQL functions for GA4 analysis
- `SAFE_DIVIDE(a, b)` — returns NULL instead of error on division by zero
- `SAFE.parse_date()` — returns NULL on bad data instead of error
- `MAX_BY(column, ordering_column)` — get value from row with max of another column (replaces ROW_NUMBER pattern)
- `QUALIFY` — filter on window function results without extra CTE
- `STRING_AGG(value, separator ORDER BY ...)` — concatenate unknown number of values
- `ARRAY_AGG(value ORDER BY ... ASC)[OFFSET(0)]` — get first value in ordered group
- Named windows: `WINDOW w AS (PARTITION BY user_pseudo_id ORDER BY event_timestamp)` to avoid repeating window definitions
