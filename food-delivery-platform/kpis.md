# KPI Queries — Food Delivery Platform

This document contains SQL queries for the key performance indicators (KPIs) supported by the data model defined in [README.md](./README.md).

---

## 1. Average Delivery Time

Overall average delivery time across all orders:

```sql
SELECT
    ROUND(AVG(delivery_time_minutes), 2) AS avg_delivery_time_minutes
FROM fact_orders;
```

Average delivery time broken down by year and month:

```sql
SELECT
    d.year,
    d.month,
    ROUND(AVG(f.delivery_time_minutes), 2) AS avg_delivery_time_minutes
FROM fact_orders f
JOIN dim_date d ON f.date_id = d.date_id
GROUP BY d.year, d.month
ORDER BY d.year, d.month;
```

---

## 2. Average Order Value

Overall average order value:

```sql
SELECT
    ROUND(AVG(total_amount), 2) AS avg_order_value
FROM fact_orders;
```

Average order value per customer:

```sql
SELECT
    customer_id,
    ROUND(AVG(total_amount), 2) AS avg_order_value
FROM fact_orders
GROUP BY customer_id;
```

---

## 3. Restaurant Rating Trends

Average per-order restaurant rating over time, by restaurant:

```sql
SELECT
    f.restaurant_id,
    d.year,
    d.month,
    ROUND(AVG(f.restaurant_rating), 2) AS avg_restaurant_rating
FROM fact_orders f
JOIN dim_date d ON f.date_id = d.date_id
GROUP BY f.restaurant_id, d.year, d.month
ORDER BY f.restaurant_id, d.year, d.month;
```

Comparing per-order ratings against the restaurant's stored average rating:

```sql
SELECT
    f.restaurant_id,
    d.year,
    d.month,
    ROUND(AVG(f.delivery_rating), 2) AS avg_order_delivery_rating,
    ROUND(AVG(r.avg_rating), 2) AS restaurant_stored_avg_rating
FROM fact_orders f
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_restaurant r ON f.restaurant_id = r.restaurant_id
GROUP BY f.restaurant_id, d.year, d.month
ORDER BY f.restaurant_id, d.year, d.month;
```

---

## 4. Driver Efficiency

Average delivery time and rating per driver, with order volume:

```sql
SELECT
    dr.driver_id,
    dr.name,
    COUNT(f.order_id) AS total_orders,
    ROUND(AVG(f.delivery_time_minutes), 2) AS avg_delivery_time_minutes,
    ROUND(AVG(f.delivery_rating), 2) AS avg_delivery_rating
FROM fact_orders f
JOIN dim_driver dr ON f.driver_id = dr.driver_id
GROUP BY dr.driver_id, dr.name
ORDER BY avg_delivery_time_minutes ASC;
```

Composite efficiency score (higher rating relative to delivery time = more efficient):

```sql
SELECT
    dr.driver_id,
    dr.name,
    ROUND(AVG(f.delivery_time_minutes), 2) AS avg_delivery_time,
    ROUND(AVG(f.delivery_rating), 2) AS avg_rating,
    COUNT(f.order_id) AS total_orders,
    ROUND(AVG(f.delivery_rating) / NULLIF(AVG(f.delivery_time_minutes), 0), 4) AS efficiency_score
FROM fact_orders f
JOIN dim_driver dr ON f.driver_id = dr.driver_id
GROUP BY dr.driver_id, dr.name
ORDER BY efficiency_score DESC;
```

---

## 5. Repeat Order Rate per Customer

Overall repeat order rate (percentage of customers with more than one order):

```sql
WITH order_counts AS (
    SELECT
        customer_id,
        COUNT(order_id) AS num_orders
    FROM fact_orders
    GROUP BY customer_id
)
SELECT
    COUNT(*) AS total_customers,
    SUM(CASE WHEN num_orders > 1 THEN 1 ELSE 0 END) AS repeat_customers,
    ROUND(
        SUM(CASE WHEN num_orders > 1 THEN 1 ELSE 0 END)::numeric
        / COUNT(*) * 100,
        2
    ) AS repeat_order_rate_pct
FROM order_counts;
```

Per-customer order count, flagging repeat customers:

```sql
SELECT
    c.customer_id,
    c.name,
    COUNT(f.order_id) AS total_orders,
    CASE WHEN COUNT(f.order_id) > 1 THEN TRUE ELSE FALSE END AS is_repeat_customer
FROM fact_orders f
JOIN dim_customer c ON f.customer_id = c.customer_id
GROUP BY c.customer_id, c.name
ORDER BY total_orders DESC;
```