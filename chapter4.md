# Chapter 4: Metaclasses and Attributes
The name of *metaclasses* vaguely implies a concept above and beyond a class. Simply put, metaclasses let you intercept Python's `class` statement and provide special behavior each time a class is defined.

It's important that you follow the *rule of least surprise* and only use the these mechanisms to implement well-understood idioms.

## Item 29: Use Plain Attributes Instead of Get and Set Methods
Define new class interfaces using simple public attributes, and avoid set and get methods.

Use `@property` to define special behavior when attributes are accessed on your objects, if necessary.

```Python
class MyClass(object):
    def __init__(self, x):
        self.x = x

    @property
    def x(self):
        return self.__x

    @x.setter
    def x(self, x):
        self.__x = x

if __name__ == '__main__':
    myclass = MyClass(10)
    print(myclass.x)
    myclass.x = 20
    print(myclass.x)

myclass = MyClass(10)
print(myclass.x)
myclass.x = 20
print(myclass.x)

>>>
10
>>>
20
```

Follow the rule of least surprise and avoid weird side effects in your `@property` methods.
For example, don't set other attributes in getter property methods. The best policy is to only modify related object state in `@property.setter` methods.

Ensure that `@property` methods are fast; do slow or complex work using normal methods.

## Item 30: Consider `@property` Instead of Refactoring Attributes
User `@property` to give existing instance attributes new functionality.

Make incremental progress toward better data models by using `@property`.

Consider refactoring a class and all call sites when you find yourself using `@property` too heavily.

## Item 31: Use Descriptors for Reusable `@property` Methods
The big problem with the `@property` built-in is reuse. The methods it decorates can't be reused for multiple attributes of the same class. They also can't be reused by unrelated classes.

For example, say you want a class to validate that the grade received by a student on a homework assignment is a percentage.

If you can implement it by the `@property`, but a better way to do this in Python is to use a *descriptor*. The descriptor protocol defines how attributes access is interpreted by the language. A descriptor class can provide `__get__` and `__set__` methods that let you reuse the   
grade validation behavior without any boilerplate. For this purpose, descriptors are also better than mix-ins, because they let you reuse the same logic for many different attributes in a single class.

```Python
class Grade(object):
    def __init__(self):
        self._value = 0

    def __get__(self, instance, instance_type):
        return self._value

    def __set__(self, instance, value):
        if not (0 <= value <= 100):
            raise ValueError('Grade must be between 0 and 100')
        self._value = value


class Exam(object):
    # Class attributes
    math_grade = Grade()
    writing_grade = Grade()
    science_grade = Grade()


if __name__ == '__main__':
    first_exam = Exam()
    first_exam.writing_grade = 82
    first_exam.science_grade = 99

    print(f'Writing:{first_exam.writing_grade}')
    print(f'Science:{first_exam.science_grade}')
    
>>>
Writing:82
Science:99
```

When you assign a property:

```Python
first_exam = Exam()
first_exam.writing_grade = 82
```

it will be interpreted as:

```Python
Exam.__dict__['writing_grade'].__set__(exam, 82)
```

When you receive a property:

```Python
print(f'Writing:{first_exam.writing_grade}')
```

it will be interpreted as:

```Python
print(f"Writing:{Exam.__dict__['writing_grade'].__get__(exam, Exam)}")
```

What drives this behavior is the `__getattribute__` method of `object`. In short, when an `Exam` instance doesn't have an attribute named `wrting_grade`, Python will fall back to the `Exam` class's attribute instead. If this class attribute is an object that has `__get__` and `__set__` methods, Python will assume you want to follow the descriptor protocol.

But last implementation is wrong and will result in broken behavior. Accessing multiple attributes on a single `Exam` instance works as expected, but accessing these attributes on multiple `Exam` instances will have unexpected behavior.

```Python
second_exam = Exam()
second_exam.writing_grade = 75

print(f'Second {second_exam.writing_grade} is right')
print(f'First {first_exam.writing_grade} is wrong')

>>>
Second 75 is right
First 75 is wrong
```

The problem is that a single `Grade` instance is shared across all `Exam` instances for the class attribute `writing_grade`. The `Grade` instance for this attribute is constructed once in the program lifetime when the `Exam` class is first defined, not each time an `Exam` instance is created.

```Python
class Grade(object):
    def __init__(self):
        self._values = {}

    def __get__(self, instance, instance_type):
        if instance is None:
            return self
        return self._values.get(instance, 0)

    def __set__(self, instance, value):
        if not (0 <= value <= 100):
            raise ValueError('Grade must be between 0 and 100')
        self._values[instance] = value

>>>
Writing:82
Science:99
Second 75 is right
First 82 is right
```

This implementation is simple and works well, but there's still one gotcha: It leaks memory. The `_values` dictionary will hold a reference to every instance of `Exam` ever passed to `__set__` over the lifetime of the program. This cause instances to never have their reference count go to zero, preventing cleanup by the garbage collector.

To fix this, I can use Python's `weakref` built-in module. This module provides a special class called `WeakKeyDictionary` that can take the place of the simple dictionary used for `_values`. The unique behavior of `WeakKeyDictionary` is that it will remove exam instances from its set of keys when the runtime knows it's holding the instance's last remaining reference in the program. Python will do the bookkeeping for you and ensure that the `_values` dictionary will be empty when all `Exam` instances are no longer in use.

```Python
from weakref import WeakKeyDictionary
class Grade(object):
    def __init__(self):
        self._values = WeakKeyDictionary()
    
    # ...
```

Don't get bogged down trying to understand exactly how `__getattribute__` attributes uses the descriptor protocol for getting and setting attributes.

P.S. [Descriptor HowTo Guide](https://docs.python.org/3/howto/descriptor.html)

## Item 32: Use `__getattr__`, `__getattribute__`, and `__setattr__` for Lazy Attributes
If your class defines `__getattr__`, that method is called every time an attribute can't be found in an object's instance dictionary. But the `__getattribute__` method is called every time an attribute is accessed on an object, even in cases where is *does* exist in the attribute dictionary.

Python code implementing generic functionality often relies on the `hasattr` built-in function to determine when properties exist, and the `getattr` built-in function to retrieve property values. These functions also hook in the instance dictionary for an attribute name before calling `__getattr__`.

`__setattr__` is a similar language hook that lets you intercept arbitary attribute assignments. Unlike retrieving an attribute with `__getattr__` and `__getattribute__`, there is no need for two separate methods. The `__setattr__` is always called every time an attribute is assigned on an instance (either directly or through the `setattr` built-in function).

The problem with `__getattribute__` and `__setattr__` is that they are called on every attribute access for an object, even when you may not want that to happen.

For example, say you want attribute accesses on your object to actually look up keys in an associated dictionary.

```Python
class BrokenDictionaryDB(object):
    def __init__(self, data):
        self._data = data

    def __getattribute__(self, name):
        print(f'Called __getattribute__({name})')
        return self._data[name]


if __name__ == '__main__':
    data = BrokenDictionaryDB({'foo': 3})
    data.foo

>>>
Called __getattribute__(foo)
Called __getattribute__(_data)
Called __getattribute__(_data)
Called __getattribute__(_data)
...
Traceback (most recent call last):
...
RecursionError: maximum recursion depth exceeded while calling a Python object
```

The problem is that `__getattribute__` accesses `self.data`, which causes `__getattribute__` to run again, which accesses `self.data` again, and so on.

The solution is to use the `super().__getattribute__` method on your instance to fetch values from the instance attribute dictionary. This avoid recursion.

```Python
class BrokenDictionaryDB(object):
    def __init__(self, data):
        self._data = data

    def __getattribute__(self, name):
        print(f'Called __getattribute__({name})')
        data_dict = super().__getattribute__('_data')
        return data_dict[name]


if __name__ == '__main__':
    data = BrokenDictionaryDB({'foo': 3})
    print(data.foo)

>>>
Called __getattribute__(foo)
3
```

## Item 33: Validate Subclasses with Metaclasses

A metaclas is defined by inheriting from `type`. In the default case, a metaclass reveives the contents of associated `class` statements in its `__new__` method. Here, you can modify the class information before the type is actually constructed.

```Python
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        print((meta, name, bases, class_dict))
        return type.__new__(meta, name, bases, class_dict)


# Python 3
class MyClass(object, metaclass=Meta):
    stuff = 123

    def foo(self):
        pass

# Python 2
class MyClassInPython2(object):
    __metaclass__ = Meta
    stuff = 123

    def foo(self):
        pass

if __name__ == '__main__':
    pass

> python3 test3.py

(<class '__main__.Meta'>, 'MyClass', (<class 'object'>,), {'__module__': '__main__', '__qualname__': 'MyClass', 'stuff': 123, 'foo': <function MyClass.foo at 0x100d87840>})

> python test3.py

(<class '__main__.Meta'>, 'MyClassInPython2', (<type 'object'>,), {'__module__': '__main__', 'stuff': 123, '__metaclass__': <class '__main__.Meta'>, 'foo': <function foo at 0x1010712a8>})
```

The `__new__` method of metaclasses is run after the `class` statement's entire body has been processed.

You can add functionality to the `Meta.__new__` method in order to validate all of the parameters of a class before it's defined.

For example, say you want to represent any type of multisided polygon. You can do this by defining a special validating metaclass and using it in the base class of your polygon class hierarchy. Note that it's important not to apply the same validation to the base class.

```Python
class ValidationPolygon(type):
    def __new__(meta, name, bases, class_dict):
        if bases != (object,):
            if class_dict['sides'] < 3:
                raise ValueError('Polygons need 3+ sides')
        return type.__new__(meta, name, bases, class_dict)


class Polygon(object, metaclass=ValidationPolygon):
    sides = None

    @classmethod
    def interior_angles(cls):
        return (cls.sides - 2) * 180


class Triangle(Polygon):
    sides = 3



print('Before class')
class Line(Polygon):
    print('Before sides')
    sides = 1
    print('After sides')
print('After class')


if __name__ == '__main__':
    pass
    
>>>
Before class
Before sides
After sides
Traceback (most recent call last):
  File "test3.py", line 23, in <module>
    class Line(Polygon):
  File "test3.py", line 5, in __new__
    raise ValueError('Polygons need 3+ sides')
ValueError: Polygons need 3+ sides
```

## Item 34: Register Class Existence with Metaclasses
Class registration is a helpful pattern for building modular Python programs.

Metaclasses let you run registration code automatically each time your base class is subclassed in a program.

Using metaclasses for class registration avoids errors by ensuring that you never miss a registration call.

## Item 35: Annotate Classes Attributes with Metaclasses
Metaclasses enable you to modify a class's attributes before the class is fully defined.

Descriptors and metaclasses make a powerful combination for declarative behavior and runtime introspection.

You can avoid both memory leaks and the `weakref` module by using metaclasses along with descriptors.

(END OF CHAPTER 4)

