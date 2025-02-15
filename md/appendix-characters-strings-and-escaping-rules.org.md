Appendix. Characters, strings, and escaping rules
=================================================

This appendix covers the escaping rules used to represent non-ASCII
characters in Haskell character and string literals. Haskell\'s escaping
rules follow the pattern established by the C programming language, but
expand considerably upon them.

Writing character and string literals
-------------------------------------

A single character is surrounded by ASCII single quotes, `'`, and has
type `Char`.

``` {.screen}
ghci> 'c'
'c'
ghci> :type 'c'
'c' :: Char
```

A string literal is surrounded by double quotes, `"`, and has type
`[Char]` (more often written as `String`).

``` {.screen}
ghci> "a string literal"
"a string literal"
ghci> :type "a string literal"
"a string literal" :: [Char]
```

The double-quoted form of a string literal is just syntactic sugar for
list notation.

``` {.screen}
ghci> ['a', ' ', 's', 't', 'r', 'i', 'n', 'g'] == "a string"
True
```

International language support
------------------------------

Haskell uses Unicode internally for its `Char` data type. Since `String`
is just an alias for `[Char]`, a list of \~Char\~s, Unicode is also used
to represent strings.

Different Haskell implementations place limitations on the character
sets they can accept in source files. GHC allows source files to be
written in the UTF-8 encoding of Unicode, so in a source file, you can
use UTF-8 literals inside a character or string constant. Do be aware
that if you use UTF-8, other Haskell implementations may not be able to
parse your source files.

When you run the `ghci` interpreter interactively, it may not be able to
deal with international characters in character or string literals that
you enter at the keyboard.

::: {.NOTE}
Note

Although Haskell represents characters and strings internally using
Unicode, there is no standardised way to do I/O on files that contain
Unicode data. Haskell\'s standard text I/O functions treat text as a
sequence of 8-bit characters, and do not perform any character set
conversion.

There exist third-party libraries that will convert between the many
different encodings used in files and Haskell\'s internal Unicode
representation.
:::

Escaping text
-------------

Some characters must be escaped to be represented inside a character or
string literal. For example, a double quote character inside a string
literal must be escaped, or else it will be treated as the end of the
string.

### Single-character escape codes

Haskell uses essentially the same single-character escapes as the C
language and many other popular languages.

  Escape   Unicode   Character
  -------- --------- ---------------------
  `\0`     U+0000    null character
  `\a`     U+0007    alert
  `\b`     U+0008    backspace
  `\f`     U+000C    form feed
  `\n`     U+000A    newline (line feed)
  `\r`     U+000D    carriage return
  `\t`     U+0009    horizontal tab
  `\v`     U+000B    vertical tab
  `\"`     U+0022    double quote
  `\&`     *n/a*     empty string
  `\'`     U+0027    single quote
  `\\`     U+005C    backslash

  : Single-character escape codes

### Multiline string literals

To write a string literal that spans multiple lines, terminate one line
with a backslash, and resume the string with another backslash. An
arbitrary amount of whitespace (of any kind) can fill the gap between
the two backslashes.

``` {.haskell}
"this is a \
\long string,\
\ spanning multiple lines"
```

### ASCII control codes

Haskell recognises the escaped use of the standard two- and three-letter
abbreviations of ASCII control codes.

  Escape   Unicode   Meaning
  -------- --------- ---------------------------
  `\NUL`   U+0000    null character
  `\SOH`   U+0001    start of heading
  `\STX`   U+0002    start of text
  `\ETX`   U+0003    end of text
  `\EOT`   U+0004    end of transmission
  `\ENQ`   U+0005    enquiry
  `\ACK`   U+0006    acknowledge
  `\BEL`   U+0007    bell
  `\BS`    U+0008    backspace
  `\HT`    U+0009    horizontal tab
  `\LF`    U+000A    line feed (newline)
  `\VT`    U+000B    vertical tab
  `\FF`    U+000C    form feed
  `\CR`    U+000D    carriage return
  `\SO`    U+000E    shift out
  `\SI`    U+000F    shift in
  `\DLE`   U+0010    data link escape
  `\DC1`   U+0011    device control 1
  `\DC2`   U+0012    device control 2
  `\DC3`   U+0013    device control 3
  `\DC4`   U+0014    device control 4
  `\NAK`   U+0015    negative acknowledge
  `\SYN`   U+0016    synchronous idle
  `\ETB`   U+0017    end of transmission block
  `\CAN`   U+0018    cancel
  `\EM`    U+0019    end of medium
  `\SUB`   U+001A    substitute
  `\ESC`   U+001B    escape
  `\FS`    U+001C    file separator
  `\GS`    U+001D    group separator
  `\RS`    U+001E    record separator
  `\US`    U+001F    unit separator
  `\SP`    U+0020    space
  `\DEL`   U+007F    delete

  :  ASCII control code abbreviations

### Control-with-character escapes

Haskell recognises an alternate notation for control characters, which
represents the archaic effect of pressing the `control` key on a
keyboard and chording it with another key. These sequences begin with
the characters `\^`, followed by a symbol or uppercase letter.

  Escape                Unicode                 Meaning
  --------------------- ----------------------- ------------------
  `\^@`                 U+0000                  null character
  `\^A` through `\^Z`   U+0001 through U+001A   control codes
  `\^[`                 U+001B                  escape
  `\^\`                 U+001C                  file separator
  `\^]`                 U+001D                  group separator
  `\^^`                 U+001E                  record separator
  `\^_`                 U+001F                  unit separator

  : Control-with-character escapes

### Numeric escapes

Haskell allows Unicode characters to be written using numeric escapes. A
decimal character begins with a digit, e.g. `\1234`. A hexadecimal
character begins with an `x`, e.g. `\xbeef`. An octal character begins
with an `o`, e.g. `\o1234`.

The maximum value of a numeric literal is `\1114111`, which may also be
written `\x10ffff` or `\o4177777`.

### The zero-width escape sequence

String literals can contain a zero-width escape sequence, written `\&`.
This is not a real character, as it represents the empty string.

``` {.screen}
ghci> "\&"
""
ghci> "foo\&bar"
"foobar"
```

The purpose of this escape sequence is to make it possible to write a
numeric escape followed immediately by a regular ASCII digit.

``` {.screen}
ghci> "\130\&11"
"\130\&11"
```

Because the empty escape sequence represents an empty string, it is not
legal in a character literal.
