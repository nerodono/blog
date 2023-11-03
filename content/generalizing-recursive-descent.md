+++
title = 'Generalizing recursive descent'
description = 'My generalization of the recursive descent parser algorithm'
date = 2023-11-03T23:09:04+03:00
draft = true
+++

The recursive descent parser is the first algorithm that comes to mind when your task is to write
expression parser, so, the example grammar of simple language that can add and multiple things will look like this:

```ebnf
digit  = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";
number = digit {digit};

term   = factor { "*" factor };
factor = number
       | "(" expression ")"
       | "-" factor;

expression = term { "+" term };
```

Whitespaces are ignored. This will parse the following expressions:
- `1+2*3` as `(+ 1 (* 2 3))`
- `1+2*4+3` as `(+ (+ 1 (* 2 4)) 3)`

And so on, grammar is really primitive, so further examples are meaningless.
Main idea of the algorithm is that items of lower precedence are compund of higher precedence items, like so:
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

I added exponentiation to make schema easier to understand.
But what if we want
- Arbitrary number of operators?
- Arbitrary number of precedences?

It can be useful to tweak parser with no actual effort or to build more flexible language, so what?

# Generalization

Here we go, our task is to write mathematical expression evaluator that can:
- Handle arbitrary number of operators
- Handle arbitrary number of precedence

The task is not that hard though, our algorithm is:
1. Write tokenizer
2. Write parser
3. Evaluate

## 1. Write tokenizer

Tokenizer is the easiest part of writing evaluator, we just should break up our text into tokens:

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
- Operators compound of `+-*<>/!?` chars, for example: <$>, <!>, <>, >, <, */ and so on
- Integer literals: 1, 2, 3, 4, ...
- Brackets

## 2. Write parser

Main objective of this article, basically our parser will consist of two components:
1. Precedence store  - store where 
2. Expression parser - main logic of the parser

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
the naive algorithm general algorithm will do as follows:
- Pick lowest precedence from the store
- Remove it from the **scope**
- Proceed it

And it will work... As far your operators map doesn't contain operators with same precedence:
```text
1 + 2 - 3
```

It will successfuly parse (+ 1 2) and return "- 3" as the tail, because `-` has same precedence as the `+`. Why is that so? It'll
explain it further later, so pay attention!

### Actual parser

We need exactly two things:
- factor - items with the highest possible precedence
- expression - items with the lower precedence

or not two, there's one more thing: AST representation
