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

