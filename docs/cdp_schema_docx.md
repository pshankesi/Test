# CDP Schema Creation Process

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
    registration_date
FROM customer_info;
```

### 1.2 temp_purchase_stats
Purpose: Calculate purchase-related statistics
```sql
CREATE TEMPORARY TABLE temp_purchase_stats AS
SELECT 
    customer_id,
    COUNT(DISTINCT transaction_id) AS total_purchases,
    SUM(total_amount) AS total_spend,
    AVG(total_amount) AS avg_order_value,
    MAX(purchase_date) AS last_purchase_date
FROM purchase_transactions
GROUP BY customer_id;
```

### 1.3 temp_product_preferences
Purpose: Determine favorite product category and brand
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
    WHERE pt2.customer_id = pt.customer_id 
    GROUP BY pt2.product_id 
    ORDER BY COUNT(*) DESC 
    LIMIT 1
);
```

### 1.4 temp_customer_service_stats
Purpose: Aggregate customer service interactions
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

### 1.5 temp_campaign_engagement
Purpose: Summarize marketing campaign engagement
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

### 1.6 temp_website_behavior
Purpose: Aggregate website behavior metrics
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

## Step 2: Create the Customer 360 View

Now, we'll join these temporary tables to create a comprehensive customer profile:

```sql
CREATE TABLE customer_360 AS
SELECT 
    cb.*,
    ps.total_purchases,
    ps.total_spend,
    ps.avg_order_value,
    ps.last_purchase_date,
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
    wb.last_visit_date,
    -- Derived fields
    DATEDIFF(CURRENT_DATE, ps.last_purchase_date) AS days_since_last_purchase,
    DATEDIFF(CURRENT_DATE, wb.last_visit_date) AS days_since_last_visit,
    (ps.total_spend / NULLIF(DATEDIFF(CURRENT_DATE, cb.registration_date), 0)) * 365 AS annual_spend,
    (css.resolved_interactions::FLOAT / NULLIF(css.total_interactions, 0)) * 100 AS resolution_rate,
    (ce.positive_responses::FLOAT / NULLIF(ce.campaigns_engaged, 0)) * 100 AS campaign_response_rate
FROM temp_customer_basic cb
LEFT JOIN temp_purchase_stats ps ON cb.customer_id = ps.customer_id
LEFT JOIN temp_product_preferences pp ON cb.customer_id = pp.customer_id
LEFT JOIN temp_customer_service_stats css ON cb.customer_id = css.customer_id
LEFT JOIN temp_campaign_engagement ce ON cb.customer_id = ce.customer_id
LEFT JOIN temp_website_behavior wb ON cb.customer_id = wb.customer_id;
```

## Step 3: Add Derived Fields for Customer Analysis and Segmentation

Based on the developer insights, we'll add more derived fields for advanced analysis:

```sql
ALTER TABLE customer_360
ADD COLUMN customer_lifetime_value DECIMAL(12,2),
ADD COLUMN customer_segment VARCHAR(20),
ADD COLUMN churn_risk_score INTEGER,
ADD COLUMN engagement_score INTEGER;

UPDATE customer_360
SET 
    customer_lifetime_value = total_spend * (1 + (campaign_response_rate / 100)),
    customer_segment = CASE 
        WHEN total_spend > 10000 THEN 'High Value'
        WHEN total_spend > 5000 THEN 'Medium Value'
        ELSE 'Low Value'
    END,
    churn_risk_score = CASE 
        WHEN days_since_last_purchase > 365 THEN 5
        WHEN days_since_last_purchase > 180 THEN 4
        WHEN days_since_last_purchase > 90 THEN 3
        WHEN days_since_last_purchase > 30 THEN 2
        ELSE 1
    END,
    engagement_score = (
        LEAST(total_purchases, 10) * 10 +
        LEAST(total_sessions, 10) * 5 +
        LEAST(campaigns_engaged, 5) * 10 +
        LEAST(CAST(avg_satisfaction_score AS INTEGER), 5) * 10
    );
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
    "registration_date": "DATE",
    "total_purchases": "INTEGER",
    "total_spend": "REAL",
    "avg_order_value": "REAL",
    "last_purchase_date": "DATE",
    "favorite_category": "TEXT",
    "favorite_brand": "TEXT",
    "total_interactions": "INTEGER",
    "avg_satisfaction_score": "REAL",
    "resolved_interactions": "INTEGER",
    "campaigns_engaged": "INTEGER",
    "positive_responses": "INTEGER",
    "total_sessions": "INTEGER",
    "avg_pages_per_session": "REAL",
    "avg_time_per_session": "INTEGER",
    "last_visit_date": "DATE",
    "days_since_last_purchase": "INTEGER",
    "days_since_last_visit": "INTEGER",
    "annual_spend": "REAL",
    "resolution_rate": "REAL",
    "campaign_response_rate": "REAL",
    "customer_lifetime_value": "DECIMAL(12,2)",
    "customer_segment": "VARCHAR(20)",
    "churn_risk_score": "INTEGER",
    "engagement_score": "INTEGER"
  }
}
```

## Explanations for Derived Fields

1. **days_since_last_purchase**: Calculates the number of days since the customer's last purchase, useful for identifying potentially churning customers.

2. **days_since_last_visit**: Calculates the number of days since the customer's last website visit, helping to gauge recent engagement.

3. **annual_spend**: Estimates the customer's yearly spend based on their total spend and the duration of their relationship with the company.

4. **resolution_rate**: Percentage of customer service interactions that were successfully resolved, indicating service quality.

5. **campaign_response_rate**: Percentage of marketing campaigns the customer positively responded to, measuring marketing effectiveness.

6. **customer_lifetime_value**: Calculated based on total spend and campaign response rate, providing an estimate of the customer's long-term value to the company.

7. **customer_segment**: Categorizes customers into value segments based on their total spend.

8. **churn_risk_score**: Assigns a risk score based on the recency of purchases, with higher scores indicating higher churn risk.

9. **engagement_score**: A composite score based on purchases, website sessions, campaign engagement, and satisfaction scores, providing an overall measure of customer engagement.

These derived fields incorporate insights from the developer conversations, particularly focusing on customer segmentation, churn risk assessment, and engagement measurement. The schema is designed to provide a comprehensive view of each customer, enabling advanced analytics and personalized marketing strategies.