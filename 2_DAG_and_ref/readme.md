## 1. The `ref()` Function: More Than a Table Name

In traditional SQL, you hardcode table names (e.g., `FROM raw_data.orders`). In dbt, you use `{{ ref('orders') }}`. This is **critical** for two reasons:

* **Environment Agnostic:** dbt automatically resolves the reference to the correct schema based on your environment (e.g., `dev_jsmith.orders` vs. `prod.orders`).
* **The Dependency Link:** Every time you use `ref()`, you aren't just fetching data; you are explicitly telling dbt: *"This model cannot run until the referenced model is finished."*

> ### 💡 Why is this "Critical"?
>
> Without `ref()`, dbt has no way of knowing how your models relate. If you hardcode table names, dbt will try to run all models simultaneously. If Model B depends on Model A and they run at the same time, Model B will likely fail or use stale data because Model A hasn't finished updating yet.

## 2. Dependency Graph (DAG)

When dbt parses your project, it looks at every `ref()` and `source()` call. It then constructs a **Directed Acyclic Graph (DAG)**.

* **Directed:** Data flows in one direction (upstream to downstream).
* **Acyclic:** There are no loops. (e.g., Model A cannot depend on Model B if Model B depends on Model A). If you accidentally create a loop, dbt will throw a **Circular Dependency** error.
* **Graph:** A visual map of how every node (model, seed, test, or snapshot) connects.

## 3. Execution Order (Topological Sorting)

Once the DAG is built, dbt uses a mathematical process called **Topological Sorting** to determine the execution order.

1. **Level 0 (Sources):** dbt identifies models with no upstream dependencies (usually your staging models pulling from raw sources).
2. **Level 1:** dbt runs models that only depend on Level 0 models.
3. **Parallelism:** If two models have no dependencies on each other, dbt will run them **simultaneously** (depending on your `threads` configuration) to save time.

**The "Skip" Logic:** If you use `dbt build`, and an upstream model fails, dbt will automatically **skip** all downstream models in the DAG. This prevents "garbage in, garbage out" scenarios where you calculate metrics based on broken data.

## 4. Demo

The best way to visualise your DAG is through the **dbt Docs UI**. Here is how you navigate it:

### Step 1: Generate and Serve

Run these commands in your terminal:

```bash
dbt docs generate
dbt docs serve
```

### Step 2: The Lineage View

1. In the browser window that opens, look at the bottom right corner. Click the **green expansion button**.
2. **The Mini-Map:** You’ll see a focused graph showing the immediate "Parents" (left) and "Children" (right) of the model you are currently viewing.
3. **The Full DAG:** Click "View Full Screen" to see the entire spiderweb of your project.

### Step 3: Interactive Selection

* **Focusing:** Click on any node to highlight its path.
* **Filtering:** Use the search bar at the bottom to filter the graph.
  * `+model_name`: Shows the model and all its **parents** (upstream).
  * `model_name+`: Shows the model and all its **children** (downstream).
  * `+model_name+`: Shows the entire lineage "sandwich" for that specific model.

### Summary Table

| Feature | Role | Analogy |
| :-- | :-- | :-- |
| **`ref()`** | The Connector | The "glue" between two Lego bricks. |
| **DAG** | The Blueprint | The instruction manual showing how the bricks fit together. |
| **Execution Order**| The Builder | The actual process of putting brick A down before brick B. |
