# Customer Data Platform (CDP) Schema Creation Process

## Step 1: Analyze Source Tables and Create Temporary Tables

Based on the provided DDL scripts and developer insights, we'll create the following temporary tables:

### 1.1 temp_customer_basic
Purpose: Store basic customer information
```sql
CREATE TEMPORARY TABLE temp_customer_basic AS
SELECT 
    customer_id,
    first_name,
    last_name,
    email,
    phone_number,
    date_of_birth,
    registration_date,
    DATEDIFF(YEAR, date_of_birth, CURRENT_DATE) AS age,
    DATEDIFF(DAY, registration_date, CURRENT_DATE) AS customer_tenure_days
FROM customer_info;
```

### 1.2 temp_purchase_stats
Purpose: Calculate purchase-related statistics
```sql
CREATE TEMPORARY TABLE temp_purchase_stats AS
SELECT 
    customer_id,
    COUNT(DISTINCT transaction_id) AS total_orders,
    SUM(total_amount) AS total_lifetime_value,
    AVG(total_amount) AS average_order_value,
    MAX(purchase_date) AS last_purchase_date,
    DATEDIFF(DAY, MAX(purchase_date), CURRENT_DATE) AS days_since_last_purchase
FROM purchase_transactions
GROUP BY customer_id;
```

### 1.3 temp_product_preferences
Purpose: Determine favorite product categories and brands
```sql
CREATE TEMPORARY TABLE temp_product_preferences AS
SELECT 
    pt.customer_id,
    pc.category AS favorite_category,
    pc.brand AS favorite_brand
FROM purchase_transactions pt
JOIN product_catalog pc ON pt.product_id = pc.product_id
GROUP BY pt.customer_id, pc.category, pc.brand
HAVING COUNT(*) = (
    SELECT COUNT(*) 
    FROM purchase_transactions pt2
    JOIN product_catalog pc2 ON pt2.product_id = pc2.product_id
    WHERE pt2.customer_id = pt.customer_id
    GROUP BY pc2.category, pc2.brand
    ORDER BY COUNT(*) DESC
    LIMIT 1
);
```

### 1.4 temp_customer_service
Purpose: Summarize customer service interactions
```sql
CREATE TEMPORARY TABLE temp_customer_service AS
SELECT 
    customer_id,
    COUNT(*) AS total_interactions,
    AVG(satisfaction_score) AS avg_satisfaction_score,
    SUM(CASE WHEN resolution_status = 'Resolved' THEN 1 ELSE 0 END) AS resolved_interactions
FROM customer_service
GROUP BY customer_id;
```

### 1.5 temp_marketing_engagement
Purpose: Analyze marketing campaign engagement
```sql
CREATE TEMPORARY TABLE temp_marketing_engagement AS
SELECT 
    cr.customer_id,
    COUNT(DISTINCT cr.campaign_id) AS campaigns_engaged,
    SUM(CASE WHEN cr.response_type = 'Conversion' THEN 1 ELSE 0 END) AS campaign_conversions
FROM campaign_responses cr
JOIN marketing_campaigns mc ON cr.campaign_id = mc.campaign_id
GROUP BY cr.customer_id;
```

### 1.6 temp_website_behavior
Purpose: Summarize website behavior
```sql
CREATE TEMPORARY TABLE temp_website_behavior AS
SELECT 
    customer_id,
    COUNT(*) AS total_sessions,
    AVG(pages_viewed) AS avg_pages_per_session,
    AVG(time_spent) AS avg_time_per_session,
    MAX(visit_date) AS last_visit_date
FROM website_behavior
GROUP BY customer_id;
```

## Step 2: Create Customer 360 View

Join the temporary tables to create a comprehensive customer profile:

```sql
CREATE TABLE customer_360 AS
SELECT 
    cb.*,
    ps.total_orders,
    ps.total_lifetime_value,
    ps.average_order_value,
    ps.last_purchase_date,
    ps.days_since_last_purchase,
    pp.favorite_category,
    pp.favorite_brand,
    cs.total_interactions,
    cs.avg_satisfaction_score,
    cs.resolved_interactions,
    me.campaigns_engaged,
    me.campaign_conversions,
    wb.total_sessions,
    wb.avg_pages_per_session,
    wb.avg_time_per_session,
    wb.last_visit_date
FROM temp_customer_basic cb
LEFT JOIN temp_purchase_stats ps ON cb.customer_id = ps.customer_id
LEFT JOIN temp_product_preferences pp ON cb.customer_id = pp.customer_id
LEFT JOIN temp_customer_service cs ON cb.customer_id = cs.customer_id
LEFT JOIN temp_marketing_engagement me ON cb.customer_id = me.customer_id
LEFT JOIN temp_website_behavior wb ON cb.customer_id = wb.customer_id;
```

## Step 3: Add Derived Fields

Add derived fields for customer segmentation and analysis:

```sql
ALTER TABLE customer_360
ADD COLUMN customer_segment VARCHAR(20),
ADD COLUMN churn_risk_score INTEGER,
ADD COLUMN customer_lifetime_value_score INTEGER,
ADD COLUMN engagement_score INTEGER;

UPDATE customer_360
SET 
    customer_segment = CASE 
        WHEN total_lifetime_value > 10000 THEN 'High Value'
        WHEN total_lifetime_value > 5000 THEN 'Medium Value'
        ELSE 'Low Value'
    END,
    churn_risk_score = CASE 
        WHEN days_since_last_purchase > 365 THEN 5
        WHEN days_since_last_purchase > 180 THEN 4
        WHEN days_since_last_purchase > 90 THEN 3
        WHEN days_since_last_purchase > 30 THEN 2
        ELSE 1
    END,
    customer_lifetime_value_score = NTILE(5) OVER (ORDER BY total_lifetime_value),
    engagement_score = (
        COALESCE(campaigns_engaged, 0) + 
        COALESCE(total_sessions, 0) + 
        COALESCE(total_interactions, 0)
    ) / 3;
```

## Final Customer 360 Table Schema

```json
{
  "customer_360": {
    "customer_id": "INTEGER",
    "first_name": "TEXT",
    "last_name": "TEXT",
    "email": "TEXT",
    "phone_number": "TEXT",
    "date_of_birth": "DATE",
    "registration_date": "DATE",
    "age": "INTEGER",
    "customer_tenure_days": "INTEGER",
    "total_orders": "INTEGER",
    "total_lifetime_value": "REAL",
    "average_order_value": "REAL",
    "last_purchase_date": "DATE",
    "days_since_last_purchase": "INTEGER",
    "favorite_category": "TEXT",
    "favorite_brand": "TEXT",
    "total_interactions": "INTEGER",
    "avg_satisfaction_score": "REAL",
    "resolved_interactions": "INTEGER",
    "campaigns_engaged": "INTEGER",
    "campaign_conversions": "INTEGER",
    "total_sessions": "INTEGER",
    "avg_pages_per_session": "REAL",
    "avg_time_per_session": "REAL",
    "last_visit_date": "DATE",
    "customer_segment": "VARCHAR(20)",
    "churn_risk_score": "INTEGER",
    "customer_lifetime_value_score": "INTEGER",
    "engagement_score": "INTEGER"
  }
}
```

## Explanation of Derived Fields

1. **customer_segment**: Categorizes customers based on their total lifetime value.
2. **churn_risk_score**: Assesses the risk of customer churn based on the days since their last purchase.
3. **customer_lifetime_value_score**: Ranks customers into quintiles based on their total lifetime value.
4. **engagement_score**: Measures overall customer engagement by combining campaign engagement, website sessions, and customer service interactions.

## Incorporating Developer Insights

1. **Data Integration**: We've used SQL to create temporary tables, which can be easily implemented in GCP Cloud SQL as suggested by the Cloud Architect.
2. **Scalability**: The schema is designed to handle large volumes of data and can be optimized for real-time processing as mentioned by the Backend Developer.
3. **Real-time Updates**: The schema supports real-time updates to customer profiles, aligning with the Backend Developer's suggestion for event-driven architecture.
4. **Personalization**: The inclusion of favorite categories and brands supports the development of recommendation systems, as suggested by the Cloud Architect.
5. **Data Visualization**: The schema provides various metrics that can be visualized using Plotly.js, as recommended by the UI Developer.
6. **Customer Journey**: The inclusion of various touchpoints (purchases, customer service, marketing, website behavior) supports customer journey mapping using Mermaid.js, as suggested by the UI Developer.

This CDP schema provides a comprehensive view of customer data, incorporating insights from various sources and enabling advanced analytics and personalization capabilities for the e-commerce company.