# Ecommerce Data Ingestion and Analysis using MySQL and Python

This project demonstrates how to efficiently load CSV files into a MySQL database, process the data, and perform various SQL queries for ecommerce analysis. The data consists of customer details, order information, product listings, and other key tables, which are analyzed using Python libraries like Pandas and MySQL Connector.

## Table of Contents

- Project Structure
- Features
- Prerequisites
- Installation
- Data Ingestion
- Queries
  - Basic Queries
  - Intermediate Queries
  - Advanced Queries
- Usage
- Feedback

## Project Structure

- **customers.csv**: Customer data
- **order_items.csv**: Data about items in customer orders
- **sellers.csv**: Seller information
- **products.csv**: Product catalog
- **geolocation.csv**: Geographic data of customers
- **payments.csv**: Payment details for each order
- **orders.csv**: Order metadata

## Features

- Load CSV files into a MySQL database with dynamic table creation based on column types.
- Perform various queries on the database, including customer insights, sales trends, and retention rates.
- Generate visualizations from SQL results using Matplotlib and Seaborn.

## Prerequisites

- Python 3.7+
- MySQL server and client
- Libraries: Pandas, MySQL Connector, Matplotlib, Seaborn

## Installation

1. Clone the repository:

```bash
git clone https://github.com/yourusername/ecommerce-data-analysis.git
```

2. Install the required Python packages:

```bash
pip install pandas mysql-connector-python matplotlib seaborn
```

3. Set up MySQL and create a database named `ecommerce`:

```sql
CREATE DATABASE ecommerce;
```

4. Update the database connection credentials in the Python script.

## Data Ingestion

The provided Python script dynamically ingests multiple CSV files and loads them into corresponding tables in a MySQL database.

- The `get_sql_type` function dynamically assigns SQL types (INT, FLOAT, TEXT, etc.) based on the Pandas dataframe's column types.
- Before inserting data, the script cleans column names and replaces `NaN` values with `NULL`.

```python
# Example CSV file ingestion process
df = pd.read_csv(file_path)
df = df.where(pd.notnull(df), None)
# Table creation and data insertion in MySQL
cursor.execute(create_table_query)
cursor.execute(insert_query)
```

## Queries

Below are the queries used to extract insights from the ecommerce dataset.

### Basic Queries

1. **List all unique cities where customers are located.**
    ```sql
    SELECT DISTINCT Customer_city FROM customers;
    ```
    **Output**:  
![image](https://github.com/user-attachments/assets/d875de4f-45fa-4dd8-97c9-34af899bb00b)


2. **Count the number of orders placed in 2017.**
    ```sql
    SELECT COUNT(*) FROM orders WHERE YEAR(order_purchase_timestamp) = 2017;
    ```
    **Output**: Total orders placed in 2017 are 45,101.

3. **Find the total sales per category.**
    ```sql
    SELECT UPPER(p.product_category),
           ROUND(SUM(oi.price + oi.freight_value)/1000) total_sales
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_category
    ORDER BY total_sales DESC
    LIMIT 10;
    ```
    **Output**:   
![image](https://github.com/user-attachments/assets/fed9f900-8eab-4d83-93ff-0d48db977f0d)


4. **Calculate the percentage of orders that were paid in installments.**
    ```sql
    SELECT (SUM(CASE WHEN payment_installments >= 1 THEN 1 ELSE 0 END) / COUNT(*)) * 100
    FROM payments;
    ```
    **Output**: The percentage of orders paid in installments is 99.9981%.

5. **Count the number of customers from each state.**
    ```sql
    SELECT customer_state, COUNT(*) AS cust_count
    FROM customers
    GROUP BY customer_state
    ORDER BY cust_count DESC
    LIMIT 10;
    ```
   **Output**:  
![image](https://github.com/user-attachments/assets/e30b6ea6-39ac-431b-94f5-d1822d28d37f)

### Intermediate Queries

1. **Calculate the number of orders per month in 2018.**
    ```sql
    SELECT MONTHNAME(order_purchase_timestamp) AS months, COUNT(order_id)
    FROM orders
    WHERE YEAR(order_purchase_timestamp) = 2018
    GROUP BY months;
    ```
   **Output**:  
![image](https://github.com/user-attachments/assets/5023be77-fbe5-486e-b35a-1e0e4860f469)


2. **Find the average number of products per order, grouped by customer city.**
    ```sql
    WITH count_per_order AS (
        SELECT o.order_id, o.customer_id, COUNT(oi.order_id) order_count
        FROM orders o
        JOIN order_items oi ON o.order_id = oi.order_id
        GROUP BY o.order_id, o.customer_id
    )
    SELECT c.customer_city, ROUND(AVG(cpo.order_count), 2) avg_order_count
    FROM customers c
    JOIN count_per_order cpo ON c.customer_id = cpo.customer_id
    GROUP BY c.customer_city
    ORDER BY avg_order_count DESC
    LIMIT 10;
    ``` 
   **Output**:
![image](https://github.com/user-attachments/assets/6060313e-65cf-46b6-a6ae-8c2ef77dc303)


3. **Calculate the percentage of total revenue contributed by each product category.**
    ```sql
    SELECT UPPER(p.product_category) category,
           ROUND((SUM(payments.payment_value) / (SELECT SUM(payments.payment_value) FROM payments)) * 100, 2) percentage_distribution
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    JOIN payments ON payments.order_id = oi.order_id
    GROUP BY p.product_category
    ORDER BY percentage_distribution DESC
    LIMIT 10;
    ```
   **Output**:  
![image](https://github.com/user-attachments/assets/cd03d5b4-f02d-4842-9b91-247b13d435a5)


4. **Identify the correlation between product price and the number of times a product has been purchased.**
    ```sql
    SELECT p.product_category,
           COUNT(oi.product_id),
           ROUND(AVG(oi.price), 2) avg_price
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_category;
    ```
   **Output**:  
![image](https://github.com/user-attachments/assets/1b052ffe-e24d-4665-8873-db0bf9d686c0)


5. **Calculate the total revenue generated by each seller, and rank them by revenue.**
    ```sql
    SELECT *, DENSE_RANK() OVER (ORDER BY revenue DESC) AS ranks
    FROM (
        SELECT oi.seller_id, SUM(p.payment_value) revenue
        FROM order_items oi
        JOIN payments p ON oi.order_id = p.order_id
        GROUP BY oi.seller_id
    ) a
    LIMIT 10;
    ```
   **Output**:
![image](https://github.com/user-attachments/assets/aaf9f870-972b-4a79-856c-1214a1d69426)


### Advanced Queries

1. **Calculate the moving average of order values for each customer over their order history.**
    ```sql
    SELECT customer_id, order_purchase_timestamp, payment,
           AVG(payment) OVER (PARTITION BY customer_id ORDER BY order_purchase_timestamp ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
    FROM (
        SELECT o.customer_id, o.order_purchase_timestamp, p.payment_value AS payment
        FROM orders o
        JOIN payments p ON o.order_id = p.order_id
    ) a;
    ```
   **Output**:
![image](https://github.com/user-attachments/assets/622130cb-2d22-4fe1-a710-957c8dffb3cd)


2. **Calculate the cumulative sales per month for each year.**
    ```sql
    SELECT years, months, payments, SUM(payments) OVER (ORDER BY years, months) AS cumulative_sales
    FROM (
        SELECT YEAR(o.order_purchase_timestamp) years, MONTH(o.order_purchase_timestamp) months,
               ROUND(SUM(p.payment_value), 2) payments
        FROM orders o
        JOIN payments p ON o.order_id = p.order_id
        GROUP BY years, months
        ORDER BY years, months
    ) a;
    ```
   **Output**:
![image](https://github.com/user-attachments/assets/50c00e67-b84c-49a7-9fcf-d4ec136ab163)


3. **Calculate the year-over-year growth rate of total sales.**
    ```sql
    WITH a AS (
        SELECT YEAR(o.order_purchase_timestamp) years,
               ROUND(SUM(p.payment_value), 2) payments
        FROM orders o
        JOIN payments p ON o.order_id = p.order_id
        GROUP BY years
        ORDER BY years
    )
    SELECT years, 
           ((payments - LAG(payments, 1) OVER (ORDER BY years)) / LAG(payments, 1) OVER (ORDER BY years)) * 100 AS yoy_growth
    FROM a;
    ```
   **Output**:
![image](https://github.com/user-attachments/assets/8d6b0e80-6b52-465b-b1f7-d931881ca63d)


4. **Calculate the retention rate of customers.**
    ```sql
    WITH a AS (
        SELECT customers.customer_id,
               MIN(orders.order_purchase_timestamp) AS first_order
        FROM customers
        JOIN orders ON customers.customer_id = orders.customer_id
        GROUP BY customers.customer_id
    ),
    b AS (
        SELECT a.customer_id, COUNT(DISTINCT orders.order_purchase_timestamp) AS next_order_count
        FROM a
        JOIN orders ON a.customer_id = orders.customer_id
        AND orders.order_purchase_timestamp > a.first_order
        AND orders.order_purchase_timestamp > DATE_ADD(a.first_order, INTERVAL 6 MONTH)
        GROUP BY a.customer_id
    )
    SELECT 100 * (COUNT(DISTINCT a.customer_id) / COUNT(DISTINCT b.customer_id)) AS retention_rate
    FROM a
    JOIN b ON a.customer_id = b.customer_id;
    ```
   **Output**:
![image](https://github.com/user-attachments/assets/86978aaa-4165-48db-a337-d76643b74030)


5. **Identify the top 3 customers who spent the most money in each year.**
    ```sql
    SELECT *
    FROM (
        SELECT YEAR(orders.order_purchase_timestamp) AS years,
               orders.customer_id,
               SUM(payments.payment_value) payment,
               DENSE_R

    ```
   **Output**:
![image](https://github.com/user-attachments/assets/c4359bcb-59bd-44f0-85fd-331b2f44cd3c)


## Usage

1. Execute the Python script to load CSV files into MySQL tables.
2. Run SQL queries to retrieve customer and sales insights.
3. Generate visualizations based on query results using Matplotlib and Seaborn.

## 📩 Feedback
If you have any feedback, please reach out to me at [Linkedin](https://www.linkedin.com/in/vineet-patel416/)
