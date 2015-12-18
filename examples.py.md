Install
=======

The best way to get distributed is to point `pip` to the github repository

    pip install https://github.com/blaze/distributed.git --upgrade

Setup
=====

We set up a center on one node

    $ dcenter     # We use the below address to refer to the cluster
    Start center at 127.0.0.1:8787

And set up workers on other processes or computers that point to this address

For simplicity we used localhost here (`127.0.0.1`) but we could have used
other machines and real addresses if desired.

    $ dworker 127.0.0.1:8787
    Start worker at             127.0.0.1:46990
    Registered with center at:  127.0.0.1:8787

    $ dworker 127.0.0.1:8787
    Start worker at             127.0.0.1:46990
    Registered with center at:  127.0.0.1:8787


Executor
=======

Connect an executor to the center node of the cluster

    from distributed import Executor
    e = Executor('127.0.0.1:8787')


Submit and Result
=================

Submit a function for remote execution and receive a future.

    def inc(x):
        return x + 1

    >>> future = e.submit(inc, 1)  # calls inc(1) on a worker
    >>> future
    <Future: status: finished, key: inc-79963f7e613e5e55838d2232920baed2>

The future doesn't hold the actual result.  It points to the result on a remote
worker.

Collect the remote result from the remote worker to the local process by
calling the `.result()` method on the future.

    >>> future.result()  # collect result from remote worker
    2


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

`Future.result` and `Executor.gather` are a bit redundant.  Generally
`Executor.gather` is more efficient.


Submit and map on futures
=========================

We `submit` and `map` functions directly onto futures.

We don't need to collect the intermediate result, `future`, to the local
process.  This avoids sending large remote results across the network.  We move
the function to the data rather than move the data to the function.

    def inc(x):
        return x + 1

    def double(x):
        return 2 * x

    future = e.submit(inc, 1)
    future2 = e.submit(double, future)

You should avoid data-movement operations like `future.result()` or `
e.gather()` when possible

    # future.result()  # avoid expensive calls to result/gather if unnecessary

We move the function to the worker node that already has the data for the
input.  This minimizes communication within the network.


Submit and map with many inputs
===============================

Submit and map can consume multiple inputs and keyword arguments

    >>> e.submit(function, *args, **kwargs)
    >>> e.map(function, *iterables, **kwargs)

This includes consuming multiple futures

    >>> futures = e.map(double, range(10))

    >>> total = e.submit(sum, futures)   # call sum on a list of futures
    >>> total.result()
    90

If the remote inputs live on different worker nodes then we move all of the
necessary data to a single node where we run the function.  Generally we run
choose the node that would require the least communication of bytes.


Scatter
=======

Send local data to the network with the `scatter` function.

    >>> e.scatter([1, 2, 3])
    [<Future: status: finished, key: 526d1868-a5a1-11e5-901d-60672020cfac-0>,
     <Future: status: finished, key: 526d1868-a5a1-11e5-901d-60672020cfac-1>,
     <Future: status: finished, key: 526d1868-a5a1-11e5-901d-60672020cfac-2>]

This is common when we have data on our local disk that we want to load and
then upload to the network

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
