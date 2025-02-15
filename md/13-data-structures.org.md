Chapter 13. Data Structures
===========================

Association Lists
-----------------

Often, we have to deal with data that is unordered but is indexed by a
key. For instance, a Unix administrator might have a list of numeric
UIDs (user IDs) and the textual usernames that they correspond to. The
value of this list lies in being able to look up a textual username for
a given UID, not in the order of the data. In other words, the UID is a
key into a database.

In Haskell, there are several ways to handle data that is structured in
this way. The two most common are association lists and the `Map` type
provided by `Data.Map` module. Association lists are handy because they
are simple. They are standard Haskell lists, so all the familiar list
functions work with association lists. However, for large data sets,
`Map` will have a considerable performance advantage over association
lists. We\'ll use both in this chapter.

An association list is just a normal list containing (key, value)
tuples. The type of a list of mappings from UID to username might be
`[(Integer, String)]`. We could use just about any type[^1] for both the
key and the value.

We can build association lists just we do any other list. Haskell comes
with one built-in function called `Data.List.lookup` to look up data in
an association list. Its type is `Eq a => a -> [(a, b)] -> Maybe b`. Can
you guess how it works from that type? Let\'s take a look in `ghci`.

``` {.screen}
ghci> al = [(1, "one"), (2, "two"), (3, "three"), (4, "four")]
ghci> lookup 1 al
Just "one"
ghci> lookup 5 al
Nothing
```

The `lookup` function is really simple. Here\'s one way we could write
it:

<div>

[lookup.hs]{.label}

``` {.haskell}
myLookup :: Eq a => a -> [(a, b)] -> Maybe b
myLookup _ [] = Nothing
myLookup key ((thiskey,thisval):rest) =
    if key == thiskey
       then Just thisval
       else myLookup key rest
```

</div>

This function returns `Nothing` if passed the empty list. Otherwise, it
compares the key with the key we\'re looking for. If a match is found,
the corresponding value is returned. Otherwise, it searches the rest of
the list.

Let\'s take a look at a more complex example of association lists. On
Unix/Linux machines, there is a file called `/etc/passwd` that stores
usernames, UIDs, home directories, and various other data. We will write
a program that parses such a file, creates an association list, and lets
the user look up a username by giving a UID.

<div>

[passwd-al.hs]{.label}

``` {.haskell}
import Data.List
import System.IO
import Control.Monad(when)
import System.Exit
import System.Environment(getArgs)

main = do
    -- Load the command-line arguments
    args <- getArgs

    -- If we don't have the right amount of args, give an error and abort
    when (length args /= 2) $ do
        putStrLn "Syntax: passwd-al filename uid"
        exitFailure

    -- Read the file lazily
    content <- readFile (args !! 0)

    -- Compute the username in pure code
    let username = findByUID content (read (args !! 1))

    -- Display the result
    case username of
         Just x -> putStrLn x
         Nothing -> putStrLn "Could not find that UID"

-- Given the entire input and a UID, see if we can find a username.
findByUID :: String -> Integer -> Maybe String
findByUID content uid =
    let al = map parseline . lines $ content
        in lookup uid al

-- Convert a colon-separated line into fields
parseline :: String -> (Integer, String)
parseline input =
    let fields = split ':' input
        in (read (fields !! 2), fields !! 0)

{- | Takes a delimiter and a list.  Break up the list based on the
-  delimiter. -}
split :: Eq a => a -> [a] -> [[a]]

-- If the input is empty, the result is a list of empty lists.
split _ [] = [[]]
split delim str =
    let -- Find the part of the list before delim and put it in "before".
        -- The rest of the list, including the leading delim, goes
        -- in "remainder".
        (before, remainder) = span (/= delim) str
        in
        before : case remainder of
                      [] -> []
                      x -> -- If there is more data to process,
                           -- call split recursively to process it
                           split delim (tail x)
```

</div>

Let\'s look at this program. The heart of it is `findByUID`, which is a
simple function that parses the input one line at a time, then calls
`lookup` over the result. The remaining program is concerned with
parsing the input. The input file looks like this:

    root:x:0:0:root:/root:/bin/bash
    daemon:x:1:1:daemon:/usr/sbin:/bin/sh
    bin:x:2:2:bin:/bin:/bin/sh
    sys:x:3:3:sys:/dev:/bin/sh
    sync:x:4:65534:sync:/bin:/bin/sync
    games:x:5:60:games:/usr/games:/bin/sh
    man:x:6:12:man:/var/cache/man:/bin/sh
    lp:x:7:7:lp:/var/spool/lpd:/bin/sh
    mail:x:8:8:mail:/var/mail:/bin/sh
    news:x:9:9:news:/var/spool/news:/bin/sh
    jgoerzen:x:1000:1000:John Goerzen,,,:/home/jgoerzen:/bin/bash

Its fields are separated by colons, and include a username, numeric user
ID, numeric group ID, full name, home directory, and shell. No field may
contain an internal colon.

Maps
----

The `Data.Map` module provides a `Map` type with behavior that is
similar to association lists, but has much better performance.

Maps give us the same capabilities as hash tables do in other languages.
Internally, a map is implemented as a balanced binary tree. Compared to
a hash table, this is a much more efficient representation in a language
with immutable data. This is the most visible example of how deeply pure
functional programming affects how we write code: we choose data
structures and algorithms that we can express cleanly and that perform
efficiently, but our choices for specific tasks are often different
their counterparts in imperative languages.

Some functions in the `Data.Map` module have the same names as those in
the prelude. Therefore, we will import it with
`import qualified Data.Map as Map` and use `Map.name` to refer to names
in that module. Let\'s start our tour of `Data.Map` by taking a look at
some ways to build a map.

<div>

[buildmap.hs]{.label}

``` {.haskell}
import qualified Data.Map as Map

-- Functions to generate a Map that represents an association list
-- as a map

al = [(1, "one"), (2, "two"), (3, "three"), (4, "four")]

{- | Create a map representation of 'al' by converting the association
-  list using Map.fromList -}
mapFromAL =
    Map.fromList al

{- | Create a map representation of 'al' by doing a fold -}
mapFold =
    foldl (\map (k, v) -> Map.insert k v map) Map.empty al

{- | Manually create a map with the elements of 'al' in it -}
mapManual =
    Map.insert 2 "two" .
    Map.insert 4 "four" .
    Map.insert 1 "one" .
    Map.insert 3 "three" $ Map.empty
```

</div>

Functions like `Map.insert` work in the usual Haskell way: they return a
copy of the input data, with the requested change applied. This is quite
handy with maps. It means that you can use `foldl` to build up a map as
in the `mapFold` example. Or, you can chain together calls to
`Map.insert` as in the `mapManual` example. Let\'s use `ghci` to verify
that all of these maps are as expected:

``` {.screen}
ghci> :l buildmap.hs
[1 of 1] Compiling Main             ( buildmap.hs, interpreted )
Ok, one module loaded.
ghci> al
[(1,"one"),(2,"two"),(3,"three"),(4,"four")]
ghci> mapFromAL
fromList [(1,"one"),(2,"two"),(3,"three"),(4,"four")]
ghci> mapFold
fromList [(1,"one"),(2,"two"),(3,"three"),(4,"four")]
ghci> mapManual
fromList [(1,"one"),(2,"two"),(3,"three"),(4,"four")]
```

Notice that the output from `mapManual` differs from the order of the
list we used to construct the map. Maps do not guarantee that they will
preserve the original ordering.

Maps operate similarly in concept to association lists. The `Data.Map`
module provides functions for adding and removing data from maps. It
also lets us filter them, modify them, fold over them, and convert to
and from association lists. The library documentation for this module is
good, so instead of going into detail on each function, we will present
an example that ties together many of the concepts we\'ve discussed in
this chapter.

Functions Are Data, Too
-----------------------

Part of Haskell\'s power is the ease with which it lets us create and
manipulate functions. Let\'s take a look at a record that stores a
function as one of its fields:

<div>

[funcrecs.hs]{.label}

``` {.haskell}
{- | Our usual CustomColor type to play with -}
data CustomColor =
  CustomColor {red :: Int,
               green :: Int,
               blue :: Int}
  deriving (Eq, Show, Read)

{- | A new type that stores a name and a function.

The function takes an Int, applies some computation to it, and returns
an Int along with a CustomColor -}
data FuncRec =
    FuncRec {name :: String,
             colorCalc :: Int -> (CustomColor, Int)}

plus5func color x = (color, x + 5)

purple = CustomColor 255 0 255

plus5 = FuncRec {name = "plus5", colorCalc = plus5func purple}
always0 = FuncRec {name = "always0", colorCalc = \_ -> (purple, 0)}
```

</div>

Notice the type of the `colorCalc` field: it\'s a function. It takes an
`Int` and returns a tuple of `(CustomColor, Int)`. We create two
`FuncRec` records: `plus5` and `always0`. Notice that the `colorCalc`
for both of them will always return the color purple. `FuncRec` itself
has no field to store the color in, yet that value somehow becomes part
of the function itself. This is called a *closure*. Let\'s play with
this a bit:

``` {.screen}
ghci> :l funcrecs.hs
[1 of 1] Compiling Main             ( funcrecs.hs, interpreted )
Ok, one module loaded.
ghci> :t plus5
plus5 :: FuncRec
ghci> name plus5
"plus5"
ghci> :t colorCalc plus5
colorCalc plus5 :: Int -> (CustomColor, Int)
ghci> (colorCalc plus5) 7
(CustomColor {red = 255, green = 0, blue = 255},12)
ghci> :t colorCalc always0
colorCalc always0 :: Int -> (CustomColor, Int)
ghci> (colorCalc always0) 7
(CustomColor {red = 255, green = 0, blue = 255},0)
```

That worked well enough, but you might wonder how to do something more
advanced, such as making a piece of data available in multiple places. A
type construction function can be helpful. Here\'s an example:

<div>

[funcrecs2.hs]{.label}

``` {.haskell}
data FuncRec =
    FuncRec {name :: String,
             calc :: Int -> Int,
             namedCalc :: Int -> (String, Int)}

mkFuncRec :: String -> (Int -> Int) -> FuncRec
mkFuncRec name calcfunc =
    FuncRec {name = name,
             calc = calcfunc,
             namedCalc = \x -> (name, calcfunc x)}

plus5 = mkFuncRec "plus5" (+ 5)
always0 = mkFuncRec "always0" (\_ -> 0)
```

</div>

Here we have a function called `mkFuncRec` that takes a `String` and
another function as parameters, and returns a new `FuncRec` record.
Notice how both parameters to `mkFuncRec` are used in multiple places.
Let\'s try it out:

``` {.screen}
ghci> :l funcrecs2.hs
[1 of 1] Compiling Main             ( funcrecs2.hs, interpreted )
Ok, one module loaded.
ghci> :t plus5
plus5 :: FuncRec
ghci> name plus5
"plus5"
ghci> (calc plus5) 5
10
ghci> (namedCalc plus5) 5
("plus5",10)
ghci> plus5a = plus5 {name = "PLUS5A"}
ghci> name plus5a
"PLUS5A"
ghci> (namedCalc plus5a) 5
("plus5",10)
```

Notice the creation of `plus5a`. We changed the `name` field, but not
the `namedCalc` field. That\'s why `name` has the new name, but
`namedCalc` still returns the name that was passed to `mkFuncRec`; it
doesn\'t change unless we explicitly change it.

Extended Example: /etc/passwd
-----------------------------

In order to illustrate the usage of a number of different data
structures together, we\'ve prepared an extended example. This example
parses and stores entries from files in the format of a typical
`/etc/passwd` file.

<div>

[passwdmap.hs]{.label}

``` {.haskell}
import Data.List
import qualified Data.Map as Map
import System.IO
import Text.Printf(printf)
import System.Environment(getArgs)
import System.Exit
import Control.Monad(when)

{- | The primary piece of data this program will store.
   It represents the fields in a POSIX /etc/passwd file -}
data PasswdEntry = PasswdEntry {
    userName :: String,
    password :: String,
    uid :: Integer,
    gid :: Integer,
    gecos :: String,
    homeDir :: String,
    shell :: String}
    deriving (Eq, Ord)

{- | Define how we get data to a 'PasswdEntry'. -}
instance Show PasswdEntry where
    show pe = printf "%s:%s:%d:%d:%s:%s:%s"
                (userName pe) (password pe) (uid pe) (gid pe)
                (gecos pe) (homeDir pe) (shell pe)

{- | Converting data back out of a 'PasswdEntry'. -}
instance Read PasswdEntry where
    readsPrec _ value =
        case split ':' value of
             [f1, f2, f3, f4, f5, f6, f7] ->
                 -- Generate a 'PasswdEntry' the shorthand way:
                 -- using the positional fields.  We use 'read' to convert
                 -- the numeric fields to Integers.
                 [(PasswdEntry f1 f2 (read f3) (read f4) f5 f6 f7, [])]
             x -> error $ "Invalid number of fields in input: " ++ show x
        where
        {- | Takes a delimiter and a list.  Break up the list based on the
        -  delimiter. -}
        split :: Eq a => a -> [a] -> [[a]]

        -- If the input is empty, the result is a list of empty lists.
        split _ [] = [[]]
        split delim str =
            let -- Find the part of the list before delim and put it in
                -- "before".  The rest of the list, including the leading
                -- delim, goes in "remainder".
                (before, remainder) = span (/= delim) str
                in
                before : case remainder of
                              [] -> []
                              x -> -- If there is more data to process,
                                   -- call split recursively to process it
                                   split delim (tail x)

-- Convenience aliases; we'll have two maps: one from UID to entries
-- and the other from username to entries
type UIDMap = Map.Map Integer PasswdEntry
type UserMap = Map.Map String PasswdEntry

{- | Converts input data to maps.  Returns UID and User maps. -}
inputToMaps :: String -> (UIDMap, UserMap)
inputToMaps inp =
    (uidmap, usermap)
    where
    -- fromList converts a [(key, value)] list into a Map
    uidmap = Map.fromList . map (\pe -> (uid pe, pe)) $ entries
    usermap = Map.fromList .
              map (\pe -> (userName pe, pe)) $ entries
    -- Convert the input String to [PasswdEntry]
    entries = map read (lines inp)

main = do
    -- Load the command-line arguments
    args <- getArgs

    -- If we don't have the right number of args,
    -- give an error and abort

    when (length args /= 1) $ do
        putStrLn "Syntax: passwdmap filename"
        exitFailure

    -- Read the file lazily
    content <- readFile (head args)
    let maps = inputToMaps content
    mainMenu maps

mainMenu maps@(uidmap, usermap) = do
    putStr optionText
    hFlush stdout
    sel <- getLine
    -- See what they want to do.  For every option except 4,
    -- return them to the main menu afterwards by calling
    -- mainMenu recursively
    case sel of
         "1" -> lookupUserName >> mainMenu maps
         "2" -> lookupUID >> mainMenu maps
         "3" -> displayFile >> mainMenu maps
         "4" -> return ()
         _ -> putStrLn "Invalid selection" >> mainMenu maps

    where
    lookupUserName = do
        putStrLn "Username: "
        username <- getLine
        case Map.lookup username usermap of
             Nothing -> putStrLn "Not found."
             Just x -> print x
    lookupUID = do
        putStrLn "UID: "
        uidstring <- getLine
        case Map.lookup (read uidstring) uidmap of
             Nothing -> putStrLn "Not found."
             Just x -> print x
    displayFile =
        putStr . unlines . map (show . snd) . Map.toList $ uidmap
    optionText =
          "\npasswdmap options:\n\
           \\n\
           \1   Look up a user name\n\
           \2   Look up a UID\n\
           \3   Display entire file\n\
           \4   Quit\n\n\
           \Your selection: "
```

</div>

This example maintains two maps: one from username to `PasswdEntry` and
another one from UID to `PasswdEntry`. Database developers may find it
convenient to think of this as having two different indices into the
data to speed searching on different fields.

Take a look at the `Show` and `Read` instances for `PasswdEntry`. There
is already a standard format for rendering data of this type as a
string: the colon-separated version already used by the system. So our
`Show` function displays a `PasswdEntry` in the format, and `Read`
parses that format.

Extended example: Numeric Types
-------------------------------

We\'ve told you how powerful and expressive Haskell\'s type system is.
We\'ve shown you a lot of ways to use that power. Here\'s a chance to
really see that in action.

Back in [the section called \"Numeric
Types\"](6-using-typeclasses.org::*Numeric Types) type classes that come
with Haskell. Let\'s see what we can do by defining new types and
utilizing the numeric type classes to integrate them with basic
mathematics in Haskell.

Let\'s start by thinking through what we\'d like to see out of `ghci`
when we interact with our new types. To start with, it might be nice to
render numeric expressions as strings, making sure to indicate proper
precedence. Perhaps we could create a function called `prettyShow` to do
that. We\'ll show you how to write it in a bit, but first we\'ll look at
how we might use it.

``` {.screen}
ghci> :l num.hs
[1 of 1] Compiling Main             ( num.hs, interpreted )
Ok, one module loaded.
ghci> 5 + 1 * 3
8
ghci> prettyShow $ 5 + 1 * 3
"5+(1*3)"
ghci> prettyShow $ 5 * 1 + 3
"(5*1)+3"
```

That looks nice, but it wasn\'t all that smart. We could easily simplify
out the `1 *` part of the expression. How about a function to do some
very basic simplification?

``` {.screen}
ghci> prettyShow $ simplify $ 5 + 1 * 3
"5+3"
```

How about converting a numeric expression to Reverse Polish Notation
(RPN)? RPN is a postfix notation that never requires parentheses, and is
commonly found on HP calculators. RPN is a stack-based notation. We push
numbers onto the stack, and when we enter operations, they pop the most
recent numbers off the stack and place the result on the stack.

``` {.screen}
ghci> rpnShow $ 5 + 1 * 3
"5 1 3 * +"
ghci> rpnShow $ simplify $ 5 + 1 * 3
"5 3 +"
```

Maybe it would be nice to be able to represent simple expressions with
symbols for the unknowns.

``` {.screen}
ghci> prettyShow $ 5 + (Symbol "x") * 3
"5+(x*3)"
```

It\'s often important to track units of measure when working with
numbers. For instance, when you see the number 5, does it mean 5 meters,
5 feet, or 5 bytes? Of course, if you divide 5 meters by 2 seconds, the
system ought to be able to figure out the appropriate units. Moreover,
it should stop you from adding 2 seconds to 5 meters.

``` {.screen}
ghci> 5 / 2
2.5
ghci> (units 5 "m") / (units 2 "s")
2.5_m/s
ghci> (units 5 "m") + (units 2 "s")
*** Exception: Mis-matched units in add or subtract
CallStack (from HasCallStack):
  error, called at num.hs:109:19 in main:Main
ghci> (units 5 "m") + (units 2 "m")
7_m
ghci> (units 5 "m") / 2
2.5_m
ghci> 10 * (units 5 "m") / (units 2 "s")
25.0_m/s
```

If we define an expression or a function that is valid for all numbers,
we should be able to calculate the result, or render the expression. For
instance, if we define `test` to have type `Num a => a`, and say
`test = 2 * 5 + 3`, then we ought to be able to do this:

``` {.screen}
ghci> test
13
ghci> rpnShow test
"2 5 * 3 +"
ghci> prettyShow test
"(2*5)+3"
ghci> test + 5
18
ghci> prettyShow (test + 5)
"((2*5)+3)+5"
ghci> rpnShow (test + 5)
"2 5 * 3 + 5 +"
```

Since we have units, we should be able to handle some basic trigonometry
as well. Many of these operations operate on angles. Let\'s make sure
that we can handle both degrees and radians.

``` {.screen}
ghci> sin (pi / 2)
1.0
ghci> sin (units (pi / 2) "rad")
1.0_1.0
ghci> sin (units 90 "deg")
1.0_1.0
ghci> (units 50 "m") * sin (units 90 "deg")
50.0_m
```

Finally, we ought to be able to put all this together and combine
different kinds of expressions together.

``` {.screen}
ghci> ((units 50 "m") * sin (units 90 "deg")) :: Units (SymbolicManip Double)
50.0*sin(((2.0*pi)*90.0)/360.0)_m
ghci> prettyShow $ dropUnits $ (units 50 "m") * sin (units 90 "deg")
"50.0*sin(((2.0*pi)*90.0)/360.0)"
ghci> rpnShow $ dropUnits $ (units 50 "m") * sin (units 90 "deg")
"50.0 2.0 pi * 90.0 * 360.0 / sin *"
ghci> (units (Symbol "x") "m") * sin (units 90 "deg")
x*sin(((2.0*pi)*90.0)/360.0)_m
```

Everything you\'ve just seen is possible with Haskell types and classes.
In fact, you\'ve been reading a real `ghci` session demonstrating
`num.hs`, which you\'ll see shortly.

### First Steps

Let\'s think about how we would accomplish everything shown above. To
start with, we might use `ghci` to check the type of `(+)`, which is
`Num a => a -> a -> a`. If we want to make possible some custom behavior
for the plus operator, then we will have to define a new type and make
it an instance of `Num`. This type will need to store an expression
symbolically. We can start by thinking of operations such as addition.
To store that, we will need to store the operation itself, its left
side, and its right side. The left and right sides could themselves be
expressions.

We can therefore think of an expression as a sort of tree. Let\'s start
with some simple types.

<div>

[numsimple.hs]{.label}

``` {.haskell}
-- The "operators" that we're going to support
data Op = Plus | Minus | Mul | Div | Pow
        deriving (Eq, Show)

{- The core symbolic manipulation type -}
data SymbolicManip a =
          Number a           -- Simple number, such as 5
        | Arith Op (SymbolicManip a) (SymbolicManip a)
          deriving (Eq, Show)

{- SymbolicManip will be an instance of Num.  Define how the Num
operations are handled over a SymbolicManip.  This will implement things
like (+) for SymbolicManip. -}
instance Num a => Num (SymbolicManip a) where
    a + b = Arith Plus a b
    a - b = Arith Minus a b
    a * b = Arith Mul a b
    negate a = Arith Mul (Number (-1)) a
    abs a = error "abs is unimplemented"
    signum _ = error "signum is unimplemented"
    fromInteger i = Number (fromInteger i)
```

</div>

First, we define a type called `Op`. This type simply represents some of
the operations we will support. Next, there is a definition for
`SymbolicManip a`. Because of the `Num a` constraint, any `Num` can be
used for the `a`. So a full type may be something like
`SymbolicManip Int`.

A `SymbolicManip` type can be a plain number, or it can be some
arithmetic operation. The type for the `Arith` constructor is recursive,
which is perfectly legal in Haskell. `Arith` creates a `SymbolicManip`
out of an `Op` and two other `SymbolicManip` items. Let\'s look at an
example:

``` {.screen}
ghci> :l numsimple.hs
[1 of 1] Compiling Main             ( numsimple.hs, interpreted )
Ok, modules loaded: Main.
ghci> Number 5
Number 5
ghci> :t Number 5
Number 5 :: Num a => SymbolicManip a
ghci> :t Number (5::Int)
Number (5::Int) :: SymbolicManip Int
ghci> Number 5 * Number 10
Arith Mul (Number 5) (Number 10)
ghci> (5 * 10)::SymbolicManip Int
Arith Mul (Number 5) (Number 10)
ghci> (5 * 10 + 2)::SymbolicManip Int
Arith Plus (Arith Mul (Number 5) (Number 10)) (Number 2)
```

You can see that we already have a very basic representation of
expressions working. Notice how Haskell \"converted\" `5 * 10 + 2` into
a `SymbolicManip`, and even handled order of evaluation properly. This
wasn\'t really a true conversion; `SymbolicManip` is a first-class
number now. Integer numeric literals are internally treated as being
wrapped in `fromInteger` anyway, so `5` is just as valid as a
`SymbolicManip Int` as it as an `Int`.

From here, then, our task is simple: extend the `SymbolicManip` type to
be able to represent all the operations we will want to perform,
implement instances of it for the other numeric type classes, and
implement our own instance of `Show` for `SymbolicManip` that renders
this tree in a more accessible fashion.

### Completed Code

Here is the completed `num.hs`, which was used with the `ghci` examples
at the beginning of this chapter. Let\'s look at this code one piece at
a time.

<div>

[num.hs]{.label}

``` {.haskell}
import Data.List

--------------------------------------------------
-- Symbolic/units manipulation
--------------------------------------------------

-- The "operators" that we're going to support
data Op = Plus | Minus | Mul | Div | Pow
        deriving (Eq, Show)

{- The core symbolic manipulation type.  It can be a simple number,
a symbol, a binary arithmetic operation (such as +), or a unary
arithmetic operation (such as cos)

Notice the types of BinaryArith and UnaryArith: it's a recursive
type.  So, we could represent a (+) over two SymbolicManips. -}
data SymbolicManip a =
          Number a           -- Simple number, such as 5
        | Symbol String      -- A symbol, such as x
        | BinaryArith Op (SymbolicManip a) (SymbolicManip a)
        | UnaryArith String (SymbolicManip a)
          deriving (Eq)
```

</div>

In this section of code, we define an `Op` that is identical to the one
we used before. We also define `SymbolicManip`, which is similar to what
we used before. In this version, we now support unary arithmetic
operations (those which take only one parameter) such as `abs` or `cos`.
Next we define our instance of `Num`.

<div>

[num.hs]{.label}

``` {.haskell}
{- SymbolicManip will be an instance of Num.  Define how the Num
operations are handled over a SymbolicManip.  This will implement things
like (+) for SymbolicManip. -}
instance Num a => Num (SymbolicManip a) where
    a + b = BinaryArith Plus a b
    a - b = BinaryArith Minus a b
    a * b = BinaryArith Mul a b
    negate a = BinaryArith Mul (Number (-1)) a
    abs a = UnaryArith "abs" a
    signum _ = error "signum is unimplemented"
    fromInteger i = Number (fromInteger i)
```

</div>

This is pretty straightforward and also similar to our earlier code.
Note that earlier we weren\'t able to properly support `abs`, but now
with the `UnaryArith` constructor, we can. Next we define some more
instances.

<div>

[num.hs]{.label}

``` {.haskell}
{- Make SymbolicManip an instance of Fractional -}
instance (Fractional a) => Fractional (SymbolicManip a) where
    a / b = BinaryArith Div a b
    recip a = BinaryArith Div (Number 1) a
    fromRational r = Number (fromRational r)

{- Make SymbolicManip an instance of Floating -}
instance (Floating a) => Floating (SymbolicManip a) where
    pi = Symbol "pi"
    exp a = UnaryArith "exp" a
    log a = UnaryArith "log" a
    sqrt a = UnaryArith "sqrt" a
    a ** b = BinaryArith Pow a b
    sin a = UnaryArith "sin" a
    cos a = UnaryArith "cos" a
    tan a = UnaryArith "tan" a
    asin a = UnaryArith "asin" a
    acos a = UnaryArith "acos" a
    atan a = UnaryArith "atan" a
    sinh a = UnaryArith "sinh" a
    cosh a = UnaryArith "cosh" a
    tanh a = UnaryArith "tanh" a
    asinh a = UnaryArith "asinh" a
    acosh a = UnaryArith "acosh" a
    atanh a = UnaryArith "atanh" a
```

</div>

This section of code defines some fairly straightforward instances of
`Fractional` and `Floating`. Now let\'s work on converting our
expressions to strings for display.

<div>

[num.hs]{.label}

``` {.haskell}
{- Show a SymbolicManip as a String, using conventional
algebraic notation -}
prettyShow :: (Show a, Num a) => SymbolicManip a -> String

-- Show a number or symbol as a bare number or serial
prettyShow (Number x) = show x
prettyShow (Symbol x) = x

prettyShow (BinaryArith op a b) =
    let pa = simpleParen a
        pb = simpleParen b
        pop = op2str op
        in pa ++ pop ++ pb
prettyShow (UnaryArith opstr a) =
    opstr ++ "(" ++ show a ++ ")"

op2str :: Op -> String
op2str Plus = "+"
op2str Minus = "-"
op2str Mul = "*"
op2str Div = "/"
op2str Pow = "**"

{- Add parenthesis where needed.  This function is fairly conservative
and will add parenthesis when not needed in some cases.

Haskell will have already figured out precedence for us while building
up the SymbolicManip. -}
simpleParen :: (Show a, Num a) => SymbolicManip a -> String
simpleParen (Number x) = prettyShow (Number x)
simpleParen (Symbol x) = prettyShow (Symbol x)
simpleParen x@(BinaryArith _ _ _) = "(" ++ prettyShow x ++ ")"
simpleParen x@(UnaryArith _ _) = prettyShow x

{- Showing a SymbolicManip calls the prettyShow function on it -}
instance (Show a, Num a) => Show (SymbolicManip a) where
    show a = prettyShow a
```

</div>

We start by defining a function `prettyShow`. It renders an expression
using conventional style. The algorithm is fairly simple: bare numbers
and symbols are rendered bare; binary arithmetic is rendered with the
two sides plus the operator in the middle, and of course we handle the
unary operators as well. `op2str` simply converts an `Op` to a `String`.
In `simpleParen`, we have a quite conservative algorithm that adds
parenthesis to keep precedence clear in the result. Finally, we make
`SymbolicManip` an instance of `Show` and use `prettyShow` to accomplish
that. Now let\'s implement an algorithm that converts an expression to s
string in RPN format.

<div>

[num.hs]{.label}

``` {.haskell}
{- Show a SymbolicManip using RPN.  HP calculator users may
find this familiar. -}
rpnShow :: (Show a, Num a) => SymbolicManip a -> String
rpnShow i =
    let toList (Number x) = [show x]
        toList (Symbol x) = [x]
        toList (BinaryArith op a b) = toList a ++ toList b ++
           [op2str op]
        toList (UnaryArith op a) = toList a ++ [op]
        join :: [a] -> [[a]] -> [a]
        join delim l = concat (intersperse delim l)
    in join " " (toList i)
```

</div>

Fans of RPN will note how much simpler this algorithm is compared to the
algorithm to render with conventional notation. In particular, we
didn\'t have to worry about where to add parenthesis, because RPN can,
by definition, only be evaluated one way. Next, let\'s see how we might
implement a function to do some rudimentary simplification on
expressions.

<div>

[num.hs]{.label}

``` {.haskell}
{- Perform some basic algebraic simplifications on a SymbolicManip. -}
simplify :: (Num a, Eq a) => SymbolicManip a -> SymbolicManip a
simplify (BinaryArith op ia ib) =
    let sa = simplify ia
        sb = simplify ib
        in
        case (op, sa, sb) of
                (Mul, Number 1, b) -> b
                (Mul, a, Number 1) -> a
                (Mul, Number 0, b) -> Number 0
                (Mul, a, Number 0) -> Number 0
                (Div, a, Number 1) -> a
                (Plus, a, Number 0) -> a
                (Plus, Number 0, b) -> b
                (Minus, a, Number 0) -> a
                _ -> BinaryArith op sa sb
simplify (UnaryArith op a) = UnaryArith op (simplify a)
simplify x = x
```

</div>

This function is pretty simple. For certain binary arithmetic
operations---for instance, multiplying any value by 1---we are able to
easily simplify the situation. We begin by obtaining simplified versions
of both sides of the calculation (this is where recursion hits) and then
simplify the result. We have little to do with unary operators, so we
just simplify the expression they act upon.

From here on, we will add support for units of measure to our
established library. This will let us represent quantities such as \"5
meters\". We start, as before, by defining a type.

<div>

[num.hs]{.label}

``` {.haskell}
{- New data type: Units.  A Units type contains a number
and a SymbolicManip, which represents the units of measure.
A simple label would be something like (Symbol "m") -}
data Units a = Units a (SymbolicManip a)
           deriving (Eq)
```

</div>

So, a `Units` contains a number and a label. The label is itself a
`SymbolicManip`. Next, it will probably come as no surprise to see an
instance of `Num` for `Units`.

<div>

[num.hs]{.label}

``` {.haskell}
{- Implement Units for Num.  We don't know how to convert between
arbitrary units, so we generate an error if we try to add numbers with
different units.  For multiplication, generate the appropriate
new units. -}
instance (Num a, Eq a) => Num (Units a) where
    (Units xa ua) + (Units xb ub)
        | ua == ub = Units (xa + xb) ua
        | otherwise = error "Mis-matched units in add or subtract"
    (Units xa ua) - (Units xb ub) = (Units xa ua) + (Units (xb * (-1)) ub)
    (Units xa ua) * (Units xb ub) = Units (xa * xb) (ua * ub)
    negate (Units xa ua) = Units (negate xa) ua
    abs (Units xa ua) = Units (abs xa) ua
    signum (Units xa _) = Units (signum xa) (Number 1)
    fromInteger i = Units (fromInteger i) (Number 1)
```

</div>

Now it may become clear why we use a `SymbolicManip` instead of a
`String` to store the unit of measure. As calculations such as
multiplication occur, the unit of measure also changes. For instance, if
we multiply 5 meters by 2 meters, we obtain 10 square meters. We force
the units for addition to match, and implement subtraction in terms of
addition. Let\'s look at more type class instances for `Units`.

<div>

[num.hs]{.label}

``` {.haskell}
{- Make Units an instance of Fractional -}
instance (Fractional a, Eq a) => Fractional (Units a) where
    (Units xa ua) / (Units xb ub) = Units (xa / xb) (ua / ub)
    recip a = 1 / a
    fromRational r = Units (fromRational r) (Number 1)

{- Floating implementation for Units.

Use some intelligence for angle calculations: support deg and rad
-}
instance (Floating a, Eq a) => Floating (Units a) where
    pi = (Units pi (Number 1))
    exp _ = error "exp not yet implemented in Units"
    log _ = error "log not yet implemented in Units"
    (Units xa ua) ** (Units xb ub)
        | ub == Number 1 = Units (xa ** xb) (ua ** Number xb)
        | otherwise = error "units for RHS of ** not supported"
    sqrt (Units xa ua) = Units (sqrt xa) (sqrt ua)
    sin (Units xa ua)
        | ua == Symbol "rad" = Units (sin xa) (Number 1)
        | ua == Symbol "deg" = Units (sin (deg2rad xa)) (Number 1)
        | otherwise = error "Units for sin must be deg or rad"
    cos (Units xa ua)
        | ua == Symbol "rad" = Units (cos xa) (Number 1)
        | ua == Symbol "deg" = Units (cos (deg2rad xa)) (Number 1)
        | otherwise = error "Units for cos must be deg or rad"
    tan (Units xa ua)
        | ua == Symbol "rad" = Units (tan xa) (Number 1)
        | ua == Symbol "deg" = Units (tan (deg2rad xa)) (Number 1)
        | otherwise = error "Units for tan must be deg or rad"
    asin (Units xa ua)
        | ua == Number 1 = Units (rad2deg $ asin xa) (Symbol "deg")
        | otherwise = error "Units for asin must be empty"
    acos (Units xa ua)
        | ua == Number 1 = Units (rad2deg $ acos xa) (Symbol "deg")
        | otherwise = error "Units for acos must be empty"
    atan (Units xa ua)
        | ua == Number 1 = Units (rad2deg $ atan xa) (Symbol "deg")
        | otherwise = error "Units for atan must be empty"
    sinh = error "sinh not yet implemented in Units"
    cosh = error "cosh not yet implemented in Units"
    tanh = error "tanh not yet implemented in Units"
    asinh = error "asinh not yet implemented in Units"
    acosh = error "acosh not yet implemented in Units"
    atanh = error "atanh not yet implemented in Units"
```

</div>

We didn\'t supply implementations for every function, but quite a few
have been defined. Now let\'s define a few utility functions for working
with units.

<div>

[num.hs]{.label}

``` {.haskell}
{- A simple function that takes a number and a String and returns an
appropriate Units type to represent the number and its unit of measure -}
units :: (Num z) => z -> String -> Units z
units a b = Units a (Symbol b)

{- Extract the number only out of a Units type -}
dropUnits :: (Num z) => Units z -> z
dropUnits (Units x _) = x

{- Utilities for the Unit implementation -}
deg2rad x = 2 * pi * x / 360
rad2deg x = 360 * x / (2 * pi)
```

</div>

First, we have `units`, which makes it easy to craft simple expressions.
It\'s faster to say `units 5 "m"` than `Units 5 (Symbol "m")`. We also
have a corresponding `dropUnits`, which discards the unit of measure and
returns the embedded bare `Num`. Finally, we define some functions for
use by our earlier instances to convert between degrees and radians.
Next, we just define a `Show` instance for `Units`.

<div>

[num.hs]{.label}

``` {.haskell}
{- Showing units: we show the numeric component, an underscore,
then the prettyShow version of the simplified units -}
instance (Show a, Num a, Eq a) => Show (Units a) where
    show (Units xa ua) = show xa ++ "_" ++ prettyShow (simplify ua)
```

</div>

That was simple. For one last piece, we define a variable `test` to
experiment with.

<div>

[num.hs]{.label}

``` {.haskell}
test :: (Num a) => a
test = 2 * 5 + 3
```

</div>

So, looking back over all this code, we have done what we set out to
accomplish: implemented more instances for `SymbolicManip`. We have also
introduced another type called `Units` which stores a number and a unit
of measure. We implement several show-like functions which render the
`SymbolicManip` or `Units` in different ways.

There is one other point that this example drives home. Every
language---even those with objects and overloading---has some parts of
the language that are special in some way. In Haskell, the \"special\"
bits are extremely small. We have just developed a new representation
for something as fundamental as a number, and it has been really quite
easy. Our new type is a first-class type, and the compiler knows what
functions to use with it at compile time. Haskell takes code reuse and
interchangability to the extreme. It is easy to make code generic and
work on things of many different types. It\'s also easy to make up new
types and make them automatically be first-class features of the system.

Remember our `ghci` examples at the beginning of the chapter? All of
them were made with the code in this example. You might want to try them
out for yourself and see how they work.

### Exercises

1.  Extend the `prettyShow` function to remove unnecessary parentheses.

Taking advantage of functions as data
-------------------------------------

In an imperative language, appending two lists is cheap and easy.
Here\'s a simple C structure in which we maintain a pointer to the head
and tail of a list.

``` {.c org-language="C"}
struct list {
    struct node *head, *tail;
};
```

When we have one list, and want to append another list onto its end, we
modify the last node of the existing list to point to its `head` node,
then update its `tail` pointer to point to its `tail` node.

Obviously, this approach is off limits to us in Haskell if we want to
stay pure. Since pure data is immutable, we can\'t go around modifying
lists in place. Haskell\'s `(++)` operator appends two lists by creating
a new one.

<div>

[Append.hs]{.label}

``` {.haskell}
(++) :: [a] -> [a] -> [a]
(x:xs) ++ ys = x : xs ++ ys
_      ++ ys = ys
```

</div>

From inspecting the code, we can see that the cost of creating a new
list depends on the length of the initial list[^2].

We often need to append lists over and over, to construct one big list.
For instance, we might be generating the contents of a web page as a
`String`, emitting a chunk at a time as we traverse some data structure.
Each time we have a chunk of markup to add to the page, we will
naturally want to append it onto the end of our existing `String`.

If a single append has a cost proportional to the length of the initial
list, and each repeated append makes the initial list longer, we end up
in an unhappy situation: the cost of all of the repeated appends is
proportional to the *square* of the length of the final list.

To understand this, let\'s dig in a little. The `(++)` operator is right
associative.

``` {.screen}
ghci> :info (++)
(++) :: [a] -> [a] -> [a]     -- Defined in GHC.Base
infixr 5 ++
```

This means that a Haskell implementation will evaluate the expression
`"a" ++ "b" ++ "c"` as if we had put parentheses around it as follows:
`"a" ++ ("b" ++ "c")`. This makes good performance sense, because it
keeps the left operand as short as possible.

When we repeatedly append onto the end of a list, we defeat this
associativity. Let\'s say we start with the list `"a"` and append `"b"`,
and save the result as our new list. If we later append `"c"` onto this
new list, our left operand is now `"ab"`. In this scheme, every time we
append, our left operand gets longer.

Meanwhile, the imperative programmers are cackling with glee, because
the cost of *their* repeated appends only depends on the number of them
that they perform. They have linear performance; ours is quadratic.

When something as common as repeated appending of lists imposes such a
performance penalty, it\'s time to look at the problem from another
angle.

The expression `("a"++)` is a section, a partially applied function.
What is its type?

``` {.screen}
ghci> :type ("a" ++)
("a" ++) :: [Char] -> [Char]
```

Since this is a function, we can use the `(.)` operator to compose it
with another section, let\'s say `("b"++)`.

``` {.screen}
ghci> :type ("a" ++) . ("b" ++)
("a" ++) . ("b" ++) :: [Char] -> [Char]
```

Our new function has the same type. What happens if we stop composing
functions, and instead provide a `String` to the function we\'ve
created?

``` {.screen}
ghci> f = ("a" ++) . ("b" ++)
ghci> f []
"ab"
```

We\'ve appended the strings! We\'re using these partially applied
functions to store data, which we can retrieve by providing an empty
list. Each partial application of `(++)` and `(.)` *represents* an
append, but it doesn\'t actually *perform* the append.

There are two very interesting things about this approach. The first is
that the cost of a partial application is constant, so the cost of many
partial applications is linear. The second is that when we finally
provide a `[]` value to unlock the final list from its chain of partial
applications, application proceeds from right to left. This keeps the
left operand of `(++)` small, and so the overall cost of all of these
appends is linear, not quadratic.

By choosing an unfamiliar data representation, we\'ve avoided a nasty
performance quagmire, while gaining a new perspective on the usefulness
of treating functions as data. By the way, this is an old trick, and
it\'s usually called a *difference list*.

We\'re not yet finished, though. As appealing as difference lists are in
theory, ours won\'t be very pleasant in practice if we leave all the
plumbing of `(++)`, `(.)`, and partial application exposed. We need to
turn this mess into something pleasant to work with.

### Turning difference lists into a proper library

Our first step is to use a `newtype` declaration to hide the underlying
type from our users. We\'ll create a new type, and call it `DList`. Like
a regular list, it will be a parameterised type.

<div>

[DList.hs]{.label}

``` {.haskell}
newtype DList a = DL {
    unDL :: [a] -> [a]
}
```

</div>

The `unDL` function is our deconstructor, which removes the `DL`
constructor. When we go back and decide what we want to export from our
module, we will omit our data constructor and deconstruction function,
so the `DList` type will be completely opaque to our users. They\'ll
only be able to work with the type using the other functions we export.

<div>

[DList.hs]{.label}

``` {.haskell}
append :: DList a -> DList a -> DList a
append xs ys = DL (unDL xs . unDL ys)
```

</div>

Our `append` function may seem a little complicated, but it\'s just
performing some book-keeping around the same use of the `(.)` operator
that we demonstrated earlier. To compose our functions, we must first
unwrap them from their `DL` constructor, hence the uses of `unDL`. We
then re-wrap the resulting function with the `DL` constructor so that it
will have the right type.

Here\'s another way of writing the same function, in which we perform
the unwrapping of `xs` and `ys` via pattern matching.

<div>

[DList.hs]{.label}

``` {.haskell}
append' :: DList a -> DList a -> DList a
append' (DL xs) (DL ys) = DL (xs . ys)
```

</div>

Our `DList` type won\'t be much use if we can\'t convert back and forth
between the `DList` representation and a regular list.

<div>

[DList.hs]{.label}

``` {.haskell}
fromList :: [a] -> DList a
fromList xs = DL (xs ++)

toList :: DList a -> [a]
toList (DL xs) = xs []
```

</div>

Once again, compared to the original versions of these functions that we
wrote, all we\'re doing is a little book-keeping to hide the plumbing.

If we want to make `DList` useful as a substitute for regular lists, we
need to provide some more of the common list operations.

<div>

[DList.hs]{.label}

``` {.haskell}
empty :: DList a
empty = DL id

-- equivalent of the list type's (:) operator
cons :: a -> DList a -> DList a
cons x (DL xs) = DL ((x:) . xs)
infixr `cons`

dfoldr :: (a -> b -> b) -> b -> DList a -> b
dfoldr f z xs = foldr f z (toList xs)
```

</div>

Although the `DList` approach makes appends cheap, not all list-like
operations are easily available. The `head` function has constant cost
for lists. Our `DList` equivalent requires that we convert the entire
`DList` to a regular list, so it is much more expensive than its list
counterpart: its cost is linear in the number of appends we have
performed to construct the DList.

<div>

[DList.hs]{.label}

``` {.haskell}
safeHead :: DList a -> Maybe a
safeHead xs = case toList xs of
                (y:_) -> Just y
                _ -> Nothing
```

</div>

To support an equivalent of `map`, we can make our `DList` type a
functor.

<div>

[DList.hs]{.label}

``` {.haskell}
dmap :: (a -> b) -> DList a -> DList b
dmap f = dfoldr go empty
    where go x xs = cons (f x) xs

instance Functor DList where
    fmap = dmap
```

</div>

Once we decide that we have written enough equivalents of list
functions, we go back to the top of our source file, and add a module
header.

<div>

[DList.hs]{.label}

``` {.haskell}
module DList
    (
      DList
    , fromList
    , toList
    , empty
    , append
    , cons
    , dfoldr
    ) where
```

</div>

### Lists, difference lists, and monoids

In abstract algebra, there exists a simple abstract structure called a
*monoid*. Many mathematical objects are monoids, because the \"bar to
entry\" is very low. In order to be considered a monoid, an object must
have two properties.

-   An associative binary operator. Let\'s call it `(*)`: the expression
    `a * (b * c)` must give the same result as `(a * b) * c`.

-   An identity value. If we call this `e`, it must obey two rules:
    `a * e == a` and `e * a == a`.

The rules for monoids don\'t say what the binary operator must do,
merely that such an operator must exist. Because of this, lots of
mathematical objects are monoids. If we take addition as the binary
operator and zero as the identity value, integers form a monoid. With
multiplication as the binary operator and one as the identity value,
integers form a different monoid.

Monoids are ubiquitous in Haskell[^3]. The `Monoid` type class is
defined in the `Data.Monoid` module.

<div>

[Monoid.hs]{.label}

``` {.haskell}
class Monoid a where
    mempty  :: a                -- the identity
    mappend :: a -> a -> a      -- associative binary operator
```

</div>

If we take `(++)` as the binary operator and `[]` as the identity, lists
form a monoid.

<div>

[Monoid.hs]{.label}

``` {.haskell}
instance Monoid [a] where
    mempty  = []
    mappend = (++)
```

</div>

Since lists and `DLists` are so closely related, it follows that our
`DList` type must be a monoid, too.

<div>

[DList.hs]{.label}

``` {.haskell}
instance Monoid (DList a) where
    mempty = empty
    mappend = append
```

</div>

When working with a GHC version prior to 8.4 then that should be enough
and the module is ready to be load in `ghci`. When working with GHC 8.4
or later then you must make `DList` an instance of `Semigroup` too.

<div>

[DList.hs]{.label}

``` {.haskell}
instance Semigroup (DList a) where
    (<>) = append

instance Monoid (DList a) where
    mempty = empty
```

</div>

A semigroup is a mathematical object with and associative binary
operator, i. e. every monoid is also a semigroup (but not al semigroups
are monoids) and that\'s what GHC 8.4 is forcing us to express.

::: {.TIP}
Tip

If we need compatibility with GHC prior to version 8.4 we can [write
compatible
code](https://prime.haskell.org/wiki/Libraries/Proposals/SemigroupMonoid#Writingcompatiblecode).
:::

Let\'s try our the methods of the `Monoid` type class in `ghci`.

``` {.screen}
ghci> "foo" `mappend` "bar"
"foobar"
ghci> toList (fromList [1,2] `mappend` fromList [3,4])
[1,2,3,4]
ghci> mempty `mappend` [1]
[1]
```

::: {.TIP}
Tip

Although from a mathematical perspective, integers can be monoids in two
different ways, we can\'t write two differing `Monoid` instances for
`Int` in Haskell: the compiler would complain about duplicate instances.

In those rare cases where we really need several `Monoid` instances for
the same type, we can use some `newtype` trickery to create distinct
types for the purpose.

<div>

[Monoid.hs]{.label}

``` {.haskell}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}

newtype AInt = A { unA :: Int }
    deriving (Show, Eq, Num)

-- monoid under addition
instance Semigroup AInt where
    (<>) = (+)

instance Monoid AInt where
    mempty = 0

newtype MInt = M { unM :: Int }
    deriving (Show, Eq, Num)

-- monoid under multiplication
instance Semigroup MInt where
    (<>) = (*)

instance Monoid MInt where
    mempty = 1
```

</div>

We\'ll then get different behaviour depending on the type we use.

``` {.screen}
ghci> 2 `mappend` 5 :: MInt
M {unM = 10}
ghci> 2 `mappend` 5 :: AInt
A {unA = 7}
```
:::

We will have more to say about difference lists and their monoidal
nature in [the section called \"The writer monad and
lists\"](16-programming-with-monads.org::*The writer monad and lists)

::: {.TIP}
Tip

As with the rules for functors, Haskell cannot check the rules for
monoids on our behalf. If we\'re defining a `Monoid` instance, we can
easily write QuickCheck properties to give us high statistical
confidence that our code is following the monoid rules.
:::

General purpose sequences
=========================

Both Haskell\'s built-in list type and the `DList` type that we defined
above have poor performance characteristics under some circumstances.
The `Data.Sequence` module defines a `Seq` container type that gives
good performance for a wider variety of operations.

As with other modules, `Data.Sequence` is intended to be used via
qualified import.

<div>

[DataSequence.hs]{.label}

``` {.haskell}
import qualified Data.Sequence as Seq
```

</div>

We can construct an empty `Seq` using `empty`, and a single-element
container using `singleton`.

``` {.screen}
ghci> :l DataSequence.hs
[1 of 1] Compiling Main             ( DataSequence.hs, interpreted )
Ok, one module loaded.
ghci> Seq.empty
fromList []
ghci> Seq.singleton 1
fromList [1]
```

We can create a `Seq` from a list using `fromList`.

``` {.screen}
ghci> a = Seq.fromList [1,2,3]
```

The `Data.Sequence` module provides some constructor functions in the
form of operators. When we perform a qualified import, we must qualify
the name of an operator in our code, which is ugly.

``` {.screen}
ghci> 1 Seq.<| Seq.singleton 2
fromList [1,2]
```

If we import the operators explicitly, we can avoid the need to qualify
them.

<div>

[DataSequence.hs]{.label}

``` {.haskell}
import Data.Sequence ((><), (<|), (|>))
```

</div>

By removing the qualification from the operator, we improve the
readability of our code.

``` {.screen}
ghci> Seq.singleton 1 |> 2
fromList [1,2]
```

A useful way to remember the `(<|)` and `(|>)` functions is that the
\"arrow\" points to the element we\'re adding to the `Seq`. The element
will be added on the side to which the arrow points: `(<|)` adds on the
left, `(|>)` on the right.

Both adding on the left and adding on the right are constant-time
operations. Appending two \~Seq\~s is also cheap, occurring in time
proportional to the logarithm of whichever is shorter. To append, we use
the `(><)` operator.

``` {.screen}
ghci> left = Seq.fromList [1,3,3]
ghci> right = Seq.fromList [7,1]
ghci> left >< right
fromList [1,3,3,7,1]
```

If we want to create a list from a `Seq`, we must use the
`Data.Foldable` module, which is best imported qualified.

<div>

[DataSequence.hs]{.label}

``` {.haskell}
import qualified Data.Foldable as Foldable
```

</div>

This module defines a type class, `Foldable`, which `Seq` implements.

``` {.screen}
ghci> Foldable.toList (Seq.fromList [1,2,3])
[1,2,3]
```

If we want to fold over a `Seq`, we use the fold functions from the
`Data.Foldable` module.

``` {.screen}
ghci> Foldable.foldl' (+) 0 (Seq.fromList [1,2,3])
6
```

The `Data.Sequence` module provides a number of other useful list-like
functions. Its documentation is very thorough, giving time bounds for
each operation.

If `Seq` has so many desirable characteristics, why is it not the
default sequence type? Lists are simpler and have less overhead, and so
quite often they are good enough for the task at hand. They are also
well suited to a lazy setting, where `Seq` does not fare well.

Footnotes
---------

[^1]: The type we use for the key must be a member of the `Eq` type
    class.

[^2]: Non-strict evaluation makes the cost calculation more subtle. We
    only pay for an append if we actually use the resulting list. Even
    then, we only pay for as much as we actually use.

[^3]: Indeed, monoids are ubiquitous throughout programming. The
    difference is that in Haskell, we recognize them, and talk about
    them.
