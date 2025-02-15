---
margin-left: 30px
---

Introduction: Why Haskell?
==========================

Have we got a deal for you!
---------------------------

Haskell is a deep language, and we think that learning it is a hugely
rewarding experience. We will focus on three elements as we explain why.
The first is *novelty*: we invite you to think about programming from a
different and valuable perspective. The second is *power*: we\'ll show
you how to create software that is short, fast, and safe. Lastly, we
offer you a lot of *fun*: the pleasure of applying beautiful programming
techniques to solve real problems.

### Novelty

Haskell is most likely quite different from any language you\'ve ever
used before. Compared to the usual set of concepts in a programmer\'s
mental toolbox, functional programming offers us a profoundly different
way to think about software.

In Haskell, we de-emphasise code that modifies data. Instead, we focus
on functions that take immutable values as input and produce new values
as output. Given the same inputs, these functions always return the same
results. This is a core idea behind functional programming.

Along with not modifying data, our Haskell functions usually don\'t talk
to the external world; we call these functions *pure*. We make a strong
distinction between pure code and the parts of our programs that read or
write files, communicate over network connections, or make robot arms
move. This makes it easier to organize, reason about, and test our
programs.

We abandon some ideas that might seem fundamental, such as having a
`for` loop built into the language. We have other, more flexible, ways
to perform repetitive tasks.

Even the way in which we evaluate expressions is different in Haskell.
We defer every computation until its result is actually needed: Haskell
is a *lazy* language. Laziness is not merely a matter of moving work
around: it profoundly affects how we write programs.

### Power

Throughout this book, we will show you how Haskell\'s alternatives to
the features of traditional languages are powerful, flexible, and lead
to reliable code. Haskell is positively crammed full of cutting edge
ideas about how to create great software.

Since pure code has no dealings with the outside world, and the data it
works with is never modified, the kinds of nasty surprise in which one
piece of code invisibly corrupts data used by another are very rare.
Whatever context we use a pure function in, it will behave consistently.

Pure code is easier to test than code that deals with the outside world.
When a function only responds to its visible inputs, we can easily state
properties of its behavior that should always be true. We can
automatically test that those properties hold for a huge body of random
inputs, and when our tests pass, we move on. We still use traditional
techniques to test code that must interact with files, networks, or
exotic hardware. Since there is much less of this impure code than we
would find in a traditional language, we gain much more assurance that
our software is solid.

Lazy evaluation has some spooky effects. Let\'s say we want to find the
*k* least-valued elements of an unsorted list. In a traditional
language, the obvious approach would be to sort the list and take the
first *k* elements, but this is expensive. For efficiency, we would
instead write a special function that takes these values in one pass,
and it would have to perform some moderately complex book-keeping. In
Haskell, the sort-then-take approach actually performs well: laziness
ensures that the list will only be sorted enough to find the *k* minimal
elements.

Better yet, our Haskell code that operates so efficiently is tiny, and
uses standard library functions.

<div>

[KMinima.hs]{.label}

``` {.haskell}
-- lines beginning with "--" are comments.

minima k xs = take k (sort xs)
```

</div>

It can take a while to develop an intuitive feel for when lazy
evaluation is important, but when we exploit it, the resulting code is
often clean, brief, and efficient.

As the above example shows, an important aspect of Haskell\'s power lies
in the compactness of the code we write. Compared to working in popular
traditional languages, when we develop in Haskell we often write much
less code, in substantially less time, and with fewer bugs.

### Enjoyment

We believe that it is easy to pick up the basics of Haskell programming,
and that you will be able to successfully write small programs within a
matter of hours or days.

Since effective programming in Haskell differs greatly from other
languages, you should expect that mastering both the language itself and
functional programming techniques will require plenty of thought and
practice.

Harking back to our own days of getting started with Haskell, the good
news is that the fun begins early: it\'s simply an entertaining
challenge to dig into a new language, in which so many commonplace ideas
are different or missing, and to figure out how to write simple
programs.

For us, the initial pleasure lasted as our experience grew and our
understanding deepened. In other languages, it\'s difficult to see any
connection between science and the nuts-and-bolts of programming. In
Haskell, we have imported some ideas from abstract mathematics and put
them to work. Even better, we find that not only are these ideas easy to
pick up, they have a practical payoff in helping us to write more
compact, reusable code.

Furthermore, we won\'t be putting any \"brick walls\" in your way: there
are no especially difficult or gruesome techniques in this book that you
must master in order to be able to program effectively.

That being said, Haskell is a rigorous language: it will make you
perform more of your thinking up front. It can take a little while to
adjust to debugging much of your code before you ever run it, in
response to the compiler telling you that something about your program
does not make sense. Even with years of experience, we remain astonished
and pleased by how often our Haskell programs simply work on the first
try, once we fix those compilation errors.

What to expect from this book
-----------------------------

We started this project because a growing number of people are using
Haskell to solve everyday problems. Because Haskell has its roots in
academia, few of the Haskell books that currently exist focus on the
problems and techniques of everyday programming that we\'re interested
in.

With this book, we want to show you how to use functional programming
and Haskell to solve realistic problems. This is a hands-on book: every
chapter contains dozens of code samples, and many contain complete
applications. Here are a few examples of the libraries, techniques and
tools that we\'ll show you how to develop.

-   Create an application that downloads podcast episodes from the
    Internet, and stores its history in an SQL database.
-   Test your code in an intuitive and powerful way. Describe properties
    that ought to be true, then let the QuickCheck library generate test
    cases automatically.
-   Take a grainy phone camera snapshot of a barcode, and turn it into
    an identifier that you can use to query a library or bookseller\'s
    web site.
-   Write code that thrives on the web. Exchange data with servers and
    clients written in other languages using JSON notation. Develop a
    concurrent link checker.

### A little bit about you

What will you need to know before reading this book? We expect that you
already know how to program, but if you\'ve never used a functional
language, that\'s fine.

No matter what your level of experience is, we have tried to anticipate
your needs: we go out of our way to explain new and potentially tricky
ideas in depth, usually with examples and images to drive our points
home.

As a new Haskell programmer, you\'ll inevitably start out writing quite
a bit of code by hand for which you could have used a library function
or programming technique, had you just known of its existence. We\'ve
packed this book with information to help you to come up to speed as
quickly as possible.

Of course, there will always be a few bumps along the road. If you start
out anticipating an occasional surprise or difficulty along with the fun
stuff, you will have the best experience. Any rough patches you might
hit won\'t last long.

As you become a more seasoned Haskell programmer, the way that you write
code will change. Indeed, over the course of this book, the way that we
present code will evolve, as we move from the basics of the language to
increasingly powerful and productive features and techniques.

What to expect from Haskell
---------------------------

Haskell is a general purpose programming language. It was designed
without any application niche in mind. Although it takes a strong stand
on how programs should be written, it does not favour one problem domain
over others.

While at its core, the language encourages a pure, lazy style of
functional programming, this is the *default*, not the only option.
Haskell also supports the more traditional models of procedural code and
strict evaluation. Additionally, although the focus of the language is
squarely on writing statically typed programs, it is possible (though
rarely seen) to write Haskell code in a dynamically typed manner.

### Compared to traditional static languages

Languages that use simple static type systems have been the mainstay of
the programming world for decades. Haskell is statically typed, but its
notion of what types are for, and what we can do with them, is much more
flexible and powerful than traditional languages. Types make a major
contribution to the brevity, clarity, and efficiency of Haskell
programs.

Although powerful, Haskell\'s type system is often also unobtrusive. If
we omit explicit type information, a Haskell compiler will automatically
infer the type of an expression or function. Compared to traditional
static languages, to which we must spoon-feed large amounts of type
information, the combination of power and inference in Haskell\'s type
system significantly reduces the clutter and redundancy of our code.

Several of Haskell\'s other features combine to further increase the
amount of work we can fit into a screenful of text. This brings
improvements in development time and agility: we can create reliable
code quickly, and easily refactor it in response to changing
requirements.

Sometimes, Haskell programs may run more slowly than similar programs
written in C or C++. For most of the code we write, Haskell\'s large
advantages in productivity and reliability outweigh any small
performance disadvantage.

Multicore processors are now ubiquitous, but they remain notoriously
difficult to program using traditional techniques. Haskell provides
unique technologies to make multicore programming more tractable. It
supports parallel programming, software transactional memory for
reliable concurrency, and scales to hundreds of thousands of concurrent
threads.

### Compared to modern dynamic languages

Over the past decade, dynamically typed, interpreted languages have
become increasingly popular. They offer substantial benefits in
developer productivity. Although this often comes at the cost of a huge
performance hit, for many programming tasks productivity trumps
performance, or performance isn\'t a significant factor in any case.

Brevity is one area in which Haskell and dynamically typed languages
perform similarly: in each case, we write much less code to solve a
problem than in a traditional language. Programs are often around the
same size in dynamically typed languages and Haskell.

When we consider runtime performance, Haskell almost always has a huge
advantage. Code compiled by the Glasgow Haskell Compiler (GHC) is
typically between 20 and 60 times faster than code run through a dynamic
language\'s interpreter. GHC also provides an interpreter, so you can
run scripts without compiling them.

Another big difference between dynamically typed languages and Haskell
lies in their philosophies around types. A major reason for the
popularity of dynamically typed languages is that only rarely do we need
to explicitly mention types. Through automatic type inference, Haskell
offers the same advantage.

Beyond this surface similarity, the differences run deep. In a
dynamically typed language, we can create constructs that are difficult
to express in a statically typed language. However, the same is true in
reverse: with a type system as powerful as Haskell\'s, we can structure
a program in a way that would be unmanageable or infeasible in a
dynamically typed language.

It\'s important to recognise that each of these approaches involves
tradeoffs. Very briefly put, the Haskell perspective emphasises safety,
while the dynamically typed outlook favours flexibility. If someone had
already discovered one way of thinking about types that was always best,
we imagine that everyone would know about it by now.

Of course, we have our own opinions about which tradeoffs are more
beneficial. Two of us have years of experience programming in
dynamically typed languages. We love working with them; we still use
them every day; but usually, we prefer Haskell.

### Haskell in industry and open source

Here are just a few examples of large software systems that have been
created in Haskell. Some of these are open source, while others are
proprietary products.

-   ASIC and FPGA design software (Lava, products from Bluespec Inc.)
-   Music composition software (Haskore)
-   Compilers and compiler-related tools (most notably GHC)
-   Distributed revision control (Darcs)
-   Web middleware (HAppS, products from Galois Inc.)

is a sample of some of the companies using Haskell in late 2008, taken
from the [Haskell
wiki](http://www.haskell.org/haskellwiki/Haskell_in_industry).

-   ABN AMRO is an international bank. It uses Haskell in investment
    banking, to measure the counterparty risk on portfolios of financial
    derivatives.
-   Anygma is a startup company. It develops multimedia content creation
    tools using Haskell.
-   Amgen is a biotech company. It creates mathematical models and other
    complex applications in Haskell.
-   Bluespec is an ASIC and FPGA design software vendor. Its products
    are developed in Haskell, and the chip design languages that its
    products provide are influenced by Haskell.
-   Eaton uses Haskell for the design and verification of hydraulic
    hybrid vehicle systems.

### Compilation, debugging, and performance analysis

For practical work, almost as important as a language itself is the
ecosystem of libraries and tools around it. Haskell has a strong showing
in this area.

The most widely used compiler, GHC, has been actively developed for over
15 years, and provides a mature and stable set of features.

-   Compiles to efficient native code on all major modern operating
    systems and CPU architectures
-   Easy deployment of compiled binaries, unencumbered by licensing
    restrictions
-   Code coverage analysis
-   Detailed profiling of performance and memory usage
-   Thorough documentation
-   Massively scalable support for concurrent and multicore programming
-   Interactive interpreter and debugger

### Bundled and third party libraries

The GHC compiler ships with a collection of useful libraries. Here are a
few of the common programming needs that these libraries address.

-   File I/O, and filesystem traversal and manipulation
-   Network client and server programming
-   Regular expressions and parsing
-   Concurrent programming
-   Automated testing
-   Sound and graphics

The Hackage package database is the Haskell community\'s collection of
open source libraries and applications. Most libraries published on
Hackage are licensed under liberal terms that permit both commercial and
open source use. Some of the areas covered by open source libraries
include the following.

-   Interfaces to all major open source and commercial databases
-   XML, HTML, and XQuery processing
-   Network and web client and server development
-   Desktop GUIs, including cross-platform toolkits
-   Support for Unicode and other text encodings

A brief sketch of Haskell\'s history
------------------------------------

The development of Haskell is rooted in mathematics and computer science
research.

### Prehistory

A few decades before modern computers were invented, the mathematician
Alonzo Church developed a language called the lambda calculus. He
intended it as a tool for investigating the foundations of mathematics.
The first person to realize the practical connection between programming
and the lambda calculus was John McCarthy, who created Lisp in 1958.

During the 1960s, computer scientists began to recognise and study the
importance of the lambda calculus. Peter Landin and Christopher Strachey
developed ideas about the foundations of programming languages: how to
reason about what they do (operational semantics) and how to understand
what they mean (denotational semantics).

In the early 1970s, Robin Milner created a more rigorous functional
programming language named ML. While ML was developed to help with
automated proofs of mathematical theorems, it gained a following for
more general computing tasks.

The 1970s saw the emergence of lazy evaluation as a novel strategy.
David Turner developed SASL and KRC, while Rod Burstall and John
Darlington developed NPL and Hope. NPL, KRC and ML influenced the
development of several more languages in the 1980s, including Lazy ML,
Clean, and Miranda.

### Early antiquity

By the late 1980s, the efforts of researchers working on lazy functional
languages were scattered across more than a dozen languages. Concerned
by this diffusion of effort, a number of researchers decided to form a
committee to design a common language. After three years of work, the
committee published the Haskell 1.0 specification in 1990. It named the
language after Haskell Curry, an influential logician.

Many people are rightfully suspicious of \"design by committee\", but
the work of the Haskell committee is a beautiful example of the best
work a committee can do. They produced an elegant, considered language
design, and succeeded in unifying the fractured efforts of their
research community. Of the thicket of lazy functional languages that
existed in 1990, only Haskell is still actively used.

Since its publication in 1990, the Haskell language standard has seen
several revisions, most recently in 2010. A number of Haskell
implementations have been written, and several are still actively
developed.

During the 1990s, Haskell served two main purposes. On one side, it gave
language researchers a stable language in which to experiment with
making lazy functional programs run efficiently. Other researchers
explored how to construct programs using lazy functional techniques.
Still others used it as a teaching language.

### The modern era

While these basic explorations of the 1990s proceeded, Haskell remained
firmly an academic affair. The informal slogan of those inside the
community was to \"avoid success at all costs\". Few outsiders had heard
of the language at all. Indeed, functional programming as a field was
quite obscure.

During this time, the mainstream programming world experimented with
relatively small tweaks: from programming in C, to C++, to Java.
Meanwhile, on the fringes, programmers were beginning to tinker with
new, more dynamic languages. Guido van Rossum designed Python; Larry
Wall created Perl; and Yukihiro Matsumoto developed Ruby.

As these newer languages began to seep into wider use, they spread some
crucial ideas. The first was that programmers are not merely capable of
working in expressive languages; in fact, they flourish. The second was
in part a byproduct of the rapid growth in raw computing power of that
era: it\'s often smart to sacrifice some execution performance in
exchange for a big increase in programmer productivity. Finally, several
of these languages borrowed from functional programming.

Over the past half a decade, Haskell has successfully escaped from
academia, buoyed in part by the visibility of Python, Ruby, and even
Javascript. The language now has a vibrant and fast-growing culture of
open source and commercial users, and researchers continue to use it to
push the boundaries of performance and expressiveness.

Helpful resources
-----------------

As you work with Haskell, you\'re sure to have questions and want more
information about things. Here are some Internet resources where you can
look up information and interact with other Haskell programmers.

### Reference material

-   [The Haskell Hierarchical Libraries
    reference](http://www.haskell.org/ghc/docs/latest/html/libraries/index.html)
    provides the documentation for the standard library that comes with
    your compiler. This is one of the most valuable online assets for
    Haskell programmers.
-   For questions about language syntax and features, the [Haskell 2010
    Report](http://haskell.org/onlinereport/haskell2010/) describes the
    Haskell 2010 language standard.
-   Various extensions to the language have become commonplace since the
    Haskell 2010 Report was released. The [GHC Users\'s
    Guide](http://www.haskell.org/ghc/docs/latest/html/users_guide/index.html)
    contains detailed documentation on the extensions supported by GHC,
    as well as some GHC-specific features.
-   [Hoogle](http://haskell.org/hoogle/) and
    [Hayoo](http://holumbus.fh-wedel.de/hayoo/hayoo.html) are Haskell
    API search engines. They can search for functions by name or by
    type.

### Applications and libraries

If you\'re looking for a Haskell library to use for a particular task,
or an application written in Haskell, check out the following resources.

-   The Haskell community maintains a central repository of open source
    Haskell libraries and applications. It\'s called
    [Hackage](http://hackage.haskell.org/), and it lets you search for
    software to download, or browse its collection by category.
-   The [Haskell
    Wiki](http://haskell.org/haskellwiki/Applications_and_libraries)
    contains a section dedicated to information about particular Haskell
    libraries.

### The Haskell community

There are a number of ways you can get in touch with other Haskell
programmers, to ask questions, learn what other people are talking
about, and simply do some social networking with your peers.

-   The first stop on your search for community resources should be the
    [Haskell web site](http://www.haskell.org/). This page contains the
    most current links to various communities and information, as well
    as a huge and actively maintained wiki.
-   Haskellers use a number of [mailing
    lists](http://haskell.org/haskellwiki/Mailing_lists) for topical
    discussions. Of these, the most generally interesting is named
    haskell-cafe. It has a relaxed, friendly atmosphere, where
    professionals and academics rub shoulders with casual hackers and
    beginners.
-   For real-time chat, the [Haskell IRC
    channel](http://haskell.org/haskellwiki/IRC_channel), named
    \#haskell, is large and lively. Like haskell-cafe the atmosphere
    stays friendly and helpful in spite of the huge number of concurrent
    users.
-   There are many local user groups, meetups, academic workshops, and
    the like; here is [a list of the known user groups and
    workshops](http://haskell.org/haskellwiki/User_groups).
-   The [Haskell Communities and Activities
    Report](https://wiki.haskell.org/Haskell_Communities_and_Activities_Report)
    collects information about people that use Haskell, and what they
    are doing with it. It has been running for years, so it provides a
    good way to peer into Haskell\'s past.

Acknowledgments
---------------

This book would not exist without the Haskell community: an anarchic,
hopeful cabal of artists, theoreticians and engineers, who for twenty
years have worked to create a better, bug-free programming world. The
people of the Haskell community are unique in their combination of
friendliness and intellectual depth.

We wish to thank our editor, Mike Loukides, and the production team at
O\'Reilly for all of their advice and assistance.

### Bryan

I had a great deal of fun working with John and Don. Their independence,
good nature, and formidable talent made the writing process remarkably
smooth.

Simon Peyton Jones took a chance on a college student who emailed him
out of the blue in early 1994. Interning for him over that summer
remains a highlight of my professional life. With his generosity,
boundless energy, and drive to collaborate, he inspires the whole
Haskell community.

My children, Cian and Ruairi, always stood ready to help me to unwind
with wonderful, madcap little-boy games.

Finally, of course, I owe a great debt to my wife, Shannon, for her
love, wisdom, and support during the long gestation of this book.

### John

I am so glad to be able to work with Bryan and Don on this project. The
depth of their Haskell knowledge and experience is amazing. I enjoyed
finally being able to have the three of us sit down in the same room --
over a year after we started writing.

My 2-year-old Jacob, who decided that it would be fun to use a keyboard
too, and is always eager to have me take a break from the computer and
help him make some fun typing noises on a 50-year-old Underwood
typewriter.

Most importantly, I wouldn\'t have ever been involved in this project
without the love, support, and encouragement from my wife, Terah.

### Don

Before all else, I\'d like to thank my amazing co-conspirators, John and
Bryan, for encouragment, advice and motivation.

My colleagues at Galois, Inc., who daily wield Haskell in the real
world, provided regular feedback and war stories, and helped ensured a
steady supply of espresso.

My PhD supervisor, Manuel Chakravarty, and the PLS research group, who
provided encouragement, vision and energy, and showed me that a
rigorous, foundational approach to programming can make the impossible
happen.

And, finally, thanks to Suzie, for her insight, patience and love.

### Thank you to our reviewers

We developed this book in the open, posting drafts of chapters to our
web site as we completed them. Readers then submitted feedback using a
web application that we developed. By the time we finished writing the
book, about 800 people had submitted over 7,500 comments, an astounding
figure.

We deeply appreciate the time that so many people volunteered to help us
to improve our book. Their encouragement and enthusiasm over the 15
months we spent writing made the process a pleasure.

The breadth and depth of the comments we received have profoundly
improved the quality of this book. Nevertheless, all errors and
omissions are, of course, ours.

The following people each contributed over 1% of the total number of
review comments that we received. We would like to thank them for their
care in providing us with so much detailed feedback.

Alex Stangl, Andrew Bromage, Brent Yorgey, Bruce Turner, Calvin Smith,
David Teller, Henry Lenzi, Jay Scott, John Dorsey, Justin Dressel, Lauri
Pesonen, Lennart Augustsson, Luc Duponcheel, Matt Hellige, Michael T.
Richter, Peter McLain, Rob deFriesse, Rüdiger Hanke, Tim Chevalier, Tim
Stewart, William N. Halchin.

We are also grateful to the people below, each of whom contributed at
least 0.2% of all comments.

Achim Schneider, Adam Jones, Alexander Semenov, Andrew Wagner, Arnar
Birgisson, Arthur van Leeuwen, Bartek Ćwikłowski, Bas Kok, Ben Franksen,
Björn Buckwalter, Brian Brunswick, Bryn Keller, Chris Holliday, Chris
Smith, Dan Scott, Dan Weston, Daniel Larsson, Davide Marchignoli, Derek
Elkins, Dirk Ullrich, Doug Kirk, Douglas Silas, Emmanuel Delaborde, Eric
Lavigne, Erik Haugen, Erik Jones, Fred Ross, Geoff King, George
Moschovitis, Hans van Thiel, Ionuț Arțăriși, Isaac Dupree, Isaac
Freeman, Jared Updike, Joe Thornber, Joeri van Eekelen, Joey Hess, Johan
Tibell, John Lenz, Josef Svenningsson, Joseph Garvin, Josh Szepietowski,
Justin Bailey, Kai Gellien, Kevin Watters, Konrad Hinsen, Lally Singh,
Lee Duhem, Luke Palmer, Magnus Therning, Marc DeRosa, Marcus Eskilsson,
Mark Lee Smith, Matthew Danish, Matthew Manela, Michael Vanier, Mike
Brauwerman, Neil Mitchell, Nick Seow, Pat Rondon, Raynor Vliegendhart,
Richard Smith, Runar Bjarnason, Ryan W. Porter, Salvatore Insalaco, Sean
Brewer, Sebastian Sylvan, Sebastien Bocq, Sengan Baring-Gould, Serge Le
Huitouze, Shahbaz Chaudhary, Shawn M Moore, Tom Tschetter, Valery V.
Vorotyntsev, Will Newton, Wolfgang Meyer, Wouter Swierstra.

We would like to acknowledge the following people, many of whom
submitted a number of comments.

Aaron Hall, Abhishek Dasgupta, Adam Copp, Adam Langley, Adam Warrington,
Adam Winiecki, Aditya Mahajan, Adolfo Builes, Al Hoang, Alan Hawkins,
Albert Brown, Alec Berryman, Alejandro Dubrovsky, Alex Hirzel, Alex
Rudnick, Alex Young, Alexander Battisti, Alexander Macdonald, Alexander
Strange, Alf Richter, Alistair Bayley, Allan Clark, Allan Erskine, Allen
Gooch, Andre Nathan, Andreas Bernstein, Andreas Schropp, Andrei Formiga,
Andrew Butterfield, Andrew Calleja, Andrew Rimes, Andrew The, Andy
Carson, Andy Payne, Angelos Sphyris, Ankur Sethi, António Pedro Cunha,
Anthony Moralez, Antoine Hersen, Antoine Latter, Antoine S., Antonio
Cangiano, Antonio Piccolboni, Antonios Antoniadis, Antonis Antoniadis,
Aristotle Pagaltzis, Arjen van Schie, Artyom Shalkhakov, Ash Logan,
Austin Seipp, Avik Das, Avinash Meetoo, BVK Chaitanya, Babu Srinivasan,
Barry Gaunt, Bas van Dijk, Ben Burdette, Ben Ellis, Ben Moseley, Ben
Sinclair, Benedikt Huber, Benjamin Terry, Benoit Jauvin-Girard, Bernie
Pope, Björn Edström, Bob Holness, Bobby Moretti, Boyd Adamson, Brad
Ediger, Bradley Unterrheiner, Brendan J. Overdiep, Brendan Macmillan,
Brett Morgan, Brian Bloniarz, Brian Lewis, Brian Palmer, Brice Lin, C
Russell, Cale Gibbard, Carlos Aya, Chad Scherrer, Chaddaï Fouché, Chance
Coble, Charles Krohn, Charlie Paucard, Chen Yufei, Cheng Wei, Chip
Grandits, Chris Ball, Chris Brew, Chris Czub, Chris Gallagher, Chris
Jenkins, Chris Kuklewicz, Chris Wright, Christian Lasarczyk, Christian
Vest Hansen, Christophe Poucet, Chung-chieh Shan, Conal Elliott, Conor
McBride, Conrad Parker, Cosmo Kastemaa, Creighton Hogg, Crutcher
Dunnavant, Curtis Warren, D Hardman, Dafydd Harries, Dale Jordan, Dan
Doel, Dan Dyer, Dan Grover, Dan Orias, Dan Schmidt, Dan Zwell, Daniel
Chicayban Bastos, Daniel Karch, Daniel Lyons, Daniel Patterson, Daniel
Wagner, Daniil Elovkov, Danny Yoo, Darren Mutz, Darrin Thompson, Dave
Bayer, Dave Hinton, Dave Leimbach, Dave Peterson, Dave Ward, David
Altenburg, David B. Wildgoose, David Carter, David Einstein, David
Ellis, David Fox, David Frey, David Goodlad, David Mathers, David
McBride, David Sabel, Dean Pucsek, Denis Bueno, Denis Volk, Devin
Mullins, Diego Moya, Dino Morelli, Dirk Markert, Dmitry Astapov, Dougal
Stanton, Dr Bean, Drew Smathers, Duane Johnson, Durward McDonell, E.
Jones, Edwin DeNicholas, Emre Sevinc, Eric Aguiar, Eric Frey, Eric Kidd,
Eric Kow, Eric Schwartz, Erik Hesselink, Erling Alf, Eruc Frey, Eugene
Grigoriev, Eugene Kirpichov, Evan Farrer, Evan Klitzke, Evan Martin,
Fawzi Mohamed, Filippo Tampieri, Florent Becker, Frank Berthold, Fred
Rotbart, Frederick Ross, Friedrich Dominicus, Gal Amram, Ganesh
Sittampalam, Gen Zhang, Geoffrey King, George Bunyan, George Rogers,
German Vidal, Gilson Silveira, Gleb Alexeyev, Glenn Ehrlich, Graham
Fawcett, Graham Lowe, Greg Bacon, Greg Chrystall, Greg Steuck, Grzegorz
Chrupała, Guillaume Marceau, Haggai Eran, Harald Armin Massa, Henning
Hasemann, Henry Laxen, Hitesh Jasani, Howard B. Golden, Ilmari Vacklin,
Imam Tashdid ul Alam, Ivan Lazar Miljenovic, Ivan Miljenovic, J. Pablo
Fernández, J.A. Zaratiegui, Jaap Weel, Jacques Richer, Jake McArthur,
Jake Poznanski, Jakub Kotowski, Jakub Labath, James Cunningham, James
Smith, Jamie Brandon, Jan Sabbe, Jared Roberts, Jason Dusek, Jason F,
Jason Kikel, Jason Mobarak, Jason Morton, Jason Rogers, Jeff Balogh,
Jeff Caldwell, Jeff Petkau, Jeffrey Bolden, Jeremy Crosbie, Jeremy
Fitzhardinge, Jeremy O\'Donoghue, Jeroen Pulles, Jim Apple, Jim Crayne,
Jim Snow, Joan Jiménez, Joe Fredette, Joe Healy, Joel Lathrop, Joeri
Samson, Johannes Laire, John Cowan, John Doe, John Hamilton, John
Hornbeck, John Lien, John Stracke, Jonathan Guitton, Joseph Bruce,
Joseph H. Buehler, Josh Goldfoot, Josh Lee, Josh Stone, Judah Jacobson,
Justin George, Justin Goguen, Kamal Al-Marhubi, Kamil Dworakowski,
Keegan Carruthers-Smith, Keith Fahlgren, Keith Willoughby, Ken Allen,
Ken Shirriff, Kent Hunter, Kevin Hely, Kevin Scaldeferri, Kingdon
Barrett, Kristjan Kannike, Kurt Jung, Lanny Ripple, Laurențiu Nicola,
Laurie Cheers, Lennart Kolmodin, Liam Groener, Lin Sun, Lionel Barret de
Nazaris, Loup Vaillant, Luke Plant, Lutz Donnerhacke, Maarten
Hazewinkel, Malcolm Reynolds, Marco Piccioni, Mark Hahnenberg, Mark
Woodward, Marko Tosic, Markus Schnell, Martijn van Egdom, Martin Bayer,
Martin DeMello, Martin Dybdal, Martin Geisler, Martin Grabmueller, Matúš
Tejiščák, Mathew Manela, Matt Brandt, Matt Russell, Matt Trinneer, Matti
Niemenmaa, Matti Nykänen, Max Cantor, Maxime Henrion, Michael Albert,
Michael Brauwerman, Michael Campbell, Michael Chermside, Michael Cook,
Michael Dougherty, Michael Feathers, Michael Grinder, Michael Kagalenko,
Michael Kaplan, Michael Orlitzky, Michael Smith, Michael Stone, Michael
Walter, Michel Salim, Mikael Vejdemo Johansson, Mike Coleman, Mike
Depot, Mike Tremoulet, Mike Vanier, Mirko Rahn, Miron Brezuleanu, Morten
Andersen, Nathan Bronson, Nathan Stien, Naveen Nathan, Neil Bartlett,
Neil Whitaker, Nick Gibson, Nick Messenger, Nick Okasinski, Nicola
Paolucci, Nicolas Frisby, Niels Aan de Brugh, Niels Holmgaard Andersen,
Nima Negahban, Olaf Leidinger, Oleg Anashkin, Oleg Dopertchouk, Oleg
Taykalo, Oliver Charles, Olivier Boudry, Omar Antolín Camarena, Parnell
Flynn, Patrick Carlisle, Paul Brown, Paul Delhanty, Paul Johnson, Paul
Lotti, Paul Moore, Paul Stanley, Paulo Tanimoto, Per Vognsen, Pete
Kazmier, Peter Aarestad, Peter Ipacs, Peter Kovaliov, Peter Merel, Peter
Seibel, Peter Sumskas, Phil Armstrong, Philip Armstrong, Philip Craig,
Philip Neustrom, Philip Turnbull, Piers Harding, Piet Delport, Pragya
Agarwal, Raúl Gutiérrez, Rafael Alemida, Rajesh Krishnan, Ralph Glass,
Rauli Ruohonen, Ravi Nanavati, Raymond Pasco, Reid Barton, Reto Kramer,
Reza Ziaei, Rhys Ulerich, Ricardo Herrmann, Richard Harris, Richard
Warburton, Rick van Hattem, Rob Grainger, Robbie Kop, Rogan Creswick,
Roman Gonzalez, Rory Winston, Ruediger Hanke, Rusty Mellinger, Ryan
Grant, Ryan Ingram, Ryan Janzen, Ryan Kaulakis, Ryan Stutsman, Ryan T.
Mulligan, S Pai, Sam Lee, Sandy Nicholson, Scott Brickner, Scott Rankin,
Scott Ribe, Sean Cross, Sean Leather, Sergei Trofimovich, Sergio
Urinovsky, Seth Gordon, Seth Tisue, Shawn Boyette, Simon Brenner, Simon
Farnsworth, Simon Marlow, Simon Meier, Simon Morgan, Sriram Srinivasan,
Sriram Srinivasan, Stefan Aeschbacher, Stefan Muenzel, Stephan
Friedrichs, Stephan Nies, Stephan-A. Posselt, Stephyn Butcher, Steven
Ashley, Stuart Dootson, Terry Michaels, Thomas Cellerier, Thomas
Fuhrmann, Thomas Hunger, Thomas M. DuBuisson, Thomas Moertel, Thomas
Schilling, Thorsten Seitz, Tibor Simic, Tilo Wiklund, Tim Clark, Tim
Eves, Tim Massingham, Tim Rakowski, Tim Wiess, Timo B. Hübel, Timothy
Fitz, Tom Moertel, Tomáš Janoušek, Tony Colston, Travis B. Hartwell,
Tristan Allwood, Tristan Seligmann, Tristram Brelstaff, Vesa
Kaihlavirta, Victor Nazarov, Ville Aine, Vincent Foley, Vipul Ved
Prakash, Vlad Skvortsov, Vojtěch Fried, Wei Cheng, Wei Hu, Will Barrett,
Will Farr, Will Leinweber, Will Robertson, Will Thompson, Wirt Wolff,
Wolfgang Jeltsch, Yuval Kogman, Zach Kozatek, Zachary Smestad, Zohar
Kelrich.

Finally, we wish to thank those readers who submitted over 800 comments
anonymously.
