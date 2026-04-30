# GA4 BigQuery Export: Schema & Table Structure

## Dataset Naming
- Dataset: `analytics_<property_id>` (e.g., `analytics_250794857`)
- Property ID = GA4 property ID from property settings

## Table Types
- **Daily export**: `events_YYYYMMDD` — full daily table, created once processing completes (typically ~5-6 AM property timezone)
- **Streaming/intraday export**: `events_intraday_YYYYMMDD` — populated continuously; deleted when daily table is complete
- **User data export** (optional, must be activated separately):
  - `pseudonymous_users_YYYYMMDD` — one row per `user_pseudo_id`
  - `users_YYYYMMDD` — one row per `user_id`

## Late-arriving events
Analytics updates daily tables for up to **3 days** after the event date. Events arriving after that window are dropped.

## Row structure
Each row = one event. Repeated fields (event_params, user_properties, items) create nested data within each row. A single event like `page_view` may have many event_params key-value pairs.

## Core top-level fields
| Field | Type | Description |
|-------|------|-------------|
| event_date | STRING | YYYYMMDD in property timezone |
| event_timestamp | INTEGER | Microseconds, UTC |
| event_name | STRING | e.g., page_view, session_start, purchase |
| event_params | RECORD (REPEATED) | Key-value pairs for event parameters |
| user_pseudo_id | STRING | Client ID / cookie-based identifier |
| user_id | STRING | Optional custom user ID set via setUserId |
| user_properties | RECORD (REPEATED) | Key-value user properties |
| user_first_touch_timestamp | INTEGER | Microseconds |
| user_ltv | RECORD | revenue, currency |
| device | RECORD | category, browser, OS, etc. |
| geo | RECORD | country, region, city, etc. |
| traffic_source | RECORD | User-scoped first-touch: source, medium, name |
| collected_traffic_source | RECORD | Event-scoped raw traffic data |
| session_traffic_source_last_click | RECORD | Session-scoped last-click attribution |
| ecommerce | RECORD | transaction_id, purchase_revenue, etc. |
| items | RECORD (REPEATED) | Item-level data for ecommerce events |
| privacy_info | RECORD | analytics_storage, ads_storage consent |
| is_active_user | BOOLEAN | Available from 2023-07-17 |
| stream_id | STRING | Numeric stream ID |
| platform | STRING | WEB or APP |
| batch_page_id | INTEGER | Available from 2024-07-11 |
| batch_ordering_id | INTEGER | Available from 2024-07-11 |
| batch_event_index | INTEGER | Available from 2024-07-11 |

## Three repeated/nested fields
1. **event_params**: key (STRING) + value (string_value, int_value, float_value, double_value). Only ONE value type is populated per parameter.
2. **user_properties**: Same structure as event_params. User property does NOT persist on every event — must be propagated manually.
3. **items**: Flat struct with item_id, item_name, price, quantity, etc. + nested item_params (available from 2023-10-25).

## Data scopes
GA4 has four data scopes:
- **User scope**: Applies to the user across all sessions (e.g., traffic_source, user_properties)
- **Session scope**: Applies to a session (e.g., session_traffic_source_last_click)
- **Event scope**: Applies to individual events (e.g., event_params, collected_traffic_source)
- **Item scope**: Applies to individual items within ecommerce events (e.g., items array)

## Sample datasets for testing
- **Web ecommerce**: `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*` (Nov 2020 - Jan 2021)
- **App gaming**: `firebase-public-project.analytics_153293282.events_*` (Jun - Oct 2018)
- Data is obfuscated; cannot be compared to GA4 demo account
