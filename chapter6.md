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




