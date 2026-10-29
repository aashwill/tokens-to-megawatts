---
## Part 5: Predictions, before loading a single row

- Which columns does the schema say *can be null*?
- What does the schema say about pods with zero GPU request?
- What state buckets exist, and which ones should count as demand?
- Is a pod's `used_gpu_hours` allowed to be bigger than its `gpu_request`?
- The partition columns are in the **folder path**. Will they be inside the file?
- What timezone is `hour` in?

| # | Expected issue | How I'll detect it |
|---|---|---|
| 1 |null workload_id | pods["workload_id"].isna().mean() |
| 2 | Demand|anything except Unknown |
| 3 | used_gpu_hours>gpu_request|(g["used_gpu_hours"] > g["gpu_request"]).sum() |
| 4 |no timezone for hour |use an assumption |
| 5 | day/hour not inside the parquet file (they're in the folder path)| pods.columns to understand which columns the dataset has|
| 6 |First and last days are partial (Day 2 partial-week trap) |file_size in pod_parts function() to determine the size of every day |

Suprsied to find that pods["state_public"].value_counts() returns just unknown values in state_public --Have a check or find a workworund for this
