# Chapter 6: Built-in Modules

## Item 42: Define Function Decorators with `functools.wraps`
Decorators have the ability to run additional code before and after any calls to the functions they wrap. This allow them to access and modify input arguments and return values.

For example, say you want to print the arguments and return value of a function call. This is especially helpful when debugging a stack of function calls from a recursive function.

```Python
def trace(func):
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        print(f'{func.__name__}({args, kwargs}) -> {result}')
        return result
    return wrapper


@trace
def fibonacci(n):
    if n in (0, 1):
        return n
    return (fibonacci(n-2) + fibonacci(n-1))
    

fibonacci(3)

>>>
fibonacci(((1,), {})) -> 1
fibonacci(((0,), {})) -> 0
fibonacci(((1,), {})) -> 1
fibonacci(((2,), {})) -> 1
fibonacci(((3,), {})) -> 2
```

This works well, but it has an unintended side effect. The value returned by the decorator - the function that's called above - doesn't think it's named `fibonacci`.

```Python
print(fibonacci)

>>>
<function trace.<locals>.wrapper at 0x103b28840>
```

The cause of this  isn't hard to see. The `trace` function returns the `wrapper` it defines. This behavior is problematic because it  undermines tools that do introspection, such as debuggers and object serializers.

```Python
help(fibonacci)

>>>
Help on function wrapper in module __main__:

wrapper(*args, **kwargs)
(END)
```

The solution is to use the `wraps` helper function from the `functools` built-in module. This is a decorator that helps you write decorators. Applying it to the wrapper function will copy all of the important meta-data about the inner function to the outer function.

```Python
from functools import wraps


def trace(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        print(f'{func.__name__}({args, kwargs}) -> {result}')
        return result
    return wrapper
    
>>>
<function fibonacci at 0x1094ab840>

Help on function fibonacci in module __main__:

fibonacci(n)
(END)
```

## Item 43: Consider `contextlib` and `with` Statements for Reusable `try/finally` Behavior
The `with` statement in Python is used to indicate when code is running in a special context. The `with` statement eliminates the need to write the repetitive code of the `try/finally` construction.

It's easy to make your objects and functions capable of use in `with` statements by using the `contextlib` built-in module. This module contains the `contextmanager` decorator, which lets a simple function be used in `with` statements. This is much easier than defining a new class with the special methods `__enter__` and `__exit__` (the standard way).

For example, say you want a region of your code to have more debug logging sometimes.

```Python
import logging


def my_function():
    logging.debug('Some debug data')
    logging.error('Error log here')
    logging.debug('More debug data')


my_function()

>>>
ERROR:root:Error log here
```

I can elevate the log level of this function temporarily by defining a context manager. This helper function boosts the logging severity level before running the code in the `with` block and reduces the logging severity level afterward.

```Python
import logging
from contextlib import contextmanager


def my_function():
    logging.debug('Some debug data')
    logging.error('Error log here')
    logging.debug('More debug data')

@contextmanager
def debug_logging(level):
    logger = logging.getLogger()
    old_level = logger.getEffectiveLevel()
    logger.setLevel(level)
    try:
        yield
    finally:
        logger.setLevel(old_level)

with debug_logging(logging.DEBUG):
    my_function()

>>>
DEBUG:root:Some debug data
ERROR:root:Error log here
DEBUG:root:More debug data
```

The `yield` expression is the point at which the `with` block's content will execute. Any exceptions that happen in the `with` block will be re-raised by the `yield` expression for you to catch in the helper function.

The standard way to implement it is as following:

```Python
import logging


class DebugLogging(object):

    def __init__(self, level):
        self.level = level

    def __enter__(self):
        logger = logging.getLogger()
        self.old_level = logger.getEffectiveLevel()
        logger.setLevel(self.level)

    def __exit__(self, *args):
        logger = logging.getLogger()
        logger.setLevel(self.old_level)


def my_function():
    logging.debug('Some debug data')
    logging.error('Error log here')
    logging.debug('More debug data')


with DebugLogging(logging.DEBUG):
    my_function()
    

>>>
DEBUG:root:Some debug data
ERROR:root:Error log here
DEBUG:root:More debug data
```

The context manager passed to a `with` statement may also return an object. This object is assigned to a local variable in the `as` part of the compound statement. This gives the code running in the `with` block the ability to directly interact with its context.

To enable your own functions to supply values for `as` targets, all you need t o do is `yield` a value from your context manager.

## Item 44: Make `pickle` Reliable with `copyreg`
The `pickle` built-in module can serialize Python objects into a stream of bytes and deserialize bytes back into objects. Pickled byte streams shouldn't be used to communicate untrusted parties. The purpose of `pickle` is to let you pass Python objects between programs that you control over binary channels.

The `pickle` module's serialization format is *unsafe* by design. The serialized data contains what is essentially a program that describes how to reconstruct the original Python object. This means a malicious pickle `pickle` payload could be used to compromise any part of the Python program that attempts to deserialize it.

In contrast, the `json` module is safe by design. Serialized JSON data contains a simple description of an object hierarchy. Deserializing JSON data does not expose a Python program to any additional risk. Formats like JSON should be used for communication between programs or people that don't trust each other.

For example, say you want to use a Python object to represent the state of a player's progress in a game. The game state includes the level the player is on and the number of lives he or she has remaining.

```Python
import pickle


class GameState(object):
    def __init__(self):
        self.level = 0
        self.lives = 4


state = GameState()
state.level += 1
state.lives -= 1

state_path = '/tmp/game_state.bin'
with open(state_path, 'wb') as f:
    pickle.dump(state, f)

with open(state_path, 'rb') as f:
    state_after = pickle.load(f)

print(state_after.__dict__)

>>>
{'level': 1, 'lives': 3}
```

The problem with this approach is what happens as the game's features expand over time. Imagine you want the player to earn points towards a high score. To track the player's points, you'd add a new field to the `GameState` class.

```Python
import pickle


class GameState(object):
    def __init__(self):
        self.level = 0
        self.lives = 4
        self.point = 0


state = GameState()
serialized = pickle.dumps(state)
state_after = pickle.loads(serialized)
print(state_after.__dict__)

>>>
{'level': 0, 'lives': 4, 'point': 0}
```

But what happens to older saved `GameState` objects that the used may want to resume?

```Python
state_path = '/tmp/game_state.bin'
with open(state_path, 'rb') as f:
    state_after = pickle.load(f)
print(state_after.__dict__)

>>>
{'level': 1, 'lives': 3}
```

The `points` attributes is missing! This is especially confusing because the returned object is an instance of the new `GameState` class.

```Python
assert isinstance(state_after, GameState)
```

Fixing these problems is straightforward using the `copyreg` built-in module. The `copyreg` module lets you register the functions responsible for serializing Python objects, allowing you to control behavior of `pickle` and make it more reliable.

In the simplest case, you can use a constructor with default arguments to ensure that `GameState` objects will always have all attributes after unpickling.

```Python
class GameState(object):
    def __init__(self, level=0, lives=4, points=0):
        self.level = level
        self.lives = lives
        self.point = points
```

To use this constructor for pickling, I define a helper function that takes a `GameState` object and turns it into a tuple of parameters for the `copyreg` module.

```Python
def pickle_game_state(game_state):
    kwargs = game_state.__dict__
    return unpickle_game_state, (kwargs, )
```

Now I need to define the `unpickle_game_state` helper. This function takes serialized data and parameters from `pickle_game_state` and returns the corresponding `GameState` object.

```Python
def unpickle_game_state(kwargs):
    return GameState(**kwargs)
```

Now, I register these with the `copyreg` built-in module.

```Python
copyreg.pickle(GameState, pickle_game_state)
```

Serializing and deserializing works as before.

```Python
state = GameState()
state.points += 1000
serialized = pickle.dumps(state)
state_after = pickle.loads(serialized)
print(state_after.__dict__)

>>>
{'level': 0, 'lives': 4, 'points': 1000}
```

Sometimes you'll need to make backwards-incompatible changes to your Python objects by removing fields. This prevents the default argument approach to serialization from working.


For example, say you realize that a limited number of lives is a bad idea, and you want to remove the concept of lives from the game.

```Python
class GameState(object):
    def __init__(self, level=0, points=0, magic=5):
        self.level = level
        self.points = points
        self.magic = magic
```

The problem is that this breaks deserialization old game data. All fields from the old data, even ones removed from the class, will be passed to the `GameState` constructor by the `unpickle_game_state` function.

The solution is to add a version parameter to the functions supplied to `copyreg`.

```Python
class GameState(object):
    def __init__(self, level=0, points=0, magic=5):
        self.level = level
        self.points = points
        self.magic = magic


def pickle_game_state(game_state):
    kwargs = game_state.__dict__
    kwargs['version'] = 2
    return unpickle_game_state, (kwargs, )


def unpickle_game_state(kwargs):
    version = kwargs.pop('version', 1)
    if version == 1:
        kwargs.pop('lives')
    return GameState(**kwargs)
copyreg.pickle(GameState, pickle_game_state)
state_after = pickle.loads(serialized)
print(state_after.__dict__)

>>>
{'level': 0, 'points': 1000, 'magic': 5}
```
One other issue you may encounter with `pickle` is breakage from renaming a class. Often over the life cycle of a program, you'll refactor your code by renaming classes and moving them to other modules. Unfortunately, this will break the `pickle` module unless you're careful.

Here, I rename the `GameState` class to `BetterGameState`, removing the old class from the program entirely.

Attempting to deserialize an old `GameState` object will now fail because the class can't be found. The cause of this exception is that the import path of the serialized object's class is encoded in the pickled data.

The solution is to use `copyreg` again. You can specify a stable identifier for the function to use for unpickling an object. This allow you to transition pickled data to different classes with different names when it's deserialized.

```Python
copyreg.pickle(BetterGameState, pickle_game_state)
```

After using `copyreg`, you can see that the import path to `pickle_game_state` is encoded in the serialized data instead of `BetterGame`.

The only gotcha is that you can't change the path of the module in which the `unpickle_game_state` function is present. Once you serialize data with a function, it must remain available on that import path for deserializing in the future.

## Item 45: Use `datetime` Instead of `time` for Local Clocks
Coordinate Universal Time (UTC) is the standard, time-zone-independent representation of time. UTC works great for computers that represent time as seconds since the UNIX epoch. But UTC isn't ideal for humans. Humans reference time relative to where they're currently located. If your program handles time, you'll probably find yourself converting time between UTC and local clocks to make to easier for humans to understand.

Python provides two ways of accomplishing time zone conversions. The old way, using the `time` built-in module, is disastrously error prone. The new way, using the `datetime` built-in module, works great with some help from the community-built package named `pytz・

The `localtime` function from the `time` build-in module lets you convert a UNIX timestamp to a local time that matches the host computer's time zone.

```Python
from time import localtime, strftime

now = 1407694710
local_tuple = localtime(now)
time_format = '%Y-%m-%d %H:%M:%S'
time_str = strftime(time_format, local_tuple)
print(time_str)

>>>
2014-08-11 02:18:30
```

If you want to convert a local time into the UTC time, you can do this by using the `strptime` function to parse the time string, then call `mktime` to convert local time to a UNIX timestamp.

```Python
from time import mktime, strptime

time_tuple = strptime(time_str, time_format)
utc_now = mktime(time_tuple)
print(utc_now)

>>>
1407694710.0
```

How do you convert local time in one time zone to local time in another? Directly manipulating the return value from the `time`, `localtime`, and `strptime` functions to do time zone conversions is a bad idea. Time zones change all the time due to the local laws. Many operating systems have configuration files that keep up with the time zone changes automatically. Python lets you use these time zones through the `time` module.

```Python
parse_format = '%Y-%m-%d %H:%M:%S %Z'
depart_cst = '2014-05-01 15:45:16 CST'
time_tuple = strptime(depart_cst, parse_format)
time_str = strftime(time_format, time_tuple)
print(time_str)

>>>
2014-05-01 15:45:16
```

You might assume that other time zones known to to my computer will also work. Unfortunately, this isn't the case.

```Python
arrival_nyc = '2014-05-01 23:33:24 EDT'
time_tuple = strptime(arrival_nyc, time_format)

>>>
Traceback (most recent call last):
...
ValueError: unconverted data remains:  EDT
```

The problem is the platform-dependent nature of the `time` module. Its actual behavior is determined by how the underlying C functions work with the host operating system. This makes the functionality of the `time` module unreliable in Python. The `time` module fails to consistently work properly for multiple local times. Thus you should avoid the `time` module for this purpose. If you must use `time`, only use it to convert between UTC and the host computer's local time. For all other types of conversions, use the `datetime` module.

The second option for representing times in Python is the `datetime` class from the `datetime` built-in module. Like the `time` module, `datetime` can be used to convert from the current time in UTC  to local time.

```Python
from datetime import datetime, timezone


now = datetime(2014, 8, 10, 18, 18, 30)
now_utc = now.replace(tzinfo=timezone.utc)
now_local = now_utc.astimezone()
print(now_local)

>>>
2014-08-11 02:18:30+08:00
```

The `datetime` module can also easily convert a local time back to a UNIX timestamp in UTC.

```Python
time_format = '%Y-%m-%d %H:%M:%S'
time_str = '2014-08-10 11:18:30'
now = datetime.strptime(time_str, time_format)
time_tuple = now.timetuple()
utc_now = mktime(time_tuple)
print(utc_now)

>>>
1407640710.0
```

Unlike the `time` module, the `datetime` module has facilities for reliably converting from one local time to another local time. However, `datetime` only provides the machinery for time zone operations with its `tzinfo` class and related methods. What's missing are the time zone definitions besides UTC.

Luckily, the Python community has addressed this gap with the `pytz` module that's available for download from the Python Package Index. `pytz` contains a full databases of every time zone definition you might need.

To use `pytz` effectively, you should always convert local time to UTC first. Perform any `datetime` operations you need on the UTC values (such as offsetting). Then convert to local times as a final step.

```Python
arrival_nyc = '2014-05-01 23:33:24'
nyc_dt_naive = datetime.strptime(arrival_nyc, time_format)
eastern = pytz.timezone('US/Eastern')
nyc_dt = eastern.localize(nyc_dt_naive)
utc_dt = pytz.utc.normalize(nyc_dt.astimezone(pytz.utc))
print(utc_dt)

>>>
2014-05-02 03:33:24+00:00
```

Once I have a UTC `datetime`, I can convert it to San Francisco local time.

```Python
pacific = pytz.timezone('US/Pacific')
sf_dt = pacific.normalize(utc_dt.astimezone(pacific))
print(sf_dt)

>>>
2014-05-01 20:33:24-07:00
```

## Item 46: Use Built-in Algorithms and Data Structures

The Python standard library has many of the algorithms and data structures you'll need to use built in. Besides speed, using these common algorithms and data structures can make your life easier. Some of the most valuable tools you may want to use are tricky to implement correctly. Avoiding reimplementation of common functionality will save you time and headaches.

### Double-ended Queue
The `deque` class from the `collections` module is a double-ended queue. It provides constant time operations for inserting or removing items from its beginning or end. This makes it ideal for first-in-first-out (FIFO) queues.

```Python
from collections import deque

fifo = deque()
fifo.append(1)
x = fifo.popleft()
```

The `list` built-in type also contains an ordered sequence of items like a queue. You can insert or remove items from the end of a list in constant time. But inserting or removing items from the head of a list takes linear time, which is much slower than the constant time of a `deque`.

### Ordered Dictionary
Standard dictionary are unordered. That means a `dict` with the same keys and values can result in different orders of iteration. This behavior is s surprising byproduct of the way the dictionary's fast hash table is implemented.

The `OrderedDict` class from the `collections` module is a special type of dictionary that keeps track of the order in which its keys were inserted. Iterating the keys of an `OrderedDict` has predictable behavior.

```Python
from collections import OrderedDict

a = OrderedDict()
a['foo'] = 1
a['bar'] = 2

b = OrderedDict()
b['foo'] = 'red'
b['bar'] = 'blue'

for value1, value2 in zip(a.values(), b.values()):
    print(value1, value2)
    
>>>
1 red
2 blue
```

### Default Dictionary
Dictionaries are useful for bookkeeping and tracking statistics. One  problem with dictionaries is that you can't assume any keys are already present. The makes it clumsy to do simple things like increment a counter stored in a dictionary.

```Python
stats = {}
key = 'my_counter'

if key not in stats:
    stats[key] = 0
stats[key] += 1
```

The `defaultdict` class from the `collections` module simplifies this by automatically storing a default value when a key doesn't exist. All you have to do is provide a function that will return the default value each time a key is missing.

```Python
from collections import defaultdict

stats = defaultdict(int)
stats['my_counter'] += 1
```

### Heap Queue
Heaps are useful data structures for maintaining a priority queue. The `heapq` module provides functions for creating heaps in standard `list` type with functions like `heappush`, `heappop`, and `nsmallest・

Items of any priority can be inserted into the heap in any order.

```Python
from heapq import heappush, heappop, nsmallest

a = []
heappush(a, 5)
heappush(a, 3)
heappush(a, 7)
heappush(a, 4)
```

Items are always removed by highest priority (lowest number) first.

```Python
print(heappop(a), heappop(a), heappop(a), heappop(a))

>>>
3 4 5 7
```

The resulting `list` is easy to use outside of `heapq・ Accessing the 0 index of the heap will always return the smallest item.

```Python
heappush(a, 5)
heappush(a, 3)
heappush(a, 7)
heappush(a, 4)
assert a[0] == nsmallest(1, a)[0] == 3
```

Calling the `sort` method on the list maintains the heap invariant.

```Python
print('Before:', a)
a.sort()
print('After:', a)

>>>
Before: [3, 4, 7, 5]
After: [3, 4, 5, 7]
```

Each of these `heapq` operations takes logarithmic time in proportion to the length of the list. Doing the same work with a standard Python list would scale linearly.

### Bisection
Searching for an item in a `list` takes linear time proportional to its length when you call the `index` method.

The `bisect` module's functions, such as `bisect_left`, provide an efficient binary search through a sequence  of sorted of sorted items. The index it returns is the insertion point of the value into the sequence.

```Python
from bisect import bisect_left

x = list(range(100 ** 6))
i = bisect_left(x, 991234)
```

The complexity of a binary search is logarithmic. That means using `bisect` to search a list of           1 million items takes roughly the same amount of time as using `index` to linearly search a list of 14 items.

### Iterator Tools
The `itertools` built-in module contains a large number of functions that are useful for organizing and interacting with iterators. Not all of these are available in Python 2, but they can easily be built using simple recepes documented in the module.

The `itertools` functions fall into three main categories:

* Linking iterators together
    * `chain`: Combine multiple iterators into a single sequential iterator.
    * `cycle`: Repeats an iterator's items forever.
    * `tee`: Splits a single iterator into multiple parallel iterators.
    * `zip_longest`: A variant of the `zip` built-in function that works well with iterators of different lengths.
* Filtering items from an iterator
    * `islice`: Slices an iterator by numerical indexes without copying.
    * `takewhile`: Returns items from an iterator while a predicate function returns `True`.
    * `dropwhile`: Returns items from an iterator once the predicate function reuturns `False` for the first time.
    * `filterfalse`: Returns all items from an iterator where a predicate function returns `False`. The opposite of the `filte` built-in function.
* Combinations of items from iterators
    * `product`: Returns the Cartesian product of items from an iterator, which is a nice alternative to deeply nested list comprehension.
    * `permutations`: Returns ordered permutations of length N with items from an iterator.
    * `combination` Returns the unordered combinations of length N with unrepeated items from an iterator.

