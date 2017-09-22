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






