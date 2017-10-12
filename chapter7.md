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


