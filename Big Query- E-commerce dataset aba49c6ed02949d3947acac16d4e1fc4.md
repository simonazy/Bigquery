# Big Query- E-commerce dataset

In this note, I will record how to use the powerful BigQuery product for data-to-insights analysis. BigQuery is behaves like a database along with writing SQL queries on the cloud. The storage is cheap and fast, while analyst can focus on QUERIES, instead of infrastructure. Next, I will present exploring an E-commerce dataset using SQL in Google BigQuery. 

1. Click **Navigation menu** > **BigQuery**.

![Untitled](Big%20Query-%20E-commerce%20dataset%20aba49c6ed02949d3947acac16d4e1fc4/Untitled.png)

1. Click Done

BigQuery public datasets are not displayed by default in the BigQuery web UI. To open the public datasets project, open [https://console.cloud.google.com/bigquery?p=data-to-insights&page=project](https://console.cloud.google.com/bigquery?p=data-to-insights&page=project) in a new browser window.

1. In the left pane, under **Viewing pinned projects** you will see the **data-to-insights** project.
2. Open **all_sessions_raw**.

In the right pane, a section opens that provides 3 views of the table data:

- Schema tab: Field name, Type, Mode, and Description; the logical constraints used to organize the data
- Details tab: Table metadata
- Preview tab: Table preview

![Untitled](Big%20Query-%20E-commerce%20dataset%20aba49c6ed02949d3947acac16d4e1fc4/Untitled%201.png)

1. Now click the EDITOR, then we can write some Queries to test. 

(1) find duplicate records across all columns field

```sql
SELECT
  COUNT(*) AS num_duplicate_rows,
  *
FROM
  `data-to-insights.ecommerce.all_sessions_raw`
GROUP BY
  fullVisitorId,
  channelGrouping,
  time,
  country,
  city,
  totalTransactionRevenue,
  transactions,
  timeOnSite,
  pageviews,
  sessionQualityDim,
  date,
  visitId,
  type,
  productRefundAmount,
  productQuantity,
  productPrice,
  productRevenue,
  productSKU,
  v2ProductName,
  v2ProductCategory,
  productVariant,
  currencyCode,
  itemQuantity,
  itemRevenue,
  transactionRevenue,
  transactionId,
  pageTitle,
  searchKeyword,
  pagePathLevel1,
  eCommerceAction_type,
  eCommerceAction_step,
  eCommerceAction_option
HAVING
  num_duplicate_rows > 1;
```

I really like the "Format Query" function. No matter how messy my queries are, the console will organize it and make it tidy as above. And you can also save the Query for later use by saving it as a name you like. 

(2) Write a query that shows total unique visitors

```sql
SELECT
  COUNT(*) AS product_views,
  COUNT(DISTINCT fullVisitorId) AS unique_visitors
FROM `data-to-insights.ecommerce.all_sessions`;
```

 (3) Write a query that shows total unique visitors(`fullVisitorID`) by the referring site (`channelGrouping`): 

```sql
SELECT
  COUNT(DISTINCT fullVisitorId) AS unique_visitors,
  channelGrouping
FROM `data-to-insights.ecommerce.all_sessions`
GROUP BY channelGrouping
ORDER BY channelGrouping DESC;
# Tip: In SQL, the ORDER BY clauses defaults to Ascending (ASC) A-->Z. 
# If you want the reverse, try ORDER BY field_name DESC
```

(4) Write a query to list all the unique product names (`v2ProductName`) alphabetically:

```sql
SELECT
  (v2ProductName) AS ProductName
FROM `data-to-insights.ecommerce.all_sessions`
GROUP BY ProductName
ORDER BY ProductName
```

(5) You can use the SQL `WITH` clause to help break apart a complex query into multiple steps. Here we first create a query that finds each unique product per visitor and counts them once. Then the second query performs the aggregation across all visitors and products.

```sql
WITH unique_product_views_by_person AS (
-- find each unique product viewed by each visitor
SELECT
 fullVisitorId,
 (v2ProductName) AS ProductName
FROM `data-to-insights.ecommerce.all_sessions`
WHERE type = 'PAGE'
GROUP BY fullVisitorId, v2ProductName )
-- aggregate the top viewed products and sort them
SELECT
  COUNT(*) AS unique_view_count,
  ProductName
FROM unique_product_views_by_person
GROUP BY ProductName
ORDER BY unique_view_count DESC
LIMIT 5
```

(6)Write a query to see the orders, product names and views.

```sql

SELECT
  COUNT(*) AS product_views,
  COUNT(productQuantity) AS orders,
  SUM(productQuantity) AS quantity_product_ordered,
  SUM(productQuantity) / COUNT(productQuantity) AS avg_per_order,
  (v2ProductName) AS ProductName
FROM `data-to-insights.ecommerce.all_sessions`
WHERE type = 'PAGE'
GROUP BY v2ProductName
ORDER BY product_views DESC
LIMIT 5;
```

### There are 3 challenges for the dataset:

1. Calculate a conversion rate

```sql
SELECT
  COUNT(*) AS product_views,
  COUNT(productQuantity) AS potential_orders,
  SUM(productQuantity) AS quantity_product_added,
  (COUNT(productQuantity) / COUNT(*)) AS conversion_rate,
  v2ProductName
FROM `data-to-insights.ecommerce.all_sessions`
WHERE LOWER(v2ProductName) NOT LIKE '%frisbee%'
GROUP BY v2ProductName
HAVING quantity_product_added > 1000
ORDER BY conversion_rate DESC
LIMIT 10;
```

Note: Alias do not exist when filtering in where, will error in where clause, allowed in order by, group by, having only. 

1. Track visitor checkout progress

```sql
SELECT
  COUNT(DISTINCT fullVisitorId) AS number_of_unique_visitors,
  eCommerceAction_type,
  CASE eCommerceAction_type
	WHEN '0' THEN 'Unknown'
	WHEN '1' THEN 'Click through of product lists'
  WHEN '2' THEN 'Product detail views'
  WHEN '3' THEN 'Add product(s) to cart'
  WHEN '4' THEN 'Remove product(s) from cart'
  WHEN '5' THEN 'Check out'
  WHEN '6' THEN 'Completed purchase'
  WHEN '7' THEN 'Refund of purchase'
  WHEN '8' THEN 'Checkout options'
	ELSE 'ERROR'
	END AS eCommerceAction_type_label
FROM `data-to-insights.ecommerce.all_sessions`
GROUP BY eCommerceAction_type
ORDER BY eCommerceAction_type;
```

The result table is as follows: 

![Untitled](Big%20Query-%20E-commerce%20dataset%20aba49c6ed02949d3947acac16d4e1fc4/Untitled%202.png)

1. Track abandoned carts from high quality sessions

```sql
# high quality abandoned carts
SELECT  
  #unique_session_id
  CONCAT(fullVisitorId,CAST(visitId AS STRING)) AS unique_session_id,
  sessionQualityDim,
  SUM(productRevenue) AS transaction_revenue,
  MAX(eCommerceAction_type) AS checkout_progress
FROM `data-to-insights.ecommerce.all_sessions`
WHERE sessionQualityDim > 60 # high quality session
GROUP BY unique_session_id, sessionQualityDim
HAVING
  checkout_progress = '3' # 3 = added to cart
  AND (transaction_revenue = 0 OR transaction_revenue IS NULL)
```

The result table is as follows:

![Untitled](Big%20Query-%20E-commerce%20dataset%20aba49c6ed02949d3947acac16d4e1fc4/Untitled%203.png)

Those customers added product to their cart but new completed checkout, to whom could be potential customers and the operation team should pay attention. 

The date related functions in SQL: 

![Untitled](Big%20Query-%20E-commerce%20dataset%20aba49c6ed02949d3947acac16d4e1fc4/Untitled%204.png)

Parse string values:

![Untitled](Big%20Query-%20E-commerce%20dataset%20aba49c6ed02949d3947acac16d4e1fc4/Untitled%205.png)

Wildcard filter with LIKE:

```sql
SELECT
	ein,
	name,
FROM
`bigquery-public-data.irs_990.irs_990_ein`
WHERE
	LOWER(name) LIKE '%help%'
LIMIT 10;
```

```sql
SELECT
  COUNT(DISTINCT fullVisitorId) as visitor_count, 
  hits_page_pageTitle
FROM `data-to-insights.ecommerce.rev_transactions` LIMIT 1000
GROUP BY hits_page_pageTitle
```

```sql
SELECT
  geoNetwork_city,
  SUM(totals_transactions) AS totals_transactions,
  COUNT( DISTINCT fullVisitorId) AS distinct_visitors
FROM
  `data-to-insights.ecommerce.rev_transactions`
GROUP BY
  geoNetwork_city
```

```sql
SELECT
  geoNetwork_city,
  SUM(totals_transactions) AS total_products_ordered,
  COUNT( DISTINCT fullVisitorId) AS distinct_visitors,
  SUM(totals_transactions) / COUNT( DISTINCT fullVisitorId) AS avg_products_ordered,
FROM
  `data-to-insights.ecommerce.rev_transactions`
GROUP BY
  geoNetwork_city
ORDER BY
  avg_products_ordered DESC
```

```sql
# filter the result to only return only avg>20 
SELECT
geoNetwork_city,
SUM(totals_transactions) AS total_products_ordered,
COUNT( DISTINCT fullVisitorId) AS distinct_visitors,
SUM(totals_transactions) / COUNT( DISTINCT fullVisitorId) AS avg_products_ordered
FROM
`data-to-insights.ecommerce.rev_transactions`
GROUP BY
geoNetwork_city
HAVING
avg_products_ordered > 20
ORDER BY
avg_products_ordered DESC
```

```sql
SELECT
COUNT(DISTINCT hits_product_v2ProductName) as number_of_products,
hits_product_v2ProductCategory
FROM `data-to-insights.ecommerce.rev_transactions`
WHERE hits_product_v2ProductName IS NOT NULL
GROUP BY hits_product_v2ProductCategory
ORDER BY number_of_products DESC
LIMIT 5
```