# Traffic Sources: The Four Scopes of Attribution

## CRITICAL: GA4 has FOUR different traffic source locations
This is the most confusing aspect of GA4 BigQuery export. Each serves a different purpose.

---

## 1. `traffic_source` — User-scoped (first-touch acquisition)
**Scope**: User (set ONCE per user_pseudo_id, never changes)
**Available since**: Start of GA4 export (2019)
**GA4 UI equivalent**: "First user source/medium/campaign"
**Fields**: traffic_source.source, traffic_source.medium, traffic_source.name

```sql
SELECT
  traffic_source.source,
  traffic_source.medium,
  traffic_source.name AS campaign,
  COUNT(DISTINCT user_pseudo_id) AS users
FROM `{project}.{dataset}.events_*`
WHERE _table_suffix BETWEEN '{start}' AND '{end}'
GROUP BY 1, 2, 3
```

---

## 2. `session_traffic_source_last_click` — Session-scoped (last non-direct click)
**Scope**: Session (same value on ALL events in a session)
**Available since**: July 2024 (manual_campaign, google_ads), October 2024 (cross_channel)
**GA4 UI equivalent**: "Session source/medium/campaign"
**Rules**:
- Takes first event-based traffic info from the session
- Follows **last non-direct attribution**: direct visits inherit most recent non-direct source if one exists

**Sub-records**:
- `.manual_campaign` — UTM-based: source, medium, campaign_id, campaign_name, term, content, source_platform, creative_format, marketing_tactic
- `.google_ads_campaign` — Google Ads: customer_id, account_name, campaign_id, campaign_name, ad_group_id, ad_group_name
- `.cross_channel_campaign` — Default/primary channel data

```sql
SELECT
  session_traffic_source_last_click.manual_campaign.source,
  session_traffic_source_last_click.manual_campaign.medium,
  session_traffic_source_last_click.manual_campaign.campaign_name
FROM `{project}.{dataset}.events_*`
WHERE _table_suffix BETWEEN '{start}' AND '{end}'
```

---

## 3. `collected_traffic_source` — Event-scoped (raw, unprocessed)
**Scope**: Event (can have different values per event within a session)
**Available since**: May 2023 (but only on session_start since Nov 2023 for some properties)
**GA4 UI equivalent**: Closest to raw source/medium/campaign
**Key characteristic**: Values are RAW — no session scoping, no last-non-direct attribution applied

**Fields**: manual_campaign_id, manual_campaign_name, manual_source, manual_medium, manual_term, manual_content, gclid, dclid, srsltid, manual_source_platform (from July 2024), manual_creative_format (from July 2024), manual_marketing_tactic (from July 2024)

```sql
SELECT
  collected_traffic_source.manual_source,
  collected_traffic_source.manual_medium,
  collected_traffic_source.manual_campaign_name,
  collected_traffic_source.gclid
FROM `{project}.{dataset}.events_*`
WHERE _table_suffix BETWEEN '{start}' AND '{end}'
```

---

## 4. Event parameters (source, medium, campaign, gclid)
**Scope**: Event (same data as collected_traffic_source, harder to access, potentially costlier)
**Available since**: Start of GA4 export (2019)
**Use case**: Legacy approach, or when collected_traffic_source is not available

```sql
SELECT
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'source') AS source,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'medium') AS medium,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'campaign') AS campaign
FROM `{project}.{dataset}.events_*`
```

---

## Which to use?
| Need | Use | Why |
|------|-----|-----|
| User acquisition (first-touch) | `traffic_source` | User-scoped, shows how user was first acquired |
| Session attribution (standard reports) | `session_traffic_source_last_click` | Matches GA4 UI session reports |
| Raw event-level traffic data | `collected_traffic_source` | No attribution applied, raw values |
| Legacy / pre-2023 data | event_params | Only option for old data |

## Session-scoped traffic (calculated from event params)
Before `session_traffic_source_last_click` was available, session source/medium had to be calculated:

```sql
WITH prep AS (
  SELECT
    user_pseudo_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS session_id,
    ARRAY_AGG((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'source') 
      IGNORE NULLS ORDER BY event_timestamp)[SAFE_OFFSET(0)] AS source,
    ARRAY_AGG((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'medium') 
      IGNORE NULLS ORDER BY event_timestamp)[SAFE_OFFSET(0)] AS medium
  FROM `{project}.{dataset}.events_*`
  WHERE _table_suffix BETWEEN '{start}' AND '{end}'
  GROUP BY user_pseudo_id, session_id
)
SELECT
  COALESCE(source, '(direct)') AS session_source,
  COALESCE(medium, '(none)') AS session_medium,
  COUNT(DISTINCT CONCAT(user_pseudo_id, session_id)) AS sessions
FROM prep
GROUP BY 1, 2
ORDER BY sessions DESC
```

## Known bugs/issues
- **GA4 misattribution bug**: Paid search events/sessions can be misattributed as organic or direct traffic when gclid is present. See ga4bigquery.com for fix tutorial.
- **collected_traffic_source NULL bug**: Since summer 2023, many properties have most events losing collected_traffic_source values (becoming NULL). Only page_view and session_start events may have it populated.

## Default channel grouping
Must be calculated manually in BigQuery using CASE WHEN logic based on source/medium patterns. See the full channel grouping CASE statement in the traffic sources reference file.
