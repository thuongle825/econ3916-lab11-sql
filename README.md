# econ3916-lab11-sql
# SQL for Economists — Joins, Windows and the Cardinality Trap

## Objective
I translated pandas operations into SQL with DuckDB to see that they are the same operations, and used that to find where a join, a missing value or an AI translation can silently change a result.

## Methodology
- Built two synthetic tables with a fixed seed: `customers` (600 rows, one per customer, with region and revenue) and `orders` (3,612 rows, several per customer).
- Ran select, filter, sort, `groupby` and `merge` in pandas and the same steps in SQL with DuckDB, and checked that both returned the same rows.
- Joined `customers` to `orders` with a `LEFT JOIN`, counted rows before and after, and ran the same Welch t-test of East against West revenue on both tables.
- Used window functions (`AVG(...) OVER (PARTITION BY region)` and `RANK()`) to attach each region's average and each customer's rank to every row without collapsing rows.
- Built a five-row survey with two missing incomes and compared `WHERE income != 50000` in SQL with the same filter in pandas.
- Queried a public Parquet file of about 3 million NYC taxi trips over HTTP with `read_parquet`, without downloading it, and grouped trips by passenger count.
- Asked an AI to translate three of my pandas steps (`groupby`, `merge` and a filter) into SQL, revised the prompt once, and compared the row counts of its queries with my pandas results.

## Key Findings
- **pandas and SQL agree.** Each pandas step and its SQL version returned the same table. Grouping by region gave East 282 customers with mean revenue 490.91 and West 318 with 490.22 both ways.
- **The cardinality trap.** The one-to-many `LEFT JOIN` turned 600 customer rows into 3,612 rows with no error or warning, because each customer's revenue was repeated once per order. The same Welch t-test on the same revenue figures went from t = 0.07 (p = 0.9413) to t = 0.71 (p = 0.4749). The repeated rows shrank the standard error from 9.35 to 3.75 and moved the East–West difference from 0.69 to 2.68, since customers with more orders counted more times.
- **Window functions keep every row.** 600 rows went in and 600 came out, each carrying its region's average and its rank within the region.
- **Three-valued logic.** `WHERE income != 50000` kept 3 of 5 survey rows, because comparing a missing income with 50,000 is unknown rather than true, and `WHERE` keeps only rows where the condition is true. The same filter in pandas kept 5. Adding `OR income IS NULL` makes SQL keep all 5.
- **Querying a file where it sits.** The taxi query read only the 3 columns it needed out of 19, applied the filters while reading, and returned 6 rows, one per passenger count from 1 to 6. The filter also dropped trips with a missing passenger count.
- **Checking AI translations.** All three of the AI's queries matched my pandas row counts in both versions: 2, 3,612 and 5. In version 1 the AI already handled the missing incomes in the filter on its own; the choice it made by itself was to ignore row order. Version 2 asked for rows in the same order as pandas. Because the counts were identical in both versions, row counts alone could not tell the two apart.
