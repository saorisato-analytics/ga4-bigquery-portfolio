# BigQuery SQL分析クエリ集

このファイルには、GA4データを活用した実践的な分析クエリを収録しています。

## 🔧 基本設定

### プロジェクト情報
- **プロジェクトID**: `portfolio-analytics-468111`
- **データセット**: `analytics_*`
- **テーブル**: `events_*`

---

## 📊 基本分析クエリ

### 1. データ確認・概要把握

```sql
-- 基本的なデータ確認
SELECT 
  event_date,
  event_name,
  COUNT(*) as event_count,
  COUNT(DISTINCT user_pseudo_id) as unique_users
FROM `portfolio-analytics-468111.analytics_*.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20240101' AND '20241231'
GROUP BY event_date, event_name
ORDER BY event_date DESC, event_count DESC
LIMIT 100;
```

### 2. 日別トラフィック分析

```sql
-- 日別ページビュー・ユーザー数
SELECT 
  event_date,
  COUNT(CASE WHEN event_name = 'page_view' THEN 1 END) as page_views,
  COUNT(DISTINCT user_pseudo_id) as unique_users,
  COUNT(DISTINCT CONCAT(user_pseudo_id, 
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id'))) as sessions,
  ROUND(COUNT(CASE WHEN event_name = 'page_view' THEN 1 END) / 
    COUNT(DISTINCT CONCAT(user_pseudo_id, 
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id'))), 2) as pages_per_session
FROM `portfolio-analytics-468111.analytics_*.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) 
  AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY event_date
ORDER BY event_date DESC;
```

---

## 🛒 Eコマース分析クエリ

### 3. 商品別パフォーマンス分析

```sql
-- 商品別の閲覧・カート・購入数とコンバージョン率
SELECT 
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'item_id') as product_id,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'item_name') as product_name,
  COUNT(CASE WHEN event_name = 'view_item' THEN 1 END) as views,
  COUNT(CASE WHEN event_name = 'add_to_cart' THEN 1 END) as add_to_carts,
  COUNT(CASE WHEN event_name = 'purchase' THEN 1 END) as purchases,
  
  -- コンバージョン率計算
  ROUND(COUNT(CASE WHEN event_name = 'add_to_cart' THEN 1 END) / 
        NULLIF(COUNT(CASE WHEN event_name = 'view_item' THEN 1 END), 0) * 100, 2) as cart_rate_percent,
  ROUND(COUNT(CASE WHEN event_name = 'purchase' THEN 1 END) / 
        NULLIF(COUNT(CASE WHEN event_name = 'add_to_cart' THEN 1 END), 0) * 100, 2) as purchase_rate_percent,
  ROUND(COUNT(CASE WHEN event_name = 'purchase' THEN 1 END) / 
        NULLIF(COUNT(CASE WHEN event_name = 'view_item' THEN 1 END), 0) * 100, 2) as overall_cvr_percent

FROM `portfolio-analytics-468111.analytics_*.events_*`
WHERE event_name IN ('view_item', 'add_to_cart', 'purchase')
  AND _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) 
  AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY product_id, product_name
HAVING product_id IS NOT NULL
ORDER BY views DESC;
```

### 4. 売上分析

```sql
-- 日別・商品別売上分析
SELECT 
  event_date,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'item_name') as product_name,
  COUNT(CASE WHEN event_name = 'purchase' THEN 1 END) as transactions,
  SUM(CASE WHEN event_name = 'purchase' THEN 
    (SELECT value.double_value FROM UNNEST(event_params) WHERE key = 'value') 
    ELSE 0 END) as total_revenue,
  AVG(CASE WHEN event_name = 'purchase' THEN 
    (SELECT value.double_value FROM UNNEST(event_params) WHERE key = 'value') 
    ELSE NULL END) as avg_order_value

FROM `portfolio-analytics-468111.analytics_*.events_*`
WHERE event_name = 'purchase'
  AND _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) 
  AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY event_date, product_name
HAVING product_name IS NOT NULL
ORDER BY event_date DESC, total_revenue DESC;
```

---

## 👥 ユーザー行動分析

### 5. ユーザーセッション分析

```sql
-- ユーザー別のエンゲージメント分析
SELECT 
  user_pseudo_id,
  COUNT(DISTINCT CONCAT(user_pseudo_id, 
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id'))) as sessions,
  COUNT(*) as total_events,
  COUNT(CASE WHEN event_name = 'page_view' THEN 1 END) as page_views,
  COUNT(CASE WHEN event_name = 'add_to_cart' THEN 1 END) as cart_additions,
  COUNT(CASE WHEN event_name = 'purchase' THEN 1 END) as purchases,
  
  -- 売上計算
  SUM(CASE WHEN event_name = 'purchase' THEN 
    (SELECT value.double_value FROM UNNEST(event_params) WHERE key = 'value') 
    ELSE 0 END) as total_revenue,
    
  -- 初回・最終セッション日
  MIN(PARSE_DATE('%Y%m%d', event_date)) as first_seen,
  MAX(PARSE_DATE('%Y%m%d', event_date)) as last_seen,
  DATE_DIFF(MAX(PARSE_DATE('%Y%m%d', event_date)), 
            MIN(PARSE_DATE('%Y%m%d', event_date)), DAY) as days_active

FROM `portfolio-analytics-468111.analytics_*.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) 
  AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY user_pseudo_id
HAVING sessions > 0
ORDER BY total_revenue DESC
LIMIT 100;
```

### 6. コンバージョンファネル分析

```sql
-- カスタマージャーニーのファネル分析
WITH user_journey AS (
  SELECT 
    user_pseudo_id,
    MAX(CASE WHEN event_name = 'page_view' THEN 1 ELSE 0 END) as had_page_view,
    MAX(CASE WHEN event_name = 'view_item' THEN 1 ELSE 0 END) as viewed_product,
    MAX(CASE WHEN event_name = 'add_to_cart' THEN 1 ELSE 0 END) as added_to_cart,
    MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END) as made_purchase
  FROM `portfolio-analytics-468111.analytics_*.events_*`
  WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) 
    AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  GROUP BY user_pseudo_id
)

SELECT 
  'Page View' as funnel_step,
  SUM(had_page_view) as users,
  ROUND(SUM(had_page_view) / COUNT(*) * 100, 2) as conversion_rate
FROM user_journey

UNION ALL

SELECT 
  'Product View' as funnel_step,
  SUM(viewed_product) as users,
  ROUND(SUM(viewed_product) / NULLIF(SUM(had_page_view), 0) * 100, 2) as conversion_rate
FROM user_journey

UNION ALL

SELECT 
  'Add to Cart' as funnel_step,
  SUM(added_to_cart) as users,
  ROUND(SUM(added_to_cart) / NULLIF(SUM(viewed_product), 0) * 100, 2) as conversion_rate
FROM user_journey

UNION ALL

SELECT 
  'Purchase' as funnel_step,
  SUM(made_purchase) as users,
  ROUND(SUM(made_purchase) / NULLIF(SUM(added_to_cart), 0) * 100, 2) as conversion_rate
FROM user_journey

ORDER BY 
  CASE funnel_step 
    WHEN 'Page View' THEN 1
    WHEN 'Product View' THEN 2
    WHEN 'Add to Cart' THEN 3
    WHEN 'Purchase' THEN 4
  END;
```

---

## 📈 高度な分析クエリ

### 7. RFM分析（顧客セグメンテーション）

```sql
-- RFM分析: Recency, Frequency, Monetary
WITH customer_metrics AS (
  SELECT 
    user_pseudo_id,
    -- Recency: 最後の購入からの日数
    DATE_DIFF(CURRENT_DATE(), MAX(PARSE_DATE('%Y%m%d', event_date)), DAY) as recency_days,
    -- Frequency: 購入回数
    COUNT(CASE WHEN event_name = 'purchase' THEN 1 END) as frequency,
    -- Monetary: 総購入金額
    SUM(CASE WHEN event_name = 'purchase' THEN 
      (SELECT value.double_value FROM UNNEST(event_params) WHERE key = 'value') 
      ELSE 0 END) as monetary_value
  FROM `portfolio-analytics-468111.analytics_*.events_*`
  WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)) 
    AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  GROUP BY user_pseudo_id
  HAVING frequency > 0
),

rfm_scores AS (
  SELECT 
    user_pseudo_id,
    recency_days,
    frequency,
    monetary_value,
    -- RFMスコア算出（1-5のスケール）
    NTILE(5) OVER (ORDER BY recency_days DESC) as r_score,
    NTILE(5) OVER (ORDER BY frequency ASC) as f_score,
    NTILE(5) OVER (ORDER BY monetary_value ASC) as m_score
  FROM customer_metrics
)

SELECT 
  r_score,
  f_score,
  m_score,
  CONCAT(CAST(r_score AS STRING), CAST(f_score AS STRING), CAST(m_score AS STRING)) as rfm_segment,
  COUNT(*) as customer_count,
  AVG(recency_days) as avg_recency,
  AVG(frequency) as avg_frequency,
  AVG(monetary_value) as avg_monetary,
  -- セグメント分類
  CASE 
    WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
    WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 'Loyal Customers'
    WHEN r_score >= 3 AND f_score <= 2 AND m_score >= 3 THEN 'Big Spenders'
    WHEN r_score <= 2 AND f_score >= 3 AND m_score >= 3 THEN 'At Risk'
    WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost Customers'
    ELSE 'Others'
  END as customer_segment

FROM rfm_scores
GROUP BY r_score, f_score, m_score
ORDER BY customer_count DESC;
```

### 8. コホート分析（リテンション率）

```sql
-- 月次コホート分析
WITH first_purchase AS (
  SELECT 
    user_pseudo_id,
    MIN(PARSE_DATE('%Y%m%d', event_date)) as first_purchase_date,
    DATE_TRUNC(MIN(PARSE_DATE('%Y%m%d', event_date)), MONTH) as cohort_month
  FROM `portfolio-analytics-468111.analytics_*.events_*`
  WHERE event_name = 'purchase'
    AND _TABLE_SUFFIX BETWEEN '20240101' AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  GROUP BY user_pseudo_id
),

user_activities AS (
  SELECT 
    fp.user_pseudo_id,
    fp.cohort_month,
    DATE_TRUNC(PARSE_DATE('%Y%m%d', e.event_date), MONTH) as activity_month,
    DATE_DIFF(DATE_TRUNC(PARSE_DATE('%Y%m%d', e.event_date), MONTH), fp.cohort_month, MONTH) as month_number
  FROM first_purchase fp
  JOIN `portfolio-analytics-468111.analytics_*.events_*` e
    ON fp.user_pseudo_id = e.user_pseudo_id
  WHERE e.event_name = 'purchase'
    AND e._TABLE_SUFFIX BETWEEN '20240101' AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
)

SELECT 
  cohort_month,
  COUNT(DISTINCT CASE WHEN month_number = 0 THEN user_pseudo_id END) as cohort_size,
  COUNT(DISTINCT CASE WHEN month_number = 1 THEN user_pseudo_id END) as month_1,
  COUNT(DISTINCT CASE WHEN month_number = 2 THEN user_pseudo_id END) as month_2,
  COUNT(DISTINCT CASE WHEN month_number = 3 THEN user_pseudo_id END) as month_3,
  
  -- リテンション率
  ROUND(COUNT(DISTINCT CASE WHEN month_number = 1 THEN user_pseudo_id END) / 
        NULLIF(COUNT(DISTINCT CASE WHEN month_number = 0 THEN user_pseudo_id END), 0) * 100, 2) as retention_month_1,
  ROUND(COUNT(DISTINCT CASE WHEN month_number = 2 THEN user_pseudo_id END) / 
        NULLIF(COUNT(DISTINCT CASE WHEN month_number = 0 THEN user_pseudo_id END), 0) * 100, 2) as retention_month_2,
  ROUND(COUNT(DISTINCT CASE WHEN month_number = 3 THEN user_pseudo_id END) / 
        NULLIF(COUNT(DISTINCT CASE WHEN month_number = 0 THEN user_pseudo_id END), 0) * 100, 2) as retention_month_3

FROM user_activities
GROUP BY cohort_month
ORDER BY cohort_month;
```

---

## 🔍 デバッグ・監視クエリ

### 9. データ品質チェック

```sql
-- データ品質監視クエリ
SELECT 
  event_date,
  event_name,
  COUNT(*) as event_count,
  COUNT(DISTINCT user_pseudo_id) as unique_users,
  -- 異常値検知
  COUNT(CASE WHEN event_timestamp IS NULL THEN 1 END) as null_timestamps,
  COUNT(CASE WHEN user_pseudo_id IS NULL THEN 1 END) as null_user_ids,
  -- データの完全性チェック
  ROUND(COUNT(DISTINCT user_pseudo_id) / COUNT(*) * 100, 2) as user_event_ratio
FROM `portfolio-analytics-468111.analytics_*.events_*`
WHERE _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
GROUP BY event_date, event_name
ORDER BY event_count DESC;
```

### 10. パフォーマンス監視

```sql
-- 処理性能監視クエリ
SELECT 
  job_id,
  creation_time,
  start_time,
  end_time,
  state,
  TIMESTAMP_DIFF(end_time, start_time, SECOND) as duration_seconds,
  total_bytes_processed,
  total_slot_ms
FROM `portfolio-analytics-468111.region-asia-northeast1.INFORMATION_SCHEMA.JOBS_BY_PROJECT`
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
ORDER BY creation_time DESC
LIMIT 10;
```

---

## 📚 クエリ使用方法

### 実行手順
1. BigQuery Console にアクセス
2. プロジェクト `portfolio-analytics-468111` を選択
3. 上記クエリをコピー&ペースト
4. 日付範囲を適切に調整
5. 実行してレポート作成

### 注意事項
- データが蓄積されるまで24-48時間必要
- `_TABLE_SUFFIX` の日付範囲を適切に設定
- パフォーマンスを考慮してLIMIT句を使用
- 本番環境では適切な権限設定を実施

---

**これらのクエリを使用して、実践的なデータ分析を実行できます。**
