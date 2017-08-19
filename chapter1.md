# Effective Python
---

Author: **Brett Slatkin**

[TOC]

## Chapter 1: Pythonic Thinking

### Item 1: Know Which Version of Python You're Using

To find out exactly which version of Python you're using, you can use the `--version` flag.

```bash
$ python --version
Python 2.7.10
```

Python3 is usually available under the name `python3`.

```bash
$ python3 --version
Python 3.6.2
```

You can also figure out the version of Python you're using at runtime by inspecting values in the `sys` built-in module.

```python
>>> import sys
>>> print(sys.version_info)
sys.version_info(major=2, minor=7, micro=10, releaselevel='final', serial=0)
>>> print(sys.version)
2.7.10 (default, Feb  7 2017, 00:08:15)
[GCC 4.2.1 Compatible Apple LLVM 8.0.0 (clang-800.0.34)]
```

There are multiple popular runtimes for Python: CPython, JPython, IronPython, PyPy, etc.

### Item 2: Follow the PEP 8 Style Guide

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
