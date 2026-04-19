## 1. The Staging Layer (`stg_`)

**The Foundation.** Think of this as the "cleaning room." You’ve just pulled raw data from your source (Shopify, Stripe, or a Postgres DB), and it’s usually a bit of a mess.

* **Primary Goal:** Create a clean, consistent interface for your raw data.
* **The Rules:**
  * **1:1 Relationship:** Generally, you have one staging model for every one raw table.
  * **Renaming:** Standardize field names (e.g., changing `user_id_pk` to just `user_id`).
  * **Type Casting:** Ensure timestamps are actually `TIMESTAMP` and "1.0" is cast to an `INT`.
  * **Basic Cleaning:** Handling nulls or converting "T/F" strings into actual Booleans.
* **The Golden Rule:** **No Joins.** Staging is about the individual source. Do not start mixing data here. If the source breaks, you only want to have to fix it in one staging file.

## 2. The Intermediate Layer (`int_`)

**The Engine Room.** This is where the actual transformation logic lives. If Staging is about *formatting*, Intermediate is about *meaning*.

* **Primary Goal:** To encapsulate complex business logic and join sources together before they hit the final "product."
* **What happens here:**
  * **Complex Joins:** Bringing together `stg_orders` and `stg_customers` to see who bought what.
  * **Business Logic:** Defining what a "Churned User" actually means using `CASE WHEN` statements.
  * **Modularization:** If you have a specific way of calculating "Net Revenue," you do it here once. Then, every Mart that needs revenue pulls from this model.
* **The "Lego" Concept:** These are your building blocks. They aren't usually the "final" tables, but they are the reusable pieces that make building the final tables easy.

## 3. The Mart Layer (`fct_` & `dim_`)

**The Storefront.** This is the "Product" your business actually sees. Everything in this layer is optimized for the end-user analysts, executives, or automated systems.

* **Primary Goal:** Provide a fast, easy-to-understand dataset for reporting.
* **The Structure (Star Schema):**
  * **Fact Tables (`fct_`):** Quantitative data. These represent "Events." (e.g., `fct_orders`, `fct_website_visits`).
  * **Dimension Tables (`dim_`):** Qualitative data. These represent "Entities." (e.g., `dim_customers`, `dim_products`).
* **Characteristics:** These tables are often "wide" (denormalized). You want to make it so a BI tool can grab one or two tables and have everything it needs without writing 50 lines of SQL.

## 4. Connecting to the Pipeline

**Where the value is realized.** Once the data hits the Mart layer, it’s ready to leave the warehouse and go to work. This happens in two main ways:

### BI & Visualization

Tools like **Tableau, Looker, or PowerBI** connect directly to your Marts. Because you did the heavy lifting in the Intermediate layer, your dashboards load faster and - most importantly - everyone in the company sees the same "Single Source of Truth."

### Reverse ETL

This is the "Deep" part of the pipeline. Tools like **Hightouch or Census** take data from your `dim_customers` mart and push it back into the tools your team uses daily.

* **Example:** Your Mart identifies a "High Value Customer." Reverse ETL pushes that "High Value" tag directly into **Salesforce** or **Zendesk** so your support team knows to prioritize them instantly.
