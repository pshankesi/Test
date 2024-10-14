# Customer Data Platform (CDP) Schema Creation Process

## Step 1: Create Temporary Tables

To create a comprehensive CDP schema, we'll first create several temporary tables to process and aggregate data from our source tables. These temporary tables will help us perform intermediate calculations and transformations.

### 1.1 temp_customer_basic

Purpose: Store basic customer information and demographics.

```sql
CREATE TEMPORARY TABLE temp_customer_basic AS
SELECT
    ci.customer_id,
    ci.first_name,
    ci.last_name,
    ci.email,
    ci.phone_number,
    ci.date_of_birth,
    ci.registration_date,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, ci.date_of_birth)) AS age,
    CURRENT_DATE - ci.registration_date AS days_since_registration
FROM
    customer_info ci;
```

### 1.2 temp_purchase_stats

Purpose: Calculate purchase-related statistics for each customer.

```sql
CREATE TEMPORARY TABLE temp_purchase_stats AS
SELECT
    pt.customer_id,
    COUNT(DISTINCT pt.transaction_id) AS total_purchases,
    SUM(pt.total_amount) AS total_spend,
    AVG(pt.total_amount) AS avg_order_value,
    MAX(pt.purchase_date) AS last_purchase_date,
    CURRENT_DATE - MAX(pt.purchase_date) AS days_since_last_purchase
FROM
    purchase_transactions pt
GROUP BY
    pt.customer_id;
```

### 1.3 temp_product_preferences

Purpose: Determine customer's favorite product category and brand.

```sql
CREATE TEMPORARY TABLE temp_product_preferences AS
WITH product_counts AS (
    SELECT
        pt.customer_id,
        pc.category,
        pc.brand,
        COUNT(*) AS purchase_count,
        ROW_NUMBER() OVER (PARTITION BY pt.customer_id ORDER BY COUNT(*) DESC) AS rank
    FROM
        purchase_transactions pt
    JOIN
        product_catalog pc ON pt.product_id = pc.product_id
    GROUP BY
        pt.customer_id, pc.category, pc.brand
)
SELECT
    customer_id,
    MAX(CASE WHEN rank = 1 THEN category END) AS favorite_category,
    MAX(CASE WHEN rank = 1 THEN brand END) AS favorite_brand
FROM
    product_counts
GROUP BY
    customer_id;
```

### 1.4 temp_customer_service_stats

Purpose: Aggregate customer service interaction data.

```sql
CREATE TEMPORARY TABLE temp_customer_service_stats AS
SELECT
    customer_id,
    COUNT(*) AS total_interactions,
    AVG(satisfaction_score) AS avg_satisfaction_score,
    SUM(CASE WHEN resolution_status = 'Resolved' THEN 1 ELSE 0 END) AS resolved_interactions
FROM
    customer_service
GROUP BY
    customer_id;
```

### 1.5 temp_campaign_engagement

Purpose: Calculate customer engagement with marketing campaigns.

```sql
CREATE TEMPORARY TABLE temp_campaign_engagement AS
SELECT
    cr.customer_id,
    COUNT(DISTINCT cr.campaign_id) AS campaigns_engaged,
    SUM(CASE WHEN cr.response_type = 'Conversion' THEN 1 ELSE 0 END) AS campaign_conversions
FROM
    campaign_responses cr
GROUP BY
    cr.customer_id;
```

### 1.6 temp_website_behavior

Purpose: Aggregate website behavior data for each customer.

```sql
CREATE TEMPORARY TABLE temp_website_behavior AS
SELECT
    customer_id,
    COUNT(*) AS total_sessions,
    AVG(pages_viewed) AS avg_pages_per_session,
    AVG(time_spent) AS avg_time_per_session,
    MAX(visit_date) AS last_visit_date
FROM
    website_behavior
GROUP BY
    customer_id;
```

## Step 2: Create the Customer 360 View

Now that we have our temporary tables with aggregated and transformed data, we can create the final customer_360 table by joining these temporary tables and adding derived fields.

```sql
CREATE TABLE customer_360 AS
SELECT
    cb.customer_id,
    cb.first_name,
    cb.last_name,
    cb.email,
    cb.phone_number,
    cb.date_of_birth,
    cb.age,
    cb.registration_date,
    cb.days_since_registration,
    ps.total_purchases,
    ps.total_spend,
    ps.avg_order_value,
    ps.last_purchase_date,
    ps.days_since_last_purchase,
    pp.favorite_category,
    pp.favorite_brand,
    css.total_interactions AS customer_service_interactions,
    css.avg_satisfaction_score,
    css.resolved_interactions,
    ce.campaigns_engaged,
    ce.campaign_conversions,
    wb.total_sessions,
    wb.avg_pages_per_session,
    wb.avg_time_per_session,
    wb.last_visit_date,
    -- Derived fields
    CASE
        WHEN ps.total_spend > 1000 AND ps.total_purchases > 10 THEN 'High Value'
        WHEN ps.total_spend > 500 OR ps.total_purchases > 5 THEN 'Medium Value'
        ELSE 'Low Value'
    END AS customer_segment,
    CASE
        WHEN ps.days_since_last_purchase <= 30 THEN 'Active'
        WHEN ps.days_since_last_purchase <= 90 THEN 'At Risk'
        ELSE 'Churned'
    END AS customer_status,
    (ps.total_spend / NULLIF(cb.days_since_registration, 0)) * 30 AS monthly_avg_spend,
    (ce.campaign_conversions::FLOAT / NULLIF(ce.campaigns_engaged, 0)) * 100 AS campaign_conversion_rate,
    (css.resolved_interactions::FLOAT / NULLIF(css.total_interactions, 0)) * 100 AS service_resolution_rate,
    COALESCE(wb.total_sessions, 0) + COALESCE(ps.total_purchases, 0) + COALESCE(css.total_interactions, 0) AS total_touchpoints
FROM
    temp_customer_basic cb
LEFT JOIN
    temp_purchase_stats ps ON cb.customer_id = ps.customer_id
LEFT JOIN
    temp_product_preferences pp ON cb.customer_id = pp.customer_id
LEFT JOIN
    temp_customer_service_stats css ON cb.customer_id = css.customer_id
LEFT JOIN
    temp_campaign_engagement ce ON cb.customer_id = ce.customer_id
LEFT JOIN
    temp_website_behavior wb ON cb.customer_id = wb.customer_id;
```

## Final Customer 360 Table Schema

Here's the final customer_360 table schema in JSON format:

```json
{
  "customer_360": {
    "customer_id": "INTEGER",
    "first_name": "TEXT",
    "last_name": "TEXT",
    "email": "TEXT",
    "phone_number": "TEXT",
    "date_of_birth": "DATE",
    "age": "INTEGER",
    "registration_date": "DATE",
    "days_since_registration": "INTEGER",
    "total_purchases": "INTEGER",
    "total_spend": "REAL",
    "avg_order_value": "REAL",
    "last_purchase_date": "DATE",
    "days_since_last_purchase": "INTEGER",
    "favorite_category": "TEXT",
    "favorite_brand": "TEXT",
    "customer_service_interactions": "INTEGER",
    "avg_satisfaction_score": "REAL",
    "resolved_interactions": "INTEGER",
    "campaigns_engaged": "INTEGER",
    "campaign_conversions": "INTEGER",
    "total_sessions": "INTEGER",
    "avg_pages_per_session": "REAL",
    "avg_time_per_session": "INTEGER",
    "last_visit_date": "DATE",
    "customer_segment": "TEXT",
    "customer_status": "TEXT",
    "monthly_avg_spend": "REAL",
    "campaign_conversion_rate": "REAL",
    "service_resolution_rate": "REAL",
    "total_touchpoints": "INTEGER"
  }
}
```

## Explanation of Derived Fields

1. **customer_segment**: Categorizes customers based on their total spend and number of purchases.
2. **customer_status**: Determines the customer's activity status based on their last purchase date.
3. **monthly_avg_spend**: Calculates the average monthly spend by dividing total spend by the number of months since registration.
4. **campaign_conversion_rate**: Percentage of campaigns that resulted in a conversion.
5. **service_resolution_rate**: Percentage of customer service interactions that were resolved.
6. **total_touchpoints**: Sum of all customer interactions across purchases, website visits, and customer service.

## Incorporating Developer Insights

Based on the developer conversations, we've incorporated the following aspects into our CDP schema:

1. **Data Integration**: We've used GCP Cloud SQL with stored procedures (temporary tables) to consolidate data from various sources.
2. **Real-time Processing**: While our current schema is batch-based, it can be easily adapted for real-time updates by modifying the temporary table creation process.
3. **Customer Profile Unification**: We've created a single view of the customer by joining data from multiple sources.
4. **Segmentation and Analytics**: We've included derived fields for customer segmentation and basic analytics.
5. **Data Activation**: The schema includes fields that can be used for personalization and marketing automation.
6. **Scalability**: By using temporary tables and efficient joins, we've designed the schema to handle large datasets.
7. **Data Quality and Consistency**: We've used LEFT JOINs to ensure all customers are included, even if they don't have data in all tables.

This CDP schema provides a comprehensive view of each customer, incorporating their basic information, purchase history, product preferences, customer service interactions, campaign engagement, and website behavior. It also includes derived fields for segmentation and analysis, making it a powerful tool for personalized marketing and customer relationship management.