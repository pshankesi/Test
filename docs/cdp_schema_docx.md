# CDP Schema Creation Process

## Step 1: Create Temporary Tables

To create a comprehensive CDP schema, we'll first create several temporary tables to aggregate and transform data from our source tables. Here are the temporary tables we'll use:

### 1. temp_customer_basic
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
    DATEDIFF(DAY, registration_date, CURRENT_DATE) AS days_since_registration
FROM customer_info;
```

### 2. temp_purchase_stats
Purpose: Calculate purchase-related statistics
```sql
CREATE TEMPORARY TABLE temp_purchase_stats AS
SELECT 
    customer_id,
    COUNT(DISTINCT transaction_id) AS total_purchases,
    SUM(total_amount) AS total_spend,
    AVG(total_amount) AS avg_purchase_value,
    MAX(purchase_date) AS last_purchase_date,
    DATEDIFF(DAY, MAX(purchase_date), CURRENT_DATE) AS days_since_last_purchase
FROM purchase_transactions
GROUP BY customer_id;
```

### 3. temp_product_preferences
Purpose: Determine favorite product categories and brands
```sql
CREATE TEMPORARY TABLE temp_product_preferences AS
SELECT 
    pt.customer_id,
    pc.category AS favorite_category,
    pc.brand AS favorite_brand
FROM purchase_transactions pt
JOIN product_catalog pc ON pt.product_id = pc.product_id
GROUP BY pt.customer_id
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

### 4. temp_customer_service_stats
Purpose: Aggregate customer service interaction data
```sql
CREATE TEMPORARY TABLE temp_customer_service_stats AS
SELECT 
    customer_id,
    COUNT(*) AS total_interactions,
    AVG(satisfaction_score) AS avg_satisfaction_score,
    SUM(CASE WHEN resolution_status = 'Resolved' THEN 1 ELSE 0 END) AS resolved_interactions
FROM customer_service
GROUP BY customer_id;
```

### 5. temp_campaign_engagement
Purpose: Calculate campaign engagement metrics
```sql
CREATE TEMPORARY TABLE temp_campaign_engagement AS
SELECT 
    cr.customer_id,
    COUNT(DISTINCT cr.campaign_id) AS campaigns_engaged,
    SUM(CASE WHEN cr.response_type = 'Positive' THEN 1 ELSE 0 END) AS positive_responses
FROM campaign_responses cr
JOIN marketing_campaigns mc ON cr.campaign_id = mc.campaign_id
GROUP BY cr.customer_id;
```

### 6. temp_website_behavior
Purpose: Aggregate website behavior data
```sql
CREATE TEMPORARY TABLE temp_website_behavior AS
SELECT 
    customer_id,
    COUNT(*) AS total_sessions,
    AVG(pages_viewed) AS avg_pages_per_session,
    AVG(time_spent) AS avg_time_per_session
FROM website_behavior
GROUP BY customer_id;
```

## Step 2: Create the Customer 360 Table

Now that we have our temporary tables, we can create the final customer_360 table by joining these tables and adding derived fields:

```sql
CREATE TABLE customer_360 AS
SELECT 
    cb.*,
    ps.total_purchases,
    ps.total_spend,
    ps.avg_purchase_value,
    ps.last_purchase_date,
    ps.days_since_last_purchase,
    pp.favorite_category,
    pp.favorite_brand,
    css.total_interactions,
    css.avg_satisfaction_score,
    css.resolved_interactions,
    ce.campaigns_engaged,
    ce.positive_responses,
    wb.total_sessions,
    wb.avg_pages_per_session,
    wb.avg_time_per_session,
    -- Derived fields
    CASE 
        WHEN ps.total_spend > 1000 THEN 'High'
        WHEN ps.total_spend > 500 THEN 'Medium'
        ELSE 'Low'
    END AS customer_value_segment,
    CASE 
        WHEN ps.days_since_last_purchase <= 30 THEN 'Active'
        WHEN ps.days_since_last_purchase <= 90 THEN 'At Risk'
        ELSE 'Churned'
    END AS customer_status,
    (css.resolved_interactions * 1.0 / NULLIF(css.total_interactions, 0)) AS issue_resolution_rate,
    (ce.positive_responses * 1.0 / NULLIF(ce.campaigns_engaged, 0)) AS campaign_response_rate,
    (wb.total_sessions * 1.0 / NULLIF(cb.days_since_registration, 0)) AS website_engagement_rate
FROM temp_customer_basic cb
LEFT JOIN temp_purchase_stats ps ON cb.customer_id = ps.customer_id
LEFT JOIN temp_product_preferences pp ON cb.customer_id = pp.customer_id
LEFT JOIN temp_customer_service_stats css ON cb.customer_id = css.customer_id
LEFT JOIN temp_campaign_engagement ce ON cb.customer_id = ce.customer_id
LEFT JOIN temp_website_behavior wb ON cb.customer_id = wb.customer_id;
```

## Step 3: Incorporate Developer Insights

Based on the developer conversations, we should consider the following enhancements:

1. Implement proper data security measures, including encryption and access controls.
2. Ensure scalability by setting up auto-scaling policies and optimizing query performance.
3. Implement comprehensive monitoring and observability using tools like Prometheus and Grafana.
4. Set up a robust CI/CD pipeline for smooth updates to the CDP.
5. Implement role-based access control (RBAC) for data governance.
6. Use WebGL-based libraries for handling large datasets in the UI.
7. Implement accessibility features following WCAG guidelines.
8. Use virtual scrolling for efficient rendering of large data lists in the UI.

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
    "registration_date": "DATE",
    "age": "INTEGER",
    "days_since_registration": "INTEGER",
    "total_purchases": "INTEGER",
    "total_spend": "REAL",
    "avg_purchase_value": "REAL",
    "last_purchase_date": "DATE",
    "days_since_last_purchase": "INTEGER",
    "favorite_category": "TEXT",
    "favorite_brand": "TEXT",
    "total_interactions": "INTEGER",
    "avg_satisfaction_score": "REAL",
    "resolved_interactions": "INTEGER",
    "campaigns_engaged": "INTEGER",
    "positive_responses": "INTEGER",
    "total_sessions": "INTEGER",
    "avg_pages_per_session": "REAL",
    "avg_time_per_session": "REAL",
    "customer_value_segment": "TEXT",
    "customer_status": "TEXT",
    "issue_resolution_rate": "REAL",
    "campaign_response_rate": "REAL",
    "website_engagement_rate": "REAL"
  }
}
```

## Explanation of Derived Fields

1. `customer_value_segment`: Categorizes customers based on their total spend.
2. `customer_status`: Determines if a customer is active, at risk, or churned based on their last purchase date.
3. `issue_resolution_rate`: Calculates the percentage of customer service interactions that were resolved.
4. `campaign_response_rate`: Measures the effectiveness of marketing campaigns by calculating the ratio of positive responses to total campaigns engaged.
5. `website_engagement_rate`: Indicates how frequently a customer visits the website based on their total sessions and days since registration.

This CDP schema provides a comprehensive view of each customer, incorporating data from various sources and including derived fields for deeper customer analysis and segmentation. The schema can be further optimized based on specific business needs and performance requirements.