---
layout: post
title:  "Writing to TM1 with TM1py"
date:   2021-07-27 16:30:00
categories: writeback
---

Writing to TM1
=====

In almost every TM1py project we need to writeback data from python into a TM1 cube.

TM1py offers different functions to write to cubes. 
There is not one ideal function to writeback, but every function comes with its pros and cons.

Below each function is discussed with it's unique characteristics and arguments.


write
=====
the write function is the default `write` function in TM1py.
It offers the most features and flexibility. 

It takes 2 compulsory arguments 
- `cube_name`: The name of the cube
- `cells_as_dict`: A dictionary of the cell updates. E.g.: `{('e1', 'e2'): 12, ('e4', 'e1'): 8}`
 
And 7 optional arguments
- `dimensions`: Dimension names in their natural order. Speeds up the execution.
- `increment`: `True` if cells should be incremented. `False` if cells should simply be written.
- `deactivate_transaction_log`: `True` to deactivate the transactionlog before writing 
- `reactivate_transaction_log`: `True` to reactivate the transactionlog after writing
- `sandbox_name`: The name of a sandbox or `None` if writing to 'base'
- `use_ti`: `True` if unbound TI processes should be used instead for writing through REST calls. It speeds up the execution significantly, but requires admin permissions. With `use_ti` as True data is written in a mode that allows minor errors. So if one cell update is invalid, all the other ones will still succeed and commit, while in the REST based write mode, all write operations abort if one cell-update is invalid.
- `use_changeset`: `True` if changeset should be used. Changesets allow undo operations on batches.


Samples:

``` python
from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1:
    tm1.cells.write(
        cube_name='c1',
        cellset_as_dict={('e1', 'e2'): 11.5, ('e2', 'e3'): 12.3, ('e4', 'e1'): 7.5})
```

``` python
from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1:
    tm1.cells.write(
        cube_name='c1',
        cellset_as_dict={('e1', 'e2'): 11.5, ('e2', 'e3'): 12.3, ('e4', 'e1'): 7.5},
        dimensions=['d1', 'd2'],
        increment=False,
        deactivate_transaction_log=True,
        reactivate_transaction_log=True,
        sandbox_name=None,
        use_ti=True,
        use_changeset=False)
```



write_async
===== 


write_dataframe
=====


write_dataframe_async
=====


write_values
=====


write_value
=====


write_through_unbound_process
=====

