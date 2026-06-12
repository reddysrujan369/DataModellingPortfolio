# Food Delivery Platform — Data Model

## Overview
This data model represents a food delivery platform, capturing orders placed by customers, deliveries made by drivers, and restaurant details. It follows a star schema design with `fact_orders` as the central fact table connected to four dimension tables.

## Entity Relationship Diagram

```mermaid
erDiagram
    dim_customer {
        VARCHAR customer_id PK
        TEXT name
        VARCHAR email
        VARCHAR phone
        TEXT city
        DATE signup_date
        BOOLEAN is_premium
        INT lifetime_orders
    }

    dim_driver {
        VARCHAR driver_id PK
        TEXT name
        VARCHAR vehicle_type
        TEXT city
        DECIMAL avg_rating
        INT total_deliveries
        DATE signup_date
    }

    dim_restaurant {
        VARCHAR restaurant_id PK
        TEXT name
        TEXT cuisine_type
        TEXT city
        BIGINT zip_code
        DECIMAL avg_rating
        VARCHAR price_range
        BOOLEAN is_partner
    }

    dim_date {
        VARCHAR date_id PK
        TIMESTAMP full_date
        INT year
        INT quarter
        INT month
        INT day_of_week
        INT hour
        BOOLEAN is_weekend
        BOOLEAN is_holiday
    }

    fact_orders {
        INT order_id PK
        VARCHAR customer_id FK
        VARCHAR restaurant_id FK
        VARCHAR driver_id FK
        VARCHAR date_id FK
        DECIMAL subtotal
        DECIMAL delivery_fee
        DECIMAL tip_amount
        DECIMAL total_amount
        FLOAT prep_time_minutes
        FLOAT delivery_time_minutes
        FLOAT restaurant_rating
        FLOAT delivery_rating
    }

    dim_customer ||--o{ fact_orders : places
    dim_driver ||--o{ fact_orders : delivers
    dim_restaurant ||--o{ fact_orders : fulfills
    dim_date ||--o{ fact_orders : occurs_on
```

## Table Definitions

### dim_customer
| Column          | Type     | Description                        |
|------------------|----------|------------------------------------|
| customer_id      | VARCHAR  | Primary key, unique customer ID    |
| name             | TEXT     | Customer name                      |
| email            | VARCHAR  | Customer email                     |
| phone            | VARCHAR  | Customer phone number              |
| city             | TEXT     | Customer's city                    |
| signup_date      | DATE     | Date customer signed up            |
| is_premium       | BOOLEAN  | Whether customer has premium plan  |
| lifetime_orders  | INT      | Total orders placed by customer    |

### dim_driver
| Column           | Type     | Description                         |
|-------------------|----------|--------------------------------------|
| driver_id         | VARCHAR  | Primary key, unique driver ID        |
| name              | TEXT     | Driver name                          |
| vehicle_type      | VARCHAR  | Type of vehicle used for delivery    |
| city              | TEXT     | Driver's operating city              |
| avg_rating        | DECIMAL  | Average rating across deliveries     |
| total_deliveries  | INT      | Total deliveries completed           |
| signup_date       | DATE     | Date driver joined the platform      |

### dim_restaurant
| Column           | Type     | Description                          |
|-------------------|----------|----------------------------------------|
| restaurant_id     | VARCHAR  | Primary key, unique restaurant ID      |
| name              | TEXT     | Restaurant name                        |
| cuisine_type      | TEXT     | Type of cuisine offered                |
| city              | TEXT     | Restaurant's city                      |
| zip_code          | BIGINT   | Restaurant's zip code                  |
| avg_rating        | DECIMAL  | Average customer rating                |
| price_range       | VARCHAR  | Price tier (e.g. $, $$, $$$)           |
| is_partner        | BOOLEAN  | Whether restaurant is a partner        |

### dim_date
| Column       | Type      | Description                          |
|---------------|-----------|----------------------------------------|
| date_id       | VARCHAR   | Primary key, unique date ID            |
| full_date     | TIMESTAMP | Full timestamp of the date             |
| year          | INT       | Year                                    |
| quarter       | INT       | Quarter of the year (1-4)              |
| month         | INT       | Month (1-12)                           |
| day_of_week   | INT       | Day of week (0-6)                      |
| hour          | INT       | Hour of day (0-23)                     |
| is_weekend    | BOOLEAN   | Whether date falls on a weekend        |
| is_holiday    | BOOLEAN   | Whether date is a holiday              |

### fact_orders
| Column                 | Type     | Description                              |
|-------------------------|----------|---------------------------------------------|
| order_id               | INT      | Primary key, unique order ID               |
| customer_id            | VARCHAR  | FK to dim_customer                         |
| restaurant_id          | VARCHAR  | FK to dim_restaurant                       |
| driver_id              | VARCHAR  | FK to dim_driver                           |
| date_id                | VARCHAR  | FK to dim_date                             |
| subtotal               | DECIMAL  | Order subtotal before fees/tips            |
| delivery_fee           | DECIMAL  | Delivery fee charged                       |
| tip_amount             | DECIMAL  | Tip given to driver                        |
| total_amount           | DECIMAL  | Final order total                          |
| prep_time_minutes      | FLOAT    | Time taken by restaurant to prepare order  |
| delivery_time_minutes  | FLOAT    | Time taken for delivery                    |
| restaurant_rating      | FLOAT    | Rating given to restaurant for this order  |
| delivery_rating        | FLOAT    | Rating given to delivery for this order    |

## KPIs Supported

This schema is designed to support the following key performance indicators:

- Average delivery time
- Average order value
- Restaurant rating trends
- Driver efficiency
- Repeat order rate per customer

See [kpis.md](./kpis.md) for the full SQL queries.