## 🛠️ The Philosophy: "Test-Driven Data Engineering"

For an engineer, a test is a **contract**. When you define a test in a `schema.yml`, you are asserting that the data must meet specific criteria before it hits your BI tool or ML model.

dbt tests are essentially **SQL queries** that return the "failing" rows. If the query returns **zero rows**, the test passes. If it returns **one or more rows**, the test fails.

## 🏗️ The Schema.yml: The Control Center

The `schema.yml` file is where you define your models and their associated constraints. It lives in your `/models` folder and serves as both documentation and configuration.

```yaml
version: 2

models:
  - name: orders
    columns:
      - name: order_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - relationships:
              to: ref('customers')
              field: customer_id
```

## Few Examples

### 1. `unique`

**What it does:** Ensures that every value in a column appears only once.

* **Engineer’s Perspective:** This is your primary key validator. In modern cloud warehouses (Snowflake, BigQuery), primary keys are often "not enforced," meaning the database won't stop you from inserting duplicates.
* **The SQL Logic:** dbt generates a query that groups by the column and filters for counts greater than one.
  * *If the result set > 0, your "primary key" is lying to you.*

### 2. `not_null`

**What it does:** Ensures that there are no empty (NULL) values in a column.

* **Engineer’s Perspective:** This prevents downstream "divide by zero" errors or broken joins. It’s the simplest yet most effective way to catch ingestion issues where source data suddenly goes missing.
* **The SQL Logic:** `select * from model where column_name is null`.

### 3. `relationships` (Referential Integrity)

**What it does:** Ensures that every value in Column A exists in Column B of another table.

* **Engineer’s Perspective:** This is how you catch "orphaned records." If an order exists for `customer_id: 99`, but that customer doesn't exist in the `customers` table, your analytics will be skewed.
* **The Parameters:** * `to`: The model you are referencing (the "parent").
  * `field`: The column name in that parent model.
* **The SQL Logic:** It performs a `left join` between the child and parent; if any child record has no match in the parent, the test fails.

## Few Concepts

### Test Severity: `warn` vs. `error`

Not all failures are fatal. You can configure dbt to allow a test to fail with a warning while still completing the build, or to hard-stop the pipeline.

```yaml
- name: total_amount
  tests:
    - not_null:
        config:
          severity: warn
          warn_after: 10 # Only warn if more than 10 rows fail
```

### Store Failures

When a test fails, the first thing an engineer asks is: *"Which rows failed?"* By adding `+store_failures: true` to your `dbt_project.yml`, dbt will actually save the "bad rows" into a separate schema in your database. This makes debugging incredibly fast - you just query the failure table to see the exact data that broke the contract.

### Generic vs. Singular

* **Generic (What we discussed):** Defined in YAML. Reusable across many columns (e.g., `unique`).
* **Singular:** Complex logic written as a standalone `.sql` file in the `/tests` folder. If you need to test that `order_date` is always after `signup_date`, you’d write a singular test.
