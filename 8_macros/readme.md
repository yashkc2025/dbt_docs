## What is Jinja?

Jinja is a **templating engine** for Python. In dbt, it allows you to write code that looks like SQL but behaves like a programming language. When you run a dbt command, dbt "compiles" your code by looking for Jinja tags, executing the logic inside them, and producing a pure SQL file that your data warehouse can actually understand.

### The Three Pillars of Jinja Syntax

1. **Expressions `{{ ... }}`:** Used when you want to **output text**. If you want to print a variable or call a function, you wrap it in "double mustaches."
2. **Statements `{% ... %}`:** Used for **logic control**. This is where your `if` statements, `for` loops, and macro definitions live.
3. **Comments `{# ... #}`:** Used to prevent text from appearing in the compiled SQL.

## Basic Macros

A **macro** is essentially a reusable piece of code - think of it as a function. Instead of writing the same complex logic in ten different models, you write it once in a macro and call it wherever you need.

### Example: A simple currency converter

Suppose you frequently need to convert cents to dollars. Instead of typing `/ 100.0` everywhere, you can create a macro.

**File: `macros/cents_to_dollars.sql`**

```sql
{% macro cents_to_dollars(column_name, decimal_places=2) -%}
    round( cast(({{ column_name }} / 100.0) as numeric), {{ decimal_places }} )
{%- endmacro %}
```

**Using it in a Model:**

```sql
select
    id,
    {{ cents_to_dollars('order_total_cents') }} as order_total_usd
from {{ ref('stg_orders') }}
```

## Advanced Macro Development

Advanced development moves beyond simple math and enters the realm of **Introspection** - the ability for your code to "look" at your database and make decisions based on what it finds.

### 1. Dynamic Column Generation

Instead of hardcoding every column, you can use Jinja to loop through a list. This is perfect for "pivoting" data.

```sql
{% set payment_methods = ['credit_card', 'coupon', 'bank_transfer', 'gift_card'] %}

select
    order_id,
    {% for method in payment_methods %}
    sum(case when payment_method = '{{ method }}' then amount else 0 end) as {{ method }}_amount
    {%- if not loop.last %},{% endif %}
    {% endfor %}
from {{ ref('raw_payments') }}
group by 1
```

### 2. Using the `adapter` and `execute`

One of the most powerful features in dbt is the ability to query your database metadata *during* compilation.

* **`adapter.get_columns_in_relation()`**: Fetches the column names and types of a table currently in your warehouse.
* **The `execute` flag**: Jinja is evaluated twice (once to parse, once to run). The `execute` variable ensures your code only tries to talk to the database during the actual run phase, preventing errors.

**Advanced Macro Example: Checking for Column Existence**

```sql
{% macro drop_old_columns(relation_name) %}
    {% set columns = adapter.get_columns_in_relation(relation_name) %}
    
    {% if execute %}
        {# Logic to compare columns and generate DROP statements #}
        {{ log("Found " ~ columns|length ~ " columns in " ~ relation_name, info=True) }}
    {% endif %}
{% endmacro %}
```

### 3. Context & Packages

Advanced developers rarely start from scratch. They use the **dbt_utils** package. This package is essentially a massive library of advanced macros (like `pivot`, `unpivot`, or `surrogate_key`) that handle the complex Jinja for you.
