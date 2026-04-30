# Users & Sessions: Definitions, Metrics, and Calculation Patterns

## User concepts in GA4
GA4 has THREE user concepts — always clarify which one you mean:

1. **Total users** = `COUNT(DISTINCT user_pseudo_id)` — all users regardless of engagement
2. **Active users** = users meeting engagement threshold. Two approaches:
   - Modern (post July 2023): `COUNT(DISTINCT CASE WHEN is_active_user IS TRUE THEN user_pseudo_id END)`
   - Legacy: `COUNT(DISTINCT CASE WHEN engagement_time_msec > 0 OR session_engaged = '1' THEN user_pseudo_id END)`
3. **Identified users** = users with `user_id` set via setUserId API

**Default preference**: Use total users (`user_pseudo_id`) unless engagement insight is specifically needed.

## Session identification
A session is uniquely identified by: `CONCAT(user_pseudo_id, ga_session_id)`

```sql
-- Unique session identifier
CONCAT(user_pseudo_id,
  (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id')
) AS session_key
```

## Core user metrics

### New users
```sql
COUNT(DISTINCT CASE 
  WHEN (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_number') = 1 
  THEN user_pseudo_id END) AS new_users
```

### % new users
```sql
COUNT(DISTINCT CASE WHEN session_number = 1 THEN user_pseudo_id END) 
  / COUNT(DISTINCT user_pseudo_id) AS pct_new_users
```

### User type (new vs returning)
```sql
CASE
  WHEN (SELECT value.int_value FROM UNNEST(event_params) 
    WHERE event_name = 'session_start' AND key = 'ga_session_number') = 1 THEN 'new visitor'
  WHEN (...) > 1 THEN 'returning visitor'
END AS user_type
```

## Core session metrics

### Sessions
```sql
COUNT(DISTINCT CONCAT(user_pseudo_id,
  (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id')
)) AS sessions
```

### Engaged sessions
```sql
COUNT(DISTINCT CASE 
  WHEN (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'session_engaged') = '1' 
  THEN CONCAT(user_pseudo_id, session_id) END) AS engaged_sessions
```

### Engagement rate
```sql
SAFE_DIVIDE(engaged_sessions, sessions) AS engagement_rate
```

### Bounce rate (inverse of engagement rate in GA4)
```sql
-- Bounces = total sessions - engaged sessions
SAFE_DIVIDE(sessions - engaged_sessions, sessions) AS bounce_rate
```

### Average session duration
```sql
WITH prep AS (
  SELECT
    CONCAT(user_pseudo_id, (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id')) AS session_id,
    (MAX(event_timestamp) - MIN(event_timestamp)) / 1000000 AS session_seconds
  FROM `{project}.{dataset}.events_*`
  WHERE _table_suffix BETWEEN '{start}' AND '{end}'
  GROUP BY session_id
)
SELECT SUM(session_seconds) / COUNT(DISTINCT session_id) AS avg_session_duration
FROM prep
```

### Engagement time (average per engaged session)
```sql
WITH prep AS (
  SELECT
    user_pseudo_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS session_id,
    MAX((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'session_engaged')) AS session_engaged,
    SUM((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'engagement_time_msec')) / 1000 AS engagement_seconds
  FROM `{project}.{dataset}.events_*`
  WHERE _table_suffix BETWEEN '{start}' AND '{end}'
  GROUP BY user_pseudo_id, session_id
)
SELECT SAFE_DIVIDE(SUM(engagement_seconds), 
  COUNT(DISTINCT CASE WHEN session_engaged = '1' THEN CONCAT(user_pseudo_id, session_id) END)) AS avg_engagement_time
FROM prep
```

### Sessions per user
```sql
COUNT(DISTINCT CONCAT(user_pseudo_id, session_id)) / COUNT(DISTINCT user_pseudo_id)
```

### Views per session
```sql
COUNTIF(event_name = 'page_view') / COUNT(DISTINCT CONCAT(user_pseudo_id, session_id))
```
