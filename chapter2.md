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
