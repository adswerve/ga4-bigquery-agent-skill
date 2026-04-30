# Ecommerce: Transactions, Items & Revenue

## Ecommerce event hierarchy
| Event | When fired | Contains items? |
|-------|-----------|----------------|
| view_item | Product page view | Yes |
| add_to_cart | Item added to cart | Yes |
| remove_from_cart | Item removed | Yes |
| view_cart | Cart viewed | Yes |
| begin_checkout | Checkout started | Yes |
| add_payment_info | Payment info added | Yes |
| add_shipping_info | Shipping info added | Yes |
| purchase | Transaction complete | Yes |
| refund | Refund processed | Yes |

## Transaction-level fields (ecommerce record)
```sql
SELECT
  ecommerce.transaction_id,
  SUM(ecommerce.total_item_quantity) AS total_items,
  SUM(ecommerce.purchase_revenue_in_usd) AS revenue_usd,
  SUM(ecommerce.purchase_revenue) AS revenue_local,
  SUM(ecommerce.shipping_value) AS shipping,
  SUM(ecommerce.tax_value) AS tax,
  SUM(ecommerce.refund_value) AS refunds,
  SUM(ecommerce.unique_items) AS unique_items
FROM `{project}.{dataset}.events_*`
WHERE event_name = 'purchase'
  AND _table_suffix BETWEEN '{start}' AND '{end}'
GROUP BY transaction_id
```

**CRITICAL**: Always filter `event_name = 'purchase'` when querying revenue. Revenue fields are only populated on purchase events.

## Item-level fields (items array — must UNNEST)
```sql
SELECT
  items.item_id,
  items.item_name,
  items.item_brand,
  items.item_variant,
  items.item_category,    -- up to item_category5
  items.price,
  items.price_in_usd,
  items.quantity,
  items.item_revenue,     -- price * quantity, purchase events only
  items.item_revenue_in_usd,
  items.item_refund_in_usd,
  items.coupon,
  items.affiliation,
  items.item_list_id,
  items.item_list_name,
  items.item_list_index,
  items.promotion_id,
  items.promotion_name,
  items.creative_name,
  items.creative_slot,
  items.location_id
FROM `{project}.{dataset}.events_*`,
  UNNEST(items) AS items
WHERE event_name = 'purchase'
  AND _table_suffix BETWEEN '{start}' AND '{end}'
```

**CRITICAL**: When querying items, you MUST specify `event_name`. Without it, quantity/revenue includes values from add_to_cart, view_item, etc.

## Ecommerce funnel pivot
```sql
WITH prep AS (
  SELECT
    item.item_name,
    user_pseudo_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS session_id,
    event_name
  FROM `{project}.{dataset}.events_*`,
    UNNEST(items) AS item
  WHERE _table_suffix BETWEEN '{start}' AND '{end}'
)
SELECT * FROM prep
PIVOT (
  COUNT(DISTINCT CONCAT(user_pseudo_id, session_id))
  FOR event_name IN ('view_item', 'add_to_cart', 'purchase')
)
ORDER BY view_item DESC
```

## Market basket analysis (product pairs)
```sql
WITH purchase_items AS (
  SELECT ecommerce.transaction_id, item.item_name
  FROM `{project}.{dataset}.events_*`, UNNEST(items) AS item
  WHERE event_name = 'purchase'
    AND ecommerce.transaction_id IS NOT NULL
    AND item.item_name IS NOT NULL
    AND _table_suffix BETWEEN '{start}' AND '{end}'
),
product_pairs AS (
  SELECT a.item_name AS item1, b.item_name AS item2, a.transaction_id
  FROM purchase_items a
  JOIN purchase_items b ON a.transaction_id = b.transaction_id
  WHERE a.item_name < b.item_name  -- avoid duplicate pairs
)
SELECT item1, item2, COUNT(DISTINCT transaction_id) AS times_bought_together
FROM product_pairs
GROUP BY 1, 2
ORDER BY 3 DESC
```
