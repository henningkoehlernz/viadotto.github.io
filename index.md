---
title: Key/FK Mining
---
## Overview

This tutorial provides a brief introduction to the Data Viadotto profiling API.
Rather than focusing on request details, it aims to provide an overview about services provided, and typical workflows.

The Data Viadotto API provides tools for discovering key and foreign key constraints by analysing relationships between data values.
This is in contrast to more basic data visualization tools, which simply visualize constraints explicitly defined.
It can thus be employed in cases where constraints are not fully known, or not known at all.
A common use case is ER schema reconstruction, but it can help with more general data exploration tasks such as data lake traversal as well.

Inputs are currently limited to relational data, but a variety of storage formats such as relational databases or CSV files are supported.

A key point of distinction between our and other data profiling tools lies in how it deals with incomplete or dirty data.

### Incomplete Data

When profiling data sets with missing values, proper handling of missing (null) values is critical.
E.g. treating null values as any other domain values can cause valid constraints to be missed.
The API supports multiple semantics in dealing with them, which will be described later on.

### Dirty Data

When constraints aren't actively enforced, they tend to get violated over time.
Thus it becomes important to search for constraints that *almost* hold.
Our profiling tool uses multiple metrics to measure the degree of constraint satisfaction, which can be used for filtering.
The optimal parameters to use here vary between data sets, and typically require some experimentation to find.

## Sampling

The first step in profiling a data set is sampling, where a suitable subset of rows is selected from each table.
The main purpose of this step is to speed up constraint discovery.
Additionally, basic characteristics such as number of null and distinct values for each column are computed as part of the sampling process, which are needed during profiling.
For this reason, constraint discovery will *always* operate on these samples.

For tables with few rows, all rows will be included in the sample.
For larger tables the *number* of rows sampled increases with the table size, but the *fraction* of rows sampled becomes smaller as the table grows.
Although sampling will inevitably result in some inaccuracies for the measures computed, sampling rates are chosen to ensure that these inaccuracies remain small.

It is possible to force all rows to be included in a sample -- doing so will avoid inaccuracies, but at the cost of reduced speed and increased memory usage.

## Mining keys

Our tool currently supports two profiling operations for keys:

- **Key Discovery**: Find all minimal column sets that form a key for a given table.
- **Key Analysis**: Compute uniqueness metrics for a given column set, and extract data examples.

For key discovery, we note that just because a constraint happens to hold for a table does not guarantee that it is meaningful.
In practice, many of the key constraints discovered will be accidental, meaning they hold only by chance -- this is particularly true for tables with few rows.

As accidental keys are typically of little interest, the set of minimal keys returned is best seen as a starting point for further manual evaluation by a domain expert.
Key analysis for selected column sets can be helpful in this.

### Incomplete Data

Currently the only supported semantic for dealing with null values is *possible semantic*, corresponding to UNIQUE constraints in SQL.
Here a column set **K** is a key if no two rows violate it, meaning they have the same values on **K**, and none of these values is missing.
Thus the column set { Name, DoB } is a key for the table below, but { Name, Salary } is not.

| Name  | DoB      | Salary | Job       |
| ----- | -------- | ------ | --------- |
| John  | 01/03/85 | 90k    | Developer |
| John  |          | 90k    | Analyst   |
| Susan | 01/03/85 |        |           |
| Dave  | 06/06/91 | 75k    | Tester    |

Note that { DoB, Salary } and { Job } are also keys for the given table, though likely accidental.

### Dirty Data

Key analysis computes the *uniqueness coefficient* of the provided column set.
This is the fraction of tuples which do not violate the key constraint (together with some other tuple).
E.g. for the table above, the uniqueness coefficient of { Name } is 0.5, while that of { Name, DoB } is 1.

Generally, uniqueness coefficients close to 1 indicate that the column set might be a key, with a few dirty data values causing it to be violated.
For low uniqueness coefficients this is unlikely.

## Mining foreign keys

Rather than just focusing on foreign keys, our tool searches for *inclusion dependencies* (INDs), meaning that the column set referenced does not have to be a key.
However, as foreign keys are the most common type of IND, we provide options for filtering out INDs where the referenced column set is not a key.

Our tool currently supports two profiling operations for INDs/FKs:

- **Foreign Key Discovery**: Given a set of tables, identify INDs / FKs between them.
- **Foreign Key Analysis**: Compute metrics for a given IND, and extract data examples.

As for key discovery, foreign key discovery will return many accidental INDs, so results returned will likely require further evaluation by a domain expert.
Foreign Key Analysis can help with this.

### Incomplete Data

Three different semantics are supported for deciding how missing values are handled for inclusion dependencies.
That is, if a row in the source table is missing a value in one of the columns participating in the IND, then satisfaction of the IND for that row depends on the semantic.

- **Simple Semantics**: The IND is satisfied.
- **Full Semantics**: The IND is violated.
- **Partial Semantics**: The IND is satisfied if there exists a row in the target table that matches the row in the source table for every column pair in the IND where the source row has a value.



### Dirty Data
