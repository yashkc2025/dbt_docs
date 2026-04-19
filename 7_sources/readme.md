**Sources** are your bridge between the messy, raw reality of your data warehouse and the clean, modeled world of your analytics. Think of them as a "contract" that defines where your raw data lives before dbt ever touches it.

Here is a deep dive into how to master them.

## 1. Defining Raw Tables (The `sources.yml`)

Instead of hardcoding table names like `raw_data.public.users` directly into your SQL, you define them in a `.yml` file (usually named `models/sources.yml` or similar).

This abstraction allows you to name the source something friendly (like `stripe`) and map it to the actual database/schema where the data lands.

```yaml
version: 2

sources:
  - name: jaffle_shop # This is the source name
    database: raw     # The physical database name
    schema: public    # The physical schema name
    tables:
      - name: orders  # The physical table name
      - name: customers
```

### Why do this?

* **Lineage:** dbt can now draw a line from your raw data to your final models.
* **Decoupling:** if your data moves from the `public` schema to `raw_ingest`, you change it in **one** YAML file instead of 50 SQL models.

## 2. The `source()` Function

Once defined in YAML, you never use a raw string for that table again. You use the `source()` function in your models.

**The Syntax:**
`{{ source('source_name', 'table_name') }}`

**The Model (`stg_orders.sql`):**

```sql
select
    id as order_id,
    status as order_status
from {{ source('jaffle_shop', 'orders') }}
```

When you run `dbt compile`, dbt replaces that Jinja tag with the full relation name (e.g., `raw.public.orders`). This is how it builds your **DAG (Directed Acyclic Graph)**.

## 3. Source Freshness

This is one of dbt’s most powerful features. It allows you to answer the question: *"Is my data stale?"* To use it, you tell dbt which column represents the "last updated" timestamp for a table. dbt will then calculate the difference between the current time and the maximum value in that column.

### Defining Freshness in YAML

```yaml
sources:
  - name: jaffle_shop
    database: raw
    tables:
      - name: orders
        freshness:
          warn_after: {count: 6, period: hour}
          error_after: {count: 24, period: hour}
        loaded_at_field: _etl_loaded_at # The timestamp column in your raw table
```

### How to Run It

You don't run freshness with `dbt run`. You use:
`dbt source freshness`

dbt will scan the `loaded_at_field` for every table defined and output a report. If the data hasn't been updated in 25 hours, the command will fail (or warn), preventing you from building models on top of "old" data and potentially alerting your data engineering team that an ingestion pipeline is broken.

## Pro-Tips for Deep Usage

### 1. Documentation & Testing

Sources can have descriptions and tests just like models. Testing a source is actually **better** because it catches data quality issues *before* they enter your transformation pipeline.

```yaml
tables:
  - name: orders
    description: "Raw order data from the website"
    tests:
      - unique:
          column_name: id
      - not_null:
          column_name: id
```

### 2. The `{{ source().identifier }}`

If you have a table that is named `orders_v2_final_final` in your database, but you just want to call it `orders` in dbt, you can use the `identifier` property:

```yaml
tables:
  - name: orders
    identifier: orders_v2_final_final
```

Now, `{{ source('shop', 'orders') }}` points to the ugly name, but your code stays clean.

### 3. Environment Overrides

You can use Jinja within your `sources.yml` to point to different databases based on whether you are in `dev` or `prod`.

```yaml
sources:
  - name: jaffle_shop
    database: "{{ 'raw_prod' if target.name == 'prod' else 'raw_dev' }}"
```
