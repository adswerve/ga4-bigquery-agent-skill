# Unnesting Event Parameters, User Properties & Items

## The core challenge
GA4 stores event_params, user_properties, and items as REPEATED RECORD fields (arrays of structs). You must UNNEST them or use correlated subqueries to access values.

## Pattern 1: Correlated subquery (PREFERRED for extracting single values)
```sql
SELECT
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location') AS page_location,
  (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS session_id
FROM `{project}.{dataset}.events_*`
```
**When to use**: When you need specific parameter values alongside other event fields. This is the most common pattern.

## Pattern 2: Cross join UNNEST (for items)
```sql
SELECT
  items.item_name,
  items.item_id,
  items.price,
  items.quantity
FROM `{project}.{dataset}.events_*`,
  UNNEST(items) AS items
WHERE event_name = 'purchase'
```
**When to use**: When you need to flatten items into individual rows (e.g., for item-level analysis).

## Pattern 3: UNNEST in FROM (for exploring all params)
```sql
SELECT 
  ep.key, ep.value.string_value, ep.value.int_value
FROM `{project}.{dataset}.events_*`, 
  UNNEST(event_params) AS ep
WHERE event_name = 'page_view'
```
**When to use**: When exploring what parameters exist. Less efficient than correlated subquery for targeted extraction.

## Value type selection
event_params values have FOUR possible types — only ONE is populated per parameter:
- `value.string_value` — most common (page_location, source, medium, campaign, session_engaged)
- `value.int_value` — numeric integers (ga_session_id, ga_session_number, entrances, engagement_time_msec)
- `value.float_value` — rarely used
- `value.double_value` — rarely used

### Common parameter keys and their value types
| Key | Value type | Description |
|-----|-----------|-------------|
| ga_session_id | int_value | Session identifier |
| ga_session_number | int_value | Session count for user |
| page_location | string_value | Full page URL |
| page_title | string_value | Page title |
| page_referrer | string_value | Referrer URL |
| source | string_value | Traffic source |
| medium | string_value | Traffic medium |
| campaign | string_value | Campaign name |
| session_engaged | string_value | '1' if engaged |
| engagement_time_msec | int_value | Engagement time in ms |
| entrances | int_value | 1 for landing page |

## User properties
Same structure as event_params. **CRITICAL**: User properties do NOT persist on every event row. When a property is set, it appears only on the event where it was set. To apply it across all user events, you must propagate it:

```sql
SELECT
  MAX(up_region) OVER (PARTITION BY user_pseudo_id) AS user_region,
  event_name,
  user_pseudo_id
FROM (
  SELECT *,
    (SELECT value.string_value FROM UNNEST(user_properties) WHERE key = 'region') AS up_region
  FROM `{project}.{dataset}.events_*`
)
```

## Item parameters (available from 2023-10-25)
Items have their own nested params:
```sql
SELECT
  items.item_name,
  (SELECT value.string_value FROM UNNEST(item_params) WHERE key = 'dimension1') AS custom_dim
FROM `{project}.{dataset}.events_*`,
  UNNEST(items) AS items
```
