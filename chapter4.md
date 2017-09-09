a# Chapter 4: Metaclasses and Attributes
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


