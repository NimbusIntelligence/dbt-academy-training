# Instructor Guide — dbt Academy

This document is for the instructor only. It contains timing notes, answers to all exercise questions, model solutions, and common student errors for each day.

---

## Daily Schedule

**9:30** Start  
**13:00** Lunch break  
**14:00** Resume  
**17:30** End

Total instructional time: **7 hours per day** (3.5h morning + 3.5h afternoon).

As a rough split: use the morning block for Guided exercises (demos, instructor types, students follow) and the afternoon block for Practice exercises (students work independently). This way any overrun in demos only affects the morning, and students always have the full afternoon for their own work.

---

## General Facilitation Notes

**Pacing rule:** The Guided parts are demos — you type, students watch and follow. The Practice parts are independent work — you circulate, don't just sit at the front. Set a visible timer.

**When students are stuck:** Ask "what does the error message say?" before explaining anything. Most dbt errors are self-explanatory once students learn to read them.

**Fast finishers:** Every exercise has implied extensions (add more tests, add more columns, explore edge cases). Tell fast students to extend before peeking at solutions.

**Slow groups:** Cut Practice parts if needed — understanding the Guided demo is more important than completing the Practice.

---

## Day 1 — Setup & First Models

### Schedule

| Time | Activity |
|---|---|
| 9:30–12:30 | dbt Theory presentation (3 h) |
| 12:30–12:45 | Ex 1 — Snowflake setup (15 min) |
| 12:45–13:00 | Ex 2 — GitHub setup (15 min) |
| 13:00–14:00 | Lunch |
| 14:00–14:45 | Ex 3 — dbt Cloud setup (45 min) |
| 14:45–16:15 | Ex 4 — First models (90 min) |
| 16:15–17:15 | Ex 5 — Sources (60 min) |
| 17:15–17:30 | Day wrap-up |

---

### Exercise 1 — Snowflake Setup

**Common errors:**
- Students forget to set the role context → `USE ROLE ACCOUNTADMIN;` must be the first statement.
- Students create schemas but forget to grant privileges to `TRANSFORMER` role. Run the entire setup script as a single block.
- `SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS` count returns NULL → navigate to Admin → Databases in Snowflake UI and enable the `SNOWFLAKE_SAMPLE_DATA` share.

**Verification answer:** `SELECT COUNT(*) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS` → **1,500,000**

---

### Exercise 2 — GitHub Setup

No common errors — purely procedural. Make sure everyone ends up with a private `dbt-academy` repo at `github.com/<username>/dbt-academy` before moving on.

---

### Exercise 3 — dbt Cloud Setup

**Common errors:**
- Wrong Snowflake account region → the account identifier must match exactly (e.g., `abc12345.eu-west-1`).
- Connection test fails with `IP not whitelisted` → disable the network policy on `DBT_USER` for the course.
- `SYSADMIN` role used instead of `TRANSFORMER`.
- Schema field auto-populated with a long generated name → replace it with `dbt_dev_<their_name>`.

**Note:** dbt Cloud generates the project structure when the student clicks **Initialize dbt project**. Students do **not** need to run `dbt init`.

---

### Exercise 4 — First Models

**Part A — Example models questions:**

- *What does the config block do, and what happens to a model without one?* → The `{{ config() }}` block overrides project-level settings for that specific model. A model without one inherits whatever is set in `dbt_project.yml`. By default dbt Cloud sets `+materialized: table`, so both example models end up as tables.

- *What does `{{ ref() }}` do that a plain SQL `FROM` doesn't?* → Two things: (1) registers the dependency in the DAG so dbt knows execution order, and (2) resolves the correct schema for the current environment — no hardcoded schema names.

- *What would you add to make the build continue when a test fails?* → Add `severity: warn`:
  ```yaml
  - not_null:
      severity: warn
  ```
  Default is `severity: error`, which skips downstream models. Note: the model that owns the failing test was already materialized — `run` happens before `test`. It's the downstream model that gets skipped.

**Part E — Did lineitem take longer than orders?**

Barely or not at all. Both are views — dbt only creates the view definition, it doesn't scan rows. The 6M rows are only read when someone queries the view. If materialized as tables, lineitem would be significantly slower.

**Part F — View vs table questions:**

- *Difference between VIEW and TABLE:* A view stores only the SQL definition and re-executes it on every query. A table stores the result set physically.

- *When view for bronze, when table?* No universal answer. **View** makes sense when the source is already in the same warehouse, queries are few, and you want bronze to always reflect the source without re-running dbt. **Table** makes sense when the source is slow or external, many downstream models read from bronze simultaneously (each would re-scan), or the source is large. In production most teams use tables — pay the scan cost once, let all downstream models read a local copy.

- *What to change to switch bronze to `view`?* Change `+materialized: table` to `+materialized: view` under `bronze:` in `dbt_project.yml`.

**Part E — bronze_tpch_lineitem solution:**
```sql
SELECT
    l_orderkey                                          AS order_id,
    l_linenumber                                        AS line_number,
    l_partkey                                           AS part_id,
    l_suppkey                                           AS supplier_id,
    l_quantity                                          AS quantity,
    l_extendedprice                                     AS extended_price,
    l_discount                                          AS discount_rate,
    l_tax                                               AS tax_rate,
    ROUND(l_extendedprice * (1 - l_discount), 2)        AS net_price,
    l_returnflag                                        AS return_flag,
    l_shipdate                                          AS ship_date,
    l_shipmode                                          AS ship_mode
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.LINEITEM
```

**Note:** Students use the hardcoded path here intentionally — they will replace it with `{{ source('tpch', 'lineitem') }}` in Exercise 5.

**Common error:** Students use lowercase table names (`lineitem`). Snowflake is case-insensitive by default so it works, but teach uppercase to be explicit.

---

### Exercise 5 — Sources

**Key teaching point:** Without `{{ source() }}`, dbt doesn't know the model depends on that table — the DAG is broken and the lineage graph is incomplete. Show the lineage graph before and after applying `{{ source() }}` so students see the difference visually.

**Part B question — what if the schema changed from TPCH_SF1 to TPCH_SF10?**
You'd change one line in `tpch_sources.yml` (`schema: TPCH_SF10`). All models using `{{ source('tpch', ...) }}` update automatically — no SQL file changes. Without sources you'd search-and-replace every model.

**Part D — bronze_tpch_nations.sql solution:**
```sql
SELECT
    n_nationkey     AS nation_id,
    n_name          AS nation_name,
    n_regionkey     AS region_id
FROM {{ source('tpch', 'nation') }}
```

**Part D — bronze_tpch_suppliers.sql solution:**
```sql
SELECT
    s_suppkey       AS supplier_id,
    s_name          AS supplier_name,
    s_nationkey     AS nation_id,
    s_acctbal       AS account_balance,
    s_phone         AS phone
FROM {{ source('tpch', 'supplier') }}
```

**Part E — freshness:**
`dbt source freshness` checks only `orders` because it's the only table with `loaded_at_field` configured. TPC-H is static data from 1998, so the warn threshold (1 day) and error threshold (30 days) will both trigger immediately. This is expected — it demonstrates how freshness alerting works in production.

**Common error:** `Expected a timestamp value when querying field 'last_modified'` — this happens when dbt falls back to Snowflake's warehouse metadata instead of running a SQL query. Two causes: (1) `loaded_at_field` not set (dbt uses metadata by default on Snowflake), or (2) `o_orderdate` is a DATE column, not TIMESTAMP. Fix: ensure `loaded_at_field: "o_orderdate::timestamp"` (with the cast and quotes) is inside the `config:` block. The cast forces dbt to use a SELECT query instead of metadata, bypassing the `last_modified` issue with shared sample databases.

**Freshness question:** Use `error_after: {count: 1, period: hour}` when your pipeline promises near-real-time data — for example, a payment events table that feeds a fraud detection dashboard. If the table stops updating, you want the pipeline to fail loudly rather than silently serve stale data.

---

## Day 2 — Refs, Materializations & Incremental

### Schedule

| Time | Activity |
|---|---|
| 9:30–12:00 | Ex 1 — Refs & DAG (2.5 h) |
| 12:00–13:00 | Ex 2 — Materializations, part 1 (1 h) |
| 13:00–14:00 | Lunch |
| 14:00–15:00 | Ex 2 — Materializations, part 2 (1 h) |
| 15:00–17:30 | Ex 3 — Incremental models (2.5 h) |

---

### Exercise 1 — Refs & the DAG

**silver_orders_enriched** is the most complex model in the course. The key insight: aggregate `lineitem` first (one row per `order_id`) before joining. Students who join without aggregating first will get row multiplication.

**Part C — Execution order questions:**

- *What models ran with `+silver_orders_enriched`?* → The four bronze parents (`bronze_tpch_orders`, `bronze_tpch_customers`, `bronze_tpch_lineitem`, `bronze_tpch_nations`) plus `silver_orders_enriched`.

- *Why does dbt know the execution order?* → dbt parses every `{{ ref() }}` call and builds a DAG. The topological sort of that graph gives the execution order.

- *What happens if you run silver before bronze exists?* → `Compilation Error: object does not exist`. The view SQL references bronze views that don't exist in Snowflake yet.

**Part D — gold_customers solution:**
```sql
WITH customers AS (
    SELECT * FROM {{ ref('bronze_tpch_customers') }}
),
nations AS (
    SELECT * FROM {{ ref('bronze_tpch_nations') }}
),
orders AS (
    SELECT
        customer_id,
        COUNT(*)            AS total_orders,
        SUM(net_revenue)    AS total_revenue,
        MIN(order_date)     AS first_order_date,
        MAX(order_date)     AS last_order_date
    FROM {{ ref('silver_orders_enriched') }}
    GROUP BY 1
)

SELECT
    c.customer_id,
    c.customer_name,
    c.market_segment,
    n.nation_name,
    COALESCE(o.total_orders, 0)     AS total_orders,
    COALESCE(o.total_revenue, 0)    AS total_revenue,
    CASE
        WHEN o.total_orders > 0
        THEN o.total_revenue / o.total_orders
        ELSE 0
    END                             AS avg_order_value,
    o.first_order_date,
    o.last_order_date,
    CASE
        WHEN COALESCE(o.total_revenue, 0) > 500000 THEN 'Platinum'
        WHEN COALESCE(o.total_revenue, 0) > 100000 THEN 'Gold'
        WHEN COALESCE(o.total_revenue, 0) > 0      THEN 'Silver'
        ELSE 'No Orders'
    END                             AS customer_tier
FROM customers c
LEFT JOIN nations n ON c.nation_id   = n.nation_id
LEFT JOIN orders  o ON c.customer_id = o.customer_id
```

**Part E — gold_orders solution:**
```sql
{{ config(materialized='table') }}

SELECT
    *,
    gross_revenue - net_revenue                                     AS discount_amount,
    ROUND(
        (gross_revenue - net_revenue) / NULLIF(gross_revenue, 0),
        4
    )                                                               AS effective_discount_rate,
    net_revenue / NULLIF(total_items, 0)                            AS revenue_per_item,
    CASE
        WHEN total_items >= 5 THEN 'Large'
        WHEN total_items >= 3 THEN 'Medium'
        ELSE 'Small'
    END                                                             AS order_size_band
FROM {{ ref('silver_orders_enriched') }}
```

*Why materialize gold as table?* If gold were a view over silver (also a view) over four bronze views, a dashboard query would trigger a full re-scan of 6M lineitem rows on every load. Materializing gold as a table pays that cost once at `dbt run` time — dashboards then query a pre-built result set.

**Part F — DAG question:**

- *If TPC-H renamed `c_nationkey`, which models break?* → `bronze_tpch_customers` first, then everything downstream: `silver_orders_enriched`, `gold_customers`, `gold_orders`, `gold_nations_summary`.

- *How to identify without running everything?* → Open `bronze_tpch_customers` in the IDE lineage tab and expand downstream. Or run `dbt compile --select bronze_tpch_customers+` — any model that fails to compile is broken.

---

### Exercise 2 — Materializations

**view vs table vs incremental:**
- `view`: no storage, recomputed on every query, always fresh. Good for lightweight bronze.
- `table`: full rebuild on each `dbt run`, fast to query. Good for gold aggregations.
- `incremental`: only processes new/changed rows. Essential for large fact tables like LINEITEM.

**Part B — gold_market_summary solution:**
```sql
WITH line_totals AS (
    SELECT * FROM {{ ref('silver_lineitem_totals') }}
),
orders AS (
    SELECT * FROM {{ ref('bronze_tpch_orders') }}
),
customers AS (
    SELECT * FROM {{ ref('bronze_tpch_customers') }}
)

SELECT
    c.market_segment,
    COUNT(DISTINCT o.order_id)                AS total_orders,
    SUM(l.net_revenue)                        AS total_revenue,
    SUM(l.net_revenue) /
        NULLIF(COUNT(DISTINCT o.order_id), 0) AS avg_order_value
FROM orders o
JOIN customers c   ON o.customer_id = c.customer_id
JOIN line_totals l ON o.order_id    = l.order_id
GROUP BY 1
```

Expected: 5 rows (AUTOMOBILE, BUILDING, FURNITURE, HOUSEHOLD, MACHINERY).

**Part B — ephemeral questions:**
1. `silver_lineitem_totals` does not appear in Snowflake because ephemeral models are inlined as CTEs into the compiled SQL of whatever model references them — dbt never issues a `CREATE` statement for them.
2. In the compiled output of `silver_orders_enriched`, the ephemeral model's SQL appears inlined as a CTE before the main query.
3. **Ephemeral** — no object in the warehouse, no storage cost, no query cost to read it. Downside: can't query it directly, can't test it, gets inlined everywhere it's used (compiled SQL gets large). Use for intermediate transformations shared by a few models. **View** — queryable, testable, can be referenced from BI tools. Use when you need visibility or want to run `dbt test` on it.

**Part D — gold_nations_summary solution:**
```sql
{{ config(materialized='view') }}

SELECT
    n.nation_name,
    COUNT(DISTINCT o.order_id)                AS total_orders,
    SUM(o.net_revenue)                        AS total_revenue,
    SUM(o.net_revenue) /
        NULLIF(COUNT(DISTINCT o.order_id), 0) AS avg_order_value
FROM {{ ref('silver_orders_enriched') }} o
JOIN {{ ref('bronze_tpch_nations') }} n ON o.nation_name = n.nation_name
GROUP BY 1
```

**Part D — config question:** `{{ config() }}` inside the file is self-contained — anyone reading the model sees immediately how it's materialized without opening `dbt_project.yml`. Preferred in teams because the configuration travels with the model. `dbt_project.yml` overrides are better for bulk changes (e.g. "make all models in this folder incremental") but create implicit behaviour that's easy to miss.

**gold_nations_summary solution (old, kept for reference):**
```sql
{{ config(materialized='view') }}

SELECT
    n.nation_name,
    COUNT(DISTINCT o.customer_id)   AS customer_count,
    COUNT(DISTINCT o.order_id)      AS order_count,
    SUM(o.net_revenue)              AS total_revenue
FROM {{ ref('silver_orders_enriched') }} o
JOIN {{ ref('bronze_tpch_nations') }} n ON o.nation_name = n.nation_name
GROUP BY 1
```

---

### Exercise 3 — Incremental Models

**Common confusion:** `is_incremental()` is `false` on first run (table doesn't exist) and `true` on subsequent runs. Show both runs explicitly.

**Part B answers:**
1. 0 rows because TPC-H is static — `MAX(order_date)` in the table already equals the highest date in the source, so the `WHERE` filter returns no rows.
2. `{{ this }}` resolves to the fully qualified name of the current model's table in the warehouse: `ANALYTICS.GOLD.GOLD_ORDERS`. It's only available inside incremental models and lets the filter reference the existing table.
3. `{% if is_incremental() %}` evaluates to `false` on the first run because the table doesn't exist yet. dbt skips the block entirely and loads all rows. From the second run onward it evaluates to `true` and the filter is applied.

**Part C answers:**
1. The `WHERE` clause comes from the `{% if is_incremental() %}` block in the model. dbt includes it only when the target table already exists — on the first run the block is skipped and all rows are loaded.
2. Without `unique_key`, dbt uses `INSERT` instead of `MERGE`. If the same `order_id` appears in two runs, it gets inserted twice — the table accumulates duplicates with every run. `unique_key` switches to MERGE: update if the key exists, insert if it doesn't.

**Part D answers:**
1. Only rows with `order_date` newer than the current max get processed. Existing rows keep the old band values. To apply the new logic to all rows, run `--full-refresh`.
2. No — backdated rows have `order_date` below `MAX(order_date)` so the filter excludes them. This is a known limitation of date-based incremental filters. Fix: run `--full-refresh`, or use a lookback window (`order_date > DATEADD(day, -3, MAX(order_date))`).

**`unique_key`:** Without it, each run appends rows without deduplication. With `unique_key='order_id'`, dbt uses MERGE in Snowflake — update if exists, insert if not.


---

## Day 3 — Tests, Documentation & Seeds

### Schedule

| Time | Activity |
|---|---|
| 9:30–11:30 | Ex 1 — Generic tests (2 h) |
| 11:30–13:00 | Ex 2 — Singular tests (1.5 h) |
| 13:00–14:00 | Lunch |
| 14:00–15:00 | Ex 3 — Documentation (1 h) |
| 15:00–16:00 | Ex 4 — Seeds (1 h) |
| 16:00–17:30 | Ex 5 — dbt build & selection (1.5 h) |

---

### Exercise 1 — Generic Tests

Structure: A (guided) → B (practice) → C (guided) → D (practice) → E (guided) → F (guided) → G (guided)

---

**Part B — `tpch_sources.yml` solution (extends the existing file from Day 1):**
```yaml
version: 2

sources:
  - name: tpch
    database: SNOWFLAKE_SAMPLE_DATA
    schema: TPCH_SF1
    description: "TPC-H benchmark dataset — read-only, built into every Snowflake account"

    tables:
      - name: orders
        description: "1.5M order headers"
        config:
          loaded_at_field: "o_orderdate::timestamp"
          freshness:
            warn_after: {count: 1, period: day}
            error_after: {count: 30, period: day}
        columns:
          - name: o_orderkey
            description: "Primary key"
            data_tests:
              - unique
              - not_null
          - name: o_custkey
            description: "Foreign key to customer"
            data_tests:
              - not_null
          - name: o_orderstatus
            data_tests:
              - accepted_values:
                  arguments:
                    values: ['O', 'F', 'P']

      - name: lineitem
        description: "6M line items — one row per part per order"
        columns:
          - name: l_orderkey
            data_tests:
              - not_null
          - name: l_linenumber
            data_tests:
              - not_null

      - name: customer
        description: "150K customer accounts"
        columns:
          - name: c_custkey
            data_tests:
              - unique
              - not_null

      - name: nation
        description: "25 country reference rows"
        columns:
          - name: n_nationkey
            data_tests:
              - unique
              - not_null

      - name: supplier
        description: "10K suppliers"
```

*Part B question:* A source test failing but bronze passing means the bronze model is filtering or transforming in a way that hides the upstream issue — e.g. a DISTINCT or WHERE clause masking duplicates. Testing both layers is what catches this.

---

**Part D — gold schema.yml solution:**

Note: `total_revenue`, `avg_order_value`, `first_order_date`, `last_order_date` in `gold_customers` are intentionally left without `not_null` — customers with tier `'No Orders'` will have NULL in those columns from the LEFT JOIN.

```yaml
version: 2

models:
  - name: gold_orders
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
      - name: order_date
        data_tests:
          - not_null
      - name: order_status
        data_tests:
          - accepted_values:
              arguments:
                values: ['O', 'F', 'P']
      - name: order_priority
        data_tests:
          - accepted_values:
              arguments:
                values: ['1-URGENT', '2-HIGH', '3-MEDIUM', '4-NOT SPECIFIED', '5-LOW']
      - name: customer_id
        data_tests:
          - not_null
      - name: market_segment
        data_tests:
          - accepted_values:
              arguments:
                values: ['AUTOMOBILE', 'BUILDING', 'FURNITURE', 'MACHINERY', 'HOUSEHOLD']
      - name: total_items
        data_tests:
          - not_null
      - name: gross_revenue
        data_tests:
          - not_null
      - name: net_revenue
        data_tests:
          - not_null
      - name: discount_amount
        data_tests:
          - not_null
      - name: effective_discount_rate
        data_tests:
          - not_null
      - name: order_size_band
        data_tests:
          - accepted_values:
              arguments:
                values: ['Large', 'Medium', 'Small']

  - name: gold_customers
    columns:
      - name: customer_id
        data_tests:
          - unique
          - not_null
      - name: customer_name
        data_tests:
          - not_null
      - name: market_segment
        data_tests:
          - accepted_values:
              arguments:
                values: ['AUTOMOBILE', 'BUILDING', 'FURNITURE', 'MACHINERY', 'HOUSEHOLD']
      - name: nation_name
        data_tests:
          - not_null
      - name: total_orders
        data_tests:
          - not_null
      - name: customer_tier
        data_tests:
          - accepted_values:
              arguments:
                values: ['Platinum', 'Gold', 'Silver', 'No Orders']

  - name: gold_market_summary
    columns:
      - name: market_segment
        data_tests:
          - unique
          - not_null
          - accepted_values:
              arguments:
                values: ['AUTOMOBILE', 'BUILDING', 'FURNITURE', 'MACHINERY', 'HOUSEHOLD']
      - name: total_orders
        data_tests:
          - not_null
      - name: total_revenue
        data_tests:
          - not_null
      - name: avg_order_value
        data_tests:
          - not_null

  - name: gold_nations_summary
    columns:
      - name: nation_name
        data_tests:
          - unique
          - not_null
      - name: total_orders
        data_tests:
          - not_null
      - name: total_revenue
        data_tests:
          - not_null
      - name: avg_order_value
        data_tests:
          - not_null
```

---

**Part F — `severity: warn` vs `error`:**
The exercise deliberately omits `'P'` from the accepted values list so the test actually fires — TPC-H data contains all three statuses, so `['O', 'F', 'P']` would always pass and students would never see a warning in the log. Remind them to restore `'P'` after the exercise.

- `error`: primary key uniqueness, referential integrity, not_null on columns used in joins. Anything that breaks downstream consumers.
- `warn`: business-rule validations where anomalies are expected and monitored but don't block the pipeline (e.g. a new order status code appears in staging before the enum is officially updated).

---

**Part G — If bronze_tpch_customers fails its unique test:**
`silver_orders_enriched`, `gold_customers`, `gold_orders`, `gold_nations_summary` could all have duplicated rows or inflated aggregations. Use `dbt test --select bronze_tpch_customers+` to run tests on the model and everything downstream.

---

### Exercise 2 — Singular Tests

Singular tests are plain SQL in `tests/`. dbt considers a test passing if it returns 0 rows.

**assert_lineitem_revenue_matches_orders solution:**
```sql
WITH lineitem_totals AS (
    SELECT
        order_id,
        SUM(net_price) AS lineitem_net_revenue
    FROM {{ ref('bronze_tpch_lineitem') }}
    GROUP BY 1
),
order_totals AS (
    SELECT
        order_id,
        net_revenue AS order_net_revenue
    FROM {{ ref('gold_orders') }}
)

SELECT
    l.order_id,
    l.lineitem_net_revenue,
    o.order_net_revenue,
    ABS(l.lineitem_net_revenue - o.order_net_revenue) AS discrepancy
FROM lineitem_totals l
JOIN order_totals o ON l.order_id = o.order_id
WHERE ABS(l.lineitem_net_revenue - o.order_net_revenue) > 0.01
```

**assert_all_orders_have_customers solution:**
```sql
SELECT o.order_id, o.customer_id
FROM {{ ref('bronze_tpch_orders') }} o
LEFT JOIN {{ ref('bronze_tpch_customers') }} c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL
```
Should return 0 rows — TPC-H is referentially clean.

---

### Exercise 3 — Documentation

**Key teaching point:** `dbt docs generate` + `dbt docs serve` produces a full data catalog. This is the handoff artifact between data engineers and analysts.

`{{ doc("block_name") }}` lets you write long descriptions in separate `.md` files and reference them in `schema.yml`.

---

### Exercise 4 — Seeds

**Part C question answ
**Part C question answers:**
1. Seed lands in `RAW` because of `+schema: raw` in `dbt_project.yml` under `seeds:`.
2. Running `dbt seed` again without `--full-refresh` truncates and reloads the table (TRUNCATE + INSERT).

**Part D question answer:** `column_types` is configured in `dbt_project.yml` — the `seeds.yml` is only for descriptions. `customer_id` is stored as `NUMBER` (integer) and `account_balance` as `FLOAT`.

**Part E — seed / source / neither:**
1. 3-row status code mapping in git → **seed**
2. 200 cost centers updated quarterly as CSV → **seed**
3. Currency rates loaded into Snowflake by Airflow daily → **source**
4. 10M-row CSV on a local machine → **neither**

---

### Exercise 5 — dbt build & Selection

**Part A questions:**
- *Build order:* DAG order — parents before children. Each node runs its model then its tests before downstream nodes start.
- *Upstream test failure:* downstream models are skipped (`SKIP` status).
- *Prefer dbt run:* quick iteration during development, or when tests are slow and you're just checking SQL output.

**Part B — expected outputs:**
1. `dbt build --select silver_orders_enriched+` → `silver_orders_enriched`, `gold_orders`, `gold_customers`, `gold_market_summary`, `gold_nations_summary`
2. `dbt build --select +silver_orders_enriched` → all 4 bronze parents + `silver_orders_enriched`
3. `dbt build --select bronze.*+` → all bronze models + entire downstream project

**Part C:** Create two jobs in dbt Cloud: one hourly (`dbt build --select tag:hourly`), one daily (`dbt build --select tag:daily`).

**Part D:** `dbt build --exclude gold_orders`

**Part E — scenario answers:**
1. `dbt build --full-refresh`
2. `dbt build --select silver_orders_enriched+`
3. `dbt build --select bronze.*`
4. `dbt build --select gold_orders`
5. `dbt build --exclude assert_payment_matches_order`

---

## Day 4 — Incremental, Macros, Packages & Snapshots

### Schedule

| Time | Activity |
|---|---|
| 9:30–10:30 | Ex 1 — Incremental Part 2 (1 h) |
| 10:30–13:00 | Ex 2 — Macros (2.5 h) |
| 13:00–14:00 | Lunch |
| 14:00–15:30 | Ex 3 — Packages (1.5 h) |
| 15:30–17:30 | Ex 4 — Snapshots (2 h) |

---

### Exercise 1 — Incremental (Part 2)

**CSV files needed (all in `exercises/day-4/`):**
- `customers_seed_v2.csv` — 35 rows (original 30 + 5 new, October 2024)
- `customers_seed_v3.csv` — 35 rows, customer_id=1 updated (account_balance=9500, Nov 2024)

**Part A — silver_customers solution:**
```sql
{{ config(
    materialized = 'incremental',
    unique_key   = 'customer_id'
) }}

SELECT
    customer_id,
    first_name || ' ' || last_name AS full_name,
    email,
    country,
    account_balance,
    updated_at,
    CURRENT_TIMESTAMP()            AS loaded_at
FROM {{ ref('customers_seed') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**Part B — first run:** 30 rows, all same `loaded_at`, `is_incremental()` = false.

**Part C question answers:**
1. 5 rows — filter matched only October 2024 dates (IDs 31–35)
2. `loaded_at` on original 30 rows unchanged
3. Third run processes 0 rows — `MAX(updated_at)` = 2024-10-20, no rows are newer

**Part D — MERGE demo (v3):**
- After `dbt seed` + `dbt run --select silver_customers` → still **35 rows**
- Customer 1: `account_balance=9500`, `loaded_at` updated
- Without `unique_key`: INSERT instead of MERGE → **36 rows** (duplicate)

**Part D question answer — two real-world examples of duplicates without unique_key:**
1. A customer updates their profile twice in the same day — both events arrive in the next run and get inserted as separate rows.
2. A backfill re-runs a date range already loaded — all rows in that range get inserted again.

**Part E — full refresh:**
- All 35 rows get the same new `loaded_at`
- Use `--full-refresh` when: adding/renaming columns, changing the filter logic, or recovering from a corrupt/partial load.

---

### Exercise 2 — Macros

**Part D — `{% if execute %}`:** Prevents the DROP loop from running during the parse phase. In that phase `run_query()` returns `None`, so iterating over `results` would raise a type error.

**Part E — shipping_days solution:**
```sql
{% macro shipping_days(start_col, end_col, unit='day') %}
    DATEDIFF('{{ unit }}', {{ start_col }}, {{ end_col }})
{% endmacro %}
```

Usage in `bronze_tpch_lineitem.sql`:
```sql
{{ shipping_days('l_orderdate', 'l_shipdate') }}           AS days_to_ship,
{{ shipping_days('l_shipdate', 'l_receiptdate', 'week') }} AS weeks_to_receive,
```

**Question answers:**
1. Best pair for delivery performance: `l_commitdate` vs `l_receiptdate` — committed date is the SLA, receipt date is actual delivery.
2. Swapping start/end returns a negative number.

---

### Exercise 3 — Packages

**Part B — generate_surrogate_key question answers:**
- Expands to `MD5(CAST(order_id AS VARCHAR) || '-' || CAST(line_number AS VARCHAR))` — a deterministic hash of the input columns.
- Preferable to `ROW_NUMBER()` because `ROW_NUMBER()` changes when rows are added or reordered upstream. The surrogate key would shift, breaking any downstream joins that stored the old value.

**Part C — star() question answer:**
- With `star()`: new columns added to `bronze_tpch_orders` are automatically included in `silver_orders_enriched` on the next run. No code change needed.
- With manual `SELECT col1, col2, ...`: new columns are silently ignored until someone manually adds them.

**Part D — date_spine solution:**
```sql
{{ config(materialized='view') }}

SELECT
    CAST(date_day AS DATE) AS date_day
FROM (
    {{ dbt_utils.date_spine(
        datepart   = 'day',
        start_date = "CAST('1992-01-01' AS DATE)",
        end_date   = "CAST('1999-01-01' AS DATE)"
    ) }}
)
```
Expected: 2557 rows (`SELECT COUNT(*) FROM ANALYTICS.SILVER.DATE_SPINE`).

**Part E — expression_is_true solution:**
```yaml
- name: gold_orders
  data_tests:
    - dbt_utils.expression_is_true:
        arguments:
          expression: "net_revenue <= gross_revenue"
```

---

### Exercise 4 — Snapshots

**Prerequisite:** Students must have run `dbt seed` in Day 3 Ex 4.

**Part B questions:**
1. Rows after first run → **30**
2. `dbt_valid_to` for current rows → **NULL** (NULL = current version)
3. `dbt_scd_id` → surrogate key generated by hashing `unique_key + dbt_updated_at`. Uniquely identifies each historical version.

**Part C questions:**
1. Total rows after 3 updates → **33** (30 original + 3 new versions; old versions closed with non-NULL `dbt_valid_to`)
2. `dbt_valid_to` on old rows → timestamp when `dbt snapshot` was re-run after the UPDATEs
3. Point-in-time query (answer belongs only in instructor guide):
```sql
SELECT country
FROM ANALYTICS.SNAPSHOTS.CUSTOMERS_SNAPSHOT
WHERE customer_id = 1
  AND '2024-01-15' >= dbt_valid_from
  AND (dbt_valid_to > '2024-01-15' OR dbt_valid_to IS NULL)
```

**Part D question answers:**
1. 6 rows for customers 1–3 because each has 2 versions: the original (closed) and the updated (current).
2. The point-in-time query returns the pre-update values, proving that snapshots preserve history — the original data is never overwritten.
3. `WHERE dbt_valid_to IS NULL` always returns exactly one row per customer — the current version. Past versions have a non-NULL `dbt_valid_to`.

**Part E — check strategy solution:**
```sql
{% snapshot customers_snapshot_check %}

{{ config(
    target_schema = 'snapshots',
    unique_key    = 'customer_id',
    strategy      = 'check',
    check_cols    = ['country', 'email'],
) }}

SELECT
  *
FROM {{ ref('customers_seed') }}

{% endsnapshot %}
```

```sql
UPDATE ANALYTICS.RAW.CUSTOMERS_SEED
SET email = 'sofia.garcia.check@email.com'
WHERE customer_id = 2;
```

**Part E question answer — timestamp vs check:**
- `timestamp`: compares `updated_at` between source and snapshot — fast, only touches changed rows. Requires the source to maintain a reliable `updated_at`.
- `check`: compares the actual column values on every run — slower (full scan every time), but works when source systems don't maintain `updated_at` reliably.
- Use `timestamp` by default. Use `check` when you can't trust the source's update timestamp.

---

## Day 5 — dbt Core, dbt in Snowflake & CI/CD

### Schedule

| Time | Activity |
|---|---|
| 9:30–11:00 | Ex 1 — dbt Core locally (1.5 h) |
| 11:00–12:30 | Ex 2 — dbt in Snowflake (1.5 h) |
| 12:30–13:00 | Ex 3 — Jobs & Slim CI, part 1 (30 min) |
| 13:00–14:00 | Lunch |
| 14:00–15:00 | Ex 3 — Jobs & Slim CI, part 2 (1 h) |
| 15:00–17:30 | Ex 4 — Capstone (2.5 h) |

---

### Exercise 1 — dbt Core Locally

**Part F — What the three commands do:**

-  — reads and prints the compiled SQL that dbt sent to Snowflake for that model. All Jinja is resolved:  replaced with real schema-qualified paths, macros fully expanded. Use this to debug unexpected SQL or verify a macro rendered correctly.

-  — parses  and shows a table with each model's name, whether it succeeded or failed, and how many seconds it took. This is the local equivalent of the run log in dbt Cloud.

-  — lists the first 20 node IDs from , which is the full project DAG serialized as JSON. Every model, test, seed, and snapshot is a node. Slim CI uses this file to diff what changed between a branch and the last production run.

**Part F — Question answers:**

- *What does  contain? Why is it important for CI?* → The complete DAG: every node (model, test, seed, snapshot), its dependencies, compiled SQL, and metadata. Slim CI diffs the branch manifest against the production manifest to determine  — which models changed and need to be rebuilt.

- *What is the difference between  and ?* →  contains the Jinja-resolved SQL (what dbt prepared to send).  contains the final SQL actually executed — typically wrapped in a  or  statement. Compare them to see the full DDL dbt ran.

- *In dbt Cloud, where do you find the equivalent of ?* → In the run detail page under **Deploy → Jobs → [job] → [run]**, in the **Artifacts** tab. You can download  directly from there.

---

### Exercise 1 — dbt Core Locally

**Part F — What the three commands do:**

- `Get-Content target\compiled\...\gold_orders.sql` — reads the compiled SQL that dbt sent to Snowflake. All Jinja is resolved: `ref()` replaced with real schema-qualified paths, macros fully expanded. Use this to debug unexpected SQL or verify a macro rendered correctly.

- `run_results.json` command — parses run results and shows a table with each model's name, whether it succeeded or failed, and execution time in seconds. Local equivalent of the run log in dbt Cloud.

- `manifest.json` command — lists the first 20 node IDs from the manifest, which is the full project DAG serialized as JSON. Every model, test, seed, and snapshot is a node. Slim CI diffs the branch manifest against the production manifest to determine which models changed.

**Part F — Question answers:**

- *What does `target/manifest.json` contain? Why is it important for CI?* → The complete DAG: every node (model, test, seed, snapshot), its dependencies, compiled SQL, and metadata. Slim CI diffs the branch manifest against the production manifest to determine `state:modified+` — which models changed and need to be rebuilt.

- *What is the difference between `target/compiled/` and `target/run/`?* → `compiled/` contains the Jinja-resolved SQL (what dbt prepared to send). `target/run/` contains the final SQL actually executed — typically wrapped in a `CREATE TABLE AS` or `CREATE VIEW AS` statement. Compare them to see the full DDL dbt ran.

- *In dbt Cloud, where do you find the equivalent of `run_results.json`?* → In the run detail page under **Deploy → Jobs → [job] → [run]**, in the **Artifacts** tab. Download `run_results.json` directly from there.

---

### Exercise 3 — Jobs & Slim CI

**Slim CI command:**
```bash
dbt build --select state:modified+ --defer --state ./target
```
`--defer` means: for any node NOT in `state:modified+`, use the production version from the warehouse instead of rebuilding.

---

### Common Cross-Day Errors

| Error | Cause | Fix |
|---|---|---|
| `object 'X' does not exist` | Parent model not run yet | `dbt run --select +model_name` |
| `Database 'ANALYTICS' does not exist` | Wrong database in connection | Check dbt Cloud connection settings |
| `role 'TRANSFORMER' does not exist` | Wrong role | Re-run Day 1 SQL setup with ACCOUNTADMIN |
| `Snowflake syntax error` | Jinja rendered unexpected value | `dbt compile --select model_name` to see rendered SQL |
| `Snapshot did not detect changes` | `updated_at` not updated | UPDATE must set both the changed column AND `updated_at = CURRENT_TIMESTAMP()` |
| `dbt deps: package not found` | Typo or wrong version in packages.yml | Check hub.getdbt.com |
| `is_incremental() always false` | Model materialized as view | Incremental models need `config(m