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

