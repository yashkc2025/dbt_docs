## 1. What is a Model?

At its simplest, a dbt model is a **single `.sql` file** containing a `SELECT` statement.

* **The File:** `models/my_first_model.sql`
* **The Transformation:** You write the logic to clean, join, or aggregate data.
* **The Magic:** When you run `dbt run`, dbt wraps that `SELECT` statement in DDL (Data Definition Language) like `CREATE VIEW` or `CREATE TABLE` and executes it against BigQuery.

**The result:** Your SQL file is transformed into a physical object (a table or view) in your BigQuery dataset.

## 2. Materialization Types

Materialization refers to the strategy dbt uses to persist the model in your warehouse. You define this in the `dbt_project.yml` or at the top of the SQL file using a config block: `{{ config(materialized='...') }}`.

### **View**

The default materialization. It doesn't store data on disk; it stores the query itself.

* **How it works:** Every time you query the view, BigQuery runs the underlying model logic from scratch.
* **When to use:** For small transformations, models that are rarely queried, or when you need real-time data from the source.

### **Table**

Creates a physical table in BigQuery.

* **How it works:** dbt executes a `CREATE TABLE AS SELECT` (CTAS) statement. It drops the old table and builds a brand-new one every time the model runs.
* **When to use:** For models that are queried frequently by BI tools (like Looker or Tableau) or for complex logic that is too slow to run as a view.

### **Incremental** (The High-Performance Choice)

Incremental models allow dbt to insert or update only the *new* records since the last time the model ran.

* **How it works:** You define a "lookback" window (e.g., "only process data from the last 3 days"). dbt then appends that data to the existing table instead of rebuilding it from scratch.
* **When to use:** Essential for large event-stream data (logs, clicks, transactions) where rebuilding the entire history every day is too expensive or slow.

## 3. Cost & Performance Tradeoffs in BigQuery

BigQuery charges based on **Analysis (Bytes Processed)** and **Storage**. Choosing the right materialization directly impacts your bill.

| Strategy | Performance (Read Speed) | BigQuery Cost (Compute) | Storage Cost |
| :--- | :--- | :--- | :--- |
| **View** | Slower (runs logic every time) | **High** (charged per query) | **Zero** |
| **Table** | Fast (pre-computed) | **Medium** (charged per full rebuild) | **Standard** |
| **Incremental**| Fast (pre-computed) | **Low** (only processes new data) | **Standard** |

### **Strategic Advice for BigQuery Users:**

1. **Start with Views:** Use views for your "Staging" layer (the first layer of cleaning). They are free to build and keep your warehouse clean.
2. **Move to Tables for BI:** Once you get to your "Marts" layer (the tables your business users see), use tables. Users hate waiting for dashboards to load, and it prevents BigQuery from re-calculating the same logic 100 times a day.
3. **Use Incremental for "Big Data":** If a table takes more than a few minutes to build or processes terabytes of data, it **must** be incremental.
    * *Pro-Tip:* Always pair incremental models with **Partitioning** (e.g., `partition_by={'field': 'created_at'}`) in BigQuery to further reduce costs.
