+++
title = 'Generalizing recursive descent'
description = 'My generalization of the recursive descent parser algorithm'
date = 2023-11-03T23:09:04+03:00
draft = true
+++

# Warning, it's a draft!

This is just for review

---

The recursive descent parser is the first algorithm that comes to mind when your task is to write
expression parser, so, the example grammar of a simple language that can add multiple things will look like this:

```ebnf
digit  = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";
number = digit {digit};

term   = factor { "*" factor };
factor = number
       | "(" expression ")"
       | "-" factor;

expression = term { "+" term };
```

Whitespaces are ignored. This will parse into the following expressions:
- `1+2*3` as `(+ 1 (* 2 3))`
- `1+2*4+3` as `(+ (+ 1 (* 2 4)) 3)`

And so on, grammar is really primitive, so further examples are unnecessary.
Main idea of the algorithm is that items of lower precedence are compound of higher precedence items, like so:
```text
t - term
f - factor
p - exponentiation

     t   +   t
     |       |
    p*p     p*p
    | |     | |
     |       |
    f^f     f^f
```

I added exponentiation to make the schema easier to understand.
But what if we want
- Arbitrary number of operators?
- Arbitrary number of precedences?

It can be useful to be able to easily tweak the parser or build a more flexible language, so what?

# Generalization

Here we go, our task is to write mathematical expression evaluator that can:
- Handle arbitrary number of operators
- Handle arbitrary number of precedences

The task is not that hard though, our algorithm is:
1. Write a tokenizer
2. Write a parser
3. Evaluate

## 1. Writing a tokenizer

Tokenizer is the easiest part of writing an evaluator, we just need to break up our text into tokens:

```haskell
import Data.Char ( isDigit )

data BType = Open | Close
           deriving Show

newtype Operator = Operator String
                 deriving(Show, Eq, Ord) -- Ord will be needed later

data Token = TOperator String
           | TNumber   Integer
           | TBracket  BType
           deriving Show

compoundOperatorChars :: String
compoundOperatorChars = "+-*<>/!?"

tokenize :: String -> [Token]

-- Skipping the whitespaces
tokenize (h:t) | h `elem` " \t" =
       tokenize t

-- Brackets
tokenize ('(':t) = TBracket Open  : tokenize t
tokenize (')':t) = TBracket Close : tokenize t

tokenize (h:t) | h `elem` compoundOperatorChars =
       TOperator compound : tokenize tail'
       where (rest, tail') = span (`elem` compoundOperatorChars) t
             compound      = h:rest

-- Parsing digits
tokenize (h:t) | isDigit h =
       token : tokenize tail'
       where (rest, tail') = span isDigit t
             token         = TNumber $ read (h:rest)

tokenize (h:t) = error $ "Unexpected char " ++ [h]
tokenize [] = []
```

#### Demonstration

```haskell
ghci> tokenize "1 + 2 + 3 * 4 <> 5"
[TNumber 1,TOperator "+",TNumber 2,TOperator "+",TNumber 3,TOperator "*",TNumber 4,TOperator "<>",TNumber 5]
```

So far so good, it will parse:
- Operators compound of `+-*<>/!?` characters, for example: <$>, <!>, <>, >, <, */ and so on
- Integer literals: 1, 2, 3, 4, ...
- Brackets

## 2. Writing a parser

Main objective of this article, basically our parser will consist of two components:
1. Precedence store  - where to store 
2. Expression parser - main logic of the parser

**NOTE**: The parser will not implement unary operations, it's pretty easy to tweak the parser to implement them.

### What's a Precedence store?

Basically just a map with some additional quirks for hierarchy:

```haskell
import qualified Data.Map       as M
import qualified Data.Bifunctor as Bi

import Prelude hiding ( scope )

type Precedence = Integer
data Store = Store { operators   :: M.Map Operator   Precedence
                   , scope       :: M.Map Precedence Integer
                   }
           deriving Show

splitMin :: Store -> Maybe (Precedence, Store)
splitMin (Store operators scope) =
       -- M.keys returns keys in the ascending order
       -- so the first returned key is the least
       case M.keys scope of
           [] -> Nothing
           key:_ ->
              -- Here' we remove the least precedence from scope
              -- if number of operators with that precedence is 1
              -- or just subtract one, if there's more\
              let scope' = M.updateWithKey f key scope
              f _ v      = if v == 1 then Nothing else Just (v - 1)
              in Just (key, Store operators scope')

fromList :: [( Operator, Precedence )] -> Store
fromList list' =
       let -- Construct map of operator:precedence
           operators' = M.fromList list'
           -- Create so-called "scope"
           -- it's just map of precedence:number-of-operators
           scope'     = foldl f M.empty list'

           f map' (op, prec') =
              -- insert or add 1 to existing entry in the scope map
              M.insertWith (+) prec' 1 map'
       in Store operators' scope'
```

Wait, why is **scope** even needed? Well, that's the key aspect of generalization, I call it "splitting".
Still remember the idea of recursive descent? The lower precedence items are compound of higher precedence,
the general algorithm will do the following:
- Pick lowest precedence from the store
- Remove it from the **scope**
- Process it

And it will work... So far your operators map doesn't contain operators with same precedence:
```text
1 + 2 - 3
```

It will successfuly parse (+ 1 2) and return "- 3" as the tail, because `-` has same precedence as the `+`. Why is that so? I'll
explain it in further detail later, so pay attention!

### Actual parser

We need exactly two things:
- factor - items with the highest possible precedence
- expression - items with lower precedence

or not two, there's one more thing: AST representation.

```haskell
data Expr = EBinary Operator Expr Expr
          | ENumber Integer
```

That's all we need
