Chapter 28. Software transactional memory
=========================================

In the traditional threaded model of concurrent programming, when we
share data among threads, we keep it consistent using locks, and we
notify threads of changes using condition variables. Haskell\'s `MVar`
mechanism improves somewhat upon these tools, but it still suffers from
all of the same problems.

-   Race conditions due to forgotten locks
-   Deadlocks resulting from inconsistent lock ordering
-   Corruption caused by uncaught exceptions
-   Lost wakeups induced by omitted notifications

These problems frequently affect even the smallest concurrent programs,
but the difficulties they pose become far worse in larger code bases, or
under heavy load.

For instance, a program with a few big locks is somewhat tractable to
write and debug, but contention for those locks will clobber us under
heavy load. If we react with finer-grained locking, it becomes *far*
harder to keep our software working at all. The additional book-keeping
will hurt performance even when loads are light.

The basics
----------

Software transactional memory (STM) gives us a few simple, but powerful,
tools with which we can address most of these problems. We execute a
block of actions as a transaction using the `atomically` combinator.
Once we enter the block, other threads cannot see any modifications we
make until we exit, nor can our thread see any changes made by other
threads. These two properties mean that our execution is *isolated*.

Upon exit from a transaction, exactly one of the following things will
occur.

-   If no other thread concurrently modified the same data as us, all of
    our modifications will simultaneously become visible to other
    threads.
-   Otherwise, our modifications are discarded without being performed,
    and our block of actions is automatically restarted.

This all-or-nothing nature of an `atomically` block is referred to as
*atomic*, hence the name of the combinator. If you have used databases
that support transactions, you should find that working with STM feels
quite familiar.

Some simple examples
--------------------

In a multi-player role playing game, a player\'s character will have
some state such as health, possessions, and money. To explore the world
of STM, let\'s start with a few simple functions and types based around
working with some character state for a game. We will refine our code as
we learn more about the API.

The STM API is provided by the `stm` package, and its modules are in the
`Control.Concurrent.STM` hierarchy.

``` {.example}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}

import Control.Concurrent.STM
import Control.Monad

data Item = Scroll
          | Wand
          | Banjo
            deriving (Eq, Ord, Show)

newtype Gold = Gold Int
    deriving (Eq, Ord, Show, Num)

newtype HitPoint = HitPoint Int
    deriving (Eq, Ord, Show, Num)

type Inventory = TVar [Item]
type Health = TVar HitPoint
type Balance = TVar Gold

data Player = Player {
      balance :: Balance,
      health :: Health,
      inventory :: Inventory
    }
```

The `TVar` parameterized type is a mutable variable that we can read or
write inside an `atomically` block. For simplicity, we represent a
player\'s inventory as a list of items. Notice, too, that we use
`newtype` declarations so that we cannot accidentally confuse wealth
with health.

To perform a basic transfer of money from one `Balance` to another, all
we have to do is adjust the values in each `TVar`.

``` {.example}
basicTransfer qty fromBal toBal = do
  fromQty <- readTVar fromBal
  toQty   <- readTVar toBal
  writeTVar fromBal (fromQty - qty)
  writeTVar toBal   (toQty + qty)
```

Let\'s write a small function to try this out.

``` {.example}
transferTest = do
  alice <- newTVar (12 :: Gold)
  bob   <- newTVar 4
  basicTransfer 3 alice bob
  liftM2 (,) (readTVar alice) (readTVar bob)
```

If we run this in `ghci`, it behaves as we should expect.

``` {.screen}
ghci> :load GameInventory
[1 of 1] Compiling Main             ( GameInventory.hs, interpreted )
Ok, modules loaded: Main.
ghci> atomically transferTest
Loading package array-0.1.0.0 ... linking ... done.
Loading package stm-2.1.1.0 ... linking ... done.
(Gold 9,Gold 7)
```

The properties of atomicity and isolation guarantee that if another
thread sees a change in `bob`\'s balance, they will also be able to see
the modification of `alice`\'s balance.

Even in a concurrent program, we strive to keep as much of our code as
possible purely functional. This makes our code easier both to reason
about and to test. It also gives the underlying STM engine less work to
do, since the data involved is not transactional. Here\'s a pure
function that removes an item from the list we use to represent a
player\'s inventory.

``` {.example}
removeInv :: Eq a => a -> [a] -> Maybe [a]
removeInv x xs =
    case takeWhile (/= x) xs of
      (_:ys) -> Just ys
      []     -> Nothing
```

The result uses `Maybe` so that we can tell whether the item was
actually present in the player\'s inventory.

Here is a transactional function to give an item to another player. It
is slightly complicated by the need to determine whether the donor
actually *has* the item in question.

``` {.example}
maybeGiveItem item fromInv toInv = do
  fromList <- readTVar fromInv
  case removeInv item fromList of
    Nothing      -> return False
    Just newList -> do
      writeTVar fromInv newList
      destItems <- readTVar toInv
      writeTVar toInv (item : destItems)
      return True
```

STM and safety
--------------

If we are to provide atomic, isolated transactions, it is critical that
we cannot either deliberately or accidentally escape from an
`atomically` block. Haskell\'s type system enforces this on our behalf,
via the STM monad.

``` {.screen}
ghci> :type atomically
atomically :: STM a -> IO a
```

The `atomically` block takes an action in the STM monad, executes it,
and makes its result available to us in the `IO` monad. This is the
monad in which all transactional code executes. For instance, the
functions that we have seen for manipulating `TVar` values operate in
the `STM` monad.

``` {.screen}
ghci> :type newTVar
newTVar :: a -> STM (TVar a)
ghci> :type readTVar
readTVar :: TVar a -> STM a
ghci> :type writeTVar
writeTVar :: TVar a -> a -> STM ()
```

This is also true of the transactional functions we defined earlier.

``` {.example}
basicTransfer :: Gold -> Balance -> Balance -> STM ()
maybeGiveItem :: Item -> Inventory -> Inventory -> STM Bool
```

The `STM` monad does not let us perform I/O or manipulate
non-transactional mutable state, such as `MVar` values. This lets us
avoid operations that might violate the transactional guarantees.

Retrying a transaction
----------------------

The API of our `maybeGiveItem` function is somewhat awkward. It only
gives an item if the character actually possesses it, which is
reasonable, but by returning a `Bool`, it complicates the code of its
callers. Here is an item sale function that has to look at the result of
`maybeGiveItem` to decide what to do next.

``` {.example}
maybeSellItem :: Item -> Gold -> Player -> Player -> STM Bool
maybeSellItem item price buyer seller = do
  given <- maybeGiveItem item (inventory seller) (inventory buyer)
  if given
    then do
      basicTransfer price (balance buyer) (balance seller)
      return True
    else return False
```

Not only do we have to check whether the item was given, we have to
propagate an indication of success back to our caller. The complexity
thus cascades outwards.

There is a more elegant way to handle transactions that cannot succeed.
The STM API provides a `retry` action which will immediately terminate
an `atomically` block that cannot proceed. As the name suggests, when
this occurs, execution of the block is restarted from scratch, with any
previous modifications unperformed. Here is a rewrite of `maybeGiveItem`
to use `retry`.

``` {.example}
giveItem :: Item -> Inventory -> Inventory -> STM ()

giveItem item fromInv toInv = do
  fromList <- readTVar fromInv
  case removeInv item fromList of
    Nothing -> retry
    Just newList -> do
      writeTVar fromInv newList
      readTVar toInv >>= writeTVar toInv . (item :)
```

Our `basicTransfer` from earlier had a different kind of flaw: it did
not check the sender\'s balance to see if they had sufficient money to
transfer. We can use `retry` to correct this, while keeping the
function\'s type the same.

``` {.example}
transfer :: Gold -> Balance -> Balance -> STM ()

transfer qty fromBal toBal = do
  fromQty <- readTVar fromBal
  when (qty > fromQty) $
    retry
  writeTVar fromBal (fromQty - qty)
  readTVar toBal >>= writeTVar toBal . (qty +)
```

Now that we are using `retry`, our item sale function becomes
dramatically simpler.

``` {.example}
sellItem :: Item -> Gold -> Player -> Player -> STM ()
sellItem item price buyer seller = do
  giveItem item (inventory seller) (inventory buyer)
  transfer price (balance buyer) (balance seller)
```

Its behavior is slightly different from our earlier function. Instead of
immediately returning `False` if the seller doesn\'t have the item, it
will block (if necessary) until both the seller has the item and the
buyer has enough money to pay for it.

The beauty of STM lies in the cleanliness of the code it lets us write.
We can take two functions that work correctly, and use them to create a
third that will also behave itself, all with minimal effort.

### What happens when we retry?

The `retry` function doesn\'t just make our code cleaner: its underlying
behavior seems nearly magical. When we call it, it doesn\'t restart our
transaction immediately. Instead, it blocks our thread until one or more
of the variables that we touched before calling `retry` is changed by
another thread.

For instance, if we invoke `transfer` with insufficient funds, `retry`
will *automatically wait* until our balance changes before it starts the
`atomically` block again. The same happens with our new `giveItem`
function: if the sender doesn\'t currently have the item in their
inventory, the thread will block until they do.

Choosing between alternatives
-----------------------------

We don\'t always want to restart an `atomically` action if it calls
`retry` or fails due to concurrent modification by another thread. For
instance, our new `sellItem` function will retry indefinitely as long as
we are missing either the item or enough money, but we might prefer to
just try the sale once.

The `orElse` combinator lets us perform a \"backup\" action if the main
one fails.

``` {.screen}
ghci> :type orElse
orElse :: STM a -> STM a -> STM a
```

If `sellItem` fails, then `orElse` will invoke the `return False`
action, causing our sale function to return immediately.

### Using higher order code with transactions

Imagine that we\'d like to be a little more ambitious, and buy the first
item from a list that is both in the possession of the seller and
affordable to us, but do nothing if we cannot afford something right
now. We could of course write code to do this in a direct manner.

``` {.example}
crummyList :: [(Item, Gold)] -> Player -> Player
             -> STM (Maybe (Item, Gold))
crummyList list buyer seller = go list
    where go []                         = return Nothing
          go (this@(item,price) : rest) = do
              sellItem item price buyer seller
              return (Just this)
           `orElse`
              go rest
```

This function suffers from the familiar problem of muddling together
what we want to do with how we ought to do it. A little inspection
suggests that there are two reusable patterns buried in this code.

The first of these is to make a transaction fail immediately, instead of
retrying.

``` {.example}
maybeSTM :: STM a -> STM (Maybe a)
maybeSTM m = (Just `liftM` m) `orElse` return Nothing
```

Secondly, we want to try an action over successive elements of a list,
stopping at the first that succeeds, or performing a `retry` if every
one fails. Conveniently for us, STM is an instance of the `MonadPlus`
type class.

``` {.example}
instance MonadPlus STM where
  mzero = retry
  mplus = orElse
```

The `Control.Monad` module defines the `msum` function as follows, which
is exactly what we need.

``` {.example}
msum :: MonadPlus m => [m a] -> m a
msum =  foldr mplus mzero
```

We now have a few key pieces of machinery that will help us to write a
much clearer version of our function.

``` {.example}
shoppingList :: [(Item, Gold)] -> Player -> Player
             -> STM (Maybe (Item, Gold))
shoppingList list buyer seller = maybeSTM . msum $ map sellOne list
    where sellOne this@(item,price) = do
            sellItem item price buyer seller
            return this
```

Since STM is an instance of the `MonadPlus` type class, we can
generalize `maybeSTM` to work over any `MonadPlus`.

``` {.example}
maybeM :: MonadPlus m => m a -> m (Maybe a)
maybeM m = (Just `liftM` m) `mplus` return Nothing
```

This gives us a function that is useful in a greater variety of
situations.

I/O and STM
-----------

The STM monad forbids us from performing arbitrary I/O actions because
they can break the guarantees of atomicity and isolation that the monad
provides. Of course the need to perform I/O still arises; we just have
to treat it very carefully.

Most often, we will need to perform some I/O action as a result of a
decision we made inside an `atomically` block. In these cases, the right
thing to do is usually to return a piece of data from `atomically`,
which will tell the caller in the `IO` monad what to do next. We can
even return the action to perform, since actions are first class values.

``` {.example}
someAction :: IO a

stmTransaction :: STM (IO a)
stmTransaction = return someAction

doSomething :: IO a
doSomething = join (atomically stmTransaction)
```

We occasionally need to perform an I/O operation from within STM. For
instance, reading immutable data from a file that must exist does not
violate the STM guarantees of isolation or atomicity. In these cases, we
can use `unsafeIOToSTM` to execute an `IO` action. This function is
exported by the low-level `GHC.Conc` module, so we must go out of our
way to use it.

``` {.screen}
ghci> :m +GHC.Conc
ghci> :type unsafeIOToSTM
unsafeIOToSTM :: IO a -> STM a
```

The `IO` action that we execute must not start another `atomically`
transaction. If a thread tries to nest transactions, the runtime system
will throw an exception.

Since the type system can\'t help us to ensure that our `IO` code is
doing something sensible, we will be safest if we limit our use of
`unsafeIOToSTM` as much as possible. Here is a typical error that can
arise with `IO` in an `atomically` block.

``` {.example}
launchTorpedoes :: IO ()

notActuallyAtomic = do
  doStuff
  unsafeIOToSTM launchTorpedoes
  mightRetry
```

If the `mightRetry` block causes our transaction to restart, we will
call `launchTorpedoes` more than once. Indeed, we can\'t predict how
many times it will be called, since the runtime system handles retries
for us. The solution is not to perform these kinds of non-idempotent[^1]
I/O operations inside a transaction.

Communication between threads
-----------------------------

As well as the basic `TVar` type, the `stm` package provides two types
that are more useful for communicating between threads. A `TMVar` is the
STM equivalent of an `MVar`: it can hold either `Just` a value, or
`Nothing`. The `TChan` type is the STM counterpart of `Chan`, and
implements a typed FIFO channel.

A concurrent web link checker
-----------------------------

As a practical example of using STM, we will develop a program that
checks an HTML file for broken links, that is, URLs that either point to
bad web pages or dead servers. This is a good problem to address via
concurrency: if we try to talk to a dead server, it will take up to two
minutes before our connection attempt times out. If we use multiple
threads, we can still get useful work done while one or two are stuck
talking to slow or dead servers.

We can\'t simply create one thread per URL, because that may overburden
either our CPU or our network connection if (as we expect) most of the
links are live and responsive. Instead, we use a fixed number of worker
threads, which fetch URLs to download from a queue.

``` {.example}
{-# LANGUAGE FlexibleContexts, GeneralizedNewtypeDeriving,
             PatternGuards #-}

import Control.Concurrent (forkIO)
import Control.Concurrent.STM
import Control.Exception (catch, finally)
import Control.Monad.Error
import Control.Monad.State
import Data.Char (isControl)
import Data.List (nub)
import Network.URI
import Prelude hiding (catch)
import System.Console.GetOpt
import System.Environment (getArgs)
import System.Exit (ExitCode(..), exitWith)
import System.IO (hFlush, hPutStrLn, stderr, stdout)
import Text.Printf (printf)
import qualified Data.ByteString.Lazy.Char8 as B
import qualified Data.Set as S

-- This requires the HTTP package, which is not bundled with GHC
import Network.HTTP

type URL = B.ByteString

data Task = Check URL | Done
```

Our `main` function provides the top-level scaffolding for our program.

``` {.example}
main :: IO ()
main = do
    (files,k) <- parseArgs
    let n = length files

    -- count of broken links
    badCount <- newTVarIO (0 :: Int)

    -- for reporting broken links
    badLinks <- newTChanIO

    -- for sending jobs to workers
    jobs <- newTChanIO

    -- the number of workers currently running
    workers <- newTVarIO k

    -- one thread reports bad links to stdout
    forkIO $ writeBadLinks badLinks

    -- start worker threads
    forkTimes k workers (worker badLinks jobs badCount)

    -- read links from files, and enqueue them as jobs
    stats <- execJob (mapM_ checkURLs files)
                     (JobState S.empty 0 jobs)

    -- enqueue "please finish" messages
    atomically $ replicateM_ k (writeTChan jobs Done)

    waitFor workers

    broken <- atomically $ readTVar badCount

    printf fmt broken
               (linksFound stats)
               (S.size (linksSeen stats))
               n
  where
    fmt   = "Found %d broken links. " ++
            "Checked %d links (%d unique) in %d files.\n"
```

When we are in the `IO` monad, we can create new `TVar` values using the
`newTVarIO` function. There are also counterparts for creating `TMVar`
and `TChan` values.

Notice that we use the `printf` function to print a report at the end.
Unlike its counterpart in C, the Haskell `printf` function can check its
argument types, and their number, at runtime.

``` {.screen}
ghci> :m +Text.Printf
ghci> printf "%d and %d\n" (3::Int)
3 and *** Exception: Printf.printf: argument list ended prematurely
ghci> printf "%s and %d\n" "foo" (3::Int)
foo and 3
```

Try evaluating `printf "%d" True` at the `ghci` prompt, and see what
happens.

Supporting `main` are several short functions.

``` {.example}
modifyTVar_ :: TVar a -> (a -> a) -> STM ()
modifyTVar_ tv f = readTVar tv >>= writeTVar tv . f

forkTimes :: Int -> TVar Int -> IO () -> IO ()
forkTimes k alive act =
  replicateM_ k . forkIO $
    act
    `finally`
    (atomically $ modifyTVar_ alive (subtract 1))
```

The `forkTimes` function starts a number of identical worker threads,
and decreases the \"alive\" count each time a thread exits. We use a
`finally` combinator to ensure that the count is always decremented, no
matter how the thread terminates.

Next, the `writeBadLinks` function prints each broken or dead link to
`stdout`.

``` {.example}
writeBadLinks :: TChan String -> IO ()
writeBadLinks c =
  forever $
    atomically (readTChan c) >>= putStrLn >> hFlush stdout
```

We use the `forever` combinator above, which repeats an action
endlessly.

``` {.screen}
ghci> :m +Control.Monad
ghci> :type forever
forever :: (Monad m) => m a -> m ()
```

Our `waitFor` function uses `check`, which calls `retry` if its argument
evaluates to `False`.

``` {.example}
waitFor :: TVar Int -> IO ()
waitFor alive = atomically $ do
  count <- readTVar alive
  check (count == 0)
```

### Checking a link

Here is a naive function to check the state of a link. This code is
similar to the podcatcher that we developed in [Chapter 22, *Extended
Example: Web Client Programming*](22-web-client-programming.org), with a
few small differences.

``` {.example}
getStatus :: URI -> IO (Either String Int)
getStatus = chase (5 :: Int)
  where
    chase 0 _ = bail "too many redirects"
    chase n u = do
      resp <- getHead u
      case resp of
        Left err -> bail (show err)
        Right r ->
          case rspCode r of
            (3,_,_) ->
               case findHeader HdrLocation r of
                 Nothing -> bail (show r)
                 Just u' ->
                   case parseURI u' of
                     Nothing -> bail "bad URL"
                     Just url -> chase (n-1) url
            (a,b,c) -> return . Right $ a * 100 + b * 10 + c
    bail = return . Left

getHead :: URI -> IO (Result Response)
getHead uri = simpleHTTP Request { rqURI = uri,
                                   rqMethod = HEAD,
                                   rqHeaders = [],
                                   rqBody = "" }
```

We follow a HTTP redirect response just a few times, to avoid endless
redirect loops. To determine whether a URL is valid, we use the HTTP
standard\'s HEAD verb, which uses less bandwidth than a full GET.

This code has the classic \"marching off the left of the screen\" style
that we have learned to be wary of. Here is a rewrite that offers
greater clarity via the `ErrorT` monad transformer and a few generally
useful functions.

``` {.example}
getStatusE = runErrorT . chase (5 :: Int)
  where
    chase :: Int -> URI -> ErrorT String IO Int
    chase 0 _ = throwError "too many redirects"
    chase n u = do
      r <- embedEither show =<< liftIO (getHead u)
      case rspCode r of
        (3,_,_) -> do
            u'  <- embedMaybe (show r)  $ findHeader HdrLocation r
            url <- embedMaybe "bad URL" $ parseURI u'
            chase (n-1) url
        (a,b,c) -> return $ a*100 + b*10 + c

-- This function is defined in Control.Arrow.
left :: (a -> c) -> Either a b -> Either c b
left f (Left x)  = Left (f x)
left _ (Right x) = Right x

-- Some handy embedding functions.
embedEither :: (MonadError e m) => (s -> e) -> Either s a -> m a
embedEither f = either (throwError . f) return

embedMaybe :: (MonadError e m) => e -> Maybe a -> m a
embedMaybe err = maybe (throwError err) return
```

You might notice that, for once, we are explicitly using

### Worker threads

Each worker thread reads a task off the shared queue. It either checks
the given URL or exits.

``` {.example}
worker :: TChan String -> TChan Task -> TVar Int -> IO ()
worker badLinks jobQueue badCount = loop
  where
    -- Consume jobs until we are told to exit.
    loop = do
        job <- atomically $ readTChan jobQueue
        case job of
            Done  -> return ()
            Check x -> checkOne (B.unpack x) >> loop

    -- Check a single link.
    checkOne url = case parseURI url of
        Just uri -> do
            code <- getStatus uri `catch` (return . Left . show) 
            case code of
                Right 200 -> return ()
                Right n   -> report (show n)
                Left err  -> report err
        _ -> report "invalid URL"

        where report s = atomically $ do
                           modifyTVar_ badCount (+1)
                           writeTChan badLinks (url ++ " " ++ s)
```

### Finding links

We structure our link finding around a state monad transformer stacked
on the `IO` monad. Our state tracks links that we have already seen (so
we don\'t check a repeated link more than once), the total number of
links we have encountered, and the queue to which we should add the
links that we will be checking.

``` {.example}
data JobState = JobState { linksSeen :: S.Set URL,
                           linksFound :: Int,
                           linkQueue :: TChan Task }

newtype Job a = Job { runJob :: StateT JobState IO a }
    deriving (Monad, MonadState JobState, MonadIO)

execJob :: Job a -> JobState -> IO JobState
execJob = execStateT . runJob
```

Strictly speaking, for a small standalone program, we don\'t need the
`newtype` wrapper, but we include it here as an example of good practice
(it only costs a few lines of code, anyway).

The `main` function maps `checkURLs` over each input file, so
`checkURLs` only needs to read a single file.

``` {.example}
checkURLs :: FilePath -> Job ()
checkURLs f = do
    src <- liftIO $ B.readFile f
    let urls = extractLinks src
    filterM seenURI urls >>= sendJobs
    updateStats (length urls)

updateStats :: Int -> Job ()
updateStats a = modify $ \s ->
    s { linksFound = linksFound s + a }

-- | Add a link to the set we have seen.
insertURI :: URL -> Job ()
insertURI c = modify $ \s ->
    s { linksSeen = S.insert c (linksSeen s) }

-- | If we have seen a link, return False.  Otherwise, record that we
-- have seen it, and return True.
seenURI :: URL -> Job Bool
seenURI url = do
    seen <- (not . S.member url) `liftM` gets linksSeen
    insertURI url
    return seen

sendJobs :: [URL] -> Job ()
sendJobs js = do
    c <- gets linkQueue
    liftIO . atomically $ mapM_ (writeTChan c . Check) js
```

Our `extractLinks` function doesn\'t attempt to properly parse a HTML or
text file. Instead, it looks for strings that appear to be URLs, and
treats them as \"good enough\".

``` {.example}
extractLinks :: B.ByteString -> [URL]
extractLinks = concatMap uris . B.lines
  where uris s      = filter looksOkay (B.splitWith isDelim s)
        isDelim c   = isControl c || c `elem` " <>\"{}|\\^[]`"
        looksOkay s = http `B.isPrefixOf` s
        http        = B.pack "http:"
```

### Command line parsing

To parse our command line arguments, we use the `System.Console.GetOpt`
module. It provides useful code for parsing arguments, but it is
slightly involved to use.

``` {.example}
data Flag = Help | N Int
            deriving Eq

parseArgs :: IO ([String], Int)
parseArgs = do
    argv <- getArgs
    case parse argv of
        ([], files, [])                     -> return (nub files, 16)
        (opts, files, [])
            | Help `elem` opts              -> help
            | [N n] <- filter (/=Help) opts -> return (nub files, n)
        (_,_,errs)                          -> die errs
  where
    parse argv = getOpt Permute options argv
    header     = "Usage: urlcheck [-h] [-n n] [file ...]"
    info       = usageInfo header options
    dump       = hPutStrLn stderr
    die errs   = dump (concat errs ++ info) >> exitWith (ExitFailure 1)
    help       = dump info                  >> exitWith ExitSuccess
```

The `getOpt` function takes three arguments.

-   An argument ordering, which specifies whether options can be mixed
    with other arguments (`Permute`, which we use above) or must appear
    before them.
-   A list of option definitions. Each consists of a list of short names
    for the option, a list of long names for the option, a description
    of the option (e.g. whether it accepts an argument), and an
    explanation for users.
-   A list of the arguments and options, as returned by `getArgs`.

The function returns a triple which consists of the parsed options, the
remaining arguments, and any error messages that arose.

We use the `Flag` algebraic data type to represent the options our
program can accept.

``` {.example}
options :: [OptDescr Flag]
options = [ Option ['h'] ["help"] (NoArg Help)
                   "Show this help message",
            Option ['n'] []       (ReqArg (\s -> N (read s)) "N")
                   "Number of concurrent connections (default 16)" ]
```

Our `options` list describes each option that we accept. Each
description must be able to create a `Flag` value. Take a look at our
uses of `NoArg` and `ReqArg` above. These are constructors for the
`GetOpt` module\'s `ArgDescr` type.

``` {.example}
data ArgDescr a = NoArg a
                | ReqArg (String -> a) String
                | OptArg (Maybe String -> a) String
```

-   The `NoArg` constructor accepts a parameter that will represent this
    option. In our case, if a user invokes our program with `-h` or
    `--help`, we will use the value `Help`.
-   The `ReqArg` constructor accepts a function that maps a required
    argument to a value. Its second argument is used when printing help.
    Here, we convert a string into an integer, and pass it to our `Flag`
    type\'s `N` constructor.
-   The `OptArg` constructor is similar to the `ReqArg` constructor, but
    it permits the use of options that can be used without arguments.

### Pattern guards

We sneaked one last language extension into our definition of
`parseArgs`. Pattern guards let us write more concise guard expressions.
They are enabled via the `PatternGuards` language extension.

A pattern guard has three components: a pattern, a `<-` symbol, and an
expression. The expression is evaluated and matched against the pattern.
If it matches, any variables present in the pattern are bound. We can
mix pattern guards and normal `Bool` guard expressions in a single guard
by separating them with commas.

``` {.example}
{-# LANGUAGE PatternGuards #-}

testme x xs | Just y <- lookup x xs, y > 3 = y
            | otherwise                    = 0
```

In the above example, we return a value from the alist `xs` if its
associated key `x` is present, provided the value is greater than

1.  The above definition is equivalent to the following.

``` {.example}
testme_noguards x xs = case lookup x xs of
                         Just y | y > 3 -> y
                         _              -> 0
```

Pattern guards let us \"collapse\" a collection of guards and `case`
expressions into a single guard, allowing us to write more succinct and
descriptive guards.

Practical aspects of STM
------------------------

We have so far been quiet about the specific benefits that STM gives us.
Most obvious is how well it *composes*: to add code to a transaction, we
just use our usual monadic building blocks, `(>>=)` and `(>>)`.

The notion of composability is critical to building modular software. If
we take two pieces of code that individually work correctly, the
composition of the two should also be correct. While normal threaded
programming makes composability impossible, STM restores it as a key
assumption that we can rely upon.

The `STM` monad prevents us from accidentally performing
non-transactional I/O actions. We don\'t need to worry about lock
ordering, since our code contains no locks. We can forget about lost
wakeups, since we don\'t have condition variables. If an exception is
thrown, we can either catch it using `catchSTM`, or be bounced out of
our transaction, leaving our state untouched. Finally, the `retry` and
`orElse` functions give us some beautiful ways to structure our code.

Code that uses STM will not deadlock, but it is possible for threads to
starve each other to some degree. A long-running transaction can cause
another transaction to `retry` often enough that it will make
comparatively little progress. To address a problem like this, make your
transactions as short as you can, while keeping your data consistent.

### Getting comfortable with giving up control

Whether with concurrency or memory management, there will be times when
we must retain control: some software must make solid guarantees about
latency or memory footprint, so we will be forced to spend the extra
time and effort managing and debugging explicit code. For many
interesting, practical uses of software, garbage collection and STM will
do more than well enough.

STM is not a complete panacea. It is useful to compare it with the use
of garbage collection for memory management. When we abandon explicit
memory management in favour of garbage collection, we give up control in
return for safer code. Likewise, with STM, we abandon the low-level
details, in exchange for code that we can better hope to understand.

### Using invariants

STM cannot eliminate certain classes of bug. For instance, if we
withdraw money from an account in one `atomically` block, return to the
`IO` monad, then deposit it to another account in a different
`atomically` block, our code will have an inconsistency. There will be a
window of time in which the money is present in neither account.

``` {.example}
bogusTransfer qty fromBal toBal = do
  fromQty <- atomically $ readTVar fromBal
  -- window of inconsistency
  toQty   <- atomically $ readTVar toBal
  atomically $ writeTVar fromBal (fromQty - qty)
  -- window of inconsistency
  atomically $ writeTVar toBal   (toQty + qty)

bogusSale :: Item -> Gold -> Player -> Player -> IO ()
bogusSale item price buyer seller = do
  atomically $ giveItem item (inventory seller) (inventory buyer)
  bogusTransfer price (balance buyer) (balance seller)
```

In concurrent programs, these kinds of problems are notoriously
difficult to find and reproduce. For instance, the inconsistency that we
describe above will usually only occur for a brief period of time.
Problems like this often refuse to show up during development, instead
only occurring in the field, under heavy load.

The `alwaysSucceeds` function lets us define an *invariant*, a property
of our data that must always be true.

``` {.screen}
ghci> :type alwaysSucceeds
alwaysSucceeds :: STM a -> STM ()
```

When we create an invariant, it will immediately be checked. To fail,
the invariant must raise an exception. More interestingly, the invariant
will subsequently be checked automatically at the end of *every*
transaction. If it fails at any point, the transaction will be aborted,
and the exception raised by the invariant will be propagated. This means
that we will get immediate feedback as soon as one of our invariants is
violated.

For instance, here are a few functions to populate our game world from
the beginning of this chapter with players.

``` {.example}
newPlayer :: Gold -> HitPoint -> [Item] -> STM Player
newPlayer balance health inventory =
    Player `liftM` newTVar balance
              `ap` newTVar health
              `ap` newTVar inventory

populateWorld :: STM [Player]
populateWorld = sequence [ newPlayer 20 20 [Wand, Banjo],
                           newPlayer 10 12 [Scroll] ]
```

This function returns an invariant that we can use to ensure that the
world\'s money balance is always consistent: the balance at any point in
time should be the same as at the creation of the world.

``` {.example}
consistentBalance :: [Player] -> STM (STM ())
consistentBalance players = do
    initialTotal <- totalBalance
    return $ do
      curTotal <- totalBalance
      when (curTotal /= initialTotal) $
        error "inconsistent global balance"
  where totalBalance   = foldM addBalance 0 players
        addBalance a b = (a+) `liftM` readTVar (balance b)
```

Let\'s write a small function that exercises this.

``` {.example}
tryBogusSale = do
  players@(alice:bob:_) <- atomically populateWorld
  atomically $ alwaysSucceeds =<< consistentBalance players
  bogusSale Wand 5 alice bob
```

If we run it in `ghci`, it should detect the inconsistency caused by our
incorrect use of `atomically` in the `bogusTransfer` function we wrote.

``` {.screen}
ghci> tryBogusSale
*** Exception: inconsistent global balance
```

Footnotes
---------

[^1]: An idempotent action gives the same result every time it is
    invoked, no matter how many times this occurs.
