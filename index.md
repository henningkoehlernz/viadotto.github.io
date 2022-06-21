---
title: Key/FK Mining
links: <a href="#overview">overview</a> <a href="#sampling">sampling</a> <a href="#keys">keys</a> <a href="#foreign-keys">foreign keys</a> <a href="#workflows">workflows</a>
---

## Overview {#overview}

This tutorial provides a brief introduction to the Data Viadotto profiling API.
Rather than focusing on request details, it aims to provide an overview about services provided, and typical workflows.

The Data Viadotto API provides tools for discovering key and foreign key constraints by analysing relationships between data values.
This is in contrast to more basic data visualization tools, which simply visualize constraints explicitly defined.
It can thus be employed in cases where constraints are not fully known, or not known at all.
Some common use cases are:

- schema reverse engineering
- data lake traversal
- data integration

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

We note that data can be dirty in a variety of ways (e.g. inconsistent, outdated, or simply wrong).
For constraint discovery, only dirtiness that causes data inconsistency matters.

## Sampling {#sampling}

The first step in profiling a data set is sampling, where a suitable subset of rows is selected from each table.
The main purpose of this step is to speed up constraint discovery.
Additionally, basic characteristics such as number of null and distinct values for each column are computed as part of the sampling process, which are needed during profiling.
For this reason, constraint discovery will *always* operate on these samples.

For tables with few rows, all rows will be included in the sample.
For larger tables the *number* of rows sampled increases with the table size, but the *fraction* of rows sampled becomes smaller as the table grows.
Although sampling will inevitably result in some inaccuracies for the measures computed, sampling rates are chosen to ensure that these inaccuracies remain small.

It is possible to force all rows to be included in a sample -- doing so will avoid inaccuracies, but at the cost of reduced speed and increased memory usage.

## Mining keys {#keys}

Our tool currently supports two profiling operations for keys:

- **Key Discovery**: Find all minimal column sets that form a key for a given table.
- **Key Analysis**: Compute uniqueness metrics for a given column set, and extract data examples.

For key discovery, we note that just because a constraint happens to hold for a table does not guarantee that it is meaningful.
In practice, many of the key constraints discovered will be accidental, meaning they hold only by chance -- this is particularly true for tables with few rows.

As accidental keys are typically of little interest, the set of minimal keys returned is best seen as a starting point for further manual evaluation by a domain expert.
Key analysis for selected column sets can be helpful in this.

### Incomplete Data

There are different approaches for dealing with null values.
Conceptually we consider *possible worlds*, where missing values are replaced with arbitrary values.
Depending on whether we require a key constraint to be satisfied by all possible worlds or only some, we obtain different semantics.

The default is *possible semantic*, corresponding to UNIQUE constraints in SQL.
Here a column set **K** is a key if no two rows violate it, meaning they have the same values on **K**, and none of these values is missing.
Thus the column set { Name, DoB } is a key for the table below, but { Name, Salary } is not.

| Name  | DoB      | Salary | Job       |
| ----- | -------- | ------ | --------- |
| John  | 01/03/85 | 90k    | Developer |
| John  |          | 90k    | Analyst   |
| Susan | 01/03/85 |        |           |
| Dave  | 06/06/91 | 75k    | Tester    |

Note that { DoB, Salary } and { Job } are also keys for the given table, though likely accidental.

Under *certain semantic* each pair of rows must strongly disagree on at least one key column, meaning values are present and different.
For the table above, { Name, Job } is the only minimal key.

### Dirty Data

Key analysis computes the *uniqueness coefficient* of the provided column set.
This is the fraction of tuples which do not violate the key constraint (together with some other tuple).
E.g. for the table above, the uniqueness coefficient of { Name } is 0.5.

Generally, uniqueness coefficients close to 1 indicate that the column set might be a key, with a few dirty data values causing it to be violated.
For low uniqueness coefficients this is unlikely.

For incomplete data we compute the minimum and maximum uniqueness coefficient amongst all possible worlds.
E.g. for the table above, the uniqueness coefficient of { Name, DoB } varies between 0.5 and 1.
Keys under possible semantic have a maximum uniqueness coefficient of 1, while keys under certain semantic have a minimum uniqueness coefficient of 1.

*Note: Computation of minimum uniqueness coefficients has not been implemented yet.*

## Mining foreign keys {#foreign-keys}

Rather than just focusing on foreign keys, our tool searches for *inclusion dependencies* (INDs), meaning that the column set referenced does not have to be a key.
However, as foreign keys are the most common type of IND, we provide options for filtering out INDs where the referenced column set is not a key.

Our tool currently supports two profiling operations for INDs/FKs:

- **Foreign Key Discovery**: Given a set of tables, identify INDs / FKs between them.
- **Foreign Key Analysis**: Compute metrics for a given IND, and extract data examples.

As for key discovery, foreign key discovery will return many accidental INDs, so results returned will likely require further evaluation by a domain expert.
Foreign Key Analysis can help with this.

Note that foreign key discovery also returns non-maximal INDs.
This is redundant when dealing with exact INDs only.
However, when data sets are dirty, the degree of satisfaction tends to be greater for non-maximal INDs, which may make them more relevant.

### Incomplete Data

Different semantics are supported for deciding how missing values are handled for inclusion dependencies.
That is, if a row in the source table is missing a value in one of the columns participating in the IND, then satisfaction of the IND for that row depends on the semantic.

- **Simple Semantics**: The IND is satisfied.
- **Partial Semantics**: The IND is satisfied if there exists a row in the target table that matches the row in the source table for every column pair in the IND where the source row has a value.
- **Full Semantics**: The IND is violated.

FOREIGN KEY constraints in SQL use simple semantics.
Thus for schema reverse engineering tasks where FOREIGN KEY constraints were previously enforced (but are now unavailable for some reason), simple semantics can be an appropriate choice.

Partial semantics can be a sensible choice for most tasks, as it best captures the intent of an inclusion dependency (references must target existing data) and should be considered the default.

For some datasets, references will almost always be complete (no missing values).
Here full semantics is best at eliminating false positives, as it is the most restrictive.

We note that for a column containing only null values, any IND with such a column as source would technically be satisfied under simple and partial semantics.
However, we exclude such columns from the mining process, as the resulting INDs would not be useful.

Consider the following two tables, Orders and Accounts:

<table><tr><td>

<table>
<tr><th>OrderID</th><th>Customer</th><th>Company</th></tr>
<tr><td>1</td><td>Darrel</td><td>Bugs-R-Us</td></tr>
<tr><td>2</td><td>Susan</td><td></td></tr>
<tr><td>3</td><td>Edward</td><td></td></tr>
<tr><td>4</td><td>Jenny</td><td></td></tr>
<tr><td>5</td><td>Tom</td><td>EazyCode</td></tr>
</table>

</td><td>

<table>
<tr><th>Name</th><th>Company</th><th>AccountNr</th></tr>
<tr><td>Darrel</td><td>Bugs-R-Us</td><td>99-666-00</td></tr>
<tr><td>Susan</td><td></td><td>71-922-88</td></tr>
<tr><td>Edward</td><td>Bugs-R-Us</td><td>99-666-00</td></tr>
<tr><td>Darrel</td><td>EazyCode</td><td>12-345-67</td></tr>
<tr><td>Tom</td><td></td><td>57-902-46</td></tr>
</table>

</td></tr></table>

The inclusion dependency Orders[Customer,Company] &sube; Accounts[Name,Company] is satisfied for the first order under all semantics.
Orders 2 and 3 satisfy it under simple and partial semantics, but not under full semantics.
Order 4 satisfies it under simple semantics only, while order 5 violates it under all semantics.

### Dirty Data

The *inclusion coefficient* of an IND measures the degree to which a dataset satisfies it.
It is computed as the fraction of rows in the source table that satisfy the IND.
As satisfaction depends on the choice of semantics, we obtain multiple inclusion coefficients, one per semantic.
E.g. for the example above, the inclusion coefficient for Orders[Customer,Company] &sube; Accounts[Name,Company] is 0.8 under simple semantics, 0.6 under partial semantics, and 0.2 under full semantics.

During foreign key discovery and analysis, inclusion coefficients (ICs) are computed w.r.t. all three semantics.
E.g. for the table above, foreign key discovery will find

- Orders[Customer] &sube; Accounts[Name] with ICs of 0.8, 0,8 and 0.8
- Orders[Company] &sube; Accounts[Company] with ICs of 1.0, 1.0 and 0.4
- Orders[Customer,Company] &sube; Accounts[Name,Company] with ICs of 0.8, 0.6 and 0.2

### Filtering

Without restrictions, foreign key discovery is likely to return an overwhelming number of false positives -- INDs with non-zero inclusion coefficients that aren't meaningful -- as the number of potential INDs grows exponentially in the number of columns.
This can also result in timeouts or memory overflows, even for small datasets.
Columns with small integer values are particularly prone to this, as they are likely to have at least some values in common.

To avoid this, INDs must be filtered.
The most obvious way to do this is by defining thresholds for the different inclusions coefficients, so that only INDs whose inclusion coefficients meet or exceed these thresholds are returned.
Thresholds for partial and full semantics are used for pruning during the discovery process, while the threshold for simple semantics is only applied afterwards.
Thus higher thresholds for partial and full semantics will speed up the discovery process, while a higher threshold for simple semantics will not.

{% comment %}
While an inclusion dependency only requires that the set of values in the source columns forms a subset of those in the target columns, in many cases they will actually be identical.
E.g. if every order contains at least one item, then the set of OrderID values in the OrderItems will be identical to those in the Order table.
To measure how similar the sets of source and target values are, we employ the *resemblance* measure, defined as the fraction of values common to both sets:

resemblance(S,T) = (S &cap; T) / (S &cup; T)

It is possible to filter INDs by setting a threshold for resemblance.
This is particularly helpful for eliminating false positives between ID columns which had values auto-populated from 1 to n, leading to accidental INDs with high inclusion coefficients.
However, as meaningful INDs may have low resemblance (e.g. Managers referencing Persons), this must be used with care.
It can be sensible to set only low resemblance thresholds, and use resemblance mainly to guide manual evaluation.

We note that fitering by resemblance is done after mining, so the resemblance threshold has no significant impact on processing speed.
{% endcomment %}

While an inclusion dependency does not require all rows in the target table to be referenced, in practice they often are.
E.g. if every order contains at least one item, then every OrderID value will be referenced by an OrderItem.
The *coverage* measure describes the fraction of rows in the target table that are complete on the target columns and referenced by one or more rows in the source table.
Filtering INDs by coverage is particularly helpful for eliminating false positives between ID columns which had values auto-populated from 1 to n, leading to accidental INDs with high inclusion coefficients.
However, as meaningful INDs may have low coverage (e.g. Managers referencing Persons), this must be used with care.
It can be sensible to set only low coverage thresholds, and use coverage mainly to guide manual evaluation.

## Common Workflows {#workflows}

While workflows for data profiling will depend on how data sets are being processed, some sequences of steps arise frequently.
We list some of these below.
It is assumed that data sets have already been sampled, as this is required for all further processing.

### mine keys -> curate -> mine foreign keys

Mining foreign keys directly can result in long processing times and large numbers of false positives.
By specifying a set of target keys for each table (possibly empty), we can address both of these issues.
We can obtain these target keys by first mining for keys, then manually curating the keys found.
If keys are of interest, this may be necessary anyway.
Note that this will restrict inclusion dependencies found to foreign keys, which may not always be desirable.

### mine with low thresholds -> explore -> tighten thresholds

By specifying the right thresholds for a data set during mining, one can strike a good balance between meaningful constraints found, false positives elminiated, and processing time.
However, which thresholds are the "right" ones varies between data sets, and tasks performed.
It thus becomes necessary to develop some intuition which thresholds work for filtering out unwanted constraints.

We do this by first mining with low thresholds, to ensure meaningful constraints aren't missed.
Once a better understanding has been obtained by examining the constraints discovered, thresholds can be tightened.
Note that tightening of thresholds can be done by filtering locally, i.e., the mining algorithm does not need to be invoked again.

### mine with high thresholds -> loosen thresholds -> explore -> tighten thresholds

This extention of the previous workflow becomes necessary if mining with low thresholds turns out to be too expensive, resulting in timeouts or memory overflows.
Here we start with high thresholds to ensure we get some results quickly, then loosen them gradually, until a good balance between capturing meaningful constraints and processing time is reached.
Note that this will require the mining process to be invoked again.
Afterwards we proceed as before, exploring constraints discovered and tightening thresholds to eliminate false positives.
