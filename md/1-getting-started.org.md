Chapter 1: Getting Started
==========================

As you read the early chapters of this book, keep in mind that we will
sometimes introduce ideas in restricted, simplified form. Haskell is a
deep language, and presenting every aspect of a given subject all at
once is likely to prove overwhelming. As we build a solid foundation in
Haskell, we will expand upon these initial explanations.

Your Haskell environment
------------------------

The Glasgow Haskell Compiler (GHC) is the most widely used. It compiles
to native code, supports parallel execution, and provides useful
performance analysis and debugging tools.

GHC has three main components.

-   `ghc` is an optimizing compiler that generates fast native code.
-   `ghci` is an interactive interpreter and debugger.
-   `runghc` is a program for running Haskell programs as scripts,
    without needing to compile them first.

::: {.NOTE}
How we refer to the components of GHC

When we discuss the GHC system as a whole, we will refer to it as GHC.
If we are talking about a specific command, we will mention `ghc`,
`ghci`, or `runghc` by name.
:::

In this book, we assume that you\'re using at least version 8.2.2 of
GHC, which was released in 2017. To obtain a copy of GHC visit [the GHC
download page](http://www.haskell.org/downloads), and look for the list
of binary packages and installers.

Many Linux distributors, and providers of BSD and other Unix variants,
make custom binary packages of GHC available. Because these are built
specifically for each environment, they are much easier to install and
use than the generic binary packages that are available from the GHC
download page. You can find a list of distributions that custom-build
GHC at the GHC [distribution
packages](http://www.haskell.org/ghc/distribution_packages.html) page.

For more detailed information about how to install GHC on a variety of
popular platforms, we\'ve provided some instructions in [Appendix A,
*Installing GHC and Haskell
libraries*](installing-ghc-and-haskell-libraries.html).

Getting started with ghci, the interpreter
------------------------------------------

The interactive interpreter for GHC is a program named `ghci`. It lets
us enter and evaluate Haskell expressions, explore modules, and debug
our code. If you are familiar with Python or Ruby, `ghci` is somewhat
similar to `python` and `irb`, the interactive Python and Ruby
interpreters.

::: {.NOTE}
The `ghci` command has a narrow focus

We typically cannot copy some code out of a Haskell source file and
paste it into `ghci`. This does not have a significant effect on
debugging pieces of code, but it can initially be surprising if you are
used to, say, the interactive Python interpreter.
:::

On Unix-like systems, we run `ghci` as a command in a shell window. On
Windows, it\'s available via the Start Menu. For example, if you
installed using the GHC installer on Windows, you should go to
\"Programs\", then \"GHC\"; you will then see `ghci` in the list. (See
[the section called
\"Windows\"](installing-ghc-and-haskell-libraries.org::*Windows)
screenshot.)

When we run `ghci`, it displays a startup banner, followed by a
`Prelude>` prompt. Here, we\'re showing version 8.2.2.

``` {.screen}
$ ghci
GHCi, version 8.2.2: http://www.haskell.org/ghc/  :? for help
Prelude>
```

The word `Prelude` in the prompt indicates that `Prelude`, a standard
library of useful functions, is loaded and ready to use. When we load
other modules or source files, they will show up in the prompt, too.

::: {.TIP}
Getting help

If you enter `:?` at the `ghci` prompt, it will print a long help
message.
:::

The `Prelude` module is sometimes referred to as \"the standard
prelude\", because its contents are defined by the Haskell 2010
standard. Usually, it\'s simply shortened to \"the prelude\".

::: {.NOTE}
About the ghci prompt

The prompt displayed by `ghci` changes frequently depending on what
modules we have loaded. It can often grow long enough to leave little
visual room on a single line for our input.

For brevity and consistency, we have replaced `ghci`\'s default prompts
throughout this book with the prompt string `ghci>`.

If you want to do this yourself, use `ghci`\'s `:set prompt` directive,
as follows.

``` {.screen}
Prelude> :set prompt "ghci> "
ghci>
```
:::

The prelude is always implicitly available; we don\'t need to take any
actions to use the types, values, or functions it defines. To use
definitions from other modules, we must load them into `ghci`, using the
`:module` command.

``` {.screen}
ghci> :module + Data.Ratio
```

We can now use the functionality of the `Data.Ratio` module, which lets
us work with rational numbers (fractions).

Basic interaction: using ghci as a calculator
---------------------------------------------

In addition to providing a convenient interface for testing code
fragments, `ghci` can function as a readily accessible desktop
calculator. We can easily express any calculator operation in `ghci`
and, as an added bonus, we can add more complex operations as we become
more familiar with Haskell. Even using the interpreter in this simple
way can help us to become more comfortable with how Haskell works.

### Simple arithmetic

We can immediately start entering expressions, to see what `ghci` will
do with them. Basic arithmetic works similarly to languages like C and
Python: we write expressions in *infix* form, where an operator appears
between its operands.

``` {.screen}
ghci> 2 + 2
4
ghci> 31337 * 101
3165037
ghci> 7.0 / 2.0
3.5
```

The infix style of writing an expression is just a convenience: we can
also write an expression in *prefix* form, where the operator precedes
its arguments. To do this, we must enclose the operator in parentheses.

``` {.screen}
ghci> 2 + 2
4
ghci> (+) 2 2
4
```

As the expressions above imply, Haskell has a notion of integers and
floating point numbers. Integers can be arbitrarily large. Here, `(^)`
provides integer exponentiation.

``` {.screen}
ghci> 313 ^ 15
27112218957718876716220410905036741257
```

### An arithmetic quirk: writing negative numbers

Haskell presents us with one peculiarity in how we must write numbers:
it\'s often necessary to enclose a negative number in parentheses. This
affects us as soon as we move beyond the simplest expressions.

We\'ll start by writing a negative number.

``` {.screen}
ghci> -3
-3
```

The `-` above is a unary operator. In other words, we didn\'t write the
single number \"-3\"; we wrote the number \"3\", and applied the
operator `-` to it. The `-` operator is Haskell\'s only unary operator,
and we cannot mix it with infix operators.

``` {.screen}
ghci> 2 + -3

<interactive>:1:1: error:
    Precedence parsing error
        cannot mix ‘+’ [infixl 6] and prefix `-' [infixl 6] in the same infix expression
```

If we want to use the unary minus near an infix operator, we must wrap
the expression it applies to in parentheses.

``` {.screen}
ghci> 2 + (-3)
-1
ghci> 3 + (-(13 * 37))
-478
```

This avoids a parsing ambiguity. When we apply a function in Haskell, we
write the name of the function, followed by its argument, for example
`f 3`. If we did not need to wrap a negative number in parentheses, we
would have two profoundly different ways to read `f-3`: it could be
either \"apply the function `f` to the number `-3`\", or \"subtract the
number `3` from the variable `f`\".

*Most* of the time, we can omit white space (\"blank\" characters such
as space and tab) from expressions, and Haskell will parse them as we
intended. But not always. Here is an expression that works:

``` {.screen}
ghci> 2*3
6
```

And here is one that seems similar to the problematic negative number
example above, but results in a different error message.

``` {.screen}
ghci> 2*-3

<interactive>:1:1: error:
    • Variable not in scope: (*-) :: Integer -> Integer -> t
    • Perhaps you meant one of these:
        ‘*’ (imported from Prelude), ‘-’ (imported from Prelude),
        ‘*>’ (imported from Prelude)
```

Here, the Haskell implementation is reading `*-` as a single operator.
Haskell lets us define new operators (a subject that we will return to
later), but we haven\'t defined `*-`. Once again, a few parentheses get
us and `ghci` looking at the expression in the same way.

``` {.screen}
ghci> 2*(-3)
-6
```

Compared to other languages, this unusual treatment of negative numbers
might seem annoying, but it represents a reasoned trade-off. Haskell
lets us define new operators at any time. This is not some kind of
esoteric language feature; we will see quite a few user-defined
operators in the chapters ahead. The language designers chose to accept
a slightly cumbersome syntax for negative numbers in exchange for this
expressive power.

### Boolean logic, operators, and value comparisons

The values of Boolean logic in Haskell are `True` and `False`. The
capitalization of these names is important. The language uses
C-influenced operators for working with Boolean values: `(&&)` is
logical \"and\", and `(||)` is logical \"or\".

``` {.screen}
ghci> True && False
False
ghci> False || True
True
```

While some programming languages treat the number zero as synonymous
with `False`, Haskell does not, nor does it consider a non-zero value to
be `True`.

``` {.screen}
ghci> True && 1

<interactive>:1:9: error:
    • No instance for (Num Bool) arising from the literal ‘1’
    • In the second argument of ‘(&&)’, namely ‘1’
      In the expression: True && 1
      In an equation for ‘it’: it = True && 1
```

Once again, we are faced with a substantial-looking error message. In
brief, it tells us that the boolean type, `Bool`, is not a member of the
family of numeric types, `Num`. The error message is rather long because
`ghci` is pointing out the location of the problem, and hinting at a
possible change we could make that might fix the problem.

Here is a more detailed breakdown of the error message.

-   \"`No instance for (Num Bool)`\" tells us that `ghci` is trying to
    treat the numeric value 1 as having a Bool type, but it cannot.
-   \"`` arising from the literal `1' ``\" indicates that it was our use
    of the number `1` that caused the problem.
-   \"`` In the definition of `it' ``\" refers to a `ghci` short cut
    that we will revisit in a few pages.

::: {.TIP}
Remain fearless in the face of error messages

We have an important point to make here, which we will repeat throughout
the early sections of this book. If you run into problems or error
messages that you do not yet understand, *don\'t panic*. Early on, all
you have to do is figure out enough to make progress on a problem. As
you acquire experience, you will find it easier to understand parts of
error messages that initially seem obscure.

The numerous error messages have a purpose: they actually help us in
writing correct code, by making us perform some amount of debugging \"up
front\", before we ever run a program. If you are coming from a
background of working with more permissive languages, this way of
working may come as something of a shock. Bear with us.
:::

Most of Haskell\'s comparison operators are similar to those used in C
and the many languages it has influenced.

``` {.screen}
ghci> 1 == 1
True
ghci> 2 < 3
True
ghci> 4 >= 3.99
True
```

One operator that differs from its C counterpart is \"is not equal to\".
In C, this is written as `!=`. In Haskell, we write `(/=)`, which
resembles the ≠ notation used in mathematics.

``` {.screen}
ghci> 2 /= 3
True
```

Also, where C-like languages often use `!` for logical negation, Haskell
uses the `not` function.

``` {.screen}
ghci> not True
False
```

### Operator precedence and associativity

Like written algebra and other programming languages that use infix
operators, Haskell has a notion of operator precedence. We can use
parentheses to explicitly group parts of an expression, and precedence
allows us to omit a few parentheses. For example, the multiplication
operator has a higher precedence than the addition operator, so Haskell
treats the following two expressions as equivalent.

``` {.screen}
ghci> 1 + (4 * 4)
17
ghci> 1 + 4 * 4
17
```

Haskell assigns numeric precedence values to operators, with 1 being the
lowest precedence and 9 the highest. A higher-precedence operator is
applied before a lower-precedence operator. We can use `ghci` to inspect
the precedence levels of individual operators, using its `:info`
command.

``` {.screen}
ghci> :info (+)
class (Eq a, Show a) => Num a where
  (+) :: a -> a -> a
  ...
    -- Defined in GHC.Num
infixl 6 +
ghci> :info (*)
class (Eq a, Show a) => Num a where
  ...
  (*) :: a -> a -> a
  ...
    -- Defined in GHC.Num
infixl 7 *
```

The information we seek is in the line \"`infixl 6 +`\", which indicates
that the `(+)` operator has a precedence of 6. (We will explain the
other output in a later chapter.) The \"`infixl 7 *`\" tells us that the
`(*)` operator has a precedence of 7. Since `(*)` has a higher
precedence than `(+)`, we can now see why `1 + 4 * 4` is evaluated as
`1 + (4 * 4)`, and not `(1 + 4) * 4`.

Haskell also defines *associativity* of operators. This determines
whether an expression containing multiple uses of an operator is
evaluated from left to right, or right to left. The `(+)` and `(*)`
operators are left associative, which is represented as `infixl` in the
`ghci` output above. A right associative operator is displayed with
`infixr`.

``` {.screen}
ghci> :info (^)
(^) :: (Num a, Integral b) => a -> b -> a  -- Defined in GHC.Real
infixr 8 ^
```

The combination of precedence and associativity rules are usually
referred to as *fixity* rules.

### Undefined values, and introducing variables

Haskell\'s prelude, the standard library we mentioned earlier, defines
at least one well-known mathematical constant for us.

``` {.screen}
ghci> pi
3.141592653589793
```

But its coverage of mathematical constants is not comprehensive, as we
can quickly see. Let us look for Euler\'s number, `e`.

``` {.screen}
ghci> e

<interactive>:1:1: error: Variable not in scope: e
```

Oh well. We have to define it ourselves.

::: {.NOTE}
Don\'t worry about the error message

If the above \"not in scope\" error message seems a little daunting, do
not worry. All it means is that there is no variable defined with the
name `e`.
:::

Using `ghci`\'s `let` construct, we can make a temporary definition of
`e` ourselves.

``` {.screen}
ghci> e = exp 1
```

This is an application of the exponential function, `exp`, and our first
example of applying a function in Haskell. While languages like Python
require parentheses around the arguments to a function, Haskell does
not.

With `e` defined, we can now use it in arithmetic expressions. The `(^)`
exponentiation operator that we introduced earlier can only raise a
number to an integer power. To use a floating point number as the
exponent, we use the `(**)` exponentiation operator.

``` {.screen}
ghci> (e ** pi) - pi
19.99909997918947
```

::: {.WARNING}
This syntax is ghci-specific

The syntax for `let` that `ghci` accepts is not the same as we would use
at the \"top level\" of a normal Haskell program. We will see the normal
syntax in [the section called \"Introducing local
variables\"](3-defining-types-streamlining-functions.org::*Introducing local variables)
:::

### Dealing with precedence and associativity rules

It is sometimes better to leave at least some parentheses in place, even
when Haskell allows us to omit them. Their presence can help future
readers (including ourselves) to understand what we intended.

Even more importantly, complex expressions that rely completely on
operator precedence are notorious sources of bugs. A compiler and a
human can easily end up with different notions of what even a short,
parenthesis-free expression is supposed to do.

There is no need to remember all of the precedence and associativity
rules numbers: it is simpler to add parentheses if you are unsure.

Command line editing in ghci
----------------------------

On most systems, `ghci` has some amount of command line editing ability.
In case you are not familiar with command line editing, it\'s a huge
time saver. The basics are common to both Unix-like and Windows systems.
Pressing the ↑ key on your keyboard recalls the last line of input you
entered; pressing ↑ repeatedly cycles through earlier lines of input.
You can use the ← and → arrow keys to move around inside a line of
input. On Unix (but not Windows, unfortunately), the `tab` key completes
partially entered identifiers.

::: {.TIP}
Where to look for more information

We\'ve barely scratched the surface of command line editing here. Since
you can work more effectively if you\'re more familiar with the
capabilities of your command line editing system, you might find it
useful to do some further reading. `ghci` uses the Haskeline library
under the hood, which is
[powerful](https://github.com/judah/haskeline/wiki/KeyBindings) and
[customisable](https://github.com/judah/haskeline/wiki/UserPreferences).
:::

Lists
-----

A list is surrounded by square brackets; the elements are separated by
commas.

``` {.screen}
ghci> [1, 2, 3]
[1,2,3]
```

::: {.NOTE}
Commas are separators, not terminators

Some languages permit the last element in a list to be followed by an
optional trailing comma before a closing bracket, but Haskell doesn\'t
allow this. If you leave in a trailing comma (e.g. `[1,2,]`), you\'ll
get a parse error.
:::

A list can be of any length. The empty list is written `[]`.

``` {.screen}
ghci> []
[]
ghci> ["foo", "bar", "baz", "quux", "fnord", "xyzzy"]
["foo","bar","baz","quux","fnord","xyzzy"]
```

All elements of a list must be of the same type. Here, we violate this
rule: our list starts with two Bool values, but ends with a string.

``` {.screen}
ghci> [True, False, "testing"]

<interactive>:1:15: error:
    • Couldn't match expected type ‘Bool’ with actual type ‘[Char]’
    • In the expression: "testing"
      In the expression: [True, False, "testing"]
      In an equation for ‘it’: it = [True, False, "testing"]
```

Once again, `ghci`\'s error message is verbose, but it\'s simply telling
us that there is no way to turn the string into a Boolean value, so the
list expression isn\'t properly typed.

If we write a series of elements using *enumeration notation*, Haskell
will fill in the contents of the list for us.

``` {.screen}
ghci> [1..10]
[1,2,3,4,5,6,7,8,9,10]
```

Here, the `..` characters denote an *enumeration*. We can only use this
notation for types whose elements we can enumerate. It makes no sense
for text strings, for instance: there is not any sensible, general way
to enumerate `["foo".."quux"]`.

By the way, notice that the above use of range notation gives us a
*closed interval*; the list contains both endpoints.

When we write an enumeration, we can optionally specify the size of the
step to use by providing the first two elements, followed by the value
at which to stop generating the enumeration.

``` {.screen}
ghci> [1.0,1.25..2.0]
[1.0,1.25,1.5,1.75,2.0]
ghci> [1,4..15]
[1,4,7,10,13]
ghci> [10,9..1]
[10,9,8,7,6,5,4,3,2,1]
```

In the latter case above, the list is quite sensibly missing the end
point of the enumeration, because it isn\'t an element of the series we
defined.

We can omit the end point of an enumeration. If a type doesn\'t have a
natural \"upper bound\", this will produce values indefinitely. For
example, if you type `[1..]` at the `ghci` prompt, you\'ll have to
interrupt or kill `ghci` to stop it from printing an infinite succession
of ever-larger numbers. If you are tempted to do this, type `Ctrl-C` to
halt the enumeration. We will find later on that infinite lists are
often useful in Haskell.

::: {.WARNING}
Beware enumerating floating point numbers

Here\'s a non-intuitive bit of behaviour.

``` {.screen}
ghci> [1.0..1.8]
[1.0,2.0]
```

Behind the scenes, to avoid floating point roundoff problems, the
Haskell implementation enumerates from `1.0` to `1.8+0.5`.

Using enumeration notation over floating point numbers can pack more
than a few surprises, so if you use it at all, be careful. Floating
point behavior is quirky in all programming languages; there is nothing
unique to Haskell here.
:::

### Operators on lists

There are two ubiquitous operators for working with lists. We
concatenate two lists using the `(++)` operator.

``` {.screen}
ghci> [3,1,3] ++ [3,7]
[3,1,3,3,7]
ghci> [] ++ [False,True] ++ [True]
[False,True,True]
```

More basic is the `(:)` operator, which adds an element to the front of
a list. This is pronounced \"cons\" (short for \"construct\").

``` {.screen}
ghci> 1 : [2,3]
[1,2,3]
ghci> 1 : []
[1]
```

You might be tempted to try writing `[1,2] : 3` to add an element to the
end of a list, but `ghci` will reject this with an error message,
because the first argument of `(:)` must be an element, and the second
must be a list.

Strings and characters
----------------------

If you know a language like Perl or C, you\'ll find Haskell\'s notations
for strings familiar.

A text string is surrounded by double quotes.

``` {.screen}
ghci> "This is a string."
"This is a string."
```

As in many languages, we can represent hard-to-see characters by
\"escaping\" them. Haskell\'s escape characters and escaping rules
follow the widely used conventions established by the C language. For
example, `'\n'` denotes a newline character, and `'\t'` is a tab
character. For complete details, see [Appendix B, *Characters, strings,
and escaping
rules*](appendix-characters-strings-and-escaping-rules.org).

``` {.screen}
ghci> putStrLn "Here's a newline -->\n<-- See?"
Here's a newline -->
<-- See?
```

Haskell makes a distinction between single characters and text strings.
A single character is enclosed in single quotes.

``` {.screen}
ghci> 'a'
'a'
```

In fact, a text string is simply a list of individual characters.
Here\'s a painful way to write a short string, which `ghci` gives back
to us in a more familiar form.

``` {.screen}
ghci> a = ['l', 'o', 't', 's', ' ', 'o', 'f', ' ', 'w', 'o', 'r', 'k']
ghci> a
"lots of work"
ghci> a == "lots of work"
True
```

The empty string is written `""`, and is a synonym for `[]`.

``` {.screen}
ghci> "" == []
True
```

Since a string is a list of characters, we can use the regular list
operators to construct new strings.

``` {.screen}
ghci> 'a':"bc"
"abc"
ghci> "foo" ++ "bar"
"foobar"
```

First steps with types
----------------------

While we\'ve talked a little about types already, our interactions with
`ghci` have so far been free of much type-related thinking. We haven\'t
told `ghci` what types we\'ve been using, and it\'s mostly been willing
to accept our input.

Haskell requires type names to start with an uppercase letter, and
variable names must start with a lowercase letter. Bear this in mind as
you read on; it makes it much easier to follow the names.

The first thing we can do to start exploring the world of types is to
get `ghci` to tell us more about what it\'s doing. `ghci` has a command,
`:set`, that lets us change a few of its default behaviours. We can tell
it to print more type information as follows.

``` {.screen}
ghci> :set +t
ghci> 'c'
'c'
it :: Char
ghci> "foo"
"foo"
it :: [Char]
```

What the `+t` does is tell `ghci` to print the type of an expression
after the expression. That cryptic `it` in the output can be very
useful: it\'s actually the name of a special variable, in which `ghci`
stores the result of the last expression we evaluated. (This isn\'t a
Haskell language feature; it\'s specific to `ghci` alone.) Let\'s break
down the meaning of the last line of `ghci` output.

-   It\'s telling us about the special variable `it`.
-   We can read text of the form `x :: y` as meaning \"the expression
    `x` has the type `y`\".
-   Here, the expression \"it\" has the type `[Char]`. (The name
    `String` is often used instead of `[Char]`. It is simply a synonym
    for `[Char]`.)

::: {.TIP}
The joy of \"it\"

That `it` variable is a handy `ghci` shortcut. It lets us use the result
of the expression we just evaluated in a new expression.

``` {.screen}
ghci> "foo"
"foo"
it :: [Char]
ghci> it ++ "bar"
"foobar"
it :: [Char]
```

When evaluating an expression, `ghci` won\'t change the value of `it` if
the evaluation fails. This lets you write potentially bogus expressions
with something of a safety net.

``` {.screen}
ghci> it
"foobar"
it :: [Char]
ghci> it ++ 3

<interactive>:1:1: error
    • No instance for (Num [Char]) arising from the literal ‘3’
    • In the second argument of ‘(++)’, namely ‘3’
      In the expression: it ++ 3
      In an equation for ‘it’: it = it ++ 3
ghci> it
"foobar"
it :: [Char]
ghci> it ++ "baz"
"foobarbaz"
it :: [Char]
```

When we couple `it` with liberal use of the arrow keys to recall and
edit the last expression we typed, we gain a decent way to experiment
interactively: the cost of mistakes is very low. Take advantage of the
opportunity to make cheap, plentiful mistakes when you\'re exploring the
language!
:::

Here are a few more of Haskell\'s names for types, from expressions of
the sort we\'ve already seen.

``` {.screen}
ghci> 7 ^ 80
40536215597144386832065866109016673800875222251012083746192454448001
it :: Integer
```

Haskell\'s integer type is named `Integer`. The size of an `Integer`
value is bounded only by your system\'s memory capacity.

Rational numbers don\'t look quite the same as integers. To construct a
rational number, we use the `(%)` operator. The numerator is on the
left, the denominator on the right.

``` {.screen}
ghci> :m +Data.Ratio
ghci> 11 % 29
11%29
it :: Integral a => Ratio a
```

For convenience, `ghci` lets us abbreviate many commands, so we can
write `:m` instead of `:module` to load a module.

Notice *two* words on the right hand side of the `::` above. We can read
this as a \"ratio of integer\". We might guess that a `Ratio` must have
values of type Integer as both numerator and denominator. Sure enough,
if we try to construct a `Ratio` where the numerator and denominator are
of different types, or of the same non-integral type, `ghci` complains.

``` {.screen}
ghci> 3.14 % 8

<interactive>:1:1: error:
    • Ambiguous type variable ‘a0’ arising from a use of ‘print’
      prevents the constraint ‘(Show a0)’ from being solved.
      Probable fix: use a type annotation to specify what ‘a0’ should be.
      These potential instances exist:
        instance Show a => Show (Ratio a) -- Defined in ‘GHC.Real’
        instance Show Ordering -- Defined in ‘GHC.Show’
        instance Show Integer -- Defined in ‘GHC.Show’
        ...plus 23 others
        ...plus 11 instances involving out-of-scope types
        (use -fprint-potential-instances to see them all)
    • In a stmt of an interactive GHCi command: print it
ghci> 1.2 % 3.4

<interactive>:1:1: error:
    • Ambiguous type variable ‘a0’ arising from a use of ‘print’
      prevents the constraint ‘(Show a0)’ from being solved.
      Probable fix: use a type annotation to specify what ‘a0’ should be.
      These potential instances exist:
        instance Show a => Show (Ratio a) -- Defined in ‘GHC.Real’
        instance Show Ordering -- Defined in ‘GHC.Show’
        instance Show Integer -- Defined in ‘GHC.Show’
        ...plus 23 others
        ...plus 11 instances involving out-of-scope types
        (use -fprint-potential-instances to see them all)
    • In a stmt of an interactive GHCi command: print it
```

Although it is initially useful to have `:set +t` giving us type
information for every expression we enter, this is a facility we will
quickly outgrow. After a while, we will often know what type we expect
an expression to have. We can turn off the extra type information at any
time, using the `:unset` command.

``` {.screen}
ghci> :unset +t
ghci> 2
2
```

Even with this facility turned off, we can still get that type
information easily when we need it, using another `ghci` command.

``` {.screen}
ghci> :type 'a'
'a' :: Char
ghci> "foo"
"foo"
ghci> :type it
it :: [Char]
```

The `:type` command will print type information for any expression we
give it (including `it`, as we see above). It won\'t actually evaluate
the expression; it only checks its type and prints that.

Why are the types reported for these two expressions different?

``` {.screen}
ghci> 3 + 2
5
ghci> :type it
it :: Integer
ghci> :type 3 + 2
3 + 2 :: (Num t) => t
```

Haskell has several numeric types. For example, a literal number such as
`1` could, depending on the context in which it appears, be an integer
or a floating point value. When we force `ghci` to evaluate the
expression `3 + 2`, it has to choose a type so that it can print the
value, and it defaults to `Integer`. In the second case, we ask `ghci`
to print the type of the expression without actually evaluating it, so
it does not have to be so specific. It answers, in effect, \"its type is
numeric\". We will see more of this style of type annotation in
[Chapter 6, Using Type Classes](6-using-typeclasses.org).

A simple program
----------------

Let\'s take a small leap ahead, and write a small program that counts
the number of lines in its input. Don\'t expect to understand this yet;
it\'s just fun to get our hands dirty. In a text editor, enter the
following code into a file, and save it as `WC.hs`.

<div>

[wc.hs]{.label}

``` {.haskell}
-- lines beginning with "--" are comments.

main = interact wordCount
    where wordCount input = show (length (lines input)) ++ "\n"
```

</div>

Find or create a text file; let\'s call it `quux.txt`[^1].

``` {.screen}
$ cat quux.txt
Teignmouth, England
Paris, France
Ulm, Germany
Auxerre, France
Brunswick, Germany
Beaumont-en-Auge, France
Ryazan, Russia
```

From a shell or command prompt, run the following command.

``` {.screen}
$ runghc wc < quux.txt
7
```

We have successfully written a simple program that interacts with the
real world! In the chapters that follow, we will successively fill the
gaps in our understanding until we can write programs of our own.

Exercises
---------

1.  Enter the following expressions into `ghci`. What are their types?

    -   `5 + 8`
    -   `3 * 5 + 8`
    -   `2 + 4`
    -   `(+) 2 4`
    -   `sqrt 16`
    -   `succ 6`
    -   `succ 7`
    -   `pred 9`
    -   `pred 8`
    -   `sin (pi / 2)`
    -   `truncate pi`
    -   `round 3.5`
    -   `round 3.4`
    -   `floor 3.7`
    -   `ceiling 3.3`

2.  From `ghci`, type `:?` to print some help. Define a variable, such
    as `x = 1`, then type `:show bindings`. What do you see?

3.  The `words` function counts the number of words in a string. Modify
    the `wc.hs` example to count the number of words in a file.

4.  Modify the `wc.hs` example again, to print the number of characters
    in a file.

Footnotes
---------

[^1]: Incidentally, what do these cities have in common?
