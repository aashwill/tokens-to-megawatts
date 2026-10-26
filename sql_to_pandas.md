# SQL → pandas cheat sheet

Examples use a jobs table called `jobs` (columns like `job_id`, `user`, `status`, `gpu_type`, `gpus`, `gpu_hours`, `submit_ts`) and a lookup table `specs`. Every pandas line here has been run.

```python
import pandas as pd
import numpy as np
jobs = pd.read_csv("trace_jobs.csv")
```

**Golden rules**
- Wrap every condition in brackets and use `&` `|` `~`, not `and` `or` `not`.
- Missing values (`NaN`) behave like SQL `NULL`: `NaN == NaN` is False, and `sum()` and `mean()` skip them.
- Most commands return a **new** table. Save the result (`jobs = jobs.sort_values(...)`) or it's lost.

---

## 1. Look at the data

| SQL | pandas | Watch out |
|---|---|---|
| `SELECT * FROM jobs LIMIT 5` | `jobs.head(5)` | `jobs.tail(5)` for the last rows, `jobs.sample(5)` for random ones |
| `SELECT COUNT(*) FROM jobs` | `len(jobs)` | |
| `DESCRIBE jobs` | `jobs.info()` or `jobs.dtypes` | `info()` also shows non-null counts |
| `SELECT MIN, MAX, AVG ...` on all numbers | `jobs.describe()` | |
| `COUNT(DISTINCT gpu_type)` | `jobs["gpu_type"].nunique()` | Ignores NaN |
| `SELECT DISTINCT gpu_type` | `jobs["gpu_type"].unique()` | Includes NaN |
| `SELECT gpu_type, COUNT(*) GROUP BY gpu_type ORDER BY 2 DESC` | `jobs["gpu_type"].value_counts(dropna=False)` | Without `dropna=False`, missing values are hidden |
| `COUNT(*) - COUNT(col)` (nulls per column) | `jobs.isna().sum()` | |

## 2. SELECT columns

| SQL | pandas | Watch out |
|---|---|---|
| `SELECT gpu_type, gpus` | `jobs[["gpu_type", "gpus"]]` | Double brackets give a table; single brackets give one column (a Series) |
| `SELECT gpus AS gpu_count` | `jobs.rename(columns={"gpus": "gpu_count"})` | |
| `SELECT *, gpus * 24 AS gpu_day` | `jobs["gpu_day"] = jobs["gpus"] * 24` | Changes `jobs` in place |
| Same, inside a chain | `jobs.assign(gpu_day=lambda d: d["gpus"] * 24)` | Returns a new table |
| `SELECT * EXCEPT (user)` (DuckDB) | `jobs.drop(columns=["user"])` | |
| `CAST(gpus AS INT)` | `jobs["gpus"].astype(int)` | Fails if the column has NaN |
| `CAST(x AS DOUBLE)` from messy text | `pd.to_numeric(jobs["x"], errors="coerce")` | Bad values become NaN instead of erroring |

## 3. WHERE

| SQL | pandas | Watch out |
|---|---|---|
| `WHERE gpu_type = 'V100'` | `jobs[jobs["gpu_type"] == "V100"]` | `==`, not `=` |
| `WHERE gpus > 1 AND status = 'Failed'` | `jobs[(jobs["gpus"] > 1) & (jobs["status"] == "Failed")]` | Brackets around each condition are required |
| `WHERE a OR b` | `jobs[(cond_a) \| (cond_b)]` | |
| `WHERE NOT (status = 'Failed')` | `jobs[~(jobs["status"] == "Failed")]` | |
| `WHERE gpu_type IN ('V100', 'T4')` | `jobs[jobs["gpu_type"].isin(["V100", "T4"])]` | |
| `WHERE gpu_type NOT IN (...)` | `jobs[~jobs["gpu_type"].isin(["V100", "T4"])]` | Unlike SQL, NaN rows are kept |
| `WHERE gpus BETWEEN 1 AND 4` | `jobs[jobs["gpus"].between(1, 4)]` | Inclusive at both ends, like SQL |
| `WHERE end_time IS NULL` | `jobs[jobs["end_time"].isna()]` | Never `== np.nan`; it always gives False |
| `WHERE end_time IS NOT NULL` | `jobs[jobs["end_time"].notna()]` | |
| `WHERE user LIKE 'user_0%'` | `jobs[jobs["user"].str.startswith("user_0")]` | `.str.endswith()` for `'%x'` |
| `WHERE gpu_type LIKE '%100%'` | `jobs[jobs["gpu_type"].str.contains("100", na=False)]` | `na=False` stops NaN rows from breaking the filter |
| `WHERE LOWER(gpu_type) = 'v100'` | `jobs[jobs["gpu_type"].str.lower() == "v100"]` | |
| SQL-style text filter | `jobs.query("gpus > 1 and status == 'Failed'")` | Inside `query()` you can write `and` / `or` |

## 4. ORDER BY and LIMIT

| SQL | pandas | Watch out |
|---|---|---|
| `ORDER BY gpu_hours DESC` | `jobs.sort_values("gpu_hours", ascending=False)` | NaN goes last by default (`na_position="first"` to change it) |
| `ORDER BY gpu_type, gpu_hours DESC` | `jobs.sort_values(["gpu_type", "gpu_hours"], ascending=[True, False])` | |
| `ORDER BY gpu_hours DESC LIMIT 10` | `jobs.nlargest(10, "gpu_hours")` | `nsmallest` for the opposite |
| `LIMIT 10 OFFSET 20` | `jobs.iloc[20:30]` | Positions start at 0 |

## 5. GROUP BY

| SQL | pandas | Watch out |
|---|---|---|
| `SELECT gpu_type, SUM(gpu_hours) GROUP BY gpu_type` | `jobs.groupby("gpu_type", as_index=False)["gpu_hours"].sum()` | `as_index=False` keeps `gpu_type` as a normal column |
| `COUNT(*)` per group | `jobs.groupby("gpu_type").size()` | Counts rows, including NaN values |
| `COUNT(col)` per group | `jobs.groupby("gpu_type")["end_time"].count()` | Counts non-null values only, like SQL |
| `COUNT(DISTINCT user)` per group | `jobs.groupby("gpu_type")["user"].nunique()` | |
| Several aggregates with names | `jobs.groupby("gpu_type", as_index=False).agg(jobs=("job_id", "count"), gpu_hours=("gpu_hours", "sum"), avg_gpus=("gpus", "mean"))` | Format: `new_name=("column", "function")` |
| `GROUP BY gpu_type, status` | `jobs.groupby(["gpu_type", "status"], as_index=False)["gpu_hours"].sum()` | |
| `HAVING SUM(gpu_hours) > 1000` | `g = jobs.groupby("gpu_type", as_index=False)["gpu_hours"].sum()` then `g[g["gpu_hours"] > 1000]` | No HAVING in pandas: group first, then filter |
| Keep a NULL group | `jobs.groupby("gpu_type", dropna=False)...` | Without `dropna=False`, rows with missing `gpu_type` silently disappear from the result |

Functions you can use in `agg`: `sum`, `mean`, `median`, `min`, `max`, `count`, `nunique`, `std`, `first`, `last`.

## 6. Joins and UNION

| SQL | pandas | Watch out |
|---|---|---|
| `INNER JOIN specs USING (gpu_type)` | `jobs.merge(specs, on="gpu_type", how="inner")` | `how="inner"` is the default |
| `LEFT JOIN` | `jobs.merge(specs, on="gpu_type", how="left")` | |
| `RIGHT JOIN` / `FULL OUTER JOIN` | `how="right"` / `how="outer"` | |
| `CROSS JOIN` | `jobs.merge(specs, how="cross")` | |
| `ON a.type = b.gpu_model` | `jobs.merge(specs, left_on="type", right_on="gpu_model")` | Both key columns are kept; drop one afterwards |
| Join on two columns | `on=["gpu_type", "region"]` | |
| Guard against fan-out | `jobs.merge(specs, on="gpu_type", how="left", validate="many_to_one")` | Errors if `specs` has duplicate keys; **use on every lookup join** |
| Anti-join (`LEFT JOIN ... WHERE b.key IS NULL`) | `m = jobs.merge(specs, on="gpu_type", how="left", indicator=True)` then `m[m["_merge"] == "left_only"]` | `_merge` is `both`, `left_only` or `right_only` |
| `WHERE gpu_type IN (SELECT gpu_type FROM specs)` | `jobs[jobs["gpu_type"].isin(specs["gpu_type"])]` | A semi-join; never adds rows |
| `UNION ALL` | `pd.concat([a, b], ignore_index=True)` | Columns match by name, not position |
| `UNION` (removes duplicates) | `pd.concat([a, b]).drop_duplicates()` | |
| `SELECT DISTINCT *` | `jobs.drop_duplicates()` | `subset=["job_id"]` to de-duplicate on one key |

**After any join:** check `len()` before and after, and the total of a key number such as GPU-hours.

## 7. CASE WHEN

| SQL | pandas | Watch out |
|---|---|---|
| `CASE WHEN gpus >= 4 THEN 'large' ELSE 'small' END` | `np.where(jobs["gpus"] >= 4, "large", "small")` | For one condition |
| Several `WHEN`s | `np.select([jobs["gpus"] >= 4, jobs["gpus"] >= 1], ["large", "medium"], default="small")` | First true condition wins, like SQL |
| `CASE gpu_type WHEN 'TESLA-V100' THEN 'V100' ... ELSE gpu_type END` | `jobs["gpu_type"].replace({"TESLA-V100": "V100"})` | Values not in the dict stay as they are |
| Same, but unmatched become NULL | `jobs["gpu_type"].map({"V100": 300, "T4": 70})` | Unmatched values become NaN |
| Buckets (`CASE WHEN x < 1 ... WHEN x < 4 ...`) | `pd.cut(jobs["gpus"], bins=[0, 1, 4, np.inf], labels=["<=1", "1-4", ">4"])` | Bins include the right edge: (0, 1], (1, 4] |

## 8. NULL handling

| SQL | pandas | Watch out |
|---|---|---|
| `COALESCE(gpu_type, 'UNKNOWN')` | `jobs["gpu_type"].fillna("UNKNOWN")` | |
| `COALESCE(a, b)` | `jobs["a"].fillna(jobs["b"])` | |
| `NULLIF(gpus, 0)` | `jobs["gpus"].replace(0, np.nan)` | Handy before dividing |
| Delete rows where a column is NULL | `jobs.dropna(subset=["end_time"])` | Plain `dropna()` drops a row if **any** column is NaN; think first |
| `NULL = NULL` | `np.nan == np.nan` → `False` | Same as SQL; Exercise 1 on Day 3 |
| `SUM()` ignores NULL | `jobs["kwh"].sum()` | Same; a column that's all NaN sums to 0, not NULL |

## 9. Strings

Add `.str` before any string function.

| SQL | pandas |
|---|---|
| `UPPER(x)` / `LOWER(x)` | `jobs["x"].str.upper()` / `.str.lower()` |
| `TRIM(x)` | `jobs["x"].str.strip()` |
| `REPLACE(x, '-', '_')` | `jobs["x"].str.replace("-", "_")` |
| `LENGTH(x)` | `jobs["x"].str.len()` |
| `SUBSTRING(x, 1, 4)` | `jobs["x"].str[0:4]` (positions start at 0) |
| `SPLIT_PART(job_id, '_', 2)` | `jobs["job_id"].str.split("_").str[1]` |
| `a \|\| '-' \|\| b` (CONCAT) | `jobs["a"] + "-" + jobs["b"]` (use `.astype(str)` on number columns first) |
| `REGEXP_MATCHES(x, 'V\d+')` | `jobs["x"].str.contains(r"V\d+", na=False)` |

## 10. Dates and times

Add `.dt` before any date function.

| SQL | pandas | Watch out |
|---|---|---|
| `CAST(s AS TIMESTAMP)` | `pd.to_datetime(jobs["s"])` | |
| `TO_TIMESTAMP(epoch_seconds)` | `pd.to_datetime(jobs["t"], unit="s")` | |
| Seconds since trace start → timestamp | `pd.Timestamp("2026-03-04") + pd.to_timedelta(jobs["submit_time"], unit="s")` | Pattern from Days 2–3 |
| `DATE_TRUNC('day', ts)` | `jobs["submit_ts"].dt.floor("D")` | `"h"` for hour |
| `DATE_TRUNC('week', ts)` | `jobs["submit_ts"].dt.to_period("W").dt.start_time` | Weeks start on Monday |
| `DATE_TRUNC('month', ts)` | `jobs["submit_ts"].dt.to_period("M").dt.start_time` | |
| `EXTRACT(HOUR FROM ts)` | `jobs["submit_ts"].dt.hour` | `.dt.day`, `.dt.month`, `.dt.year` too |
| `EXTRACT(DOW FROM ts)` | `jobs["submit_ts"].dt.dayofweek` | **Monday = 0** in pandas; many SQL engines use Sunday = 0 |
| Day name | `jobs["submit_ts"].dt.day_name()` | |
| `ts + INTERVAL '2 hours'` | `jobs["submit_ts"] + pd.Timedelta(hours=2)` | |
| `DATEDIFF('second', a, b)` | `(jobs["b"] - jobs["a"]).dt.total_seconds()` | Subtracting gives a duration; `.dt.total_seconds()` turns it into a number |
| `WHERE ts >= '2026-03-10'` | `jobs[jobs["submit_ts"] >= "2026-03-10"]` | Text dates compare correctly against a datetime column |
| `GROUP BY DATE_TRUNC('day', ts)` | `jobs.groupby(jobs["submit_ts"].dt.floor("D"))["gpu_hours"].sum()` | |
| Same, filling empty days with 0 | `jobs.set_index("submit_ts").resample("D")["gpu_hours"].sum()` | `resample` adds days that had no rows; `groupby` doesn't |

## 11. Window functions

| SQL | pandas | Watch out |
|---|---|---|
| `SUM(gpu_hours) OVER (PARTITION BY gpu_type)` | `jobs.groupby("gpu_type")["gpu_hours"].transform("sum")` | `transform` keeps every row; `sum()` collapses to one row per group |
| Share of group total | `jobs["gpu_hours"] / jobs.groupby("gpu_type")["gpu_hours"].transform("sum")` | |
| `ROW_NUMBER() OVER (PARTITION BY user ORDER BY submit_ts)` | `jobs.sort_values("submit_ts").groupby("user").cumcount() + 1` | Sort first; `cumcount` starts at 0 |
| `RANK() OVER (ORDER BY gpu_hours DESC)` | `jobs["gpu_hours"].rank(method="min", ascending=False)` | `method="dense"` for `DENSE_RANK` |
| Running total `SUM(x) OVER (ORDER BY day)` | `daily["gpu_hours"].cumsum()` | Sort by date first |
| Running total per group | `daily.groupby("gpu_type")["gpu_hours"].cumsum()` | |
| `AVG(x) OVER (ORDER BY day ROWS 6 PRECEDING)` | `daily["gpu_hours"].rolling(7).mean()` | First 6 rows are NaN; `min_periods=1` to avoid that |
| Same, per group | `daily.groupby("gpu_type")["gpu_hours"].transform(lambda s: s.rolling(7).mean())` | |
| `LAG(x, 1) OVER (ORDER BY day)` | `daily["gpu_hours"].shift(1)` | |
| `LEAD(x, 1)` | `daily["gpu_hours"].shift(-1)` | |
| `LAG` per group | `daily.groupby("gpu_type")["gpu_hours"].shift(1)` | |
| Week-over-week % change | `weekly["gpu_hours"].pct_change()` | Same as `x / LAG(x) - 1` |
| `PERCENTILE_CONT(0.95)` | `jobs["gpu_hours"].quantile(0.95)` | Per group: `groupby(...)["gpu_hours"].quantile(0.95)` |

## 12. Pivot and unpivot

| SQL | pandas | Watch out |
|---|---|---|
| `SUM(CASE WHEN gpu_type='V100' THEN gpu_hours END) AS v100, ...` per day | `jobs.pivot_table(index="day", columns="gpu_type", values="gpu_hours", aggfunc="sum", fill_value=0)` | One column per GPU type; `fill_value=0` for empty cells |
| Add a totals row and column | `pivot_table(..., margins=True)` | |
| Quick count crosstab | `pd.crosstab(jobs["status"], jobs["gpu_type"])` | |
| `UNPIVOT` (wide → long) | `wide.melt(id_vars="day", var_name="gpu_type", value_name="gpu_hours")` | |

## 13. Save and query

| SQL | pandas |
|---|---|
| `CREATE TABLE clean AS SELECT ...` | `clean.to_csv("clean.csv", index=False)` or `clean.to_parquet("clean.parquet")` |
| `COPY ... FROM 'file.csv'` | `pd.read_csv("file.csv")` (`pd.read_parquet` for Parquet) |
| Run real SQL on a pandas table | `import duckdb` then `duckdb.sql("SELECT gpu_type, SUM(gpu_hours) FROM jobs GROUP BY 1").df()` |

The DuckDB line is handy for large Alibaba trace files: write the SQL you know, get a pandas table back.

---

## The 8 gaps that cause real bugs

1. **`value_counts()` and `groupby` drop NaN by default.** Pass `dropna=False` when you need to see missing values.
2. **`NaN == NaN` is False.** Use `.isna()`, never `== np.nan`.
3. **`merge` never warns about duplicate keys.** Use `validate="many_to_one"` and check row counts.
4. **`and` / `or` don't work on columns.** Use `&` / `|` with brackets around each condition.
5. **Most methods return a copy.** `jobs.sort_values(...)` alone changes nothing; save it.
6. **`dropna()` with no arguments drops a row if any column is missing.** Use `subset=`.
7. **`dayofweek` counts Monday as 0.** Check what your SQL engine uses before comparing.
8. **`groupby` skips empty days; `resample` fills them.** This matters for averages and peaks.
