+++
title = 'Greater Numbers, From Ground Up'
date = 2024-02-05T11:10:54+03:00
description = 'Calculating some really big numbers, from ground up'
+++

# Note

The conception I'll talk about is called [hyperoperation](https://en.wikipedia.org/wiki/Hyperoperation), nothing new here will be discussed,
if you are familar with hyperoperation, it may be not that interesting.

For demonstration I'll use `Python` programming language since learning LaTeX just for this article seems irrational.

I am using the word "order" to express "grade" of operation (as it's called on wikipedia).

---

# Natural numbers

Let's invent our own numbers, natural numbers. For that purpose we need exactly two things:

- `Z` - zero (identity)
- Successor function - `succ(x)`

For every natural number there are successor function, so

- `s(z)` = 1
- `s(s(z))` = 2
- `s(s(s(z)))` = 3
- ...

```python
from __future__ import annotations
from typing import cast
from abc import ABCMeta, abstractmethod

# Nat - Natural
class Nat(metaclass=ABCMeta):
  @abstractmethod
  def __repr__(self) -> str:
    raise NotImplementedError

  # Convert our `Nat` to `int`
  @abstractmethod
  def __int__(self) -> int:
    raise NotImplementedError

  # for every natural number successor function is defined
  @property
  def succ(self) -> S:
    return S(self)

class _Z(Nat):
  def __repr__(self) -> str:
    return "Z"

  def __int__(self) -> int:
    return 0
Z = _Z()

class S(Nat):
  def __init__(self, of: Nat) -> None:
    self.of = of

  def __repr__(self) -> str:
    return f"S({self.of!r})"

  def __int__(self) -> int:
    return 1 + int(self.of)

```

Numbers are nothing without operations, since our goal is to increase the numbers, we'll define only increasing functions.

The easiest one is addition

```python
def add(lhs: Nat, rhs: Nat) -> Nat:
  raise NotImplemented
```

How should it work? Addition is something that "steals" successor function application from left or right hand side, for example:
- 2 + 2 = 4. `s(s(z)) + s(s(z)) = s(s(s(s(z))))`
- 1 + 0 = 1. `s(z) + z = s(z)` since `z` has zero applications of successor function

Our exact algorithm is:
- for `add(x, z)` return x, since `z` has zero applications of successor function
- for `add(x, s(y))` we return `s(add(x, y))`. What is the `add(x, s(y))` notation? Since we already defined addition for zero right hand side,
remaining case is successor function application to some value, we just unwrap that application.

Let's add `2` to `2` as an example:
- `s(s(z)) = 2`

1. `add(s(s(z)), s(s(z)))` - right hand side is the successor function application, let's unwrap it
2. `s(add(s(s(z)), s(z)))` - same here
3. `s(s( add( s(s(z)), z ) ))` - right hand side is zero
4. `s(s(s(s(z))))` - our result

In python, this would be something like (without using our previous implementation of natural numbers, just plain ints):
```python
def add(x: int, y: int) -> int:
  if (x < 0) or (y < 0):
    raise TypeError("x or y is not natural")

  while y != 0:
    x += 1
    y -= 1

  return x
```

It's time to implement addition on **our** natural numbers:
```python
def add(lhs: Nat, rhs: Nat) -> Nat:
  if rhs is Z:
    return lhs

  # otherwise, rhs is successor function of something
  return S(add(lhs, cast(S, rhs).of))
```

Let's test it

```python
>>> from test import *
>>> _2 = S(S(Z))
>>> _4 = add(_2, _2)
>>> int(_4)
4
>>> _4
S(S(S(S(Z))))
>>>
```

It works as expected

# Multiplication, exponentiation and tetration (and pentation, and much more)

More interestring part would be writing multiplication, as we may know,
multiplication of natural numbers is process of adding left hand side right hand side times, our algorithm would be:
1. If right hand side is zero - return `z`, since adding something zero times is zero
2. Otherwise, we're dealing with successor on the right hand side, we add left hand side to result of multiplication of left hand
side and predecessor (unwrapped) value of right hand side

Or, more precisely:
1. `mul(x, z) = z`
2. `mul(x, s(y)) = add(x, mul(x, y))`

In python:
```python
def mul(lhs: Nat, rhs: Nat) -> Nat:
  if rhs is Z:
    return Z
  return add(lhs, mul(lhs, cast(S, rhs).of))
```

Let's test it:
```python
>>> from test import *
>>> _2 = S(S(Z))
>>> _2 = S(S(Z))
>>> mul(_2, _2)
S(S(S(S(Z))))
>>> int(mul(_2, _2))
4
>>> _4 = add(_2, _2)
>>> _16 = mul(_4, _4)
>>> _16
S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(Z))))))))))))))))
>>> int(_16)
16
>>>
```

Once again, works as expected, may be you got a little tired, since definition of addition is similar to multiplication, but,
I promise, exponentiation is the last thing you must endure before the more general beast.

### Exponentiation

Exponentiation is the same as multiplication in definition, but instead of addition we'll use multiplication and simplest condition would be
`pow(x, z) = 1` (proving of that is out of scope)

defining exactly...

1. `pow(x, z) = 1`
2. `pow(x, s(y)) = mul(x, pow(x, y))`

In python:

```python
def pow(x: Nat, y: Nat) -> Nat:
  if y is Z:
    return S(Z)
  return mul(x, pow(x, cast(S, y).of))
```

Testing...

```python
>>> from test import *
>>> _2 = S(S(Z))
>>> _4 = mul(_2, _2)
>>> _16 = mul(_4, _4)
>>> _64 = mul(_16, _4)
>>> int(_64)
64
>>> _25 = add( pow(add(_2, S(Z)), _2), pow(_4, _2))
>>> int(_25)
25
```

Let's try computing bigger numbers, we're just invented quite fast-growing number operation:

```python
>>> _2in64 = pow(_2, _64)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/nerodono/Dev/Front/blog/test.py", line 57, in pow
    return mul(x, pow(x, cast(S, y).of))
  File "/home/nerodono/Dev/Front/blog/test.py", line 57, in pow
    return mul(x, pow(x, cast(S, y).of))
  File "/home/nerodono/Dev/Front/blog/test.py", line 57, in pow
    return mul(x, pow(x, cast(S, y).of))
  [Previous line repeated 52 more times]
  File "/home/nerodono/Dev/Front/blog/test.py", line 51, in mul
    return add(lhs, mul(lhs, cast(S, rhs).of))
  File "/home/nerodono/Dev/Front/blog/test.py", line 51, in mul
    return add(lhs, mul(lhs, cast(S, rhs).of))
  File "/home/nerodono/Dev/Front/blog/test.py", line 51, in mul
    return add(lhs, mul(lhs, cast(S, rhs).of))
  [Previous line repeated 78 more times]
  File "/home/nerodono/Dev/Front/blog/test.py", line 45, in add
    return S(add(lhs, cast(S, rhs).of))
  File "/home/nerodono/Dev/Front/blog/test.py", line 45, in add
    return S(add(lhs, cast(S, rhs).of))
  File "/home/nerodono/Dev/Front/blog/test.py", line 45, in add
    return S(add(lhs, cast(S, rhs).of))
  [Previous line repeated 859 more times]
RecursionError: maximum recursion depth exceeded
```

Woops, recursion limit! Rewriting our recursive functions in an iterative approach should help:

```python
def add(lhs: Nat, rhs: Nat) -> Nat:
  while rhs is not Z:
    rhs = cast(S, rhs).of
    lhs = S(lhs)
  return lhs

def mul(lhs: Nat, rhs: Nat) -> Nat:
  result = Z
  while rhs is not Z:
    rhs = cast(S, rhs).of
    result = add(lhs, result)
  return result

def pow(lhs: Nat, rhs: Nat) -> Nat:
  result = S(Z)
  while rhs is not Z:
    rhs = cast(S, rhs).of
    result = mul(lhs, result)
  return result
```

```python
>>> _2in64 = pow(_2, _64)
-- waiting --
```

...
...
...

It was quite brave to use our natural numbers to raise 2 in power of 64, thinking optimistically: one natural number object
occupies 16 bytes of memory (optimistic measurement!), so, if we want to raise 2 in power of 64, we need ~ `2^64 * 2^4 <=> 2^68` bytes of memory.
I suppose that's impossible on my machine, let's limit our appetite to `2^8`

```python
>>> _2in8 = pow(_2, mul(_4, _2))
>>> _2in8
S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(S(...
>>> int(_2in8)
256
>>>
```

Wow! Such a long application... (I cutted it down since it overflows content, lmao).

I will not torment you any loger

# Hyperoperation

Our previous inventions looked really similar, we just used previously defined function to implement next "order" function the same way as
previous function is defined, so, actually, it can be simplified.
But we should rewrite our natural number operations first, so they would be more optimal (addition, multiplication and power),
since it would take an eternity to compute tetration or pentation.

Beware! Arbitrary arithmetic function

```python
def operate(x: int, y: int, order: int) -> int:
  if (x < 0) or (y < 0) or (order < 0):
    raise TypeError("not natural x, y or order")

  if order == 0:
    return x + 1  # successor function
  elif order == 1:
    return x + y  # addition
  elif order == 2:
    return x * y  # multiplication
  elif order == 3:
    return x ** y # exponential
  # Higher order has the same relations when rhs is zero (proving this is out of scope)
  elif y == 0:
    return 1
  else:
    return operate(x, operate(x, y - 1, order), order - 1)
```

Let's test it:
```python
>>> # testing 0, 1, 2, 3 is out of scope, since they're implemented as underlying python integer operations
>>> operate(2, 3, 5) # 2[5]3, [5] is the binary operator, means that a[n]b the operation of order `n` would be applied to a
65536
>>> # Pentation of 2 and 3 is 65536! Let's test tetration
>>> operate(2, 3, 4)
16
>>> operate(2, 4, 4)
65536
>>> # Nice
>>> # let's test hexation
>>> operate(2, 3, 6)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/nerodono/Dev/Front/blog/hyper.py", line 18, in operate
    return operate(x, operate(x, y - 1, order), order - 1)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/nerodono/Dev/Front/blog/hyper.py", line 18, in operate
    return operate(x, operate(x, y - 1, order), order - 1)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/nerodono/Dev/Front/blog/hyper.py", line 18, in operate
    return operate(x, operate(x, y - 1, order), order - 1)
                      ^^^^^^^^^^^^^^^^^^^^^^^^
  [Previous line repeated 996 more times]
RecursionError: maximum recursion depth exceeded
>>> # Such a fast growing
>>> import sys
>>> sys.setrecursionlimit(2**30)
>>> operate(2, 3, 6)
^C
>>> # this was long
>>> operate(2, 2, 6)
4
>>> # Surprisingly small
```

# Conclusion

Today you learned some power (not only power, addition, multiplication, tetration, pentation, дохуя-tation).
I hope it was interesting.

## Futher reading

- Your delusions
- [Wikipedia](https://en.wikipedia.org/wiki/Hyperoperation)


