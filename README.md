# GA4 E-commerce Analytics Pipeline Project

## Overview
This Dataform project transforms raw Google Analytics 4 (GA4) e-commerce sample data into structured **Bronze**, **Silver**, and **Gold** layers for business reporting. It is designed to run regularly to provide updated monthly reports on traffic source performance.

---

## Data Lineage and Dependencies 📈

Below is a screenshot of the Dataform execution graph, illustrating the direct dependencies between the project's data objects.

<img width="1575" height="255" alt="image" src="https://github.com/user-attachments/assets/44b63d95-3551-451c-af9e-a06f9656f973" />

---

### Dependency Explanation

The pipeline follows a standard Medallion Architecture:

| Layer | Object Name | Dependency | Explanation |
| :--- | :--- | :--- | :--- |
| **Bronze** | `events_source` | **External Source** | This is a **declaration** referencing the public BigQuery table `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`. It has **no upstream dependencies** within the project. |
| **Silver** | `purchase_traffic_source_medium` | $\rightarrow$ `events_source` | This table is built **directly from the Bronze layer**. It filters, cleans, and denormalizes the raw purchase events. |
| **Gold** | `top_traffic_source_medium` | $\rightarrow$ `purchase_traffic_source_medium` | This final report table is built **directly from the Silver layer**. It aggregates the clean, denormalized purchase data monthly to calculate key business metrics. |


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

## Handover Notes
* **Schema Convention:** All output tables are written to separate datasets: `bronze`, `silver`, and `gold`. Ensure these datasets exist in the target BigQuery project.
* **Idempotency:** The tables use the `table` type, meaning they are rebuilt entirely on each run. For high volume, consider switching Silver and Gold to `incremental` with appropriate partition keys (`event_date` or `purchase_month`).
* **Missing Metric:** Due to source data simplification in the Silver layer, the "Total number of purchased items" is approximated or replaced by "Average Value per Purchase." If item count is required, the Silver query must be updated to unnest/extract item details from the source.
