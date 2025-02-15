Chapter 19. Error handling
==========================

Error handling is one of the most important---and overlooked---topics
for programmers, regardless of the language used. In Haskell, you will
find two major types of error handling employed: \"pure\" error handling
and exceptions.

When we speak of \"pure\" error handling, we are referring to algorithms
that do not require anything from the `IO` monad. We can often implement
error handling for them by simply using Haskell\'s expressive data type
system to our advantage. Haskell also has an exception system. Due to
the complexities of lazy evaluation, exceptions in Haskell can be thrown
anywhere, but only caught within the `IO` monad. In this chapter, we\'ll
consider both.

Error Handling with Data Types
------------------------------

Let\'s begin our discussion of error handling with a very simple
function. Let\'s say that we wish to perform division on a series of
numbers. We have a constant numerator, but wish to vary the denominator.
We might come up with a function like this:

``` {.example}
divBy :: Integral a => a -> [a] -> [a]
divBy numerator = map (numerator `div`)
```

Very simple, right? We can play around with this a bit in `ghci`:

``` {.screen}
ghci> divBy 50 [1,2,5,8,10]
[50,25,10,6,5]
ghci> take 5 (divBy 100 [1..])
[100,50,33,25,20]
```

This behaves as expected: `50 / 1` is `50`, `50 / 2` is `25`, and so
forth.[^1] This even worked with the infinite list `[1..]`. What happens
if we sneak a `0` into our list somewhere?

``` {.screen}
ghci> divBy 50 [1,2,0,8,10]
[50,25,*** Exception: divide by zero
```

Isn\'t that interesting? `ghci` started displaying the output, then
stopped with an exception when it got to the zero. That\'s lazy
evaluation at work---it calculated results as needed.

As we will see later in this chapter, in the absence of an explicit
exception handler, this exception will crash the program. That\'s
obviously not desirable, so let\'s consider better ways we could
indicate an error in this pure function.

### Use of `Maybe`

One immediately-recognizable easy way to indicate failure is to use
`Maybe`.[^2] Instead of just returning a list and throwing an exception
on failure, we can return `Nothing` if the input list contained a zero
anywhere, or `Just` with the results otherwise. Here\'s an
implementation of such an algorithm:

``` {.example}
divBy :: Integral a => a -> [a] -> Maybe [a]
divBy _ [] = Just []
divBy _ (0:_) = Nothing
divBy numerator (denom:xs) =
    case divBy numerator xs of
      Nothing -> Nothing
      Just results -> Just ((numerator `div` denom) : results)
```

If you try it out in `ghci`, you\'ll see that it works:

``` {.screen}
ghci> divBy 50 [1,2,5,8,10]
Just [50,25,10,6,5]
ghci> divBy 50 [1,2,0,8,10]
Nothing
```

The function that calls `divBy` can now use a `case` statement to see if
the call was successful, just as `divBy` does when it calls itself.

::: {.TIP}
Tip

You may note that you could use a monadic implementation of the above,
like so:

``` {.example}
divBy :: Integral a => a -> [a] -> Maybe [a]
divBy numerator denominators =
    mapM (numerator `safeDiv`) denominators
    where safeDiv _ 0 = Nothing
          safeDiv x y = x `div` y
```

We will be avoiding the monadic implementation in this chapter for
simplicity, but wanted to point out that it exists.
:::

1.  Loss and Preservation of Laziness

    The use of `Maybe` was convenient, but has come at a cost. `divBy`
    can no longer handle infinite lists as input. Since the result is
    `Maybe [a]`, the entire input list must be examined before we can be
    sure that we won\'t be returning `Nothing` due to a zero somewhere
    in it. You can verify this is the case by attempting one of our
    earlier examples:

    ``` {.screen}
    ghci> divBy 100 [1..]
    *** Exception: stack overflow
    ```

    Note that you don\'t start seeing partial output here; you get *no*
    output. Notice that at each step in `divBy` (except for the case of
    an empty input list or a zero at the start of the list), the results
    from every subsequent element must be known before the results from
    the current element can be known. Thus this algorithm can\'t work on
    infinite lists, and it is also not very space-efficient for large
    finite lists.

    Having said all that, `Maybe` is often a fine choice. In this
    particular case, we don\'t know whether there will be a problem
    until we get into evaluating the entire input. Sometimes we know of
    a problem up front, for instance, that `tail []` in `ghci` produces
    an exception. We could easily write an infinite-capable `tail` that
    doesn\'t have this problem:

    ``` {.example}
    safeTail :: [a] -> Maybe [a]
    safeTail [] = Nothing
    safeTail (_:xs) = Just xs
    ```

    This simply returns `Nothing` if given an empty input list, or
    `Just` with the result for anything else. Since we only have to make
    sure the list is non-empty before knowing whether or not we have an
    error, using `Maybe` here doesn\'t reduce our laziness. We can test
    this out in `ghci` and see how it compares with regular `tail`:

    ``` {.screen}
    ghci> tail [1,2,3,4,5]
    [2,3,4,5]
    ghci> safeTail [1,2,3,4,5]
    Just [2,3,4,5]
    ghci> tail []
    *** Exception: Prelude.tail: empty list
    ghci> safeTail []
    Nothing
    ```

    Here, we can see our `safeTail` performed as expected. But what
    about infinite lists? We don\'t want to print out an infinite number
    of results, so we can test with `take 5 (tail [1..])` and a similar
    construction with `safeTail`:

    ``` {.screen}
    ghci> take 5 (tail [1..])
    [2,3,4,5,6]
    ghci> case safeTail [1..] of {Nothing -> Nothing; Just x -> Just (take 5 x)}
    Just [2,3,4,5,6]
    ghci> take 5 (tail [])
    *** Exception: Prelude.tail: empty list
    ghci> case safeTail [] of {Nothing -> Nothing; Just x -> Just (take 5 x)}
    Nothing
    ```

    Here you can see that both `tail` and `safeTail` handled infinite
    lists just fine. Note that we were able to deal better with an empty
    input list; instead of throwing an exception, we decided to return
    `Nothing` in that situation. We were able to achieve error handling
    at no expense to laziness.

    But how do we apply this to our `divBy` example? Let\'s consider the
    situation there: failure is a property of an individual bad input,
    not of the input list itself. How about making failure a property of
    an individual output element, rather than the output list itself?
    That is, instead of a function of type `a -> [a] -> Maybe [a]`,
    instead we will have `a -> [a] -> [Maybe a]`. This will have the
    benefit of preserving laziness, plus the caller will be able to
    determine exactly where in the list the problem was---or even just
    filter out the problem results if desired. Here\'s an
    implementation:

    ``` {.example}
    divBy :: Integral a => a -> [a] -> [Maybe a]
    divBy numerator denominators =
        map worker denominators
        where worker 0 = Nothing
              worker x = Just (numerator `div` x)
    ```

    Take a look at this function. We\'re back to using `map`, which is a
    good thing for both laziness and simplicity. We can try it out in
    `ghci` and see that it works for finite and infinite lists just
    fine:

    ``` {.screen}
    ghci> divBy 50 [1,2,5,8,10]
    [Just 50,Just 25,Just 10,Just 6,Just 5]
    ghci> divBy 50 [1,2,0,8,10]
    [Just 50,Just 25,Nothing,Just 6,Just 5]
    ghci> take 5 (divBy 100 [1..])
    [Just 100,Just 50,Just 33,Just 25,Just 20]
    ```

    We hope that you can take from this discussion the point that there
    is a distinction between the input not being well-formed (as in the
    case of `safeTail`) and the input potentially containing some bad
    data, as in the case of `divBy`. These two cases can often justify
    different handling of the results.

2.  Usage of the `Maybe` Monad

    Back in [the section called \"Use of
    Maybe\"](19-error-handling.org::*Use of Maybe) program named
    `divby2.hs`. This example didn\'t preserve laziness, but returned a
    value of type `Maybe [a]`. The exact same algorithm could be
    expressed using a monadic style. For more information and important
    background on monads, please refer to [Chapter 14,
    *Monads*](15-monads.org). Here\'s our new monadic-style algorithm:

    ``` {.example}
    divBy :: Integral a => a -> [a] -> Maybe [a]
    divBy _ [] = return []
    divBy _ (0:_) = fail "division by zero in divBy"
    divBy numerator (denom:xs) =
        do next <- divBy numerator xs
           return ((numerator `div` denom) : next)
    ```

    The `Maybe` monad has made the expression of this algorithm look
    nicer. For the `Maybe` monad, `return` is the same as `Just`, and
    `fail _ = Nothing`, so our error explanation string is never
    actually seen anywhere. We can test this algorithm with the same
    tests we used against `divby2.hs` if we want:

    ``` {.screen}
    ghci> divBy 50 [1,2,5,8,10]
    Just [50,25,10,6,5]
    ghci> divBy 50 [1,2,0,8,10]
    Nothing
    ghci> divBy 100 [1..]
    *** Exception: stack overflow
    ```

    The code we wrote actually isn\'t specific to the `Maybe` monad. By
    simply changing the type, we can make it work for *any* monad.
    Let\'s try it:

    ``` {.example}
    divBy :: Integral a => a -> [a] -> Maybe [a]
    divBy = divByGeneric

    divByGeneric :: (Monad m, Integral a) => a -> [a] -> m [a]
    divByGeneric _ [] = return []
    divByGeneric _ (0:_) = fail "division by zero in divByGeneric"
    divByGeneric numerator (denom:xs) =
        do next <- divByGeneric numerator xs
           return ((numerator `div` denom) : next)
    ```

    The function `divByGeneric` contains the same code as `divBy` did
    before; we just gave it a more general type. This is, in fact, the
    type that `ghci` infers if no type would be given. We also defined a
    convenience function `divBy` with a more specific type.

    Let\'s try this out in `ghci`.

    ``` {.screen}
    ghci> :l divby5.hs
    [1 of 1] Compiling Main             ( divby5.hs, interpreted )
    Ok, modules loaded: Main.
    ghci> divBy 50 [1,2,5,8,10]
    Just [50,25,10,6,5]
    ghci> (divByGeneric 50 [1,2,5,8,10])::(Integral a => Maybe [a])
    Just [50,25,10,6,5]
    ghci> divByGeneric 50 [1,2,5,8,10]
    [50,25,10,6,5]
    ghci> divByGeneric 50 [1,2,0,8,10]
    *** Exception: user error (division by zero in divByGeneric)
    ```

    The first two examples both produce the same output we see before.
    Since `divByGeneric` doesn\'t have a specific return type, we must
    either give one or let the interpreter infer one from the
    environment. If we don\'t give a specific return type, `ghci` infers
    the `IO` monad. You can see that in the third and fourth examples.
    The `IO` monad converts `fail` into an exception, as you can see
    with the fourth example.

    The `Control.Monad.Error` module in the `mtl` package makes
    `Either String` into a monad as well. If you use `Either`, you can
    get a pure result that preserves the error message, like so:

    ``` {.screen}
    ghci> :m +Control.Monad.Error
    ghci> (divByGeneric 50 [1,2,5,8,10])::(Integral a => Either String [a])
    Loading package mtl-1.1.0.0 ... linking ... done.
    Right [50,25,10,6,5]
    ghci> (divByGeneric 50 [1,2,0,8,10])::(Integral a => Either String [a])
    Left "division by zero in divByGeneric"
    ```

    This leads us into our next topic of discussion: using `Either` for
    returning error information.

### Use of `Either`

The `Either` type is similar to the `Maybe` type, with one key
difference: it can carry attached data both for an error and a success
(\"the `Right` answer\").[^3] Although the language imposes no
restrictions, by convention, a function returning an `Either` uses a
`Left` return value to indicate an error, and `Right` to indicate
success. If it helps you remember, you can think of getting the `Right`
answer. We can start with our `divby2.hs` example from the earlier
section on `Maybe` and adapt it to work with `Either`:

``` {.example}
divBy :: Integral a => a -> [a] -> Either String [a]
divBy _ [] = Right []
divBy _ (0:_) = Left "divBy: division by 0"
divBy numerator (denom:xs) =
    case divBy numerator xs of
      Left x -> Left x
      Right results -> Right ((numerator `div` denom) : results)
```

This code is almost identical to the `Maybe` code; we\'ve substituted
`Right` for every `Just`. `Left` compares to `Nothing`, but now it can
carry a message. Let\'s check it out in `ghci`:

``` {.screen}
ghci> divBy 50 [1,2,5,8,10]
Right [50,25,10,6,5]
ghci> divBy 50 [1,2,0,8,10]
Left "divBy: division by 0"
```

1.  Custom Data Types for Errors

    While a `String` indicating the cause of an error may be useful to
    humans down the road, it\'s often helpful to define a custom error
    type that we can use to programmatically decide on a course of
    action based upon exactly what the problem was. For instance, let\'s
    say that for some reason, besides 0, we also don\'t want to divide
    by 10 or 20. We could define a custom error type like so:

    ``` {.example}
    data DivByError a = DivBy0
                     | ForbiddenDenominator a
                       deriving (Eq, Read, Show)

    divBy :: Integral a => a -> [a] -> Either (DivByError a) [a]
    divBy _ [] = Right []
    divBy _ (0:_) = Left DivBy0
    divBy _ (10:_) = Left (ForbiddenDenominator 10)
    divBy _ (20:_) = Left (ForbiddenDenominator 20)
    divBy numerator (denom:xs) =
        case divBy numerator xs of
          Left x -> Left x
          Right results -> Right ((numerator `div` denom) : results)
    ```

    Now, in the event of an error, the `Left` data could be inspected to
    find the exact cause. Or, it could simply be printed out with
    `show`, which will generate a reasonable idea of the problem as
    well. Here\'s this function in action:

    ``` {.screen}
    ghci> divBy 50 [1,2,5,8]
    Right [50,25,10,6]
    ghci> divBy 50 [1,2,5,8,10]
    Left (ForbiddenDenominator 10)
    ghci> divBy 50 [1,2,0,8,10]
    Left DivBy0
    ```

    ::: {.WARNING}
    Warning

    All of these `Either` examples suffer from the lack of laziness that
    our early `Maybe` examples suffered from. We address that with an
    exercise question at the end of this chapter.
    :::

2.  Monadic Use of `Either`

    Back in [the section called \"Usage of the Maybe
    Monad\"](19-error-handling.org::*Usage of the Maybe Monad) how to
    use `Maybe` in a monad. `Either` can be used in a monad too, but can
    be slightly more complicated. The reason is that `fail` is
    hard-coded to accept only a `String` as the failure code, so we have
    to have a way to map such a string into whatever type we used for
    `Left`. As you saw earlier, `Control.Monad.Error` provides built-in
    support for `Either String a`, which involves no mapping for the
    argument to `fail`. Here\'s how we can set up our example to work
    with `Either` in the monadic style:

    ``` {.example}
    {-# LANGUAGE FlexibleContexts #-}

    import Control.Monad.Error

    data Show a =>
        DivByError a = DivBy0
                      | ForbiddenDenominator a
                      | OtherDivByError String
                        deriving (Eq, Read, Show)

    instance Error (DivByError a) where
        strMsg x = OtherDivByError x

    divBy :: Integral a => a -> [a] -> Either (DivByError a) [a]
    divBy = divByGeneric

    divByGeneric :: (Integral a, MonadError (DivByError a) m) =>
                     a -> [a] -> m [a]
    divByGeneric _ [] = return []
    divByGeneric _ (0:_) = throwError DivBy0
    divByGeneric _ (10:_) = throwError (ForbiddenDenominator 10)
    divByGeneric _ (20:_) = throwError (ForbiddenDenominator 20)
    divByGeneric numerator (denom:xs) =
        do next <- divByGeneric numerator xs
           return ((numerator `div` denom) : next)
    ```

    Here, we needed to turn on the `FlexibleContexts` language extension
    in order to provide the type signature for `divByGeneric`. The
    `divBy` function works exactly the same as before. For
    `divByGeneric`, we make `divByError` a member of the `Error` class,
    by defining what happens when someone calls `fail` (the `strMsg`
    function). We also convert `Right` to `return` and `Left` to
    `throwError` to enable this to be generic.

Exceptions
----------

Exception handling is found in many programming languages, including
Haskell. It can be useful because, when a problem occurs, it can provide
an easy way of handling it, even if it occurred several layers down
through a chain of function calls. With exceptions, it\'s not necessary
to check the return value of every function call to check for errors,
and take care to produce a return value that reflects the error, as C
programmers must do. In Haskell, thanks to monads and the `Either` and
`Maybe` types, you can often achieve the same effects in pure code
without the need to use exceptions and exception handling.

Some problems---especially those involving I/O---call for working with
exceptions. In Haskell, exceptions may be thrown from any location in
the program. However, due to the unspecified evaluation order, they can
only be caught in the `IO` monad. Haskell exception handling doesn\'t
involve special syntax as it does in Python or Java. Rather, the
mechanisms to catch and handle exceptions are---surprise---functions.

### First Steps with Exceptions

In the `Control.Exception` module, various functions and types relating
to exceptions are defined. There is an `Exception` type defined there;
all exceptions are of type `Exception`. There are also functions for
catching and handling exceptions. Let\'s start by looking at `try`,
which has type `IO a -> IO (Either Exception a)`. This wraps an `IO`
action with exception handling. If an exception was thrown, it will
return a `Left` value with the exception; otherwise, a `Right` value
with the original result. Let\'s try this out in `ghci`. We\'ll first
trigger an unhandled exception, and then try to catch it.

``` {.screen}
ghci> :m Control.Exception
ghci> let x = 5 `div` 0
ghci> let y = 5 `div` 1
ghci> print x
*** Exception: divide by zero
ghci> print y
5
ghci> try (print x)
Left divide by zero
ghci> try (print y)
5
Right ()
```

Notice that no exception was thrown by the `let` statements. That\'s to
be expected due to lazy evaluation; the division by zero won\'t be
attempted until it is demanded by the attempt to print out `x`. Also,
notice that there were two lines of output from `try (print y)`. The
first line was produced by `print`, which displayed the digit 5 on the
terminal. The second was produced by `ghci`, which is showing you that
`print y` returned `()` and didn\'t throw an exception.

### Laziness and Exception Handling

Now that you know how `try` works, let\'s try another experiment. Let\'s
say we want to catch the result of `try` for future evaluation, so we
can handle the result of division. Perhaps we would do it like this:

``` {.screen}
ghci> result <- try (return x)
Right *** Exception: divide by zero
```

What happened here? Let\'s try to piece it together, and illustrate with
another attempt:

``` {.screen}
ghci> let z = undefined
ghci> try (print z)
Left Prelude.undefined
ghci> result <- try (return z)
Right *** Exception: Prelude.undefined
```

As before, assigning `undefined` to `z` was not a problem. The key to
this puzzle, and to the division puzzle, lies with lazy evaluation.
Specifically, it lies with `return`, which does not force the evaluation
of its argument; it only wraps it up. So, the result of
`try (return undefined)` would be `Right undefined`. Now, `ghci` wants
to display this result on the terminal. It gets as far as printing out
`"Right "`, but you can\'t print out `undefined` (or the result of
division by zero). So when you see the exception message, it\'s coming
from `ghci`, not your program.

This is a key point. Let\'s think about why our earlier example worked
and this one didn\'t. Earlier, we put `print x` inside `try`. Printing
the value of something, of course, requires it to be evaluated, so the
exception was detected at the right place. But simply using `return`
does not force evaluation. To solve this problem, the
`Control.Exception` module defines the `evaluate` function. It behaves
just like `return`, but forces its argument to be evaluated immediately.
Let\'s try it:

``` {.screen}
ghci> let z = undefined
ghci> result <- try (evaluate z)
Left Prelude.undefined
ghci> result <- try (evaluate x)
Left divide by zero
```

There, that\'s what was expected. This worked for both `undefined` and
our division by zero example.

::: {.TIP}
Tip

Remember: whenever you are trying to catch exceptions thrown by pure
code, use `evaluate` instead of `return` inside your exception-catching
function.
:::

### Using handle

Often, you may wish to perform one action if a piece of code completes
without an exception, and a different action otherwise. For situations
like this, there\'s a function called `handle`. This function has type
`(Exception -> IO a) -> IO a -> IO a`. That is, it takes two parameters:
the first is a function to call in the event there is an exception while
performing the second. Here\'s one way we could use it:

``` {.screen}
ghci> :m Control.Exception
ghci> let x = 5 `div` 0
ghci> let y = 5 `div` 1
ghci> handle (\_ -> putStrLn "Error calculating result") (print x)
Error calculating result
ghci> handle (\_ -> putStrLn "Error calculating result") (print y)
5
```

This way, we can print out a nice message if there is an error in the
calculations. It\'s nicer than having the program crash with a division
by zero error, for sure.

### Selective Handling of Exceptions

One problem with the above example is that it prints
`"Error calculating result"` for *any* exception. There may have been an
exception other than a division by zero exception. For instance, there
may have been an error displaying the output, or some other exception
could have been thrown by the pure code.

There\'s a function `handleJust` for these situations. It lets you
specify a test to see whether you are interested in a given exception.
Let\'s take a look:

``` {.example}
#+CAPTION: hj1.hs
import Control.Exception

catchIt :: Exception -> Maybe ()
catchIt (ArithException DivideByZero) = Just ()
catchIt _ = Nothing

handler :: () -> IO ()
handler _ = putStrLn "Caught error: divide by zero"

safePrint :: Integer -> IO ()
safePrint x = handleJust catchIt handler (print x)
```

`catchIt` defines a function that decides whether or not we\'re
interested in a given exception. It returns `Just` if so, and `Nothing`
if not. Also, the value attached to `Just` will be passed to our
handler. We can now use `safePrint` nicely:

``` {.screen}
ghci> :l hj1.hs
[1 of 1] Compiling Main             ( hj1.hs, interpreted )
Ok, modules loaded: Main.
ghci> let x = 5 `div` 0
ghci> let y = 5 `div` 1
ghci> safePrint x
Caught error: divide by zero
ghci> safePrint y
5
```

The `Control.Exception` module also presents a number of functions that
we can use as part of the test in `handleJust` to narrow down the kinds
of exceptions we care about. For instance, there is a function
`arithExceptions` of type `Exception -> Maybe ArithException` that will
pick out any `ArithException`, but ignore any other one. We could use it
like this:

``` {.example}
import Control.Exception

handler :: ArithException -> IO ()
handler e = putStrLn $ "Caught arithmetic error: " ++ show e

safePrint :: Integer -> IO ()
safePrint x = handleJust arithExceptions handler (print x)
```

In this way, we can catch all types of `ArithException`, but still let
other exceptions pass through unmodified and uncaught. We can see it
work like so:

``` {.screen}
ghci> :l hj2.hs
[1 of 1] Compiling Main             ( hj2.hs, interpreted )
Ok, modules loaded: Main.
ghci> let x = 5 `div` 0
ghci> let y = 5 `div` 1
ghci> safePrint x
Caught arithmetic error: divide by zero
ghci> safePrint y
5
```

Of particular interest, you might notice the `ioErrors` test, which
corresponds to the large class of I/O-related exceptions.

### I/O Exceptions

Perhaps the largest source of exceptions in any program is I/O. All
sorts of things can go wrong when dealing with the outside world: disks
can be full, networks can go down, or files can be empty when you expect
them to have data. In Haskell, an I/O exception is just like any other
exception in that can be represented by the `Exception` data type. On
the other hand, because there are so many types of I/O exceptions, a
special module---\~System.IO.Error\~ exists for dealing with them.

`System.IO.Error` defines two functions: `catch` and `try` which, like
their counterparts in `Control.Exception`, are used to deal with
exceptions. Unlike the `Control.Exception` functions, however, these
functions will only trap I/O errors, and will pass all other exceptions
through uncaught. In Haskell, I/O errors all have type `IOError`, which
is defined as the same as `IOException`.

\#+BEGIN~WARNING~ Be careful which names you use

Because both `System.IO.Error` and `Control.Exception` define functions
with the same names, if you import both in your program, you will get an
error message about an ambiguous reference to a function. You can import
one or the other module `qualified`, or hide the symbols from one module
or the other.

Note that `Prelude` exports `System.IO.Error`\'s version of `catch`,
*not* the version provided by `Control.Exception`. Remember that the
former can only catch I/O errors, while the latter can catch all
exceptions. In other words, the `catch` in `Control.Exception` is almost
always the one you will want, but it is *not* the one you will get by
default. \#+END~WARNING~

Let\'s take a look at one approach to using exceptions in the I/O system
to our benefit. Back in [the section called \"Working With Files and
Handles\"](7-io.org::*Working With Files and Handles) program that used
an imperative style to read lines from a file one by one. Although we
subsequently demonstrated more compact, \"Haskelly\" ways to solve that
problem, let\'s revisit that example here. In the `mainloop` function,
we had to explicitly test if we were at the end of the input file before
each attempt to read a line from it. Instead, we could check if the
attempt to read a line resulted in an `EOF` error, like so:

``` {.example}
import System.IO
import System.IO.Error
import Data.Char(toUpper)

main :: IO ()
main = do
       inh <- openFile "input.txt" ReadMode
       outh <- openFile "output.txt" WriteMode
       mainloop inh outh
       hClose inh
       hClose outh

mainloop :: Handle -> Handle -> IO ()
mainloop inh outh =
    do input <- try (hGetLine inh)
       case input of
         Left e ->
             if isEOFError e
                then return ()
                else ioError e
         Right inpStr ->
             do hPutStrLn outh (map toUpper inpStr)
                mainloop inh outh
```

Here, we use the `System.IO.Error` version of `try` to check whether
`hGetLine` threw an `IOError`. If it did, we use `isEOFError` (defined
in `System.IO.Error`) to see if the thrown exception indicated that we
reached the end of the file. If it did, we exit the loop. If the
exception was something else, we call `ioError` to re-throw it.

There are many such tests and ways to extract information from `IOError`
defined in `System.IO.Error`. We recommend that you consult that page in
the library reference when you need to know about them.

### Throwing Exceptions

Thus far, we have talked in detail about handling exceptions. There is
another piece to the puzzle: throwing exceptions[^4]. In the examples we
have visited so far in this chapter, the Haskell system throws
exceptions for you. However, it is possible to throw any exception
yourself. We\'ll show you how.

You\'ll notice that most of these functions appear to return a value of
type `a` or `IO a`. This means that the function can appear to return a
value of any type. In fact, because these functions throw exceptions,
they never \"return\" anything in the normal sense. These return values
let you use these functions in various contexts where various different
types are expected.

Let\'s start our tour of ways to throw exceptions with the functions in
`Control.Exception`. The most generic function is `throw`, which has
type `Exception -> a`. This function can throw any `Exception`, and can
do so in a pure context. There is a companion function `throwIO` with
type `Exception -> IO a` that throws an exception in the `IO` monad.
Both functions require an `Exception` to throw. You can craft an
`Exception` by hand, or reuse an `Exception` that was previously
created.

There is also a function `ioError`, which is defined identically in both
`Control.Exception` and `System.IO.Error` with type `IOError -> IO a`.
This is used when you want to generate an arbitrary I/O-related
exception.

### Dynamic Exceptions

This makes use of two little-used Haskell modules: `Data.Dynamic` and
`Data.Typeable`. We will not go into a great level of detail on those
modules here, but will give you the tools you need to craft and use your
own dynamic exception type.

In [Chapter 21, *Using Databases*](21-using-databases.org), you will see
that the HDBC database library uses dynamic exceptions to indicate
errors from SQL databases back to applications. Errors from database
engines often have three components: an integer that represents an error
code, a state, and a human-readable error message. We will build up our
own implementation of the HDBC `SqlError` type here in this chapter.
Let\'s start with the data structure representing the error itself:

``` {.example}
{-# LANGUAGE DeriveDataTypeable #-}

import Data.Dynamic
import Control.Exception

data SqlError = SqlError {seState :: String,
                          seNativeError :: Int,
                          seErrorMsg :: String}
                deriving (Eq, Show, Read, Typeable)
```

By deriving the `Typeable` type class, we\'ve made this type available
for dynamically typed programming. In order for GHC to automatically
generate a `Typeable` instance, we had to enable the
`DeriveDataTypeable` language extension[^5].

Now, let\'s define a `catchSql` and a `handleSql` that can be used to
catch an exception that is an `SqlError`. Note that the regular `catch`
and `handle` functions cannot catch our `SqlError`, because it is not a
type of `Exception`.

``` {.example}
{- | Execute the given IO action.

If it raises a 'SqlError', then execute the supplied
handler and return its return value.  Otherwise, proceed
as normal. -}
catchSql :: IO a -> (SqlError -> IO a) -> IO a
catchSql = catchDyn

{- | Like 'catchSql', with the order of arguments reversed. -}
handleSql :: (SqlError -> IO a) -> IO a -> IO a
handleSql = flip catchSql
```

These functions are simply thin wrappers around `catchDyn`, which has
type `Typeable exception => IO a -> (exception -> IO a) -> IO a`. We
here simply restrict the type of this so that it catches only SQL
exceptions.

Normally, when an exception is thrown, but not caught anywhere, the
program will crash and will display the exception to standard error.
With a dynamic exception, however, the system will not know how to
display this, so you will simply see an unhelpful \"unknown exception\"
message. We can provide a utility so that application writers can simply
say `main = handleSqlError $ do ...`, and have confidence that any
exceptions thrown (in that thread) will be displayed. Here\'s how to
write `handleSqlError`:

``` {.example}
{- | Catches 'SqlError's, and re-raises them as IO errors with fail.
Useful if you don't care to catch SQL errors, but want to see a sane
error message if one happens.  One would often use this as a 
high-level wrapper around SQL calls. -}
handleSqlError :: IO a -> IO a
handleSqlError action =
    catchSql action handler
    where handler e = fail ("SQL error: " ++ show e)
```

Finally, let\'s give you an example of how to throw an `SqlError` as an
exception. Here\'s a function that will do just that:

``` {.example}
throwSqlError :: String -> Int -> String -> a
throwSqlError state nativeerror errormsg =
    throwDyn (SqlError state nativeerror errormsg)

throwSqlErrorIO :: String -> Int -> String -> IO a
throwSqlErrorIO state nativeerror errormsg =
    evaluate (throwSqlError state nativeerror errormsg)
```

::: {.TIP}
Tip

As a reminder, `evaluate` is like `return` but forces the evaluation of
its argument.
:::

This completes our dynamic exception support. That was a lot of code,
and you may not have needed that much, but we wanted to give you an
example of the dynamic exception itself and the utilities that often go
with it. In fact, these examples reflect almost exactly what is present
in the HDBC library. Let\'s play with these in `ghci` for a bit:

``` {.screen}
ghci> :l dynexc.hs
[1 of 1] Compiling Main             ( dynexc.hs, interpreted )
Ok, modules loaded: Main.
ghci> throwSqlErrorIO "state" 5 "error message"
*** Exception: (unknown)
ghci> handleSqlError $ throwSqlErrorIO "state" 5 "error message"
*** Exception: user error (SQL error: SqlError {seState = "state", seNativeError = 5, seErrorMsg = "error message"})
ghci> handleSqlError $ fail "other error"
*** Exception: user error (other error)
```

From this, you can see that `ghci` doesn\'t know how to display an SQL
error by itself. However, you can also see that our `handleSqlError`
function helped out with that, but also passed through other errors
unmodified. Let\'s finally try out a custom handler:

``` {.screen}
ghci> handleSql (fail . seErrorMsg) (throwSqlErrorIO "state" 5 "my error")
*** Exception: user error (my error)
```

Here, we defined a custom error handler that threw a new exception,
consisting of the message in the `seErrorMsg` field of the `SqlError`.
You can see that it worked as intended.

Exercises
---------

1.  Take the `Either` example and made it work with laziness in the
    style of the `Maybe` example.

Error handling in monads
------------------------

Because we must catch exceptions in the `IO` monad, if we try to use
them inside a monad, or in a stack of monad transformers, we\'ll get
bounced out to the `IO` monad. This is almost never what we would
actually like.

We defined a `MaybeT` transformer in [the section called \"Understanding
monad transformers by building
one\"](18-monad-transformers.org::*Understanding monad transformers by building one)
but it is more useful as an aid to understanding than a programming
tool. Fortunately, a dedicated---and more useful---monad transformer
already exists: `ErrorT`, which is defined in the `Control.Monad.Error`
module.

The `ErrorT` transformer lets us add exceptions to a monad, but it uses
its own special exception machinery, separate from that provided the
`Control.Exception` module. It gives us some interesting capabilities.

-   If we stick with the `ErrorT` interfaces, we can both throw and
    catch exceptions within this monad.
-   Following the naming pattern of other monad transformers, the
    execution function is named `runErrorT`. An uncaught ErrorT
    exception will stop propagating upwards when it reaches `runErrorT`.
    We will not be kicked out to the `IO` monad.
-   We control the type our exceptions will have.

::: {.NOTE}
Do not confuse `ErrorT` with regular exceptions

If we use the `throw` function from `Control.Exception` inside `ErrorT`
(or if we use `error` or `undefined`), we will *still* be bounced out to
the `IO` monad.
:::

As with other `mtl` monads, the interface that `ErrorT` provides is
defined by a type class.

``` {.example}
class (Monad m) => MonadError e m | m -> e where
    throwError :: e             -- error to throw
               -> m a

    catchError :: m a           -- action to execute
               -> (e -> m a)    -- error handler
               -> m a
```

The type variable `e` represents the error type we want to use. Whatever
our error type is, we must make it an instance of the `Error` type
class.

``` {.example}
class Error a where
    -- create an exception with no message
    noMsg  :: a

    -- create an exception with a message
    strMsg :: String -> a
```

The `strMsg` function is used by `ErrorT`\'s implementation of `fail`.
It throws `strMsg` as an exception, passing it the string argument it
received. As for `noMsg`, it is used to provide an `mzero`
implementation for the `MonadPlus` type class.

To support the `strMsg` and `noMsg` functions, our `ParseError` type
will have a `Chatty` constructor. This will be used as the constructor
if, for example, someone calls `fail` in our monad.

One last piece of plumbing that we need to know about is the type of the
execution function `runErrorT`.

``` {.screen}
ghci> :t runErrorT
runErrorT :: ErrorT e m a -> m (Either e a)
```

### A tiny parsing framework

To illustrate the use of `ErrorT`, let\'s develop the bare bones of a
parsing library similar to Parsec.

``` {.example}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}

import Control.Monad.Error
import Control.Monad.State
import qualified Data.ByteString.Char8 as B

data ParseError = NumericOverflow
                | EndOfInput
                | Chatty String
                  deriving (Eq, Ord, Show)

instance Error ParseError where
    noMsg  = Chatty "oh noes!"
    strMsg = Chatty
```

For our parser\'s state, we will create a very small monad transformer
stack. A `State` monad carries around the `ByteString` to parse, and
stacked on top is `ErrorT` to provide error handling.

``` {.example}
newtype Parser a = P {
      runP :: ErrorT ParseError (State B.ByteString) a
    } deriving (Functor, Applicative, Monad, MonadError ParseError)
```

As usual, we have wrapped our monad stack in a `newtype`. This costs us
nothing in performance, but adds type safety. We have deliberately
avoided deriving an instance of `MonadState
B.ByteString`. This means that users of the `Parser` monad will not be
able to use `get` or `put` to query or modify the parser\'s state. As a
result, we force ourselves to do some manual lifting to get at the
`State` monad in our stack. This is, however, very easy to do.

``` {.example}
liftP :: State B.ByteString a -> Parser a
liftP m = P (lift m)

satisfy :: (Char -> Bool) -> Parser Char
satisfy p = do
  s <- liftP get
  case B.uncons s of
    Nothing         -> throwError EndOfInput
    Just (c, s')
        | p c       -> liftP (put s') >> return c
        | otherwise -> throwError (Chatty "satisfy failed")
```

The `catchError` function is useful for tasks beyond simple error
handling. For instance, we can easily defang an exception, turning it
into a more friendly form.

``` {.example}
optional :: Parser a -> Parser (Maybe a)
optional p = (Just `liftM` p) `catchError` \_ -> return Nothing
```

Our execution function merely plugs together the various layers, and
rearranges the result into a tidier form.

``` {.example}
runParser :: Parser a -> B.ByteString
          -> Either ParseError (a, B.ByteString)
runParser p bs = case runState (runErrorT (runP p)) bs of
                   (Left err, _) -> Left err
                   (Right r, bs) -> Right (r, bs)
```

If we load this into `ghci`, we can put it through its paces.

``` {.screen}
ghci> :m +Data.Char
ghci> let p = satisfy isDigit
Loading package array-0.1.0.0 ... linking ... done.
Loading package bytestring-0.9.0.1 ... linking ... done.
Loading package mtl-1.1.0.0 ... linking ... done.
ghci> runParser p (B.pack "x")
Left (Chatty "satisfy failed")
ghci> runParser p (B.pack "9abc")
Right ('9',"abc")
ghci> runParser (optional p) (B.pack "x")
Right (Nothing,"x")
ghci> runParser (optional p) (B.pack "9a")
Right (Just '9',"a")
```

### Exercises

1.  Write a many parser, with type `Parser a -> Parser [a]`. It should
    apply a parser until it fails.
2.  Use many to write an int parser, with type `Parser Int`. It should
    accept negative as well as positive integers.
3.  Modify your int parser to throw a `NumericOverflow` exception if it
    detects a numeric overflow while parsing.

Footnotes
---------

[^1]: We\'re using integral division here, so `50 / 8` shows as `6`
    instead of `6.25`. We\'re not using floating-point arithmetic in
    this example because division by zero with a `Double` produces the
    special value `Infinity` rather than an error.

[^2]: For an introduction to `Maybe`, refer to [the section called \"A
    more controlled
    approach\"](3-defining-types-streamlining-functions.org::*A more controlled approach)

[^3]: For more information on `Either`, refer to [the section called
    \"Handling errors through API
    design\"](8-efficient-file-processing-regular-expressions-and-file-name-matching.org::*Handling errors through API design)

[^4]: In some other languages, throwing an exception is referred to as
    *raising* it.

[^5]: It is possible to derive `Typeable` instances by hand, but that is
    cumbersome.
