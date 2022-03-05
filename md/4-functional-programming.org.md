---
name: Exercise 4.1
---

Chapter 4: Functional Programming
=================================

Thinking in Haskell
-------------------

Our early learning of Haskell has two distinct aspects. The first is
coming to terms with the shift in mindset from imperative programming to
functional: we have to replace our programming habits from other
languages. We do this not because imperative techniques are bad, but
because in a functional language other techniques work better.

Our second challenge is learning our way around the standard Haskell
libraries. As in any language, the libraries act as a lever, enabling us
to multiply our problem solving power. Haskell libraries tend to operate
at a higher level of abstraction than those in many other languages.
We\'ll need to work a little harder to learn to use the libraries, but
in exchange they offer a lot of power.

In this chapter, we\'ll introduce a number of common functional
programming techniques. We\'ll draw upon examples from imperative
languages to highlight the shift in thinking that we\'ll need to make.
As we do so, we\'ll walk through some of the fundamentals of Haskell\'s
standard libraries. We\'ll also intermittently cover a few more language
features along the way.

A simple command line framework
-------------------------------

In most of this chapter, we will concern ourselves with code that has no
interaction with the outside world. To maintain our focus on practical
code, we will begin by developing a gateway between our \"pure\" code
and the outside world. Our framework simply reads the contents of one
file, applies a function to the file, and writes the result to another
file.

<div>

[InteractWith.hs]{.label}

``` {.haskell}
-- Save this in a source file, e.g. Interact.hs

import System.Environment (getArgs)

interactWith function inputFile outputFile = do
  input <- readFile inputFile
  writeFile outputFile (function input)

main = mainWith myFunction
  where mainWith function = do
          args <- getArgs
          case args of
            [input,output] -> interactWith function input output
            _ -> putStrLn "error: exactly two arguments needed"

        -- replace "id" with the name of our function below
        myFunction = id
```

</div>

This is all we need to write simple, but complete, file processing
programs. This is a complete program. We can compile it to an executable
named `InteractWith` as follows.

``` {.screen}
$ ghc --make InteractWith
[1 of 1] Compiling Main             ( InteractWith.hs, InteractWith.o )
Linking InteractWith ...
```

If we run this program from the shell or command prompt, it will accept
two file names: the name of a file to read, and the name of a file to
write.

``` {.screen}
$ ./Interact
error: exactly two arguments needed
$ ./Interact hello-in.txt hello-out.txt
$ cat hello-in.txt
hello world
$ cat hello-out.txt
hello world
```

Some of the notation in our source file is new. The `do` keyword
introduces a block of *actions* that can cause effects in the real
world, such as reading or writing a file. The `<-` operator is the
equivalent of assignment inside a `do` block. This is enough explanation
to get us started. We will talk in much more detail about these details
of notation, and I/O in general, in [Chapter 7, *I/O*](7-io.org).

When we want to test a function that cannot talk to the outside world,
we simply replace the name `id` in the code above with the name of the
function we want to test. Whatever our function does, it will need to
have the type `String -> String`: in other words, it must accept a
string, and return a string.

Warming up: portably splitting lines of text
--------------------------------------------

Haskell provides a built-in function, `lines`, that lets us split a text
string on line boundaries. It returns a list of strings with line
termination characters omitted.

``` {.screen}
ghci> :type lines
lines :: String -> [String]
ghci> lines "line 1\nline 2"
["line 1","line 2"]
ghci> lines "foo\n\nbar\n"
["foo","","bar"]
```

While `lines` looks useful, it relies on us reading a file in \"text
mode\" in order to work. Text mode is a feature common to many
programming languages: it provides a special behavior when we read and
write files on Windows. When we read a file in text mode, the file I/O
library translates the line ending sequence `"\r\n"` (carriage return
followed by newline) to `"\n"` (newline alone), and it does the reverse
when we write a file. On Unix-like systems, text mode does not perform
any translation. As a result of this difference, if we read a file on
one platform that was written on the other, the line endings are likely
to become a mess. (Both `readFile` and `writeFile` operate in text
mode.)

``` {.screen}
ghci> lines "a\r\nb"
["a\r","b"]
```

The `lines` function only splits on newline characters, leaving carriage
returns dangling at the ends of lines. If we read a Windows-generated
text file on a Linux or Unix box, we\'ll get trailing carriage returns
at the end of each line.

We have comfortably used Python\'s \"universal newline\" support for
years: this transparently handles Unix and Windows line ending
conventions for us. We would like to provide something similar in
Haskell.

Since we are still early in our career of reading Haskell code, we will
discuss our Haskell implementation in quite some detail.

<div>

[SplitLines.hs]{.label}

``` {.haskell}
splitLines :: String -> [String]
```

</div>

Our function\'s type signature indicates that it accepts a single
string, the contents of a file with some unknown line ending convention.
It returns a list of strings, representing each line from the file.

<div>

[SplitLines.hs]{.label}

``` {.haskell}
splitLines [] = []
splitLines cs =
    let (pre, suf) = break isLineTerminator cs
    in  pre : case suf of
                ('\r':'\n':rest) -> splitLines rest
                ('\r':rest)      -> splitLines rest
                ('\n':rest)      -> splitLines rest
                _                -> []

isLineTerminator c = c == '\r' || c == '\n'
```

</div>

Before we dive into detail, notice first how we have organized our code.
We have presented the important pieces of code first, keeping the
definition of `isLineTerminator` until later. Because we have given the
helper function a readable name, we can guess what it does even before
we\'ve read it, which eases the smooth \"flow\" of reading the code.

The prelude defines a function named `break` that we can use to
partition a list into two parts. It takes a function as its first
parameter. That function must examine an element of the list, and return
a `Bool` to indicate whether to break the list at that point. The
`break` function returns a pair, which consists of the sublist consumed
before the predicate returned `True` (the *prefix*), and the rest of the
list (the *suffix*).

``` {.screen}
ghci> break odd [2,4,5,6,8]
([2,4],[5,6,8])
ghci> :module +Data.Char
ghci> break isUpper "isUpper"
("is","Upper")
```

Since we only need to match a single carriage return or newline at a
time, examining one element of the list at a time is good enough for our
needs.

The first equation of `splitLines` indicates that if we match an empty
string, we have no further work to do.

In the second equation, we first apply `break` to our input string. The
prefix is the substring before a line terminator, and the suffix is the
remainder of the string. The suffix will include the line terminator, if
any is present.

The \"`pre :`\" expression tells us that we should add the `pre` value
to the front of the list of lines. We then use a `case` expression to
inspect the suffix, so we can decide what to do next. The result of the
`case` expression will be used as the second argument to the `(:)` list
constructor.

The first pattern matches a string that begins with a carriage return,
followed by a newline. The variable `rest` is bound to the remainder of
the string. The other patterns are similar, so they ought to be easy to
follow.

A prose description of a Haskell function isn\'t necessarily easy to
follow. We can gain a better understanding by stepping into `ghci`, and
oberving the behavior of the function in different circumstances.

Let\'s start by partitioning a string that doesn\'t contain any line
terminators.

``` {.screen}
ghci> splitLines "foo"
["foo"]
```

Here, our application of `break` never finds a line terminator, so the
suffix it returns is empty.

``` {.screen}
ghci> break isLineTerminator "foo"
("foo","")
```

The `case` expression in `splitLines` must thus be matching on the
fourth branch, and we\'re finished. What about a slightly more
interesting case?

``` {.screen}
ghci> splitLines "foo\r\nbar"
["foo","bar"]
```

Our first application of `break` gives us a non-empty suffix.

``` {.screen}
ghci> break isLineTerminator "foo\r\nbar"
("foo","\r\nbar")
```

Because the suffix begins with a carriage return, followed by a newline,
we match on the first branch of the `case` expression. This gives us
`pre` bound to `"foo"`, and `suf` bound to `"bar"`. We apply
`splitLines` recursively, this time on `"bar"` alone.

``` {.screen}
ghci> splitLines "bar"
["bar"]
```

The result is that we construct a list whose head is `"foo"` and whose
tail is `["bar"]`.

``` {.screen}
ghci> "foo" : ["bar"]
["foo","bar"]
```

This sort of experimenting with `ghci` is a helpful way to understand
and debug the behavior of a piece of code. It has an even more important
benefit that is almost accidental in nature. It can be tricky to test
complicated code from `ghci`, so we will tend to write smaller
functions. This can further help the readability of our code.

This style of creating and reusing small, powerful pieces of code is a
fundamental part of functional programming.

### A line ending conversion program

Let\'s hook our `splitLines` function into the little framework we wrote
earlier. Make a copy of the `Interact.hs` source file; let\'s call the
new file `FixLines.hs`. Add the `splitLines` function to the new source
file. Since our function must produce a single string, we must stitch
the list of lines back together. The prelude provides an `unlines`
function that concatenates a list of strings, adding a newline to the
end of each.

<div>

[SplitLines.hs]{.label}

``` {.haskell}
fixLines :: String -> String
fixLines input = unlines (splitLines input)
```

</div>

If we replace the `id` function with `fixLines`, we can compile an
executable that will convert a text file to our system\'s native line
ending.

``` {.screen}
$ ghc --make FixLines
[1 of 1] Compiling Main             ( FixLines.hs, FixLines.o )
Linking FixLines ...
```

If you are on a Windows system, find and download a text file that was
created on a Unix system (for example
[gpl-3.0.txt](http://www.gnu.org/licenses/gpl-3.0.txt)). Open it in the
standard Notepad text editor. The lines should all run together, making
the file almost unreadable. Process the file using the `FixLines`
command you just created, and open the output file in Notepad. The line
endings should now be fixed up.

On Unix-like systems, the standard pagers and editors hide Windows line
endings. This makes it more difficult to verify that `FixLines` is
actually eliminating them. Here are a few commands that should help.

``` {.screen}
$ file gpl-3.0.txt
gpl-3.0.txt: ASCII English text
$ unix2dos gpl-3.0.txt
unix2dos: converting file gpl-3.0.txt to DOS format ...
$ file gpl-3.0.txt
gpl-3.0.txt: ASCII English text, with CRLF line terminators
```

Infix functions
---------------

Usually, when we define or apply a function in Haskell, we write the
name of the function, followed by its arguments. This notation is
referred to as *prefix*, because the name of the function comes before
its arguments.

If a function or constructor takes two or more arguments, we have the
option of using it in *infix* form, where we place it *between* its
first and second arguments. This allows us to use functions as infix
operators.

To define or apply a function or value constructor using infix notation,
we enclose its name in backtick characters (sometimes known as
backquotes). Here are simple infix definitions of a function and a type.

<div>

[Plus.hs]{.label}

``` {.haskell}
a `plus` b = a + b

data a `Pair` b = a `Pair` b
                  deriving (Show)

-- we can use the constructor either prefix or infix
foo = Pair 1 2
bar = True `Pair` "quux"
```

</div>

Since infix notation is purely a syntactic convenience, it does not
change a function\'s behavior.

``` {.screen}
ghci> 1 `plus` 2
3
ghci> plus 1 2
3
ghci> True `Pair` "something"
True `Pair` "something"
ghci> Pair True "something"
True `Pair` "something"
```

Infix notation can often help readability. For instance, the prelude
defines a function, `elem`, that indicates whether a value is present in
a list. If we use `elem` using prefix notation, it is fairly easy to
read.

``` {.screen}
ghci> elem 'a' "camogie"
True
```

If we switch to infix notation, the code becomes even easier to
understand. It is now clearer that we\'re checking to see if the value
on the left is present in the list on the right.

``` {.screen}
ghci> 3 `elem` [1,2,4,8]
False
```

We see a more pronounced improvement with some useful functions from the
`Data.List` module. The `isPrefixOf` function tells us if one list
matches the beginning of another.

``` {.screen}
ghci> :module +Data.List
ghci> "foo" `isPrefixOf` "foobar"
True
```

The `isInfixOf` and `isSuffixOf` functions match anywhere in a list and
at its end, respectively.

``` {.screen}
ghci> "needle" `isInfixOf` "haystack full of needle thingies"
True
ghci> "end" `isSuffixOf` "the end"
True
```

There is no hard-and-fast rule that dictates when you ought to use infix
versus prefix notation, although prefix notation is far more common.
It\'s best to choose whichever makes your code more readable in a
specific situation.

::: {.NOTE}
Beware familiar notation in an unfamiliar language

A few other programming languages use backticks, but in spite of the
visual similarities, the purpose of backticks in Haskell does not
remotely resemble their meaning in, for example, Perl, Python, or Unix
shell scripts.

The only legal thing we can do with backticks in Haskell is wrap them
around the name of a function. We can\'t, for example, use them to
enclose a complex expression whose value is a function. It might be
convenient if we could, but that\'s not how the language is today.
:::

Working with lists
------------------

As the bread and butter of functional programming, lists deserve some
serious attention. The standard prelude defines dozens of functions for
dealing with lists. Many of these will be indispensable tools, so it\'s
important that we learn them early on.

For better or worse, this section is going to read a bit like a
\"laundry list\" of functions. Why present so many functions at once?
These functions are both easy to learn and absolutely ubiquitous. If we
don\'t have this toolbox at our fingertips, we\'ll end up wasting time
by reinventing simple functions that are already present in the standard
libraries. So bear with us as we go through the list; the effort you\'ll
save will be huge.

The `Data.List` module is the \"real\" logical home of all standard list
functions. The prelude merely re-exports a large subset of the functions
exported by `Data.List`. Several useful functions in `Data.List` are
*not* re-exported by the standard prelude. As we walk through list
functions in the sections that follow, we will explicitly mention those
that are only in `Data.List`.

``` {.screen}
ghci> :module +Data.List
```

Because none of these functions is complex or takes more than about
three lines of Haskell to write, we\'ll be brief in our descriptions of
each. In fact, a quick and useful learning exercise is to write a
definition of each function after you\'ve read about it.

### Basic list manipulation

The `length` function tells us how many elements are in a list.

``` {.screen}
ghci> :type length
length :: [a] -> Int
ghci> length []
0
ghci> length [1,2,3]
3
ghci> length "strings are lists, too"
22
```

If you need to determine whether a list is empty, use the `null`
function.

``` {.screen}
ghci> :type null
null :: [a] -> Bool
ghci> null []
True
ghci> null "plugh"
False
```

To access the first element of a list, we use the `head` function.

``` {.screen}
ghci> :type head
head :: [a] -> a
ghci> head [1,2,3]
1
```

The converse, `tail`, returns all *but* the head of a list.

``` {.screen}
ghci> :type tail
tail :: [a] -> [a]
ghci> tail "foo"
"oo"
```

Another function, `last`, returns the very last element of a list.

``` {.screen}
ghci> :type last
last :: [a] -> a
ghci> last "bar"
'r'
```

The converse of `last` is `init`, which returns a list of all but the
last element of its input.

``` {.screen}
ghci> :type init
init :: [a] -> [a]
ghci> init "bar"
"ba"
```

Several of the functions above behave poorly on empty lists, so be
careful if you don\'t know whether or not a list is empty. What form
does their misbehavior take?

``` {.screen}
ghci> head []
*** Exception: Prelude.head: empty list
```

Try each of the above functions in `ghci`. Which ones crash when given
an empty list?

### Safely and sanely working with crashy functions

When we want to use a function like `head`, where we know that it might
blow up on us if we pass in an empty list, the temptation might
initially be strong to check the length of the list before we call
`head`. Let\'s construct an artificial example to illustrate our point.

<div>

[EfficientList.hs]{.label}

``` {.haskell}
myDumbExample xs = if length xs > 0
                   then head xs
                   else 'Z'
```

</div>

If we\'re coming from a language like Perl or Python, this might seem
like a perfectly natural way to write this test. Behind the scenes,
Python lists are arrays; and Perl arrays are, well, arrays. So they
necessarily know how long they are, and calling `len(foo)` or
`scalar(@foo)` is a perfectly natural thing to do. But as with many
other things, it\'s not a good idea to blindly transplant such an
assumption into Haskell.

We\'ve already seen the definition of the list algebraic data type many
times, and know that a list doesn\'t store its own length explicitly.
Thus, the only way that `length` can operate is to walk the entire list.

Therefore, when we only care whether or not a list is empty, calling
`length` isn\'t a good strategy. It can potentially do a lot more work
than we want, if the list we\'re working with is finite. Since Haskell
lets us easily create infinite lists, a careless use of `length` may
even result in an infinite loop.

A more appropriate function to call here instead is `null`, which runs
in constant time. Better yet, using `null` makes our code indicate what
property of the list we really care about. Here are two improved ways of
expressing `myDumbExample`.

<div>

[EfficientList.hs]{.label}

``` {.haskell}
mySmartExample xs = if not (null xs)
                    then head xs
                    else 'Z'

myOtherExample (x:_) = x
myOtherExample [] = 'Z'
```

</div>

### Partial and total functions

Functions that only have return values defined for a subset of valid
inputs are called *partial* functions (calling `error` doesn\'t qualify
as returning a value!). We call functions that return valid results over
their entire input domains *total* functions.

It\'s always a good idea to know whether a function you\'re using is
partial or total. Calling a partial function with an input that it
can\'t handle is probably the single biggest source of straightforward,
avoidable bugs in Haskell programs.

Some Haskell programmers go so far as to give partial functions names
that begin with a prefix such as `unsafe`, so that they can\'t shoot
themselves in the foot accidentally.

It\'s arguably a deficiency of the standard prelude that it defines
quite a few \"unsafe\" partial functions, like `head`, without also
providing \"safe\" total equivalents.

### More simple list manipulations

Haskell\'s name for the \"append\" function is `(++)`.

``` {.screen}
ghci> :type (++)
(++) :: [a] -> [a] -> [a]
ghci> "foo" ++ "bar"
"foobar"
ghci> [] ++ [1,2,3]
[1,2,3]
ghci> [True] ++ []
[True]
```

The `concat` function takes a list of lists, all of the same type, and
concatenates them into a single list.

``` {.screen}
ghci> :type concat
concat :: [[a]] -> [a]
ghci> concat [[1,2,3], [4,5,6]]
[1,2,3,4,5,6]
```

It removes one level of nesting.

``` {.screen}
ghci> concat [[[1,2],[3]], [[4],[5],[6]]]
[[1,2],[3],[4],[5],[6]]
ghci> concat (concat [[[1,2],[3]], [[4],[5],[6]]])
[1,2,3,4,5,6]
```

The `reverse` function returns the elements of a list in reverse order.

``` {.screen}
ghci> :type reverse
reverse :: [a] -> [a]
ghci> reverse "foo"
"oof"
```

For lists of `Bool`, the `and` and `or` functions generalise their
two-argument cousins, `(&&)` and `(||)`, over lists.

``` {.screen}
ghci> :type and
and :: [Bool] -> Bool
ghci> and [True,False,True]
False
ghci> and []
True
ghci> :type or
or :: [Bool] -> Bool
ghci> or [False,False,False,True,False]
True
ghci> or []
False
```

They have more useful cousins, `all` and `any`, which operate on lists
of any type. Each one takes a predicate as its first argument; `all`
returns `True` if that predicate succeeds on every element of the list,
while `any` returns `True` if the predicate succeeds on at least one
element of the list.

``` {.screen}
ghci> :type all
all :: (a -> Bool) -> [a] -> Bool
ghci> all odd [1,3,5]
True
ghci> all odd [3,1,4,1,5,9,2,6,5]
False
ghci> all odd []
True
ghci> :type any
any :: (a -> Bool) -> [a] -> Bool
ghci> any even [3,1,4,1,5,9,2,6,5]
True
ghci> any even []
False
```

### Working with sublists

The `take` function, which we already met in [the section called
\"Function
application\"](2-types-and-functions.org::*Function application)
consisting of the first *k* elements from a list. Its converse, `drop`,
drops *k* elements from the start of the list.

``` {.screen}
ghci> :type take
take :: Int -> [a] -> [a]
ghci> take 3 "foobar"
"foo"
ghci> take 2 [1]
[1]
ghci> :type drop
drop :: Int -> [a] -> [a]
ghci> drop 3 "xyzzy"
"zy"
ghci> drop 1 []
[]
```

The `splitAt` function combines the functions of `take` and `drop`,
returning a pair of the input list, split at the given index.

``` {.screen}
ghci> :type splitAt
splitAt :: Int -> [a] -> ([a], [a])
ghci> splitAt 3 "foobar"
("foo","bar")
```

The `takeWhile` and `dropWhile` functions take predicates: `takeWhile`
takes elements from the beginning of a list as long as the predicate
returns `True`, while `dropWhile` drops elements from the list as long
as the predicate returns `True`.

``` {.screen}
ghci> :type takeWhile
takeWhile :: (a -> Bool) -> [a] -> [a]
ghci> takeWhile odd [1,3,5,6,8,9,11]
[1,3,5]
ghci> :type dropWhile
dropWhile :: (a -> Bool) -> [a] -> [a]
ghci> dropWhile even [2,4,6,7,9,10,12]
[7,9,10,12]
```

Just as `splitAt` \"tuples up\" the results of `take` and `drop`, the
functions `break` (which we already saw in [the section called \"Warming
up: portably splitting lines of
text\"](4-functional-programming.org::*Warming up: portably splitting lines of text)
and `span` tuple up the results of `takeWhile` and `dropWhile`.

Each function takes a predicate; `break` consumes its input while its
predicate fails, while `span` consumes while its predicate succeeds.

``` {.screen}
ghci> :type span
span :: (a -> Bool) -> [a] -> ([a], [a])
ghci> span even [2,4,6,7,9,10,11]
([2,4,6],[7,9,10,11])
ghci> :type break
break :: (a -> Bool) -> [a] -> ([a], [a])
ghci> break even [1,3,5,6,8,9,10]
([1,3,5],[6,8,9,10])
```

### Searching lists

As we\'ve already seen, the `elem` function indicates whether a value is
present in a list. It has a companion function, `notElem`.

``` {.screen}
ghci> :type elem
elem :: (Eq a) => a -> [a] -> Bool
ghci> 2 `elem` [5,3,2,1,1]
True
ghci> 2 `notElem` [5,3,2,1,1]
False
```

For a more general search, `filter` takes a predicate, and returns every
element of the list on which the predicate succeeds.

``` {.screen}
ghci> :type filter
filter :: (a -> Bool) -> [a] -> [a]
ghci> filter odd [2,4,1,3,6,8,5,7]
[1,3,5,7]
```

In `Data.List`, three predicates, `isPrefixOf`, `isInfixOf`, and
`isSuffixOf`, let us test for the presence of sublists within a bigger
list. The easiest way to use them is using infix notation.

The `isPrefixOf` function tells us whether its left argument matches the
beginning of its right argument.

``` {.screen}
ghci> :module +Data.List
ghci> :type isPrefixOf
isPrefixOf :: (Eq a) => [a] -> [a] -> Bool
ghci> "foo" `isPrefixOf` "foobar"
True
ghci> [1,2] `isPrefixOf` []
False
```

The `isInfixOf` function indicates whether its left argument is a
sublist of its right.

``` {.screen}
ghci> :module +Data.List
ghci> [2,6] `isInfixOf` [3,1,4,1,5,9,2,6,5,3,5,8,9,7,9]
True
ghci> "funk" `isInfixOf` "sonic youth"
False
```

The operation of `isSuffixOf` shouldn\'t need any explanation.

``` {.screen}
ghci> :module +Data.List
ghci> ".c" `isSuffixOf` "crashme.c"
True
```

### Working with several lists at once

The `zip` function takes two lists and \"zips\" them into a single list
of pairs. The resulting list is the same length as the shorter of the
two inputs.

``` {.screen}
ghci> :type zip
zip :: [a] -> [b] -> [(a, b)]
ghci> zip [12,72,93] "zippity"
[(12,'z'),(72,'i'),(93,'p')]
```

More useful is `zipWith`, which takes two lists and applies a function
to each pair of elements, generating a list that is the same length as
the shorter of the two.

``` {.screen}
ghci> :type zipWith
zipWith :: (a -> b -> c) -> [a] -> [b] -> [c]
ghci> zipWith (+) [1,2,3] [4,5,6]
[5,7,9]
```

Haskell\'s type system makes it an interesting challenge to write
functions that take variable numbers of arguments[^1]. So if we want to
zip three lists together, we call `zip3` or `zipWith3`, and so on up to
`zip7` and `zipWith7`.

### Special string-handling functions

We\'ve already encountered the standard `lines` function in [the section
called \"Warming up: portably splitting lines of
text\"](4-functional-programming.org::*Warming up: portably splitting lines of text)
and its standard counterpart, `unlines`. Notice that `unlines` always
places a newline on the end of its result.

``` {.screen}
ghci> lines "foo\nbar"
["foo","bar"]
ghci> unlines ["foo", "bar"]
"foo\nbar\n"
```

The `words` function splits an input string on any white space. Its
counterpart, `unwords`, uses a single space to join a list of words.

``` {.screen}
ghci> words "the  \r  quick \t  brown\n\n\nfox"
["the","quick","brown","fox"]
ghci> unwords ["jumps", "over", "the", "lazy", "dog"]
"jumps over the lazy dog"
```

### Exercises

1.  Write your own \"safe\" definitions of the standard partial list
    functions, but make sure that yours never fail. As a hint, you might
    want to consider using the following types.

    <div>

    [exercises.hs]{.label}
    ``` {.haskell}
    safeHead :: [a] -> Maybe a
    safeTail :: [a] -> Maybe [a]
    safeLast :: [a] -> Maybe a
    safeInit :: [a] -> Maybe [a]
    ```

    </div>

2.  Write a function `splitWith` that acts similarly to `words`, but
    takes a predicate and a list of any type, and splits its input list
    on every element for which the predicate returns `False`.

    <div>

    [exercises.hs]{.label}
    ``` {.haskell}
    splitWith :: (a -> Bool) -> [a] -> [[a]]
    ```

    </div>

3.  Using the command framework from [the section called \"A simple
    command line
    framework\"](4-functional-programming.org::*A simple command line framework)
    program that prints the first word of each line of its input.

4.  Write a program that transposes the text in a file. For instance, it
    should convert `"hello\nworld\n"` to `"hw\neo\nlr\nll\nod\n"`.

How to think about loops
------------------------

Unlike traditional languages, Haskell has neither a `for` loop nor a
`while` loop. If we\'ve got a lot of data to process, what do we use
instead? There are several possible answers to this question.

### Explicit recursion

``` {.c org-language="C"}
int as_int(char *str)
{
    int acc; /* accumulate the partial result */

    for (acc = 0; isdigit(*str); str++) {
    acc = acc * 10 + (*str - '0');
    }

    return acc;
}
```

Given that Haskell doesn\'t have any looping constructs, how should we
think about representing a fairly straightforward piece of code like
this?

We don\'t have to start off by writing a type signature, but it helps to
remind us of what we\'re working with.

<div>

[IntParse.hs]{.label}

``` {.haskell}
import Data.Char (digitToInt) -- we'll need ord shortly

asInt :: String -> Int
```

</div>

The C code computes the result incrementally as it traverses the string;
the Haskell code can do the same. However, in Haskell, we can express
the equivalent of a loop as a function. We\'ll call ours `loop` just to
keep things nice and explicit.

<div>

[IntParse.hs]{.label}

``` {.haskell}
loop :: Int -> String -> Int

asInt xs = loop 0 xs
```

</div>

That first parameter to `loop` is the accumulator variable we\'ll be
using. Passing zero into it is equivalent to initialising the `acc`
variable in C at the beginning of the loop.

Rather than leap into blazing code, let\'s think about the data we have
to work with. Our familiar `String` is just a synonym for `[Char]`, a
list of characters. The easiest way for us to get the traversal right is
to think about the structure of a list: it\'s either empty, or a single
element followed by the rest of the list.

We can express this structural thinking directly by pattern matching on
the list type\'s constructors. It\'s often handy to think about the easy
cases first: here, that means we will consider the empty-list case.

<div>

[IntParse.hs]{.label}

``` {.haskell}
loop acc [] = acc
```

</div>

An empty list doesn\'t just mean \"the input string is empty\"; it\'s
also the case we\'ll encounter when we traverse all the way to the end
of a non-empty list. So we don\'t want to \"error out\" if we see an
empty list. Instead, we should do something sensible. Here, the sensible
thing is to terminate the loop, and return our accumulated value.

The other case we have to consider arises when the input list is not
empty. We need to do something with the current element of the list, and
something with the rest of the list.

<div>

[IntParse.hs]{.label}

``` {.haskell}
loop acc (x:xs) = let acc' = acc * 10 + digitToInt x
                  in loop acc' xs
```

</div>

We compute a new value for the accumulator, and give it the name `acc'`.
We then call the `loop` function again, passing it the updated value
`acc'` and the rest of the input list; this is equivalent to the loop
starting another round in C.

::: {.NOTE}
Single quotes in variable names

Remember, a single quote is a legal character to use in a Haskell
variable name, and is pronounced \"prime\". There\'s a common idiom in
Haskell programs involving a variable, say `foo`, and another variable,
say `foo'`. We can usually assume that `foo'` is somehow related to
`foo`. It\'s often a new value for `foo`, as in our code above.

Sometimes we\'ll see this idiom extended, such as `foo''`. Since keeping
track of the number of single quotes tacked onto the end of a name
rapidly becomes tedious, use of more than two in a row is thankfully
rare. Indeed, even one single quote can be easy to miss, which can lead
to confusion on the part of readers. It might be better to think of the
use of single quotes as a coding convention that you should be able to
recognize, and less as one that you should actually follow.
:::

Each time the `loop` function calls itself, it has a new value for the
accumulator, and it consumes one element of the input list. Eventually,
it\'s going to hit the end of the list, at which time the `[]` pattern
will match, and the recursive calls will cease.

How well does this function work? For positive integers, it\'s perfectly
cromulent.

``` {.screen}
ghci> asInt "33"
33
```

But because we were focusing on how to traverse lists, not error
handling, our poor function misbehaves if we try to feed it nonsense.

``` {.screen}
ghci> asInt ""
0
ghci> asInt "potato"
*** Exception: Char.digitToInt: not a digit 'p'
```

We\'ll defer fixing our function\'s shortcomings to *Q:1*.

Because the last thing that `loop` does is simply call itself, it\'s an
example of a tail recursive function. There\'s another common idiom in
this code, too. Thinking about the structure of the list, and handling
the empty and non-empty cases separately, is a kind of approach called
*structural recursion*.

We call the non-recursive case (when the list is empty) the *base case*
(sometimes the *terminating case*). We\'ll see people refer to the case
where the function calls itself as the recursive case (surprise!), or
they might give a nod to mathematical induction and call it the
*inductive case*.

As a useful technique, structural recursion is not confined to lists; we
can use it on other algebraic data types, too. We\'ll have more to say
about it later.

::: {.NOTE}
In an imperative language, a loop executes in constant space. Lacking
loops, we use tail recursive functions in Haskell instead. Normally, a
recursive function allocates some space each time it applies itself, so
it knows where to return to.

Clearly, a recursive function would be at a huge disadvantage relative
to a loop if it allocated memory for every recursive application: this
would require linear space instead of constant space. However,
functional language implementations detect uses of tail recursion, and
transform tail recursive calls to run in constant space; this is called
*tail call optimisation*, abbreviated TCO.

Few imperative language implementations perform TCO; this is why using
any kind of ambitiously functional style in an imperative language often
leads to memory leaks and poor performance.
:::

### Transforming every piece of input

Consider another C function, `square`, which squares every element in an
array.

``` {.c org-language="C"}
void square(double *out, const double *in, size_t length)
{
    for (size_t i = 0; i < length; i++) {
        out[i] = in[i] * in[i];
    }
}
```

This contains a straightforward and common kind of loop, one that does
exactly the same thing to every element of its input array. How might we
write this loop in Haskell?

<div>

[Map.hs]{.label}

``` {.haskell}
square :: [Double] -> [Double]

square (x:xs) = x*x : square xs
square []     = []
```

</div>

Our `square` function consists of two pattern matching equations. The
first \"deconstructs\" the beginning of a non-empty list, to get its
head and tail. It squares the first element, then puts that on the front
of a new list, which is constructed by calling `square` on the remainder
of the empty list. The second equation ensures that `square` halts when
it reaches the end of the input list.

The effect of `square` is to construct a new list that\'s the same
length as its input list, with every element in the input list
substituted with its square in the output list.

Here\'s another such C loop, one that ensures that every letter in a
string is converted to uppercase.

``` {.c org-language="C"}
#include <ctype.h>

char *uppercase(const char *in)
{
    char *out = strdup(in);

    if (out != NULL) {
        for (size_t i = 0; out[i] != '\0'; i++) {
            out[i] = toupper(out[i]);
        }
    }

    return out;
}
```

Let\'s look at a Haskell equivalent.

<div>

[Map.hs]{.label}

``` {.haskell}
import Data.Char (toUpper)

upperCase :: String -> String

upperCase (x:xs) = toUpper x : upperCase xs
upperCase []     = []
```

</div>

Here, we\'re importing the `toUpper` function from the standard
`Data.Char` module, which contains lots of useful functions for working
with `Char` data.

Our `upperCase` function follows a similar pattern to our earlier
`square` function. It terminates with an empty list when the input list
is empty; and when the input isn\'t empty, it calls `toUpper` on the
first element, then constructs a new list cell from that and the result
of calling itself on the rest of the input list.

These examples follow a common pattern for writing recursive functions
over lists in Haskell. The *base case* handles the situation where our
input list is empty. The *recursive case* deals with a non-empty list;
it does something with the head of the list, and calls itself
recursively on the tail.

### Mapping over a list

The `square` and `upperCase` functions that we just defined produce new
lists that are the same lengths as their input lists, and do only one
piece of work per element. This is such a common pattern that Haskell\'s
prelude defines a function, `map`, to make it easier. `map` takes a
function, and applies it to every element of a list, returning a new
list constructed from the results of these applications.

Here are our `square` and `upperCase` functions rewritten to use `map`.

<div>

[Map.hs]{.label}

``` {.haskell}
square2 xs = map squareOne xs
    where squareOne x = x * x

upperCase2 xs = map toUpper xs
```

</div>

This is our first close look at a function that takes another function
as its argument. We can learn a lot about what `map` does by simply
inspecting its type.

``` {.screen}
ghci> :type map
map :: (a -> b) -> [a] -> [b]
```

The signature tells us that `map` takes two arguments. The first is a
function that takes a value of one type, `a`, and returns a value of
another type, `b`.

Since `map` takes a function as argument, we refer to it as a
*higher-order* function. (In spite of the name, there\'s nothing
mysterious about higher-order functions; it\'s just a term for functions
that take other functions as arguments, or return functions.)

Since `map` abstracts out the pattern common to our `square` and
`upperCase` functions so that we can reuse it with less boilerplate, we
can look at what those functions have in common and figure out how to
implement it ourselves.

<div>

[Map.hs]{.label}

``` {.haskell}
myMap :: (a -> b) -> [a] -> [b]

myMap f (x:xs) = f x : myMap f xs
myMap _ _      = []
```

</div>

::: {.NOTE}
What are those wild cards doing there?

If you\'re new to functional programming, the reasons for matching
patterns in certain ways won\'t always be obvious. For example, in the
definition of `myMap` above, the first equation binds the function
we\'re mapping to the variable `f`, but the second uses wild cards for
both parameters. What\'s going on?

We use a wild card in place of `f` to indicate that we aren\'t calling
the function `f` on the right hand side of the equation. What about the
list parameter? The list type has two constructors. We\'ve already
matched on the non-empty constructor in the first equation that defines
`myMap`. By elimination, the constructor in the second equation is
necessarily the empty list constructor, so there\'s no need to perform a
match to see what its value really is.

As a matter of style, it is fine to use wild cards for well known simple
types like lists and `Maybe`. For more complicated or less familiar
types, it can be safer and more readable to name constructors
explicitly.
:::

We try out our `myMap` function to give ourselves some assurance that it
behaves similarly to the standard `map`.

``` {.screen}
ghci> :module +Data.Char
ghci> map toLower "SHOUTING"
"shouting"
ghci> myMap toUpper "whispering"
"WHISPERING"
ghci> map negate [1,2,3]
[-1,-2,-3]
```

This pattern of spotting a repeated idiom, then abstracting it so we can
reuse (and write less!) code, is a common aspect of Haskell programming.
While abstraction isn\'t unique to Haskell, higher order functions make
it remarkably easy.

### Selecting pieces of input

Another common operation on a sequence of data is to comb through it for
elements that satisfy some criterion. Here\'s a function that walks a
list of numbers and returns those that are odd. Our code has a recursive
case that\'s a bit more complex than our earlier functions: it only puts
a number in the list it returns if the number is odd. Using a guard
expresses this nicely.

<div>

[Filter.hs]{.label}

``` {.haskell}
oddList :: [Int] -> [Int]

oddList (x:xs) | odd x     = x : oddList xs
               | otherwise = oddList xs
oddList _                  = []
```

</div>

Let\'s see that in action.

``` {.screen}
ghci> oddList [1,1,2,3,5,8,13,21,34]
[1,1,3,5,13,21]
```

Once again, this idiom is so common that the prelude defines a function,
`filter`, which we have already introduced. It removes the need for
boilerplate code to recurse over the list.

``` {.screen}
ghci> :type filter
filter :: (a -> Bool) -> [a] -> [a]
ghci> filter odd [3,1,4,1,5,9,2,6,5]
[3,1,1,5,9,5]
```

The `filter` function takes a predicate and applies it to every element
in its input list, returning a list of only those for which the
predicate evaluates to `True`. We\'ll revisit `filter` again soon, in
[the section called \"Folding from the
right\"](4-functional-programming.org::*Folding from the right)

### Computing one answer over a collection

Another common thing to do with a collection is reduce it to a single
value. A simple example of this is summing the values of a list.

<div>

[Sum.hs]{.label}

``` {.haskell}
mySum xs = helper 0 xs
    where helper acc (x:xs) = helper (acc + x) xs
          helper acc _      = acc
```

</div>

Our `helper` function is tail recursive, and uses an accumulator
parameter, `acc`, to hold the current partial sum of the list. As we
already saw with `asInt`, this is a \"natural\" way to represent a loop
in a pure functional language.

For something a little more complicated, let\'s take a look at the
Adler-32 checksum. This is a popular checksum algorithm; it concatenates
two 16-bit checksums into a single 32-bit checksum. The first checksum
is the sum of all input bytes, plus one. The second is the sum of all
intermediate values of the first checksum. In each case, the sums are
computed modulo 65521. Here\'s a straightforward, unoptimised Java
implementation. (It\'s safe to skip it if you don\'t read Java.)

``` {.java}
public class Adler32
{
    private static final int base = 65521;

    public static int compute(byte[] data, int offset, int length)
    {
        int a = 1, b = 0;

        for (int i = offset; i < offset + length; i++) {
            a = (a + (data[i] & 0xff)) % base;
            b = (a + b) % base;
        }

        return (b << 16) | a;
    }
}
```

Although Adler-32 is a simple checksum, this code isn\'t particularly
easy to read on account of the bit-twiddling involved. Can we do any
better with a Haskell implementation?

<div>

[Adler32.hs]{.label}

``` {.haskell}
import Data.Char (ord)
import Data.Bits (shiftL, (.&.), (.|.))

base = 65521

adler32 xs = helper 1 0 xs
    where helper a b (x:xs) = let a' = (a + (ord x .&. 0xff)) `mod` base
                                  b' = (a' + b) `mod` base
                              in helper a' b' xs
          helper a b _      = (b `shiftL` 16) .|. a
```

</div>

This code isn\'t exactly easier to follow than the Java code, but let\'s
look at what\'s going on. First of all, we\'ve introduced some new
functions. The `shiftL` function implements a logical shift left;
`(.&.)` provides bitwise \"and\"; and `(.|.)` provides bitwise \"or\".

Once again, our `helper` function is tail recursive. We\'ve turned the
two variables we updated on every loop iteration in Java into
accumulator parameters. When our recursion terminates on the end of the
input list, we compute our checksum and return it.

If we take a step back, we can restructure our Haskell `adler32` to more
closely resemble our earlier `mySum` function. Instead of two
accumulator parameters, we can use a pair as the accumulator.

<div>

[Adler32.hs]{.label}

``` {.haskell}
adler32_try2 xs = helper (1,0) xs
    where helper (a,b) (x:xs) =
              let a' = (a + (ord x .&. 0xff)) `mod` base
                  b' = (a' + b) `mod` base
              in helper (a',b') xs
          helper (a,b) _      = (b `shiftL` 16) .|. a
```

</div>

Why would we want to make this seemingly meaningless structural change?
Because as we\'ve already seen with `map` and `filter`, we can extract
the common behavior shared by `mySum` and `adler32_try2` into a
higher-order function. We can describe this behavior as \"do something
to every element of a list, updating an accumulator as we go, and
returning the accumulator when we\'re done\".

This kind of function is called a *fold*, because it \"folds up\" a
list. There are two kinds of fold over lists, `foldl` for folding from
the left (the start) and `foldr` for folding from the right (the end).

### The left fold

Here is the definition of `foldl`.

<div>

[Fold.hs]{.label}

``` {.haskell}
foldl :: (a -> b -> a) -> a -> [b] -> a

foldl step zero (x:xs) = foldl step (step zero x) xs
foldl _    zero []     = zero
```

</div>

The `foldl` function takes a \"step\" function, an initial value for its
accumulator, and a list. The \"step\" takes an accumulator and an
element from the list, and returns a new accumulator value. All `foldl`
does is call the \"stepper\" on the current accumulator and an element
of the list, and passes the new accumulator value to itself recursively
to consume the rest of the list.

We refer to `foldl` as a \"left fold\" because it consumes the list from
left (the head) to right.

Here\'s a rewrite of `mySum` using `foldl`.

<div>

[Sum.hs]{.label}

``` {.haskell}
foldlSum xs = foldl step 0 xs
    where step acc x = acc + x
```

</div>

That local function `step` just adds two numbers, so let\'s simply use
the addition operator instead, and eliminate the unnecessary `where`
clause.

<div>

[Sum.hs]{.label}

``` {.haskell}
niceSum :: [Integer] -> Integer
niceSum xs = foldl (+) 0 xs
```

</div>

Notice how much simpler this code is than our original `mySum`? We\'re
no longer using explicit recursion, because `foldl` takes care of that
for us. We\'ve simplified our problem down to two things: what the
initial value of the accumulator should be (the second parameter to
`foldl`), and how to update the accumulator (the `(+)` function). As an
added bonus, our code is now shorter, too, which makes it easier to
understand.

Let\'s take a deeper look at what `foldl` is doing here, by manually
writing out each step in its evaluation when we call `niceSum [1,2,3]`.

<div>

[Fold.hs]{.label}

``` {.haskell}
foldl (+) 0 (1:2:3:[])
          == foldl (+) (0 + 1)             (2:3:[])
          == foldl (+) ((0 + 1) + 2)       (3:[])
          == foldl (+) (((0 + 1) + 2) + 3) []
          ==           (((0 + 1) + 2) + 3)
```

</div>

We can rewrite `adler32_try2` using `foldl` to let us focus on the
details that are important.

<div>

[Adler32.hs]{.label}

``` {.haskell}
adler32_foldl xs = let (a, b) = foldl step (1, 0) xs
                   in (b `shiftL` 16) .|. a
    where step (a, b) x = let a' = a + (ord x .&. 0xff)
                          in (a' `mod` base, (a' + b) `mod` base)
```

</div>

Here, our accumulator is a pair, so the result of `foldl` will be, too.
We pull the final accumulator apart when `foldl` returns, and
bit-twiddle it into a \"proper\" checksum.

### Why use folds, maps, and filters?

A quick glance reveals that `adler32_foldl` isn\'t really any shorter
than `adler32_try2`. Why should we use a fold in this case? The
advantage here lies in the fact that folds are extremely common in
Haskell, and they have regular, predictable behavior.

This means that a reader with a little experience will have an easier
time understanding a use of a fold than code that uses explicit
recursion. A fold isn\'t going to produce any surprises, but the
behavior of a function that recurses explicitly isn\'t immediately
obvious. Explicit recursion requires us to read closely to understand
exactly what\'s going on.

This line of reasoning applies to other higher-order library functions,
including those we\'ve already seen, `map` and `filter`. Because
they\'re library functions with well-defined behavior, we only need to
learn what they do once, and we\'ll have an advantage when we need to
understand any code that uses them. These improvements in readability
also carry over to writing code. Once we start to think with higher
order functions in mind, we\'ll produce concise code more quickly.

### Folding from the right

The counterpart to `foldl` is `foldr`, which folds from the right of a
list.

<div>

[Fold.hs]{.label}

``` {.haskell}
foldr :: (a -> b -> b) -> b -> [a] -> b
foldr step zero (x:xs) = step x (foldr step zero xs)
foldr _    zero []     = zero
```

</div>

Let\'s follow the same manual evaluation process with
`foldr (+) 0 [1,2,3]` as we did with `niceSum` in [the section called
\"The left fold\"](4-functional-programming.org::*The left fold)

<div>

[Fold.hs]{.label}

``` {.haskell}
foldr (+) 0 (1:2:3:[])
          == 1 +           foldr (+) 0 (2:3:[])
          == 1 + (2 +      foldr (+) 0 (3:[])
          == 1 + (2 + (3 + foldr (+) 0 []))
          == 1 + (2 + (3 + 0))
```

</div>

The difference between `foldl` and `foldr` should be clear from looking
at where the parentheses and the \"empty list\" elements show up. With
`foldl`, the empty list element is on the left, and all the parentheses
group to the left. With `foldr`, the `zero` value is on the right, and
the parentheses group to the right.

There is a lovely intuitive explanation of how `foldr` works: it
replaces the empty list with the `zero` value, and every constructor in
the list with an application of the step function.

<div>

[Fold.hs]{.label}

``` {.haskell}
1 : (2 : (3 : []))
1 + (2 + (3 + 0 ))
```

</div>

At first glance, `foldr` might seem less useful than `foldl`: what use
is a function that folds from the right? But consider the prelude\'s
`filter` function, which we last encountered in [the section called
\"Selecting pieces of
input\"](4-functional-programming.org::*Selecting pieces of input)
`filter` using explicit recursion, it will look something like this.

<div>

[Filter.hs]{.label}

``` {.haskell}
filter :: (a -> Bool) -> [a] -> [a]
filter p []   = []
filter p (x:xs)
    | p x       = x : filter p xs
    | otherwise = filter p xs
```

</div>

Perhaps surprisingly, though, we can write `filter` as a fold, using
`foldr`.

<div>

[Filter.hs]{.label}

``` {.haskell}
myFilter p xs = foldr step [] xs
    where step x ys | p x       = x : ys
                    | otherwise = ys
```

</div>

This is the sort of definition that could cause us a headache, so let\'s
examine it in a little depth. Like `foldl`, `foldr` takes a function and
a base case (what to do when the input list is empty) as arguments. From
reading the type of `filter`, we know that our `myFilter` function must
return a list of the same type as it consumes, so the base case should
be a list of this type, and the `step` helper function must return a
list.

Since we know that `foldr` calls `step` on one element of the input list
at a time, with the accumulator as its second argument, what `step` does
must be quite simple. If the predicate returns `True`, it pushes that
element onto the accumulated list; otherwise, it leaves the list
untouched.

The class of functions that we can express using `foldr` is called
*primitive recursive*. A surprisingly large number of list manipulation
functions are primitive recursive. For example, here\'s `map` written in
terms of `foldr`.

<div>

[Fold.hs]{.label}

``` {.haskell}
myMap :: (a -> b) -> [a] -> [b]
myMap f xs = foldr step [] xs
    where step x ys = f x : ys
```

</div>

In fact, we can even write `foldl` using `foldr`!

<div>

[Fold.hs]{.label}

``` {.haskell}
myFoldl :: (a -> b -> a) -> a -> [b] -> a
myFoldl f z xs = foldr step id xs z
    where step x g a = g (f a x)
```

</div>

::: {.TIP}
Understanding foldl in terms of foldr

If you want to set yourself a solid challenge, try to follow the above
definition of `foldl` using `foldr`. Be warned: this is not trivial! You
might want to have the following tools at hand: some headache pills and
a glass of water, `ghci` (so that you can find out what the `id`
function does), and a pencil and paper.

You will want to follow the same manual evaluation process as we
outlined above to see what `foldl` and `foldr` were really doing. If you
get stuck, you may find the task easier after reading [the section
called \"Partial function application and
currying\"](4-functional-programming.org::*Partial function application and currying)
:::

Returning to our earlier intuitive explanation of what `foldr` does,
another useful way to think about it is that it *transforms* its input
list. Its first two arguments are \"what to do with each head/tail
element of the list\", and \"what to substitute for the end of the
list\".

The \"identity\" transformation with `foldr` thus replaces the empty
list with itself, and applies the list constructor to each head/tail
pair:

<div>

[Fold.hs]{.label}

``` {.haskell}
identity :: [a] -> [a]
identity xs = foldr (:) [] xs
```

</div>

It transforms a list into a copy of itself.

``` {.screen}
ghci> identity [1,2,3]
[1,2,3]
```

If `foldr` replaces the end of a list with some other value, this gives
us another way to look at Haskell\'s list append function, `(++)`.

``` {.screen}
ghci> [1,2,3] ++ [4,5,6]
[1,2,3,4,5,6]
```

All we have to do to append a list onto another is substitute that
second list for the end of our first list.

<div>

[Fold.hs]{.label}

``` {.haskell}
append :: [a] -> [a] -> [a]
append xs ys = foldr (:) ys xs
```

</div>

Let\'s try this out.

``` {.screen}
ghci> append [1,2,3] [4,5,6]
[1,2,3,4,5,6]
```

Here, we replace each list constructor with another list constructor,
but we replace the empty list with the list we want to append onto the
end of our first list.

As our extended treatment of folds should indicate, the `foldr` function
is nearly as important a member of our list-programming toolbox as the
more basic list functions we saw in [the section called \"Working with
lists\"](4-functional-programming.org::*Working with lists) produce a
list incrementally, which makes it useful for writing lazy data
processing code.

### Left folds, laziness, and space leaks

To keep our initial discussion simple, we used `foldl` throughout most
of this section. This is convenient for testing, but we will never use
`foldl` in practice.

The reason has to do with Haskell\'s non-strict evaluation. If we apply
`foldl (+) [1,2,3]`, it evaluates to the expression
`(((0 + 1) + 2) + 3)`. We can see this occur if we revisit the way in
which the function gets expanded.

<div>

[Fold.hs]{.label}

``` {.haskell}
foldl (+) 0 (1:2:3:[])
          == foldl (+) (0 + 1)             (2:3:[])
          == foldl (+) ((0 + 1) + 2)       (3:[])
          == foldl (+) (((0 + 1) + 2) + 3) []
          ==           (((0 + 1) + 2) + 3)
```

</div>

The final expression will not be evaluated to `6` until its value is
demanded. Before it is evaluated, it must be stored as a thunk. Not
surprisingly, a thunk is more expensive to store than a single number,
and the more complex the thunked expression, the more space it needs.
For something cheap like arithmetic, thunking an expresion is more
computationally expensive than evaluating it immediately. We thus end up
paying both in space and in time.

When GHC is evaluating a thunked expression, it uses an internal stack
to do so. Because a thunked expression could potentially be infinitely
large, GHC places a fixed limit on the maximum size of this stack.
Thanks to this limit, we can try a large thunked expression in `ghci`
without needing to worry that it might consume all of memory.

``` {.screen}
ghci> foldl (+) 0 [1..1000]
500500
```

From looking at the expansion above, we can surmise that this creates a
thunk that consists of 1000 integers and 999 applications of `(+)`.
That\'s a lot of memory and effort to represent a single number! With a
larger expression, although the size is still modest, the results are
more dramatic.

``` {.screen}
ghci> foldl (+) 0 [1..1000000]
*** Exception: stack overflow
```

On small expressions, `foldl` will work correctly but slowly, due to the
thunking overhead that it incurs. We refer to this invisible thunking as
a *space leak*, because our code is operating normally, but using far
more memory than it should.

On larger expressions, code with a space leak will simply fail, as
above. A space leak with `foldl` is a classic roadblock for new Haskell
programmers. Fortunately, this is easy to avoid.

The `Data.List` module defines a function named `foldl'` that is similar
to `foldl`, but does not build up thunks. The difference in behavior
between the two is immediately obvious.

``` {.screen}
ghci> foldl  (+) 0 [1..1000000]
*** Exception: stack overflow
ghci> :module +Data.List
ghci> foldl' (+) 0 [1..1000000]
500000500000
```

Due to the thunking behavior of `foldl`, it is wise to avoid this
function in real programs: even if it doesn\'t fail outright, it will be
unnecessarily inefficient. Instead, import `Data.List` and use `foldl'`.

### Exercises

1.  Use a fold (choosing the appropriate fold will make your code much
    simpler) to rewrite and improve upon the `asInt` function from [the
    section called \"Explicit
    recursion\"](4-functional-programming.org::*Explicit recursion)

    <div>

    [exercises.hs]{.label}
    ``` {.haskell}
    asInt_fold :: String -> Int
    ```

    </div>

    Your function should behave as follows.

    ``` {.screen}
    ghci> asInt_fold "101"
    ghci> asInt_fold "-31337"
    -31337
    ghci> asInt_fold "1798"
    1798
    ```

    Extend your function to handle the following kinds of exceptional
    conditions by calling `error`.

    ``` {.screen}
    ghci> asInt_fold ""
    0
    ghci> asInt_fold "-"
    0
    ghci> asInt_fold "-3"
    -3
    ghci> asInt_fold "2.7"
    *** Exception: Char.digitToInt: not a digit '.'
    ghci> asInt_fold "314159265358979323846"
    564616105916946374
    ```

2.  The `asInt_fold` function uses `error`, so its callers cannot handle
    errors. Rewrite it to fix this problem.

    <div>

    [exercises.hs]{.label}
    ``` {.haskell}
    type ErrorMessage = String
    asInt_either :: String -> Either ErrorMessage Int
    ```

    </div>

    ``` {.screen}
    ghci> asInt_either "33"
    Right 33
    ghci> asInt_either "foo"
    Left "non-digit 'o'"
    ```

3.  The Prelude function `concat` concatenates a list of lists into a
    single list, and has the following type.

    <div>

    [exercises.hs]{.label}
    ``` {.haskell}
    concat :: [[a]] -> [a]
    ```

    </div>

    Write your own definition of `concat` using `foldr`.

4.  Write your own definition of the standard `takeWhile` function,
    first using explicit recursion, then `foldr`.

5.  The `Data.List` module defines a function, `groupBy`, which has the
    following type.

    <div>

    [exercises.hs]{.label}
    ``` {.haskell}
    groupBy :: (a -> a -> Bool) -> [a] -> [[a]]
    ```

    </div>

    Use `ghci` to load the `Data.List` module and figure out what
    `groupBy` does, then write your own implementation using a fold.

6.  How many of the following prelude functions can you rewrite using
    list folds?

    -   `any`
    -   `cycle`
    -   `words`
    -   `unlines`

    For those functions where you can use either `foldl'` or `foldr`,
    which is more appropriate in each case?

### Further reading

The article \[[Hutton99](bibliography.org::Hutton99)\] is an excellent
and deep tutorial covering folds. It includes many examples of how to
use simple, systematic calculation techniques to turn functions that use
explicit recursion into folds.

Anonymous (lambda) functions
----------------------------

In many of the function definitions we\'ve seen so far, we\'ve written
short helper functions.

<div>

[Partial.hs]{.label}

``` {.haskell}
isInAny needle haystack = any inSequence haystack
    where inSequence s = needle `isInfixOf` s
```

</div>

Haskell lets us write completely anonymous functions, which we can use
to avoid the need to give names to our helper functions. Anonymous
functions are often called \"lambda\" functions, in a nod to their
heritage in the lambda calculus. We introduce an anonymous function with
a backslash character, `\`, pronounced *lambda*[^2]. This is followed by
the function\'s arguments (which can include patterns), then an arrow
`->` to introduce the function\'s body.

Lambdas are most easily illustrated by example. Here\'s a rewrite of
`isInAny` using an anonymous function.

<div>

[Partial.hs]{.label}

``` {.haskell}
isInAny2 needle haystack = any (\s -> needle `isInfixOf` s) haystack
```

</div>

We\'ve wrapped the lambda in parentheses here so that Haskell can tell
where the function body ends.

Anonymous functions behave in every respect identically to functions
that have names, but Haskell places a few important restrictions on how
we can define them. Most importantly, while we can write a normal
function using multiple clauses containing different patterns and
guards, a lambda can only have a single clause in its definition.

The limitation to a single clause restricts how we can use patterns in
the definition of a lambda. We\'ll usually write a normal function with
several clauses to cover different pattern matching possibilities.

<div>

[Lambda.hs]{.label}

``` {.haskell}
safeHead (x:_) = Just x
safeHead _ = Nothing
```

</div>

But as we can\'t write multiple clauses to define a lambda, we must be
certain that any patterns we use will match.

<div>

[Lambda.hs]{.label}

``` {.haskell}
unsafeHead = \(x:_) -> x
```

</div>

This definition of `unsafeHead` will explode in our faces if we call it
with a value on which pattern matching fails.

``` {.screen}
ghci> :type unsafeHead
unsafeHead :: [t] -> t
ghci> unsafeHead [1]
1
ghci> unsafeHead []
*** Exception: Lambda.hs:7:13-23: Non-exhaustive patterns in lambda
```

The definition type-checks, so it will compile, so the error will occur
at runtime. The moral of this story is to be careful in how you use
patterns when defining an anonymous function: make sure your patterns
can\'t fail!

Another thing to notice about the `isInAny` and `isInAny2` functions we
showed above is that the first version, using a helper function that has
a name, is a little easier to read than the version that plops an
anonymous function into the middle. The named helper function doesn\'t
disrupt the \"flow\" of the function in which it\'s used, and the
judiciously chosen name gives us a little bit of information about what
the function is expected to do.

In contrast, when we run across a lambda in the middle of a function
body, we have to switch gears and read its definition fairly carefully
to understand what it does. To help with readability and
maintainability, then, we tend to avoid lambdas in many situations where
we could use them to trim a few characters from a function definition.
Very often, we\'ll use a partially applied function instead, resulting
in clearer and more readable code than either a lambda or an explicit
function. Don\'t know what a partially applied function is yet? Read on!

We don\'t intend these caveats to suggest that lambdas are useless,
merely that we ought to be mindful of the potential pitfalls when we\'re
thinking of using them. In later chapters, we will see that they are
often invaluable as \"glue\".

Partial function application and currying
-----------------------------------------

You may wonder why the `->` arrow is used for what seems to be two
purposes in the type signature of a function.

``` {.screen}
ghci> :type dropWhile
dropWhile :: (a -> Bool) -> [a] -> [a]
```

It looks like the `->` is separating the arguments to `dropWhile` from
each other, but that it also separates the arguments from the return
type. But in fact `->` has only one meaning: it denotes a function that
takes an argument of the type on the left, and returns a value of the
type on the right.

The implication here is very important: in Haskell, *all functions take
only one argument*. While `dropWhile` *looks* like a function that takes
two arguments, it is actually a function of one argument, which returns
a function that takes one argument. Here\'s a perfectly valid Haskell
expression.

``` {.screen}
ghci> :module +Data.Char
ghci> :type dropWhile isSpace
dropWhile isSpace :: [Char] -> [Char]
```

Well, *that* looks useful. The value `dropWhile isSpace` is a function
that strips leading white space from a string. How is this useful? As
one example, we can use it as an argument to a higher order function.

``` {.screen}
ghci> map (dropWhile isSpace) [" a","f","   e"]
["a","f","e"]
```

Every time we supply an argument to a function, we can \"chop\" an
element off the front of its type signature. Let\'s take `zip3` as an
example to see what we mean; this is a function that zips three lists
into a list of three-tuples.

``` {.screen}
ghci> :type zip3
zip3 :: [a] -> [b] -> [c] -> [(a, b, c)]
ghci> zip3 "foo" "bar" "quux"
[('f','b','q'),('o','a','u'),('o','r','u')]
```

If we apply `zip3` with just one argument, we get a function that
accepts two arguments. No matter what arguments we supply to this
compound function, its first argument will always be the fixed value we
specified.

``` {.screen}
ghci> :type zip3 "foo"
zip3 "foo" :: [b] -> [c] -> [(Char, b, c)]
ghci> let zip3foo = zip3 "foo"
ghci> :type zip3foo
zip3foo :: [b] -> [c] -> [(Char, b, c)]
ghci> (zip3 "foo") "aaa" "bbb"
[('f','a','b'),('o','a','b'),('o','a','b')]
ghci> zip3foo "aaa" "bbb"
[('f','a','b'),('o','a','b'),('o','a','b')]
ghci> zip3foo [1,2,3] [True,False,True]
[('f',1,True),('o',2,False),('o',3,True)]
```

When we pass fewer arguments to a function than the function can accept,
we call this *partial application* of the function: we\'re applying the
function to only some of its arguments.

In the example above, we have a partially applied function,
`zip3 "foo"`, and a new function, `zip3foo`. We can see that the type
signatures of the two and their behavior are identical.

This applies just as well if we fix two arguments, giving us a function
of just one argument.

``` {.screen}
ghci> let zip3foobar = zip3 "foo" "bar"
ghci> :type zip3foobar
zip3foobar :: [c] -> [(Char, Char, c)]
ghci> zip3foobar "quux"
[('f','b','q'),('o','a','u'),('o','r','u')]
ghci> zip3foobar [1,2]
[('f','b',1),('o','a',2)]
```

Partial function application lets us avoid writing tiresome throwaway
functions. It\'s often more useful for this purpose than the anonymous
functions we introduced in [the section called \"Anonymous (lambda)
functions\"](4-functional-programming.org::*Anonymous (lambda) functions)
the `isInAny` function we defined there, here\'s how we\'d use a
partially applied function instead of a named helper function or a
lambda.

<div>

[Partial.hs]{.label}

``` {.haskell}
isInAny3 needle haystack = any (isInfixOf needle) haystack
```

</div>

Here, the expression `isInfixOf needle` is the partially applied
function. We\'re taking the function `isInfixOf`, and \"fixing\" its
first argument to be the `needle` variable from our parameter list. This
gives us a partially applied function that has exactly the same type and
behavior as the helper and lambda in our earlier definitions.

Partial function application is named *currying*, after the logician
Haskell Curry (for whom the Haskell language is named).

As another example of currying in use, let\'s return to the list-summing
function we wrote in [the section called \"The left
fold\"](4-functional-programming.org::*The left fold)

<div>

[Sum.hs]{.label}

``` {.haskell}
niceSum :: [Integer] -> Integer
niceSum xs = foldl (+) 0 xs
```

</div>

We don\'t need to fully apply `foldl`; we can omit the list `xs` from
both the parameter list and the parameters to `foldl`, and we\'ll end up
with a more compact function that has the same type.

<div>

[Sum.hs]{.label}

``` {.haskell}
nicerSum :: [Integer] -> Integer
nicerSum = foldl (+) 0
```

</div>

### Sections

Haskell provides a handy notational shortcut to let us write a partially
applied function in infix style. If we enclose an operator in
parentheses, we can supply its left or right argument inside the
parentheses to get a partially applied function. This kind of partial
application is called a *section*.

``` {.screen}
ghci> (1+) 2
3
ghci> map (*3) [24,36]
[72,108]
ghci> map (2^) [3,5,7,9]
[8,32,128,512]
```

If we provide the left argument inside the section, then calling the
resulting function with one argument supplies the operator\'s right
argument. And vice versa.

Recall that we can wrap a function name in backquotes to use it as an
infix operator. This lets us use sections with functions.

``` {.screen}
ghci> :type (`elem` ['a'..'z'])
(`elem` ['a'..'z']) :: Char -> Bool
```

The above definition fixes `elem`\'s second argument, giving us a
function that checks to see whether its argument is a lowercase letter.

``` {.screen}
ghci> (`elem` ['a'..'z']) 'f'
True
```

Using this as an argument to `all`, we get a function that checks an
entire string to see if it\'s all lowercase.

``` {.screen}
ghci> all (`elem` ['a'..'z']) "Frobozz"
False
```

If we use this style, we can further improve the readability of our
earlier `isInAny3` function.

<div>

[Partial.hs]{.label}

``` {.haskell}
isInAny4 needle haystack = any (needle `isInfixOf`) haystack
```

</div>

As-patterns
-----------

Haskell\'s `tails` function, in the `Data.List` module, generalises the
`tail` function we introduced earlier. Instead of returning one \"tail\"
of a list, it returns *all* of them.

``` {.screen}
ghci> :m +Data.List
ghci> tail "foobar"
"oobar"
ghci> tail (tail "foobar")
"obar"
ghci> tails "foobar"
["foobar","oobar","obar","bar","ar","r",""]
```

Each of these strings is a *suffix* of the initial string, so `tails`
produces a list of all suffixes, plus an extra empty list at the end. It
always produces that extra empty list, even when its input list is
empty.

``` {.screen}
ghci> tails []
```

What if we want a function that behaves like `tails`, but which *only*
returns the non-empty suffixes? One possibility would be for us to write
our own version by hand. We\'ll use a new piece of notation, the `@`
symbol.

<div>

[SuffixTree.hs]{.label}

``` {.haskell}
suffixes :: [a] -> [[a]]
suffixes xs@(_:xs') = xs : suffixes xs'
suffixes _ = []
```

</div>

The pattern `xs @ (_ : xs')` is called an *as-pattern*, and it means
\"bind the variable `xs` to the value that matches the right side of the
`@` symbol\".

In our example, if the pattern after the \"@\" matches, `xs` will be
bound to the entire list that matched, and `xs'` to all but the head of
the list (we used the wild card `_` pattern to indicate that we\'re not
interested in the value of the head of the list).

``` {.screen}
ghci> tails "foo"
["foo","oo","o",""]
ghci> suffixes "foo"
["foo","oo","o"]
```

The as-pattern makes our code more readable. To see how it helps, let us
compare a definition that lacks an as-pattern.

<div>

[SuffixTree.hs]{.label}

``` {.haskell}
noAsPattern :: [a] -> [[a]]
noAsPattern (x:xs) = (x:xs) : noAsPattern xs
noAsPattern _ = []
```

</div>

Here, the list that we\'ve deconstructed in the pattern match just gets
put right back together in the body of the function.

As-patterns have a more practical use than simple readability: they can
help us to share data instead of copying it. In our definition of
`noAsPattern`, when we match `(x : xs)`, we construct a new copy of it
in the body of our function. This causes us to allocate a new list node
at run time. That may be cheap, but it isn\'t free. In contrast, when we
defined `suffixes`, we reused the value `xs` that we matched with our
as-pattern. Since we reuse an existing value, we avoid a little
allocation.

Code reuse through composition
------------------------------

It seems a shame to introduce a new function, `suffixes`, that does
almost the same thing as the existing `tails` function. Surely we can do
better?

Recall the `init` function we introduced in [the section called
\"Working with
lists\"](4-functional-programming.org::*Working with lists) last element
of a list.

<div>

[SuffixTree.hs]{.label}

``` {.haskell}
suffixes2 xs = init (tails xs)
```

</div>

This `suffixes2` function behaves identically to `suffixes`, but it\'s a
single line of code.

``` {.screen}
ghci> suffixes2 "foo"
["foo","oo","o"]
```

If we take a step back, we see the glimmer of a pattern here: we\'re
applying a function, then applying another function to its result.
Let\'s turn that pattern into a function definition.

<div>

[SuffixTree.hs]{.label}

``` {.haskell}
compose :: (b -> c) -> (a -> b) -> a -> c
compose f g x = f (g x)
```

</div>

We now have a function, `compose`, that we can use to \"glue\" two other
functions together.

<div>

[SuffixTree.hs]{.label}

``` {.haskell}
suffixes3 xs = compose init tails xs
```

</div>

Haskell\'s automatic currying lets us drop the `xs` variable, so we can
make our definition even shorter.

<div>

[SuffixTree.hs]{.label}

``` {.haskell}
suffixes4 = compose init tails
```

</div>

Fortunately, we don\'t need to write our own `compose` function.
Plugging functions into each other like this is so common that the
Prelude provides function composition via the `(.)` operator.

<div>

[SuffixTree.hs]{.label}

``` {.haskell}
suffixes5 = init . tails
```

</div>

The `(.)` operator isn\'t a special piece of language syntax; it\'s just
a normal operator.

``` {.screen}
ghci> :type (.)
(.) :: (b -> c) -> (a -> b) -> a -> c
ghci> :type suffixes
suffixes :: [a] -> [[a]]
ghci> :type suffixes5
suffixes5 :: [a] -> [[a]]
ghci> suffixes5 "foo"
["foo","oo","o"]
```

We can create new functions at any time by writing chains of composed
functions, stitched together with `(.)`, so long (of course) as the
result type of the function on the right of each `(.)` matches the type
of parameter that the function on the left can accept.

As an example, let\'s solve a simple puzzle: counting the number of
words in a string that begin with a capital letter.

``` {.screen}
ghci> :module +Data.Char
ghci> let capCount = length . filter (isUpper . head) . words
ghci> capCount "Hello there, Mom!"
2
```

We can understand what this composed function does by examining its
pieces. The `(.)` function is right associative, so we will proceed from
right to left.

``` {.screen}
ghci> :type words
words :: String -> [String]
```

The `words` function has a result type of `[String]`, so whatever is on
the left side of `(.)` must accept a compatible argument.

``` {.screen}
ghci> :type isUpper . head
isUpper . head :: [Char] -> Bool
```

This function returns `True` if a word begins with a capital letter (try
it in `ghci`), so `filter (isUpper . head)` returns a list of `Strings`
containing only words that begin with capital letters.

``` {.screen}
ghci> :type filter (isUpper . head)
filter (isUpper . head) :: [[Char]] -> [[Char]]
```

Since this expression returns a list, all that remains is calculate the
length of the list, which we do with another composition.

Here\'s another example, drawn from a real application. We want to
extract a list of macro names from a C header file shipped with
`libpcap`, a popular network packet filtering library. The header file
contains a large number definitions of the following form.

``` {.c org-language="C"}
#define DLT_EN10MB      1       /* Ethernet (10Mb) */
#define DLT_EN3MB       2       /* Experimental Ethernet (3Mb) */
#define DLT_AX25        3       /* Amateur Radio AX.25 */
```

Our goal is to extract names such as `DLT_EN10MB` and `DLT_AX25`.

<div>

[dlts.hs]{.label}

``` {.haskell}
import Data.List (isPrefixOf)

dlts :: String -> [String]

dlts = foldr step [] . lines
```

</div>

We treat an entire file as a string, split it up with `lines`, then
apply `foldr step []` to the resulting list of lines. The `step` helper
function operates on a single line.

<div>

[dlts.hs]{.label}

``` {.haskell}
where step l ds
        | "#define DLT_" `isPrefixOf` l = secondWord l : ds
        | otherwise                     = ds
      secondWord = head . tail . words
```

</div>

If we match a macro definition with our guard expression, we cons the
name of the macro onto the head of the list we\'re returning; otherwise,
we leave the list untouched.

While the individual functions in the body of `secondWord` are by now
familiar to us, it can take a little practice to piece together a chain
of compositions like this. Let\'s walk through the procedure.

Once again, we proceed from right to left. The first function is
`words`.

``` {.screen}
ghci> :type words
words :: String -> [String]
ghci> words "#define DLT_CHAOS    5"
["#define","DLT_CHAOS","5"]
```

We then apply `tail` to the result of `words`.

``` {.screen}
ghci> :type tail
tail :: [a] -> [a]
ghci> tail ["#define","DLT_CHAOS","5"]
["DLT_CHAOS","5"]
ghci> :type tail . words
tail . words :: String -> [String]
ghci> (tail . words) "#define DLT_CHAOS    5"
["DLT_CHAOS","5"]
```

Finally, applying `head` to the result of `drop 1 . words` will give us
the name of our macro.

``` {.screen}
ghci> :type head . tail . words
head . tail . words :: String -> String
ghci> (head . tail . words) "#define DLT_CHAOS    5"
"DLT_CHAOS"
```

### Use your head wisely

After warning against unsafe list functions in [the section called
\"Safely and sanely working with crashy
functions\"](4-functional-programming.org::*Safely and sanely working with crashy functions)
here we are calling both `head` and `tail`, two of those unsafe list
functions. What gives?

In this case, we can assure ourselves by inspection that we\'re safe
from a runtime failure. The pattern guard in the definition of `step`
contains two words, so when we apply `words` to any string that makes it
past the guard, we\'ll have a list of at least two elements, `"#define"`
and some macro beginning with `"DLT_"`.

This the kind of reasoning we ought to do to convince ourselves that our
code won\'t explode when we call partial functions. Don\'t forget our
earlier admonition: calling unsafe functions like this requires care,
and can often make our code more fragile in subtle ways. If we for some
reason modified the pattern guard to only contain one word, we could
expose ourselves to the possibility of a crash, as the body of the
function assumes that it will receive two words.

Tips for writing readable code
------------------------------

So far in this chapter, we\'ve come across two tempting looking features
of Haskell: tail recursion and anonymous functions. As nice as these
are, we don\'t often want to use them.

Many list manipulation operations can be most easily expressed using
combinations of library functions such as `map`, `take`, and `filter`.
Without a doubt, it takes some practice to get used to using these. In
return for our initial investment, we can write and read code more
quickly, and with fewer bugs.

The reason for this is simple. A tail recursive function definition has
the same problem as a loop in an imperative language: it\'s completely
general. It might perform some filtering, some mapping, or who knows
what else. We are forced to look in detail at the entire definition of
the function to see what it\'s really doing. In contrast, `map` and most
other list manipulation functions do only *one* thing. We can take for
granted what these simple building blocks do, and focus on the idea the
code is trying to express, not the minute details of how it\'s
manipulating its inputs.

In the middle ground between tail recursive functions (with complete
generality) and our toolbox of list manipulation functions (each of
which does one thing) lie the folds. A fold takes more effort to
understand than, say, a composition of `map` and `filter` that does the
same thing, but it behaves more regularly and predictably than a tail
recursive function. As a general rule, don\'t use a fold if you can
compose some library functions, but otherwise try to use a fold in
preference to a hand-rolled a tail recursive loop.

As for anonymous functions, they tend to interrupt the \"flow\" of
reading a piece of code. It is very often as easy to write a local
function definition in a `let` or `where` clause, and use that, as it is
to put an anonymous function into place. The relative advantages of a
named function are twofold: we don\'t need to understand the function\'s
definition when we\'re reading the code that uses it; and a well chosen
function name acts as a tiny piece of local documentation.

Space leaks and strict evaluation
---------------------------------

The `foldl` function that we discussed earlier is not the only place
where space leaks can arise in Haskell code. We will use it to
illustrate how non-strict evaluation can sometimes be problematic, and
how to solve the difficulties that can arise.

::: {.TIP}
Do you need to know all of this right now?

It is perfectly reasonable to skip this section until you encounter a
space leak \"in the wild\". Provided you use `foldr` if you are
generating a list, and `foldl'` instead of `foldl` otherwise, space
leaks are unlikely to bother you in practice for a while.
:::

### Avoiding space leaks with seq

We refer to an expression that is not evaluated lazily as *strict*, so
`foldl'` is a strict left fold. It bypasses Haskell\'s usual non-strict
evaluation through the use of a special function named `seq`.

<div>

[Fold.hs]{.label}

``` {.haskell}
foldl' _    zero []     = zero
foldl' step zero (x:xs) =
    let new = step zero x
    in  new `seq` foldl' step new xs
```

</div>

This `seq` function has a peculiar type, hinting that it is not playing
by the usual rules.

``` {.screen}
ghci> :type seq
seq :: a -> t -> t
```

It operates as follows: when a `seq` expression is evaluated, it forces
its first argument to be evaluated, then returns its second argument. It
doesn\'t actually do anything with the first argument: `seq` exists
solely as a way to force that value to be evaluated. Let\'s walk through
a brief application to see what happens.

<div>

[Fold.hs]{.label}

``` {.haskell}
foldl' (+) 1 (2:[])
```

</div>

This expands as follows.

<div>

[Fold.hs]{.label}

``` {.haskell}
let new = 1 + 2
in new `seq` foldl' (+) new []
```

</div>

The use of `seq` forcibly evaluates `new` to `3`, and returns its second
argument.

<div>

[Fold.hs]{.label}

``` {.haskell}
foldl' (+) 3 []
```

</div>

We end up with the following result.

<div>

[Fold.hs]{.label}

``` {.haskell}
3
```

</div>

Thanks to `seq`, there are no thunks in sight.

### Learning to use seq

Without some direction, there is an element of mystery to using `seq`
effectively. Here are some useful rules for using it well.

To have any effect, a `seq` expression must be the first thing evaluated
in an expression.

<div>

[Fold.hs]{.label}

``` {.haskell}
-- incorrect: seq is hidden by the application of someFunc
-- since someFunc will be evaluated first, seq may occur too late
hiddenInside x y = someFunc (x `seq` y)

-- incorrect: a variation of the above mistake
hiddenByLet x y z = let a = x `seq` someFunc y
                    in anotherFunc a z

-- correct: seq will be evaluated first, forcing evaluation of x
onTheOutside x y = x `seq` someFunc y
```

</div>

To strictly evaluate several values, chain applications of `seq`
together.

<div>

[Fold.hs]{.label}

``` {.haskell}
chained x y z = x `seq` y `seq` someFunc z
```

</div>

A common mistake is to try to use `seq` with two unrelated expressions.

<div>

[Fold.hs]{.label}

``` {.haskell}
badExpression step zero (x:xs) =
    seq (step zero x)
        (badExpression step (step zero x) xs)
```

</div>

Here, the apparent intention is to evaluate `step zero x` strictly.
Since the expression is duplicated in the body of the function, strictly
evaluating the first instance of it will have no effect on the second.
The use of `let` from the definition of `foldl'` above shows how to
achieve this effect correctly.

When evaluating an expression, `seq` stops as soon as it reaches a
constructor. For simple types like numbers, this means that it will
evaluate them completely. Algebraic data types are a different story.
Consider the value `(1 + 2) : (3 + 4) : []`. If we apply `seq` to this,
it will evaluate the `(1 + 2)` thunk. Since it will stop when it reaches
the first `(:)` constructor, it will have no effect on the second thunk.
The same is true for tuples: `seq ((1 + 2), (3 + 4)) True` will do
nothing to the thunks inside the pair, since it immediately hits the
pair\'s constructor.

If necessary, we can use normal functional programming techniques to
work around these limitations.

<div>

[Fold.hs]{.label}

``` {.haskell}
strictPair (a,b) = a `seq` b `seq` (a,b)
strictList (x:xs) = x `seq` x : strictList xs
strictList []     = []
```

</div>

It is important to understand that `seq` isn\'t free: it has to perform
a check at runtime to see if an expression has been evaluated. Use it
sparingly. For instance, while our `strictPair` function evaluates the
contents of a pair up to the first constructor, it adds the overheads of
pattern matching, two applications of `seq`, and the construction of a
new tuple. If we were to measure its performance in the inner loop of a
benchmark, we might find it to slow the program down.

Aside from its performance cost if overused, `seq` is not a miracle
cure-all for memory consumption problems. Just because you *can*
evaluate something strictly doesn\'t mean you *should*. Careless use of
`seq` may do nothing at all; move existing space leaks around; or
introduce new leaks.

The best guides to whether `seq` is necessary, and how well it is
working, are performance measurement and profiling, which we will cover
in [Chapter 25, *Profiling and
optimization*](25-profiling-and-optimization.org). From a base of
empirical measurement, you will develop a reliable sense of when `seq`
is most useful.

Footnotes
---------

[^1]: Unfortunately, we do not have room to address that challenge in
    this book.

[^2]: The backslash was chosen for its visual resemblance to the Greek
    letter lambda, `λ`. Although GHC can accept Unicode input, it
    correctly treats `λ` as a letter, not as a synonym for `\`.
