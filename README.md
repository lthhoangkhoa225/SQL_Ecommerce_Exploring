# SQL_Ecommerce_Exploring
## I. Introduction
This project contains an eCommerce dataset that I will explore using SQL on Google BigQuery. The dataset is based on the Google Analytics public dataset and contains data from an eCommerce website.
## II. Requirements
- Google Cloud Platform account
- Project on Google Cloud Platform
- Google BigQuery API enabled
- SQL query editor or IDE
## III. Dataset Access
The eCommerce dataset is stored in a public Google BigQuery dataset. To access the dataset, follow these steps:

- Log in to your Google Cloud Platform account and create a new project.
- Navigate to the BigQuery console and select your newly created project.
- In the navigation panel, select "Add Data" and then "Search a project".
- Enter the project ID **"bigquery-public-data.google_analytics_sample.ga_sessions"** and click "Enter".
- Click on the **"ga_sessions_"** table to open it.
## IV. Exploring the Dataset
In this project, I will write 08 query in Bigquery base on Google Analytics dataset
### Query 01: Calculate total visit, pageview, transaction and revenue for January, February and March 2017 order by month

```
SELECT LEFT(date,6)                     month
      , SUM(totals.visits)              visits 
      , SUM(totals.pageviews)           pageviews
      , SUM(totals.transactions)        transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix BETWEEN '0101' AND '0331'
GROUP BY month
ORDER BY month;
```
### Query 02: Bounce rate per traffic source in July 2017
```
SELECT trafficSource.`source`            source
      ,SUM(totals.visits)                total_visits
      ,SUM(totals.bounces)               total_no_of_bounces
      ,ROUND(SUM(totals.bounces)/
      SUM(totals.visits)*100, 8)  bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
GROUP BY source
ORDER BY total_visits DESC;
```
### Query 03: Revenue by traffic source by week, by month in June 2017
```
WITH 
    week_revenue AS (  
      SELECT 
            'Week'                        time_type
            ,FORMAT_DATE('%G%V',
            PARSE_DATE('%Y%m%d', date))   time 
            ,trafficSource.`source`       source
            ,ROUND(SUM(product.productRevenue)/
            1000000,4)                    revenue
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
      UNNEST (hits)                       hits,
      UNNEST (hits.product)               product
      WHERE product.productRevenue IS NOT NULL
      GROUP BY time,source)
    ,month_revenue AS (  
      SELECT 
            'Month'                       time_type
            ,FORMAT_DATE('%Y%m',
            PARSE_DATE('%Y%m%d', date))   time 
            ,trafficSource.`source`       source
            ,ROUND(SUM(product.productRevenue)/
            1000000,4)                    revenue
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
      UNNEST (hits)                       hits,
      UNNEST (hits.product)               product
      WHERE product.productRevenue IS NOT NULL
      GROUP BY time,source)

      SELECT * FROM week_revenue
      UNION ALL
      SELECT * FROM month_revenue
      ORDER BY revenue DESC;
```
### Query 04: Average number of product pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017
```
WITH
    months AS(
      SELECT 
      fullVisitorId
      ,totals.pageviews
      ,totals.transactions
      ,product.productRevenue             productRevenue
      ,FORMAT_DATE('%Y%m',
      PARSE_DATE('%Y%m%d', date))         month             
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
      UNNEST (hits)                       hits,
      UNNEST (hits.product)               product
      WHERE _table_suffix BETWEEN '0601' AND '0731')
    ,purchase AS (
      SELECT month
            ,ROUND(SUM(pageviews)/
        COUNT(DISTINCT fullVisitorId),8)  avg_pageviews_purchase
      FROM months 
      WHERE transactions >=1 AND productRevenue IS NOT NULL
      GROUP BY month)
    ,non_purchase AS (
      SELECT month
            ,ROUND(SUM(pageviews)/
        COUNT(DISTINCT fullVisitorId),7)  avg_pageviews_non_purchase
      FROM months 
      WHERE transactions IS NULL AND productRevenue IS NULL
      GROUP BY month)

  SELECT p.month
        ,p.avg_pageviews_purchase
        ,np.avg_pageviews_non_purchase
  FROM purchase                            p
  LEFT JOIN non_purchase                   np
  USING(month)
  ORDER BY p.month;
```
### Query 05: Average number of transactions per user that made a purchase in July 2017
```
SELECT 
  FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date))                     month
  ,ROUND(SUM(totals.transactions)/COUNT(DISTINCT fullVisitorId), 9)  Avg_total_transactions_per_user          
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST (hits)                       hits,
  UNNEST (hits.product)               product
WHERE totals.transactions >=1
AND totals.totalTransactionRevenue IS NOT NULL
AND product.productRevenue IS NOT NULL
GROUP BY month;
```
### Query 06: Average amount of money spent per session. Only include purchaser data in July 2017
```
SELECT 
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date))         month
  ,ROUND(
  (SUM(product.productRevenue)/SUM(totals.visits))
  /POWER(10,6), 2)                                        avg_revenue_by_user_per_visit
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits)                       hits,
UNNEST (hits.product)               product
WHERE product.productRevenue IS NOT NULL 
GROUP BY month;
```
### Query 07: Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. Output should show product name and the quantity was ordered
```
WITH  
    fields AS (
    SELECT 
      fullVisitorId
      ,totals.transactions
      ,product.productRevenue             productRevenue
      ,product.v2ProductName              v2ProductName    
      ,product.productQuantity            quantity
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
      UNNEST (hits)                       hits,
      UNNEST (hits.product)               product)
  
  SELECT v2ProductName                    other_purchased_products
        ,SUM(quantity)                    quantity
  FROM fields
  WHERE fullVisitorId IN (
      SELECT fullVisitorId
      FROM fields
      WHERE transactions >=1 
      AND productRevenue IS NOT NULL 
      AND v2ProductName = "YouTube Men's Vintage Henley")  
  AND v2ProductName <> "YouTube Men's Vintage Henley"
  AND transactions >=1 
  AND productRevenue IS NOT NULL
  GROUP BY other_purchased_products
  ORDER BY quantity DESC;
```
### Query 08: Calculate cohort map from pageview to addtocart to purchase in last 3 month
```
WITH product_data AS(
  SELECT
      FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date))         month
      ,COUNT(CASE WHEN eCommerceAction.action_type = '2' THEN product.v2ProductName END)  num_product_view
      ,COUNT(CASE WHEN eCommerceAction.action_type = '3' THEN product.v2ProductName END)  num_add_to_cart
      ,COUNT(CASE WHEN eCommerceAction.action_type = '6' AND product.productRevenue IS NOT NULL THEN product.v2ProductName END)  num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`
    ,UNNEST(hits)             hits
    ,UNNEST (hits.product)    product
  WHERE _table_suffix BETWEEN '20170101' AND '20170331'
  AND eCommerceAction.action_type IN ('2','3','6')
  GROUP BY month
  ORDER BY month)

  SELECT
      *,
      ROUND(num_add_to_cart/num_product_view * 100, 2)  add_to_cart_rate,
      ROUND(num_purchase/num_product_view * 100, 2)     purchase_rate
  FROM product_data;
```
## V. Conclusion


