Chapter 14. Monads
==================

Introduction
------------

In [Chapter 7, *I/O*](7-io.org) and [Chapter 14, Using
Parsec](14-using-parsec.org) we talked about the `IO` and `GenParser`
monads respectively, but we intentionally kept the discussion narrowly
focused on how to use them. We didn\'t discuss what a monad *is*.

When we had practical problems to solve in earlier chapters, we
introduced structures that, as we will soon see, are actually monads. We
aim to show you that a monad is often an *obvious* and *useful* tool to
help solve a problem. We\'ll define a few monads in this chapter, to
show how easy it is.

Revisiting earlier code examples
--------------------------------

### Maybe chaining

Let\'s take another look at the `parseP5` function that we wrote in
[Chapter 10, *Code case study: parsing a binary data
format*](10-parsing-a-binary-data-format.org).

<div>

[PNM.hs]{.label}

``` {.haskell}
matchHeader :: L.ByteString -> L.ByteString -> Maybe L.ByteString

-- "nat" here is short for "natural number"
getNat :: L.ByteString -> Maybe (Int, L.ByteString)

getBytes :: Int -> L.ByteString
         -> Maybe (L.ByteString, L.ByteString)

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

When we introduced this function, it threatened to march off the right
side of the page if it got much more complicated. We brought the
staircasing under control using the `(>>?)` function.

<div>

[PNM.hs]{.label}

``` {.haskell}
(>>?) :: Maybe a -> (a -> Maybe b) -> Maybe b
Nothing >>? _ = Nothing
Just v  >>? f = f v
```

</div>

We carefully chose the type of `(>>?)` to let us chain together
functions that return a `Maybe` value. So long as the result type of one
function matches the parameter of the next, we can chain functions
returning `Maybe` together indefinitely. The body of `(>>?)` hides the
details of whether the chain of functions we build is short-circuited
somewhere, due to one returning `Nothing`, or completely evaluated.

### Implicit state

Useful as `(>>?)` was for cleaning up the structure of `parseP5`, we had
to incrementally consume pieces of a string as we parsed it. This forced
us to pass the current value of the string down our chain of \~Maybe\~s,
wrapped up in a tuple. Each function in the chain put a result into one
element of the tuple, and the unconsumed remainder of the string into
the other.

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

Once again, we were faced with a pattern of repeated behaviour: consume
some string, return a result, and return the remaining string for the
next function to consume. However, this pattern was more insidious: if
we wanted to pass another piece of information down the chain, we\'d
have to modify nearly every element of the chain, turning each two-tuple
into a three-tuple!

We addressed this by moving the responsibility for managing the current
piece of string out of the individual functions in the chain, and into
the function that we used to chain them together.

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

We also hid the details of the parsing state in the `ParseState` type.
Even the `getState` and `putState` functions don\'t inspect the parsing
state, so any modification to `ParseState` will have no effect on any
existing code.

Looking for shared patterns
---------------------------

When we look at the above examples in detail, they don\'t seem to have
much in common. Obviously, they\'re both concerned with chaining
functions together, and with hiding details to let us write tidier code.
However, let\'s take a step back and consider them in *less* detail.

First, let\'s look at the type definitions.

``` {.haskell}
data Maybe a = Nothing
             | Just a
```

<div>

[Parse.hs]{.label}

``` {.haskell}
newtype Parse a = Parse {
    runParse :: ParseState -> Either String (a, ParseState)
}
```

</div>

The common feature of these two types is that each has a single type
parameter on the left of the definition, which appears somewhere on the
right. These are thus generic types, which know nothing about their
payloads.

Next, we\'ll examine the chaining functions that we wrote for the two
types.

``` {.screen}
ghci> :l Parse.hs
[1 of 2] Compiling PNM              ( PNM.hs, interpreted )
[2 of 2] Compiling Parse            ( Parse.hs, interpreted )
Ok, two modules loaded.
ghci> :type (>>?)
(>>?) :: Maybe a -> (a -> Maybe b) -> Maybe b
ghci> :type (==>)
(==>) :: Parse a -> (a -> Parse b) -> Parse b
```

These functions have strikingly similar types. If we were to turn those
type constructors into a type variable, we\'d end up with a single more
abstract type.

``` {.haskell}
chain :: m a -> (a -> m b) -> m b
```

Finally, in each case we have a function that takes a \"plain\" value,
and \"injects\" it into the target type. For `Maybe`, this function is
simply the value constructor `Just`, but the injector for `Parse` is
more complicated.

<div>

[Parse.hs]{.label}

``` {.haskell}
identity :: a -> Parse a
identity a = Parse (\s -> Right (a, s))
```

</div>

Again, it\'s not the details or complexity that we\'re interested in,
it\'s the fact that each of these types has an \"injector\" function,
which looks like this.

``` {.haskell}
inject :: a -> m a
```

It is *exactly* these three properties, and a few rules about how we can
use them together, that define a monad in Haskell. Let\'s revisit the
above list in condensed form.

-   A type constructor `m`.
-   A function of type `m a -> (a -> m b) -> m b` for chaining the
    output of one function into the input of another.
-   A function of type `a -> m a` for injecting a normal value into the
    chain, i.e. it wraps a type a with the type constructor `m`.

The properties that make the `Maybe` type a monad are its type
constructor `Maybe a`, our chaining function `(>>?)`, and the injector
function `Just`.

For `Parse`, the corresponding properties are the type constructor
`Parse a`, the chaining function `(==>)`, and the injector function
`identity`.

We have intentionally said nothing about how the chaining and injection
functions of a monad should behave, and that\'s because this almost
doesn\'t matter. In fact, monads are ubiquitous in Haskell code
precisely because they are so simple. Many common programming patterns
have a monadic structure: passing around implicit data, or
short-circuiting a chain of evaluations if one fails, to choose but two.

The Monad type class
--------------------

We can capture the notions of chaining and injection, and the types that
we want them to have, in a Haskell type class. The standard `Prelude`
already defines just such a type class, named `Monad`.

``` {.haskell}
class Applicative m => Monad m where
    -- chain
    (>>=) :: m a -> (a -> m b) -> m b
    -- inject
    return :: a -> m a
```

As you can see every monad is also an applicative functor in the same
way that every functor is an applicative functor. These terms as well as
their relationship are borrowed from a branch of mathematics called
category theory which is a source of inspiration for a lot of Haskell
design since Phillip Wadler, one of its authors, suggested in his paper
[Comprehending
monads](https://www.cambridge.org/core/journals/mathematical-structures-in-computer-science/article/div-classtitlecomprehending-monadsa-hreffn01-ref-typefnspan-classsupspanadiv/8678CDA48EB1DF29B9C2C9943AF6BC29)
to follow Eugenio Moggi\'s idea of using monads to structure programs.

Here, `(>>=)` is our chaining function. We\'ve already been introduced
to it in [the section called \"Sequencing\"](7-io.org::*Sequencing)
referred to as \"bind\", as it binds the result of the computation on
the left to the parameter of the one on the right.

Our injection function is `return`. As we noted in [the section called
\"The True Nature of Return\"](7-io.org::*The True Nature of Return)
name `return` is a little unfortunate. That name is widely used in
imperative languages, where it has a fairly well understood meaning. In
Haskell, its behaviour is much less constrained. In particular, calling
`return` in the middle of a chain of functions won\'t cause the chain to
exit early. A useful way to link its behavior to its name is that it
*returns* a pure value (of type `a`) into a monad (of type `m a`).

While `(>>=)` and `return` are the core functions of the `Monad` type
class, it also defines two other functions. The first is `(>>)`. Like
`(>>=)`, it performs chaining, but it ignores the value on the left.

<div>

[Maybe.hs]{.label}

``` {.haskell}
(>>) :: m a -> m b -> m b
a >> f = a >>= \_ -> f
```

</div>

We use this function when we want to perform actions in a certain order,
but don\'t care what the result of one is. This might seem pointless:
why would we not care what a function\'s return value is? Recall,
though, that we defined a `(==>&)` combinator earlier to express exactly
this. Alternatively, consider a function like `print`, which provides a
placeholder result that we do not need to inspect.

``` {.screen}
ghci> :type print "foo"
print "foo" :: IO ()
```

If we use plain `(>>=)`, we have to provide as its right hand side a
function that ignores its argument.

``` {.screen}
ghci> print "foo" >>= \_ -> print "bar"
"foo"
"bar"
```

But if we use `(>>)`, we can omit the needless function.

``` {.screen}
ghci> print "baz" >> print "quux"
"baz"
"quux"
```

As we showed above, the default implementation of `(>>)` is defined in
terms of `(>>=)`.

The second non-core `Monad` function is `fail`, which takes an error
message and does something to make the chain of functions fail.

``` {.haskell}
fail :: String -> m a
fail = error
```

::: {.WARNING}
Beware of fail

Many `Monad` instances don\'t override the default implementation of
`fail` that we show here, so in those monads, `fail` uses `error`.
Calling `error` is usually highly undesirable, since it throws an
exception that callers either cannot catch or will not expect.

Even if you know that right now you\'re executing in a monad that has
`fail` do something more sensible, we still recommend avoiding it. It\'s
far too easy to cause yourself a problem later when you refactor your
code and forget that a previously safe use of `fail` might be dangerous
in its new context.
:::

To revisit the parser that we developed in [Chapter 10, *Code case
study: parsing a binary data
format*](10-parsing-a-binary-data-format.org), here is its `Monad`
instance.

<div>

[Parse.hs]{.label}

``` {.haskell}
instance Functor Parse where
    fmap = liftM

instance Applicative Parse where
    pure = identity
    (<*>) = ap

instance Monad Parse where
    return = pure
    (>>=) = (==>)
    fail = bail
```

</div>

We are going to see `liftM` in [the section called \"Mixing pure and
monadic code\"](15-monads.org::*Mixing pure and monadic code) [the
section called \"Generalised
lifting\"](16-programming-with-monads.org::*Generalised lifting) they
are \"aliases\" to known functions.

Notice that `pure` is defined as our `identity` function and `return` is
defined as `pure`. You can actually get rid of the `return` definition
and use `pure` instead but there\'s enough code in the wild using
`return` to cover it here.

And now, a jargon moment
------------------------

There are a few terms of jargon around monads that you may not be
familiar with. These aren\'t formal terms, but they\'re in common use,
so it\'s helpful to know about them.

-   \"Monadic\" simply means \"pertaining to monads\". A monadic *type*
    is an instance of the `Monad` type class; a monadic *value* has a
    monadic type.
-   When we say that a type \"is a monad\", this is really a shorthand
    way of saying that it\'s an instance of the `Monad` type class.
    Being an instance of `Monad` gives us the necessary monadic triple
    of type constructor, injection function, and chaining function.
-   In the same way, a reference to \"the Foo monad\" implies that
    we\'re talking about the type named `Foo`, and that it\'s an
    instance of `Monad`.
-   An \"action\" is another name for a monadic value. This use of the
    word probably originated with the introduction of monads for I/O,
    where a monadic value like `print "foo"` can have an observable side
    effect. A function with a monadic return type might also be referred
    to as an action, though this is a little less common.

Using a new monad: show your work!
----------------------------------

In our introduction to monads, we showed how some pre-existing code was
already monadic in form. Now that we are beginning to grasp what a monad
is, and we\'ve seen the `Monad` type class, let\'s build a monad with
foreknowledge of what we\'re doing. We\'ll start out by defining its
interface, then we\'ll put it to use. Once we have those out of the way,
we\'ll finally build it.

Pure Haskell code is wonderfully clean to write, but of course it can\'t
perform I/O. Sometimes, we\'d like to have a record of decisions we
made, without writing log information to a file. Let\'s develop a small
library to help with this.

Recall the `globToRegex` function that we developed in [the section
called \"Translating a glob pattern into a regular
expression\"](8-efficient-file-processing-regular-expressions-and-file-name-matching.org::*Translating a glob pattern into a regular expression)
We will modify it so that it keeps a record of each of the special
pattern sequences that it translates. We are revisiting familiar
territory for a reason: it lets us compare non-monadic and monadic
versions of the same code.

To start off, we\'ll wrap our result type with a `Logger` type
constructor.

<div>

[Logger.hs]{.label}

``` {.haskell}
globToRegex :: String -> Logger String
```

</div>

### Information hiding

We\'ll intentionally keep the internals of the `Logger` module abstract.

<div>

[Logger.hs]{.label}

``` {.haskell}
module Logger
    (
      Logger
    , Log
    , runLogger
    , record
    ) where

import Control.Monad (ap)
```

</div>

Hiding the details like this has two benefits: it grants us considerable
flexibility in how we implement our monad, and more importantly, it
gives users a simple interface.

Our `Logger` type is purely a *type* constructor. We don\'t export the
*value* constructor that a user would need to create a value of this
type. All they can use `Logger` for is writing type signatures.

The `Log` type is just a synonym for a list of strings, to make a few
signatures more readable. We use a list of strings to keep the
implementation simple.

<div>

[Logger.hs]{.label}

``` {.haskell}
type Log = [String]
```

</div>

Instead of giving our users a value constructor, we provide them with a
function, `runLogger`, that evaluates a logged action. This returns both
the result of an action and whatever was logged while the result was
being computed.

<div>

[Logger.hs]{.label}

``` {.haskell}
runLogger :: Logger a -> (a, Log)
```

</div>

### Controlled escape

The `Monad` type class doesn\'t provide any means for values to escape
their monadic shackles. We can inject a value into a monad using
`return`. We can extract a value from a monad using `(>>=)` but the
function on the right, which can see an unwrapped value, has to wrap its
own result back up again.

Most monads have one or more `runLogger`-like functions. The notable
exception is of course `IO`, which we usually only escape from by
exiting a program.

A monad execution function runs the code inside the monad and unwraps
its result. Such functions are usually the only means provided for a
value to escape from its monadic wrapper. The author of a monad thus has
complete control over how whatever happens inside the monad gets out.

Some monads have several execution functions. In our case, we can
imagine a few alternatives to `runLogger`: one might only return the log
messages, while another might return just the result and drop the log
messages.

### Leaving a trace

When executing inside a `Logger` action, user code calls `record` to
record something.

<div>

[Logger.hs]{.label}

``` {.haskell}
record :: String -> Logger ()
```

</div>

Since recording occurs in the plumbing of our monad, our action\'s
result supplies no information.

Usually, a monad will provide one or more helper functions like our
`record`. These are our means for accessing the special behaviors of
that monad.

Our module also defines the `Monad` instance for the `Logger` type.
These definitions are all that a client module needs in order to be able
to use this monad.

Here is a preview, in `ghci`, of how our monad will behave.

``` {.screen}
ghci> simple = return True :: Logger Bool
ghci> runLogger simple
(True,[])
```

When we run the logged action using `runLogger`, we get back a pair. The
first element is the result of our code; the second is the list of items
logged while the action executed. We haven\'t logged anything, so the
list is empty. Let\'s fix that.

``` {.screen}
ghci> runLogger (record "hi mom!" >> return 3.1337)
(3.1337,["hi mom!"])
```

### Using the `Logger` monad

Here\'s how we kick off our glob-to-regexp conversion inside the
`Logger` monad.

<div>

[Logger.hs]{.label}

``` {.haskell}
globToRegex cs =
    globToRegex' cs >>= \ds ->
    return ('^':ds)
```

</div>

There are a few coding style issues worth mentioning here. The body of
the function starts on the line after its name. By doing this, we gain
some horizontal white space. We\'ve also \"hung\" the parameter of the
anonymous function at the end of the line. This is common practice in
monadic code.

Remember the type of `(>>=)`: it extracts the value on the left from its
`Logger` wrapper, and passes the unwrapped value to the function on the
right. The function on the right must, in turn, wrap *its* result with
the `Logger` wrapper. This is exactly what `return` does: it takes a
pure value, and wraps it in the monad\'s type constructor.

``` {.screen}
ghci> :type (>>=)
(>>=) :: Monad m => m a -> (a -> m b) -> m b
ghci> :type (globToRegex "" >>=)
(globToRegex "" >>=) :: (String -> Logger b) -> Logger b
```

Even when we write a function that does almost nothing, we must call
`return` to wrap the result with the correct type.

<div>

[Logger.hs]{.label}

``` {.haskell}
globToRegex' :: String -> Logger String
globToRegex' "" = return "$"
```

</div>

When we call `record` to save a log entry, we use `(>>)` instead of
`(>>=)` to chain it with the following action.

<div>

[Logger.hs]{.label}

``` {.haskell}
globToRegex' ('?':cs) =
    record "any" >>
    globToRegex' cs >>= \ds ->
    return ('.':ds)
```

</div>

Recall that this is a variant of `(>>=)` that ignores the result on the
left. We know that the result of `record` will always be `()`, so
there\'s no point in capturing it.

We can use `do` notation, which we first encountered in [the section
called \"Sequencing\"](7-io.org::*Sequencing)

<div>

[Logger.hs]{.label}

``` {.haskell}
globToRegex' ('*':cs) = do
    record "kleene star"
    ds <- globToRegex' cs
    return (".*" ++ ds)
```

</div>

The choice of `do` notation versus explicit `(>>=)` with anonymous
functions is mostly a matter of taste, though almost everyone\'s taste
is to use `do` notation for anything longer than about two lines. There
is one significant difference between the two styles, though, which
we\'ll return to in [the section called \"Desugaring of do
blocks\"](15-monads.org::*Desugaring of do blocks)

Parsing a character class mostly follows the same pattern that we\'ve
already seen.

<div>

[Logger.hs]{.label}

``` {.haskell}
globToRegex' ('[':'!':c:cs) =
    record "character class, negative" >>
    charClass cs >>= \ds ->
    return ("[^" ++ c : ds)
globToRegex' ('[':c:cs) =
    record "character class" >>
    charClass cs >>= \ds ->
    return ("[" ++ c : ds)
globToRegex' ('[':_) =
    fail "unterminated character class"
```

</div>

Mixing pure and monadic code
----------------------------

Based on the code we\'ve seen so far, monads seem to have a substantial
shortcoming: the type constructor that wraps a monadic value makes it
tricky to use a normal, pure function on a value trapped inside a
monadic wrapper. Here\'s a simple illustration of the apparent problem.
Let\'s say we have a trivial piece of code that runs in the `Logger`
monad and returns a string.

``` {.screen}
ghci> m = return "foo" :: Logger String
```

If we want to find out the length of that string, we can\'t simply call
`length`: the string is wrapped, so the types don\'t match up.

``` {.screen}
ghci> length m

<interactive>:1:7: error:
    • No instance for (Foldable Logger) arising from a use of ‘length’
    • In the expression: length m
      In an equation for ‘it’: it = length m
```

What we\'ve done so far to work around this is something like the
following.

``` {.screen}
ghci> :type m >>= \s -> return (length s)
m >>= \s -> return (length s) :: Logger Int
```

We use `(>>=)` to unwrap the string, then write a small anonymous
function that calls `length` and rewraps the result using `return`.

This need crops up often in Haskell code. We won\'t be surprised to
learn that a shorthand already exists: we use the *lifting* technique
that we introduced for functors in [the section called \"Introducing
functors\"](10-parsing-a-binary-data-format.org::*Introducing functors)
into a functor usually involves unwrapping the value inside the functor,
calling the function on it, and rewrapping the result with the same
constructor.

We do exactly the same thing with a monad. Because the `Monad` type
class already provides the `(>>=)` and `return` functions that know how
to unwrap and wrap a value, the `liftM` function doesn\'t need to know
any details of a monad\'s implementation.

<div>

[Logger.hs]{.label}

``` {.haskell}
liftM :: Monad m => (a -> b) -> m a -> m b
liftM f m = m >>= \i ->
            return (f i)
```

</div>

When we declare a type to be an instance of the `Functor` type class, we
have to write our own version of `fmap` specially tailored to that type.
By contrast, `liftM` doesn\'t need to know anything of a monad\'s
internals, because they\'re abstracted by `(>>=)` and `return`. We only
need to write it once, with the appropriate type constraint.

The `liftM` function is predefined for us in the standard
`Control.Monad` module.

To see how `liftM` can help readability, we\'ll compare two otherwise
identical pieces of code. First, the familiar kind that does not use
`liftM`.

<div>

[Logger.hs]{.label}

``` {.haskell}
charClass_wordy (']':cs) =
    globToRegex' cs >>= \ds ->
    return (']':ds)
charClass_wordy (c:cs) =
    charClass_wordy cs >>= \ds ->
    return (c:ds)
```

</div>

Now we can eliminate the `(>>=)` and anonymous function cruft with
`liftM`.

<div>

[Logger.hs]{.label}

``` {.haskell}
charClass (']':cs) = (']':) `liftM` globToRegex' cs
charClass (c:cs) = (c:) `liftM` charClass cs
```

</div>

As with `fmap`, we often use `liftM` in infix form. An easy way to read
such an expression is \"apply the pure function on the left to the
result of the monadic action on the right\".

The `liftM` function is so useful that `Control.Monad` defines several
variants, which combine longer chains of actions. We can see one in the
last clause of our `globToRegex'` function.

<div>

[Logger.hs]{.label}

``` {.haskell}
globToRegex' (c:cs) = liftM2 (++) (escape c) (globToRegex' cs)

escape :: Char -> Logger String
escape c
    | c `elem` regexChars = record "escape" >> return ['\\',c]
    | otherwise           = return [c]
  where regexChars = "\\+()^$.{}]|"
```

</div>

The `liftM2` function that we use above is defined as follows.

<div>

[Logger.hs]{.label}

``` {.haskell}
liftM2 :: (Monad m) => (a -> b -> c) -> m a -> m b -> m c
liftM2 f m1 m2 =
    m1 >>= \a ->
    m2 >>= \b ->
    return (f a b)
```

</div>

It executes the first action, then the second, then combines their
results using the pure function `f`, and wraps that result. In addition
to `liftM2`, the variants in `Control.Monad` go up to `liftM5`.

Putting a few misconceptions to rest
------------------------------------

We\'ve now seen enough examples of monads in action to have some feel
for what\'s going on. Before we continue, there are a few oft-repeated
myths about monads that we\'re going to address. You\'re bound to
encounter these assertions \"in the wild\", so you might as well be
prepared with a few good retorts.

-   *Monads can be hard to understand.* We\'ve already shown that monads
    \"fall out naturally\" from several problems. We\'ve found that the
    best key to understanding them is to explain several concrete
    examples, then talk about what they have in common.
-   *Monads are only useful for I/O and imperative coding.* While we use
    monads for I/O in Haskell, they\'re valuable for many other purposes
    besides. We\'ve already used them for short-circuiting a chain of
    computations, hiding complicated state, and logging. Even so, we\'ve
    barely scratched the surface.
-   *Monads are unique to Haskell.* Haskell is probably the language
    that makes the most explicit use of monads, but people write them in
    other languages, too, ranging from C++ to OCaml. They happen to be
    particularly tractable in Haskell, due to `do` notation, the power
    and inference of the type system, and the language\'s syntax.
-   *Monads are for controlling the order of evaluation.*

Building the `Logger` monad
---------------------------

The definition of our `Logger` type is very simple.

<div>

[Logger.hs]{.label}

``` {.haskell}
newtype Logger a = Logger { execLogger :: (a, Log) } deriving Show
```

</div>

It\'s a pair, where the first element is the result of an action, and
the second is a list of messages logged while that action was run.

We\'ve wrapped the tuple in a `newtype` to make it a distinct type. The
`runLogger` function extracts the tuple from its wrapper. The function
that we\'re exporting to execute a logged action, `runLogger`, is just a
synonym for `execLogger`.

<div>

[Logger.hs]{.label}

``` {.haskell}
runLogger = execLogger
```

</div>

Our `record` helper function creates a singleton list of the message we
pass it.

<div>

[Logger.hs]{.label}

``` {.haskell}
record s = Logger ((), [s])
```

</div>

The result of this action is `()`, so that\'s the value we put in the
result slot.

Let\'s begin our `Monad` instance with `return`, which is trivial: it
logs nothing, and stores its input in the result slot of the tuple so it
is equal to `pure`.

<div>

[Logger.hs]{.label}

``` {.haskell}
instance Functor Logger where
    fmap = liftM

instance Applicative Logger where
    pure a = Logger (a, [])
    (<*>) = ap

instance Monad Logger where
    return = pure
```

</div>

Slightly more interesting is `(>>=)`, which is the heart of the monad.
It combines an action and a monadic function to give a new result and a
new log.

<div>

[Logger.hs]{.label}

``` {.haskell}
-- (>>=) :: Logger a -> (a -> Logger b) -> Logger b
m >>= k = let (a, w) = execLogger m
              n      = k a
              (b, x) = execLogger n
          in Logger (b, w ++ x)
```

</div>

Let\'s spell out explicitly what is going on. We use `runLogger` to
extract the result `a` from the action `m`, and we pass it to the
monadic function `k`. We extract the result `b` from that in turn, and
put it into the result slot of the final action. We concatenate the logs
`w` and `x` to give the new log.

### Sequential logging, not sequential evaluation

Our definition of `(>>=)` ensures that messages logged on the left will
appear in the new log before those on the right. However, it says
nothing about when the values `a` and `b` are evaluated: `(>>=)` is
lazy.

Like most other aspects of a monad\'s behaviour, strictness is under the
control of the monad\'s implementor. It is not a constant shared by all
monads. Indeed, some monads come in multiple flavours, each with
different levels of strictness.

### The writer monad

Our `Logger` monad is a specialised version of the standard `Writer`
monad, which can be found in the `Control.Monad.Writer` module of the
`mtl` package. We will present a `Writer` example in [the section called
\"Using type
classes\"](16-programming-with-monads.org::*Using type classes)

The Maybe monad
---------------

The `Maybe` type is very nearly the simplest instance of `Monad`. It
represents a computation that might not produce a result.

``` {.haskell}
instance Functor Maybe where
    fmap = liftM

instance Applicative Maybe where
    pure x = Just x

instance Monad Maybe where
    Just x >>= k  = k x
    Nothing >>= _ = Nothing

    Just _ >> k   = k
    Nothing >> _  = Nothing

    return        = pure

    fail _        = Nothing
```

When we chain together a number of computations over `Maybe` using
`(>>=)` or `(>>)`, if any of them returns `Nothing`, then we don\'t
evaluate any of the remaining computations.

Note, though, that the chain is not completely short-circuited. Each
`(>>=)` or `(>>)` in the chain will still match a `Nothing` on its left,
and produce a `Nothing` on its right, all the way to the end. It\'s easy
to forget this point: when a computation in the chain fails, the
subsequent production, chaining, and consumption of `Nothing` values is
cheap at runtime, but it\'s not free.

### Executing the Maybe monad

A function suitable for executing the `Maybe` monad is `maybe`.
(Remember that \"executing\" a monad involves evaluating it and
returning a result that\'s had the monad\'s type wrapper removed.)

<div>

[Maybe.hs]{.label}

``` {.haskell}
maybe :: b -> (a -> b) -> Maybe a -> b
maybe n _ Nothing  = n
maybe _ f (Just x) = f x
```

</div>

Its first parameter is the value to return if the result is `Nothing`.
The second is a function to apply to a result wrapped in the `Just`
constructor; the result of that application is then returned.

Since the `Maybe` type is so simple, it\'s about as common to simply
pattern-match on a `Maybe` value as it is to call `maybe`. Each one is
more readable in different circumstances.

### Maybe at work

Here\'s an example of `Maybe` in use as a monad. Given a customer\'s
name, we want to find the billing address of their mobile phone carrier.

<div>

[Carrier.hs]{.label}

``` {.haskell}
import qualified Data.Map as M

type PersonName = String
type PhoneNumber = String
type BillingAddress = String
data MobileCarrier = Honest_Bobs_Phone_Network
                   | Morrisas_Marvelous_Mobiles
                   | Petes_Plutocratic_Phones
                     deriving (Eq, Ord)

findCarrierBillingAddress :: PersonName
                          -> M.Map PersonName PhoneNumber
                          -> M.Map PhoneNumber MobileCarrier
                          -> M.Map MobileCarrier BillingAddress
                          -> Maybe BillingAddress
```

</div>

Our first version is the dreaded ladder of code marching off the right
of the screen, with many boilerplate `case` expressions.

<div>

[Carrier.hs]{.label}

``` {.haskell}
variation1 person phoneMap carrierMap addressMap =
    case M.lookup person phoneMap of
      Nothing -> Nothing
      Just number ->
          case M.lookup number carrierMap of
            Nothing -> Nothing
            Just carrier -> M.lookup carrier addressMap
```

</div>

We can make more sensible use of `Maybe`\'s status as a monad.

<div>

[Carrier.hs]{.label}

``` {.haskell}
variation2 person phoneMap carrierMap addressMap = do
  number <- M.lookup person phoneMap
  carrier <- M.lookup number carrierMap
  address <- M.lookup carrier addressMap
  return address
```

</div>

If any of these lookups fails, the definitions of `(>>=)` and `(>>)`
mean that the result of the function as a whole will be `Nothing`, just
as it was for our first attempt that used `case` explicitly.

This version is much tidier, but the `return` isn\'t necessary.
Stylistically, it makes the code look more regular, and perhaps more
familiar to the eyes of an imperative programmer, but behaviourally
it\'s redundant. Here\'s an equivalent piece of code.

<div>

[Carrier.hs]{.label}

``` {.haskell}
variation2a person phoneMap carrierMap addressMap = do
  number <- M.lookup person phoneMap
  carrier <- M.lookup number carrierMap
  M.lookup carrier addressMap
```

</div>

When we introduced maps, we mentioned in [the section called \"Partial
application
awkwardness\"](12-barcode-recognition.org::*Partial application awkwardness)
signatures of functions in the `Data.Map` module often make them awkward
to partially apply. The `lookup` function is a good example. If we
`flip` its arguments, we can write the function body as a one-liner.

<div>

[Carrier.hs]{.label}

``` {.haskell}
variation3 person phoneMap carrierMap addressMap =
    lookup phoneMap person >>= lookup carrierMap >>= lookup addressMap
  where lookup = flip M.lookup
```

</div>

The list monad
--------------

While the `Maybe` type can represent either no value or one, there are
many situations where we might want to return some number of results
that we do not know in advance. Obviously, a list is well suited to this
purpose. The type of a list suggests that we might be able to use it as
a monad, because its type constructor has one free variable. And sure
enough, we can use a list as a monad.

Rather than simply present the `Prelude`\'s `Monad` instance for the
list type, let\'s try to figure out what an instance *ought* to look
like. This is easy to do: we\'ll look at the types of `(>>=)` and
`return`, and perform some substitutions, and see if we can use a few
familiar list functions.

The more obvious of the two functions is `return`. We know that it takes
a type `a`, and wraps it in a type constructor `m` to give the type
`m a`. We also know that the type constructor here is `[]`. Substituting
this type constructor for the type variable `m` gives us the type `[] a`
(yes, this really is valid notation!), which we can rewrite in more
familiar form as `[a]`.

We now know that `return` for lists should have the type `a -> [a]`.
There are only a few sensible possibilities for an implementation of
this function. It might return the empty list, a singleton list, or an
infinite list. The most appealing behaviour, based on what we know so
far about monads, is the singleton list: it doesn\'t throw information
away, nor does it repeat it infinitely.

<div>

[ListMonad.hs]{.label}

``` {.haskell}
returnSingleton :: a -> [a]
returnSingleton x = [x]
```

</div>

If we perform the same substitution trick on the type of `(>>=)` as we
did with `return`, we discover that it should have the type
`[a] -> (a -> [b]) -> [b]`. This seems close to the type of `map`.

``` {.screen}
ghci> :type (>>=)
(>>=) :: Monad m => m a -> (a -> m b) -> m b
ghci> :type map
map :: (a -> b) -> [a] -> [b]
```

The ordering of the types in `map`\'s arguments doesn\'t match, but
that\'s easy to fix.

``` {.screen}
ghci> :type (>>=)
(>>=) :: Monad m => m a -> (a -> m b) -> m b
ghci> :type flip map
flip map :: [a] -> (a -> b) -> [b]
```

We\'ve still got a problem: the second argument of `flip map` has the
type `a -> b`, whereas the second argument of `(>>=)` for lists has the
type `a -> [b]`. What do we do about this?

Let\'s do a little more substitution and see what happens with the
types. The function `flip map` can return any type `b` as its result. If
we substitute `[b]` for `b` in both places where it appears in
`flip map`\'s type signature, its type signature reads as
`a -> (a -> [b]) -> [[b]]`. In other words, if we map a function that
returns a list over a list, we get a list of lists back.

``` {.screen}
ghci> flip map [1,2,3] (\a -> [a,a+100])
[[1,101],[2,102],[3,103]]
```

Interestingly, we haven\'t really changed how closely our type
signatures match. The type of `(>>=)` is `[a] -> (a -> [b]) -> [b]`,
while that of `flip map` when the mapped function returns a list is
`[a] -> (a -> [b]) -> [[b]]`. There\'s still a mismatch in one type
term; we\'ve just moved that term from the middle of the type signature
to the end. However, our juggling wasn\'t in vain: we now need a
function that takes a `[[b]]` and returns a `[b]`, and one readily
suggests itself in the form of `concat`.

``` {.screen}
ghci> :type concat
:: Foldable t => t [a] -> [a]
```

A list is `Foldable`, i.e. it supports `foldr` so we can interpret the
type as `[[a]] -> [a]`. It suggest that we should flip the arguments to
`map`, then `concat` the results to give a single list.

``` {.screen}
ghci> :type \xs f -> concat (map f xs)
\xs f -> concat (map f xs) :: [a1] -> (a1 -> [a2]) -> [a2]
```

This is exactly the definition of `(>>=)` for lists.

<div>

[ListMonad.hs]{.label}

``` {.haskell}
instance Monad [] where
    return x = [x]
    xs >>= f = concat (map f xs)
```

</div>

It applies `f` to every element in the list `xs`, and concatenates the
results to return a single list.

With our two core `Monad` definitions in hand, the implementations of
the non-core definitions that remain, `(>>)` and `fail`, ought to be
obvious.

<div>

[ListMonad.hs]{.label}

``` {.haskell}
xs >> f = concat (map (\_ -> f) xs)
fail _ = []
```

</div>

### Understanding the list monad

The list monad is similar to a familiar Haskell tool, the list
comprehension. We can illustrate this similarity by computing the
Cartesian product of two lists. First, we\'ll write a list
comprehension.

<div>

[CartesianProduct.hs]{.label}

``` {.haskell}
comprehensive xs ys = [(x,y) | x <- xs, y <- ys]
```

</div>

For once, we\'ll use bracketed notation for the monadic code instead of
layout notation. This will highlight how structurally similar the
monadic code is to the list comprehension.

<div>

[CartesianProduct.hs]{.label}

``` {.haskell}
monadic xs ys = do { x <- xs; y <- ys; return (x,y) }
```

</div>

The only real difference is that the value we\'re constructing comes at
the end of the sequence of expressions, instead of the beginning as in
the list comprehension. Also, the results of the two functions are
identical.

``` {.screen}
ghci> comprehensive [1,2] "bar"
[(1,'b'),(1,'a'),(1,'r'),(2,'b'),(2,'a'),(2,'r')]
ghci> comprehensive [1,2] "bar" == monadic [1,2] "bar"
True
```

It\'s easy to be baffled by the list monad early on, so let\'s walk
through our monadic Cartesian product code again in more detail. This
time, we\'ll rearrange the function to use layout instead of brackets.

<div>

[CartesianProduct.hs]{.label}

``` {.haskell}
blockyDo xs ys = do
    x <- xs
    y <- ys
    return (x, y)
```

</div>

For every element in the list `xs`, the rest of the function is
evaluated once, with `x` bound to a different value from the list each
time. Then for every element in the list `ys`, the remainder of the
function is evaluated once, with `y` bound to a different value from the
list each time.

What we really have here is a doubly nested loop! This highlights an
important fact about monads: you *cannot* predict how a block of monadic
code will behave unless you know what monad it will execute in.

We\'ll now walk through the code even more explicitly, but first let\'s
get rid of the `do` notation, to make the underlying structure clearer.
We\'ve indented the code a little unusually to make the loop nesting
more obvious.

<div>

[CartesianProduct.hs]{.label}

``` {.haskell}
blockyPlain xs ys =
    xs >>=
    \x -> ys >>=
    \y -> return (x, y)

blockyPlain_reloaded xs ys =
    concat (map (\x ->
                 concat (map (\y ->
                              return (x, y))
                         ys))
            xs)
```

</div>

If `xs` has the value `[1,2,3]`, the two lines that follow are evaluated
with `x` bound to `1`, then to `2`, and finally to `3`. If `ys` has the
value `[True, False]`, the final line is evaluated *six* times: once
with `x` as `1` and `y` as `True`; again with `x` as `1` and `y` as
`False`; and so on. The `return` expression wraps each tuple in a
single-element list.

### Putting the list monad to work

Here is a simple brute force constraint solver. Given an integer, it
finds all pairs of positive integers that, when multiplied, give that
value (this is the constraint being solved).

<div>

[MultiplyTo.hs]{.label}

``` {.haskell}
guarded :: Bool -> [a] -> [a]
guarded True  xs = xs
guarded False _  = []

multiplyTo :: Int -> [(Int, Int)]
multiplyTo n = do
  x <- [1..n]
  y <- [x..n]
  guarded (x * y == n) $
    return (x, y)
```

</div>

Let\'s try this in `ghci`.

``` {.screen}
ghci> multiplyTo 8
[(1,8),(2,4)]
ghci> multiplyTo 100
[(1,100),(2,50),(4,25),(5,20),(10,10)]
ghci> multiplyTo 891
[(1,891),(3,297),(9,99),(11,81),(27,33)]
```

Desugaring of do blocks
-----------------------

Haskell\'s `do` syntax is an example of *syntactic sugar*: it provides
an alternative way of writing monadic code, without using `(>>=)` and
anonymous functions. *Desugaring* is the translation of syntactic sugar
back to the core language.

The rules for desugaring a `do` block are easy to follow. We can think
of a compiler as applying these rules mechanically and repeatedly to a
`do` block until no more `do` keywords remain.

A `do` keyword followed by a single action is translated to that action
by itself.

  ----------------------- -----------------------
  \#+BEGIN~SRC~ haskell   \#+BEGIN~SRC~ haskell
  doNotation1 =           translated1 =
  do act                  act
  \#+END~SRC~             \#+END~SRC~
  ----------------------- -----------------------

A `do` keyword followed by more than one action is translated to the
first action, then `(>>)`, followed by a `do` keyword and the remaining
actions. When we apply this rule repeatedly, the entire `do` block ends
up chained together by applications of `(>>)`.

  ----------------------- -----------------------
  \#+BEGIN~SRC~ haskell   \#+BEGIN~SRC~ haskell
  doNotation2 =           translated2 =
  do act1                 act1 \>\>
  act2                    do act2
  {- ... etc. -}          {- ... etc. -}
  actN                    actN
  \#+END~SRC~             
                          finalTranslation2 =
                          act1 \>\>
                          act2 \>\>
                          {- ... etc. -}
                          actN
                          \#+END~SRC~
  ----------------------- -----------------------

The `<-` notation has a translation that\'s worth paying close attention
to. On the left of the `<-` is a normal Haskell pattern. This can be a
single variable or something more complicated. A guard expression is not
allowed.

  ----------------------- -------------------------
  \#+BEGIN~SRC~ haskell   \#+BEGIN~SRC~ haskell
  doNotation3 =           translated3 =
  do pattern \<- act1     let f pattern = do act2
  act2                    {- ... etc. -}
  {- ... etc. -}          actN
  actN                    f \_ = fail \"...\"
  \#+END~SRC~             in act1 \>\>= f
                          \#+END~SRC~
  ----------------------- -------------------------

This pattern is translated into a `let` binding that declares a local
function with a unique name (we\'re just using `f` as an example above).
The action on the right of the `<-` is then chained with this function
using `(>>=)`.

What\'s noteworthy about this translation is that if the pattern match
fails, the local function calls the monad\'s `fail` implementation.
Here\'s an example using the `Maybe` monad.

<div>

[Do.hs]{.label}

``` {.haskell}
robust :: [a] -> Maybe a
robust xs = do (_:x:_) <- Just xs
               return x
```

</div>

The `fail` implementation in the `Maybe` monad simply returns `Nothing`.
If the pattern match in the above function fails, we thus get `Nothing`
as our result.

``` {.screen}
ghci> robust [1,2,3]
Just 2
ghci> robust [1]
Nothing
```

Finally, when we write a `let` expression in a `do` block, we can omit
the usual `in` keyword. Subsequent actions in the block must be lined up
with the `let` keyword.

  ----------------------- -----------------------
  \#+BEGIN~SRC~ haskell   \#+BEGIN~SRC~ haskell
  doNotation4 =           translated4 =
  do let val1 = expr1     let val1 = expr1
  val2 = expr2            val2 = expr2
  {- ... etc. -}          valN = exprN
  valN = exprN            in do act1
  act1                    act2
  act2                    {- ... etc. -}
  {- ... etc. -}          actN
  actN                    \#+END~SRC~
  \#+END~SRC~             
  ----------------------- -----------------------

### Monads as a programmable semicolon

Back in [the section called \"The offside rule is not
mandatory\"](3-defining-types-streamlining-functions.org::*The offside rule is not mandatory)
mentioned that layout is the norm in Haskell, but it\'s not *required*.
We can write a `do` block using explicit structure instead of layout.

  ----------------------- -------------------------------
  \#+BEGIN~SRC~ haskell   \#+BEGIN~SRC~ haskell
  semicolon = do          semicolonTranslated =
  {                       act1 \>\>
  act1;                   let f val1 = let val2 = expr1
  val1 \<- act2;          in actN
  let { val2 = expr1 };   f \_ = fail \"...\"
  actN;                   in act2 \>\>= f
  }                       \#+END~SRC~
  \#+END~SRC~             
  ----------------------- -------------------------------

Even though this use of explicit structure is rare, the fact that it
uses semicolons to separate expressions has given rise to an apt slogan:
monads are a kind of \"programmable semicolon\", because the behaviours
of `(>>)` and `(>>=)` are different in each monad.

### Why go sugar-free?

When we write `(>>=)` explicitly in our code, it reminds us that we\'re
stitching functions together using combinators, not simply sequencing
actions.

As long as you feel like a novice with monads, we think you should
prefer to explicitly write `(>>=)` over the syntactic sugar of `do`
notation. The repeated reinforcement of what\'s really happening seems,
for many programmers, to help to keep things clear. (It can be easy for
an imperative programmer to relax a little too much from exposure to the
`IO` monad, and assume that a `do` block means nothing more than a
simple sequence of actions.)

Once you\'re feeling more familiar with monads, you can choose whichever
style seems more appropriate for writing a particular function. Indeed,
when you read other people\'s monadic code, you\'ll see that it\'s
unusual, but by no means rare, to mix *both* `do` notation and `(>>=)`
in a single function.

The `(=<<)` function shows up frequently whether or not we use `do`
notation. It is a flipped version of `(>>=)`.

``` {.screen}
ghci> :type (>>=)
(>>=) :: Monad m => m a -> (a -> m b) -> m b
ghci> :type (=<<)
(=<<) :: Monad m => (a -> m b) -> m a -> m b
```

It comes in handy if we want to compose monadic functions in the usual
Haskell right-to-left style.

<div>

[CartesianProduct.hs]{.label}

``` {.haskell}
wordCount = print . length . words =<< getContents
```

</div>

The state monad
---------------

We discovered earlier in this chapter that the `Parse` from [Chapter 10,
*Code case study: parsing a binary data
format*](10-parsing-a-binary-data-format.org) was a monad. It has two
logically distinct aspects. One is the idea of a parse failing, and
providing a message with the details: we represented this using the
`Either` type. The other involves carrying around a piece of implicit
state, in our case the partially consumed `ByteString`.

This need for a way to read and write state is common enough in Haskell
programs that the standard libraries provide a monad named `State` that
is dedicated to this purpose. This monad lives in the
`Control.Monad.State` module.

Where our `Parse` type carried around a `ByteString` as its piece of
state, the `State` monad can carry any type of state. We\'ll refer to
the state\'s unknown type as `s`.

What\'s an obvious and general thing we might want to do with a state?
Given a state value, we inspect it, then produce a result and a new
state value. Let\'s say the result can be of any type `a`. A type
signature that captures this idea is `s -> (a, s)`: take a state `s`, do
something with it, and return a result `a` and possibly a new state `s`.

### Almost a state monad

Let\'s develop some simple code that\'s *almost* the `State` monad, then
we\'ll take a look at the real thing. We\'ll start with our type
definition, which has exactly the obvious type we described above.

<div>

[SimpleState.hs]{.label}

``` {.haskell}
type SimpleState s a = s -> (a, s)
```

</div>

Our monad is a function that transforms one state into another, yielding
a result when it does so. Because of this, the `State` monad is
sometimes called the *state transformer monad*.

Yes, this is a type synonym, not a new type, and so we\'re cheating a
little. Bear with us for now; this simplifies the description that
follows.

Earlier in this chapter, we said that a monad has a type constructor
with a single type variable, and yet here we have a type with two
parameters. The key here is to understand that we can partially apply a
*type* just as we can partially apply a normal function. This is easiest
to follow with an example.

<div>

[SimpleState.hs]{.label}

``` {.haskell}
type StringState a = SimpleState String a
```

</div>

Here, we\'ve bound the type variable `s` to `String`. The type
`StringState` still has a type parameter `a`, though. It\'s now more
obvious that we have a suitable type constructor for a monad. In other
words, our monad\'s type constructor is `SimpleState s`, not
`SimpleState` alone.

The next ingredient we need to make a monad is a definition for the
`return` function.

<div>

[SimpleState.hs]{.label}

``` {.haskell}
returnSt :: a -> SimpleState s a
returnSt a = \s -> (a, s)
```

</div>

All this does is take the result and the current state, and \"tuple them
up\". You may by now be used to the idea that a Haskell function with
multiple parameters is just a chain of single-parameter functions, but
just in case you\'re not, here\'s a more familiar way of writing
`returnSt` that makes it more obvious how simple this function is.

<div>

[SimpleState.hs]{.label}

``` {.haskell}
returnAlt :: a -> SimpleState s a
returnAlt a s = (a, s)
```

</div>

Our final piece of the monadic puzzle is a definition for `(>>=)`. Here
it is, using the actual variable names from the standard library\'s
definition of `(>>=)` for `State`.

<div>

[SimpleState.hs]{.label}

``` {.haskell}
bindSt :: (SimpleState s a) -> (a -> SimpleState s b) -> SimpleState s b
bindSt m k = \s -> let (a, s') = m s
                   in (k a) s'
```

</div>

Those single-letter variable names aren\'t exactly a boon to
readability, so let\'s see if we can substitute some more meaningful
names.

<div>

[SimpleState.hs]{.label}

``` {.haskell}
-- m == step
-- k == makeStep
-- s == oldState

bindAlt step makeStep oldState =
    let (result, newState) = step oldState
    in  (makeStep result) newState
```

</div>

To understand this definition, remember that `step` is a function with
the type `s -> (a, s)`. When we evaluate this, we get a tuple, and we
have to use this to return a new function of type `s -> (a, s)`. This is
perhaps easier to follow if we get rid of the `SimpleState` type
synonyms from `bindAlt`\'s type signature, and examine the types of its
parameters and result.

<div>

[SimpleState.hs]{.label}

``` {.haskell}
bindAlt :: (s -> (a, s))        -- step
        -> (a -> s -> (b, s))   -- makeStep
        -> (s -> (b, s))        -- (makeStep result) newState
```

</div>

### Reading and modifying the state

The definitions of `(>>=)` and `return` for the `State` monad simply act
as plumbing: they move a piece of state around, but they don\'t touch it
in any way. We need a few other simple functions to actually do useful
work with the state.

<div>

[SimpleState.hs]{.label}

``` {.haskell}
getSt :: SimpleState s s
getSt = \s -> (s, s)

putSt :: s -> SimpleState s ()
putSt s = \_ -> ((), s)
```

</div>

The `getSt` function simply takes the current state and returns it as
the result, while `putSt` ignores the current state and replaces it with
a new state.

### Will the real state monad please stand up?

The only simplifying trick we played in the previous section was to use
a type synonym instead of a type definition for `SimpleState`. If we had
introduced a `newtype` wrapper at the same time, the extra wrapping and
unwrapping would have made our code harder to follow.

In order to define a `Monad` instance, we have to provide a proper type
constructor as well as definitions for `(>>=)` and `return`. This leads
us to the following definition of `State`.

<div>

[State.hs]{.label}

``` {.haskell}
newtype State s a = State {
    runState :: s -> (a, s)
}
```

</div>

All we\'ve done is wrap our `s -> (a, s)` type in a `State` constructor.
By using Haskell\'s record syntax to define the type, we\'re
automatically given a `runState` function that will unwrap a `State`
value from its constructor. The type of `runState` is
`State s a -> s -> (a, s)`.

The definition of `return` is almost the same as for `SimpleState`,
except we wrap our function with a `State` constructor.

<div>

[State.hs]{.label}

``` {.haskell}
returnState :: a -> State s a
returnState a = State $ \s -> (a, s)
```

</div>

The definition of `(>>=)` is a little more complicated, because it has
to use `runState` to remove the `State` wrappers.

<div>

[State.hs]{.label}

``` {.haskell}
bindState :: State s a -> (a -> State s b) -> State s b
bindState m k = State $ \s -> let (a, s') = runState m s
                              in runState (k a) s'
```

</div>

This function differs from our earlier `bindSt` only in adding the
wrapping and unwrapping of a few values. By separating the \"real work\"
from the bookkeeping, we\'ve hopefully made it clearer what\'s really
happening.

We modify the functions for reading and modifying the state in the same
way, by adding a little wrapping.

<div>

[State.hs]{.label}

``` {.haskell}
get :: State s s
get = State $ \s -> (s, s)

put :: s -> State s ()
put s = State $ \_ -> ((), s)
```

</div>

### Using the `State` monad: generating random values

We\'ve already used `Parse`, our precursor to the `State` monad, to
parse binary data. In that case, we wired the type of the state we were
manipulating directly into the `Parse` type.

The `State` monad, by contrast, accepts any type of state as a
parameter. We supply the type of the state, to give e.g. `State`
`ByteString`.

The `State` monad will probably feel more familiar to you than many
other monads if you have a background in imperative languages. After
all, imperative languages are all about carrying around some implicit
state, reading some parts, and modifying others through assignment, and
this is just what the `State` monad is for.

So instead of unnecessarily cheerleading for the idea of using the
`State` monad, we\'ll begin by demonstrating how to use it for something
simple: pseudorandom value generation. In an imperative language,
there\'s usually an easily available source of uniformly distributed
pseudorandom numbers. For example, in C, there\'s a standard `rand`
function that generates a pseudorandom number, using a global state that
it updates.

Haskell\'s standard random value generation module is named
`System.Random` and is located in the `random` package. It allows the
generation of random values of any type, not just numbers. The module
contains several handy functions that live in the `IO` monad. For
example, a rough equivalent of C\'s `rand` function would be the
following:

<div>

[Random.hs]{.label}

``` {.haskell}
import System.Random

rand :: IO Int
rand = getStdRandom (randomR (0, maxBound))
```

</div>

(The `randomR` function takes an inclusive range within which the
generated random value should lie.)

The `System.Random` module provides a type class, `RandomGen`, that lets
us define new sources of random `Int` values. The type `StdGen` is the
standard `RandomGen` instance. It generates pseudorandom values. If we
had an external source of truly random data, we could make it an
instance of `RandomGen` and get truly random, instead of merely
pseudorandom, values.

Another type class, `Random`, indicates how to generate random values of
a particular type. The module defines `Random` instances for all of the
usual simple types.

Incidentally, the definition of `rand` above reads and modifies a
built-in global random generator that inhabits the `IO` monad.

### A first attempt at purity

After all of our emphasis so far on avoiding the `IO` monad wherever
possible, it would be a shame if we were dragged back into it just to
generate some random values. Indeed, `System.Random` contains pure
random number generation functions.

The traditional downside of purity is that we have to get or create a
random number generator, then ship it from the point we created it to
the place where it\'s needed. When we finally call it, it returns a
*new* random number generator: we\'re in pure code, remember, so we
can\'t modify the state of the existing generator.

If we forget about immutability and reuse the same generator within a
function, we get back exactly the same \"random\" number every time.

<div>

[Random.hs]{.label}

``` {.haskell}
twoBadRandoms :: RandomGen g => g -> (Int, Int)
twoBadRandoms gen = (fst $ random gen, fst $ random gen)
```

</div>

Needless to say, this has unpleasant consequences.

``` {.screen}
ghci> twoBadRandoms `fmap` getStdGen
(945769311181683171,945769311181683171)
```

The `random` function uses an implicit range instead of the
user-supplied range used by `randomR`. The `getStdGen` function
retrieves the current value of the global standard number generator from
the `IO` monad.

Unfortunately, correctly passing around and using successive versions of
the generator does not make for palatable reading. Here\'s a simple
example.

<div>

[Random.hs]{.label}

``` {.haskell}
twoGoodRandoms :: RandomGen g => g -> ((Int, Int), g)
twoGoodRandoms gen = let (a, gen') = random gen
                         (b, gen'') = random gen'
                     in ((a, b), gen'')
```

</div>

Now that we know about the `State` monad, though, it looks like a fine
candidate to hide the generator. The `State` monad lets us manage our
mutable state tidily, while guaranteeing that our code will be free of
other unexpected side effects, such as modifying files or making network
connections. This makes it easier to reason about the behavior of our
code.

### Random values in the `State` monad

Here\'s a state monad that carries around a `StdGen` as its piece of
state.

<div>

[Random.hs]{.label}

``` {.haskell}
-- import Control.Mand.State at the beginning
type RandomState a = State StdGen a
```

</div>

The type synonym is of course not necessary, but it\'s handy. It saves a
little keyboarding, and if we wanted to swap another random generator
for `StdGen`, it would reduce the number of type signatures we\'d need
to change.

Generating a random value is now a matter of fetching the current
generator, using it, then modifying the state to replace it with the new
generator.

<div>

[Random.hs]{.label}

``` {.haskell}
getRandom :: Random a => RandomState a
getRandom =
  get >>= \gen ->
  let (val, gen') = random gen in
  put gen' >>
  return val
```

</div>

We can now use some of the monadic machinery that we saw earlier to
write a much more concise function for giving us a pair of random
numbers.

<div>

[Random.hs]{.label}

``` {.haskell}
getTwoRandoms :: Random a => RandomState (a, a)
getTwoRandoms = liftM2 (,) getRandom getRandom
```

</div>

1.  Exercises

    1.  Rewrite `getRandom` to use do notation.

### Running the state monad

As we\'ve already mentioned, each monad has its own specialised
evaluation functions. In the case of the `State` monad, we have several
to choose from.

-   `runState` returns both the result and the final state.
-   `evalState` returns only the result, throwing away the final state.
-   `execState` throws the result away, returning only the final state.

The `evalState` and `execState` functions are simply compositions of
`fst` and `snd` with `runState`, respectively. Thus, of the three,
`runState` is the one most worth remembering.

Here\'s a complete example of how to implement our `getTwoRandoms`
function.

<div>

[Random.hs]{.label}

``` {.haskell}
runTwoRandoms :: IO (Int, Int)
runTwoRandoms = do
  oldState <- getStdGen
  let (result, newState) = runState getTwoRandoms oldState
  setStdGen newState
  return result
```

</div>

The call to `runState` follows a standard pattern: we pass it a function
in the `State` monad and an initial state. It returns the result of the
function and the final state.

The code surrounding the call to `runState` merely obtains the current
global `StdGen` value, then replaces it afterwards so that subsequent
calls to `runTwoRandoms` or other random generation functions will pick
up the updated state.

### What about a bit more state?

It\'s a little hard to imagine writing much interesting code in which
there\'s only a single state value to pass around. When we want to track
multiple pieces of state at once, the usual trick is to maintain them in
a data type. Here\'s an example: keeping track of the number of random
numbers we are handing out.

<div>

[Random.hs]{.label}

``` {.haskell}
data CountedRandom = CountedRandom {
      crGen :: StdGen
    , crCount :: Int
    }

type CRState = State CountedRandom

getCountedRandom :: Random a => CRState a
getCountedRandom = do
  st <- get
  let (val, gen') = random (crGen st)
  put CountedRandom { crGen = gen', crCount = crCount st + 1 }
  return val
```

</div>

This example happens to consume both elements of the state, and
construct a completely new state, every time we call into it. More
frequently, we\'re likely to read or modify only part of a state. This
function gets the number of random values generated so far.

<div>

[Random.hs]{.label}

``` {.haskell}
getCount :: CRState Int
getCount = crCount `liftM` get
```

</div>

This example illustrates why we used record syntax to define our
`CountedRandom` state. It gives us accessor functions that we can glue
together with `get` to read specific pieces of the state.

If we want to partially update a state, the code doesn\'t come out quite
so appealingly.

<div>

[Random.hs]{.label}

``` {.haskell}
putCount :: Int -> CRState ()
putCount a = do
  st <- get
  put st { crCount = a }
```

</div>

Here, instead of a function, we\'re using record update syntax. The
expression `st { crCount ~ a }` creates a new value that\'s an identical
copy of `st`, except in its `crCount` field, which is given the value
`a`. Because this is a syntactic hack, we don\'t get the same kind of
flexibility as with a function. Record syntax may not exhibit Haskell\'s
usual elegance, but it at least gets the job done.

There exists a function named `modify` that combines the `get` and `put`
steps. It takes as argument a state transformation function, but it\'s
hardly more satisfactory: we still can\'t escape from the clumsiness of
record update syntax.

<div>

[Random.hs]{.label}

``` {.haskell}
putCountModify :: Int -> CRState ()
putCountModify a = modify $ \st -> st { crCount = a }
```

</div>

Another way of looking at monads
--------------------------------

Now that we know enough about monads structure we can look back at the
list monad and see something interesting. Specifically, take a look at
the definition of `(>>=)` for lists.

<div>

[ListMonad.hs]{.label}

``` {.haskell}
instance Monad [] where
    return x = [x]
    xs >>= f = concat (map f xs)
```

</div>

Recall that `f` has type `a -> [a]`. When we call `map f xs`, we get
back a value of type `[[a]]`, which we have to \"flatten\" using
`concat`.

Since `fmap` for lists is defined to be `map`, we could replace `map`
with `fmap` in the definition of `(>>=)`. This is not very interesting
by itself, but suppose we could go further.

The `concat` function is of type `[[a]] -> [a]`: as we mentioned, it
flattens the nesting of lists. We could generalise this type signature
from lists to monads, giving us the \"remove a level of nesting\" type
`m (m a) -> m a`. The function that has this type is conventionally
named `join`.

If we had definitions of `join` and `fmap`, we wouldn\'t need to write a
definition of `(>>=)` for every monad, because it would be completely
generic. Here\'s what an alternative definition of the `Monad` type
class might look like, along with a definition of `(>>=)`.

``` {.haskell}
import Prelude hiding ((>>=), return)

class Applicative m => AltMonad m where
    join :: m (m a) -> m a
    return :: a -> m a -- Or simply pure

(>>=) :: AltMonad m => m a -> (a -> m b) -> m b
xs >>= f = join (fmap f xs)
```

Neither definition of a monad is \"better\", since if we have `join` we
can write `(>>=)`, and vice versa, but the different perspectives can be
refreshing.

Removing a layer of monadic wrapping can, in fact, be useful in
realistic circumstances. We can find a generic definition of `join` in
the `Control.Monad` module.

``` {.haskell}
join :: Monad m => m (m a) -> m a
join x = x >>= id
```

Here are some examples of what it does.

``` {.screen}
ghci> join (Just (Just 1))
Just 1
ghci> join Nothing
Nothing
ghci> join [[1],[2,3]]
[1,2,3]
```

The monad laws, and good coding style
-------------------------------------

In [the section called \"Thinking more about
functors\"](10-parsing-a-binary-data-format.org::*Thinking more about functors)
introduced two rules for how functors should always behave.

``` {.haskell}
fmap id == id
fmap (f . g) == fmap f . fmap g
```

There are also rules for how monads ought to behave. The three laws
below are referred to as the monad laws. A Haskell implementation
doesn\'t enforce these laws: it\'s up to the author of a `Monad`
instance to follow them.

The monad laws are simply formal ways of saying \"a monad shouldn\'t
surprise me\". In principle, we could probably get away with skipping
over them entirely. It would be a shame if we did, however, because the
laws contain gems of wisdom that we might otherwise overlook.

::: {.TIP}
Reading the laws

You can read each law below as \"the expression on the left of the `==`
is equivalent to that on the right.\"
:::

The first law states that `return` is a *left identity* for `(>>=)`.

``` {.haskell}
return x >>= f == f x
```

Another way to phrase this is that there\'s no reason to use `return` to
wrap up a pure value if all you\'re going to do is unwrap it again with
`(>>=)`. It\'s actually a common style error among programmers new to
monads to wrap a value with `return`, then unwrap it with `(>>=)` a few
lines later in the same function. Here\'s the same law written with `do`
notation.

``` {.haskell}
do y <- return x
   f y == f x
```

This law has practical consequences for our coding style: we don\'t want
to write unnecessary code, and the law lets us assume that the terse
code will be identical in its effect to the more verbose version.

The second monad law states that `return` is a *right identity* for
`(>>=)`.

``` {.haskell}
m >>= return == m
```

This law also has style consequences in real programs, particularly if
you\'re coming from an imperative language: there\'s no need to use
`return` if the last action in a block would otherwise be returning the
correct result. Let\'s look at this law in `do` notation.

``` {.haskell}
do y <- m
   return y == m
```

Once again, if we assume that a monad obeys this law, we can write the
shorter code in the knowledge that it will have the same effect as the
longer code.

The final law is concerned with associativity.

``` {.haskell}
m >>= (\x -> f x >>= g) == (m >>= f) >>= g
```

This law can be a little more difficult to follow, so let\'s look at the
contents of the parentheses on each side of the equation. We can rewrite
the expression on the left as follows.

``` {.haskell}
m >>= s
  where s x = f x >>= g
```

On the right, we can also rearrange things.

``` {.haskell}
t >>= g
  where t = m >>= f
```

We\'re now claiming that the following two expressions are equivalent.

``` {.haskell}
m >>= s == t >>= g
```

What this means is if we want to break up an action into smaller pieces,
it doesn\'t matter which sub-actions we hoist out to make new actions
with, provided we preserve their ordering. If we have three actions
chained together, we can substitute the first two and leave the third in
place, or we can replace the second two and leave the first in place.

Even this more complicated law has a practical consequence. In the
terminology of software refactoring, the \"extract method\" technique is
a fancy term for snipping out a piece of inline code, turning it into a
function, and calling the function from the site of the snipped code.
This law essentially states that this technique can be applied to
monadic Haskell code.

We\'ve now seen how each of the monad laws offers us an insight into
writing better monadic code. The first two laws show us how to avoid
unnecessary use of `return`. The third suggests that we can safely
refactor a complicated action into several simpler ones. We can now
safely let the details fade, in the knowledge that our \"do what I
mean\" intuitions won\'t be violated when we use properly written
monads.

Incidentally, a Haskell compiler cannot guarantee that a monad actually
follows the monad laws. It is the responsibility of a monad\'s author to
satisfy---or, preferably, prove to---themselves that their code follows
the laws.
