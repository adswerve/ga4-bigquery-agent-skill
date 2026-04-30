# Page Tracking, Events & Date/Time Dimensions

## Page tracking dimensions

### Page location (URL)
```sql
(SELECT value.string_value FROM UNNEST(event_params) WHERE event_name = 'page_view' AND key = 'page_location') AS page
```

### Page title
```sql
(SELECT value.string_value FROM UNNEST(event_params) WHERE event_name = 'page_view' AND key = 'page_title') AS page_title
```

### Landing page
```sql
CASE WHEN (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'page_view' AND key = 'entrances') = 1 
  THEN (SELECT value.string_value FROM UNNEST(event_params) WHERE event_name = 'page_view' AND key = 'page_location') 
END AS landing_page
```

### Previous page (using LAG window function)
```sql
LAG((SELECT value.string_value FROM UNNEST(event_params) WHERE event_name = 'page_view' AND key = 'page_location'), 1) 
  OVER (PARTITION BY user_pseudo_id, 
    (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'page_view' AND key = 'ga_session_id') 
    ORDER BY event_timestamp ASC) AS previous_page
```

### Page path levels (split URL)
```sql
CONCAT('/', SPLIT(SPLIT(page_location, '/')[SAFE_ORDINAL(4)], '?')[SAFE_ORDINAL(1)]) AS pagepath_level_1
```

### Hostname
```sql
device.web_info.hostname
```

## Page metrics
- **Page views**: `COUNTIF(event_name = 'page_view')`
- **Unique pageviews**: `COUNT(DISTINCT CASE WHEN event_name = 'page_view' THEN CONCAT(user_pseudo_id, session_id) END)`
- **Pages per session**: `COUNTIF(event_name = 'page_view') / COUNT(DISTINCT session_key)`
- **Entrances**: `COUNT(CASE WHEN entrances = 1 THEN session_key END)`

## Event dimensions
- `event_date` — YYYYMMDD in property timezone
- `event_timestamp` — microseconds, UTC
- `event_name` — e.g., page_view, session_start, purchase, scroll, click
- `event_value_in_usd` — currency-converted value
- `batch_page_id`, `batch_ordering_id`, `batch_event_index` — from 2024-07-11

## Date/time dimensions

### CRITICAL timezone note
- `event_date` is in the **property's registered timezone** (YYYYMMDD string)
- `event_timestamp` is in **UTC microseconds**
- **Best practice**: Use `event_timestamp` as source for all time dimensions to prevent timezone errors

### Date formatting patterns
```sql
-- Parse event_date to DATE type
PARSE_DATE('%Y%m%d', event_date) AS event_date_parsed

-- From event_timestamp (preferred)
DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date_utc

-- Year
FORMAT_DATE('%Y', PARSE_DATE('%Y%m%d', event_date)) AS year

-- Month of year (YYYYMM)
FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', event_date)) AS month_of_year

-- ISO week
FORMAT_DATE('%G%V', PARSE_DATE('%Y%m%d', event_date)) AS iso_week

-- Day of week (Sunday=0)
FORMAT_DATE('%w', PARSE_DATE('%Y%m%d', event_date)) AS day_of_week

-- Day of week name
FORMAT_DATE('%A', PARSE_DATE('%Y%m%d', event_date)) AS day_name

-- Hour (from timestamp)
FORMAT('%02d', EXTRACT(HOUR FROM TIMESTAMP_MICROS(event_timestamp))) AS hour

-- Minute
FORMAT('%02d', EXTRACT(MINUTE FROM TIMESTAMP_MICROS(event_timestamp))) AS minute
```

## Device, geo & platform dimensions

### Device
```sql
device.category          -- mobile, tablet, desktop
device.operating_system
device.browser
device.language
device.mobile_brand_name
```

### Geo
```sql
geo.continent
geo.country
geo.region
geo.city
geo.metro
```

### Platform/stream
```sql
platform    -- WEB or APP
stream_id
```
