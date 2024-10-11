# Customer Data Platform (CDP) Application Design Document

## Page 1: Project Overview and Authentication

### Project Name and Heading

**Project Name:** ManufactureCDP

**Tagline:** Unifying customer data for intelligent manufacturing insights

### Google SSO Authentication

The CDP application will implement Google SSO authentication using the @react-oauth/google library. This process will ensure secure and seamless access for users while maintaining data privacy.

Authentication Flow:
1. User clicks "Sign in with Google" button
2. Google OAuth consent screen appears
3. User grants permission
4. Application receives OAuth token
5. Backend verifies token and creates a session
6. User is redirected to the dashboard

Error Handling:
- Implement try-catch blocks for API calls
- Display user-friendly error messages
- Log detailed errors for debugging

Security Considerations:
- Use HTTPS for all communications
- Implement token refresh mechanism
- Store tokens securely using HttpOnly cookies

Authentication Interface:
- Minimalist design with Google SSO button
- Option for email/password login (empty inputs initially)
- Clear error messaging and loading indicators

## Page 2: Dashboard Design and Components

### Dashboard Layout and Structure

The dashboard will use a responsive 12-column grid system to ensure optimal display across devices. The layout will consist of:

1. Header: Logo, user profile, and global actions
2. Sidebar: Main navigation menu
3. Main content area: Data visualization components
4. Footer: Additional links and information

Navigation Design:
- Collapsible sidebar for space efficiency
- Top-level categories: Overview, Customers, Campaigns, Products, Analytics
- Sub-navigation for detailed views within each category

Screen Real Estate Allocation:
- Header: 10% of vertical space
- Sidebar: 20% of horizontal space (collapsible)
- Main content: 70-80% of total space
- Footer: 5% of vertical space

### Key Dashboard Components and API Mapping

#### a. Key Performance Indicators (KPIs)

Visual Description: Card-based layout with 4-5 key metrics
Data Visualization: Large numbers with trend indicators
Interactivity: Hover for detailed tooltip, click for drill-down view
Data Update: Real-time updates with subtle animations
CDP Schema Fields: total_lifetime_value, total_orders, avg_satisfaction_score, website_engagement_rate

API Endpoint: `/kpis`

```json
{
  "total_customers": 10000,
  "average_lifetime_value": 5000,
  "customer_satisfaction_score": 4.2,
  "website_engagement_rate": 0.75
}
```

#### b. Customer Segment Distribution (Pie Chart)

Visual Description: Interactive pie chart with legend
Data Visualization: D3.js or ECharts for smooth animations
Interactivity: Hover for segment details, click to filter dashboard
Data Update: Daily refresh with transition animations
CDP Schema Fields: customer_segment

API Endpoint: `/customer_segments`

```json
{
  "segments": [
    {"name": "High Value", "value": 2000},
    {"name": "Medium Value", "value": 5000},
    {"name": "Low Value", "value": 3000}
  ]
}
```

#### c. Monthly Revenue Trend (Line Chart)

Visual Description: Multi-line chart showing revenue trends
Data Visualization: Plotly.js for interactive zooming and panning
Interactivity: Hover for point values, double-click to reset zoom
Data Update: Monthly with smooth transitions
CDP Schema Fields: total_lifetime_value (aggregated monthly)

API Endpoint: `/monthly_revenue`

```json
{
  "months": ["Jan", "Feb", "Mar", "Apr", "May"],
  "revenue": [100000, 120000, 115000, 130000, 125000]
}
```

#### d. Top 5 Customers by Lifetime Value (Table)

Visual Description: Sortable table with customer details
Data Visualization: React Table library for sorting and pagination
Interactivity: Click column headers to sort, click row for customer profile
Data Update: Daily refresh with highlight for changes
CDP Schema Fields: customer_id, first_name, last_name, total_lifetime_value

API Endpoint: `/top_customers`

```json
{
  "top_customers": [
    {"id": 1, "name": "John Doe", "lifetime_value": 50000},
    {"id": 2, "name": "Jane Smith", "lifetime_value": 45000},
    // ... more customers
  ]
}
```

#### e. Product Category Performance (Bar Chart)

Visual Description: Horizontal bar chart showing category performance
Data Visualization: D3.js for custom styling and animations
Interactivity: Hover for details, click to view category breakdown
Data Update: Weekly with transition animations
CDP Schema Fields: favorite_category (aggregated)

API Endpoint: `/product_category_performance`

```json
{
  "categories": [
    {"name": "Electronics", "value": 500000},
    {"name": "Furniture", "value": 350000},
    // ... more categories
  ]
}
```

#### f. Customer Satisfaction Score (Gauge Chart)

Visual Description: Radial gauge chart with color-coded zones
Data Visualization: ECharts for smooth animations and styling
Interactivity: Hover for historical scores, click for detailed feedback
Data Update: Real-time updates with smooth needle movement
CDP Schema Fields: avg_satisfaction_score

API Endpoint: `/customer_satisfaction`

```json
{
  "current_score": 4.2,
  "previous_score": 4.0,
  "target_score": 4.5
}
```

#### g. Churn Risk Distribution (Pie Chart)

Visual Description: Pie chart showing customer churn risk levels
Data Visualization: Plotly.js for consistent styling with other charts
Interactivity: Hover for segment details, click to view at-risk customers
Data Update: Weekly with transition animations
CDP Schema Fields: days_since_last_purchase (used to calculate churn risk)

API Endpoint: `/churn_risk`

```json
{
  "risk_levels": [
    {"level": "Low", "count": 6000},
    {"level": "Medium", "count": 3000},
    {"level": "High", "count": 1000}
  ]
}
```

#### h. RFM Segmentation (Scatter Plot)

Visual Description: 3D scatter plot showing Recency, Frequency, Monetary value
Data Visualization: Plotly.js for 3D rendering and interactivity
Interactivity: Rotate, zoom, and pan the 3D space, click points for details
Data Update: Monthly with transition animations
CDP Schema Fields: days_since_last_purchase, total_orders, total_lifetime_value

API Endpoint: `/rfm_segmentation`

```json
{
  "segments": [
    {"recency": 5, "frequency": 20, "monetary": 5000, "count": 100},
    {"recency": 10, "frequency": 15, "monetary": 3000, "count": 200},
    // ... more segments
  ]
}
```

### Dashboard Interactivity and User Experience

Global Filtering:
- Implement a date range selector affecting all components
- Add customer segment filter to refine data across visualizations

Cross-component Data Linking:
- Clicking a segment in the Customer Segment Distribution updates other charts
- Selecting a product category filters customer data in other components

Customization Options:
- Drag-and-drop interface for rearranging dashboard widgets
- User-specific saved layouts and preferences
- Option to add/remove widgets based on user role

## Page 3: Architecture Visualization and Technical Considerations

### Sankey Diagram Implementation using Plotly

The Sankey diagram represents the data flow in our CDP, from source systems to the final customer_360 view. Here's how to implement it using Plotly:

1. Data Preparation:
   - Nodes: Represent each table and temporary table in the CDP schema
   - Links: Show the flow of data between nodes

2. Implementation Steps:
   ```python
   import plotly.graph_objects as go

   # Define nodes and links based on the CDP schema
   nodes = [
       {'id': 'customer_info'}, {'id': 'purchase_transactions'},
       # ... other nodes
   ]
   links = [
       {'source': 'customer_info', 'target': 'temp_basic_info', 'value': 1},
       # ... other links
   ]

   # Create the Sankey diagram
   fig = go.Figure(data=[go.Sankey(
       node = dict(
         pad = 15,
         thickness = 20,
         line = dict(color = "black", width = 0.5),
         label = [node['id'] for node in nodes],
         color = "blue"
       ),
       link = dict(
         source = [nodes.index(link['source']) for link in links],
         target = [nodes.index(link['target']) for link in links],
         value = [link['value'] for link in links]
   ))])

   fig.update_layout(title_text="CDP Data Flow", font_size=10)
   fig.show()
   ```

3. Styling Guidelines:
   - Use a color scheme that matches the overall CDP application design
   - Adjust node sizes to reflect the importance of each data source/table
   - Use consistent naming conventions for nodes to improve readability

4. Interactivity Features:
   - Implement hover tooltips to show detailed information about each node
   - Allow users to click on nodes to highlight related data flows
   - Provide zoom and pan capabilities for exploring complex data relationships

### Data Integration and Performance

1. Sample Data Generation:
   - Create a data generation script that produces sample data matching the CDP schema
   - Ensure generated data covers various scenarios and edge cases

2. Efficient Data Loading:
   - Implement lazy loading techniques for dashboard components
   - Use pagination for large datasets (e.g., customer lists)

3. State Management:
   - Utilize Redux for global state management
   - Implement local component state for isolated UI interactions

4. Caching Mechanisms:
   - Use browser local storage for user preferences and frequently accessed data
   - Implement server-side caching using Redis for API responses

5. Update Strategies:
   - Use WebSockets for real-time updates on critical metrics
   - Implement polling for less time-sensitive data

6. Error Handling:
   - Create fallback UI components for failed data loads
   - Implement retry mechanisms for failed API calls
   - Display user-friendly error messages with options to refresh or contact support

### Responsive Design and Cross-platform Considerations

1. Breakpoints:
   - Desktop: 1200px and above
   - Tablet: 768px to 1199px
   - Mobile: Below 768px

2. Layout Adjustments:
   - Desktop: Full sidebar, multi-column dashboard layout
   - Tablet: Collapsible sidebar, two-column dashboard layout
   - Mobile: Hidden sidebar (hamburger menu), single-column dashboard layout

3. Progressive Enhancement:
   - Ensure core functionality works on all devices
   - Add advanced interactive features for devices with higher capabilities
   - Use feature detection to provide fallbacks for older browsers

4. Touch Optimization:
   - Increase touch target sizes on mobile and tablet devices
   - Implement swipe gestures for navigation on touch-enabled devices

5. Performance Optimization:
   - Use code splitting to reduce initial load times
   - Implement image optimization techniques (e.g., lazy loading, responsive images)

By following these guidelines and leveraging the CDP schema, developer insights, and the Sankey diagram, we can create a robust, user-friendly, and performant CDP application that meets the needs of our manufacturing context.

## Page 4: Website Flow and API Integration

### Website Flow

1. User Authentication:
   - User navigates to the CDP application URL
   - Google SSO login screen is presented
   - User authenticates using their Google credentials
   - Upon successful authentication, user is redirected to the main dashboard

2. Dashboard Initialization:
   - Application checks for user permissions and role
   - Dashboard layout is loaded based on user preferences or default layout
   - Skeleton screens are displayed for each dashboard component

3. Data Loading Process:
   - API calls are made to fetch data for each dashboard component
   - Data is loaded asynchronously to improve perceived performance
   - Components are populated with data as it becomes available

4. User Interactions:
   - Users can interact with individual dashboard components (e.g., filtering, sorting)
   - Global filters affect data across all relevant components
   - Users can customize dashboard layout and save preferences

### API Integration

The CDP application integrates with backend APIs to fetch data for each dashboard component. Here's a detailed explanation of the API integration process:

```mermaid
graph TD
    A[User Authentication] --> B[Dashboard Initialization]
    B --> C{Fetch Data}
    C --> D[/kpis]
    C --> E[/customer_segments]
    C --> F[/monthly_revenue]
    C --> G[/top_customers]
    C --> H[/product_category_performance]
    C --> I[/customer_satisfaction]
    C --> J[/churn_risk]
    C --> K[/rfm_segmentation]
    D --> L[Update KPI Component]
    E --> M[Update Customer Segment Chart]
    F --> N[Update Revenue Trend Chart]
    G --> O[Update Top Customers Table]
    H --> P[Update Product Category Chart]
    I --> Q[Update Satisfaction Gauge]
    J --> R[Update Churn Risk Chart]
    K --> S[Update RFM Scatter Plot]
    L --> T[Dashboard Fully Loaded]
    M --> T
    N --> T
    O --> T
    P --> T
    Q --> T
    R --> T
    S --> T
```

1. KPIs (`/kpis`):
   - Fetches overall performance metrics
   - Updates KPI cards on the dashboard
   - Implements real-time updates using WebSocket connection

2. Customer Segments (`/customer_segments`):
   - Retrieves customer segment distribution data
   - Updates the pie chart visualization
   - Implements daily data refresh with smooth transitions

3. Monthly Revenue (`/monthly_revenue`):
   - Fetches historical revenue data
   - Updates the line chart showing revenue trends
   - Implements monthly updates with interpolation for smooth transitions

4. Top Customers (`/top_customers`):
   - Retrieves data for top customers by lifetime value
   - Updates the sortable table component
   - Implements daily refresh with highlighting of changes

5. Product Category Performance (`/product_category_performance`):
   - Fetches performance data for each product category
   - Updates the horizontal bar chart
   - Implements weekly updates with transition animations

6. Customer Satisfaction (`/customer_satisfaction`):
   - Retrieves current and historical satisfaction scores
   - Updates the gauge chart component
   - Implements real-time updates with smooth animations

7. Churn Risk (`/churn_risk`):
   - Fetches customer churn risk distribution data
   - Updates the pie chart visualization
   - Implements weekly updates with transition animations

8. RFM Segmentation (`/rfm_segmentation`):
   - Retrieves Recency, Frequency, and Monetary value data for customers
   - Updates the 3D scatter plot visualization
   - Implements monthly updates with transition animations

### Data Refresh and Real-time Updates

1. Real-time Updates:
   - Implement WebSocket connections for KPIs and Customer Satisfaction components
   - Use Socket.io library for managing real-time connections
   - Update component state and trigger re-renders on new data receipt

2. Periodic Refresh:
   - Implement different refresh intervals for various components:
     - Daily: Customer Segments, Top Customers
     - Weekly: Product Category Performance, Churn Risk
     - Monthly: Monthly Revenue, RFM Segmentation
   - Use React hooks (useState, useEffect) to manage refresh cycles

3. User-triggered Refresh:
   - Provide a global refresh button to manually update all components
   - Implement individual refresh buttons for each component

4. Optimistic Updates:
   - For user actions that modify data, update the UI immediately
   - Confirm changes with the backend and reconcile if necessary

5. Error Handling:
   - Implement retry logic for failed API calls
   - Display error states in components when data cannot be fetched
   - Provide options for users to manually trigger retries

6. Caching Strategy:
   - Implement client-side caching using Redux persist
   - Use service workers for offline support and faster subsequent loads

By following this API integration strategy and implementing efficient data refresh mechanisms, the CDP application will provide users with up-to-date, accurate information while maintaining a smooth and responsive user experience.# Customer Compass 360 Backend API Documentation

## Overview

This document provides detailed information about the backend API for the Customer Compass 360 application. The API is built using FastAPI and provides various endpoints to retrieve customer data, performance metrics, and visualizations.

## Base URL

The base URL for all API endpoints will depend on your deployment environment. Replace `{BASE_URL}` in the endpoint URLs with the appropriate base URL for your deployment.

## Authentication

Currently, the API does not implement authentication. If required, authentication mechanisms should be added before deploying to a production environment.

## Endpoints

### 1. Root

- **URL:** `{BASE_URL}/`
- **Method:** GET
- **Description:** Provides a welcome message for the API.
- **Response:**
  ```json
  {
    "message": "Welcome to the CDP Dashboard API"
  }
  ```

### 2. Key Performance Indicators (KPIs)

- **URL:** `{BASE_URL}/kpis`
- **Method:** GET
- **Description:** Retrieves key performance indicators for the business.
- **Response:** JSON object containing:
  - `total_customers`: Integer representing the total number of customers
  - `total_lifetime_value`: Float representing the total lifetime value of all customers
  - `average_order_value`: Float representing the average order value of all customers
  - `retention_rate`: Float representing the retention rate of all customers (percentage)
- **Example Response:**
  ```json
  {
    "total_customers": 10000,
    "total_lifetime_value": 5000000.50,
    "average_order_value": 150.75,
    "retention_rate": 85.5
  }
  ```

### 3. Customer Segments

- **URL:** `{BASE_URL}/customer_segments`
- **Method:** GET
- **Description:** Retrieves customer segment distribution data.
- **Response:** JSON object representing a pie chart of customer segment distribution.
- **Note:** The response is a Plotly figure JSON that can be used to render a chart on the frontend.

### 4. Monthly Revenue

- **URL:** `{BASE_URL}/monthly_revenue`
- **Method:** GET
- **Description:** Retrieves monthly revenue trend data.
- **Response:** JSON object representing a line chart of monthly revenue trend.
- **Note:** The response is a Plotly figure JSON that can be used to render a chart on the frontend.

### 5. Top Customers

- **URL:** `{BASE_URL}/top_customers`
- **Method:** GET
- **Description:** Retrieves the top 5 customers by total lifetime value.
- **Response:** JSON array of objects, each containing:
  - `customer_id`: String representing the customer's unique identifier
  - `first_name`: String representing the customer's first name
  - `last_name`: String representing the customer's last name
  - `total_lifetime_value`: Float representing the customer's total lifetime value
- **Example Response:**
  ```json
  [
    {
      "customer_id": "C001",
      "first_name": "John",
      "last_name": "Doe",
      "total_lifetime_value": 25000.50
    },
    {
      "customer_id": "C002",
      "first_name": "Jane",
      "last_name": "Smith",
      "total_lifetime_value": 22000.75
    }
  ]
  ```

### 6. Product Category Performance

- **URL:** `{BASE_URL}/product_category_performance`
- **Method:** GET
- **Description:** Retrieves revenue performance data for different product categories.
- **Response:** JSON object representing a bar chart of product category performance.
- **Note:** The response is a Plotly figure JSON that can be used to render a chart on the frontend.

### 7. Customer Satisfaction

- **URL:** `{BASE_URL}/customer_satisfaction`
- **Method:** GET
- **Description:** Retrieves the average customer satisfaction score.
- **Response:** JSON object representing a gauge chart of customer satisfaction score.
- **Note:** The response is a Plotly figure JSON that can be used to render a chart on the frontend.

### 8. Churn Risk

- **URL:** `{BASE_URL}/churn_risk`
- **Method:** GET
- **Description:** Retrieves the distribution of churn risk scores.
- **Response:** JSON object representing a pie chart of churn risk distribution.
- **Note:** The response is a Plotly figure JSON that can be used to render a chart on the frontend.

### 9. RFM Segmentation

- **URL:** `{BASE_URL}/rfm_segmentation`
- **Method:** GET
- **Description:** Retrieves RFM (Recency, Frequency, Monetary) segmentation data.
- **Response:** JSON object representing a 3D scatter plot of RFM segmentation.
- **Note:** The response is a Plotly figure JSON that can be used to render a chart on the frontend.

## Error Handling

All endpoints will return appropriate HTTP status codes:
- 200: Successful request
- 4xx: Client errors (e.g., 404 for not found, 400 for bad request)
- 5xx: Server errors

In case of errors, a JSON object with an "error" key describing the issue will be returned.

Example error response:
```json
{
  "error": "Resource not found"
}
```

## Data Format

All responses are in JSON format. For endpoints returning chart data (customer segments, monthly revenue, product category performance, customer satisfaction, churn risk, and RFM segmentation), the response is a JSON representation of a Plotly figure, which can be directly used to render charts on the frontend.

## Database

The API interacts with an SQLite database named `pg_cdp_demo.db`. This database contains the following tables:
- `customer_360`: Contains comprehensive customer data
- `purchase_transactions`: Contains data about customer purchases
- `product_catalog`: Contains information about products

## Dependencies

The backend relies on the following main Python libraries:
- FastAPI: Web framework for building the API
- SQLite3: Database interaction
- Pandas: Data manipulation and analysis
- Plotly: Generating interactive charts
- Anthropic: For potential AI-powered features (API key required)

## Running the API

To run the API locally:

1. Ensure all dependencies are installed:
   ```
   pip install fastapi uvicorn sqlite3 pandas plotly
   ```

2. Run the FastAPI application using Uvicorn:
   ```
   uvicorn main:app --host 0.0.0.0 --port 8000
   ```

The API will be available at `http://localhost:8000`. You can access the interactive API documentation at `http://localhost:8000/docs`.

## Deployment

For production deployment, consider the following:
- Use a production-grade ASGI server like Gunicorn with Uvicorn workers
- Implement proper authentication and authorization mechanisms
- Use environment variables for sensitive information (e.g., database credentials)
- Consider using a production-ready database system (e.g., PostgreSQL)
- Implement HTTPS for secure communication
- Set up proper logging and monitoring

## Conclusion

This API provides a comprehensive set of endpoints to retrieve various customer and business metrics for the Customer Compass 360 application. It's designed to be easily integrated with a frontend application to create a full-featured customer data platform dashboard.