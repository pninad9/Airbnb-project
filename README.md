# Airbnb Analytics Pipeline (AWS S3 → Snowflake → dbt Bronze/Silver/Gold → Star Schema + SCD2)

## 1) Project Description
This project demonstrates an end-to-end data engineering pipeline using an Airbnb-style dataset.  
Raw CSV files are landed in AWS S3, loaded into Snowflake staging tables, and transformed using dbt into a Medallion-style architecture (Bronze/Silver/Gold).  
The Gold layer includes a metadata-driven One Big Table (OBT), followed by a Star Schema with dbt snapshots (SCD Type 2) to track historical changes in dimensions.

**Tech Stack:** AWS S3, Snowflake, dbt (incremental models, macros, lineage, snapshots)

- High-level pipeline overview
<img src="https://github.com/pninad9/Airbnb-project/blob/73c078343aaa5492c6b2cfdb6cb5e4efe01b4afd/screenshot/Architecture.png" />
---

## 2) Raw Landing (AWS S3) → Snowflake Warehouse (Ingestion Flow)
Raw source data is provided as CSV files and placed into an AWS S3 bucket path. Snowflake then ingests these files using an external stage and a CSV file format. Finally, `COPY INTO` loads the data into Snowflake staging tables.

### Step-by-step flow
1. **Land raw CSVs in S3**
   - `hosts.csv`
   - `listings.csv`
   - `bookings.csv`

2. **Create Snowflake File Format**
   - Defines how Snowflake reads CSVs (delimiter, header skip)

3. **Create Snowflake Stage pointing to S3**
   - The stage references the S3 bucket/path where CSVs are stored.

4. **Load into Snowflake tables using `COPY INTO`**
   - Data is loaded into staging tables:
     - `HOSTS`
     - `LISTINGS`
     - `BOOKINGS`

-  shows FILE FORMAT + STAGE + COPY INTO
<img src="https://github.com/pninad9/Airbnb-project/blob/73c078343aaa5492c6b2cfdb6cb5e4efe01b4afd/screenshot/Snowflake%20reasources.png" />

---

## 3) dbt Transformation Layers (Medallion Architecture)
dbt orchestrates transformations across the Medallion layers:
- **Bronze**: raw incremental ingestion (append new records)
- **Silver**: cleaning + standardization + derived columns
- **Gold**: analytics-ready tables (OBT + dimensions + fact)

dbt also provides:
- Lineage graph (docs)
- Incremental models for scalable loads
- Macros for reusable logic
- Snapshots for SCD2 history tracking

- sources → bronze → silver → obt
 <img src="https://github.com/pninad9/Airbnb-project/blob/73c078343aaa5492c6b2cfdb6cb5e4efe01b4afd/screenshot/lineage%201.png" />
---

## 4) Bronze Layer (Raw + Incremental)
The Bronze layer stores raw data coming from Snowflake staging with minimal transformation.  
Models are built as **incremental**, filtering new records using `CREATED_AT` to avoid full reloads.

**Models:**
- `bronze_hosts`
- `bronze_listings`
- `bronze_bookings`

 <img src="https://github.com/pninad9/Airbnb-project/blob/73c078343aaa5492c6b2cfdb6cb5e4efe01b4afd/screenshot/bronze_listings%20incremental%20mode.png" />

---

## 5) Silver Layer (Clean + Enriched + Incremental)
The Silver layer cleans and enriches raw Bronze data:
- Standardized naming
- Derived fields (tags/quality buckets)
- Incremental materialization using unique keys for scalable updates

Example transformations:
- `PRICE_PER_NIGHT_TAG` using a dbt macro
- Host response rate quality bucket (Very Good / Good / Fair / Poor)


- Incremental + macro + ref()
 <img src="https://github.com/pninad9/Airbnb-project/blob/73c078343aaa5492c6b2cfdb6cb5e4efe01b4afd/screenshot/silver_listings%20incremental%20model.png" />
---

## 6) Gold Layer + Fact (Metadata-Driven OBT)
The Gold layer contains analytics-ready models.

### One Big Table (OBT)
An OBT is built by joining bookings + listings + hosts into a single wide table.  
This project implements the OBT using a **metadata-driven configuration** so joins/selected columns can be controlled via a config structure, reducing repetitive SQL and improving maintainability.

### Fact Model
A fact model is then created on top of the OBT to support downstream reporting and dimensional modeling.

- Metadata-driven join pattern
 <img src="https://github.com/pninad9/Airbnb-project/blob/73c078343aaa5492c6b2cfdb6cb5e4efe01b4afd/screenshot/metadata%20driven%20obt%20model.png" />

---

## 7) Star Schema with Snapshots (SCD Type 2)
To support dimensional modeling and historical tracking:
- Ephemeral models extract dimension-shaped datasets from the OBT
- dbt **snapshots** create SCD Type 2 dimension tables
- This enables tracking changes over time (via `dbt_valid_from`, `dbt_valid_to`)

**Snapshots:**
- `dim_hosts` (SCD2)
- `dim_listings` (SCD2)
- `dim_bookings` (SCD2)


- OBT → ephemeral → snapshots dims 
 <img src="https://github.com/pninad9/Airbnb-project/blob/73c078343aaa5492c6b2cfdb6cb5e4efe01b4afd/screenshot/lineage%202.png" />

- Snapshot config: unique_key, updated_at, strategy
 <img src="https://github.com/pninad9/Airbnb-project/blob/73c078343aaa5492c6b2cfdb6cb5e4efe01b4afd/screenshot/dim_bookings%20snapshot.png" />



---

## How to Run (High-level)
1. Upload raw CSV files to S3.
2. Create Snowflake file format + stage, then load staging tables using `COPY INTO`.
3. Configure dbt profile for Snowflake.
4. Run:
   - `dbt build` (build models + run snapshots)



## Final Word

The goal of this project was to simulate how real analytics platforms are built: scalable ingestion, incremental transformations, clear modeling layers, and trusted reporting outputs. The combination of dbt incremental models, reusable macros, a metadata-driven OBT, and SCD2 snapshots makes the pipeline maintainable and ready for growth. If you’re hiring for Data Engineering / Analytics Engineering roles, I’d love to discuss the design choices and trade-offs behind this build.
