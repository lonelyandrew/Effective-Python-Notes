# Chapter 3: Classes and Inheritance

## Item 22: Prefer Helper Classes Over Bookkeeping with Dictionaries and Tuples
Avoid making dictionaries with values that are other dictionaries or long tuples.

Use `namedtuple` for lightweight, immutable data containers before you need the flexibility of a full class.

### Limitations of `namedtuple`:
+ You can not specify default argument values for `namedtuple` classes.
+ The attribute values of `namedtuple` instances are still accessible using numerical indexes and iteration. If you move to a real class later, then it is hard to keep the backward compatibility.

Move your bookkeeping code to use multiple helper classes when your internal state dictionaries get complicated.



