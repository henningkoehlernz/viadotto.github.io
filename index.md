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

Inputs are currently limited to relational data, but a variety of input formats (relational databases, CSV files, ...) are supported.

### Incomplete Data

When profiling data sets with missing values, proper handling of missing (null) values is critical.
E.g. treating null values as any other domain values could cause valid foreign key constraints to be missed.
The API supports multiple semantics in dealing with them, which will be described later on.

### Dirty Data

## Sampling

## Mining keys

## Mining foreign keys
