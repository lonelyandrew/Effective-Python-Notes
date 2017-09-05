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

## Item 25: Initialize Parent Classes with `super`

Diamond inheritance happens when a subclass inherits from two separate classes that have the same superclass somewhere in the hierarchy. Diamond inheritance causes the common superclass's `__init__` method to run multiple times, causing unexpected behavior.

```Python
# Python 2
class MyBaseClass(object):
    def __init__(self, value):
        self.value = value

class TimesFive(MyBaseClass):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        self.value *= 5

class PlusTwo(MyBaseClass):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        self.value += 2

class ThisWay(TimesFive, PlusTwo):
    def __init__(self, value):
        TimesFive.__init__(self, value)
        PlusTwo.__init__(self, value)  # call the super class's __init__ again
        
foo = ThisWay(5)
print('Should be (5 * 5) + 2 = 27 but is ', foo.value)
>>>
('Should be (5 * 5) + 2 = 27 but is ', 7)
```

To solve this problem, Python 2.2 added the `super` built-in function and defined the method resolution order (**MRO**). The MRO standardizes which superclasses are initialized before others (e.g. depth-first, left-to-right). It also ensures that common superclasses in diamond hierarchies are only run once.

```Python
class TimesFiveCorrect(MyBaseClass):
    def __init__(self, value):
        super(TimesFiveCorrect, self).__init__(value)
        self.value *= 5

class PlusTwoCorrect(MyBaseClass):
    def __init__(self, value):
        super(PlusTwoCorrect, self).__init__(value)
        self.value += 2

class GoodWay(TimesFiveCorrect, PlusTwoCorrect):
    def __init__(self, value):
        super(GoodWay, self).__init__(value)
    
foo = GoodWay(5)
print('Should be 5 * (5 + 2) = 35 but is ', foo.value)

from pprint import pprint
pprint(GoodWay.mro())

>>>
('Should be 5 * (5 + 2) = 35 but is ', 35)
>>>
[<class '__main__.GoodWay'>,
 <class '__main__.TimesFiveCorrect'>,
 <class '__main__.PlusTwoCorrect'>,
 <class '__main__.MyBaseClass'>,
 <type 'object'>]
```

In Python 3, the call of superclass's `__init__` become more clear.

```Python
# Python 3

class MyBaseClass(object):
    def __init__(self, value):
        self.value = value

class Explicit(MyBaseClass):
    def __init__(self, value):
        super(__class__, self).__init__(value * 2)

class Implicit(MyBaseClass):
    def __init__(self, value):
        super().__init__(value * 2)
```

You may guess that you could use `self.__class__` as an argument to `super`, but this breaks because of the way `super` is implemented in Python 2.

