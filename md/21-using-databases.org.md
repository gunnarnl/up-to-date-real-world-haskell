Chapter 21. Using Databases
===========================

Everything from web forums to podcatchers or even backup programs
frequently use databases for persistent storage. SQL-based databases are
often quite convenient: they are fast, can scale from tiny to massive
sizes, can operate over the network, often help handle locking and
transactions, and can even provide failover and redundancy improvements
for applications. Databases come in many different shapes: the large
commercial databases such as Oracle, Open Source engines such as
PostgreSQL or MySQL, and even embeddable engines such as Sqlite.

Because databases are so important, Haskell support for them is
important as well. In this chapter, we will introduce you to one of the
Haskell frameworks for working with databases. We will also use this
framework to begin building a podcast downloader, which we will further
develop in [Chapter 22, *Extended Example: Web Client
Programming*](22-web-client-programming.org).

Overview of HDBC
----------------

At the bottom of the database stack is the database engine. The database
engine is responsible for actually storing data on disk. Well-known
database engines include PostgreSQL, MySQL, and Oracle.

Most modern database engines support SQL, the Structured Query Language,
as a standard way of getting data into and out of relational databases.
This book will not provide a tutorial on SQL or relational database
management.[^1]

Once you have a database engine that supports SQL, you need a way to
communicate with it. Each database has its own protocol. Since SQL is
reasonably constant across databases, it is possible to make a generic
interface that uses drivers for each individual protocol.

Haskell has several different database frameworks available, some
providing high-level layers atop others. For this chapter, we will
concentrate on HDBC, the Haskell DataBase Connectivity system. HDBC is a
database abstraction library. That is, you can write code that uses HDBC
and can access data stored in almost any SQL database with little or no
modification.[^2] Even if you never need to switch underlying database
engines, the HDBC system of drivers makes a large number of choices
available to you with a single interface.

Another database abstraction library for Haskell is HSQL, which shares a
similar purpose with HDBC. There is also a higher-level framework called
HaskellDB, which sits atop either HDBC or HSQL, and is designed to help
insulate the programmer from the details of working with SQL. However,
it does not have as broad appeal because its design limits it to
certain---albeit quite common--- database access patterns. Finally,
Takusen is a framework that uses a \"left fold\" approach to reading
data from the database.

Installing HDBC and Drivers
---------------------------

To connect to a given database with HDBC, you need at least two
packages: the generic interface, and a driver for your specific
database. You can obtain the generic HDBC package, and all of the other
drivers, from [Hackage](http://hackage.haskell.org/)[^3]. For this
chapter, we will use HDBC version 1.1.3 for examples.

You\'ll also need a database backend and backend driver. For this
chapter, we\'ll use Sqlite version 3. Sqlite is an embedded database, so
it doesn\'t require a separate server and is easy to set up. Many
operating systems already ship with Sqlite version 3. If yours doesn\'t,
you can download it from <http://www.sqlite.org/>. The HDBC homepage has
a link to known HDBC backend drivers. The specific driver for Sqlite
version 3 can be obtained from Hackage.

If you want to use HDBC with other databases, check out the HDBC Known
Drivers page at <http://software.complete.org/hdbc/wiki/KnownDrivers>.
There you will find a link to the ODBC binding, which lets you connect
to virtually any database on virtually any platform (Windows, POSIX, and
others). You will also find a PostgreSQL binding. MySQL is supported via
the ODBC binding, and specific information for MySQL users can be found
in the [HDBC-ODBC API
documentation](http://software.complete.org/static/hdbc-odbc/doc/HDBC-odbc/).

Connecting to Databases
-----------------------

To connect to a database, you will use a connection function from a
database backend driver. Each database has its own unique method of
connecting. The initial connection is generally the only time you will
call anything from a backend driver module directly.

The database connection function will return a database handle. The
precise type of this handle may vary from one driver to the next, but it
will always be an instance of the `IConnection` type class. All of the
functions you will use to operate on databases will work with any type
that is an instance of `IConnection`. When you\'re done talking to the
database, call the `disconnect` function. It will disconnect you from
the database. Here\'s an example of connecting to a Sqlite database:

``` {.screen}
ghci> :module Database.HDBC Database.HDBC.Sqlite3
ghci> conn <- connectSqlite3 "test1.db"
Loading package array-0.1.0.0 ... linking ... done.
Loading package containers-0.1.0.1 ... linking ... done.
Loading package bytestring-0.9.0.1 ... linking ... done.
Loading package old-locale-1.0.0.0 ... linking ... done.
Loading package old-time-1.0.0.0 ... linking ... done.
Loading package mtl-1.1.0.0 ... linking ... done.
Loading package HDBC-1.1.5 ... linking ... done.
Loading package HDBC-sqlite3-1.1.4.0 ... linking ... done.
ghci> :type conn
conn :: Connection
ghci> disconnect conn
```

Transactions
------------

Most modern SQL databases have a notion of transactions. A transaction
is designed to ensure that all components of a modification get applied,
or that none of them do. Furthermore, transactions help prevent other
processes accessing the same database from seeing partial data from
modifications that are in progress.

Many databases require you to either explicitly commit all your changes
before they appear on disk, or to run in an \"autocommit\" mode. The
\"autocommit\" mode runs an implicit commit after every statement. This
may make the adjustment to transactional databases easier for
programmers not accustomed to them, but is just a hindrance to people
who actually want to use multi-statement transactions.

HDBC intentionally does not support autocommit mode. When you modify
data in your databases, you must explicitly cause it to be committed to
disk. There are two ways to do that in HDBC: you can call `commit` when
you\'re ready to write the data to disk, or you can use the
`withTransaction` function to wrap around your modification code.
`withTransaction` will cause data to be committed upon successful
completion of your function.

Sometimes a problem will occur while you are working on writing data to
the database. Perhaps you get an error from the database or discover a
problem with the data. In these instances, you can \"roll back\" your
changes. This will cause all changes you were making since your last
`commit` or roll back to be forgotten. In HDBC, you can call the
`rollback` function to do this. If you are using `withTransaction`, any
uncaught exception will cause a roll back to be issued.

Note that a roll back operation only rolls back the changes since the
last `commit`, `rollback`, or `withTransaction`. A database does not
maintain an extensive history like a version-control system. You will
see examples of `commit` later in this chapter.

::: {.WARNING}
Warning

One popular database, MySQL, does not support transactions with its
default table type. In its default configuration, MySQL will silently
ignore calls to `commit` or `rollback` and will commit all changes to
disk immediately. The HDBC ODBC driver has instructions for configuring
MySQL to indicate to HDBC that it does not support transactions, which
will cause `commit` and `rollback` to generate errors. Alternatively,
you can use InnoDB tables with MySQL, which do support transactions.
InnoDB tables are recommended for use with HDBC.
:::

Simple Queries
--------------

Some of the simplest queries in SQL involve statements that don\'t
return any data. These queries can be used to create tables, insert
data, delete data, and set database parameters.

The most basic function for sending queries to a database is `run`. This
function takes an `IConnection`, a `String` representing the query
itself, and a list of parameters. Let\'s use it to set up some things in
our database.

``` {.screen}
ghci> :module Database.HDBC Database.HDBC.Sqlite3
ghci> conn <- connectSqlite3 "test1.db"
Loading package array-0.1.0.0 ... linking ... done.
Loading package containers-0.1.0.1 ... linking ... done.
Loading package bytestring-0.9.0.1 ... linking ... done.
Loading package old-locale-1.0.0.0 ... linking ... done.
Loading package old-time-1.0.0.0 ... linking ... done.
Loading package mtl-1.1.0.0 ... linking ... done.
Loading package HDBC-1.1.5 ... linking ... done.
Loading package HDBC-sqlite3-1.1.4.0 ... linking ... done.
ghci> run conn "CREATE TABLE test (id INTEGER NOT NULL, desc VARCHAR(80))" []
0
ghci> run conn "INSERT INTO test (id) VALUES (0)" []
1
ghci> commit conn
ghci> disconnect conn
```

After connecting to the database, we first created a table called
`test`. Then we inserted one row of data into the table. Finally, we
committed the changes and disconnected from the database. Note that if
we hadn\'t called `commit`, no final change would have been written to
the database at all.

The `run` function returns the number of rows each query modified. For
the first query, which created a table, no rows were modified. The
second query inserted a single row, so `run` returned `1`.

`SqlValues`
-----------

Before proceeding, we need to discuss a data type introduced in HDBC:
`SqlValue`. Since both Haskell and SQL are strongly-typed systems, HDBC
tries to preserve type information as much as possible. At the same
time, Haskell and SQL types don\'t exactly mirror each other.
Furthermore, different databases have different ways of representing
things such as dates or special characters in strings.

`SqlValue` is a data type that has a number of constructors such as
`SqlString`, `SqlBool`, `SqlNull`, `SqlInteger`, and more. This lets you
represent various types of data in argument lists to the database, and
to see various types of data in the results coming back, and still store
it all in a list. There are convenience functions `toSql` and `fromSql`
that you will normally use. If you care about the precise representation
of data, you can still manually construct `SqlValue` data if you need
to.

Query Parameters
----------------

HDBC, like most databases, supports a notion of replaceable parameters
in queries. There are three primary benefits of using replaceable
parameters: they prevent SQL injection attacks or trouble when the input
contains quote characters, they improve performance when executing
similar queries repeatedly, and they permit easy and portable insertion
of data into queries.

Let\'s say you wanted to add thousands of rows into our new table
`test`. You could issue thousands of queries looking like
`INSERT INTO test VALUES (0, 'zero')` and
`INSERT INTO test VALUES (1, 'one')`. This forces the database server to
parse each SQL statement individually. If you could replace the two
values with a placeholder, the server could parse the SQL query once,
and just execute it multiple times with the different data.

A second problem involves escaping characters. What if you wanted to
insert the string `"I don't like 1"`? SQL uses the single quote
character to show the end of the field. Most SQL databases would require
you to write this as `'I don''t like 1'`. But rules for other special
characters such as backslashes differ between databases. Rather than
trying to code this yourself, HDBC can handle it all for you. Let\'s
look at an example.

``` {.screen}
ghci> conn <- connectSqlite3 "test1.db"
ghci> run conn "INSERT INTO test VALUES (?, ?)" [toSql 0, toSql "zero"]
1
ghci> commit conn
ghci> disconnect conn
```

The question marks in the `INSERT` query in this example are the
placeholders. We then passed the parameters that are going to go there.
`run` takes a list of `SqlValue`, so we used `toSql` to convert each
item into an `SqlValue`. HDBC automatically handled conversion of the
`String` `"zero"` into the appropriate representation for the database
in use.

This approach won\'t actually achieve any performance benefits when
inserting large amounts of data. For that, we need more control over the
process of creating the SQL query. We\'ll discuss that in the next
section.

::: {.NOTE}
Using replaceable parameters

Replaceable parameters only work for parts of the queries where the
server is expecting a value, such as a `WHERE` clause in a `SELECT`
statement or a value for an `INSERT` statement. You cannot say
`run "SELECT * from ?" [toSql "tablename"]` and expect it to work. A
table name is not a value, and most databases will not accept this
syntax. That\'s not a big problem in practice, because there is rarely a
call for replacing things that aren\'t values in this way.
:::

Prepared Statements
-------------------

HDBC defines a function `prepare` that will prepare a SQL query, but it
does not yet bind the parameters to the query. `prepare` returns a
`Statement` representing the compiled query.

Once you have a `Statement`, you can do a number of things with it. You
can call `execute` on it one or more times. After calling `execute` on a
query that returns data, you can use one of the fetch functions to
retrieve that data. Functions like `run` and `quickQuery'` use
statements and `execute` internally; they are simply shortcuts to let
you perform common tasks quickly. When you need more control over
what\'s happening, you can use a `Statement` instead of a function like
`run`.

Let\'s look at using statements to insert multiple values with a single
query. Here\'s an example:

``` {.screen}
ghci> conn <- connectSqlite3 "test1.db"
ghci> stmt <- prepare conn "INSERT INTO test VALUES (?, ?)"
ghci> execute stmt [toSql 1, toSql "one"]
1
ghci> execute stmt [toSql 2, toSql "two"]
1
ghci> execute stmt [toSql 3, toSql "three"]
1
ghci> execute stmt [toSql 4, SqlNull]
1
ghci> commit conn
ghci> disconnect conn
```

In this example, we created a prepared statement and called it `stmt`.
We then executed that statement four times, and passed different
parameters each time. These parameters are used, in order, to replace
the question marks in the original query string. Finally, we commit the
changes and disconnect the database.

HDBC also provides a function `executeMany` that can be useful in
situations such as this. `executeMany` simply takes a list of rows of
data to call the statement with. Here\'s an example:

``` {.screen}
ghci> conn <- connectSqlite3 "test1.db"
ghci> stmt <- prepare conn "INSERT INTO test VALUES (?, ?)"
ghci> executeMany stmt [[toSql 5, toSql "five's nice"], [toSql 6, SqlNull]]
ghci> commit conn
ghci> disconnect conn
```

::: {.NOTE}
More efficient execution

On the server, most databases will have an optimization that they can
apply to `executeMany` so that they only have to compile this query
string once, rather than twice.[^4] This can lead to a dramatic
performance gain when inserting large amounts of data at once. Some
databases can also apply this optimization to `execute`, but not all.
:::

Reading Results
---------------

So far, we have discussed queries that insert or change data. Let\'s
discuss getting data back out of the database. The type of the function
`quickQuery'` looks very similar to `run`, but it returns a list of
results instead of a count of changed rows. `quickQuery'` is normally
used with `SELECT` statements. Let\'s see an example:

``` {.screen}
ghci> conn <- connectSqlite3 "test1.db"
ghci> quickQuery' conn "SELECT * from test where id < 2" []
[[SqlString "0",SqlNull],[SqlString "0",SqlString "zero"],[SqlString "1",SqlString "one"]]
ghci> disconnect conn
```

`quickQuery'` works with replaceable parameters, as we discussed above.
In this case, we aren\'t using any, so the set of values to replace is
the empty list at the end of the `quickQuery'` call. `quickQuery'`
returns a list of rows, where each row is itself represented as
`[SqlValue]`. The values in the row are listed in the order returned by
the database. You can use `fromSql` to convert them into regular Haskell
types as needed.

It\'s a bit hard to read that output. Let\'s extend this example to
format the results nicely. Here\'s some code to do that:

<div>

[query.hs]{.label}

``` {.haskell}
import Database.HDBC.Sqlite3 (connectSqlite3)
import Database.HDBC

{- | Define a function that takes an integer representing the maximum
id value to look up.  Will fetch all matching rows from the test database
and print them to the screen in a friendly format. -}
query :: Int -> IO ()
query maxId =
    do -- Connect to the database
       conn <- connectSqlite3 "test1.db"

       -- Run the query and store the results in r
       r <- quickQuery' conn
            "SELECT id, desc from test where id <= ? ORDER BY id, desc"
            [toSql maxId]

       -- Convert each row into a String
       let stringRows = map convRow r

       -- Print the rows out
       mapM_ putStrLn stringRows

       -- And disconnect from the database
       disconnect conn

    where convRow :: [SqlValue] -> String
          convRow [sqlId, sqlDesc] =
              show intid ++ ": " ++ desc
              where intid = (fromSql sqlId)::Integer
                    desc = case fromSql sqlDesc of
                             Just x -> x
                             Nothing -> "NULL"
          convRow x = fail $ "Unexpected result: " ++ show x
```

</div>

This program does mostly the same thing as our example with `ghci`, but
with a new addition: the `convRow` function. This function takes a row
of data from the database and converts it to a `String`. This string can
then be easily printed out.

Notice how we took `intid` from `fromSql` directly, but processed
`fromSql sqlDesc` as a `Maybe String` type. If you recall, we declared
that the first column in this table can never contain a `NULL` value,
but that the second column could. Therefore, we can safely ignore the
potential for a `NULL` in the first column, but not in the second. It is
possible to use `fromSql` to convert the second column to a `String`
directly, and it would even work---until a row with a `NULL` in that
position was encountered, which would cause a runtime exception. So, we
convert a SQL `NULL` value into the string `"NULL"`. When printed, this
will be indistinguishable from a SQL string `'NULL'`, but that\'s
acceptable for this example. Let\'s try calling this function in `ghci`:

``` {.screen}
ghci> :load query.hs
[1 of 1] Compiling Main             ( query.hs, interpreted )
Ok, modules loaded: Main.
ghci> query 2
0: NULL
0: zero
1: one
2: two
```

### Reading with Statements

As we discussed in [the section called \"Prepared
Statements\"](21-using-databases.org::*Prepared Statements) you can use
statements for reading. There are a number of ways of reading data from
statements that can be useful in certain situations. Like `run`,
`quickQuery'` is a convenience function that in fact uses statements to
accomplish its task.

To create a statement for reading, you use `prepare` just as you would
for a statement that will be used to write data. You also use `execute`
to execute it on the database server. Then, you can use various
functions to read data from the `Statement`. The `fetchAllRows'`
function returns `[[SqlValue]]`, just like `quickQuery'`. There is also
a function called `sFetchAllRows'`, which converts every column\'s data
to a `Maybe String` before returning it. Finally, there is
`fetchAllRowsAL'`, which returns `(String, SqlValue)` pairs for each
column. The `String` is the column name as returned by the database; see
[the section called \"Database
Metadata\"](21-using-databases.org::*Database Metadata)

You can also read data one row at a time by calling `fetchRow`, which
returns `IO (Maybe [SqlValue])`. It will be `Nothing` if all the results
have already been read, or one row otherwise.

### Lazy Reading

Back in [the section called \"Lazy I/O\"](7-io.org::*Lazy I/O) from
files. It is also possible to read data lazily from databases. This can
be particularly useful when dealing with queries that return an
exceptionally large amount of data. By reading data lazily, you can
still use convenient functions such as `fetchAllRows` instead of having
to manually read each row as it comes in. If you are careful in your use
of the data, you can avoid having to buffer all of the results in
memory.

Lazy reading from a database, however, is more complex than reading from
a file. When you\'re done reading data lazily from a file, the file is
closed, and that\'s generally fine. When you\'re done reading data
lazily from a database, the database connection is still open---you may
be submitting other queries with it, for instance. Some databases can
even support multiple simultaneous queries, so HDBC can\'t just close
the connection when you\'re done.

When using lazy reading, it is critically important that you finish
reading the entire data set before you attempt to close the connection
or execute a new query. We encourage you to use the strict functions, or
row-by-row processing, wherever possible to minimize complex
interactions with lazy reading.

::: {.TIP}
Tip

If you are new to HDBC or the concept of lazy reading, but have lots of
data to read, repeated calls to `fetchRow` may be easier to understand.
Lazy reading is a powerful and useful tool, but must be used correctly.
:::

To read lazily from a database, you use the same functions you used
before, without the apostrophe. For instance, you\'d use `fetchAllRows`
instead of `fetchAllRows'`. The types of the lazy functions are the same
as their strict cousins. Here\'s an example of lazy reading:

``` {.screen}
ghci> conn <- connectSqlite3 "test1.db"
ghci> stmt <- prepare conn "SELECT * from test where id < 2"
ghci> execute stmt []
0
ghci> results <- fetchAllRowsAL stmt
[[("id",SqlString "0"),("desc",SqlNull)],[("id",SqlString "0"),("desc",SqlString "zero")],[("id",SqlString "1"),("desc",SqlString "one")]]
ghci> mapM_ print results
[("id",SqlString "0"),("desc",SqlNull)]
[("id",SqlString "0"),("desc",SqlString "zero")]
[("id",SqlString "1"),("desc",SqlString "one")]
ghci> disconnect conn
```

Note that you could have used `fetchAllRowsAL'` here as well. However,
if you had a large data set to read, it would have consumed a lot of
memory. By reading the data lazily, we can print out extremely large
result sets using a constant amount of memory. With the lazy version,
results will be evaluated in chunks; with the strict version, all
results are read up front, stored in RAM, then printed.

Database Metadata
-----------------

Sometimes it can be useful for a program to learn information about the
database itself. For instance, a program may want to see what tables
exist so that it can automatically create missing tables or upgrade the
database schema. In some cases, a program may need to alter its behavior
depending on the database backend in use.

First, there is a `getTables` function that will obtain a list of
defined tables in a database. You can also use the `describeTable`
function, which will provide information about the defined columns in a
given table.

You can learn about the database server in use by calling `dbServerVer`
and `proxiedClientName`, for instance. The `dbTransactionSupport`
function can be used to determine whether or not a given database
supports transactions. Let\'s look at an example of some of these items:

``` {.screen}
ghci> conn <- connectSqlite3 "test1.db"
ghci> getTables conn
["test"]
ghci> proxiedClientName conn
"sqlite3"
ghci> dbServerVer conn
"3.5.9"
ghci> dbTransactionSupport conn
True
ghci> disconnect conn
```

You can also learn about the results of a specific query by obtaining
information from its statement. The `describeResult` function returns
`[(String, SqlColDesc)]`, a list of pairs. The first item gives the
column name, and the second provides information about the column: the
type, the size, whether it may be `NULL`. The full specification is
given in the HDBC API reference.

Please note that some databases may not be able to provide all this
metadata. In these circumstances, an exception will be raised. Sqlite3,
for instance, does not support `describeResult` or `describeTable` as of
this writing.

Error Handling
--------------

HDBC will raise exceptions when errors occur. The exceptions have type
`SqlError`. They convey information from the underlying SQL engine, such
as the database\'s state, the error message, and the database\'s numeric
error code, if any.

`ghc` does not know how to display an `SqlError` on the screen when it
occurs. While the exception will cause the program to terminate, it will
not display a useful message. Here\'s an example:

``` {.screen}
ghci> conn <- connectSqlite3 "test1.db"
ghci> quickQuery' conn "SELECT * from test2" []
*** Exception: (unknown)
ghci> disconnect conn
```

Here we tried to `SELECT` data from a table that didn\'t exist. The
error message we got back wasn\'t helpful. There\'s a utility function,
`handleSqlError`, that will catch an `SqlError` and re-raise it as an
`IOError`. In this form, it will be printable on-screen, but it will be
more difficult to extract specific pieces of information
programmatically. Let\'s look at its usage:

``` {.screen}
ghci> conn <- connectSqlite3 "test1.db"
ghci> handleSqlError $ quickQuery' conn "SELECT * from test2" []
*** Exception: user error (SQL error: SqlError {seState = "", seNativeError = 1, seErrorMsg = "prepare 20: SELECT * from test2: no such table: test2"})
ghci> disconnect conn
```

Here we got more information, including even a message saying that there
is no such table as test2. This is much more helpful. Many HDBC
programmers make it a standard practice to start their programs with
`main = handleSqlError $ do`, which will ensure that every un-caught
`SqlError` will be printed in a helpful manner.

There are also `catchSql` and `handleSql~—similar to the standard
~catch` and `handle` functions. `catchSql` and `handleSql` will
intercept only HDBC errors. For more information on error handling,
refer to [Chapter 19, *Error handling*](19-error-handling.org).

Footnotes
---------

[^1]: The O\'Reilly books *Learning SQL* and *SQL in a Nutshell* may be
    useful if you don\'t have experience with SQL.

[^2]: This assumes you restrict yourself to using standard SQL.

[^3]: For more information on installing Haskell software, please refer
    to [the section called \"Installing Haskell
    software\"](installing-ghc-and-haskell-libraries.org::*Installing Haskell software)

[^4]: HDBC emulates this behavior for databases that do not provide it,
    providing programmers a unified API for running queries repeatedly.
