In standard SQL (like PostgreSQL), you have **Databases** and **Schemas**. In BigQuery, the terminology shifts slightly, and it is crucial for your dbt configuration.

* **Google Cloud Project:** This is the highest level (the "Account").
* **Dataset:** This is the **Schema**. When you define your `target` in dbt’s `profiles.yml`, the `dataset` you specify is where dbt will physically build your tables and views.
* **Tables/Views:** These are the individual objects dbt creates.

> **Key Takeaway:** When you run `dbt run`, dbt looks at your models and says, "Okay BigQuery, go into Project X and create a table inside Dataset Y."

## 2. How dbt Runs Queries in BigQuery

dbt does not actually "process" your data. Your local machine (or your CI/CD server) isn't doing any math. Instead, dbt performs a **three-step orchestration**:

### A. Compilation

dbt takes your code - which is a mix of SQL and **Jinja** (the `{{ config(...) }}` and `{{ ref(...) }}` tags) - and compiles it into "Pure SQL."

* It replaces `{{ ref('my_model') }}` with the actual physical address: `project_id.dataset_id.my_model`.

### B. The Adapter Handshake

dbt uses a specific **BigQuery Adapter**. It opens a connection to the BigQuery API and sends the compiled SQL string over the wire.

### C. Execution via CTAS

BigQuery receives the SQL. However, dbt doesn't just send a `SELECT` statement. To make the data persist, dbt wraps your query in a **CTAS** (Create Table As Select) statement.

**What you wrote:**

```sql
SELECT * FROM {{ ref('raw_sales') }} WHERE status = 'shipped'
```

**What BigQuery actually executes:**

```sql
CREATE OR REPLACE TABLE `my-project`.`my_dataset`.`shipped_sales` AS (
  SELECT * FROM `my-project`.`my_dataset`.`raw_sales` WHERE status = 'shipped'
);
```

## 3. Table Creation & Materializations

dbt gives you control over *how* the data is stored in BigQuery via **materializations**. This is defined in the `config` block at the top of your `.sql` file.

| Materialization | What it does in BigQuery | Use Case |
| :-- | :-- | :-- |
| **View** | Creates a virtual table (a saved query). No data is stored. | Small transformations or when you need real-time data. |
| **Table** | Drops the old table and creates a fresh one with all data. | Medium-sized datasets where a full refresh is fast. |
| **Incremental** | Only appends/updates *new* rows since the last run. | Huge datasets (billions of rows) to save on BigQuery costs. |
| **Ephemeral** | Does not create anything in BQ; it becomes a CTE in the next model. | Internal logic you don't want to clutter your schema with. |

## 4. Zero Downtime Swap

One of the best features of dbt + BigQuery is how it handles table refreshes. If you are replacing a table that people are currently using for a dashboard, you don't want them to see an "Object Not Found" error mid-run.

1. dbt creates a **temporary table** (e.g., `my_table__dbt_tmp`).
2. It populates that table with your new data.
3. Once successful, it performs an **atomic swap**: it drops the old table and renames the temp table to the final name in a single transaction.
4. If the query fails halfway through, the old table remains untouched. **Your production data never breaks.**

## 5. BigQuery-Specific Optimizations

When dbt talks to BigQuery, it can also pass along specific instructions to make BigQuery faster and cheaper. You can define these in your dbt model:

* **Partitioning:** Tells BigQuery to segment the table by a column (like `created_at`). This ensures that when you query "last week's data," BigQuery only scans that specific folder of data, saving you money.
* **Clustering:** Sorts the data within those partitions (like by `user_id`), making filter-heavy queries lightning fast.

```sql
{{ config(
    materialized='table',
    partition_by={
      "field": "created_at",
      "data_type": "timestamp",
      "granularity": "day"
    },
    cluster_by="user_id"
) }}

SELECT ...
```
