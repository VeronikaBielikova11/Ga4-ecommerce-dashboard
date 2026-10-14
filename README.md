# GA4 Ecommerce Dashboard

## Overview

This project focuses on analyzing e-commerce user behavior, conversion funnels, landing pages, and marketing campaign performance using Google Analytics 4 (GA4) data and SQL.

The data was transformed with SQL to create a session-level dataset and visualised in an interactive Tableau dashboard. The dashboard enables marketing teams to monitor key ecommerce KPIs, analyse the conversion funnel, and explore performance across different traffic sources and user segments.

## Tech Stack

- BigQuery SQL
- Google Analytics 4
- Tableau Public

## Data Preparation

**Source:** Google Analytics 4 Public Ecommerce Dataset (BigQuery)

After executing the SQL query, a session-level table was created containing the following information:

- User and session identifiers
- Session date and timestamp
- Funnel event (Session Start → Purchase)
- Landing page
- Traffic source, medium and campaign
- Country
- Device category
- Device language
- Operating system

## Dashboard Features

- KPI cards (Visits, Orders, Conversion Rate)
- Ecommerce conversion funnel
- Sessions by device category
- Sessions by device language
- Sessions by country
- Operating system distribution
- Weekly conversion trend

## Interactive Filters

- Session date
- Landing page
- Source
- Medium
- Campaign
- Device category
- Device language
- Operating system

## Completed Work

### Task 1. E-commerce Funnel Dataset

Created a SQL query to combine session information with key e-commerce events, including product views, cart additions, checkout steps, and purchases.

The query extracts user and session identifiers, landing pages, traffic sources, campaigns, countries, and device attributes to support e-commerce funnel analysis.

```sql
WITH
  sessions_info AS (
    SELECT
      user_pseudo_id,
      (
        SELECT value.int_value
        FROM UNNEST(event_params)
        WHERE key = 'ga_session_id'
        LIMIT 1
      ) AS session_id,
      CONCAT(
        user_pseudo_id,
        CAST(
          (
            SELECT value.int_value
            FROM UNNEST(event_params)
            WHERE key = 'ga_session_id'
            LIMIT 1
          ) AS STRING
        )
      ) AS user_session_id,
      REGEXP_EXTRACT(
        (
          SELECT value.string_value
          FROM UNNEST(event_params)
          WHERE key = 'page_location'
          LIMIT 1
        ),
        r'(?:https:\/\/)?[^\/]+\/(.*)'
      ) AS landing_page_location,
      traffic_source.source,
      traffic_source.medium,
      traffic_source.name AS campaign,
      geo.country,
      device.category AS device_category,
      device.language AS device_language,
      device.operating_system
    FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
    WHERE event_name = 'session_start'
  ),
  events AS (
    SELECT
      event_date,
      TIMESTAMP_MICROS(event_timestamp) AS event_timestamp,
      event_name,
      user_pseudo_id,
      (
        SELECT value.int_value
        FROM UNNEST(event_params)
        WHERE key = 'ga_session_id'
        LIMIT 1
      ) AS session_id,
      CONCAT(
        user_pseudo_id,
        CAST(
          (
            SELECT value.int_value
            FROM UNNEST(event_params)
            WHERE key = 'ga_session_id'
            LIMIT 1
          ) AS STRING
        )
      ) AS user_session_id
    FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
    WHERE event_name IN (
      'session_start',
      'view_item',
      'add_to_cart',
      'begin_checkout',
      'add_shipping_info',
      'add_payment_info',
      'purchase'
    )
  )
SELECT
  s.user_pseudo_id,
  s.session_id,
  s.user_session_id,
  e.event_date,
  e.event_timestamp,
  e.event_name,
  s.landing_page_location,
  s.source,
  s.medium,
  s.campaign,
  s.country,
  s.device_category,
  s.device_language,
  s.operating_system
FROM sessions_info s
LEFT JOIN events e
  ON s.user_session_id = e.user_session_id;
```

### Task 2. GA4 Event-Level Data Extraction

Extracted GA4 event-level data using SQL to analyze user activity across key e-commerce events. The query includes session identifiers, timestamps, countries, device categories, traffic sources, and campaign information for January 2021.

```sql
SELECT
  TIMESTAMP_MICROS(event_timestamp) AS event_timestamp,
  user_pseudo_id,
  (
    SELECT value.int_value
    FROM UNNEST(event_params)
    WHERE key = 'ga_session_id'
  ) AS session_id,
  geo.country AS country,
  device.category AS device_category,
  traffic_source.medium AS medium,
  traffic_source.name AS campaign,
  traffic_source.source AS source,
  event_name
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20210101' AND '20210131'
  AND event_name IN (
    'session_start',
    'view_item',
    'add_to_cart',
    'begin_checkout',
    'add_shipping_info',
    'add_payment_info',
    'purchase'
  )
LIMIT 100;
```

### Task 3. E-commerce Funnel Conversion Analysis

Analyzed e-commerce funnel performance across marketing campaigns and traffic sources. The query calculates the percentages of sessions that include cart additions, checkout initiations, and purchases.

The results are grouped by event date, campaign, source, and medium to support marketing performance analysis.

```sql
WITH
  concat_user_session_id AS (
    SELECT
      DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
      traffic_source.medium AS medium,
      traffic_source.name AS campaign,
      traffic_source.source AS source,
      CONCAT(
        user_pseudo_id,
        '-',
        (
          SELECT value.int_value
          FROM UNNEST(event_params)
          WHERE key = 'ga_session_id'
        )
      ) AS user_session_id,
      event_name
    FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_20210131`
  ),
  count_events AS (
    SELECT
      event_date,
      campaign,
      source,
      medium,
      COUNT(DISTINCT CASE
        WHEN event_name = 'session_start' THEN user_session_id
      END) AS user_sessions_count,
      COUNT(DISTINCT CASE
        WHEN event_name = 'add_to_cart' THEN user_session_id
      END) AS visit_to_cart,
      COUNT(DISTINCT CASE
        WHEN event_name = 'begin_checkout' THEN user_session_id
      END) AS visit_to_checkout,
      COUNT(DISTINCT CASE
        WHEN event_name = 'purchase' THEN user_session_id
      END) AS visit_to_purchase
    FROM concat_user_session_id
    GROUP BY event_date, campaign, source, medium
  )
SELECT
  event_date,
  campaign,
  source,
  medium,
  ROUND(
    SAFE_DIVIDE(visit_to_cart, user_sessions_count) * 100,
    4
  ) AS visit_to_cart,
  ROUND(
    SAFE_DIVIDE(visit_to_checkout, user_sessions_count) * 100,
    4
  ) AS visit_to_checkout,
  ROUND(
    SAFE_DIVIDE(visit_to_purchase, user_sessions_count) * 100,
    4
  ) AS visit_to_purchase
FROM count_events;
```

### Task 4. Landing Page Performance and Conversion Analysis

Analyzed landing page performance by connecting session start events with purchase events. The query calculates unique sessions, purchases, and conversion rates for each landing page, helping identify pages associated with stronger purchase performance.

```sql
WITH
  ses_user_id AS (
    SELECT
      DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
      CONCAT(
        user_pseudo_id,
        '-',
        (
          SELECT value.int_value
          FROM UNNEST(event_params)
          WHERE key = 'ga_session_id'
        )
      ) AS user_session_id,
      (
        SELECT value.string_value
        FROM UNNEST(event_params)
        WHERE key = 'page_location'
      ) AS page_loc,
      event_name
    FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
    WHERE _TABLE_SUFFIX BETWEEN '20200101' AND '20201231'
  ),
  session_paths AS (
    SELECT
      user_session_id,
      REPLACE(
        page_loc,
        'https://shop.googlemerchandisestore.com/',
        ' '
      ) AS page_path
    FROM ses_user_id
    WHERE event_name = 'session_start'
  ),
  purchases AS (
    SELECT DISTINCT user_session_id
    FROM ses_user_id
    WHERE event_name = 'purchase'
  )
SELECT
  sp.page_path,
  COUNT(DISTINCT sp.user_session_id) AS unique_sessions,
  COUNT(DISTINCT p.user_session_id) AS purchases,
  ROUND(
    SAFE_DIVIDE(
      COUNT(DISTINCT p.user_session_id),
      COUNT(DISTINCT sp.user_session_id)
    ) * 100,
    2
  ) AS conversion_rate
FROM session_paths sp
LEFT JOIN purchases p
  ON sp.user_session_id = p.user_session_id
GROUP BY sp.page_path
ORDER BY purchases DESC;
```

## Dashboard Link

**Tableau Public:** [View Interactive Dashboard](https://public.tableau.com/views/E-commercePerfomance/E-commercePerfomance?:language=en-GB&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

## Dashboard Preview

<img width="1329" height="776" alt="Dashboard Preview" src="https://github.com/user-attachments/assets/33e1aaac-689c-463c-b222-7ea09123612c" />

## Key Skills Demonstrated

- SQL querying and data extraction
- Common Table Expressions (CTEs)
- GA4 event and session analysis
- Working with nested event parameters
- E-commerce funnel and conversion rate calculations
- Traffic source and campaign performance analysis
- Landing page performance analysis
- Data visualization with Tableau
