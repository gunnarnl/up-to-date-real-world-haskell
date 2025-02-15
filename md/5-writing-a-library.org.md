Chapter 5: Writing a library: working with JSON data
====================================================

A whirlwind tour of JSON
------------------------

In this chapter, we\'ll develop a small, but complete, Haskell library.
Our library will manipulate and serialize data in a popular form known
as JSON.

The JSON (JavaScript Object Notation) language is a small, simple
representation for storing and transmitting structured data, for example
over a network connection. It is most commonly used to transfer data
from a web service to a browser-based JavaScript application. The JSON
format is described at [www.json.org](http://www.json.org/), and in
greater detail by [RFC 4627](http://www.ietf.org/rfc/rfc4627.txt).

JSON supports four basic types of value: strings, numbers, booleans, and
a special value named `null`.

``` {.haskell}
"a string" 12345 true
      null
```

The language provides two compound types: an *array* is an ordered
sequence of values, and an *object* is an unordered collection of
name/value pairs. The names in an object are always strings; the values
in an object or array can be of any type.

``` {.haskell}
[-3.14, true, null, "a string"]
      {"numbers": [1,2,3,4,5], "useful": false}
```

Representing JSON data in Haskell
---------------------------------

To work with JSON data in Haskell, we use an algebraic data type to
represent the range of possible JSON types.

<div>

[SimpleJSON.hs]{.label}

``` {.haskell}
data JValue = JString String
            | JNumber Double
            | JBool Bool
            | JNull
            | JObject [(String, JValue)]
            | JArray [JValue]
              deriving (Eq, Ord, Show)
```

</div>

For each JSON type, we supply a distinct value constructor. Some of
these constructors have parameters: if we want to construct a JSON
string, we must provide a `String` value as an argument to the `JString`
constructor.

To start experimenting with this code, save the file `SimpleJSON.hs` in
your editor, switch to a `ghci` window, and load the file into `ghci`.

``` {.screen}
ghci> :load SimpleJSON
[1 of 1] Compiling SimpleJSON       ( SimpleJSON.hs, interpreted )
Ok, one module loaded.
ghci> JString "foo"
JString "foo"
ghci> JNumber 2.7
JNumber 2.7
ghci> :type JBool True
JBool True :: JValue
```

We can see how to use a constructor to take a normal Haskell value and
turn it into a `JValue`. To do the reverse, we use pattern matching.
Here\'s a function that we can add to `SimpleJSON.hs` that will extract
a string from a JSON value for us. If the JSON value actually contains a
string, our function will wrap the string with the `Just` constructor.
Otherwise, it will return `Nothing`.

<div>

[SimpleJSON.hs]{.label}

``` {.haskell}
getString :: JValue -> Maybe String
getString (JString s) = Just s
getString _           = Nothing
```

</div>

When we save the modified source file, we can reload it in `ghci` and
try the new definition. (The `:reload` command remembers the last source
file we loaded, so we do not need to name it explicitly.)

``` {.screen}
ghci> :reload
Ok, one module loaded.
ghci> getString (JString "hello")
Just "hello"
ghci> getString (JNumber 3)
Nothing
```

A few more accessor functions, and we\'ve got a small body of code to
work with.

<div>

[SimpleJSON.hs]{.label}

``` {.haskell}
getInt :: JValue -> Maybe Int
getInt (JNumber n) = Just (truncate n)
getInt _           = Nothing

getDouble :: JValue -> Maybe Double
getDouble (JNumber n) = Just n
getDouble _           = Nothing

getBool :: JValue -> Maybe Bool
getBool (JBool b) = Just b
getBool _         = Nothing

getObject :: JValue -> Maybe [(String, JValue)]
getObject (JObject o) = Just o
getObject _           = Nothing

getArray :: JValue -> Maybe [JValue]
getArray (JArray a) = Just a
getArray _          = Nothing

isNull :: JValue -> Bool
isNull v = v == JNull
```

</div>

The `truncate` function turns a floating point or rational number into
an integer by dropping the digits after the decimal point.

``` {.screen}
ghci> truncate 5.8
5
ghci> :module +Data.Ratio
ghci> truncate (22 % 7)
3
```

The anatomy of a Haskell module
-------------------------------

A Haskell source file contains a definition of a single *module*. A
module lets us determine which names inside the module are accessible
from other modules.

A source file begins with a *module declaration*. This must precede all
other definitions in the source file.

<div>

[SimpleJSON.hs]{.label}

``` {.haskell}
module SimpleJSON
    ( JValue(..)
    , getString
    , getInt
    , getDouble
    , getBool
    , getObject
    , getArray
    , isNull
    ) where
```

</div>

The word `module` is reserved. It is followed by the name of the module,
which must begin with a capital letter. A source file must have the same
*base name* (the component before the suffix) as the name of the module
it contains. This is why our file `SimpleJSON.hs` contains a module
named `SimpleJSON`.

Following the module name is a list of *exports*, enclosed in
parentheses. The `where` keyword indicates that the body of the module
follows.

The list of exports indicates which names in this module are visible to
other modules. This lets us keep private code hidden from the outside
world. The special notation `(..)` that follows the name `JValue`
indicates that we are exporting both the type and all of its
constructors.

It might seem strange that we can export a type\'s name (i.e. its type
constructor), but not its value constructors. The ability to do this is
important: it lets us hide the details of a type from its users, making
the type *abstract*. If we cannot see a type\'s value constructors, we
cannot pattern match against a value of that type, nor can we construct
a new value of that type. Later in this chapter, we\'ll discuss some
situations in which we might want to make a type abstract.

If we omit the exports (and the parentheses that enclose them) from a
module declaration, every name in the module will be exported.

<div>

[Exporting.hs]{.label}

``` {.haskell}
module ExportEverything where
```

</div>

To export no names at all (which is rarely useful), we write an empty
export list using a pair of parentheses.

<div>

[Exporting.hs]{.label}

``` {.haskell}
module ExportNothing () where
```

</div>

Compiling Haskell source
------------------------

In addition to the `ghci` interpreter, the GHC distribution includes a
compiler, `ghc`, that generates native code. If you are already familiar
with a command line compiler such as `gcc` or `cl` (the C++ compiler
component of Microsoft\'s Visual Studio), you\'ll immediately be at home
with `ghc`.

To compile a source file, we first open a terminal or command prompt
window, then invoke `ghc` with the name of the source file to compile.

``` {.screen}
ghc -c SimpleJSON.hs
```

The `-c` option tells `ghc` to only generate object code. If we were to
omit the `-c` option, the compiler would attempt to generate a complete
executable. That would fail, because we haven\'t written a `main`
function, which GHC calls to start the execution of a standalone
program.

After `ghc` completes, if we list the contents of the directory, it
should contain two new files: `SimpleJSON.hi` and `SimpleJSON.o`. The
former is an *interface file*, in which `ghc` stores information about
the names exported from our module in machine-readable form. The latter
is an *object file*, which contains the generated machine code.

Generating a Haskell program, and importing modules
---------------------------------------------------

Now that we\'ve successfully compiled our minimal library, we\'ll write
a tiny program to exercise it. Create the following file in your text
editor, and save it as `Main.hs`.

<div>

[Main.hs]{.label}

``` {.haskell}
module Main where

import SimpleJSON

main = print (JObject [("foo", JNumber 1), ("bar", JBool False)])
```

</div>

Notice the `import` directive that follows the module declaration. This
indicates that we want to take all of the names that are exported from
the `SimpleJSON` module, and make them available in our module. Any
`import` directives must appear in a group at the beginning of a module.
They must appear after the module declaration, but before all other
code. We cannot, for example, scatter them throughout a source file.

Our choice of naming for the source file and function is deliberate. To
create an executable, `ghc` expects a module named `Main` that contains
a function named `main`. The `main` function is the one that will be
called when we run the program once we\'ve built it.

``` {.screen}
ghc -o simple Main.hs
```

This time around, we\'re omitting the `-c` option when we invoke `ghc`,
so it will attempt to generate an executable. The process of generating
an executable is called *linking*. As our command line suggests, `ghc`
is perfectly able to both compile source files and link an executable in
a single invocation.

We pass `ghc` a new option, `-o`, which takes one argument: this is the
name of the executable that `ghc` should create[^1]. Here, we\'ve
decided to name the program `simple`. On Windows, the program will have
the suffix `.exe`, but on Unix variants there will not be a suffix.

Finally, we supply the name of our new source file, `Main.hs`. If `ghc`
notices that it has already compiled a source file into an object file,
it will only recompile the source file if we\'ve modified it.

Once `ghc` has finished compiling and linking our `simple` program, we
can run it from the command line.

Printing JSON data
------------------

Now that we have a Haskell representation for JSON\'s types, we\'d like
to be able to take Haskell values and render them as JSON data.

There are a few ways we could go about this. Perhaps the most direct
would be to write a rendering function that prints a value in JSON form.
Once we\'re done, we\'ll explore some more interesting approaches.

<div>

[PutJSON.hs]{.label}

``` {.haskell}
module PutJSON where

import Data.List (intercalate)
import SimpleJSON

renderJValue :: JValue -> String

renderJValue (JString s)   = show s
renderJValue (JNumber n)   = show n
renderJValue (JBool True)  = "true"
renderJValue (JBool False) = "false"
renderJValue JNull         = "null"

renderJValue (JObject o) = "{" ++ pairs o ++ "}"
  where pairs []         = ""
        pairs ps         = intercalate ", " (map renderPair ps)
        renderPair (k,v) = show k ++ ": " ++ renderJValue v

renderJValue (JArray a) = "[" ++ values a ++ "]"
  where values [] = ""
        values vs = intercalate ", " (map renderJValue vs)
```

</div>

Good Haskell style involves separating pure code from code that performs
I/O. Our `renderJValue` function has no interaction with the outside
world, but we still need to be able to print a `JValue`.

<div>

[PutJSON.hs]{.label}

``` {.haskell}
putJValue :: JValue -> IO ()
putJValue v = putStrLn (renderJValue v)
```

</div>

Printing a JSON value is now easy.

Why should we separate the rendering code from the code that actually
prints a value? This gives us flexibility. For instance, if we wanted to
compress the data before writing it out, and we intermixed rendering
with printing, it would be much more difficult to adapt our code to that
change in circumstances.

This idea of separating pure from impure code is powerful, and pervasive
in Haskell code. Several Haskell compression libraries exist, all of
which have simple interfaces: a compression function accepts an
uncompressed string and returns a compressed string. We can use function
composition to render JSON data to a string, then compress to another
string, postponing any decision on how to actually display or transmit
the data.

Type inference is a double-edged sword
--------------------------------------

A Haskell compiler\'s ability to infer types is powerful and valuable.
Early on, you\'ll probably be faced by a strong temptation to take
advantage of type inference by omitting as many type declarations as
possible: let\'s simply make the compiler figure the whole lot out!

Skimping on explicit type information has a downside, one that
disproportionately affects new Haskell programmer. As a new Haskell
programmer, we\'re extremely likely to write code that will fail to
compile due to straightforward type errors.

When we omit explicit type information, we force the compiler to figure
out our intentions. It will infer types that are logical and consistent,
but perhaps not at all what we meant. If we and the compiler unknowingly
disagree about what is going on, it will naturally take us longer to
find the source of our problem.

Suppose, for instance, that we write a function that we believe returns
a `String`, but we don\'t write a type signature for it.

<div>

[Trouble.hs]{.label}

``` {.haskell}
import Data.Char

upcaseFirst (c:cs) = toUpper c -- forgot ":cs" here
```

</div>

Here, we want to upper-case the first character of a word, but we\'ve
forgotten to append the rest of the word onto the result. We think our
function\'s type is `String -> String`, but the compiler will correctly
infer its type as `String -> Char`. Let\'s say we then try to use this
function somewhere else.

<div>

[Trouble.hs]{.label}

``` {.haskell}
camelCase :: String -> String
camelCase xs = concat (map upcaseFirst (words xs))
```

</div>

When we try to compile this code or load it into `ghci`, we won\'t
necessarily get an obvious error message.

``` {.screen}
ghci> :load Trouble
[1 of 1] Compiling Main             ( Trouble.hs, interpreted )

Trouble.hs:6:24: error:
    • Couldn't match type ‘Char’ with ‘[Char]’
      Expected type: [[Char]]
        Actual type: [Char]
    • In the first argument of ‘concat’, namely
        ‘(map upcaseFirst (words xs))’
      In the expression: concat (map upcaseFirst (words xs))
      In an equation for ‘camelCase’:
          camelCase xs = concat (map upcaseFirst (words xs))
  |
6 | camelCase xs = concat (map upcaseFirst (words xs))
  |                        ^^^^^^^^^^^^^^^^^^^^^^^^^^
Failed, no modules loaded.
```

Notice that the error is reported where we *use* the `upcaseFirst`
function. If we\'re erroneously convinced that our definition and type
for `upcaseFirst` are correct, we may end up staring at the wrong piece
of code for quite a while, until enlightenment strikes.

Every time we write a type signature, we remove a degree of freedom from
the type inference engine. This reduces the likelihood of divergence
between our understanding of our code and the compiler\'s. Type
declarations also act as shorthand for ourselves as readers of our own
code, making it easier for us to develop a sense of what must be going
on.

This is not to say that we need to pepper every tiny fragment of code
with a type declaration. It is, however, usually good form to add a
signature to every top-level definition in our code. It\'s best to start
out fairly aggressive with explicit type signatures, and slowly ease
back as your mental model of how type checking works becomes more
accurate.

::: {.TIP}
Explicit types, undefined values, and error

The special value `undefined` will happily type-check no matter where we
use it, as will an expression like `error "argh!"`. It is especially
important that we write type signatures when we use these. Suppose we
use `undefined` or `error "write me"` to act as a placeholder in the
body of a top-level definition. If we omit a type signature, we may be
able to use the value we have defined in places where a correctly typed
version would be rejected by the compiler. This can easily lead us
astray.
:::

A more general look at rendering
--------------------------------

Our JSON rendering code is narrowly tailored to the exact needs of our
data types and the JSON formatting conventions. The output it produces
can be unfriendly to human eyes. We will now look at rendering as a more
generic task: how can we build a library that is useful for rendering
data in a variety of situations?

We would like to produce output that is suitable either for human
consumption (e.g. for debugging) or for machine processing. Libraries
that perform this job are referred to as *pretty printers*. There
already exist several Haskell pretty printing libraries. We are creating
one of our own not to replace them, but for the many useful insights we
will gain into both library design and functional programming
techniques.

We will call our generic pretty printing module `Prettify`, so our code
will go into a source file named `Prettify.hs`.

::: {.NOTE}
Naming

In our `Prettify` module, we will base our names on those used by
several established Haskell pretty printing libraries. This will give us
a degree of compatibility with existing mature libraries.
:::

To make sure that `Prettify` meets practical needs, we write a new JSON
renderer that uses the `Prettify` API. After we\'re done, we\'ll go back
and fill in the details of the `Prettify` module.

Instead of rendering straight to a string, our `Prettify` module will
use an abstract type that we\'ll call `Doc`. By basing our generic
rendering library on an abstract type, we can choose an implementation
that is flexible and efficient. If we decide to change the underlying
code, our users will not be able to tell.

We will name our new JSON rendering module `PrettyJSON.hs`, and retain
the name `renderJValue` for the rendering function. Rendering one of the
basic JSON values is straightforward.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
renderJValue :: JValue -> Doc
renderJValue (JBool True)  = text "true"
renderJValue (JBool False) = text "false"
renderJValue JNull         = text "null"
renderJValue (JNumber num) = double num
renderJValue (JString str) = string str
```

</div>

The `text`, `double`, and `string` functions will be provided by our
`Prettify` module.

Developing Haskell code without going nuts
------------------------------------------

Early on, as we come to grips with Haskell development, we have so many
new, unfamiliar concepts to keep track of at one time that it can be a
challenge to write code that compiles at all.

As we write our first substantial body of code, it\'s a *huge* help to
pause every few minutes and try to compile what we\'ve produced so far.
Because Haskell is so strongly typed, if our code compiles cleanly,
we\'re assuring ourselves that we\'re not wandering too far off into the
programming weeds.

One useful technique for quickly developing the skeleton of a program is
to write placeholder, or *stub* versions of types and functions. For
instance, we mentioned above that our `string`, `text` and `double`
functions would be provided by our `Prettify` module. If we don\'t
provide definitions for those functions or the `Doc` type, our attempts
to \"compile early, compile often\" with our JSON renderer will fail, as
the compiler won\'t know anything about those functions. To avoid this
problem, we write stub code that doesn\'t do anything.

<div>

[Prettify.hs]{.label}

``` {.haskell}
import SimpleJSON

data Doc = ToBeDefined deriving (Show)

string :: String -> Doc
string str = undefined

text :: String -> Doc
text str = undefined

double :: Double -> Doc
double num = undefined
```

</div>

The special value `undefined` has the type `a`, so it always
type-checks, no matter where we use it. If we attempt to evaluate it, it
will cause our program to crash.

``` {.screen}
ghci> :type undefined
undefined :: a
ghci> undefined
CallStack (from HasCallStack):
  error, called at libraries/base/GHC/Err.hs:79:14 in base:GHC.Err
  undefined, called at <interactive>:2:1 in interactive:Ghci1
ghci> :type double
double :: Double -> Doc
ghci> double 3.14
*** Exception: Prelude.undefined
CallStack (from HasCallStack):
  error, called at libraries/base/GHC/Err.hs:79:14 in base:GHC.Err
  undefined, called at PrettyStub.hs:11:14 in main:Main
```

Even though we can\'t yet run our stubbed code, the compiler\'s type
checker will ensure that our program is sensibly typed.

Pretty printing a string
------------------------

When we must pretty print a string value, JSON has moderately involved
escaping rules that we must follow. At the highest level, a string is
just a series of characters wrapped in quotes.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
string :: String -> Doc
string = enclose '"' '"' . hcat . map oneChar
```

</div>

::: {.NOTE}
Point-free style

This style of writing a definition exclusively as a composition of other
functions is called *point-free style*. The use of the word \"point\" is
not related to the \"`.`\" character used for function composition. The
term *point* is roughly synonymous (in Haskell) with *value*, so a
*point-free* expression makes no mention of the values that it operates
on.

Contrast the point-free definition of `string` above with this
\"pointy\" version, which uses a variable `s` to refer to the value on
which it operates.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
pointyString :: String -> Doc
pointyString s = enclose '"' '"' (hcat (map oneChar s))
```

</div>
:::

The `enclose` function simply wraps a `Doc` value with an opening and
closing character.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
enclose :: Char -> Char -> Doc -> Doc
enclose left right x = char left <> x <> char right
```

</div>

We provide a `(<>)` function in our pretty printing library. It appends
two `Doc` values, so it\'s the `Doc` equivalent of `(++)`.

<div>

[Prettify.hs]{.label}

``` {.haskell}
(<>) :: Doc -> Doc -> Doc
a <> b = undefined

char :: Char -> Doc
char c = undefined
```

</div>

Our pretty printing library also provides `hcat`, which concatenates
multiple `Doc` values into one: it\'s the analogue of `concat` for
lists.

<div>

[Prettify.hs]{.label}

``` {.haskell}
hcat :: [Doc] -> Doc
hcat xs = undefined
```

</div>

Our `string` function applies the `oneChar` function to every character
in a string, concatenates the lot, and encloses the result in quotes.
The `oneChar` function escapes or renders an individual character.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
oneChar :: Char -> Doc
oneChar c = case lookup c simpleEscapes of
              Just r -> text r
              Nothing | mustEscape c -> hexEscape c
                      | otherwise    -> char c
    where mustEscape c = c < ' ' || c == '\x7f' || c > '\xff'

simpleEscapes :: [(Char, String)]
simpleEscapes = zipWith ch "\b\n\f\r\t\\\"/" "bnfrt\\\"/"
    where ch a b = (a, ['\\',b])
```

</div>

The `simpleEscapes` value is a list of pairs. We call a list of pairs an
*association list*, or *alist* for short. Each element of our alist
associates a character with its escaped representation.

``` {.screen}
ghci> take 4 simpleEscapes
[('\b',"\\b"),('\n',"\\n"),('\f',"\\f"),('\r',"\\r")]
```

Our `case` expression attempts to see if our character has a match in
this alist. If we find the match, we emit it, otherwise we might need to
escape the character in a more complicated way. If so, we perform this
escaping. Only if neither kind of escaping is required do we emit the
plain character. To be conservative, the only unescaped characters we
emit are printable ASCII characters.

The more complicated escaping involves turning a character into the
string \"`\u`\" followed by a four-character sequence of hexadecimal
digits representing the numeric value of the Unicode character.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
smallHex :: Int -> Doc
smallHex x  = text "\\u"
           <> text (replicate (4 - length h) '0')
           <> text h
    where h = showHex x ""
```

</div>

The `showHex` function comes from the `Numeric` library (you will need
to import this at the beginning of `Prettify.hs`), and returns a
hexadecimal representation of a number.

``` {.screen}
ghci> showHex 114111 ""
"1bdbf"
```

The `replicate` function is provided by the `Prelude`, and builds a
fixed-length repeating list of its argument.

``` {.screen}
ghci> replicate 5 "foo"
["foo","foo","foo","foo","foo"]
```

There\'s a wrinkle: the four-digit encoding that `smallHex` provides can
only represent Unicode characters up to `0xffff`. Valid Unicode
characters can range up to `0x10ffff`. To properly represent a character
above `0xffff` in a JSON string, we follow some complicated rules to
split it into two. This gives us an opportunity to perform some
bit-level manipulation of Haskell numbers.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
astral :: Int -> Doc
astral n = smallHex (a + 0xd800) <> smallHex (b + 0xdc00)
    where a = (n `shiftR` 10) .&. 0x3ff
          b = n .&. 0x3ff
```

</div>

The `shiftR` function comes from the `Data.Bits` module, and shifts a
number to the right. The `(.&.)` function, also from `Data.Bits`,
performs a bit-level *and* of two values.

``` {.screen}
ghci> 0x10000 `shiftR` 4   :: Int
4096
ghci> 7 .&. 2   :: Int
2
```

Now that we\'ve written `smallHex` and `astral`, we can provide a
definition for `hexEscape`.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
hexEscape :: Char -> Doc
hexEscape c | d < 0x10000 = smallHex d
            | otherwise   = astral (d - 0x10000)
    where d = ord c
```

</div>

Arrays and objects, and the module header
-----------------------------------------

Compared to strings, pretty printing arrays and objects is a snap. We
already know that the two are visually similar: each starts with an
opening character, followed by a series of values separated with commas,
followed by a closing character. Let\'s write a function that captures
the common structure of arrays and objects.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
series :: Char -> Char -> (a -> Doc) -> [a] -> Doc
series open close item = enclose open close
                       . fsep . punctuate (char ',') . map item
```

</div>

We\'ll start by interpreting this function\'s type. It takes an opening
and closing character, then a function that knows how to pretty print a
value of some unknown type `a`, followed by a list of values of type
`a`, and it returns a value of type `Doc`.

Notice that although our type signature mentions four parameters, we
have only listed three in the definition of the function. We are simply
following the same rule that lets us simplify a definition like
`myLength xs = length xs` to `myLength = length`.

We have already written `enclose`, which wraps a `Doc` value in opening
and closing characters. The `fsep` function will live in our `Prettify`
module. It combines a list of `Doc` values into one, possibly wrapping
lines if the output will not fit on a single line.

<div>

[Prettify.hs]{.label}

``` {.haskell}
fsep :: [Doc] -> Doc
fsep xs = undefined
```

</div>

By now, you should be able to define your own stubs in `Prettify.hs`, by
following the examples we have supplied. We will not explicitly define
any more stubs.

The `punctuate` function will also live in our `Prettify` module, and we
can define it in terms of functions for which we\'ve already written
stubs.

<div>

[Prettify.hs]{.label}

``` {.haskell}
punctuate :: Doc -> [Doc] -> [Doc]
punctuate _ []       = []
punctuate _ [d]      = [d]
punctuate p (d : ds) = (d <> p) : punctuate p ds
```

</div>

With this definition of `series`, pretty printing an array is entirely
straightforward. We add this equation to the end of the block we\'ve
already written for our `renderJValue` function.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
renderJValue (JArray ary) = series '[' ']' renderJValue ary
```

</div>

To pretty print an object, we need to do only a little more work: for
each element, we have both a name and a value to deal with.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
renderJValue (JObject obj) = series '{' '}' field obj
    where field (name,val) = string name
                          <> text ": "
                          <> renderJValue val
```

</div>

Writing a module header
-----------------------

Now that we have written the bulk of our `PrettyJSON.hs` file, we must
go back to the top and add a module declaration.

<div>

[PrettyJSON.hs]{.label}

``` {.haskell}
module PrettyJSON
    ( renderJValue
    ) where

import Numeric (showHex)
import Data.Char (ord)
import Data.Bits (shiftR, (.&.))

import SimpleJSON (JValue(..))
import Prettify
    (Doc
    , (<>)
    , char
    , double
    , fsep
    , hcat
    , punctuate
    , text
    , compact)
```

</div>

We export just one name from this module: `renderJValue`, our JSON
rendering function. The other definitions in the module exist purely to
support `renderJValue`, so there\'s no reason to make them visible to
other modules.

Regarding imports, the `Numeric` and `Data.Bits` modules are distributed
with GHC. We\'ve already written the `SimpleJSON` module, and filled our
`Prettify` module with skeletal definitions. Notice that there\'s no
difference in the way we import standard modules from those we\'ve
written ourselves.

With each `import` directive, we explicitly list each of the names we
want to bring into our module\'s namespace. This is not required: if we
omit the list of names, all of the names exported from a module will be
available to us. However, it\'s generally a good idea to write an
explicit import list.

-   An explicit list makes it clear which names we\'re importing from
    where. This will make it easier for a reader to look up
    documentation if they encounter an unfamiliar function.
-   Occasionally, a library maintainer will remove or rename a function.
    If a function disappears from a third party module that we use, any
    resulting compilation error is likely to happen long after we\'ve
    written the module. The explicit list of imported names can act as a
    reminder to ourselves of where we had been importing the missing
    name from, which will help us to pinpoint the problem more quickly.
-   It can also occur that someone will add a name to a module that is
    identical to a name already in our own code. If we don\'t use an
    explicit import list, we\'ll end up with the same name in our module
    twice. If we use that name, GHC will report an error due to the
    ambiguity. An explicit list lets us avoid the possibility of
    accidentally importing an unexpected new name.

This idea of using explicit imports is a guideline that usually makes
sense, not a hard-and-fast rule. Occasionally, we\'ll need so many names
from a module that listing each one becomes messy. In other cases, a
module might be so widely used that a moderately experienced Haskell
programmer will probably know which names come from that module.

Fleshing out the pretty printing library
----------------------------------------

In our `Prettify` module, we represent our `Doc` type as an algebraic
data type.

<div>

[Prettify.hs]{.label}

``` {.haskell}
data Doc = Empty
         | Char Char
         | Text String
         | Line
         | Concat Doc Doc
         | Union Doc Doc
           deriving (Show, Eq)
```

</div>

Observe that the `Doc` type is actually a tree. The `Concat` and `Union`
constructors create an internal node from two other `Doc` values, while
the `Empty` and other simple constructors build leaves.

In the header of our module, we will export the name of the type, but
not any of its constructors: this will prevent modules that use the
`Doc` type from creating and pattern matching against `Doc` values.

Instead, to create a `Doc`, a user of the `Prettify` module will call a
function that we provide. Here are the simple construction functions. As
we add real definitions, we must replace any stubbed versions already in
the `Prettify.hs` source file.

<div>

[Prettify.hs]{.label}

``` {.haskell}
empty :: Doc
empty = Empty

char :: Char -> Doc
char c = Char c

text :: String -> Doc
text "" = Empty
text s  = Text s

double :: Double -> Doc
double d = text (show d)
```

</div>

The `Line` constructor represents a line break. The `line` function
creates *hard* line breaks, which always appear in the pretty printer\'s
output. Sometimes we\'ll want a *soft* line break, which is only used if
a line is too wide to fit in a window or page. We\'ll introduce a
`softline` function shortly.

<div>

[Prettify.hs]{.label}

``` {.haskell}
line :: Doc
line = Line
```

</div>

Almost as simple as the basic constructors is the `(<>)` function, which
concatenates two `Doc` values.

<div>

[Prettify.hs]{.label}

``` {.haskell}
(<>) :: Doc -> Doc -> Doc
Empty <> y = y
x <> Empty = x
x <> y = x `Concat` y
```

</div>

We pattern match against `Empty` so that concatenating a `Doc` value
with `Empty` on the left or right will have no effect. This keeps us
from bloating the tree with useless values.

``` {.screen}
ghci> text "foo" <> text "bar"
Concat (Text "foo") (Text "bar")
ghci> text "foo" <> empty
Text "foo"
ghci> empty <> text "bar"
Text "bar"
```

::: {.TIP}
A mathematical moment

If we briefly put on our mathematical hats, we can say that `Empty` is
the identity under concatenation, since nothing happens if we
concatenate a `Doc` value with `Empty`. In a similar vein, 0 is the
identity for adding numbers, and 1 is the identity for multiplying them.
Taking the mathematical perspective has useful practical consequences,
as we will see in a number of places throughout this book.
:::

Our `hcat` and `fsep` functions concatenate a list of `Doc` values into
one. In [the section called
\"Exercises\"](4-functional-programming.org::*Exercises) could define
concatenation for lists using `foldr`.

<div>

[Concat.hs]{.label}

``` {.haskell}
concat :: [[a]] -> [a]
concat = foldr (++) []
```

</div>

Since `(<>)` is analogous to `(++)`, and `empty` to `[]`, we can see how
we might write `hcat` and `fsep` as folds, too.

<div>

[Prettify.hs]{.label}

``` {.haskell}
hcat :: [Doc] -> Doc
hcat = fold (<>)

fold :: (Doc -> Doc -> Doc) -> [Doc] -> Doc
fold f = foldr f empty
```

</div>

The definition of `fsep` depends on several other functions.

<div>

[Prettify.hs]{.label}

``` {.haskell}
fsep :: [Doc] -> Doc
fsep = fold (</>)

(</>) :: Doc -> Doc -> Doc
x </> y = x <> softline <> y

softline :: Doc
softline = group line
```

</div>

These take a little explaining. The `softline` function should insert a
newline if the current line has become too wide, or a space otherwise.
How can we do this if our `Doc` type doesn\'t contain any information
about rendering? Our answer is that every time we encounter a soft
newline, we maintain *two* alternative representations of the document,
using the `Union` constructor.

<div>

[Prettify.hs]{.label}

``` {.haskell}
group :: Doc -> Doc
group x = flatten x `Union` x
```

</div>

Our `flatten` function replaces a `Line` with a space, turning two lines
into one longer line.

<div>

[Prettify.hs]{.label}

``` {.haskell}
flatten :: Doc -> Doc
flatten (x `Concat` y) = flatten x `Concat` flatten y
flatten Line           = Char ' '
flatten (x `Union` _)  = flatten x
flatten other          = other
```

</div>

Notice that we always call `flatten` on the left element of a `Union`:
the left of each `Union` is always the same width (in characters) as, or
wider than, the right. We\'ll be making use of this property in our
rendering functions below.

### Compact rendering

We frequently need to use a representation for a piece of data that
contains as few characters as possible. For example, if we\'re sending
JSON data over a network connection, there\'s no sense in laying it out
nicely: the software on the far end won\'t care whether the data is
pretty or not, and the added white space needed to make the layout look
good would add a lot of overhead.

For these cases, and because it\'s a simple piece of code to start with,
we provide a bare-bones compact rendering function.

<div>

[Prettify.hs]{.label}

``` {.haskell}
compact :: Doc -> String
compact x = transform [x]
    where transform [] = ""
          transform (d:ds) =
              case d of
                Empty        -> transform ds
                Char c       -> c : transform ds
                Text s       -> s ++ transform ds
                Line         -> '\n' : transform ds
                a `Concat` b -> transform (a:b:ds)
                _ `Union` b  -> transform (b:ds)
```

</div>

The `compact` function wraps its argument in a list, and applies the
`transform` helper function to it. The `transform` function treats its
argument as a stack of items to process, where the first element of the
list is the top of the stack.

The `transform` function\'s `(d:ds)` pattern breaks the stack into its
head, `d`, and the remainder, `ds`. In our `case` expression, the first
several branches recurse on `ds`, consuming one item from the stack for
each recursive application. The last two branches add items in front of
`ds`: the `Concat` branch adds both elements to the stack, while the
`Union` branch ignores its left element, on which we called `flatten`,
and adds its right element to the stack.

We have now fleshed out enough of our original skeletal definitions that
we can try out our `compact` function in `ghci`.

``` {.screen}
ghci> let value = renderJValue (JObject [("f", JNumber 1), ("q", JBool True)])
ghci> :type value
value :: Doc
ghci> putStrLn (compact value)
{"f": 1.0,
"q": true
}
```

To better understand how the code works, let\'s look at a simpler
example in more detail.

``` {.screen}
ghci> char 'f' <> text "oo"
Concat (Char 'f') (Text "oo")
ghci> compact (char 'f' <> text "oo")
"foo"
```

When we apply `compact`, it turns its argument into a list and applies
`transform`.

-   The `transform` function receives a one-item list, which matches the
    `(d:ds)` pattern. Thus `d` is the value `Concat (Char 'f')
     (Text "oo")`, and `ds` is the empty list, `[]`.

    Since `d`\'s constructor is `Concat`, the `Concat` pattern matches
    in the `case` expression. On the right hand side, we add `Char 'f'`
    and `Text "oo"` to the stack, and apply `transform` recursively.

    -   The `transform` function receives a two-item list, again
        matching the `(d:ds)` pattern. The variable `d` is bound to
        `Char 'f'`, and `ds` to `[Text "oo"]`.

        The `case` expression matches in the `Char` branch. On the right
        hand side, we use `(:)` to construct a list whose head is `'f'`,
        and whose body is the result of a recursive application of
        `transform`.

        -   The recursive invocation receives a one-item list. The
            variable `d` is bound to `Text "oo"`, and `ds` to `[]`.

            The `case` expression matches in the `Text` branch. On the
            right hand side, we use `(++)` to concatenate `"oo"` with
            the result of a recursive application of `transform`.

            -   In the final invocation, `transform` is invoked with an
                empty list, and returns an empty string.

        -   The result is `"oo" ++ ""`.

    -   The result is `'f' : "oo" ++ ""`.

### True pretty printing

While our `compact` function is useful for machine-to-machine
communication, its result is not always easy for a human to follow:
there\'s very little information on each line. To generate more readable
output, we\'ll write another function, `pretty`. Compared to `compact`,
`pretty` takes one extra argument: the maximum width of a line, in
columns. (We\'re assuming that our typeface is of fixed width.)

<div>

[Prettify.hs]{.label}

``` {.haskell}
pretty :: Int -> Doc -> String
```

</div>

To be more precise, this `Int` parameter controls the behaviour of
`pretty` when it encounters a `softline`. Only at a `softline` does
`pretty` have the option of either continuing the current line or
beginning a new line. Elsewhere, we must strictly follow the directives
set out by the person using our pretty printing functions.

Here\'s the core of our implementation

<div>

[Prettify.hs]{.label}

``` {.haskell}
pretty width x = best 0 [x]
    where best col (d:ds) =
              case d of
                Empty        -> best col ds
                Char c       -> c :  best (col + 1) ds
                Text s       -> s ++ best (col + length s) ds
                Line         -> '\n' : best 0 ds
                a `Concat` b -> best col (a:b:ds)
                a `Union` b  -> nicest col (best col (a:ds))
                                           (best col (b:ds))
          best _ _ = ""

          nicest col a b | (width - least) `fits` a = a
                         | otherwise                = b
                         where least = min width col
```

</div>

Our `best` helper function takes two arguments: the number of columns
emitted so far on the current line, and the list of remaining `Doc`
values to process.

In the simple cases, `best` updates the `col` variable in
straightforward ways as it consumes the input. Even the `Concat` case is
obvious: we push the two concatenated components onto our stack/list,
and don\'t touch `col`.

The interesting case involves the `Union` constructor. Recall that we
applied `flatten` to the left element, and did nothing to the right.
Also, remember that `flatten` replaces newlines with spaces. Therefore,
our job is to see which (if either) of the two layouts, the
`flatten~ed one or the original, will fit into our
~width` restriction.

To do this, we write a small helper that determines whether a single
line of a rendered `Doc` value will fit into a given number of columns.

<div>

[Prettify.hs]{.label}

``` {.haskell}
fits :: Int -> String -> Bool
w `fits` _ | w < 0 = False
w `fits` ""        = True
w `fits` ('\n':_)  = True
w `fits` (c:cs)    = (w - 1) `fits` cs
```

</div>

### Following the pretty printer

In order to understand how this code works, let\'s first consider a
simple `Doc` value.

``` {.screen}
ghci> empty </> char 'a'
Concat (Union (Char ' ') Line) (Char 'a')
```

We\'ll apply `pretty 2` on this value. When we first apply `best`, the
value of `col` is zero. It matches the `Concat` case, pushes the values
`Union (Char ' ') Line` and `Char 'a'` onto the stack, and applies
itself recursively. In the recursive application, it matches on
`Union (Char ' ') Line`.

At this point, we\'re going to ignore Haskell\'s usual order of
evaluation. This keeps our explanation of what\'s going on simple,
without changing the end result. We now have two subexpressions,
`best 0 [Char ' ', Char 'a']` and `best 0 [Line, Char 'a']`. The first
evaluates to `" a"`, and the second to `"\na"`. We then substitute these
into the outer expression to give `nicest 0 " a"
"\na"`.

To figure out what the result of `nicest` is here, we do a little
substitution. The values of `width` and `col` are 0 and 2, respectively,
so `least` is 0, and `width - least` is 2. We quickly evaluate
`` 2 `fits` " a" `` in `ghci`.

``` {.screen}
ghci> 2 `fits` " a"
True
```

Since this evaluates to `True`, the result of `nicest` here is `"
a"`.

If we apply our `pretty` function to the same JSON data as earlier, we
can see that it produces different output depending on the width that we
give it.

``` {.screen}
ghci> putStrLn (pretty 10 value)
{"f": 1.0,
"q": true
}
ghci> putStrLn (pretty 20 value)
{"f": 1.0, "q": true
}
ghci> putStrLn (pretty 30 value)
{"f": 1.0, "q": true }
```

### Exercises

Our current pretty printer is spartan, so that it will fit within our
space constraints, but there are a number of useful improvements we can
make.

1.  Write a function, `fill`, with the following type signature.

    <div>

    [Prettify.hs]{.label}
    ``` {.haskell}
    fill :: Int -> Doc -> Doc
    ```

    </div>

    It should add spaces to a document until it is the given number of
    columns wide. If it is already wider than this value, it should add
    no spaces.

2.  Our pretty printer does not take *nesting* into account. Whenever we
    open parentheses, braces, or brackets, any lines that follow should
    be indented so that they are aligned with the opening character
    until aa matching closing character is encountered.

    Add support for nesting, with a controllable amount of indentation.

    <div>

    [Prettify.hs]{.label}
    ``` {.haskell}
    nest :: Int -> Doc -> Doc
    ```

    </div>

Creating a package
------------------

The Haskell community has built a standard set of tools, named Cabal,
that help with building, installing, and distributing software. Cabal
organises software as a *package*. A package contains one library, and
possibly several executable programs.

### Writing a package description

To do anything with a package, Cabal needs a description of it. This is
contained in a text file whose name ends with the suffix `.cabal`. This
file belongs in the top-level directory of your project. It has a simple
format, which we\'ll describe below.

A Cabal package must have a name. Usually, the name of the package
matches the name of the `.cabal` file. We\'ll call our package
`mypretty`, so our file is `mypretty.cabal`. Often, the directory that
contains a `.cabal` file will have the same name as the package, e.g.
`mypretty`.

A package description begins with a series of global properties, which
apply to every library and executable in the package.

``` {.haskell}
name:    mypretty
version: 0.1

-- This is a comment. It stretches to the end of the line.
```

Package names must be unique. If you create and install a package that
has the same name as a package already present on your system, GHC will
become very confused.

The global properties include a substantial amount of information that
is intended for human readers, not Cabal itself.

``` {.haskell}
synopsis:    My pretty printing library, with JSON support
description: A simple pretty printing library that illustrates how to
             develop a Haskell library.
author:      Real World Haskell
maintainer:  nobody@realworldhaskell.org
```

As the `description` field indicates, a field can span multiple lines,
provided they\'re indented.

Also included in the global properties is license information. Most
Haskell packages are licensed under the BSD license, which Cabal calls
`BSD3`[^2]. (Obviously, you\'re free to choose whatever license you
think is appropriate.) The optional `license-file` field lets us specify
the name of a file that contains the exact text of our package\'s
licensing terms.

The features supported by successive versions of Cabal evolve over time,
so it\'s wise to indicate what versions of Cabal we expect to be
compatible with. The features we are describing are supported by
versions 1.2 and higher of Cabal.

``` {.haskell}
cabal-version: >= 1.2
```

To describe an individual library within a package, we write a *library*
section. The use of indentation here is significant: the contents of a
section must be indented.

``` {.haskell}
library
  exposed-modules: Prettify
                   PrettyJSON
                   SimpleJSON
  build-depends:   base >= 2.0
```

The `exposed-modules` field contains a list of modules that should be
available to users of this package. An optional field, `other-modules`,
contains a list of *internal* modules. These are required for this
library to function, but will not be visible to users.

The `build-depends` field contains a comma-separated list of packages
that our library requires to build. For each package, we can optionally
specify the range of versions with which this library is known to work.
The `base` package contains many of the core Haskell modules, such as
the Prelude, so it\'s effectively always required.

::: {.TIP}
Figuring out build dependencies

We don\'t have to guess or do any research to establish which packages
we depend on. If we try to build our package without a `build-depends`
field, compilation will fail with a useful error message. Here\'s an
example where we commented out the dependency on the `base` package.

``` {.screen}
$ runghc Setup build
Preprocessing library mypretty-0.1...
Building mypretty-0.1...

PrettyJSON.hs:8:7:
    Could not find module `Data.Bits':
      it is a member of package base, which is hidden
```

The error message makes it clear that we need to add the `base` package,
even though `base` is already installed. Forcing us to be explicit about
every package we need has a practical benefit: a command line tool named
`cabal-install` will automatically download, build, and install a
package and all of the packages it depends on.
:::

### GHC\'s package manager

GHC includes a simple package manager that tracks which packages are
installed, and what the versions of those packages are. A command line
tool named `ghc-pkg` lets us work with its package databases.

We say *databases* because GHC distinguishes between *system-wide*
packages, which are available to every user, and *per-user* packages,
which are only visible to the current user. The per-user database lets
us avoid the need for administrative privileges to install packages.

The `ghc-pkg` command provides subcommands to address different tasks.
Most of the time, we\'ll only need two of them. The `ghc-pkg
list` command lets us see what packages are installed. When we want to
uninstall a package, `ghc-pkg unregister` tells GHC that we won\'t be
using a particular package any longer. (We will have to manually delete
the installed files ourselves.)

### Setting up, building, and installing

In addition to a `.cabal` file, a package must contain a *setup* file.
This allows Cabal\'s build process to be heavily customised, if a
package needs it. The simplest setup file looks like this.

<div>

[Setup.hs]{.label}

``` {.haskell}
import Distribution.Simple
main = defaultMain
```

</div>

We save this file under the name `Setup.hs`.

Once we have the `.cabal` and `Setup.hs` files written, we have three
steps left.

To instruct Cabal how to build and where to install a package, we run a
simple command.

``` {.screen}
$ runghc Setup configure
```

This ensures that the packages we need are available, and stores
settings to be used later by other Cabal commands.

If we do not provide any arguments to `configure`, Cabal will install
our package in the system-wide package database. To install it into our
home directory and our personal package database, we must provide a
little more information.

``` {.screen}
$ runghc Setup configure --prefix=$HOME --user
```

Following the `configure` step, we build the package.

``` {.screen}
$ runghc Setup build
```

If this succeeds, we can install the package. We don\'t need to indicate
where to install to: Cabal will use the settings we provided in the
`configure` step. It will install to our own directory and update GHC\'s
per-user package database.

``` {.screen}
$ runghc Setup install
```

Practical pointers and further reading
--------------------------------------

GHC already bundles a pretty printing library,
`Text.PrettyPrint.HughesPJ`. It provides the same basic API as our
example, but a much richer and more useful set of pretty printing
functions. We recommend using it, rather than writing your own.

The design of the `HughesPJ` pretty printer was introduced by John
Hughes in \[[Hughes95](bibliography.org::Hughes95)\]. The library was
subsequently improved by Simon Peyton Jones, hence the name. Hughes\'s
paper is long, but well worth reading for his discussion of how to
design a library in Haskell.

In this chapter, our pretty printing library is based on a simpler
system described by Philip Wadler in
\[[Wadler98](bibliography.org::Wadler98)\]. His library was extended by
Daan Leijen; this version is available for download from Hackage as
`wl-pprint`. If you use the `cabal` command line tool, you can download,
build, and install it in one step with `cabal install wl-pprint`.

Footnotes
---------

[^1]: Memory aid: `-o` stands for \"output\" or \"object file\".

[^2]: The \"3\" in `BSD3` refers to the number of clauses in the
    license. An older version of the BSD license contained 4 clauses,
    but it is no longer used.
