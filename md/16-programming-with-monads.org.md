Chapter 15. Programming with monads
===================================

Golfing practice: association lists
-----------------------------------

Web clients and servers often pass information around as a simple
textual list of key-value pairs.

``` {.haskell}
name=Attila+%42The+Hun%42&occupation=Khan
```

The encoding is named `application/x-www-form-urlencoded`, and it\'s
easy to understand. Each key-value pair is separated by an `&`
character. Within a pair, a key is a series of characters, followed by
an `=`, followed by a value.

We can obviously represent a key as a `String`, but the HTTP
specification is not clear about whether a key must be followed by a
value. We can capture this ambiguity by representing a value as a
`Maybe String`. If we use `Nothing` for a value, then there was no value
present. If we wrap a string in `Just`, then there was a value. Using
`Maybe` lets us distinguish between \"no value\" and \"empty value\".

Haskell programmers use the name *association list* for the type
`[(a, b)]`, where we can think of each element as an association between
a key and a value. The name originates in the Lisp community, where
it\'s usually abbreviated as an *alist*. We could thus represent the
above string as the following Haskell value.

``` {.haskell}
[("name",       Just "Attila \"The Hun\""),
 ("occupation", Just "Khan")]
```

In [the section called \"Parsing an URL-encoded query
string\"](14-using-parsec.org::*Parsing an URL-encoded query string)
parsed an `application/x-www-form-urlencoded` string, and represented
the result as an alist of `[(String, Maybe String)]`. Let\'s say we want
to use one of these alists to fill out a data structure.

<div>

[MovieReview.hs]{.label}

``` {.haskell}
import Control.Monad

data MovieReview = MovieReview {
      revTitle :: String
    , revUser :: String
    , revReview :: String
    }
```

</div>

We\'ll begin by belabouring the obvious with a naive function.

<div>

[MovieReview.hs]{.label}

``` {.haskell}
simpleReview :: [(String, Maybe String)] -> Maybe MovieReview
simpleReview alist =
  case lookup "title" alist of
    Just (Just title@(_:_)) ->
      case lookup "user" alist of
        Just (Just user@(_:_)) ->
          case lookup "review" alist of
            Just (Just review@(_:_)) ->
                Just (MovieReview title user review)
            _ -> Nothing -- no review
        _ -> Nothing -- no user
    _ -> Nothing -- no title
```

</div>

It only returns a `MovieReview` if the alist contains all of the
necessary values, and they\'re all non-empty strings. However, the fact
that it validates its inputs is its only merit: it suffers badly from
the \"staircasing\" that we\'ve learned to be wary of, and it knows the
intimate details of the representation of an alist.

Since we\'re now well acquainted with the `Maybe` monad, we can tidy up
the staircasing.

<div>

[MovieReview.hs]{.label}

``` {.haskell}
maybeReview alist = do
    title <- lookup1 "title" alist
    user <- lookup1 "user" alist
    review <- lookup1 "review" alist
    return (MovieReview title user review)

lookup1 key alist = case lookup key alist of
                      Just (Just s@(_:_)) -> Just s
                      _ -> Nothing
```

</div>

Although this is much tidier, we\'re still repeating ourselves. We can
take advantage of the fact that the `MovieReview` constructor acts as a
normal, pure function by *lifting* it into the monad, as we discussed in
[the section called \"Mixing pure and monadic
code\"](15-monads.org::*Mixing pure and monadic code)

<div>

[MovieReview.hs]{.label}

``` {.haskell}
liftedReview alist =
    liftM3 MovieReview (lookup1 "title" alist)
                       (lookup1 "user" alist)
                       (lookup1 "review" alist)
```

</div>

We still have some repetition here, but it is dramatically reduced, and
also more difficult to remove.

Generalised lifting
-------------------

Although using `liftM3` tidies up our code, we can\'t use a
`liftM`-family function to solve this sort of problem in general,
because they\'re only defined up to `liftM5` by the standard libraries.
We could write variants up to whatever number we pleased, but that would
amount to drudgery.

If we had a constructor or pure function that took, say, ten parameters,
and decided to stick with the standard libraries you might think we\'d
be out of luck.

Of course, our toolbox isn\'t yet empty. In `Control.Monad`, there\'s a
function named `ap` with an interesting type signature.

``` {.screen}
ghci> :m +Control.Monad
ghci> :type ap
ap :: Monad m => m (a -> b) -> m a -> m b
```

You might wonder who would put a single-argument pure function inside a
monad, and why. Recall, however, that *all* Haskell functions really
take only one argument, and you\'ll begin to see how this might relate
to the `MovieReview` constructor.

``` {.screen}
ghci> :type MovieReview
MovieReview :: String -> String -> String -> MovieReview
```

We can just as easily write that type as
`String -> (String -> (String -> MovieReview))`. If we use plain old
`liftM` to lift `MovieReview` into the `Maybe` monad, we\'ll have a
value of type `Maybe (String -> (String -> (String -> MovieReview)))`.
We can now see that this type is suitable as an argument for `ap`, in
which case the result type will be
`Maybe (String -> (String -> MovieReview))`. We can pass this, in turn,
to `ap`, and continue to chain until we end up with this definition.

<div>

[MovieReview.hs]{.label}

``` {.haskell}
apReview alist =
    MovieReview `liftM` lookup1 "title" alist
                   `ap` lookup1 "user" alist
                   `ap` lookup1 "review" alist
```

</div>

We can chain applications of `ap` like this as many times as we need to,
thereby bypassing the `liftM` family of functions.

Another helpful way to look at `ap` is that it\'s the monadic equivalent
of the familiar `(<*>)` operator: think of pronouncing `ap` as *apply*.
We can see this clearly when we compare the type signatures of the two
functions.

``` {.screen}
ghci> :type (<*>)
(<*>) :: Applicative f => f (a -> b) -> f a -> f b
ghci> :type ap
ap :: Monad m => m (a -> b) -> m a -> m b
```

And that\'s why, as we saw in [Chapter 15, Monads](15-monads.org), they
are synonyms.

Looking for alternatives
------------------------

Here\'s a simple representation of a person\'s phone numbers.

<div>

[VCard.hs]{.label}

``` {.haskell}
import Control.Monad

data Context = Home | Mobile | Business
               deriving (Eq, Show)

type Phone = String

albulena = [(Home, "+355-652-55512")]

nils = [(Mobile, "+47-922-55-512"), (Business, "+47-922-12-121"),
        (Home, "+47-925-55-121"), (Business, "+47-922-25-551")]

twalumba = [(Business, "+260-02-55-5121")]
```

</div>

Suppose we want to get in touch with someone to make a personal call. We
don\'t want their business number, and we\'d prefer to use their home
number (if they have one) instead of their mobile number.

<div>

[VCard.hs]{.label}

``` {.haskell}
onePersonalPhone :: [(Context, Phone)] -> Maybe Phone
onePersonalPhone ps = case lookup Home ps of
                        Nothing -> lookup Mobile ps
                        Just n -> Just n
```

</div>

Of course, if we use `Maybe` as the result type, we can\'t accommodate
the possibility that someone might have more than one number that meet
our criteria. For that, we switch to a list.

<div>

[VCard.hs]{.label}

``` {.haskell}
allBusinessPhones :: [(Context, Phone)] -> [Phone]
allBusinessPhones ps = map snd numbers
    where numbers = case filter (contextIs Business) ps of
                      [] -> filter (contextIs Mobile) ps
                      ns -> ns

contextIs a (b, _) = a == b
```

</div>

Notice that these two functions structure their `case` expressions
similarly: one alternative handles the case where the first lookup
returns an empty result, while the other handles the non-empty case.

``` {.screen}
ghci> onePersonalPhone twalumba
Nothing
ghci> onePersonalPhone albulena
Just "+355-652-55512"
ghci> allBusinessPhones nils
["+47-922-12-121","+47-922-25-551"]
```

Haskell\'s `Control.Monad` module defines a type class, `MonadPlus`,
that lets us abstract the common pattern out of our `case` expressions.

<div>

[MonadPlus.hs]{.label}

``` {.haskell}
class Monad m => MonadPlus m where
   mzero :: m a
   mplus :: m a -> m a -> m a
```

</div>

The value `mzero` represents an empty result, while `mplus` combines two
results into one. Here are the standard definitions of `mzero` and
`mplus` for `Maybe` and lists.

<div>

[MonadPlus.hs]{.label}

``` {.haskell}
instance MonadPlus [] where
   mzero = []
   mplus = (++)

instance MonadPlus Maybe where
   mzero = Nothing

   Nothing `mplus` ys  = ys
   xs      `mplus` _ = xs
```

</div>

We can now use `mplus` to get rid of our `case` expressions entirely.
For variety, let\'s fetch one business and all personal phone numbers.

<div>

[VCard.hs]{.label}

``` {.haskell}
oneBusinessPhone :: [(Context, Phone)] -> Maybe Phone
oneBusinessPhone ps = lookup Business ps `mplus` lookup Mobile ps

allPersonalPhones :: [(Context, Phone)] -> [Phone]
allPersonalPhones ps = map snd $ filter (contextIs Home) ps `mplus`
                                 filter (contextIs Mobile) ps
```

</div>

In these functions, because we know that `lookup` returns a value of
type `Maybe`, and `filter` returns a list, it\'s obvious which version
of `mplus` is going to be used in each case.

What\'s more interesting is that we can use `mzero` and `mplus` to write
functions that will be useful for *any* `MonadPlus` instance. As an
example, here\'s the standard `lookup` function, which returns a value
of type `Maybe`.

``` {.haskell}
lookup :: (Eq a) => a -> [(a, b)] -> Maybe b
lookup _ []                      = Nothing
lookup k ((x,y):xys) | x == k    = Just y
                     | otherwise = lookup k xys
```

We can easily generalise the result type to any instance of `MonadPlus`
as follows.

<div>

[VCard.hs]{.label}

``` {.haskell}
lookupM :: (MonadPlus m, Eq a) => a -> [(a, b)] -> m b
lookupM _ []    = mzero
lookupM k ((x,y):xys)
    | x == k    = return y `mplus` lookupM k xys
    | otherwise = lookupM k xys
```

</div>

This lets us get either no result or one, if our result type is `Maybe`;
all results, if our result type is a list; or something more appropriate
for some other exotic instance of `MonadPlus`.

For small functions, such as those we present above, there\'s little
benefit to using `mplus`. The advantage lies in more complex code and in
code that is independent of the monad in which it executes. Even if you
don\'t find yourself needing `MonadPlus` for your own code, you are
likely to encounter it in other people\'s projects.

### The name `mplus` does not imply addition

Even though the `mplus` function contains the text \"plus\", you should
not think of it as necessarily implying that we\'re trying to add two
values.

Depending on the monad we\'re working in, `mplus` *may* implement an
operation that looks like addition. For example, `mplus` in the list
monad is implemented as the `(++)` operator.

``` {.screen}
ghci> [1,2,3] `mplus` [4,5,6]
[1,2,3,4,5,6]
```

However, if we switch to another monad, the obvious similarity to
addition falls away.

``` {.screen}
ghci> Just 1 `mplus` Just 2
Just 1
```

### Rules for working with `MonadPlus`

Instances of the `MonadPlus` type class must follow a few simple rules,
in addition to the usual monad rules.

An instance must *short circuit* if `mzero` appears on the left of a
bind expression. In other words, an expression `mzero >>= f` must
evaluate to the same result as `mzero` alone.

``` {.haskell}
mzero >>= f == mzero
```

An instance must short circuit if `mzero` appears on the *right* of a
sequence expression.

``` {.haskell}
v >> mzero == mzero
```

### Failing safely with `MonadPlus`

When we introduced the `fail` function in [the section called \"The
Monad type class\"](15-monads.org::*The Monad type class) warn against
its use: in many monads, it\'s implemented as a call to `error`, which
has unpleasant consequences.

The `MonadPlus` type class gives us a gentler way to fail a computation,
without `fail` or `error` blowing up in our faces. The rules that we
introduced above allow us to introduce an `mzero` into our code wherever
we need to, and computation will short circuit at that point.

In the `Control.Monad` module, the standard function `guard` packages up
this idea in a convenient form.

<div>

[MonadPlus.hs]{.label}

``` {.haskell}
guard        :: (MonadPlus m) => Bool -> m ()
guard True   =  return ()
guard False  =  mzero
```

</div>

As a simple example, here\'s a function that takes a number `x` and
computes its value modulo some other number `n`. If the result is zero,
it returns `x`, otherwise the current monad\'s `mzero`.

<div>

[MonadPlus.hs]{.label}

``` {.haskell}
x `zeroMod` n = guard ((x `mod` n) == 0) >> return x
```

</div>

Adventures in hiding the plumbing
---------------------------------

In [the section called \"Using the state monad: generating random
values\"](15-monads.org::*Using the state monad: generating random values)
we showed how to use the `State` monad to give ourselves access to
random numbers in a way that is easy to use.

A drawback of the code we developed is that it\'s *leaky*: someone who
uses it knows that they\'re executing inside the `State` monad. This
means that they can inspect and modify the state of the random number
generator just as easily as we, the authors, can.

Human nature dictates that if we leave our internal workings exposed,
someone will surely come along and monkey with them. For a sufficiently
small program, this may be fine, but in a larger software project, when
one consumer of a library modifies its internals in a way that other
consumers are not prepared for, the resulting bugs can be among the
hardest of all to track down. These bugs occur at a level where we\'re
unlikely to question our basic assumptions about a library until long
after we\'ve exhausted all other avenues of inquiry.

Even worse, once we leave our implementation exposed for a while, and
some well-intentioned person inevitably bypasses our APIs and uses the
implementation directly, we create a nasty quandary for ourselves if we
need to fix a bug or make an enhancement. Either we can modify our
internals, and break code that depends on them; or we\'re stuck with our
existing internals, and must try to find some other way to make the
change we need.

How can we revise our random number monad so that the fact that we\'re
using the `State` monad is hidden? We need to somehow prevent our users
from being able to call `get` or `put`. This is not difficult to do, and
it introduces some tricks that we\'ll reuse often in day-to-day Haskell
programming.

To widen our scope, we\'ll move beyond random numbers, and implement a
monad that supplies unique values of *any* kind. The name we\'ll give to
our monad is `Supply`. We\'ll provide the execution function,
`runSupply`, with a list of values; it will be up to us to ensure that
each one is unique.

<div>

[Supply.hs]{.label}

``` {.haskell}
runSupply :: Supply s a -> [s] -> (a, [s])
```

</div>

The monad won\'t care what the values are: they might be random numbers,
or names for temporary files, or identifiers for HTTP cookies.

Within the monad, every time a consumer asks for a value, the `next`
action will take the next one from the list and give it to the consumer.
Each value is wrapped in a `Maybe` constructor in case the list isn\'t
long enough to satisfy the demand.

<div>

[Supply.hs]{.label}

``` {.haskell}
next :: Supply s (Maybe s)
```

</div>

To hide our plumbing, in our module declaration we only export the type
constructor, the execution function, and the `next` action.

<div>

[Supply.hs]{.label}

``` {.haskell}
module Supply
    (
      Supply
    , next
    , runSupply
    ) where
```

</div>

Since a module that imports the library can\'t see the internals of the
monad, it can\'t manipulate them.

Our plumbing is exceedingly simple: we use a `newtype` declaration to
wrap the existing `State` monad.

<div>

[Supply.hs]{.label}

``` {.haskell}
import Control.Monad.State

newtype Supply s a = S (State [s] a)
```

</div>

The `s` parameter is the type of the unique values we are going to
supply, and `a` is the usual type parameter that we must provide in
order to make our type a monad.

Our use of `newtype` for the `Supply` type and our module header join
forces to prevent our clients from using the `State` monad\'s `get` and
`set` actions. Because our module does not export the `s` data
constructor, clients have no programmatic way to see that we\'re
wrapping the `State` monad, or to access it.

At this point, we\'ve got a type, `Supply`, that we need to make an
instance of the `Monad` type class. We could follow the usual pattern of
defining `(>>=)` and `return`, but this would be pure boilerplate code.
All we\'d be doing is wrapping and unwrapping the `State` monad\'s
versions of `(>>=)` and `return` using our `s` value constructor. Here
is how such code would look.

<div>

[AltSupply.hs]{.label}

``` {.haskell}
import Control.Monad.State

newtype Supply s a = S (State [s] a)

unwrapS :: Supply s a -> State [s] a
unwrapS (S s) = s

instance Functor (Supply s) where
    fmap = liftM

instance Applicative (Supply s) where
    pure = S . pure
    (<*>) = ap

instance Monad (Supply s) where
    s >>= m = S (unwrapS s >>= unwrapS . m)
```

</div>

Haskell programmers are not fond of boilerplate, and sure enough, GHC
has a lovely language extension that eliminates the work. To use it, we
add the following directive to the top of our source file, before the
module header.

<div>

[Supply.hs]{.label}

``` {.haskell}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
```

</div>

Usually, we can only automatically derive instances of a handful of
standard type classes, such as `Show` and `Eq`. As its name suggests,
the `GeneralizedNewtypeDeriving` extension broadens our ability to
derive type class instances, and it is specific to `newtype`
declarations. If the type we\'re wrapping is an instance of any type
class, the extensions can automatically make our new type an instance of
that type class as follows.

<div>

[Supply.hs]{.label}

``` {.haskell}
deriving (Monad)
```

</div>

This takes the underlying type\'s implementations of `(>>=)` and
`return`, adds the necessary wrapping and unwrapping with our `s` data
constructor, and uses the new versions of those functions to derive a
`Monad` instance for us.

What we gain here is very useful beyond just this example. We can use
`newtype` to wrap any underlying type; we selectively expose only those
type class instances that we want; and we expend almost no effort to
create these narrower, more specialised types.

Sadly currently GHC is not able to automatically derive the required
functor and applicative instance of our monad so we still need a little
bit of boilerplate:

<div>

[Supply.hs]{.label}

``` {.haskell}
instance Functor (Supply s) where
    fmap = liftM

instance Applicative (Supply s) where
    pure = return
    (<*>) = ap
```

</div>

Now that we\'ve seen the `GeneralizedNewtypeDeriving` technique, all
that remains is to provide definitions of `next` and `runSupply`.

<div>

[Supply.hs]{.label}

``` {.haskell}
next = S $ do st <- get
              case st of
                [] -> return Nothing
                (x:xs) -> do put xs
                             return (Just x)

runSupply (S m) xs = runState m xs
```

</div>

If we load our module into `ghci`, we can try it out in a few simple
ways.

``` {.screen}
ghci> :load Supply
[1 of 1] Compiling Supply           ( Supply.hs, interpreted )
Ok, one module loaded.
ghci> runSupply next [1,2,3]
(Just 1,[2,3])
ghci> runSupply (liftM2 (,) next next) [1,2,3]
((Just 1,Just 2),[3])
ghci> runSupply (liftM2 (,) next next) [1]
((Just 1,Nothing),[])
```

### Supplying random numbers

If we want to use our `Supply` monad as a source of random numbers, we
have a small difficulty to face. Ideally, we\'d like to be able to
provide it with an infinite stream of random numbers. We can get a
`StdGen` in the `IO` monad, but we must \"put back\" a different
`StdGen` when we\'re done. If we don\'t, the next piece of code to get a
`StdGen` will get the same state as we did. This means it will generate
the same random numbers as we did, which is potentially catastrophic.

From the parts of the `System.Random` module we\'ve seen so far, it\'s
difficult to reconcile these demands. We can use `getStdRandom`, whose
type ensures that when we get a `StdGen`, we put one back.

``` {.screen}
ghci> :t System.Random.getStdRandom
System.Random.getStdRandom :: (StdGen -> (a, StdGen)) -> IO a
```

We can use `random` to get back a new `StdGen` when they give us a
random number. And we can use `randoms` to get an infinite list of
random numbers. But how do we get both an infinite list of random
numbers *and* a new `StdGen`?

The answer lies with the `RandomGen` type class\'s `split` function,
which takes one random number generator, and turns it into two
generators. Splitting a random generator like this is a most unusual
thing to be able to do: it\'s obviously tremendously useful in a pure
functional setting, but essentially never either necessary or provided
by an impure language.

Using the `split` function, we can use one `StdGen` to generate an
infinite list of random numbers to feed to `runSupply`, while we give
the other back to the `IO` monad.

<div>

[RandomSupply.hs]{.label}

``` {.haskell}
module RandomSupply where

import Supply
import System.Random hiding (next)

randomsIO :: Random a => IO [a]
randomsIO =
    getStdRandom $ \g ->
        let (a, b) = split g
        in (randoms a, b)
```

</div>

If we\'ve written this function properly, our example ought to print a
different random number on each invocation.

``` {.screen}
ghci> :load RandomSupply
[1 of 2] Compiling Supply           ( Supply.hs, interpreted )
[2 of 2] Compiling RandomSupply     ( RandomSupply.hs, interpreted )
Ok, two modules loaded.
ghci> (fst . runSupply next) `fmap` randomsIO
Just 6077368651500926723
ghci> (fst . runSupply next) `fmap` randomsIO
Just (-7180502226926150515)
```

Recall that our `runSupply` function returns both the result of
executing the monadic action and the unconsumed remainder of the list.
Since we passed it an infinite list of random numbers, we compose with
`fst` to ensure that we don\'t get drowned in random numbers when `ghci`
tries to print the result.

### Another round of golf

The pattern of applying a function to one element of a pair, and
constructing a new pair with the other original element untouched, is
common enough in Haskell code that it has been turned into standard
code.

In the `Control.Arrow` module are two functions, `first` and `second`,
that perform this operation.

``` {.screen}
ghci> :m +Control.Arrow
ghci> first (+3) (1,2)
(4,2)
ghci> second odd ('a',1)
('a',True)
```

(Indeed, we already encountered `second`, in [the section called \"JSON
type classes without overlapping
instances\"](6-using-typeclasses.org::*JSON type classes without overlapping instances)
We can use `first` to golf our definition of `randomsIO`, turning it
into a one-liner.

<div>

[RandomGolf.hs]{.label}

``` {.haskell}
import Control.Arrow (first)
import System.Random

randomsIO_golfed :: Random a => IO [a]
randomsIO_golfed = getStdRandom (first randoms . split)
```

</div>

Separating interface from implementation
----------------------------------------

In the previous section, we saw how to hide the fact that we\'re using a
`State` monad to hold the state for our `Supply` monad.

Another important way to make code more modular involves separating its
/interface/---what the code can do---from its /implementation/---how it
does it.

The standard random number generator in `System.Random` is known to be
quite slow. If we use our `randomsIO` function to provide it with random
numbers, then our `next` action will not perform well.

One simple and effective way that we could deal with this is to provide
`Supply` with a better source of random numbers. Let\'s set this idea
aside, though, and consider an alternative approach, one that is useful
in many settings. We will separate the actions we can perform with the
monad from how it works using a type class.

<div>

[SupplyClass.hs]{.label}

``` {.haskell}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE FunctionalDependencies #-}
{-# LANGUAGE MultiParamTypeClasses #-}

import qualified Supply as S

class (Monad m) => MonadSupply s m | m -> s where
    next :: m (Maybe s)
```

</div>

This type class defines the interface that any supply monad must
implement. It bears careful inspection, since it uses several unfamiliar
Haskell language extensions. We will cover each one in the sections that
follow.

### Multi-parameter type classes

How should we read the snippet `MonadSupply s m` in the type class? If
we add parentheses, an equivalent expression is `(MonadSupply s) m`,
which is a little clearer. In other words, given some type variable `m`
that is a monad, we can make it an instance of the type class
`MonadSupply s`. unlike a regular type class, this one has a
*parameter*.

As this language extension allows a type class to have more than one
parameter, its name is `MultiParamTypeClasses`. The parameter `s` serves
the same purpose as the `Supply` type\'s parameter of the same name: it
represents the type of the values handed out by the `next` function.

Notice that we don\'t need to mention `(>>=)` or `return` in the
definition of `MonadSupply s`, since the type class\'s context
(superclass) requires that a `MonadSupply s` must already be a monad.

### Functional dependencies

To revisit a snippet that we ignored earlier, `| m -> s` is a
*functional dependency*, often called a *fundep*. We can read the
vertical bar `|` as \"such that\", and the arrow `->` as \"uniquely
determines\". Our functional dependency establishes a *relationship*
between `m` and `s`.

The availability of functional dependencies is governed by the
`FunctionalDependencies` language pragma.

The purpose behind us declaring a relationship is to help the type
checker. Recall that a Haskell type checker is essentially a theorem
prover, and that it is conservative in how it operates: it insists that
its proofs must terminate. A non-terminating proof results in the
compiler either giving up or getting stuck in an infinite loop.

With our functional dependency, we are telling the type checker that
every time it sees some monad `m` being used in the context of a
`MonadSupply s`, the type `s` is the only acceptable type to use with
it. If we were to omit the functional dependency, the type checker would
simply give up with an error message.

It\'s hard to picture what the relationship between `m` and `s` really
means, so let\'s look at an instance of this type class.

<div>

[SupplyClass.hs]{.label}

``` {.haskell}
instance MonadSupply s (S.Supply s) where
    next = S.next
```

</div>

Here, the type variable `m` is replaced by the type `S.Supply s`. Thanks
to our functional dependency, the type checker now knows that when it
sees a type `S.Supply s`, the type can be used as an instance of the
type class `MonadSupply s`.

If we didn\'t have a functional dependency, the type checker would not
be able to figure out the relationship between the type parameter of the
class `MonadSupply s` and that of the type `Supply s`, and it would
abort compilation with an error. The definition itself would compile;
the type error would not arise until the first time we tried to use it.

To strip away one final layer of abstraction, consider the type
`S.Supply Int`. Without a functional dependency, we could declare this
an instance of `MonadSupply s`. However, if we tried to write code using
this instance, the compiler would not be able to figure out that the
type\'s `Int` parameter needs to be the same as the type class\'s `s`
parameter, and it would report an error.

Functional dependencies can be tricky to understand, and once we move
beyond simple uses, they often prove difficult to work with in practice.
Fortunately, the most frequent use of functional dependencies is in
situations as simple as ours, where they cause little trouble.

### Rounding out our module

If we save our type class and instance in a source file named
`SupplyClass.hs`, we\'ll need to add a module header such as the
following.

<div>

[SupplyClass.hs]{.label}

``` {.haskell}
module SupplyClass
    (
      MonadSupply(..)
    , S.Supply
    , S.runSupply
    ) where
```

</div>

Notice that we\'re re-exporting the `runSupply` and `Supply` names from
this module. It\'s perfectly legal to export a name from one module even
though it\'s defined in another. In our case, it means that client code
only needs to import the `SupplyClass` module, without also importing
the `Supply` module. This reduces the number of \"moving parts\" that a
user of our code needs to keep in mind.

### Programming to a monad\'s interface

Here is a simple function that fetches two values from our `Supply`
monad, formats them as a string, and returns them.

<div>

[Supply.hs]{.label}

``` {.haskell}
showTwo :: (Show s) => Supply s String
showTwo = do
  a <- next
  b <- next
  return (show "a: " ++ show a ++ ", b: " ++ show b)
```

</div>

This code is tied by its result type to our `Supply` monad. We can
easily generalize to any monad that implements our `MonadSupply`
interface by modifying our function\'s type. Notice that the body of the
function remains unchanged.

<div>

[SupplyClass.hs]{.label}

``` {.haskell}
showTwo_class :: (Show s, Monad m, MonadSupply s m) => m String
showTwo_class = do
  a <- next
  b <- next
  return (show "a: " ++ show a ++ ", b: " ++ show b)
```

</div>

The reader monad
----------------

The `State` monad lets us plumb a piece of mutable state through our
code. Sometimes, we would like to be able to pass some *immutable* state
around, such as a program\'s configuration data. We could use the
`State` monad for this purpose, but we could then find ourselves
accidentally modifying data that should remain unchanged.

Let\'s forget about monads for a moment and think about what a
*function* with our desired characteristics ought to do. It should
accept a value of some type `e` (for *environment*) that represents the
data that we\'re passing in, and return a value of some other type `a`
as its result. The overall type we want is `e -> a`.

To turn this type into a convenient `Monad` instance, we\'ll wrap it in
a `newtype`.

<div>

[SupplyInstance.hs]{.label}

``` {.haskell}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE MultiParamTypeClasses  #-}

module SupplyInstance where

import Control.Monad
import SupplyClass

newtype Reader e a = R { runReader :: e -> a }
```

</div>

Making this into a `Monad` instance doesn\'t take much work.

<div>

[SupplyInstance.hs]{.label}

``` {.haskell}
instance Functor (Reader a) where
    fmap = liftM

instance Applicative (Reader a) where
    pure a = R $ \_ -> a
    (<*>) = ap

instance Monad (Reader e) where
    m >>= k = R $ \r -> runReader (k (runReader m r)) r
```

</div>

We can think of our value of type `e` as an *environment* in which
we\'re evaluating some expression. The `return` action should have the
same effect no matter what the environment is, so our version ignores
its environment.

Our definition of `(>>=)` is a little more complicated, but only because
we have to make the environment---here the variable \~r\~---available
both in the current computation and in the computation we\'re chaining
into.

How does a piece of code executing in this monad find out what\'s in its
environment? It simply has to `ask`.

<div>

[SupplyInstance.hs]{.label}

``` {.haskell}
ask :: Reader e e
ask = R id
```

</div>

Within a given chain of actions, every invocation of `ask` will return
the same value, since the value stored in the environment doesn\'t
change. Our code is easy to test in `ghci`.

``` {.screen}
ghci> :l SupplyInstance.hs
[1 of 1] Compiling Main             ( SupplyInstance.hs, interpreted )
Ok, one module loaded.
ghci> runReader (ask >>= \x -> return (x * 3)) 2
6
```

The `Reader` monad is included in the standard `mtl` library, which is
usually bundled with GHC. You can find it in the `Control.Monad.Reader`
module. The motivation for this monad may initially seem a little thin,
because it is most often useful in complicated code. We\'ll often need
to access a piece of configuration information deep in the bowels of a
program; passing that information in as a normal parameter would require
a painful restructuring of our code. By hiding this information in our
monad\'s plumbing, intermediate functions that don\'t care about the
configuration information don\'t need to see it.

The clearest motivation for the `Reader` monad will come in [Chapter 18,
*Monad transformers*](18-monad-transformers.org), when we discuss
combining several monads to build a new monad. There, we\'ll see how to
gain finer control over state, so that our code can modify some values
via the `State` monad, while other values remain immutable, courtesy of
the `Reader` monad.

A return to automated deriving
------------------------------

Now that we know about the `Reader` monad, let\'s use it to create an
instance of our `MonadSupply` type class. To keep our example simple,
we\'ll violate the spirit of `MonadSupply` here: our `next` action will
always return the same value, instead of always returning a different
value.

It would be a bad idea to directly make the `Reader` type an instance of
the `MonadSupply` class, because then *any* `Reader` could act as a
`MonadSupply`. This would usually not make any sense.

Instead, we create a `newtype` based on `Reader`. The `newtype` hides
the fact that we\'re using `Reader` internally. We must now make our
type an instance of both of the type classes we care about. With the
`GeneralizedNewtypeDeriving` extension enabled, GHC will do most of the
hard work for us.

<div>

[SupplyInstance.hs]{.label}

``` {.haskell}
newtype MySupply e a = MySupply { runMySupply :: Reader e a }
    deriving (Monad)

instance Functor (MySupply a) where
    fmap = liftM

instance Applicative (MySupply a) where
    pure = return
    (<*>) = ap

instance MonadSupply e (MySupply e) where
    next = MySupply $ do
             v <- ask
             return (Just v)

    -- more concise:
    -- next = MySupply (Just `liftM` ask)
```

</div>

Notice that we must make our type an instance of `MonadSupply e`, not
`MonadSupply`. If we omit the type variable, the compiler will complain.

To try out our `MySupply` type, we\'ll first create a simple function
that should work with any `MonadSupply` instance.

<div>

[SupplyInstance.hs]{.label}

``` {.haskell}
xy :: (Num s, MonadSupply s m) => m s
xy = do
  Just x <- next
  Just y <- next
  return (x * y)
```

</div>

If we use this with our `Supply` monad and `randomsIO` function, we get
a different answer every time, as we expect.

<div>

[RandomSupplyInstance.hs]{.label}

``` {.haskell}
import Supply
import SupplyInstance
import RandomSupply
```

</div>

``` {.screen}
ghci> :l RandomSupplyInstance.hs
[1 of 5] Compiling Supply           ( Supply.hs, interpreted )
[2 of 5] Compiling RandomSupply     ( RandomSupply.hs, interpreted )
[3 of 5] Compiling SupplyClass      ( SupplyClass.hs, interpreted )
[4 of 5] Compiling SupplyInstance   ( SupplyInstance.hs, interpreted)
[5 of 5] Compiling Main             ( RandomSupplyInstance.hs, interp reted )
Ok, five modules loaded.
ghci> (fst . runSupply xy) `fmap` randomsIO
-15697064270863081825448476392841917578
ghci> (fst . runSupply xy) `fmap` randomsIO
17182983444616834494257398042360119726
```

Because our `MySupply` monad has two layers of `newtype` wrapping, we
can make it easier to use by writing a custom execution function for it.

<div>

[SupplyInstance.hs]{.label}

``` {.haskell}
runMS :: MySupply i a -> i -> a
runMS = runReader . runMySupply
```

</div>

When we apply our `xy` action using this execution function, we get the
same answer every time. Our code remains the same, but because we are
executing it in a different implementation of `MonadSupply`, its
behavior has changed.

``` {.screen}
ghci> :r
[4 of 5] Compiling SupplyInstance   ( SupplyInstance.hs, interpreted)
[5 of 5] Compiling Main             ( RandomSupplyInstance.hs, interp reted ) [SupplyInstance changed]
Ok, five modules loaded.
ghci> runMS xy 2
4
ghci> runMS xy 2
4
```

Like our `MonadSupply` type class and `Supply` monad, almost all of the
common Haskell monads are built with a split between interface and
implementation. For example, the `get` and `put` functions that we
introduced as \"belonging to\" the `State` monad are actually methods of
the `MonadState` type class; the `State` type is an instance of this
class.

Similarly, the standard `Reader` monad is an instance of the
`MonadReader` type class, which specifies the `ask` method.

While the separation of interface and implementation that we\'ve
discussed above is appealing for its architectural cleanliness, it has
important practical applications that will become clearer later. When we
start combining monads in [Chapter 18, *Monad
transformers*](18-monad-transformers.org), we will save a lot of effort
through the use of `GeneralizedNewtypeDeriving` and type classes.

Hiding the `IO` monad
---------------------

The blessing and curse of the `IO` monad is that it is extremely
powerful. If we believe that careful use of types helps us to avoid
programming mistakes, then the `IO` monad should be a great source of
unease. Because the `IO` monad imposes no restrictions on what we can
do, it leaves us vulnerable to all kinds of accidents.

How can we tame its power? Let\'s say that we would like to guarantee to
ourselves that a piece of code can read and write files on the local
file system, but that it will not access the network. We can\'t use the
plain `IO` monad, because it won\'t restrict us.

### Using a `newtype`

Let\'s create a module that provides a small set of functionality for
reading and writing files.

<div>

[HandleIO.hs]{.label}

``` {.haskell}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}

module HandleIO
    (
      HandleIO
    , Handle
    , IOMode(..)
    , runHandleIO
    , openFile
    , hClose
    , hPutStrLn
    ) where

import Control.Monad
import Control.Monad.Trans (MonadIO(..))
import System.Directory (removeFile)
import System.IO (Handle, IOMode(..))
import qualified System.IO
```

</div>

Our first approach to creating a restricted version of `IO` is to wrap
it with a `newtype`.

<div>

[HandleIO.hs]{.label}

``` {.haskell}
newtype HandleIO a = HandleIO { runHandleIO :: IO a }
    deriving (Monad)

instance Functor HandleIO where
    fmap = liftM

instance Applicative HandleIO where
    pure = return
    (<*>) = ap
```

</div>

We do the by-now familiar trick of exporting the type constructor and
the `runHandleIO` execution function from our module, but not the data
constructor. This will prevent code running within the `HandleIO` monad
from getting hold of the `IO` monad that it wraps.

All that remains is for us to wrap each of the actions we want our monad
to allow. This is a simple matter of wrapping each `IO` with a
`HandleIO` data constructor.

<div>

[HandleIO.hs]{.label}

``` {.haskell}
openFile :: FilePath -> IOMode -> HandleIO Handle
openFile path mode = HandleIO (System.IO.openFile path mode)

hClose :: Handle -> HandleIO ()
hClose = HandleIO . System.IO.hClose

hPutStrLn :: Handle -> String -> HandleIO ()
hPutStrLn h s = HandleIO (System.IO.hPutStrLn h s)
```

</div>

We can now use our restricted `HandleIO` monad to perform I/O.

<div>

[HandleIO.hs]{.label}

``` {.haskell}
safeHello :: FilePath -> HandleIO ()
safeHello path = do
  h <- openFile path WriteMode
  hPutStrLn h "hello world"
  hClose h
```

</div>

To run this action, we use `runHandleIO`.

``` {.screen}
ghci> :load HandleIO
[1 of 1] Compiling HandleIO         ( HandleIO.hs, interpreted )
Ok, one module loaded.
ghci> runHandleIO (safeHello "hello_world_101.txt")
ghci> :m +System.Directory
ghci> removeFile "hello_world_101.txt"
```

If we try to sequence an action that runs in the `HandleIO` monad with
one that is not permitted, the type system forbids it.

``` {.screen}
ghci> runHandleIO (safeHello "goodbye" >> removeFile "goodbye")

<interactive>:1:36: error:
    • Couldn't match type ‘IO’ with ‘HandleIO’
      Expected type: HandleIO ()
        Actual type: IO ()
    • In the second argument of ‘(>>)’, namely ‘removeFile "goodbye"’
      In the first argument of ‘runHandleIO’, namely
        ‘(safeHello "goodbye" >> removeFile "goodbye")’
      In the expression:
        runHandleIO (safeHello "goodbye" >> removeFile "goodbye")
```

### Designing for unexpected uses

There\'s one small, but significant, problem with our `HandleIO` monad:
it doesn\'t take into account the possibility that we might occasionally
need an escape hatch. If we define a monad like this, it is likely that
we will occasionally need to perform an I/O action that isn\'t allowed
for by the design of our monad.

Our purpose in defining a monad like this is to make it easier for us to
write solid code in the common case, not to make corner cases
impossible. Let\'s thus give ourselves a way out.

The `Control.Monad.Trans` module defines a \"standard escape hatch\",
the `MonadIO` type class. This defines a single function, `liftIO`,
which lets us embed an `IO` action in another monad.

``` {.screen}
ghci> :m +Control.Monad.Trans
ghci> :info MonadIO
class Monad m => MonadIO (m :: * -> *) where
  liftIO :: IO a -> m a
  {-# MINIMAL liftIO #-}
        -- Defined in ‘Control.Monad.IO.Class’
instance [safe] MonadIO IO -- Defined in ‘Control.Monad.IO.Class’
```

Our implementation of this type class is trivial: we just wrap `IO` with
our data constructor.

<div>

[HandleIO.hs]{.label}

``` {.haskell}
instance MonadIO HandleIO where
    liftIO = HandleIO
```

</div>

With judicious use of `liftIO`, we can escape our shackles and invoke
`IO` actions where necessary.

<div>

[HandleIO.hs]{.label}

``` {.haskell}
tidyHello :: FilePath -> HandleIO ()
tidyHello path = do
  safeHello path
  liftIO (removeFile path)
```

</div>

::: {.TIP}
Automatic derivation and `MonadIO`

We could have had the compiler automatically derive an instance of
`MonadIO` for us by adding the type class to the `deriving` clause of
`HandleIO`. In fact, in production code, this would be our usual
strategy. We avoided that here simply to separate the presentation of
the earlier material from that of `MonadIO`.
:::

### Using type classes

The disadvantage of hiding `IO` in another monad is that we\'re still
tied to a concrete implementation. If we want to swap `HandleIO` for
some other monad, we must change the type of every action that uses
`HandleIO`.

As an alternative, we can create a type class that specifies the
interface we want from a monad that manipulates files.

<div>

[MonadHandle.hs]{.label}

``` {.haskell}
{-# LANGUAGE FunctionalDependencies, MultiParamTypeClasses #-}

module MonadHandle (MonadHandle(..)) where

import System.IO (IOMode(..))

class Monad m => MonadHandle h m | m -> h where
    openFile :: FilePath -> IOMode -> m h
    hPutStr :: h -> String -> m ()
    hClose :: h -> m ()
    hGetContents :: h -> m String

    hPutStrLn :: h -> String -> m ()
    hPutStrLn h s = hPutStr h s >> hPutStr h "\n"
```

</div>

Here, we\'ve chosen to abstract away both the type of the monad and the
type of a file handle. To satisfy the type checker, we\'ve added a
functional dependency: for any instance of `MonadHandle`, there is
exactly one handle type that we can use. When we make the `IO` monad an
instance of this class, we use a regular Handle.

<div>

[MonadHandleIO.hs]{.label}

``` {.haskell}
{-# LANGUAGE FunctionalDependencies, MultiParamTypeClasses #-}

import MonadHandle
import qualified System.IO

import System.IO (IOMode(..))
import Control.Monad.Trans (MonadIO(..), MonadTrans(..))
import System.Directory (removeFile)

import SafeHello

instance MonadHandle System.IO.Handle IO where
    openFile = System.IO.openFile
    hPutStr = System.IO.hPutStr
    hClose = System.IO.hClose
    hGetContents = System.IO.hGetContents
    hPutStrLn = System.IO.hPutStrLn
```

</div>

Because any `MonadHandle` must also be a `Monad`, we can write code that
manipulates files using normal `do` notation, without caring what monad
it will finally execute in.

<div>

[SafeHello.hs]{.label}

``` {.haskell}
module SafeHello where

import MonadHandle
import System.IO(IOMode(..))

safeHello :: MonadHandle h m => FilePath -> m ()
safeHello path = do
  h <- openFile path WriteMode
  hPutStrLn h "hello world"
  hClose h
```

</div>

Because we made `IO` an instance of this type class, we can execute this
action from `ghci`.

``` {.screen}
ghci> :l MonadHandleIo.hs
[1 of 3] Compiling MonadHandle      ( MonadHandle.hs, interpreted )
[2 of 3] Compiling SafeHello        ( SafeHello.hs, interpreted )
[3 of 3] Compiling Main             ( MonadHandleIo.hs, interpreted )
Ok, three modules loaded.
ghci> safeHello "hello to my fans in domestic surveillance"
ghci> removeFile "hello to my fans in domestic surveillance"
```

The beauty of the type class approach is that we can swap one underlying
monad for another without touching much code, as most of our code
doesn\'t know or care about the implementation. For instance, we could
replace `IO` with a monad that compresses files as it writes them out.

Defining a monad\'s interface through a type class has a further
benefit. It lets other people hide our implementation in a `newtype`
wrapper, and automatically derive instances of just the type classes
they want to expose.

### Isolation and testing

In fact, because our `safeHello` function doesn\'t use the `IO` type, we
can even use a monad that *can\'t* perform I/O. This allows us to test
code that would normally have side effects in a completely pure,
controlled environment.

To do this, we will create a monad that doesn\'t perform I/O, but
instead logs every file-related event for later processing.

<div>

[WriterIO.hs]{.label}

``` {.haskell}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE TypeSynonymInstances #-}

import Control.Monad.Writer
import MonadHandle
import SafeHello
import System.IO(IOMode(..))

data Event = Open FilePath IOMode
           | Put String String
           | Close String
           | GetContents String
             deriving (Show)
```

</div>

Although we already developed a `Logger` type in [the section called
\"Using a new monad: show your
work!\"](15-monads.org::*Using a new monad: show your work!) we\'ll use
the standard, and more general, `Writer` monad. Like other `mtl` monads,
the API provided by `Writer` is defined in a type class, in this case
`MonadWriter`. Its most useful method is `tell`, which logs a value.

``` {.screen}
ghci> :m +Control.Monad.Writer
ghci> :type tell
tell :: MonadWriter w m => w -> m ()
```

The values we log can be of any `Monoid` type. Since the list type is a
`Monoid`, we\'ll log to a list of `Event`.

We could make `Writer [Event]` an instance of `MonadHandle`, but it\'s
cheap, easy, and safer to make a special-purpose monad.

<div>

[WriterIO.hs]{.label}

``` {.haskell}
newtype WriterIO a = W { runW :: Writer [Event] a }
    deriving (Monad, MonadWriter [Event])
```

</div>

Our execution function simply removes the `newtype` wrapper we added,
then calls the normal Writer monad\'s execution function.

<div>

[WriterIO.hs]{.label}

``` {.haskell}
runWriterIO :: WriterIO a -> (a, [Event])
runWriterIO = runWriter . runW

instance Functor WriterIO where
    fmap = liftM

instance Applicative WriterIO where
    pure = return
    (<*>) = ap

instance MonadHandle FilePath WriterIO where
    openFile path mode = tell [Open path mode] >> return path
    hPutStr h str = tell [Put h str]
    hClose h = tell [Close h]
    hGetContents h = tell [GetContents h] >> return ""
```

</div>

When we try this code out in `ghci`, it gives us a log of the
function\'s file activities.

``` {.screen}
ghci> :load WriterIO
[1 of 3] Compiling MonadHandle      ( MonadHandle.hs, interpreted )
[2 of 3] Compiling SafeHello        ( SafeHello.hs, interpreted )
[3 of 3] Compiling Main             ( WriterIo.hs, interpreted )
Ok, three modules loaded.
ghci> runWriterIO (safeHello "foo")
((),[Open "foo" WriteMode,Put "foo" "hello world",Put "foo" "\n",Close "foo"])
```

### The writer monad and lists

The writer monad uses the monoid\'s `mappend` function every time we use
`tell`. Because `mappend` for lists is `(++)`, lists are not a good
practical choice for use with `Writer`: repeated appends are expensive.
We use lists above purely for simplicity.

In production code, if you want to use the `Writer` monad and you need
list-like behaviour, use a type with better append characteristics. One
such type is the difference list, which we introduced in [the section
called \"Taking advantage of functions as
data\"](13-data-structures.org::*Taking advantage of functions as data)
You don\'t need to roll your own difference list implementation: a well
tuned library is available for download from Hackage, the Haskell
package database. Alternatively, you can use the `Seq` type from the
`Data.Sequence` module, which we introduced in [the section called
\"General purpose
sequences\"](13-data-structures.org::*General purpose sequences)

### Arbitrary I/O revisited

If we use the type class approach to restricting `IO`, we may still want
to retain the ability to perform arbitrary I/O actions. We might try
adding `MonadIO` as a constraint on our type class.

<div>

[MonadHandleIO.hs]{.label}

``` {.haskell}
class (MonadHandle h m, MonadIO m) => MonadHandleIO h m | m -> h

instance MonadHandleIO System.IO.Handle IO

tidierHello :: (MonadHandleIO h m) => FilePath -> m ()
tidierHello path = do
  safeHello path
  liftIO (removeFile path)
```

</div>

This approach has a problem, though: the added `MonadIO` constraint
loses us the ability to test our code in a pure environment, because we
can no longer tell whether a test might have damaging side effects. The
alternative is to move this constraint from the type class, where it
\"infects\" all functions, to only those functions that really need to
perform I/O.

<div>

[MonadHandleIO.hs]{.label}

``` {.haskell}
tidyHello :: (MonadIO m, MonadHandle h m) => FilePath -> m ()
tidyHello path = do
  safeHello path
  liftIO (removeFile path)
```

</div>

We can use pure property tests on the functions that lack `MonadIO`
constraints, and traditional unit tests on the rest.

Unfortunately, we\'ve substituted one problem for another: we can\'t
invoke code with both `MonadIO` and `MonadHandle` constraints from code
that has the `MonadHandle` constraint alone. If we find that somewhere
deep in our `MonadHandle`-only code, we really need the `MonadIO`
constraint, we must add it to all the code paths that lead to this
point.

Allowing arbitrary I/O is risky, and has a profound effect on how we
develop and test our code. When we have to choose between being
permissive on the one hand, and easier reasoning and testing on the
other, we usually opt for the latter.

### Exercises

1.  Using QuickCheck, write a test for an action in the `MonadHandle`
    monad, in order to see if it tries to write to a file handle that is
    not open. Try it out on `safeHello`.
2.  Write an action that tries to write to a file handle that it has
    closed. Does your test catch this bug?
3.  In a form-encoded string, the same key may appear several times,
    with or without values, e.g., `key&key=1&key=2`. What type might you
    use to represent the values associated with a key in this sort of
    string? Write a parser that correctly captures all of the
    information.
