Chapter 24. Concurrent and multicore programming
================================================

As we write this book, the landscape of CPU architecture is changing
more rapidly than it has in decades.

Defining concurrency and parallelism
------------------------------------

A *concurrent* program needs to perform several possibly unrelated tasks
at the same time. Consider the example of a game server: it is typically
composed of dozens of components, each of which has complicated
interactions with the outside world. One component might handle
multi-user chat; several more will process the inputs of players, and
feed state updates back to them; while another performs physics
calculations.

The correct operation of a concurrent program does not require multiple
cores, though they may improve performance and responsiveness.

In contrast, a *parallel* program solves a single problem. Consider a
financial model that attempts to predict the next minute of fluctuations
in the price of a single stock. If we want to apply this model to every
stock listed on an exchange, for example to estimate which ones we
should buy and sell, we hope to get an answer more quickly if we run the
model on five hundred cores than if we use just one. As this suggests, a
parallel program does not usually depend on the presence of multiple
cores to work correctly.

Another useful distinction between concurrent and parallel programs lies
in their interaction with the outside world. By definition, a concurrent
program deals continuously with networking protocols, databases, and the
like. A typical parallel program is likely to be more focused: it
streams data in, crunches it for a while (with little further I/O), then
streams data back out.

Many traditional languages further blur the already indistinct boundary
between concurrent and parallel programming, because they force
programmers to use the same primitives to construct both kinds of
program.

In this chapter, we will concern ourselves with concurrent and parallel
programs that operate within the boundaries of a single operating system
process.

Concurrent programming with threads
-----------------------------------

As a building block for concurrent programs, most programming languages
provide a way of creating multiple independent *threads of control*.
Haskell is no exception, though programming with threads in Haskell
looks somewhat different than in other languages.

In Haskell, a thread is an `IO` action that executes independently from
other threads. To create a thread, we import the `Control.Concurrent`
module and use the `forkIO` function.

``` {.screen}
ghci> :m +Control.Concurrent
ghci> :t forkIO
forkIO :: IO () -> IO ThreadId
ghci> :m +System.Directory
ghci> forkIO (writeFile "xyzzy" "seo craic nua!") >> doesFileExist "xyzzy"
False
```

The new thread starts to execute almost immediately, and the thread that
created it continues to execute concurrently. The thread will stop
executing when it reaches the end of its `IO` action.

### Threads are nondeterministic

The runtime component of GHC does not specify an order in which it
executes threads. As a result, in our example above, the file `xyzzy`
created by the new thread *may or may not* have been created by the time
the original thread checks for its existence. If we try this example
once, then remove `xyzzy` and try again, we may get a different result
the second time.

### Hiding latency

Suppose we have a large file to compress and write to disk, but we want
to handle a user\'s input quickly enough that they will perceive our
program as responding immediately. If we use `forkIO` to write the file
out in a separate thread, we can do both simultaneously.

<div>

[Compressor.hs]{.label}

``` {.haskell}
import Control.Concurrent (forkIO)
import Control.Exception (handle)
import Control.Monad (forever)
import qualified Data.ByteString.Lazy as L
import System.Console.Readline (readline)

-- Provided by the 'zlib' package on http://hackage.haskell.org/
import Codec.Compression.GZip (compress)

main = do
    maybeLine <- readline "Enter a file to compress> "
    case maybeLine of
      Nothing -> return ()      -- user entered EOF
      Just "" -> return ()      -- treat no name as "want to quit"
      Just name -> do
           handle print $ do
             content <- L.readFile name
             forkIO (compressFile name content)
             return ()
           main
  where compressFile path = L.writeFile (path ++ ".gz") . compress
```

</div>

Because we\'re using lazy `ByteString` I/O here, all we really do in the
main thread is open the file. The actual reading occurs on demand in the
other thread.

The use of `handle print` gives us a cheap way to print an error message
if the user enters the name of a file that does not exist.

Simple communication between threads
------------------------------------

The simplest way to share information between two threads is to let them
both use a variable. In our file compression example, the `main` thread
shares both the name of a file and its contents with the other thread.
Because Haskell data is immutable by default, this poses no risks:
neither thread can modify the other\'s view of the file\'s name or
contents.

We often need to have threads actively communicate with each other. For
example, GHC does not provide a way for one thread to find out whether
another is still executing, has completed, or has crashed[^1]. However,
it provides a *synchronizing variable* type, the `MVar`, which we can
use to create this capability for ourselves.

An `MVar` acts like a single-element box: it can be either full or
empty. We can put something into the box, making it full, or take
something out, making it empty.

``` {.screen}
ghci> :t putMVar
putMVar :: MVar a -> a -> IO ()
ghci> :t takeMVar
takeMVar :: MVar a -> IO a
```

If we try to put a value into an `MVar` that is already full, our thread
is put to sleep until another thread takes the value out. Similarly, if
we try to take a value from an empty `MVar`, our thread is put to sleep
until some other thread puts a value in.

<div>

[MVarExample.hs]{.label}

``` {.haskell}
import Control.Concurrent

communicate = do
  m <- newEmptyMVar
  forkIO $ do
    v <- takeMVar m
    putStrLn ("received " ++ show v)
  putStrLn "sending"
  putMVar m "wake up!"
```

</div>

The `newEmptyMVar` function has a descriptive name. To create an `MVar`
that starts out non-empty, we\'d use `newMVar`.

``` {.screen}
ghci> :t newEmptyMVar
newEmptyMVar :: IO (MVar a)
ghci> :t newMVar
newMVar :: a -> IO (MVar a)
```

Let\'s run our example in `ghci`.

``` {.screen}
ghci> :load MVarExample
[1 of 1] Compiling Main             ( MVarExample.hs, interpreted )
Ok, modules loaded: Main.
ghci> communicate
sending
rece
```

If you\'re coming from a background of concurrent programming in a
traditional language, you can think of an `MVar` as being useful for two
familiar purposes.

-   Sending a message from one thread to another, e.g. a notification.
-   Providing *mutual exclusion* for a piece of mutable data that is
    shared among threads. We put the data into the `MVar` when it is not
    being used by any thread, and one thread takes it out temporarily to
    read or modify it.

The main thread and waiting for other threads
---------------------------------------------

GHC\'s runtime system treats the program\'s original thread of control
differently from other threads. When this thread finishes executing, the
runtime system considers the program as a whole to have completed. If
any other threads are executing at the time, they are terminated.

As a result, when we have long-running threads that must not be killed,
we must make special arrangements to ensure that the main thread
doesn\'t complete until the others do. Let\'s develop a small library
that makes this easy to do.

<div>

[NiceFork.hs]{.label}

``` {.haskell}
import Control.Concurrent
import Control.Exception (Exception, try)
import qualified Data.Map as M

data ThreadStatus = Running
                  | Finished         -- terminated normally
                  | Threw Exception  -- killed by uncaught exception
                    deriving (Eq, Show)

-- | Create a new thread manager.
newManager :: IO ThreadManager

-- | Create a new managed thread.
forkManaged :: ThreadManager -> IO () -> IO ThreadId

-- | Immediately return the status of a managed thread.
getStatus :: ThreadManager -> ThreadId -> IO (Maybe ThreadStatus)

-- | Block until a specific managed thread terminates.
waitFor :: ThreadManager -> ThreadId -> IO (Maybe ThreadStatus)

-- | Block until all managed threads terminate.
waitAll :: ThreadManager -> IO ()
```

</div>

We keep our `ThreadManager` type abstract using the usual recipe: we
wrap it in a `newtype`, and prevent clients from creating values of this
type. Among our module\'s exports, we list the type constructor and the
`IO` action that constructs a manager, but we do not export the data
constructor.

<div>

[NiceFork.hs]{.label}

``` {.haskell}
module NiceFork
    (
      ThreadManager
    , newManager
    , forkManaged
    , getStatus
    , waitFor
    , waitAll
    ) where
```

</div>

For the implementation of `ThreadManager`, we maintain a map from thread
`ID` to thread state. We\'ll refer to this as the *thread map*.

<div>

[NiceFork.hs]{.label}

``` {.haskell}
newtype ThreadManager =
    Mgr (MVar (M.Map ThreadId (MVar ThreadStatus)))
    deriving (Eq)

newManager = Mgr `fmap` newMVar M.empty
```

</div>

We have two levels of `MVar` use here. We keep the `Map` in an `MVar`.
This lets us \"modify\" the map by replacing it with a new version. We
also ensure that any thread that uses the Map will see a consistent view
of it.

For each thread that we manage, we maintain an `MVar`. A per-thread
`MVar` starts off empty, which indicates that the thread is executing.
When the thread finishes or is killed by an uncaught exception, we put
this information into the `MVar`.

To create a thread and watch its status, we must perform a little bit of
book-keeping.

<div>

[NiceFork.hs]{.label}

``` {.haskell}
forkManaged (Mgr mgr) body =
    modifyMVar mgr $ \m -> do
      state <- newEmptyMVar
      tid <- forkIO $ do
        result <- try body
        putMVar state (either Threw (const Finished) result)
      return (M.insert tid state m, tid)
```

</div>

### Safely modifying an `MVar`

The `modifyMVar` function that we used in `forkManaged` above is very
useful: it\'s a safe combination of `takeMVar` and `putMVar`.

``` {.screen}
ghci> :t modifyMVar
ved "wake up!"
modifyMVar :: MVar a -> (a -> IO (a, b)) -> IO b
```

It takes the value from an `MVar`, and passes it to a function. This
function can both generate a new value and return a result. If the
function throws an exception, `modifyMVar` puts the original value back
into the `MVar`, otherwise it puts the new value in. It returns the
other element of the function as its own result.

When we use `modifyMVar` instead of manually managing an `MVar` with
`takeMVar` and `putMVar`, we avoid two common kinds of concurrency bug.

-   Forgetting to put a value back into an `MVar`. This can result in
    *deadlock*, in which some thread waits forever on an `MVar` that
    will never have a value put into it.
-   Failure to account for the possibility that an exception might be
    thrown, disrupting the flow of a piece of code. This can result in a
    call to `putMVar` that *should* occur not actually happening, again
    leading to deadlock.

Because of these nice safety properties, it\'s wise to use `modifyMVar`
whenever possible.

### Safe resource management: a good idea, and easy besides

We can the take the pattern that `modifyMVar` follows, and apply it to
many other resource management situations. Here are the steps of the
pattern.

1.  Acquire a resource. 2. Pass the resource to a function that

will do something with it. 3. Always release the resource, even if the
function throws an exception. If that occurs, rethrow the exception so
it can be caught by application code.

Safety aside, this approach has another benefit: it can make our code
shorter and easier to follow. As we can see from looking at
`forkManaged` above, Haskell\'s lightweight syntax for anonymous
functions makes this style of coding visually unobtrusive.

Here\'s the definition of `modifyMVar`, so that you can see a specific
form of this pattern.

<div>

[ModifyMVar.hs]{.label}

``` {.haskell}
import Control.Concurrent (MVar, putMVar, takeMVar)
import Control.Exception (block, catch, throw, unblock)
import Prelude hiding (catch) -- use Control.Exception's version

modifyMVar :: MVar a -> (a -> IO (a,b)) -> IO b
modifyMVar m io =
  block $ do
    a <- takeMVar m
    (b,r) <- unblock (io a) `catch` \e ->
             putMVar m a >> throw e
    putMVar m b
    return r
```

</div>

You should easily be able to adapt this to your particular needs,
whether you\'re working with network connections, database handles, or
data managed by a C library.

### Finding the status of a thread

Our `getStatus` function tells us the current state of a thread. If the
thread is no longer managed (or was never managed in the first place),
it returns `Nothing`.

<div>

[NiceFork.hs]{.label}

``` {.haskell}
getStatus (Mgr mgr) tid =
  modifyMVar mgr $ \m ->
    case M.lookup tid m of
      Nothing -> return (m, Nothing)
      Just st -> tryTakeMVar st >>= \mst -> case mst of
                   Nothing -> return (m, Just Running)
                   Just sth -> return (M.delete tid m, Just sth)
```

</div>

If the thread is still running, it returns `Just Running`. Otherwise, it
indicates why the thread terminated, *and* stops managing the thread.

If the `tryTakeMVar` function finds that the `MVar` is empty, it returns
`Nothing` immediately instead of blocking.

``` {.screen}
ghci> :t tryTakeMVar
tryTakeMVar :: MVar a -> IO (Maybe a)
```

Otherwise, it extracts the value from the `MVar` as usual.

The `waitFor` function behaves similarly, but instead of returning
immediately, it blocks until the given thread terminates before
returning.

<div>

[NiceFork.hs]{.label}

``` {.haskell}
waitFor (Mgr mgr) tid = do
  maybeDone <- modifyMVar mgr $ \m ->
    return $ case M.updateLookupWithKey (\_ _ -> Nothing) tid m of
      (Nothing, _) -> (m, Nothing)
      (done, m') -> (m', done)
  case maybeDone of
    Nothing -> return Nothing
    Just st -> Just `fmap` takeMVar st
```

</div>

It first extracts the `MVar` that holds the thread\'s state, if it
exists. The `Map` type\'s `updateLookupWithKey` function is useful: it
combines looking up a key with modifying or removing the value.

``` {.screen}
ghci> :m +Data.Map
ghci> :t updateLookupWithKey
updateLookupWithKey :: (Ord k) =>
                       (k -> a -> Maybe a) -> k -> Map k a -> (Maybe a, Map k a)
```

In this case, we want to always remove the `MVar` holding the thread\'s
state if it is present, so that our thread manager will no longer be
managing the thread. If there was a value to extract, we take the
thread\'s exit status from the `MVar` and return it.

Our final useful function simply waits for all currently managed threads
to complete, and ignores their exit statuses.

<div>

[NiceFork.hs]{.label}

``` {.haskell}
waitAll (Mgr mgr) = modifyMVar mgr elems >>= mapM_ takeMVar
    where elems m = return (M.empty, M.elems m)
```

</div>

### Writing tighter code

Our definition of `waitFor` above is a little unsatisfactory, because
we\'re performing more or less the same case analysis in two places:
inside the function called by `modifyMVar`, and again on its return
value.

Sure enough, we can apply a function that we came across earlier to
eliminate this duplication. The function in question is `join`, from the
`Control.Monad` module.

``` {.screen}
ghci> :m +Control.Monad
ghci> :t join
join :: (Monad m) => m (m a) -> m a
```

The trick here is to see that we can get rid of the second `case`
expression by having the first one return the `IO` action that we should
perform once we return from `modifyMVar`. We\'ll use `join` to execute
the action.

<div>

[NiceFork.hs]{.label}

``` {.haskell}
waitFor2 (Mgr mgr) tid =
  join . modifyMVar mgr $ \m ->
    return $ case M.updateLookupWithKey (\_ _ -> Nothing) tid m of
      (Nothing, _) -> (m, return Nothing)
      (Just st, m') -> (m', Just `fmap` takeMVar st)
```

</div>

This is an interesting idea: we can create a monadic function or action
in pure code, then pass it around until we end up in a monad where we
can use it. This can be a nimble way to write code, once we develop an
eye for when it makes sense.

Communicating over channels
---------------------------

For one-shot communications between threads, an `MVar` is perfectly
good. Another type, `Chan`, provides a one-way communication channel.
Here is a simple example of its use.

<div>

[Chan.hs]{.label}

``` {.haskell}
import Control.Concurrent
import Control.Concurrent.Chan

chanExample = do
  ch <- newChan
  forkIO $ do
    writeChan ch "hello world"
    writeChan ch "now i quit"
  readChan ch >>= print
  readChan ch >>= print
```

</div>

If a `Chan` is empty, `readChan` blocks until there is a value to read.
The `writeChan` function never blocks: it writes a new value into a
`Chan` immediately.

Useful things to know about
---------------------------

### `MVar` and `Chan` are non-strict

Like most Haskell container types, both `MVar` and `Chan` are
non-strict: neither evaluates its contents. We mention this not because
it\'s a problem, but because it\'s a common blind spot: people tend to
assume that these types are strict, perhaps because they\'re used in the
`IO` monad.

As for other container types, the upshot of a mistaken guess about the
strictness of an `MVar` or `Chan` type is often a space or performance
leak. Here\'s a plausible scenario to consider.

We fork off a thread to perform some expensive computation on another
core.

<div>

[Expensive.hs]{.label}

``` {.haskell}
import Control.Concurrent

notQuiteRight = do
  mv <- newEmptyMVar
  forkIO $ expensiveComputation_stricter mv
  someOtherActivity
  result <- takeMVar mv
  print result
```

</div>

It *seems* to do something, and puts its result back into the `MVar`.

<div>

[Expensive.hs]{.label}

``` {.haskell}
expensiveComputation mv = do
  let a = "this is "
      b = "not really "
      c = "all that expensive"
  putMVar mv (a ++ b ++ c)
```

</div>

When we take the result from the `MVar` in the parent thread and attempt
to do something with it, our thread starts computing furiously, because
we never forced the computation to actually occur in the other thread!

As usual, the solution is straightforward, once we know there\'s a
potential for a problem: we add strictness to the forked thread, to
ensure that the computation occurs there. This strictness is best added
in one place, to avoid the possibility that we might forget to add it.

<div>

[ModifyMVarStrict.hs]{.label}

``` {.haskell}
{-# LANGUAGE BangPatterns #-}

import Control.Concurrent (MVar, putMVar, takeMVar)
import Control.Exception (block, catch, throw, unblock)
import Prelude hiding (catch) -- use Control.Exception's version

modifyMVar_strict :: MVar a -> (a -> IO a) -> IO ()
modifyMVar_strict m io = block $ do
  a <- takeMVar m
  !b <- unblock (io a) `catch` \e ->
        putMVar m a >> throw e
  putMVar m b
```

</div>

::: {.TIP}
It\'s always worth checking Hackage

In the Hackage package database, you will find a library,
`strict-concurrency`, that provides strict versions of the `MVar` and
`Chan` types.
:::

The `!` pattern above is simple to use, but it is not always sufficient
to ensure that our data is evaluated. For a more complete approach, see
[the section called \"Separating algorithm from
evaluation\"](24-concurrent-and-multicore-programming.org::*Separating algorithm from evaluation)
below.

### Chan is unbounded

Because `writeChan` always succeeds immediately, there is a potential
risk to using a `Chan`. If one thread writes to a `Chan` more often than
another thread reads from it, the `Chan` will grow in an unchecked
manner: unread messages will pile up as the reader falls further and
further behind.

Shared-state concurrency is still hard
--------------------------------------

Although Haskell has different primitives for sharing data between
threads than other languages, it still suffers from the same fundamental
problem: writing correct concurrent programs is fiendishly difficult.
Indeed, several pitfalls of concurrent programming in other languages
apply equally to Haskell. Two of the better known problems are
*deadlock* and *starvation*.

### Deadlock

In a *deadlock* situation, two or more threads get stuck forever in a
clash over access to shared resources. One classic way to make a
multithreaded program deadlock is to forget the order in which we must
acquire locks. This kind of bug is so common, it has a name: *lock order
inversion*. While Haskell doesn\'t provide locks, the `MVar` type is
prone to the order inversion problem. Here\'s a simple example.

<div>

[LockHierarchy.hs]{.label}

``` {.haskell}
import Control.Concurrent

nestedModification outer inner = do
  modifyMVar_ outer $ \x -> do
    yield  -- force this thread to temporarily yield the CPU
    modifyMVar_ inner $ \y -> return (y + 1)
    return (x + 1)
  putStrLn "done"

main = do
  a <- newMVar 1
  b <- newMVar 2
  forkIO $ nestedModification a b
  forkIO $ nestedModification b a
```

</div>

If we run this in `ghci`, it will usually---but not always---print
nothing, indicating that both threads have gotten stuck.

The problem with the `nestedModification` function is easy to spot. In
the first thread, we take the `MVar` `a`, then `b`. In the second, we
take `b`, then `a`. If the first thread succeeds in taking `a` and the
second takes `b`, both threads will block: each tries to take an `MVar`
that the other has already emptied, so neither can make progress.

Across languages, the usual way to solve an order inversion problem is
to always follow a consistent order when acquiring resources. Since this
approach requires manual adherence to a coding convention, it is easy to
miss in practice.

To make matters more complicated, these kinds of inversion problems can
be difficult to spot in real code. The taking of `MVars` is often spread
across several functions in different files, making visual inspection
more tricky. Worse, these problems are often *intermittent*, which makes
them tough to even reproduce, never mind isolate and fix.

### Starvation

Concurrent software is also prone to *starvation*, in which one thread
\"hogs\" a shared resource, preventing another from using it. It\'s easy
to imagine how this might occur: one thread calls `modifyMVar` with a
body that executes for 100 milliseconds, while another calls
`modifyMVar` on the same `MVar` with a body that executes for 1
millisecond. The second thread cannot make progress until the first puts
a value back into the `MVar`.

The non-strict nature of the `MVar` type can either exacerbate or cause
a starvation problem. If we put a thunk into an `MVar` that will be
expensive to evaluate, and take it out of the `MVar` in a thread that
otherwise looks like it *ought* to be cheap, that thread could suddenly
become computationally expensive if it has to evaluate the thunk. This
makes the advice we gave in [the section called \"MVar and Chan are
non-strict\"](24-concurrent-and-multicore-programming.org::*MVar and Chan are non-strict)

### Is there any hope?

Fortunately, the APIs for concurrency that we have covered here are by
no means the end of the story. A more recent addition to Haskell,
Software Transactional Memory, is both easier and safer to work with. We
will discuss it in chapter [Chapter 28, *Software transactional
memory*](28-software-transactional-memory.org).

Exercises
---------

1.  The `Chan` type is implemented using `MVar~s. Use ~MVar~s to
      develop a ~BoundedChan` library.
2.  Your `newBoundedChan` function should accept an `Int` parameter,
    limiting the number of unread items that can be present in a
    `BoundedChan` at once.
3.  If this limit is hit, a call to your `writeBoundedChan` function
    must block until a reader uses `readBoundedChan` to consume a value.
4.  Although we\'ve already mentioned the existence of the
    strict-concurrency package in the Hackage repository, try developing
    your own, as a wrapper around the built-in `MVar` type. Following
    classic Haskell practice, make your library type safe, so that users
    cannot accidentally mix uses of strict and non-strict `MVars`.

Using multiple cores with GHC
-----------------------------

By default, GHC generates programs that use just one core, even when we
write explicitly concurrent code. To use multiple cores, we must
explicitly choose to do so. We make this choice at *link time*, when we
are generating an executable program.

-   The \"non-threaded\" runtime library runs all Haskell threads in a
    single operating system thread. This runtime is highly efficient for
    creating threads and passing data around in `MVars`.
-   The \"threaded\" runtime library uses multiple operating system
    threads to run Haskell threads. It has somewhat more overhead for
    creating threads and using `MVars`.

If we pass the `-threaded` option to the compiler, it will link our
program against the threaded runtime library. We do not need to use
`-threaded` when we are compiling libraries or source files, only when
we are finally generating an executable.

Even when we select the threaded runtime for our program, it will still
default to using only one core when we run it. We must explicitly tell
the runtime how many cores to use.

### Runtime options

We can pass options to GHC\'s runtime system on the command line of our
program. Before handing control to our code, the runtime scans the
program\'s arguments for the special command line option `+RTS`. It
interprets everything that follows, until the special option `-RTS`, as
an option for the runtime system, not our program. It hides all of these
options from our code. When we use the `System.Environment` module\'s
`getArgs` function to obtain our command line arguments, we will not
find any runtime options in the list.

The threaded runtime accepts an option `-N`[^2]. This takes one
argument, which specifies the number of cores that GHC\'s runtime system
should use. The option parser is picky: there must be no spaces between
`-N` and the number that follows it. The option `-N4` is acceptable, but
`-N 4` is not.

### Finding the number of available cores from Haskell

The module `GHC.Conc` exports a variable, `numCapabilities`, that tells
us how many cores the runtime system has been given with the `-N` RTS
option.

<div>

[NumCapabilities.hs]{.label}

``` {.haskell}
import GHC.Conc (numCapabilities)
import System.Environment (getArgs)

main = do
  args <- getArgs
  putStrLn $ "command line arguments: " ++ show args
  putStrLn $ "number of cores: " ++ show numCapabilities
```

</div>

If we compile and run the above program, we can see that the options to
the runtime system are not visible to the program, but that it can see
how many cores it can run on.

``` {.screen}
$ ghc -c NumCapabilities.hs
$ ghc -threaded -o NumCapabilities NumCapabilities.o
$ ./NumCapabilities +RTS -N4 -RTS foo
command line arguments: ["foo"]
number of cores: 4
```

### Choosing the right runtime

The decision of which runtime to use is not completely clear cut. While
the threaded runtime can use multiple cores, it has a cost: threads and
sharing data between them are more expensive than with the non-threaded
runtime.

Furthermore, the garbage collector used by GHC as of version 6.8.3 is
single threaded: it pauses all other threads while it runs, and executes
on one core. This limits the performance improvement we can hope to see
from using multiple cores[^3].

In many real world concurrent programs, an individual thread will spend
most of its time waiting for a network request or response. In these
cases, if a single Haskell program serves tens of thousands of
concurrent clients, the lower overhead of the non-threaded runtime may
be helpful. For example, instead of having a single server program use
the threaded runtime on four cores, we might see better performance if
we design our server so that we can run four copies of it
simultaneously, and use the non-threaded runtime.

Our purpose here is not to dissuade you from using the threaded runtime.
It is not much more expensive than the non-threaded runtime: threads
remain amazingly cheap compared to the runtimes of most other
programming languages. We merely want to make it clear that switching to
the threaded runtime will not necessarily result in an automatic win.

Parallel programming in Haskell
-------------------------------

We will now switch our focus to parallel programming. For many
computationally expensive problems, we could calculate a result more
quickly if we could divide up the solution, and evaluate it on many
cores at once. Computers with multiple cores are already ubiquitous, but
few programs can take advantage of the computing power of even a modern
laptop.

In large part, this is because parallel programming is traditionally
seen as very difficult. In a typical programming language, we would use
the same libraries and constructs that we apply to concurrent programs
to develop a parallel program. This forces us to contend with the
familiar problems of deadlocks, race conditions, starvation, and sheer
complexity.

While we could certainly use Haskell\'s concurrency features to develop
parallel code, there is a much simpler approach available to us. We can
take a normal Haskell function, apply a few simple transformations to
it, and have it evaluated in parallel.

### Normal form and head normal form

The familiar `seq` function evaluates an expression to what we call
*head normal form* (abbreviated HNF). It stops once it reaches the
outermost constructor (the \"head\"). This is distinct from *normal
form* (NF), in which an expression is completely evaluated.

You will also hear Haskell programmers refer to *weak* head normal form
(WHNF). For normal data, weak head normal form is the same as head
normal form. The difference only arises for functions, and is too
abstruse to concern us here.

### Sequential sorting

Here is a normal Haskell function that sorts a list using a
divide-and-conquer approach.

<div>

[Sorting.hs]{.label}

``` {.haskell}
sort :: (Ord a) => [a] -> [a]
sort (x:xs) = lesser ++ x:greater
    where lesser  = sort [y | y <- xs, y <  x]
          greater = sort [y | y <- xs, y >= x]
sort _ = []
```

</div>

This function is inspired by the well-known Quicksort algorithm, and it
is a classic among Haskell programmers: it is often presented as a
one-liner early in a Haskell tutorial, to tease the reader with an
example of Haskell\'s expressiveness. Here, we\'ve split the code over a
few lines, to make it easier to compare the serial and parallel
versions.

Here is a very brief description of how `sort` operates.

1.  It chooses an element from the list. This is called the *pivot*. Any
    element would do as the pivot; the first is merely the easiest to
    pattern match on.
2.  It creates a sublist of all elements less than the pivot, and
    recursively sorts them.
3.  It creates a sublist of all elements greater than or equal to the
    pivot, and recursively sorts them.
4.  It appends the two sorted sublists.

### Transforming our code into parallel code

The parallel version of the function is only a little more complicated
than the initial version.

<div>

[Sorting.hs]{.label}

``` {.haskell}
module Sorting where

import Control.Parallel (par, pseq)

parSort :: (Ord a) => [a] -> [a]
parSort (x:xs)    = force greater `par` (force lesser `pseq`
                                         (lesser ++ x:greater))
    where lesser  = parSort [y | y <- xs, y <  x]
          greater = parSort [y | y <- xs, y >= x]
parSort _         = []
```

</div>

We have barely perturbed the code: all we have added are three
functions, `par`, `pseq`, and `force`.

The `par` function is provided by the `Control.Parallel` module. It
serves a similar purpose to `seq`: it evaluates its left argument to
weak head normal form, and returns its right. As its name suggests,
`par` can evaluate its left argument in parallel with whatever other
evaluations are occurring.

As for `pseq`, it is similar to `seq`: it evaluates the expression on
the left to WHNF before returning the expression on the right. The
difference between the two is subtle, but important for parallel
programs: the compiler does not *promise* to evaluate the left argument
of `seq` if it can see that evaluating the right argument first would
improve performance. This flexibility is fine for a program executing on
one core, but it is not strong enough for code running on multiple
cores. In contrast, the compiler *guarantees* that `pseq` will evaluate
its left argument before its right.

These changes to our code are remarkable for all the things we have
*not* needed to say.

-   How many cores to use.
-   What threads do to communicate with each other.
-   How to divide up work among the available cores.
-   Which data are shared between threads, and which are private.
-   How to determine when all the participants are finished.

### Knowing what to evaluate in parallel

The key to getting decent performance out of parallel Haskell code is to
find meaningful chunks of work to perform in parallel. Non-strict
evaluation can get in the way of this, which is why we use the `force`
function in our parallel sort. To best explain what the `force` function
is for, we will first look at a mistaken example.

<div>

[Sorting.hs]{.label}

``` {.haskell}
sillySort (x:xs) = greater `par` (lesser `pseq`
                                  (lesser ++ x:greater))
    where lesser   = sillySort [y | y <- xs, y <  x]
          greater  = sillySort [y | y <- xs, y >= x]
sillySort _        = []
```

</div>

Take a look at the small changes in each use of `par`. Instead of
`force lesser` and `force greater`, here we evaluate `lesser` and
`greater`.

Remember that evaluation to WHNF only computes enough of an expression
to see its *outermost* constructor. In this mistaken example, we
evaluate each sorted sublist to WHNF. Since the outermost constructor in
each case is just a single list constructor, we are in fact only forcing
the evaluation of the first element of each sorted sublist! Every other
element of each list remains unevaluated. In other words, we do almost
no useful work in parallel: our `sillySort` is nearly completely
sequential.

We avoid this with our `force` function by forcing the entire spine of a
list to be evaluated before we give back a constructor.

<div>

[Sorting.hs]{.label}

``` {.haskell}
force :: [a] -> ()
force xs = go xs `pseq` ()
    where go (_:xs) = go xs
          go [] = 1
```

</div>

Notice that we don\'t care what\'s in the list; we walk down its spine
to the end, then use `pseq` once. There is clearly no magic involved
here: we are just using our usual understanding of Haskell\'s evaluation
model. And because we will be using `force` on the left hand side of
`par` or `pseq`, we don\'t need to return a meaningful value.

Of course, in many cases we will need to force the evaluation of
individual elements of the list, too. Below, we will discuss a type
class-based solution to this problem.

### What promises does par make?

The `par` function does not actually promise to evaluate an expression
in parallel with another. Instead, it undertakes to do so if it \"makes
sense\". This wishy-washy non-promise is actually more useful than a
guarantee to always evaluate an expression in parallel. It gives the
runtime system the freedom to act intelligently when it encounters a use
of `par`.

For instance, the runtime could decide that an expression is too cheap
to be worth evaluating in parallel. Or it might notice that all cores
are currently busy, so that \"sparking\" a new parallel evaluation will
lead to there being more runnable threads than there are cores available
to execute them.

This lax specification in turn affects how we write parallel code. Since
`par` may be somewhat intelligent at runtime, we can use it almost
wherever we like, on the assumption that performance will not be bogged
down by threads contending for busy cores.

### Running our code, and measuring performance

To try our code out, let\'s save `sort`, `parSort`, and `parSort2` to a
module named `Sorting.hs`. We create a small driver program that we can
use to time the performance of one of those sorting function.

<div>

[SortMain.hs]{.label}

``` {.haskell}
module Main where

import Data.Time.Clock (diffUTCTime, getCurrentTime)
import System.Environment (getArgs)
import System.Random (StdGen, getStdGen, randoms)

import Sorting

-- testFunction = sort
-- testFunction = seqSort
testFunction = parSort
-- testFunction = parSort2 2

randomInts :: Int -> StdGen -> [Int]
randomInts k g = let result = take k (randoms g)
                 in force result `seq` result

main = do
  args <- getArgs
  let count | null args = 500000
            | otherwise = read (head args)
  input <- randomInts count `fmap` getStdGen
  putStrLn $ "We have " ++ show (length input) ++ " elements to sort."
  start <- getCurrentTime
  let sorted = testFunction input
  putStrLn $ "Sorted all " ++ show (length sorted) ++ " elements."
  end <- getCurrentTime
  putStrLn $ show (end `diffUTCTime` start) ++ " elapsed."
```

</div>

For simplicity, we choose the sorting function to benchmark at
compilation time, via the `testFunction` variable.

Our program accepts a single optional command line argument, the length
of the random list to generate.

Non-strict evaluation can turn performance measurement and analysis into
something of a minefield. Here are some potential problems that we
specifically work to avoid in our driver program.

-   *Measuring several things, when we think we are looking at just
    one.* Haskell\'s default pseudorandom number generator (PRNG) is
    slow, and the `randoms` function generates random numbers on demand.

    Before we record our starting time, we force every element of the
    input list to be evaluated, and we print the length of the list:
    this ensures that we create all of the random numbers that we will
    need in advance.

    If we were to omit this step, we would interleave the generation of
    random numbers with attempts to work with them in parallel. We would
    thus be measuring both the cost of sorting the numbers and, less
    obviously, the cost of generating them.

-   *Invisible data dependencies.* When we generate the list of random
    numbers, simply printing the length of the list would not perform
    enough evaluation. This would evaluate the *spine* of the list, but
    not its elements. The actual random numbers would not be evaluated
    until the sort compares them.

    This can have serious consequences for performance. The value of a
    random number depends on the value of the preceding random number in
    the list, but we have scattered the list elements randomly among our
    processor cores. If we did not evaluate the list elements prior to
    sorting, we would suffer a terrible \"ping pong\" effect: not only
    would evaluation bounce from one core to another, performance would
    suffer.

    Try snipping out the application of `force` from the body of `main`
    above: you should find that the parallel code can easily end up
    three times *slower* than the non-parallel code.

-   *Benchmarking a thunk, when we believe that the code is performing
    meaningful work.* To force the sort to take place, we print the
    length of the result list before we record the ending time. Without
    `putStrLn` demanding the length of the list in order to print it,
    the sort would not occur at all.

When we build the program, we enable optimization and GHC\'s threaded
runtime.

``` {.screen}
$ ghc -threaded -O2 --make SortMain
[1 of 2] Compiling Sorting          ( Sorting.hs, Sorting.o )
[2 of 2] Compiling Main             ( SortMain.hs, SortMain.o )
Linking SortMain ...
```

When we run the program, we must tell GHC\'s runtime how many cores to
use. Initially, we try the original `sort`, to establish a performance
baseline.

``` {.screen}
$ ./Sorting +RTS -N1 -RTS 700000
We have 700000 elements to sort.
Sorted all 700000 elements.
3.178941s elapsed.
```

Enabling a second core ought to have no effect on performance.

``` {.screen}
$ ./Sorting +RTS -N2 -RTS 700000
We have 700000 elements to sort.
Sorted all 700000 elements.
3.259869s elapsed.
```

If we recompile and test the performance of `parSort`, the results are
less than stellar.

``` {.screen}
$ ./Sorting +RTS -N1 -RTS 700000
We have 700000 elements to sort.
Sorted all 700000 elements.
3.915818s elapsed.
$ ./Sorting +RTS -N2 -RTS 700000
We have 700000 elements to sort.
Sorted all 700000 elements.
4.029781s elapsed.
```

We have gained nothing in performance. It seems that this could be due
to one of two factors: either `par` is intrinsically expensive, or we
are using it too much. To help us to distinguish between the two
possibilities, here is a sort is identical to `parSort`, but it uses
`pseq` instead of `par`.

<div>

[Sorting.hs]{.label}

``` {.haskell}
seqSort :: (Ord a) => [a] -> [a]
seqSort (x:xs) = lesser `pseq` (greater `pseq`
                                (lesser ++ x:greater))
    where lesser  = seqSort [y | y <- xs, y <  x]
          greater = seqSort [y | y <- xs, y >= x]
seqSort _ = []
```

</div>

We also drop the use of `force`, so compared to our original `sort`, we
should only be measuring the cost of using `pseq`. What effect does
`pseq` alone have on performance?

``` {.screen}
$ ./Sorting +RTS -N1 -RTS 700000
We have 700000 elements to sort.
Sorted all 700000 elements.
3.848295s elapsed.
```

This suggests that `par` and `pseq` have similar costs. What can we do
to improve performance?

### Tuning for performance

In our `parSort`, we perform twice as many applications of `par` as
there are elements to sort. While `par` is *cheap*, as we have seen, it
is not *free*. When we recursively apply `parSort`, we eventually apply
`par` to individual list elements. At this fine granularity, the cost of
using `par` outweighs any possible usefulness. To reduce this effect, we
switch to our non-parallel `sort` after passing some threshold.

<div>

[Sorting.hs]{.label}

``` {.haskell}
parSort2 :: (Ord a) => Int -> [a] -> [a]
parSort2 d list@(x:xs)
  | d <= 0     = sort list
  | otherwise = force greater `par` (force lesser `pseq`
                                     (lesser ++ x:greater))
      where lesser      = parSort2 d' [y | y <- xs, y <  x]
            greater     = parSort2 d' [y | y <- xs, y >= x]
            d' = d - 1
parSort2 _ _            = []
```

</div>

Here, we stop recursing and sparking new parallel evaluations at a
controllable depth. If we knew the size of the data we were dealing
with, we could stop subdividing and switch to the non-parallel code once
we reached a sufficiently small amount of remaining work.

``` {.screen}
$ ./Sorting +RTS -N2 -RTS 700000
We have 700000 elements to sort.
Sorted all 700000 elements.
2.947872s elapsed.
```

On a dual core system, this gives us roughly a 25% speedup. This is not
a huge number, but consider the number of changes we had to make in
return for this performance improvement: just a few annotations.

This sorting function is particularly resistant to good parallel
performance. The amount of memory allocation it performs forces the
garbage collector to run frequently. We can see the effect by running
our program with the `-sstderr` RTS option, which prints garbage
collection statistics to the screen. This indicates that our program
spends roughly 40% of its time collecting garbage. Since the garbage
collector in GHC 6.8 stops all threads and runs on a single core, it
acts as a bottleneck.

You can expect more impressive performance improvements from less
allocation-heavy code when you use `par` annotations. We have seen some
simple numerical benchmarks run 1.8 times faster on a dual core system
than with a single core. As we write this book, a parallel garbage
collector is under development for GHC, which should help considerably
with the performance of allocation-heavy code on multicore systems.

::: {.WARNING}
Beware a GC bug in GHC 6.8.2

The garbage collector in release 6.8.2 of GHC has a bug that can cause
programs using `par` to crash. If you want to use `par` and you are
using 6.8.2, we suggest upgrading to at least 6.8.3.
:::

### Exercises

1.  It can be difficult to determine when to switch from `parSort2` to
    sort. An alternative approach to the one we outline above would be
    to decide based on the length of a sublist. Rewrite `parList2` so
    that it switches to sort if the list contains more than some number
    of elements.
2.  Measure the performance of the length-based approach, and compare
    with the depth approach. Which gives better performance results?

Parallel strategies and `MapReduce`
-----------------------------------

Within the programming community, one of the most famous software
systems to credit functional programming for inspiration is Google\'s
MapReduce infrastructure for parallel processing of bulk data.

We can easily construct a greatly simplified, but still useful, Haskell
equivalent. To focus our attention, we will look at the processing of
web server log files, which tend to be both huge and plentiful[^4].

As an example, here is a log entry for a page visit recorded by the
Apache web server. The entry originally filled one line; we have split
it across several lines to fit.

``` {.example}
201.49.94.87 - - [08/Jun/2008:07:04:20 -0500] "GET / HTTP/1.1"
200 2097 "http://en.wikipedia.org/wiki/Mercurial_(software)"
"Mozilla/5.0 (Windows; U; Windows XP 5.1; en-GB; rv:1.8.1.12)
Gecko/20080201 Firefox/2.0.0.12" 0 hgbook.red-bean.com
```

While we could create a straightforward implementation without much
effort, we will resist the temptation to dive in. If we think about
solving a *class* of problems instead of a single one, we may end up
with more widely applicable code.

When we develop a parallel program, we are always faced with a few \"bad
penny\" problems, which turn up no matter what the underlying
programming language is.

-   Our algorithm quickly becomes obscured by the details of
    partitioning and communication. This makes it difficult to
    understand code, which in turn makes modifying it risky.
-   Choosing a \"grain size\"---the smallest unit of work parceled out
    to a core---can be difficult. If the grain size is too small, cores
    spend so much of their time on book-keeping that a parallel program
    can easily become slower than a serial counterpart. If the grain
    size is too large, some cores may lie idle due to poor load
    balancing.

### Separating algorithm from evaluation

In parallel Haskell code, the clutter that would arise from
communication code in a traditional language is replaced with the
clutter of `par` and `pseq` annotations. As an example, this function
operates similarly to `map`, but evaluates each element to weak head
normal form (WHNF) in parallel as it goes.

<div>

[ParMap.hs]{.label}

``` {.haskell}
import Control.Parallel (par)

parallelMap :: (a -> b) -> [a] -> [b]
parallelMap f (x:xs) = let r = f x
                       in r `par` r : parallelMap f xs
parallelMap _ _      = []
```

</div>

The type `b` might be a list, or some other type for which evaluation to
WHNF doesn\'t do a useful amount of work. We\'d prefer not to have to
write a special `parallelMap` for lists, and for every other type that
needs special handling.

To address this problem, we will begin by considering a simpler problem:
how to force a value to be evaluated. Here is a function that forces
every element of a list to be evaluated to WHNF.

<div>

[ParMap.hs]{.label}

``` {.haskell}
forceList :: [a] -> ()
forceList (x:xs) = x `pseq` forceList xs
forceList _      = ()
```

</div>

Our function performs no computation on the list. (In fact, from
examining its type signature, we can tell that it *cannot* perform any
computation, since it knows nothing about the elements of the list.) Its
only purpose is to ensure that the spine of the list is evaluated to
head normal form. The only place that it makes any sense to apply this
function is in the first argument of `seq` or `par`, for example as
follows.

<div>

[ParMap.hs]{.label}

``` {.haskell}
stricterMap :: (a -> b) -> [a] -> [b]
stricterMap f xs = forceList xs `seq` map f xs
```

</div>

This still leaves us with the elements of the list evaluated only to
WHNF. We address this by adding a function as parameter that can force
an element to be evaluated more deeply.

<div>

[ParMap.hs]{.label}

``` {.haskell}
forceListAndElts :: (a -> ()) -> [a] -> ()
forceListAndElts forceElt (x:xs) =
    forceElt x `seq` forceListAndElts forceElt xs
forceListAndElts _        _      = ()
```

</div>

The `Control.Parallel.Strategies` module generalizes this idea into
something we can use as a library. It introduces the idea of an
*evaluation strategy*.

<div>

[Strat.hs]{.label}

``` {.haskell}
type Done = ()

type Strategy a = a -> Done
```

</div>

An evaluation strategy performs no computation; it simply ensures that a
value is evaluated to some extent. The simplest strategy is named `r0`,
and does nothing at all.

<div>

[Strat.hs]{.label}

``` {.haskell}
r0 :: Strategy a
r0 _ = ()
```

</div>

Next is `whnf`, which evaluates a value to weak head normal form.

<div>

[Strat.hs]{.label}

``` {.haskell}
rwhnf :: Strategy a
rwhnf x = x `seq` ()
```

</div>

To evaluate a value to normal form, the module provides a type class
with a method named `rnf`.

<div>

[Strat.hs]{.label}

``` {.haskell}
class NFData a where
  rnf :: Strategy a
  rnf = rwhnf
```

</div>

::: {.TIP}
Remembering those names

If the names of these functions and types are not sticking in your head,
look at them as acronyms. The name `rwhnf` expands to \"reduce to weak
head normal form\"; NFData becomes \"normal form data\"; and so on.
:::

For the basic types, such as `Int`, weak head normal form and normal
form are the same thing, which is why the `NFData` type class uses
`rwhnf` as the default implementation of `rnf`. For many common types,
the `Control.Parallel.Strategies` module provides instances of `NFData`.

<div>

[Strat.hs]{.label}

``` {.haskell}
instance NFData Char
instance NFData Int

instance NFData a => NFData (Maybe a) where
    rnf Nothing  = ()
    rnf (Just x) = rnf x

{- ... and so on ... -}
```

</div>

From these examples, it should be clear how you might write an `NFData`
instance for a type of your own. Your implementation of `rnf` must
handle every constructor, and apply `rnf` to every field of a
constructor.

### Separating algorithm from strategy

From these strategy building blocks, we can construct more elaborate
strategies. Many are already provided by `Control.Parallel.Strategies`.
For instance, `parList` applies an evaluation strategy in parallel to
every element of a list.

<div>

[Strat.hs]{.label}

``` {.haskell}
parList :: Strategy a -> Strategy [a]
parList strat []     = ()
parList strat (x:xs) = strat x `par` (parList strat xs)
```

</div>

The module uses this to define a parallel `map` function.

<div>

[Strat.hs]{.label}

``` {.haskell}
parMap :: Strategy b -> (a -> b) -> [a] -> [b]
parMap strat f xs = map f xs `using` parList strat
```

</div>

This is where the code becomes interesting. On the left of `using`, we
have a normal application of `map`. On the right, we have an evaluation
strategy. The `using` combinator tells us how to apply a strategy to a
value, allowing us to keep the code separate from how we plan to
evaluate it.

<div>

[Strat.hs]{.label}

``` {.haskell}
using :: a -> Strategy a -> a
using x s = s x `seq` x
```

</div>

The `Control.Parallel.Strategies` module provides many other functions
that provide fine control over evaluation. For instance, `parZipWith`
that applies `zipWith` in parallel, using an evaluation strategy.

<div>

[Strat.hs]{.label}

``` {.haskell}
vectorSum' :: (NFData a, Num a) => [a] -> [a] -> [a]
vectorSum' = parZipWith rnf (+)
```

</div>

### Writing a simple MapReduce definition

We can quickly suggest a type for a `mapReduce` function by considering
what it must do. We need a *map* component, to which we will give the
usual type `a -> b`. And we need a *reduce*; this term is a synonym for
*fold*. Rather than commit ourselves to using a specific kind of fold,
we\'ll use a more general type, `[b] -> c`. This type lets us use a left
or right fold, so we can choose the one that suits our data and
processing needs.

If we plug these types together, the complete type looks like this.

<div>

[MapReduce.hs]{.label}

``` {.haskell}
simpleMapReduce
    :: (a -> b)      -- map function
    -> ([b] -> c)    -- reduce function
    -> [a]           -- list to map over
    -> c
```

</div>

The code that goes with the type is extremely simple.

<div>

[MapReduce.hs]{.label}

``` {.haskell}
simpleMapReduce mapFunc reduceFunc = reduceFunc . map mapFunc
```

</div>

### MapReduce and strategies

Our definition of `simpleMapReduce` is too simple to really be
interesting. To make it useful, we want to be able to specify that some
of the work should occur in parallel. We\'ll achieve this using
strategies, passing in a strategy for the map phase and one for the
reduction phase.

<div>

[MapReduce.hs]{.label}

``` {.haskell}
mapReduce
    :: Strategy b    -- evaluation strategy for mapping
    -> (a -> b)      -- map function
    -> Strategy c    -- evaluation strategy for reduction
    -> ([b] -> c)    -- reduce function
    -> [a]           -- list to map over
    -> c
```

</div>

Both the type and the body of the function must grow a little in size to
accommodate the strategy parameters.

<div>

[MapReduce.hs]{.label}

``` {.haskell}
mapReduce mapStrat mapFunc reduceStrat reduceFunc input =
    mapResult `pseq` reduceResult
  where mapResult    = parMap mapStrat mapFunc input
        reduceResult = reduceFunc mapResult `using` reduceStrat
```

</div>

### Sizing work appropriately

To achieve decent performance, we must ensure that the work that we do
per application of `par` substantially outweighs its book-keeping costs.
If we are processing a huge file, splitting it on line boundaries gives
us far too little work compared to overhead.

We will develop a way to process a file in larger chunks in a later
section. What should those chunks consist of? Because a web server log
file ought to contain only ASCII text, we will see excellent performance
with a lazy `ByteString`: this type is highly efficient, and consumes
little memory when we stream it from a file.

<div>

[LineChunks.hs]{.label}

``` {.haskell}
module LineChunks
    (
      chunkedReadWith
    ) where

import Control.Exception (bracket, finally)
import Control.Monad (forM, liftM)
import Control.Parallel.Strategies (NFData, rnf)
import Data.Int (Int64)
import qualified Data.ByteString.Lazy.Char8 as LB
import GHC.Conc (numCapabilities)
import System.IO

data ChunkSpec = CS {
      chunkOffset :: !Int64
    , chunkLength :: !Int64
    } deriving (Eq, Show)

withChunks :: (NFData a) =>
              (FilePath -> IO [ChunkSpec])
           -> ([LB.ByteString] -> a)
           -> FilePath
           -> IO a
withChunks chunkFunc process path = do
  (chunks, handles) <- chunkedRead chunkFunc path
  let r = process chunks
  (rnf r `seq` return r) `finally` mapM_ hClose handles

chunkedReadWith :: (NFData a) =>
                   ([LB.ByteString] -> a) -> FilePath -> IO a
chunkedReadWith func path =
    withChunks (lineChunks (numCapabilities * 4)) func path
```

</div>

We consume each chunk in parallel, taking careful advantage of lazy I/O
to ensure that we can stream these chunks safely.

1.  Mitigating the risks of lazy I/O

    Lazy I/O poses a few well known hazards that we would like to avoid.

    -   We may invisibly keep a file handle open for longer than
        necessary, by not forcing the computation that pulls data from
        it to be evaluated. Since an operating system will typically
        place a small, fixed limit on the number of files we can have
        open at once, if we do not address this risk, we can
        accidentally starve some other part of our program of file
        handles.
    -   If we do not explicitly close a file handle, the garbage
        collector will automatically close it for us. It may take a long
        time to notice that it should close the file handle. This poses
        the same starvation risk as above.
    -   We can avoid starvation by explicitly closing a file handle. If
        we do so too early, though, we can cause a lazy computation to
        fail if it expects to be able to pull more data from a closed
        file handle.

    On top of these well-known risks, we cannot use a single file handle
    to supply data to multiple threads. A file handle has a single
    \"seek pointer\" that tracks the position from which it should be
    reading, but when we want to read multiple chunks, each needs to
    consume data from a different position in the file.

    With these ideas in mind, let\'s fill out the lazy I/O picture.

    <div>

    [LineChunks.hs]{.label}
    ``` {.haskell}
    chunkedRead :: (FilePath -> IO [ChunkSpec])
                -> FilePath
                -> IO ([LB.ByteString], [Handle])
    chunkedRead chunkFunc path = do
      chunks <- chunkFunc path
      liftM unzip . forM chunks $ \spec -> do
        h <- openFile path ReadMode
        hSeek h AbsoluteSeek (fromIntegral (chunkOffset spec))
        chunk <- LB.take (chunkLength spec) `liftM` LB.hGetContents h
        return (chunk, h)
    ```

    </div>

    We avoid the starvation problem by explicitly closing file handles.
    We allow multiple threads to read different chunks at once by
    supplying each one with a distinct file handle, all reading the same
    file.

    The final problem that we try to mitigate is that of a lazy
    computation having a file handle closed behind its back. We use
    `rnf` to force all of our processing to complete before we return
    from `withChunks`. We can then close our file handles explicitly, as
    they should no longer be read from. If you must use lazy I/O in a
    program, it is often best to \"firewall\" it like this, so that it
    cannot cause problems in unexpected parts of your code.

    ::: {.TIP}
    Processing chunks via a fold

    We can adapt the fold-with-early-termination technique from [the
    section called \"Another way of looking at
    traversal\"](9-a-library-for-searching-the-file-system.org::*Another way of looking at traversal)
    stream-based file processing. While this requires more work than the
    lazy I/O approach, it nicely avoids the above problems.
    :::

### Efficiently finding line-aligned chunks

Since a server log file is line-oriented, we need an efficient way to
break a file into large chunks, while making sure that each chunk ends
on a line boundary. Since a chunk might be tens of megabytes in size, we
don\'t want to scan all of the data in a chunk to determine where its
final boundary should be.

Our approach works whether we choose a fixed chunk size or a fixed
number of chunks. Here, we opt for the latter. We begin by seeking to
the approximate position of the end of a chunk, then scan forwards until
we reach a newline character. We then start the next chunk after the
newline, and repeat the procedure.

<div>

[LineChunks.hs]{.label}

``` {.haskell}
lineChunks :: Int -> FilePath -> IO [ChunkSpec]
lineChunks numChunks path = do
  bracket (openFile path ReadMode) hClose $ \h -> do
    totalSize <- fromIntegral `liftM` hFileSize h
    let chunkSize = totalSize `div` fromIntegral numChunks
        findChunks offset = do
          let newOffset = offset + chunkSize
          hSeek h AbsoluteSeek (fromIntegral newOffset)
          let findNewline off = do
                eof <- hIsEOF h
                if eof
                  then return [CS offset (totalSize - offset)]
                  else do
                    bytes <- LB.hGet h 4096
                    case LB.elemIndex '\n' bytes of
                      Just n -> do
                        chunks@(c:_) <- findChunks (off + n + 1)
                        let coff = chunkOffset c
                        return (CS offset (coff - offset):chunks)
                      Nothing -> findNewline (off + LB.length bytes)
          findNewline newOffset
    findChunks 0
```

</div>

The last chunk will end up a little shorter than its predecessors, but
this difference will be insignificant in practice.

### Counting lines

This simple example illustrates how to use the scaffolding we have
built.

<div>

[LineCount.hs]{.label}

``` {.haskell}
module Main where

import Control.Monad (forM_)
import Data.Int (Int64)
import qualified Data.ByteString.Lazy.Char8 as LB
import System.Environment (getArgs)

import LineChunks (chunkedReadWith)
import MapReduce (mapReduce, rnf)

lineCount :: [LB.ByteString] -> Int64
lineCount = mapReduce rnf (LB.count '\n')
                      rnf sum

main :: IO ()
main = do
  args <- getArgs
  forM_ args $ \path -> do
    numLines <- chunkedReadWith lineCount path
    putStrLn $ path ++ ": " ++ show numLines
```

</div>

If we compile this program with `ghc -O2 --make -threaded`, it should
perform well after an initial run to \"warm\" the filesystem cache. On a
dual core laptop, processing a log file 248 megabytes (1.1 million
lines) in size, this program runs in 0.576 seconds using a single core,
and 0.361 with two (using `+RTS -N2`).

### Finding the most popular URLs

In this example, we count the number of times each URL is accessed. This
example comes from \[[Google08](bibliography.org::Google08)\], Google\'s
original paper discussing MapReduce. In the *map* phase, for each chunk,
we create a `Map` from URL to the number of times it was accessed. In
the *reduce* phase, we union-merge these maps into one.

<div>

[CommonURLs.hs]{.label}

``` {.haskell}
module Main where

import Control.Parallel.Strategies (NFData(..), rwhnf)
import Control.Monad (forM_)
import Data.List (foldl', sortBy)
import qualified Data.ByteString.Lazy.Char8 as L
import qualified Data.ByteString.Char8 as S
import qualified Data.Map as M
import Text.Regex.PCRE.Light (compile, match)

import System.Environment (getArgs)
import LineChunks (chunkedReadWith)
import MapReduce (mapReduce)

countURLs :: [L.ByteString] -> M.Map S.ByteString Int
countURLs = mapReduce rwhnf (foldl' augment M.empty . L.lines)
                      rwhnf (M.unionsWith (+))
  where augment map line =
            case match (compile pattern []) (strict line) [] of
              Just (_:url:_) -> M.insertWith' (+) url 1 map
              _ -> map
        strict  = S.concat . L.toChunks
        pattern = S.pack "\"(?:GET|POST|HEAD) ([^ ]+) HTTP/"
```

</div>

To pick a URL out of a line of the log file, we use the bindings to the
PCRE regular expression library that we developed in [Chapter 17,
*Interfacing with C: the FFI*](17-interfacing-with-c.org).

Our driver function prints the ten most popular URLs. As with the line
counting example, this program runs about 1.8 times faster with two
cores than with one, taking 1.7 seconds to process the a log file
containing 1.1 million entries.

### Conclusions

Given a problem that fits its model well, the MapReduce programming
model lets us write \"casual\" parallel programs in Haskell with good
performance, and minimal additional effort. We can easily extend the
idea to use other data sources, such as collections of files, or data
sourced over the network.

In many cases, the performance bottleneck will be streaming data at a
rate high enough to keep up with a core\'s processing capacity. For
instance, if we try to use either of the above sample programs on a file
that is not cached in memory or streamed from a high-bandwidth storage
array, we will spend most of our time waiting for disk I/O, gaining no
benefit from multiple cores.

Footnotes
---------

[^1]: As we will show later, GHC threads are extraordinarily
    lightweight. If the runtime were to provide a way to check the
    status of every thread, the overhead of every thread would increase,
    even if this information were never used.

[^2]: The non-threaded runtime does not understand this option, and will
    reject it with an error message.

[^3]: As we write this book, the garbage collector is being retooled to
    use multiple cores, but we cannot yet predict its future effect.

[^4]: The genesis of this idea comes from Tim Bray.
