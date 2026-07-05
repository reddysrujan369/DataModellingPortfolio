# Ride-Sharing Application — Data Model Documentation

## 1. Business Requirements & Scope

### 1.1 Overview
This data model supports a ride-hailing application (Uber-style) covering the core entities: riders, drivers, vehicles, rides, payments, ratings, and geo-locations. The model is designed to support KPI reporting, AI/ML use cases, and business optimization analysis.

### 1.2 Key Business Requirements
- **Driver onboarding lifecycle**: Track signup, background check, document check, and activation dates — including re-verification cycles when a driver goes inactive and is later reactivated.
- **Vehicle compliance lifecycle**: Track insurance, pollution certificate (PUC), registration (RC), and fitness certificate validity over time.
- **Third-party / fleet vehicles**: Support vehicles not owned by the driver directly (fleet companies, rental partners), and drivers who may use different vehicles over time.
- **Cancellations**: Track ride cancellations by both rider and driver, with reasons and timing.
- **Payments**: Track transaction amount, driver earnings, and platform commission per ride.
- **Ratings**: Track bidirectional ratings — rider → driver and driver → rider — including rater/ratee gender, to support gender-bias analysis in ratings.
- **Core KPIs required**:
  - Monthly revenue per driver
  - Cancellation rate (by driver, by rider)
  - Ratings by gender pairing (bias detection)
  - Driver utilization rate
  - Average trip duration
  - Rider retention rate
  - Surge pricing patterns

### 1.3 Modeling Approach Decisions
| Decision | Choice | Rationale |
|---|---|---|
| Normalization for source entities | Light-touch (no formal 3NF proof) | Project is OLAP/analytics-focused, not a live OLTP system |
| Warehouse modeling style | Kimball star schema | Optimized for BI queries and KPI reporting |
| Fact table strategy | Multiple fact tables (Fact_Ride, Fact_Payment, Fact_Rating) | Loose coupling, clean grain per business process |
| Cancellation modeling | Merged into Fact_Ride (not a separate fact) | Same grain as Ride (0 or 1 cancellation per ride) |
| Date/Time dimension | Single merged Dim_Date (hourly grain) | Simpler joins; hourly granularity sufficient for all stated KPIs |
| Driver onboarding history | SCD Type 2, single combined table | Tracks full history of bg check / doc check / activation together |
| Vehicle compliance history | SCD Type 2 | Tracks insurance/PUC/RC/fitness validity over time |
| Vehicle ownership | Separate Vehicle_Owner entity + Driver_Vehicle_Mapping | Supports fleet/third-party vehicles and drivers using multiple vehicles over time |
| Cancellation reason | Free text (not a lookup table) | Kept simple per project scope |

---

## 2. Rough Entities (Conceptual Model)

1. **Rider** — the customer requesting rides
2. **Driver** — the person driving
3. **Driver_Onboarding_History** — SCD2 history of driver verification lifecycle
4. **Driver_Session** — driver online/offline sessions (needed for utilization KPIs)
5. **Vehicle** — the car used for rides
6. **Vehicle_Owner** — owner of the vehicle (self, fleet company, rental partner)
7. **Driver_Vehicle_Mapping** — many-to-many relationship between drivers and vehicles over time
8. **Vehicle_Compliance_History** — SCD2 history of vehicle documents (insurance, PUC, RC, fitness)
9. **Ride / Trip** — the core transactional event (includes cancellation attributes)
10. **Payment** — financial transaction tied to a ride
11. **Rating** — bidirectional rating event (rider↔driver)
12. **Geo_Location** — pickup/drop coordinates and zone info

### 2.1 Relationship Summary
- Rider (1) → (M) Ride
- Driver (1) → (M) Ride
- Driver (1) → (M) Driver_Onboarding_History
- Driver (1) → (M) Driver_Session
- Driver (M) ↔ (M) Vehicle *(via Driver_Vehicle_Mapping)*
- Vehicle_Owner (1) → (M) Vehicle
- Vehicle (1) → (M) Vehicle_Compliance_History
- Ride (1) → (0/1) Payment
- Ride (1) → (0/2) Rating *(rider→driver, driver→rider)*
- Ride (M) → (1) Geo_Location *(pickup)*
- Ride (M) → (1) Geo_Location *(drop)*
- Ride (M) → (1) Vehicle *(snapshot at time of ride)*

---

## 3. Logical Model (Source Entities)

### 3.1 Rider
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| rider_id | INT | No | PK |
| name | VARCHAR(100) | No | |
| gender | VARCHAR(10) | Yes | |
| email | VARCHAR(100) | No | |
| phone | VARCHAR(15) | No | |
| signup_date | DATE | No | |
| home_city | VARCHAR(50) | No | |
| status | VARCHAR(20) | No | active/inactive/banned |

### 3.2 Driver
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| driver_id | INT | No | PK |
| name | VARCHAR(100) | No | |
| gender | VARCHAR(10) | Yes | |
| phone | VARCHAR(15) | No | |
| email | VARCHAR(100) | No | |
| city | VARCHAR(50) | No | |
| current_status | VARCHAR(20) | No | active/inactive/suspended |

### 3.3 Driver_Onboarding_History (SCD2)
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| onboarding_sk | BIGINT | No | PK (surrogate) |
| driver_id | INT | No | FK → Driver |
| bg_check_status | VARCHAR(20) | Yes | |
| bg_check_date | DATE | Yes | |
| doc_check_status | VARCHAR(20) | Yes | |
| doc_check_date | DATE | Yes | |
| activation_status | VARCHAR(20) | Yes | |
| activation_date | DATE | Yes | |
| deactivation_reason | VARCHAR(200) | Yes | |
| effective_start_date | DATE | No | |
| effective_end_date | DATE | Yes | NULL = current |
| is_current | BOOLEAN | No | |

### 3.4 Driver_Session
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| session_id | BIGINT | No | PK |
| driver_id | INT | No | FK → Driver |
| login_time | TIMESTAMP | No | |
| logout_time | TIMESTAMP | Yes | NULL = still online |
| city | VARCHAR(50) | No | |
| geo_id | INT | Yes | FK → Geo_Location |

### 3.5 Vehicle
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| vehicle_id | INT | No | PK |
| make | VARCHAR(50) | No | |
| model | VARCHAR(50) | No | |
| year | SMALLINT | No | |
| plate_number | VARCHAR(20) | No | Unique |
| vehicle_type | VARCHAR(20) | No | economy/premium/xl |
| registration_date | DATE | No | |
| ownership_type | VARCHAR(20) | No | self_owned/fleet_leased/third_party |
| owner_id | INT | No | FK → Vehicle_Owner |

### 3.6 Vehicle_Owner
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| owner_id | INT | No | PK |
| owner_name | VARCHAR(100) | No | |
| owner_type | VARCHAR(20) | No | individual/fleet_company/rental_partner |
| contact_info | VARCHAR(100) | Yes | |
| city | VARCHAR(50) | Yes | |
| partnership_start_date | DATE | Yes | |

### 3.7 Driver_Vehicle_Mapping
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| mapping_id | BIGINT | No | PK |
| driver_id | INT | No | FK → Driver |
| vehicle_id | INT | No | FK → Vehicle |
| assigned_start_date | DATE | No | |
| assigned_end_date | DATE | Yes | NULL = current |
| is_current | BOOLEAN | No | |

### 3.8 Vehicle_Compliance_History (SCD2)
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| compliance_sk | BIGINT | No | PK (surrogate) |
| vehicle_id | INT | No | FK → Vehicle |
| insurance_status | VARCHAR(20) | Yes | |
| insurance_expiry_date | DATE | Yes | |
| puc_status | VARCHAR(20) | Yes | |
| puc_expiry_date | DATE | Yes | |
| rc_status | VARCHAR(20) | Yes | |
| rc_expiry_date | DATE | Yes | |
| fitness_status | VARCHAR(20) | Yes | |
| fitness_expiry_date | DATE | Yes | |
| effective_start_date | DATE | No | |
| effective_end_date | DATE | Yes | NULL = current |
| is_current | BOOLEAN | No | |

### 3.9 Ride / Trip
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| ride_id | BIGINT | No | PK |
| rider_id | INT | No | FK → Rider |
| driver_id | INT | Yes | FK → Driver (NULL if no_driver_found) |
| vehicle_id | INT | Yes | FK → Vehicle (snapshot) |
| request_time | TIMESTAMP | No | |
| offered_time | TIMESTAMP | Yes | |
| pickup_time | TIMESTAMP | Yes | |
| drop_time | TIMESTAMP | Yes | |
| pickup_geo_id | INT | No | FK → Geo_Location |
| drop_geo_id | INT | Yes | FK → Geo_Location |
| distance_km | DECIMAL(6,2) | Yes | |
| duration_min | INT | Yes | |
| fare_amount | DECIMAL(10,2) | Yes | |
| surge_multiplier | DECIMAL(3,2) | Yes | |
| ride_status | VARCHAR(30) | No | completed/cancelled_by_rider/cancelled_by_driver/no_driver_found/no_show |
| cancelled_by | VARCHAR(10) | Yes | rider/driver/NULL |
| cancelled_time | TIMESTAMP | Yes | |
| cancellation_reason_code | VARCHAR(50) | Yes | free text |

### 3.10 Payment
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| payment_id | BIGINT | No | PK |
| ride_id | BIGINT | No | FK → Ride |
| amount | DECIMAL(10,2) | No | |
| payment_mode | VARCHAR(20) | No | card/wallet/cash/upi |
| payment_status | VARCHAR(20) | No | success/failed/refunded |
| transaction_time | TIMESTAMP | No | |
| driver_earning | DECIMAL(10,2) | No | |
| platform_commission | DECIMAL(10,2) | No | |

### 3.11 Rating
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| rating_id | BIGINT | No | PK |
| ride_id | BIGINT | No | FK → Ride |
| rater_id | INT | No | |
| rater_type | VARCHAR(10) | No | rider/driver |
| ratee_id | INT | No | |
| ratee_type | VARCHAR(10) | No | driver/rider |
| rating_value | TINYINT | No | CHECK (1–5) |
| rating_comment | VARCHAR(500) | Yes | |
| rating_time | TIMESTAMP | No | |

### 3.12 Geo_Location
| Column | Data Type | Nullable | Key |
|---|---|---|---|
| geo_id | INT | No | PK |
| latitude | DECIMAL(9,6) | No | |
| longitude | DECIMAL(9,6) | No | |
| city | VARCHAR(50) | No | |
| zone_name | VARCHAR(50) | Yes | |
| geohash | VARCHAR(12) | Yes | |

---

## 4. Physical Model — Star Schema

### 4.1 Design Principles
- Grain is explicitly defined for every fact table to prevent fan-out errors.
- All dimension keys are surrogate keys (INT/BIGINT), decoupled from source system natural keys.
- SCD Type 2 dimensions (`Dim_Driver_Onboarding`, `Dim_Vehicle_Compliance`) carry `is_current` for easy latest-state filtering.
- Fact tables contain measures and foreign keys only — no descriptive attributes.
- Date and time are combined into a single `Dim_Date` table at **hourly grain** (one row per hour per day) rather than separate Date/Time dimensions, to simplify joins while retaining time-of-day analysis capability.

### 4.2 Fact Tables

#### Fact_Ride
**Grain: one row = one ride request** (regardless of outcome — completed, cancelled, or no driver found). Cancellation attributes are merged in here rather than modeled as a separate fact, since a ride has at most one cancellation event (same grain).

| Column | Data Type | Type | Notes |
|---|---|---|---|
| ride_key | BIGINT | PK (surrogate) | |
| ride_id | BIGINT | Degenerate dim | natural key |
| rider_key | INT | FK → Dim_Rider | |
| driver_key | INT | FK → Dim_Driver | nullable |
| vehicle_key | INT | FK → Dim_Vehicle | nullable, snapshot |
| pickup_geo_key | INT | FK → Dim_Geo_Location | |
| drop_geo_key | INT | FK → Dim_Geo_Location | nullable |
| datetime_key | BIGINT | FK → Dim_Date | derived from request_time |
| ride_status | VARCHAR(30) | Degenerate dim | |
| cancelled_by | VARCHAR(10) | Degenerate dim | nullable |
| cancellation_reason_code | VARCHAR(50) | Degenerate dim | nullable |
| distance_km | DECIMAL(6,2) | Measure | nullable |
| duration_min | INT | Measure | nullable |
| fare_amount | DECIMAL(10,2) | Measure | nullable |
| surge_multiplier | DECIMAL(3,2) | Measure | nullable |
| wait_time_min | INT | Measure | derived: pickup_time − offered_time |
| is_cancelled | BOOLEAN | Measure (flag) | |
| is_completed | BOOLEAN | Measure (flag) | |

#### Fact_Payment
**Grain: one row = one payment transaction.** Rider/driver/date keys are duplicated from Fact_Ride (rather than joining fact-to-fact) so this table can be queried standalone.

| Column | Data Type | Type | Notes |
|---|---|---|---|
| payment_key | BIGINT | PK (surrogate) | |
| payment_id | BIGINT | Degenerate dim | |
| ride_id | BIGINT | Degenerate dim | traceability to Fact_Ride |
| rider_key | INT | FK → Dim_Rider | |
| driver_key | INT | FK → Dim_Driver | |
| datetime_key | BIGINT | FK → Dim_Date | derived from transaction_time |
| payment_mode | VARCHAR(20) | Degenerate dim | |
| payment_status | VARCHAR(20) | Degenerate dim | |
| amount | DECIMAL(10,2) | Measure | |
| driver_earning | DECIMAL(10,2) | Measure | |
| platform_commission | DECIMAL(10,2) | Measure | |

#### Fact_Rating
**Grain: one row = one rating event** (rider→driver OR driver→rider).

> **Open design item**: `rater_key`/`ratee_key` are polymorphic — a rater/ratee can be either a rider or a driver, which is a role-playing dimension problem. Two resolution options were discussed:
> - **Option A (recommended for gender-bias KPI)**: introduce a unified `Dim_Person` (merging Rider + Driver attributes like gender) so both keys point to one dimension — makes gender-pairing analysis a single join.
> - **Option B**: keep Dim_Rider/Dim_Driver separate and denormalize `rater_gender`/`ratee_gender` directly onto Fact_Rating at ETL time.
> This decision is still pending confirmation.

| Column | Data Type | Type | Notes |
|---|---|---|---|
| rating_key | BIGINT | PK (surrogate) | |
| rating_id | BIGINT | Degenerate dim | |
| ride_id | BIGINT | Degenerate dim | traceability |
| rater_key | INT | FK → Dim_Rider or Dim_Driver* | *pending Dim_Person decision |
| ratee_key | INT | FK → Dim_Rider or Dim_Driver* | *pending Dim_Person decision |
| rater_type | VARCHAR(10) | Degenerate dim | rider/driver |
| ratee_type | VARCHAR(10) | Degenerate dim | driver/rider |
| datetime_key | BIGINT | FK → Dim_Date | |
| rating_value | TINYINT | Measure | 1–5 |

### 4.3 Dimension Tables

#### Dim_Rider
| Column | Type | Notes |
|---|---|---|
| rider_key (PK) | INT | surrogate |
| rider_id | INT | natural key |
| name | VARCHAR(100) | |
| gender | VARCHAR(10) | |
| email | VARCHAR(100) | |
| phone | VARCHAR(15) | |
| signup_date | DATE | |
| home_city | VARCHAR(50) | |
| status | VARCHAR(20) | |

#### Dim_Driver
| Column | Type | Notes |
|---|---|---|
| driver_key (PK) | INT | surrogate |
| driver_id | INT | natural key |
| name | VARCHAR(100) | |
| gender | VARCHAR(10) | |
| phone | VARCHAR(15) | |
| email | VARCHAR(100) | |
| city | VARCHAR(50) | |
| current_status | VARCHAR(20) | |

#### Dim_Driver_Onboarding (SCD2)
| Column | Type | Notes |
|---|---|---|
| onboarding_key (PK) | BIGINT | surrogate |
| driver_key | INT | FK → Dim_Driver |
| bg_check_status | VARCHAR(20) | |
| bg_check_date | DATE | |
| doc_check_status | VARCHAR(20) | |
| doc_check_date | DATE | |
| activation_status | VARCHAR(20) | |
| activation_date | DATE | |
| deactivation_reason | VARCHAR(200) | |
| effective_start_date | DATE | |
| effective_end_date | DATE | nullable |
| is_current | BOOLEAN | |

#### Dim_Vehicle
| Column | Type | Notes |
|---|---|---|
| vehicle_key (PK) | INT | surrogate |
| vehicle_id | INT | natural key |
| make | VARCHAR(50) | |
| model | VARCHAR(50) | |
| year | SMALLINT | |
| plate_number | VARCHAR(20) | |
| vehicle_type | VARCHAR(20) | |
| ownership_type | VARCHAR(20) | |
| owner_key | INT | FK → Dim_Vehicle_Owner |

#### Dim_Vehicle_Owner
| Column | Type | Notes |
|---|---|---|
| owner_key (PK) | INT | surrogate |
| owner_id | INT | natural key |
| owner_name | VARCHAR(100) | |
| owner_type | VARCHAR(20) | |
| city | VARCHAR(50) | |

#### Dim_Vehicle_Compliance (SCD2)
| Column | Type | Notes |
|---|---|---|
| compliance_key (PK) | BIGINT | surrogate |
| vehicle_key | INT | FK → Dim_Vehicle |
| insurance_status | VARCHAR(20) | |
| insurance_expiry_date | DATE | |
| puc_status | VARCHAR(20) | |
| puc_expiry_date | DATE | |
| rc_expiry_date | DATE | |
| fitness_expiry_date | DATE | |
| effective_start_date | DATE | |
| effective_end_date | DATE | nullable |
| is_current | BOOLEAN | |

#### Dim_Geo_Location
| Column | Type | Notes |
|---|---|---|
| geo_key (PK) | INT | surrogate |
| geo_id | INT | natural key |
| latitude | DECIMAL(9,6) | |
| longitude | DECIMAL(9,6) | |
| city | VARCHAR(50) | |
| zone_name | VARCHAR(50) | |
| geohash | VARCHAR(12) | |

#### Dim_Date (merged date + time, hourly grain)
**Grain: one row = one hour of one calendar day** (8,760 rows/year).

| Column | Type | Notes |
|---|---|---|
| datetime_key (PK) | BIGINT | format YYYYMMDDHH |
| full_date | DATE | |
| day_of_week | VARCHAR(10) | |
| is_weekend | BOOLEAN | |
| day_of_month | TINYINT | |
| month | TINYINT | |
| month_name | VARCHAR(15) | |
| quarter | TINYINT | |
| year | SMALLINT | |
| week_of_year | TINYINT | |
| is_holiday | BOOLEAN | |
| month_year_label | VARCHAR(20) | e.g. "Jun 2024" |
| hour_24 | TINYINT | 0–23 |
| am_pm | VARCHAR(2) | |
| time_bucket | VARCHAR(20) | Morning Peak / Evening Peak / Off-Peak / Night |
| is_peak_hour | BOOLEAN | |
| shift_period | VARCHAR(20) | Morning/Afternoon/Evening/Night |

### 4.4 Star Schema Relationship Summary
- Fact_Ride → Dim_Rider, Dim_Driver, Dim_Vehicle, Dim_Geo_Location (pickup + drop), Dim_Date
- Fact_Payment → Dim_Rider, Dim_Driver, Dim_Date
- Fact_Rating → Dim_Rider/Dim_Driver (rater, ratee — pending Dim_Person decision), Dim_Date
- Dim_Driver → Dim_Driver_Onboarding (1:M, SCD2)
- Dim_Vehicle → Dim_Vehicle_Compliance (1:M, SCD2)
- Dim_Vehicle → Dim_Vehicle_Owner (M:1)

---

## 5. Open Items Carried Forward
1. **Dim_Person decision**: whether to unify Dim_Rider and Dim_Driver into a single Dim_Person for clean handling of Fact_Rating's polymorphic rater/ratee roles (recommended, pending final confirmation).
2. **KPI SQL definitions**: to be built in a later phase (monthly driver revenue, cancellation rate, gender-bias rating analysis, driver utilization, average trip duration, rider retention).
3. **AI/ML use case data prep**: to be scoped separately (e.g., ETA prediction, surge prediction, fraud detection, churn prediction, demand forecasting).

---

