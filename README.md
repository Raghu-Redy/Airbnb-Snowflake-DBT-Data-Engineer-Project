# Airbnb Snowflake DBT Data Engineering Project

## Overview

A dbt (data build tool) project that transforms raw Airbnb listing, booking, and host data stored in Snowflake into analytics-ready tables. Implements the Medallion Architecture (Bronze → Silver → Gold) for layered data transformation.

---

## Tech Stack

| Tool | Version | Role |
|---|---|---|
| dbt | 1.11.8+ | Transformation orchestration |
| dbt-snowflake | 1.11.4+ | Snowflake adapter |
| Snowflake | - | Cloud data warehouse |
| Python | 3.12+ | Runtime environment |
| Jinja2 | via dbt | SQL templating |

---

## Architecture — Medallion Layers

```
SourceData/ (CSV files)
    ↓
staging.bookings / hosts / listings   ← Raw Snowflake staging tables
    ↓
BRONZE LAYER  (incremental ingestion, schema: bronze)
    ↓
SILVER LAYER  (cleaned + transformed, schema: silver)
    ↓
GOLD LAYER    (analytics-ready: fact + OBT tables, schema: gold)
    ↓
SNAPSHOTS     (SCD Type 2 history tracking)
```

---

## Project Structure

```
Airbnb_Snowflake_DBT_Data_Engineer_Project/
├── SourceData/                        # Raw CSV source files
│   ├── bookings.csv
│   ├── listings.csv
│   └── hosts.csv
│
└── aws_dbt_snwoflake_project/         # Main dbt project
    ├── dbt_project.yml                # Core dbt configuration
    ├── profiles.yml                   # Snowflake connection settings
    ├── models/
    │   ├── sources.yml                # Source table definitions
    │   ├── bronze/                    # Raw incremental ingestion
    │   ├── silver/                    # Cleaned and transformed
    │   │   └── schema.yml             # Generic tests for silver layer
    │   └── gold/                      # Analytics-ready tables
    │       └── ephemeral/             # Reusable CTE models
    ├── macros/                        # Reusable Jinja2 functions
    ├── snapshots/                     # SCD Type 2 configurations
    ├── tests/                         # Singular (custom SQL) tests
    └── analyses/                      # Ad-hoc exploration queries
```

---

## Models

### Bronze Layer — Raw Ingestion
Incremental tables that pull data from staging as-is.

| Model | Source | Key Column |
|---|---|---|
| bronze_bookings | staging.bookings | BOOKING_ID |
| bronze_hosts | staging.hosts | HOST_ID |
| bronze_listings | staging.listings | LISTING_ID |

### Silver Layer — Cleaned & Transformed
Deduplicated and enriched with business logic.

| Model | Transformations |
|---|---|
| silver_bookings | Calculates `total_amount` via `multiply()` macro |
| silver_hosts | Categorizes `response_rate` into VERY GOOD / GOOD / FAIR / POOR |
| silver_listings | Tags `price_per_night` as low / medium / high via `tag()` macro |

### Gold Layer — Analytics Ready
Denormalized tables joining all silver models.

| Model | Type | Description |
|---|---|---|
| obt.sql | Table | One Big Table — full denormalized fact table |
| fact.sql | Table | Fact table with dimension joins |
| ephemeral/ | Ephemeral | Reusable CTEs (not persisted in Snowflake) |

---

## Macros

| Macro | Purpose |
|---|---|
| `multiply(x, y)` | Multiplies two values and rounds to 2 decimal places |
| `tag(col)` | Categorizes price: low (<100), medium (100-199), high (>=200) |
| `trimmer()` | Trims and uppercases string columns |
| `generate_schema_name()` | Custom schema routing logic |

---

## Snapshots (SCD Type 2)

Tracks historical changes using timestamp strategy with `valid_from` / `valid_to` dates.

| Snapshot | Unique Key | Updated At |
|---|---|---|
| dim_bookings | BOOKING_ID | CREATED_AT |
| dim_hosts | HOST_ID | HOST_CREATED_AT |
| dim_listings | LISTING_ID | LISTING_CREATED_AT |

---

## Testing

### Generic Tests (schema.yml)
Defined in `models/silver/schema.yml`. dbt generates and runs the SQL automatically.

| Column | Tests Applied |
|---|---|
| BOOKING_ID | unique, not_null |
| LISTING_ID | not_null, relationships → silver_listings |
| BOOKING_STATUS | accepted_values: confirmed / cancelled / pending |
| TOTAL_AMOUNT | not_null |
| PRICE_PER_NIGHT_TAG | accepted_values: low / medium / high |
| HOST_ID | not_null, relationships → silver_hosts |
| RESPONSE_RATE_QUALITY | accepted_values: VERY GOOD / GOOD / FAIR / POOR |

### Singular Tests (tests/)
Custom SQL files. Test passes if query returns 0 rows.

| Test File | Severity | Description |
|---|---|---|
| assert_total_amount_positive.sql | error | Fails if total_amount <= 0 |
| assert_no_cancelled_bookings.sql | warn | Warns if cancelled bookings exist |

### Running Tests

```bash
# Run all tests
dbt test

# Run tests for a specific model
dbt test --select silver_bookings

# Run tests for an entire layer
dbt test --select silver.*

# Run models and tests together
dbt build

# Save failed rows to Snowflake for inspection
dbt test --store-failures
```

---

## Snowflake Connection

| Setting | Value |
|---|---|
| Account | DSWUXHV-ON96746 |
| Database | AIRBNB |
| Warehouse | COMPUTE_WH |
| Role | ACCOUNTADMIN |
| Default Schema | dbt_demo |

> Note: Store credentials as environment variables in production. Do not hardcode in profiles.yml.

---

## Source Data

| File | Rows (approx) | Description |
|---|---|---|
| bookings.csv | ~9,000 | Booking records with dates, amounts, status |
| listings.csv | ~500 | Property listings with price and location |
| hosts.csv | - | Host profiles with response rates |
