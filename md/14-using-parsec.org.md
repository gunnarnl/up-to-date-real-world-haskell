Chapter 16. Using Parsec
========================

The task of parsing a file, or data of various types, is a common one
for programmers. We already learned about Haskell\'s support for regular
expressions back in [the section called \"Regular expressions in
Haskell\"](8-efficient-file-processing-regular-expressions-and-file-name-matching.org::*Regular expressions in Haskell)
expressions are nice for many tasks, but they rapidly become unwieldy,
or cannot be used at all, when dealing with a complex data format. For
instance, we cannot use regular expressions to parse source code from
most programming languages.

Parsec is a useful parser combinator library, with which we combine
small parsing functions to build more sophisticated parsers. Parsec
provides some simple parsing functions, as well as functions to tie them
all together. It should come as no surprise that this parser library for
Haskell is built around the notion of functions.

It\'s helpful to know where Parsec fits compared to the tools used for
parsing in other languages. Parsing is sometimes divided into two
stages: lexical analysis (the domain of tools like Flex) and parsing
itself (performed by programs such as Bison). Parsec can perform both
lexical analysis and parsing.

First Steps with Parsec: Simple CSV Parsing
-------------------------------------------

::: {.WARNING}
The book uses a deprecated API of Parsec and should be updated.
:::

Let\'s jump right in by writing some code for parsing a CSV file. CSV
files are often used as a plain text representation of spreadsheets or
databases. Each line is a record, and each field in the record is
separated from the next by a comma. There are ways of dealing with
fields that contain commas, but to start with, we won\'t worry about it.

This first example is much longer than it really needs to be. We will
introduce more Parsec features in a little bit that will shrink the
parser down to only four lines!

<div>

[csv1.hs]{.label}

``` {.haskell}
import Text.ParserCombinators.Parsec

{- A CSV file contains 0 or more lines, each of which is terminated
   by the end-of-line character (eol). -}
csvFile :: GenParser Char st [[String]]
csvFile =
    do result <- many line
       eof
       return result

-- Each line contains 1 or more cells, separated by a comma
line :: GenParser Char st [String]
line =
    do result <- cells
       eol                       -- end of line
       return result

-- Build up a list of cells.  Try to parse the first cell, then figure out
-- what ends the cell.
cells :: GenParser Char st [String]
cells =
    do first <- cellContent
       next <- remainingCells
       return (first : next)

-- The cell either ends with a comma, indicating that 1 or more cells follow,
-- or it doesn't, indicating that we're at the end of the cells for this line
remainingCells :: GenParser Char st [String]
remainingCells =
    (char ',' >> cells)            -- Found comma?  More cells coming
    <|> (return [])                -- No comma?  Return [], no more cells

-- Each cell contains 0 or more characters, which must not be a comma or
-- EOL
cellContent :: GenParser Char st String
cellContent =
    many (noneOf ",\n")

-- The end of line character is \n
eol :: GenParser Char st Char
eol = char '\n'

parseCSV :: String -> Either ParseError [[String]]
parseCSV input = parse csvFile "(unknown)" input
```

</div>

Let\'s take a look at the code for this example. We didn\'t use many
shortcuts here, so remember that this will get shorter and simpler!

We\'ve built it from the top down, so our first function is `csvFile`.
The type of this function is `GenParser Char st [[String]]`. This means
that the type of the input is a sequence of characters, which is exactly
what a Haskell string is, since `String` is the same as `[Char]`. It
also means that we will return a value of type `[[String]]`: a list of a
list of strings. The `st` can be ignored for now.

Parsec programmers often omit type declarations, since we write so many
small functions. Haskell\'s type inference can figure it out. We\'ve
listed the types for the first example here so you can get a better idea
of what\'s going on. You can always use `:t` in `ghci` to inspect types
as well.

The `csvFile` uses a `do` block. As this implies, Parsec is a monadic
library: it defines its own special parsing monad, `GenParser`.

We start by running `many line`. `many` is a function that takes a
function as an argument. It tries to repeatedly parse the input using
the function passed to it. It gathers up the results from all that
repeated parsing and returns a list of them. So, here, we are storing
the results of parsing all lines in `result`. Then we look for the
end-of-file indicator, called `eof`. Finally, we return the `result`.
So, a CSV file is made up of many lines, then the end of file. We can
often read out Parsec functions in plain English just like this.

Now we must answer the question: what is a line? We define the `line`
function to do just that. Reading the function, we can see that a line
consists of cells followed by the end of line character.

So what are cells? We defined them in the `cells` function. The cells of
a line start with the content of the first cell, then continue with the
content of the remaining cells, if any. The result is simply the first
cell and the remaining cells assembled into a list.

Let\'s skip over `remainingCells` for a minute and look at
`cellContent`. A cell contains any number of characters, but each
character must not be a comma or end of line character. The `noneOf`
function matches one item, so long as it isn\'t in the list of items
that we pass. So, saying `many (noneOf ",\n")` defines a cell the way we
want it.

Back in `remainingCells`, we have the first example of a choice in
Parsec. The choice operator is `<|>`. This operator behaves like this:
it will first try the parser on the left. If it consumed no input[^1],
it will try the parser on the right.

So, in `remainingCells`, our task is to come up with all the cells after
the first. Recall that `cellContent` uses `noneOf ",\n"`. So it will not
consume the comma or end-of-line character from the input. If we see a
comma after parsing a cell, it means that at least one more cell
follows. Otherwise, we\'re done. So, our first choice in
`remainingCells` is `char ','`. This parser simply matches the passed
character in the input. If we found a comma, we want this function to
return the remaining cells on the line. At this point, the \"remaining
cells\" looks exactly like the start of the line, so we call `cells`
recursively to parse them. If we didn\'t find a comma, we return the
empty list, signifying no remaining cells on the line.

Finally, we must define what the end-of-line indicator is. We set it to
`char '\n'`, which will suit our purposes fine for now.

At the very end of the program, we define a function `parseCSV` that
takes a `String` and parses it as a CSV file. This function is just a
shortcut that calls Parsec\'s `parse` function, filling in a few
parameters. `parse` returns `Either ParseError [[String]]` for the CSV
file. If there was an error, the return value will be `Left` with the
error; otherwise, it will be `Right` with the result.

Now that we understand this code, let\'s play with it a bit and see what
it does.

``` {.screen}
ghci> :l csv1.hs
[1 of 1] Compiling Main             ( csv1.hs, interpreted )
Ok, modules loaded: Main.
ghci> parseCSV ""
Right []
```

That makes sense: parsing the empty string returns an empty list. Let\'s
try parsing a single cell.

``` {.screen}
ghci> parseCSV "hi"
Left "(unknown)" (line 1, column 3):
unexpected end of input
expecting "," or "\n"
```

Look at that. Recall how we defined that each line must end with the
end-of-line character, and we didn\'t give it. Parsec\'s error message
helpfully indicated the line number and column number of the problem,
and even told us what it was expecting! Let\'s give it an end-of-line
character and continue experimenting.

``` {.screen}
ghci> parseCSV "hi\n"
Right [["hi"]]
ghci> parseCSV "line1\nline2\nline3\n"
Right [["line1"],["line2"],["line3"]]
ghci> parseCSV "cell1,cell2,cell3\n"
Right [["cell1","cell2","cell3"]]
ghci> parseCSV "l1c1,l1c2\nl2c1,l2c2\n"
Right [["l1c1","l1c2"],["l2c1","l2c2"]]
ghci> parseCSV "Hi,\n\n,Hello\n"
Right [["Hi",""],[""],["","Hello"]]
```

You can see that `parseCSV` is doing exactly what we wanted it to do.
It\'s even handling empty cells and empty lines properly.

The `sepBy` and `endBy` Combinators
-----------------------------------

We promised you earlier that we could simplify our CSV parser
significantly by using a few Parsec helper functions. There are two that
will dramatically simplify this code.

The first tool is the `sepBy` function. This function takes two
functions as arguments: the first function parses some sort of content,
while the second function parses a separator. `sepBy` starts by trying
to parse content, then separators, and alternates back and forth until
it can\'t parse a separator. It returns a list of all the content that
it was able to parse.

The second tool is `endBy`. It\'s similar to `sepBy`, but expects the
very last item to be followed by the separator. That is, it continues
parsing until it can\'t parse any more content.

So, we can use `endBy` to parse lines, since every line must end with
the end-of-line character. We can use `sepBy` to parse cells, since the
last cell will not end with a comma. Take a look at how much simpler our
parser is now:

<div>

[csv2.hs]{.label}

``` {.haskell}
import Text.ParserCombinators.Parsec

csvFile = endBy line eol
line = sepBy cell (char ',')
cell = many (noneOf ",\n")
eol = char '\n'

parseCSV :: String -> Either ParseError [[String]]
parseCSV input = parse csvFile "(unknown)" input
```

</div>

This program behaves exactly the same as the first one. We can verify
this by using `ghci` to re-run our examples from the earlier example.
We\'ll get the same result from every one. Yet the program is much
shorter and more readable. It won\'t be long before you can translate
Parsec code like this into a file format definition in plain English. As
you read over this code, you can see that:

-   A CSV file contains 0 or more lines, each of which is terminated by
    the end-of-line character.
-   A line contains 1 or more cells, separated by a comma.
-   A cell contains 0 or more characters, which must be neither the
    comma nor the end-of-line character.
-   The end-of-line character is the newline, `\n`.

Choices and Errors
------------------

Different operating systems use different characters to mark the
end-of-line. Most Unix-like systems, plus Windows in text mode, use
simply `"\n"`. DOS and Windows systems use `"\r\n"`, and macOS
traditionally used `"\r"`. We could add in support for `"\n\r"` too,
just in case anybody uses that.

We could easily adapt our example to be able to handle all these types
of line endings in a single file. We would need to make two
modifications: adjust `eol` to recognize the different endings, and
adjust the `noneOf` pattern in `cell` to ignore `\r`.

This must be done carefully. Recall that our earlier definition of `eol`
was simply `char '\n'`. There is a parser called `string` that we can
use to match the multi-character patterns. Let\'s start by thinking of
how we would add support for `\n\r`.

Our first attempt might look like this:

<div>

[csv3.hs]{.label}

``` {.haskell}
import Text.ParserCombinators.Parsec

eol :: Parser String -- To avoid the monomorphic restriction
-- This function is not correct!
eol = string "\n" <|> string "\n\r"
```

</div>

This isn\'t quite right. Recall that the `<|>` operator always tries the
left alternative first. Looking for the single character `\n` will match
both types of line endings, so it will look to the system that the
following line begins with `\r`. Not what we want. Try it in `ghci`:

``` {.screen}
ghci> :l csv3.hs
ghci> parse eol "" "\n"
Right "\n"
ghci> parse eol "" "\n\r"
Right "\n"
```

It may seem like the parser worked for both endings, but actually
looking at it this way, we can\'t tell. If it left something un-parsed,
we don\'t know, because we\'re not trying to consume anything else from
the input. So let\'s look for the end-of-file after our end of line:

``` {.screen}
ghci> parse (eol >> eof) "" "\n\r"
Left (line 2, column 1):
unexpected '\r'
expecting end of input
ghci> parse (eol >> eof) "" "\n"
Right ()
```

As expected, we got an error from the `\n\r` ending. So the next
temptation may be to try it this way:

<div>

[csv4.hs]{.label}

``` {.haskell}
import Text.ParserCombinators.Parsec

eol :: Parser String
-- This function is not correct!
eol = string "\n\r" <|> string "\n"
```

</div>

This also isn\'t right. Recall that `<|>` only attempts the option on
the right if the option on the left consumed no input. But by the time
we are able to see if there is a `\r` after the `\n`, we\'ve already
consumed the `\n`. This time, we fail on the other case in `ghci`:

``` {.screen}
ghci> :l csv4.hs
ghci> parse (eol >> eof) "" "\n\r"
Right ()
ghci> parse (eol >> eof) "" "\n"
Left (line 1, column 1):
unexpected end of input
expecting "\n\r"
```

We\'ve stumbled upon the lookahead problem. It turns out that, when
writing parsers, it\'s often very convenient to be able to \"look
ahead\" at the data that\'s coming in. Parsec supports this, but before
showing you how to use it, let\'s see how you would have to write this
to get along without it. You\'d have to manually expand all the options
after the `\n` like this:

<div>

[csv5.hs]{.label}

``` {.haskell}
import Text.ParserCombinators.Parsec

eol :: Parser Char
eol =
    do char '\n'
       char '\r' <|> return '\n'
```

</div>

This function first looks for `\n`. If it is found, then it will look
for `\r`, consuming it if possible. Since the return type of `char '\r'`
is a `Char`, the alternative action is to simply return a `Char` without
attempting to parse anything. Parsec has a function `option` that can
also express this idiom as `option '\n' (char '\r')`. Let\'s test this
with `ghci`.

``` {.screen}
ghci> :l csv5.hs
[1 of 1] Compiling Main             ( csv5.hs, interpreted )
Ok, one module loaded.
ghci> parse (eol >> eof) "" "\n\r"
Right ()
ghci> parse (eol >> eof) "" "\n"
Right ()
```

This time, we got the right result! But we could have done it easier
with Parsec\'s lookahead support.

### Lookahead

Parsec has a function called `try` that is used to express lookaheads.
`try` takes one function, a parser. It applies that parser. If the
parser doesn\'t succeed, `try` behaves as if it hadn\'t consumed any
input at all. So, when you use `try` on the left side of `<|>`, Parsec
will try the option on the right even if the left side failed after
consuming some input. `try` only has an effect if it is on the left of a
`<|>`. Keep in mind, though, that many functions use `<|>` internally.
Here\'s a way to add expanded end-of-line support to our CSV parser
using `try`:

<div>

[csv6.hs]{.label}

``` {.haskell}
import Text.ParserCombinators.Parsec

csvFile = endBy line eol
line = sepBy cell (char ',')
cell = many (noneOf ",\n\r")

eol =   try (string "\n\r")
    <|> try (string "\r\n")
    <|> string "\n"
    <|> string "\r"

parseCSV :: String -> Either ParseError [[String]]
parseCSV input = parse csvFile "(unknown)" input
```

</div>

Here we put both of the two-character endings first, and run both tests
under `try`. Both of them occur to the left of a `<|>`, so they will do
the right thing. We could have put `string "\n"` within a `try`, but it
wouldn\'t have altered any behavior since they look at only one
character anyway. We can load this up and test the `eol` function in
`ghci`.

``` {.screen}
ghci> :l csv6.hs
[1 of 1] Compiling Main             ( csv6.hs, interpreted )
Ok, one module loaded.
ghci> parse (eol >> eof) "" "\n\r"
Right ()
ghci> parse (eol >> eof) "" "\n"
Right ()
ghci> parse (eol >> eof) "" "\r\n"
Right ()
ghci> parse (eol >> eof) "" "\r"
Right ()
```

All four endings were handled properly. You can also test the full CSV
parser with some different endings like this:

``` {.screen}
ghci> parseCSV "line1\r\nline2\nline3\n\rline4\rline5\n"
Right [["line1"],["line2"],["line3"],["line4"],["line5"]]
```

As you can see, this program even supports different line endings within
a single file.

### Error Handling

At the beginning of this chapter, you saw how Parsec could generate
error messages that list the location where the error occurred as well
as what was expected. As parsers get more complex, the list of what was
expected can become cumbersome. Parsec provides a way for you to specify
custom error messages in the event of parse failures.

Let\'s look at what happens when our current CSV parser encounters an
error:

``` {.screen}
ghci> parseCSV "line1"
Left "(unknown)" (line 1, column 6):
unexpected end of input
expecting ",", "\n\r", "\r\n", "\n" or "\r"
```

That\'s a pretty long, and technical, error message. We could make an
attempt to resolve this by using the monad `fail` function like so:

<div>

[csv6.hs]{.label}

``` {.haskell}
<|> fail "Couldn't find EOL"
```

</div>

Under `ghci`, we can see the result:

``` {.screen}
ghci> :r
[1 of 1] Compiling Main             ( csv7.hs, interpreted )
Ok, one module loaded.
ghci> parseCSV "line1"
Left "(unknown)" (line 1, column 6):
unexpected end of input
expecting ",", "\n\r", "\r\n", "\n" or "\r"
Couldn't find EOL
```

We added to the error result, but didn\'t really help clean up the
output. Parsec has an `<?>` operator that is designed for just these
situations. It is similar to `<|>` in that it first tries the parser on
its left. Instead of trying another parser in the event of a failure, it
presents an error message. Here\'s how we\'d use it:

<div>

[csv6.hs]{.label}

``` {.haskell}
-- <|> fail "Couldn't find EOL"
<?> "end of line"
```

</div>

Now, when you generate an error, you\'ll get more helpful output:

``` {.screen}
ghci> :r
[1 of 1] Compiling Main             ( csv8.hs, interpreted )
Ok, one module loaded.
ghci> parseCSV "line1"
Left "(unknown)" (line 1, column 6):
unexpected end of input
expecting "," or end of line
```

That\'s pretty helpful! The general rule of thumb is that you put a
human description of what you\'re looking for to the right of `<?>`.

Extended Example: Full CSV Parser
---------------------------------

Our earlier CSV examples have had an important flaw: they weren\'t able
to handle cells that contain a comma. CSV generating programs typically
put quotation marks around such data. But then you have another problem:
what to do if a cell contains a quotation mark and a comma. In these
cases, the embedded quotation marks are doubled up.

Here is a full CSV parser. You can use this from `ghci`, or if you
compile it to a standalone program, it will parse a CSV file on standard
input and convert it to a different format on output.

<div>

[csv7.hs]{.label}

``` {.haskell}
import Text.ParserCombinators.Parsec

csvFile = endBy line eol
line = sepBy cell (char ',')
cell = quotedCell <|> many (noneOf ",\n\r")

quotedCell =
    do char '"'
       content <- many quotedChar
       char '"' <?> "quote at end of cell"
       return content

quotedChar =
        noneOf "\""
    <|> try (string "\"\"" >> return '"')

eol =   try (string "\n\r")
    <|> try (string "\r\n")
    <|> string "\n"
    <|> string "\r"
    <?> "end of line"

parseCSV :: String -> Either ParseError [[String]]
parseCSV input = parse csvFile "(unknown)" input

main =
    do c <- getContents
       case parse csvFile "(stdin)" c of
            Left e -> do putStrLn "Error parsing input:"
                         print e
            Right r -> mapM_ print r
```

</div>

That\'s a full-featured CSV parser in just 21 lines of code, plus an
additional 10 lines for the `parseCSV` and `main` utility functions.

Let\'s look at the changes in this program from the previous versions.
First, a cell may now be either a bare cell or a \"quoted\" cell. We
give the `quotedCell` option first, because we want to follow that path
if the first character in a cell is the quote mark.

The `quotedCell` begins and ends with a quote mark, and contains zero or
more characters. These characters can\'t be copied directly, though,
because they may contain embedded, doubled-up, quote marks themselves.
So we define a custom `quotedChar` to process them.

When we\'re processing characters inside a quoted cell, we first say
`noneOf "\""`. This will match and return any single character as long
as it\'s not the quote mark. Otherwise, if it is the quote mark, we see
if we have two of them in a row. If so, we return a single quote mark to
go on our result string.

Notice that `try` in `quotedChar` on the *right* side of `<|>`. Recall
that I said that `try` only has an effect if it is on the left side of
`<|>`. This `try` does occur on the left side of a `<|>`, but on the
left of one that must be within the implementation of `many`.

This `try` is important. Let\'s say we are parsing a quoted cell, and
are getting towards the end of it. There will be another cell following.
So we will expect to see a quote to end the current cell, followed by a
comma. When we hit `quotedChar`, we will fail the `noneOf` test and
proceed to the test that looks for two quotes in a row. We\'ll also fail
that one because we\'ll have a quote, then a comma. If we hadn\'t used
`try`, we\'d crash with an error at this point, saying that it was
expecting the second quote, because the first quote was already
consumed. Since we use `try`, this is properly recognized as not a
character that\'s part of the cell, so it terminates the
`many quotedChar` expression as expected. Lookahead has once again
proven very useful, and the fact that it is so easy to add makes it a
remarkable tool in Parsec.

We can test this program with `ghci` over some quoted cells.

``` {.screen}
ghci> :l csv7.hs
[1 of 1] Compiling Main             ( csv9.hs, interpreted )
Ok, one module loaded.
ghci> parseCSV "\"This, is, one, big, cell\"\n"
Right [["This, is, one, big, cell"]]
ghci> parseCSV "\"Cell without an end\n"
Left "(unknown)" (line 2, column 1):
unexpected end of input
expecting "\"\"" or quote at end of cell
```

Let\'s run it over a real CSV file. Here\'s one generated by a
spreadsheet program:

``` {.example}
"Product","Price"
"O'Reilly Socks",10
"Shirt with ""Haskell"" text",20
"Shirt, ""O'Reilly"" version",20
"Haskell Caps",15
```

Now, we can run this under our test program and watch:

``` {.screen}
$ runhaskell csv7.hs < test.csv
["Product","Price"]
["O'Reilly Socks","10"]
["Shirt with \"Haskell\" text","20"]
["Shirt, \"O'Reilly\" version","20"]
["Haskell Caps","15"]
```

Parsec and MonadPlus
--------------------

Parsec\'s `GenParser` monad is an instance of the `MonadPlus` type class
that we introduced in [the section called \"Looking for
alternatives\"](16-programming-with-monads.org::*Looking for alternatives)
represents a parse failure, while `mplus` combines two alternative
parses into one, using `(<|>)`.

``` {.haskell}
instance MonadPlus (GenParser tok st) where
    mzero = fail "mzero"
    mplus = (<|>)
```

Parsing an URL-encoded query string
-----------------------------------

When we introduced `application/x-www-form-urlencoded` text in [the
section called \"Golfing practice: association
lists\"](16-programming-with-monads.org::*Golfing%20practice:%20association%20lists),
we mentioned that we\'d write a parser for these strings. We can quickly
and easily do this using Parsec.

Each key-value pair is separated by the `&` character.

<div>

[FormParse.hs]{.label}

``` {.haskell}
import Numeric
import Text.ParserCombinators.Parsec

p_query :: CharParser () [(String, Maybe String)]
p_query = p_pair `sepBy` char '&'
```

</div>

Notice that in the type signature, we\'re using `Maybe` to represent a
value: the HTTP specification is unclear about whether a key *must* have
an associated value, and we\'d like to be able to distinguish between
\"no value\" and \"empty value\".

<div>

[FormParse.hs]{.label}

``` {.haskell}
p_pair :: CharParser () (String, Maybe String)
p_pair = do
  name <- many1 p_char
  value <- optionMaybe (char '=' >> many p_char)
  return (name, value)
```

</div>

The `many1` function is similar to `many`: it applies its parser
repeatedly, returning a list of their results. While `many` will succeed
and return an empty list if its parser never succeeds, `many1` will fail
if its parser never succeeds, and will otherwise return a list of at
least one element.

The `optionMaybe` function modifies the behaviour of a parser. If the
parser fails, `optionMaybe` doesn\'t fail: it returns `Nothing`.
Otherwise, it wraps the parser\'s successful result with `Just`. This
gives us the ability to distinguish between \"no value\" and \"empty
value\", as we mentioned above.

Individual characters can be encoded in one of several ways.

<div>

[FormParse.hs]{.label}

``` {.haskell}
p_char :: CharParser () Char
p_char = oneOf urlBaseChars
     <|> (char '+' >> return ' ')
     <|> p_hex

urlBaseChars = ['a'..'z']++['A'..'Z']++['0'..'9']++"$-_.!*'(),"

p_hex :: CharParser () Char
p_hex = do
  char '%'
  a <- hexDigit
  b <- hexDigit
  let ((d, _):_) = readHex [a,b]
  return . toEnum $ d
```

</div>

Some characters can be represented literally. Spaces are treated
specially, using a `+` character. Other characters must be encoded as a
`%` character followed by two hexadecimal digits. The `Numeric`
module\'s `readHex` parses a hex string as a number.

``` {.screen}
ghci> parseTest p_query "foo=bar&a%21=b+c"
[("foo",Just "bar"),("a!",Just "b c")]
```

As appealing and readable as this parser is, we can profit from stepping
back and taking another look at some of our building blocks.

Supplanting regular expressions for casual parsing
--------------------------------------------------

In many popular languages, people tend to put regular expressions to
work for \"casual\" parsing. They\'re notoriously tricky for this
purpose: hard to write, difficult to debug, nearly incomprehensible
after a few months of neglect, and provide no error messages on failure.

If we can write compact Parsec parsers, we\'ll gain in readability,
expressiveness, and error reporting. Our parsers won\'t be as short as
regular expressions, but they\'ll be close enough to negate much of the
temptation of regexps.

Parsing without variables
-------------------------

A few of our parsers above use `do` notation and bind the result of an
intermediate parse to a variable, for later use. One such function is
`p_pair`.

<div>

[FormParse.hs]{.label}

``` {.haskell}
p_pair :: CharParser () (String, Maybe String)
p_pair = do
  name <- many1 p_char
  value <- optionMaybe (char '=' >> many p_char)
  return (name, value)
```

</div>

We can get rid of the need for explicit variables by using the `liftM2`
combinator from `Control.Monad`.

<div>

[FormParse.hs]{.label}

``` {.haskell}
-- Import Control.Monad at the beginning of the file

p_pair_app1 =
    liftM2 (,) (many1 p_char) (optionMaybe (char '=' >> many p_char))
```

</div>

This parser has exactly the same type and behaviour as `p_pair`, but
it\'s one line long. Instead of writing our parser in a \"procedural\"
style, we\'ve simply switched to a programming style that emphasises
that we\'re *applying* parsers and *combining* their results.

We can take this applicative style of writing a parser much further. In
most cases, the extra compactness that we will gain will *not* come at
any cost in readability, beyond the initial effort of coming to grips
with the idea.

Applicative functors for parsing
--------------------------------

The standard Haskell libraries include a module named
`Control.Applicative`, which defines a type class named `Applicative`,
which represents an *applicative functor*. Because every applicative
functor is also a functor they are represented as a hierarchy.

``` {.haskell}
class Functor f => Applicative f where
  pure :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
```

The `pure` function lifts a value into an applicative functor and `<*>`
is like `fmap` but the function to be applied is in a functor so `<*>`
takes care of applying it.

``` {.screen}
ghci> :type fmap
fmap :: Functor f => (a -> b) -> f a -> f b
ghci> :type (<*>)
(<*>) :: Applicative f => f (a -> b) -> f a -> f b
```

As usual, we think that the best way to introduce applicative functors
is by putting them to work.

Applicative parsing by example
------------------------------

We\'ll begin by rewriting our existing form parser from the bottom up,
beginning with `p_hex`, which parses a hexadecimal escape sequence.
Here\'s the code in normal `do` notation style.

<div>

[FormApp.hs]{.label}

``` {.haskell}
import Control.Monad
import Numeric
import Text.ParserCombinators.Parsec

p_hex :: CharParser () Char
p_hex = do
  char '%'
  a <- hexDigit
  b <- hexDigit
  let ((d, _):_) = readHex [a,b]
  return . toEnum $ d
```

</div>

Because Parsec includes an applicative instance it is easy to write our
applicative version.

<div>

[FormApp.hs]{.label}

``` {.haskell}
hexify :: Char -> Char -> Char
hexify a b = toEnum . fst . head .readHex $ [a, b]

a_hex :: Parser Char
a_hex = hexify <$> (char '%' *> hexDigit) <*> hexDigit
```

</div>

Although the individual parsers are mostly untouched, the combinators
that we\'re gluing them together with have changed. The familiar ones
are `(<$>)`, which we already know is a synonym for `fmap` and `(<*>)`
which is plain old `fmap` lifted to applicative functors so it applies
the parser on its left, then the parser on its right, and applies the
function that\'s the result of the left parse to the value that\'s the
result of the right.

The unfamiliar combinator is `(*>)`, which applies its first argument,
throws away its result, then applies the second and returns its result.
In other words, it\'s similar to `(>>)`.

::: {.TIP}
A handy tip about angle brackets

Before we continue, here\'s a useful aid for remembering what all the
angle brackets are for in the combinators from `Control.Applicative`: if
there\'s an angle bracket pointing to some side, the result from that
side should be used.

For example, `(*>)` returns the result on its right; `(<*>)` returns
results from both sides; and `(<*)`, which we have not yet seen, returns
the result on its left.
:::

Parsec\'s `hexDigit` parser parses a single hexadecimal digit.

``` {.screen}
ghci> :type hexDigit
hexDigit :: (Stream s m Char) => ParsecT s u m Char
```

Therefore, `char '%' *> hexDigit` has the same type, since `(*>)`
returns the result on its right.

``` {.screen}
ghci> :type char '%' *> hexDigit
char '%' *> hexDigit :: (Stream s m Char) => ParsecT s u m Char
```

The expression `hexify <$> (char '%' *> hexDigit)` is a parser that
matches a \"%\" character followed by hex digit, and whose result is a
function.

``` {.screen}
ghci> :l FormApp.hs
[1 of 1] Compiling Main             ( FormApp.hs, interpreted )
Ok, one module loaded.
ghci> :type hexify <$> (char '%' *> hexDigit)
hexify <$> (char '%' *> hexDigit)
  :: Stream s m Char =>
     ParsecT s u m (Char -> Char)
```

Next, we\'ll consider the `p_char` parser.

<div>

[FormApp.hs]{.label}

``` {.haskell}
p_char :: CharParser () Char
p_char = oneOf urlBaseChars
     <|> (char '+' >> return ' ')
     <|> p_hex

urlBaseChars = ['a'..'z']++['A'..'Z']++['0'..'9']++"$-_.!*'(),"
```

</div>

This remains almost the same in an applicative style, save for one piece
of convenient notation.

<div>

[FormApp.hs]{.label}

``` {.haskell}
a_char = oneOf urlBaseChars
     <|> (' ' <$ char '+')
     <|> a_hex
```

</div>

Here, the `(<$)` combinator uses the value on the left if the parser on
the right succeeds.

Finally, the equivalent of `p_pair_app1` is almost identical.

<div>

[FormParse.hs]{.label}

``` {.haskell}
p_pair_app1 =
    liftM2 (,) (many1 p_char) (optionMaybe (char '=' >> many p_char))
```

</div>

All we\'ve changed is the combinator we use for lifting: the `liftA`
functions act in the same ways as their `liftM` cousins.

<div>

[FormApp.hs]{.label}

``` {.haskell}
a_pair :: CharParser () (String, Maybe String)
a_pair = liftA2 (,) (many1 a_char) (optionMaybe (char '=' *> many a_char))
```

</div>

Parsing JSON data
-----------------

To give ourselves a better feel for parsing with applicative functors,
and to explore a few more corners of Parsec, we\'ll write a JSON parser
that follows the definition in RFC 4627.

At the top level, a JSON value must be either an object or an array.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
{-# LANGUAGE FlexibleContexts #-}

import Control.Monad (mzero)
import JSONClass
import Numeric
import Text.ParserCombinators.Parsec

p_text :: CharParser () JValue
p_text = spaces *> text
     <?> "JSON text"
    where text = JObject <$> p_object
             <|> JArray <$> p_array
```

</div>

These are structurally similar, with an opening character, followed by
one or more items separated by commas, followed by a closing character.
We capture this similarity by writing a small helper function.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
p_series :: Char -> CharParser () a -> Char -> CharParser () [a]
p_series left parser right =
    between (char left <* spaces) (char right) $
            (parser <* spaces) `sepBy` (char ',' <* spaces)
```

</div>

Here, we finally have a use for the `(<*)` combinator that we introduced
earlier. We use it to skip over any white space that might follow
certain tokens. With this `p_series` function, parsing an array is
simple.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
p_array :: CharParser () (JAry JValue)
p_array = JAry <$> p_series '[' p_value ']'
```

</div>

Dealing with a JSON object is hardly more complicated, requiring just a
little additional effort to produce a name/value pair for each of the
object\'s fields.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
p_object :: CharParser () (JObj JValue)
p_object = JObj <$> p_series '{' p_field '}'
    where p_field = (,) <$> (p_string <* char ':' <* spaces) <*> p_value
```

</div>

Parsing an individual value is a matter of calling an existing parser,
then wrapping its result with the appropriate `JValue` constructor.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
p_value :: CharParser () JValue
p_value = value <* spaces
  where value = JString <$> p_string
            <|> JNumber <$> p_number
            <|> JObject <$> p_object
            <|> JArray <$> p_array
            <|> JBool <$> p_bool
            <|> JNull <$ string "null"
            <?> "JSON value"

p_bool :: CharParser () Bool
p_bool = True <$ string "true"
     <|> False <$ string "false"
```

</div>

The `choice` combinator allows us to represent this kind of
ladder-of-alternatives as a list. It returns the result of the first
parser to succeed.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
p_value_choice = value <* spaces
  where value = choice [ JString <$> p_string
                       , JNumber <$> p_number
                       , JObject <$> p_object
                       , JArray <$> p_array
                       , JBool <$> p_bool
                       , JNull <$ string "null"
                       ]
                <?> "JSON value"
```

</div>

This leads us to the two most interesting parsers, for numbers and
strings. We\'ll deal with numbers first, since they\'re simpler.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
p_number :: CharParser () Double
p_number = do s <- getInput
              case readSigned readFloat s of
                [(n, s')] -> n <$ setInput s'
                _         -> empty
```

</div>

The only piece of functionality that applicative functors are missing,
compared to monads, is the ability to bind a value to a variable, which
we need here in order to be able to validate the value we\'re trying to
decode.

Our trick here is to take advantage of Haskell\'s standard number
parsing library functions, which are defined in the `Numeric` module.
The `readFloat` function reads an unsigned floating point number, and
`readSigned` takes a parser for an unsigned number and turns it into a
parser for possibly signed numbers.

Since these functions know nothing about Parsec, we have to work with
them specially. Parsec\'s `getInput` function gives us direct access to
Parsec\'s unconsumed input stream. If `readSigned readFloat` succeeds,
it returns both the parsed number and the rest of the unparsed input. We
then use `setInput` to give this back to Parsec as its new unconsumed
input stream.

Parsing a string isn\'t difficult, merely detailed.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
p_string :: CharParser () String
p_string = between (char '\"') (char '\"') (many jchar)
    where jchar = char '\\' *> (p_escape <|> p_unicode)
              <|> satisfy (`notElem` "\"\\")
```

</div>

We can parse and decode an escape sequence with the help of the `choice`
combinator that we just met.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
p_escape = choice (zipWith decode "bnfrt\\\"/" "\b\n\f\r\t\\\"/")
    where decode c r = r <$ char c
```

</div>

Finally, JSON lets us encode a Unicode character in a string as \"`\u`\"
followed by four hexadecimal digits.

<div>

[JSONParsec.hs]{.label}

``` {.haskell}
p_unicode :: CharParser () Char
p_unicode = char 'u' *> (decode <$> count 4 hexDigit)
    where decode x = toEnum code
              where ((code,_):_) = readHex x
```

</div>

Parsing a HTTP request
----------------------

As another example of applicative parsing, we will develop a basic
parser for HTTP requests.

<div>

[HttpRequestParser.hs]{.label}

``` {.haskell}
module HttpRequestParser
    (
      HttpRequest(..)
    , Method(..)
    , p_request
    ) where

import Numeric (readHex)
import Control.Monad (liftM4)
import Text.ParserCombinators.Parsec
import System.IO (Handle)
```

</div>

An HTTP request consists of a method, an identifier, a series of
headers, and an optional body. For simplicity, we\'ll focus on just two
of the six method types specified by the HTTP 1.1 standard. A `POST`
method has a body; a `GET` has none.

<div>

[HttpRequestParser.hs]{.label}

``` {.haskell}
data Method = Get | Post
          deriving (Eq, Ord, Show)

data HttpRequest = HttpRequest {
      reqMethod :: Method
    , reqURL :: String
    , reqHeaders :: [(String, String)]
    , reqBody :: Maybe String
    } deriving (Eq, Show)
```

</div>

Because we\'re writing in an applicative style, our parser can be both
brief and readable. Readable, that is, if you\'re becoming used to the
applicative parsing notation.

<div>

[HttpRequestParser.hs]{.label}

``` {.haskell}
p_request :: CharParser () HttpRequest
p_request = q "GET" Get (pure Nothing)
        <|> q "POST" Post (Just <$> many anyChar)
  where q name ctor body = liftM4 HttpRequest req url p_headers body
            where req = ctor <$ string name <* char ' '
        url = optional (char '/') *>
              manyTill notEOL (try $ string " HTTP/1." <* oneOf "01")
              <* crlf
```

</div>

Briefly, the `q` helper function accepts a method name, the type
constructor to apply to it, and a parser for a request\'s optional body.
The `url` helper does not attempt to validate a URL, because the HTTP
specification does not specify what characters an URL contain. The
function just consumes input until either the line ends or it reaches a
HTTP version identifier.

### Backtracking and its discontents

The `try` combinator has to hold onto input in case it needs to restore
it, so that an alternative parser can be used. This practice is referred
to as *backtracking*. Because `try` must save input, it is expensive to
use. Sprinkling a parser with unnecessary uses of `try` is a very
effective way to slow it down, sometimes to the point of unacceptable
performance.

The standard way to avoid the need for backtracking is to tidy up a
parser so that we can decide whether it will succeed or fail using only
a single token of input. In this case, the two parsers consume the same
initial tokens, so we turn them into a single parser.

``` {.screen}
ghci> :set -XFlexibleContexts
ghci> :m Text.ParserCombinators.Parsec
ghci> parser = (++) <$> string "HT" <*> (string "TP" <|> string "ML")
ghci> parseTest parser "HTTP"
"HTTP"
ghci> parseTest parser "HTML"
"HTML"
```

Even better, Parsec gives us an improved error message if we feed it
non-matching input.

``` {.screen}
ghci> parseTest parser "HTXY"
parse error at (line 1, column 3):
unexpected "X"
expecting "TP" or "ML"
```

### Parsing headers

Following the first line of a HTTP request is a series of zero or more
headers. A header begins with a field name, followed by a colon,
followed by the content. If the lines that follow begin with spaces,
they are treated as *continuations* of the current content.

<div>

[HttpRequestParser.hs]{.label}

``` {.haskell}
p_headers :: CharParser st [(String, String)]
p_headers = header `manyTill` crlf
  where header = liftA2 (,) fieldName (char ':' *> spaces *> contents)
        contents = liftA2 (++) (many1 notEOL <* crlf)
                               (continuation <|> pure [])
        continuation = liftA2 (:) (' ' <$ many1 (oneOf " \t")) contents
        fieldName = (:) <$> letter <*> many fieldChar
        fieldChar = letter <|> digit <|> oneOf "-_"

crlf :: CharParser st ()
crlf = (() <$ string "\r\n") <|> (() <$ newline)

notEOL :: CharParser st Char
notEOL = noneOf "\r\n"
```

</div>

### Exercises

Our HTTP request parser is too simple to be useful in real deployments.
It is missing vital functionality, and is not resistant to even the most
basic denial of service attacks.

1.  Make the parser honour the `Content-Length` field properly, if it is
    present.

2.  A popular denial of service attack against naive web servers is
    simply to send unreasonably long headers. A single header might
    contain tens or hundreds of megabytes of garbage text, causing a
    server to run out of memory.

    Restructure the header parser so that it will fail if any line is
    longer than 4096 characters. It must fail immediately when this
    occurs; it cannot wait until the end of a line eventually shows up.

3.  Add the ability to honour the `Transfer-Encoding`: chunked header if
    it is present. See [section 3.6.1 of RFC
    2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.6.1)
    for details.

4.  Another popular attack is to open a connection and either leave it
    idle or send data extremely slowly. Write a wrapper in the `IO`
    monad that will invoke the parser. Use the `System.Timeout` module
    to close the connection if the parser has not completed within 30
    seconds.

Footnotes
---------

[^1]: For information on dealing with choices that may consume some
    input before failing, see [the section called
    \"Lookahead\"](14-using-parsec.org::*Lookahead)
