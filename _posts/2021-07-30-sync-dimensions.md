---
layout: post
title:  "Synchronizing hierarchies with TM1py"
date:   2021-07-30 10:30:00
categories: introduction
---

The Problem
=====


In large organizations that employ IBM Planning Analytics (TM1) for years, you often find multiple dedicated smaller TM1 instances *(= models)* instead of one monolithic TM1 model used by everyone in the organization.

Having multiple TM1 instances is not a problem in itself. In fact it helps in many ways:
 - scaling models horizontally over multiple machines ensures more resources (CPU, RAM) per instance and better performance for end-users
 - reduces restart times and guarantees local impact of a (god forbid) server crash
 - it helps with separation of concerns and laying our responsibilities from a development and admin perspective


However, laying out your TM1 environment "horizontally" comes with unique challenges.
- You need to guarantee that certain shared hierarchies are in sync. For instance, there must only be one CoA rollup in the organization.
- Especially shared dimensions that are edited by power users within TM1 can be challenging 
 
 <img src="https://images.squarespace-cdn.com/content/v1/5268c662e4b0269256614e9a/1627476400785-SUAXHCO8WVZ3ZQ1RWQF0/tm1py-Picture2.png?format=500w" style="width: 70%; height: 70%;text-align: center"/>

 
The Solution
=====

TM1py makes it easy to spot differences and synchronize dimensions between TM1 instances.
Here is a sample that determines if a dimension is in sync in 6 lines of code.

```python
from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1_source:
    with TM1Service(address="", port=12297, ssl=True, user="admin", password="apple") as tm1_target:
        dimension_source = tm1_source.dimensions.get(dimension_name="d2")
        dimension_target = tm1_target.dimensions.get(dimension_name="d2")
        print(dimension_source == dimension_target)
```

Now if we know that a dimension is out of sync, we can use TM1py to re-synchronize in 5 lines.

```python
from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1_source:
    with TM1Service(address="", port=12297, ssl=True, user="admin", password="apple") as tm1_target:
        dimension = tm1_source.dimensions.get(dimension_name="d2")
        tm1_target.dimensions.update_or_create(dimension)
```

The above code considers the dimension with its hierarchies, elements and edges (= parent child relationships) and attributes.
However it does __not__ consider the element attribute values and subsets.

Those need to be transferred separately:

```python
from mdxpy import MdxBuilder, MdxHierarchySet
from TM1py import TM1Service, Process

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1_source:
    with TM1Service(address="", port=12297, ssl=True, user="admin", password="apple") as tm1_target:
        dimension_name = "d1"
        attribute_cube = "}ElementAttributes_" + dimension_name

        query = MdxBuilder.from_cube(attribute_cube)
        query = query.non_empty(axis=0)
        query = query.add_hierarchy_set_to_column_axis(MdxHierarchySet.tm1_subset_all(dimension_name))
        query = query.add_hierarchy_set_to_column_axis(MdxHierarchySet.tm1_subset_all(attribute_cube))
        mdx = query.to_mdx()

        attribute_cells = tm1_source.cells.execute_mdx(
            mdx=mdx,
            element_unique_names=False,
            skip_cell_properties=True,
            skip_rule_derived_cells=True)

        clear_process = Process(f"CubeClearData('{attribute_cube}');")
        success, _, _ = tm1_target.processes.execute_process_with_return(clear_process)
        if not success:
            raise RuntimeError(f"Failed to clear attribute cube for dimension: '{dimension_name}'")

        tm1_target.cells.write(
            cube_name=attribute_cube,
            cellset_as_dict=attribute_cells,
            use_ti=True,
            deactivate_transaction_log=True,
            reactivate_transaction_log=True)
```

The script skips rule derived values, as it assumes that the rules in the `}ElementAttributes` cubes are in sync.

Further Thoughts
=====

_____

### `(1 to n)` vs `(n to n)`
There are two ways to implement synchronizations in a multi instance environment. 
- `(1 to n)`: All manual changes happen in the master instances and will be broadcasted to the satellite instances 
- `(n to n)`: A change can happen in any instance at any time and will be broadcasted to all other instances

TM1py can support both ways but in the (n to n) setup you will need to write smarter code to monitor all instances and to cater for potential conflicts.

### Optimizations 

When dealing with very large dimensions _(> 100k elements)_, you may want to optimize the script.
There are two easy tricks to boost the performance of your script
- Multi thread attribute updates. You will notice that typically a large part of the time is consumed to write attribute values to TM1.
Imagine a 500k element dimension with 20 attributes produces 10M string cell updates. Check out the [write_async](https://cubewise-code.github.io/tm1py-tales/2021/write-with-tm1py.html) function.
- Avoid full dimension retrieval and comparision. For very large dimension it is highly expensive to retrieve the full dimension from both instances and compare them in python.
Instead you can use the `get_hierarchy_summary` function to retrieve a summary _(e.g. `{'Elements': 5, 'Edges': 4, 'ElementAttributes': 4, 'Members': 5, 'Levels': 2}`)_ for each hierarchy instead and compare the summaries.

```python
from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1_source:
    with TM1Service(address="", port=12297, ssl=True, user="admin", password="apple") as tm1_target:
        dimension_name = "d1"

        source_hierarchy_summary = tm1_source.hierarchies.get_hierarchy_summary(
            dimension_name=dimension_name,
            hierarchy_name=dimension_name)

        target_hierarchy_summary = tm1_target.hierarchies.get_hierarchy_summary(
            dimension_name=dimension_name,
            hierarchy_name=dimension_name)

        print(source_hierarchy_summary == target_hierarchy_summary)

```

### Deploying TM1py

For TM1py to react to changes swiftly, it makes sense have it run at all times as a service.
We will discuss how to deploy TM1py as a service in a separate post.

Needless to say, there are also challenges w.r.t shared data and security. 
We will discuss how TM1py can help with those in a separate post.


_____

Written by [![GitHub](https://i.stack.imgur.com/tskMh.png) Marius Wirtz](https://github.com/mariuswirtz)




