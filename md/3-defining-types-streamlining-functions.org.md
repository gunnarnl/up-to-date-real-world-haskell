Chapter 3: Defining Types, Streamlining Functions
=================================================

Defining a new data type
------------------------

Although lists and tuples are useful, we\'ll often want to construct new
data types of our own. This allows us to add structure to the values in
our programs. Instead of using an anonymous tuple, we can give a
collection of related values a name and a distinct type. Defining our
own types also improves the type safety of our code: Haskell will not
allow us to accidentally mix values of two types that are structurally
similar but have different names.

For motivation, we\'ll consider a few kinds of data that a small online
bookstore might need to manage. We won\'t make any attempt at complete
or realistic data definitions, but at least we\'re tying them to the
real world.

We define a new data type using the `data` keyword.

<div>

[BookStore.hs]{.label}

``` {.haskell}
data BookInfo = Book Int String [String]
                deriving (Show)
```

</div>

The `BookInfo` after the `data` keyword is the name of our new type. We
call `BookInfo` a *type constructor*. Once we have defined a type, we
will use its type constructor to refer to it. As we\'ve already
mentioned, a type name, and hence a type constructor, must start with a
capital letter.

The `Book` that follows is the name of the *value constructor*
(sometimes called a data constructor). We use this to create a value of
the `BookInfo` type. A value constructor\'s name must also start with a
capital letter.

After `Book`, the `Int`, `String`, and `[String]` that follow are the
*components* of the type. A component serves the same purpose in Haskell
as a field in a structure or class would in another language: it\'s a
\"slot\" where we keep a value. (We\'ll often refer to components as
fields.)

In this example, the `Int` represents a book\'s identifier (e.g. in a
stock database), `String` its title, and `[String]` the names of its
authors.

To make the link to a concept we\'ve already seen, the `BookInfo` Type
contains the same components as a 3-tuple of type
`(Int, String, [String])`, but it has a distinct type. We can\'t
accidentally (or deliberately) use one in a context where the other is
expected. For instance, a bookstore is also likely to carry magazines.

<div>

[BookStore.hs]{.label}

``` {.haskell}
data MagazineInfo = Magazine Int String [String]
                    deriving (Show)
```

</div>

Even though this `MagazineInfo` type has the same structure as our
`BookInfo` type, Haskell treats the types as distinct because their type
and value constructors have different names.

::: {.NOTE}
Deriving what?

We\'ll explain the full meaning of `deriving (Show)` later, in [the
section called \"Show\"](6-using-typeclasses.org::*Show) need to tack
this onto a type declaration so that `ghci` will automatically know how
to print a value of this type.
:::

We can create a new value of type `BookInfo` by treating `Book` as a
function, and applying it with arguments of types `Int`, `String`, and
`[String]`.

<div>

[BookStore.hs]{.label}

``` {.haskell}
myInfo = Book 9780135072455 "Algebra of Programming"
         ["Richard Bird", "Oege de Moor"]
```

</div>

Once we have defined a type, we can experiment with it in `ghci`. We
begin by using the `:load` command to load our source file.

``` {.screen}
ghci> :load BookStore
[1 of 1] Compiling Main             ( BookStore.hs, interpreted )
Ok, one module loaded.
```

Remember the `myInfo` variable we defined in our source file? Here it
is.

``` {.screen}
ghci> myInfo
Book 9780135072455 "Algebra of Programming" ["Richard Bird","Oege de Moor"]
ghci> :type myInfo
myInfo :: BookInfo
```

We can construct new values interactively in `ghci`, too.

``` {.screen}
ghci> Book 0 "The Book of Imaginary Beings" ["Jorge Luis Borges"]
Book 0 "The Book of Imaginary Beings" ["Jorge Luis Borges"]
```

The `ghci` command `:type` lets us see what the type of an expression
is.

``` {.screen}
ghci> :type Book 1 "Cosmicomics" ["Italo Calvino"]
Book 1 "Cosmicomics" ["Italo Calvino"] :: BookInfo
ghci> cities = Book 173 "Use of Weapons" ["Iain M. Banks"]
```

To find out more about a type, we can use some of `ghci`\'s browsing
capabilities. The `:info` command gets `ghci` to tell us everything it
knows about a name.

``` {.screen}
ghci> :info BookInfo
data BookInfo = Book Int String [String]
        -- Defined at BookStore.hs:2:1
instance [safe] Show BookInfo -- Defined at BookStore.hs:3:27
```

We can also find out why we use `Book` to construct a new value of type
BookStore.

``` {.screen}
ghci> :type Book
Book :: Int -> String -> [String] -> BookInfo
```

We can treat a value constructor as just another function, one that
happens to create and return a new value of the type we desire.

### Naming types and values

When we introduced the type `BookInfo`, we deliberately chose to give
the type constructor `BookInfo` a different name from the value
constructor `Book`, purely to make it obvious which was which.

However, in Haskell, the names of types and values are independent of
each other. We only use a type constructor (i.e. the type\'s name) in a
type declaration or a type signature. We only use a value constructor in
actual code. Because these uses are distinct, there is no ambiguity if
we give a type constructor and a value constructor the same name. If we
are writing a type signature, we must be referring to a type
constructor. If we are writing an expression, we must be using the value
constructor.

<div>

[BookStore.hs]{.label}

``` {.haskell}
-- We will introduce the CustomerID type shortly.

data BookReview = BookReview BookInfo CustomerID String
```

</div>

This definition says that the type named `BookReview` has a value
constructor that is also named `BookReview`.

Not only is it *legal* for a value constructor to have the same name as
its type constructor, it\'s *normal*: you\'ll see this all the time in
regular Haskell code.

Type synonyms
-------------

We can introduce a *synonym* for an existing type at any time, to give a
type a more descriptive name. For example, the `String` in our
`BookReview` type doesn\'t tell us what the string is for, but we can
clarify this.

<div>

[BookStore.hs]{.label}

``` {.haskell}
type CustomerID = Int
type ReviewBody = String

data BetterReview = BetterReview BookInfo CustomerID ReviewBody
```

</div>

The `type` keyword introduces a type synonym. The new name is on the
left of the `=`, with the existing name on the right. The two names
identify the same type, so type synonyms are *purely* for making code
more readable.

We can also use a type synonym to create a shorter name for a verbose
type.

<div>

[BookStore.hs]{.label}

``` {.haskell}
type BookRecord = (BookInfo, BookReview)
```

</div>

This states that we can use `BookRecord` as a synonym for the tuple
`(BookInfo, BookReview)`. A type synonym only creates a new name that
refers to an existing type[^1]. We still use the same value constructors
to create a value of the type.

Algebraic data types
--------------------

The familiar `Bool` is the simplest common example of a category of type
called an *algebraic data type*. An algebraic data type can have more
than one value constructor.

<div>

[Bool.hs]{.label}

``` {.haskell}
data Bool = False | True
```

</div>

The `Bool` type has two value constructors, `True` and `False`. Each
value constructor is separated in the definition by a `|` character,
which we can read as \"or\": we can construct a `Bool` that has the
value `True`, or the value `False`. When a type has more than one value
constructor, they are usually referred to as *alternatives* or *cases*.
We can use any one of the alternatives to create a value of that type.

::: {.NOTE}
A note about naming

Although the phrase \"algebraic data type\" is long, we\'re being
careful to avoid using the acronym \"ADT\". That acronym is already
widely understood to stand for \"*abstract* data type\". Since Haskell
supports both algebraic and abstract data types, we\'ll be explicit and
avoid the acronym entirely.
:::

Each of an algebraic data type\'s value constructors can take zero or
more arguments. As an example, here\'s one way we might represent
billing information.

<div>

[BookStore.hs]{.label}

``` {.haskell}
type CardHolder = String
type CardNumber = String
type Address = [String]

data BillingInfo = CreditCard CardNumber CardHolder Address
                 | CashOnDelivery
                 | Invoice CustomerID
                   deriving (Show)
```

</div>

Here, we\'re saying that we support three ways to bill our customers. If
they want to pay by credit card, they must supply a card number, the
holder\'s name, and the holder\'s billing address as arguments to the
`CreditCard` value constructor. Alternatively, they can pay the person
who delivers their shipment. Since we don\'t need to store any extra
information about this, we specify no arguments for the `CashOnDelivery`
constructor. Finally, we can send an invoice to the specified customer,
in which case we need their CustomerID as an argument to the `Invoice`
constructor.

When we use a value constructor to create a value of type `BillingInfo`,
we must supply the arguments that it requires.

``` {.screen}
ghci> :type CreditCard
CreditCard :: CardNumber -> CardHolder -> Address -> BillingInfo
ghci> CreditCard "2901650221064486" "Thomas Gradgrind" ["Dickens", "England"]
CreditCard "2901650221064486" "Thomas Gradgrind" ["Dickens","England"]
ghci> :type it
it :: BillingInfo
ghci> Invoice

<interactive>:1:1: error:
    • No instance for (Show (CustomerID -> BillingInfo))
        arising from a use of ‘print’
        (maybe you haven't applied a function to enough arguments?)
    • In a stmt of an interactive GHCi command: print it
ghci> :type it
it :: BillingInfo
```

The `No instance` error message arose because we did not supply an
argument to the `Invoice` constructor. As a result, we were trying to
print the `Invoice` constructor itself. That constructor requires an
argument and returns a value, so it is a function. We cannot print
functions in Haskell, which is ultimately why the interpreter
complained.

### Tuples, algebraic data types, and when to use each

There is some overlap between tuples and user-defined algebraic data
types. If we wanted to, we could represent our `BookInfo` type from
earlier as an `(Int, String, [String])` tuple.

``` {.screen}
ghci> Book 2 "The Wealth of Networks" ["Yochai Benkler"]
Book 2 "The Wealth of Networks" ["Yochai Benkler"]
ghci> (2, "The Wealth of Networks", ["Yochai Benkler"])
(2,"The Wealth of Networks",["Yochai Benkler"])
```

Algebraic data types allow us to distinguish between otherwise identical
pieces of information. Two tuples with elements of the same type are
structurally identical, so they have the same type.

<div>

[Distinction.hs]{.label}

``` {.haskell}
a = ("Porpoise", "Grey")
b = ("Table", "Oak")
```

</div>

Since they have different names, two algebraic data types have distinct
types, even if they are otherwise structurally equivalent.

<div>

[Distinction.hs]{.label}

``` {.haskell}
data Cetacean = Cetacean String String
data Furniture = Furniture String String

c = Cetacean "Porpoise" "Grey"
d = Furniture "Table" "Oak"
```

</div>

This lets us bring the type system to bear in writing programs with
fewer bugs. With the tuples we defined above, we could conceivably pass
a description of a whale to a function expecting a chair, and the type
system could not help us. With the algebraic data types, there is no
such possibility of confusion.

Here is a more subtle example. Consider the following representations of
a two-dimensional vector.

<div>

[AlgebraicVector.hs]{.label}

``` {.haskell}
-- x and y coordinates or lengths.
data Cartesian2D = Cartesian2D Double Double
                   deriving (Eq, Show)

-- Angle and distance (magnitude).
data Polar2D = Polar2D Double Double
               deriving (Eq, Show)
```

</div>

The cartesian and polar forms use the same types for their two elements.
However, the *meanings* of the elements are different. Because
`Cartesian2D` and `Polar2D` are distinct types, the type system will not
let us accidentally use a `Cartesian2D` value where a `Polar2D` is
expected, or vice versa.

``` {.screen}
ghci> Cartesian2D (sqrt 2) (sqrt 2) == Polar2D (pi / 4) 2

<interactive>:2:34: error:
    • Couldn't match expected type ‘Cartesian2D’
                  with actual type ‘Polar2D’
    • In the second argument of ‘(==)’, namely ‘Polar2D (pi / 4) 2’
      In the expression:
        Cartesian2D (sqrt 2) (sqrt 2) == Polar2D (pi / 4) 2
      In an equation for ‘it’:
          it = Cartesian2D (sqrt 2) (sqrt 2) == Polar2D (pi / 4) 2
```

The `(==)` operator requires its arguments to have the same type.

::: {.TIP}
Comparing for equality

Notice that in the `deriving` clause for our vector types, we added
another word, `Eq`. This causes the Haskell implementation to generate
code that lets us compare the values for equality.
:::

If we used tuples to represent these values, we could quickly land
ourselves in hot water by mixing the two representations
inappropriately.

``` {.screen}
ghci> (1, 2) == (1, 2)
True
```

The type system can\'t rescue us here: as far as it\'s concerned, we\'re
comparing two `(Double, Double)` pairs, which is a perfectly valid thing
to do. Indeed, we cannot tell by inspection which of these values is
supposed to be polar or cartesian, but `(1,2)` has a different meaning
in each representation.

There is no hard and fast rule for deciding when it\'s better to use a
tuple or a distinct data type, but here\'s a rule of thumb to follow. If
you\'re using compound values widely in your code (as almost all
non-trivial programs do), adding `data` declarations will benefit you in
both type safety and readability. For smaller, localised uses, a tuple
is usually fine.

### Analogues to algebraic data types in other languages

Algebraic data types provide a single powerful way to describe data
types. Other languages often need several different features to achieve
the same degree of expressiveness. Here are some analogues from C and
C++, which might make it clearer what we can do with algebraic data
types, and how they relate to concepts that might be more familiar.

1.  The structure

    With just one constructor, an algebraic data type is similar to a
    tuple: it groups related values together into a compound value. It
    corresponds to a `struct` in C or C++, and its components correspond
    to the fields of a `struct`. Here\'s a C equivalent of the
    `BookInfo` type that we defined earlier.

    ``` {.c org-language="C"}
    struct book_info {
        int id;
        char *name;
        char **authors;
    };
    ```

    The main difference between the two is that the fields in the
    Haskell type are anonymous and positional.

    <div>

    [BookStore.hs]{.label}
    ``` {.haskell}
    data BookInfo = Book Int String [String]
                    deriving (Show)
    ```

    </div>

    By *positional*, we mean that the section number is in the first
    field of the Haskell type, and the title is in the second. We refer
    to them by location, not by name.

    In [the section called \"Pattern
    matching\"](3-defining-types-streamlining-functions.org::*Pattern matching)
    the fields of a `BookStore` value. In [the section called \"Record
    syntax\"](3-defining-types-streamlining-functions.org::*Record syntax)
    syntax for defining data types that looks a little more C-like.

2.  The enumeration

    Algebraic data types also serve where we\'d use an `enum` in C or
    C++, to represent a range of symbolic values. Such algebraic data
    types are sometimes referred to as enumeration types. Here\'s an
    example from C.

    ``` {.c org-language="C"}
    enum roygbiv {
        red,
        orange,
        yellow,
        green,
        blue,
        indigo,
        violet,
    };
    ```

    And here\'s a Haskell equivalent.

    <div>

    [Roygbiv.hs]{.label}
    ``` {.haskell}
    data Roygbiv = Red
                 | Orange
                 | Yellow
                 | Green
                 | Blue
                 | Indigo
                 | Violet
                   deriving (Eq, Show)
    ```

    </div>

    We can try these out in `ghci`.

    ``` {.screen}
    ghci> :type Yellow
    Yellow :: Roygbiv
    ghci> :type Red
    Red :: Roygbiv
    ghci> Red == Yellow
    False
    ghci> Green == Green
    True
    ```

    In C, the elements of an `enum` are integers. We can use an integer
    in a context where an `enum` is expected, and vice versa: a C
    compiler will automatically convert values between the two types.
    This can be a source of nasty bugs. In Haskell, this kind of problem
    does not occur. For example, we cannot use a `Roygbiv` value where
    an `Int` is expected.

    ``` {.screen}
    ghci> take 3 "foobar"
    "foo"
    ghci> take Red "foobar"

    <interactive>:3:6: error:
        • Couldn't match expected type ‘Int’ with actual type ‘Roygbiv’
        • In the first argument of ‘take’, namely ‘Red’
          In the expression: take Red "foobar"
          In an equation for ‘it’: it = take Red "foobar"
    ```

3.  The discriminated union

    If an algebraic data type has multiple alternatives, we can think of
    it as similar to a `union` in C or C++. A big difference between the
    two is that a union doesn\'t tell us which alternative is actually
    present; we have to explicitly and manually track which alternative
    we\'re using, usually in another field of an enclosing struct. This
    means that unions can be sources of nasty bugs, where our notion of
    which alternative we should be using is incorrect.

    ``` {.c org-language="C"}
    enum shape_type {
        shape_circle,
        shape_poly,
    };

    struct circle {
        struct vector centre;
        float radius;
    };

    struct poly {
        size_t num_vertices;
        struct vector *vertices;
    };

    struct shape
    {
        enum shape_type type;
        union {
        struct circle circle;
        struct poly poly;
        } shape;
    };
    ```

    In the example above, the `union` can contain valid data for either
    a `struct circle` or a `struct poly`. We have to use the
    `enum shape_type` by hand to indicate which kind of value is
    currently stored in the `union`.

    The Haskell version of this code is both dramatically shorter and
    safer than the C equivalent.

    <div>

    [ShapeUnion.hs]{.label}
    ``` {.haskell}
    type Vector = (Double, Double)

    data Shape = Circle Vector Double
               | Poly [Vector]
    ```

    </div>

    If we create a `Shape` value using the `Circle` constructor, the
    fact that we created a `Circle` is stored. When we later use a
    `Circle`, we can\'t accidentally treat it as a `Square`. We will see
    why in [the section called \"Pattern
    matching\"](3-defining-types-streamlining-functions.org::*Pattern matching)

    ::: {.TIP}
    A few notes

    From reading the preceding sections, it should now be clear that
    *all* of the data types that we define with the `data` keyword are
    algebraic data types. Some may have just one alternative, while
    others have several, but they\'re all using the same machinery.
    :::

Pattern matching
----------------

Now that we\'ve seen how to construct values with algebraic data types,
let\'s discuss how we work with these values. If we have a value of some
type, there are two things we would like to be able to do.

-   If the type has more than one value constructor, we need to be able
    to tell which value constructor was used to create the value.
-   If the value constructor has data components, we need to be able to
    extract those values.

Haskell has a simple, but tremendously useful, *pattern matching*
facility that lets us do both of these things.

A pattern lets us look inside a value and bind variables to the data it
contains. Here\'s an example of pattern matching in action on a `Bool`
value: we\'re going to reproduce the `not` function.

<div>

[MyNot.hs]{.label}

``` {.haskell}
myNot True  = False
myNot False = True
```

</div>

It might seem that we have two functions named `myNot` here, but Haskell
lets us define a function as a *series of equations*: these two clauses
are defining the behavior of the same function for different patterns of
input. On each line, the patterns are the items following the function
name, up until the `=` sign.

To understand how pattern matching works, let\'s step through an
example, say `myNot False`.

When we apply `myNot`, the Haskell runtime checks the value we supply
against the value constructor in the first pattern. This does not match,
so it tries against the second pattern. That match succeeds, so it uses
the right hand side of that equation as the result of the function
application.

Here is a slightly more extended example. This function adds together
the elements of a list.

<div>

[SumList.hs]{.label}

``` {.haskell}
sumList (x:xs) = x + sumList xs
sumList []     = 0
```

</div>

Let us step through the evaluation of `sumList [1,2]`. The list notation
`[1,2]` is shorthand for the expression `(1 : (2 : []))`. We begin by
trying to match the pattern in the first equation of the definition of
`sumList`. In the `(x : xs)` pattern, the \"`:`\" is the familiar list
constructor, `(:)`. We are now using it to match against a value, not to
construct one. The value `(1 : (2 : []))` was constructed with `(:)`, so
the constructor in the value matches the constructor in the pattern. We
say that the pattern *matches*, or that the match *succeeds*.

The variables `x` and `xs` are now \"bound to\" the constructor\'s
arguments, so `x` is given the value `1`, and `xs` the value `2 : []`.

The expression we are now evaluating is `1 + sumList (2 : [])`. We must
now recursively apply `sumList` to the value `2 : []`. Once again, this
was constructed using `(:)`, so the match succeeds. In our recursive
application of `sumList`, `x` is now bound to `2`, and `xs` to `[]`.

We are now evaluating `1 + (2 + sumList [])`. In this recursive
application of `sumList`, the value we are matching against is `[]`. The
value\'s constructor does not match the constructor in the first
pattern, so we skip this equation. Instead, we \"fall through\" to the
next pattern, which matches. The right hand side of this equation is
thus chosen as the result of this application.

The result of `sumList [1,2]` is thus `1 + (2 + (0))`, or `3`.

::: {.NOTE}
Ordering is important

As we have already mentioned, a Haskell implementation checks patterns
for matches in the order in which we specify them in our equations.
Matching proceeds from top to bottom, and stops at the first success.
Equations below a successful match have no effect.
:::

As a final note, there already exists a standard function, `sum`, that
performs this sum-of-a-list for us. Our `sumList` is purely for
illustration.

### Construction and deconstruction

Let\'s step back and take a look at the relationship between
constructing a value and pattern matching on it.

We apply a value constructor to build a value. The expression
`Book 9 "Close Calls" ["John Long"]` applies the `Book` constructor to
the values `9`, `"Close Calls"`, and `["John Long"]` to produce a new
value of type `BookInfo`.

When we pattern match against the `Book` constructor, we *reverse* the
construction process. First of all, we check to see if the value was
created using that constructor. If it was, we inspect it to obtain the
individual values that we originally supplied to the constructor when we
created the value.

Let\'s consider what happens if we match the pattern
`(Book id name authors)` against our example expression.

-   The match will succeed, because the constructor in the value matches
    the one in our pattern.
-   The variable `id` will be bound to `9`.
-   The variable `name` will be bound to `"Close Calls"`.
-   The variable `authors` will be bound to `["John Long"]`.

Because pattern matching acts as the inverse of construction, it\'s
sometimes referred to as /de/construction.

::: {.NOTE}
Deconstruction doesn\'t destroy anything

If you\'re steeped in object oriented programming jargon, don\'t confuse
deconstruction with destruction! Matching a pattern has no effect on the
value we\'re examining: it just lets us \"look inside\" it.
:::

### Further adventures

The syntax for pattern matching on a tuple is similar to the syntax for
constructing a tuple. Here\'s a function that returns the last element
of a 3-tuple.

<div>

[Tuple.hs]{.label}

``` {.haskell}
third (a, b, c) = c
```

</div>

There\'s no limit on how \"deep\" within a value a pattern can look.
This definition looks both inside a tuple and inside a list within that
tuple.

<div>

[Tuple.hs]{.label}

``` {.haskell}
complicated (True, a, x:xs, 5) = (a, xs)
```

</div>

We can try this out interactively.

``` {.screen}
ghci> :load Tuple.hs
[1 of 1] Compiling Main             ( Tuple.hs, interpreted )
Ok, one module loaded.
ghci> complicated (True, 1, [1,2,3], 5)
(1,[2,3])
```

Wherever a literal value is present in a pattern (`True` and `5` in the
tuple pattern above), that value must match exactly for the pattern
match to succeed. If every pattern within a series of equations fails to
match, we get a runtime error.

``` {.screen}
ghci> complicated (False, 1, [1,2,3], 5)
*** Exception: Tuple.hs:10:0-39: Non-exhaustive patterns in function complicated
```

For an explanation of this error message, skip forward a little, to [the
section called \"Exhaustive patterns and wild
cards\"](3-defining-types-streamlining-functions.org::*Exhaustive patterns and wild cards)

We can pattern match on an algebraic data type using its value
constructors. Recall the `BookInfo` type we defined earlier: we can
extract the values from a `BookInfo` as follows.

<div>

[BookStore.hs]{.label}

``` {.haskell}
bookID      (Book id title authors) = id
bookTitle   (Book id title authors) = title
bookAuthors (Book id title authors) = authors
```

</div>

Let\'s see it in action.

``` {.screen}
ghci> bookID (Book 3 "Probability Theory" ["E.T.H. Jaynes"])
3
ghci> bookTitle (Book 3 "Probability Theory" ["E.T.H. Jaynes"])
"Probability Theory"
ghci> bookAuthors (Book 3 "Probability Theory" ["E.T.H. Jaynes"])
["E.T.H. Jaynes"]
```

The compiler can infer the types of the accessor functions based on the
constructor we\'re using in our pattern.

``` {.screen}
ghci> :type bookID
bookID :: BookInfo -> Int
ghci> :type bookTitle
bookTitle :: BookInfo -> String
ghci> :type bookAuthors
bookAuthors :: BookInfo -> [String]
```

If we use a literal value in a pattern, the corresponding part of the
value we\'re matching against must contain an identical value. For
instance, the pattern `(3 : xs)` first of all checks that a value is a
non-empty list, by matching against the `(:)` constructor. It also
ensures that the head of the list has the exact value `3`. If both of
these conditions hold, the tail of the list will be bound to the
variable `xs`.

### Variable naming in patterns

As you read functions that match on lists, you\'ll frequently find that
the names of the variables inside a pattern resemble `(x : xs)` or
`(d : ds)`. This is a popular naming convention. The idea is that the
name `xs` has an \"`s`\" on the end of its name as if it\'s the
\"plural\" of `x`, because `x` contains the head of the list, and `xs`
the remaining elements.

### The wild card pattern

We can indicate that we don\'t care what is present in part of a
pattern. The notation for this is the underscore character \"`_`\",
which we call a *wild card*. We use it as follows.

<div>

[BookStore.hs]{.label}

``` {.haskell}
nicerID      (Book id _     _      ) = id
nicerTitle   (Book _  title _      ) = title
nicerAuthors (Book _  _     authors) = authors
```

</div>

Here, we have tidier versions of the accessor functions we introduced
earlier. Now, there\'s no question about which element we\'re using in
each function.

In a pattern, a wild card acts similarly to a variable, but it doesn\'t
bind a new variable. As the examples above indicate, we can use more
than one wild card in a single pattern.

Another advantage of wild cards is that a Haskell compiler can warn us
if we introduce a variable name in a pattern, but do not use it in a
function\'s body. Defining a variable, but forgetting to use it, can
often indicate the presence of a bug, so this is a helpful feature. If
we use a wild card instead of a variable that we do not intend to use,
the compiler won\'t complain.

### Exhaustive patterns and wild cards

When writing a series of patterns, it\'s important to cover all of a
type\'s constructors. For example, if we\'re inspecting a list, we
should have one equation that matches the non-empty constructor `(:)`,
and one that matches the empty-list constructor `[]`.

Let\'s see what happens if we fail to cover all the cases. Here, we
deliberately omit a check for the `[]` constructor.

<div>

[BadPattern.hs]{.label}

``` {.haskell}
badExample (x:xs) = x + badExample xs
```

</div>

If we apply this to a value that it cannot match, we\'ll get an error at
runtime: our software has a bug!

``` {.screen}
ghci> badExample []
*** Exception: BadPattern.hs:4:0-36: Non-exhaustive patterns in function badExample
```

In this example, no equation in the function\'s definition matches the
value `[]`.

::: {.TIP}
Warning about incomplete patterns

GHC provides a helpful compilation option, `-fwarn-incomplete-patterns`,
that will cause it to print a warning during compilation if a sequence
of patterns don\'t match all of a type\'s value constructors.
:::

If we need to provide a default behavior in cases where we don\'t care
about specific constructors, we can use a wild card pattern.

<div>

[BadPattern.hs]{.label}

``` {.haskell}
goodExample (x:xs) = x + goodExample xs
goodExample _      = 0
```

</div>

The wild card above will match the `[]` constructor, so applying this
function does not lead to a crash.

``` {.screen}
ghci> goodExample []
0
ghci> goodExample [1,2]
3
```

Record syntax
-------------

Writing accessor functions for each of a data type\'s components can be
repetitive and tedious.

<div>

[BookStore.hs]{.label}

``` {.haskell}
nicerID      (Book id _     _      ) = id
nicerTitle   (Book _  title _      ) = title
nicerAuthors (Book _  _     authors) = authors
```

</div>

We call this kind of code *boilerplate*: necessary, but bulky and
irksome. Haskell programmers don\'t like boilerplate. Fortunately, the
language addresses this particular boilerplate problem: we can define a
data type, and accessors for each of its components, simultaneously.
(The positions of the commas here is a matter of preference. If you
like, put them at the end of a line instead of the beginning.)

<div>

[BookStore.hs]{.label}

``` {.haskell}
data Customer = Customer {
      customerID      :: CustomerID
    , customerName    :: String
    , customerAddress :: Address
    } deriving (Show)
```

</div>

This is almost exactly identical in meaning to the following, more
familiar form.

<div>

[AltCustomer.hs]{.label}

``` {.haskell}
data Customer = Customer Int String [String]
                deriving (Show)

customerID :: Customer -> Int
customerID (Customer id _ _) = id

customerName :: Customer -> String
customerName (Customer _ name _) = name

customerAddress :: Customer -> [String]
customerAddress (Customer _ _ address) = address
```

</div>

For each of the fields that we name in our type definition, Haskell
creates an accessor function of that name.

``` {.screen}
ghci> :type customerID
customerID :: Customer -> CustomerID
```

We can still use the usual application syntax to create a value of this
type.

<div>

[BookStore.hs]{.label}

``` {.haskell}
customer1 = Customer 271828 "J.R. Hacker"
            ["255 Syntax Ct",
             "Milpitas, CA 95134",
             "USA"]
```

</div>

Record syntax adds a more verbose notation for creating a value. This
can sometimes make code more readable.

<div>

[BookStore.hs]{.label}

``` {.haskell}
customer2 = Customer {
              customerID = 271828
            , customerAddress = ["1048576 Disk Drive",
                                 "Milpitas, CA 95134",
                                 "USA"]
            , customerName = "Jane Q. Citizen"
            }
```

</div>

If we use this form, we can vary the order in which we list fields.
Here, we have moved the name and address fields from their positions in
the declaration of the type.

When we define a type using record syntax, it also changes the way the
type\'s values are printed.

``` {.screen}
ghci> customer1
Customer {customerID = 271828, customerName = "J.R. Hacker", customerAddress = ["255 Syntax Ct","Milpitas, CA 95134","USA"]}
```

For comparison, let\'s look at a `BookInfo` value; we defined this type
without record syntax.

``` {.screen}
ghci> cities
Book 173 "Use of Weapons" ["Iain M. Banks"]
```

The accessor functions that we get \"for free\" when we use record
syntax really are normal Haskell functions.

``` {.screen}
ghci> :type customerName
customerName :: Customer -> String
ghci> customerName customer1
"J.R. Hacker"
```

The standard `System.Time` module makes good use of record syntax.
Here\'s a type defined in that module:

``` {.haskell}
data CalendarTime = CalendarTime {
  ctYear                      :: Int,
  ctMonth                     :: Month,
  ctDay, ctHour, ctMin, ctSec :: Int,
  ctPicosec                   :: Integer,
  ctWDay                      :: Day,
  ctYDay                      :: Int,
  ctTZName                    :: String,
  ctTZ                        :: Int,
  ctIsDST                     :: Bool
}
```

In the absence of record syntax, it would be painful to extract specific
fields from a type like this. The notation makes it easier to work with
large structures.

Parameterised types
-------------------

We\'ve repeatedly mentioned that the list type is polymorphic: the
elements of a list can be of any type. We can also add polymorphism to
our own types. To do this, we introduce type variables into a type
declaration. The prelude defines a type named `Maybe`: we can use this
to represent a value that could be either present or missing, e.g. a
field in a database row that could be null.

<div>

[Nullable.hs]{.label}

``` {.haskell}
data Maybe a = Just a
             | Nothing
```

</div>

Here, the variable `a` is not a regular variable: it\'s a type variable.
It indicates that the `Maybe` type takes another type as its parameter.
This lets us use Maybe on values of any type.

<div>

[Nullable.hs]{.label}

``` {.haskell}
someBool = Just True

someString = Just "something"
```

</div>

As usual, we can experiment with this type in `ghci`.

``` {.screen}
ghci> Just 1.5
Just 1.5
ghci> Nothing
Nothing
ghci> :type Just "invisible bike"
Just "invisible bike" :: Maybe [Char]
```

Maybe is a polymorphic, or generic, type. We give the `Maybe` type
constructor a parameter to create a specific type, such as `Maybe Int`
or `Maybe [Bool]`. As we might expect, these types are distinct.

We can nest uses of parameterised types inside each other, but when we
do, we may need to use parentheses to tell the Haskell compiler how to
parse our expression.

<div>

[Nullable.hs]{.label}

``` {.haskell}
wrapped = Just (Just "wrapped")
```

</div>

To once again extend an analogy to more familiar languages,
parameterised types bear some resemblance to templates in C++, and to
generics in Java. Just be aware that this is a shallow analogy.
Templates and generics were added to their respective languages long
after the languages were initially defined, and have an awkward feel.
Haskell\'s parameterised types are simpler and easier to use, as the
language was designed with them from the beginning.

Recursive types
---------------

The familiar list type is *recursive*: it\'s defined in terms of itself.
To understand this, let\'s create our own list-like type. We\'ll use
`Cons` in place of the `(:)` constructor, and `Nil` in place of `[]`.

``` {.haskell}
-- file ListADT.hs
data List a = Cons a (List a)
            | Nil
              deriving (Show)
```

Because `List a` appears on both the left and the right of the `=` sign,
the type\'s definition refers to itself. If we want to use the `Cons`
constructor to create a new value, we must supply one value of type `a`,
and another of type `List a`. Let\'s see where this leads us in
practice.

The simplest value of type `List a` that we can create is `Nil`. Save
the type definition in a file, then load it into `ghci`.

``` {.screen}
ghci> Nil
Nil
```

Because `Nil` has a `List` type, we can use it as a parameter to `Cons`.

``` {.screen}
ghci> Cons 0 Nil
Cons 0 Nil
```

And because `Cons 0 Nil` has the type `List a`, we can use this as a
parameter to `Cons`.

``` {.screen}
ghci> Cons 1 it
Cons 1 (Cons 0 Nil)
ghci> Cons 2 it
Cons 2 (Cons 1 (Cons 0 Nil))
ghci> Cons 3 it
Cons 3 (Cons 2 (Cons 1 (Cons 0 Nil)))
```

We could continue in this fashion indefinitely, creating ever longer
`Cons` chains, each with a single `Nil` at the end.

::: {.TIP}
Is `List` an acceptable list?

We can easily prove to ourselves that our `List a` type has the same
shape as the built-in list type `[a]`. To do this, we write a function
that takes any value of type `[a]`, and produces a value of type
`List a`.

<div>

[ListADT.hs]{.label}

``` {.haskell}
fromList (x:xs) = Cons x (fromList xs)
fromList []     = Nil
```

</div>

By inspection, this clearly substitutes a `Cons` for every `(:)`, and a
`Nil` for each `[]`. This covers both of the built-in list type\'s
constructors. The two types are *isomorphic*; they have the same shape.

``` {.screen}
ghci> fromList "durian"
Cons 'd' (Cons 'u' (Cons 'r' (Cons 'i' (Cons 'a' (Cons 'n' Nil)))))
ghci> fromList [Just True, Nothing, Just False]
Cons (Just True) (Cons Nothing (Cons (Just False) Nil))
```
:::

For a third example of what a recursive type is, here is a definition of
a binary tree type.

<div>

[Tree.hs]{.label}

``` {.haskell}
data Tree a = Node a (Tree a) (Tree a)
            | Empty
              deriving (Show)
```

</div>

A binary tree is either a node with two children, which are themselves
binary trees, or an empty value.

This time, let\'s search for insight by comparing our definition with
one from a more familiar language. Here\'s a similar class definition in
Java.

``` {.java}
class Tree<A>
{
    A value;
    Tree<A> left;
    Tree<A> right;

    public Tree(A v, Tree<A> l, Tree<A> r)
    {
    value = v;
    left = l;
    right = r;
    }
}
```

The one significant difference is that Java lets us use the special
value `null` anywhere to indicate \"nothing\", so we can use `null` to
indicate that a node is missing a left or right child. Here\'s a small
function that constructs a tree with two leaves (a leaf, by convention,
has no children).

``` {.java}
class Example
{
    static Tree<String> simpleTree()
    {
    return new Tree<String>(
            "parent",
        new Tree<String>("left leaf", null, null),
        new Tree<String>("right leaf", null, null));
    }
}
```

In Haskell, we don\'t have an equivalent of `null`. We could use the
`Maybe` type to provide a similar effect, but that bloats the pattern
matching. Instead, we\'ve decided to use a no-argument `Empty`
constructor. Where the Java example provides `null` to the `Tree`
constructor, we supply `Empty` in Haskell.

<div>

[Tree.hs]{.label}

``` {.haskell}
simpleTree = Node "parent" (Node "left child" Empty Empty)
                           (Node "right child" Empty Empty)
```

</div>

### Exercises

1.  Write the converse of `fromList` for the List type: a function that
    takes a List a and generates a `[a]`.
2.  Define a tree type that has only one constructor, like our Java
    example. Instead of the `Empty` constructor, use the `Maybe` type to
    refer to a node\'s children.

Reporting errors
----------------

Haskell provides a standard function, `error :: String -> a`, that we
can call when something has gone terribly wrong in our code. We give it
a string parameter, which is the error message to display. Its type
signature looks peculiar: how can it produce a value of any type `a`
given only a string?

It has a result type of `a` so that we can call it anywhere and it will
always have the right type. However, it does not return a value like a
normal function: instead, it *immediately aborts evaluation*, and prints
the error message we give it.

The `mySecond` function returns the second element of its input list,
but fails if its input list isn\'t long enough.

<div>

[MySecond.hs]{.label}

``` {.haskell}
mySecond :: [a] -> a

mySecond xs = if null (tail xs)
              then error "list too short"
              else head (tail xs)
```

</div>

As usual, we can see how this works in practice in `ghci`.

``` {.screen}
ghci> mySecond "xi"
'i'
ghci> mySecond [2]
*** Exception: list too short
ghci> head (mySecond [[9]])
*** Exception: list too short
```

Notice the third case above, where we try to use the result of the call
to `mySecond` as the argument to another function. Evaluation still
terminates and drops us back to the `ghci` prompt. This is the major
weakness of using `error`: it doesn\'t let our caller distinguish
between a recoverable error and a problem so severe that it really
should terminate our program.

As we have already seen, a pattern matching failure causes a similar
unrecoverable error.

``` {.screen}
ghci> mySecond []
*** Exception: Prelude.tail: empty list
```

### A more controlled approach

We can use the `Maybe` type to represent the possibility of an error.

If we want to indicate that an operation has failed, we can use the
`Nothing` constructor. Otherwise, we wrap our value with the `Just`
constructor.

Let\'s see how our `mySecond` function changes if we return a `Maybe`
value instead of calling `error`.

<div>

[MySecond.hs]{.label}

``` {.haskell}
safeSecond :: [a] -> Maybe a

safeSecond [] = Nothing
safeSecond xs = if null (tail xs)
                then Nothing
                else Just (head (tail xs))
```

</div>

If the list we\'re passed is too short, we return `Nothing` to our
caller. This lets them decide what to do, where a call to `error` would
force a crash.

``` {.screen}
ghci> safeSecond []
Nothing
ghci> safeSecond [1]
Nothing
ghci> safeSecond [1,2]
Just 2
ghci> safeSecond [1,2,3]
Just 2
```

To return to an earlier topic, we can further improve the readability of
this function with pattern matching.

<div>

[MySecond.hs]{.label}

``` {.haskell}
tidySecond :: [a] -> Maybe a

tidySecond (_:x:_) = Just x
tidySecond _       = Nothing
```

</div>

The first pattern only matches if the list is at least two elements long
(it contains two list constructors), and it binds the variable `x` to
the list\'s second element. The second pattern is matched if the first
fails.

Introducing local variables
---------------------------

Within the body of a function, we can introduce new local variables
whenever we need them, using a `let` expression. Here is a simple
function that determines whether we should lend some money to a
customer. We meet a money reserve of at least 100, we return our new
balance after subtracting the amount we have loaned.

<div>

[Lending.hs]{.label}

``` {.haskell}
lend amount balance = let reserve    = 100
                          newBalance = balance - amount
                      in if balance < reserve
                         then Nothing
                         else Just newBalance
```

</div>

The keywords to look out for here are `let`, which starts a block of
variable declarations, and `in`, which ends it. Each line introduces a
new variable. The name is on the left of the `=`, and the expression to
which it is bound is on the right.

::: {.NOTE}
Special notes

Let us re-emphasise our wording: a name in a `let` block is bound to an
*expression*, not to a *value*. Because Haskell is a lazy language, the
expression associated with a name won\'t actually be evaluated until
it\'s needed. In the above example, we will not compute the value of
`newBalance` if we do not meet our reserve.

When we define a variable in a `let` block, we refer to it as a
*`let`-bound* variable. This simply means what it says: we have bound
the variable in a `let` block.

Also, our use of white space here is important. We\'ll talk in more
detail about the layout rules in [the section called \"The offside rule
and white space in an
expression\"](3-defining-types-streamlining-functions.org::*The offside rule and white space in an expression)
:::

We can use the names of a variable in a `let` block both within the
block of declarations and in the expression that follows the `in`
keyword.

In general, we\'ll refer to the places within our code where we can use
a name as the name\'s *scope*. If we can use a name, it\'s *in scope*,
otherwise it\'s *out of scope*. If a name is visible throughout a source
file, we say it\'s at the *top level*.

### Shadowing

We can \"nest\" multiple `let` blocks inside each other in an
expression.

<div>

[NestedLets.hs]{.label}

``` {.haskell}
foo = let a = 1
      in let b = 2
         in a + b
```

</div>

It\'s perfectly legal, but not exactly wise, to repeat a variable name
in a nested `let` expression.

<div>

[NestedLets.hs]{.label}

``` {.haskell}
bar = let x = 1
      in ((let x = "foo" in x), x)
```

</div>

Here, the inner `x` is hiding, or *shadowing*, the outer `x`. It has the
same name, but a different type and value.

``` {.screen}
ghci> bar
("foo",1)
```

We can also shadow a function\'s parameters, leading to even stranger
results. What is the type of this function?

<div>

[NestedLets.hs]{.label}

``` {.haskell}
quux a = let a = "foo"
         in a ++ "eek!"
```

</div>

Because the function\'s argument `a` is never used in the body of the
function, due to being shadowed by the `let`-bound `a`, the argument can
have any type at all.

``` {.screen}
ghci> :type quux
quux :: t -> [Char]
```

::: {.TIP}
Compiler warnings are your friends

Shadowing can obviously lead to confusion and nasty bugs, so GHC has a
helpful `-fwarn-name-shadowing` option. When enabled, GHC will print a
warning message any time we shadow a name.
:::

### The where clause

We can use another mechanism to introduce local variables: the `where`
clause. The definitions in a `where` clause apply to the code that
*precedes* it. Here\'s a similar function to `lend`, using `where`
instead of `let`.

<div>

[Lending.hs]{.label}

``` {.haskell}
lend2 amount balance = if amount < reserve * 0.5
                       then Just newBalance
                       else Nothing
    where reserve    = 100
          newBalance = balance - amount
```

</div>

While a `where` clause may initially seem weird, it offers a wonderful
aid to readability. It lets us direct our reader\'s focus to the
important details of an expression, with the supporting definitions
following afterwards. After a while, you may find yourself missing
`where` clauses in languages that lack them.

As with `let` expressions, white space is significant in `where`
clauses. We will talk more about the layout rules shortly, in [the
section called \"The offside rule and white space in an
expression\"](3-defining-types-streamlining-functions.org::*The offside rule and white space in an expression)

### Local functions, global variables

You\'ll have noticed that Haskell\'s syntax for defining a variable
looks very similar to its syntax for defining a function. This symmetry
is preserved in `let` and `where` blocks: we can define local
*functions* just as easily as local *variables*.

<div>

[LocalFunction.hs]{.label}

``` {.haskell}
pluralise :: String -> [Int] -> [String]
pluralise word counts = map plural counts
    where plural 0 = "no " ++ word ++ "s"
          plural 1 = "one " ++ word
          plural n = show n ++ " " ++ word ++ "s"
```

</div>

We have defined a local function, `plural`, that consists of several
equations. Local functions can freely use variables from the scopes that
enclose them: here, we use `word` from the definition of the outer
function `pluralise`. In the definition of `pluralise`, the `map`
function (which we\'ll be revisiting in the next chapter) applies the
local function `plural` to every element of the `counts` list.

We can also define variables, as well as functions, at the top level of
a source file.

<div>

[GlobalVariable.hs]{.label}

``` {.haskell}
itemName = "Weighted Companion Cube"
```

</div>

The offside rule and white space in an expression
-------------------------------------------------

In our definitions of `lend` and `lend2`, the left margin of our text
wandered around quite a bit. This was not an accident: normally in
Haskell white space has meaning: it uses the code layout [as defined in
the
report](https://www.haskell.org/onlinereport/haskell2010/haskellch10.html#x17-17800010.3)
as a cue to parse it. This is sometimes called the *offside rule*.

At the beginning of a source file, the first top level declaration or
definition can start in any column, and the Haskell compiler or
interpreter remembers that indentation level. Every subsequent top level
declaration must have the same indentation.

Here\'s an illustration of the top level indentation rule. Our first
file, `GoodIndent.hs`, is well behaved.

<div>

[GoodIndent.hs]{.label}

``` {.haskell}
-- This is the leftmost column.

  -- It's fine for top-level declarations to start in any column...
  firstGoodIndentation = 1

  -- ...provided all subsequent declarations do, too!
  secondGoodIndentation = 2
```

</div>

Our second, `BadIndent.hs`, doesn\'t play by the rules.

<div>

[BadIndent.hs]{.label}

``` {.haskell}
-- This is the leftmost column.

    -- Our first declaration is in column 4.
    firstBadIndentation = 1

  -- Our second is left of the first, which is illegal!
  secondBadIndentation = 2
```

</div>

Here\'s what happens when we try to load the two files into `ghci`.

``` {.screen}
ghci> :load GoodIndent.hs
[1 of 1] Compiling Main             ( GoodIndent.hs, interpreted )
Ok, one module loaded.
ghci> :load BadIndent.hs
[1 of 1] Compiling Main             ( BadIndent.hs, interpreted )

BadIndent.hs:8:3: error:
    parse error on input ‘secondBadIndentation’
  |
8 |   secondBadIndentation = 2
  |   ^^^^^^^^^^^^^^^^^^^^
Failed, no modules loaded.
```

An empty following line is treated as a continuation of the current
item, as is a following line indented further to the right.

The rules for `let` expressions and `where` clauses are similar. After a
`let` or `where` keyword, the Haskell compiler or interpreter remembers
the indentation of the next token it sees. If the line that follows is
empty, or its indentation is further to the right, it is considered to
continue the previous line. If the indentation is the same as the start
of the preceding item, this is treated as beginning a new item in the
same block.

<div>

[Indentation.hs]{.label}

``` {.haskell}
foo = let firstDefinition = blah blah
          -- a comment-only line is treated as empty
                            continuation blah

          -- we reduce the indentation, so this is a new definition
          secondDefinition = yada yada
                             continuation yada
      in whatever
```

</div>

Here are nested uses of `let` and `where`.

<div>

[LetLet.hs]{.label}

``` {.haskell}
bar = let b = 2
          c = True
      in let a = b
         in (a, c)
```

</div>

The name `a` is only visible within the inner `let` expression. It\'s
not visible in the outer `let`. If we try to use the name `a` there,
we\'ll get a compilation error. The indentation gives both us and the
compiler a visual cue as to what is currently in scope.

<div>

[WhereWhere.hs]{.label}

``` {.haskell}
foo = x
    where x = y
              where y = 2
```

</div>

Similarly, the scope of the first `where` clause is the definition of
`foo`, but the scope of the second is just the first `where` clause.

The indentation we use for the `let` and `where` clauses makes our
intentions easy to figure out.

### A note about tabs versus spaces

The default in Haskell code is to indent using spaces. `ghc` and `ghci`
will warn you if you indent with tabs unless you disable it with the
`-Wno-tabs` flag. The reason for this is that it is easier to align
expressions using spaces.

If you like tabs you can use them as long as you correctly align
expressions. For the compiler a tab equals to eight spaces and uses this
amount to determine if an indented expression is correctly aligned. In
the next example you\'ll see the `b` aligned with the `a` only if the
tab width equals eight in whichever app you are using to read this text.

<div>

[TabAlign.hs]{.label}

``` {.haskell}
x = let a = 1
    b = 2
    in a + b
```

</div>

We need the amount of characters before the `a` to be equal to eight or
the `b` won\'t be aligned and won\'t compile. Totally impracticall. It
is better to break the expressions as below.

<div>

[TabAlign2.hs]{.label}

``` {.haskell}
x = let
        a = 1
        b = 2
    in a + b
```

</div>

As long as you correctly align expressions you can even [mix spaces and
tabs](http://dmwit.com/tabs/).

### The offside rule is not mandatory

We can use explicit structuring instead of layout to indicate what we
mean. To do so, we start a block of equations with an opening curly
brace; separate each item with a semicolon; and finish the block with a
closing curly brace. The following two uses of `let` have the same
meanings.

<div>

[Braces.hs]{.label}

``` {.haskell}
bar = let a = 1
          b = 2
          c = 3
      in a + b + c

foo = let { a = 1;  b = 2;
        c = 3 }
      in a + b + c
```

</div>

When we use explicit structuring, the normal layout rules don\'t apply,
which is why we can get away with unusual indentation in the second
`let` expression.

We can use explicit structuring anywhere that we\'d normally use layout.
It\'s valid for `where` clauses, and even top-level declarations. Just
remember that although the facility exists, explicit structuring is
hardly ever actually *used* in Haskell programs.

The case expression
-------------------

Function definitions are not the only place where we can use pattern
matching. The `case` construct lets us match patterns within an
expression. Here\'s what it looks like. This function (defined for us in
`Data.Maybe`) unwraps a `Maybe` value, using a default if the value is
`Nothing`.

<div>

[Guard.hs]{.label}

``` {.haskell}
fromMaybe defval wrapped =
    case wrapped of
      Nothing     -> defval
      Just value  -> value
```

</div>

The `case` keyword is followed by an arbitrary expression: the pattern
match is performed against the result of this expression. The `of`
keyword signifies the end of the expression and the beginning of the
block of patterns and expressions.

Each item in the block consists of a pattern, followed by an arrow `->`,
followed by an expression to evaluate if that pattern matches. These
expressions must all have the same type. The result of the `case`
expression is the result of the expression associated with the first
pattern to match. Matches are attempted from top to bottom.

To express \"here\'s the expression to evaluate if none of the other
patterns match\", we just use the wild card pattern `_` as the last in
our list of patterns. If a pattern match fails, we will get the same
kind of runtime error as we saw earlier.

Common beginner mistakes with patterns
--------------------------------------

There are a few ways in which new Haskell programmers can misunderstand
or misuse patterns. Here are some attempts at pattern matching gone
awry. Depending on what you expect one of these examples to do, it might
contain a surprise.

### Incorrectly matching against a variable

<div>

[BogusPattern.hs]{.label}

``` {.haskell}
data Fruit = Apple | Orange

apple = "apple"

orange = "orange"

whichFruit :: String -> Fruit
whichFruit f = case f of
                 apple  -> Apple
                 orange -> Orange
```

</div>

A naive glance suggests that this code is trying to check the value `f`
to see whether it matches the value `apple` or `orange`.

It is easier to spot the mistake if we rewrite the code in an equational
style.

<div>

[BogusPattern.hs]{.label}

``` {.haskell}
equational apple = Apple
equational orange = Orange
```

</div>

Now can you see the problem? Here, it is more obvious `apple` does not
refer to the top level value named `apple`: it is a local pattern
variable.

::: {.NOTE}
Irrefutable patterns

We refer to a pattern that always succeeds as *irrefutable*. Plain
variable names and the wild card `_` are examples of irrefutable
patterns.
:::

Here\'s a corrected version of this function.

<div>

[BogusPattern.hs]{.label}

``` {.haskell}
betterFruit f = case f of
                  "apple"  -> Apple
                  "orange" -> Orange
```

</div>

We fixed the problem by matching against the literal values `"apple"`
and `"orange"`.

### Incorrectly trying to compare for equality

What if we want to compare the values stored in two nodes of type Tree,
and return one of them if they\'re equal? Here\'s an attempt.

<div>

[BadTree.hs]{.label}

``` {.haskell}
bad_nodesAreSame (Node a _ _) (Node a _ _) = Just a
bad_nodesAreSame _            _            = Nothing
```

</div>

A name can only appear once in a set of pattern bindings. We cannot
place a variable in multiple positions to express the notion \"this
value and that should be identical\". Instead, we\'ll solve this problem
using *guards*, another invaluable Haskell feature.

Conditional evaluation with guards
----------------------------------

Pattern matching limites us to performing fixed tests of a value\'s
shape. Although this is useful, we will often want to make a more
expressive check before evaluating a function\'s body. Haskell provides
a feature, *guards*, that give us this ability. We\'ll introduce the
idea with a modification of the function we wrote to compare two nodes
of a tree.

<div>

[BadTree.hs]{.label}

``` {.haskell}
nodesAreSame (Node a _ _) (Node b _ _)
    | a == b     = Just a
nodesAreSame _ _ = Nothing
```

</div>

In this example, we use pattern matching to ensure that we are looking
at values of the right shape, and a guard to compare pieces of them.

A pattern can be followed by zero or more guards, each an expression of
type `Bool`. A guard is introduced by a `|` symbol. This is followed by
the guard expression, then an `=` symbol (or `->` if we\'re in a `case`
expression), then the body to use if the guard expression evaluates to
`True`. If a pattern matches, each guard associated with that pattern is
evaluated, in the order in which they are written. If a guard succeeds,
the body affiliated with it is used as the result of the function. If no
guard succeeds, pattern matching moves on to the next pattern.

When a guard expression is evaluated, all of the variables mentioned in
the pattern with which it is associated are bound and can be used.

Here is a reworked version of our `lend` function that uses guards.

<div>

[Lending.hs]{.label}

``` {.haskell}
lend3 amount balance
     | amount <= 0            = Nothing
     | amount > reserve * 0.5 = Nothing
     | otherwise              = Just newBalance
    where reserve    = 100
          newBalance = balance - amount
```

</div>

The special-looking guard expression `otherwise` is simply a variable
bound to the value `True`, to aid readability.

We can use guards anywhere that we can use patterns. Writing a function
as a series of equations using pattern matching and guards can make it
much clearer. Remember the `myDrop` function we defined in [the section
called \"Conditional
evaluation\"](2-types-and-functions.org::*Conditional evaluation)

<div>

[myDrop.hs]{.label}

``` {.haskell}
myDrop n xs = if n <= 0 || null xs
              then xs
              else myDrop (n - 1) (tail xs)
```

</div>

Here is a reformulation that uses patterns and guards.

<div>

[myDrop.hs]{.label}

``` {.haskell}
niceDrop n xs | n <= 0 = xs
niceDrop _ []          = []
niceDrop n (_:xs)      = niceDrop (n - 1) xs
```

</div>

This change in style lets us enumerate *up front* the cases in which we
expect a function to behave differently. If we bury the decisions inside
a function as `if` expressions, the code becomes harder to read.

Exercises
---------

1.  Write a function that computes the number of elements in a list. To
    test it, ensure that it gives the same answers as the standard
    `length` function.
2.  Add a type signature for your function to your source file. To test
    it, load the source file into `ghci` again.
3.  Write a function that computes the mean of a list, i.e. the sum of
    all elements in the list divided by its length. (You may need to use
    the `fromIntegral` function to convert the length of the list from
    an integer into a floating point number).
4.  Turn a list into a palindrome, i.e. it should read the same both
    backwards and forwards. For example, given the list `[1,2,3]`, your
    function should return `[1,2,3,3,2,1]`.
5.  Write a function that determines whether its input list is a
    palindrome.
6.  Create a function that sorts a list of lists based on the length of
    each sublist. (You may want to look at the `sortBy` function from
    the `Data.List` module.)
7.  Define a function that joins a list of lists together using a
    separator value:

<div>

[Intersperse.hs]{.label}

``` {.haskell}
intersperse :: a -> [[a]] -> [a]
```

</div>

The separator should appear between elements of the list, but should not
follow the last element. Your function should behave as follows.

``` {.screen}
ghci> :load Intersperse
[1 of 1] Compiling Main             ( Intersperse.hs, interpreted )
Ok, modules loaded: Main.
ghci> intersperse ',' []
""
ghci> intersperse ',' ["foo"]
"foo"
ghci> intersperse ',' ["foo","bar","baz","quux"]
"foo,bar,baz,quux"
```

1.  Using the binary tree type that we defined earlier in this chapter,
    write a function that will determine the height of the tree. The
    height is the largest number of hops from the root to an `Empty`.
    For example, the tree `Empty` has height zero;
    `Node "x" Empty Empty` has height one;
    `Node "x" Empty (Node "y" Empty Empty)` has height two; and so on.
2.  Consider three two-dimensional points *a*, *b*, and *c*. If we look
    at the angle formed by the line segment from *a* to *b* and the line
    segment from *b* to *c*, it either turns left, turns right, or forms
    a straight line. Define a `Direction` data type that lets you
    represent these possibilities.
3.  Write a function that calculates the turn made by three 2D points
    and returns a Direction.
4.  Define a function that takes a list of 2D points and computes the
    direction of each successive triple. Given a list of points
    `[a,b,c,d,e]`, it should begin by computing the turn made by
    `[a,b,c]`, then the turn made by `[b,c,d]`, then `[c,d,e]`. Your
    function should return a list of `Direction`.
5.  Using the code from the preceding three exercises, implement
    Graham\'s scan algorithm for the convex hull of a set of 2D points.
    You can find good description of what a [convex
    hull](http://en.wikipedia.org/wiki/Convex_hull) is, and how the
    [Graham scan algorithm](http://en.wikipedia.org/wiki/Graham_scan)
    should work, on [Wikipedia](http://en.wikipedia.org/).

Footnotes
---------

[^1]: If you are familiar with C or C++, it is analogous to a `typedef`.
