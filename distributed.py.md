Install
=======

The best way to get distributed is to point `pip` to the github repository

    pip install https://github.com/blaze/distributed.git --upgrade

Setup
=====

Set up a scheduler on one node

    $ dscheduler     # Use the below address to refer to the cluster
    Start scheduler at 127.0.0.1:8786

Set up workers on other processes or computers that point to this address

For simplicity we used localhost here (`127.0.0.1`) but we could have used
other machines and real addresses if desired.

`distributed` communicates over TCP sockets.

    $ dworker 127.0.0.1:8786
    Start worker at                127.0.0.1:46990
    Registered with scheduler at:  127.0.0.1:8786

    $ dworker 127.0.0.1:8786
    Start worker at                127.0.0.1:38618
    Registered with scheduler at:  127.0.0.1:8786


Executor
=======

Connect an executor to the scheduler node of the cluster

    from distributed import Executor
    e = Executor('127.0.0.1:8786')


Submit and Gather
=================

Submit a function for remote execution and receive a future.

The future doesn't hold the actual result.  It stores a location on a remote
worker where the actual result currently lives.

    def inc(x):
        return x + 1

    >>> future = e.submit(inc, 1)  # calls inc(1) on a worker
    >>> future
    <Future: status: finished, key: inc-79963f7e613e5e55838d2232920baed2>

Remote data stays remote until we gather it explicitly.  Collect the remote
result from the remote worker to the local process with the calling the
`e.gather()` method.  This expects a list of futures and returns a list of
concrete results.

    >>> e.gather([future])  # collect result from remote worker
    [2]


map and gather
==============

Apply the same function to many inputs with `map`

    >>> futures = e.map(inc, range(5))  # calls inc(0), inc(1), inc(2), ...
    >>> futures
    [<Future: status: finished, key: inc-dd870045da2e4a018bd1f00a85a3fca1>,
     <Future: status: finished, key: inc-79963f7e613e5e55838d2232920baed2>,
     <Future: status: finished, key: inc-4f805cf88530097292a3bbfd148172f5>,
     <Future: status: finished, key: inc-8d6eb486f5d44cf3484056b8066c0185>,
     <Future: status: finished, key: inc-8efcc563abc786b74d7003c06b8ae7b7>]


Collect all of the results with a single call to the `gather()` method on the
`Executor`

    >>> e.gather(futures)  # collect remote data to local process
    [1, 2, 3, 4, 5]


Submit and map on futures
=========================

Call `submit` and `map` functions directly onto futures.

We don't need to collect the intermediate result, `future`, to the local
process.  This avoids sending large remote results across the network.  We move
the function to the data rather than move the data to the function.

    def inc(x):
        return x + 1

    def double(x):
        return 2 * x

    future1 = e.submit(inc, 1)
    future2 = e.submit(double, future1)  # use submit directly on future

Avoid moving data with `e.gather` when possible

    e.gather([future1])  # BAD: avoid unnecessary gather calls

We move the function to the worker node that already has the data for the
input.  This minimizes communication within the network.


Submit and map with many inputs
===============================

Submit and map can consume multiple inputs and keyword arguments

    >>> e.submit(function, *args, **kwargs)
    >>> e.map(function, *iterables, **kwargs)

This includes consuming multiple futures

If the remote inputs live on different worker nodes then we move all of the
necessary data to a single node where we run the function.  Generally we run
choose the node that would require the least communication of bytes.

    def add(x, y):
        return x + y

    >>> futures = e.map(add, range(10), range(10))

    >>> total = e.submit(sum, futures)   # call sum on a list of futures
    >>> e.gather([total])
    [90]


Scatter
=======

Send local data to the network with the `scatter` function.

    >>> e.scatter([1, 2, 3])
    [<Future: status: finished, key: 526d1868-a5a1-11e5-901d-60672020cfac-0>,
     <Future: status: finished, key: 526d1868-a5a1-11e5-901d-60672020cfac-1>,
     <Future: status: finished, key: 526d1868-a5a1-11e5-901d-60672020cfac-2>]

This is common when we have data on our local disk that we want to load locally
and then upload to the network

    >>> from glob import glob
    >>> filenames = glob('2015-*-*.csv')

    >>> futures = []
    >>> for fn in filenames:
    ...     df = pd.read_csv(fn, parse_dates=['date'])
    ...     futures.extend(e.scatter([df]))

But its best to use the workers themselves to load the data if the data is
globally available (such as on a network file system or S3).

    >>> filename_futures = e.scatter(filenames)
    >>> futures = e.map(pd.read_csv, filename_futures, parse_dates=['date'])


Exceptions
==========

We make mistakes; functions err.

    def div(x, y):
        return x / y

    >>> future = e.submit(div, 1, 0)  # divides by zero
    >>> future
    <Future: status: error, key: div-2149e07b88c7cda532abb6304548ffec>

Check the status of a future with the `.status` field

Excplicitly collect the exception and traceback

    >>> future.status
    'error'

    >>> future.exception()  # collect exception from remote worker
    ZeroDivisionError('division by zero')

    >>> future.traceback()  # collect traceback from remote worker

Futures that depend on other futures also fail

    >>> future2 = e.submit(inc, future)  # computations on failed futures fail
    >>> future2
    <Future: status: error, key: inc-6e3ab83dd1f6c214c21ce7df8353306d>

Gathering failed futures raises their exceptions

    >>> e.gather([future])  # gathering bad futures raise exceptions
    ---------------------------------------------------------------------------
    ZeroDivisionError                         Traceback (most recent call last)
    ...
    ZeroDivisionError: division by zero


Restart
=======

Remote computing gets messy.  Restart the network with the `restart` method.

    >>> e.restart()
    distributed.scheduler - CRITICAL - Lost all workers
    distributed.scheduler - INFO - Worker failed from closed stream: ('127.0.0.1', 38618)
    distributed.scheduler - INFO - Worker failed from closed stream: ('127.0.0.1', 32955)
    distributed.executor - INFO - Receive restart signal from scheduler

This cancels all running computations, destroys all data, and sets the status
of all previous futures to `cancelled`.

    >>> future
    <Future: status: cancelled, key: div-2149e07b88c7cda532abb6304548ffec>


Wait on futures
===============

### wait

Submit and map calls return immediately, even if the remote computations take a
while.

    >>> from time import sleep
    >>> %time a, b, c = e.map(sleep, [1, 2, 3])  # returns immediately
    Wall time: 829 µs

Explicitly block on these futures with the wait command

    >>> from distributed import wait
    >>> %time wait([a, b, c])
    Wall time: 3.00 s

### progress

Block on all of the futures (like `wait`) but get visual feedback with
`progress`

In the console `progress` provides a text progressbar as shown here.  In the
notebook it provides a more enticing widget that returns asynchronously.

    >>> from distributed import progress
    >>> progress([a, b, c])
    [########################################] | 100% Completed |  3.0s

### as_completed

We can also consume futures as they complete.  The `as_completed` function
returns an iterator of futures.

The results are ordered by time completed, not by the order of input.

    >>> from distributed import as_completed
    >>> seq = as_completed([a, b, c])
    >>> next(seq)
    <Future: status: finished, key: sleep-ce033b8d1e3927a69d8110154bb272eb>
    >>> next(seq)
    <Future: status: finished, key: sleep-81ed9178e93db3b0d45c06730d7079ff>
    >>> next(seq)
    <Future: status: finished, key: sleep-308f0c825431822d43330848333d0af6>


Manage Memory
=============

Distributed computations trigger immediately

This can fill all of distributed our memory.

    >>> futures1 = e.map(load, filenames)    # starts to load all data immediately
    >>> futures2 = e.map(process, futures1)  # intermediate results stay in RAM
    >>> futures3 = e.map(store, futures2)    # even after they're not needed

The scheduler holds on to all remote data for which there is an active future.
So we can clear up space on our cluster by deleting futures for intermediate
results.

    >>> del futures2


Dask collections
================

Distributed plays nicely with [dask](http://dask.pydata.org/en/latest/)
collections, like
`dask.`[`array`](http://dask.pydata.org/en/latest/array.html)/[`bag`](http://dask.pydata.org/en/latest/array.html)/[`dataframe`](http://dask.pydata.org/en/latest/dataframe.html)/[`imperative`](http://dask.pydata.org/en/latest/dataframe.html).

    >>> x = da.random.normal(loc=10, scale=0.1, size=(10000, 10000),
    ...                      chunks=(1000, 1000))
    >>> y = x.mean()

### get

Executors include a dask compatible `get` method that can be passed to a
collection:

    >>> y.compute(get=e.get)
    9.9999898105974729

Alternatively we can set this get function globally within dask

    >>> import dask
    >>> dask.set_options(get=e.get)
    >>> y.compute()
    9.9999898105974729

### compute

The operations above are synchronous.  They block until complete.  Most of
distributed is asynchronous, it returns futures immediately.  The
Executor.compute method converts dask collections into futures.  It expects a
list of collections and returns a list of futures

    >>> futures = e.compute(y)
    >>> e.gather(futures)
    9.9999898105974729


Dask Imperative
===============

Dask collections are lazy.  They don't start right away.

    >>> from dask.imperative import do
    >>> values1 = [do(load)(filename) for filename in filenames]  # lazy
    >>> values2 = [do(process)(val1) for val1 in values1]
    >>> values3 = [do(store)(val2) for val2 in values2]

When we submit `values3` to the executor it's clear that we don't need
`values1` and values2` to persist .  The distributed scheduler removes these
intermediate results during computation, freeing up precious space.

    >>> futures3 = e.compute(*values3)  # only final results persist


Dask Imperative and Futures
===========================

Dask imperative knows how to handle futures.  A common pattern is to persist
input data in memory and then build complex computations off of these futures
with dask imperative

    >>> futures1 = e.map(load, filenames) # base computations on persistent data

    >>> values2 = [do(process)(future) for future in futures1]  # complex work
    >>> values3 = [do(store)(val2) for val2 in values2]         # more work

    >>> futures3 = e.compute(*values3)  # Trigger execution, persist result in

The more computation we give to the scheduler at once, the smarter it can
schedule our work.
