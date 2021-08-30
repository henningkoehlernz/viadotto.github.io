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
- **Key Analysis**: Compute uniqueness metrics for a given column set.

### Incomplete Data

Currently the only supported semantic for dealing with null values is *possible semantic*, corresponding to UNIQUE constraints in SQL.
Here a column set **K** is a key if no two rows have the same values on **K**, unless one of these values is null.
Thus the column set { Name, DoB } is a key for the table below, but { Name, Salary } is not.

| Name  |   DoB    | Salary |
| ----- | -------- | ------ |
| John  | 01/03/85 | 80k    |
| John  |          | 80k    |
| Susan | 01/03/85 |        |
| Dave  | 06/06/91 | 75k    |

### Dirty Data

## Mining foreign keys

### Incomplete Data

### Dirty Data
