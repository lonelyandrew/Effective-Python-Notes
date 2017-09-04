# Chapter 3: Classes and Inheritance

## Item 22: Prefer Helper Classes Over Bookkeeping with Dictionaries and Tuples
Avoid making dictionaries with values that are other dictionaries or long tuples.

Use `namedtuple` for lightweight, immutable data containers before you need the flexibility of a full class.

### Limitations of `namedtuple`:
+ You can not specify default argument values for `namedtuple` classes.
+ The attribute values of `namedtuple` instances are still accessible using numerical indexes and iteration. If you move to a real class later, then it is hard to keep the backward compatibility.

Move your bookkeeping code to use multiple helper classes when your internal state dictionaries get complicated.

## Item 23: Accept Functions for Simple Interfaces Instead of Classes
Instead of defining and instantiating classes, functions are often all you need for simple interfaces between components in Python.

References to functions and methods in Python are first class, meaning they can be used in expressions like any other type.

The `__call__` special method enables instances of a class to be called like plain Python functions.

When you need a function to maintain state, consider defining a class that provides the `__call__` method instead of defining a stateful closure.

## Item 24: Use `@classmethod` Polymorphism to Construct Objects Generically
Python only supports a single constructor per class, the `__init__` method.

Use `@classmethod` to define alternative constructors for your classes.

Use class method polymorphism to provide generic ways to build and connect subclasses.


