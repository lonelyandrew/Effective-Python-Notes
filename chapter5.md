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

