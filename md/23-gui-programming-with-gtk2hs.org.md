Chapter 23. GUI Programming with gtk2hs
=======================================

Throughout this book, we have been developing simple text-based tools.
While these are often ideal interfaces, sometimes a graphical user
interface (GUI) is required. There are several GUI toolkits available
for Haskell. In this chapter, we will look at one of the, gtk2hs.[^1]

Installing gtk2hs
-----------------

Before we dive in to working with gtk2hs, you\'ll need to get it
installed. On most Linux, BSD, or other POSIX platforms, you will find
ready-made gtk2hs packages. You will generally need to install the GTK+
development environment, Glade, and gtk2hs. The specifics of doing so
vary by distribution.

Windows and Mac developers should consult the gtk2hs downloads site at
<http://www.haskell.org/gtk2hs/download/>. Begin by downloading gtk2hs
from there. Then you will also need Glade version 3. Mac developers can
find this at <http://www.macports.org/>, while Windows developers should
consult <http://sourceforge.net/projects/gladewin32>.

Overview of the GTK+ Stack
--------------------------

Before diving in to the code, let\'s pause a brief moment and consider
the architecture of the system we are going to use. First off, we have
GTK+. GTK+ is a cross-platform GUI-building toolkit, implemented in C.
It runs on Windows, Mac, Linux, BSDs, and more. It is also the toolkit
beneath the Gnome desktop environment.

Next, we have Glade. Glade is a user interface designer, which lets you
graphically lay out your application\'s windows and dialogs. Glade saves
the interface in XML files, which your application will load at runtime.

The last piece of this puzzle is gtk2hs. This is the Haskell binding for
GTK+, Glade, and several related libraries. It is one of many language
bindings available for GTK+.

User Interface Design with Glade
--------------------------------

In this chapter, we are going to develop a GUI for the podcast
downloader we first developed in [Chapter 22, *Extended Example: Web
Client Programming*](22-web-client-programming.org). Our first task is
to design the user interface in Glade. Once we have accomplished that,
we will write the Haskell code to integrate it with the application.

Because this is a Haskell book, rather than a GUI design book, we will
move fast through some of these early parts. For more information on
interface design with Glade, you may wish to refer to one of these
resources:

-   The Glade homepage, which contains documentation for Glade.
    <http://glade.gnome.org/>
-   The GTK+ homepage contains information about the different widgets.
    Refer to the documentation section, then the stable GTK
    documentation area. <http://www.gtk.org/>
-   The gtk2hs homepage also has a useful documentation section, which
    contains an API reference to gtk2hs as well as a glade tutorial.
    <http://www.haskell.org/gtk2hs/documentation/>

### Glade Concepts

Glade is a user interface design tool. It lets us use a graphical
interface to design our graphical interface. We could build up the
window components using a bunch of calls to GTK+ functions, but it is
usually easier to do this with Glade.

The fundamental \"thing\" we work with in GTK+ is the *widget*. A widget
represents any part of the GUI, and may contain other widgets. Some
examples of widgets include a window, dialog box, button, and text
within the button.

Glade, then, is a widget layout tool. We set up a whole tree of widgets,
with top-level windows at the top of the tree. You can think of Glade
and widgets in somewhat the same terms as HTML: you can arrange widgets
in a table-like layout, set up padding rules, and structure the entire
description in a hierarchical way.

Glade saves the widget descriptions into an XML file. Our program loads
this XML file at runtime. We load the widgets by asking the Glade
runtime library to load a widget with a specific name.

Here\'s a screenshot of an example working with Glade to design our
application\'s main screen:

![](figs/gui-glade-3.png)

In the downloadable material available for this book, you can find the
full Glade XML file as `podresources.glade`. You can load this file in
Glade and edit it if you wish.

Event-Driven Programming
------------------------

GTK+, like many GUI toolkits, is an *event-driven* toolkit. That means
that instead of, say, displaying a dialog box and waiting for the user
to click on a button, we instead tell gtk2hs what function to call if a
certain button is clicked, but don\'t sit there waiting for a click in
the dialog box.

This is different from the model traditionally used for console
programs. When you think about it, though, it almost has to be. A GUI
program could have multiple windows open, and writing code to sit there
waiting for input in the particular combination of open windows could be
a complicated proposition.

Event-driven programming complements Haskell nicely. As we\'ve discussed
over and over in this book, functional languages thrive on passing
around functions. So we\'ll be passing functions to gtk2hs that get
called when certain events occur. These are known as *callback
functions*.

At the core of a GTK+ program is the *main loop*. This is the part of
the program that waits for actions from the user or commands from the
program and carries them out. The GTK+ main loop is handled entirely by
GTK+. To us, it looks like an I/O action that we execute, that doesn\'t
return until the GUI has been disposed of.

Since the main loop is responsible for doing everything from handling
clicks of a mouse to redrawing a window when it has been uncovered, it
must always be available. We can\'t just run a long-running task---such
as downloading a podcast episode---from within the main loop. This would
make the GUI unresponsive, and actions such as clicking a Cancel button
wouldn\'t be processed in a timely manner.

Therefore, we will be using multithreading to handle these long-running
tasks. More information on multithreading can be found in [Chapter 24,
*Concurrent and multicore
programming*](24-concurrent-and-multicore-programming.org). For now,
just know that we will use `forkIO` to create new threads for
long-running tasks such as downloading podcast feeds and episodes. For
very quick tasks, such as adding a new podcast to the database, we will
not bother with a separate thread since it will be executed so fast the
user will never notice.

Initializing the GUI
--------------------

Our first steps are going to involve initializing the GUI for our
program. For reasons that we\'ll explain in [the section called \"Using
Cabal\"](23-gui-programming-with-gtk2hs.org::*Using Cabal) file called
`PodLocalMain.hs` that loads `PodMain` and passes to it the path to
`podresources.glade`, the XML file saved by Glade that gives the
information about our GUI widgets.

<div>

[PodLocalMain.hs]{.label}

``` {.haskell}
module Main where

import qualified PodMainGUI

main = PodMainGUI.main "podresources.glade"
```

</div>

Now, let\'s consider `PodMainGUI.hs`. This file is the only Haskell
source file that we had to modify from the example in [Chapter 22,
*Extended Example: Web Client
Programming*](22-web-client-programming.org) to make it work as a GUI.
Let\'s start by looking at the start of our new `PodMainGUI.hs`
file---we\'ve renamed it from `PodMain.hs` for clarity.

<div>

[PodMainGUI.hs]{.label}

``` {.haskell}
module PodMainGUI where

import PodDownload
import PodDB
import PodTypes
import System.Environment
import Database.HDBC
import Network.Socket(withSocketsDo)

-- GUI libraries

import Graphics.UI.Gtk hiding (disconnect)
import Graphics.UI.Gtk.Glade

-- Threading

import Control.Concurrent
```

</div>

This first part of `PodMainGUI.hs` is similar to our non-GUI version. We
import three additional components, however. First, we have
`Graphics.UI.Gtk`, which provides most of the GTK+ functions we will be
using. Both this module and `Database.HDBC` provide a function named
`disconnect`. Since we\'ll be using the HDBC version, but not the GTK+
version, we don\'t import that function from `Graphics.UI.Gtk`.
`Graphics.UI.Gtk.Glade` contains functions needed for loading and
working with our Glade file.

We also import `Control.Concurrent`, which has the basics needed for
multi-threaded programming. We\'ll use a few functions from here as
described above once we get into the guts of the program. Next, let\'s
define a type to store information about our GUI.

<div>

[PodMainGUI.hs]{.label}

``` {.haskell}
-- | Our main GUI type
data GUI = GUI {
      mainWin :: Window,
      mwAddBt :: Button,
      mwUpdateBt :: Button,
      mwDownloadBt :: Button,
      mwFetchBt :: Button,
      mwExitBt :: Button,
      statusWin :: Dialog,
      swOKBt :: Button,
      swCancelBt :: Button,
      swLabel :: Label,
      addWin :: Dialog,
      awOKBt :: Button,
      awCancelBt :: Button,
      awEntry :: Entry}
```

</div>

Our new `GUI` type stores all the widgets we will care about in the
entire program. Large programs may not wish to have a monolithic type
like this. For this small example, it makes sense because it can be
easily passed around to different functions, and we\'ll know that we
always have the information we need available.

Within this record, we have fields for a `Window` (a top-level window),
`Dialog` (dialog window), `Button` (clickable button), `Label` (piece of
text), and `Entry` (place for the user to enter text). Let\'s now look
at our `main` function:

<div>

[PodMainGUI.hs]{.label}

``` {.haskell}
main :: FilePath -> IO ()
main gladepath = withSocketsDo $ handleSqlError $
    do initGUI                  -- Initialize GTK+ engine

       -- Every so often, we try to run other threads.
       timeoutAddFull (yield >> return True)
                      priorityDefaultIdle 100

       -- Load the GUI from the Glade file
       gui <- loadGlade gladepath

       -- Connect to the database
       dbh <- connect "pod.db"

       -- Set up our events
       connectGui gui dbh

       -- Run the GTK+ main loop; exits after GUI is done
       mainGUI

       -- Disconnect from the database at the end
       disconnect dbh
```

</div>

Remember that the type of this `main` function is a little different
than usual because it is being called by `main` in `PodLocalMain.hs`. We
start by calling `initGUI`, which initializes the GTK+ system. Next, we
have a call to `timeoutAddFull`. This call is only needed for
multithreaded GTK+ programs. It tells the GTK+ main loop to pause to
give other threads a chance to run every so often.

After that, we call our `loadGlade` function (see below) to load the
widgets from our Glade XML file. After that, we connect to our database,
call our `connectGui` function to set up our callback functions. Then,
we fire up the GTK+ main loop. We expect it could be minutes, hours, or
even days before `mainGUI` returns. When it does, it means the user has
closed the main window or clicked the Exit button. After that, we
disconnect from the database and close the program. Now, let\'s look at
our `loadGlade` function.

<div>

[PodMainGUI.hs]{.label}

``` {.haskell}
loadGlade gladepath =
    do -- Load XML from glade path.
       -- Note: crashes with a runtime error on console if fails!
       Just xml <- xmlNew gladepath

       -- Load main window
       mw <- xmlGetWidget xml castToWindow "mainWindow"

       -- Load all buttons

       [mwAdd, mwUpdate, mwDownload, mwFetch, mwExit, swOK, swCancel,
        auOK, auCancel] <-
           mapM (xmlGetWidget xml castToButton)
           ["addButton", "updateButton", "downloadButton",
            "fetchButton", "exitButton", "okButton", "cancelButton",
            "auOK", "auCancel"]

       sw <- xmlGetWidget xml castToDialog "statusDialog"
       swl <- xmlGetWidget xml castToLabel "statusLabel"

       au <- xmlGetWidget xml castToDialog "addDialog"
       aue <- xmlGetWidget xml castToEntry "auEntry"

       return $ GUI mw mwAdd mwUpdate mwDownload mwFetch mwExit
              sw swOK swCancel swl au auOK auCancel aue
```

</div>

This function starts by calling `xmlNew`, which loads the Glade XML
file. It returns `Nothing` on error. Here we are using pattern matching
to extract the result value on success. If it fails, there will be a
console (not graphical) exception displayed; one of the exercises at the
end of this chapter addresses this.

Now that we have Glade\'s XML file loaded, you will see a bunch of calls
to `xmlGetWidget`. This Glade function is used to load the XML
definition of a widget, and return a GTK+ widget type for that widget.
We have to pass along to that function a value indicating what GTK+ type
we expect---we\'ll get a runtime error if these don\'t match.

We start by creating a widget for the main window. It is loaded from the
XML widget defined with name `"mainWindow"` and stored in the `mw`
variable. We then use pattern matching and `mapM` to load up all the
buttons. Then, we have two dialogs, a label, and an entry to load.
Finally, we use all of these to build up the GUI type and return it.
Next, we need to set our callback functions up as event handlers.

<div>

[PodMainGUI.hs]{.label}

``` {.haskell}
connectGui gui dbh =
    do -- When the close button is clicked, terminate GUI loop
       -- by calling GTK mainQuit function
       onDestroy (mainWin gui) mainQuit

       -- Main window buttons
       onClicked (mwAddBt gui) (guiAdd gui dbh)
       onClicked (mwUpdateBt gui) (guiUpdate gui dbh)
       onClicked (mwDownloadBt gui) (guiDownload gui dbh)
       onClicked (mwFetchBt gui) (guiFetch gui dbh)
       onClicked (mwExitBt gui) mainQuit

       -- We leave the status window buttons for later
```

</div>

We start out the `connectGui` function by calling `onDestroy`. This
means that when somebody clicks on the operating system\'s close button
(typically an X in the titlebar on Windows or Linux, or a red circle on
Mac OS X), on the main window, we call the `mainQuit` function.
`mainQuit` closes all GUI windows and terminates the GTK+ main loop.

Next, we call `onClicked` to register event handlers for clicking on our
five different buttons. For buttons, these handlers are also called if
the user selects the button via the keyboard. Clicking on these buttons
will call our functions such as `guiAdd`, passing along the GUI record
as well as a database handle.

At this point, we have completely defined the main window for the GUI
podcatcher. It looks like this:

![](figs/gui-pod-mainwin.png)

The Add Podcast Window
----------------------

Now that we\'ve covered the main window, let\'s talk about the other
windows that our application presents, starting with the Add Podcast
window. When the user clicks the button to add a new podcast, we need to
pop up a dialog box to prompt for the URL of the podcast. We have
defined this dialog box in Glade, so all we need to do is set it up.

<div>

[PodMainGUI.hs]{.label}

``` {.haskell}
guiAdd gui dbh =
    do -- Initialize the add URL window
       entrySetText (awEntry gui) ""
       onClicked (awCancelBt gui) (widgetHide (addWin gui))
       onClicked (awOKBt gui) procOK

       -- Show the add URL window
       windowPresent (addWin gui)
    where procOK =
              do url <- entryGetText (awEntry gui)
                 widgetHide (addWin gui) -- Remove the dialog
                 add dbh url             -- Add to the DB
```

</div>

We start by calling `entrySetText` to set the contents of the entry box
(the place where the user types in the URL) to the empty string. That\'s
because the same widget gets reused over the lifetime of the program,
and we don\'t want the last URL the user entered to remain there. Next,
we set up actions for the two buttons in the dialog. If the users clicks
on the cancel button, we simply remove the dialog box from the screen by
calling `widgetHide` on it. If the user clicks the OK button, we call
`procOK`.

`procOK` starts by retrieving the supplied URL from the entry widget.
Next, it uses `widgetHide` to get rid of the dialog box. Finally, it
calls `add` to add the URL to the database. This `add` is exactly the
same function as we had in the non-GUI version of the program.

The last thing we do in `guiAdd` is actually display the pop-up window.
That\'s done by calling `windowPresent`, which is the opposite of
`widgetHide`.

Note that the `guiAdd` function returns almost immediately. It just sets
up the widgets and causes the box to be displayed; at no point does it
block waiting for input. Here\'s what the dialog box looks like:

![](figs/gui-pod-addwin.png)

Long-Running Tasks
------------------

As we think about the buttons available in the main window, three of
them correspond to tasks that could take a while to complete: update,
download, and fetch. While these operations take place, we\'d like to do
two things with our GUI: provide the user with the status of the
operation, and provide the user with the ability to cancel the operation
as it is in progress.

Since all three of these things are very similar operations, it makes
sense to provide a generic way to handle this interaction. We have
defined a single status window widget in the Glade file that will be
used by all three of these. In our Haskell source code, we\'ll define a
generic `statusWindow` function that will be used by all three of these
operations as well.

`statusWindow` takes four parameters: the GUI information, the database
information, a `String` giving the title of the window, and a function
that will perform the operation. This function will itself be passed a
function that it can call to report its progress. Here\'s the code:

<div>

[PodMainGUI.hs]{.label}

``` {.haskell}
statusWindow :: IConnection conn =>
                GUI
             -> conn
             -> String
             -> ((String -> IO ()) -> IO ())
             -> IO ()
statusWindow gui dbh title func =
    do -- Clear the status text
       labelSetText (swLabel gui) ""

       -- Disable the OK button, enable Cancel button
       widgetSetSensitivity (swOKBt gui) False
       widgetSetSensitivity (swCancelBt gui) True

       -- Set the title
       windowSetTitle (statusWin gui) title

       -- Start the operation
       childThread <- forkIO childTasks

       -- Define what happens when clicking on Cancel
       onClicked (swCancelBt gui) (cancelChild childThread)

       -- Show the window
       windowPresent (statusWin gui)
    where childTasks =
              do updateLabel "Starting thread..."
                 func updateLabel
                 -- After the child task finishes, enable OK
                 -- and disable Cancel
                 enableOK

          enableOK =
              do widgetSetSensitivity (swCancelBt gui) False
                 widgetSetSensitivity (swOKBt gui) True
                 onClicked (swOKBt gui) (widgetHide (statusWin gui))
                 return ()

          updateLabel text =
              labelSetText (swLabel gui) text
          cancelChild childThread =
              do killThread childThread
                 yield
                 updateLabel "Action has been cancelled."
                 enableOK
```

</div>

This function starts by clearing the label text from the last run. Next,
we disable (gray out) the OK button and enable the cancel button. While
the operation is in progress, clicking OK doesn\'t make much sense. And
when it\'s done, clicking Cancel doesn\'t make much sense.

Next, we set the title of the window. The title is the part that is
displayed by the system in the title bar of the window. Finally, we
start off the new thread (represented by `childTasks`) and save off its
thread ID. Then, we define what to do if the user clicks on Cancel ---we
call `cancelChild`, passing along the thread ID. Finally, we call
`windowPresent` to show the status window.

In `childTasks`, we display a message saying that we\'re starting the
thread. Then we call the actual worker function, passing `updateLabel`
as the function to use for displaying status messages. Note that a
command-line version of the program could pass `putStrLn` here.

Finally, after the worker function exits, we call `enableOK`. This
function disables the cancel button, enables the OK button, and defines
that a click on the OK button causes the status window to go away.

`updateLabel` simply calls `labelSetText` on the label widget to update
it with the displayed text. Finally, `cancelChild` kills the thread
processing the task, updates the label, and enables the OK button.

We now have the infrastructure in place to define our three GUI
functions. They look like this:

<div>

[PodMainGUI.hs]{.label}

``` {.haskell}
guiUpdate :: IConnection conn => GUI -> conn -> IO ()
guiUpdate gui dbh =
    statusWindow gui dbh "Pod: Update" (update dbh)

guiDownload gui dbh =
    statusWindow gui dbh "Pod: Download" (download dbh)

guiFetch gui dbh =
    statusWindow gui dbh "Pod: Fetch"
                     (\logf -> update dbh logf >> download dbh logf)
```

</div>

For brevity, we have given the type for only the first one, but all
three have the same type, and Haskell can work them out via type
inference. Notice our implementation of `guiFetch`. We don\'t call
`statusWindow` twice, but rather combine functions in its action.

The final piece of the puzzle consists of the three functions that do
our work. `add` is unmodified from the command-line chapter. `update`
and `download` are modified only to take a logging function instead of
calling `putStrLn` for status updates.

<div>

[PodMainGUI.hs]{.label}

``` {.haskell}
add dbh url =
    do addPodcast dbh pc
       commit dbh
    where pc = Podcast {castId = 0, castURL = url}

update :: IConnection conn => conn -> (String -> IO ()) -> IO ()
update dbh logf =
    do pclist <- getPodcasts dbh
       mapM_ procPodcast pclist
       logf "Update complete."
    where procPodcast pc =
              do logf $ "Updating from " ++ (castURL pc)
                 updatePodcastFromFeed dbh pc

download dbh logf =
    do pclist <- getPodcasts dbh
       mapM_ procPodcast pclist
       logf "Download complete."
    where procPodcast pc =
              do logf $ "Considering " ++ (castURL pc)
                 episodelist <- getPodcastEpisodes dbh pc
                 let dleps = filter (\ep -> epDone ep == False)
                             episodelist
                 mapM_ procEpisode dleps
          procEpisode ep =
              do logf $ "Downloading " ++ (epURL ep)
                 getEpisode dbh ep
```

</div>

Here\'s what the final result looks like after running an update:

![](figs/gui-update-complete.png)

Using Cabal
-----------

We presented a Cabal file to build this project for the command-line
version in [the section called \"Main
Program\"](22-web-client-programming.org::*Main Program) for it to work
with our GUI version. First, there\'s the obvious need to add the gtk2hs
packages to the list of build dependencies. There is also the matter of
the Glade XML file.

Earlier, we wrote a `PodLocalMain.hs` that simply assumed this file was
named `podresources.glade` and stored in the current working directory.
For a real, system-wide installation, we can\'t make that assumption.
Moreover, different systems may place the file at different locations.

Cabal provides a way around this problem. It automatically generates a
module that exports functions that can interrogate the environment. We
must add a `Data-files` line to our Cabal description file. This file
names all data files that will be part of a system-wide installation.
Then, Cabal will export a `Paths_pod` module (the \"pod\" part comes
from the `Name` line in the Cabal file) that we can interrogate for the
location at runtime. Here\'s our new Cabal description file:

<div>

[pod.cabal]{.label}

    Name: pod
    Version: 1.0.0
    Build-type: Simple
    Build-Depends: HTTP, HaXml, network, HDBC, HDBC-sqlite3, base, 
                   gtk, glade
    Data-files: podresources.glade

    Executable: pod
    Main-Is: PodCabalMain.hs
    GHC-Options: -O2

</div>

And, to go with it, `PodCabalMain.hs`:

<div>

[PodCabalMain.hs]{.label}

``` {.haskell}
module Main where

import qualified PodMainGUI
import Paths_pod(getDataFileName)

main =
    do gladefn <- getDataFileName "podresources.glade"
       PodMainGUI.main gladefn
```

</div>

Exercises
---------

1.  Present a helpful GUI error message if the call to `xmlNew` returns
    `Nothing`.
2.  Modify the podcatcher to be able to run with either the GUI or the
    command-line interface from a single code base. Hint: move common
    code out of `PodMainGUI.hs`, then have two different `main` modules,
    one for the GUI, and one for the command line.
3.  Why does `guiFetch` combine worker functions instead of calling
    `statusWindow` twice?

Footnotes
---------

[^1]: Several alternatives also exist. Alongside gtk2hs, wxHaskell is
    also a prominent cross-platform GUI toolkit.
