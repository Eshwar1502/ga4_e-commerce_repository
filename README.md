-- definitions/documentation.md
# GA4 E-commerce Analytics Pipeline Project

## Overview
This Dataform project transforms raw Google Analytics 4 (GA4) e-commerce sample data into structured Bronze, Silver, and Gold layers for business reporting. It is designed to run regularly to provide updated monthly reports on traffic source performance.

## Data Lineage
The pipeline follows a standard Medallion Architecture:
1.  **Bronze:** External data reference.
2.  **Silver:** Cleaned, denormalized transactions.
3.  **Gold:** Aggregated business report.

---

### Layer Details

#### 1. Bronze Layer
* **Object:** `events_source` (Declaration)
* **Source:** `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
* **Purpose:** Provides a consistent reference to the raw, sharded GA4 public dataset.

#### 2. Silver Layer (Clean & Denormalized)
* **Object:** `purchase_traffic_source_medium` (Table)
* **Logic:** Filters the Bronze source to include only `purchase` events. It performs an essential cleaning step by removing rows where the traffic medium is marked as `(data deleted)`. It extracts `ga_session_id` from event parameters.
* **Key Columns:** `event_date`, `medium`, `user_pseudo_id`, `ga_session_id`, `event_value_in_usd`.

#### 3. Gold Layer (Business Report)
* **Object:** `top_traffic_source_medium` (Table)
* **Logic:** Aggregates the Silver table data on a **monthly** basis by `traffic_medium`.
* **Key KPIs:**
    * `purchase_month` (Monthly grouping)
    * `total_purchased_value_usd` (Sum of `event_value_in_usd`)
    * `total_purchases_count` (Count of distinct purchase sessions)
    * `avg_value_per_purchase_usd` (Total Value / Total Purchases Count)

## Handover Notes for a Dataform Colleague
* **Schema Convention:** All output tables are written to separate datasets: `bronze`, `silver`, and `gold`. Ensure these datasets exist in the target BigQuery project.
* **Idempotency:** The tables use the `table` type, meaning they are rebuilt entirely on each run. For high volume, consider switching Silver and Gold to `incremental` with appropriate partition keys (`event_date` or `purchase_month`).
* **Missing Metric:** Due to source data simplification in the Silver layer, the "Total number of purchased items" is approximated or replaced by "Average Value per Purchase." If item count is required, the Silver query must be updated to unnest/extract item details from the source.
