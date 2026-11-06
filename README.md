# PostgreSQL Index Performance Lab

A small, reproducible experiment investigating how a B-tree index affects a selective lookup on a synthetic e-commerce orders table in PostgreSQL.

## Objective

Does an index on `customer_id` reduce the execution time and work required to retrieve a customer's orders?

I compared the same query before and after creating an index, using PostgreSQL's `EXPLAIN (ANALYZE, BUFFERS)`.

## Dataset

The experiment uses a synthetic table generated with PostgreSQL's `generate_series()`.

- 1,000,000 orders
- 10,000 customer IDs
- 100 orders per customer
- Columns: `order_id`, `customer_id`, `order_date`, `amount`, `status`
- Primary key: `order_id`
- Tested lookup: `customer_id = 4242`

The lookup returns 100 rows, representing 0.01% of the table.
### Data Distribution

The generated dataset contains four order statuses distributed equally:

| Order Status | Percentage|
|              |           |
| Pending      | 25%       |
| Shipped      | 25%       |
| Delivered    | 25%       |
| Cancelled    | 25%       |

Each customer has 100 orders, and the order dates are distributed across a 730-day period.

## Query under test

```sql
SELECT order_id, order_date, amount, status
FROM index_lab_orders
WHERE customer_id = 4242;
```

## Experiment

1. Create and populate the table.
2. Run the query and capture its execution plan before adding the customer index.
3. Create a B-tree index on `customer_id`.
4. Run `ANALYZE`.
5. Execute the same query again under normal planner settings.
6. Compare execution plans, buffer activity, and five recorded execution times per condition.

## Results

| Metric | Before index | After index |
|---|---:|---:|
| Median execution time | 74.725 ms | 0.315 ms |
| Scan plan | Parallel Seq Scan | Bitmap Index + Bitmap Heap Scan |
| Rows returned | 100 | 100 |
| Shared buffer hits | 7,875 | 103 |

The observed median execution time was approximately 237 times lower after the index was created.

The indexed plan used `idx_orders_customer_id` to identify matching rows, then accessed the corresponding heap blocks. The original plan scanned the table in parallel and filtered out nonmatching rows.

## Interpretation

For this selective lookup, the index-based plan accessed substantially fewer shared-buffer pages and had a much lower measured execution time.

The result illustrates why indexes can be useful when a query retrieves a small subset of a table.

It does not establish that every query becomes faster with an index. PostgreSQL may choose a sequential scan when a query needs a large proportion of the table, and indexes have storage and write-maintenance costs.

## Limitations

- The dataset is synthetic and has a deliberately uniform customer distribution.
- Only one lookup pattern was tested.
- Five timings per condition are a small sample.
- Cache state and system activity were not controlled rigorously.
- The baseline timings varied, including one unusually high measurement.
- The measured speedup is specific to this setup; it should not be generalized to production workloads.

A more rigorous follow-up would repeat the benchmark under controlled conditions, test multiple selectivities, and examine the index's write and storage costs.

## Reproduce

1. Use a PostgreSQL database for experimentation.
2. Execute `sql/01_setup.sql`.
3. Run the baseline section of `sql/02_benchmark.sql`.
4. Create the index and run `ANALYZE`.
5. Run the indexed query with normal planner settings.
6. Compare your plans with the files in `results/`.

See `results/benchmark_results.csv` for the recorded timings and the saved execution plans for the original evidence.
