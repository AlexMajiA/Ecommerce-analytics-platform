# Ecommerce Analytics Platform

## Overview

End-to-end **Data Engineering** project built with **Snowflake** and **dbt**, using the public [Olist Brazilian E-Commerce dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce).

The project implements an **ELT pipeline**, from raw ingestion through analytical models, with a focus on:

- Layered data architecture
- Dimensional modeling
- Reusable dbt transformations
- Data quality and business-rule testing
- Historical tracking with dbt snapshots
- Customer and sales analytics
- NLP enrichment with Snowflake Cortex AI

The goal is not simply to transform the source data, but to build a maintainable analytical data platform where business-facing models are separated from technical transformations.

---

## Architecture

### Technology stack

| Layer | Technology |
|---|---|
| Cloud data warehouse | Snowflake |
| Transformation | dbt |
| Ingestion | AWS S3 → Snowflake |
| Modeling | Dimensional / Kimball-style |
| AI / NLP | Snowflake Cortex |
| Package management | dbt packages |

### Data flow

```text
Olist CSV datasets
       │
       ▼
AWS S3
       │
       ▼
Snowflake RAW / Bronze
       │
       ▼
STAGING / Silver
(cleaning, standardization, casting)
       │
       ▼
INTERMEDIATE / Silver
(reusable business transformations)
       │
       ├───────────────┐
       ▼               ▼
CORE / Gold        AI Enrichment
facts & dimensions   Snowflake Cortex
       │               │
       └───────┬───────┘
               ▼
         ANALYTICS / Gold
        business-facing marts
```

### Snowflake layers

- **RAW / Bronze** — source data loaded into Snowflake.
- **STAGING / Silver** — cleaning, normalization and type casting.
- **INTERMEDIATE / Silver** — reusable transformations shared by downstream models.
- **CORE / Gold** — fact and dimension models.
- **ANALYTICS / Gold** — business-oriented marts for reporting and analysis.

---

## Data Model

The core analytical layer follows a dimensional modeling approach.

### Dimensions

- `dim_customers`
- `dim_products`
- `dim_seller`
- `dim_date`

### Fact

- `fct_sales`

The grain of `fct_sales` is **one row per order item** (`order_id` + `order_item_id`).

Foreign-key relationships are tested against the relevant dimensions, and the fact grain is validated with a composite uniqueness test.

---

## Analytical Marts

### Sales analytics

- **`mart_sales_daily`** — daily sales KPIs.
- **`mart_sales_by_state`** — sales performance by Brazilian state.
- **`mart_top_products`** — product-level performance.
- **`mart_customer_base`** — customer classification over time.
- **`mart_customer_rfm`** — RFM segmentation using Recency, Frequency and Monetary value.
- **`mart_price_history`** — historical product-price analysis.

### Customer review analytics

- **`mart_review_sentiment`** — sentiment and review metrics by product, seller and period.
- **`mart_category_sentiment`** — sentiment analysis aggregated by product category.

These marts provide business-oriented outputs rather than exposing the underlying transformation logic directly.

---

## Snowflake Cortex AI

A dedicated intermediate model, `int_order_reviews_enriched`, enriches Olist customer reviews using Snowflake Cortex.

The source contains customer reviews in Portuguese. The pipeline:

1. Translates non-null review text from Portuguese to English with `AI_TRANSLATE`.
2. Calculates sentiment with `SNOWFLAKE.CORTEX.SENTIMENT`.
3. Publishes the enriched result as a table for downstream marts.

### Why the enrichment is materialized as a table

The Cortex functions are computationally and credit-sensitive operations. Materializing the enriched dataset avoids recalculating the same AI transformation every time a downstream query reads the model.

The enrichment is also centralized in the **intermediate layer**, so multiple analytical marts consume one transformed source rather than repeating the AI computation.

NULL review text is handled explicitly before calling Cortex functions.

> For Snowflake accounts where Cortex cross-region inference is required, the account must be configured accordingly.

---

## Historical Tracking

The project includes a dbt snapshot for order-item history:

`snapshot_order_items`

It uses:

- **Timestamp strategy**
- Composite business key: `order_id` + `order_item_id`
- `ingest_timestamp` as the update timestamp

This provides a simple example of preserving historical changes instead of maintaining only the latest state.

---

## Data Quality

Data quality is handled at both the schema and business-rule level.

### Schema tests

- `not_null`
- `unique`
- `accepted_values`
- `relationships`
- `dbt_utils.unique_combination_of_columns`

### Singular tests

The project also includes SQL tests for business rules and analytical consistency, including:

- Duplicate customer/month combinations
- Negative revenue
- Invalid order counts
- Sales item/order consistency
- Revenue consistency
- Review score ranges
- Sentiment score ranges
- RFM score ranges
- Top-product consistency

The intention is to validate not only whether columns contain values, but whether the resulting analytical models make business sense.

---

## dbt Design Patterns

### Separation of concerns

The project separates:

```text
staging → intermediate → core → analytics
```

This keeps source cleanup, reusable transformations, dimensional modeling and business-facing logic independent.

### DRY transformations

Reusable calculations are performed once in intermediate models and then referenced by downstream marts.

### Development safeguard

A custom macro, `limit_dev`, limits selected intermediate development models when the active dbt target is `dev`. This makes local experimentation safer and reduces unnecessary warehouse processing.

---

## Project Structure

```text
.
├── models/
│   ├── staging/
│   │   ├── sources.yml
│   │   └── stg_*.sql
│   │
│   └── marts/
│       └── core/
│           ├── core.yml
│           ├── intermediate/
│           │   ├── intermediate.yml
│           │   └── int_*.sql
│           │
│           ├── sales/
│           │   ├── dim_*.sql
│           │   └── fct_sales.sql
│           │
│           └── analytics/
│               ├── sales/
│               │   └── mart_*.sql
│               └── sentiment/
│                   ├── mart_review_sentiment.sql
│                   └── mart_category_sentiment.sql
│
├── snapshots/
│   └── snapshot_order_items.sql
│
├── tests/
│   └── test_*.sql
│
├── macros/
│   └── limit_dev.sql
│
├── packages.yml
└── dbt_project.yml
```

---

## Running the project

The original project was developed against a private Snowflake environment, so the warehouse and source data are not included in this repository.

To reproduce the project, you need:

1. A Snowflake account.
2. The Olist dataset loaded into a RAW layer.
3. A dbt profile configured for Snowflake.
4. The required dbt packages installed.

Then:

```bash
dbt deps
dbt build
```

You can also run the standard workflow separately:

```bash
dbt run
dbt test
```

The repository does **not** contain Snowflake credentials or private connection configuration.

---

## Project Status

The core pipeline and analytical models are implemented.

Possible future extensions:

- Power BI dashboard consuming the Gold layer
- More incremental processing patterns
- Additional tests for analytical marts
- Expanded model documentation
- Further performance optimization

---

## Why this project matters

This project demonstrates the full path from raw operational data to business-ready analytical data:

```text
Raw data
   ↓
Cleaning & standardization
   ↓
Reusable transformations
   ↓
Dimensional model
   ↓
Business marts
   ↓
Data quality
   ↓
AI-enriched analytics
```

The emphasis is on **architecture, data modeling, maintainability and business-oriented analytical design**, rather than on isolated SQL transformations.
