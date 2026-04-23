# Airbnb Snowflake DBT Data Engineering Project

A production-style ELT pipeline built with **dbt** and **Snowflake** that transforms raw Airbnb data into analytics-ready tables using the **Medallion Architecture** (Bronze → Silver → Gold).

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Setup & Installation](#setup--installation)
- [Environment Variables](#environment-variables)
- [Models](#models)
- [Macros](#macros)
- [Snapshots](#snapshots)
- [Testing](#testing)
- [How to Run](#how-to-run)
- [Source Data](#source-data)

---

## Overview

This project ingests raw Airbnb listing, booking, and host data into Snowflake and transforms it through three layers of refinement — Bronze, Silver, and Gold — before serving it to analytics consumers. It demonstrates key data engineering concepts including:

- Incremental data loading
- Data quality testing
- Slowly Changing Dimensions (SCD Type 2)
- Reusable macro-based transformations
- Secure credential management via environment variables

---

## Tech Stack

| Tool | Version | Role |
|---|---|---|
| dbt Core | 1.11.8+ | Transformation orchestration |
| dbt-snowflake | 1.11.4+ | Snowflake adapter |
| Snowflake | - | Cloud data warehouse |
| Python | 3.12+ | Runtime environment |
| Jinja2 | via dbt | SQL templating and macros |
| UV | - | Python package manager |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    SOURCE DATA (CSV)                    │
│          bookings.csv  listings.csv  hosts.csv          │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│               STAGING LAYER (Snowflake)                 │
│       staging.bookings  staging.hosts  staging.listings │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│              BRONZE LAYER  (schema: bronze)             │
│   Raw incremental ingestion — no transformations        │
│   bronze_bookings  bronze_hosts  bronze_listings        │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│              SILVER LAYER  (schema: silver)             │
│   Cleaned, deduplicated, business logic applied         │
│   silver_bookings  silver_hosts  silver_listings        │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│               GOLD LAYER  (schema: gold)                │
│   Denormalized analytics-ready tables                   │
│            obt (One Big Table)   fact                   │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    SNAPSHOTS                            │
│   SCD Type 2 history tracking with valid_from/valid_to  │
│   dim_bookings   dim_hosts   dim_listings               │
└─────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
Airbnb_Snowflake_DBT_Data_Engineer_Project/
│
├── SourceData/                            # Raw CSV source files
│   ├── bookings.csv                       # ~9,000 booking records
│   ├── listings.csv                       # ~500 property listings
│   └── hosts.csv                          # Host profiles
│
└── aws_dbt_snwoflake_project/             # Main dbt project
    ├── dbt_project.yml                    # Core dbt configuration
    ├── profiles.yml                       # Snowflake connection (uses env vars)
    ├── .env                               # Local credentials (never committed)
    │
    ├── models/
    │   ├── sources.yml                    # Source table definitions + tests
    │   ├── bronze/                        # Layer 1: Raw incremental ingestion
    │   │   ├── bronze_bookings.sql
    │   │   ├── bronze_hosts.sql
    │   │   └── bronze_listings.sql
    │   ├── silver/                        # Layer 2: Cleaned & transformed
    │   │   ├── silver_bookings.sql
    │   │   ├── silver_hosts.sql
    │   │   ├── silver_listings.sql
    │   │   └── schema.yml                 # Generic data quality tests
    │   └── gold/                          # Layer 3: Analytics-ready
    │       ├── obt.sql                    # One Big Table
    │       ├── fact.sql                   # Fact table
    │       └── ephemeral/                 # Reusable CTEs (not persisted)
    │           ├── bookings.sql
    │           ├── hosts.sql
    │           └── listings.sql
    │
    ├── macros/                            # Reusable Jinja2 functions
    │   ├── multiply.sql
    │   ├── tag.sql
    │   ├── trimmer.sql
    │   └── generate_schema_name.sql
    │
    ├── snapshots/                         # SCD Type 2 tracking
    │   ├── dim_bookings.yml
    │   ├── dim_hosts.yml
    │   └── dim_listings.yml
    │
    ├── tests/                             # Singular custom SQL tests
    │   ├── assert_total_amount_positive.sql
    │   └── assert_no_cancelled_bookings.sql
    │
    └── analyses/                          # Ad-hoc exploration queries
        ├── explore.sql
        ├── if_else.sql
        └── loop.sql
```

---

## Setup & Installation

### Prerequisites
- Python 3.12+
- Snowflake account
- UV package manager

### Steps

**1. Clone the repository**
```bash
git clone https://github.com/Raghu-Redy/Airbnb-Snowflake-DBT-Data-Engineer-Project.git
cd Airbnb-Snowflake-DBT-Data-Engineer-Project
```

**2. Create and activate virtual environment**
```bash
python -m venv .venv
source .venv/Scripts/activate        # Windows
source .venv/bin/activate            # Mac/Linux
```

**3. Install dependencies**
```bash
pip install dbt-core dbt-snowflake
```

**4. Set up environment variables**
```bash
cd aws_dbt_snwoflake_project
cp .env.example .env                 # create your .env from template
# Edit .env with your Snowflake credentials
```

**5. Verify connection**
```bash
dbt debug
```

---

## Environment Variables

Credentials are stored in `.env` and never committed to GitHub.

Create a `.env` file inside `aws_dbt_snwoflake_project/` with the following:

```env
SNOWFLAKE_ACCOUNT=your_account
SNOWFLAKE_USER=your_username
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_ROLE=your_role
SNOWFLAKE_DATABASE=AIRBNB
SNOWFLAKE_WAREHOUSE=COMPUTE_WH
SNOWFLAKE_SCHEMA=dbt_demo
```

Load the environment variables before running dbt:

```bash
# Mac/Linux/Git Bash
source .env

# Windows PowerShell
Get-Content .env | ForEach-Object {
  $name, $value = $_ -split '=', 2
  [System.Environment]::SetEnvironmentVariable($name, $value)
}
```

---

## Models

### Bronze Layer — Raw Ingestion

Incremental tables that ingest source data as-is with no transformations. Only new records are loaded on each run using timestamp-based incremental logic.

| Model | Source Table | Unique Key | Materialization |
|---|---|---|---|
| bronze_bookings | staging.bookings | BOOKING_ID | Incremental |
| bronze_hosts | staging.hosts | HOST_ID | Incremental |
| bronze_listings | staging.listings | LISTING_ID | Incremental |

### Silver Layer — Cleaned & Transformed

Deduplicated models with business logic, calculated fields, and standardized values.

| Model | Key Transformations | Unique Key |
|---|---|---|
| silver_bookings | `total_amount = nights × rate` via `multiply()` macro | BOOKING_ID |
| silver_hosts | Response rate categorized: VERY GOOD / GOOD / FAIR / POOR | HOST_ID |
| silver_listings | Price tagged as low / medium / high via `tag()` macro | LISTING_ID |

### Gold Layer — Analytics Ready

Denormalized tables that join silver models for direct consumption by BI tools and analysts.

| Model | Type | Description |
|---|---|---|
| obt.sql | Table | One Big Table — full denormalized fact with all dimensions |
| fact.sql | Table | Fact table with dimension joins |
| ephemeral/ | Ephemeral | Reusable CTEs — not persisted in Snowflake |

---

## Macros

Reusable Jinja2 functions defined in the `macros/` folder.

| Macro | Syntax | Purpose |
|---|---|---|
| `multiply` | `{{ multiply('col1', 'col2') }}` | Multiplies two columns, rounds to 2 decimal places |
| `tag` | `{{ tag('PRICE_PER_NIGHT') }}` | Categorizes price: low (<100), medium (100-199), high (>=200) |
| `trimmer` | `{{ trimmer('col') }}` | Trims whitespace and uppercases string columns |
| `generate_schema_name` | auto | Custom schema routing for Bronze/Silver/Gold layers |

---

## Snapshots

Slowly Changing Dimension (SCD) Type 2 tracking. Captures historical changes in all three entities with `valid_from` and `valid_to` timestamps.

| Snapshot | Unique Key | Timestamp Column | Current Record Flag |
|---|---|---|---|
| dim_bookings | BOOKING_ID | CREATED_AT | valid_to = 9999-12-31 |
| dim_hosts | HOST_ID | HOST_CREATED_AT | valid_to = 9999-12-31 |
| dim_listings | LISTING_ID | LISTING_CREATED_AT | valid_to = 9999-12-31 |

---

## Testing

### Generic Tests — schema.yml

Defined in `models/silver/schema.yml`. dbt generates and executes the SQL automatically.

| Model | Column | Tests |
|---|---|---|
| silver_bookings | BOOKING_ID | unique, not_null |
| silver_bookings | LISTING_ID | not_null, relationships → silver_listings |
| silver_bookings | BOOKING_STATUS | accepted_values: confirmed, cancelled, pending |
| silver_bookings | TOTAL_AMOUNT | not_null |
| silver_listings | LISTING_ID | unique, not_null |
| silver_listings | PRICE_PER_NIGHT_TAG | accepted_values: low, medium, high |
| silver_listings | HOST_ID | not_null, relationships → silver_hosts |
| silver_hosts | HOST_ID | unique, not_null |
| silver_hosts | RESPONSE_RATE_QUALITY | accepted_values: VERY GOOD, GOOD, FAIR, POOR |

### Singular Tests — tests/

Custom SQL files. A test **passes if the query returns 0 rows**.

| Test File | Severity | What it checks |
|---|---|---|
| assert_total_amount_positive.sql | error | total_amount must be greater than 0 |
| assert_no_cancelled_bookings.sql | warn | monitors cancelled bookings without breaking pipeline |

### Test Severity

| Severity | Pipeline behaviour |
|---|---|
| `error` | Pipeline stops — must fix before proceeding |
| `warn` | Pipeline continues — issue is flagged for review |

---

## How to Run

```bash
# Check Snowflake connection
dbt debug

# Run all models
dbt run

# Run a specific layer
dbt run --select bronze.*
dbt run --select silver.*
dbt run --select gold.*

# Run a specific model
dbt run --select silver_bookings

# Run all tests
dbt test

# Run tests for a specific model
dbt test --select silver_bookings

# Run models and tests together (recommended for production)
dbt build

# Inspect failing rows in Snowflake
dbt test --store-failures

# Run snapshots
dbt snapshot

# Generate and serve documentation
dbt docs generate
dbt docs serve
```

---

## Source Data

Raw CSV files stored in `SourceData/` and loaded into Snowflake staging tables.

| File | Approx Rows | Key Columns |
|---|---|---|
| bookings.csv | ~9,000 | booking_id, listing_id, nights_booked, booking_amount, booking_status, created_at |
| listings.csv | ~500 | listing_id, host_id, property_type, room_type, city, price_per_night, created_at |
| hosts.csv | - | host_id, host_name, is_superhost, response_rate, host_since |

---

## Author

**Raghu Kanukula**
GitHub: [Raghu-Redy](https://github.com/Raghu-Redy)
