# Default Channel Grouping: Full CASE Logic

## User-scoped channel grouping (based on traffic_source)
```sql
CASE
  WHEN (traffic_source.source IS NULL OR traffic_source.source = '(direct)') 
    AND (traffic_source.medium IS NULL OR traffic_source.medium IN ('(not set)', '(none)')) THEN 'Direct'
  WHEN traffic_source.name LIKE '%cross-network%' THEN 'Cross-network'
  WHEN (REGEXP_CONTAINS(traffic_source.source, 'alibaba|amazon|google shopping|shopify|etsy|ebay|stripe|walmart')
    OR REGEXP_CONTAINS(traffic_source.name, '^(.*(([^a-df-z]|^)shop|shopping).*)$'))
    AND REGEXP_CONTAINS(traffic_source.medium, '^(.*cp.*|ppc|retargeting|paid.*)$') THEN 'Paid Shopping'
  WHEN REGEXP_CONTAINS(traffic_source.source, 'baidu|bing|duckduckgo|ecosia|google|yahoo|yandex') 
    AND REGEXP_CONTAINS(traffic_source.medium, '^(.*cp.*|ppc|retargeting|paid.*)$') THEN 'Paid Search'
  WHEN REGEXP_CONTAINS(traffic_source.source, 'badoo|facebook|fb|instagram|linkedin|pinterest|tiktok|twitter|whatsapp')
    AND REGEXP_CONTAINS(traffic_source.medium, '^(.*cp.*|ppc|retargeting|paid.*)$') THEN 'Paid Social'
  WHEN REGEXP_CONTAINS(traffic_source.source, 'dailymotion|disneyplus|netflix|youtube|vimeo|twitch|vimeo|youtube') 
    AND REGEXP_CONTAINS(traffic_source.medium, '^(.*cp.*|ppc|retargeting|paid.*)$') THEN 'Paid Video'
  WHEN traffic_source.medium IN ('display', 'banner', 'expandable', 'interstitial', 'cpm') THEN 'Display'
  WHEN REGEXP_CONTAINS(traffic_source.medium, '^(.*cp.*|ppc|retargeting|paid.*)$') THEN 'Paid Other'
  WHEN REGEXP_CONTAINS(traffic_source.source, 'alibaba|amazon|google shopping|shopify|etsy|ebay|stripe|walmart') 
    OR REGEXP_CONTAINS(traffic_source.name, '^(.*(([^a-df-z]|^)shop|shopping).*)$') THEN 'Organic Shopping'
  WHEN REGEXP_CONTAINS(traffic_source.source, 'badoo|facebook|fb|instagram|linkedin|pinterest|tiktok|twitter|whatsapp') 
    OR traffic_source.medium IN ('social', 'social-network', 'social-media', 'sm', 'social network', 'social media') THEN 'Organic Social'
  WHEN REGEXP_CONTAINS(traffic_source.source, 'dailymotion|disneyplus|netflix|youtube|vimeo|twitch|vimeo|youtube') 
    OR REGEXP_CONTAINS(traffic_source.medium, '^(.*video.*)$') THEN 'Organic Video'
  WHEN REGEXP_CONTAINS(traffic_source.source, 'baidu|bing|duckduckgo|ecosia|google|yahoo|yandex') 
    OR traffic_source.medium = 'organic' THEN 'Organic Search'
  WHEN traffic_source.medium IN ('referral', 'app', 'link') THEN 'Referral'
  WHEN REGEXP_CONTAINS(traffic_source.source, 'email|e-mail|e_mail|e mail') 
    OR REGEXP_CONTAINS(traffic_source.medium, 'email|e-mail|e_mail|e mail') THEN 'Email'
  WHEN traffic_source.medium = 'affiliate' THEN 'Affiliates'
  WHEN traffic_source.medium = 'audio' THEN 'Audio'
  WHEN traffic_source.source = 'sms' OR traffic_source.medium = 'sms' THEN 'SMS'
  WHEN traffic_source.medium LIKE '%push' OR REGEXP_CONTAINS(traffic_source.medium, 'mobile|notification') 
    OR traffic_source.source = 'firebase' THEN 'Mobile Push Notifications'
  ELSE 'Unassigned'
END AS channel_grouping
```

## Session-scoped channel grouping
Same CASE logic but applied to session-level source/medium (from CTE that extracts session source/medium). Replace `traffic_source.source` with `source`, `traffic_source.medium` with `medium`, `traffic_source.name` with `campaign`.

## Important notes
- Based on Google's official documentation on how GA4 classifies traffic
- Not all conditions can be reconstructed — some variables aren't available in BigQuery export
- The order of CASE WHEN matters — paid channels checked before organic
- Channel list may need periodic updates as Google adds new classifications
