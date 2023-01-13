---
layout: post
title:  "How to partition MDX queries with TM1py"
date:   2023-01-13 18:30:00
categories: introduction
---

### Querying TM1 data with TM1py is easy, right?

Using TM1py, it is quite easy to read data from TM1 into Python using MDX queries:

``` python
from TM1py import TM1Service

mdx = """
SELECT 
{Tm1SubsetAll([d1])} ON 0,
{TM1SubsetAll([d2])} ON 1
FROM [c1]
"""

with TM1Service(base_url="https://localhost:12297", user="admin", password="") as tm1:
    df = tm1.cells.execute_mdx_dataframe(mdx=mdx)

```

However, When dealing with large or very large data volumes, it can make sense to divide the MDX query into small partitions.

While this introduces additional complexity into the python code, there are three good reasons why people would choose to implement partitioning:

- avoid excessive RAM consumption in Python
- use partitioning for parallel processing
- Avoid TM1 `SystemOutOfMemory` error, that is raised when the view size exceeds the `MaximumViewSize` value configured in tm1s.cfg 

Let's look at these reasons in more depth before we look into the solution.

> 1. Avoid excessive RAM consumption in Python

Python uses quite a lot of memory compared to the memory efficient TM1 engine. 
Depending on your data you can assume that python requires 0.5 GB to retrieve a million cells out of TM1.
That is purely for storage and retrieval. 
It is easy to imagine how a large query (e.g. zero suppression was forgotten) can cause a lack of a memory on the machine.

Needless to say, this scenario is discomforting when Py runs on same machine as the TM1 server.

So in order to avoid this disruption it can make sense to query no more than n cells at a time. 
If the source query has more than n cells it must be processed in partitions!

Also bear in mind that sometimes good code is broken by invalid user input.
Huge queries can be created by good code with invalid user input (e.g. a user triggering a forecast for a combination of all products by all regions)
It's never a bad idea to avoid this scenario.

Of course partitioning of the source query is not the only solution to avoid a memory overflow.
Alternatively You can configure a `MaximumViewSize` or abort the script if a query has a size greater than n.

``` python
from TM1py import TM1Service

mdx = """
SELECT 
{Tm1SubsetAll([d1])} ON 0,
{TM1SubsetAll([d2])} ON 1
FROM [c1]
"""

with TM1Service(base_url="https://localhost:12297", user="admin", password="") as tm1:
    cellset_id = tm1.cells.create_cellset(mdx)
    size = tm1.cells.extract_cellset_cellcount(cellset_id=cellset_id, delete_cellset=False)

    if size > 5_000_000:
        raise ValueError(f"Source query greater than 5M cells: '{mdx}'")

    df = tm1.cells.extract_cellset_dataframe(cellset_id=cellset_id)

```

> 2. Use partitioning for parallel processing

Depending on the problem at hand and the available resource it can make sense to make use of parallel processing.

If your large problem can be broken down into self-contained smaller problems that can be addressed individually, 
you can benefit from parallel processing. 

Here are some examples of problems that can be broken down and processed in parallel:

- Forecast demand for many product/region pairs
- Transfer twelve months of data from a cube in TM1 on-premise to TM1 on-cloud
- Calculate IRR or NPV on a large set of projects

> 3.  Avoid TM1's `SystemOutOfMemory` error

A casual TM1 user may easily request a very large dataset unintentionally using the cube viewer. 
For instance if they forget to suppress empty cells in their selection!

This can lead to frustration on the user side _"TM1 is slow", "My cube viewer is frozen"_ and can cause unnecessary work on the TM1 server side.

For that reason it is generally a good practice to maintain a `MaximumViewSize` in the `tm1s.cfg`.
To protect the system from users and to protect users from themselves.

However, for TM1py or other functional applications using REST, this threshold is often a stepping stone.
A script that worked yesterday may fail today because a source query is now exceeding the memory threshold in TM1.

bear in mind that in theory even small grid of highly aggregated cells can trigger the `SystemOutOfMemory`, because the underlying calculation tree may require lotsaram to evaluate.

Using partitioning we can make sure that our queries are using no more memory than is allowed.



### How to do it?

There are at least three convenient ways to divide your queries into sub queries.

> Option 1: partition by one dimension

This option is generally a good choice if you can assume that data is roughly evenly allocated along the elements in a dimension.
It tends to be much faster than Option 3. 

The obvious scenario for this case to split a full year query into 12 sub queries based on the months.

You can easily implement it like this:
- define a base MDX query with a place holder for the month 
- loop through 12 months and substitute the placeholder to build the sub query
- execute the sub query

```python
from TM1py import TM1Service

months = ["202001", "202002", "202003", "202004", "202005", "202006",
          "202007", "202008", "202009", "202010", "202011", "202012"]

base_mdx = """
SELECT
{} ON COLUMNS,
NON EMPTY 
TM1FilterByLevel(Tm1SubsetAll([Business Unit]),0) *
TM1FilterByLevel(Tm1SubsetAll([Customer]),0) *
TM1FilterByLevel(Tm1SubsetAll([Executive]),0) *
TM1FilterByLevel(Tm1SubsetAll([Industry]),0) *
TM1FilterByLevel(Tm1SubsetAll([Product]),0) *
TM1FilterByLevel(Tm1SubsetAll([State]),0) ON ROWS
FROM [Sales]
WHERE ([Version].[Actual], [SalesMeasure].[Revenue])
"""

with TM1Service(base_url="https://localhost:12297", user="admin", password="apple") as tm1:
    for month in months:
        mdx = base_mdx.format("{[time].[" + month + "]}")

        df = tm1.cells.execute_mdx_dataframe(mdx)

```

Using logical grouping _(in contrast to the arbitrary groups that are formed in Option 2 and Option 3)_ is convenient for troubleshooting. 
If your script fails for one sub query it is much easier to troubleshoot knowing that it was the march data than knowing it failed for an arbitrary selection of cells.

> Option 2: Partition with MDX

We can use the MDX [SUBSET](https://learn.microsoft.com/en-us/sql/mdx/subset-mdx?view=sql-server-ver16) function to slice an MDX set.

For instance we could divide the source set: `[Month].[01]:[Month].[12]` in
6 chunks of 2 elements each:
  - `SUBSET([Month].[01]:[Month].[12], 0, 2)`
  - `SUBSET([Month].[01]:[Month].[12], 2, 4)`
  - `...`
  - `SUBSET([Month].[01]:[Month].[12], 10, 12)`

Or we could divide the source set in 12 chunks of 1 element each _(as in the example below)_


This allows to build dynamic sub queries based off an original base query.

```python
from TM1py import TM1Service

base_mdx = """
SELECT
{} ON COLUMNS,
NON EMPTY 
TM1FilterByLevel(Tm1SubsetAll([Business Unit]),0) *
TM1FilterByLevel(Tm1SubsetAll([Customer]),0) *
TM1FilterByLevel(Tm1SubsetAll([Executive]),0) *
TM1FilterByLevel(Tm1SubsetAll([Industry]),0) *
TM1FilterByLevel(Tm1SubsetAll([Product]),0) *
TM1FilterByLevel(Tm1SubsetAll([State]),0) ON ROWS
FROM [Sales]
WHERE ([Version].[Actual], [SalesMeasure].[Revenue])
"""

with TM1Service(base_url="https://localhost:12297", user="admin", password="apple") as tm1:
    processed_months = 0

    while processed_months < 12:
        time_mdx = f"SUBSET([Time].[202001]:[Time].[202012],{processed_months},1)"
        mdx = base_mdx.format(time_mdx)

        df = tm1.cells.execute_mdx_dataframe(mdx)
        processed_months += 1
```

> Option 3: partition with top & skip

TM1py offers `top` and `skip` arguments that can be used when reading data from a TM1 cellset.

So once you know the overall size of the cellset, you can break it into even chunks of n cells and retrieve and process each chunk separately:

``` python
from TM1py import TM1Service

mdx = """
SELECT
[Time].[202001]:[Time].[202012] ON COLUMNS,
NON EMPTY 
TM1FilterByLevel(Tm1SubsetAll([Business Unit]),0) *
TM1FilterByLevel(Tm1SubsetAll([Customer]),0) *
TM1FilterByLevel(Tm1SubsetAll([Executive]),0) *
TM1FilterByLevel(Tm1SubsetAll([Industry]),0) *
TM1FilterByLevel(Tm1SubsetAll([Product]),0) *
TM1FilterByLevel(Tm1SubsetAll([State]),0) ON ROWS
FROM [Sales]
WHERE ([Version].[Actual], [SalesMeasure].[Revenue])
"""

with TM1Service(base_url="https://localhost:12297", user="admin", password="") as tm1:
    cellset_id = tm1.cells.create_cellset(mdx)
    size = tm1.cells.get_cellset_cells_count(mdx)
    
    chunk_size = 100_000
    processed_cells = 0

    while processed_cells < size:
        cells = tm1.cells.extract_cellset(cellset_id=cellset_id, top=chunk_size, skip=processed_cells,
                                          delete_cellset=False)

        processed_cells += len(cells)

```

Through the variable `chunk_size`, that is used as a value for `top` in the query, you can specify exactly how many cells you want to process in each iteration.

This approach is slower compared to option1 and option2. It can also not be used with parallel processing, because the cellset object in TM1 does not allow parallel access.
Option1 and Option2 can be used for parallel processing without constraints!






###