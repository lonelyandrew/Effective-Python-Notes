# Chapter 7: Collaboration

## Item 49: Write Docstrings for Every Function, Class, and Module

Documentation in Python is extremely important because of the dynamic nature of the language. Python provides built-in support for attaching documentation to blocks of code. Unlike many other languages, the documentation from a program's source code is directly accessible as the program runs.

```Python
def palindrome(word):
    '''Return True if the given word is a palindrome.'''
    return word == word[::-1]

print(repr(palindrome.__doc__))

>>>
'Return True if the given word is a palindrome.'
```

Docstrings can be attached to functions, classes, and modules. This connection is part of the process of compiling and running a Python program. Support for docstrings and the `__doc__` attribute has three consequences:

+ The accessibility of documentation makes interactive development easier. You can inspect functions, classes, and modules to their documentation by using the `help` built-in function. This makes the Python interactive interpreter (the Python shell) and tools like IPython Notebook a joy to use while you're developing algorithms, testing APIs, and writing code snippets.
+ A standard way of defining documentation makes it easy to build tools that convert the text into more appealing formats (like HTML). This has led to excellent documentation-generation tools for the Python community, such as Sphinx. It's also enabled community-funded sites like Read the Docs that provide free hosting of beautiful-looking documentation for open source Python projects.
+ Python's first class, accessible, and good-looking documentation encourages people to write more documentation. The member of the Python community have a strong belief in the importance of documentation. There's an assumption that good code also means well-documented code. This means that you can expect most open source Python libraries to have decent documentation.

To participate in this excellent culture of documentation, you need to follow a few guidelines when you write docstrings. The full details are discussed in PEP 257. There are a few best-practices you should be sure to follow.

### Documenting Modules
Each module should have a top-level docstring. This is a string literal that is the first statement in a source file. The goal of this docstring is to introduce the module and its contents.

The first line of the docstring should be a single sentence describing the module's purpose. The paragraphs that follow should contain the details that all users of the module should know about its operation. The module docstring is also a jumping-off point where you can highlight important classes and functions found in the module.

If the module is a command-line utility, the module docstring is also a great place to put usage information for running the tool from the command-line.

### Documenting Classes

Each class should have a class-level docstring. This largely follows the same pattern as the module-level docstring. The first line is the single-sentence purpose of the class. Paragraphs that follow discuss important details of the class's operation.

Important public attributes and methods of the class should be highlighted in the class-level docstring. It should also provide guidance to subclasses on how to properly interact with protected attributes and the superclass's methods.

### Documenting Functions

Each public function and method should have a docstring. This follows the same pattern as modules and classes. The first line is the single-sentence description of what the function does. The paragraphs that follow should describe any specific behaviors and the arguments for the function. Any return values should be mentioned. Any exceptions that callers must handle as part of the function's interface should be explained.

## Item 50: Use Packages to Organize Modules and Provide Stable APIs

At some point, you'll find yourself with so many modules that you need another layer in your program to make it understandable. For this purpose, Python provides packages. Packages are modules that contain other modules.

In most cases, packages are defined by putting an empty file named `__init__.py` into a directory. Once `__init__.py` is present, any other Python files in that directory will be available for import using a path relative to the directory.

The first use of packages is to help divide your modules into separate namespaces. This allow you to have many modules with the same filename but different absolute paths that are unique.

For example, here's a program that imports attributes from two modules with the same name, `utils.py`. This works because the modules can be addressed by their absolute paths.

```python
# main.py
from analysis.utils import log_base2_bucket
from frontend.utils import stringify
```

This approach breaks down when the functions, classes, or submodules defined in packages have the same names. 

```python
# main2.py
from analysis.utils import inspect
from frontend.utils import inspect # Overwrites!
```

The solution is to use the `as` clause of the `import` statement to rename whatever you've imported for the current scope.

```python
# main3.py
from analysis.utils import inspect as analysis_inspect
from frontend.utils import inspect as frontend_inspect
```

Another approach for avoiding imported name conflicts is to always access names by their highest unique module name. For the example above, you'd first access the `inspect` functions with the full paths of `analysis.utils.inspect` and `frontend.utils.inspect`.

This approach allows you to avoid the `as` clause altogether. It also makes it abundantly clear to new readers of the code where each function is defined.

The second use of packages in Python is to provide strict, stable APIs for external consumers.

Python can limit the surface area exposed to API consumer by using the `__all__` special attribute of a module or package. The value of `__all__` is a list of every name to export from the module as part of its public API. When consuming code does `from foo import *`, only the attributes in `foo.__all__` will be imported from `foo`. If `__all__` isn't present in `foo`, they only public attributes, those without a leading underscore, are imported.

Import statements like `from x import y` are clear because the source of `y` is explicitly the `x` package or module. Wildcard imports like `from foo import *` can also be useful, especially in interactive Python sessions. However, wildcards makes code more difficult to understand.

`from foo import *` hides the source of names from new readers of the code. If a module has multiple `import *` statements, you'll need to check all of the referenced modules to figure out where a name was defined.

Names from `import *` statements will overwrite any conflicting names within the containing module. This can lead to strange bugs caused by accidental interactions between your code and overlapping names from multiple `import *` statements.

The safest approach is to avoid `import *` in your code and explicitly imports names with the `from x import y` style.

