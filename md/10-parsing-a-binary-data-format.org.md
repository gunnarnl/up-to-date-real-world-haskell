Chapter 10: Code case study: parsing a binary data format
=========================================================

In this chapter, we\'ll discuss a common task: parsing a binary file. We
will use this task for two purposes. Our first is indeed to talk a
little about parsing, but our main goal is to talk about program
organisation, refactoring, and \"boilerplate removal\". We will
demonstrate how you can tidy up repetitious code, and set the stage for
our discussion of monads in [Chapter 14, Monads](15-monads.org).

The file formats that we will work with come from the netpbm suite, an
ancient and venerable collection of programs and file formats for
working with bitmap images. These file formats have the dual advantages
of wide use and being fairly easy, though not completely trivial, to
parse. Most importantly for our convenience, netpbm files are not
compressed.

Greyscale files
---------------

The name of netpbm\'s greyscale file format is PGM (\"portable grey
map\"). It is actually not one format, but two; the \"plain\" (or
\"P2\") format is encoded as ASCII, while the more common \"raw\"
(\"P5\") format is mostly binary.

A file of either format starts with a header, which in turn begins with
a \"magic\" string describing the format. For a plain file, the string
is `P2`, and for raw, it\'s `P5`. The magic string is followed by white
space, then by three numbers: the width, height, and maximum grey value
of the image. These numbers are represented as ASCII decimal numbers,
separated by white space.

After the maximum grey value comes the image data. In a raw file, this
is a string of binary values. In a plain file, the values are
represented as ASCII decimal numbers separated by single space
characters.

A raw file can contain a sequence of images, one after the other, each
with its own header. A plain file contains only one image.

Parsing a raw PGM file
----------------------

For our first try at a parsing function, we\'ll only worry about raw PGM
files. We\'ll write our PGM parser as a *pure* function. It\'s not
responsible for obtaining the data to parse, just for the actual
parsing. This is a common approach in Haskell programs. By separating
the reading of the data from what we subsequently do with it, we gain
flexibility in where we take the data from.

We\'ll use the `ByteString` type to store our greymap data, because
it\'s compact. Since the header of a PGM file is ASCII text, but its
body is binary, we import both the text and binary-oriented `ByteString`
modules.

<div>

[PNM.hs]{.label}

``` {.haskell}
import qualified Data.ByteString.Lazy.Char8 as L8
import qualified Data.ByteString.Lazy as L
import Data.Char (isSpace)
```

</div>

For our purposes, it doesn\'t matter whether we use a lazy or strict
`ByteString`, so we\'ve somewhat arbitrarily chosen the lazy kind.

We\'ll use a straightforward data type to represent PGM images.

<div>

[PNM.hs]{.label}

``` {.haskell}
data Greymap = Greymap {
      greyWidth :: Int
    , greyHeight :: Int
    , greyMax :: Int
    , greyData :: L.ByteString
    } deriving (Eq)
```

</div>

Normally, a Haskell `Show` instance should produce a string
representation that we can read back by calling `read`. However, for a
bitmap graphics file, this would potentially produce huge text strings,
for example if we were to `show` a photo. For this reason, we\'re not
going to let the compiler automatically derive a `Show` instance for us:
we\'ll write our own, and intentionally simplify it.

<div>

[PNM.hs]{.label}

``` {.haskell}
instance Show Greymap where
    show (Greymap w h m _) = "Greymap " ++ show w ++ "x" ++ show h ++
                             " " ++ show m
```

</div>

Because our `Show` instance intentionally avoids printing the bitmap
data, there\'s no point in writing a `Read` instance, as we can\'t
reconstruct a valid Greymap from the result of `show`.

Here\'s an obvious type for our parsing function.

<div>

[PNM.hs]{.label}

``` {.haskell}
parseP5 :: L.ByteString -> Maybe (Greymap, L.ByteString)
```

</div>

This will take a `ByteString`, and if the parse succeeds, it will return
a single parsed `Greymap`, along with the string that remains after
parsing. That residual string will be available for future parses.

Our parsing function has to consume a little bit of its input at a time.
First, we need to assure ourselves that we\'re really looking at a raw
PGM file; then we need to parse the numbers from the remainder of the
header; then we consume the bitmap data. Here\'s an obvious way to
express this, which we will use as a base for later improvements.

<div>

[PNM.hs]{.label}

``` {.haskell}
matchHeader :: L.ByteString -> L.ByteString -> Maybe L.ByteString

-- "nat" here is short for "natural number"
getNat :: L.ByteString -> Maybe (Int, L.ByteString)

getBytes :: Int -> L.ByteString -> Maybe (L.ByteString, L.ByteString)

parseP5 s =
  case matchHeader (L8.pack "P5") s of
    Nothing -> Nothing
    Just s1 ->
      case getNat s1 of
        Nothing -> Nothing
        Just (width, s2) ->
          case getNat (L8.dropWhile isSpace s2) of
            Nothing -> Nothing
            Just (height, s3) ->
              case getNat (L8.dropWhile isSpace s3) of
                Nothing -> Nothing
                Just (maxGrey, s4)
                  | maxGrey > 255 -> Nothing
                  | otherwise ->
                      case getBytes 1 s4 of
                        Nothing -> Nothing
                        Just (_, s5) ->
                          case getBytes (width * height) s5 of
                            Nothing -> Nothing
                            Just (bitmap, s6) ->
                              Just (Greymap width height maxGrey bitmap, s6)
```

</div>

This is a very literal piece of code, performing all of the parsing in
one long staircase of `case` expressions. Each function returns the
residual `ByteString` left over after it has consumed all it needs from
its input string. We pass each residual string along to the next step.
We deconstruct each result in turn, either returning `Nothing` if the
parsing step failed, or building up a piece of the final result as we
proceed. Here are the bodies of the functions that we apply during
parsing. Their types are commented out because we already presented them
above.

<div>

[PNM.hs]{.label}

``` {.haskell}
-- L.ByteString -> L.ByteString -> Maybe L.ByteString
matchHeader prefix str
    | prefix `L8.isPrefixOf` str
        = Just (L8.dropWhile isSpace (L.drop (L.length prefix) str))
    | otherwise
        = Nothing

-- L.ByteString -> Maybe (Int, L.ByteString)
getNat s = case L8.readInt s of
             Nothing -> Nothing
             Just (num,rest)
                 | num <= 0    -> Nothing
                 | otherwise -> Just (fromIntegral num, rest)

-- Int -> L.ByteString -> Maybe (L.ByteString, L.ByteString)
getBytes n str = let count           = fromIntegral n
                     both@(prefix,_) = L.splitAt count str
                 in if L.length prefix < count
                    then Nothing
                    else Just both
```

</div>

Getting rid of boilerplate code
-------------------------------

While our `parseP5` function works, the style in which we wrote it is
somehow not pleasing. Our code marches steadily to the right of the
screen, and it\'s clear that a slightly more complicated function would
soon run out of visual real estate. We repeat a pattern of constructing
and then deconstructing `Maybe` values, only continuing if a particular
value matches `Just`. All of the similar `case` expressions act as
\"boilerplate code\", busywork that obscures what we\'re really trying
to do. In short, this function is begging for some abstraction and
refactoring.

If we step back a little, we can see two patterns. First is that many of
the functions that we apply have similar types. Each takes a
`ByteString` as its last argument, and returns `Maybe` something else.
Secondly, every step in the \"ladder\" of our `parseP5` function
deconstructs a `Maybe` value, and either fails or passes the unwrapped
result to a function.

We can quite easily write a function that captures this second pattern.

<div>

[PNM.hs]{.label}

``` {.haskell}
(>>?) :: Maybe a -> (a -> Maybe b) -> Maybe b
Nothing >>? _ = Nothing
Just v  >>? f = f v
```

</div>

The `(>>?)` function acts very simply: it takes a value as its left
argument, and a function as its right. If the value is not `Nothing`, it
applies the function to whatever is wrapped in the `Just` constructor.
We have defined our function as an operator so that we can use it to
chain functions together. Finally, we haven\'t provided a fixity
declaration for `(>>?)`, so it defaults to `infixl 9` (left associative,
strongest operator precedence). In other words, `a >>? b >>? c` will be
evaluated from left to right, as `(a >>? b) >>? c)`.

With this chaining function in hand, we can take a second try at our
parsing function.

<div>

[PNM.hs]{.label}

``` {.haskell}
parseP5_take2 :: L.ByteString -> Maybe (Greymap, L.ByteString)
parseP5_take2 s =
    matchHeader (L8.pack "P5") s      >>?
    \s -> skipSpace ((), s)           >>?
    (getNat . snd)                    >>?
    skipSpace                         >>?
    \(width, s) ->   getNat s         >>?
    skipSpace                         >>?
    \(height, s) ->  getNat s         >>?
    \(maxGrey, s) -> getBytes 1 s     >>?
    (getBytes (width * height) . snd) >>?
    \(bitmap, s) -> Just (Greymap width height maxGrey bitmap, s)

skipSpace :: (a, L.ByteString) -> Maybe (a, L.ByteString)
skipSpace (a, s) = Just (a, L8.dropWhile isSpace s)
```

</div>

The key to understanding this function is to think about the chaining.
On the left hand side of each `(>>?)` is a `Maybe` value; on the right
is a function that returns a `Maybe` value. Each left-and-right-sides
expression is thus of type `Maybe`, suitable for passing to the
following `(>>?)` expression.

The other change that we\'ve made to improve readability is add a
`skipSpace` function. With these changes, we\'ve halved the number of
lines of code compared to our original parsing function. By removing the
boilerplate `case` expressions, we\'ve made the code easier to follow.

While we warned against overuse of anonymous functions in [the section
called \"Anonymous (lambda)
functions\"](4-functional-programming.org::*Anonymous (lambda) functions)
in our chain of functions here. Because these functions are so small, we
wouldn\'t improve readability by giving them names.

Implicit state
--------------

We\'re not yet out of the woods. Our code explicitly passes pairs
around, using one element for an intermediate part of the parsed result
and the other for the current residual `ByteString`. If we want to
extend the code, for example to track the number of bytes we\'ve
consumed so that we can report the location of a parse failure, we
already have eight different spots that we will need to modify, just to
pass a three-tuple around.

This approach makes even a small body of code difficult to change. The
problem lies with our use of pattern matching to pull values out of each
pair: we have embedded the knowledge that we are always working with
pairs straight into our code. As pleasant and helpful as pattern
matching is, it can lead us in some undesirable directions if we do not
use it carefully.

Let\'s do something to address the inflexibility of our new code. First,
we will change the type of state that our parser uses.

<div>

[Parse.hs]{.label}

``` {.haskell}
import qualified Data.ByteString.Lazy.Char8 as L8
import qualified Data.ByteString.Lazy as L
import Data.Int

data ParseState = ParseState {
      string :: L.ByteString
    , offset :: Int64
    } deriving (Show)
```

</div>

In our switch to an algebraic data type, we added the ability to track
both the current residual string and the offset into the original string
since we started parsing. The more important change was our use of
record syntax: we can now *avoid* pattern matching on the pieces of
state that we pass around, and use the accessor functions `string` and
`offset` instead.

We have given our parsing state a name. When we name something, it can
become easier to reason about. For example, we can now look at parsing
as a kind of function: it consumes a parsing state, and produces both a
new parsing state and some other piece of information. We can directly
represent this as a Haskell type.

<div>

[Parse.hs]{.label}

``` {.haskell}
simpleParse :: ParseState -> (a, ParseState)
simpleParse = undefined
```

</div>

To provide more help to our users, we would like to report an error
message if parsing fails. This only requires a minor tweak to the type
of our parser.

<div>

[Parse.hs]{.label}

``` {.haskell}
betterParse :: ParseState -> Either String (a, ParseState)
betterParse = undefined
```

</div>

In order to future-proof our code, it is best if we do not expose the
implementation of our parser to our users. When we explicitly used pairs
for state earlier, we found ourselves in trouble almost immediately,
once we considered extending the capabilities of our parser. To stave
off a repeat of that difficulty, we will hide the details of our parser
type using a `newtype` declaration.

<div>

[Parse.hs]{.label}

``` {.haskell}
newtype Parse a = Parse {
    runParse :: ParseState -> Either String (a, ParseState)
}
```

</div>

Remember that the `newtype` definition is just a compile-time wrapper
around a function, so it has no run-time overhead. When we want to use
the function, we will apply the `runParser` accessor.

If we do not export the `Parse` value constructor from our module, we
can ensure that nobody else will be able to accidentally create a
parser, nor will they be able to inspect its internals via pattern
matching.

### The identity parser

Let\'s try to define a simple parser, the *identity* parser. All it does
is turn whatever it is passed into the result of the parse. In this way,
it somewhat resembles the `id` function.

<div>

[Parse.hs]{.label}

``` {.haskell}
identity :: a -> Parse a
identity a = Parse (\s -> Right (a, s))
```

</div>

This function leaves the parse state untouched, and uses its argument as
the result of the parse. We wrap the body of the function in our `Parse`
type to satisfy the type checker. How can we use this wrapped function
to parse something?

The first thing we must do is peel off the `Parse` wrapper so that we
can get at the function inside. We do so using the `runParse` function.
We also need to construct a `ParseState`, then run our parsing function
on that parse state. Finally, we\'d like to separate the result of the
parse from the final `ParseState`.

<div>

[Parse.hs]{.label}

``` {.haskell}
parse :: Parse a -> L.ByteString -> Either String a
parse parser initState
    = case runParse parser (ParseState initState 0) of
        Left err          -> Left err
        Right (result, _) -> Right result
```

</div>

Because neither the `identity` parser nor the `parse` function examines
the parse state, we don\'t even need to create an input string in order
to try our code.

``` {.screen}
ghci> :load Parse
[1 of 1] Compiling Main             ( Parse.hs, interpreted )
Ok, one module loaded.
ghci> :type parse (identity 1) undefined
parse (identity 1) undefined :: Num a => Either String a
ghci> parse (identity 1) undefined
Right 1
ghci> parse (identity "foo") undefined
Right "foo"
```

A parser that doesn\'t even inspect its input might not seem
interesting, but we will shortly see that in fact it is useful.
Meanwhile, we have gained confidence that our types are correct and that
we understand the basic workings of our code.

### Record syntax, updates, and pattern matching

Record syntax is useful for more than just accessor functions: we can
use it to copy and partly change an existing value. In use, the notation
looks like this.

<div>

[Parse.hs]{.label}

``` {.haskell}
modifyOffset :: ParseState -> Int64 -> ParseState
modifyOffset initState newOffset = initState { offset = newOffset }
```

</div>

This creates a new `ParseState` value identical to `initState`, but with
its `offset` field set to whatever value we specify for `newOffset`.

``` {.screen}
ghci> let before = ParseState (L8.pack "foo") 0
ghci> let after = modifyOffset before 3
ghci> before
ParseState {string = "foo", offset = 0}
ghci> after
ParseState {string = "foo", offset = 3}
```

We can set as many fields as we want inside the curly braces, separating
them using commas.

### A more interesting parser

Let\'s focus now on writing a parser that does something meaningful.
We\'re not going to get too ambitious yet: all we want to do is parse a
single byte.

<div>

[Parse.hs]{.label}

``` {.haskell}
-- import the Word8 type from Data.Word
parseByte :: Parse Word8
parseByte =
    getState ==> \initState ->
    case L.uncons (string initState) of
      Nothing ->
          bail "no more input"
      Just (byte,remainder) ->
          putState newState ==> \_ ->
          identity byte
        where newState = initState { string = remainder,
                                     offset = newOffset }
              newOffset = offset initState + 1
```

</div>

There are a number of new functions in our definition.

The `L8.uncons` function takes the first element from a `ByteString`.

``` {.screen}
ghci> L8.uncons (L8.pack "foo")
Just ('f',Chunk "oo" Empty)
ghci> L8.uncons L8.empty
Nothing
```

Our `getState` function retrieves the current parsing state, while
`putState` replaces it. The `bail` function terminates parsing and
reports an error. The `(==>)` function chains parsers together. We will
cover each of these functions shortly.

::: {.TIP}
Hanging lambdas

The definition of `parseByte` has a visual style that we haven\'t
discussed before. It contains anonymous functions in which the
parameters and `->` sit at the end of a line, with the function\'s body
following on the next line.

This style of laying out an anonymous function doesn\'t have an official
name, so let\'s call it a \"hanging lambda\". Its main use is to make
room for more text in the body of the function. It also makes it more
visually clear that there\'s a relationship between one function and the
one that follows. Often, for instance, the result of the first function
is being passed as a parameter to the second.
:::

### Obtaining and modifying the parse state

Our `parseByte` function doesn\'t take the parse state as an argument.
Instead, it has to call `getState` to get a copy of the state, and
`putState` to replace the current state with a new one.

<div>

[Parse.hs]{.label}

``` {.haskell}
getState :: Parse ParseState
getState = Parse (\s -> Right (s, s))

putState :: ParseState -> Parse ()
putState s = Parse (\_ -> Right ((), s))
```

</div>

When reading these functions, recall that the left element of the tuple
is the result of a `Parse`, while the right is the current `ParseState`.
This makes it easier to follow what these functions are doing.

The `getState` function extracts the current parsing state, so that the
caller can access the string. The `putState` function replaces the
current parsing state with a new one. This becomes the state that will
be seen by the next function in the `(==>)` chain.

These functions let us move explicit state handling into the bodies of
only those functions that need it. Many functions don\'t need to know
what the current state is, and so they\'ll never call `getState` or
`putState`. This lets us write more compact code than our earlier
parser, which had to pass tuples around by hand. We will see the effect
in some of the code that follows.

We\'ve packaged up the details of the parsing state into the
`ParseState` type, and we work with it using accessors instead of
pattern matching. Now that the parsing state is passed around
implicitly, we gain a further benefit. If we want to add more
information to the parsing state, all we need to do is modify the
definition of `ParseState`, and the bodies of whatever functions need
the new information. Compared to our earlier parsing code, where all of
our state was exposed through pattern matching, this is much more
modular: the only code we affect is code that needs the new information.

### Reporting parse errors

We carefully defined our `Parse` type to accommodate the possibility of
failure. The `(==>)` combinator checks for a parse failure and stops
parsing if it runs into a failure. But we haven\'t yet introduced the
`bail` function, which we use to report a parse error.

<div>

[Parse.hs]{.label}

``` {.haskell}
bail :: String -> Parse a
bail err = Parse $ \s -> Left $
           "byte offset " ++ show (offset s) ++ ": " ++ err
```

</div>

After we call `bail`, `(==>)` will successfully pattern match on the
`Left` constructor that it wraps the error message with, and it will not
invoke the next parser in the chain. This will cause the error message
to percolate back through the chain of prior callers.

### Chaining parsers together

The `(==>)` function serves a similar purpose to our earlier `(>>?)`
function: it is \"glue\" that lets us chain functions together.

<div>

[Parse.hs]{.label}

``` {.haskell}
(==>) :: Parse a -> (a -> Parse b) -> Parse b
firstParser ==> secondParser = Parse chainedParser
  where chainedParser initState =
          case runParse firstParser initState of
            Left errMessage ->
                Left errMessage
            Right (firstResult, newState) ->
                runParse (secondParser firstResult) newState
```

</div>

The body of `(==>)` is interesting, and ever so slightly tricky. Recall
that the `Parse` type represents really a function inside a wrapper.
Since `(==>)` lets us chain two `Parse` values to produce a third, it
must return a function, in a wrapper.

The function doesn\'t really \"do\" much: it just creates a *closure* to
remember the values of `firstParser` and `secondParser`.

::: {.TIP}
Tip

A closure is simply the pairing of a function with its *environment*,
the bound variables that it can see. Closures are commonplace in
Haskell. For instance, the section `(+5)` is a closure. An
implementation must record the value `5` as the second argument to the
`(+)` operator, so that the resulting function can add `5` to whatever
value it is passed.
:::

This closure will not be unwrapped and applied until we apply `parse`.
At that point, it will be applied with a `ParseState`. It will apply
`firstParser` and inspect its result. If that parse fails, the closure
will fail too. Otherwise, it will pass the result of the parse and the
new `ParseState` to `secondParser`.

This is really quite fancy and subtle stuff: we\'re effectively passing
the `ParseState` down the chain of `Parse` values in a hidden argument.
(We\'ll be revisiting this kind of code in a few chapters, so don\'t
fret if that description seemed dense.)

Introducing functors
--------------------

We\'re by now thoroughly familiar with the `map` function, which applies
a function to every element of a list, returning a list of possibly a
different type.

``` {.screen}
ghci> map (+1) [1,2,3]
[2,3,4]
ghci> map show [1,2,3]
["1","2","3"]
ghci> :type map show
map show :: (Show a) => [a] -> [String]
```

This `map`-like activity can be useful in other instances. For example,
consider a binary tree.

<div>

[TreeMap.hs]{.label}

``` {.haskell}
data Tree a = Node (Tree a) (Tree a)
            | Leaf a
              deriving (Show)
```

</div>

If we want to take a tree of strings and turn it into a tree containing
the lengths of those strings, we could write a function to do this.

<div>

[TreeMap.hs]{.label}

``` {.haskell}
treeLengths (Leaf s) = Leaf (length s)
treeLengths (Node l r) = Node (treeLengths l) (treeLengths r)
```

</div>

Now that our eyes are attuned to looking for patterns that we can turn
into generally useful functions, we can see a possible case of this
here.

<div>

[TreeMap.hs]{.label}

``` {.haskell}
treeMap :: (a -> b) -> Tree a -> Tree b
treeMap f (Leaf a)   = Leaf (f a)
treeMap f (Node l r) = Node (treeMap f l) (treeMap f r)
```

</div>

As we might hope, `treeLengths` and `treeMap length` give the same
results.

``` {.screen}
ghci> :l TreeMap.hs
[1 of 1] Compiling Main             ( TreeMap.hs, interpreted )
Ok, one module loaded.
ghci> let tree = Node (Leaf "foo") (Node (Leaf "x") (Leaf "quux"))
ghci> treeLengths tree
Node (Leaf 3) (Node (Leaf 1) (Leaf 4))
ghci> treeMap length tree
Node (Leaf 3) (Node (Leaf 1) (Leaf 4))
ghci> treeMap (odd . length) tree
Node (Leaf True) (Node (Leaf True) (Leaf False))
```

Haskell provides a well-known type class to further generalise
`treeMap`. This type class is named `Functor`, and it defines one
function, `fmap`.

<div>

[TreeMap.hs]{.label}

``` {.haskell}
class Functor f where
    fmap :: (a -> b) -> f a -> f b
```

</div>

We can think of `fmap` as a kind of *lifting* function, as we introduced
in [the section called \"Avoiding boilerplate with
lifting\"](9-a-library-for-searching-the-file-system.org::*Avoiding boilerplate with lifting)
function over ordinary values `a -> b` and lifts it to become a function
over a type whose constructor takes one type parameter `f a -> f b`,
where `f` is the type.

If we substitute `Tree` for the type variable `f`, for example, the type
of `fmap` is identical to the type of `treeMap`, and in fact we can use
`treeMap` as the implementation of `fmap` over Trees.

<div>

[TreeMap.hs]{.label}

``` {.haskell}
instance Functor Tree where
    fmap = treeMap
```

</div>

`map` is actually the implementation of `fmap` for lists.

<div>

[TreeMap.hs]{.label}

``` {.haskell}
instance Functor [] where
    fmap = map
```

</div>

So we can use `fmap` over different types.

``` {.screen}
ghci> fmap length ["foo","quux"]
[3,4]
ghci> fmap length (Node (Leaf "Livingstone") (Leaf "I presume"))
Node (Leaf 11) (Leaf 9)
```

The Prelude defines instances of `Functor` for several common types,
notably lists and `Maybe`.

<div>

[TreeMap.hs]{.label}

``` {.haskell}
instance Functor Maybe where
    fmap _ Nothing  = Nothing
    fmap f (Just x) = Just (f x)
```

</div>

The instance for `Maybe` makes it particularly clear what an `fmap`
implementation needs to do. The implementation must have a sensible
behaviour for each of a type\'s constructors. If a value is wrapped in
`Just`, for example, the `fmap` implementation calls the function on the
unwrapped value, then rewraps it in `Just`.

The definition of `Functor` imposes a few obvious restrictions on what
we can do with `fmap`. For example, we can only make instances of
`Functor` from types that have exactly one type parameter.

We can\'t write an `fmap` implementation for `Either a b` or `(a, b)`,
for example, because these have two type parameters. We also can\'t
write one for `Bool` or `Int`, as they have no type parameters.

In addition, we can\'t place any constraints on our type definition.
What does this mean? To illustrate, let\'s first look at a normal `data`
definition and its `Functor` instance.

<div>

[ValidFunctor.hs]{.label}

``` {.haskell}
data Foo a = Foo a

instance Functor Foo where
    fmap f (Foo a) = Foo (f a)
```

</div>

When we define a new type, we can add a type constraint just after the
`data` keyword as follows.

<div>

[ValidFunctor.hs]{.label}

``` {.haskell}
data Eq a => Bar a = Bar a

instance Functor Bar where
    fmap f (Bar a) = Bar (f a)
```

</div>

This says that we can only put a type `a` into a `Foo` if `a` is a
member of the `Eq` type class. However, the constraint renders it
impossible to write a `Functor` instance for `Bar`.

``` {.screen}
ghci> :load ValidFunctor
[1 of 1] Compiling Main             ( ValidFunctor.hs, interpreted )

ValidFunctor.hs:1:6: error:
    Illegal datatype context (use DatatypeContexts): Eq a =>
  |
1 | data Eq a => Bar a = Bar a
  |      ^^^^
Failed, no modules loaded.
```

### Constraints on type definitions are bad

Adding a constraint to a type definition is essentially never a good
idea. It has the effect of forcing you to add type constraints to
*every* function that will operate on values of that type. Let\'s say
that we need a stack data structure that we want to be able to query to
see whether its elements obey some ordering. Here\'s a naive definition
of the data type.

<div>

[TypeConstraint.hs]{.label}

``` {.haskell}
data (Ord a) => OrdStack a = Bottom
                           | Item a (OrdStack a)
                             deriving (Show)
```

</div>

If we want to write a function that checks the stack to see whether it
is increasing (i.e. every element is bigger than the element below it),
we\'ll obviously need an `Ord` constraint to perform the pairwise
comparisons.

<div>

[TypeConstraint.hs]{.label}

``` {.haskell}
isIncreasing :: (Ord a) => OrdStack a -> Bool
isIncreasing (Item a rest@(Item b _))
    | a < b     = isIncreasing rest
    | otherwise = False
isIncreasing _  = True
```

</div>

However, because we wrote the type constraint on the type definition,
that constraint ends up infecting places where it isn\'t needed: we need
to add the `Ord` constraint to `push`, which does not care about the
ordering of elements on the stack.

<div>

[TypeConstraint.hs]{.label}

``` {.haskell}
push :: (Ord a) => a -> OrdStack a -> OrdStack a
push a s = Item a s
```

</div>

Try removing that `Ord` constraint above, and the definition of `push`
will fail to type-check.

This is why our attempt to write a `Functor` instance for `Bar` failed
earlier: it would have required an `Eq` constraint to somehow get
retroactively added to the signature of `fmap`.

Now that we\'ve tentatively established that putting a type constraint
on a type definition is a misfeature of Haskell, what\'s a more sensible
alternative? The answer is simply to omit type constraints from type
definitions, and instead place them on the functions that need them.

In this example, we can drop the `Ord` constraints from `OrdStack` and
`push`. It needs to stay on `isIncreasing`, which otherwise couldn\'t
call `(<)`. We now have the constraints where they actually matter. This
has the further benefit of making the type signatures better document
the true requirements of each function.

Several Haskell types follow this pattern. The `Map` type in the
`Data.Map` module requires that its keys be ordered, but the type itself
does not have such a constraint. The constraint is expressed on
functions like `insert`, where it\'s actually needed, and not on `size`,
where ordering isn\'t used.

### Infix use of `fmap`

Quite often, you\'ll see `fmap` called as an operator.

``` {.screen}
ghci> (1+) `fmap` [1,2,3] ++ [4,5,6]
[2,3,4,4,5,6]
```

Perhaps strangely, plain old `map` is almost never used in this way.

One possible reason for the stickiness of the `fmap`-as-operator meme is
that this use lets us omit parentheses from its second argument. Fewer
parentheses leads to reduced mental juggling while reading a function.

``` {.screen}
ghci> fmap (1+) ([1,2,3] ++ [4,5,6])
[2,3,4,5,6,7]
```

There\'s also a `(<$>)` operator that is an alias for `fmap`. The `$` in
its name appeals to the similarity between applying a function to its
arguments (using the `($)` operator) and lifting a function into a
functor. We will see that this works well for parsing when we return to
the code that we have been writing.

### Thinking more about functors

We\'ve made a few implicit assumptions about how functors ought to work.
It\'s helpful to make these explicit and to think of them as rules to
follow, because this lets us treat functors as uniform, well-behaved
objects. We have only two rules to remember, and they\'re simple.

Our first rule is that a functor must preserve *identity*. That is,
applying `fmap id` to a value should give us back an identical value.

``` {.screen}
ghci> fmap id (Node (Leaf "a") (Leaf "b"))
Node (Leaf "a") (Leaf "b")
```

Our second rule is that functors must be *composable*. That is,
composing two uses of `fmap` should give the same result as one `fmap`
with the same functions composed.

``` {.screen}
ghci> (fmap even . fmap length) (Just "twelve")
Just True
ghci> fmap (even . length) (Just "twelve")
Just True
```

Another way of looking at these two rules is that a functor must
preserve *shape*. The structure of a collection should not be affected
by a functor; only the values that it contains should change.

``` {.screen}
ghci> fmap odd (Just 1)
Just True
ghci> fmap odd Nothing
Nothing
```

If you\'re writing a `Functor` instance, it\'s useful to keep these
rules in mind, and indeed to test them, because the compiler can\'t
check the rules we\'ve listed above. On the other hand, if you\'re
simply *using* functors, the rules are \"natural\" enough that there\'s
no need to memorise them. They just formalize a few intuitive notions of
\"do what I mean\". Here is a pseudocode representation of the expected
behavior.

<div>

[FunctorLaws.hs]{.label}

``` {.haskell}
fmap id      == id
fmap (f . g) == fmap f . fmap g
```

</div>

Writing a functor instance for `Parse`
--------------------------------------

For the types we have surveyed so far, the behaviour we ought to expect
of `fmap` has been obvious. This is a little less clear for `Parse`, due
to its complexity. A reasonable guess is that the function we\'re
\~fmap\~ping should be applied to the current result of a parse, and
leave the parse state untouched.

<div>

[Parse.hs]{.label}

``` {.haskell}
instance Functor Parse where
    fmap f parser = parser ==> \result ->
                    identity (f result)
```

</div>

This definition is easy to read, so let\'s perform a few quick
experiments to see if we\'re following our rules for functors.

First, we\'ll check that identity is preserved. Let\'s try this first on
a parse that ought to fail: parsing a byte from an empty string
(remember that `(<$>)` is `fmap`).

``` {.screen}
ghci> parse parseByte L.empty
Left "byte offset 0: no more input"
ghci> parse (id <$> parseByte) L.empty
Left "byte offset 0: no more input"
```

Good. Now for a parse that should succeed.

``` {.screen}
ghci> input = L8.pack "foo"
ghci> L.head input
102
ghci> parse parseByte input
Right 102
ghci> parse (id <$> parseByte) input
Right 102
```

By inspecting the results above, we can also see that our functor
instance is obeying our second rule, that of preserving shape. Failure
is preserved as failure, and success as success.

Finally, we\'ll ensure that composability is preserved.

``` {.screen}
ghci> parse ((chr . fromIntegral) <$> parseByte) input
Right 'f'
ghci> parse (chr <$> fromIntegral <$> parseByte) input
Right 'f'
```

On the basis of this brief inspection, our `Functor` instance appears to
be well behaved.

Using functors for parsing
--------------------------

All this talk of functors had a purpose: they often let us write tidy,
expressive code. Recall the `parseByte` function that we introduced
earlier. In recasting our PGM parser to use our new parser
infrastructure, we\'ll often want to work with ASCII characters instead
of `Word8` values.

While we could write a `parseChar` function that has a similar structure
to `parseByte`, we can now avoid this code duplication by taking
advantage of the functor nature of `Parse`. Our functor takes the result
of a parse and applies a function to it, so what we need is a function
that turns a `Word8` into a `Char`.

<div>

[Parse.hs]{.label}

``` {.haskell}
-- import Data.Char

w2c :: Word8 -> Char
w2c = chr . fromIntegral

parseChar :: Parse Char
parseChar = w2c <$> parseByte
```

</div>

We can also use functors to write a compact \"peek\" function. This
returns `Nothing` if we\'re at the end of the input string. Otherwise,
it returns the next character without consuming it (i.e. it inspects,
but doesn\'t disturb, the current parsing state).

<div>

[Parse.hs]{.label}

``` {.haskell}
peekByte :: Parse (Maybe Word8)
peekByte = (fmap fst . L.uncons . string) <$> getState
```

</div>

The same lifting trick that let us define `parseChar` lets us write a
compact definition for `peekChar`.

<div>

[Parse.hs]{.label}

``` {.haskell}
peekChar :: Parse (Maybe Char)
peekChar = fmap w2c <$> peekByte
```

</div>

Notice that `peekByte` and `peekChar` each make two calls to `fmap`, one
of which is disguised as `(<$>)`. This is necessary because the type
`Parse (Maybe a)` is a functor within a functor. We thus have to lift a
function twice to \"get it into\" the inner functor.

Finally, we\'ll write another generic combinator, which is the `Parse`
analogue of the familiar `takeWhile`: it consumes its input while its
predicate returns `True`.

<div>

[Parse.hs]{.label}

``` {.haskell}
parseWhile :: (Word8 -> Bool) -> Parse [Word8]
parseWhile p = (fmap p <$> peekByte) ==> \mp ->
               if mp == Just True
               then parseByte ==> \b ->
                    (b:) <$> parseWhile p
               else identity []
```

</div>

Once again, we\'re using functors in several places (doubled up, when
necessary) to reduce the verbosity of our code. Here\'s a rewrite of the
same function in a more direct style that does not use functors.

<div>

[Parse.hs]{.label}

``` {.haskell}
parseWhileVerbose p =
    peekByte ==> \mc ->
    case mc of
      Nothing -> identity []
      Just c | p c ->
                 parseByte ==> \b ->
                 parseWhileVerbose p ==> \bs ->
                 identity (b:bs)
             | otherwise ->
                 identity []
```

</div>

The more verbose definition is likely easier to read when you are less
familiar with functors. However, use of functors is sufficiently common
in Haskell code that the more compact representation should become
second nature (both to read and to write) fairly quickly.

Rewriting our PGM parser
------------------------

With our new parsing code, what does the raw PGM parsing function look
like now?

<div>

[Parse.hs]{.label}

``` {.haskell}
-- import PNM

parseRawPGM =
    parseWhileWith w2c notWhite ==> \header -> skipSpaces ==>&
    assert (header == "P5") "invalid raw header" ==>&
    parseNat ==> \width -> skipSpaces ==>&
    parseNat ==> \height -> skipSpaces ==>&
    parseNat ==> \maxGrey ->
    parseByte ==>&
    parseBytes (width * height) ==> \bitmap ->
    identity (Greymap width height maxGrey bitmap)
  where notWhite = (`notElem` " \r\n\t")
```

</div>

This definition makes use of a few more helper functions that we present
here, following a pattern that should by now be familiar.

<div>

[Parse.hs]{.label}

``` {.haskell}
parseWhileWith :: (Word8 -> a) -> (a -> Bool) -> Parse [a]
parseWhileWith f p = fmap f <$> parseWhile (p . f)

parseNat :: Parse Int
parseNat = parseWhileWith w2c isDigit ==> \digits ->
           if null digits
           then bail "no more input"
           else let n = read digits
                in if n < 0
                   then bail "integer overflow"
                   else identity n

(==>&) :: Parse a -> Parse b -> Parse b
p ==>& f = p ==> \_ -> f

skipSpaces :: Parse ()
skipSpaces = parseWhileWith w2c isSpace ==>& identity ()

assert :: Bool -> String -> Parse ()
assert True  _   = identity ()
assert False err = bail err
```

</div>

The `(==>&)` combinator chains parsers like `(==>)`, but the right hand
side ignores the result from the left. The `assert` function lets us
check a property, and abort parsing with a useful error message if the
property is `False`.

Notice how few of the functions that we have written make any reference
to the current parsing state. Most notably, where our old `parseP5`
function explicitly passed two-tuples down the chain of dataflow, all of
the state management in `parseRawPGM` is hidden from us.

Of course, we can\'t completely avoid inspecting and modifying the
parsing state. Here\'s a case in point, the last of the helper functions
needed by `parseRawPGM`.

<div>

[Parse.hs]{.label}

``` {.haskell}
parseBytes :: Int -> Parse L.ByteString
parseBytes n =
    getState ==> \st ->
    let n' = fromIntegral n
        (h, t) = L.splitAt n' (string st)
        st' = st { offset = offset st + L.length h, string = t }
    in putState st' ==>&
       assert (L.length h == n') "end of input" ==>&
       identity h
```

</div>

Future directions
-----------------

Our main theme in this chapter has been abstraction. We found passing
explicit state down a chain of functions to be unsatisfactory, so we
abstracted this detail away. We noticed some recurring needs as we
worked out our parsing code, and abstracted those into common functions.
Along the way, we introduced the notion of a functor, which offers a
generalised way to map over a parameterised type.

We will revisit parsing in [Chapter 16, *Using
Parsec*](14-using-parsec.org), to discuss Parsec, a widely used and
flexible parsing library. And in [Chapter 14, Monads](15-monads.org), we
will return to our theme of abstraction, where we will find that much of
the code that we have developed in this chapter can be further
simplified by the use of monads.

For efficiently parsing binary data represented as a `ByteString`, a
number of packages are available via the Hackage package database. At
the time of writing, the most popular is named `binary`, which is easy
to use and offers high performance.

Exercises
---------

1.  Write a parser for \"plain\" PGM files.

2.  In our description of \"raw\" PGM files, we omitted a small detail.
    If the \"maximum grey\" value in the header is less than 256, each
    pixel is represented by a single byte. However, it can range up to
    65535, in which case each pixel will be represented by two bytes, in
    big endian order (most significant byte first).

    Rewrite the raw PGM parser to accommodate both the single and
    double-byte pixel formats.

3.  Extend your parser so that it can identify a raw or plain PGM file,
    and parse the appropriate file type.
