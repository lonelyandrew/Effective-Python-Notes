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

## Item 26: Use Multiple Inheritance Only for Mix-in Utility Classes
If you find yourself desiring the convenience and encapsulation that comes with multiple inheritance, consider writing a **mix-in** instead.

A mix-in is a small class that only defines a set of additional methods that a class should provide. Mix-in classes don't define their own instance attributes nor their `__init__` constructor to be called.

Here, I define an example mix-in that accomplishes this with a new public method that's added to any class that inherits from it:

```Python
class ToDictMixin(object):
    def to_dict(self):
        return self._traverse_dict(self.__dict__)

    def _traverse_dict(self, instance_dict):
        output = {}
        for key, value in instance_dict.items():
            output[key] = self._traverse(key, value)
        return output

    def _traverse(self, key, value):
        if isinstance(value, ToDictMixin):
            return value.to_dict()
        elif isinstance(value, dict):
            return self._traverse_dict(value)
        elif isinstance(value, list):
            return [self._traverse(key, i) for i in value]
        elif hasattr(value, '__dict__'):
            return self._traverse_dict(value.__dict__)
        else:
            return value


class BinaryTree(ToDictMixin):
    def __init__(self, value, *, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right


if __name__ == '__main__':
    tree = BinaryTree(10,
                      left=BinaryTree(7, right=BinaryTree(9)),
                      right=BinaryTree(13, left=BinaryTree(11)))
    from pprint import pprint
    pprint(tree.to_dict())

>>>
{'left': {'left': None,
          'right': {'left': None, 'right': None, 'value': 9},
          'value': 7},
 'right': {'left': {'left': None, 'right': None, 'value': 11},
           'right': None,
           'value': 13},
 'value': 10}
```

## Item 27: Prefer Public Attributes Over Private Ones
In Python, there are only two types of attribute visibility for a class's attributes: *public* and *private*.

Public attributes can be accessed by anyone using the dot operator on the object. Private field are specified by prefixing an attribute's name with a  double underscore. They can be accessed directly by methods of the containing class.

Class methods also have access to private attributes because they are declared within the surrounding `class` block.

```Python
@classmethod
def get_private_field_of_instance(cls, instance):
    return instance.__private_field
```

A subclass can't access its parent class's private fields.

The private attribute behavior is implemented with a simple transformation of the attribute name. When the Python compiler sees private attribute access in methods like `MyChildObject.get_private_field`, it translates `__private_field` to access `_MyChildObject__private_field` instead. Accessing the parent's private attribute from the child class fails simply because the transformed attributes name doesn't match.

Why doesn't the syntax for private attributes actually enforce strict visibility? The simplest answer is one often-quoted motto of Python:

> We are all consenting adults here.

Python programmers believe that the benefits of being open outweigh the downsides of being closed.

To minimize the damage of accessing internals unknowingly, Python programmers follow a naming convention defined in the style guide. Field prefixed by a single underscore are *protected*, meaning external users of the class should proceed with caution.

In general, it's better to err on the side of allowing subclasses to do more by using protected attributes. Document each protected field and explain which are internal APIs available to subclasses and which should be left alone entirely.

Only consider using private attributes to avoid naming conflicts with subclasses that are out of your control.


