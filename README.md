# 🎬 Netflix Data Engineering Project with dbt + Snowflake

[![dbt](https://img.shields.io/badge/dbt-1.0+-orange.svg)](https://docs.getdbt.com/)
[![Snowflake](https://img.shields.io/badge/Snowflake-Data%20Platform-blue.svg)](https://www.snowflake.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 📌 Overview
This project implements a **modern data stack** using **Snowflake** and **dbt** to model and analyze the [MovieLens dataset](https://grouplens.org/datasets/movielens/).  

It follows a **layered architecture**:
1. **Raw layer (staging in Snowflake)** → Data is ingested from CSV files stored in S3.
2. **Staging models (`src_*`)** → Standardize raw tables into consistent column names and formats.
3. **Dimension models (`dim_*`)** → Define descriptive entities like movies, users, and tags.
4. **Fact models (`fct_*`)** → Capture business events such as ratings and genome scores.
5. **Mart models (`mart_*`)** → Deliver analytics-ready datasets for BI and reporting.

---

## 🏗️ Project Structure
```
netflix/
├── models/
│   ├── staging/                # Cleaned sources from Snowflake RAW schema
│   │   ├── src_movies.sql
│   │   ├── src_ratings.sql
│   │   ├── src_tags.sql
│   │   ├── src_genome_score.sql
│   │   ├── src_genome_tags.sql
│   │   └── src_links.sql
│   │
│   ├── dim/                    # Dimension tables
│   │   ├── dim_movies.sql
│   │   ├── dim_users.sql
│   │   ├── dim_genome_tags.sql
│   │   └── dim_genomes_with_tags.sql
│   │
│   ├── fct/                    # Fact tables
│   │   ├── fct_ratings.sql
│   │   └── fct_genome_scores.sql
│   │
│   ├── mart/                   # Analytics-ready marts
│   │   └── mart_movie_releases.sql
│   │
│   └── sources.yml             # Source definitions for Snowflake RAW tables
│
├── seeds/                      # CSV seed files (e.g. movie release dates)
├── snapshots/                  # Historical tracking (if needed)
├── macros/                     # Custom dbt macros
├── tests/                      # Data quality tests
└── dbt_project.yml            # Project configuration
```

<img width="1918" height="985" alt="image" src="https://github.com/user-attachments/assets/c77e90cf-00db-495c-984d-d706be7c1bb0" />

---

## ⚙️ Snowflake Setup

Run the following SQL script to prepare Snowflake:

```sql
-- Create role, warehouse, user, and database
USE ROLE ACCOUNTADMIN;

CREATE ROLE IF NOT EXISTS TRANSFORM;
GRANT ROLE TRANSFORM TO ROLE ACCOUNTADMIN;

CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH;

CREATE USER IF NOT EXISTS dbt
  PASSWORD='dbtPassword123'
  DEFAULT_ROLE=TRANSFORM
  DEFAULT_WAREHOUSE='COMPUTE_WH'
  DEFAULT_NAMESPACE='MOVIELENS.RAW';

GRANT ROLE TRANSFORM TO USER dbt;

CREATE DATABASE IF NOT EXISTS MOVIELENS;
CREATE SCHEMA IF NOT EXISTS MOVIELENS.RAW;

GRANT ALL ON DATABASE MOVIELENS TO ROLE TRANSFORM;
GRANT ALL ON SCHEMA MOVIELENS.RAW TO ROLE TRANSFORM;
```

Data is loaded from Amazon S3 into Snowflake tables using `COPY INTO` commands (see `scripts/` folder if included).

---

## 📊 Models Breakdown

### 🔹 Staging (models/staging/)
- **`src_movies.sql`** → Standardizes RAW_MOVIES.
- **`src_ratings.sql`** → Converts raw ratings with proper timestamps.
- **`src_tags.sql`** → Cleans user-generated tags.
- **`src_genome_score.sql`** → Extracts movie–tag relevance scores.
- **`src_genome_tags.sql`** → Standardizes genome tag labels.
- **`src_links.sql`** → Maps movie IDs to IMDB and TMDB.

### 🔹 Dimensions (models/dim/)
- **`dim_movies.sql`** → Clean movie titles, genres, and create genre_array.
- **`dim_users.sql`** → Consolidates unique users from ratings and tags.
- **`dim_genome_tags.sql`** → Human-readable tag names.
- **`dim_genomes_with_tags.sql`** → Combines movies, tags, and scores (ephemeral model).

### 🔹 Facts (models/fct/)
- **`fct_ratings.sql`** → Incremental model of user ratings with deduplication.
- **`fct_genome_scores.sql`** → Relevance scores between movies and tags.

### 🔹 Marts (models/mart/)
- **`mart_movie_releases.sql`** → Joins ratings with seeded release dates, flags whether release info is available.

---

## 📈 Data Flow

```mermaid
flowchart TD
    subgraph RAW[Snowflake RAW Schema]
        A[raw_movies] --> B[src_movies]
        C[raw_ratings] --> D[src_ratings]
        E[raw_tags] --> F[src_tags]
        G[raw_genome_scores] --> H[src_genome_score]
        I[raw_genome_tags] --> J[src_genome_tags]
        K[raw_links] --> L[src_links]
    end

    B --> M[dim_movies]
    J --> N[dim_genome_tags]
    D --> O[dim_users]
    F --> O
    H --> P[fct_genome_scores]
    D --> Q[fct_ratings]

    M --> R[dim_genomes_with_tags]
    N --> R
    P --> R

    Q --> S[mart_movie_releases]
```

---

## ▶️ How to Run

### 1. Install dependencies:
```bash
pip install dbt-snowflake
```

### 2. Configure your `profiles.yml`:
```yaml
netflix:
  outputs:
    dev:
      type: snowflake
      account: <your_account>
      user: dbt
      password: dbtPassword123
      role: TRANSFORM
      warehouse: COMPUTE_WH
      database: MOVIELENS
      schema: DEV
      threads: 1
  target: dev
```

### 3. Run dbt commands:
```bash
dbt debug          # Test connection
dbt seed           # Load seed data (e.g. release dates)
dbt run            # Build models
dbt test           # Run data quality tests
```

---

## 🛠️ Additional dbt Features


### Snapshots

Snapshots in dbt allow you to **track changes in your data over time**, creating a historical record for auditing or slowly changing dimensions.

**Example Snapshot**: `snap_tags`

```sql
{% snapshot snap_tags %}

{{
    config(
        target_schema='snapshots',
        unique_key=['user_id','movie_id','tag'],
        strategy='timestamp',
        updated_at='tag_timestamp',
        invalidate_hard_deletes=True
    )
}}

SELECT
{{ dbt_utils.generate_surrogate_key(['user_id','movie_id','tag']) }} AS row_key,
    user_id,
    movie_id,
    tag,
    CAST(tag_timestamp AS TIMESTAMP_NTZ) AS tag_timestamp
FROM {{ ref('src_tags') }}
LIMIT 100 

{% endsnapshot %}
```

### Tests

This project includes **custom and built-in dbt tests** to ensure data quality and consistency.

**Custom Test Example**: `tests/high_relevance_score.sql`

```sql
SELECT
    movie_id,
    tag_id,
    relevance_score
FROM {{ ref('fct_genome_scores') }}
WHERE relevance_score > 3

```

### Seeds
**Seeds** are CSV files that contain static data loaded directly into your data warehouse. In this project, they're stored in the `seeds/` directory.

```bash
dbt seed           # Load all seed files
dbt seed --select movie_release_dates  # Load specific seed
```

Seeds are useful for:
- Reference data (e.g., movie release dates)
- Lookup tables
- Small dimensional data

### Views vs Tables in dbt
By default, dbt creates **views** in your data warehouse. You can control this behavior in your model files:

```sql
-- Create as a table
{{ config(materialized='table') }}
SELECT * FROM {{ ref('src_movies') }}

-- Create as a view (default)
{{ config(materialized='view') }}
SELECT * FROM {{ ref('src_movies') }}

-- Create as an incremental table
{{ config(materialized='incremental') }}
SELECT * FROM {{ ref('src_ratings') }}
```

**Views** are virtual tables that don't store data, while **tables** store actual data in your warehouse.

### Generate Documentation
dbt automatically generates documentation for your project:

```bash
dbt docs generate    # Generate documentation
dbt docs serve       # Serve documentation locally (localhost:8080)
```

The documentation includes:
- **Lineage graphs** showing data flow between models
- **Column descriptions** and data types
- **Test results** and freshness checks
- **Source definitions** and their relationships

### Model Materialization Types
- **`view`** → Virtual table (default)
- **`table`** → Physical table stored in warehouse
- **`incremental`** → Append/update only new records
- **`ephemeral`** → CTE that exists only during query compilation

---

## ✅ Key Features

- ✅ **Snowflake + dbt pipeline** with layered modeling
- ✅ **Incremental loads** for ratings (`fct_ratings`)
- ✅ **Ephemeral models** for performance (`dim_genomes_with_tags`)
- ✅ **Seeds support** for static reference data
- ✅ **Data quality tests** and documentation generation
- ✅ **Flexible materialization** (views, tables, incremental)

---

## 📚 References

- [dbt Documentation](https://docs.getdbt.com/)
- [Snowflake Docs](https://docs.snowflake.com/)
- [MovieLens Dataset](https://grouplens.org/datasets/movielens/)

---

## 👨‍💻 Author
**Gerardo Hernandez Widman**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue.svg)](https://linkedin.com/in/your-profile)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black.svg)](https://github.com/your-username)
