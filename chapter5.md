# Chapter 5: Concurrency and Parallelism

**Concurrency** is when a computer does many different things *seemingly* at the same time. For example, one a computer with one CPU core, the operating system will rapidly change which program is running on the single processor. This interleaves execution of the programs, providing the illusion that the programs are running simultaneously.

**Parallelism** is *actually* doing many different  things at the same. Computers with multiple CPU cores can execute multiple programs simultaneously. Each CPU core runs the instructions of a separate program, allowing each program to make forward progress during the same instant.

The key difference between parallelism and concurrency is *speedup*. When two distinct paths of execution in a program make forward progress in parallel, the time it takes to do the total work is cut in half; the speed of execution is faster by a factor of two. In contrast, concurrent programs may run thousands of separate paths of execution seemingly in parallel but provide no speedup for the total work.

Python makes it easy to write concurrent programs. Python can also be used to do parallel work through system call, subprocesses, and C-extensions. But it can be very *difficult* to make concurrent Python code truly run in parallel.

## Item 36: Use `subprocess` to Manage Child Processes
Python has battle-hardened libraries for running and managing child processes. Child processes started by Python are able to run in parallel, enabling you to use Python to consume all of the CPU cores of your machine and maximize the throughput of your programs. Although Python itself may be CPU bound, it's easy to use Python to drive and coordinate CPU-intensive workloads.

The best and simplest choice of managing child processes is to use the `subprocess` built-in module.

Running a child process with `subprocess` is simple. Here, the `Popen` constructor starts the process. The communicate method reads the child process's output and waits for termination.

```Python
import subprocess
proc = subprocess.Popen(['echo', 'Hello from the child!'],
                        stdout=subprocess.PIPE)
out, err = proc.communicate()
print(out.decode('utf-8'))

>>>
Hello from the child!

```

The recommended approach to invoking subprocesses is to use the `run()` function for all use cases it can handle. The `run()` function was added in Python 3.5.

```Python
import subprocess
completed_proc = subprocess.run(['echo', 'Hello from the child!'],
                                stdout=subprocess.PIPE)
print(completed_proc.stdout.decode('utf-8'))
print(completed_proc.stderr)

>>>
Hello from the child!

None
```
Child processes will run independently from their parent process, the Python interpreter. Their status can be *polled* periodically while Python does other work. A `None` value returned by `proc.poll()` indicates that the process hasn’t terminated yet.

```Python
import subprocess

proc = subprocess.Popen(['sleep', '0.3'])
while proc.poll() is None:
    print('working...')
    # Some time-consuming work here
    # ...
    
print(f'Exit status:{proc.poll()}')

>>>
working...
working...
working...
working...
working...
Exit status:0
```

Decoupling the child process from the parent means that the parent process is free to run many child processes in parallel. You can do this by starting all the child processes together upfront.

```Python
import subprocess
from time import time

def run_sleep(period):
    proc = subprocess.Popen(['sleep', str(period)])
    return proc

start = time()
procs = []

for _ in range(10):
    proc = run_sleep(0.1)
    procs.append(proc)

for proc in procs:
    proc.communicate()

end = time()

print(f'Finished in {end-start:.3f} seconds')

>>>
Finished in 0.147 seconds
```

You can also pipe data from your Python program into a subprocess and retrieve its output. This allows you to utilize other programs to do work in parallel. For example, say you want to use the `openssl` command-line tool to encrypt some data. Starting the child process with command-line arguments and I/O pipes is easy.

```Python
import subprocess
import os


def run_openssl(data):
    env = os.environ.copy()
    env['password'] = b'password'
    proc = subprocess.Popen(
            ['openssl', 'enc', '-des3', '-pass', 'env:password'],
            env=env,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE)
    proc.stdin.write(data)
    proc.stdin.flush()
    return proc


procs = []
for _ in range(3):
    data = os.urandom(10)
    proc = run_openssl(data)
    procs.append(proc)

for proc in procs:
    out, err = proc.communicate()
    print(out[-10:])
```

You can also create chains of parallel processes just like UNIX pipes, connecting the output of one child process into the input of another, and so on. Here is a function that starts a child process that will cause the `md5` command-line tool to consume an input stream:

```Python
def run_md5(input_stdin):
    proc = subprocess.Popen(['md5'], stdin=input_stdin,
                            stdout=subprocess.PIPE)
    return proc


input_procs = []
hash_procs = []

for _ in range(3):
    data = os.urandom(10)
    proc = run_openssl(data)
    input_procs.append(proc)
    hash_proc = run_md5(proc.stdout)
    hash_procs.append(hash_proc)

for proc in input_procs:
    proc.communicate()
for proc in hash_procs:
    out, err = proc.communicate()
    print(out.strip())
    
>>>
b'd41d8cd98f00b204e9800998ecf8427e'
b'd41d8cd98f00b204e9800998ecf8427e'
b'c994cfb6408c8863394b157628baeda7'
```

If you're worried about the child processes never finishing or somehow blocking on input or output pipes, then be sure to pass the `timeout` parameter to the `communicate` method. This will cause an exception to be raised if the child process hasn't responded within a time period, giving you a chance to terminate the misbehaving child.

```Python
import subprocess


def run_sleep(period):
    proc = subprocess.Popen(['sleep', str(period)])
    return proc


proc = run_sleep(10)

try:
    proc.communicate(timeout=0.1)
except subprocess.TimeoutExpired:
    proc.terminate()
    proc.wait()

print('Exit status', proc.poll())

>>>
Exit status -15
```

Unfortunately, the `timeout` parameter is only available in Python 3.3 and later. In earlier versions of Python, you'll need to use the `select` build-in module on `proc.stdin`, `proc.stdout`, and `proc.stderr` in order to enforce timeouts on I/O.

## Item 37: Use Threads for Blocking I/O, Avoid for Parallelism
The standard implementation of Python is called CPython. CPython runs a Python program in two steps. First, it parses and compiles the source text into bytecode. Then it runs the bytecode using a stack-based interpreter. The bytecode interpreter has state that must be maintained and coherent while the Python program executes. Python enforces coherence with a mechanism called the *global interpreter lock* (GIL).

Essentially, the GIL is a mutual-exclusion lock (mutex) that prevents CPython from being affected by pre-emptive multithreading, where one thread takes control of a program by interrupting another thread. Such an interruption could corrupt the interpreter state if it comes at an unexpected time. The GIL prevents these interruptions and ensures that every bytecode instruction works correctly with the CPython implementation and its C-extension modules.

The GIL has an important negative side effect. With programs written in languages like C++ or Java, having multiple threads of execution means your program could utilize multiple CPU cores at the same time. Although Python supports multiple threads of execution, the GIL causes only one of them to make forward progress at a time. This means that when you reach for threads to do parallel computation and speed up your Python programs, you will be sorely disappointed.

Knowing these limitations you may wonder, why does Python support threads at all? There are two good reasons:

+ First, multiple threads make it easy for your program to seem like it's doing multiple things at the same time. Managing the juggling act of simultaneous tasks is difficult to implement yourself.
+ The second reason is to deal with blocking I/O, which happens when Python does certain types of system calls. System calls are how your Python program asks your computer's operation system to interact with the external environment on your behalf. Blocking I/O includes things like reading and writing files, interacting with networks, communicating with devices like displays, etc. Threads help you handle blocking I/O by insulating your program from the time it takes for the operating system to respond to your requests. When you find yourself needing to do blocking I/O and computation simultaneously, it's time to consider moving your system calls to threads. The GIL prevents Python code from running in parallel, but it has no negative effect on system calls. This works because Python threads release the GIL just before they make system calls and reacquire the GIL as soon as the system calls are done.

## Item 38: Use `Lock` to Prevent Data Races in Threads
Even though Python has a global interpreter lock, you're still responsible for protecting against data races between the threads in your programs.

Your programs will corrupt their data structures if you allow multiple threads to modify the same objects without locks.

The `Lock` class in the `threading` build-in module is Python's standard mutual exclusion lock implementation.

## Item 39: Use `Queue` to Coordinate Work Between Threads
Python programs that do many things concurrently often need to coordinate their work. One of the most useful arrangements for concurrent work is a pipeline of functions.

A pipeline works like an assembly line used in manufacturing. For example, say you want to build a system that will take a constant stream of images from your digital camera, resize them, and then add them to a photo gallery online. New images are retrieved in the first phase. The downloaded images are passed through the resize function in the second phase. The resize images are consumed by the upload function in the final phase.

```Python
from collections import deque
from threading import Lock
from threading import Thread
from time import sleep
import sys


def download(item):
    sleep(0.1)
    print('download ', str(item))
    return f'downloaded {item}'


def resize(item):
    sleep(0.1)
    print('resize ', str(item))
    return f'resized {item}'


def upload(item):
    sleep(0.1)
    print('upload ', str(item))
    return f'uploaded resized {item}'


class MyQueue(object):
    def __init__(self):
        self.items = deque()
        self.lock = Lock()

    def put(self, item):
        with self.lock:
            self.items.append(item)

    def get(self):
        with self.lock:
            return self.items.popleft()


class Worker(Thread):
    def __init__(self, func, in_queue, out_queue):
        super().__init__()
        self.func = func
        self.in_queue = in_queue
        self.out_queue = out_queue


        self.polled_count = 0
        self.work_done = 0

    def run(self):
        while True:
            self.polled_count += 1
            try:
                item = self.in_queue.get()
            except IndexError:
                sleep(0.01)
            else:
                result = self.func(item)
                self.out_queue.put(result)
                self.work_done = 1


if __name__ == '__main__':
    download_queue = MyQueue()
    resize_queue = MyQueue()
    upload_queue = MyQueue()
    done_queue = MyQueue()
    threads = [
        Worker(download, download_queue, resize_queue),
        Worker(resize, resize_queue, upload_queue),
        Worker(upload, upload_queue, done_queue)
    ]

    for thread in threads:
        thread.start()
    for i in range(100):
        download_queue.put(i)

    while len(done_queue.items) < 100:
        pass
    processed = len(done_queue.items)
    polled = sum(t.polled_count for t in threads)
    print('Processed ', processed, ' items after polling ', polled, ' times')
    
>>>
...
Processed  100  items after polling  353  times
```

When the worker functions vary in speeds, an earlier phase can prevent progress in later phases, backing up the pipeline. This cause later phases to starve and constantly check their input queues for new work in a tight loop. The out come is that worker threads waste CPU time doing nothing useful (they are constantly raising and catching `IndexError` exceptions).

There are three more problems that you should also avoid. First, determining that all of the input work is complete requires yet another busy wait on the `done_queue`.  Second, in `Worker` the `run` method will execute forever in its busy loop. There's no way to signal to a worker thread that it's time to exit. Third, and worst of all, a backup in the pipeline can cause the program to crash arbitrarily.

The lesson here is not that pipelines are bad; it's that it's hard to build a good producer-consumer queue yourself.

The `Queue` class from the `queue` built-in module provides all of the functionality you need to solve these problems.

`Queue` eliminates the busy waiting in the worker by making the `get` method block until new data is available.

```Python
from queue import Queue
from threading import Thread

queue = Queue()

def consumer():
    print('Consumer waiting')
    queue.get()  # Runs after put() below
    print('Consumer done')

thread = Thread(target=consumer)
thread.start()

print('Producer putting')
queue.put(object())
thread.join()
print('Producer done')

>>>
Consumer waiting
Producer putting
Consumer done
Producer done
```

To solve the pipeline backup issue, the `Queue` class lets you specify the maximum amount of pending work you will allow between two phases. The buffer size causes calls to `put` to block when the queue is already full.

```Python
from queue import Queue
from threading import Thread
from time import sleep

queue = Queue(1)  # Buffer size of 1

def consumer():
    sleep(0.1)  # Wait
    queue.get()  # Runs second
    print('Consumer got 1')
    queue.get()  # Runs fourth
    print('Consumer got 2')

thread = Thread(target=consumer)
thread.start()

queue.put(object())  # Runs first
print('Producer put 1')
queue.put(object())  # Runs third
print('Producer put 2')
thread.join()
print('Producer done')

>>>
Producer put 1
Consumer got 1
Producer put 2
Consumer got 2
Producer done
```

The `Queue` class can also track the progress of work using the `task_done` method. This lets you wait for a phase's input queue to drain and eliminates the need for polling the `done_queue` at the end of your pipeline.

```Python
from queue import Queue
from threading import Thread

in_queue = Queue()

def consumer():
    print('Consumer waiting')
    work = in_queue.get()
    print('Consumer working')
    print('Consumer done')
    in_queue.task_done()

Thread(target=consumer).start()

in_queue.put(object())
print('Producer waiting')
in_queue.join()
print('Producer done')

>>>
Consumer waiting
Producer waiting
Consumer working
Consumer done
Producer done
```

I can put all of these behaviors together into a `Queue` subclass that also tells the worker thread when it should stop processing.

```Python
from threading import Thread
from time import sleep
import sys
from queue import Queue


def download(item):
    sleep(0.1)
    print('download ', str(item))
    return f'downloaded {item}'


def resize(item):
    sleep(0.1)
    print('resize ', str(item))
    return f'resized {item}'


def upload(item):
    sleep(0.1)
    print('upload ', str(item))
    return f'uploaded resized {item}'


class ClosableQueue(Queue):
    SENTINEL = object()

    def close(self):
        self.put(self.SENTINEL)

    def __iter__(self):
        while True:
            item = self.get()
            try:
                if item is self.SENTINEL:
                    return
                yield item
            finally:
                self.task_done()

class StoppableWorker(Thread):
    def __init__(self, func, in_queue, out_queue):
        super().__init__()
        self.func = func
        self.in_queue = in_queue
        self.out_queue = out_queue

    def run(self):
        for item in self.in_queue:
        result = self.func(item)
        self.out_queue.put(result)


download_queue = ClosableQueue()
resize_queue = ClosableQueue()
upload_queue = ClosableQueue()
done_queue = ClosableQueue()

threads = [
    StoppableWorker(download, download_queue, resize_queue),
    StoppableWorker(resize, resize_queue, upload_queue),
    StoppableWorker(upload, upload_queue, done_queue),
]

for thread in threads:
    thread.start()

for i in range(100):
    download_queue.put(i)

download_queue.close()
download_queue.join()
resize_queue.close()
resize_queue.join()
upload_queue.close()
upload_queue.join()
print(done_queue.qsize(), 'items finished')
```

## Item 40: Consider Coroutines to Run Many Functions Concurrently
Threads give Python programmers a way to run multiple functions seemingly at the same time. But there are three big problems with threads:

+ They require special tools to coordinate with each other safely. This makes code that uses threads harder to reason about than procedural, single-threaded code. This complexity makes threaded code more difficult to extend and maintain over time.
+ Threads require a lot of memory, about 8MB per executing thread. On many computers, that amount of memory doesn't matter for a dozen threads or so. But what if you want your program to run tens of thousands of functions "simultaneously"? These functions may correspond to user requests to a server, pixels on a screen, particles in a simulation, etc. Running a thread per unique activity just won't work.
+ Threads are costly to start. If you want to constantly be creating new concurrent functions and finishing them, the overhead of using threads becomes large and slows everything down.

Python can work around all these issues with *coroutines*. Coroutines let you have many seemingly simultaneous functions in your Python programs. They are implemented as an extension to generators. The cost of starting a generator coroutine is a function call. Once active, they each use less than 1KB of memory until they're exhausted.

Coroutines work by enabling the code consuming a generator to `send` a value back into the generator function after each `yield` expression. The generator function receives the value passed to the `send` function as the result of the corresponding `yield` expression.

```Python
def my_coroutine():
    while True:
        received = yield
        print('Received:', received)


it = my_coroutine()
next(it)  # Prime the coroutine
it.send('First')
it.send('Second')

>>>
Received: First
Received: Second
```

The initial call to `next` is required to prepare the generator for receiving the first `send` by advancing it to the first `yield` expression. You can also prepare the generator by the expression `it.send(None)`.

For example, say you want to implement a generator coroutine that yields the minimum value it's been sent so far. Here, the bare `yield` prepares the coroutine with the initial minimum value sent in from the outside. Then the generator repeatedly yields the new minimum in exchange for the next value to consider.

```Python
def minimize():
    current = yield
    while True:
        value = yield current
        current = min(value, current)

it = minimize()
next(it)
print(it.send(10))
print(it.send(4))
print(it.send(22))
print(it.send(-1))

>>>
10
4
4
-1
```

The code for game of life:


```Python
from collections import namedtuple


ALIVE = '*'
EMPTY = '-'

Query = namedtuple('Query', ('y', 'x'))


def count_neighbors(y, x):
    n_ = yield Query(y+1, x+0)  # North
    ne = yield Query(y+1, x+1)  # North East
    e_ = yield Query(y+0, x+1)  # East
    se = yield Query(y-1, x+1)  # South East
    s_ = yield Query(y-1, x+0)  # South
    sw = yield Query(y-1, x-1)  # South West
    w_ = yield Query(y+0, x-1)  # West
    nw = yield Query(y+1, x-1)  # North West

    neighbor_states = [n_, ne, e_, se, s_, sw, w_, nw]
    count = 0
    for state in neighbor_states:
        if state == ALIVE:
            count += 1
    return count


it = count_neighbors(10, 5)
q1 = next(it)
print('First yield: ', q1)
q2 = it.send(ALIVE)
print('Second yield: ', q2)
q3 = it.send(ALIVE)
print('Third yield: ', q3)
q4 = it.send(EMPTY)
print('Fourth yield: ', q4)
q5 = it.send(EMPTY)
print('Fifth yield: ', q5)
q6 = it.send(EMPTY)
print('Sixth yield: ', q6)
q7 = it.send(EMPTY)
print('Seven yield: ', q7)
q8 = it.send(EMPTY)
print('Eighth yield: ', q8)

try:
    count = it.send(EMPTY)
except StopIteration as e:
    print('Count: ', e.value)


Transition = namedtuple('Transition', ('y', 'x', 'state'))


def game_logic(state, neighbors):
    if state == ALIVE:
        if neighbors < 2:
            return EMPTY
        elif neighbors > 3:
            return EMPTY
    else:
        if neighbors == 3:
            return ALIVE
    return state


def step_cell(y, x):
    state = yield Query(y, x)
    neighbors = yield from count_neighbors(y, x)
    next_state = game_logic(state, neighbors)
    yield Transition(y, x, next_state)


it = step_cell(10, 5)
q0 = next(it)
print('Me:      ', q0)
q1 = it.send(ALIVE)
print('Q1       ', q1)
print('...')
q2 = it.send(ALIVE)
q3 = it.send(ALIVE)
q4 = it.send(ALIVE)
q5 = it.send(ALIVE)
q6 = it.send(EMPTY)
q7 = it.send(EMPTY)
q8 = it.send(EMPTY)
t1 = it.send(EMPTY)
print('Outcome: ', t1)


TICK = object()


def simulate(height, width):
    while True:
        for y in range(height):
            for x in range(width):
                yield from step_cell(y, x)
        yield TICK


class Grid(object):
    def __init__(self, height, width):
        self.height = height
        self.width = width
        self.rows = []
        for _ in range(self.height):
            self.rows.append([EMPTY] * self.width)

    def __str__(self):
        output = ''
        for row in self.rows:
            for cell in row:
                output += cell
            output += '\n'
        return output

    def query(self, y, x):
        return self.rows[y % self.height][x % self.width]

    def assign(self, y, x, state):
        self.rows[y % self.height][x % self.width] = state


def live_a_generation(grid, sim):
    progeny = Grid(grid.height, grid.width)
    item = next(sim)
    while item is not TICK:
        if isinstance(item, Query):
            state = grid.query(item.y, item.x)
            item = sim.send(state)
        else:
            progeny.assign(item.y, item.x, item.state)
            item = next(sim)
    return progeny


grid = Grid(5, 9)
grid.assign(0, 3, ALIVE)
grid.assign(1, 4, ALIVE)
grid.assign(2, 2, ALIVE)
grid.assign(2, 3, ALIVE)
grid.assign(2, 4, ALIVE)
print(grid)


class ColumnPrinter(object):
    def __init__(self):
        self.columns = []

    def append(self, data):
        self.columns.append(data)

    def __str__(self):
        row_count = 1
        for data in self.columns:
            row_count = max(row_count, len(data.splitlines()) + 1)
        rows = [''] * row_count
        for j in range(row_count):
            for i, data in enumerate(self.columns):
                line = data.splitlines()[max(0, j - 1)]
                if j == 0:
                    padding = ' ' * (len(line) // 2)
                    rows[j] += padding + str(i) + padding
                else:
                    rows[j] += line
                if (i + 1) < len(self.columns):
                    rows[j] += ' | '
        return '\n'.join(rows)

columns = ColumnPrinter()
sim = simulate(grid.height, grid.width)
for i in range(5):
    print('round', i)
    columns.append(str(grid))
    grid = live_a_generation(grid, sim)

print(columns)

>>>
First yield:  Query(y=11, x=5)
Second yield:  Query(y=11, x=6)
Third yield:  Query(y=10, x=6)
Fourth yield:  Query(y=9, x=6)
Fifth yield:  Query(y=9, x=5)
Sixth yield:  Query(y=9, x=4)
Seven yield:  Query(y=10, x=4)
Eighth yield:  Query(y=11, x=4)
Count:  2
Me:       Query(y=10, x=5)
Q1        Query(y=11, x=5)
...
Outcome:  Transition(y=10, x=5, state='-')
---*-----
----*----
--***----
---------
---------

round 0
round 1
round 2
round 3
round 4
    0     |     1     |     2     |     3     |     4
---*----- | --------- | --------- | --------- | ---------
----*---- | --*-*---- | ----*---- | ---*----- | ----*----
--***---- | ---**---- | --*-*---- | ----**--- | -----*---
--------- | ---*----- | ---**---- | ---**---- | ---***---
--------- | --------- | --------- | --------- | ---------
```

Python 2 is missing some of the syntactical sugar that makes coroutines so elegant in Python 3. There are two limitations.

First, there is no `yield from` expression. That means that when you want to compose generator coroutines in Python 2, you need to include an additional loop at the delegation point.

```Python
def delegated():
    yield 1
    yield 2


def composed():
    yield 'A'
    # yield from delegated()
    for value in delegated():
        yield value
    yield 'B'

print(list(composed()))

>>>
['A', 1, 2, 'B']
```

The second limitation is that there is no support for the `return` statement in Python 2 generators. To get the same behavior that interacts correctly with `try`/`except`/`finally` blocks, you need to define your own exception type and raise it when you want to return a value.

```Python
class MyReturn(Exception):
    def __init__(self, value):
        self.value = value


def delegated():
    yield 1
    raise MyReturn(2)
    yield 'Not reached'


def composed():
    try:
        for value in delegated():
            yield value
    except MyReturn as e:
        output = e.value
    yield output * 4

print(list(composed()))

>>>
[1, 8]
```

### Item 41: Consider `concurrent.futures` for True Parallelism
The `multiprocessing` built-in module, easily accessed via the `concurrent.futures` built-in module, enables Python to utilize multiple CPU cores in parallel by running additional interpreters as child processes. These child processes are separate from the main interpreter, so their global interpreter locks are also separate. Each child can fully utilize one CPU core. Each child has a link to the main process where it receives instructions to do computation and returns results.

```Python
def gcd(pair):
    a, b = pair
    low = min(a, b)
    for i in range(low, 0, -1):
        if a % i == 0 and b % i == 0:
            return i


from time import time
numbers = [(1963309, 2265973), (2030677, 3814172),
           (1551645, 2229620), (2039045, 2020802)]
start = time()
results = list(map(gcd, numbers))
end = time()
print('Took %.3f seconds' % (end - start))

>>>
Took 1.053 seconds
```

Running this code on multiple Python threads will yield no speed improvement because the GIL prevents Python from using multiple CPU cores in parallel.

```Python
from concurrent.futures import ThreadPoolExecutor


def gcd(pair):
    a, b = pair
    low = min(a, b)
    for i in range(low, 0, -1):
        if a % i == 0 and b % i == 0:
            return i


from time import time
numbers = [(1963309, 2265973), (2030677, 3814172),
           (1551645, 2229620), (2039045, 2020802)]
start = time()
pool = ThreadPoolExecutor(max_workers=2)
results = list(pool.map(gcd, numbers))
end = time()
print('Took %.3f seconds' % (end - start))

>>>
Took 1.068 seconds
```

It's even slower this time because of the overhead of starting and communicating with the pool of threads.

If I replace the `ThreadPoolExecutor` with the `ProcessPoolExecutor` from the `concurrent.futures` module, everything speeds up.

```Python
from concurrent.futures import ProcessPoolExecutor


def gcd(pair):
    a, b = pair
    low = min(a, b)
    for i in range(low, 0, -1):
        if a % i == 0 and b % i == 0:
            return i


from time import time
numbers = [(1963309, 2265973), (2030677, 3814172),
           (1551645, 2229620), (2039045, 2020802)]
start = time()
pool = ProcessPoolExecutor(max_workers=2)
results = list(pool.map(gcd,numbers))
end = time()
print('Took %.3f seconds' % (end - start))

>>>
Took 0.556 seconds
```

Here's what the `ProcessPoolExecutor` class actually does (via the low-level constructs provided by the `multiprocessing` module):

1. It takes each item from the `numbers` input data to `map`.
2. It serializes it into binary data using the `pickle` module.
3. It copies the serialized data from the main interpreter process to a child interpreter process over a local socket.
4. Next, it deserialized the data back into Python objects using `pickle` in the child process.
5. It then imports the Python module containing the `gcd` function.
6. It runs the function on the input data in parallel with other child processes.
7. It serialized the result back into bytes.
8. It copies those bytes back through the socket.
9. It deserializes the bytes back into Python objects in the parent process.
10. Finally, it merges the results from multiple children into a single list to return.

This scheme is well suited to certain types of isolated, high-leverage tasks. By *isolated*, I mean functions that don't need to share state with other parts of the program. By *high-leverage*, I mean situations in which only a small amount of data must be transferred between the parent and children processes to enable a large amount of computation.

(END OF CHAPTER 5)

