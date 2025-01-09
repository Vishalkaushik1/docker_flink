# Data Pipelining and Real-Time Data Streaming with Apache Flink and Flink SQL

## Objective

This project demonstrates a comprehensive pipeline to handle streaming data by integrating Confluent schemas, Kafka topics, Apache Flink, and Elasticsearch. The primary goal is to:

- Generate and process data using Confluent schemas.
- Feed the processed data into multiple Kafka topics.
- Stream and analyze the data in real-time using Apache Flink and Flink SQL.
- Create a dashboard for visualization and derive insights using Kibana.

---

## Dataset Description

The datasets represent different entities for product, user, sales, and view logs:

### Products (xyz-1)

Represents product information.

**Schema:**

- `id`: Unique identifier for the product.
- `brand`: Brand name of the product.
- `name`: Product name.
- `sale_price`: Sale price of the product.
- `rating`: Rating of the product.

**Example Data:**

```json
{"id":"33b35eff-fddf-4561-b526-7a735d810d46","brand":"Will Inc","name":"Max Perf 679","sale_price":10995,"rating":4.3}
{"id":"219aa213-102a-4d2a-a5dc-13d76d074b1d","brand":"Huel-Ortiz","name":"Perf Max 205","sale_price":6495,"rating":4.7}
```

### Sales (xyz-2)

Represents order details.

**Schema:**

- `order_id`: Unique identifier for the order.
- `product_id`: Unique identifier for the product.
- `customer_id`: Unique identifier for the customer.
- `ts`: Timestamp of the order.

**Example Data:**

```json
{"order_id":1000,"product_id":"a0b8ad3b-e7b7-4d8a-8c24-4091538fcc70","customer_id":"73abd6cb-4c51-48ce-8dcc-489b70b7d528","ts":1609459200000}
{"order_id":1001,"product_id":"219aa213-102a-4d2a-a5dc-13d76d074b1d","customer_id":"d648722b-9777-4fe6-8939-2a9f16c2b99a","ts":1609459300000}
```

### Users (xyz-3)

Represents customer information.

**Schema:**

- `id`: Unique identifier for the user.
- `first_name`: First name of the user.
- `last_name`: Last name of the user.
- `email`: Email address of the user.
- `phone`: Phone number of the user.
- `street_address`: Address of the user.
- `state`: State of residence.
- `zip_code`: Zip code.
- `country`: Country.

**Example Data:**

```json
{"id":"f6bf0555-fba4-4f3f-aa16-e62de5446838","first_name":"Peria","last_name":"Minkin","email":"rtamblingson7l@sitemeter.com","state":"Minnesota"}
{"id":"cf144e67-d9c6-4f3c-b15b-04736c062df9","first_name":"Morton","last_name":"Westmore","email":"mmccanny6d@shutterfly.com","state":"Pennsylvania"}
```

### Views (xyz-4)

Represents product views by users.

**Schema:**

- `product_id`: Unique identifier for the product.
- `user_id`: Unique identifier for the user.
- `view_time`: Time spent viewing the product.
- `page_url`: URL of the viewed product.
- `ip`: IP address of the user.
- `ts`: Timestamp of the view.

**Example Data:**

```json
{"product_id":"0c389ab7-a526-4008-91d2-0b71bfa8b86a","user_id":"c81d29bf-76fc-4145-9023-39f800124895","view_time":78,"page_url":"https://www.acme.com/product/juxpw"}
{"product_id":"6403ac40-622a-4bdd-adec-a1c9111c5293","user_id":"0d0d080c-1cb0-4065-b969-561dfea6c8a3","view_time":116,"page_url":"https://www.acme.com/product/ghiju"}
```

---

## Data Pipeline Process

### 1. Data Generation

Data was generated using synthetic tools and transformed into JSON format.

### 2. Data Transformation

The JSON files were converted into key-value pairs using Python scripts to prepare them for Kafka ingestion.

### 3. Kafka Ingestion

Data was streamed into Kafka topics:

- `xyz1.json` -> `products`
- `xyz2.json` -> `sales`
- `xyz3.json` -> `users`
- `xyz4.json` -> `views`

### 4. Real-Time Analysis with Apache Flink

Apache Flink and Flink SQL were utilized for real-time data streaming.

#### Table Creation

**Products Table:**

```sql
CREATE TABLE products (
    id STRING,
    brand STRING,
    name STRING,
    sale_price DOUBLE,
    rating DOUBLE
) WITH (
    'connector' = 'kafka',
    'topic' = 'products',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json'
);
```

**Sales Table:**

```sql
CREATE TABLE sales (
    order_id BIGINT,
    product_id STRING,
    customer_id STRING,
    ts TIMESTAMP(3)
) WITH (
    'connector' = 'kafka',
    'topic' = 'sales',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json'
);
```

**Users Table:**

```sql
CREATE TABLE users (
    id STRING,
    first_name STRING,
    last_name STRING,
    email STRING,
    state STRING
) WITH (
    'connector' = 'kafka',
    'topic' = 'users',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json'
);
```

**Views Table:**

```sql
CREATE TABLE views (
    product_id STRING,
    user_id STRING,
    view_time INT,
    page_url STRING,
    ts TIMESTAMP(3)
) WITH (
    'connector' = 'kafka',
    'topic' = 'views',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json'
);
```

### 5. Data Enrichment

A consolidated view was created to join the data from all sources:

```sql
CREATE VIEW enriched_data AS
SELECT 
    v.product_id,
    v.user_id,
    u.first_name,
    u.last_name,
    p.name AS product_name,
    p.brand,
    s.order_id,
    s.ts AS order_date,
    v.view_time
FROM views v
LEFT JOIN users u ON v.user_id = u.id
LEFT JOIN products p ON v.product_id = p.id
LEFT JOIN sales s ON v.product_id = s.product_id;
```

### 6. Data Insertion into Elasticsearch

Data from the enriched view was inserted into an Elasticsearch index:

```sql
CREATE TABLE product_dashboard (
    product_id STRING,
    user_id STRING,
    first_name STRING,
    last_name STRING,
    product_name STRING,
    brand STRING,
    order_id BIGINT,
    order_date TIMESTAMP(3),
    view_time INT
) WITH (
    'connector' = 'elasticsearch-7',
    'hosts' = 'http://elasticsearch:9200',
    'index' = 'product_dashboard',
    'format' = 'json'
);

INSERT INTO product_dashboard
SELECT * FROM enriched_data;
```

---

## Dashboard Creation with Kibana

The data in the `product_dashboard` index was visualized using Kibana.

### Dashboard Highlights:

1. **Unique Product Count (Gauge):**

   - Observation: The dashboard shows 341 unique products.
   - **Insights:** Indicates a diverse product portfolio; helps in identifying top-performing products.

2. **Average Sale Price (Gauge):**

   - Observation: The average sale price across all products is 10,134.
   - **Insights:** Helps determine pricing strategies; indicates room for premium product introduction.

3. **Customer State Distribution (Bar Chart):**

   - Observation: States like Alaska, Alabama, and Arizona have significant customer engagement.
   - **Insights:** Guides geographic targeting for marketing campaigns.

4. **Top Brands by Revenue (Bar Chart):**

   - Observation: Brands such as "Ankunding and Sons" and "Berge Group" are top contributors.
   - **Insights:** Focus marketing efforts on promoting these brands.

5. **Sales Over Time (Line Chart):**

   - Observation: Sales show a gradual increase over time.
   - **Insights:** Indicates growing platform adoption; plan for scalability and inventory optimization.

6. **Page Views by Time of Day (Line Chart):**

   - Observation: Peaks in views are observed during specific time slots.
   - **Insights:** Optimize campaigns and promotions during high traffic hours.

7. **Customer Order Frequency (Bar Chart):**

   - Observation: Some customers place frequent orders.
   - **Insights:** Identify loyal customers and implement reward programs.

8. **Revenue Contribution by Product (Bar Chart):**

   - Observation: "Pro Perf 793" and "Max Spectra 457" are leading revenue generators.
   - **Insights:** Prioritize inventory and marketing for these products.

---

## Managerial Insights

1. **Customer Engagement:**

   - Identify top-viewed products and plan promotional campaigns.
   - Reward loyal customers with special discounts.

2. **Sales Optimization:**

   - Analyze sales trends and optimize inventory accordingly.

3. **Geographic Expansion:**

   - Focus on states with high engagement for targeted marketing.

4. **Product Insights:**

   - Enhance features of most viewed and purchased products.

---

## Learnings

- Leveraging Kafka and Flink enables efficient real-time data processing.
- Combining Elasticsearch and Kibana provides powerful visual analytics.
- Enriching data across multiple sources delivers actionable insights.

