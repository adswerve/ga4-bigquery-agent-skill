# GA4 BigQuery ML Dataset Examples

Concrete GA4-specific examples for ML datasets from BigQuery export. Covers purchase propensity (session-level) and predictive LTV (user-level with snapshots).

## GA4 Fundamentals

### Identifiers

| Identifier | Use Case |
|------------|----------|
| `user_pseudo_id` | Default visitor (cookie-based) |
| `user_id` | Authenticated user (more reliable, not always available) |
| Session key | `user_pseudo_id \|\| '/' \|\| event_date \|\| '/' \|\| ga_session_id` |

### Table Access

```sql
FROM `project.analytics_PROPERTY_ID.events_*`
WHERE SAFE.PARSE_DATE('%Y%m%d', _TABLE_SUFFIX)
  BETWEEN date_start - LOOKBACK_DAYS AND date_end + LOOKAHEAD_DAYS
```

### Common Events

| Event | Signal |
|-------|--------|
| `page_view` | Browsing depth |
| `view_item` / `select_item` | Product interest |
| `add_to_cart` | Purchase intent |
| `begin_checkout` | High intent |
| `purchase` | Conversion |
| `view_promotion` / `select_promotion` | Marketing engagement |
| `view_search_results` | Active searching |
| `scroll` / `user_engagement` | Content engagement |
| `video_start` / `video_complete` | Video engagement |
| `login` / `first_visit` | Identity/lifecycle signals |

---

## Example 1: Purchase Propensity (Session-Level)

**Objective**: Predict whether a visitor will purchase/convert within `LOOKAHEAD_DAYS` after their current session.

**Grain**: One row per session. **Label**: Binary (purchase occurred in lookahead window).

```sql
CREATE OR REPLACE PROCEDURE `project.dataset.create_propensity_dataset`(
  table_name STRING, date_start DATE, date_end DATE, mode STRING
)
BEGIN
  DECLARE LOOKBACK_DAYS INT64 DEFAULT 56;
  DECLARE LOOKAHEAD_DAYS INT64 DEFAULT 21;

  CREATE OR REPLACE TEMP TABLE dataset AS (
    WITH
      -- 1. VISITOR POOL
      visitor_pool AS (
        SELECT
          user_pseudo_id,
          MAX(event_date) as last_event_date,
          MAX(event_tstamp) as last_event_tstamp,
          COUNTIF(event_name = 'page_view'
            AND event_tstamp BETWEEN TIMESTAMP_SUB(last_event_tstamp, INTERVAL LOOKBACK_DAYS DAY) AND last_event_tstamp
          ) as pageviews
        FROM (
          SELECT user_pseudo_id, event_name,
            SAFE.PARSE_DATE('%Y%m%d', _TABLE_SUFFIX) as event_date,
            TIMESTAMP_MICROS(event_timestamp) as event_tstamp,
            MAX(TIMESTAMP_MICROS(event_timestamp)) OVER (PARTITION BY user_pseudo_id) as last_event_tstamp
          FROM `project.dataset.events_*`
          WHERE SAFE.PARSE_DATE('%Y%m%d', _TABLE_SUFFIX) BETWEEN date_start - LOOKBACK_DAYS AND date_end
        )
        GROUP BY user_pseudo_id
        HAVING last_event_date BETWEEN date_start AND date_end AND pageviews > 1
      ),

      -- 2. BASE LAYER: Session metrics + site content + device/geo/traffic
      base_sessions AS (
        SELECT
          ga.user_pseudo_id,
          ga.user_pseudo_id || '/' || event_date || '/' ||
            (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') as session_id,
          MIN(SAFE.PARSE_DATE('%Y%m%d', _TABLE_SUFFIX)) as date,
          MIN(TIMESTAMP_MICROS(event_timestamp)) as session_start_tstamp,
          MAX(TIMESTAMP_MICROS(event_timestamp)) as session_end_tstamp,

          -- Engagement metrics
          TIMESTAMP_DIFF(MAX(TIMESTAMP_MICROS(event_timestamp)),
            MIN(TIMESTAMP_MICROS(event_timestamp)), SECOND) as time_on_site,
          COUNTIF(event_name = 'page_view') as pageviews,
          COUNTIF(event_name = 'view_item') as view_items,
          COUNTIF(event_name = 'view_promotion') as view_promotions,
          COUNTIF(event_name = 'select_promotion') as select_promotion,
          COUNTIF(event_name = 'view_search_results') as view_search_results,
          COUNTIF(event_name = 'add_to_cart') as add_to_carts,
          COUNTIF(event_name = 'begin_checkout') as begin_checkout,
          COUNTIF(event_name = 'purchase') as purchase,
          COUNTIF(event_name = 'scroll') as scrolls,
          COUNTIF(event_name = 'video_start') as video_start,
          COUNTIF(event_name = 'video_complete') as video_complete,

          -- Site content features (page_location patterns) — adapt URL patterns to your site
          COUNTIF(event_name = 'page_view' AND
            LOWER((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location'))
            LIKE '%checkout%') AS checkout_pageviews,
          COUNTIF(event_name = 'page_view' AND
            LOWER((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location'))
            LIKE '%product%') AS product_pageviews,
          COUNTIF(event_name = 'page_view' AND
            LOWER((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location'))
            LIKE '%my-account%') AS account_pageviews,

          -- REGEXP for URL query param filtering (e.g. ?category=X or ?vertical=Y)
          COUNTIF(event_name = 'page_view' AND REGEXP_CONTAINS(
            LOWER((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location')),
            r'[?&]category=target_value([%&]|$)')
          ) AS category_page_views,

          -- Device one-hot
          MAX(IF(device.category = 'desktop', 1, 0)) as desktop,
          MAX(IF(device.category = 'mobile', 1, 0)) as mobile,
          MAX(IF(device.category = 'tablet', 1, 0)) as tablet,

          -- Geo one-hot (pick top N countries/regions by session volume for your market)
          MAX(IF(geo.country = 'CountryA', 1, 0)) as is_country_a,
          MAX(IF(geo.region = 'RegionA', 1, 0)) as is_region_a,
          MAX(IF(geo.region = 'RegionB', 1, 0)) as is_region_b,

          -- Traffic source one-hot
          MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'cpc', 1, 0)) as is_medium_cpc,
          MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'organic', 1, 0)) as is_medium_organic,
          MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'email', 1, 0)) as is_medium_email,

          -- Event params extraction
          MAX(IFNULL((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'engaged_session_event' LIMIT 1), 0)) as engaged_session,
          MAX(IFNULL((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'percent_scrolled' LIMIT 1), 0)) as max_percent_scrolled,
          COALESCE(MAX((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'logged_in_status')), 0) as logged_in_status

        FROM `project.dataset.events_*` as ga
        INNER JOIN visitor_pool as vp
          ON vp.user_pseudo_id = ga.user_pseudo_id
            AND TIMESTAMP_MICROS(ga.event_timestamp) BETWEEN
              TIMESTAMP_SUB(vp.last_event_tstamp, INTERVAL LOOKBACK_DAYS DAY)
              AND TIMESTAMP_ADD(vp.last_event_tstamp, INTERVAL LOOKAHEAD_DAYS DAY)
        WHERE SAFE.PARSE_DATE('%Y%m%d', _TABLE_SUFFIX)
          BETWEEN date_start - LOOKBACK_DAYS AND date_end + LOOKAHEAD_DAYS
        GROUP BY 1, 2
      ),

      -- 3. LABEL: Did conversion occur in lookahead window?
      conv AS (
        SELECT
          bs1.user_pseudo_id, bs1.session_id,
          IF(MAX(bs2.purchase) > 0, 1, 0) as label
        FROM base_sessions as bs1
        LEFT JOIN base_sessions as bs2
          ON bs2.user_pseudo_id = bs1.user_pseudo_id
            AND bs2.session_start_tstamp BETWEEN bs1.session_end_tstamp
              AND TIMESTAMP_ADD(bs1.session_end_tstamp, INTERVAL LOOKAHEAD_DAYS DAY)
        GROUP BY 1, 2
      )

    -- 4. FEATURE ASSEMBLY
    SELECT
      bs.user_pseudo_id, bs.session_id, bs.date,
      bs.session_start_tstamp, bs.session_end_tstamp,

      -- Current session features
      bs.time_on_site, bs.pageviews, bs.view_items, bs.view_promotions,
      bs.select_promotion, bs.view_search_results, bs.add_to_carts,
      bs.begin_checkout, bs.purchase, bs.scrolls, bs.video_start, bs.video_complete,

      -- Site content (current session) — list all page_location features from base_sessions
      bs.checkout_pageviews, bs.product_pageviews, bs.account_pageviews, bs.category_page_views,

      -- Device/geo/traffic (current session)
      bs.desktop, bs.mobile, bs.tablet,
      bs.is_country_a, bs.is_region_a, bs.is_region_b,
      bs.is_medium_cpc, bs.is_medium_organic, bs.is_medium_email,

      -- Event params
      bs.engaged_session, bs.max_percent_scrolled, bs.logged_in_status,

      -- Cumulative features (running totals)
      COUNT(*) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as sessions,
      SUM(bs.time_on_site) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ttl_time_on_site,
      SUM(bs.pageviews) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ttl_pageviews,
      SUM(bs.view_items) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ttl_view_items,
      SUM(bs.add_to_carts) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ttl_add_to_carts,
      SUM(bs.begin_checkout) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ttl_begin_checkout,
      SUM(bs.purchase) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ttl_purchase,

      -- Cumulative site content (repeat SUM OVER for each page_location feature)
      SUM(bs.checkout_pageviews) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ttl_checkout_pageviews,
      SUM(bs.product_pageviews) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ttl_product_pageviews,

      -- Cumulative logged-in sessions
      SUM(bs.logged_in_status) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ttl_logged_in_sessions,

      -- "Ever seen" flags (cumulative MAX for one-hot features over the lookback window)
      MAX(bs.is_medium_cpc) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ever_medium_cpc,
      MAX(bs.is_medium_organic) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ever_medium_organic,
      MAX(bs.is_region_a) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) as ever_region_a,

      -- Recency: hours since last purchase
      IFNULL(TIMESTAMP_DIFF(bs.session_end_tstamp,
        MAX(IF(bs.purchase > 0, bs.session_end_tstamp, NULL))
          OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp),
        HOUR), LOOKBACK_DAYS * 24 * 2) as last_purchase_hours_ago,

      -- Derived features
      SAFE_DIVIDE(bs.time_on_site, bs.pageviews) as avg_time_per_page,
      COUNT(*) OVER (PARTITION BY bs.user_pseudo_id ORDER BY bs.session_start_tstamp) / LOOKBACK_DAYS as sessions_per_day,

      c.label
    FROM base_sessions as bs
    INNER JOIN conv as c USING (session_id)
  );

  -- 5. OUTPUT
  IF mode = 'TRAINING' THEN
    EXECUTE IMMEDIATE FORMAT("""
      CREATE OR REPLACE TABLE %s
      OPTIONS(expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY),
              labels = [("vai-mlops", "training")])
      AS SELECT
        CASE
          WHEN MOD(ABS(FARM_FINGERPRINT(user_pseudo_id)), 100) < 70 THEN 'TRAIN'
          WHEN MOD(ABS(FARM_FINGERPRINT(user_pseudo_id)), 100) < 90 THEN 'EVAL'
          ELSE 'TEST'
        END as data_split,
        * EXCEPT (user_pseudo_id, session_id, date, session_start_tstamp, session_end_tstamp)
      FROM dataset WHERE date <= @de;
    """, table_name) USING date_end - LOOKAHEAD_DAYS as de;
  END IF;

  IF mode = 'INFERENCE' THEN
    EXECUTE IMMEDIATE FORMAT("""
      CREATE OR REPLACE TABLE %s
      OPTIONS(expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 1 DAY),
              labels = [("vai-mlops", "inference")])
      AS SELECT * EXCEPT (label)
      FROM dataset WHERE date BETWEEN @ds AND @de;
    """, table_name) USING date_start as ds, date_end as de;
  END IF;

  DROP TABLE dataset;
END;
```

### Key Patterns Demonstrated

1. **Session-level grain**: Each session is a decision point ("retarget this visitor?")
2. **Visitor pool**: >1 pageview excludes bounced visitors
3. **Label via self-join**: Future sessions within lookahead checked for conversion
4. **Three feature layers**: Current session → cumulative running totals → recency
5. **Site content features**: page_location LIKE/REGEXP patterns (applied at session + cumulative)
6. **One-hot device/geo/traffic**: Session-level flags + `MAX OVER (...)` for "ever seen" lookback flags
7. **Event params**: Integer extraction with IFNULL default
8. **Conditional label**: Can be customized (e.g., `purchase WHERE program='AUTO' AND product NOT LIKE '%accessory%'`)

---

## Example 2: Predictive LTV (User-Level with Snapshots)

**Objective**: Predict total revenue in next N days. **Grain**: One row per user per snapshot (weekly Mondays).

**Label**: Continuous — revenue in `[snapshot + 1, snapshot + LOOKAHEAD_DAYS]`.

```sql
CREATE OR REPLACE PROCEDURE `project.dataset.create_ltv_dataset`(
  table_name STRING, date_start DATE, date_end DATE, mode STRING
)
BEGIN
  DECLARE LOOKBACK_DAYS INT64 DEFAULT 395;
  DECLARE LOOKAHEAD_DAYS INT64 DEFAULT 395;
  DECLARE snapshot_dates ARRAY<DATE>;

  IF mode = 'TRAINING' THEN
    SET snapshot_dates = ARRAY(
      SELECT d FROM UNNEST(GENERATE_DATE_ARRAY(date_start, date_end, INTERVAL 1 DAY)) d
      WHERE EXTRACT(DAYOFWEEK FROM d) = 2 ORDER BY d DESC
    );
  ELSEIF mode = 'INFERENCE' THEN
    SET snapshot_dates = [DATE_TRUNC(date_end, WEEK(MONDAY))];
    SET date_end = snapshot_dates[SAFE_OFFSET(0)];
    SET date_start = date_end - LOOKBACK_DAYS;
  END IF;

  CREATE OR REPLACE TEMP TABLE dataset AS (
    WITH
      user_pool AS (
        SELECT user_id FROM (
          SELECT user_id, DATE(activity_timestamp) as activity_date
          FROM events_table
          WHERE DATE(activity_timestamp) BETWEEN date_start - LOOKBACK_DAYS AND date_end
            AND user_id IS NOT NULL
        )
        GROUP BY user_id
        HAVING MAX(activity_date) BETWEEN date_start AND date_end
      ),

      base_orders AS (
        SELECT user_id, DATE(created_at) as date,
          COUNT(*) as daily_orders, SUM(revenue) as daily_revenue, SUM(items) as daily_items
        FROM orders INNER JOIN user_pool USING (user_id)
        WHERE DATE(created_at) BETWEEN date_start - LOOKBACK_DAYS AND date_end + LOOKAHEAD_DAYS
        GROUP BY ALL
      ),

      future_revenue AS (
        SELECT d as snapshot_date, user_id, SUM(IFNULL(daily_revenue, 0)) as label
        FROM base_orders
        INNER JOIN UNNEST(snapshot_dates) d ON date BETWEEN d + 1 AND d + LOOKAHEAD_DAYS
        GROUP BY ALL
      )

    SELECT d as date, user_id,
      -- Time-windowed spend
      IFNULL(SUM(IF(DATE_DIFF(d, o.date, DAY) BETWEEN 0 AND 28, daily_revenue, 0)), 0) as spend_last_28d,
      IFNULL(SUM(IF(DATE_DIFF(d, o.date, DAY) BETWEEN 29 AND 70, daily_revenue, 0)), 0) as spend_29d_to_70d,
      IFNULL(SUM(IF(DATE_DIFF(d, o.date, DAY) BETWEEN 71 AND 140, daily_revenue, 0)), 0) as spend_71d_to_140d,
      IFNULL(SUM(IF(DATE_DIFF(d, o.date, DAY) > 140, daily_revenue, 0)), 0) as spend_over_140d_ago,
      -- Recency & totals
      IFNULL(DATE_DIFF(d, MAX(o.date), DAY), LOOKBACK_DAYS + 1) as last_order_days_ago,
      IFNULL(SUM(daily_orders), 0) as total_orders,
      IFNULL(SUM(daily_revenue), 0) as total_revenue,
      -- Seasonal
      IFNULL(SAFE_DIVIDE(
        SUM(IF(EXTRACT(MONTH FROM o.date) IN (11, 12), daily_revenue, 0)), SUM(daily_revenue)
      ), 0) as pct_spend_gifting_season,
      -- Label
      IFNULL(fr.label, 0) as label
    FROM base_orders o
    INNER JOIN UNNEST(snapshot_dates) d ON o.date BETWEEN d - LOOKBACK_DAYS AND d
    LEFT JOIN future_revenue fr ON fr.user_id = o.user_id AND fr.snapshot_date = d
    GROUP BY ALL
  );

  IF mode = 'TRAINING' THEN
    EXECUTE IMMEDIATE FORMAT("""
      CREATE OR REPLACE TABLE %s OPTIONS(expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)) AS
      SELECT CASE
          WHEN MOD(ABS(FARM_FINGERPRINT(user_id)), 100) < 70 THEN 'TRAIN'
          WHEN MOD(ABS(FARM_FINGERPRINT(user_id)), 100) < 90 THEN 'EVAL'
          ELSE 'TEST' END as data_split,
        * EXCEPT (user_id)
      FROM dataset WHERE date <= @de
      QUALIFY ROW_NUMBER() OVER (
        PARTITION BY user_id ORDER BY FARM_FINGERPRINT(CONCAT(user_id, '/', CAST(date AS STRING)))
      ) = 1;
    """, table_name) USING CURRENT_DATE() - LOOKAHEAD_DAYS as de;
  END IF;

  IF mode = 'INFERENCE' THEN
    EXECUTE IMMEDIATE FORMAT("""
      CREATE OR REPLACE TABLE %s OPTIONS(expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)) AS
      SELECT user_id, * EXCEPT (user_id, label) FROM dataset WHERE date = @de;
    """, table_name) USING snapshot_dates[SAFE_OFFSET(0)] as de;
  END IF;

  DROP TABLE dataset;
END;
```

### Key Patterns

1. **Snapshot dates**: Weekly Mondays activate historical data from all lifecycle phases
2. **Long windows (395d)**: LTV needs seasonal patterns, purchase frequency over a year
3. **One snapshot per user**: `QUALIFY ROW_NUMBER()` with FARM_FINGERPRINT to decorrelate
4. **Feature join enforces boundary**: `ON o.date BETWEEN d - LOOKBACK_DAYS AND d`
5. **Label from separate CTE**: Clean separation of feature vs label logic

---

## Feature Catalog by Category

### Behavioral Events (session-level COUNTIF)

```sql
COUNTIF(event_name = 'page_view') as pageviews,
COUNTIF(event_name = 'add_to_cart') as add_to_carts,
COUNTIF(event_name = 'begin_checkout') as begin_checkout,
COUNTIF(event_name = 'purchase') as purchase,
COUNTIF(event_name = 'view_search_results') as view_search_results,
COUNTIF(event_name = 'select_item') as select_item,
COUNTIF(event_name = 'login') as login,
COUNTIF(event_name = 'scroll') as scrolls,
COUNTIF(event_name = 'user_engagement') as user_engagements,
```

### Site Content (page_location patterns)

```sql
-- Simple LIKE (substring match on URL path)
COUNTIF(event_name = 'page_view' AND LOWER(page_location) LIKE '%checkout%') AS checkout_pageviews,
COUNTIF(event_name = 'page_view' AND LOWER(page_location) LIKE '%account%') AS account_pageviews,
COUNTIF(event_name = 'page_view' AND LOWER(page_location) LIKE '%product%') AS product_pageviews,

-- REGEXP (URL path or query param pattern)
COUNTIF(event_name = 'page_view' AND REGEXP_CONTAINS(LOWER(page_location),
  r'/section/subsection')) AS section_page_views,
```

### Device / Geo / Platform

```sql
MAX(IF(device.category = 'desktop', 1, 0)) as desktop,
MAX(IF(device.category = 'mobile', 1, 0)) as mobile,
MAX(IF(geo.country = 'CountryA', 1, 0)) as is_country_a,
MAX(IF(geo.region = 'RegionA', 1, 0)) as is_region_a,
MAX(IF(platform = 'WEB', 1, 0)) as is_web,
MAX(IF(platform = 'IOS', 1, 0)) as is_ios,
```

### Traffic Source

```sql
MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'cpc', 1, 0)) as is_medium_cpc,
MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'organic', 1, 0)) as is_medium_organic,
MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'email', 1, 0)) as is_medium_email,
MAX(IF(LOWER(session_traffic_source_last_click.cross_channel_campaign.medium) = 'referral', 1, 0)) as is_medium_referral,
```

### Event Params

```sql
MAX(IFNULL((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'engaged_session_event' LIMIT 1), 0)) as engaged_session,
MAX(IFNULL((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'percent_scrolled' LIMIT 1), 0)) as max_percent_scrolled,
SUM(IFNULL((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'video_duration' LIMIT 1), 0)) as total_video_duration_sec,
COALESCE(MAX((SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'logged_in_status')), 0) as logged_in_status,
```

### External Enrichment

```sql
-- User characteristics CTE (joined after features)
LEFT JOIN user_characteristics uc ON bs.user_pseudo_id = uc.user_pseudo_id

-- Holiday proximity
LEFT JOIN (SELECT MIN(holiday_date) as next_holiday FROM holidays WHERE holiday_date >= f.date) ...
DATE_DIFF(next_holiday_date, date, DAY) as days_to_next_holiday,

-- User lookup tables (subscription status, customer type)
LEFT JOIN user_lookup u ON f.user_pseudo_id = u.user_pseudo_id AND f.date = u.date
```

### Conditional Labels (Business-Specific Conversion)

```sql
-- Simple purchase
IF(MAX(bs2.purchase) > 0, 1, 0) as label

-- Non-purchase event as conversion (e.g. subscription upgrade, registration)
IF(MAX(bs2.conversion_event_count) > 0, 1, 0) as label

-- Conditional purchase (filter by event_params to narrow the conversion definition)
COUNTIF(event_name = 'purchase'
  AND (SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'param_key') = 'target_value'
  AND LOWER((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'product_category')) NOT LIKE '%excluded_type%'
) as conversion
```

### Training with Population Filters

```sql
-- Exclude an existing user segment from training (model targets new conversions only)
FROM dataset WHERE date <= @de AND has_existing_subscription = 0

-- Inference: apply the same population filter
FROM dataset WHERE date BETWEEN @ds AND @de AND has_existing_subscription = 0
```
