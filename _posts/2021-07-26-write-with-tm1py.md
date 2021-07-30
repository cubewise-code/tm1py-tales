---
layout: post
title:  "Writing to TM1 with TM1py"
date:   2021-07-27 16:30:00
categories: writeback
---

# Writing to TM1

In almost every TM1py project it is necessary to write back data from python into a TM1 cube.

TM1py offers different functions to write to cubes. 
There is not one ideal function to write back, but every function comes with its pros and cons.

Below the most important functions are discussed with their unique characteristics and arguments.

<img src="https://images.squarespace-cdn.com/content/v1/5268c662e4b0269256614e9a/1626686941888-TS0IJZAW9V8Y90NLRPX7/Picture5.png?format=500w" style="width: 70%; height: 70%;text-align: center"/>

All samples are based on a cube called `c1`, with two dimensions: `d1`, `d2` and simple elements: `e1`, `e2`, `e3`, `e4`.



### `write`
_________________

The `write` function is the default function for writeback in TM1py.
It offers the most features and flexibility. 

It takes 2 compulsory arguments:
- `cube_name`: The name of the cube
- `cells_as_dict`: A dictionary with the cell updates. E.g.: `{('e1', 'e2'): 12, ('e4', 'e1'): 8}`
 
And 7 optional arguments:
- `dimensions`: Dimension names in their natural order. Speeds up the execution.
- `increment`: `True` if cells should be incremented. `False` if cells should simply be written.
- `deactivate_transaction_log`: `True` to deactivate the transaction log before writing 
- `reactivate_transaction_log`: `True` to reactivate the transaction log after writing
- `sandbox_name`: The name of a sandbox or `None` if writing to 'base'
- `use_ti`: `True` if unbound TI processes should be used instead for writing through REST calls. It speeds up the execution significantly, but requires admin permissions. With `use_ti` as `True` data is written in a mode that allows minor errors. So if one cell update is invalid, all the other ones will still succeed and commit, while in the REST based write mode, all write operations abort if one cell-update is invalid.
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



### `write_async`
_________________

The `write_async` function is a wrapper function around the default `write` function.
It manages the parallel execution of the write function through two additional arguments: `slice_size` and `max_workers`. 

The TM1 write performance can be increased through parallelization, assuming that the TM1 Server has sufficient CPU available.

TM1py does not suggest default values for `slice_size` and `max_workers` as the optimal value for each variable depends on your use case and server architecture.
The ideal number must be determined iteratively through testing on your environment.   

This function makes use of the `use_ti=True` option by default, to guarantee maximum performance.

It takes 4 compulsory arguments:
- `cube_name`: The name of the cube
- `cells`: A dictionary with the cell updates. E.g.: `{('e1', 'e2'): 12, ('e4', 'e1'): 8}`
- `slice_size`: size of each chunk
- `max_workers`: number of threads / workers
 
And 5 optional arguments:
- `dimensions`: Dimension names in their natural order. Speeds up the execution.
- `increment`: `True` if cells should be incremented. `False` if cells should simply be written.
- `deactivate_transaction_log`: `True` to deactivate the transaction log before writing 
- `reactivate_transaction_log`: `True` to reactivate the transaction log after writing
- `sandbox_name`: The name of a sandbox or `None` if writing to 'base'


Samples:


``` python
from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1:
    tm1.cells.write_async(
        cube_name='c1',
        cells={('e1', 'e2'): 11.5, ('e2', 'e3'): 12.3, ('e4', 'e1'): 7.5},
        slice_size=1,
        max_workers=3)
```

``` python
from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1:
    tm1.cells.write_async(
        cube_name='c1',
        cells={('e1', 'e2'): 11.5, ('e2', 'e3'): 12.3, ('e4', 'e1'): 7.5},
        slice_size=1,
        max_workers=3,
        dimensions=['d1', 'd2'],
        increment=True,
        deactivate_transaction_log=True,
        reactivate_transaction_log=True,
        sandbox_name=None)
```

### `write_dataframe`
_________________

The `write_dataframe` function writes a [pandas dataframe](https://www.google.de/search?q=what+is+a+pandas+dataframe%3F) to a cube.
The column order in the data frame must match the dimensions in the target cube with an additional column for the values.

It takes 2 compulsory arguments:
- `cube_name`: The name of the cube
- `data`: A pandas data frame with the cell updates
 
And 7 optional arguments:
- `dimensions`: Dimension names in their natural order. Speeds up the execution.
- `increment`: `True` if cells should be incremented. `False` if cells should simply be written.
- `deactivate_transaction_log`: `True` to deactivate the transactionlog before writing 
- `reactivate_transaction_log`: `True` to reactivate the transactionlog after writing
- `sandbox_name`: The name of a sandbox or `None` if writing to 'base'
- `use_ti`: `True` if unbound TI processes should be used instead for writing through REST calls. It speeds up the execution significantly, but requires admin permissions. With `use_ti` as True data is written in a mode that allows minor errors. So if one cell update is invalid, all the other ones will still succeed and commit, while in the REST based write mode, all write operations abort if one cell-update is invalid.
- `use_changeset`: `True` if changeset should be used. Changesets allow undo operations on batches.

Samples:

```python
import pandas as pd

from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1:
    data = pd.DataFrame({
        'd1': ['e1', 'e2', 'e4'],
        'd2': ['e2', 'e3', 'e1'],
        'value': [11.5, 12.3, 7.5]
    })

    tm1.cells.write_dataframe(
        cube_name='c1',
        data=data)
```

```python
import pandas as pd

from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1:
    data = pd.DataFrame({
        'd1': ['e1', 'e2', 'e4'],
        'd2': ['e2', 'e3', 'e1'],
        'value': [11.5, 12.3, 7.5]
    })

    tm1.cells.write_dataframe(
        cube_name='c1',
        data=data,
        dimensions=['d1', 'd2'],
        increment=False,
        deactivate_transaction_log=True,
        reactivate_transaction_log=True,
        sandbox_name=None,
        use_ti=True,
        use_changeset=True)
```

### `write_dataframe_async`
_________________

The `write_dataframe_async` function is a wrapper function around the default `write_dataframe` function.
The column order in the data frame must match the dimensions in the target cube with an additional column for the values.

It manages the parallel execution of the write function through two additional arguments: `slice_size` and `max_workers`.

The TM1 write performance can be increased through parallelization, assuming that the TM1 Server has sufficient CPU available.

TM1py does not suggest default values for `slice_size` and `max_workers` as the optimal value for each variable depends on your use case and server architecture.
The ideal number must be determined iteratively through testing on your environment.   

This function makes use of the `use_ti=True` option by default, to guarantee maximum performance.

It takes 4 compulsory arguments:
- `cube_name`: The name of the cube
- `data`: A pandas data frame with the cell updates
- `slice_size_of_dataframe`: size of each chunk
- `max_workers`: number of threads / workers
 
And 5 optional arguments:
- `dimensions`: Dimension names in their natural order. Speeds up the execution.
- `increment`: `True` if cells should be incremented. `False` if cells should simply be written.
- `deactivate_transaction_log`: `True` to deactivate the transactionlog before writing 
- `reactivate_transaction_log`: `True` to reactivate the transactionlog after writing
- `sandbox_name`: The name of a sandbox or `None` if writing to 'base'

Samples:

```python
import pandas as pd

from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1:
    data = pd.DataFrame({
        'd1': ['e1', 'e2', 'e4'],
        'd2': ['e2', 'e3', 'e1'],
        'value': [11.5, 12.3, 7.5]
    })

    tm1.cells.write_dataframe_async(
        cube_name='c1',
        data=data,
        slice_size_of_dataframe=1,
        max_workers=3)
```

```python
import pandas as pd

from TM1py import TM1Service

with TM1Service(address="", port=12354, ssl=True, user="admin", password="apple") as tm1:
    data = pd.DataFrame({
        'd1': ['e1', 'e2', 'e4'],
        'd2': ['e2', 'e3', 'e1'],
        'value': [11.5, 12.3, 7.5]
    })

    tm1.cells.write_dataframe_async(
        cube_name='c1',
        data=data,
        slice_size_of_dataframe=1,
        max_workers=3,
        dimensions=['d1', 'd2'],
        increment=False,
        deactivate_transaction_log=True,
        reactivate_transaction_log=True,
        sandbox_name=None)
```

_____
Written by [![GitHub](https://i.stack.imgur.com/tskMh.png) Marius Wirtz](https://github.com/mariuswirtz)