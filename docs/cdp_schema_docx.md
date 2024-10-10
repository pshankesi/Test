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
    SELECT MAX(purchase_count)
    FROM (
        SELECT customer_id, COUNT(*) AS purchase_count
        FROM purchase_transactions
        GROUP BY customer_id, product_id
    ) AS subquery
    WHERE subquery.customer_id = pt.customer_id
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
    AVG(time_spent) AS avg_time_per_session,
    MAX(visit_date) AS last_visit_date
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
    wb.last_visit_date,
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
    (ps.total_spend / NULLIF(cb.days_since_registration, 0)) * 365 AS annual_spend_rate,
    (css.resolved_interactions::FLOAT / NULLIF(css.total_interactions, 0)) * 100 AS service_resolution_rate,
    (ce.positive_responses::FLOAT / NULLIF(ce.campaigns_engaged, 0)) * 100 AS campaign_response_rate,
    (wb.total_sessions::FLOAT / NULLIF(cb.days_since_registration, 0)) * 30 AS monthly_session_frequency
FROM temp_customer_basic cb
LEFT JOIN temp_purchase_stats ps ON cb.customer_id = ps.customer_id
LEFT JOIN temp_product_preferences pp ON cb.customer_id = pp.customer_id
LEFT JOIN temp_customer_service_stats css ON cb.customer_id = css.customer_id
LEFT JOIN temp_campaign_engagement ce ON cb.customer_id = ce.customer_id
LEFT JOIN temp_website_behavior wb ON cb.customer_id = wb.customer_id;
```

## Step 3: Incorporate Developer Insights

Based on the developer conversations, we should consider the following enhancements:

1. Implement proper security measures and access controls (DevOps insight).
2. Ensure scalability by setting up auto-scaling policies and optimizing query performance (DevOps insight).
3. Add visualization-friendly fields for easy integration with data visualization libraries like D3.js, Recharts, or Plotly (UI Dev insight).
4. Include fields that support real-time updates for dynamic visualizations (UI Dev insight).

Let's add these considerations to our schema:

```sql
ALTER TABLE customer_360
ADD COLUMN data_last_updated TIMESTAMP,
ADD COLUMN customer_segment VARCHAR(50),
ADD COLUMN churn_risk_score FLOAT,
ADD COLUMN lifetime_value_projection FLOAT,
ADD COLUMN engagement_score FLOAT;

-- Update the new columns
UPDATE customer_360
SET
    data_last_updated = CURRENT_TIMESTAMP,
    customer_segment = CASE
        WHEN customer_value_segment = 'High' AND customer_status = 'Active' THEN 'High Value Active'
        WHEN customer_value_segment = 'High' AND customer_status = 'At Risk' THEN 'High Value At Risk'
        WHEN customer_value_segment = 'Medium' AND customer_status = 'Active' THEN 'Medium Value Active'
        WHEN customer_value_segment = 'Medium' AND customer_status = 'At Risk' THEN 'Medium Value At Risk'
        WHEN customer_value_segment = 'Low' AND customer_status = 'Active' THEN 'Low Value Active'
        WHEN customer_value_segment = 'Low' AND customer_status = 'At Risk' THEN 'Low Value At Risk'
        ELSE 'Churned'
    END,
    churn_risk_score = CASE
        WHEN customer_status = 'Active' THEN 0.2
        WHEN customer_status = 'At Risk' THEN 0.6
        ELSE 0.9
    END,
    lifetime_value_projection = total_spend * (1 + (churn_risk_score * -1 + 1)),
    engagement_score = (
        (total_purchases * 0.3) +
        (total_interactions * 0.2) +
        (campaigns_engaged * 0.2) +
        (total_sessions * 0.3)
    ) / (days_since_registration + 1);
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
    "last_visit_date": "DATE",
    "customer_value_segment": "TEXT",
    "customer_status": "TEXT",
    "annual_spend_rate": "REAL",
    "service_resolution_rate": "REAL",
    "campaign_response_rate": "REAL",
    "monthly_session_frequency": "REAL",
    "data_last_updated": "TIMESTAMP",
    "customer_segment": "VARCHAR(50)",
    "churn_risk_score": "FLOAT",
    "lifetime_value_projection": "FLOAT",
    "engagement_score": "FLOAT"
  }
}
```

## Explanation of Derived Fields

1. `age`: Calculated as the difference between the current date and the customer's date of birth.
2. `days_since_registration`: Number of days since the customer registered.
3. `customer_value_segment`: Categorizes customers based on their total spend.
4. `customer_status`: Determines if a customer is active, at risk, or churned based on their last purchase date.
5. `annual_spend_rate`: Estimates the customer's annual spend based on their total spend and registration duration.
6. `service_resolution_rate`: Percentage of customer service interactions that were resolved.
7. `campaign_response_rate`: Percentage of marketing campaigns that received a positive response from the customer.
8. `monthly_session_frequency`: Estimated number of website sessions per month.
9. `customer_segment`: Combines value segment and status for more detailed segmentation.
10. `churn_risk_score`: Estimated probability of customer churn based on their status.
11. `lifetime_value_projection`: Projected customer lifetime value based on current spend and churn risk.
12. `engagement_score`: Overall engagement score based on purchases, interactions, campaign engagement, and website sessions.

This comprehensive CDP schema incorporates data from various source tables, includes derived fields for customer analysis and segmentation, and takes into account the insights from the developer conversations. It provides a holistic view of each customer, enabling advanced analytics and personalized marketing strategies.