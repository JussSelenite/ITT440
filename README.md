# PARALLEL PROGRAMMING IN PYTHON
## DIANA BINTI ZULKIFLI
## 2025920341
## M3CS2554A

### What is parallel programming? 
Parallel programming is a type of computation where a task is divided into smaller subtasks to be processed simultaneously. 

#### Hyper-threading
Intel brought **hyper-threading** to consumer CPUs in 2002.

With hyper-threading, one physical core *appears as 2 logical processors* to the operating system. The two share that core's execution resources, so the core can use parts that would otherwise sit idle while it waits.

#### Adding cores to CPU
Adding physical cores has a bigger effect than hyper-threading, because each core has its own execution resources instead of sharing them. For example, a dual-core CPU with hyper-threading has **2 cores** but shows up as **4 logical processors**.

### Concurrency vs Parallelism
**Concurrency** is when multiple tasks make progress during the same period
by taking turns. A single core can do this by switching quickly between tasks,
so they only *appear* to run at once.

**Parallelism** is when multiple tasks truly run at the same instant, each on
its own core. It needs more hardware, and it speeds up heavy computation.

*Analogy: one cook juggling three pots is concurrency. Three cooks with three
pots each is parallelism.*


### Parallel programming in Python
In Python, one of the modules used for parallel programming is the `multiprocessing` module. It lets user start child processes from a program (the parent process). A process is an isolated program instance with its own memory, and the OS can run the said processes on different CPU cores at the same time.

The Glocal Interpreter Lock (GIL) lets only one thread to run Python code at a time, so threads don't speed up heavy computation. Each process has its own interpreter and its own GIL, so several processes can crunch numbers truly in parallel.

A `Pool` starts a fixed set of workers once and feeds them many small tasks, so no startup cost needed for evey task

Processes have **no shared memory**, data is pickled and sent through `Queue`, `Pipe`, or pool channels; so anything passed to a worker *must be picklable*. Besides that, `if__name__ == "__main__":` is aldo needed on Windows because each child re-imports the main file, and without the guard it would keep starting new processes.

#### Multiprocessing example using Process
`Process` runs a callable in a child process. Start with `start()`, then wait with `join()`.

```python
from multiprocessing import Process

def say_hi(name):
	print(f"hello from worker, {name}!")


if __name__ == "__main__":
	p = Process(target=say_hi, args=("Ahmad",))
	p.start()
	p.join()
	print("parent done")
```
The output should look like this:
```
hello from worker, Ahmad!
parent done
```
the worker prints from its own process, the parent prints after the child has finished.

#### Pass arguments to a process
Use `args` for positional arguments, and `kwargs` for keyword arguments (must be picklable).

```python
from multiprocessing import Process


def add(a, b, label="sum"):
    print(label, a + b)


if __name__ == "__main__":
    p = Process(target=add, args=(10, 20), kwargs={"label": "hasilnya"})
    p.start()
    p.join()
```
The output should look like:
```
hasilnya 30
```
**Note that `Process.start()` returns `None`**, so keep the `Process` in a variable, then call `start()` and `join()` on that object.

#### Wait for processes using join()
`join()` blocks the parent until the child terminates.
```python
from multiprocessing import Process
import time


def work():
    time.sleep(0.1)


if __name__ == "__main__":
    p = Process(target=work)
    p.start()
    p.join(timeout=2)
    print("alive?", p.is_alive())
    print("exit code", p.exitcode)
```
In the example, the `timeout` is 2 seconds; `join` returns after that time passed, even if the child is running. `is_alive()` is to see whether it finished. After a normal exit, `exitcode` is usually 0.

#### Run multiple processes in a loop
Create a list of `Process` objects, start them all, then join them all so the parent waits for every child.

```python
from multiprocessing import Process


def square(n):
    print(n * n)


if __name__ == "__main__":
    procs = [Process(target=square, args=(i,)) for i in range(3)]
    for p in procs:
        p.start()
    for p in procs:
        p.join()
    print("all workers finished")

```
The output can vary, as the OS schedule processes independently. 

Sample output:
```
0
4
1
all workers finished
```


#### Get results from processes using Queue
`Queue` is a process-safe way for workers to send results back to the parent. Each worker `put` its result in the queue, and the parent `get` them.
```python
from multiprocessing import Process, Queue


def worker(n, q):
    q.put(n * n)


if __name__ == "__main__":
    q = Queue()
    procs = [Process(target=worker, args=(i, q)) for i in range(4)]
    for p in procs:
        p.start()
    for p in procs:
        p.join()
    results = [q.get() for _ in procs]
    print(sorted(results))

```
The output should look like:
```[0, 1, 4, 9]```


### Using multiprocessing Pool

`Pool` keeps fixed number of worker processes and reuses them for many tasks. Much cheaper than spawning a new `Process` per item, for large batches.

```python
from multiprocessing import Pool


def inc(x):
    return x + 1


if __name__ == "__main__":
    with Pool(2) as pool:
        print(pool.map(inc, [1, 2, 3, 4]))

```

Output:```[2, 3, 4, 5]```

To ensure workers shut down cleanly, always use pool as a context manager (`with Pool(...) as pool:`) or call `close()` and `join()`.

#### Pool.map() example
`map(func, iterable)` splits the iterable across workers and preserves **result order**.

```python
from multiprocessing import Pool


def is_even(n):
    return n % 2 == 0


if __name__ == "__main__":
    with Pool(3) as pool:
        print(pool.map(is_even, range(6)))

```
Output: ```[True, False, True, False, True, False]```

#### Pool.starmap() example
`starmap` is similar to `map` but each input must be an iterable of arguments unpacked with *.
```python
from multiprocessing import Pool


def mul(a, b):
    return a * b


if __name__ == "__main__":
    tasks = [(2, 3), (4, 5), (6, 7)]
    with Pool(2) as pool:
        print(pool.starmap(mul, tasks))

```
It's useful when each task needs several parameters.
Output:```[6, 20, 42]```

## Frameworks used in Python
Several frameworks have been made for multiprocessing, including **Dask**, **Ray**, **Spark**, and **Joblib**. [This video](https://youtu.be/gP78DhjAEzY) shows a demonstration for the 4 frameworks.

### Dask
A pure Python parallel computing library first released in 2015 by Continuum Analytics (now Anaconda). It has a single design principle: make distributed computation feel like native Python. At its core, Dask build task graphs, directed acylic graphs (DAG) where each node is a Python callable and each edge is a data dependency. Dask distributed extends this model across multiple machines, supporting DataFrames, arrays, and bags with a lightweight configuration foorprint.

### Ray
A distributed computing framework from UC Berkeley, designed to geenralize parallel computing across any Python application. It provides Ray tasks (remote functions) and actors (stateful distributed processes) that can run across a ray cluster of any size and hardware composition. Ray has also become a standard infrastructure layer for large language model (LLM) workloads.

### Spark
Apache Spark framework provides an open-source distributed computation engine developed at UC Berkeley's AMPLab, since 2009. It represents an immutable, fault-tolerant collection of data partitioned across a cluster. Spark is written in Scala, with PySpark providing Python bindings. For data engineering and large scale data processing, Spark remains the most mature and commercially supported option.

### Joblib
Joblib is a pacakge for parallel computing and disk-based caching in Python. It simplifies the creation of pipelines which can orchestrate multi-step analyses. It is also optimized to be fast and robust on large data in particular, and has specific optimizations for _numpy_ arrays.

## Sources
- [Python Multiprocessing Example: Process, Pool, and Queue](https://www.golinuxcloud.com/python-multiprocessing/)
- [Dask vs Spark vs Ray: Choosing the right distributed computing framework](https://domino.ai/blog/spark-dask-ray-choosing-the-right-framework)
- [Parallel in Python](https://docs.ycrc.yale.edu/parallel_python/)
- [Joblib Documentation](https://joblib.readthedocs.io/en/stable/)