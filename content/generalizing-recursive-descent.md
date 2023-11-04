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

data Token = TOperator Operator
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
             compound      = Operator $ h:rest

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

import Prelude hiding ( scope, lookup )

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

lookup :: Operator -> Store -> Precedence
lookup op (Store operators _) =
    case M.lookup op operators of
       Just r -> r
       Nothing -> error $ "No such operator" ++ show op

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

or not two, there's one more thing: AST representation and evaluation.

```haskell
data Expr = EBinary Operator Expr Expr
          | ENumber Integer
          deriving Show

-- The function that will reduce two expressions to one
-- (perform binary operation)
type BinEvalFn = Operator -> Integer -> Integer -> Integer

eval :: BinEvalFn -> Expr -> Integer
eval reduce_binary expr =
    case expr of
       EBinary operator lhs rhs ->
           reduce_binary operator (eval' lhs) (eval' rhs)
       ENumber number ->
           number
    where eval' = eval reduce_binary
```

1. Skeleton of the `expression` function will look like this:

```haskell
type Tailed a = Maybe (a, [Token])

expression :: Store -> [Token] -> Tailed Expr
expression store =
    case splitMin store of
       Just (precedence, whats_left) ->
           -- We still have ways go to down
           undefined
       Nothing ->
           -- Here we on the factor's precedence
           undefined
```

Key aspect is the `case splitMin store of` part: `splitMin` returns lowest precedence on the `Store`, returns it as the head and whats left as the tail, so we'll go down from lowest to highest precedence.

Adding factor:

```haskell
type Tailed a = Maybe (a, [Token])

factor :: Store -> [Token] -> Tailed Expr
factor store (h:t) =
    case h of
       TNumber number -> Just ( ENumber number
                              , t
                              )
       TBracket Open  -> do
           (expr, t') <- expression store t
           case t' of
              -- Expect next token to be a closing bracket
              TBracket Close:t'' ->
                  Just ( expr
                       , t'' )
              -- Fail if it isn't
              _ -> Nothing

       
       -- Unexpected token
       _ -> Nothing
factor _ [] = Nothing

type ParseF = [Token] -> Tailed Expr
expression :: Store -> [Token] -> Tailed Expr
expression store =
    case splitMin store of
       Just (precedence, whats_left) ->
           -- We still have ways go to down
           binary precedence $ expression whats_left
       Nothing ->
           -- Here we on the factor's precedence
           factor store
    where
       binary :: Integer -> ParseF -> [Token] -> Tailed Expr
       binary current_precedence parse tokens =
           parse tokens >>= \(lhs, t) ->
              case t of
                  -- Next token should be infix operator
                  TOperator operator:t' ->
                     -- If we're not parsing operator with that precedence
                     -- return only lhs
                     if lookup operator store == current_precedence then
                         do
                             (rhs, t'') <- parse t'
                             Just ( EBinary operator lhs rhs
                                  , t'' )
                     else
                         Just ( lhs
                              , t )

                  -- Or it's just the end, for example:
                  -- 2: from + to * we got only factor
                  -- but it's still OK
                  _ -> Just ( lhs
                            , t )
```

But wait, will the `binary` parse correctly only the something like `a + b` and `a + b + c` would parse as `(+ a b)` and return `"+ c"` as the tail? Definetely! So we should parse sequential operator application as well:

```haskell
type Tailed a = Maybe (a, [Token])

factor :: Store -> [Token] -> Tailed Expr
factor store (h:t) =
    case h of
       TNumber number -> Just ( ENumber number
                              , t
                              )
       TBracket Open  -> do
           (expr, t') <- expression store t
           case t' of
              -- Expect next token to be a closing bracket
              TBracket Close:t'' ->
                  Just ( expr
                       , t'' )
              -- Fail if it isn't
              _ -> Nothing

       
       -- Unexpected token
       _ -> Nothing
factor _ [] = Nothing

type ParseF = [Token] -> Tailed Expr

-- Function that consumes left hand side expression
-- and remaining tokens and returns pair of resulting expression and the tail
type FoldFn = Expr -> [Token] -> Tailed Expr

expression :: Store -> [Token] -> Tailed Expr
expression store =
    -- Result of this case would be curried
    -- So actually we return `[Token] -> Tailed Expr`
    case splitMin store of
       Just (precedence, whats_left) ->
           -- We still have ways go to down
           binary precedence $ expression whats_left
       Nothing ->
           -- Here we on the factor's precedence
           factor store
    where
       binary :: Integer -> ParseF -> [Token] -> Tailed Expr
       binary current_precedence parse tokens =
           parse tokens >>= \(lhs, t) ->
              case t of
                  -- Next token should be infix operator
                  TOperator operator:t' ->
                     -- If we're not parsing operator with that precedence
                     -- return only lhs
                     if lookup operator store == current_precedence then
                         -- Before we met first operator
                         foldlExpr lhs (leftFoldFn parse $ expectSameOp operator) t
                     else
                         Just ( lhs
                              , t )

                  -- Or it's just the end, for example:
                  -- 2: from + to * we got only factor
                  -- but it's still OK
                  _ -> Just ( lhs
                            , t )

       leftFoldFn :: ParseF -> ([Token] -> Tailed Operator) -> Expr -> [Token] -> Tailed Expr
       leftFoldFn parse expectOperator lhs tokens = do
           (op, t) <- expectOperator tokens
           (rhs, t') <- parse t

           Just ( EBinary op lhs rhs
                , t' )
       expectSameOp :: Operator -> [Token] -> Tailed Operator
       expectSameOp op (TOperator got_op:t)
           | got_op == op = Just (got_op, t)
       expectSameOp _ _   = Nothing

       -- This is just modification of the standard `foldl`
       -- Folds multiple expressions into one
       foldlExpr :: Expr -> FoldFn -> [Token] -> Tailed Expr
       foldlExpr expr fold_fn tokens =
           case fold_fn expr tokens of
              Just (expr', tail') ->
                  foldlExpr expr' fold_fn tail'
              Nothing -> Just (expr, tokens)
```

For comfortable testing we'll need to add this:
```haskell
defaultStore :: Store
defaultStore = fromList [(Operator "+", 1), (Operator "-", 1), (Operator "*", 2)]

parseText :: String -> Tailed Expr
parseText = expression defaultStore . tokenize

-- Parses and returns S-expression
parseTextAsSExpr :: String -> Tailed String 
parseTextAsSExpr =
    fmap (Bi.first toSExpr) . parseText

toSExpr :: Expr -> String
toSExpr (EBinary (Operator op) lhs rhs) =
    "(" ++ op ++ " " ++ toSExpr lhs ++ " " ++ toSExpr rhs ++ ")"
toSExpr (ENumber number) =
    show number
```

Testing:

```haskell
ghci> parseTextAsSExpr "1 + 2 * 3"
Just ("(+ 1 (* 2 3))",[])
ghci> parseTextAsSExpr "1 + 2 + 3*3 + 4"
Just ("(+ (+ (+ 1 2) (* 3 3)) 4)",[])
```

Nice! Works as expected, but what if we try using brackets?
```haskell
ghci> parseTextAsSExpr "2 + 2 * 2"
Just ("(+ 2 (* 2 2))",[])
ghci> parseTextAsSExpr "(2 + 2) * 2"
Nothing
ghci>
```

What? It fails, what if...
```haskell
ghci> parseTextAsSExpr "(2) * 2"
Just ("(* 2 2)",[])
ghci>
```

Yeah, so just essentially our problem is `factor`, going in the place...
```haskell
expression :: Store -> [Token] -> Tailed Expr
expression store =
    -- Result of this case would be curried
    -- So actually we return `[Token] -> Tailed Expr`
    case splitMin store of
       Just (precedence, whats_left) ->
           -- We still have ways go to down
           binary precedence $ expression whats_left
           --                  ^^^^^^^^^^^^^^^^^^^^^
           --                        Cutted off
       Nothing ->
           -- Here we on the factor's precedence
           factor store
           --     ^^^^^
```

Nah, our `store`'s scope is empty at the moment where it gets to the `factor` and `factor` internally can call an `expression`, the result is nested expressions can be only `factor`s:

```haskell
ghci> parseTextAsSExpr "((2)) * 2"
Just ("(* 2 2)",[])
ghci> parseTextAsSExpr "(((((2))))) * 2"
Just ("(* 2 2)",[])
ghci>
```

So the solution is just to keep track of the **"root"** `scope`:
```haskell
import qualified Data.Map       as M
import qualified Data.Bifunctor as Bi

import Prelude hiding ( scope, lookup )

type Precedence = Integer
data Store = Store { operators   :: M.Map Operator   Precedence
                   , scope       :: M.Map Precedence Integer
                   , root        :: M.Map Precedence Integer
                   }
           deriving Show

splitMin :: Store -> Maybe (Precedence, Store)
splitMin (Store operators scope root) =
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
              in Just (key, Store operators scope' root)

restoreRoot :: Store -> Store
restoreRoot (Store operators _ root) =
    Store operators root root

lookup :: Operator -> Store -> Precedence
lookup op (Store operators _ _) =
    case M.lookup op operators of
       Just r -> r
       Nothing -> error $ "No such operator" ++ show op

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
       in Store operators' scope' scope'
```

Here we just save the **root** precedences list and chopping plain `scope`, fixing original code:
```haskell

-- snip --

expression :: Store -> [Token] -> Tailed Expr
expression store =
    -- Result of this case would be curried
    -- So actually we return `[Token] -> Tailed Expr`
    case splitMin store of
       Just (precedence, whats_left) ->
           -- We still have ways go to down
           binary precedence $ expression whats_left
       Nothing ->
           -- Here we on the factor's precedence
           factor $ restoreRoot store
    -- snip --
```

Rechecking gives...
```haskell
ghci> parseTextAsSExpr "(2 + 2) * 2"
Just ("(* (+ 2 2) 2)",[])
ghci> parseTextAsSExpr "(2 + 2 * 2) * 2"
Just ("(* (+ 2 (* 2 2)) 2)",[])
ghci>
```

The right answer, nice!

# 3. Evaluate

The last part is evaluation, the easiest since we've done the most

```haskell
evalText :: String -> Integer
evalText text =
    case parseText text of
       Just (tree, []) ->
           eval evaluateBinary tree
       Just (tree, t) ->
           error $ "Failed to parse entire expr (" ++ toSExpr tree ++ "): tail is " ++ show t
       Nothing ->
           error "Failed to parse expression"
    where
        evaluateBinary op lhs rhs =
            case op of
                Operator "+" -> lhs + rhs
                Operator "-" -> lhs - rhs
                Operator "*" -> lhs * rhs
```

let's test it:
```haskell
ghci> evalText "2 + 2 * 2"
6
ghci> evalText "(2 + 2) * 2"
8
ghci> evalText "3*3 + 4*4"
25
ghci> evalText "2 + 2 + 2 - 2"
4
ghci>
```

Works as expected

# Further reading

1. [Crafting Interpreters](https://craftinginterpreters.com/) book - easy to start
2. [Simple but Powerful Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html) - matklad's article, alternative to the recursive descent, can be generalized the same way

# At the end

As you can see recursive descent is pretty easily can be generalized and, as said in the craftinginterpreters book: [It rocks.](https://craftinginterpreters.com/parsing-expressions.html#recursive-descent-parsing)

Now go write your own language for the great good!
Resulting code you can find on the [gist](https://gist.github.com/nerodono/c78a012c009331e8ab8b40a0f3845f8d)
