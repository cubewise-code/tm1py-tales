---
layout: post
title:  'Dynamic TM1 queries with MDXpy'
date:   2022-05-08 20:20:00
categories: mdx
---

# Dynamic TM1 queries with MDXpy

In most TM1py scripts it is necessary to query cube data from TM1. Usually, TM1py scripts are also
parametrized. For instance, a timeseries forecasting script must work for any given `region` and `product` element. 

So TM1py scripts must be capable of querying TM1 data dynamically based on passed parameters.

The best way to accomplish this is to use MDX with MDXpy. Before looking into the merits and advantages of the
suggested approach, let's first look at the alternatives to MDXpy.

Consider the following task:

- the script receives 3 parameters: `year`, `region` and `product`
- based on the passed parameters the script retrieves data from the `Sales` cube
- If a consolidated element is passed, the script must retrieve data from the leaves underneath (e.g. retrieve `Denmark`
  , `Sweden`, `Norway`, `Finland` for `Scandinavia`)
- for all other dimensions in the cube all leaf elements are retrieved 

![The Sales Cube](https://github.com/cubewise-code/tm1py-tales/blob/master/_images/2022-05-08-sales-cube.png?raw=true)  



### Alternative 1. Use existing cube views

TM1py provides easy to use functions to read data based on existing cube views such as `execute_view_dataframe`.
This approach keeps python code easy and requires zero knowledge about MDX

However, there are two problems about this approach:

- it is not dynamic
- it creates an unnecessary dependence on a cube view in TM1

###  Alternative 2. Use plain MDX

TM1py also provides easy to use functions to read data based on MDX such as `execute_mdx_dataframe`.

You can use plain python functions to create an MDX query with string concatenation. This approach provides you with
sufficient flexibility.

``` python
import sys

from TM1py import TM1Service

year, region, product = sys.argv[1:]

with TM1Service(base_url='https://localhost:12354', user='admin', password='apple') as tm1:
    mdx = 'SELECT'
    mdx += '{[TIME].[' + year + '].CHILDREN} ON 0,'
    mdx += 'NON EMPTY '
    mdx += '{TM1FILTERBYLEVEL({DESCENDANTS([REGION].[' + region + ' ])},0)}*'
    mdx += '{TM1FILTERBYLEVEL({DESCENDANTS([PRODUCT].[' + product + ' ])},0)}*'

    for dimension_name in tm1.cubes.get_dimension_names('Sales'):
        if dimension_name in ['Product', 'Region', 'Time']:
            continue

        mdx += '{TM1FILTERBYLEVEL({TM1SUBSETALL([' + dimension_name + '])},0)}*'

    mdx = mdx.rstrip('*')
    mdx += ' ON 1 '
    mdx += 'FROM [SALES]'

    df = tm1.cells.execute_mdx_dataframe(mdx=mdx)
```

However, the approach has obvious drawbacks:

- it requires advanced MDX knowledge
- it is error-prone and difficult to create MDX through plain string concatenations _(just imagine getting all the `{}`
  , `[]` and `()` right)_



### MDXpy solves this

MDXpy combines the advantages of MDX without the drawbacks, such as the complex string concatenations to create valid MDX.


Here is the solution with MDXpy:

``` python
import sys

from TM1py import TM1Service
from mdxpy import MdxBuilder, MdxHierarchySet, Member

year, region, product = sys.argv[1:]

with TM1Service(base_url='https://localhost:12354', user='admin', password='apple') as tm1:
    query = MdxBuilder.from_cube('Sales')
    query.rows_non_empty()

    months_selection = MdxHierarchySet.children(Member.of('Time', year))
    query.add_hierarchy_set_to_column_axis(months_selection)

    region_selection = MdxHierarchySet.descendants(Member.of('Region', region))
    region_selection = region_selection.filter_by_level(0)
    query.add_hierarchy_set_to_row_axis(region_selection)

    product_selection = MdxHierarchySet.descendants(Member.of('Product', product))
    product_selection = product_selection.filter_by_level(0)
    query.add_hierarchy_set_to_row_axis(product_selection)

    for dimension_name in tm1.cubes.get_dimension_names('Sales'):
        if dimension_name in ['Product', 'Region', 'Time']:
            continue

        query.add_hierarchy_set_to_row_axis(MdxHierarchySet.all_leaves(
            dimension=dimension_name,
            hierarchy=dimension_name))

    df = tm1.cells.execute_mdx_dataframe(mdx=query.to_mdx())

```

_____

Written by [![GitHub](https://i.stack.imgur.com/tskMh.png) Marius Wirtz](https://github.com/mariuswirtz)










