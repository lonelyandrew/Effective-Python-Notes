# Chapter 1: Pythonic Thinking

## Item 1: Know Which Version of Python You're Using

To find out exactly which version of Python you're using, you can use the `--version` flag.

```bash
$ python --version
Python 2.7.10
```

Python 3 is usually available under the name `python3`.

```bash
$ python3 --version
Python 3.6.2
```

You can also figure out the version of Python you're using at runtime by inspecting values in the `sys` built-in module.

```
>>> import sys
>>> print(sys.version_info)
sys.version_info(major=2, minor=7, micro=10, releaselevel='final', serial=0)
>>> print(sys.version)
2.7.10 (default, Feb  7 2017, 00:08:15)
[GCC 4.2.1 Compatible Apple LLVM 8.0.0 (clang-800.0.34)]
```

There are multiple popular runtimes for Python: CPython, JPython, IronPython, PyPy, etc.

## Item 2: Follow the PEP 8 Style Guide

Python Enhancement Proposal #8, otherwise known as PEP 8, is the style guide for how to format Python code.

Here are a few rules you should be sure to follow:

**Whitespace**: In Python, whitespace is syntactically significant. Python programmers are especially sensitive to the effects of whitespace on code clarity.

+ Use spaces instead of tabs for indentation.
+ Use **four spaces** for each level of syntactically significant indenting.
+ Lines should be **79** characters in length or less.
+ Continuations of long expressions onto additional lines should be intended by four extra spaces from their normal indentation level.
+ In a file, functions and classes should be separated by **two** blank lines.
+ In a class, methods should be separated by **one** blank line.
+ Don't put spaces around list indexes, function calls, or keyword argument assignments.
+ Put one - and only one - space before and after variable assignment.

**Naming**: PEP 8 suggests unique styles of naming for different parts in the language. This makes it easy to distinguish which type corresponds to each name when reading code.

+ Functions, variables, and attributes should be in `lowercase_underscore` format.
+ Protected instance attributes should be in `_leading_underscore` format.
+ Private instance attributes should be in `__double_leading_underscore` format.
+ Classes and exceptions should be in `CapitalizedWord` format.
+ Module-level constants should be in `ALL_CAPS` format.
+ Instance methods in the classes should use `self` as the name of the first parameter (which refers to the object).
+ Class methods should use `cls` as the name of the first parameter (which refers to the class).

**Expressiona and Statements**: *The Zen of Python* states: "There should be one --- and preferably only one --- obvious way to do it."

+ Use inline negation (`if a is not b`) instead of negation of positive expressions (`if not a is b`).
+ Don't check for empty values (like `[]` or `''`) by checking the length (`if len(somelist) == 0`). Use `if not somelist` and assume empty values implicitly evaluate to `False`.
+ The same thing goes for non-empty values (like `[1]` or 'hi'). The statement `if somelist` is implicitly `True` for non-empty values.
+ Avoid single-line `if` statement, `for` and `while` loops, and `except` compound statements. Spread these over multiple lines for clarity.
+ Always put `import` statements at the top of a file.
+ Always use absolute names for modules when importing them, not names relative to the current module's own path. For example, to import the `foo` module from the `bar` package, you should do `from bar import foo`, not just `import foo`.
+ If you must do relative imports, use the explicit syntax `from . import foo`.
+ Imports should be in sections in the following order: standard library modules, third-party modules, your own modules. Each subsection should have imports in alphabetical order.

P.S.
+ If you have not got a detailed Python style guide, a recommendation is [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html).
+ A better way to check the code style is to install a lint tool in your editor (also mentioned in the Google Python Style Guide), such as `Pylint` and `flake8`.

## Item 3: Know the Differences Between `bytes`, `str`, and `unicode`

In Python 3, there are two types that represent sequences of characters: `bytes` and `str`. Instances of `bytes` contains raw 8-bit values. Instances of `str` contain Unicode characters.

In Python 2, there are two types that represent sequences of characters: `str` and `unicode`. Instances of `str` contains raw 8-bit values. Instances of `unicode` contain Unicode characters.

`str` instances in Python 3 and `unicode` instances in Python 2 do not have an associated binary encoding. To convert Unicode character to binary data, you must use the `encode` method.
To convert binary data to Unicode characters, you must use the `decode` method.

The core of your program should use Unicode character types (`str` in Python 3, `unicode` in Python 2) and should not assume anything about character encodings.

In Python 3, operations involving file handles defaults to UTF-8 encoding. In Python 2, file operations default to binary encoding.

## Item 4: Write Helper Functions Instead of Complex Expressions

Python's syntax makes it all too easy to write single-line expressions that are overly complicated and difficult to read.

Move complex expressions into helper functions, especially if you need to use the same logic repeatedly.

The `if/else` expression provides a more readable alternative to using Boolean operators like `or` and `and` in expressions.

## Item 5: Know How to Slice Sequences

Slicing can be extended to any Python class that implements the `__getitem__` and `__setitem__` special methods (see Item 28: "Inherit from `collections.abc` for Custom Container Types").

When `n` is zero, the expression `somelist[-0:]` will result in a copy the original list.

## Item 6: Avoid Using `start`, `end`, and `stride` in a Single Slice

Specifying `start`, `end`, and `stride` in a slice can be extremely confusing.

Prefer using positive `stride` values in slices without `start` or `end` indexes. Avoid negative `stride` values if possible.

Avoid using `start`, `end`, and `stride` together in a single slice. If you need all three parameters, consider doing two assignments (one to slice, another to stride) or using `islice` from the `itertools` built-in module.
