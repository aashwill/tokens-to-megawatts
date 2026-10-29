# Glossary

Day01: **Utilization:** share of installed GPUs that are busy.
**Headroom:** installed minus in use; the buffer for spikes and failures.
**Planning threshold:** utilization level (often 80–85%) that triggers action.
**Peak vs average:** size capacity to the peak (or P95); the average hides the afternoon crunch.
**GPU-hours:** GPUs × hours used; the basic unit of compute demand.
**Coincident vs non-coincident peak:** whether demand types peak at the same time; decides fleet peak vs sum of peaks.

Right-censored job: still running when the trace ends; count its usage up to the trace end, don't drop it
Fan-out: a join on a non-unique key that silently duplicates rows and inflates totals
Anti-join: rows in one table with no match in another (LEFT JOIN ... WHERE right.key IS NULL)
Reconciliation: proving every input row is kept, de-duplicated, or dropped with a reason

Day03:

 Explain why dropping rows with a missing end_time would understate recent demand
 Say what validate="many_to_one" protects against, in one sentence
 clean.merge(specs, on="gpu_type", how="left", validate="many_to_one")
 #the above code means specs dataframe cannot have duplicates in gpu_type in the right table for the join to happen/ if it does then many_to_one will prevent the join and throw and error
 
 Name the four checks you'd run first on any new trace: 
 1. rows reconcile (raw = clean + dupes + dropped)
 2. no negative durations
 3. merge kept GPU-hours
 4. every known GPU type has specs

Day04:
Grain: what one row of a table represents; check it before any join or count
Partitioning: splitting a big table into files by a column (here day and hour) so you can read only what you need
Installed / allocated / used: supply, reserved, and actually busy; the gaps are idle capacity and reserved-but-idle capacity
Pending demand: pods waiting for GPUs; unmet demand that utilization alone hides
