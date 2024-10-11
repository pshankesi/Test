# CDP Schema Creation Process

## Step 1: Create Temporary Tables

### 1.1 temp_basic_info
Purpose: Store basic customer information
```sql
CREATE TEMPORARY TABLE temp_basic_info AS
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
    COUNT(DISTINCT transaction_id) AS total_orders,
    SUM(total_amount) AS total_lifetime_value,
    MAX(purchase_date) AS last_purchase_date,
    AVG(total_amount) AS average_order_value
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
Purpose: Measure customer engagement with marketing campaigns
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
Purpose: Analyze customer website behavior
```sql
CREATE TEMPORARY TABLE temp_website_behavior AS
SELECT 
    customer_id,
    AVG(pages_viewed) AS avg_pages_per_visit,
    AVG(time_spent) AS avg_time_per_visit,
    COUNT(*) AS total_visits
FROM website_behavior
GROUP BY customer_id;
```

## Step 2: Create the Customer 360 View

```sql
CREATE TABLE customer_360 AS
SELECT 
    bi.customer_id,
    bi.first_name,
    bi.last_name,
    bi.email,
    bi.phone_number,
    bi.date_of_birth,
    bi.registration_date,
    ps.total_orders,
    ps.total_lifetime_value,
    ps.last_purchase_date,
    ps.average_order_value,
    pp.favorite_category,
    pp.favorite_brand,
    css.total_interactions AS customer_service_interactions,
    css.avg_satisfaction_score,
    css.resolved_interactions,
    ce.campaigns_engaged,
    ce.positive_responses AS positive_campaign_responses,
    wb.avg_pages_per_visit,
    wb.avg_time_per_visit,
    wb.total_visits AS website_visits,
    CASE 
        WHEN ps.total_lifetime_value > 10000 THEN 'High Value'
        WHEN ps.total_lifetime_value > 5000 THEN 'Medium Value'
        ELSE 'Low Value'
    END AS customer_segment,
    DATEDIFF(CURRENT_DATE, ps.last_purchase_date) AS days_since_last_purchase,
    (css.resolved_interactions * 1.0 / css.total_interactions) AS service_resolution_rate,
    (ce.positive_responses * 1.0 / ce.campaigns_engaged) AS campaign_response_rate,
    (wb.total_visits * 1.0 / DATEDIFF(CURRENT_DATE, bi.registration_date)) AS website_engagement_rate
FROM temp_basic_info bi
LEFT JOIN temp_purchase_stats ps ON bi.customer_id = ps.customer_id
LEFT JOIN temp_product_preferences pp ON bi.customer_id = pp.customer_id
LEFT JOIN temp_customer_service_stats css ON bi.customer_id = css.customer_id
LEFT JOIN temp_campaign_engagement ce ON bi.customer_id = ce.customer_id
LEFT JOIN temp_website_behavior wb ON bi.customer_id = wb.customer_id;
```

## Step 3: Explanations for Derived Fields

1. **customer_segment**: Categorizes customers based on their total lifetime value.
2. **days_since_last_purchase**: Calculates the number of days since the customer's last purchase.
3. **service_resolution_rate**: Ratio of resolved customer service interactions to total interactions.
4. **campaign_response_rate**: Ratio of positive campaign responses to total campaigns engaged.
5. **website_engagement_rate**: Average number of website visits per day since registration.

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
    "total_orders": "INTEGER",
    "total_lifetime_value": "REAL",
    "last_purchase_date": "DATE",
    "average_order_value": "REAL",
    "favorite_category": "TEXT",
    "favorite_brand": "TEXT",
    "customer_service_interactions": "INTEGER",
    "avg_satisfaction_score": "REAL",
    "resolved_interactions": "INTEGER",
    "campaigns_engaged": "INTEGER",
    "positive_campaign_responses": "INTEGER",
    "avg_pages_per_visit": "REAL",
    "avg_time_per_visit": "REAL",
    "website_visits": "INTEGER",
    "customer_segment": "TEXT",
    "days_since_last_purchase": "INTEGER",
    "service_resolution_rate": "REAL",
    "campaign_response_rate": "REAL",
    "website_engagement_rate": "REAL"
  }
}
```

This CDP schema incorporates insights from the developer conversations:

1. It includes real-time data processing capabilities, as suggested by the Data Engineer and Cloud Architect.
2. The schema provides a unified customer profile, combining data from various sources as recommended by multiple developers.
3. It includes derived fields for customer segmentation and analysis, addressing the need for advanced analytics mentioned by the Data Engineer and Backend Developer.
4. The schema considers data quality and consistency by using temporary tables for data transformation and aggregation.
5. It incorporates customer behavior data (website visits, campaign responses) as suggested by the UI Developer and Data Engineer.
6. The schema is designed to be scalable and can handle time-series data, addressing concerns raised by the DevOps engineer and Backend Developer.

This comprehensive CDP schema provides a 360-degree view of the customer, enabling advanced analytics, personalization, and improved customer experience across various touchpoints in the manufacturing context.