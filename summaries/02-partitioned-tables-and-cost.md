# Querying Partitioned Tables: _table_suffix Patterns

## CRITICAL: Always filter with _table_suffix
GA4 data is stored as one table per day. Using `events_*` wildcard without `_table_suffix` filter scans ALL tables = expensive.

## Table reference pattern
```sql
FROM `{project}.{dataset}.events_*`
WHERE _table_suffix BETWEEN '{start}' AND '{end}'
```

## Date filtering patterns

### Static date range
```sql
WHERE _table_suffix BETWEEN '20240701' AND '20240731'
```

### Dynamic rolling window (last 30 days)
```sql
WHERE _table_suffix BETWEEN 
  format_date('%Y%m%d', date_sub(current_date(), INTERVAL 30 DAY))
  AND format_date('%Y%m%d', date_sub(current_date(), INTERVAL 1 DAY))
```

### Fixed start + dynamic end (RECOMMENDED for most use cases)
```sql
WHERE _table_suffix BETWEEN '20240101' 
  AND format_date('%Y%m%d', date_sub(current_date(), INTERVAL 1 DAY))
```
BigQuery figures out which tables exist and returns data only for matching tables.

### Including intraday (streaming) data
Intraday tables have format `events_intraday_YYYYMMDD`. To include both daily and intraday:
```sql
WHERE regexp_extract(_table_suffix, r'[0-9]+') BETWEEN 
  format_date('%Y%m%d', date_sub(current_date(), INTERVAL 7 DAY))
  AND format_date('%Y%m%d', current_date())
```
Use `regexp_replace(_table_suffix, 'intraday_', '')` to normalize the date column in results.

## Intraday table limitations
- Streaming export does NOT include user-attribution data for new users (e.g., `traffic_source.medium`)
- For existing users, user-attribution takes ~24 hours to process
- **Recommendation**: Do NOT rely on intraday for user-attribution data; use daily export instead

## Cost optimization rules
1. **Always use `_table_suffix`** to limit scanned tables
2. **Select only needed columns** — avoid `SELECT *`
3. Use `event_name` filter early in WHERE clause to reduce rows
4. Use `SAFE_DIVIDE` instead of raw division to avoid division-by-zero errors
5. Use the `SAFE.` prefix (e.g., `SAFE.parse_date()`) to return NULL instead of errors on bad data
