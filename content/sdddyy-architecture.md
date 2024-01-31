+++
title = 'Sdddyy Architecture'
date = 2024-01-31T06:44:50+03:00
description = 'The most perspective and perfect architectural pattern for us, mortals'
+++

## Introduction

Many of us struggling to implement code that is:
- Easy to reason
- Easy to maintain
- Easy to extend
- Easy to read

many things for that purpose were done and the biggest minds in that area are **Martin Fowler** & **Robert Martin**
(A.K.A **Uncle Bob**), I want to put my name in that list, so I propose SDDDYY pattern.

### SDDDYY axioms

**SDDDYY** is an architectural pattern, it built around five simple axioms:
- Your code must go in the `skibidi/` folder in the project root by analogy with `src/`
- `skibidi` contain `dop`s, there only can be exactly three `dop`s in the folder
- `dop`s contain `dop`s, only three of those including next `dop`s group, except the situation where `dop` contain `yes`
- `yes` contain single `yes` and must go inside the deepest `dop`, this is why exception for three dops was made
- Every `dop` can access only `dop`s down the hierarchy, no `dop` can contain `dop` upper from it

Done with theory, let's jump directly in beautiful world of **SDDDYY**

## Skibidi calculator

I will show on example, let's implement simplest possible calculator in Python.

1. Create the directory for it
```shell
$ mkdir calc
$ cd calc
```

2. Create needed files
```shell
$ mkdir -p skibidi/dop/dop/dop/yes # we will need that dop(s) later
$ touch main.py skibidi/{__init__.py,dop/__init__.py}
$ touch skibidi/dop/dop/{__init__.py,dop/__init__.py}
$ touch skibidi/dop/dop/dop/yes/{__init__.py,yes.py}
```

Now, your directory should be the following structure:
```text
.
├── main.py
└── skibidi
    ├── dop
    │   ├── dop
    │   │   ├── dop
    │   │   │   ├── __init__.py
    │   │   │   └── yes
    │   │   │       ├── __init__.py
    │   │   │       └── yes.py
    │   │   └── __init__.py
    │   └── __init__.py
    └── __init__.py
```

Let's write the code, we need the following dops:
- `TokenDop` - abstract dop for our tokens
- `OperatorDop`, `BracketDop`, `NumberDop` - dops representing the actual tokens
- `ExprDop` - abstract dop for the expression tree
- `BinOpDop`, `UnaryOpDop`, `NumberExprDop` - dops representing tree leafs

- First **dop**
1. `skibidi/dop/dop/dop/token.py`

```python
from abc import ABCMeta

class TokenDop(metaclass=ABCMeta):
    ...
```

2. `skibidi/dop/number.py`

```python
from .dop.dop.token import TokenDop

from dataclasses import dataclass

@dataclass(frozen=True)
class NumberDop(TokenDop):
    value: int
```

3. `skibidi/dop/operator.py`

```python
from .dop.dop.token import TokenDop

from dataclasses import dataclass
from typing import Literal as L


@dataclass(frozen=True)
class OperatorDop(TokenDop):
    op: L['+'] | L['-'] | L['*'] | L['^'] | L['/']
```

- Second **dop**

1. `skibidi/dop/dop/bracket.py`

```python
from .dop.token import TokenDop

from dataclasses import dataclass

@dataclass(frozen=True)
class BracketDop(TokenDop):
    open: bool
```

Now, our next `dop` will contain `yes` - actual logic for `dop`s, so second deepest dop will contain exactly two `dop`s

- Third **dop**

1. `skibidi/dop/dop/dop/expr.py` - The expression dop

```python
from dataclasses import dataclass
from abc import ABCMeta, abstractmethod

from skibidi.dop.operator import OperatorDop
from skibidi.dop.number import NumberDop

class ExprDop(metaclass=ABCMeta):
    @abstractmethod
    def eval(self) -> int:
        raise NotImplemented


@dataclass(frozen=True)
class BinOpDop(ExprDop):
    lhs: ExprDop
    op: OperatorDop
    rhs: ExprDop

    def eval(self) -> int:
        lhs = self.lhs.eval()
        rhs = self.rhs.eval()

        match self.op.op:
            case '+':
                return lhs + rhs
            case '-':
                return lhs - rhs
            case '*':
                return lhs * rhs
            case '/':
                return lhs // rhs

            case o:
                raise TypeError(f"Unknown binary operator: {o}")


@dataclass(frozen=True)
class NumberExprDop(ExprDop):
    value: NumberDop

    def eval(self) -> int:
        return self.value.value


@dataclass(frozen=True)
class UnaryOpDop(ExprDop):
    op: OperatorDop
    rhs: ExprDop

    def eval(self) -> int:
        rhs = self.rhs.eval()
        match self.op.op:
            case '-':
                return -rhs
            case un:
                raise TypeError(f"Unknown unary operator: {un}")
```

## The `yes`

Now, we are ready to write `yes`

`skibidi/dop/dop/dop/yes/yes.py`

```python
from ..token import TokenDop
from ..expr import (
    ExprDop,
    BinOpDop,
    UnaryOpDop,
    NumberExprDop
)

from skibidi.dop.operator import OperatorDop
from skibidi.dop.number import NumberDop
from skibidi.dop.dop.bracket import BracketDop

from typing import Callable
from itertools import takewhile

OPERATORS = {'+', '-', '*', '/', '^'}

class Eof(Exception): ...

def span(text: str, predicate: Callable[[str], bool]) -> tuple[str, str]:
    matched = ''.join(list(takewhile(predicate, text)))
    return (matched, text[len(matched):])

def tokenize(text: str) -> list[TokenDop]:
    text = text.strip()  # get rid of leading/trailing whitespaces
    if not text:
        return []

    head, tail = text[0], text[1:]
    if head.isdigit():
        (rest, tail) = span(tail, str.isdigit)
        number = int(head + rest)

        return [NumberDop(number)] + tokenize(tail)
    elif head.isspace():
        return tokenize(tail)
    elif head in OPERATORS:
        return [OperatorDop(head)] + tokenize(tail)
    elif head in '()':
        return [BracketDop(head == '(')] + tokenize(tail)
    else:
        raise ValueError(f"Unknown token {head!r}")

def factor(tokens: list[TokenDop]) -> tuple[ExprDop, list[TokenDop]]:
    if not tokens:
        raise Eof()

    head, tail = tokens[0], tokens[1:]
    if isinstance(head, BracketDop):
        if head.open:
            expr, left = parse_tailed(tail)
            if isinstance(left[0], BracketDop) and not left[0].open:
                return expr, left[1:]
            raise SyntaxError("Unmatched closing bracket")
    elif isinstance(head, NumberDop):
        return NumberExprDop(head), tail
    elif isinstance(head, OperatorDop):
        unary_operand, left = factor(tail)
        return UnaryOpDop(head, unary_operand), left

    raise SyntaxError(f"Unexpected token {head!r}")

def expect_operator(tokens: list[TokenDop], one_of: tuple[str, ...]) -> tuple[OperatorDop, list[TokenDop]] | None:
    if not tokens:
        return None
    head, tail = tokens[0], tokens[1:]
    if isinstance(head, OperatorDop) and head.op in one_of:
        return (head, tail)
    return None

def left_associative(
    tokens: list[TokenDop],
    ops: tuple[str, ...],
    down: Callable[[list[TokenDop]], tuple[ExprDop, list[TokenDop]]],
) -> tuple[ExprDop, list[TokenDop]]:
    lhs, t = down(tokens)
    o = expect_operator(t, one_of=ops)
    if o is None:
        return lhs, t
    operator, t = o

    rhs, t = down(t)
    expr = BinOpDop(lhs, operator, rhs)

    # Left associativity
    while True:
        o = expect_operator(t, one_of=ops)
        if o is None:
            break
        operator, t = o
        rhs, t = down(t)

        expr = BinOpDop(expr, operator, rhs)
    return expr, t

def parse_tailed(tokens: list[TokenDop]) -> tuple[ExprDop, list[TokenDop]]:
    return left_associative(
        tokens,
        ('+', '-'),
        lambda tks: left_associative(
            tks,
            ('*', '/'),
            factor,
        )
    )
```

Easy, right? Let's write simple example of usage in `main.py`:

```python
import sys
import pprint

from skibidi.dop.dop.dop.yes.yes import tokenize, parse_tailed

tks = tokenize("(2 + 2 + 3 + 4) * 2")
expr, tail = parse_tailed(tks)

if tail:
    print(f"[-] Parse error, left tokens: {tail}")
    sys.exit(1)

print("Expr:")
pprint.pprint(expr)
print("Result:")
print(expr.eval())
```

# Conclusion

I suppose you may be confused, but! Give it a try, it's a best ever invented architectural pattern, after all,
its merits can understand only most experienced of us.

All code for this introduction can be found at https://github.com/NeroResearches/sdddyy
