# Chapter 2: Functions

## Item 14: Prefer Exceptions to Returning `None`

Functions that return `None` to indicate special meaning are error prone because `None` and other values all evaluate to `False` in conditional expressions.

Raise exceptions to indicate special situations instead of returning `None`. Except the calling code to handle exceptions properly when they are documented.

P.S.

Some values are evaluated to `False` in conditional expressions, such as:
+ The Boolean value `False` itself
+ Any numerical value equal to 0 (0, 0.0 but not 2 or -3.1)
+ The special value `None`
+ Any empty sequence or collection, including the empty string(`''`, but not `'0'` or `'hi'` or `'False'`) and the empty list (`[]`, but not `[1,2, 3]` or `[0]`)

Reference: [3.4. Arbitrary Types Treated As Boolean](http://anh.cs.luc.edu/python/hands-on/3.1/handsonHtml/boolean.html)

## Item 15: Know How Closures Interact with Variable Scope

When you reference a variable in an expression, the Python interpreter will traverse the scope to resolve the reference in this order:
1. The current function's scope
2. Any enclosing scopes
3. The scope of the module that contains the code (also called the *global* scope)
4. The built-in scope

In Python 3, use the `nonlocal` statement to indicate when a closure can modify a variable in its enclosing scopes. It's complementary to the `global` statement, which indicates that a variable's assignment should go directly into
the module scope.

In Python 2, use a mutable value (like a single-item list) to work around the lack of the `nonlocal` statement.

Avoid using `nonlocal` statements for anything beyond simple functions.

## Item 16: Consider Generators Instead of Returning Lists

Using generators can be clearer than the alternative of returning lists of accumulated results.

The iterator returned by a generator produces the set of values passed to `yield` expressions within the generator function's body.

Generators can produce a sequence of outputs for arbitrarily large inputs because their working memory doesn't include all inputs and outputs.

## Item 17: Be Defensive When Iterating Over Arguments

Beware of functions that iterate over input arguments multiple times. If these arguments are iterators, you may see strange behavior and missing values.

Python's iterator protocol defines how containers and iterators interact with the `iter` and `next` built-in functions, `for` loops, and related expressions.

You can easily define your own iterable container type by implementing the `__iter__` method as a generator.

You can detect that a value is an iterator (instead of a container) if calling `iter` on it twice produces the same result, `iter(x) is iter(x)`, which can then be progressed with the next built-in function.

## Item 18: Reduce Visual Noise with Variable Positional Arguments

A value passed to a function (or method) when calling the function. There are two kinds of argument:
+ *keyword argument*: an argument preceded by an identifier (e.g. `name=`) in a function call or passed as a value in a dictionary preceded by `**`.
For example, 3 and 5 are both keyword arguments in the following calls to `complex()`:
```python
complex(real=3, imag=5)
complex(**{'real': 3, 'imag': 5})
```
+ *positional argument*: an argument that is not a keyword argument. Positional arguments can appear at the beginning of an argument list and/or be passed as elements of an iterable preceded by `*`. For example, 3 and 5 are both positional arguments in the following calls:
```python
complex(3, 5)
complex(*(3, 5))
```

Accepting optional positional arguments (often called *star args* in reference to the conventional name for the parameter, `*args`) can make a function call more clear and remove visual noise.

```python
def log(message, *values):
    if not values:
        print(message)
    else:
        values_str = ', '.join(str(x) for x in values)
        print(f'{message}: {values_str}')
```

The first issue is that variable arguments are always turned into a tuple before they are passed to your function. This means that if the caller of your function use the `*` operator on a generator, it will be iterated
until it's exhausted. Functions that accept `*args` are best for situations where you know the number of inputs in the argument list will be reasonably small.

The second issue with `*args` is that you can't add new positional arguments to your function in the future without migrating every caller. To avoid this possibility entirely, you should use keyword-only arguments when you want to extend functions that accept `*args`.

## Item 19: Provide Optional Behavior with Keyword Arguments
The flexibility of keyword arguments provides three significant benefits.

The first advantage is that keyword arguments make the function call clearer to new readers of the code.

The second impact of keyword arguments is that they can have default values specified in the function definition. This can eliminate repetitive code and reduce noise.

The third reason to use keyword arguments is that they provide a powerful way to extend a function's parameters while remaining backwards compatible with existing callers.

## Item 20: Use `None` and Docstrings to Specify Dynamic Default Arguments
Default argument values are evaluated only once per module load, which usually happens when a program starts up.This can cause odd behaviors for dynamic values (like `{}` or `or`).

Use `None` as the default value for keyword arguments that have a dynamic value. Document the actual default behavior in the function's docstring.

```python
def log(message, when=None):
    '''Log a message with a timestamp.
    
    Args:
        message: Message to print.
        when: datetime of when the message occured.
            Defaults to the present time.
    '''
    when = datetime.now() if when is None else when
    print(f'{when}: {message}')
```


