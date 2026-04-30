# BigQuery vs GA4 UI: Known Discrepancies & Gotchas

## Expect differences — this is normal
Google's own developer advocate confirms: "the standard reporting surfaces and the BigQuery export data aren't expected to be reconcilable" — they serve different use cases.

## Why numbers differ

### 1. User definitions
- GA4 UI defaults to **active users** (engagement threshold), BigQuery gives you raw `user_pseudo_id`
- Must manually calculate active users in BQ: `COUNT(DISTINCT CASE WHEN is_active_user IS TRUE THEN user_pseudo_id END)`

### 2. Session estimation
- GA4 UI uses **HyperLogLog++ (HLL++) approximation** for session counts, not exact counts
- BigQuery gives exact `COUNT(DISTINCT)` — will differ slightly

### 3. Scopes mixing
- Mixing user-scope, session-scope, and event-scope data incorrectly produces meaningless results
- BigQuery won't tell you if your results are wrong as long as SQL is valid

### 4. Google Signals
- Data exported to BigQuery may show MORE users compared to GA4 reports that use Google Signals data

### 5. Sampling & cardinality
- GA4 UI can sample data or apply cardinality limits; BigQuery export is unsampled

### 6. Data modeling (consent mode)
- GA4 UI applies behavioral modeling for consented/unconsented users
- BigQuery export contains raw data — no modeling applied

### 7. Late-arriving events
- Daily tables update for up to 3 days after the event date
- Running queries on "yesterday" may not have all data yet

## Consent mode impact on BigQuery data

### Basic consent mode
- `privacy_info.analytics_storage` = 'Yes' only (no data for non-consented users)
- Clean but potentially biased dataset

### Advanced consent mode
- Contains "cookieless pings" where `analytics_storage` = 'No'
- These events lack `user_pseudo_id` and `ga_session_id`
- Each cookieless page view triggers a new `session_start` (inflates session counts)
- **Strategy**: Filter with `WHERE privacy_info.analytics_storage = 'Yes'` for clean data (introduces bias)
- **Alternative**: Focus on event metrics/trends rather than user/session counts

### Check your consent mode implementation
```sql
SELECT
  privacy_info.analytics_storage,
  privacy_info.ads_storage,
  COUNT(*) AS events_total,
  COUNT(DISTINCT user_pseudo_id) AS users,
  COUNT(DISTINCT CONCAT(user_pseudo_id, (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id'))) AS sessions
FROM `{project}.{dataset}.events_YYYYMMDD`
GROUP BY 1, 2
```

## User data export tables (pseudonymous_users / users)
- Contains audience membership, prediction scores (purchase_score_7d, churn_score_7d, revenue_28d_in_usd)
- Updated daily, no historical backfill
- Will differ from event-based calculations (heavily processed by Google)
- Useful for: audience data, prediction scores — data that can't be derived from events

## GA4 Dataform project
Google's official `ga4_dataform` project provides SQL models to transform raw GA4 export into friendly tables:
- Builds unique `user_key` and `ga_session_key`
- Creates session, user_transaction_daily, and event output tables
- Includes helper functions for unnesting, URL extraction, channel grouping
- Provides event-level last-click attribution
- **Caveat**: Not officially supported by Google, device-based only (not user_id), web properties only
- Last updated ~2 years ago — use as reference/inspiration rather than production dependency
