# GA4 Conversion Rate Analysis with BigQuery

## Goal

Form a table of:

- **URLs**
- **Converted visits**
- **Non-converted visits**
- **Conversion rate**

---

## Tools

- **BigQuery**
- **Google Analytics 4 (GA4)**

---

## Testing Dataset

BigQuery sample dataset for Google Analytics ecommerce web implementation:  
https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset

Represented in SQL as `future-abacus-432804-b6.ecom_sample.small_table`

To form this table from your GA4 dataset you need to use the next SQL
```sql
CREATE TABLE AS
SELECT
*
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_2021**`
```

---

# Introduction

We have the following situations:

1. **Non-converted users**  
   A user visits the website, views some pages, and does **not** convert  
   → we count **non-converted visits**.

2. **Converted users**  
   A user visits the website, views some pages, **converts**, and optionally continues browsing.  
   In this scenario, we **must cut off** all page visits **after the purchase event** to form **converted visits** only.

---

## Relevant GA4 Event Fields

- **`event_name`**  
  We are interested only in:
  - `"page_view"`
  - `"purchase"`

- **`event_timestamp`**  
  Used to cut all events that happened **after** the purchase.

- **`user_pseudo_id`**  
  Allows allocating events to each user and determining whether the user converted.

- **`event_params`**  
  Contains attributes of each event.  
  We need only:
  - `"page_location"`

---

# Raw Table Example

| event_name | event_timestamp     | user_pseudo_id         | kkey          | string_value |
|------------|----------------------|-------------------------|----------------|--------------|
| purchase   | 1610190654784273     | 4753564.8168919712      | page_location | https://shop.googlemerchandisestore.com/ordercompleted.html |
| purchase   | 1611623602149258     | 64920042.7172093895     | page_location | https://shop.googlemerchandisestore.com/ordercompleted.html |
| purchase   | 1608631521164025     | 1056498.5939985013      | page_location | https://shop.googlemerchandisestore.com/ordercompleted.html |
| purchase   | 1606662235500223     | 9038823.9978976976      | page_location | https://shop.googlemerchandisestore.com/ordercompleted.html |
| purchase   | 1609212506753932     | 6698829.2913246021      | page_location | https://shop.googlemerchandisestore.com/ordercompleted.html |
| page_view  | 1610802751842518     | 59922034.5119352133     | page_location | https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Hats |
| page_view  | 1605998854380318     | 9397545.7933031135      | page_location | https://shop.googlemerchandisestore.com/basket.html |
| page_view  | 1607022884423192     | 3072270.1606967669      | page_location | https://shop.googlemerchandisestore.com/Google+Redesign/Shop+by+Brand/Google |
| page_view  | 1604911574451108     | 73419256.0536577404     | page_location | https://shop.googlemerchandisestore.com/ |
| page_view  | 1608126768145859     | 27864124.5566371181     | page_location | https://shop.googlemerchandisestore.com/Google+Redesign/Campus+Collection |

---

To extract it from GA4 data in BigQuery we use SQL

### SQL Query



```sql
SELECT
  event_name,
  event_timestamp,
  user_pseudo_id,
  param.key AS kkey,
  param.value.string_value AS string_value
FROM `future-abacus-432804-b6.ecom_sample.small_table`
CROSS JOIN UNNEST(event_params) AS param
WHERE param.key IN ('page_location')
  AND event_name IN ("page_view", "purchase");
```


## Count non-converted visits

We need to choose *unique* **used_pseudo_id**, **visited_pages** combinations, filter **non-converted users**, and count non-converted visits for every page to form the table

| visited_page                                                                                           | non_converted_visits |
|--------------------------------------------------------------------------------------------------------|------------------------|
| https://shop.googlemerchandisestore.com/                                                               | 2040                   |
| https://shop.googlemerchandisestore.com/Google+Redesign/Apparel                                        | 1160                   |
| https://googlemerchandisestore.com/                                                                    | 770                    |
| https://shop.googlemerchandisestore.com/store.html                                                     | 652                    |
| https://shop.googlemerchandisestore.com/Google+Redesign/Shop+by+Brand/YouTube                          | 607                    |
| https://www.googlemerchandisestore.com/                                                                | 557                    |
| https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Google+Dino+Game+Tee                   | 460                    |
| https://shop.googlemerchandisestore.com/basket.html                                                    | 431                    |
| https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Mens                                   | 431                    |
| https://shop.googlemerchandisestore.com/Google+Redesign/Clearance                                      | 421                    |


### SQL Query


```sql
WITH non_converted_visits AS
(SELECT DISTINCT
    user_pseudo_id,
    param.key AS kkey,
    param.value.string_value AS string_value
  FROM `future-abacus-432804-b6.ecom_sample.small_table`
  CROSS JOIN UNNEST(event_params) AS param
WHERE param.key IN ('page_location') AND
event_name = "page_view"
AND user_pseudo_id NOT IN (SELECT user_pseudo_id FROM `future-abacus-432804-b6.ecom_sample.small_table` WHERE event_name = "purchase"))
SELECT string_value AS visited_page, count(string_value) AS non_converted_visits  FROM non_converted_visits GROUP BY string_value ORDER BY non_converted_visits DESC
```


## Cutting Post-Purchase Visits

Next, we need to identify **converted visits** for specific URLs and **remove all visits that happened after the purchase**.

To achieve this, we join the raw event table **to itself** by `user_pseudo_id` and pull the **minimum `event_timestamp`** where `event_name = "purchase"` for each user.  
This minimum timestamp represents the **first conversion time**.

---

### Example of the Resulting Table

| event_name | event_timestamp     | user_pseudo_id         | kkey          | string_value                                                                                              | first_conversion     |
|------------|----------------------|-------------------------|----------------|-------------------------------------------------------------------------------------------------------------|----------------------|
| page_view  | 1611293028277679     | 10092926.3786306416     | page_location | https://shop.googlemerchandisestore.com/Google+Redesign/Lifestyle/Bags                                     | 1611293856871176     |
| page_view  | 1611293016291786     | 10092926.3786306416     | page_location | https://shop.googlemerchandisestore.com/Google+Redesign/Apparel                                            | 1611293856871176     |
| page_view  | 1611293067647757     | 10092926.3786306416     | page_location | https://shop.googlemerchandisestore.com/Google+Redesign/Clearance                                          | 1611293856871176     |
| page_view  | 1611293326976845     | 10092926.3786306416     | page_location | https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Socks                                      | 1611293856871176     |
| page_view  | 1611293676367371     | 10092926.3786306416     | page_location | https://shop.googlemerchandisestore.com/yourinfo.html                                                      | 1611293856871176     |
| page_view  | 1607603088270889     | 10111055.8768683862     | page_location | https://shop.googlemerchandisestore.com/signin.html                                                        | 1607603334901943     |
| page_view  | 1607602735718750     | 10111055.8768683862     | page_location | https://shop.googlemerchandisestore.com/                                                                   | 1607603334901943     |
| page_view  | 1607603233055739     | 10111055.8768683862     | page_location | https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Mens                                       | 1607603334901943     |
| page_view  | 1607603244106979     | 10111055.8768683862     | page_location | https://shop.googlemerchandisestore.com/basket.html                                                        | 1607603334901943     |
| page_view  | 1607603261470461     | 10111055.8768683862     | page_location | https://shop.googlemerchandisestore.com/yourinfo.html                                                      | 1607603334901943     |

---

### SQL Query


```sql
WITH unnest_table AS (
  SELECT
    event_name,
    event_timestamp,
    user_pseudo_id,
    param.key AS kkey,
    param.value.string_value AS string_value
  FROM `future-abacus-432804-b6.ecom_sample.small_table`
  CROSS JOIN UNNEST(event_params) AS param
WHERE param.key IN ('page_location') AND
event_name IN ("page_view","purchase"))
SELECT
	lt.event_name,
    lt.event_timestamp,
    lt.user_pseudo_id,
    lt.kkey,
    lt.string_value,
    rt.event_timestamp AS first_conversion
FROM unnest_table lt
JOIN (SELECT user_pseudo_id, MIN(event_timestamp) AS event_timestamp FROM unnest_table WHERE event_name = 'purchase' GROUP BY user_pseudo_id) rt 
ON 
rt.user_pseudo_id = lt.user_pseudo_id
WHERE lt.event_name = "page_view" AND lt.event_timestamp <= rt.event_timestamp
```


## Redundant duplicates

Note, we can have a conversion rate here of over 100% when somebody visits some page twice, so we need to delete duplicates and form the next table


| user_pseudo_id         | unique_visited_page                                                                                         |
|------------------------|--------------------------------------------------------------------------------------------------------|
| 10092926.3786306416    | https://shop.googlemerchandisestore.com/Google+Redesign/Lifestyle/Bags                                |
| 10092926.3786306416    | https://shop.googlemerchandisestore.com/Google+Redesign/Apparel                                       |
| 10092926.3786306416    | https://shop.googlemerchandisestore.com/Google+Redesign/Clearance                                     |
| 10092926.3786306416    | https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Socks                                 |
| 10092926.3786306416    | https://shop.googlemerchandisestore.com/yourinfo.html                                                 |
| 10111055.8768683862    | https://shop.googlemerchandisestore.com/signin.html                                                   |
| 10111055.8768683862    | https://shop.googlemerchandisestore.com/                                                              |
| 10111055.8768683862    | https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Mens                                  |
| 10111055.8768683862    | https://shop.googlemerchandisestore.com/basket.html                                                   |
| 10111055.8768683862    | https://shop.googlemerchandisestore.com/yourinfo.html                                                 |




### SQL Query


```sql
WITH final_table AS(
WITH unnest_table AS (
  SELECT
    event_name,
    event_timestamp,
    user_pseudo_id,
    param.key AS kkey,
    param.value.string_value AS string_value
  FROM `future-abacus-432804-b6.ecom_sample.small_table`
  CROSS JOIN UNNEST(event_params) AS param
WHERE param.key IN ('page_location') AND
event_name IN ("page_view","purchase"))
SELECT
	lt.event_name,
    lt.event_timestamp,
    lt.user_pseudo_id,
    lt.kkey,
    lt.string_value,
    rt.event_timestamp AS first_conversion
FROM unnest_table lt
JOIN (SELECT user_pseudo_id, MIN(event_timestamp) AS event_timestamp FROM unnest_table WHERE event_name = 'purchase' GROUP BY user_pseudo_id) rt 
ON 
rt.user_pseudo_id = lt.user_pseudo_id
WHERE lt.event_name = "page_view" AND lt.event_timestamp <= rt.event_timestamp)
SELECT DISTINCT user_pseudo_id, string_value AS visited_page FROM final_table
;
```


## Converted visits

We form a table of URL and visits

| visited_page                                                                                  | converted_visits |
|-----------------------------------------------------------------------------------------------|--------|
| https://shop.googlemerchandisestore.com/payment.html                                          | 99     |
| https://shop.googlemerchandisestore.com/basket.html                                           | 99     |
| https://shop.googlemerchandisestore.com/yourinfo.html                                         | 99     |
| https://shop.googlemerchandisestore.com/signin.html                                           | 87     |
| https://shop.googlemerchandisestore.com/                                                      | 84     |
| https://shop.googlemerchandisestore.com/ordercompleted.html                                   | 83     |
| https://shop.googlemerchandisestore.com/store.html                                            | 73     |
| https://shop.googlemerchandisestore.com/Google+Redesign/Clearance                             | 59     |
| https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Mens                          | 55     |
| https://shop.googlemerchandisestore.com/revieworder.html                                      | 54     |

### SQL Query

```sql
WITH final_table AS(
WITH unnest_table AS (
  SELECT
    event_name,
    event_timestamp,
    user_pseudo_id,
    param.key AS kkey,
    param.value.string_value AS string_value
  FROM `future-abacus-432804-b6.ecom_sample.small_table`
  CROSS JOIN UNNEST(event_params) AS param
WHERE param.key IN ('page_location') AND
event_name IN ("page_view","purchase"))
SELECT
	lt.event_name,
    lt.event_timestamp,
    lt.user_pseudo_id,
    lt.kkey,
    lt.string_value,
    rt.event_timestamp AS first_conversion
FROM unnest_table lt
JOIN (SELECT user_pseudo_id, MIN(event_timestamp) AS event_timestamp FROM unnest_table WHERE event_name = 'purchase' GROUP BY user_pseudo_id) rt 
ON 
rt.user_pseudo_id = lt.user_pseudo_id
WHERE lt.event_name = "page_view" AND lt.event_timestamp <= rt.event_timestamp)
SELECT visited_page, count(visited_page) AS converted_visits FROM
(SELECT DISTINCT user_pseudo_id, string_value AS visited_page FROM final_table) a
GROUP BY visited_page
ORDER BY converted_visits DESC
;
```


## Final Table

Now we need only to join the non-converted visits and the converted visits tables

| visited_page                                                                                           | non_converted_visits | converted_visits | conversion_rate_percent |
|--------------------------------------------------------------------------------------------------------|--------------------|-----------------|------------------------|
| https://shop.googlemerchandisestore.com/payment.html                                                  | 24                 | 99              | 80.49                  |
| https://shop.googlemerchandisestore.com/yourinfo.html                                                 | 122                | 99              | 44.8                   |
| https://shop.googlemerchandisestore.com/registersuccess.html                                         | 124                | 33              | 21.02                  |
| https://shop.googlemerchandisestore.com/Google+Redesign/Stationery/Writing                            | 83                 | 20              | 19.42                  |
| https://shop.googlemerchandisestore.com/basket.html                                                   | 431                | 99              | 18.68                  |
| https://shop.googlemerchandisestore.com/signin.html                                                  | 395                | 87              | 18.05                  |
| https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Socks                                 | 119                | 24              | 16.78                  |
| https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Womens                                 | 169                | 32              | 15.92                  |
| https://shop.googlemerchandisestore.com/Google+Redesign/Lifestyle/Small+Goods                          | 179                | 32              | 15.17                  |
| https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Kids                                   | 161                | 25              | 13.44                  |


### SQL Query

```sql
WITH non_converted_visits AS (
  SELECT
    string_value AS visited_page,
    COUNT(*) AS non_converted_visits
  FROM (
    SELECT DISTINCT
      user_pseudo_id,
      param.value.string_value
    FROM `future-abacus-432804-b6.ecom_sample.small_table`
    CROSS JOIN UNNEST(event_params) AS param
    WHERE param.key = 'page_location'
      AND event_name = "page_view"
      AND user_pseudo_id NOT IN (
        SELECT user_pseudo_id
        FROM `future-abacus-432804-b6.ecom_sample.small_table`
        WHERE event_name = "purchase"
      )
  )
  GROUP BY visited_page
),

converted_visits AS (
  WITH unnest_table AS (
    SELECT
      event_name,
      event_timestamp,
      user_pseudo_id,
      param.value.string_value AS visited_page
    FROM `future-abacus-432804-b6.ecom_sample.small_table`
    CROSS JOIN UNNEST(event_params) AS param
    WHERE param.key = 'page_location'
      AND event_name IN ("page_view","purchase")
  ),
  final_table AS (
    SELECT
      lt.user_pseudo_id,
      lt.visited_page
    FROM unnest_table lt
    JOIN (
      SELECT
        user_pseudo_id,
        MIN(event_timestamp) AS first_conversion
      FROM unnest_table
      WHERE event_name = 'purchase'
      GROUP BY user_pseudo_id
    ) conv
    ON lt.user_pseudo_id = conv.user_pseudo_id
    WHERE lt.event_name = "page_view"
      AND lt.event_timestamp <= conv.first_conversion
  )
  SELECT
    visited_page,
    COUNT(*) AS converted_visits
  FROM (
    SELECT DISTINCT user_pseudo_id, visited_page FROM final_table
  )
  GROUP BY visited_page
)

SELECT
  COALESCE(n.visited_page, c.visited_page) AS visited_page,
  COALESCE(n.non_converted_visits, 0) AS non_converted_visits,
  COALESCE(c.converted_visits, 0) AS converted_visits,
  ROUND(
    COALESCE(c.converted_visits, 0) * 100.0 / 
    NULLIF(COALESCE(c.converted_visits, 0) + COALESCE(n.non_converted_visits, 0), 0),
    2
  ) AS conversion_rate_percent
FROM non_converted_visits n
FULL OUTER JOIN converted_visits c
  ON n.visited_page = c.visited_page
ORDER BY (COALESCE(n.non_converted_visits, 0) + COALESCE(c.converted_visits, 0)) DESC;
```

## Next steps

Based on the data from the website, we can improve our measurements with the following actions:
1. Apply data-driven attribution, so we can group the customer journey by web pages and find the most efficient and the least efficient pages that affect the journey
2. Apply values to the visited page, like time spent on the specific landing page, to see engaged pages and temporary visits
3. Apply some engagement actions on every page to identify converted elements (E.g. clicked "Subscribe Now")
