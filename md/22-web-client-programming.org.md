Chapter 22. Extended Example: Web Client Programming
====================================================

By this point, you\'ve seen how to interact with a database, parse
things, and handle errors. Let\'s now take this a step farther and
introduce a web client library to the mix.

We\'ll develop a real application in this chapter: a podcast downloader,
or \"podcatcher\". The idea of a podcatcher is simple. It is given a
list of URLs to process. Downloading each of these URLs results in an
XML file in the RSS format. Inside this XML file, we\'ll find references
to URLs for audio files to download.

Podcatchers usually let the user subscribe to podcasts by adding RSS
URLs to their configuration. Then, the user can periodically run an
update operation. The podcatcher will download the RSS documents,
examine them for audio file references, and download any audio files
that haven\'t already been downloaded on behalf of this user.

::: {.TIP}
Tip

Users often call the RSS document a podcast or the podcast feed, and
each individual audio file an episode.
:::

To make this happen, we need to have several things:

-   An HTTP client library to download files
-   An XML parser
-   A way to specify and persistently store which podcasts we\'re
    interested in
-   A way to persistently store which podcast episodes we\'ve already
    downloaded

The last two items can be accommodated via a database we\'ll set up
using HDBC. The first two can be accommodated via other library modules
we\'ll introduce in this chapter.

::: {.TIP}
Tip

The code in this chapter was written specifically for this book, but is
based on code written for hpodder, an existing podcatcher written in
Haskell. hpodder has many more features than the examples presented
here, which make it too long and complex for coverage in this book. If
you are interested in studying hpodder, its source code is freely
available at <http://software.complete.org/hpodder>.
:::

We\'ll write the code for this chapter in pieces. Each piece will be its
own Haskell module. You\'ll be able to play with each piece by itself in
`ghci`. At the end, we\'ll write the final code that ties everything
together into a finished application. We\'ll start with the basic types
we\'ll need to use.

Basic Types
-----------

The first thing to do is have some idea of the basic information that
will be important to the application. This will generally be information
about the podcasts the user is interested in, plus information about
episodes that we have seen and processed. It\'s easy enough to change
this later if needed, but since we\'ll be importing it just about
everywhere, we\'ll define it first.

<div>

[PodTypes.hs]{.label}

``` {.haskell}
module PodTypes where

data Podcast =
    Podcast {castId :: Integer, -- ^ Numeric ID for this podcast
             castURL :: String  -- ^ Its feed URL
            }
    deriving (Eq, Show, Read)

data Episode =
    Episode {epId :: Integer,     -- ^ Numeric ID for this episode
             epCast :: Podcast, -- ^ The ID of the podcast it came from
             epURL :: String,     -- ^ The download URL for this episode
             epDone :: Bool       -- ^ Whether or not we are done with this ep
            }
    deriving (Eq, Show, Read)
```

</div>

We\'ll be storing this information in a database. Having a unique
identifier for both a podcast and an episode makes it easy to find which
episodes belong to a particular podcast, load information for a
particular podcast or episode, or handle future cases such as changing
URLs for podcasts.

The Database
------------

Next, we\'ll write the code to make possible persistent storage in a
database. We\'ll primarily be interested in moving data between the
Haskell structures we defined in `PodTypes.hs` and the database on disk.
Also, the first time the user runs the program, we\'ll need to create
the database tables that we\'ll use to store our data.

We\'ll use HDBC (see [Chapter 21, *Using
Databases*](21-using-databases.org)) to interact with a Sqlite database.
Sqlite is lightweight and self-contained, which makes it perfect for
this project. For information on installing HDBC and Sqlite, consult
[the section called \"Installing HDBC and
Drivers\"](21-using-databases.org::*Installing HDBC and Drivers)

<div>

[PodDB.hs]{.label}

``` {.haskell}
module PodDB where

import Database.HDBC
import Database.HDBC.Sqlite3
import PodTypes
import Control.Monad(when)
import Data.List(sort)

-- | Initialize DB and return database Connection
connect :: FilePath -> IO Connection
connect fp =
    do dbh <- connectSqlite3 fp
       prepDB dbh
       return dbh

{- | Prepare the database for our data.

We create two tables and ask the database engine to verify some pieces
of data consistency for us:

* castid and epid both are unique primary keys and must never be duplicated
* castURL also is unique
* In the episodes table, for a given podcast (epcast), there must be only
  one instance of each given URL or episode ID
-}
prepDB :: IConnection conn => conn -> IO ()
prepDB dbh =
    do tables <- getTables dbh
       when (not ("podcasts" `elem` tables)) $
           do run dbh "CREATE TABLE podcasts (\
                       \castid INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,\
                       \castURL TEXT NOT NULL UNIQUE)" []
              return ()
       when (not ("episodes" `elem` tables)) $
           do run dbh "CREATE TABLE episodes (\
                       \epid INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,\
                       \epcastid INTEGER NOT NULL,\
                       \epurl TEXT NOT NULL,\
                       \epdone INTEGER NOT NULL,\
                       \UNIQUE(epcastid, epurl),\
                       \UNIQUE(epcastid, epid))" []
              return ()
       commit dbh

{- | Adds a new podcast to the database.  Ignores the castid on the
incoming podcast, and returns a new object with the castid populated.

An attempt to add a podcast that already exists is an error. -}
addPodcast :: IConnection conn => conn -> Podcast -> IO Podcast
addPodcast dbh podcast =
    handleSql errorHandler $
      do -- Insert the castURL into the table.  The database
         -- will automatically assign a cast ID.
         run dbh "INSERT INTO podcasts (castURL) VALUES (?)"
             [toSql (castURL podcast)]
         -- Find out the castID for the URL we just added.
         r <- quickQuery' dbh "SELECT castid FROM podcasts WHERE castURL = ?"
              [toSql (castURL podcast)]
         case r of
           [[x]] -> return $ podcast {castId = fromSql x}
           y -> fail $ "addPodcast: unexpected result: " ++ show y
    where errorHandler e =
              do fail $ "Error adding podcast; does this URL already exist?\n"
                     ++ show e

{- | Adds a new episode to the database.

Since this is done by automation, instead of by user request, we will
simply ignore requests to add duplicate episodes.  This way, when we are
processing a feed, each URL encountered can be fed to this function,
without having to first look it up in the DB.

Also, we generally won't care about the new ID here, so don't bother
fetching it. -}
addEpisode :: IConnection conn => conn -> Episode -> IO ()
addEpisode dbh ep =
    run dbh "INSERT OR IGNORE INTO episodes (epCastId, epURL, epDone) \
                \VALUES (?, ?, ?)"
                [toSql (castId . epCast $ ep), toSql (epURL ep),
                 toSql (epDone ep)]
    >> return ()

{- | Modifies an existing podcast.  Looks up the given podcast by
ID and modifies the database record to match the passed Podcast. -}
updatePodcast :: IConnection conn => conn -> Podcast -> IO ()
updatePodcast dbh podcast =
    run dbh "UPDATE podcasts SET castURL = ? WHERE castId = ?"
            [toSql (castURL podcast), toSql (castId podcast)]
    >> return ()

{- | Modifies an existing episode.  Looks it up by ID and modifies the
database record to match the given episode. -}
updateEpisode :: IConnection conn => conn -> Episode -> IO ()
updateEpisode dbh episode =
    run dbh "UPDATE episodes SET epCastId = ?, epURL = ?, epDone = ? \
             \WHERE epId = ?"
             [toSql (castId . epCast $ episode),
              toSql (epURL episode),
              toSql (epDone episode),
              toSql (epId episode)]
    >> return ()

{- | Remove a podcast.  First removes any episodes that may exist
for this podcast. -}
removePodcast :: IConnection conn => conn -> Podcast -> IO ()
removePodcast dbh podcast =
    do run dbh "DELETE FROM episodes WHERE epcastid = ?"
         [toSql (castId podcast)]
       run dbh "DELETE FROM podcasts WHERE castid = ?"
         [toSql (castId podcast)]
       return ()

{- | Gets a list of all podcasts. -}
getPodcasts :: IConnection conn => conn -> IO [Podcast]
getPodcasts dbh =
    do res <- quickQuery' dbh
              "SELECT castid, casturl FROM podcasts ORDER BY castid" []
       return (map convPodcastRow res)

{- | Get a particular podcast.  Nothing if the ID doesn't match, or
Just Podcast if it does. -}
getPodcast :: IConnection conn => conn -> Integer -> IO (Maybe Podcast)
getPodcast dbh wantedId =
    do res <- quickQuery' dbh
              "SELECT castid, casturl FROM podcasts WHERE castid = ?"
              [toSql wantedId]
       case res of
         [x] -> return (Just (convPodcastRow x))
         [] -> return Nothing
         x -> fail $ "Really bad error; more than one podcast with ID"

{- | Convert the result of a SELECT into a Podcast record -}
convPodcastRow :: [SqlValue] -> Podcast
convPodcastRow [svId, svURL] =
    Podcast {castId = fromSql svId,
             castURL = fromSql svURL}
convPodcastRow x = error $ "Can't convert podcast row " ++ show x

{- | Get all episodes for a particular podcast. -}
getPodcastEpisodes :: IConnection conn => conn -> Podcast -> IO [Episode]
getPodcastEpisodes dbh pc =
    do r <- quickQuery' dbh
            "SELECT epId, epURL, epDone FROM episodes WHERE epCastId = ?"
            [toSql (castId pc)]
       return (map convEpisodeRow r)
    where convEpisodeRow [svId, svURL, svDone] =
              Episode {epId = fromSql svId, epURL = fromSql svURL,
                       epDone = fromSql svDone, epCast = pc}
```

</div>

In the `PodDB` module, we have defined functions to connect to the
database, create the needed database tables, add data to the database,
query the database, and remove data from the database. Here is an
example `ghci` session demonstrating interacting with the database. It
will create a database file named `poddbtest.db` in the current working
directory and add a podcast and an episode to it.

``` {.screen}
ghci> :load PodDB.hs
[1 of 2] Compiling PodTypes         ( PodTypes.hs, interpreted )
[2 of 2] Compiling PodDB            ( PodDB.hs, interpreted )
Ok, modules loaded: PodDB, PodTypes.
ghci> dbh <- connect "poddbtest.db"
ghci> :type dbh
dbh :: Connection
ghci> getTables dbh
["episodes","podcasts","sqlite_sequence"]
ghci> let url = "http://feeds.thisamericanlife.org/talpodcast"
ghci> pc <- addPodcast dbh (Podcast {castId=0, castURL=url})
Podcast {castId = 1, castURL = "http://feeds.thisamericanlife.org/talpodcast"}
ghci> getPodcasts dbh
[Podcast {castId = 1, castURL = "http://feeds.thisamericanlife.org/talpodcast"}]
ghci> addEpisode dbh (Episode {epId = 0, epCast = pc, epURL = "http://www.example.com/foo.mp3", epDone = False})
ghci> getPodcastEpisodes dbh pc
[Episode {epId = 1, epCast = Podcast {castId = 1, castURL = "http://feeds.thisamericanlife.org/talpodcast"}, epURL = "http://www.example.com/foo.mp3", epDone = False}]
ghci> commit dbh
ghci> disconnect dbh
```

The Parser
----------

Now that we have the database component, we need to have code to parse
the podcast feeds. These are XML files that contain various information.
Here\'s an example XML file to show you what they look like:

``` {.xml}
<?xml version="1.0" encoding="UTF-8"?>
<rss xmlns:itunes="http://www.itunes.com/DTDs/Podcast-1.0.dtd" version="2.0">
  <channel>
    <title>Haskell Radio</title>
    <link>http://www.example.com/radio/</link>
    <description>Description of this podcast</description>
    <item>
      <title>Episode 2: Lambdas</title>
      <link>http://www.example.com/radio/lambdas</link>
      <enclosure url="http://www.example.com/radio/lambdas.mp3"
       type="audio/mpeg" length="10485760"/>
    </item>
    <item>
      <title>Episode 1: Parsec</title>
      <link>http://www.example.com/radio/parsec</link>
      <enclosure url="http://www.example.com/radio/parsec.mp3"
       type="audio/mpeg" length="10485150"/>
    </item>
  </channel>
</rss>
```

Out of these files, we are mainly interested in two things: the podcast
title and the enclosure URLs. We use the [HaXml
toolkit](http://www.cs.york.ac.uk/fp/HaXml/) to parse the XML file.
Here\'s the source code for this component:

<div>

[PodParser.hs]{.label}

``` {.haskell}
module PodParser where

import PodTypes
import Text.XML.HaXml
import Text.XML.HaXml.Parse
import Text.XML.HaXml.Html.Generate(showattr)
import Data.Char
import Data.List

data PodItem = PodItem {itemtitle :: String,
                  enclosureurl :: String
                  }
          deriving (Eq, Show, Read)

data Feed = Feed {channeltitle :: String,
                  items :: [PodItem]}
            deriving (Eq, Show, Read)

{- | Given a podcast and an PodItem, produce an Episode -}
item2ep :: Podcast -> PodItem -> Episode
item2ep pc item =
    Episode {epId = 0,
             epCast = pc,
             epURL = enclosureurl item,
             epDone = False}

{- | Parse the data from a given string, with the given name to use
in error messages. -}
parse :: String -> String -> Feed
parse content name =
    Feed {channeltitle = getTitle doc,
          items = getEnclosures doc}

    where parseResult = xmlParse name (stripUnicodeBOM content)
          doc = getContent parseResult

          getContent :: Document -> Content
          getContent (Document _ _ e _) = CElem e

          {- | Some Unicode documents begin with a binary sequence;
             strip it off before processing. -}
          stripUnicodeBOM :: String -> String
          stripUnicodeBOM ('\xef':'\xbb':'\xbf':x) = x
          stripUnicodeBOM x = x

{- | Pull out the channel part of the document.

Note that HaXml defines CFilter as:

> type CFilter = Content -> [Content]
-}
channel :: CFilter
channel = tag "rss" /> tag "channel"

getTitle :: Content -> String
getTitle doc =
    contentToStringDefault "Untitled Podcast"
        (channel /> tag "title" /> txt $ doc)

getEnclosures :: Content -> [PodItem]
getEnclosures doc =
    concatMap procPodItem $ getPodItems doc
    where procPodItem :: Content -> [PodItem]
          procPodItem item = concatMap (procEnclosure title) enclosure
              where title = contentToStringDefault "Untitled Episode"
                               (keep /> tag "title" /> txt $ item)
                    enclosure = (keep /> tag "enclosure") item

          getPodItems :: CFilter
          getPodItems = channel /> tag "item"

          procEnclosure :: String -> Content -> [PodItem]
          procEnclosure title enclosure =
              map makePodItem (showattr "url" enclosure)
              where makePodItem :: Content -> PodItem
                    makePodItem x = PodItem {itemtitle = title,
                                       enclosureurl = contentToString [x]}

{- | Convert [Content] to a printable String, with a default if the
passed-in [Content] is [], signifying a lack of a match. -}
contentToStringDefault :: String -> [Content] -> String
contentToStringDefault msg [] = msg
contentToStringDefault _ x = contentToString x

{- | Convert [Content] to a printable string, taking care to unescape it.

An implementation without unescaping would simply be:

> contentToString = concatMap (show . content)

Because HaXml's unescaping only works on Elements, we must make sure that
whatever Content we have is wrapped in an Element, then use txt to
pull the insides back out. -}
contentToString :: [Content] -> String
contentToString =
    concatMap procContent
    where procContent x =
              verbatim $ keep /> txt $ CElem (unesc (fakeElem x))

          fakeElem :: Content -> Element
          fakeElem x = Elem "fake" [] [x]

          unesc :: Element -> Element
          unesc = xmlUnEscape stdXmlEscaper
```

</div>

Let\'s look at this code. First, we declare two types: `PodItem` and
`Feed`. We will be transforming the XML document into a `Feed`, which
then contains items. We also provide a function to convert an `PodItem`
into an `Episode` as defined in `PodTypes.hs`.

Next, it is on to parsing. The `parse` function takes a `String`
representing the XML content as well as a `String` representing a name
to use in error messages, and returns a `Feed`.

HaXml is designed as a \"filter\" converting data of one type to
another. It can be a simple straightforward conversion of XML to XML, or
of XML to Haskell data, or of Haskell data to XML. HaXml has a data type
called `CFilter`, which is defined like this:

``` {.haskell}
type CFilter = Content -> [Content]
```

That is, a `CFilter` takes a fragment of an XML document and returns 0
or more fragments. A `CFilter` might be asked to find all children of a
specified tag, all tags with a certain name, the literal text contained
within a part of an XML document, or any of a number of other things.
There is also an operator `(/>)` that chains `CFilter` functions
together. All of the data that we\'re interested in occurs within the
`<channel>` tag, so first we want to get at that. We define a simple
`CFilter`:

``` {.haskell}
channel = tag "rss" /> tag "channel"
```

When we pass a document to `channel`, it will search the top level for
the tag named `rss`. Then, within that, it will look for the `channel`
tag.

The rest of the program follows this basic approach. `txt` extracts the
literal text from a tag, and by using `CFilter` functions, we can get at
any part of the document.

Downloading
-----------

The next part of our program is a module to download data. We\'ll need
to download two different types of data: the content of a podcast, and
the audio for each episode. In the former case, we\'ll parse the data
and update our database. For the latter, we\'ll write the data out to a
file on disk.

We\'ll be downloading from HTTP servers, so we\'ll use a Haskell [HTTP
library](http://www.haskell.org/http/). For downloading podcast feeds,
we\'ll download the document, parse it, and update the database. For
episode audio, we\'ll download the file, write it to disk, and mark it
downloaded in the database. Here\'s the code:

<div>

[PodDownload.hs]{.label}

``` {.haskell}
module PodDownload where
import PodTypes
import PodDB
import PodParser
import Network.HTTP
import System.IO
import Database.HDBC
import Data.Maybe
import Network.URI

{- | Download a URL.  (Left errorMessage) if an error,
(Right doc) if success. -}
downloadURL :: String -> IO (Either String String)
downloadURL url =
    do resp <- simpleHTTP request
       case resp of
         Left x -> return $ Left ("Error connecting: " ++ show x)
         Right r ->
             case rspCode r of
               (2,_,_) -> return $ Right (rspBody r)
               (3,_,_) -> -- A HTTP redirect
                 case findHeader HdrLocation r of
                   Nothing -> return $ Left (show r)
                   Just url -> downloadURL url
               _ -> return $ Left (show r)
    where request = Request {rqURI = uri,
                             rqMethod = GET,
                             rqHeaders = [],
                             rqBody = ""}
          uri = fromJust $ parseURI url

{- | Update the podcast in the database. -}
updatePodcastFromFeed :: IConnection conn => conn -> Podcast -> IO ()
updatePodcastFromFeed dbh pc =
    do resp <- downloadURL (castURL pc)
       case resp of
         Left x -> putStrLn x
         Right doc -> updateDB doc

    where updateDB doc =
              do mapM_ (addEpisode dbh) episodes
                 commit dbh
              where feed = parse doc (castURL pc)
                    episodes = map (item2ep pc) (items feed)

{- | Downloads an episode, returning a String representing
the filename it was placed into, or Nothing on error. -}
getEpisode :: IConnection conn => conn -> Episode -> IO (Maybe String)
getEpisode dbh ep =
    do resp <- downloadURL (epURL ep)
       case resp of
         Left x -> do putStrLn x
                      return Nothing
         Right doc ->
             do file <- openBinaryFile filename WriteMode
                hPutStr file doc
                hClose file
                updateEpisode dbh (ep {epDone = True})
                commit dbh
                return (Just filename)
          -- This function ought to apply an extension based on the filetype
    where filename = "pod." ++ (show . castId . epCast $ ep) ++ "." ++
                     (show (epId ep)) ++ ".mp3"
```

</div>

This module defines three functions: `downloadURL`, which simply
downloads a URL and returns it as a `String`; `updatePodcastFromFeed`,
which downloads an XML feed file, parses it, and updates the database;
and `getEpisode`, which downloads a given episode and marks it done in
the database.

::: {.WARNING}
Warning

The HTTP library used here does not read the HTTP result lazily. As a
result, it can result in the consumption of a large amount of RAM when
downloading large files such as podcasts. Other libraries are available
that do not have this limitation. We used this one because it is stable,
easy to install, and reasonably easy to use. We suggest mini-http,
available from Hackage, for serious HTTP needs.
:::

Main Program
------------

Finally, we need a main program to tie it all together. Here\'s our main
module:

<div>

[PodMain.hs]{.label}

``` {.haskell}
module Main where

import PodDownload
import PodDB
import PodTypes
import System.Environment
import Database.HDBC
import Network.Socket(withSocketsDo)

main = withSocketsDo $ handleSqlError $
    do args <- getArgs
       dbh <- connect "pod.db"
       case args of
         ["add", url] -> add dbh url
         ["update"] -> update dbh
         ["download"] -> download dbh
         ["fetch"] -> do update dbh
                         download dbh
         _ -> syntaxError
       disconnect dbh

add dbh url =
    do addPodcast dbh pc
       commit dbh
    where pc = Podcast {castId = 0, castURL = url}

update dbh =
    do pclist <- getPodcasts dbh
       mapM_ procPodcast pclist
    where procPodcast pc =
              do putStrLn $ "Updating from " ++ (castURL pc)
                 updatePodcastFromFeed dbh pc

download dbh =
    do pclist <- getPodcasts dbh
       mapM_ procPodcast pclist
    where procPodcast pc =
              do putStrLn $ "Considering " ++ (castURL pc)
                 episodelist <- getPodcastEpisodes dbh pc
                 let dleps = filter (\ep -> epDone ep == False)
                             episodelist
                 mapM_ procEpisode dleps
          procEpisode ep =
              do putStrLn $ "Downloading " ++ (epURL ep)
                 getEpisode dbh ep

syntaxError = putStrLn
  "Usage: pod command [args]\n\
  \\n\
  \pod add url      Adds a new podcast with the given URL\n\
  \pod download     Downloads all pending episodes\n\
  \pod fetch        Updates, then downloads\n\
  \pod update       Downloads podcast feeds, looks for new episodes\n"
```

</div>

We have a very simple command-line parser with a function to indicate a
command-line syntax error, plus small functions to handle the different
command-line arguments.

You can compile this program with a command like this:

``` {.screen}
ghc --make -O2 -o pod -package HTTP -package HaXml -package network \
    -package HDBC -package HDBC-sqlite3 PodMain.hs
```

Alternatively, you could use a Cabal file as documented in [the section
called \"Creating a
package\"](5-writing-a-library.org::*Creating a package)

<div>

[pod.cabal]{.label}

    Name: pod
    Version: 1.0.0
    Build-type: Simple
    Build-Depends: HTTP, HaXml, network, HDBC, HDBC-sqlite3, base

    Executable: pod
    Main-Is: PodMain.hs
    GHC-Options: -O2

</div>

Also, you\'ll want a simple `Setup.hs` file:

``` {.haskell}
import Distribution.Simple
main = defaultMain
```

Now, to build with Cabal, you just run:

``` {.screen}
runghc Setup.hs configure
runghc Setup.hs build
```

And you\'ll find a `dist` directory containing your output. To install
the program system-wide, run `runghc Setup.hs install`.
