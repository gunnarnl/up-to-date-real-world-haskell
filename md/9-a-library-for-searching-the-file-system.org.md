Chapter 9: I/O case study: a library for searching the filesystem
=================================================================

The find command
----------------

If you don\'t use a Unix-like operating system, or you\'re not a heavy
shell user, it\'s quite possible you may not have heard of `find`. Given
a list of directories, it searches each one recursively and prints the
name of every entry that matches an expression.

Individual expressions can take such forms as \"name matches this glob
pattern\", \"entry is a plain file\", \"last modified before this
date\", and many more. They can be stitched together into more complex
expressions using \"and\" and \"or\" operators.

Starting simple: recursively listing a directory
------------------------------------------------

Before we plunge into designing our library, let\'s solve a few smaller
problems. Our first problem is to recursively list the contents of a
directory and its subdirectories.

<div>

[RecursiveContents.hs]{.label}

``` {.haskell}
module RecursiveContents (getRecursiveContents) where

import Control.Monad (forM)
import System.Directory (doesDirectoryExist, getDirectoryContents)
import System.FilePath ((</>))

getRecursiveContents :: FilePath -> IO [FilePath]
getRecursiveContents topdir = do
  names <- getDirectoryContents topdir
  let properNames = filter (`notElem` [".", ".."]) names
  paths <- forM properNames $ \name -> do
    let path = topdir </> name
    isDirectory <- doesDirectoryExist path
    if isDirectory
      then getRecursiveContents path
      else return [path]
  return (concat paths)
```

</div>

The `filter` expression ensures that a listing for a single directory
won\'t contain the special directory names `.` or `..`, which refer to
the current and parent directory, respectively. If we forgot to filter
these out, we\'d recurse endlessly.

We encountered `forM` in the previous chapter; it is `mapM` with its
arguments flipped.

``` {.screen}
ghci> :m +Control.Monad
ghci> :type mapM
mapM :: (Monad m, Traversable t) => (a -> m b) -> t a -> m (t b)
ghci> :type forM
forM :: (Monad m, Traversable t) => t a -> (a -> m b) -> m (t b)
```

The body of the loop checks to see whether the current entry is a
directory. If it is, it recursively calls `getRecursiveContents` to list
that directory. Otherwise, it returns a single-element list that is the
name of the current entry. (Don\'t forget that the `return` function has
a unique meaning in Haskell: it wraps a value with the monad\'s type
constructor.)

Another thing worth pointing out is the use of the variable
`isDirectory`. In an imperative language such as Python, we\'d normally
write `if os.path.isdir(path)`. However, the `doesDirectoryExist`
function is an *action*; its return type is `IO Bool`, not `Bool`. Since
an `if` expression requires an expression of type `Bool`, we have to use
`<-` to get the `Bool` result of the action out of its `IO` wrapper, so
that we can use the plain, unwrapped `Bool` in the `if`.

Each iteration of the loop body yields a list of names, so the result of
`forM` here is `IO [[FilePath]]`. We use `concat` to flatten it into a
single list.

### Revisiting anonymous and named functions

In [the section called \"Anonymous (lambda)
functions\"](4-functional-programming.org::*Anonymous (lambda) functions)
some reasons not to use anonymous functions, and yet here we are, using
one as the body of a loop. This is one of the most common uses of
anonymous functions in Haskell.

We\'ve already seen from their types that `forM` and `mapM` take
functions as arguments. Most loop bodies are blocks of code that only
appear once in a program. Since we\'re most likely to use a loop body in
only one place, why give it a name?

Of course, it sometimes happens that we need to deploy exactly the same
code in several different loops. Rather than cutting and pasting the
same anonymous function, it makes sense in such cases to give a name to
an existing anonymous function.

### Why provide both `mapM` and `forM`?

It might seem a bit odd that there exist two functions that are
identical but for the order in which they accept their arguments.
However, `mapM` and `forM` are convenient in different circumstances.

Consider our example above, using an anonymous function as a loop body.
If we were to use `mapM` instead of `forM`, we\'d have to place the
variable `properNames` after the body of the function. In order to get
the code to parse correctly, we\'d have to wrap the entire anonymous
function in parentheses, or replace it with a named function that would
otherwise be unnecessary. Try it yourself: copy the code above,
replacing `forM` with `mapM`, and see what this does to the readability
of the code.

By contrast, if the body of the loop was already a named function, and
the list over which we were looping was computed by a complicated
expression, we\'d have a good case for using `mapM` instead.

The stylistic rule of thumb to follow here is to use whichever of `mapM`
or `forM` lets you write the tidiest code. If the loop body and the
expression computing the data over which you\'re looping are both short,
it doesn\'t matter which you use. If the loop is short, but the data is
long, use `mapM`. If the loop is long, but the data short, use `forM`.
And if both are long, use a `let` or `where` clause to make one of them
short. With just a little practice, it will become obvious which of
these approaches is best in every instance.

A naive finding function
------------------------

We can use our `getRecursiveContents` function as the basis for a
simple-minded file finder.

<div>

[SimpleFinder.hs]{.label}

``` {.haskell}
import RecursiveContents (getRecursiveContents)

simpleFind :: (FilePath -> Bool) -> FilePath -> IO [FilePath]
simpleFind p path = do
  names <- getRecursiveContents path
  return (filter p names)
```

</div>

This function takes a predicate that we use to filter the names returned
by `getRecursiveContents`. Each name passed to the predicate is a
complete path, so how can we perform a common operation like \"find all
files ending in the extension `.c`\"?

The `System.FilePath` module contains numerous invaluable functions that
help us to manipulate file names. In this case, we want `takeExtension`.

``` {.screen}
ghci> :m +System.FilePath
ghci> :type takeExtension
takeExtension :: FilePath -> String
ghci> takeExtension "foo/bar.c"
".c"
ghci> takeExtension "quux"
""
```

This gives us a simple matter of writing a function that takes a path,
extracts its extension, and compares it with `.c`.

``` {.screen}
ghci> :load SimpleFinder
[1 of 2] Compiling RecursiveContents ( RecursiveContents.hs, interpreted )
[2 of 2] Compiling Main             ( SimpleFinder.hs, interpreted )
Ok, two modules loaded.
ghci> :type simpleFind (\p -> takeExtension p == ".c")
simpleFind (\p -> takeExtension p == ".c") :: FilePath -> IO [FilePath]
```

While `simpleFind` works, it has a few glaring problems. The first is
that the predicate is not very expressive. It can only look at the name
of a directory entry; it cannot, for example, find out whether it\'s a
file or a directory. This means that our attempt to use `simpleFind`
will list directories ending in `.c` as well as files with the same
extension.

The second problem is that `simpleFind` gives us no control over how it
traverses the filesystem. To see why this is significant, consider the
problem of searching for a source file in a tree managed by the
Subversion revision control system. Subversion maintains a private
`.svn` directory in every directory that it manages; each one contains
many subdirectories and files that are of no interest to us. While we
can easily enough filter out any path containing `.svn`, it\'s more
efficient to simply avoid traversing these directories in the first
place. For example, one of us has a Subversion source tree containing
45,000 files, 30,000 of which are stored in 1,200 different `.svn`
directories. It\'s cheaper to avoid traversing those 1,200 directories
than to filter out the 30,000 files they contain.

Finally, `simpleFind` is strict, because it consists of a series of
actions executed in the IO monad. If we have a million files to
traverse, we encounter a long delay, then receive one huge result
containing a million names. This is bad for both resource usage and
responsiveness. We might prefer a lazy stream of results delivered as
they arrive.

In the sections that follow, we\'ll overcome each one of these problems.

Predicates: from poverty to riches, while remaining pure
--------------------------------------------------------

Our predicates can only look at file names. This excludes a wide variety
of interesting behaviours: for instance, what if we\'d like to list
files of greater than a given size?

An easy reaction to this is to reach for `IO`: instead of our predicate
being of type `FilePath -> Bool`, why don\'t we change it to
`FilePath -> IO Bool`? This would let us perform arbitrary I/O as part
of our predicate. As appealing as this might seem, it\'s also
potentially a problem: such a predicate could have arbitrary side
effects, since a function with return type `IO` a can have whatever side
effects it pleases.

Let\'s enlist the type system in our quest to write more predictable,
less buggy code: we\'ll keep predicates pure by avoiding the taint of
\"IO\". This will ensure that they can\'t have any nasty side effects.
We\'ll feed them more information, too, so that they can gain the
expressiveness we want without also becoming potentially dangerous.

Haskell\'s portable `System.Directory` module provides a useful, albeit
limited, set of file metadata.

``` {.screen}
ghci> :m +System.Directory
```

-   We can use `doesFileExist` and `doesDirectoryExist` to determine
    whether a directory entry is a file or a directory. There are not
    yet portable ways to query for other file types that have become
    widely available in recent years, such as named pipes, hard links
    and symbolic links.

    ``` {.screen}
    ghci> :type doesFileExist
    doesFileExist :: FilePath -> IO Bool
    ghci> doesFileExist "."
    False
    ghci> :type doesDirectoryExist
    doesDirectoryExist :: FilePath -> IO Bool
    ghci> doesDirectoryExist "."
    True
    ```

-   The `getPermissions` function lets us find out whether certain
    operations on a file or directory are allowed.

    ``` {.screen}
    ghci> :type getPermissions
    getPermissions :: FilePath -> IO Permissions
    ghci> :info Permissions
    data Permissions
      = System.Directory.Permissions {readable :: Bool,
                                      writable :: Bool,
                                      executable :: Bool,
                                      searchable :: Bool}
            -- Defined in ‘System.Directory’
    instance [safe] Eq Permissions -- Defined in ‘System.Directory’
    instance [safe] Ord Permissions -- Defined in ‘System.Directory’
    instance [safe] Show Permissions -- Defined in ‘System.Directory’
    instance [safe] Read Permissions -- Defined in ‘System.Directory’
    ghci> getPermissions "."
    Permissions {readable = True, writable = True, executable = False, searchable = True}
    ghci> :type searchable
    searchable :: Permissions -> Bool
    ghci> searchable it
    True
    ```

    (If you cannot recall the special `ghci` variable `it`, take a look
    back at [the section called \"First steps with
    types\"](1-getting-started.org::*First steps with types) A directory
    will be `searchable` if we have permission to list its contents;
    files are never `searchable`.

-   Finally, `getModificationTime` tells us when an entry was last
    modified.

    ``` {.screen}
    ghci> :type getModificationTime
    getModificationTime
      :: FilePath
         -> IO time-1.8.0.2:Data.Time.Clock.Internal.UTCTime.UTCTime
    ghci> getModificationTime "."
    2018-05-20 22:59:06 UTC
    ```

If we stick with portable, standard Haskell code, these functions are
all we have at our disposal. (We can also find a file\'s size using a
small hack; see below.) They\'re also quite enough to let us illustrate
the principles we\'re interested in, without letting us get carried away
with an example that\'s too expansive. If you need to write more
demanding code, the `System.Posix` and `System.Win32` module families
provide much more detailed file metadata for the two major modern
computing platforms. There also exists a `unix-compat` package on
Hackage, which provides a Unix-like API on Windows.

How many pieces of data does our new, richer predicate need to see?
Since we can find out whether an entry is a file or a directory by
looking at its permissions, we don\'t need to pass in the results of
`doesFileExist` or `doesDirectoryExist`. We thus have four pieces of
data that a richer predicate needs to look at.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
import Control.Exception
  ( bracket
  , handle
  , SomeException(..)
  )
import Control.Monad (filterM)
import System.Directory
  ( Permissions(..)
  , getModificationTime
  , getPermissions
  )
import System.FilePath (takeExtension)
import System.IO
  ( IOMode(..)
  , hClose
  , hFileSize
  , openFile
  )
import Data.Time.Clock (UTCTime(..))

-- the function we wrote earlier
import RecursiveContents (getRecursiveContents)

type Predicate =  FilePath      -- path to directory entry
               -> Permissions   -- permissions
               -> Maybe Integer -- file size (Nothing if not file)
               -> UTCTime       -- last modified
               -> Bool
```

</div>

Our `Predicate` type is just a synonym for a function of four arguments.
It will save us a little keyboard work and screen space.

Notice that the return value of this predicate is `Bool`, not `IO
Bool`: the predicate is pure, and cannot perform I/O. With this type in
hand, our more expressive finder function is still quite trim.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
-- soon to be defined
getFileSize :: FilePath -> IO (Maybe Integer)

betterFind :: Predicate -> FilePath -> IO [FilePath]
betterFind p path = getRecursiveContents path >>= filterM check
    where check name = do
            perms <- getPermissions name
            size <- getFileSize name
            modified <- getModificationTime name
            return (p name perms size modified)
```

</div>

Let\'s walk through the code. We\'ll talk about `getFileSize` in some
detail soon, so let\'s skip over it for now.

We can\'t use `filter` to call our predicate `p`, as `p`\'s purity means
it cannot do the I/O needed to gather the metadata it requires.

This leads us to the unfamiliar function `filterM`. It behaves like the
normal `filter` function, but in this case it evaluates its predicate in
the IO monad, allowing the predicate to perform I/O.

``` {.screen}
ghci> :m +Control.Monad
ghci> :type filterM
filterM :: Applicative m => (a -> m Bool) -> [a] -> m [a]
```

Our `check` predicate is an I/O-capable wrapper for our pure predicate
`p`. It does all the \"dirty\" work of I/O on `p`\'s behalf, so that we
can keep `p` incapable of unwanted side effects. After gathering the
metadata, `check` calls `p`, then uses `return` to wrap `p`\'s result
with IO.

Sizing a file safely
--------------------

Although `System.Directory` doesn\'t let us find out how large a file
is, we can use the similarly portable `System.IO` module to do this. It
contains a function named `hFileSize`, which returns the size in bytes
of an open file. Here\'s a simple function that wraps it.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
simpleFileSize :: FilePath -> IO Integer
simpleFileSize path = do
  h <- openFile path ReadMode
  size <- hFileSize h
  hClose h
  return size
```

</div>

While this function works, it\'s not yet suitable for us to use. In
`betterFind`, we call `getFileSize` unconditionally on any directory
entry; it should return `Nothing` if an entry is not a plain file, or
the size wrapped by `Just` otherwise. This function instead throws an
exception if an entry is not a plain file or could not be opened
(perhaps due to insufficient permissions), and returns the size
unwrapped.

Here\'s a safer version of this function.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
saferFileSize :: FilePath -> IO (Maybe Integer)
saferFileSize path = handle (\_ -> return Nothing) $ do
  h <- openFile path ReadMode
  size <- hFileSize h
  hClose h
  return (Just size)
```

</div>

The body of the function is almost identical, save for the `handle`
clause.

Our exception handler above ignores the exception it\'s passed, and
returns `Nothing`. The only change to the body that follows is that it
wraps the file size with `Just`.

The `saferFileSize` function now has the correct type signature, and it
won\'t throw any exceptions. But it\'s still not completely well
behaved. There are directory entries on which `openFile` will succeed,
but `hFileSize` will throw an exception. This can happen with, for
example, named pipes. Such an exception will be caught by `handle`, but
our call to `hClose` will never occur.

A Haskell implementation will automatically close the file handle when
it notices that the handle is no longer being used. That will not occur
until the garbage collector runs, and the delay until the next garbage
collection pass is not predictable.

File handles are scarce resources. Their scarcity is enforced by the
underlying operating system. On Linux, for example, a process is by
default only allowed to have 1024 files open simultaneously.

It\'s not hard to imagine a scenario in which a program that called a
version of `betterFind` that used `saferFileSize` could crash due to
`betterFind` exhausting the supply of open file handles before enough
garbage file handles could be closed.

This is a particularly pernicious kind of bug: it has several aspects
that combine to make it incredibly difficult to track down. It will only
be triggered if `betterFind` visits a sufficiently large number of
non-files to hit the process\'s limit on open file handles, and then
returns to a caller that tries to open another file before any of the
accumulated garbage file handles is closed.

To make matters worse, any subsequent error will be caused by data that
is no longer reachable from within the program, and has yet to be
garbage collected. Such a bug is thus dependent on the structure of the
program, the contents of the filesystem, and how close the current run
of the program is to triggering the garbage collector.

This sort of problem is easy to overlook during development, and when it
later occurs in the field (as these awkward problems always seem to do),
it will be much harder to diagnose.

Fortunately, we can avoid this kind of error very easily, while also
making our function *shorter*.

### The acquire-use-release cycle

We need `hClose` to always be called if `openFile` succeeds. The
`Control.Exception` module provides the `bracket` function for exactly
this purpose.

``` {.screen}
ghci> :type bracket
bracket :: IO a -> (a -> IO b) -> (a -> IO c) -> IO c
```

The `bracket` function takes three actions as arguments. The first
action acquires a resource. The second releases the resource. The third
runs in between, while the resource is acquired; let\'s call this the
\"use\" action. If the \"acquire\" action succeeds, the \"release\"
action is *always* called. This guarantees that the resource will always
be released. The \"use\" and \"release\" actions are each passed the
resource acquired by the \"acquire\" action.

If an exception occurs while the \"use\" action is executing, `bracket`
calls the \"release\" action and rethrows the exception. If the \"use\"
action succeeds, `bracket` calls the \"release\" action, and returns the
value returned by the \"use\" action.

We can now write a function that is completely safe: it will not throw
exceptions; neither will it accumulate garbage file handles that could
cause spurious failures elsewhere in our program.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
getFileSize path = handle ((\_ -> return Nothing) :: IOError -> IO (Maybe Integer)) $
  bracket (openFile path ReadMode) hClose $ \h -> do
    size <- hFileSize h
    return (Just size)
```

</div>

Look closely at the arguments of `bracket` above. The first opens the
file, and returns the open file handle. The second closes the handle.
The third simply calls `hFileSize` on the handle and wraps the result in
`Just`.

We need to use both `bracket` and `handle` for this function to operate
correctly. The former ensures that we don\'t accumulate garbage file
handles, while the latter gets rid of exceptions.

1.  Exercises

    1.  Is the order in which we call `bracket` and `handle` important?
        Why?

A domain specific language for predicates
-----------------------------------------

Let\'s take a stab at writing a predicate. Our predicate will check for
a C++ source file that is over 128KB in size.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
myTest path _ (Just size) _ =
    takeExtension path == ".cpp" && size > 131072
myTest _ _ _ _ = False
```

</div>

This isn\'t especially pleasing. The predicate takes four arguments,
always ignores two of them, and requires two equations to define. Surely
we can do better. Let\'s create some code that will help us to write
more concise predicates.

Sometimes, this kind of library is referred to as an *embedded domain
specific language*: we use our programming language\'s native facilities
(hence *embedded*) to write code that lets us solve some narrow problem
(hence *domain specific*) particularly elegantly.

Our first step is to write a function that returns one of its arguments.
This one extracts the path from the arguments passed to a `Predicate`.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
pathP path _ _ _ = path
```

</div>

If we don\'t provide a type signature, a Haskell implementation will
infer a very general type for this function. This can later lead to
error messages that are difficult to interpret, so let\'s give `pathP` a
type.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
type InfoP a =  FilePath        -- path to directory entry
             -> Permissions     -- permissions
             -> Maybe Integer   -- file size (Nothing if not file)
             -> UTCTime         -- last modified
             -> a

pathP :: InfoP FilePath
```

</div>

We\'ve created a type synonym that we can use as shorthand for writing
other, similarly structured functions. Our type synonym accepts a type
parameter so that we can specify different result types.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
sizeP :: InfoP Integer
sizeP _ _ (Just size) _ = size
sizeP _ _ Nothing     _ = -1
```

</div>

(We\'re being a little sneaky here, and returning a size of -1 for
entries that are not files, or that we couldn\'t open.)

In fact, a quick glance shows that the `Predicate` type that we defined
near the beginning of this chapter is the same type as `InfoP Bool`. (We
could thus legitimately get rid of the `Predicate` type.)

What use are `pathP` and `sizeP`? With a little more glue, we can use
them in a predicate (the `P` suffix on each name is intended to suggest
\"predicate\"). This is where things start to get interesting.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
equalP :: (Eq a) => InfoP a -> a -> InfoP Bool
equalP f k = \w x y z -> f w x y z == k
```

</div>

The type signature of `equalP` deserves a little attention. It takes an
`InfoP a`, which is compatible with both `pathP` and `sizeP`. It takes
an `a`. And it returns an `InfoP Bool`, which we already observed is a
synonym for `Predicate`. In other words, `equalP` constructs a
predicate.

The `equalP` function works by returning an anonymous function. That one
takes the arguments accepted by a predicate, passes them to `f`, and
compares the result to `k`.

This equation for `equalP` emphasises the fact that we think of it as
taking two arguments. Since Haskell curries all functions, writing
`equalP` in this way is not actually necessary. We can omit the
anonymous function and rely on currying to work on our behalf, letting
us write a function that behaves identically.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
equalP' :: (Eq a) => InfoP a -> a -> InfoP Bool
equalP' f k w x y z = f w x y z == k
```

</div>

Before we continue with our explorations, let\'s load our module into
`ghci`.

``` {.screen}
ghci> :load BetterPredicate
[1 of 2] Compiling RecursiveContents ( RecursiveContents.hs, interpreted )
[2 of 2] Compiling Main             ( BetterPredicate.hs, interpreted )
Ok, two modules loaded.
```

Let\'s see if a simple predicate constructed from these functions will
work.

``` {.screen}
ghci> :type betterFind (sizeP `equalP` 1024)
betterFind (sizeP `equalP` 1024) :: FilePath -> IO [FilePath]
```

Notice that we\'re not actually calling `betterFind`, we\'re merely
making sure that our expression type-checks. We now have a more
expressive way to list all files that are exactly some size. Our success
gives us enough confidence to continue.

### Avoiding boilerplate with lifting

Besides `equalP`, we\'d like to be able to write other binary functions.
We\'d prefer not to write a complete definition of each one, because
that seems unnecessarily verbose.

To address this, let\'s put Haskell\'s powers of abstraction to use.
We\'ll take the definition of `equalP`, and instead of calling `(==)`
directly, we\'ll pass in as another argument the binary function that we
want to call.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
liftP :: (a -> b -> c) -> InfoP a -> b -> InfoP c
liftP q f k w x y z = f w x y z `q` k

greaterP, lesserP :: (Ord a) => InfoP a -> a -> InfoP Bool
greaterP = liftP (>)
lesserP = liftP (<)
```

</div>

This act of taking a function, such as `(>)`, and transforming it into
another function that operates in a different context, here `greaterP`,
is referred to as *lifting* it into that context. This explains the
presence of `lift` in the function\'s name. Lifting lets us reuse code
and reduce boilerplate. We\'ll be using it a lot, in different guises,
throughout the rest of this book.

When we lift a function, we\'ll often refer to its original and new
versions as *unlifted* and *lifted*, respectively.

By the way, our placement of `q` (the function to lift) as the first
argument to `liftP` was quite deliberate. This made it possible for us
to write such concise definitions of `greaterP` and `lesserP`. Partial
application makes finding the \"best\" order for arguments a more
important part of API design in Haskell than in other languages. In
languages without partial application, argument ordering is a matter of
taste and convention. Put an argument in the wrong place in Haskell,
however, and we lose the concision that partial application gives.

We can recover some of that conciseness via combinators. For instance,
`forM` was not added to the `Control.Monad` module until

1.  Prior to that, people wrote `flip mapM` instead.

``` {.screen}
ghci> :m +Control.Monad
ghci> :t mapM
mapM :: (Monad m, Traversable t) => (a -> m b) -> t a -> m (t b)
ghci> :t forM
forM :: (Monad m, Traversable t) => t a -> (a -> m b) -> m (t b)
ghci> :t flip mapM
flip mapM :: (Monad m, Traversable t) => t a -> (a -> m b) -> m (t b)
```

### Gluing predicates together

If we want to combine predicates, we can of course follow the obvious
path of doing so by hand.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
simpleAndP :: InfoP Bool -> InfoP Bool -> InfoP Bool
simpleAndP f g w x y z = f w x y z && g w x y z
```

</div>

Now that we know about lifting, it becomes more natural to reduce the
amount of code we must write by lifting our existing boolean operators.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
liftP2 :: (a -> b -> c) -> InfoP a -> InfoP b -> InfoP c
liftP2 q f g w x y z = f w x y z `q` g w x y z

andP = liftP2 (&&)
orP = liftP2 (||)
```

</div>

Notice that `liftP2` is very similar to our earlier `liftP`. In fact,
it\'s more general, because we can write `liftP` in terms of `liftP2`.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
constP :: a -> InfoP a
constP k _ _ _ _ = k

liftP' q f k w x y z = f w x y z `q` constP k w x y z
```

</div>

::: {.TIP}
Combinators

In Haskell, we refer to functions that take other functions as
arguments, returning new functions, as *combinators*.
:::

Now that we have some helper functions in place, we can return to the
`myTest` function we defined earlier.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
myTest path _ (Just size) _ =
    takeExtension path == ".cpp" && size > 131072
myTest _ _ _ _ = False
```

</div>

How will this function look if we write it using our new combinators?

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
liftPath :: (FilePath -> a) -> InfoP a
liftPath f w _ _ _ = f w

myTest2 = (liftPath takeExtension `equalP` ".cpp") `andP`
          (sizeP `greaterP` 131072)
```

</div>

We\'ve added one final combinator, `liftPath`, since manipulating file
names is such a common activity.

### Defining and using new operators

We can take our domain specific language further by defining new infix
operators.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
(==?) = equalP
(&&?) = andP
(>?) = greaterP

myTest3 = (liftPath takeExtension ==? ".cpp") &&? (sizeP >? 131072)
```

</div>

We chose names like `(==?)` for the lifted functions specifically for
their visual similarity to their unlifted counterparts.

The parentheses in our definition above are necessary, because we
haven\'t told Haskell about the precedence or associativity of our new
operators. The language specifies that operators without fixity
declarations should be treated as `infixl 9`, i.e. they are evaluated
from left to right at the highest precedence level. If we were to omit
the parentheses, the expression would thus be parsed as
`(((liftPath takeExtension) ==? ".cpp") &&? sizeP) >? 131072`, which is
horribly wrong.

We can respond by writing fixity declarations for our new operators. Our
first step is to find out what the fixities of the unlifted operators
are, so that we can mimic them.

``` {.screen}
ghci> :info ==
class Eq a where
  (==) :: a -> a -> Bool
  ...
        -- Defined in ‘GHC.Classes’
infix 4 ==
ghci> :info &&
(&&) :: Bool -> Bool -> Bool -- Defined in ‘GHC.Classes’
infixr 3 &&
ghci> :info >
class Eq a => Ord a where
  ...
  (>) :: a -> a -> Bool
  ...
        -- Defined in ‘GHC.Classes’
infix 4 >
```

With these in hand, we can now write a parenthesis-free expression that
will be parsed identically to `myTest3`.

<div>

[BetterPredicate.hs]{.label}

``` {.haskell}
infix 4 ==?
infixr 3 &&?
infix 4 >?

myTest4 = liftPath takeExtension ==? ".cpp" &&? sizeP >? 131072
```

</div>

Controlling traversal
---------------------

When traversing the filesystem, we\'d like to give ourselves more
control over which directories we enter, and when. An easy way in which
we can allow this is to pass in a function that takes a list of
subdirectories of a given directory, and returns another list. This list
can have elements removed, or it can be ordered differently than the
original list, or both. The simplest such control function is `id`,
which will return its input list unmodified.

For variety, we\'re going to change a few aspects of our representation
here. Instead of an elaborate function type `InfoP a`, we\'ll use a
normal algebraic data type to represent substantially the same
information.

<div>

[ControlledVisit.hs]{.label}

``` {.haskell}
module ControlledVisit where

import Control.Monad (forM, liftM)
import Data.Time.Clock (UTCTime(..))
import System.FilePath ((</>))
import System.Directory
    ( Permissions(..)
    , getModificationTime
    , getPermissions
    , getDirectoryContents
    )
import Control.Exception
    ( bracket
    , handle
    , SomeException(..)
    )
import System.IO
    ( IOMode(..)
    , hClose
    , hFileSize
    , openFile
    )

data Info = Info
    { infoPath :: FilePath
    , infoPerms :: Maybe Permissions
    , infoSize :: Maybe Integer
    , infoModTime :: Maybe UTCTime
    } deriving (Eq, Ord, Show)

getInfo :: FilePath -> IO Info
```

</div>

We\'re using record syntax to give ourselves \"free\" accessor
functions, such as `infoPath`. The type of our `traverseDirs` function
is simple, as we proposed above. To obtain `Info` about a file or
directory, we call the `getInfo` action.

<div>

[ControlledVisit.hs]{.label}

``` {.haskell}
traverseDirs :: ([Info] -> [Info]) -> FilePath -> IO [Info]
```

</div>

The definition of `traverseDirs` is short, but dense.

<div>

[ControlledVisit.hs]{.label}

``` {.haskell}
traverseDirs order path = do
    names <- getUsefulContents path
    contents <- mapM getInfo (path : map (path </>) names)
    liftM concat $ forM (order contents) $ \info -> do
      if isDirectory info && infoPath info /= path
        then traverseDirs order (infoPath info)
        else return [info]

getUsefulContents :: FilePath -> IO [String]
getUsefulContents path = do
    names <- getDirectoryContents path
    return (filter (`notElem` [".", ".."]) names)

isDirectory :: Info -> Bool
isDirectory = maybe False searchable . infoPerms
```

</div>

While we\'re not introducing any new techniques here, this is one of the
densest function definitions we\'ve yet encountered. Let\'s walk through
it almost line by line, explaining what is going on. The first couple of
lines hold no mystery, as they\'re almost verbatim copies of code we\'ve
already seen.

Things begin to get interesting when we assign to the variable
`contents`. Let\'s read this line from right to left. We already know
that `names` is a list of directory entries. We make sure that the
current directory is prepended to every element of the list, and
included in the list itself. We use `mapM` to apply `getInfo` to the
resulting paths.

The line that follows is even more dense. Again reading from right to
left, we see that the last element of the line begins the definition of
an anonymous function that continues to the end of the paragraph. Given
one `Info` value, this function either visits a directory recursively
(there\'s an extra check to make sure we don\'t visit `path` again), or
returns that value as a single-element list (to match the result type of
`traverseDirs`).

We use `forM` to apply this function to each element of the list of
`Info` values returned by `order`, the user-supplied traversal control
function.

At the beginning of the line, we use the technique of lifting in a new
context. The `liftM` function takes a regular function, `concat`, and
lifts it into the IO monad. In other words, it takes the result of
`forM` (of type `IO [[Info]]`) out of the IO monad, applies `concat` to
it (yielding a result of type `[Info]`, which is what we need), and puts
the result back into the IO monad.

Finally, we mustn\'t forget to define our `getInfo` function.

<div>

[ControlledVisit.hs]{.label}

``` {.haskell}
maybeIO :: IO a -> IO (Maybe a)
maybeIO act = handle (\(SomeException _) -> return Nothing) (Just `liftM` act)

getInfo path = do
  perms <- maybeIO (getPermissions path)
  size <- maybeIO (bracket (openFile path ReadMode) hClose hFileSize)
  modified <- maybeIO (getModificationTime path)
  return (Info path perms size modified)
```

</div>

The only noteworthy thing here is a useful combinator, `maybeIO`, which
turns an I/O action that might throw an exception into one that wraps
its result in `Maybe`.

### Exercises {#exercises-1}

1.  What should you pass to `traverseDirs` to traverse a directory tree
    in reverse alphabetic order?
2.  Using `id` as a control function, `traverse id` performs a
    *preorder* traversal of a tree: it returns a parent directory before
    its children. Write a control function that makes `traverseDirs`
    perform a *postorder* traversal, in which it returns children before
    their parent.
3.  Take the predicates and combinators from [the section called
    \"Gluing predicates
    together\"](9-a-library-for-searching-the-file-system.org::*Gluing predicates together)
    work with our new `Info` type.
4.  Write a wrapper for `traverseDirs` that lets you control traversal
    using one predicate, and filter results using another.

Density, readability, and the learning process
----------------------------------------------

Code as dense as `traverseDirs` is not unusual in Haskell. The gain in
expressiveness is significant, and it requires a relatively small amount
of practice to be able to fluently read and write code in this style.

For comparison, here\'s a less dense presentation of the same code. This
might be more typical of a less experienced Haskell programmer.

<div>

[ControlledVisit.hs]{.label}

``` {.haskell}
traverseVerbose order path = do
    names <- getDirectoryContents path
    let usefulNames = filter (`notElem` [".", ".."]) names
    contents <- mapM getEntryName ("" : usefulNames)
    recursiveContents <- mapM recurse (order contents)
    return (concat recursiveContents)
  where getEntryName name = getInfo (path </> name)
        isDirectory info = case infoPerms info of
                             Nothing -> False
                             Just perms -> searchable perms
        recurse info = do
            if isDirectory info && infoPath info /= path
                then traverseVerbose order (infoPath info)
                else return [info]
```

</div>

All we\'ve done here is make a few substitutions. Instead of liberally
using partial application and function composition, we\'ve defined some
local functions in a `where` block. In place of the `maybe` combinator,
we\'re using a `case` expression. And instead of using `liftM`, we\'re
manually lifting `concat` ourselves.

This is not to say that density is a uniformly good property. Each line
of the original `traverseDirs` function is short. We introduce a local
variable (`usefulNames`) and a local function (`isDirectory`)
specifically to keep the lines short and the code clearer. Our names are
descriptive. While we use function composition and pipelining, the
longest pipeline contains only three elements.

The key to writing maintainable Haskell code is to find a balance
between density and readability. Where your code falls on this continuum
is likely to be influenced by your level of experience.

-   As a beginning Haskell programmer, Andrew doesn\'t know his way
    around the standard libraries very well. As a result, he unwittingly
    duplicates a lot of existing code.
-   Zack has been programming for a few months, and has mastered the use
    of `(.)` to compose long pipelines of code. Every time the needs of
    his program change slightly, he has to construct a new pipeline from
    scratch: he can\'t understand the existing pipeline any longer, and
    it is in any case too fragile to change.
-   Monica has been coding for a while. She\'s familiar enough with
    Haskell libraries and idioms to write tight code, but she avoids a
    hyperdense style. Her code is maintainable, and she finds it easy to
    refactor when faced with changing requirements.

Another way of looking at traversal
-----------------------------------

While the `traverseDirs` function gives us more control than our
original `betterFind` function, it still has a significant failing: we
can avoid recursing into directories, but we can\'t filter other names
until after we\'ve generated the entire list of names in a tree. If we
are traversing a directory containing 100,000 files of which we care
about three, we\'ll allocate a 100,000-element list before we have a
chance to trim it down to the three we really want.

One approach would be to provide a filter function as a new argument to
`traverseDirs`, which we would apply to the list of names as we generate
it. This would allow us to allocate a list of only as many elements as
we need.

However, this approach also has a weakness: say we know that we want at
most three entries from our list, and that those three entries happen to
be the first three of the 100,000 that we traverse. In this case, we\'ll
needlessly visit 99,997 other entries. This is not by any means a
contrived example: for example, the Maildir mailbox format stores a
folder of email messages as a directory of individual files. It\'s
common for a single directory representing a mailbox to contain tens of
thousands of files.

We can address the weaknesses of our two prior traversal functions by
taking a different perspective: what if we think of filesystem traversal
as a *fold* over the directory hierarchy?

The familiar folds, `foldr` and `foldl'`, neatly generalise the idea of
traversing a list while accumulating a result. It\'s hardly a stretch to
extend the idea of folding from lists to directory trees, but we\'d like
to add an element of *control* to our fold. We\'ll represent this
control as an algebraic data type.

<div>

[FoldDir.hs]{.label}

``` {.haskell}
import ControlledVisit
import Data.Char (toLower)
import Data.Time.Clock (UTCTime(..))
import System.Directory (Permissions(..))
import System.FilePath ((</>), takeExtension, takeFileName)

data Iterate seed = Done     { unwrap :: seed }
                  | Skip     { unwrap :: seed }
                  | Continue { unwrap :: seed }
                    deriving (Show)

type Iterator seed = seed -> Info -> Iterate seed
```

</div>

The `Iterator` type gives us a convenient alias for the function that we
fold with. It takes a seed and an `Info` value representing a directory
entry, and returns both a new seed and an instruction for our fold
function, where the instructions are represented as the constructors of
the `Iterate` type.

-   If the instruction is `Done`, traversal should cease immediately.
    The value wrapped by `Done` should be returned as the result.
-   If the instruction is `Skip` and the current `Info` represents a
    directory, traversal will not recurse into that directory.
-   Otherwise, the traversal should continue, using the wrapped value as
    the input to the next call to the fold function.

Our fold is logically a kind of left fold, because we start folding from
the first entry we encounter, and the seed for each step is the result
of the prior step.

<div>

[FoldDir.hs]{.label}

``` {.haskell}
foldTree :: Iterator a -> a -> FilePath -> IO a
foldTree iter initSeed path = do
    endSeed <- fold initSeed path
    return (unwrap endSeed)
  where
    fold seed subpath = getUsefulContents subpath >>= walk seed
    walk seed (name : names) = do
      let path' = path </> name
      info <- getInfo path'
      case iter seed info of
        done @ (Done _) -> return done
        Skip seed  '    -> walk seed' names
        Continue seed'
          | isDirectory info -> do
              next <- fold seed' path'
              case next of
                done @ (Done _) -> return done
                seed''          -> walk (unwrap seed'') names
          | otherwise -> walk seed' names
    walk seed _ = return (Continue seed)
```

</div>

There are a few interesting things about the way this code is written.
The first is the use of scoping to avoid having to pass extra parameters
around. The top-level `foldTree` function is just a wrapper for `fold`
that peels off the constructor of the `fold`\'s final result.

Because `fold` is a local function, we don\'t have to pass `foldTree`\'s
`iter` variable into it; it can already access it in the outer scope.
Similarly, `walk` can see `path` in its outer scope.

Another point to note is that `walk` is a tail recursive loop, instead
of an anonymous function called by `forM` as in our earlier functions.
By taking the reins ourselves, we can stop early if we need to. This
lets us drop out when our iterator returns `Done`.

Although `fold` calls `walk`, `walk` calls `fold` recursively to
traverse subdirectories. Each function returns a seed wrapped in an
`Iterate`: when `fold` is called by `walk` and returns, `walk` examines
its result to see whether it should continue or drop out because it
returned `Done`. In this way, a return of `Done` from the
caller-supplied iterator immediately terminates all mutually recursive
calls between the two functions.

What does an iterator look like in practice? Here\'s a somewhat
complicated example that looks for at most three bitmap images, and
won\'t recurse into Subversion metadata directories.

<div>

[FoldDir.hs]{.label}

``` {.haskell}
atMostThreePictures :: Iterator [FilePath]
atMostThreePictures paths info
    | length paths == 3
      = Done paths
    | isDirectory info && takeFileName path == ".svn"
      = Skip paths
    | extension `elem` [".jpg", ".png"]
      = Continue (path : paths)
    | otherwise
      = Continue paths
  where extension = map toLower (takeExtension path)
        path = infoPath info
```

</div>

To use this, we\'d call `foldTree atMostThreePictures [] "."` (where `.`
is the current directory, you can supply another path), giving us a
return value of type `IO [FilePath]`.

Of course, iterators don\'t have to be this complicated. Here\'s one
that counts the number of directories it encounters.

<div>

[FoldDir.hs]{.label}

``` {.haskell}
countDirectories count info =
    Continue (if isDirectory info
              then count + 1
              else count)
```

</div>

Here, the initial seed that we pass to `foldTree` should be the number
zero.

### Exercises {#exercises-2}

1.  Modify `foldTree` to allow the caller to change the order of
    traversal of entries in a directory.
2.  The `foldTree` function performs preorder traversal. Modify it to
    allow the caller to determine the order of traversal.
3.  Write a combinator library that makes it possible to express the
    kinds of iterators that `foldTree` accepts. Does it make the
    iterators you write any more succinct?

Useful coding guidelines
------------------------

While many good Haskell programming habits come with experience, we have
a few general guidelines to offer so that you can write readable code
more quickly.

If you find yourself proudly thinking that a particular piece of code is
fiendishly clever, stop and consider whether you\'ll be able to
understand it again after you\'ve stepped away from it for a month.

The conventional way of naming types and variables with compound names
is to use \"camel case\", i.e. `myVariableName`. This style is almost
universal in Haskell code. Regardless of your opinion of other naming
practices, if you follow a non-standard convention, your Haskell code
will be somewhat jarring to the eyes of other readers.

Until you\'ve been working with Haskell for a substantial amount of
time, spend a few minutes searching for library functions before you
write small functions. This applies particularly to ubiquitous types
like lists, `Maybe`, and `Either`. If the standard libraries don\'t
already provide exactly what you need, you might be able to combine a
few functions to obtain the result you desire.

Long pipelines of composed functions are hard to read, where \"long\"
means a series of more than three or four elements. If you have such a
pipeline, use a `let` or `where` block to break it into smaller parts.
Give each one of these pipeline elements a meaningful name, then glue
them back together. If you can\'t think of a meaningful name for an
element, ask yourself if you can even describe what it does. If the
answer is \"no\", simplify your code.

Even though it\'s easy to resize a text editor window far beyond 80
columns, this width is still very common. Wider lines are wrapped or
truncated in 80-column text editor windows, which severely hurts
readability. Treating lines as no more than 80 characters long limits
the amount of code you can cram onto a single line. This helps to keep
individual lines less complicated, therefore easier to understand.

### Common layout styles

A Haskell implementation won\'t make a fuss about indentation as long as
your code follows the layout rules and can hence be parsed
unambiguously. That said, some layout patterns are widely used.

As we already mentioned in [the section called \"A note about tabs
versus
spaces\"](3-defining-types-streamlining-functions.org::*A note about tabs versus spaces)
to use spaces.

The `in` keyword is usually aligned directly under the `let` keyword,
with the expression immediately following it.

<div>

[Style.hs]{.label}

``` {.haskell}
tidyLet = let foo = undefined
              bar = foo * 2
          in undefined
```

</div>

While it\'s *legal* to indent the `in` differently, or to let it
\"dangle\" at the end of a series of equations, the following would
generally be considered odd.

<div>

[Style.hs]{.label}

``` {.haskell}
weirdLet = let foo = undefined
               bar = foo * 2
    in undefined

strangeLet = let foo = undefined
                 bar = foo * 2 in
    undefined
```

</div>

In contrast, it\'s usual to let a `do` dangle at the end of a line,
rather than sit at the beginning of a line.

<div>

[Style.hs]{.label}

``` {.haskell}
commonDo = do
  something <- undefined
  return ()

-- not seen very often
rareDo =
  do something <- undefined
     return ()
```

</div>

Curly braces and semicolons, though legal, are almost never used.
There\'s nothing wrong with them; they just make code look strange due
to their rarity. They\'re really intended to let programs generate
Haskell code without having to implement the layout rules, not for human
use.

<div>

[Style.hs]{.label}

``` {.haskell}
unusualPunctuation =
    [ (x,y) | x <- [1..a], y <- [1..b] ] where {
                                           b = 7;
 a = 6 }

preferredLayout = [ (x,y) | x <- [1..a], y <- [1..b] ]
    where b = 7
          a = 6
```

</div>

If the right hand side of an equation starts on a new line, it\'s
usually indented a small number of spaces relative to the name of the
variable or function that it\'s defining.

<div>

[Style.hs]{.label}

``` {.haskell}
normalIndent =
    undefined

strangeIndent =
                           undefined
```

</div>

The actual number of spaces used to indent varies, sometimes within a
single file. Depths of two, three, and four spaces are about equally
common. A single space is legal, but not very visually distinctive, so
it\'s easy to misread.

When indenting a `where` clause, it\'s best to make it visually
distinctive.

<div>

[Style.hs]{.label}

``` {.haskell}
goodWhere = take 5 lambdas
    where lambdas = []

alsoGood =
    take 5 lambdas
  where
    lambdas = []

badWhere =           -- legal, but ugly and hard to read
    take 5 lambdas
    where
    lambdas = []
```

</div>

Exercises {#exercises-3}
---------

1.  Port the code from this chapter to your platform\'s native API,
    either `System.Posix` or `System.Win32`.
2.  Add the ability to find out who owns a directory entry to your code.
    Make this information available to predicates.

Footnotes
---------

Although the file finding code we described in this chapter is a good
vehicle for learning, it\'s not ideal for real systems programming
tasks, because Haskell\'s portable I/O libraries don\'t expose enough
information to let us write interesting and complicated queries.
